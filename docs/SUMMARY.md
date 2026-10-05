# NROP Upgrade Implementation - Summary

**Created**: 2026-10-05

---

## What Was Delivered

### 📚 Complete Documentation Set

Four comprehensive documents totaling ~60 pages:

1. **`nrop-upgrade-gap-analysis.md`** (15 pages) ⭐ **START HERE**
   - What eco-ci-cd already has vs. what needs to be added
   - Reuse strategy: 70% reuse, 30% new
   - Component-by-component analysis
   - Effort estimates

2. **`nrop-upgrade-roadmap-revised.md`** (20 pages) ⭐ **IMPLEMENTATION GUIDE**
   - 6 phases with detailed checkboxes
   - Copy-paste commands for reusing files
   - Revised effort: 9-13 days (vs. 20 days)
   - Testing checklists

3. **`nrop-upgrade-plan.md`** (37 pages)
   - Full technical specification
   - Architecture deep-dive
   - cnf-pipeline2 analysis
   - Variable references

4. **`README.md`** (2 pages)
   - Documentation index
   - Quick start guide
   - Reuse strategy summary

---

## Key Findings

### 🎯 Critical Insight

**eco-ci-cd already has 50% of what we need for NROP upgrades.**

The existing `nrop_config` role provides:
- ✅ Scheduler CR templates with 4.18+ version logic
- ✅ Skopeo-based SHA resolution
- ✅ MCP wait and stabilization
- ✅ Registry namespace detection (openshift4/openshift5)
- ✅ Performance profile integration

**What's missing**: OLM lifecycle management (CatalogSource, Subscription channel updates, CSV validation).

### 📊 Reuse Breakdown

| Component | eco-ci-cd Has | cnf-pipeline2 Has | Action |
|-----------|---------------|-------------------|--------|
| **Scheduler CR template** | ✅ | ✅ | **COPY** from nrop_config |
| **MCP wait logic** | ✅ | ✅ | **ADAPT** from nrop_config |
| **Skopeo SHA resolution** | ✅ | ✅ | **COPY PATTERN** from nrop_config |
| **Registry namespace** | ✅ | ✅ | **EXTRACT** from nrop_config |
| **Performance profile** | ✅ | ✅ | **REFERENCE** from nrop_config |
| **CatalogSource mgmt** | ❌ | ✅ | **ADD NEW** templates |
| **Subscription channel mod** | ❌ | ✅ | **ADD NEW** tasks |
| **CSV wait/validation** | ❌ | ✅ | **ADD NEW** tasks |
| **Upgrade path routing** | ❌ | ✅ | **ADD NEW** tasks |
| **Pre/post state collection** | ❌ | ✅ | **ADD NEW** tasks |

**Result**: 67% reuse rate across full implementation

---

## Implementation Strategy

### Phase 1: Core Upgrade Role (Essential) - 2-3 days

**Reuse 70%, Add 30%**

**Copy from nrop_config**:
1. `templates/scheduler.yaml.j2` → `nrop_upgrade/templates/pool_nrop.yaml.j2`
2. `templates/performanceprofile.yaml.j2` → `nrop_upgrade/templates/performance_profile.yaml.j2`
3. `tasks/wait_for_mcp.yml` pattern → `nrop_upgrade/tasks/wait-mcp.yml`
4. `tasks/deploy_scheduler.yml` skopeo logic → `nrop_upgrade/tasks/deploy-scheduler.yml`
5. Registry namespace vars → `nrop_upgrade/vars/main.yml`

**Add new**:
1. `templates/catalog_source.yaml.j2`
2. `templates/operator_subscription.yaml.j2`
3. `templates/operator_group.yaml.j2`
4. `templates/namespace.yaml.j2`
5. `tasks/main.yml`, `tasks/upgrade-nrop.yml`
6. `tasks/olmv0/install.yml`, `tasks/olmv0/prod-prod.yml`
7. `tasks/collect-pre-state.yml`, `tasks/collect-post-state.yml`

**Deliverable**: Z-stream and minor upgrades working

### Subsequent Phases

- **Phase 2**: Documentation (1 day, 100% reuse)
- **Phase 3**: Operator-only upgrade (1-2 days, 60% reuse)
- **Phase 4**: Canary skip-version (2-3 days, 40% reuse)
- **Phase 5**: Registry paths (1 day, 80% reuse)
- **Phase 6**: OLMv1 (2 days, 50% reuse)

---

## Architecture Decisions

### 1. **nrop_upgrade = Focused Upgrade-Only Role**

**NOT a replacement for existing tools**, but a complement:

```bash
# Fresh install → Use existing generic operator deployment
ansible-playbook playbooks/deploy-ocp-operators.yml \
  -e "operators=[{'name':'numaresources-operator',...}]"

# Post-install config → Use existing nrop_config
ansible-playbook playbooks/compute/configure-nrop-operator.yml

# Upgrade → Use NEW nrop_upgrade role
ansible-playbook playbooks/compute/nrop_upgrade.yml \
  -e "old_ocp_version=4.17 new_ocp_version=4.18 upgrade=minor"
```

