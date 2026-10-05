# NROP + OCP Upgrade Integration Strategy

**Key Finding**: eco-ci-cd ALREADY HAS OCP cluster upgrade capability via the `cluster_upgrade` role.

---

## What eco-ci-cd Has for OCP Upgrade ✅

### 1. **Cluster Upgrade Role** (`playbooks/compute/roles/cluster_upgrade/`)

**Location**: `/home/agent/work/playbooks/compute/roles/cluster_upgrade/`

**Playbook**: `/home/agent/work/playbooks/compute/cluster_upgrade.yml`

**Capabilities**:
- ✅ Auto-detects next minor version (4.17 → 4.18)
- ✅ Queries Red Hat upgrade graph for latest version in channel
- ✅ Patches ClusterVersion to trigger upgrade
- ✅ Waits for upgrade completion (up to 250 retries × 60s = 4+ hours)
- ✅ Validates all nodes are Ready and Schedulable
- ✅ Pre/post upgrade state collection (kernel params, container runtime, cgroups, reboot count)
- ✅ Workload validation (deploys test pod, verifies it survives upgrade)

**Workflow**:
```yaml
# tasks/main.yml
1. Get cluster version BEFORE upgrade
2. Deploy test workload
3. Save pre-upgrade state
4. Execute upgrade (tasks/upgrade-cluster.yml)
5. Get cluster version AFTER upgrade
6. Save post-upgrade state
7. Verify test workload still running
8. Assert Cgroup version unchanged
9. Assert Container Runtime unchanged
```

**Upgrade Logic** (`tasks/upgrade-cluster.yml`):
```yaml
1. Parse current OCP version (e.g., 4.17.5)
2. Calculate target minor version (4.17 → 4.18)
3. Fetch upgrade graph from Red Hat API:
   - URL: https://api.openshift.com/api/upgrades_info/v1/graph
   - Channel: stable-4.18 (or eus-4.18, fast-4.18)
   - Arch: amd64/arm64
4. Select latest version from graph (e.g., 4.18.12)
5. Get release image digest via `oc adm release info`
6. Patch ClusterVersion:
   spec:
     channel: stable-4.18
     desiredUpdate:
       version: 4.18.12
       image: quay.io/openshift-release-dev/ocp-release@sha256:...
7. Wait for ClusterVersion state: Completed
8. Wait for all nodes: Schedulable + Ready
```

**Default Configuration** (`defaults/main.yaml`):
```yaml
ocp_arch: "amd64"
target_channel: "stable"  # or "eus", "fast"
upgrade_timeout_retries: 250  # 250 × 60s = 4+ hours
upgrade_delay: 60  # seconds between retries
report_folder: "/tmp/reports"
artifacts_folder: "/artifacts"
```

### 2. **Usage Pattern**

**Single Hop Upgrade** (4.17 → 4.18):
```bash
ansible-playbook playbooks/compute/cluster_upgrade.yml \
  -e "kubeconfig=/path/to/kc"
```

**Variables**:
- `kubeconfig` (required): Path to cluster kubeconfig
- `target_channel` (optional): "stable" (default), "eus", "fast"
- `ocp_arch` (optional): "amd64" (default), "arm64"

**What it does**:
- Auto-calculates target version (current minor + 1)
- Queries Red Hat graph for latest stable version
- Upgrades cluster
- Validates workload survives

---

## Integration Patterns for NROP Upgrade

### Pattern 1: Sequential Range Upgrade (RECOMMENDED for Phase 1)

**Use Case**: Multi-hop upgrades (e.g., 4.17 → 4.20)

**Approach**: Alternate between OCP upgrade and NROP upgrade

```bash
# Hop 1: 4.17 → 4.18
ansible-playbook playbooks/compute/cluster_upgrade.yml \
  -e "kubeconfig=/path/to/kc"

ansible-playbook playbooks/compute/nrop_upgrade.yml \
  -e "kubeconfig=/path/to/kc" \
  -e "old_ocp_version=4.17" \
  -e "new_ocp_version=4.18" \
  -e "upgrade=minor"

# Hop 2: 4.18 → 4.19
ansible-playbook playbooks/compute/cluster_upgrade.yml \
  -e "kubeconfig=/path/to/kc"

ansible-playbook playbooks/compute/nrop_upgrade.yml \
  -e "kubeconfig=/path/to/kc" \
  -e "old_ocp_version=4.18" \
  -e "new_ocp_version=4.19" \
  -e "upgrade=minor"

# Hop 3: 4.19 → 4.20
ansible-playbook playbooks/compute/cluster_upgrade.yml \
  -e "kubeconfig=/path/to/kc"

ansible-playbook playbooks/compute/nrop_upgrade.yml \
  -e "kubeconfig=/path/to/kc" \
  -e "old_ocp_version=4.19" \
  -e "new_ocp_version=4.20" \
  -e "upgrade=minor"
```

