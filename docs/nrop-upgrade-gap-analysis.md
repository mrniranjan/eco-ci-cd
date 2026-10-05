# NROP Upgrade Gap Analysis

**Purpose**: Identify what eco-ci-cd already has for NROP and what needs to be added for upgrade support.

---

## Executive Summary

**eco-ci-cd's `nrop_config` role is a POST-INSTALLATION CONFIGURATION tool**, not an upgrade tool.

**Architecture**:
- ✅ Assumes NROP operator already installed via `deploy-ocp-operators.yml`
- ✅ Configures scheduler images and CRs
- ✅ Deploys performance profiles
- ❌ Does NOT handle operator installation from scratch
- ❌ Does NOT handle operator upgrades

**Key Finding**: We need to ADD upgrade-specific logic, but can REUSE significant existing components.

---

## What eco-ci-cd Already Has ✅

### 1. **NROP Configuration Role** (`playbooks/compute/roles/nrop_config/`)

**Capabilities**:
- Configures NROP operator post-installation
- Patches existing Subscription with scheduler image digests
- Deploys NUMAResourcesOperator CR
- Deploys NUMAResourcesScheduler CR
- Auto-detects OCP version and sets registry namespace
- Handles performance profile generation

**Key Files**:
```
roles/nrop_config/
├── tasks/
│   ├── main.yml                    # Entry point
│   ├── deploy_scheduler.yml        # ⭐ REUSE: Scheduler image resolution with skopeo
│   ├── wait_for_mcp.yml            # ⭐ REUSE: MCP stabilization logic
│   ├── preformance_profile.yml     # ⭐ REUSE: Performance profile
│   └── deploy_manifest.yml         # Template deployment helper
├── templates/
│   ├── scheduler.yaml.j2           # ⭐ REUSE: NUMAResourcesOperator + Scheduler CRs
│   └── performanceprofile.yaml.j2  # ⭐ REUSE: Performance profile
└── defaults/main.yml
```

**What Works Well (Should Reuse)**:

#### A. Version-Aware Scheduler Template
**File**: `templates/scheduler.yaml.j2`
```jinja2
{% if ocp_version_major | int == 5 or ocp_version_minor | int >= 18 %}
apiVersion: nodetopology.openshift.io/v1
kind: NUMAResourcesOperator
spec:
  nodeGroups:
  - poolName: {{ nrop_target_pool }}
{% else %}
apiVersion: nodetopology.openshift.io/v1
kind: NUMAResourcesOperator
spec:
  nodeGroups:
  - machineConfigPoolSelector:
      matchLabels:
        pools.operator.machineconfiguration.openshift.io/{{ nrop_target_pool }}: ""
{% endif %}
```
**Why Reuse**: Already handles 4.18+ poolName vs. ≤4.17 MCP selector logic

#### B. Registry Namespace Detection
**File**: `tasks/main.yml`
```yaml
- name: Set NROP registry namespace from OCP version
  ansible.builtin.set_fact:
    nrop_registry_ns: "{{ 'openshift5' if (ocp_version_major | int) >= 5 else 'openshift4' }}"

- name: Set scheduler image registry from OCP version
  ansible.builtin.set_fact:
    image_uri:
      prod: "registry.redhat.io/{{ nrop_registry_ns }}"
      stage: "registry.stage.redhat.io/{{ nrop_registry_ns }}"
```
**Why Reuse**: Auto-detects registry based on OCP major version

#### C. Skopeo SHA Resolution
**File**: `tasks/deploy_scheduler.yml`
```yaml
- name: Get scheduler sha256
  ansible.builtin.command: >-
    skopeo --override-os linux --override-arch amd64 inspect
    --authfile /tmp/pull_secret
    --format {{ (scheduler_image_version.split(':')[0] ~ '@{{.Digest}}') | quote }}
    docker://{{ scheduler_image_version }}
  register: scheduler_image_sha
  changed_when: false

- name: Patch Subscription with image digests
  kubernetes.core.k8s_json_patch:
    api_version: operators.coreos.com/v1alpha1
    kind: Subscription
    name: numaresources-operator
    namespace: openshift-numaresources
    patch:
      - op: add
        path: /spec/config/env
        value:
          - name: SCHEDULER_IMAGE_DIGESTS
            value: "{{ scheduler_image_sha.stdout.split('@')[1] }}"
```
**Why Reuse**: Proven pattern for image digest verification

