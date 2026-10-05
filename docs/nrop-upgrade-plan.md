# Plan: Add NROP Upgrade Ansible Role to eco-ci-cd

## Context

This change adds NUMA Resources Operator (NROP) upgrade capability to the main eco-ci-cd repository (`/home/agent/work`). Currently, eco-ci-cd has NROP configuration and testing roles (`nrop_config`, `deploy_nrop_tests`) but lacks upgrade functionality. A complete NROP upgrade implementation exists in cnf-pipeline2 at `/workspace/cnf-pipeline2/cnf-features/roles/upgrade/nrop/`, which needs to be ported and integrated into eco-ci-cd following its architectural patterns.

**Why this is needed**: The NROP upgrade role enables automated NUMA Resources Operator upgrades across:
- **Z-stream upgrades**: Same minor version (e.g., 4.16.7 → 4.16.8)
- **Minor upgrades**: Sequential minor versions (e.g., 4.16 → 4.17)
- **Skip-version upgrades**: Non-sequential minor versions (e.g., 4.18 → 4.20, skipping 4.19) using canary mode

This supports both production and stage registries, OLMv0 and OLMv1, and integrates with multi-hop upgrade pipelines for CNF testing.

## Architecture Overview

### Current Implementation (cnf-pipeline2)

**Primary Components**:

1. **Ansible Role**: `/workspace/cnf-pipeline2/cnf-features/roles/upgrade/nrop/`
   - Entry: `tasks/main.yaml` → `tasks/upgrade-nrop.yaml`
   - OLMv0 paths: prod→stage, stage→prod, prod→prod, stage→stage
   - OLMv1 support in `olmv1/` subdirectory
   - Templates: 10+ Jinja2 files for K8s resources
   - Vars: Upgrade path mappings, version detection

2. **Wrapper Scripts**:
   - `scripts/check_nrop_upgrades.sh`: Calculates old/new versions, calls `nrop-upgrade.yaml`
   - `scripts/nrop-canary-skip-upgrade.sh`: Bash-based canary orchestration (setup/migrate/cleanup)

3. **Playbooks**:
   - `cnf-features/nrop-upgrade.yaml`: Simple wrapper that includes the upgrade role
   - `cnf-features/upgrade-nrop-operator.yaml`: Operator-only upgrade (catalog + subscription, no CR changes)

4. **Jenkins Pipeline**: `Jenkinsfiles/nrop-range-upgrade/Jenkinsfile`
   - Supports both Range and Canary modes
   - Auto-selects z-streams using EUS upgrade graph
   - Handles pending EUS releases via two-graph resolution (production + CI nightly)
   - Cross-major support (4.22 → 5.0)

**Upgrade Workflow Patterns**:

### Pattern 1: Range Upgrade (Sequential)
Used for: All upgrade types when NOT using canary mode

```
For each hop from source → target:
  1. Upgrade OCP cluster to version X
  2. Upgrade NROP operator to version X (catalog + subscription + CR + scheduler)
  3. Wait for worker-cnf MCP to stabilize
  4. Verify deployment
  5. Run smoke tests
```

**Example**: 4.18 → 4.21 becomes three sequential hops:
- Hop 1: 4.18 → 4.19 (OCP + NROP)
- Hop 2: 4.19 → 4.20 (OCP + NROP)
- Hop 3: 4.20 → 4.21 (OCP + NROP)

**MCP**: Uses `worker-cnf` MachineConfigPool

### Pattern 2: Canary Skip-Version Upgrade
Used for: Skip-version upgrades (e.g., 4.18 → 4.20)