**Pros**:
- ✅ Simple orchestration
- ✅ Reuses existing cluster_upgrade role
- ✅ Each hop is independent
- ✅ Can stop/rollback between hops

**Cons**:
- ❌ Manual coordination required
- ❌ Slow for multi-hop scenarios

### Pattern 2: Unified Orchestration Playbook (Future Enhancement)

**Use Case**: Automated multi-hop upgrades

**Approach**: Create orchestration playbook that calls both roles

```yaml
# playbooks/compute/ocp_nrop_upgrade.yml
---
- name: Coordinated OCP + NROP Upgrade
  hosts: localhost
  gather_facts: false
  vars:
    from_version: "4.17"
    to_version: "4.20"
  
  tasks:
    - name: Calculate upgrade hops
      ansible.builtin.set_fact:
        hops: "{{ range(from_version.split('.')[1] | int + 1, to_version.split('.')[1] | int + 1) | list }}"
    
    - name: Execute upgrade for each hop
      ansible.builtin.include_tasks: upgrade_hop.yml
      loop: "{{ hops }}"
      loop_control:
        loop_var: target_minor
```

```yaml
# tasks/upgrade_hop.yml
---
- name: Upgrade OCP cluster
  ansible.builtin.include_role:
    name: cluster_upgrade

- name: Upgrade NROP operator
  ansible.builtin.include_role:
    name: nrop_upgrade
  vars:
    old_ocp_version: "4.{{ target_minor - 1 }}"
    new_ocp_version: "4.{{ target_minor }}"
    upgrade: minor
```

**Pros**:
- ✅ Fully automated multi-hop
- ✅ Single command for range upgrades
- ✅ Error handling across all hops

**Cons**:
- ❌ More complex
- ❌ Harder to debug individual hops
- ❌ Requires Phase 1 NROP upgrade complete first

### Pattern 3: Canary Skip-Version (Phase 4)

**Use Case**: Skip-version upgrades (4.18 → 4.20)

**Approach**: Upgrade NROP first, then OCP control plane, then workers

```bash
# Phase 1: Canary Setup
bash scripts/nrop-canary-skip-upgrade.sh setup

# Phase 2: Upgrade NROP Operator Only (to 4.20)
ansible-playbook playbooks/compute/nrop_operator_upgrade.yml \
  -e "kubeconfig=/path/to/kc" \
  -e "nrop_target_version=4.20"

# Phase 3: Upgrade OCP Control Plane to 4.19
ansible-playbook playbooks/compute/cluster_upgrade.yml \
  -e "kubeconfig=/path/to/kc" \
  -e "target_channel=eus"

# Phase 4: Upgrade OCP Control Plane to 4.20
ansible-playbook playbooks/compute/cluster_upgrade.yml \
  -e "kubeconfig=/path/to/kc" \
  -e "target_channel=eus"

# Phase 5: Migrate Workers
bash scripts/nrop-canary-skip-upgrade.sh migrate

# Phase 6: Cleanup
bash scripts/nrop-canary-skip-upgrade.sh cleanup
```

**Pros**:
- ✅ Supports skip-version upgrades
- ✅ Gradual worker migration
- ✅ Reduced downtime

**Cons**:
- ❌ Complex orchestration
- ❌ Requires canary script (Phase 4)
- ❌ OCP upgrade happens while NROP is on newer version

---

## Differences: cluster_upgrade vs. cnf-pipeline2 OCP Upgrade

### eco-ci-cd `cluster_upgrade` Role

**Strengths**:
- ✅ Auto-detects next minor version
- ✅ Uses Red Hat upgrade graph API
- ✅ Pre/post state validation
- ✅ Test workload validation

