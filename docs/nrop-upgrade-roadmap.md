# NROP Upgrade Implementation Roadmap

This document tracks the incremental implementation of NROP upgrade capabilities in eco-ci-cd.

**Reference**: See `docs/nrop-upgrade-plan.md` for full technical details.

---

## Phase 1: Core Role + Range Upgrade (Essential)

**Goal**: Support basic NROP upgrades for z-stream and minor versions using range mode (sequential hops).

**Scope**: OLMv0 only, prod→prod registry path only

### 1.1 Role Directory Structure

- [ ] Create `/home/agent/work/playbooks/compute/roles/nrop_upgrade/` directory
- [ ] Create `tasks/` subdirectory
- [ ] Create `templates/` subdirectory
- [ ] Create `defaults/` subdirectory
- [ ] Create `vars/` subdirectory
- [ ] Create `olmv0/` subdirectory under tasks/

### 1.2 Core Task Files

- [ ] Port `tasks/main.yml` - Entry point, delegates to upgrade type
  - Replace NTO role dependency with `ocp_version_facts`
  - Align variable names (`cnf_kubeconfig` → `kubeconfig`)
  - Add artifact collection integration
  
- [ ] Port `tasks/upgrade-nrop.yml` - Core orchestration
  - Route to OLMv0 tasks based on upgrade path
  - Set version facts using `ocp_version_facts`
  - Calculate upgrade paths (zstream vs minor)

- [ ] Port `tasks/olmv0/install.yml` - OLMv0 fresh installation
  - Create namespace
  - Apply CatalogSource
  - Apply OperatorGroup
  - Apply Subscription
  - Wait for CSV to reach Succeeded
  - Deploy NROP CR with version-specific logic (poolName for 4.18+, MCP for ≤4.17)
  - Deploy NUMAResourcesScheduler
  
- [ ] Port `tasks/olmv0/prod-prod.yml` - Production → Production upgrade path
  - Update CatalogSource
  - Update Subscription
  - Wait for new CSV
  - Update NROP CR
  - Update Scheduler

- [ ] Create `tasks/deploy-scheduler.yml` - Scheduler deployment with digest verification
  - Use skopeo to inspect scheduler image
  - Verify image digest
  - Apply NUMAResourcesScheduler CR
  - Wait for scheduler pods to be Running

- [ ] Create `tasks/verify-deployment.yml` - Post-upgrade validation
  - Create test deployment using topo-aware-scheduler
  - Verify pod is scheduled and running
  - Cleanup test resources

- [ ] Create `tasks/wait-mcp.yml` - MachineConfigPool stabilization
  - Wait for MCP Updated=True, Updating=False, Degraded=False
  - Configurable retries and delays
  - Diagnostic collection on degraded state

- [ ] Create `tasks/collect-pre-state.yml` - Pre-upgrade state collection
  - Get current CSV version
  - Get current scheduler image
  - Get NROP CR state
  - Get MCP status
  - Save to `{{ report_folder }}/nrop-upgrade/pre-state.json`

- [ ] Create `tasks/collect-post-state.yml` - Post-upgrade state collection
  - Same as pre-state
  - Save to `{{ report_folder }}/nrop-upgrade/post-state.json`

### 1.3 Templates

- [ ] Port `templates/catalog_source.yaml.j2` - CatalogSource for operator index
- [ ] Port `templates/operator_subscription.yaml.j2` - Subscription with channel
- [ ] Port `templates/pool_nrop.yaml.j2` - NROP CR (poolName for 4.18+, MCP for ≤4.17)
- [ ] Port `templates/scheduler.yaml.j2` - NUMAResourcesScheduler CR
- [ ] Port `templates/test_deployment.yaml.j2` - Test deployment for validation
- [ ] Port `templates/operator_group.yaml.j2` - OperatorGroup for namespace
- [ ] Port `templates/namespace.yaml.j2` - Namespace creation

