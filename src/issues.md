- [Issues](#issues)
	- [Design Issues](#design-issues)
		- [LastOperation is actually Next Operation](#lastoperation-is-actually-next-operation)
		- [Description misused](#description-misused)
	- [Gaps](#gaps)
	- [drainNode Error Handling](#drainnode-error-handling)
	- [VolumeAttachment](#volumeattachment)
# Issues

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

## drainNode Error Handling

- Does not set err when c.machineStatusUpdate is called

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