```
Phase 1: Setup
  - Patch ClusterVersion to target EUS channel
  - Annotate NROP CR with pause-reconciliation=enabled
  - Create worker-upgrade MCP with paused=true, maxUnavailable=0

Phase 2: Upgrade NROP Operator Only
  - Update CatalogSource to target version
  - Update Subscription to target version channel
  - Wait for CSV to reach Succeeded
  - Update NUMAResourcesScheduler image
  - Verify pause-reconciliation annotation remains intact

Phase 3: Control Plane Only Upgrade
  - For each intermediate version:
    - Upgrade CP to version X (--to-image or --to-latest)
    - Wait for ClusterVersion Available=True
  - Workers remain on source version

Phase 4: Migrate Workers
  - For each worker node (or first N if CANARY_NODE_COUNT > 0):
    - Add node to worker-upgrade MCP selector
    - Unpause worker-upgrade MCP temporarily
    - Wait for node to upgrade and become Ready
    - Verify RTE pod is Running on upgraded node
    - Re-pause worker-upgrade MCP

Phase 5: Cleanup
  - Unpause NROP CR reconciliation
  - Delete worker-upgrade MCP (workers return to base worker MCP)
  - Wait for worker MCP to stabilize
  - Verify all RTE pods are Running
```

**MCP**: Uses `worker` MachineConfigPool + creates temporary `worker-upgrade` MCP

**Critical Annotations**:
- `config.numa-operator.openshift.io/pause-reconciliation: enabled` on NROP CR
- Prevents NROP operator from reconciling during CP-only upgrade
- Removed only in cleanup phase after workers are migrated

### Pattern 3: Z-stream Upgrade
Used for: Same minor version (e.g., 4.16.7 → 4.16.8)

```
1. Update NROP operator (catalog + subscription + scheduler)
2. Wait for CSV to reach Succeeded
3. Verify deployment
```

**Note**: OCP cluster upgrade is handled separately; NROP role only upgrades the operator

### Key Variables from cnf-pipeline2

**From Jenkins/Scripts**:
- `UPGRADE_TYPE`: "minor" or "zstream"
- `UPGRADE_FROM`: "prod" or "stage" (source registry)
- `UPGRADE_TO`: "prod" or "stage" (target registry)
- `OCP_MAJOR_VERSION`: e.g., "4"
- `OCP_MINOR_VERSION`: e.g., "17"
- `LONG_UPGRADE`: "true" for multi-hop scenarios (each hop sets old=new)
- `VERIFY_DEPLOYMENT`: "true" to enable post-upgrade validation
- `SKIP_OP_INSTALL`: "true" to skip initial install
- `SKIP_OP_UPGRADE`: "true" to skip upgrade (install-only mode)
- `DEBUG`: "true" for verbose logging
- `NROP_MCP_TARGET`: "worker-cnf" (range) or "worker" (canary)
- `NROP_TARGET_VERSION`: "4.17" (for operator-only upgrade)
- `CANARY_SKIP_UPGRADE`: "true" to enable canary mode
- `CANARY_NODE_COUNT`: "0" for all nodes, ">0" for partial migration
- `OCP_TARGET_VERSION`: Target OCP version for EUS channel

**To Ansible Role**:
- `upgrade`: "minor" or "zstream"
- `upgrade_from`: "prod" or "stage"
- `upgrade_to`: "prod" or "stage"
- `old_ocp_version`: "4.16" (calculated by wrapper script)
- `new_ocp_version`: "4.17" (calculated by wrapper script)
- `verify_deployment`: true/false

### Target eco-ci-cd Architecture

**Location**: `/home/agent/work/playbooks/`

**Existing Patterns to Follow**:
- Roles in `playbooks/compute/roles/` (compute-specific)
- Cluster upgrade at `playbooks/compute/roles/cluster_upgrade/` - reference pattern
- NROP config at `playbooks/compute/roles/nrop_config/` - variable naming
- Operator roles at `playbooks/roles/` (e.g., `ocp_operator_deployment`, `ocp_operator_mirror`)

**Integration Points**:
- Use `ocp_version_facts` role instead of NTO role dependency
- Follow cluster_upgrade pattern: pre-state → upgrade → post-state → artifacts
- Use eco-ci-cd variable naming: `kubeconfig`, `report_folder`, `manifest_dir`
- Support all three upgrade types: zstream, minor, skip-version

## Recommended Approach

**Key Decisions** (confirmed):
- ✅ **Full OLM Support**: Include both OLMv0 and OLMv1 for complete feature parity
- ✅ **Standalone Playbook**: Independent playbook at `playbooks/compute/nrop_upgrade.yml`
- ✅ **Compute Subdirectory**: Role location at `playbooks/compute/roles/nrop_upgrade/`
- ✅ **All Upgrade Types**: Support zstream, minor, and skip-version (canary)