#### D. MCP Wait Logic
**File**: `tasks/wait_for_mcp.yml`
```yaml
- name: Wait for machine config pool to be updated
  kubernetes.core.k8s_info:
    api_version: machineconfiguration.openshift.io/v1
    kind: MachineConfigPool
  register: machine_config_pool_post
  retries: 360
  delay: 10
  until: |
    not machine_config_pool_post.failed
    and machine_config_pool_post.resources | length > 0
    and (machine_config_pool_post.resources
      | community.general.json_query('[*].status.readyMachineCount') ==
      (machine_config_pool_post.resources | community.general.json_query('[*].status.machineCount')))
```
**Why Reuse**: Waits for all MCPs to stabilize (readyMachineCount == machineCount)

#### E. Variable Patterns
**File**: `defaults/main.yml`
```yaml
test_env: prod  # or stage
topology_manager_scope: pod
scheduler_image_name: noderesourcetopology-scheduler
reserved_cpu_count: 4
```
**Why Reuse**: Established naming conventions for prod/stage

### 2. **Generic Operator Deployment** (`playbooks/deploy-ocp-operators.yml`)

**Capabilities**:
- Installs ANY OpenShift operator via OLM
- Supports both connected and disconnected flows
- Mirrors operators to internal registry (disconnected mode)
- Creates CatalogSource, OperatorGroup, Subscription
- Waits for CSV to reach Succeeded

**How NROP is Currently Installed**:
```bash
ansible-playbook playbooks/deploy-ocp-operators.yml \
  -e "kubeconfig=/path/to/kc" \
  -e "version=4.18" \
  -e "operators=[{
    'name':'numaresources-operator',
    'catalog':'redhat-operators',
    'nsname':'openshift-numaresources',
    'channel':'4.18',
    'deploy_default_config':true
  }]"
```

**Why This Matters**: 
- Fresh NROP installations already work via `deploy-ocp-operators.yml`
- We can leverage `ocp_operator_deployment` role patterns
- Don't need to reinvent OLM resource creation

### 3. **NROP Testing Role** (`playbooks/compute/roles/deploy_nrop_tests/`)

**Capabilities**:
- Runs upstream NROP e2e tests in containers
- Supports tier-based test filtering
- Handles reboot tests, scheduler restart tests
- Must-gather integration

**Why This Matters**: Can use for post-upgrade validation

---

## What's Missing for Upgrades ❌

### 1. **CatalogSource Management**

**Gap**: No template for CatalogSource creation/update

**Needed**:
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
  updateStrategy:
    registryPoll:
      interval: 10m
```

**Where**: `nrop_upgrade/templates/catalog_source.yaml.j2`

### 2. **Subscription Channel Modification**

**Gap**: Only patches Subscription with env vars, doesn't change channel

**Current** (`nrop_config/tasks/deploy_scheduler.yml:68-79`):
```yaml
- name: Patch Subscription with image digests
  kubernetes.core.k8s_json_patch:
    kind: Subscription
    name: numaresources-operator
    patch:
      - op: add
        path: /spec/config/env
        value:
          - name: SCHEDULER_IMAGE_DIGESTS
            value: "{{ scheduler_image_sha.stdout.split('@')[1] }}"
```

**Needed for Upgrade**:
```yaml
- name: Update Subscription channel for upgrade
  kubernetes.core.k8s:
    api_version: operators.coreos.com/v1alpha1
    kind: Subscription
    name: numaresources-operator
    namespace: openshift-numaresources
    state: present
    definition:
      spec:
        channel: "{{ target_version }}"  # e.g., "4.18"
        source: "{{ catalog_name }}"     # e.g., "redhat-operators"
```

**Where**: `nrop_upgrade/tasks/olmv0/upgrade.yml`

### 3. **CSV Wait and Validation**

**Gap**: No logic to wait for CSV to reach Succeeded phase

**Needed**:
```yaml
- name: Wait for new CSV with target version to appear and succeed
  kubernetes.core.k8s_info:
    api_version: operators.coreos.com/v1alpha1
    kind: ClusterServiceVersion
    namespace: openshift-numaresources
  register: csv_info
  until: >-
    csv_info.resources
    | selectattr('metadata.name', 'search', target_version)
    | selectattr('status.phase', 'equalto', 'Succeeded')
    | list | length > 0
  delay: 10
  retries: 80
