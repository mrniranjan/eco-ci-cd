# NROP Upgrade Implementation Roadmap (REVISED)

**Based on Gap Analysis**: See `docs/nrop-upgrade-gap-analysis.md` for detailed reuse strategy

**Key Finding**: eco-ci-cd already has 50% of what we need. Focus on adding upgrade-specific logic while reusing existing components.

---

## Reuse Strategy Summary

### ✅ What We're REUSING from eco-ci-cd

| Component | Source | Usage |
|-----------|--------|-------|
| Scheduler CR template | `nrop_config/templates/scheduler.yaml.j2` | Copy to `nrop_upgrade/templates/pool_nrop.yaml.j2` |
| Registry namespace detection | `nrop_config/tasks/main.yml:25-33` | Extract to `nrop_upgrade/vars/main.yml` |
| Skopeo SHA resolution | `nrop_config/tasks/deploy_scheduler.yml:59-83` | Adapt to `nrop_upgrade/tasks/deploy-scheduler.yml` |
| MCP wait logic | `nrop_config/tasks/wait_for_mcp.yml` | Enhance to `nrop_upgrade/tasks/wait-mcp.yml` |
| Performance profile | `nrop_config/tasks/preformance_profile.yml` | Reference as optional step |
| OCP version detection | Inline in `nrop_config` | Replace with `ocp_version_facts` role |

### ❌ What We're ADDING (Not in eco-ci-cd)

| Component | Why Needed |
|-----------|------------|
| CatalogSource templates | Fresh install & upgrade path switching |
| Subscription channel modification | Minor version upgrades (4.17→4.18) |
| CSV wait and validation | Verify operator reached Succeeded |
| Upgrade path routing | prod→prod, stage→stage, prod→stage, stage→prod |
| OperatorGroup/Namespace creation | Fresh installs via upgrade role |
| Pre/post upgrade state collection | Audit trail and validation |
| OLMv0/v1 support | Future-proofing |
| Canary orchestration | Skip-version upgrades (4.18→4.20) |

---

## Phase 1: Core Upgrade Role (Essential)

**Goal**: Support z-stream and minor upgrades with maximum reuse

**Scope**: OLMv0 only, prod→prod registry path only

**Effort**: 2-3 days (70% reuse, 30% new)

### 1.1 Create Role Structure

- [ ] Create `/home/agent/work/playbooks/compute/roles/nrop_upgrade/` directory
- [ ] Create subdirectories: `tasks/olmv0/`, `templates/`, `defaults/`, `vars/`

### 1.2 Reuse Existing Templates

- [ ] **COPY** `scheduler.yaml.j2` from `nrop_config/templates/`
  - Target: `nrop_upgrade/templates/pool_nrop.yaml.j2`
  - No changes needed - already has 4.18+ poolName logic

- [ ] **COPY** `performanceprofile.yaml.j2` from `nrop_config/templates/`
  - Target: `nrop_upgrade/templates/performance_profile.yaml.j2`
  - For optional performance profile integration

### 1.3 Add New Templates

- [ ] Create `templates/catalog_source.yaml.j2`
  ```jinja2
  apiVersion: operators.coreos.com/v1alpha1
  kind: CatalogSource
  metadata:
    name: {{ catalog_name }}
    namespace: openshift-marketplace
  spec:
    sourceType: grpc
    image: {{ registry }}
    displayName: "{{ catalog_display_name }}"
    publisher: Red Hat
  ```

- [ ] Create `templates/operator_subscription.yaml.j2`
  ```jinja2
  apiVersion: operators.coreos.com/v1alpha1
  kind: Subscription
  metadata:
    name: numaresources-operator
    namespace: {{ nrop_namespace }}
  spec:
    channel: "{{ target_version }}"
    name: numaresources-operator
    source: {{ catalog_name }}
    sourceNamespace: openshift-marketplace
  ```

- [ ] Create `templates/operator_group.yaml.j2`
  ```jinja2
  apiVersion: operators.coreos.com/v1
  kind: OperatorGroup
  metadata:
    name: numaresources-operator
    namespace: {{ nrop_namespace }}
  spec:
    targetNamespaces:
      - {{ nrop_namespace }}
  ```

- [ ] Create `templates/namespace.yaml.j2`
  ```jinja2
  apiVersion: v1
  kind: Namespace
  metadata:
    name: {{ nrop_namespace }}
  ```

