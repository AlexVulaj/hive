# SyncSet

- [Overview](#overview)
- [SyncSet Object Definition](#syncset-object-definition)
  - [How to use `applyBehavior`](#how-to-use-applybehavior)
  - [Patch and Resource Templates](#patch-and-resource-templates)
    - [`fromCDLabel` Custom Function](#fromcdlabel-custom-function)
  - [Example of SyncSet use](#example-of-syncset-use)
- [SelectorSyncSet Object Definition](#selectorsyncset-object-definition)
- [Ordering](#ordering)
- [Diagnosing SyncSet Failures](#diagnosing-syncset-failures)
- [Changing ResourceApplyMode](#changing-resourceapplymode)

## Overview

`SyncSet` and `SelectorSyncSet` objects facilitate resource management (create, update, delete, patch) in hive-managed clusters.

To use `SyncSet` objects to manage resources, you must create them in the same namespace as the `ClusterDeployment` resource that they manage. If you want to manage resources in clusters that match a specific label use `SelectorSyncSet` instead. These objects apply changes to clusters in any namespace that match the `clusterDeploymentSelector` that you set.

With both `SyncSets` and `SelectorSyncSets`, the individual resources and patches are reapplied when 2 hours have passed since their last reconciliation (by default) or if their contents are updated in the `SyncSet` or `SelectorSyncSets`.

The default `syncSetReapplyInterval` can be overridden by specifying a string duration within the `hiveconfig` such as `syncSetReapplyInterval: "1h"` for a one hour reapply interval.

## SyncSet Object Definition

`SyncSets` may contain a list of resource object definitions to create and a list of patches to be applied to existing objects.

```yaml
---
apiVersion: hive.openshift.io/v1
kind: SyncSet
metadata:
  name: mygroup
spec:
  clusterDeploymentRefs:
  - name: ClusterName

  resourceApplyMode: Upsert

  applyBehavior: CreateOnly

  enablePatchTemplates: false
  enableResourceTemplates: false

  resources:
  - apiVersion: user.openshift.io/v1
    kind: Group
    metadata:
      name: mygroup
    users:
    - myuser

  patches:
  - kind: ConfigMap
    apiVersion: v1
    name: foo
    namespace: default
    patch: |-
      { "data": { "foo": "new-bar" } }
    patchType: merge

  secretMappings:
  - sourceRef:
      name: ad-bind-password
      namespace: default
    targetRef:
      name: ad-bind-password
      namespace: openshift-config
```

| Field | Usage |
|-------|-------|
| `clusterDeploymentRefs` | List of `ClusterDeployment` names in the current namespace which the `SyncSet` will apply to. |
| `resourceApplyMode` | Defaults to `"Upsert"`, which indicates that objects will be created and updated to match the `SyncSet`. Existing `SyncSet` resources that are not listed in the `SyncSet` are not deleted. Specify `"Sync"` to allow deleting existing objects that were previously in the resources list. This includes deleting _all_ resources when the entire SyncSet is deleted. |
| `applyBehavior` | One of `Apply` (the default), `CreateOnly`, `CreateOrUpdate`. Affects how the controller computes the patch to apply to `resources` and `secretMappings` (but not `patches`). More details [below](#how-to-use-applybehavior). |
| `enablePatchTemplates  ` | If true, special use of golang's `text/templates` is allowed in `patches[].patch`. More details [below](#patch-and-resource-templates). |
| `enableResourceTemplates  ` | If true, special use of golang's `text/templates` is allowed in `resources`. More details [below](#patch-and-resource-templates). |
| `resources` | A list of resource object definitions. Resources will be created in the referenced clusters. |
| `patches` | A list of patches to apply to existing resources in the referenced clusters. You can include any valid cluster object type in the list. |
| `secretMappings` | A list of secret mappings. The secrets will be copied from the existing sources to the target resources in the referenced clusters |

### How to use `applyBehavior`
The `applyBehavior` setting affects how the controller computes and applies `resources` and `secretMappings` to the target cluster.
The default, `Apply`, causes us to use `kubectl apply`, which [uses](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/#how-apply-calculates-differences-and-merges-changes) the `kubectl.kubernetes.io/last-applied-configuration` annotation to decide whether fields _absent_ from the syncset resource should be *deleted* or *ignored* if they are present on the target object.
This is a complex topic, but in summary:
- `Apply` (the default): Asserts the `kubectl.kubernetes.io/last-applied-configuration` annotation on the target object, and uses it on subsequent reconciles.
- `CreateOrUpdate`: If the object is initially absent, it is created *without* the `kubectl.kubernetes.io/last-applied-configuration` annotation; and hive will not add the annotation.
  However, if the annotation is added some other way (e.g. the user runs `kubectl apply` on the object), hive will not remove it; and subsequent syncs *will* honor it.
  The behavior here is quirky:
  - Hive will always assert the presence/value of all fields in the syncset resource.
  - If the `last-applied-configuration` annotation is *absent*, hive will never delete fields.
  - If the annotation is *present*, the behavior is the same as `Apply` -- i.e. fields present in the annotation but absent from the syncset resource will be *removed*.
- `CreateOnly`: If initially absent, the object is created (without the `kubectl.kubernetes.io/last-applied-configuration` annotation).
  If the object is already present, it is ignored.

As a rule of thumb:
- If you want users of the spoke cluster to be able to edit the object and have their changes persist, use `applyBehavior: CreateOnly`.
- If you want to assert the exact version of the object in your [Selector]SyncSet, reverting any changes or additions made externally, use `applyBehavior: Apply` (or omit `applyBehavior` to get this behavior as the default).
- Since the behavior of `CreateOrUpdate` differs based on factors outside of your control -- i.e. whether the user adds/removes the `kubectl.kubernetes.io/last-applied-configuration` annotation from the target object -- this `applyBehavior` should probably be avoided.
  (If you come up with a good use case for it, please [open an issue](https://github.com/openshift/hive/issues/new) and tell us about it!)

This can present a special conundrum when you want to create a resource with certain attributes that should be "sticky", but allow updates elsewhere.
In such cases, you may wish to combine `resources` and `patches`:
- Create your [Selector]SyncSet with `applyBehavior: CreateOnly`
- Include a `resources` entry with the initial version of the object.
- Include `patches` to assert the fields whose values you wish to be "sticky" (i.e. revert if they are edited/removed externally).

It is safe to put these into the same [Selector]SyncSet because:
- `patches` in a given [Selector]SyncSet are applied after `resources`.
- `applyBehavior` only applies to `resources` and `secretMappings` -- it does not affect `patches`.

### Patch and Resource Templates
By setting `spec.enablePatchTemplates` and/or `spec.enableResourceTemplates` to `true`, it is
possible to use golang [text/template](https://pkg.go.dev/text/template)-isms in
`spec.patches[].patch` and/or `spec.resources[]` values, respectively.
Note, however, that there is no "dot" (data object) so the out-of-the-box functionality won't
convey a lot of power.
This feature exists to expose custom functions, described below.

Note:
- Templates are only honored on `resources[]` _values_. They are ignored for keys.
  In contrast, templates can be used anywhere in a `patches[].patch` as long as the entire patch
  string is valid JSON once rendered.
- Templates are only honored on `resources[]` values whose schema type is `string`.
  (This is because the embedded resource must be valid JSON _before_ it is parsed; and templates can't be recognized as any other valid JSON type.)
  In contrast, templates can be used for any data type in a `patches[].patch` as long as the
  rendered string contains valid and appropriate JSON data types.
- Errors parsing or processing the template will cause the SyncSet to fail and will be bubbled up
  in the ClusterSync status as usual.

#### `fromCDLabel` Custom Function
With `enableResourceTemplates` on, including a string like

`{{ fromCDLabel "any.clusterdeployment/label-key" }}`

in a resource will cause hive to substitute it with the value of that label from
the ClusterDeployment owning the [Selector]SyncSet being processed. For
example:

```yaml
---
  enableResourceTemplates  : true

  resources:
  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: mycm
      namespace: myns
    data:
      platform: the platform is {{ fromCDLabel "hive.openshift.io/cluster-platform" }}
```

...might result in the following ConfigMap on the spoke cluster:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mycm
  namespace: myns
data:
  platform: the platform is gcp
```

If the ClusterDeployment has no labels, or if there is no label with the specified key,
the empty string is substituted.

### Example of SyncSet use

In this example you can change the replicaset of a deployment running on top of a Hive managed OpenShift cluster.

* Get the required information of the deployment

```console
$ oc get deployment <deployment name> -o yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: sise
  name: sise-deploy
  namespace: default
spec:
  replicas: 2
  xxxxxxx
```

* Create the `SyncSet` object in the cluster deployment namespace in Hive managed cluster as mentioned below.

```sh
oc create -f <syncset_file.yaml> -n <namespace>
```

SyncSet File:

```yaml
apiVersion: hive.openshift.io/v1
kind: SyncSet
metadata:
  name: sise-deploy-syncset
spec:
  clusterDeploymentRefs:
  - name: <cluster name>

  patches:
  - kind: Deployment
    apiVersion: extensions/v1beta1
    name: sise-deploy
    namespace: default
    patch: |-
      { "spec": { "replicas": 3 } }
    patchType: strategic
```

* To see the syncset status, run as below.

```sh
oc get clustersync <clusterdeployment name> -o yaml
```

## SelectorSyncSet Object Definition

`SelectorSyncSet` functions identically to `SyncSet` but is applied to clusters matching `clusterDeploymentSelector` in any namespace.

```yaml
---
apiVersion: hive.openshift.io/v1
kind: SelectorSyncSet
metadata:
  name: mygroup
spec:
  resources:
  - apiVersion: user.openshift.io/v1
    kind: Group
    metadata:
      name: mygroup
    users:
    - abutcher
  clusterDeploymentSelector:
    matchLabels:
      cluster-group: abutcher
```

| Field | Usage |
|-------|-------|
| `clusterDeploymentSelector` | A key/value label pair which selects matching `ClusterDeployments` in any namespace. |

## Ordering
Hive will process [Selector]SyncSets and their resources in the following order:
1. SyncSets are processed first.
   1. Any necessary deletions are processed first.
      The order in which deletions are processed is not guaranteed.
   1. SyncSets are processed in alpha order by SyncSet name.
      Resources within a SyncSet are processed in the order in which they are supplied in the SyncSet.
1. SelectorSyncSets are processed next.
   1. Any necessary deletions are processed first.
      The order in which deletions are processed is not guaranteed.
   1. SelectorSyncSets are processed in alpha order by SelectorSyncSet name.
      Resources within a SelectorSyncSet are processed in the order in which they are supplied in the SelectorSyncSet.

Within a given [Selector]SyncSet, sections are processed in the following order:
1. `resources`
2. `secretMappings`
3. `patches`

## Diagnosing SyncSet Failures

To find the status of the syncset, check the cluster deployment's `ClusterSync` object in the cluster deployment namespace. Every cluster deployment has an associated `ClusterSync` object that records status within `ClusterSync.Status.SyncSets`.

```sh
oc get clustersync -n <namespace>
```

To see details, run as below.

```sh
oc get clustersync <clusterdeployment name> -o yaml
```

All (Selector)SyncSets for a given ClusterDeployment are handled by a single replica of the hive-clustersync StatefulSet.
The replica pods are named `hive-clustersync-N` where `N` is an integer such that `0 <= N < #replicas`.
`ClusterSync.Status.ControlledByReplica` indicates which replica is responsible for (the CD associated with) the ClusterSync.
Use this to determine which replica's pod logs to examine when debugging syncset operations.
For example:

```sh
$ REPLICA=`oc get clustersync -n <namespace> <clusterdeployment name> -o json | jq -r .status.controlledByReplica`
$ oc logs -n hive hive-clustersync-$REPLICA | grep <clusterdeployment name>
```

**Note:** This value is only set/updated when the cluster is installed, reachable, and not deleted.

**Note:** This value indicates the replica that *most recently* handled the ClusterSync.
If the hive-clustersync statefulset is scaled up or down, the controlling replica can change,
potentially causing logs to be spread across multiple pods.

## Changing ResourceApplyMode

Changing the `resourceApplyMode` from `"Sync"` to `"Upsert"` will remove `SyncSet` resources tracked for deletion within the corresponding `ClusterSync` object. It is possible that the `ClusterSync` controller could process a resource removal and a `resourceApplyMode` change simultaneously and when this occurs resources no longer tracked in the `SyncSet` will be orphaned rather than deleted.

Likewise, changing the `resourceApplyMode` from `"Upsert"` to `"Sync"` will add `SyncSet` resources to resources tracked for deletion within the corresponding `ClusterSync` object. When the `ClusterSync` controller processes a resource removal and a `resourceApplyMode` change simultaneously, resources removed will be orphaned rather than deleted.