### Phase 1: Port Core NROP Upgrade Role

**Target**: `/home/agent/work/playbooks/compute/roles/nrop_upgrade/`

**Directory Structure**:
```
playbooks/compute/roles/nrop_upgrade/
├── tasks/
│   ├── main.yml                    # Entry: delegates to upgrade type
│   ├── upgrade-nrop.yml            # Core orchestration (OLMv0/v1 routing)
│   ├── olmv0/
│   │   ├── install.yml             # OLMv0 installation
│   │   ├── prod-prod.yml           # Prod → Prod upgrade
│   │   ├── prod-stage.yml          # Prod → Stage upgrade
│   │   ├── stage-prod.yml          # Stage → Prod upgrade
│   │   └── stage-stage.yml         # Stage → Stage upgrade
│   ├── olmv1/
│   │   ├── install.yml             # OLMv1 installation
│   │   └── upgrade.yml             # OLMv1 upgrade path
│   ├── deploy-scheduler.yml        # Scheduler deployment with digest verification
│   ├── verify-deployment.yml       # Post-upgrade validation
│   ├── apply-certs.yml             # Certificate and image policy application
│   ├── collect-pre-state.yml       # Pre-upgrade artifact collection
│   ├── collect-post-state.yml      # Post-upgrade artifact collection
│   └── wait-mcp.yml                # MCP stabilization wait logic
├── templates/
│   ├── catalog_source.yaml.j2
│   ├── operator_subscription.yaml.j2
│   ├── pool_nrop.yaml.j2           # NROP CR (poolName for 4.18+, MCP for ≤4.17)
│   ├── scheduler.yaml.j2
│   ├── test_deployment.yaml.j2
│   ├── image_source_policy.yaml.j2
│   ├── certificate.yaml.j2
│   └── ... (all templates from cnf-pipeline2)
├── defaults/
│   └── main.yml                    # Default variables
└── vars/
    └── main.yml                    # Upgrade path mappings, version logic
```

**Key Adaptations**:

1. **Replace NTO Dependency**:
   ```yaml
   # OLD (cnf-pipeline2):
   - include_tasks: ../../nto/tasks/get_version.yaml
   
   # NEW (eco-ci-cd):
   - name: Get OCP version information
     ansible.builtin.include_role:
       name: ocp_version_facts
     vars:
       release: "{{ new_ocp_version }}"
   
   - name: Set OCP version facts
     ansible.builtin.set_fact:
       from_version: "{{ old_ocp_version }}"
       to_version: "{{ ocp_version_facts_major }}.{{ ocp_version_facts_minor }}"
   ```

2. **Variable Naming Alignment**:
   - `cnf_kubeconfig` → `kubeconfig`
   - Add `report_folder` for artifacts (like cluster_upgrade)
   - Add `manifest_dir` for generated manifests
   - Keep role-specific: `nrop_namespace`, `nrop_mcp_target`, `upgrade`, `upgrade_from`, `upgrade_to`

3. **MCP Waiting Logic**:
   - Extract from cnf-pipeline2 bash script into Ansible task
   - Reuse cluster_upgrade MCP wait pattern where applicable
   - Support configurable retries and delays

4. **Artifact Collection**:
   - Pre-state: CSV versions, scheduler images, NROP CR state, MCP status
   - Post-state: Same as pre-state for comparison
   - Save to `{{ report_folder }}/nrop-upgrade/` with timestamps

### Phase 2: Create Operator-Only Upgrade Playbook

**Target**: `/home/agent/work/playbooks/compute/nrop_operator_upgrade.yml`

This is a **direct port** of `/workspace/cnf-pipeline2/cnf-features/upgrade-nrop-operator.yaml` with minimal changes:
- Update paths to use eco-ci-cd role location
- Align variable names with eco-ci-cd conventions
- Keep all validation logic intact (scheduler tests, pause-reconciliation check)

**Purpose**: Used in canary mode to upgrade NROP operator **before** CP upgrade

### Phase 3: Create Canary Upgrade Support