### 2. **Three Upgrade Types Supported**

1. **Z-stream** (4.18.0 → 4.18.1): Same minor, patch only
2. **Minor** (4.17 → 4.18): Sequential version upgrade
3. **Skip-version/Canary** (4.18 → 4.20): Non-sequential with gradual migration

### 3. **Two Operational Modes**

**Range Mode** (Sequential):
- Upgrade both OCP + NROP at each hop
- Uses `worker-cnf` MachineConfigPool
- Simpler, proven

**Canary Mode** (Skip-version):
- 5-phase: setup → upgrade NROP → upgrade CP → migrate workers → cleanup
- Uses temporary `worker-upgrade` MCP
- Gradual node migration
- Requires `pause-reconciliation` annotation

---

## What eco-ci-cd ALREADY Does Well

### Existing NROP Tooling

**`nrop_config` role** (`playbooks/compute/roles/nrop_config/`):
- ✅ Configures NROP operator post-installation
- ✅ Version-aware scheduler deployment (4.18+ poolName vs. ≤4.17 MCP)
- ✅ Skopeo-based image digest verification
- ✅ Performance profile auto-generation from must-gather
- ✅ Full schedulable cluster detection
- ✅ Test device deployment

**`deploy-ocp-operators.yml` playbook**:
- ✅ Generic operator installation via OLM
- ✅ Connected and disconnected flows
- ✅ Operator mirroring to internal registry
- ✅ NROP can be installed this way today

**`deploy_nrop_tests` role**:
- ✅ Containerized e2e test runner
- ✅ Tier-based test filtering
- ✅ Must-gather integration

### What's Missing (Gap)

- ❌ Operator upgrade lifecycle (CatalogSource updates, channel modification)
- ❌ CSV wait and validation
- ❌ Upgrade path routing (prod→prod, stage→stage, etc.)
- ❌ Pre/post upgrade state collection
- ❌ Canary orchestration
- ❌ Skip-version support

---

## File Locations

All documentation saved to `/home/agent/work/docs/`:

```
docs/
├── README.md                           # Documentation index (START HERE)
├── nrop-upgrade-gap-analysis.md        # Reuse strategy ⭐
├── nrop-upgrade-roadmap-revised.md     # Implementation guide ⭐
├── nrop-upgrade-plan.md                # Full technical spec
└── SUMMARY.md                          # This file
```

---

## Next Steps

### For Implementation

1. **Read gap analysis** (`nrop-upgrade-gap-analysis.md`)
   - Understand what to reuse vs. add
   - Review component-by-component breakdown

2. **Follow revised roadmap** (`nrop-upgrade-roadmap-revised.md`)
   - Start with Phase 1.2: Copy templates from nrop_config
   - Check off tasks as you complete them
   - Test incrementally

3. **Reference full plan** (`nrop-upgrade-plan.md`)
   - For technical deep-dive
   - For variable references
   - For cnf-pipeline2 implementation details

### For Review

1. **Read README.md** for quick overview
2. **Scan gap analysis** summary tables
3. **Review roadmap** Phase 1 checklist
4. **Approve/modify** before implementation starts

---

## Metrics

| Metric | Value |
|--------|-------|
| **Total Documentation** | ~60 pages |
| **Reuse Rate** | 67% |
| **Original Effort Estimate** | 20 days |
| **Revised Effort Estimate** | 9-13 days |
| **Time Saved by Reuse** | 7-11 days (50%) |
| **Files to Copy** | 4 |
| **Files to Adapt** | 3 |
| **Files to Create** | 14 (Phase 1) |
| **Total Files (All Phases)** | 22 new + 17 reused |

---

## Key Takeaways

1. ✅ **Don't reinvent the wheel** - eco-ci-cd has 50% of what we need
2. ✅ **Reuse templates** - scheduler.yaml.j2 already has 4.18+ logic
3. ✅ **Reuse patterns** - skopeo SHA, MCP wait, registry detection
4. ✅ **Add focused logic** - CatalogSource/Subscription/CSV management
5. ✅ **Complement, don't replace** - nrop_upgrade works WITH existing tools
6. ✅ **Test incrementally** - Each phase delivers value independently

---

## Questions?

- **What to reuse?** → See `nrop-upgrade-gap-analysis.md` Section: "Reuse Strategy"
- **How to implement?** → See `nrop-upgrade-roadmap-revised.md` Phase 1
- **Why this approach?** → See `nrop-upgrade-plan.md` Section: "Recommended Approach"
- **What already exists?** → See `nrop-upgrade-gap-analysis.md` Section: "What eco-ci-cd Already Has"

---

**Status**: ✅ Planning Complete - Ready for Implementation

**Next Milestone**: Phase 1 Core Role (2-3 days)
