# NROP Upgrade Gap Analysis (CORRECTED)

**Correction**: This document corrects the initial gap analysis regarding CatalogSource management.

---

## CRITICAL CORRECTION: CatalogSource Management

### ❌ Initial Assessment (INCORRECT)
> "Gap: No template for CatalogSource creation/update"

### ✅ Actual Reality (CORRECT)

**eco-ci-cd DOES have CatalogSource management** via the `redhatci.ocp` collection.

**Evidence**:

1. **`deploy-ocp-operators.yml` successfully installs operators from stage**
   ```yaml
   # From playbooks/deploy-ocp-operators.yml
   - name: Deploy operator via olm
     ansible.builtin.include_role:
       name: redhatci.ocp.olm_operator  # ← External collection role
   ```

2. **`ocp_operator_deployment` role creates CatalogSources**
   ```yaml
   # From playbooks/roles/ocp_operator_deployment/tasks/deploy_stage.yaml:33-46
   - name: Deploy catalog source if not present
     when: catalog_source.resources | length == 0
     ansible.builtin.include_role:
       name: redhatci.ocp.catalog_source  # ← Creates CatalogSource
     vars:
       cs_name: "{{ item.catalog }}"
       cs_namespace: openshift-marketplace
       cs_image: "{{ ocp_operator_deployment_stage_repo_image }}:v{{ version }}"
       cs_publisher: "Red Hat"
       cs_secrets:
         - "{{ item.catalog }}"
       cs_update_strategy:
         registryPoll:
           interval: 15m
   ```

3. **`redhatci.ocp` collection provides OLM primitives**
   ```yaml
   # From requirements.yml
   collections:
     - name: redhatci.ocp
       version: 3.5.1781925805  # ← Provides catalog_source and olm_operator roles
   ```

---

## Revised Gap Analysis

### What eco-ci-cd HAS ✅

| Component | Implementation | Location |
|-----------|---------------|----------|
| **CatalogSource creation** | ✅ Via `redhatci.ocp.catalog_source` | External collection role |
| **Namespace creation** | ✅ Via `redhatci.ocp.olm_operator` | External collection role |
| **OperatorGroup creation** | ✅ Via `redhatci.ocp.olm_operator` | External collection role |
| **Subscription creation** | ✅ Via `redhatci.ocp.olm_operator` | External collection role |
| **CSV validation** | ✅ Via `redhatci.ocp.olm_operator` | External collection role (when `olm_operator_validate_install: true`) |
| **Scheduler CR template** | ✅ In `nrop_config` | `playbooks/compute/roles/nrop_config/templates/scheduler.yaml.j2` |
| **Skopeo SHA resolution** | ✅ In `nrop_config` | `playbooks/compute/roles/nrop_config/tasks/deploy_scheduler.yml` |
| **MCP wait logic** | ✅ In `nrop_config` | `playbooks/compute/roles/nrop_config/tasks/wait_for_mcp.yml` |
| **Registry namespace detection** | ✅ In `nrop_config` | `playbooks/compute/roles/nrop_config/tasks/main.yml` |
| **Performance profile** | ✅ In `nrop_config` | `playbooks/compute/roles/nrop_config/tasks/preformance_profile.yml` |

### What's ACTUALLY Missing ❌

| Component | Why Missing |
|-----------|-------------|
| **Subscription channel modification for upgrades** | `redhatci.ocp.olm_operator` creates new subscriptions, doesn't update existing channels |
| **Upgrade path routing logic** | No logic to handle prod→prod, stage→prod, prod→stage, stage→stage upgrade combinations |
| **Pre/post upgrade state collection** | No artifact collection for audit trail |
| **Canary orchestration** | No support for skip-version upgrades with gradual worker migration |
| **Rollback capability** | No automated rollback on upgrade failure |

---

## Updated Reuse Strategy

### Option 1: Reuse `redhatci.ocp` Roles (RECOMMENDED)

**Advantages**:
- ✅ Consistent with eco-ci-cd patterns
- ✅ Proven, tested code
- ✅ Already used by `deploy-ocp-operators.yml`
- ✅ Minimal custom code

**Usage**:
```yaml
# For fresh installation (already works today)
- name: Install NROP operator
  ansible.builtin.include_role:
    name: redhatci.ocp.olm_operator
  vars:
    operator: numaresources-operator
    source: redhat-operators  # or stage catalog
    namespace: openshift-numaresources
    channel: "4.18"

# For upgrading CatalogSource to new version
- name: Update stage catalog to new version
  ansible.builtin.include_role:
    name: redhatci.ocp.catalog_source
  vars:
    cs_name: konflux-stage-catalog
    cs_namespace: openshift-marketplace
    cs_image: "quay.io/.../numaresources-operator-fbc-4-18:latest"
    cs_publisher: "Red Hat"
```

**What We Still Need to Add**:
1. Logic to UPDATE existing Subscription channel (not create new)
2. Upgrade path routing (when to update catalog vs. subscription)
3. State collection tasks