**Limitations**:
- ⚠️ Only supports single hop at a time
- ⚠️ Doesn't support `--to-image` for custom releases
- ⚠️ Auto-increments minor (can't specify exact target)

### cnf-pipeline2 `upgrade/ocp` Role

**Strengths**:
- ✅ Supports `--to-image` for custom/RC releases
- ✅ Supports explicit target version
- ✅ Handles IDMS workarounds for specific versions (4.21)
- ✅ More flexible channel management

**Differences**:
```yaml
# eco-ci-cd approach (auto-detect from graph)
- Calculates target_version from current_version + 1
- Fetches latest from Red Hat graph
- Uses digest from `oc adm release info`

# cnf-pipeline2 approach (explicit target)
- User provides exact target version or pull spec
- Can upgrade to RC/nightly builds
- Can upgrade to custom images
```

---

## Recommendation: Which Pattern to Use

### For Phase 1 (Initial Implementation)

**Use Pattern 1: Sequential Range Upgrade**

**Rationale**:
- ✅ Simplest integration
- ✅ Reuses existing `cluster_upgrade` role as-is
- ✅ No modifications to cluster_upgrade needed
- ✅ Independent playbooks for OCP and NROP
- ✅ Can be orchestrated via wrapper script or Jenkins

**Implementation**:
```bash
# scripts/nrop_range_upgrade_wrapper.sh
#!/bin/bash
FROM_VERSION=$1
TO_VERSION=$2

FROM_MINOR=$(echo $FROM_VERSION | cut -d. -f2)
TO_MINOR=$(echo $TO_VERSION | cut -d. -f2)

for ((minor=$((FROM_MINOR + 1)); minor <= TO_MINOR; minor++)); do
  echo "=== Upgrading to 4.$minor ==="
  
  # Upgrade OCP
  ansible-playbook playbooks/compute/cluster_upgrade.yml \
    -e "kubeconfig=$KUBECONFIG"
  
  # Upgrade NROP
  ansible-playbook playbooks/compute/nrop_upgrade.yml \
    -e "kubeconfig=$KUBECONFIG" \
    -e "old_ocp_version=4.$((minor - 1))" \
    -e "new_ocp_version=4.$minor" \
    -e "upgrade=minor" \
    -e "upgrade_from=prod" \
    -e "upgrade_to=prod"
done
```

**Usage**:
```bash
export KUBECONFIG=/path/to/kc
bash scripts/nrop_range_upgrade_wrapper.sh 4.17 4.20
```

### For Phase 2+ (Future Enhancement)

**Add Pattern 2: Unified Orchestration**

**When**: After Phase 1 is proven and stable

**Benefits**:
- Better error handling
- Artifact correlation
- Single command execution
- Automated retry logic

---

## Integration Checklist

### Phase 1: NROP Upgrade Core (NO cluster_upgrade changes)

- [ ] Build `nrop_upgrade` role (reusing 85% from eco-ci-cd)
- [ ] Test `nrop_upgrade` independently on single version
- [ ] Create wrapper script for multi-hop (Pattern 1)
- [ ] Test multi-hop: 4.17 → 4.20 via wrapper
- [ ] Document integration pattern in CLAUDE.md

**Deliverable**: `nrop_upgrade` role works alongside `cluster_upgrade` (no modifications to cluster_upgrade)

### Phase 2: Unified Orchestration (Optional Enhancement)

- [ ] Create `ocp_nrop_upgrade.yml` playbook
- [ ] Add hop calculation logic
- [ ] Add error handling for partial failures
- [ ] Add rollback capability
- [ ] Test end-to-end multi-hop automation

**Deliverable**: Single-command multi-hop upgrade

### Phase 3: cluster_upgrade Enhancements (Optional)

**If needed for canary mode**:

- [ ] Add `target_version` override (skip auto-increment)
- [ ] Add `release_image` override (support custom images)
- [ ] Add `skip_channel_patch` option (for canary CP-only)

**Example**:
```yaml
# For canary mode: upgrade CP only, skip workers
- name: Upgrade control plane only
  ansible.builtin.include_role:
    name: cluster_upgrade
  vars:
    target_version: "4.20"  # Override auto-detection
    skip_channel_patch: true  # Don't change channel (for canary)
    release_image: "quay.io/.../ocp-release@sha256:..."  # Custom image
```

---

## Summary: OCP Upgrade Already Exists

| Capability | eco-ci-cd Status | Notes |
|------------|------------------|-------|
| **OCP single hop upgrade** | ✅ EXISTS | `cluster_upgrade` role |
| **Auto-version detection** | ✅ EXISTS | Queries Red Hat graph |
| **Pre/post validation** | ✅ EXISTS | State collection + workload test |
| **Multi-hop automation** | ❌ Need wrapper | Pattern 1 or 2 |
| **Custom image support** | ❌ Not in eco-ci-cd | cnf-pipeline2 has this |
| **Skip-version (canary)** | ❌ Not in eco-ci-cd | Requires Phase 4 |

**Conclusion**: 
- ✅ OCP upgrade exists and works well
- ✅ NROP upgrade should integrate via Pattern 1 initially
- ✅ No changes to `cluster_upgrade` needed for Phase 1
- ⚠️ Future enhancements can add unified orchestration

---

## Action Items

1. ✅ **Use existing cluster_upgrade** - Don't rebuild OCP upgrade
2. ✅ **Build NROP upgrade independently** - Focus on operator lifecycle only
3. ✅ **Create wrapper script** - For multi-hop coordination (Pattern 1)
4. ✅ **Document integration** - Update CLAUDE.md with both playbooks
5. ⚠️ **Consider unified playbook** - Future Phase 2 enhancement

**Next Step**: Implement `nrop_upgrade` role (Phase 1) to work alongside existing `cluster_upgrade`.
