# eco-ci-cd Documentation

This directory contains design documents, plans, and guides for the eco-ci-cd project.

## NROP Upgrade Implementation

The NROP (NUMA Resources Operator) upgrade feature is being added incrementally to eco-ci-cd.

---

## 📚 Quick Reference

**Start Here**:
1. 🎯 **`FINAL-SUMMARY.md`** - Executive summary with all corrections
2. 🔍 **`nrop-upgrade-gap-analysis-CORRECTED.md`** - What exists vs. what to build
3. 🔄 **`nrop-ocp-upgrade-integration.md`** - How OCP + NROP work together
4. 🚀 **`nrop-upgrade-roadmap-revised.md`** - Phase-by-phase implementation

---

## 📖 Key Documents

### 🎯 [FINAL-SUMMARY.md](./FINAL-SUMMARY.md) ⭐ **START HERE**
**Complete overview with all corrections** covering:
- Executive summary: eco-ci-cd has 90% of what we need
- What exists: OCP upgrade, CatalogSource mgmt, NROP config
- What's missing: NROP upgrade orchestration (15% new code)
- Revised effort: **7-10 days** (down from 20 days)
- Three-playbook architecture
- Integration strategy
- Success criteria

**Use this for**: Complete picture, what changed, why it matters, where to start.

### 🔍 [nrop-upgrade-gap-analysis-CORRECTED.md](./nrop-upgrade-gap-analysis-CORRECTED.md) ⭐ REUSE STRATEGY
**CORRECTED: What eco-ci-cd already has vs. what we need to add** covering:
- ✅ **CORRECTION 1**: CatalogSource management EXISTS via `redhatci.ocp.catalog_source`
- ✅ **CORRECTION 2**: OCP upgrade EXISTS via `cluster_upgrade` role
- ✅ Existing capabilities: nrop_config, deploy-ocp-operators, redhatci.ocp collection
- ✅ Components to reuse (85%): scheduler templates, MCP wait, OLM roles
- ❌ Components to add (15%): Subscription update, upgrade routing, state collection
- **Revised reuse**: 85% reuse, 15% new (up from 67%/33%)
- **Revised effort**: 7-10 days (down from 9-13 days)

**Use this for**: Understanding what NOT to build, what to copy, component-by-component breakdown.

### 🔄 [nrop-ocp-upgrade-integration.md](./nrop-ocp-upgrade-integration.md) ⭐ INTEGRATION
**How NROP upgrade integrates with existing OCP upgrade** covering:
- ✅ **KEY FINDING**: OCP upgrade EXISTS via `cluster_upgrade` role
- `cluster_upgrade` capabilities: auto-version detection, Red Hat graph, validation
- Pattern 1: Sequential (RECOMMENDED for Phase 1) - alternate OCP/NROP playbooks
- Pattern 2: Unified orchestration (future) - single command multi-hop
- Pattern 3: Canary skip-version (Phase 4) - gradual worker migration
- Wrapper script examples

**Use this for**: Understanding coordination, why we don't build OCP upgrade, integration patterns.

### 🚀 [nrop-upgrade-roadmap-revised.md](./nrop-upgrade-roadmap-revised.md) ⭐ IMPLEMENTATION
**Incremental implementation tracking (revised for maximum reuse)** with:
- 6 development phases with checkboxes
- Phase 1: 2-3 days, 70% reuse (core role)
- Detailed reuse strategy per task
- Copy-paste commands for reusing files
- Testing checklist per phase
- **Total effort**: 7-10 days

**Use this for**: Day-to-day development, tracking progress, step-by-step instructions.

### 📋 [nrop-upgrade-plan.md](./nrop-upgrade-plan.md)
**Original comprehensive technical specification** covering:
- Full architecture overview (range mode, canary mode, skip-version)
- cnf-pipeline2 implementation deep-dive
- All three upgrade types: z-stream, minor, skip-version
- Complete variable references
- Verification steps

**Use this for**: Deep technical reference, cnf-pipeline2 patterns, workflow details.

### 📝 [CORRECTION-SUMMARY.md](./CORRECTION-SUMMARY.md)
**What changed from initial assessment** covering:
- Correction 1: CatalogSource management exists (eliminated 3 templates)
- Correction 2: OCP upgrade exists (no need to build)
- Impact: 85% reuse (up from 67%), 7-10 days (down from 9-13)
- Before/after metrics
- Action items

**Use this for**: Understanding corrections, impact analysis, what not to build.

---

## 🎓 Quick Start Guide

### Step 1: Understand What Exists (15 min)
```bash
# Read final summary
cat docs/FINAL-SUMMARY.md

# Key takeaway: eco-ci-cd has 90% already
# - OCP upgrade ✅
# - CatalogSource management ✅  
# - NROP configuration ✅
# - Only need: NROP upgrade orchestration (15% new)
```

### Step 2: Review Gap Analysis (30 min)
```bash
# Read corrected gap analysis
cat docs/nrop-upgrade-gap-analysis-CORRECTED.md

# Key takeaway: Reuse 85%
# - Copy 5 files from nrop_config
# - Use redhatci.ocp roles for CatalogSource
# - Build 13 new files (upgrade orchestration)
```

### Step 3: Understand Integration (15 min)
```bash
# Read integration guide
cat docs/nrop-ocp-upgrade-integration.md

# Key takeaway: Pattern 1 - Sequential
# 1. Run cluster_upgrade.yml (OCP)
# 2. Run nrop_upgrade.yml (NROP)
# 3. Repeat for each hop
```

