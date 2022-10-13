🚧 WIP at the moment. Lot more material to be added here from notes. Please do not read presently.
- [Issues](#issues)
	- [Design Issues](#design-issues)
		- [Bad Packaging](#bad-packaging)
		- [LastOperation is actually Next Operation](#lastoperation-is-actually-next-operation)
		- [Description misused](#description-misused)
	- [Gaps](#gaps)
		- [Dead/Deprecated Code](#deaddeprecated-code)
			- [controller.triggerUpdationFlow](#controllertriggerupdationflow)
				- [SafetyOptions.MachineDrainTimeout](#safetyoptionsmachinedraintimeout)
			- [Dup Code](#dup-code)
	- [drainNode Handling](#drainnode-handling)
	- [Node Conditions](#node-conditions)
	- [VolumeAttachment](#volumeattachment)
			- [Dead? reconcileClusterNodeKey](#dead-reconcileclusternodekey)
		- [Dead? machine.go | triggerUpdationFlow](#dead-machinego--triggerupdationflow)
	- [Duplicate Initialization of EventRecorder in MC](#duplicate-initialization-of-eventrecorder-in-mc)
		- [Q? Internal to External Scheme Conversion](#q-internal-to-external-scheme-conversion)
# Issues

This section is very basic WIP atm. Please check after this warning has been removed. Lots more to be added here from notes and appropriately structured.

## Design Issues

### Bad Packaging

- `package controller` is inside import path `github.com/gardener/machine-controller-manager/pkg/util/provider/machinecontroller`


### LastOperation is actually Next Operation

Badly named. TODO: Describe more.

### Description misused

Error Prone stuff like below due to misuse of description.
```go
// isMachineStatusSimilar checks if the status of 2 machines is similar or not.
func isMachineStatusSimilar(s1, s2 v1alpha1.MachineStatus) bool {
	s1Copy, s2Copy := s1.DeepCopy(), s2.DeepCopy()
	tolerateTimeDiff := 30 * time.Minute

	// Since lastOperation hasn't been updated in the last 30minutes, force update this.
	if (s1.LastOperation.LastUpdateTime.Time.Before(time.Now().Add(tolerateTimeDiff * -1))) || (s2.LastOperation.LastUpdateTime.Time.Before(time.Now().Add(tolerateTimeDiff * -1))) {
		return false
	}

	if utilstrings.StringSimilarityRatio(s1Copy.LastOperation.Description, s2Copy.LastOperation.Description) > 0.75 {
		// If strings are similar, ignore comparison
		// This occurs when cloud provider errors repeats with different request IDs
		s1Copy.LastOperation.Description, s2Copy.LastOperation.Description = "", ""
	}

	// Avoiding timestamp comparison
	s1Copy.LastOperation.LastUpdateTime, s2Copy.LastOperation.LastUpdateTime = metav1.Time{}, metav1.Time{}
	s1Copy.CurrentStatus.LastUpdateTime, s2Copy.CurrentStatus.LastUpdateTime = metav1.Time{}, metav1.Time{}

	return apiequality.Semantic.DeepEqual(s1Copy.LastOperation, s2Copy.LastOperation) && apiequality.Semantic.DeepEqual(s1Copy.CurrentStatus, s2Copy.CurrentStatus)
}

```
## Gaps

TODO: Not comprehensive. Lots more to be added here

### Dead/Deprecated Code 

#### controller.triggerUpdationFlow
This is unused and appears to be dead code.

##### SafetyOptions.MachineDrainTimeout

This field is commented as deprecated but is still in `MCServer.AddFlags` and in the launch script of individual providers.
Ex
```
go run
cmd/machine-controller/main.go
...
machine-drain-timeout=5m
```

#### Dup Code

- Nearly all files in `pkg/controller/*.go`
- Ex: Types/func/smethods in `pkg/controller/machine_util.go`
  - Ex: Dup [NodeTerminationCondition](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/controller/machine_util.go#L48) in `pkg/controller/machine_util.go`. The one that is being actively used is [machineutils.NodeTerminationCondition](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/machineutils/utils.go#L70)
- Types/funcs/methods in `pkg/controller/drain.go` 

## drainNode Handling

1. Does not set err when `c.machineStatusUpdate` is called
2. `o.RunCordonOrUncordon` should use `apierrors.NotFound` while checking error returned by a get node op
3. [attemptEvict bool usage](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/drain/drain.go#L400) is confusing. Better design needed. `attemptEvict` is overridden in  `evictPodsWithoutPv`.
4. Misleading deep copy in `drain.Options.doAccountingOfPvs`
   ```go
   for podKey, persistentVolumeList := range pvMap {
		persistentVolumeListDeepCopy := persistentVolumeList
		//...
	```

## Node Conditions
- `CloneAndAddCondition` logic seems erroneous ?
## VolumeAttachment

```go
func (v *VolumeAttachmentHandler) dispatch(obj interface{}) {
//...
volumeAttachment := obj.(*storagev1.VolumeAttachment)
	if volumeAttachment == nil {
		klog.Errorf("Couldn't convert to volumeAttachment from object %v", obj)
		// Should return here.
	}
//...
```

#### Dead? reconcileClusterNodeKey 

This just delegates to `reconcileClusterNode` which does nothing..
```go
func (c *controller) reconcileClusterNode(node *v1.Node) error {
	return nil
}

```

### Dead? machine.go | triggerUpdationFlow
Can't find usages

## Duplicate Initialization of EventRecorder in MC

`pkg/util/provider/app.createRecorder` already dones this below.
```go
func createRecorder(kubeClient *kubernetes.Clientset) record.EventRecorder {
eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(klog.Infof)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: v1core.New(kubeClient.CoreV1().RESTClient()).Events("")})
	return eventBroadcaster.NewRecorder(kubescheme.Scheme, v1.EventSource{Component: controllerManagerAgentName})
}
```


We get the recorder from this eventBroadcaster and then pass it to the `pkg/util/provider/machinecontroller/controller.NewController` method which again does:
```go
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(klog.Infof)
	eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{Interface: typedcorev1.New(controlCoreClient.CoreV1().RESTClient()).Events(namespace)})
```
The above is useless.

### Q? Internal to External Scheme Conversion

Why do we do this ?
```go
internalClass := &machine.MachineClass{}
	err := c.internalExternalScheme.Convert(class, internalClass, nil)
	if err != nil {
		return err
	}
```