### 1.4 Extract and Adapt Existing Logic

- [ ] **EXTRACT** registry namespace detection
  - Source: `nrop_config/tasks/main.yml:25-33`
  - Target: `nrop_upgrade/vars/main.yml`
  ```yaml
  nrop_registry_ns: "{{ 'openshift5' if (ocp_version_major | int) >= 5 else 'openshift4' }}"
  image_uri:
    prod: "registry.redhat.io/{{ nrop_registry_ns }}"
    stage: "registry.stage.redhat.io/{{ nrop_registry_ns }}"
  ```

- [ ] **ADAPT** skopeo SHA resolution
  - Source: `nrop_config/tasks/deploy_scheduler.yml:59-83`
  - Target: `nrop_upgrade/tasks/deploy-scheduler.yml`
  - Changes: Make scheduler image URI generation more flexible

- [ ] **ENHANCE** MCP wait logic
  - Source: `nrop_config/tasks/wait_for_mcp.yml`
  - Target: `nrop_upgrade/tasks/wait-mcp.yml`
  - Enhancement: Add per-MCP targeting, diagnostic collection

### 1.5 Add New Core Tasks

- [ ] Create `tasks/main.yml` - Entry point
  ```yaml
  - name: Get OCP version using ocp_version_facts role
    ansible.builtin.include_role:
      name: ocp_version_facts
    vars:
      release: "{{ new_ocp_version }}"
  
  - name: Execute upgrade
    ansible.builtin.include_tasks: upgrade-nrop.yml
  ```

- [ ] Create `tasks/upgrade-nrop.yml` - Orchestration
  ```yaml
  - name: Determine upgrade path
    ansible.builtin.set_fact:
      upgrade_path: "{{ upgrade_from }}-{{ upgrade_to }}"
  
  - name: Execute OLMv0 upgrade
    ansible.builtin.include_tasks: "olmv0/{{ upgrade_path }}.yml"
    when: install_type == 'olmv0'
  ```

- [ ] Create `tasks/olmv0/install.yml` - Fresh installation
  - Create namespace
  - Create OperatorGroup
  - Apply CatalogSource
  - Apply Subscription
  - Wait for CSV Succeeded
  - Deploy NUMAResourcesOperator CR (reuse pool_nrop.yaml.j2)
  - Deploy NUMAResourcesScheduler (call deploy-scheduler.yml)
  - Wait for MCP (call wait-mcp.yml)

- [ ] Create `tasks/olmv0/prod-prod.yml` - Production → Production upgrade
  - Update CatalogSource to new version
  - Update Subscription channel
  - Wait for new CSV Succeeded
  - Update NUMAResourcesScheduler (call deploy-scheduler.yml)
  - Wait for MCP (call wait-mcp.yml)

- [ ] Create `tasks/collect-pre-state.yml` - Pre-upgrade state
  ```yaml
  - name: Get current CSV
    kubernetes.core.k8s_info:
      api_version: operators.coreos.com/v1alpha1
      kind: ClusterServiceVersion
      namespace: "{{ nrop_namespace }}"
    register: pre_csv
  
  - name: Get current scheduler image
    kubernetes.core.k8s_info:
      api_version: nodetopology.openshift.io/v1
      kind: NUMAResourcesScheduler
      name: numaresourcesscheduler
    register: pre_scheduler
  
  - name: Save pre-upgrade state
    ansible.builtin.copy:
      content: "{{ {'csv': pre_csv, 'scheduler': pre_scheduler} | to_nice_json }}"
      dest: "{{ report_folder }}/nrop-upgrade/pre-state.json"
  ```

- [ ] Create `tasks/collect-post-state.yml` - Post-upgrade state (same as pre)

- [ ] Create `tasks/verify-deployment.yml` - Post-upgrade validation
  - Port from cnf-pipeline2 `verify-deployment.yaml`
  - Creates test pod using topo-aware-scheduler
  - Verifies pod runs successfully

### 1.6 Create Variables

- [ ] Create `defaults/main.yml`
  ```yaml
  kubeconfig: "/root/.kube/config"
  nrop_namespace: "openshift-numaresources"
  nrop_mcp_target: "worker-cnf"
  verify_deployment: false
  report_folder: "/tmp/nrop-upgrade-artifacts"
  manifest_dir: "/tmp/nrop-manifests"
  config_dir: "/opt/config"
  install_type: "olmv0"
  upgrade: "minor"  # or "zstream"
  upgrade_from: "prod"
  upgrade_to: "prod"
  ```

