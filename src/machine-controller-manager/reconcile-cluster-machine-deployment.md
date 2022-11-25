- [Reconcile Cluster Machine Deployment](#reconcile-cluster-machine-deployment)
  - [Rollout Rolling](#rollout-rolling)
    - [1. Get new machine set corresponding to machine deployment and old machine sets](#1-get-new-machine-set-corresponding-to-machine-deployment-and-old-machine-sets)
    - [2. Taint the nodes backing the old machine sets.](#2-taint-the-nodes-backing-the-old-machine-sets)
    - [3. Add AutoScaler Scale-Down annotations to Nodes of Old Machine Sets](#3-add-autoscaler-scale-down-annotations-to-nodes-of-old-machine-sets)
  - [Helper Methods](#helper-methods)
    - [Get Machine Sets for Machine Deployment](#get-machine-sets-for-machine-deployment)
    - [Get Machine Map for Machine Deployment](#get-machine-map-for-machine-deployment)
    - [Terminate Machine Sets of Machine eDeployment](#terminate-machine-sets-of-machine-edeployment)
    - [Sync Deployment Status](#sync-deployment-status)
    - [Gets New and Old MachineSets and Sync Revision](#gets-new-and-old-machinesets-and-sync-revision)
    - [Overview](#overview)
    - [Detail](#detail)
    - [Annotate Nodes Backing Machine Sets](#annotate-nodes-backing-machine-sets)
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
3. 

## Helper Methods

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
2. Get the Selector for the Machine Set
3. 