**Option A**: Port bash script to Ansible tasks
- Create `tasks/canary/` subdirectory with setup.yml, migrate.yml, cleanup.yml
- Advantage: Pure Ansible, easier to maintain
- Disadvantage: More porting work, need to replicate bash logic

**Option B**: Keep bash script wrapper (RECOMMENDED)
- Port `/workspace/cnf-pipeline2/scripts/nrop-canary-skip-upgrade.sh` to `/home/agent/work/scripts/`
- Advantage: Proven logic, less porting risk
- Disadvantage: Mixed Ansible/bash approach

**Recommendation**: Use Option B initially, refactor to pure Ansible later if needed

**Canary Script Requirements**:
- Replace hardcoded paths: `/opt/cnf-features` → use environment variable
- Support eco-ci-cd kubeconfig conventions
- Integrate with eco-ci-cd artifact collection

### Phase 4: Create Main Playbooks

#### 4A: Range Upgrade Playbook

**Target**: `/home/agent/work/playbooks/compute/nrop_upgrade.yml`

```yaml
---
- name: NROP Upgrade (Range Mode)
  hosts: localhost
  gather_facts: false
  environment:
    KUBECONFIG: "{{ kubeconfig }}"
  
  tasks:
    - name: Ensure required variables are defined
      ansible.builtin.assert:
        that:
          - kubeconfig is defined
          - new_ocp_version is defined
          - upgrade is defined
          - upgrade_from is defined
          - upgrade_to is defined
        fail_msg: "Required variables missing (kubeconfig, new_ocp_version, upgrade, upgrade_from, upgrade_to)"
    
    - name: Create report folder
      ansible.builtin.file:
        path: "{{ report_folder }}"
        state: directory
        mode: '0755'
      when: report_folder is defined
    
    - name: Get OCP version information
      ansible.builtin.include_role:
        name: ocp_version_facts
      vars:
        release: "{{ new_ocp_version }}"
    
    - name: Execute NROP upgrade
      ansible.builtin.include_role:
        name: nrop_upgrade
```

**Usage Examples**:
```bash
# Z-stream upgrade (4.16.7 → 4.16.8)
ansible-playbook playbooks/compute/nrop_upgrade.yml \
  -e "kubeconfig=/path/to/kc" \
  -e "old_ocp_version=4.16" \
  -e "new_ocp_version=4.16" \
  -e "upgrade=zstream" \
  -e "upgrade_from=prod" \
  -e "upgrade_to=prod"

# Minor upgrade (4.16 → 4.17)
ansible-playbook playbooks/compute/nrop_upgrade.yml \
  -e "kubeconfig=/path/to/kc" \
  -e "old_ocp_version=4.16" \
  -e "new_ocp_version=4.17" \
  -e "upgrade=minor" \
  -e "upgrade_from=prod" \
  -e "upgrade_to=prod"

# Long upgrade mode (for multi-hop range, each hop called separately)
ansible-playbook playbooks/compute/nrop_upgrade.yml \
  -e "kubeconfig=/path/to/kc" \
  -e "old_ocp_version=4.17" \
  -e "new_ocp_version=4.17" \
  -e "upgrade=minor" \
  -e "upgrade_from=prod" \
  -e "upgrade_to=prod" \
  -e "long_upgrade=true"
```

#### 4B: Wrapper Script (Optional)

**Target**: `/home/agent/work/scripts/nrop_range_upgrade.sh`

Ports the logic from `check_nrop_upgrades.sh`:
```bash
#!/bin/bash
set -ex

new_ocp_version=${OCP_MAJOR_VERSION}.${OCP_MINOR_VERSION}
if [ "${UPGRADE_TYPE}" == "zstream" ]; then
    old_ocp_version=${OCP_MAJOR_VERSION}.${OCP_MINOR_VERSION}
elif [ "${UPGRADE_TYPE}" == "minor" ] && [ "${LONG_UPGRADE}" == "true" ]; then
    old_ocp_version=${OCP_MAJOR_VERSION}.${OCP_MINOR_VERSION}
    new_ocp_version=${OCP_MAJOR_VERSION}.$((OCP_MINOR_VERSION + 1))
else
    old_ocp_version=${OCP_MAJOR_VERSION}.$((OCP_MINOR_VERSION - 1))
fi

ansible-playbook /path/to/playbooks/compute/nrop_upgrade.yml \
  -e "upgrade=${UPGRADE_TYPE}" \
  -e "upgrade_from=${UPGRADE_FROM}" \
  -e "upgrade_to=${UPGRADE_TO}" \
  -e "old_ocp_version=${old_ocp_version}" \
  -e "new_ocp_version=${new_ocp_version}" \
  -e "verify_deployment=${VERIFY_DEPLOYMENT:-false}"
```