### 1.4 Variables

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
  ```

- [ ] Create `vars/main.yml`
  ```yaml
  # Registry namespaces by OCP major version
  # Upgrade path logic
  # Image URI templates
  ```

### 1.5 Main Playbook

- [ ] Create `playbooks/compute/nrop_upgrade.yml`
  - Import role with variable validation
  - Set KUBECONFIG environment
  - Create report folder
  - Call ocp_version_facts
  - Execute upgrade
  - Collect artifacts

### 1.6 Testing

- [ ] Test fresh installation on 4.18 cluster
- [ ] Test z-stream upgrade (4.18.0 → 4.18.1)
- [ ] Test minor upgrade (4.17 → 4.18)
- [ ] Test with both poolName (4.18+) and MCP (4.17) selector logic
- [ ] Verify artifacts are collected correctly
- [ ] Test with verify_deployment=true

**Deliverable**: Working NROP upgrades for z-stream and minor versions using prod registry

---

## Phase 2: Operator-Only Upgrade (Canary Prerequisite)

**Goal**: Support operator-only upgrades (catalog + subscription + scheduler) without NROP CR reconciliation.

**Scope**: Required for canary mode, standalone useful for testing

### 2.1 Playbook Creation

- [ ] Create `playbooks/compute/nrop_operator_upgrade.yml`
  - Port from `/workspace/cnf-pipeline2/cnf-features/upgrade-nrop-operator.yaml`
  - Update template paths to eco-ci-cd role location
  - Align variable names
  - Keep all validation logic

### 2.2 Scheduler Validation Suite

- [ ] Port Test 1: Scheduler CR Conditions
  - Verify Available=True, Degraded=False, Progressing=False
  - Python inline validation script
  
- [ ] Port Test 2: Scheduler Image Match
  - Verify deployment image matches target
  - Shell script validation

- [ ] Port Test 3: Scheduler Control-Plane Placement
  - Verify scheduler pods run on control-plane nodes only
  - Shell script with node label checks

- [ ] Port Test 4: Test Pod Scheduling via TAS
  - Create test namespace
  - Deploy test pod using topo-aware-scheduler
  - Verify pod is scheduled and running
  - Cleanup test namespace

### 2.3 Validation Artifacts

- [ ] Generate scheduler validation text report
- [ ] Generate scheduler validation JUnit XML report
- [ ] Save skopeo inspection results (source + target)
- [ ] Save scheduler digests (source + target)

### 2.4 Pause-Reconciliation Check

- [ ] Verify pause-reconciliation annotation is preserved
  - Critical for canary mode
  - Fail loudly if annotation is removed

### 2.5 Testing

- [ ] Test operator-only upgrade on 4.18 cluster
- [ ] Verify all 4 scheduler tests pass
- [ ] Verify JUnit XML is generated correctly
- [ ] Test with pause-reconciliation annotation
- [ ] Verify annotation is preserved after upgrade

**Deliverable**: Standalone operator-only upgrade playbook with comprehensive validation

---

## Phase 3: Canary Skip-Version Support

**Goal**: Support skip-version upgrades using canary mode (5-phase workflow).

**Scope**: Full canary implementation with gradual worker migration

### 3.1 Canary Script

- [ ] Create `scripts/nrop-canary-skip-upgrade.sh`
  - Port from `/workspace/cnf-pipeline2/scripts/nrop-canary-skip-upgrade.sh`
  - Update paths for eco-ci-cd structure
  - Support eco-ci-cd kubeconfig conventions
  - Integrate with eco-ci-cd artifact collection

### 3.2 Phase 1: Setup

- [ ] Implement setup phase
  - Patch ClusterVersion to target EUS channel
  - Annotate NROP CR with pause-reconciliation=enabled
  - Create worker-upgrade MachineConfigPool
  - Set worker-upgrade MCP: paused=true, maxUnavailable=0
  - Add SELinux annotation if OCP_FROM_MINOR < 18

### 3.3 Phase 2: Upgrade NROP Operator

- [ ] Integrate with Phase 2 operator-only playbook
  - Call `nrop_operator_upgrade.yml`
  - Verify pause-reconciliation annotation preserved

### 3.4 Phase 3: Control Plane Upgrade

- [ ] Document integration point
  - External: Call `cluster_upgrade` role or separate process
  - Canary script does NOT handle CP upgrade itself
  - Verification: ClusterVersion Available=True

### 3.5 Phase 4: Migrate Workers

- [ ] Implement migrate phase
  - Get list of worker nodes
  - For each node (or first N if CANARY_NODE_COUNT > 0):
    - Add node to worker-upgrade MCP selector
    - Unpause worker-upgrade MCP
    - Wait for node to upgrade and become Ready
    - Verify RTE pod is Running on upgraded node
    - Re-pause worker-upgrade MCP
  - Per-node validation and diagnostics

### 3.6 Phase 5: Cleanup

- [ ] Implement cleanup phase
  - Unpause NROP CR reconciliation (remove annotation)
  - Delete worker-upgrade MachineConfigPool
  - Wait for worker MCP to stabilize
  - Verify all RTE pods are Running
  - Collect final diagnostics

### 3.7 Helper Functions

- [ ] Port `wait_for_mcp_stable()` - MCP stabilization wait
- [ ] Port `wait_for_node_ready()` - Node readiness wait
- [ ] Port `assert_rte_pod_running_on_node()` - Per-node RTE validation
- [ ] Port `assert_rte_pod_ready_for_mcp()` - MCP-specific RTE validation
- [ ] Port `collect_debug_logs()` - Diagnostic collection on error

### 3.8 Testing

- [ ] Test canary skip-version upgrade (4.18 → 4.20)
  - Phase 1: Setup
  - Phase 2: Upgrade NROP operator to 4.20
  - Phase 3: Upgrade CP to 4.19, then 4.20
  - Phase 4: Migrate workers (all nodes)
  - Phase 5: Cleanup
- [ ] Test partial migration (CANARY_NODE_COUNT=2)
- [ ] Test SELinux annotation logic (pre-4.18 source)
- [ ] Test pause-reconciliation preservation
- [ ] Verify RTE pods on all nodes after completion

**Deliverable**: Full canary skip-version upgrade capability

---

## Phase 4: Complete Feature Parity

**Goal**: Match all capabilities from cnf-pipeline2 implementation.

**Scope**: OLMv1, all registry paths, wrapper scripts, Jenkins integration

### 4.1 OLMv1 Support

- [ ] Create `tasks/olmv1/` subdirectory
- [ ] Port `tasks/olmv1/install.yml` - OLMv1 installation
- [ ] Port `tasks/olmv1/upgrade.yml` - OLMv1 upgrade path
- [ ] Update `tasks/upgrade-nrop.yml` to route based on `install_type`
- [ ] Test OLMv1 fresh installation
- [ ] Test OLMv1 upgrade

### 4.2 Additional Registry Paths

- [ ] Port `tasks/olmv0/stage-prod.yml` - Stage → Production upgrade
- [ ] Port `tasks/olmv0/prod-stage.yml` - Production → Stage upgrade
- [ ] Port `tasks/olmv0/stage-stage.yml` - Stage → Stage upgrade
- [ ] Update `tasks/upgrade-nrop.yml` to route all paths
- [ ] Test stage→prod upgrade
- [ ] Test prod→stage upgrade
- [ ] Test stage→stage upgrade

### 4.3 Additional Templates

- [ ] Port `templates/image_source_policy.yaml.j2` - Image source policies
- [ ] Port `templates/certificate.yaml.j2` - Certificate management
- [ ] Port `templates/kubeletconfig.yaml.j2` - Kubelet configuration
- [ ] Port `templates/performance_profile.yaml.j2` - Performance profile
- [ ] Update `tasks/apply-certs.yml` to use new templates

### 4.4 Wrapper Scripts

- [ ] Create `scripts/nrop_range_upgrade.sh`
  - Calculate old_ocp_version/new_ocp_version from environment variables
  - Call `nrop_upgrade.yml` with correct parameters
  - Support UPGRADE_TYPE, LONG_UPGRADE, VERIFY_DEPLOYMENT variables

- [ ] Create `scripts/check_nrop_upgrades.sh` (alias/symlink)
  - For Jenkins compatibility with cnf-pipeline2 naming

### 4.5 Jenkins Integration Documentation

- [ ] Create `docs/nrop-upgrade-jenkins-integration.md`
  - Document environment variables
  - Document wrapper script usage
  - Provide Jenkinsfile examples
  - Document artifact collection patterns

### 4.6 Testing

- [ ] Test OLMv1 fresh install + upgrade
- [ ] Test all registry path combinations (6 total)
- [ ] Test wrapper scripts with Jenkins-style environment variables
- [ ] Test performance profile integration
- [ ] End-to-end Jenkins pipeline test (if available)

**Deliverable**: Complete feature parity with cnf-pipeline2 implementation

---

## Optional Enhancements

**Goal**: Improvements beyond cnf-pipeline2 implementation

### Option 1: Pure Ansible Canary

- [ ] Convert `nrop-canary-skip-upgrade.sh` to pure Ansible tasks
  - `tasks/canary/setup.yml`
  - `tasks/canary/migrate.yml`
  - `tasks/canary/cleanup.yml`
- [ ] Benefits: Unified language, easier to maintain
- [ ] Tradeoff: More porting work, need to replicate bash logic carefully

### Option 2: Multi-Registry Support

- [ ] Support custom registry URLs beyond prod/stage
- [ ] Support disconnected/air-gapped deployments
- [ ] Mirror NROP operator catalogs to internal registry

### Option 3: Enhanced Validation

- [ ] Add NUMA topology validation
- [ ] Add resource allocation validation
- [ ] Add scheduler metrics collection
- [ ] Add performance benchmarking

### Option 4: Integration with Existing Roles

- [ ] Integrate with `cluster_upgrade` role for coordinated OCP+NROP upgrades
- [ ] Add NROP upgrade to `deploy-ocp-operators.yml` workflow
- [ ] Create unified upgrade playbook for full stack

---

## Testing Checklist

### Unit Tests (Per Phase)

- [ ] Phase 1: Range upgrade (z-stream, minor)
- [ ] Phase 2: Operator-only upgrade
- [ ] Phase 3: Canary skip-version upgrade
- [ ] Phase 4: OLMv1, all registry paths

### Integration Tests

- [ ] Fresh cluster install → NROP install → z-stream upgrade
- [ ] Fresh cluster install → NROP install → minor upgrade (4.17→4.18)
- [ ] Fresh cluster install → NROP install → multi-hop upgrade (4.17→4.20)
- [ ] Fresh cluster install → NROP install → canary upgrade (4.18→4.20)

### Regression Tests

- [ ] Verify no impact on existing `nrop_config` role
- [ ] Verify no impact on existing `deploy_nrop_tests` role
- [ ] Verify no impact on `cluster_upgrade` role

### Documentation Tests

- [ ] All examples in docs/ run successfully
- [ ] All variable references are accurate
- [ ] All file paths are correct

---

## Documentation Deliverables

- [x] `docs/nrop-upgrade-plan.md` - Comprehensive technical plan (this document's companion)
- [x] `docs/nrop-upgrade-roadmap.md` - This implementation roadmap
- [ ] `docs/nrop-upgrade-usage.md` - User guide with examples
- [ ] `docs/nrop-upgrade-jenkins-integration.md` - Jenkins pipeline integration guide
- [ ] `playbooks/compute/roles/nrop_upgrade/README.md` - Role documentation
- [ ] Update `CLAUDE.md` with NROP upgrade section

---

## Success Criteria

### Phase 1 Complete
- ✅ Can perform fresh NROP installation
- ✅ Can perform z-stream upgrades
- ✅ Can perform minor upgrades
- ✅ Artifacts collected correctly
- ✅ Works with OCP 4.17-4.21+

### Phase 2 Complete
- ✅ Can perform operator-only upgrades
- ✅ All 4 scheduler validation tests pass
- ✅ JUnit reports generated
- ✅ Works standalone or as part of canary

### Phase 3 Complete
- ✅ Can perform skip-version canary upgrades
- ✅ All 5 phases execute successfully
- ✅ Supports partial migration (N nodes)
- ✅ Supports full migration (all nodes)
- ✅ SELinux annotation logic correct
- ✅ Pause-reconciliation preserved

### Phase 4 Complete
- ✅ OLMv1 support working
- ✅ All 6 registry path combinations working
- ✅ Jenkins integration tested
- ✅ Full feature parity with cnf-pipeline2

---

## Notes

- Each phase builds on previous phases
- Testing should be comprehensive at each phase before proceeding
- Phases can be merged/split based on development velocity
- Optional enhancements can be done in parallel or deferred
- Focus on production registry (prod→prod) first, then add stage support
- Document any deviations from cnf-pipeline2 implementation
- Keep backwards compatibility with existing eco-ci-cd roles

---

## Getting Started

**First Steps**:
1. Read `docs/nrop-upgrade-plan.md` for full technical details
2. Start with Phase 1.1: Create role directory structure
3. Port one task file at a time, testing as you go
4. Use cnf-pipeline2 as reference, but adapt to eco-ci-cd conventions
5. Update this roadmap by checking boxes as you complete tasks

**Development Workflow**:
1. Create feature branch: `git checkout -b nrop-upgrade-phase1`
2. Implement tasks from Phase 1
3. Test thoroughly
4. Commit with descriptive messages
5. Create PR for review
6. Merge and proceed to Phase 2

**Questions or Issues**:
- Refer to `docs/nrop-upgrade-plan.md` for architectural decisions
- Check cnf-pipeline2 implementation for reference
- Test on non-production clusters first
- Document any changes to the plan in this roadmap
