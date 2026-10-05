# Gap Analysis Correction Summary

**Date**: 2026-10-05  
**Issue**: Initial gap analysis incorrectly assessed CatalogSource management as missing

---

## What Was Corrected

### ❌ Initial Assessment (WRONG)

"**Gap**: No template for CatalogSource creation/update"

Suggested we needed to create:
- `templates/catalog_source.yaml.j2`
- `templates/namespace.yaml.j2`
- `templates/operator_group.yaml.j2`
- `templates/operator_subscription.yaml.j2`

### ✅ Corrected Assessment (RIGHT)

**eco-ci-cd ALREADY HAS CatalogSource management** via the `redhatci.ocp` collection.

**Proof**:
```yaml
# From playbooks/roles/ocp_operator_deployment/tasks/deploy_stage.yaml
- name: Deploy catalog source if not present
  ansible.builtin.include_role:
    name: redhatci.ocp.catalog_source  # ← External collection role
  vars:
    cs_name: "{{ item.catalog }}"
    cs_image: "{{ stage_catalog_index_image }}:v{{ version }}"
```

---

## Impact on Implementation

### Templates NO LONGER NEEDED

- ~~`templates/catalog_source.yaml.j2`~~ → Use `redhatci.ocp.catalog_source` role instead
- ~~`templates/namespace.yaml.j2`~~ → Use `redhatci.ocp.olm_operator` role instead
- ~~`templates/operator_group.yaml.j2`~~ → Use `redhatci.ocp.olm_operator` role instead

**Reduction**: 3 templates eliminated

### What We Still Need to Add

| Component | Status | Notes |
|-----------|--------|-------|
| Subscription channel UPDATE | ❌ Still missing | `redhatci.ocp.olm_operator` creates new, doesn't update existing |
| Upgrade path routing | ❌ Still missing | Logic to choose prod/stage catalog based on upgrade path |
| Pre/post state collection | ❌ Still missing | Artifact collection for audit trail |
| Canary orchestration | ❌ Still missing | Skip-version upgrade support |

---

## Revised Metrics

| Metric | Initial Estimate | CORRECTED Estimate | Change |
|--------|------------------|-------------------|--------|
| **Reuse Rate** | 67% | **85%** | ↑ 18% |
| **New Templates** | 4 | **1** | ↓ 3 templates |
| **New Tasks** | 14 | **11** | ↓ 3 tasks |
| **Implementation Effort** | 9-13 days | **7-10 days** | ↓ 2-3 days |

---

## Why This Matters

### 1. Consistency with eco-ci-cd Patterns

**Before (Wrong Approach)**:
- Create custom CatalogSource templates
- Duplicate functionality of existing `redhatci.ocp` roles
- Diverge from how `deploy-ocp-operators.yml` works

**After (Correct Approach)**:
- Use same `redhatci.ocp` roles as `deploy-ocp-operators.yml`
- Stay consistent with eco-ci-cd operator deployment patterns
- Leverage proven, tested code

### 2. Less Code to Maintain

**Before**: 14 new task files + 4 new templates = 18 files

**After**: 11 new task files + 1 new template = 12 files

**Reduction**: 6 files (33% less new code)

### 3. Faster Implementation

**Before**: 9-13 days (with custom templates)

**After**: 7-10 days (reusing `redhatci.ocp`)

**Time Saved**: 2-3 days

---

## Updated Phase 1 Checklist

### What to REUSE (No Changes)

- [x] **External Collection Roles**
  - `redhatci.ocp.catalog_source` - For CatalogSource creation/update
  - `redhatci.ocp.olm_operator` - For Namespace, OperatorGroup, Subscription, CSV validation

- [x] **Templates from nrop_config**
  - Copy `scheduler.yaml.j2` → `nrop_upgrade/templates/pool_nrop.yaml.j2`
  - Copy `performanceprofile.yaml.j2` (optional)

- [x] **Task Patterns from nrop_config**
  - Adapt `wait_for_mcp.yml`
  - Copy skopeo SHA resolution pattern
  - Extract registry namespace detection

### What to ADD (Reduced List)