### Phase 5: Integration Testing

**Test Scenarios**:

1. **Z-stream Upgrade**:
   ```bash
   ansible-playbook playbooks/compute/nrop_upgrade.yml \
     -e "kubeconfig=/tmp/test-kc" \
     -e "old_ocp_version=4.18" \
     -e "new_ocp_version=4.18" \
     -e "upgrade=zstream" \
     -e "upgrade_from=prod" \
     -e "upgrade_to=prod"
   ```

2. **Minor Upgrade** (single hop):
   ```bash
   ansible-playbook playbooks/compute/nrop_upgrade.yml \
     -e "kubeconfig=/tmp/test-kc" \
     -e "old_ocp_version=4.17" \
     -e "new_ocp_version=4.18" \
     -e "upgrade=minor" \
     -e "upgrade_from=prod" \
     -e "upgrade_to=prod"
   ```

3. **Operator-Only Upgrade** (for canary):
   ```bash
   ansible-playbook playbooks/compute/nrop_operator_upgrade.yml \
     -e "kubeconfig=/tmp/test-kc" \
     -e "nrop_target_version=4.20" \
     -e "nrop_upgrade_source=prod"
   ```

4. **Canary Skip-Version** (full flow):
   ```bash
   # Phase 1: Setup
   bash scripts/nrop-canary-skip-upgrade.sh setup \
     -e NROP_MCP_TARGET=worker \
     -e OCP_TARGET_VERSION=4.20
   
   # Phase 2: Upgrade NROP operator
   ansible-playbook playbooks/compute/nrop_operator_upgrade.yml \
     -e "kubeconfig=/tmp/test-kc" \
     -e "nrop_target_version=4.20" \
     -e "nrop_upgrade_source=prod"
   
   # Phase 3: CP upgrade (separate, handled by cluster_upgrade or external)
   
   # Phase 4: Migrate workers
   bash scripts/nrop-canary-skip-upgrade.sh migrate \
     -e CANARY_NODE_COUNT=0 \
     -e OCP_FROM_MINOR=18
   
   # Phase 5: Cleanup
   bash scripts/nrop-canary-skip-upgrade.sh cleanup \
     -e OCP_FROM_MINOR=18
   ```

5. **Registry Change** (prod → stage):
   ```bash
   ansible-playbook playbooks/compute/nrop_upgrade.yml \
     -e "kubeconfig=/tmp/test-kc" \
     -e "old_ocp_version=4.18" \
     -e "new_ocp_version=4.18" \
     -e "upgrade=zstream" \
     -e "upgrade_from=prod" \
     -e "upgrade_to=stage"
   ```

## Critical Files to Create/Modify

**New Ansible Role Files** (ported from cnf-pipeline2):
- `/home/agent/work/playbooks/compute/roles/nrop_upgrade/tasks/main.yml`
- `/home/agent/work/playbooks/compute/roles/nrop_upgrade/tasks/upgrade-nrop.yml`
- `/home/agent/work/playbooks/compute/roles/nrop_upgrade/tasks/olmv0/*.yml` (5 files)
- `/home/agent/work/playbooks/compute/roles/nrop_upgrade/tasks/olmv1/*.yml` (2 files)
- `/home/agent/work/playbooks/compute/roles/nrop_upgrade/tasks/deploy-scheduler.yml`
- `/home/agent/work/playbooks/compute/roles/nrop_upgrade/tasks/verify-deployment.yml`
- `/home/agent/work/playbooks/compute/roles/nrop_upgrade/tasks/apply-certs.yml`
- `/home/agent/work/playbooks/compute/roles/nrop_upgrade/tasks/collect-pre-state.yml`
- `/home/agent/work/playbooks/compute/roles/nrop_upgrade/tasks/collect-post-state.yml`
- `/home/agent/work/playbooks/compute/roles/nrop_upgrade/tasks/wait-mcp.yml`
- `/home/agent/work/playbooks/compute/roles/nrop_upgrade/defaults/main.yml`
- `/home/agent/work/playbooks/compute/roles/nrop_upgrade/vars/main.yml`
- `/home/agent/work/playbooks/compute/roles/nrop_upgrade/templates/*.j2` (10+ templates)

