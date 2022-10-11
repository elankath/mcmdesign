- [Issues](#issues)
	- [Design Issues](#design-issues)
		- [LastOperation is actually Next Operation](#lastoperation-is-actually-next-operation)
		- [Description misused](#description-misused)
	- [Gaps](#gaps)
		- [Dead/Deprecated Code](#deaddeprecated-code)
			- [controller.triggerUpdationFlow](#controllertriggerupdationflow)
				- [SafetyOptions.MachineDrainTimeout](#safetyoptionsmachinedraintimeout)
			- [Dup Methods and Constants](#dup-methods-and-constants)
	- [drainNode Error Handling](#drainnode-error-handling)
	- [Node Conditions](#node-conditions)
	- [VolumeAttachment](#volumeattachment)
# Issues

This section is WIP atm. Please check after this warning has been removed. Lots more to be added.

## Design Issues

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

#### Dup Methods and Constants

- Methods in `pkg/controller/machine_util.go`
- Dup [NodeTerminationCondition](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/controller/machine_util.go#L48) in `pkg/controller/machine_util.go`. The one that is being actively used is [machineutils.NodeTerminationCondition](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/machineutils/utils.go#L70)

## drainNode Error Handling

- Does not set err when c.machineStatusUpdate is called

## Node Conditions
- `CloneAndAddCondition` logic seems erroneous.
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