### Option 2: Custom Templates (cnf-pipeline2 approach)

**Advantages**:
- ✅ Self-contained role
- ✅ Fine-grained control
- ✅ Matches cnf-pipeline2 exactly

**Disadvantages**:
- ❌ Duplicates functionality of `redhatci.ocp`
- ❌ Diverges from eco-ci-cd patterns
- ❌ More code to maintain

---

## Revised "What's Missing" List

### 1. Subscription Channel Update (Still Missing)

**Gap**: `redhatci.ocp.olm_operator` creates NEW subscriptions, doesn't UPDATE existing subscription channels

**Current** (`redhatci.ocp.olm_operator`):
- Creates Subscription if doesn't exist
- Sets initial channel

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
        channel: "{{ target_version }}"  # Update channel to new version
        source: "{{ catalog_name }}"
```

**Where**: `nrop_upgrade/tasks/olmv0/upgrade.yml`

### 2. CatalogSource Version Update (Partially Covered)

**Current**: `redhatci.ocp.catalog_source` can create/update CatalogSource

**Needed**: Logic to decide WHEN to update CatalogSource based on upgrade path

**Example** (prod→stage upgrade):
```yaml
# Step 1: Update CatalogSource to stage registry
- name: Update to stage catalog
  ansible.builtin.include_role:
    name: redhatci.ocp.catalog_source
  vars:
    cs_name: konflux-stage-catalog
    cs_image: "quay.io/.../numaresources-operator-fbc-{{ to_version }}:latest"

# Step 2: Update Subscription to point to new catalog
- name: Update Subscription source
  kubernetes.core.k8s:
    api_version: operators.coreos.com/v1alpha1
    kind: Subscription
    name: numaresources-operator
    namespace: openshift-numaresources
    state: present
    definition:
      spec:
        source: konflux-stage-catalog  # Switch from redhat-operators to stage
        channel: "{{ to_version }}"
```

**Where**: `nrop_upgrade/tasks/olmv0/prod-stage.yml`, etc.

### 3. Upgrade Path Routing (Still Missing)

**Gap**: No orchestration logic for different registry combinations

**Needed**:
```yaml
# In tasks/upgrade-nrop.yml
- name: Determine upgrade path
  ansible.builtin.set_fact:
    upgrade_path: "{{ upgrade_from }}-{{ upgrade_to }}"

- name: Execute upgrade path
  ansible.builtin.include_tasks: "olmv0/{{ upgrade_path }}.yml"
```

**Where**: 
- `nrop_upgrade/tasks/olmv0/prod-prod.yml` - Update CatalogSource to new version
- `nrop_upgrade/tasks/olmv0/stage-stage.yml` - Update stage catalog
- `nrop_upgrade/tasks/olmv0/prod-stage.yml` - Switch from prod to stage catalog
- `nrop_upgrade/tasks/olmv0/stage-prod.yml` - Switch from stage to prod catalog

### 4. Pre/Post State Collection (Still Missing)

**Gap**: No artifact collection for upgrade tracking

**Needed** (same as before):
```yaml
- name: Collect pre-upgrade state
  kubernetes.core.k8s_info:
    api_version: operators.coreos.com/v1alpha1
    kind: ClusterServiceVersion
    namespace: openshift-numaresources
  register: pre_csv

- name: Save state
  ansible.builtin.copy:
    content: "{{ pre_csv | to_nice_json }}"
    dest: "{{ report_folder }}/nrop-upgrade/pre-state.json"