### Step 4: Start Implementation (Follow Roadmap)
```bash
# Read roadmap
cat docs/nrop-upgrade-roadmap-revised.md

# Phase 1.2: Copy templates from nrop_config
mkdir -p playbooks/compute/roles/nrop_upgrade/templates
cp playbooks/compute/roles/nrop_config/templates/scheduler.yaml.j2 \
   playbooks/compute/roles/nrop_upgrade/templates/pool_nrop.yaml.j2

# Phase 1.3: Add new templates (only 1 needed now!)
# Phase 1.4: Extract vars from nrop_config
# Phase 1.5: Add core tasks
# Phase 1.6: Add variables
# Phase 1.7: Create playbook
# Phase 1.8: Test
```

---

## 📊 Key Metrics (Corrected)

| Metric | Initial | After Corrections | Improvement |
|--------|---------|-------------------|-------------|
| **Reuse Rate** | 50% | **90%** | +40% |
| **New Templates** | 4 | **1** | -75% |
| **New Tasks** | 14 | **11** | -21% |
| **Implementation Effort** | 20 days | **7-10 days** | -50-65% |
| **Time Saved** | 0 | **10-13 days** | N/A |

---

## 🔧 What eco-ci-cd Already Has

### 1. OCP Cluster Upgrade ✅
- **Role**: `playbooks/compute/roles/cluster_upgrade/`
- **Playbook**: `playbooks/compute/cluster_upgrade.yml`
- **What it does**: Upgrades OCP to next minor version automatically

### 2. CatalogSource Management ✅
- **Collection**: `redhatci.ocp` (v3.5.1781925805)
- **Roles**: `redhatci.ocp.catalog_source`, `redhatci.ocp.olm_operator`
- **What it does**: Creates/updates CatalogSources, installs operators via OLM

### 3. NROP Configuration ✅
- **Role**: `playbooks/compute/roles/nrop_config/`
- **Playbook**: `playbooks/compute/configure-nrop-operator.yml`
- **What it does**: Configures NROP after installation (scheduler, performance profile)

### 4. Generic Operator Install ✅
- **Playbook**: `playbooks/deploy-ocp-operators.yml`
- **What it does**: Installs ANY operator including NROP

---

## ❌ What's Missing (15% New Code)

1. Subscription channel update (1 task)
2. Upgrade path routing: prod→prod, stage→stage, etc. (4 tasks)
3. Pre/post state collection (2 tasks)
4. Main orchestration (2 tasks)
5. Scheduler update (1 task, adapt from nrop_config)
6. Enhanced MCP wait (1 task, adapt from nrop_config)
7. Variables (2 files)

**Total**: 13 files (11 tasks + 2 vars)

---

## 🎯 Three-Playbook Architecture

```
┌──────────────────────────────────────┐
│ 1. OCP Upgrade (EXISTS ✅)           │
│    playbooks/compute/                │
│      cluster_upgrade.yml             │
│    Uses: cluster_upgrade role        │
└──────────────────────────────────────┘
              ↓
┌──────────────────────────────────────┐
│ 2. NROP Upgrade (NEW ❌)             │
│    playbooks/compute/                │
│      nrop_upgrade.yml                │
│    Uses: nrop_upgrade role           │
└──────────────────────────────────────┘
              ↓
┌──────────────────────────────────────┐
│ 3. NROP Config (EXISTS ✅)           │
│    playbooks/compute/                │
│      configure-nrop-operator.yml     │
│    Uses: nrop_config role            │
└──────────────────────────────────────┘
```

**Pattern 1 Integration** (RECOMMENDED):
```bash
# Multi-hop upgrade (4.17 → 4.20)
for version in 4.18 4.19 4.20; do
  ansible-playbook playbooks/compute/cluster_upgrade.yml
  ansible-playbook playbooks/compute/nrop_upgrade.yml \
    -e "new_ocp_version=$version"
done
```

---

## 📁 Documentation Files

| File | Purpose | Status |
|------|---------|--------|
| `FINAL-SUMMARY.md` | Executive summary with corrections | ✅ Complete |
| `nrop-upgrade-gap-analysis-CORRECTED.md` | Reuse strategy | ✅ Complete |
| `nrop-ocp-upgrade-integration.md` | Integration patterns | ✅ Complete |
| `nrop-upgrade-roadmap-revised.md` | Implementation roadmap | ✅ Complete |
| `CORRECTION-SUMMARY.md` | What changed | ✅ Complete |
| `nrop-upgrade-plan.md` | Full technical spec | ✅ Complete |
| `README.md` | This file | ✅ Complete |

---

## ✅ Next Steps

1. **Review** `FINAL-SUMMARY.md` (5 min)
2. **Read** `nrop-upgrade-gap-analysis-CORRECTED.md` (15 min)
3. **Read** `nrop-ocp-upgrade-integration.md` (10 min)
4. **Approve** approach and scope
5. **Start** Phase 1 implementation (7-10 days)

---

## ❓ Questions?

- **What exists?** → `FINAL-SUMMARY.md` Section: "What eco-ci-cd Already Has"
- **What to build?** → `nrop-upgrade-gap-analysis-CORRECTED.md` Section: "What's ACTUALLY Missing"
- **How to integrate?** → `nrop-ocp-upgrade-integration.md` Pattern 1
- **Implementation steps?** → `nrop-upgrade-roadmap-revised.md` Phase 1
- **What changed?** → `CORRECTION-SUMMARY.md`

---

**Status**: ✅ Planning Complete - Ready for Implementation  
**Revised Effort**: 7-10 days (85% reuse, 15% new)  
**Next Milestone**: Phase 1 Core Role Complete