- [ ] Create `vars/main.yml`
  ```yaml
  # Extracted from nrop_config
  nrop_registry_ns: "{{ 'openshift5' if (ocp_version_major | int) >= 5 else 'openshift4' }}"
  image_uri:
    prod: "registry.redhat.io/{{ nrop_registry_ns }}"
    stage: "registry.stage.redhat.io/{{ nrop_registry_ns }}"
  
  # Catalog names
  catalog_names:
    prod: "redhat-operators"
    stage: "konflux-stage-catalog"
  ```

### 1.7 Create Main Playbook

- [ ] Create `playbooks/compute/nrop_upgrade.yml`
  ```yaml
  ---
  - name: NROP Upgrade
    hosts: localhost
    gather_facts: false
    environment:
      KUBECONFIG: "{{ kubeconfig }}"
    
    tasks:
      - name: Validate required variables
        ansible.builtin.assert:
          that:
            - kubeconfig is defined
            - new_ocp_version is defined
            - upgrade is defined
          fail_msg: "Required variables missing"
      
      - name: Execute upgrade
        ansible.builtin.include_role:
          name: nrop_upgrade
  ```

### 1.8 Testing

- [ ] Syntax check: `ansible-playbook --syntax-check playbooks/compute/nrop_upgrade.yml`
- [ ] Test fresh installation on 4.18 cluster
- [ ] Test z-stream upgrade (4.18.0 → 4.18.1)
- [ ] Test minor upgrade (4.17 → 4.18)
- [ ] Verify reused templates work (scheduler poolName vs. MCP logic)
- [ ] Verify skopeo SHA resolution works
- [ ] Verify MCP wait works
- [ ] Verify artifacts collected in report_folder

**Deliverable**: Working NROP upgrades reusing 70% of existing eco-ci-cd code

---

## Phase 2: Integration Documentation

**Goal**: Document how upgrade role fits with existing playbooks

**Effort**: 1 day

### 2.1 Update CLAUDE.md

- [ ] Add NROP upgrade section
  ```markdown
  ### NROP Upgrade Pattern
  
  **Fresh Install** (use existing generic operator deployment):
  ```bash
  ansible-playbook playbooks/deploy-ocp-operators.yml \
    -e "operators=[{'name':'numaresources-operator',...}]"
  ```
  
  **Post-Install Configuration** (use existing nrop_config):
  ```bash
  ansible-playbook playbooks/compute/configure-nrop-operator.yml
  ```
  
  **Upgrade** (use new nrop_upgrade role):
  ```bash
  ansible-playbook playbooks/compute/nrop_upgrade.yml \
    -e "old_ocp_version=4.17 new_ocp_version=4.18 upgrade=minor"
  ```
  ```

### 2.2 Create Usage Guide

- [ ] Create `docs/nrop-upgrade-usage.md`
  - Fresh install examples (reference deploy-ocp-operators.yml)
  - Z-stream upgrade examples
  - Minor upgrade examples
  - Variable reference
  - Troubleshooting guide

### 2.3 Update Roadmap

- [ ] Mark Phase 1 tasks complete
- [ ] Document reuse percentages achieved
- [ ] Note any deviations from original plan

---

## Phase 3: Operator-Only Upgrade (Canary Prerequisite)

**Goal**: Support operator-only upgrades for canary mode

**Effort**: 1-2 days

### 3.1 Create Operator-Only Playbook

- [ ] Create `playbooks/compute/nrop_operator_upgrade.yml`
  - Port from `/workspace/cnf-pipeline2/cnf-features/upgrade-nrop-operator.yaml`
  - Reuse Phase 1 templates and tasks
  - Add 4-test scheduler validation suite

### 3.2 Scheduler Validation Tests

- [ ] Port Test 1: Scheduler CR Conditions (Available, Degraded, Progressing)
- [ ] Port Test 2: Scheduler Image Match
- [ ] Port Test 3: Control-Plane Placement
- [ ] Port Test 4: Test Pod Scheduling via TAS
- [ ] Generate JUnit XML report

### 3.3 Testing

