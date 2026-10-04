---
title: cluster-upgrade-caas-phase1
authors:
  - vemporop@redhat.com
creation-date: 2026-09-22
last-updated: 2026-10-04
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-1415
prd:
  - "prd.md"
see-also:
  - "../OSAC-1269-cluster-version-api/design.md"
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Cluster Upgrade — CaaS (Phase 1)

## Summary

Phase 1 of CaaS cluster upgrades lets tenants independently upgrade the control plane and individual node pools of an HCP OpenShift cluster by patching `Cluster.spec.version` (CP) or `Cluster.spec.node_sets[*].version` (per-NP). The fulfillment-service validates the request and resolves the target version to a release image. The osac-operator detects image divergence on an existing `HostedCluster` or `NodePool` and directly patches `spec.release.image` without triggering AAP re-provision. Only one upgrade operation runs at a time; upgrades are blocked when `Cluster.status.state` is `FAILED`, `DELETING`, or `DELETE_FAILED`, or when `conditions[CanUpgrade]` is not yet `True`.

## Motivation

OSAC provisions HyperShift Hosted Control Plane clusters via two control loops: the fulfillment-service projects DB state to `ClusterOrder` CRs, and the osac-operator provisions via AAP. The operator has read-only access to `HostedCluster` and `NodePool`; all mutations — including scaling — are applied through AAP. Tenants have no API surface to upgrade a cluster, and any `releaseImage` change on `ClusterOrder` triggers a full AAP re-provision.

Phase 1 adds upgrade support by having the osac-operator patch `HostedCluster.spec.release.image` and `NodePool.spec.release.image` directly, bypassing AAP.

### Goals

- Enable independent CP upgrades via `PATCH Cluster.spec.version`.
- Enable independent per-NP upgrades via `PATCH Cluster.spec.node_sets[*].version`.
- Enforce sequential upgrades: only clusters with `conditions[CanUpgrade]=True` accept upgrade spec mutations.
- Enforce version skew: NP version ≤ CP version, within N-3 minor versions of CP; CP upgrade target must not leave any existing node pool more than N-3 minor versions behind.
- Reject downgrade attempts and OBSOLETE version targets.
- Surface upgrade state, observed version, and upgrade history in `Cluster.status`.
- Support version upgrades via `osac edit cluster` (interactive) and a new non-interactive `osac upgrade cluster` command (`scale cluster` pattern: positional cluster name, `--control-plane` for CP targeting, `--node-set` for NP targeting, `--version` accepting `ClusterVersion.metadata.name` or `ClusterVersion.spec.version` string).
- Extend the OSAC UI: version selection with pre-filtering (only eligible versions shown — not OBSOLETE; excludes downgrades against current observed version; skew rules applied client-side), upgrade status monitoring, upgrade history, and upgrade action gating (trigger disabled with tooltip when `status.state ∈ {DELETING, DELETE_FAILED, FAILED}` or `conditions[CanUpgrade] != True`, surfacing the blocking reason) [NFR-1].
- Grant osac-operator write access to `HostedCluster` and `NodePool` resources.

### Non-Goals

- Concurrent upgrades (CP + NP simultaneously, or NP + NP) — one operation at a time is enforced; concurrent upgrades are a non-goal.
- Channel switching and version discovery via the OpenShift upgrade graph / OSUS — deferred to Phase 2 (FR-3).
- Risk review or explicit risk acknowledgment — auto-acknowledged in Phase 1; deferred to Phase 2 (FR-4, FR-5).
- Cancellation window before HyperShift propagation — deferred to Phase 2 (FR-6).
- Version divergence notifications when NPs lag behind CP (FR-14).
- EOL/limited-support visibility (FR-15).
- Tenant Admin fleet view for clusters requiring upgrades (NFR-admin).
- SNO or non-HCP clusters.
- Rollback or downgrade.
- Platform-initiated upgrades.
- Cancel running upgrades (HyperShift does not support it).
- AAP playbook changes — Phase 1 upgrades bypass AAP entirely.
- OSAC-owned persistent upgrade history — Phase 1 relays HyperShift's limited history; operator records NP completion.

## Proposal

Phase 1 adds two independent upgrade paths to the OSAC cluster API:

1. **CP upgrade:** tenant PATCHes `spec.version`; fulfillment-service validates (including N-3 skew) and resolves to `ClusterOrder.spec.ReleaseImage`; operator detects HC image divergence and patches `HostedCluster.spec.release.image`.
2. **NP upgrade:** tenant PATCHes `spec.node_sets[<id>].version`; fulfillment-service validates (including N-3 skew) and resolves to `ClusterOrder.spec.nodeRequests[<id>].ReleaseImage`; operator detects NP image divergence and patches the specific `NodePool.spec.release.image`.

Both paths bypass AAP. Fulfillment owns the `CanUpgrade` condition in its DB for initial HyperShift creation and upgrades: it sets `False` in the Cluster creation or upgrade-acceptance transaction and `True` in the transaction that records initial cluster readiness or a terminal upgrade result (success or failure). The osac-operator monitors HyperShift and reports status through the existing private Cluster Update path. ClusterOrder provisioning status is not changed by an upgrade.

### Workflow Description

#### High-Level Pipeline

**Control plane upgrade:**