```

**Where**: `nrop_upgrade/tasks/collect-pre-state.yml`

### 5. Canary Orchestration (Still Missing)

**Gap**: No support for skip-version upgrades

**Where**: `scripts/nrop-canary-skip-upgrade.sh` or `nrop_upgrade/tasks/canary/*.yml`

---

## Updated Implementation Strategy

### Phase 1: Core Upgrade Role (REVISED)

**Reuse 85% (up from 70%), Add 15% (down from 30%)**

#### Files to REUSE (No Changes)

1. ✅ **External Roles** (via `redhatci.ocp` collection)
   - `redhatci.ocp.catalog_source` - CatalogSource creation/update
   - `redhatci.ocp.olm_operator` - Namespace, OperatorGroup, Subscription, CSV validation
   
2. ✅ **Templates** (from `nrop_config`)
   - Copy `scheduler.yaml.j2` → `nrop_upgrade/templates/pool_nrop.yaml.j2`
   - Copy `performanceprofile.yaml.j2` → `nrop_upgrade/templates/performance_profile.yaml.j2`

3. ✅ **Task Patterns** (from `nrop_config`)
   - Adapt `wait_for_mcp.yml` → `nrop_upgrade/tasks/wait-mcp.yml`
   - Copy skopeo SHA pattern from `deploy_scheduler.yml`
   - Extract registry namespace logic to vars

#### Files to ADD (New)

1. ❌ **Core Tasks** (5 new files)
   - `tasks/main.yml` - Entry point with version detection
   - `tasks/upgrade-nrop.yml` - Orchestration and path routing
   - `tasks/collect-pre-state.yml` - Pre-upgrade state
   - `tasks/collect-post-state.yml` - Post-upgrade state  
   - `tasks/deploy-scheduler.yml` - Scheduler with SHA verification

2. ❌ **Upgrade Path Tasks** (4 new files)
   - `tasks/olmv0/prod-prod.yml` - Update catalog version
   - `tasks/olmv0/stage-stage.yml` - Update stage catalog version
   - `tasks/olmv0/prod-stage.yml` - Switch to stage catalog
   - `tasks/olmv0/stage-prod.yml` - Switch to prod catalog

3. ❌ **Variable Files** (2 new files)
   - `defaults/main.yml`
   - `vars/main.yml`

#### Files NO LONGER NEEDED (Covered by `redhatci.ocp`)

- ~~`templates/catalog_source.yaml.j2`~~ - Use `redhatci.ocp.catalog_source` role instead
- ~~`templates/namespace.yaml.j2`~~ - Use `redhatci.ocp.olm_operator` role instead
- ~~`templates/operator_group.yaml.j2`~~ - Use `redhatci.ocp.olm_operator` role instead
- ~~`templates/operator_subscription.yaml.j2`~~ - Partially covered, need manual update logic

**Revised Effort**: 1-2 days (vs. 2-3 days before)

---

## Example: prod→prod Upgrade Using `redhatci.ocp`

```yaml
# tasks/olmv0/prod-prod.yml
---
- name: Update production CatalogSource to new version
  ansible.builtin.include_role:
    name: redhatci.ocp.catalog_source
  vars:
    cs_name: redhat-operators
    cs_namespace: openshift-marketplace
    cs_image: "registry.redhat.io/redhat/redhat-operator-index:v{{ to_version }}"
    cs_publisher: "Red Hat"
    cs_update_strategy:
      registryPoll:
        interval: 10m

- name: Wait for new packagemanifest to appear
  kubernetes.core.k8s_info:
    api_version: packages.operators.coreos.com/v1
    kind: PackageManifest
    name: numaresources-operator
  register: pkg_manifest
  until: >-
    pkg_manifest.resources | length > 0 and
    to_version in (pkg_manifest.resources[0].status.channels | map(attribute='name') | list)
  retries: 30
  delay: 10

- name: Update Subscription channel to new version
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
        sourceNamespace: openshift-marketplace

- name: Wait for CSV to reach Succeeded
  kubernetes.core.k8s_info:
    api_version: operators.coreos.com/v1alpha1
    kind: ClusterServiceVersion
    namespace: openshift-numaresources
  register: csv_info
  until: >-
    csv_info.resources
    | selectattr('metadata.name', 'search', to_version)
    | selectattr('status.phase', 'equalto', 'Succeeded')
    | list | length > 0
  delay: 10
  retries: 80

- name: Update scheduler image
  ansible.builtin.include_tasks: ../deploy-scheduler.yml

- name: Wait for MCP to stabilize
  ansible.builtin.include_tasks: ../wait-mcp.yml
```

---

## Summary of Corrections

| Component | Initial Assessment | CORRECTED Assessment |
|-----------|-------------------|----------------------|
| CatalogSource mgmt | ❌ Missing | ✅ **EXISTS** via `redhatci.ocp.catalog_source` |
| Namespace creation | ❌ Missing | ✅ **EXISTS** via `redhatci.ocp.olm_operator` |
| OperatorGroup creation | ❌ Missing | ✅ **EXISTS** via `redhatci.ocp.olm_operator` |
| Subscription creation | ❌ Missing | ✅ **EXISTS** via `redhatci.ocp.olm_operator` |
| CSV validation | ❌ Missing | ✅ **EXISTS** via `redhatci.ocp.olm_operator` |
| Subscription channel UPDATE | ❌ Missing | ❌ **STILL MISSING** (create only, not update) |
| Upgrade path routing | ❌ Missing | ❌ **STILL MISSING** |
| State collection | ❌ Missing | ❌ **STILL MISSING** |
| Canary orchestration | ❌ Missing | ❌ **STILL MISSING** |

**Revised Reuse Rate**: **85%** (up from 67%)

**Revised Effort**: **7-10 days** (down from 9-13 days)

---

## Recommendation

**Use `redhatci.ocp` roles wherever possible:**

1. ✅ Use `redhatci.ocp.catalog_source` for CatalogSource creation/updates
2. ✅ Use `redhatci.ocp.olm_operator` for fresh NROP installations
3. ❌ Add manual Subscription channel update logic (not in collection)
4. ❌ Add upgrade path routing logic
5. ❌ Add state collection tasks

This maximizes reuse and stays consistent with eco-ci-cd's existing operator deployment patterns.
