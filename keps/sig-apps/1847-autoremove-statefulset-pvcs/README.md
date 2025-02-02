# KEP-1847: Auto delete PVCs created by StatefulSet

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Background](#background)
  - [Changes required](#changes-required)
  - [User Stories](#user-stories)
    - [Story 0](#story-0)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
    - [Story 3](#story-3)
  - [Notes/Constraints/Caveats (optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Objects Associated with the StatefulSet](#objects-associated-with-the-statefulset)
  - [Volume delete policy for the StatefulSet created PVCs](#volume-delete-policy-for-the-statefulset-created-pvcs)
    - [<code>whenScaled</code> policy of <code>Delete</code>.](#whenscaled-policy-of-delete)
    - [<code>whenDeleted</code> policy of <code>Delete</code>.](#whendeleted-policy-of-delete)
    - [Non-Cascading Deletion](#non-cascading-deletion)
    - [Mutating <code>PersistentVolumeClaimRetentionPolicy</code>](#mutating-persistentvolumeclaimretentionpolicy)
  - [Cluster role change for statefulset controller](#cluster-role-change-for-statefulset-controller)
  - [Test Plan](#test-plan)
    - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [E2E tests](#e2e-tests)
      - [Upgrade/downgrade &amp; feature enabled/disable tests](#upgradedowngrade--feature-enableddisable-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha release](#alpha-release)
    - [Beta release](#beta-release)
    - [GA release](#ga-release)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
      - [How can this feature be enabled / disabled in a live cluster?](#how-can-this-feature-be-enabled--disabled-in-a-live-cluster)
      - [Does enabling the feature change any default behavior?](#does-enabling-the-feature-change-any-default-behavior)
      - [Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?](#can-the-feature-be-disabled-once-it-has-been-enabled-ie-can-we-roll-back-the-enablement)
      - [What happens if we reenable the feature if it was previously rolled back?](#what-happens-if-we-reenable-the-feature-if-it-was-previously-rolled-back)
      - [Are there any tests for feature enablement/disablement?](#are-there-any-tests-for-feature-enablementdisablement)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
      - [How can a rollout fail? Can it impact already running workloads?](#how-can-a-rollout-fail-can-it-impact-already-running-workloads)
      - [What specific metrics should inform a rollback?](#what-specific-metrics-should-inform-a-rollback)
      - [Were upgrade and rollback tested? Was the upgrade-&gt;downgrade-&gt;upgrade path tested?](#were-upgrade-and-rollback-tested-was-the-upgrade-downgrade-upgrade-path-tested)
      - [Is the rollout accompanied by any deprecations and/or removals of features, APIs,](#is-the-rollout-accompanied-by-any-deprecations-andor-removals-of-features-apis)
  - [Monitoring Requirements](#monitoring-requirements)
      - [How can an operator determine if the feature is in use by workloads?](#how-can-an-operator-determine-if-the-feature-is-in-use-by-workloads)
      - [What are the SLIs (Service Level Indicators) an operator can use to determine](#what-are-the-slis-service-level-indicators-an-operator-can-use-to-determine)
      - [What are the reasonable SLOs (Service Level Objectives) for the above SLIs?](#what-are-the-reasonable-slos-service-level-objectives-for-the-above-slis)
      - [Are there any missing metrics that would be useful to have to improve observability](#are-there-any-missing-metrics-that-would-be-useful-to-have-to-improve-observability)
  - [Dependencies](#dependencies)
      - [Does this feature depend on any specific services running in the cluster?](#does-this-feature-depend-on-any-specific-services-running-in-the-cluster)
  - [Scalability](#scalability)
      - [Will enabling / using this feature result in any new API calls?](#will-enabling--using-this-feature-result-in-any-new-api-calls)
      - [Will enabling / using this feature result in introducing new API types?](#will-enabling--using-this-feature-result-in-introducing-new-api-types)
      - [Will enabling / using this feature result in any new calls to the cloud provider?](#will-enabling--using-this-feature-result-in-any-new-calls-to-the-cloud-provider)
      - [Will enabling / using this feature result in increasing size or count of the existing API objects?](#will-enabling--using-this-feature-result-in-increasing-size-or-count-of-the-existing-api-objects)
      - [Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?](#will-enabling--using-this-feature-result-in-increasing-time-taken-by-any-operations-covered-by-existing-slisslos)
      - [Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?](#will-enabling--using-this-feature-result-in-non-negligible-increase-of-resource-usage-cpu-ram-disk-io--in-any-components)
  - [Troubleshooting](#troubleshooting)
      - [How does this feature react if the API server and/or etcd is unavailable?](#how-does-this-feature-react-if-the-api-server-andor-etcd-is-unavailable)
      - [What are other known failure modes?](#what-are-other-known-failure-modes)
      - [What steps should be taken if SLOs are not being met to determine the problem?](#what-steps-should-be-taken-if-slos-are-not-being-met-to-determine-the-problem)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [X] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [X] (R) KEP approvers have approved the KEP status as `implementable`
- [X] (R) Design details are appropriately documented
- [X] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [X] e2e Tests for all Beta API Operations (endpoints)
  - [X] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [X] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [X] (R) Graduation criteria is in place
  - [X] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
- [X] (R) Production readiness review completed
- [X] (R) Production readiness review approved
- [X] "Implementation History" section is up-to-date for milestone
- [X] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [X] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary
The proposal is to add a feature to autodelete the PVCs created by StatefulSet.

## Motivation

Currently, the PVCs created automatically by the StatefulSet are not deleted when 
the StatefulSet is deleted. As can be seen by the discussion in the issue 
[55045](https://github.com/kubernetes/kubernetes/issues/55045) there are several use
cases where the PVCs which are automatically created are deleted as well. In many 
StatefulSet use cases, PVCs have a different lifecycle than the pods of the 
StatefulSet, and should not be deleted at the same time. Because of this, PVC 
deletion will be opt-in for users.

### Goals

Provide a feature to auto delete the PVCs created by StatefulSet when the volumes are no
longer in use to ease management of StatefulSets that don't live indefinitely. As
application state should survive over StatefulSet maintenance, the feature ensures that
the pod restarts due to non scale down events such as rolling update or node drain do not
delete the PVC.

### Non-Goals

This proposal does not plan to address how the underlying PVs are treated on PVC deletion. 
That functionality will continue to be governed by the reclaim policy of the storage class. 

## Proposal

### Background

The `garbagecollector` controller is responsible for ensuring that when a StatefulSet
is deleted, the corresponding pods spawned from the StatefulSet are deleted as well.  The
`garbagecollector` uses an `OwnerReference` added to the `Pod` by the StatefulSet
controller to delete the Pod. This proposal leverages a similar mechanism to automatically
delete the PVCs created by the controller from the StatefulSet's VolumeClaimTemplate.

### Changes required

The following changes are required:

1. Add `persistentVolumeClaimRetentionPolicy` to the StatefulSet spec with the following fields.
   * `whenDeleted` - specifies if the VolumeClaimTemplate PVCs are deleted when
     their StatefulSet is deleted.
   * `whenScaled` - specifies if VolumeClaimTemplate PVCs are deleted when
     their corresponding pod is deleted on a StatefulSet scale-down, that is,
     when the number of pods in a StatefulSet is reduced via the Replicas field.

   These fields may be set to the following values.
   * `Retain` - the default policy, which is also used when no policy is
      specified. This specifies the existing behavior: when a StatefulSet is
      deleted or scaled down, no action is taken with respect to the PVCs
      created by the StatefulSet.
   * `Delete` - specifies that the appropriate PVCs as described above will be
      deleted in the corresponding scenario, either on StatefulSet deletion or scale-down.
2. Add `patch` to the statefulset controller rbac cluster role for `persistentvolumeclaims`.

### User Stories

#### Story 0
The user is happy with legacy behavior of a stateful set. They leave all fields
of `PersistentVolumeClaimRetentionPolicy` to `Retain`. Nothing traditional
StatefulSet behavior changes neither on set deletion nor on scale-down.

#### Story 1
The user is running a StatefulSet as part of an application with a finite lifetime. During
the application's existence the StatefulSet maintains per-pod state, even across scale-up
and scale-down. In order to maximize performance, volumes are retained during scale-down
so that scale-up can leverage the existing volumes. When the application is finished, the
volumes created by the StatefulSet are no longer needed and can be automatically
reclaimed.

The user would set `persistentVolumeClaimRetentionPolicy.whenDeleted` to `Delete`, which
would ensure that the PVCs created automatically during the StatefulSet
activation is deleted once the StatefulSet is deleted.

#### Story 2
The user is cost conscious, and can sustain slower scale-up speeds even after a
scale-down, because scaling events are rare, and volume data can be
reconstructed, albeit slowly, during a scale up. However, it is necessary to
bring down the StatefulSet temporarily by deleting it, and then bring it back up
by reusing the volumes. This is accomplished by setting
`persistentVolumeClaimRetentionPolicy.whenScaled` to `Delete`, and leaving
`persistentVolumeClaimRetentionPolicy.whenDeleted` at `Retain`.

#### Story 3
User is very cost conscious, and can sustain slower scale-up speeds even after a
scale-down. The user does not want to pay for volumes that are not in use in any
circumstance, and so wants them to be reclaimed as soon as possible. On scale-up
a new volume will be provisioned and the new pod will have to
re-intitialize. However, for short-lived interruptions when a pod is killed &
recreated, like a rolling update or node disruptions, the data on volumes is
persisted. This is a key property that ephemeral storage, like emptyDir, cannot
provide.

User would set the `persistentVolumeClaimRetentionPolicy.whenScaled` as well as
`persistentVolumeClaimRetentionPolicy.whenDeleted` to `Delete`, ensuring PVCs are
deleted when corresponding Pods are deleted. New Pods created during scale-up
followed by a scale-down will wait for freshly created PVCs. PVCs are deleted as
well when the set is deleted, reclaiming volumes as quickly as possible and
minimizing expense.

### Notes/Constraints/Caveats (optional)

This feature applies to PVCs which are defined by the volumeClaimTemplate of a
StatefulSet. Any PVC and PV provisioned from this mechanism will function with
this feature. These PVCs are identified by the static naming scheme used by
StatefulSets. Auto-provisioned and pre-provisioned PVCs will be treated
identically, so that if a user pre-provisions a PVC matching those of a
VolumeClaimTemplate it will be deleted according to the deletion policy.

### Risks and Mitigations

Currently the PVCs created by StatefulSet are not deleted automatically. Using
`whenScaled` or `whenDeleted` set to `Delete` would delete the PVCs
automatically. Since this involves persistent data being deleted, users should
take appropriate care using this feature. Having the `Retain` behavior as
default will ensure that the PVCs remain intact by default and only a conscious
choice made by user will involve any persistent data being deleted.

This proposed API causes the PVCs associated with the StatefulSet to have
behavior close to, but not the same as, ephemeral volumes, such as emptyDir or
generic ephemeral volumes. This may cause user confusion. PVCs under this policy
will more durable than ephemeral volumes would be, as they are only deleted on
scale-down or StatefulSet deletion, and not on other pod deletion and recreation
events eviction or the death of their node.

User documentation will emphasize the race conditions associated with changing
policy or rolling back the feature concurrently with StatefulSet deletion or
scale-down. See below in [Design Detils](#design-details) for more information.

## Design Details

### Objects Associated with the StatefulSet

When a StatefulSet spec has a `VolumeClaimTemplate`, PVCs are dynamically created using a
static naming scheme, and each Pod is created with a claim to the corresponding PVC. These
are the precise PVCs meant when referring to the volume or PVC for Pod below, and these
are the only PVCs modified with an ownerRef. Other PVCs referenced by the StatefulSet Pod
template are not affected by this behavior.

OwnerReferences are used to manage PVC deletion. All such references used for
this feature will set the controller field to the StatefulSet or Pod as
appropriate. This will be used to distinguish references added by the controller
from, for example, user-created owner references. When ownerRefs is removed, it
is understood that only those ownerRefs whose controller field matches the
StatefulSet or Pod in question are affected.

The controller flag will be set for these references. If there is already a
different (non-StatefulSet) controller set for a PVC, an ownerRef will not be
added. This will mean that the autodelete functionality will not be operative. An
event will be created to reflect this.

To summarize,

**If the StatefulSet is a controller owner**,
  * the PVC lifecycle will be full managed by the StatefulSet controller
  * old owner references will be updated with `controller=false` to `controller=true`
    (see Upgrade / Downgrade Strategy, below).
  * remove itself as the owner and controller when the retain policy is specified in the StatefulSet.

**If someone else is the controller**, 
  * the PVC lifecycle will not be touched by the StatefulSet controller. The PVC
    will stay when the delete policy is specified in the StatefulSet.
  * old StatefulSet owner references will be removed.

### Volume delete policy for the StatefulSet created PVCs

A new field named `PersistentVolumeClaimRetentionPolicy` of the type
`StatefulSetPersistentVolumeClaimRetentionPolicy` will be added to the StatefulSet. This
will represent the user indication for which circumstances the associated PVCs
can be automatically deleted or not, as described above. The default policy
would be to retain PVCs in all cases.

The `PersistentVolumeClaimRetentionPolicy` object will be mutable. The deletion
mechanism will be based on reconciliation, so as long as the field is changed
far from StatefulSet deletion or scale-down, the policy will work as
expected. Mutability does introduce race conditions if it is changed while a
StatefulSet is being deleted or scaled down and may result in PVCs not being
deleted as expected when the policy is being changed from `Retain`, and PVCs
being deleted unexpectedly when the policy is being changed to `Retain`. PVCs
will be reconciled before a scale-down or deletion to reduce this race as much
as possible, although it will still occur. The former case can be mitigated by
manually deleting PVCs. The latter case will result in lost data, but only in
PVCs that were originally declared to have been deleted. Life does not always
have an undo button.

#### `whenScaled` policy of `Delete`.

If `persistentVolumeClaimRetentionPolicy.whenScaled` is set to `Delete`, the Pod will be
set as the owner of the PVCs created from the `VolumeClaimTemplates` just before
the scale-down is performed by the StatefulSet controller.  When a Pod is
deleted, the PVC owned by the Pod is also deleted.

The current StatefulSet controller implementation ensures that the manually deleted pods
are restored before the scale-down logic is run. This combined with the fact that the
owner references are set only before the scale-down will ensure that manual deletions do
not automatically delete the PVCs in question.

During scale-up, if a PVC has an OwnerRef that does not match the Pod, it indicates that
the PVC was referred to by the deleted Pod and is in the process of getting
deleted. The controller will skip the reconcile loop until PVC deletion finishes, avoiding
a race condition.

#### `whenDeleted` policy of `Delete`.

When `persistentVolumeClaimRetentionPolicy.whenDeleted` is set to `Delete`, when a
VolumeClaimTemplate PVC is created, an owner reference in PVC will be added to
point to the StatefulSet. When a scale-up or scale-down occurs, the PVC is
unchanged.  PVCs previously in use before scale-down will be used again when the
scale-up occurs.

In the existing StatefulSet reconcile loop, the associated VolumeClaimTemplate
PVCs will be checked to see if the ownerRef is correct according to the
`persistentVolumeClaimRetentionPolicy` and updated accordingly. This includes PVCs
that have been manually provisioned. It will be most consistent and easy
to reason about if all VolumeClaimTemplate PVCs are treated uniformly rather
than trying to guess at their provenance.

When the StatefulSet is deleted, these PVCs will also be deleted, but only after
the Pod gets deleted. Since the Pod StatefulSet ownership has
`blockOwnerDeletion` set to `true`, pods will get deleted before the StatefulSet
is deleted. The `blockOwnerDeletion` for PVCs will be set to `false` which
ensures that PVC deletion happens only after the StatefulSet is deleted. This is
necessary because of PVC protection which does not allow PVC deletion until all
pods referencing it are deleted.

The deletion policies may be combined in order to get the delete behavior both
on set deletion as well as scale-down.

#### Non-Cascading Deletion

When StatefulSet is deleted without cascading, eg `kubectl delete --cascade=false`, then
existing behavior is retained and no PVC will be deleted. Only the StatefulSet resource
will be affected.

#### Mutating `PersistentVolumeClaimRetentionPolicy`

Recall that as defined above, the PVCs associated with a StatefulSet are found
by the StatefulSet volumeClaimTemplate static naming scheme. The Pods associated
with the StatefulSet can be found by their controllerRef.

* **From a deletion policy to `Retain`**

When mutating any delete policy to retain, the PVC ownerRefs to the 
StatefulSet are removed. If a scale-down is in progress, each remaining PVC
ownerRef to its pod is removed, by matching the index of the PVC to the Pod
index.

* **From `Retain` to a deletion policy**

When mutating from the `Retain` policy to a deletion policy, the StatefulSet
PVCs are updated with an ownerRef to the StatefulSet. If a scale-down is in
process, remaining PVCs are given an ownerRef to their Pod (by index, as above).

### Cluster role change for statefulset controller

In order to update the PVC ownerReference, the `buildControllerRoles` will be updated with 
`patch` on PVC resource.

### Test Plan

[X] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

#### Unit tests

From https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit&include-filter-by-regex=statefulset

- `k8s.io/kubernetes/pkg/controller/statefulset`: `2024-10-07`: `86.5%`
- `k8s.io/kubernetes/pkg/registry/apps/statefulset`: `2022-10-07`: `62.7%`
- `k8s.io/kubernetes/pkg/registry/apps/statefulset/storage`: `2022-10-07`: `64%`
  
##### Integration tests

- `test/integration/statefulset`: `2024-10-07`: [No failures](https://storage.googleapis.com/k8s-triage/index.html?job=ci-kubernetes-integration&test=TestAutodeleteOwnerRefs)

Added `TestAutodeleteOwnerRefs` to `k8s.io/kubernetes/test/integration/statefulset`.

##### E2E tests

- `[gci-gce-statefulset](https://testgrid.k8s.io/google-gce#gci-gce-statefulset)`: `2024-10-07`: `0 Failures`
- [triage](https://storage.googleapis.com/k8s-triage/index.html?test=.*StatefulSetPersistentVolumeClaimPolicy.*): `2024-10-09`:
  - Flakey failures in `ci-kubernetes-kind-e2e-parallel`, `ci-kubernetes-kind-e2e-parallel-1-30` and
    `ci-kubernetes-kind-ipv6-e2e-parallel-1-31` also had failures in many other tests, so appears to
    be general infrastructure flake.
  - ` [sig-apps] StatefulSet Non-retain StatefulSetPersistentVolumeClaimPolicy should delete PVCs
    after adopting pod (WhenScaled)` seems to have real flakes which will be investigated.

Added `Feature:StatefulSetAutoDeletePVC` tests to `k8s.io/kubernetes/test/e2e/apps/`.

##### Upgrade/downgrade & feature enabled/disable tests

The following scenarios were manuall tested.

    1. Create statefulset in previous version and upgrade to the version 
       supporting this feature. The PVCs should remain intact.
    2. Downgrade to earlier version and check the PVCs with Retain
       remain intact and the others with set policies before upgrade 
       gets deleted based on if the references were already set.

Since `rancher.io/local-path` now provides a default storage class, StatefulSets
can be tested with kind with the following procedure.

* Create a [kind](https://kind.sigs.k8s.io/) cluster with the following `config.yaml`.
  ```
  apiVersion: kind.x-k8s.io/v1alpha4
  kind: Cluster
  featureGates:
    StatefulSetAutoDeletePVC: false
  nodes:
  - role: control-plane
    image: kindest/node:v1.31.0
  ```
  This is done with `kind create cluster --config config.yaml`
* The configuration adds the feature gate to all control plane services. In a
  kind cluster, these are stored in the `/etc/kubernetes/manifests` directory of
  the kind docker container serving as the control plane node. The manifests are
  reconciled to the control plane, so the cluster can be upgraded or downgraded
  from the StatefulSet retention policy feature with bash script like the
  following.
  ```
  for c in kube-apiserver kube-controller-manager kube-scheduler; do
    docker exec kind-control-plane \
      sed -i -r "s|(StatefulSetAutoDeletePVC)=false|\1=true|" \
      /etc/kubernetes/manifests/$c.yaml
    echo $c updated
  done
  ```
  To downgrade, swap false for true in the above. Note that the kind control
  plane will be unreachable for a minute or so while the reconciliation occurs.
  
For the upgrade scenario, a StatefulSet was created in a cluster with the
feature gate disabled. The feature gate was enabled, the StatefulSet was scaled
down or deleted, and it was confirmed that no PVCs were deleted.

In the downgrade scenario, four StatefulSets were created with all possibilities
of WhenScaled and WhenDeleted policies. After downgraded, it was confirmed that
(1) no PVCs are deleted when the StatefulSet is scaled down, and (2) PVCs are
deleted when the WhenDeleted policy is Delete, and the StatefulSet is deleted.

### Graduation Criteria

#### Alpha release
- (Done) Complete adding the items in the 'Changes required' section.
- (Done) Add unit, functional, upgrade and downgrade tests to automated k8s test.

#### Beta release
- (Done) Enable feature gate for e2e pipelines

#### GA release
- (Done) Validate with customer workloads. There has been no customer feedback
  aside from some unrelated issues on GKE which showed that customers were using
  delete strategies, and analysis of owner references on PVCs that motivated
  [#122400](https://github.com/kubernetes/kubernetes/issues/122400).


### Upgrade / Downgrade Strategy

This features adds a new field to the StatefulSet. The default value for the new field
maintains the existing behavior of StatefulSets.

On a downgrade, the `PersistentVolumeClaimRetentionPolicy` field will be hidden on
any StatefulSets. The behavior in this case will be identical to mutating the
policy field to `Retain`, as described above, including the edge cases
introduced if this is done during a scale-down or StatefulSet deletion.

The initial beta version did not set the controller flag in the owner
reference. This was fixed in later versions, so that the controller flag is
set. The behavior is then as follows.

* If the StatefulSet or one of its Pods is a controller, owner references are
  updated as specified by the retention policy.
* If the StatefulSet or one of its Pods is an owner but not a controller,
  * if there is no other controller, the StatefulSet or Pod owners will be set
    as controller (ie, they are assumed to be from the initial beta version and
    updated).
  * The controller will be updated as specified by the retention policy.
* If there is another resource that is a controller,
  * any (non-controler) StatefulSet or Pod owner reference is removed.
  * The retention policy is ignored.

### Version Skew Strategy
There are only apiserver and kube-controller-manager changes involved. Node
components are not involved so there is no version skew between nodes and the
control plane. Since the api changes are backwards compatible, as long as the
apiserver version which originally added the new StatefulSet fields is rolled
out before the kube-controller-manager, behavior will be correct. Since the alpha
API has been out since 1.23 and there have been no incompatible changes to the
API, the order of any modern apiserver & kube-controller-manager rollout should
not matter anyway.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

##### How can this feature be enabled / disabled in a live cluster?
  - [x] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name: StatefulSetAutoDeletePVC
    - Components depending on the feature gate
      - kube-controller-manager, which orchestrates the volume deletion.
      - kube-apiserver, to manage the new policy field in the StatefulSet
        resource (eg dropDisabledFields).
  
##### Does enabling the feature change any default behavior?
  No. What happens during StatefulSet deletion differs from current behavior
  only when the user explicitly specifies the
  `PersistentVolumeClaimDeletePolicy`.  Hence no change in any user visible
  behavior change by default.

##### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?
  Yes. Disabling the feature gate will cause the new field to be ignored. If the feature
  gate is re-enabled, the new behavior will start working.
  
  When the `PersistentVolumeClaimRetentionPolicy` has `WhenDeleted` set to
 `Delete`, then VolumeClaimTemplate PVCs ownerRefs must be removed.

  There are new corner cases here. For example, if a StatefulSet deletion is in
  process when the feature is disabled or enabled, the appropriate ownerRefs
  will not have been added and PVCs may not be deleted. The exact behavior will
  be discovered during feature testing. In any case the mitigation will be to
  manually delete any PVCs.
  
##### What happens if we reenable the feature if it was previously rolled back?  
  In the simple case of reenabling the feature without concurrent StatefulSet
  deletion or scale-down, nothing needs to be done when the deletion policy has
  `whenScaled` set to `Delete`. When the policy has `whenDeleted` set to `Delete`, the
  VolumeClaimTemplate PVC ownerRefs must be set to the StatefulSet.
  
  As above, if there is a concurrent scale-down or StatefulSet deletion, more
  care needs to be taken. This will be detailed further during feature testing.

##### Are there any tests for feature enablement/disablement?
  Feature enablement and disablement tests will be added, including for
  StatefulSet behavior during transitions in conjunction with scale-down or
  deletion.

### Rollout, Upgrade and Rollback Planning

##### How can a rollout fail? Can it impact already running workloads?
  If there is a control plane update which disables the feature while a stateful
  set is in the process of being deleted or scaled down, it is undefined which
  PVCs will be deleted. Before the update, PVCs will be marked for deletion;
  until the updated controller has a chance to reconcile some PVCs may be
  garbage collected before the controller has a chance to remove any owner
  references. We do not think this is a true failure, as it should be clear to
  an operator that there is an essential race condition when a cluster update
  happens during a stateful set scale down or delete.

##### What specific metrics should inform a rollback?
  The operator can monitor `kube_persistent_volume_*` metrics from
  kube-state-metrics to watch for large numbers of undeleted
  PersistentVolumes. If consistent behavior is required, the operator can wait
  for those metrics to stablize.

##### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?
  Yes. The race condition wasn't exposed, but we confirmed the PVCs were updated correctly.

##### Is the rollout accompanied by any deprecations and/or removals of features, APIs, 
fields of API types, flags, etc.?
  No

### Monitoring Requirements

Metrics are provided by `kube-state-metrics` unless otherwise noted.

##### How can an operator determine if the feature is in use by workloads?
  `kube_statefulset_persistent_volume_claim_retention_policy` will have nonzero
  counts for the `delete` policy fields.

##### What are the SLIs (Service Level Indicators) an operator can use to determine 
the health of the service?
  - Metric name: `kube_statefulset_status_replicas_current` should be near
    `kube_statefulset_stats_replicas_ready`.
    - [Optional] Aggregation method: `gauge`
    - Components exposing the metric: `kube-state-metrics`

##### What are the reasonable SLOs (Service Level Objectives) for the above SLIs?

  `kube_statefulset_stats_replicas_ready /
  kube_statefulset_stats_replicas_current` should be near 1.0, although as
  unhealthy replicas are often an application error rather than a problem with
  the stateful set controller, this will need to be tuned by an operator on a
  per-cluster basis.

##### Are there any missing metrics that would be useful to have to improve observability 
of this feature?

  kube-state-metrics have filled a gap in the traditional lack of metrics from
  core Kubernetes controllers.

### Dependencies

##### Does this feature depend on any specific services running in the cluster?

  No, outside of depending on the scheduler, the garbage collector and volume
  management (provisioning, attaching, etc) as does almost anything in
  Kubernetes. This feature does not add any new dependencies that did not
  already exist with the stateful set controller.


### Scalability

##### Will enabling / using this feature result in any new API calls?

  Yes and no. This feature will result in additional resource deletion calls, which will
  scale like the number of pods in the stateful set (ie, one PVC per pod and possibly one
  PV per PVC depending on the reclaim policy). There will not be additional watches,
  because the existing pod watches will be used. There will be additional
  patches to set PVC ownerRefs, scaling like the number of pods in the StatefulSet.

  However, anyone who uses this feature would have made those resource deletions
  anyway: those PVs cost money. Aside from the additional patches for onwerRefs,
  there shouldn't be much overall increase beyond the second-order effect of
  this feature allowing more automation.

##### Will enabling / using this feature result in introducing new API types?
  No.

##### Will enabling / using this feature result in any new calls to the cloud provider?
  PVC deletion may cause PV deletion, depending on reclaim policy, which will result in
  cloud provider calls through the volume API. However, as noted above, these calls would
  have been happening anyway, manually.

##### Will enabling / using this feature result in increasing size or count of the existing API objects?
  - PVC, new ownerRef; ~64 bytes
  - StatefulSet, new field; ~8 bytes (holds string enumeration either "Delete" or "Retain"

##### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?
  No. (There are currently no StatefulSet SLOs?)
  
  Note that scale-up may be slower when volumes were deleted by scale-down. This
  is by design of the feature.

##### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?
  No.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?
  No.

### Troubleshooting

##### How does this feature react if the API server and/or etcd is unavailable?

PVC deletion will be paused. If the control plane went unavailable in the middle
of a stateful set being deleted or scaled down, there may be deleted Pods whose
PVCs have not yet been deleted. Deletion will continue normally after the
control plane returns.

##### What are other known failure modes?
  - PVCs from a stateful set not being deleted as expected.
    - Detection: This can be deteted by higher than expected counts of
      `kube_persistentvolumeclaim_status_phase{phase=Bound}`, lower than
      expected counts of `kube_persistentvolume_status_phase{phase=Released}`,
      and by an operator listing and examining PVCs.
    - Mitigations: We expect this to happen only if there are other,
      operator-installed, controllers that are also managing owner refs on
      PVCs. Any such PVCs can be deleted manually. The conflicting controllers
      will have to be manually discovered.
    - Diagnostics: Logs from kube-controller-manager and stateful set controller.
    - Testing: Tests are in place for confirming owner refs are added by the
      `StatefulSet` controller, but Kubernetes does not test against external
      custom controller.

##### What steps should be taken if SLOs are not being met to determine the problem?

Stateful set SLOs are new with this feature and are in process of being
evaluated. If they are not being met, the kube-controller-manager (where the
stateful set controller lives) should be examined and/or restarted.

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

  - 1.21, KEP created.
  - 1.23, alpha implementation.
  - 1.27, graduation to beta.
  - 1.31, fix controller references.
  - 1.32, graduation to GA.

## Drawbacks
The StatefulSet field update is required.

## Alternatives
Users can delete the PVC manually. The friction associated with that is the motivation of
the KEP.
