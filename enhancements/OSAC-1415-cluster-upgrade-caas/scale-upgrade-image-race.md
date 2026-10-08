# How a scale job can reapply an old release image

This note explains the potential scale/upgrade ordering problem in [OSAC-1415 Phase 1](design-phase1.md). **It describes an unsafe interleaving to prevent, not the intended result of the selected AAP design.** That design requires the operator to wait for an active create/scale job before launching an upgrade job, wait for an active upgrade before launching create/scale work, and re-evaluate the latest `ClusterOrder` between jobs.

The upgrade flow described here is proposed. At the local `osac` revision `515ce87588967055537a8fd8e7edb0e2d728eb85`, `NodeRequest.ReleaseImage` and the AAP upgrade-only branch have not been implemented. Examples below use the proposed per-node-pool image field; the same ordering applies to `ClusterOrderSpec.ReleaseImage` and a HostedCluster control-plane upgrade.

## The two copies of desired state

An accepted scale or upgrade request first changes the Fulfillment `Cluster` database record. Fulfillment then **asynchronously** builds and patches the live Kubernetes `ClusterOrder` spec. For a node-pool upgrade, the proposed projection resolves `Cluster.spec.node_sets[<name>].version` to `ClusterOrder.spec.nodeRequests[<name>].ReleaseImage`. The operator sees that desired image and starts the selected HyperShift image patch. Thus **yes: the `ClusterOrder` is updated before the upgrade patches the HyperShift resource**. Today Fulfillment already creates or merge-patches the whole `ClusterOrder` spec from the Cluster record (`fulfillment-service/internal/controllers/cluster/cluster_reconciler_function.go:235-304`).

An AAP job has a **different copy**. When the operator launches a job, the AAP provider serializes the `ClusterOrder` into `osac_job_vars.resource` and sends it as launch variables (`osac-operator/pkg/provisioning/aap_provider.go:261-289,392-432`). The playbook reads `cluster_order` from those variables (`osac-aap/playbook_osac_create_hosted_cluster.yml:6-7`). A later patch to the live `ClusterOrder` does **not** change variables already attached to that job. The job may be queued, running pre-tasks, or waiting to acquire the per-cluster lease while its copy remains old.

Think of the live order and a launched job as two documents:

| Object | Who updates it after an upgrade is accepted? | Can an earlier scale job see that update automatically? |
|---|---|---|
| Live `ClusterOrder.spec` | Fulfillment projects the new desired image. | No. It is a separate Kubernetes object from the job payload. |
| Scale job's `osac_job_vars.resource.spec` | Fixed when that job is launched. | No. The job uses its launch-time copy unless a new read is explicitly added. |
| Live `NodePool.spec` or `HostedCluster.spec` | AAP tasks and the existing NodePool replica controller can patch it. | This is the object on which write ordering matters. |

## Why the scale job can write an image

The scale command changes node-set size, and the changed `ClusterOrder` spec triggers the provisioning lifecycle through `DesiredConfigVersion` (`fulfillment-service/internal/cmd/cli/scale/scale_cmd.go:155-165`, `osac-operator/internal/controller/clusterorder_controller.go:513-519,1227-1232`). That AAP run uses the selected template's install path. Its current HostedCluster definition includes `spec.release.image`, and its NodePool definition includes both `spec.replicas` and `spec.release.image` (`osac-aap/collections/ansible_collections/osac/service/roles/hosted_cluster/tasks/create_hosted_cluster.yaml:29-40,101-104,146-150`, `build_nodepools.yaml:14-31`).

The AAP task uses `kubernetes.core.k8s` with `state: present`. For an existing resource the vendored module uses a merge patch, not a full-spec replacement; **fields present in its submitted definition are still patch inputs** (`osac-aap/vendor/ansible_collections/kubernetes/core/plugins/module_utils/k8s/service.py:421-445`). Consequently, a scale job's NodePool update can include an image even though the tenant only asked to change the replica count. The operator's separate `patchNodePoolReplicas` action really is scoped to `spec.replicas`; it is not the whole scale flow (`osac-operator/internal/controller/baremetalworker/nodepool.go:69-93`).

Today AAP resolves `ocp_release_image` from its copied `cluster_order.spec.releaseImage` when that field is set, with template and provider fallbacks, then uses that image for the HostedCluster and every NodePool (`osac-aap/collections/ansible_collections/osac/service/roles/cluster_settings/tasks/main.yml:23-32`). Phase 1 proposes rendering each NodePool from its own `NodeRequest.ReleaseImage` while continuing to use the order's control-plane image for the HostedCluster. That change is necessary to preserve an independently upgraded NodePool on **later** scale jobs. It cannot update the payload of a scale job that was already launched with the older image.

