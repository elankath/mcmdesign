- [Reconcile Cluster Machine Deployment](#reconcile-cluster-machine-deployment)
  - [Rollout Rolling](#rollout-rolling)
    - [1. Get new machine set corresponding to machine deployment and old machine sets](#1-get-new-machine-set-corresponding-to-machine-deployment-and-old-machine-sets)
    - [2. Taint the nodes backing the old machine sets.](#2-taint-the-nodes-backing-the-old-machine-sets)
    - [3. Add AutoScaler Scale-Down annotations to Nodes of Old Machine Sets](#3-add-autoscaler-scale-down-annotations-to-nodes-of-old-machine-sets)
    - [4. Reconcile New Machine Set by calling `reconcileNewMachineSet`](#4-reconcile-new-machine-set-by-calling-reconcilenewmachineset)
  - [Helper Methods](#helper-methods)
    - [scaleMachineSet](#scalemachineset)
    - [Get Machine Sets for Machine Deployment](#get-machine-sets-for-machine-deployment)
    - [Get Machine Map for Machine Deployment](#get-machine-map-for-machine-deployment)
    - [Terminate Machine Sets of Machine eDeployment](#terminate-machine-sets-of-machine-edeployment)
    - [Sync Deployment Status](#sync-deployment-status)
    - [Gets New and Old MachineSets and Sync Revision](#gets-new-and-old-machinesets-and-sync-revision)
    - [Overview](#overview)
    - [Detail](#detail)
    - [Annotate Nodes Backing Machine Sets](#annotate-nodes-backing-machine-sets)
    - [Claim Machines](#claim-machines)
      - [Summary](#summary)
  - [Helper Functions](#helper-functions)
    - [Compute New Machine Set New Replicas](#compute-new-machine-set-new-replicas)
# Reconcile Cluster Machine Deployment


```go
func (dc *controller) reconcileClusterMachineDeployment(key string) error 
```

- Gets the deployment name.
- Gets the `MachineDeployment`
- TODO: WEIRD: freeze labels and deletion timestamp
- TODO: unclear why we do this
```go
	// Resync the MachineDeployment after 10 minutes to avoid missing out on missed out events
	defer dc.enqueueMachineDeploymentAfter(deployment, 10*time.Minute)
```
- Add finalizers if deletion time stamp is nil
- TODO: Why is observed generation only updated conditionally in the below ? Shouldn't it be done always 
```go
everything := metav1.LabelSelector{}
	if reflect.DeepEqual(d.Spec.Selector, &everything) {
		dc.recorder.Eventf(d, v1.EventTypeWarning, "SelectingAll", "This deployment is selecting all machines. A non-empty selector is required.")
		if d.Status.ObservedGeneration < d.Generation {
			d.Status.ObservedGeneration = d.Generation
			dc.controlMachineClient.MachineDeployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{})
		}
		return nil
	}
```
- Get `[]*v1alpha1.MachineSet` for this deployment using `getMachineSetsForMachineDeployment` and assign to `machineSets`
- if `deployment.DeletionTimestamp != nil` 
  - if there are no finalizers on deployment return nil
  - if `len(machineSets) == 0` delete the machine deployment finalizers and return nil
  - Call `dc.terminateMachineSets(ctx, machineSets)`
 

## Rollout Rolling

```go
func (dc *controller) rolloutRolling(ctx context.Context, 
    d *v1alpha1.MachineDeployment, 
    msList []*v1alpha1.MachineSet, 
    machineMap map[types.UID]*v1alpha1.MachineList) error 
```

### 1. Get new machine set corresponding to machine deployment and old machine sets
```go
newMS, oldMSs, err := dc.getAllMachineSetsAndSyncRevision(ctx, d, msList, machineMap, true)
allMSs := append(oldMSs, newMS)
```
### 2. Taint the nodes backing the old machine sets. 
This is a preference - the k8s scheduler will try to avoid placing a pod that does not tolerate thee taint on the node. Q: Why don't we use `NoSchedule` instead ? Any pods scheduled on this node will need to be drained - more work to be done.
```go
dc.taintNodesBackingMachineSets(
		ctx,
		oldISs, &v1.Taint{
			Key:    PreferNoScheduleKey,
			Value:  "True",
			Effect: "PreferNoSchedule",
		},
	)
```
### 3. Add AutoScaler Scale-Down annotations to Nodes of Old Machine Sets

1. Create the map. (TODO: Q: Why do we add 2 annotations ?)
   ```go
	clusterAutoscalerScaleDownAnnotations := make(map[string]string)
    clusterAutoscalerScaleDownAnnotations["cluster-autoscaler.kubernetes.io/scale-down-disabled"]="true"
    clusterAutoscalerScaleDownAnnotations["cluster-autoscaler.kubernetes.io/scale-down-disabled-by-mcm"]="true"
   ```
2. Call `annotateNodesBackingMachineSets(ctx, allMSs, clusterAutoscalerScaleDownAnnotations)`

### 4. Reconcile New Machine Set by calling `reconcileNewMachineSet`
```go
	scaledUp, err := dc.reconcileNewMachineSet(ctx, allISs, newIS, d)
```

```go
func (dc *controller) reconcileNewMachineSet(ctx context.Context, 
allMSs[]*v1alpha1.MachineSet, 
newMS *v1alpha1.MachineSet, 
deployment *v1alpha1.MachineDeployment) 
    (bool, error) 
```
1. if `newMS.Spec.Replicates == deployment.spec.Replicates` return
2. if `newMS.Spec.Replicas > deployment.Spec.Replicas`, we need to scale down. call `dc.scaleMachineSet(ctx, newMS, deployment.Spec.Replicas, "down")`
3. Compute `newReplicasCount` using `NewMSNewReplicas(deployment, allMSs, newMS)`.
4. Call `dc.scaleMachineSet(ctx, newMS, newReplicasCount, "up")`

## Helper Methods

### scaleMachineSet

```go
func (dc *controller) scaleMachineSet(ctx context.Context, 
    ms *v1alpha1.MachineSet, 
    newScale int32, 
    deployment *v1alpha1.MachineDeployment, 
    scalingOperation string) 
        (bool, *v1alpha1.MachineSet, error) {
sizeNeedsUpdate := (ms.Spec.Replicas) != newScale
}
```
TODO: fill me in.



### Get Machine Sets for Machine Deployment

```go
func (dc *controller) getMachineSetsForMachineDeployment(ctx context.Context, 
        d *v1alpha1.MachineDeployment) 
    ([]*v1alpha1.MachineSet, error) 
```
- Get all machine sets using machine set lister.
- `NewMachineSetControllerRefManager` unclear


### Get Machine Map for Machine Deployment

Returns a map from MachineSet UID to a list of Machines controlled by that MS, according to the Machine's ControllerRef.
```go
func (dc *controller) 
    getMachineMapForMachineDeployment(d *v1alpha1.MachineDeployment, 
        machineSets []*v1alpha1.MachineSet) 
     (map[types.UID]*v1alpha1.MachineList, error) {
```

### Terminate Machine Sets of Machine eDeployment

```go

func (dc *controller) terminateMachineSets(ctx context.Context, machineSets []*v1alpha1.MachineSet) 

```

### Sync Deployment Status

```go
func (dc *controller) syncStatusOnly(ctx context.Context, 
    d *v1alpha1.MachineDeployment, 
    msList []*v1alpha1.MachineSet, 
    machineMap map[types.UID]*v1alpha1.MachineList) error 
```

### Gets New and Old MachineSets and Sync Revision 

```go
func (dc *controller) getAllMachineSetsAndSyncRevision(ctx context.Context, 
    d *v1alpha1.MachineDeployment, 
    msList []*v1alpha1.MachineSet, 
    machineMap map[types.UID]*v1alpha1.MachineList, 
    createIfNotExisted bool) 
        (*v1alpha1.MachineSet, []*v1alpha1.MachineSet, error) 
```

### Overview
`getAllMachineSetsAndSyncRevision` does the following:
1.  Get all old `MachineSets` the `MachineDeployment:` `d` targets, and calculate the max revision number among them (`maxOldV`).
2.  Get new `MachineSet` this deployment targets ie whose machine template matches the deployment's and updates new machine set's revision number to (`maxOldV + 1`),
This is done only if its revision number is smaller than `(maxOldV + 1)`.  If this step failed, we'll update it in the next deployment sync loop.
3.  Copy new `MachineSet`'s revision number to the `MachineDeployment` (update deployment's revision). If this step failed, we'll update it in the next deployment sync loop. 

### Detail

- TODO: describe me

### Annotate Nodes Backing Machine Sets

```go
func (dc *controller) annotateNodesBackingMachineSets(
    ctx context.Context, 
    machineSets []*v1alpha1.MachineSet, 
    annotations map[string]string) error 
```
1. Iterate through the `machineSets`. Loop variable: `machineSet`
2. List all the machines. TODO: EXPENSIVE ??
```go
allMachines, err := dc.machineLister.List(labels.Everything())
```
3. Get the Selector for the Machine Set
```go
   	selector, err := metav1.LabelSelectorAsSelector(machineSet.Spec.Selector)
```
4. Claim the Machines for the given `machineSet` using the selector
```go
    filteredMachines, err = dc.claimMachines(ctx, machineSet, selector, allMachines)
```
5. Iterate through `filteredMachines`, loop variable: `machine` and if `Node` is not empty, add or update annotations on node.
```go
if machine.Status.Node != "" {
err = AddOrUpdateAnnotationOnNode(
    ctx,
    dc.targetCoreClient,
    machine.Status.Node,
    annotations,
    )
}
```

### Claim Machines
Basically sets or unsets the owner reference of the machines matching selector to the deployment controller.
```go
func (c *controller) claimMachines(ctx context.Context, 
    machineSet *v1alpha1.MachineSet, 
    selector labels.Selector, 
    allMachines []*v1alpha1.Machine) 
    ([]*v1alpha1.Machine, error) {
```
TODO: delegates to `MachineControllerRefManager.claimMachines`

Sets or un-sets the owner reference of the machine object to the deployment controller. 

#### Summary
- iterates through `allMachines`. Checks if `selector` matches the machine labels: `m.Selector.Matches(labels.Set(machine.Labels)`
- Gets the `controllerRef` of the machine using `metav1.GetControllerOf(machine)`
- If `controllerRef` is not nil and  the `controllerRef.UID` matches the 
- If so, then this is an adoption and calls `AdoptMachine` which patches the machines owner reference using the below:
```go
addControllerPatch := fmt.Sprintf(
		`{"metadata":{"ownerReferences":[{"apiVersion":"machine.sapcloud.io/v1alpha1","kind":"%s","name":"%s","uid":"%s","controller":true,"blockOwnerDeletion":true}],"uid":"%s"}}`,
		m.controllerKind.Kind,
		m.Controller.GetName(), m.Controller.GetUID(), machine.UID)
	err := m.machineControl.PatchMachine(ctx, machine.Namespace, machine.Name, []byte(addControllerPatch))
err := m.machineControl.PatchMachine(ctx, machine.Namespace, machine.Name, []byte(addControllerPatch))
```


## Helper Functions

### Compute New Machine Set New Replicas

`NewMSNewReplicas` calculates the number of replicas a deployment's new machine set _should_ have.
1. The new MS is saturated: newMS's replicas == deployment's replicas
2. Max number of machines allowed is reached: deployment's replicas + maxSurge == allMS's replicas

```go
func NewMSNewReplicas(deployment *v1alpha1.MachineDeployment, 
    allMSs []*v1alpha1.MachineSet, 
    newMS *v1alpha1.MachineSet) (int32, error) 
    // MS was called IS earlier (instance set)
```
1. Get the `maxSurge`
```go
maxSurge, err = intstr.GetValueFromIntOrPercent(
    deployment.Spec.Strategy.RollingUpdate.MaxSurge,
    int(deployment.Spec.Replicas),
    true
)
```
2. Compute the `currentMachineCount`: iterate through all machine sets and sum up `machineset.Status.Replicas`
3. `maxTotalMachines = deployment.Spec.Replicas + maxSurge`
4. `if currentMachineCount >= maxTotalMachines return newMS.Spec.Replicas` // cannot scale up.
5. Compute	`scaleUpCount := maxTotalMachines - currentMachineCount`
6. Make sure `scaleUpCount` does not exceed desired deployment replicas	`scaleUpCount = int32(integer.IntMin(int(scaleUpCount), int(deployment.Spec.Replicas -newMS.Spec.Replicas))`