**New Playbook Files**:
- `/home/agent/work/playbooks/compute/nrop_upgrade.yml` (main playbook)
- `/home/agent/work/playbooks/compute/nrop_operator_upgrade.yml` (operator-only)

**New Script Files** (optional):
- `/home/agent/work/scripts/nrop_range_upgrade.sh` (wrapper for environment variables)
- `/home/agent/work/scripts/nrop-canary-skip-upgrade.sh` (canary orchestration)

**Reference Files** (existing, for reuse):
- `/home/agent/work/playbooks/roles/ocp_version_facts/` - OCP version detection
- `/home/agent/work/playbooks/compute/roles/cluster_upgrade/tasks/main.yml` - Upgrade pattern
- `/home/agent/work/playbooks/compute/roles/nrop_config/` - NROP variable conventions

## Variables Reference

### Required Variables (Ansible Playbook)

**For Range Upgrade**:
- `kubeconfig` - Path to cluster kubeconfig file
- `old_ocp_version` - Source OCP version (e.g., "4.16")
- `new_ocp_version` - Target OCP version (e.g., "4.17")
- `upgrade` - "minor" or "zstream"
- `upgrade_from` - "prod" or "stage" (source registry)
- `upgrade_to` - "prod" or "stage" (target registry)

**For Operator-Only Upgrade** (canary):
- `kubeconfig` - Path to cluster kubeconfig file
- `nrop_target_version` - Target version (e.g., "4.20")
- `nrop_upgrade_source` - "prod" or "stage"

### Optional Variables

- `nrop_namespace` - NROP operator namespace (default: "openshift-numaresources")
- `nrop_mcp_target` - "worker-cnf" or "worker" (default: "worker-cnf")
- `verify_deployment` - Run post-upgrade validation (default: false)
- `report_folder` - Path for artifacts (default: "/tmp/nrop-upgrade-artifacts")
- `manifest_dir` - Path for generated manifests (default: "/tmp/nrop-manifests")
- `long_upgrade` - Set to true for multi-hop range upgrades (default: false)
- `apply_perf_profile` - Apply performance profile (default: false)
- `config_dir` - Directory with pull_secret.json (default: "/opt/config")

### Environment Variables (For Wrapper Scripts)

**From Jenkins/CI**:
- `UPGRADE_TYPE` - "minor" or "zstream"
- `UPGRADE_FROM` - "prod" or "stage"
- `UPGRADE_TO` - "prod" or "stage"
- `OCP_MAJOR_VERSION` - Major version (e.g., "4")
- `OCP_MINOR_VERSION` - Minor version (e.g., "17")
- `LONG_UPGRADE` - "true" for multi-hop
- `VERIFY_DEPLOYMENT` - "true" to enable validation
- `NROP_MCP_TARGET` - "worker-cnf" or "worker"
- `NROP_TARGET_VERSION` - For operator-only upgrade
- `KUBECONFIG` - Path to kubeconfig
- `CONFIG_DIR` - Directory with pull_secret.json
- `ARTIFACTS_DIR` - Output directory for artifacts

**For Canary Script**:
- `CANARY_NODE_COUNT` - "0" for all nodes, ">0" for partial
- `OCP_TARGET_VERSION` - Target OCP version
- `OCP_FROM_MINOR` - Source OCP minor version (for SELinux annotation logic)

### Internal Variables (Set by Role)