```
User
 │
 ▼
CLI (upgrade_cmd.go / edit_cmd.go)
 │  gRPC: ClustersUpdateRequest spec.version={name: "4-17-3"}
 │  update_mask: ["spec.version"]
 ▼
Fulfillment-Service API (clusters_server.go)
 │  1. validateUpgradeEligibility: state ∉ {DELETING,DELETE_FAILED,FAILED}, CanUpgrade=True
 │  2. validateVersionUpdate: ClusterVersion catalog lookup → resolves spec.image
 │  3. PostgreSQL: persist spec.version + resolved ReleaseImage + CanUpgrade=False (atomic)
 ▼
Cluster Reconciler (cluster_reconciler_function.go)
 │  buildSpec reads pre-resolved ReleaseImage from DB
 │  K8s PATCH: ClusterOrder.spec.ReleaseImage
 ▼
osac-operator (clusterorder_controller.go)
 │  HC image divergence → CP upgrade path
 │  PATCH HostedCluster.spec.release.image
 │  upgradeStatus.state = Pending
 ▼
HyperShift Controller
 │  desired.version == target → upgradeStatus.state = Progressing
 │  history[0].state == Completed → upgradeStatus.state = Succeeded
 ▼
Status Feedback (feedback_controller.go)
 │  upgradeStatus → private Update → DB releases lock on successful completion or terminal upgrade failure
 ▼
CLI / UI (upgrade state and observed version updated)
```

**Node pool upgrade** (same structure; `spec.node_sets[i].version` and `NodePool[i]` instead of HC):

```
User
 │
 ▼
CLI (upgrade_cmd.go / edit_cmd.go)
 │  gRPC: ClustersUpdateRequest spec.node_sets[i].version=4.16.5
 │  update_mask: ["spec.node_sets"]
 ▼
Fulfillment-Service API (clusters_server.go)
 │  1. validateUpgradeEligibility
 │  2. validateNPVersionUpdate: ClusterVersion catalog lookup → resolves spec.image
 │  3. PostgreSQL: persist spec.node_sets[i].version + resolved ReleaseImage + CanUpgrade=False (atomic)
 ▼
Cluster Reconciler (cluster_reconciler_function.go)
 │  buildSpec reads pre-resolved ReleaseImage from DB
 │  K8s PATCH: ClusterOrder.spec.nodeRequests[i].{ReleaseImage,Version}
 ▼
osac-operator (clusterorder_controller.go)
 │  NodePool[i] image divergence → NP upgrade path
 │  PATCH NodePool[i].spec.release.image
 │  upgradeStatus.state = Pending
 ▼
HyperShift Controller
 │  conditions[UpdatingVersion]=True → upgradeStatus.state = Progressing
 │  UpdatingVersion=False + version==target → upgradeStatus.state = Succeeded
 ▼
Status Feedback (feedback_controller.go)
 │  upgradeStatus + observed_version → private Update → DB releases lock on successful completion or terminal upgrade failure
 ▼
CLI / UI (upgrade state and observed version updated)
```

#### Step 1 — CLI (`osac upgrade cluster`)

**Source file:** `fulfillment-service/cmd/osac/upgrade/upgrade_cmd.go`

```bash
osac upgrade cluster <cluster-name> --control-plane --version 4.17.3
osac upgrade cluster <cluster-name> --node-set compute --version 4.16.5
```

Interactive upgrades are also available via `osac edit cluster`.

The CLI performs the following steps:

1. Looks up the cluster by name or ID.
2. Validates that `--version` is provided and exactly one of `--control-plane` or `--node-set` is specified.
3. Resolves `--version` as `osac create cluster` does: match `ClusterVersion.metadata.name` or `ClusterVersion.spec.version`, preferring the name if both match different versions. Then clone and mutate the cluster proto (using the selected version's metadata name for CP and semantic version for NP):
   - CP: `updated.GetSpec().SetVersion(publicv1.ClusterVersionReference_builder{Name: versionName}.Build())`
   - NP: `updated.GetSpec().GetNodeSets()[nodeSetName].SetVersion(newVersion)`
4. Sends update with a field mask:
   ```go
   client.Update(ctx, publicv1.ClustersUpdateRequest_builder{
       Object:     updated,
       UpdateMask: &fieldmaskpb.FieldMask{Paths: []string{"spec.version"}},  // or "spec.node_sets"
   }.Build())
   ```
5. Prints: `"Run 'osac describe cluster <name>' to monitor progress."`

The CLI has no kubeconfig and never calls the Kubernetes API.

#### Step 2 — Fulfillment-Service API

**Source file:** `fulfillment-service/internal/servers/clusters_server.go`

Resolve and validate the target `ClusterVersion` before locking the Cluster row. Then use the generic update path to lock the row (`SELECT ... FOR UPDATE`), apply the update mask, and check upgrade eligibility, observed versions, and skew against the locked Cluster.

**`validateUpgradeEligibility`** (new, called after locking the Cluster):
1. Reject `FAILED_PRECONDITION` if `state ∈ {DELETING, DELETE_FAILED, FAILED}`.
2. Reject `FAILED_PRECONDITION` if `conditions[CAN_UPGRADE].status != True`. The error message includes the blocking reason from the condition.

`PROGRESSING + CanUpgrade=True` passes both checks — AAP post-provisioning tasks may still be running, but the Cluster `READY` condition has been reported and a new upgrade can be accepted.

**`validateVersionUpdate` (CP) / `validateNPVersionUpdate` (NP)**:

Before taking the Cluster row lock, look up the target `ClusterVersion` in the OSAC catalog. Validate it and capture its release image (`ClusterVersion.spec.image`). The checks against the Cluster's observed versions run after the lock is acquired:

| Check | CP | NP |
|---|---|---|
| `ClusterVersion` exists, enabled, not OBSOLETE | ✓ (DEPRECATED allowed) | ✓ |
| target > observed current version | target > `observed_cp_version` | target > `node_sets[i].observed_version` |
| no downgrade | ✓ | ✓ |
| version skew | CP target leaves no NP > 3 minor versions behind | target NP ≤ CP version; `CP_minor − NP_minor ≤ 3` |

**Database write** (one transaction):
- With the row locked, recheck `CanUpgrade=True`; otherwise return `FAILED_PRECONDITION`.
- Save the requested CP or NP version, resolved `ReleaseImage`, and `CanUpgrade=False` together.

Hold the lock through commit. A concurrent request then sees `CanUpgrade=False` and is rejected.

#### Step 3 — Cluster Reconciler → ClusterOrder Patch

**Source file:** `fulfillment-service/internal/controllers/cluster/cluster_reconciler_function.go`

The reconciler loop reads the pre-resolved `ReleaseImage` stored in the DB cluster record and patches the `ClusterOrder` CR. No separate catalog lookup is needed:

| DB field | ClusterOrder field |
|---|---|
| resolved CP `ReleaseImage` | `spec.ReleaseImage` |
| resolved NP `ReleaseImage` | `spec.nodeRequests[i].ReleaseImage` |
| `spec.node_sets[i].version` | `spec.nodeRequests[i].Version` |

`ReleaseImage`, `nodeRequests[*].ReleaseImage`, and `nodeRequests[*].Version` are excluded from `DesiredConfigVersion` hash computation to prevent triggering AAP re-provision when only version fields change.

#### Step 4 — osac-operator Image Divergence Detection

**Source file:** `osac-operator/internal/controller/clusterorder_controller.go`

On each reconcile, the operator compares desired vs. observed release images:

- **CP:** `ClusterOrder.spec.ReleaseImage ≠ HostedCluster.spec.release.image` and HC exists → CP upgrade path.
- **NP:** `ClusterOrder.spec.nodeRequests[i].ReleaseImage ≠ NodePool[i].spec.release.image` and NP exists → NP upgrade path for that pool.

The API accepted the upgrade only after checking the DB lock for the previous operation. On divergence, the operator patches the existing resource without waiting for HyperShift to match the newly requested target:

- **CP:** `PATCH HostedCluster.spec.release.image = ClusterOrder.spec.ReleaseImage`
- **NP:** `PATCH NodePool[i].spec.release.image = ClusterOrder.spec.nodeRequests[i].ReleaseImage`

`upgradeStatus.state` remains `Pending` after the patch — the operator waits for HyperShift to confirm the upgrade has started.

If no image divergence exists and no upgrade is in flight, the existing `DesiredConfigVersion` hash comparison drives AAP provisioning (provision path unchanged).

#### Step 5 — HyperShift Signals Upgrade Started

After the HC or NP patch, the operator monitors for HyperShift's upgrade-started signal on each reconcile:

- **CP:** `HC.status.controlPlaneVersion.desired.version == target_version` → set `upgradeStatus.state = Progressing`, `startTime = now`.
- **NP:** `NodePool[i].status.conditions[UpdatingVersion].status == True` → set `upgradeStatus.state = Progressing`, `startTime = now`.

The feedback controller sends each `upgradeStatus` change through the private Cluster Update path, so the fulfillment-service reflects `Progressing` state promptly.

#### Step 6 — HyperShift Reconciliation

HyperShift's controllers perform the actual upgrade. The operator monitors for completion on each reconcile:

**CP:** The cluster version operator (CVO) upgrades the control plane components. Completion criteria:
- `HC.status.controlPlaneVersion.history[0].image == ClusterOrder.spec.ReleaseImage`
- `HC.status.controlPlaneVersion.history[0].state == "Completed"`

**NP:** The NodePool controller re-provisions worker nodes to the target version. Completion criteria:
- `NodePool[i].status.conditions[UpdatingVersion].status == False`
- `NodePool[i].status.version == ClusterOrder.spec.nodeRequests[i].Version`

#### Step 7 — Status Propagation (Feedback Loop)

**Source file:** `osac-operator/internal/controller/feedback_controller.go`

On completion, the operator updates `ClusterOrder.status`:
1. Sets `upgradeStatus.state = Succeeded`, `upgradeStatus.completionTime = now`.
2. Sets `ObservedVersion` from `HC.status.controlPlaneVersion.history[0].version` (CP) or records `node_sets[i].observed_version` in the feedback payload (NP).
3. Appends an `UpgradeHistoryEntry`.

The feedback controller sends the status through the existing private Cluster Update path. Fulfillment persists `status.upgrade` and advances the observed CP or NP version on success. When feedback records `Succeeded` or terminal `Failed` for the current component and target version, the same DB transaction sets `conditions[CAN_UPGRADE]=True`; feedback for an earlier target cannot release the lock. A failed upgrade leaves the observed version unchanged. For initial cluster creation, fulfillment records cluster readiness (control plane and node pools) and sets `CanUpgrade=True` in one DB transaction while no upgrade is active. Incoming feedback cannot overwrite this DB-owned lock directly. The `Signal` RPC only queues reconciliation by cluster ID; it carries no status payload.

---

#### End-to-End Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                            USER                                     │
│  $ osac upgrade cluster my-cluster --control-plane --version 4.17.3│
│  $ osac upgrade cluster my-cluster --node-set compute --version 4.16.5│
└────────────────────────┬────────────────────────────────────────────┘
                         │ gRPC: ClustersUpdateRequest
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│            FULFILLMENT-SERVICE  (clusters_server.go)                │
│  1. validateUpgradeEligibility: state, CanUpgrade=True              │
│  2. validateVersionUpdate / validateNPVersionUpdate:                │
│       ClusterVersion catalog lookup → ClusterVersion.spec.image     │
│  3. PostgreSQL: spec.version + ReleaseImage + CanUpgrade=False      │
└────────────────────────┬────────────────────────────────────────────┘
                         │ Reconciler loop tick
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│      CLUSTER RECONCILER  (cluster_reconciler_function.go)           │
│  1. Read DB: ReleaseImage (pre-resolved)                            │
│  2. PATCH ClusterOrder.spec.ReleaseImage (CP)                       │
│     or ClusterOrder.spec.nodeRequests[i].{ReleaseImage,Version} (NP)│
└────────────────────────┬────────────────────────────────────────────┘
                         │ controller-runtime watch event
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│         OSAC-OPERATOR  (clusterorder_controller.go)                 │
│  1. Image divergence detected → upgrade path                        │
│  2. upgradeStatus.state = Pending                                   │
│  3. PATCH HC or NodePool spec.release.image                          │
│  4. Monitor: desired.version==target / UpdatingVersion=True         │
│       → upgradeStatus.state = Progressing                           │
│  5. Monitor target completion → upgradeStatus.state = Succeeded     │
└────────────────────────┬────────────────────────────────────────────┘
                         │ NodePool/HostedCluster status watch
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│              HYPERSHIFT CONTROLLER                                  │
│  CP: CVO upgrades control plane; history[0].state=Completed         │
│  NP: NodePool controller re-provisions workers; status.version=target│
└────────────────────────┬────────────────────────────────────────────┘
                         │ controller-runtime watch (ClusterOrder status)
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│      STATUS FEEDBACK  (feedback_controller.go)                      │
│  upgradeStatus + observed version → private Cluster Update          │
│  DB stores terminal result and sets CanUpgrade=True                 │
└─────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
              User runs: osac describe cluster my-cluster
              Output shows: upgrade Succeeded, observed version updated
```

#### Upgrade status lifecycle

```mermaid
stateDiagram-v2
    [*] --> Pending : upgrade accepted, CanUpgrade=False
    Pending --> Progressing : HyperShift signals upgrade in progress
    Progressing --> Succeeded : completion criteria met
    Pending --> Failed : upgrade cannot proceed (criteria TBD)
    Progressing --> Failed : upgrade cannot proceed (criteria TBD)
    Succeeded --> [*] : CanUpgrade restored to True
    Failed --> [*] : CanUpgrade restored to True
```

**Upgrade does not change ClusterOrder provisioning status.** The ClusterOrder phase and conditions reflect provisioning; they are not changed when an upgrade is accepted, succeeds, or fails. Upgrade state is tracked only in `ClusterOrder.status.upgradeStatus` and `Cluster.status.upgrade`. A terminal failure means the upgrade cannot proceed, not necessarily that the cluster has failed. The exact terminal-failure criteria are deferred.

**API-level blocking:** Upgrade requests are rejected if `state ∈ {DELETING, DELETE_FAILED, FAILED}`, or if the DB's `conditions[CanUpgrade].status != True`. Fulfillment sets this per-cluster lock to `False` in the Cluster creation or upgrade-acceptance transaction and to `True` in the corresponding feedback transaction. `PROGRESSING + CanUpgrade=True` allows upgrades — the Cluster `READY` condition can be True while AAP post-provisioning tasks are still running.

**Cluster-internal health states are not an API gate.** HC degraded, CP temporarily unreachable, or NP unhealthy do not affect `CanUpgrade`. If an upgrade is submitted while these conditions are true, the API accepts it and the operator patches HC/NP. Whether the upgrade succeeds depends on HyperShift.


### API Extensions

#### Proto additions (`proto/private/osac/private/v1/cluster_type.proto`)

```protobuf
// Existing ClusterSpec field (used for CP upgrades; no new field required):
ClusterVersionReference version = 6;

// ClusterNodeSet additions:
string version = 5;           // desired NP version; settable via PATCH
string observed_version = 6;  // output_only; from NodePool.status.version after upgrade

// ClusterStatus additions:
string observed_cp_version = 12;                   // output_only; from HC.status.controlPlaneVersion
ClusterUpgradeStatus upgrade = 13;                 // output_only; active or most recent upgrade
repeated ClusterVersionHistoryEntry version_history = 14;  // output_only

// ClusterConditionType addition (the fulfillment DB owns this lock):
CLUSTER_CONDITION_TYPE_CAN_UPGRADE = 5;

// New messages:
message ClusterUpgradeStatus {
    ClusterUpgradeProgressState state = 1;
    string from_version = 2;
    string to_version = 3;
    google.protobuf.Timestamp started_at = 4;
    google.protobuf.Timestamp completed_at = 5;
    string message = 6;
    string component = 7;  // "control_plane" | "node_pool:<id>"
}

message ClusterVersionHistoryEntry {
    string from_version = 1;
    string to_version = 2;
    google.protobuf.Timestamp completed_at = 3;
    bool success = 4;
    string component = 5;  // "control_plane" | "node_pool:<id>"
}

enum ClusterUpgradeProgressState {
    CLUSTER_UPGRADE_PROGRESS_STATE_UNSPECIFIED = 0;
    CLUSTER_UPGRADE_PROGRESS_STATE_PENDING = 1;      // accepted; waiting for HyperShift progress
    CLUSTER_UPGRADE_PROGRESS_STATE_PROGRESSING = 2;
    CLUSTER_UPGRADE_PROGRESS_STATE_SUCCEEDED = 3;
    CLUSTER_UPGRADE_PROGRESS_STATE_FAILED = 4;
}
```

After proto changes, run `make -C proto generate` and commit generated code under `proto/gen/`.

#### NodeRequest additions (`osac-operator/api/v1alpha1/clusterorder_types.go`)

```go
type NodeRequest struct {
    ResourceClass string `json:"resourceClass"`
    NumberOfNodes int    `json:"numberOfNodes"`
    // ReleaseImage is the resolved OCI pullspec for this node pool.
    ReleaseImage  string `json:"releaseImage,omitempty"`
    // Version is the semver string corresponding to ReleaseImage; populated by
    // the fulfillment-service reconciler. Used by the operator to compare against
    // NodePool.status.version without requiring catalog access.
    Version       string `json:"version,omitempty"`
}
```

#### ClusterOrderStatus additions

```go
type ClusterOrderStatus struct {
    // ... existing fields ...
    ObservedVersion string                `json:"observedVersion,omitempty"`
    UpgradeStatus   *ClusterUpgradeStatus `json:"upgradeStatus,omitempty"`
}

type ClusterUpgradeStatus struct {
    State          UpgradeStateType      `json:"state,omitempty"`
    Component      string                `json:"component,omitempty"` // "control_plane" | "node_pool:<id>"
    FromVersion    string                `json:"fromVersion,omitempty"`
    ToVersion      string                `json:"toVersion,omitempty"`
    StartTime      *metav1.Time          `json:"startTime,omitempty"`
    CompletionTime *metav1.Time          `json:"completionTime,omitempty"`
    Message        string                `json:"message,omitempty"`
    History        []UpgradeHistoryEntry `json:"history,omitempty"`
}

type UpgradeHistoryEntry struct {
    Component      string           `json:"component"`
    FromVersion    string           `json:"fromVersion"`
    ToVersion      string           `json:"toVersion"`
    StartTime      *metav1.Time     `json:"startTime"`
    CompletionTime *metav1.Time     `json:"completionTime,omitempty"`
    State          UpgradeStateType `json:"state"`
}

type UpgradeStateType string

const (
    UpgradeStatePending     UpgradeStateType = "Pending"
    UpgradeStateProgressing UpgradeStateType = "Progressing"
    UpgradeStateSucceeded   UpgradeStateType = "Succeeded"
    UpgradeStateFailed      UpgradeStateType = "Failed"
)
```

#### DB migration

A numbered SQL migration backfills `status.observed_cp_version` from `spec.version.name` for existing clusters:

```sql
update clusters
set data = jsonb_set(
  coalesce(data, '{}'::jsonb),
  '{status,observed_cp_version}',
  to_jsonb(data->'spec'->'version'->>'name')
)
where data->'spec'->'version'->'name' is not null
  and (data->'status'->>'observed_cp_version' is null
       or data->'status'->>'observed_cp_version' = '');
```

No new tables; changes are to the JSONB `data` column. Fulfillment creates new Clusters with DB-owned `CAN_UPGRADE=False` in the same transaction. For existing Clusters, the numbered migration uses one atomic SQL `UPDATE` to set exactly one `CAN_UPGRADE` condition: `True` when persisted `status.conditions[READY].status=True`, `False` otherwise, preserving other conditions. `READY` reflects `ClusterOrder`'s `ClusterAvailable` condition and can be `True` while its phase is `Progressing`.

Current design phase will accept **the following caveat**: The condition does not prove every NodePool is ready or that requested versions have converged, so the migration can unlock a Cluster whose workers are still converging.

### Implementation Details

#### Version validation in fulfillment-service

Resolve and validate the target `ClusterVersion` before taking the Cluster row lock. Use a non-locking read of the stored Cluster to check the version reference's scope, including shared versions; the later locked read is authoritative for upgrade state and skew. Like cluster provisioning, the catalog read uses the request transaction but does not lock the `ClusterVersion`. Check `enabled` and `state` when reading it, then use its immutable `version` and `image` for the accepted request.

`validateUpgradeEligibility` (new, called after the row lock):
1. Reject with `FAILED_PRECONDITION` if `state ∈ {DELETING, DELETE_FAILED, FAILED}`.
2. Reject with `FAILED_PRECONDITION` if `conditions[CAN_UPGRADE].status != True` — initial cluster readiness feedback or a prior upgrade's terminal result is pending. The error message includes the blocking reason from the condition (e.g. `"cluster is not ready yet"`, `"control plane upgrade in progress"`).

`PROGRESSING + CanUpgrade=True` passes both checks — AAP post-provisioning tasks are running but the Cluster `READY` condition was reported, so a new upgrade can be accepted. `CanUpgrade` captures operation-completion state, not cluster health.

`validateVersionUpdate` (`fulfillment-service/internal/servers/private_clusters_server.go`):

**CP upgrade (`spec.version` change):**
1. Target `ClusterVersion` must exist, be enabled, and not OBSOLETE. Rejects with `INVALID_ARGUMENT`. DEPRECATED is allowed.
2. Target semver > `status.observed_cp_version`. Rejects equal or lesser values with `INVALID_ARGUMENT`.
3. For each existing node pool: `target_CP_minor - NP_observed_minor ≤ 3`. Rejects with `INVALID_ARGUMENT` if the target CP version would leave any node pool more than 3 minor versions behind.

**NP upgrade (`spec.node_sets[<id>].version` change):**
1. Target `ClusterVersion` must exist, be enabled, and not OBSOLETE. Rejects with `INVALID_ARGUMENT`.
2. Target semver > `status.node_sets[<id>].observed_version`. No downgrades.
3. Target semver ≤ `status.observed_cp_version`. NP version must not exceed CP version (including patch).
4. `CP_minor - target_NP_minor ≤ 3`. N-3 minor version skew constraint.

After locking the Cluster row, run `validateUpgradeEligibility` and the observed-version and skew checks against the masked request. If `CanUpgrade` is no longer `True`, return `FAILED_PRECONDITION`. Otherwise, save the requested version, pre-resolved `ReleaseImage`, `status.upgrade=Pending`, and `CanUpgrade=False` in one transaction. Private status feedback updates `status.upgrade`; fulfillment restores `CanUpgrade=True` in the transaction that records `Succeeded` or terminal `Failed`.

For version updates, move the `validateClusterStateForSpecUpdate` state check after the catalog lookup and into the locked update path. Other spec updates keep its current behavior.

#### Upgrade lock ownership

Fulfillment initializes `CanUpgrade=False` in the same transaction that creates the Cluster. For initial provisioning, the feedback controller sends the existing `ClusterAvailable` observation as `Cluster.status.conditions[READY]` through private Cluster Update. Fulfillment locks the Cluster row and stores `READY=True` with `CanUpgrade=True` in one transaction only while `status.upgrade` is unset. Later HC/NP health changes do not change the lock.

For an upgrade, the API stores the target and takes the DB lock in one acceptance transaction. The operator patches HC/NP while `CanUpgrade=False` and reports progress through private status feedback. The private Cluster Update handler releases the DB lock in the transaction that records success or terminal failure for the current component and target version.

#### buildSpec change in fulfillment-service reconciler

`buildSpec` in `fulfillment-service/internal/controllers/cluster/cluster_reconciler_function.go`:
- Reads the pre-resolved `ReleaseImage` from the DB cluster record (stored by the API handler as part of the `validateVersionUpdate` transaction) and sets `ClusterOrder.spec.ReleaseImage` (CP image).
- Reads the pre-resolved per-NP `ReleaseImage` and `Version` from the DB cluster record and sets `ClusterOrder.spec.nodeRequests[<id>].ReleaseImage` (pullspec) and `ClusterOrder.spec.nodeRequests[<id>].Version` (semver). No separate catalog lookup is needed.
- `ReleaseImage`, `nodeRequests[*].ReleaseImage`, and `nodeRequests[*].Version` are excluded from `DesiredConfigVersion` hash computation to prevent triggering AAP re-provision when only version fields change.

#### osac-operator upgrade reconciliation

In `clusterorder_controller.go`, the reconcile loop gains upgrade awareness:

**CP upgrade path:**
1. Compare `ClusterOrder.spec.ReleaseImage` with `HostedCluster.spec.release.image`. If HC exists and images differ → CP upgrade path.
2. Set `upgradeStatus = {state: Pending, component: "control_plane", fromVersion: observedVersion, toVersion: spec.version.name}`.
3. Patch `HostedCluster.spec.release.image = ClusterOrder.spec.ReleaseImage`; state remains `Pending`.
4. Monitor: when `HC.status.controlPlaneVersion.desired.version == target_version` → set `upgradeStatus.state = Progressing`, `startTime = now`.
5. Monitor completion: `HC.status.controlPlaneVersion.history[0].image == ClusterOrder.spec.ReleaseImage` AND `history[0].state == "Completed"`.
6. On completion: set `status.observedVersion` from `HC.status.controlPlaneVersion.history[0].version`; append `UpgradeHistoryEntry`; set `upgradeStatus.state = Succeeded`. ClusterOrder phase is not changed.

**NP upgrade path:**
1. Compare `ClusterOrder.spec.nodeRequests[i].ReleaseImage` with `NodePool[i].spec.release.image`. If NP exists and images differ → NP upgrade path for that pool.
2. Set `upgradeStatus = {state: Pending, component: "node_pool:<id>", fromVersion: observed, toVersion: target}`.
3. Patch `NodePool[i].spec.release.image = nodeRequests[i].ReleaseImage`; state remains `Pending`.
4. Monitor: when `NodePool[i].status.conditions[UpdatingVersion].status == True` → set `upgradeStatus.state = Progressing`, `startTime = now`.
5. Monitor completion: `NodePool[i].status.conditions[UpdatingVersion].status == False` AND `NodePool[i].status.version == ClusterOrder.spec.nodeRequests[i].Version`.
6. On completion: set `node_sets[i].observed_version` in the feedback payload; append `UpgradeHistoryEntry`; set `upgradeStatus.state = Succeeded`. ClusterOrder phase is not changed.

**Provision path** (unchanged): if HC does not yet exist or no image divergence on HC or any NP, the existing `DesiredConfigVersion` hash comparison drives AAP provisioning.

RBAC marker expanded:

```go
// +kubebuilder:rbac:groups=hypershift.openshift.io,resources=hostedclusters;nodepools,verbs=get;list;watch;patch;update
```

NodePools are discovered by listing within the cluster's namespace by `osac.openshift.io/resource_class` label selector.

#### HyperShift status fields used

| Purpose | Field | Notes |
|---------|-------|-------|
| Initial provisioning confirmation | `Cluster.status.conditions[READY].status == True` | Mapped from `ClusterOrder.conditions[ClusterAvailable]`; releases the DB lock even while phase is Progressing |
| HC version completion | `HC.status.controlPlaneVersion.history[0].image` | Must equal `ClusterOrder.spec.ReleaseImage` |
| HC version completion | `HC.status.controlPlaneVersion.history[0].state` | Must equal `"Completed"` |
| NP upgrade completion | `NodePool.status.version` | The upgraded NP must match its requested version |
| CP current version | `HC.status.controlPlaneVersion.history` (first `Completed` entry) | Semver string |
| CP upgrade in progress | `controlPlaneVersion.history[0].state == Partial` | No `completionTime` |
| CP target during upgrade | `controlPlaneVersion.desired.version` | Display as "upgrading to X" |
| CP upgrade started signal | `HC.status.controlPlaneVersion.desired.version == target_version` | Pending → Progressing transition for CP |
| NP upgrade started signal | `NodePool.status.conditions[UpdatingVersion].status == True` | Pending → Progressing transition for NP |
| CP history | `controlPlaneVersion.history[]` | maxItems: 100 |
| NP current version | `NodePool.status.version` | Flat semver string |
| NP upgrade in progress | `conditions[UpdatingVersion].status == True` | Standard condition |

No `HostedControlPlane` watch needed — all CP status is on `HostedCluster`.

#### History limitation

`controlPlaneVersion.history` is capped at 100 entries by HyperShift. NP upgrade completion events are not available from HyperShift history; Phase 1 records them directly in `ClusterOrder.status.upgradeStatus.history` when the operator observes completion. Comprehensive OSAC-owned persistent history (DB-backed, audit-grade) is deferred to a future phase.

### Security Considerations

- Upgrade requests pass through existing tenant RBAC: only tenants with `Update` permission on their `Cluster` resource can initiate upgrades.
- The osac-operator service account requires expanded RBAC on `hypershift.openshift.io/hostedclusters` and `hypershift.openshift.io/nodepools` (patch/update). These are hub-cluster permissions managed through the existing RBAC marker pattern.
- Target version images are resolved exclusively from the OSAC ClusterVersion catalog; tenants cannot inject arbitrary OCI pullspecs.
- Tenant isolation is preserved: `ClusterOrder` resources are labeled with `osac.openshift.io/tenant`, enforced by existing OPA policies.

### HyperShift Upgrade Failure Surfacing

HyperShift enforces additional upgrade constraints that OSAC's pre-flight validation cannot fully anticipate. The operator reports upgrade-specific errors in `ClusterOrder.status.upgradeStatus.message`; fulfillment exposes them in `Cluster.status.upgrade.message`. An upgrade failure does not change ClusterOrder provisioning conditions or Cluster provisioning state. The criteria for declaring a terminal upgrade failure remain to be defined.

At Phase 1, it is the **tenant's responsibility** to verify that a target version is reachable from the current version before initiating an upgrade. OSAC validates that the target exists in the ClusterVersion catalog, is not OBSOLETE, and satisfies version-skew rules — but does not check upgrade-graph reachability (FR-3 is deferred to Phase 2). Tenants can use the [Red Hat OpenShift Container Platform Update Graph](https://access.redhat.com/labs/ocpupgradegraph/update_path/) to confirm valid upgrade paths.

### Failure Handling and Recovery

| Failure mode | What happens | Recovery | User observes |
|---|---|---|---|
| Retryable HC or NP patch error | Operator retries on next reconcile with backoff. `upgradeStatus.state` remains `Pending`. | Resolve underlying issue; operator resumes automatically. | Upgrade remains Pending. |
| Operator crashes mid-upgrade | On restart, operator re-reads `ClusterOrder.spec.ReleaseImage` and `nodeRequests[*].ReleaseImage` and resumes. Patch calls are idempotent. | Automatic on restart. | Brief gap in status updates. |
| Terminal upgrade failure (criteria TBD) | Upgrade cannot proceed; operator sets `upgradeStatus.state = Failed` without changing ClusterOrder provisioning status. Fulfillment sets `CanUpgrade=True` when it records the result. | Investigate the upgrade failure; recovery details follow the terminal-failure criteria. | `status.upgrade.state = Failed` with message; Cluster state is unchanged. |
| Target version not found in the ClusterVersion catalog | Rejected at `validateVersionUpdate`. | User specifies a valid version name. | `INVALID_ARGUMENT` with message. |
| Version downgrade attempted | Rejected at `validateVersionUpdate`. | User selects a valid target. | `INVALID_ARGUMENT` with message. |
| CP upgrade violates N-3 skew against existing NPs | Rejected at `validateVersionUpdate`. | User must upgrade lagging NPs first, then retry the CP upgrade. | `INVALID_ARGUMENT` with message. |
| NP upgrade violates N-3 skew behind CP | Rejected at `validateNPVersionUpdate`. | User must upgrade CP first or choose a version within skew. | `INVALID_ARGUMENT` with message. |

### RBAC / Tenancy

- No changes to the fulfillment-service tenant RBAC model.
- osac-operator service account RBAC expanded: `patch` and `update` verbs on `hostedclusters` and `nodepools`.
- Feedback uses the existing private Cluster Update path for upgrade status; `Signal` remains unchanged and carries only the cluster ID.
- No changes to OPA policies; upgrade operations are gated by the existing `Update` verb on `Cluster`.

### Observability and Monitoring

- `ClusterOrder.status.upgradeStatus` provides operator-level upgrade visibility.
- The osac-operator emits a Kubernetes Event on the `ClusterOrder` when an upgrade starts, transitions to Progressing, completes, or fails (`Normal` for start/Progressing/complete, `Warning` for failure).

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| `ReleaseImage` hash exclusion triggers spurious AAP re-provision on controller upgrade | Validate in staging: upgrade the controller on a live cluster and verify no AAP jobs are triggered. |
| HyperShift `controlPlaneVersion.history` is capped at 100 entries | Phase 1 relays up to 100 entries; operator-owned history (DB-backed) deferred to a future phase. |
| NP history not available from HyperShift | Operator records NP completion events directly at transition time. |
| Operator has expanded RBAC on HC/NP (write access) | Scope is limited to `patch`/`update` on `hostedclusters`/`nodepools` in the managed namespaces only. |

### Drawbacks

OSAC enforces one operation at a time (no concurrent NP upgrades across different pools). A cluster with multiple node pools cannot run NP upgrades in parallel; each upgrade must complete before the next is accepted.

## Open Questions

### 9.1 NP history from HyperShift — Partially Resolved

NP-only upgrades (worker nodes catching up to an already-running CP version) likely do not produce entries in `HostedCluster.status.version.history` since the cluster CVO version doesn't change. This needs live-cluster verification. Phase 1 records NP completion events from operator observation; HyperShift-sourced NP history deferred to a future phase.

## Test Plan

The test strategy follows the touched-area map for `fulfillment-service` and `osac-operator`.

**Unit tests:**
- `validateVersionUpdate` — CP upgrade: target > current, not OBSOLETE, N-3 skew against existing NPs, and state/lock eligibility.
- `validateNPVersionUpdate` — NP upgrade: target ≤ CP, N-3 skew, target > NP current, and state/lock eligibility.
- `buildSpec` — CP and per-NP image resolution from ClusterVersion catalog.
- Operator upgrade path: image divergence detection, upgrade vs. provision routing.
- DB-owned lock: Cluster creation and upgrade acceptance each store `CanUpgrade=False` with the operation; HyperShift cluster initial readiness or terminal upgrade feedback (success or failure) stores `True` with status in one transaction. Migration seeds existing rows from `READY=True`, including `PROGRESSING` clusters; other rows get `False`.
- `DesiredConfigVersion` hash exclusion: version-only change does not trigger AAP.

**Integration tests:**
- CP upgrade end-to-end: PATCH spec.version → CanUpgrade=False (sync DB write) → ClusterOrder sync → operator patches HC → completion detected → CanUpgrade=True + history entry.
- Initial creation: Cluster and `CanUpgrade=False` are stored together; `READY=True` feedback stores `CanUpgrade=True` while ClusterOrder may still be `Progressing`.
- NP upgrade end-to-end: PATCH spec.node_sets[i].version → CanUpgrade=False (sync DB write) → NP patched → NP completion → CanUpgrade=True + per-NP observed_version.
- Blocking guard: reject upgrade request on DELETING, DELETE_FAILED, FAILED clusters; reject when CanUpgrade=False.
- Concurrent upgrades to one cluster: one succeeds; the other returns FAILED_PRECONDITION. Verify the stored version and ReleaseImage match the winner and CanUpgrade=False.
- Terminal failure releases the lock without changing Cluster or ClusterOrder provisioning status; stale terminal feedback for an earlier target cannot release the lock for a later upgrade.
- N-3 skew rejection: NP upgrade rejected when skew would exceed 3 minor versions; CP upgrade rejected when it would leave any NP more than 3 minor versions behind.
- NP version ≤ CP version enforcement: NP upgrade to version > CP rejected.

**E2E tests:** see `testplan-phase1.md` for detailed test cases. Key scenarios: CP upgrade on a live cluster, per-NP upgrade, version skew rejection, concurrent-upgrade rejection, and terminal upgrade failure with unchanged provisioning state.

## Graduation Criteria

- Stable CP and per-NP upgrades via the OSAC API, CLI, and UI.
- Correct upgrade state and history surfaced in `Cluster.status`.
- N-3 skew and downgrade rejection validated by integration tests.
- RBAC expansion for osac-operator documented and audited.
- DB backfill migration applied and verified on pre-existing clusters.

## Upgrade / Downgrade Strategy

A DB migration backfills `observed_cp_version` for pre-existing clusters (see SQL in API Extensions). The new `ClusterStatus` fields are additive and backward-compatible. The `ReleaseImage` hash exclusion is a behavioral change to the provisioning path; it must be validated on rollout to prevent spurious AAP re-provision jobs.

## Version Skew Strategy

The fulfillment-service, osac-operator, and osac-ui are affected. All three ship in the same coordinated deployment. The osac-ui generates types from the same protos; new status fields degrade gracefully (no display) on older UI builds.

## Support Procedures

### Detection

- Upgrade state is visible in `Cluster.status.upgrade` and `ClusterOrder.status.upgradeStatus`.
- Stuck upgrades (Progressing for > expected duration) surface via Kubernetes Events on `ClusterOrder`.
- Operator logs structured entries with cluster ID, component, and version on every state transition.

### Recovery

- A terminal failure releases `CanUpgrade`; investigate the upgrade-specific reason before starting another upgrade.
- If the operator is stuck (persistent HC/NP API errors), resolve the underlying hub-cluster API issue; the operator resumes automatically on reconnection.

## Infrastructure Needed

- Hub cluster RBAC: `patch`/`update` on `hostedclusters` and `nodepools` for the osac-operator service account.
- No new external services or infrastructure.

---

## Provenance

Authored: draft @ design 0.11.1 - f1d6a4b, workspace main @ b14c881c5 (dirty)
Final: revise @ design 0.11.3 - 2bd6607, workspace main @ 9c26507ef

> Context changed between draft and revise.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.3","ai_workflows":"2bd6607","source_repo":"9c26507ef","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["draft","draft","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