- [ ] Test operator-only upgrade on 4.18 cluster
- [ ] Verify all 4 tests pass
- [ ] Verify JUnit XML generated

**Deliverable**: Standalone operator-only upgrade playbook

---

## Phase 4: Canary Skip-Version Support

**Goal**: Support skip-version upgrades

**Effort**: 2-3 days

### 4.1 Port Canary Script

- [ ] Create `scripts/nrop-canary-skip-upgrade.sh`
  - Port from cnf-pipeline2
  - Update paths for eco-ci-cd
  - Integrate with Phase 3 operator-only playbook

### 4.2 Canary Phase Tasks

- [ ] Implement setup phase (ClusterVersion, pause-reconciliation, worker-upgrade MCP)
- [ ] Implement migrate phase (per-node worker upgrade with RTE validation)
- [ ] Implement cleanup phase (unpause reconciliation, delete worker-upgrade MCP)

### 4.3 Testing

- [ ] Test full canary flow (4.18 → 4.20)
- [ ] Test partial migration (CANARY_NODE_COUNT=2)
- [ ] Verify RTE pods on all nodes

**Deliverable**: Full canary skip-version capability

---

## Phase 5: Additional Registry Paths

**Goal**: Support all registry combinations

**Effort**: 1 day

### 5.1 Add Registry Path Tasks

- [ ] Create `tasks/olmv0/stage-prod.yml`
- [ ] Create `tasks/olmv0/prod-stage.yml`
- [ ] Create `tasks/olmv0/stage-stage.yml`

### 5.2 Testing

- [ ] Test each registry path combination
- [ ] Verify catalog source switching

**Deliverable**: All 4 registry paths working

---

## Phase 6: OLMv1 Support (Future)

**Goal**: Support OLMv1 operator format

**Effort**: 2 days

- [ ] Create `tasks/olmv1/install.yml`
- [ ] Create `tasks/olmv1/upgrade.yml`
- [ ] Update routing logic in `tasks/upgrade-nrop.yml`

---

## Summary: Revised Effort Estimates

| Phase | Effort | Reuse % | New Files | Reused Files |
|-------|--------|---------|-----------|--------------|
| Phase 1: Core Role | 2-3 days | 70% | 10 | 4 |
| Phase 2: Documentation | 1 day | 100% | 2 | N/A |
| Phase 3: Operator-Only | 1-2 days | 60% | 2 | 6 |
| Phase 4: Canary | 2-3 days | 40% | 3 | 2 |
| Phase 5: Registry Paths | 1 day | 80% | 3 | 2 |
| Phase 6: OLMv1 | 2 days | 50% | 2 | 3 |
| **Total** | **9-13 days** | **67%** | **22** | **17** |

**Key Insight**: By reusing 67% of existing code, we reduce implementation time from ~20 days to ~10 days.

---

## Files Reused from eco-ci-cd

1. ✅ `nrop_config/templates/scheduler.yaml.j2` → Scheduler CR with version logic
2. ✅ `nrop_config/templates/performanceprofile.yaml.j2` → Performance profile
3. ✅ `nrop_config/tasks/wait_for_mcp.yml` → MCP stabilization
4. ✅ `nrop_config/tasks/deploy_scheduler.yml` → Skopeo SHA resolution pattern
5. ✅ `nrop_config/tasks/main.yml` → Registry namespace detection
6. ✅ `playbooks/roles/ocp_version_facts/` → OCP version detection
7. ✅ `playbooks/roles/ocp_operator_deployment/` → OLM patterns (reference)

---

## Getting Started

**First Steps**:
1. Read `docs/nrop-upgrade-gap-analysis.md` to understand reuse strategy
2. Start Phase 1.2: Copy templates from nrop_config
3. Then Phase 1.4: Extract registry namespace logic
4. Then Phase 1.5: Add new core tasks

**Development Workflow**:
```bash
# Create feature branch
git checkout -b nrop-upgrade-phase1-reuse

# Copy templates
cp playbooks/compute/roles/nrop_config/templates/scheduler.yaml.j2 \
   playbooks/compute/roles/nrop_upgrade/templates/pool_nrop.yaml.j2

# Test as you build
ansible-playbook --syntax-check playbooks/compute/nrop_upgrade.yml

# Commit with descriptive messages
git commit -m "nrop-upgrade: Reuse scheduler template from nrop_config"
```