- `from_version` / `to_version` - Parsed OCP versions
- `image_uri.prod` / `image_uri.stage` - Registry URLs
- `nrop_registry_ns` - "openshift4" or "openshift5" based on OCP major version
- `catalog_name` - "redhat-operators" or "konflux-stage-catalog"
- `registry` - Full catalog source image URI
- `scheduler` - Full scheduler image URI

## Implementation Strategy

### Porting Priority

1. **Phase 1 (Essential)**: Core role + range upgrade playbook
   - Port `roles/upgrade/nrop/` Ansible role
   - Create `nrop_upgrade.yml` playbook
   - Support OLMv0 only initially (can add OLMv1 later)
   - Focus on prod→prod upgrades
   - Test with z-stream and minor upgrades

2. **Phase 2 (Operator-Only)**: Canary prerequisite
   - Port `upgrade-nrop-operator.yaml` playbook
   - Add all scheduler validation tests
   - Test operator-only upgrade independently

3. **Phase 3 (Canary)**: Skip-version support
   - Port `nrop-canary-skip-upgrade.sh` script
   - Integrate with Phase 2 operator-only playbook
   - Test full canary flow (setup → operator → CP → migrate → cleanup)

4. **Phase 4 (Complete)**: Full feature parity
   - Add OLMv1 support
   - Add all upgrade path combinations (stage→prod, etc.)
   - Add wrapper scripts for Jenkins environment variable compatibility

### Testing Approach

**Incremental Testing**:
1. Syntax validation: `ansible-playbook --syntax-check`
2. Dry run: `ansible-playbook --check` (limited value for k8s resources)
3. Fresh install test: Deploy NROP on clean cluster
4. Z-stream upgrade: Upgrade within same minor
5. Minor upgrade: Single hop (4.17 → 4.18)
6. Multi-hop: Range upgrade (4.17 → 4.20)
7. Operator-only: Canary prerequisite
8. Full canary: Skip-version (4.18 → 4.20)

**Validation Points**:
- CSV reaches Succeeded phase
- Scheduler pods are Running on control-plane nodes
- Scheduler image matches expected version
- RTE pods are Running on all worker nodes
- Test pod can be scheduled via topo-aware-scheduler
- MCP is Updated=True, Updating=False, Degraded=False
- For canary: pause-reconciliation annotation preserved

## Verification Steps

**Post-Implementation Validation**:

1. **Syntax Check**:
   ```bash
   ansible-playbook --syntax-check playbooks/compute/nrop_upgrade.yml
   ansible-playbook --syntax-check playbooks/compute/nrop_operator_upgrade.yml
   ansible-lint playbooks/compute/roles/nrop_upgrade/
   ```

2. **Variable Validation**:
   ```bash
   # Test with missing required variable (should fail)
   ansible-playbook playbooks/compute/nrop_upgrade.yml -e "kubeconfig=/tmp/test" --check
   ```

3. **Fresh Install Test**:
   ```bash
   ansible-playbook playbooks/compute/nrop_upgrade.yml \
     -e "kubeconfig=/path/to/kc" \
     -e "old_ocp_version=4.18" \
     -e "new_ocp_version=4.18" \
     -e "upgrade=minor" \
     -e "upgrade_from=prod" \
     -e "upgrade_to=prod"
   ```

4. **Verify Operator**:
   ```bash
   oc --kubeconfig /path/to/kc get csv -n openshift-numaresources
   oc --kubeconfig /path/to/kc get numaresourcesoperator -A
   oc --kubeconfig /path/to/kc get pods -n openshift-numaresources
   ```

5. **Verify Scheduler**:
   ```bash
   oc --kubeconfig /path/to/kc get deployment -n openshift-numaresources -l app=secondary-scheduler
   oc --kubeconfig /path/to/kc get pods -n openshift-numaresources -l app=secondary-scheduler
   oc --kubeconfig /path/to/kc get numaresourcesscheduler numaresourcesscheduler -o yaml
   ```

6. **Verify RTE DaemonSet**:
   ```bash
   oc --kubeconfig /path/to/kc get pods -n openshift-numaresources -l name=resource-topology -o wide
   # Should show one RTE pod per worker node, all Running
   ```