## An unsafe ordering if the jobs are allowed to overlap

Let image **A** be the current release, image **B** the accepted upgrade target, and `workers` grow from three to four nodes. `S` is a scale AAP job launched before the upgrade; `U` is the proposed image-only upgrade AAP job. The following is a possible ordering **without the design's cross-job wait**:

| Time | Event | Live `ClusterOrder` | Frozen payload in `S` | Live NodePool |
|---|---|---|---|---|
| 0 | Before either request | 3 replicas, image A | No job | 3 replicas, image A |
| 1 | Scale is accepted and projected; operator launches `S` | 4 replicas, image A | **4 replicas, image A** | 3 replicas, image A |
| 2 | Upgrade is accepted and projected | **4 replicas, image B** | **4 replicas, image A** | 3 replicas, image A |
| 3 | `U` patches only `spec.release.image` | 4 replicas, image B | 4 replicas, image A | 3 replicas, **image B** |
| 4 | Delayed `S` submits its NodePool definition | 4 replicas, image B | 4 replicas, image A | **4 replicas, image A** |

At time 4, the live `ClusterOrder` still says **B**. The older image comes from `S`'s frozen job variables, not from Fulfillment undoing the upgrade. Since the scale definition supplies `release.image`, its later merge patch can set that field back to **A**. HyperShift may react even if a later reconcile eventually restores **B**.

The existing lease does not rule out this order by itself. For example, `S` can be submitted first but remain queued or delayed before acquiring the lease. Without the cross-job wait, `U` can acquire the lease, patch B, and release it; `S` can then acquire the lease and apply its saved definition with A. The lease prevents simultaneous mutation but does not require jobs to run in submission order or replace their saved inputs.

If `S` applies its definition **before** `U` patches the image, the final state is four replicas at **B**: the image-only upgrade patch leaves `replicas` alone. Thus the problem depends on the **order of HyperShift resource writes**, not on the order in which tenants submit requests or Fulfillment patches `ClusterOrder`.

The same example applies to a control-plane upgrade: the create/scale AAP path reapplies a HostedCluster definition containing `spec.release.image`, so a delayed scale job could write an older HostedCluster image. A pure `spec.replicas` patch from the worker controller cannot revert either image because it does not include that field.

## How the selected AAP design prevents this ordering

The Phase 1 design requires coordination **before launching** the second AAP job:

1. If `S` is active when the upgrade image reaches `ClusterOrder`, the operator does not launch `U`. It waits for `S` to finish. `S` may still apply image A, but it does so **before** `U` applies B.
2. Once `S` is terminal, the operator reads the latest `ClusterOrder`, sees B, and launches `U` to patch B. Final desired image: B.
3. In the reverse order, if `U` is active when a scale change arrives, the operator defers the scale AAP job. After `U` finishes, it re-reads the latest `ClusterOrder` and launches the scale job with image B. Final replica count and image: four and B.

This is a **design requirement**, not behavior supplied automatically by the existing provisioning lifecycle. That lifecycle already waits for its own active provision job, but an upgrade job tracked separately needs an explicit check across job types (`osac-operator/pkg/provisioning/provision_lifecycle.go:59-82`). The check must cover a submitted or queued job as well as one executing AAP tasks, and the decision to dispatch a new job must not race with another dispatch. The operator must persist and recover job identity so a restart does not accidentally dispatch both paths. AAP's per-cluster lease serializes jobs that acquire it, but is acquired **after** each job's snapshot is taken. The lease alone neither refreshes stale variables nor guarantees that an earlier-submitted job writes first (`osac-aap/playbook_osac_create_hosted_cluster.yml:21-49`).

The design also requires create/scale AAP rendering to use the **latest projected control-plane and per-node-pool images** when a *new* job is launched. Waiting between jobs without that rendering change would still allow a later scale to reset an independently upgraded image.

## What remains a separate decision

To prevent this image reversal, the operator must order the **AAP writes** and launch the next job with current desired state. The current design says create/scale waits for an “active upgrade,” but does not define whether the wait ends when the AAP image patch job finishes or when HyperShift finishes upgrading. The latter also prevents operational overlap but can delay scaling for a long time, including during a nonterminal `TimedOut` upgrade. Allowing scaling after the AAP patch while HyperShift is still upgrading requires deployed validation of how new bare-metal nodes behave; it does not recreate the stale-image race if the new scale job uses image B.

There is also an earlier API-level concern: the current scale CLI updates the entire `Cluster.spec.node_sets` map, so after node sets gain a version field, a stale API request could carry an old version. That is distinct from the frozen AAP job payload illustrated here. See [upgrade execution alternatives](upgrade-execution-alternatives.md#races-and-boundaries-to-preserve) for this issue and the direct-patching alternative.