```

**Where**: `nrop_upgrade/tasks/olmv0/install.yml`, `nrop_upgrade/tasks/olmv0/upgrade.yml`

### 4. **Upgrade Path Routing**

**Gap**: No logic to handle different registry combinations

**Needed**: Tasks for each upgrade path:
- `olmv0/prod-prod.yml` - Production → Production
- `olmv0/stage-stage.yml` - Stage → Stage
- `olmv0/prod-stage.yml` - Production → Stage (downgrade for testing)
- `olmv0/stage-prod.yml` - Stage → Production (promote)

**Example** (`olmv0/prod-prod.yml`):
```yaml
# Install from production, upgrade to production
- name: Update CatalogSource to new production registry
  kubernetes.core.k8s:
    template: "{{ role_path }}/templates/catalog_source.yaml.j2"
  vars:
    catalog_name: redhat-operators
    registry: "registry.redhat.io/redhat/redhat-operator-index:v{{ to_version }}"

- name: Update Subscription channel
  kubernetes.core.k8s:
    api_version: operators.coreos.com/v1alpha1
    kind: Subscription
    name: numaresources-operator
    namespace: openshift-numaresources
    state: present
    definition:
      spec:
        channel: "{{ to_version }}"
        source: redhat-operators
```

**Where**: `nrop_upgrade/tasks/olmv0/*.yml`

### 5. **OperatorGroup and Namespace Creation**

**Gap**: Assumes namespace and OperatorGroup already exist

**Needed**:
```yaml
- name: Create namespace
  kubernetes.core.k8s:
    api_version: v1
    kind: Namespace
    name: openshift-numaresources
    state: present

- name: Create OperatorGroup
  kubernetes.core.k8s:
    api_version: operators.coreos.com/v1
    kind: OperatorGroup
    name: numaresources-operator
    namespace: openshift-numaresources
    state: present
    definition:
      spec:
        targetNamespaces:
          - openshift-numaresources
```

**Where**: `nrop_upgrade/tasks/olmv0/install.yml`

### 6. **Pre/Post Upgrade State Collection**

**Gap**: No artifact collection for upgrade tracking

**Needed**:
```yaml
# Pre-upgrade state
- name: Collect current CSV version
  kubernetes.core.k8s_info:
    api_version: operators.coreos.com/v1alpha1
    kind: ClusterServiceVersion
    namespace: openshift-numaresources
  register: pre_upgrade_csv

- name: Save pre-upgrade state
  ansible.builtin.copy:
    content: |
      CSV: {{ pre_upgrade_csv.resources[0].metadata.name }}
      Scheduler: {{ current_scheduler_image }}
      MCP Status: {{ mcp_status }}
    dest: "{{ report_folder }}/nrop-upgrade/pre-state.txt"
```

**Where**: `nrop_upgrade/tasks/collect-pre-state.yml`, `nrop_upgrade/tasks/collect-post-state.yml`

### 7. **OLMv1 Support**

**Gap**: Only works with OLMv0 (traditional OLM)

**Needed**: Separate tasks for OLMv1 resource structure
- `olmv1/install.yml`
- `olmv1/upgrade.yml`

**Where**: `nrop_upgrade/tasks/olmv1/*.yml`

### 8. **Canary Mode Orchestration**

**Gap**: No support for skip-version upgrades with gradual worker migration

**Needed**:
- Script for canary orchestration (setup/migrate/cleanup phases)
- Worker-upgrade MachineConfigPool creation
- Per-node migration logic
- Pause-reconciliation annotation management

**Where**: 
- `scripts/nrop-canary-skip-upgrade.sh` (bash orchestration)
- OR `nrop_upgrade/tasks/canary/*.yml` (pure Ansible)

### 9. **Upgrade Type Detection**

**Gap**: No logic to differentiate z-stream vs. minor vs. skip-version

**Needed**:
```yaml
- name: Determine upgrade type
  ansible.builtin.set_fact:
    upgrade_type: >-
      {% if old_ocp_version == new_ocp_version %}
      zstream
      {% elif (new_ocp_version.split('.')[1] | int) - (old_ocp_version.split('.')[1] | int) == 1 %}
      minor
      {% else %}
      skip-version
      {% endif %}
```

**Where**: `nrop_upgrade/tasks/main.yml`

### 10. **Rollback Capability**

**Gap**: No automated rollback on upgrade failure

**Needed** (optional for Phase 1, critical for production):
```yaml
rescue:
  - name: Rollback to previous CSV
    kubernetes.core.k8s:
      api_version: operators.coreos.com/v1alpha1
      kind: Subscription
      name: numaresources-operator
      namespace: openshift-numaresources
      state: present
      definition:
        spec:
          channel: "{{ from_version }}"
          startingCSV: "{{ pre_upgrade_csv_name }}"
```

**Where**: `nrop_upgrade/tasks/main.yml` (rescue block)

---

## Reuse Strategy

### Components to Directly Reuse

| Component | Source | Target | Adaptation |
|-----------|--------|--------|------------|
| Scheduler CR template | `nrop_config/templates/scheduler.yaml.j2` | `nrop_upgrade/templates/pool_nrop.yaml.j2` | None - copy as-is |
| Registry namespace detection | `nrop_config/tasks/main.yml` | `nrop_upgrade/vars/main.yml` | Extract to role vars |
| Skopeo SHA resolution | `nrop_config/tasks/deploy_scheduler.yml` | `nrop_upgrade/tasks/deploy-scheduler.yml` | Copy pattern |
| MCP wait logic | `nrop_config/tasks/wait_for_mcp.yml` | `nrop_upgrade/tasks/wait-mcp.yml` | Enhance with per-MCP targeting |
| Performance profile | `nrop_config/tasks/preformance_profile.yml` | `nrop_upgrade/tasks/olmv0/install.yml` | Reference as optional step |

### Components to Reference (Not Duplicate)

| Component | Why Reference | Integration Point |
|-----------|---------------|-------------------|
| `deploy-ocp-operators.yml` | Already handles fresh installs | Document as alternative to upgrade role for fresh installs |
| `ocp_operator_deployment` role | OLM resource patterns | Use as reference for CatalogSource/Subscription templates |
| `deploy_nrop_tests` role | Post-upgrade validation | Call after upgrade completes |
| `ocp_version_facts` role | OCP version detection | Use instead of inline version detection |

---

## Recommended Implementation Strategy

### Phase 1: Core Upgrade Role (Reuse-Heavy)

**Goal**: Build on existing `nrop_config` patterns

**Approach**:
1. Create `nrop_upgrade` role directory structure
2. **COPY** scheduler.yaml.j2 template from `nrop_config`
3. **EXTRACT** registry namespace logic to `nrop_upgrade/vars/main.yml`
4. **ADAPT** skopeo SHA resolution from `deploy_scheduler.yml`
5. **ENHANCE** MCP wait logic with per-MCP targeting
6. **ADD** new templates: catalog_source.yaml.j2, operator_subscription.yaml.j2
7. **ADD** new tasks: CSV wait, upgrade path routing

**Files to Create** (10 new, 4 reused):
- ✅ Reuse: `templates/scheduler.yaml.j2` (copy from nrop_config)
- ✅ Reuse: `tasks/wait-mcp.yml` (adapt from nrop_config)
- ✅ Reuse: Registry namespace detection (extract to vars)
- ✅ Reuse: Skopeo SHA pattern (copy from deploy_scheduler.yml)
- ❌ New: `templates/catalog_source.yaml.j2`
- ❌ New: `templates/operator_subscription.yaml.j2`
- ❌ New: `templates/operator_group.yaml.j2`
- ❌ New: `templates/namespace.yaml.j2`
- ❌ New: `tasks/main.yml`
- ❌ New: `tasks/upgrade-nrop.yml`
- ❌ New: `tasks/olmv0/install.yml`
- ❌ New: `tasks/olmv0/prod-prod.yml`
- ❌ New: `tasks/collect-pre-state.yml`
- ❌ New: `tasks/collect-post-state.yml`

**Effort Estimate**: 2-3 days (70% reuse, 30% new)

### Phase 2: Integration with Existing Playbooks

**Goal**: Leverage `deploy-ocp-operators.yml` for fresh installs

**Approach**:
1. Document that fresh NROP installs should use `deploy-ocp-operators.yml`
2. Position `nrop_upgrade` role as **upgrade-only** tool
3. Update `nrop_config` to work with both installation methods
4. Create upgrade-specific playbook: `nrop_upgrade.yml`

**Integration Points**:
```yaml
# Fresh install (existing)
ansible-playbook playbooks/deploy-ocp-operators.yml \
  -e "operators=[{'name':'numaresources-operator','catalog':'redhat-operators','nsname':'openshift-numaresources','channel':'4.18'}]"

# Post-install config (existing)
ansible-playbook playbooks/compute/configure-nrop-operator.yml

# Upgrade (NEW)
ansible-playbook playbooks/compute/nrop_upgrade.yml \
  -e "old_ocp_version=4.17 new_ocp_version=4.18 upgrade=minor"
```

**Effort Estimate**: 1 day

### Phase 3: Canary Support (Extend)

**Goal**: Add skip-version capability

**Approach**:
1. Port canary script from cnf-pipeline2
2. Create operator-only upgrade playbook (reuses Phase 1 role)
3. Integrate canary phases with existing upgrade role

**Effort Estimate**: 2-3 days

---

## Summary: What to Build

### Minimal Viable Upgrade (Phase 1)

**New Files Required**: ~14 files
- 4 templates (catalog_source, subscription, operator_group, namespace)
- 7 task files (main, upgrade-nrop, olmv0/install, olmv0/prod-prod, collect states, deploy-scheduler)
- 2 variable files (defaults/main.yml, vars/main.yml)
- 1 playbook (nrop_upgrade.yml)

**Reused Components**: ~4 files
- scheduler.yaml.j2 template (from nrop_config)
- wait_for_mcp.yml pattern (from nrop_config)
- Registry namespace detection (from nrop_config)
- Skopeo SHA resolution (from nrop_config)

**Total Effort**: ~3-4 days

### Gap Summary Table

| Capability | eco-ci-cd Status | cnf-pipeline2 Has | Action |
|------------|------------------|-------------------|--------|
| Fresh operator install | ✅ Via deploy-ocp-operators.yml | ✅ Via upgrade role | Document existing path |
| Scheduler configuration | ✅ Via nrop_config | ✅ Via upgrade role | Reuse templates |
| Version detection | ✅ Inline in nrop_config | ✅ Via NTO role | Use ocp_version_facts |
| Registry namespace | ✅ openshift4/5 detection | ✅ Same logic | Reuse pattern |
| Skopeo SHA resolution | ✅ deploy_scheduler.yml | ✅ deploy-scheduler.yml | Reuse pattern |
| MCP wait logic | ✅ wait_for_mcp.yml | ✅ NTO role | Enhance for per-MCP |
| Performance profile | ✅ pp_generator.yml | ✅ Optional integration | Reference existing |
| CatalogSource mgmt | ❌ None | ✅ templates/*.j2 | Add templates |
| Subscription channel mod | ❌ Only env vars | ✅ Full CRUD | Add tasks |
| CSV wait/validation | ❌ None | ✅ Multiple wait tasks | Add tasks |
| Upgrade path routing | ❌ None | ✅ 4 prod/stage combos | Add tasks |
| OperatorGroup creation | ❌ Assumes exists | ✅ templates/*.j2 | Add template |
| Namespace creation | ❌ Assumes exists | ✅ templates/*.j2 | Add template |
| Pre/post state collection | ❌ None | ✅ artifacts.yaml | Add tasks |
| OLMv1 support | ❌ None | ✅ olmv1/ subdirectory | Phase 4 |
| Canary orchestration | ❌ None | ✅ Bash script | Phase 3 |
| Rollback capability | ❌ None | ❌ None | Optional |

**Legend**: ✅ Exists, ❌ Missing

---

## Conclusion

**eco-ci-cd has 50% of what we need already built.**

**Reuse Strategy**:
- ✅ Use existing `nrop_config` for scheduler/CR templates
- ✅ Use existing `deploy-ocp-operators.yml` for fresh installs
- ✅ Use existing MCP wait and version detection patterns
- ❌ Add upgrade-specific OLM resource management
- ❌ Add upgrade path routing logic
- ❌ Add state collection and validation

**Key Decision**: Build `nrop_upgrade` as a **focused upgrade-only role** that complements (not replaces) existing NROP tooling.