7. **Test Workload Validation**:
   ```bash
   oc --kubeconfig /path/to/kc get deployment nrop-test-deployment -n default
   oc --kubeconfig /path/to/kc get pods -l app=nrop-test
   ```

8. **Artifact Validation**:
   ```bash
   ls -lah {{ report_folder }}/nrop-upgrade/
   # Should contain:
   # - pre-upgrade-state.json
   # - post-upgrade-state.json
   # - csv-versions.txt
   # - scheduler-image-source.txt
   # - scheduler-image-target.txt
   # - scheduler-validation-junit.xml (if operator-only)
   ```

9. **Canary Validation** (if implemented):
   ```bash
   # After canary migration:
   oc --kubeconfig /path/to/kc get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion
   # Should show target version on all nodes
   
   oc --kubeconfig /path/to/kc get numaresourcesoperator numaresourcesoperator -o yaml | grep pause-reconciliation
   # Should NOT show pause-reconciliation annotation after cleanup
   ```

## Dependencies

**Ansible Collections** (should already be in eco-ci-cd `requirements.yml`):
- `kubernetes.core` >= 2.4.2 - For k8s/k8s_info modules
- `ansible.builtin` - Core modules

**External Tools** (runtime dependencies):
- `oc` - OpenShift CLI (installed via `oc_client_install` role)
- `skopeo` - For image digest inspection
- `jq` - For JSON parsing (in canary script)

**Role Dependencies**:
- `ocp_version_facts` - OCP version detection and parsing
- `ocp_operator_deployment` (reference) - OLM patterns

**Optional** (for full Jenkins integration):
- `git-crypt` - For encrypted configuration in cnf-pipeline2

## Notes

### OLM Versions
- **OLMv0** (traditional): CatalogSource → Subscription → InstallPlan → CSV
- **OLMv1** (new): Simplified catalog format, different resource structure
- Role supports both, auto-detects based on `install_type` variable

### Upgrade Path Combinations
- **prod → prod**: Standard production upgrade
- **stage → stage**: Konflux stage-to-stage (for testing pre-release)
- **prod → stage**: Downgrade to stage (for testing fixes)
- **stage → prod**: Promote from stage to production

### Version-Specific Logic
- **OCP 4.18+**: Use `poolName` selector in NROP CR
- **OCP ≤4.17**: Use MCP-based selector in NROP CR
- **OCP 4.18+ (canary)**: SELinux annotation NOT needed
- **OCP <4.18 (canary)**: SELinux annotation REQUIRED on worker-upgrade MCP

### MachineConfigPool Behavior
- MCP updates may trigger node reboots (kernel args, container runtime, etc.)
- Role includes MCP waiting logic with configurable timeouts
- Default: 120 retries × 30s = 60 minutes max wait
- Degraded MCP triggers diagnostic collection every 10 retries

### Performance Profile Integration
- Optional via `apply_perf_profile=true`
- Integrates with Performance Addon Operator
- Applies cluster-wide performance tuning (hugepages, CPU isolation, RT kernel)

### Scheduler Deployment
- Scheduler image uses digest verification via skopeo
- Ensures image integrity and version accuracy
- Scheduler pods run on control-plane nodes only
- Deployment typically has 1 replica (can scale for HA)

### Artifacts and Reporting
- JUnit XML generated for CI integration
- Skopeo inspection results saved for audit trail
- Pre/post state comparison for validation
- All artifacts timestamped for correlation with pipeline runs

### Canary Mode Advantages
- Reduces downtime: workers upgrade gradually
- Allows validation per-node before proceeding
- Supports partial migration (e.g., 2 out of 10 workers)
- Can skip intermediate versions (4.18 → 4.20 without 4.19)

### Canary Mode Risks
- More complex orchestration
- Requires pause-reconciliation annotation management
- Temporary MCP fragmentation (worker + worker-upgrade)
- SELinux annotation needed for pre-4.18 sources

### Jenkins Integration
- Jenkins sets environment variables
- Wrapper scripts translate to Ansible variables
- Artifacts archived via Jenkins archiveArtifacts
- JUnit reports parsed via Jenkins junit plugin
- Skopeo digests collected for release tracking