- [ ] `tasks/main.yml` - Entry point
- [ ] `tasks/upgrade-nrop.yml` - Orchestration
- [ ] `tasks/olmv0/prod-prod.yml` - Use `redhatci.ocp.catalog_source` + manual Subscription update
- [ ] `tasks/olmv0/stage-stage.yml` - Same pattern
- [ ] `tasks/olmv0/prod-stage.yml` - Same pattern
- [ ] `tasks/olmv0/stage-prod.yml` - Same pattern
- [ ] `tasks/collect-pre-state.yml` - State collection
- [ ] `tasks/collect-post-state.yml` - State collection
- [ ] `tasks/deploy-scheduler.yml` - Scheduler with SHA verification
- [ ] `tasks/wait-mcp.yml` - Enhanced MCP wait
- [ ] `defaults/main.yml` - Variables
- [ ] `vars/main.yml` - Variables

**Total**: 12 new files (down from 18)

---

## Example: How to Use `redhatci.ocp` for Upgrades

### Fresh Installation (Already Works)
```bash
ansible-playbook playbooks/deploy-ocp-operators.yml \
  -e "version=4.18" \
  -e "operators=[{
    'name':'numaresources-operator',
    'catalog':'redhat-operators',
    'nsname':'openshift-numaresources',
    'channel':'4.18'
  }]"
```

This uses `redhatci.ocp.olm_operator` internally.

### Upgrade (New - Using Same Pattern)
```yaml
# In nrop_upgrade/tasks/olmv0/prod-prod.yml
---
# Step 1: Update CatalogSource to new version
- name: Update production catalog
  ansible.builtin.include_role:
    name: redhatci.ocp.catalog_source
  vars:
    cs_name: redhat-operators
    cs_image: "registry.redhat.io/redhat/redhat-operator-index:v{{ to_version }}"

# Step 2: Wait for new package version
- name: Wait for new packagemanifest
  kubernetes.core.k8s_info:
    api_version: packages.operators.coreos.com/v1
    kind: PackageManifest
    name: numaresources-operator
  register: pkg
  until: to_version in (pkg.resources[0].status.channels | map(attribute='name') | list)
  retries: 30
  delay: 10

# Step 3: Update Subscription channel (manual - not in redhatci.ocp)
- name: Update subscription channel
  kubernetes.core.k8s:
    api_version: operators.coreos.com/v1alpha1
    kind: Subscription
    name: numaresources-operator
    namespace: openshift-numaresources
    state: present
    definition:
      spec:
        channel: "{{ to_version }}"

# Step 4: Wait for new CSV (could use redhatci.ocp.olm_operator validation)
- name: Wait for CSV
  kubernetes.core.k8s_info:
    api_version: operators.coreos.com/v1alpha1
    kind: ClusterServiceVersion
    namespace: openshift-numaresources
  register: csv
  until: >-
    csv.resources 
    | selectattr('metadata.name', 'search', to_version)
    | selectattr('status.phase', 'equalto', 'Succeeded')
    | list | length > 0
  retries: 80
  delay: 10
```

---

## Action Items

1. ✅ **Read corrected gap analysis**: `nrop-upgrade-gap-analysis-CORRECTED.md`
2. ⚠️ **Ignore original gap analysis**: `nrop-upgrade-gap-analysis.md` (contains incorrect CatalogSource assessment)
3. ✅ **Use `redhatci.ocp` roles** wherever possible
4. ✅ **Follow revised roadmap** with reduced file count
5. ✅ **Stay consistent** with existing eco-ci-cd operator deployment patterns

---

## References

- **Corrected Gap Analysis**: `nrop-upgrade-gap-analysis-CORRECTED.md`
- **Proof of CatalogSource Management**: `playbooks/roles/ocp_operator_deployment/tasks/deploy_stage.yaml:33-46`
- **redhatci.ocp Collection**: `requirements.yml:5-6`
- **Example Usage**: `playbooks/deploy-ocp-operators.yml:232-239`

---

**Status**: ✅ Correction Complete - Use CORRECTED gap analysis for implementation
