- [Node Drain](#node-drain)
	- [Drain Utilities](#drain-utilities)
		- [VolumeAttachmentHandler](#volumeattachmenthandler)
			- [VolumeAttachmentHandler.AddWorker](#volumeattachmenthandleraddworker)
			- [VolumeAttachmentHandler.dispatch](#volumeattachmenthandlerdispatch)
			- [VolumeAttachmentHandler Initialization in MC](#volumeattachmenthandler-initialization-in-mc)
	- [Drain](#drain)
		- [Drain Types](#drain-types)
			- [Drain Constants](#drain-constants)
			- [drain.Options](#drainoptions)
		- [drain.PodVolumeInfo](#drainpodvolumeinfo)
		- [drain.Options.evictPod](#drainoptionsevictpod)
		- [drain.Options.deletePod](#drainoptionsdeletepod)
		- [drain.Options.getPodsForDeletion](#drainoptionsgetpodsfordeletion)
		- [drain.Options.getPVList](#drainoptionsgetpvlist)
		- [drain.Options.getVolIDsFromDriver](#drainoptionsgetvolidsfromdriver)
		- [drain.Options.doAccountingOfPvs](#drainoptionsdoaccountingofpvs)
		- [drain.filterPodsWithPv](#drainfilterpodswithpv)
		- [drain.Options.waitForDetach](#drainoptionswaitfordetach)
		- [drain.Options.waitForReattach](#drainoptionswaitforreattach)
		- [drain.Options.waitForDelete](#drainoptionswaitfordelete)
		- [drain.Options.RunDrain](#drainoptionsrundrain)
		- [drain.Options.evictPodsWithoutPv](#drainoptionsevictpodswithoutpv)
			- [drain.Options.evictPodWithoutPVInternal](#drainoptionsevictpodwithoutpvinternal)
			- [isMisconfiguredPdb](#ismisconfiguredpdb)
		- [drain.Options.evictPodsWithPv](#drainoptionsevictpodswithpv)
		- [drain.Options.evictPodsWithPVInternal](#drainoptionsevictpodswithpvinternal)

# Node Drain

Node Drain code is in [github.com/gardener/machine-controller-manager/pkg/util/provider/drain/drain.go](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/drain/drain.go)

## Drain Utilities 

### VolumeAttachmentHandler

[pkg/util/provider/drain.VolumeAttachmentHandler](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/util/provider/drain#VolumeAttachmentHandler) is an handler used to distribute
incoming [k8s.io/api/storage/v1.VolumeAttachment](https://pkg.go.dev/k8s.io/api/storage/v1#VolumeAttachment) requests to a number of workers where each worker is a channel of type `*VolumeAttachment`. 

A [k8s.io/api/storage/v1.VolumeAttachment](https://pkg.go.dev/k8s.io/api/storage/v1#VolumeAttachment) is a non-namespaced k8s object that captures the intent to attach or detach the specified volume to/from the specified node. See [VolumeAttachment](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/volume-attachment-v1/)

```go
type VolumeAttachmentHandler struct {
	sync.Mutex
	workers []chan *storagev1.VolumeAttachment
}

// NewVolumeAttachmentHandler returns a new VolumeAttachmentHandler
func NewVolumeAttachmentHandler() *VolumeAttachmentHandler {
	return &VolumeAttachmentHandler{
		Mutex:   sync.Mutex{},
		workers: []chan *storagev1.VolumeAttachment{},
	}
}

```

#### VolumeAttachmentHandler.AddWorker

`AddWorker` appends a buffered channel of size `20` of type `VolumeAttachment` to the `workers` slice  in `VolumeAttachmentHandler`
. There is an assumption that not more than 20 unprocessed objects would exist at a given time. On bufferring requests beyond this the channel will start dropping writes. See `dispatch` method.

```go
func (v *VolumeAttachmentHandler) AddWorker() chan *storagev1.VolumeAttachment {
	// chanSize is the channel buffer size to hold requests.
	// This assumes 
	// On bufferring requests beyond this the channel will start dropping writes
	const chanSize = 20

	klog.V(4).Infof("Adding new worker. Current active workers %d - %v", len(v.workers), v.workers)

	v.Lock()
	defer v.Unlock()

	newWorker := make(chan *storagev1.VolumeAttachment, chanSize)
	v.workers = append(v.workers, newWorker)

	klog.V(4).Infof("Successfully added new worker %v. Current active workers %d - %v", newWorker, len(v.workers), v.workers)
	return newWorker
}

```
#### VolumeAttachmentHandler.dispatch
The `dispatch` method is responsible for distributing incomding `VolumeAttachent`s to available channels.

```go
func (v *VolumeAttachmentHandler) dispatch(obj interface{}) {
	if len(v.workers) == 0 {
		// As no workers are registered, nothing to do here.
		return
	}
	volumeAttachment := obj.(*storagev1.VolumeAttachment)
	v.Lock()
	defer v.Unlock()

	for i, worker := range v.workers {
		select {
		// submit volume attachment to the worker channel if channel is not full
		case worker <- volumeAttachment:
		default:
			klog.Warningf("Worker %d/%v is full. Discarding value.", i, worker)
			// TODO: Umm..isn't this problematic if we miss this ?
		}
	}
}
```

The `Add|Update` methods below delegate to dispatch. The usage of this utility involves specifying the add/update methods below as the event handler callbacks on an instance of [k8s.io/client-go/informers/storage/v1.VolumeAttachmentInformer](https://pkg.go.dev/k8s.io/client-go@v0.25.2/informers/storage/v1#VolumeAttachmentInformer). This way incoming volume attachments are distributed to several worker channels.
```go
func (v *VolumeAttachmentHandler) AddVolumeAttachment(obj interface{}) {
	v.dispatch(obj)
}

func (v *VolumeAttachmentHandler) UpdateVolumeAttachment(oldObj, newObj interface{}) {
	v.dispatch(newObj)
}
```
#### VolumeAttachmentHandler Initialization in MC
During construction of the MC controller struct, we initialize the callback methods on volume attachment handler using the volume attachment informer

```go
func NewController(...) {
	//...
controller.volumeAttachmentHandler = drain.NewVolumeAttachmentHandler()
volumeAttachmentInformer.Informer().AddEventHandler(
	cache.ResourceEventHandlerFuncs{
			AddFunc:    controller.volumeAttachmentHandler.AddVolumeAttachment,
			UpdateFunc: controller.volumeAttachmentHandler.UpdateVolumeAttachment,
});
```


## Drain 

### Drain Types

#### Drain Constants

- `PodEvictionRetryInterval` is the interval in which to retry eviction for pods
- `GetPvDetailsMaxRetries` is the number of max retries to get PV details using the [PersistentVolumeLister](https://pkg.go.dev/k8s.io/client-go/listers/core/v1#PersistentVolumeLister) or [PersistentVolumeClaimLister](https://pkg.go.dev/k8s.io/client-go/listers/core/v1#PersistentVolumeClaimLister)
- `GetPvDetailsRetryInterval` is the interval in which to retry getting PV details
```go
const (
    PodEvictionRetryInterval = time.Second * 20
	GetPvDetailsMaxRetries = 3
	GetPvDetailsRetryInterval = time.Second * 5
)
```

#### drain.Options

`drain.Options` are configurable options while draining a node before deletion

NOTE: Unused fields/Fields with constant defaults omitted for brevity
```go
type Options struct {
	client                       kubernetes.Interface
	kubernetesVersion            *semver.Version
	Driver                       driver.Driver
	drainStartedOn               time.Time
	drainEndedOn                 time.Time
	ErrOut                       io.Writer
	ForceDeletePods              bool
	MaxEvictRetries              int32
	PvDetachTimeout              time.Duration
	PvReattachTimeout            time.Duration
	nodeName                     string
	Out                          io.Writer
	pvcLister                    corelisters.PersistentVolumeClaimLister
	pvLister                     corelisters.PersistentVolumeLister
	pdbV1Lister                  policyv1listers.PodDisruptionBudgetLister
	nodeLister                   corelisters.NodeLister
	volumeAttachmentHandler      *VolumeAttachmentHandler
	Timeout                      time.Duration
}

```

### drain.PodVolumeInfo

`drain.PodVolumeInfo` is the struct used to encapsulate the PV names and PV ID's for all the [PVs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) attached to the pod
```go
PodVolumeInfo struct {
	persistentVolumeList []string
	volumeList           []string
}
```
NOTE: The struct fields are badly named.
- `PodVolumeInfo.persistentVolumeList` is a slice of persistent volume names. This is from [PersistentVolumeSpec.VolumeName](https://pkg.go.dev/k8s.io/api/core/v1#PersistentVolumeSpec)
- `PodVolumeInfo.volumeList` is a slice of persistent volume IDs. This is obtained using [driver.GetVolumeIDs](../src/mcm_facilities.md#driver) given the PV Spec. This is generally the CSI volume id.


### drain.Options.evictPod

```go
func (o *Options) evictPod(ctx context.Context, pod *corev1.Pod, policyGroupVersion string) error 
```
[drain.Options.evictPod](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/drain/drain.go#L363) is a simple helper method to evict a Pod using [Eviction API](https://pkg.go.dev/k8s.io/client-go@v0.25.2/kubernetes/typed/policy/v1#EvictionExpansion)
- TODO: `GracePeriodSeconds` in the code is useless here and should be removed as it is always -1.
- TODO: Currently this method uses old `k8s.io/api/policy/v1beta1`. It must be changed to  `k8s.io/api/policy/v1`
```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

Begin(("" ))
-->InitTypeMeta["
		typeMeta:= metav1.TypeMeta{
			APIVersion: policyGroupVersion,
			Kind:       'Eviction',
		},
"]
-->InitObjectMeta["
		objectMeta := ObjectMeta: metav1.ObjectMeta{
			Name:      pod.Name,
			Namespace: pod.Namespace,
		},
"]
-->InitEviction["
eviction := &.Eviction{TypeMeta: typeMeta, ObjectMeta: objectMeta }
"]
-->EvictPod["
 err := o.client.PolicyV1beta1().Evictions(eviction.Namespace).Evict(ctx, eviction)
"]
-->ReturnErr(("return err"))

style InitEviction text-align:left
```

### drain.Options.deletePod

Simple helper method to delete a Pod
```go
func (o *Options) deletePod(ctx context.Context, pod *corev1.Pod) error {
```
Just delegates to [PodInterface.Delete](https://pkg.go.dev/k8s.io/client-go@v0.25.2/kubernetes/typed/core/v1#PodInterface)

```go
o.client.CoreV1().Pods(pod.Namespace).Delete(ctx, pod.Name, metav1.DeleteOptions{} )
```

### drain.Options.getPodsForDeletion

`drain.getPodsForDeletion` returns all the pods we're going to delete.  If there are  any pods preventing us from deleting, we return that list in an error.

```go
func (o *Options) getPodsForDeletion(ctx context.Context) 
	(pods []corev1.Pod, err error)
```
1. Get all pods associated with the node.
   ```go
   podList, err := o.client.CoreV1().Pods(metav1.NamespaceAll).List(ctx, metav1.ListOptions{
		FieldSelector: fields.SelectorFromSet(fields.Set{"spec.nodeName": o.nodeName}).String()})
   ```
2. Iterate through `podList`.
3. Apply a bunch of pod filters. 
   1. Remove mirror pods from consideration for deletion. See [Static Pods](https://github.com/kubernetes/enhancements/blob/master/keps/sig-auth/1314-node-restriction-pods/README.md#background)
   2. Local Storage Filter. Discuss: seems useless. If Pod has local storge, remove it from consideration. This filter iterates through `Pod.Spec.Volumes` slice and checks whether `Volume.EmptyDir` is non nil in order to determine
   3. A Pod whose `Pod.Status.Phase` is [PodSucceeded](https://pkg.go.dev/k8s.io/api/core/v1#PodSucceeded) or [PodFailed](https://pkg.go.dev/k8s.io/api/core/v1#PodFailed) is eligible for deletion
      1. If a Pod has a controller owner reference, it is eligible for deletion. (TODO: Unsure why this makes a difference anyways)
   4. The final pod filter `daemonsetFilter` seems useless. Discuss.


### drain.Options.getPVList

NOTE: Should be called `getPVNames`.  Gets a slice of the persistent volume names bound to the Pod through its claims.  Contains time.sleep and retry handling to a limit. Unsure if this is the best way. Discuss.

```go
func (o *Options) getPVList(pod *corev1.Pod) (pvNames []string, err error) 
```
1. Iterate over `pod.Spec.Volumes`.
2. If `volume.PersistentVolumeClaim` reference is not nil, gets the `PersistentVolumeClaim` using `o.pvcLister` using `vol.PersistentVolumeClaim.ClaimName`.
   1. Implements error handling and retry till `GetPvDetailsMaxRetries` is reached with interval `GetPvDetailsRetryInterval` for the above.
3. Adds `pvc.Spec.VolumeName` to `pvNames`
4. Return `pvNames`



### drain.Options.getVolIDsFromDriver

Given a slice of PV Names, this method gets the corresponding volume ids from the driver. 
- It does this by first getting the [PersistentVolumeSpec](https://pkg.go.dev/k8s.io/api/core/v1#PersistentVolumeSpec) using `o.pvLister.Get(pvName)` for each PV name and adding to the `pvSpecs` slice of type `PersistentVolumeSpec`. See [k8s.io/client-go/listers/core/v1.PersistentVolumeLister](https://pkg.go.dev/k8s.io/client-go/listers/core/v1#PersistentVolumeLister)
-  Retry handling is implemented here while looking up pvName till `GetPvDetailsMaxRetries` is reached with sleep interval of `GetPvDetailsRetryInterval` between each retry attempt.
- Once `pvSpecs` slice is populated it constructs a [driver.GetVolumeIDsRequest](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/util/provider/driver#GetVolumeIDsRequest) from the same and then invokes `driver.GetVolumeIDs(driver.GetVolumeIDsRequest))` to obtain the [driver.GetVolumeIDsResponse](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/util/provider/driver#GetVolumeIDsResponse) and retruns `driver.GetVolumeIDsResponse.VolumeIDs`

TODO: BUG ? In case the PV is not found or retry limit is reached the slice of volume ids will not have a 1:1 correspondence with slice of PV names passed in.

```go
func (o *Options) getVolIDsFromDriver(ctx context.Context, pvNames []string) ([]string, error)
```

### drain.Options.doAccountingOfPvs

[drain.Options.doAccountingOfPvs](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/drain/drain.go#L509) returns a map of the pod key `pod.Namespace + '/' + pod.Name` to a [PodVolumeInfo](#drainpodvolumeinfo) struct which holds a slice of PV names and PV IDs.

NOTES:
- See [filterSharedPVs](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/drain/drain.go#L544)

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD
Begin((" "))
-->Init["
	podKey2VolNamesMap = make(map[string][]string)
	podKey2VolInfoMap = make(map[string]PodVolumeInfo)
"]
-->RangePods["
	for pod := range pods
"]
-->PopPod2VolNames["
	podKey2VolNamesMap[pod.Namespace + '/' pod.Name] = o.getPVList(pod)
"]
--loop-->RangePods
PopPod2VolNames--done-->FilterSharedPVs["
	filterSharedPVs(podKey2VolNamesMap)
// filters out the PVs that are shared among pods.
"]
-->RangePodKey2VolNamesMap["
	for podKey, volNames := range podKey2VolNamesMap
"]
-->GetVolumeIds["
	volumeIds, err := o.getVolIDsFromDriver(ctx, volNames)
	if err != nil continue; //skip set of volumes
"]
-->InitPodVolInfo["
	podVolumeInfo := PodVolumeInfo{
			persistentVolumeList: volNames,
			volumeList:           volumeIds
	}
	//struct field names are bad.
"]
-->PopPodVolInfoMap["
	podVolumeInfoMap[podKey] = podVolumeInfo
"]
--loop-->RangePodKey2VolNamesMap
PopPodVolInfoMap--done-->Return(("return podVolumeInfoMap"))
```

### drain.filterPodsWithPv

NOTE: should have been named `partitionPodsWithPVC`

Utility function that iterates through given `pods` and for each `pod`, iterates through its `pod.Spec.Volumes`. For each such pod `volume` checks `volume.PersistentVolumeClaim`. If not nil, adds `pod` to slice `podsWithPV` else adds `pod` to slice `podsWithoutPV`

```go
func filterPodsWithPv(pods []corev1.Pod) 
    (podsWithPV []*corev1.Pod, podsWithoutPV []*corev1.Pod) 
```

### drain.Options.waitForDetach

```go
func (o *Options) waitForDetach(ctx context.Context, 
	podVolumeInfo PodVolumeInfo, 
	nodeName string) error
```
Summary: 
1. Initiaze boolean `found` to true. (Representing that a volume is still attached to a node).
2. Begins a loop while `found` is true
3. Uses a `select ` and checks to see if a signal is received from `context.Done()` (ie context cancelled). If so, return an error with the message that a timeout occurred while waiting for PV's to be detached.
4. Sets `found` to false.
5. Gets the `Node` associated with `nodeName` using the `nodeLister` and assign to `node`. If there is an error return from the function.
6. Gets the `node.Status.VolumesAttached` which returns a [[]AttachedVolume](https://pkg.go.dev/k8s.io/api/core/v1#AttachedVolume) and assign to `attachedVols`.
   1. Return `nil` if this slice is empty.
7. Begin iteration `range podVolumeInfo.volumeList` assigning `volumeID` in parent iteration. Label this iteration as `LookUpVolume`.
   1. Begin inner iteration over `attachedVols` assigning `attachedVol` in nested iteration
   2. If `attachedVol.Name` is contained in `volumeID` then this volume is still attached to the node.  
      1. Set `found` to true
      2. Sleep for `VolumeDetachPollInterval` seconds (5 seconds)
      3. Break out of `LookUpVolume`

### drain.Options.waitForReattach

Purpose: Waits for persistent volume names in `podVolumeInfo.persistentVolumeList` to be re-attached (to another node).

But I am still confused on why we need to call this during the drain flow. Why should we wait for PVs to be attached to another node. After all, there is no guarantee they will be attached, right ?

```go
func (o *Options) waitForReattach(ctx context.Context, 
	podVolumeInfo PodVolumeInfo, 
	previousNodeName string, 
	volumeAttachmentEventCh chan *VolumeAttachment) error 
```
1. Construct a map: `var pvsWaitingForReattachments map[string]bool`
2. Initiamize the above map by ranging through `podVolumeInfo.persistentVolumeList` taking the `persistentVolumeName` and set `pvsWaitingForReattachments[persistentVolumeName] = true`
3. Start a `for` loop.
   1. Commence a `select` with following cases:
      1. Case: Check to see if context is closed/cancelled by  reading: `<-ctx.Done()`. If so, return an error with the message that timeout occurred while waiting for PV's to reattach to another node.
      2. Case: Obtain a [*VolumeAttachment](https://pkg.go.dev/k8s.io/api/storage/v1#VolumeAttachment) by reading from channel:  `incomingEvent := <-volumeAttachmentEventCh`
         1. Get the `persistentVolumeName` associated with this attachment event. 
         2. `persistentVolumeName := *incomingEvent.Spec.Source.PersistentVolumeName`
         3.  Check if this persistent volume was being tracked: `pvsWaitingForReattachments[persistentVolumeName]` is present 
         4. Check if the volume was attached to another node
         5.  `incomingEvent.Status.Attached && incomingEvent.Spec.NodeName != previousNodeName`
         6. If above is true, then delete entry corresponding to `persistentVolumeName` from the `pvsWaitingForReattachments` map.
     1. if `pvsWaitingForReattachments` is empty break from the loop.
  1. Log that the volumes in ` podVolumeInfo.persistentVolumeList` have been successfully re-attached and return nil.

### drain.Options.waitForDelete

NOTE: Ideally should have been named `waitForPodDisappearance`

[pkg/util/provider/drain.Options.waitForDelete](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/drain/drain.go#L1068) is a helper method defined on `drain.Options` that leverages [wait.PollImmediate](./k8s_facilities.md#waitpollimmediate) and the `getPodFn` (get pod by name and namespace) and checks that all pods have disappeared within `timeout`. The set of pods that did not disappear within timeout is returned as `pendingPods`

```go
func (o *Options) waitForDelete(
        pods []*corev1.Pod, interval, 
        timeout time.Duration,  
        getPodFn func(string, string) (*corev1.Pod, error)
    ) (pendingPods []*corev1.Pod, err error) 
```


### drain.Options.RunDrain

Context: `drain.Options.RunDrain` is called from the MC helper method `controller.drainNode` which in turn is called from `controller.triggerDeletionFlow` when the `machine.Status.LastOperation.Description` contains operation `machineutils.InitiateDrain`.

If `RunDrain` returns an error, then the drain is retried at a later time by putting back the machine key into the queue. Unless the `force-deletion` label on the machine object is true - in which case we proceed to VM deletion.

```go
func (o *Options) RunDrain(ctx context.Context) error
```

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD


GetNode["node, err = .client.CoreV1().Nodes().Get(ctx, o.nodeName, metav1.GetOptions{})"]
-->ChkGetNodeErr{err != nil}

ChkGetNodeErr--Yes-->ReturnNilEarly(("return nil
(case where deletion 
triggered during machine creation, 
so node is nil. 
TODO: should use apierrors.NotFound)"))

ChkGetNodeErr--No-->ChkNodeUnschedulable{node.Spec.Unschedulable?}

ChkNodeUnschedulable--Yes-->GetPodsForDeletion["
    pods, err := o.getPodsForDeletion(ctx)
    if err!=nil return err"]
ChkNodeUnschedulable--No-->CloneNode["clone := node.DeepCopy()
		clone.Spec.Unschedulable = true"]
        -->UpdateNode["_, err = o.client.CoreV1().Nodes().Update(ctx, clone, metav1.UpdateOptions{})
        if err != nil return err
        "]

UpdateNode-->GetPodsForDeletion
GetPodsForDeletion-->GetEvictionPGV["
    policyGroupVersion, err := SupportEviction(o.client)
    if err != nil return err
    "]
-->
DefineAttemptEvict["
attemptEvict := !o.ForceDeletePods && len(policyGroupVersion) > 0
// useless boolean which confuses matters considerably.
"]
-->
DefineGetPodFn["
getPodFn := func(namespace, name string) (*corev1.Pod, error) {
		return o.client.CoreV1().Pods(namespace).Get(ctx, name, metav1.GetOptions{})
}"]
-->
CreateReturnChannel["
    returnCh := make(chan error, len(pods))
	defer close(returnCh)
    "]
-->ChkForceDelPods{"o.ForceDeletePods?"}

ChkForceDelPods--Yes-->EvictPodsWithoutPV["go o.evictPodsWithoutPv(ctx, attemptEvict, pods, policyGroupVersion, getPodFn, returnCh)
// go-routine feels un-necessary here."]
ChkForceDelPods--No-->FilterPodsWithPv["
podsWithPv, podsWithoutPv := filterPodsWithPv(pods)
"]

FilterPodsWithPv-->EvictPodsWithPv["
go o.evictPodsWithPv(ctx, attemptEvict, podsWithPv, policyGroupVersion, getPodFn, returnCh)
"]
-->EvictPodsWithoutPV1["
	go o.evictPodsWithoutPv(ctx, attemptEvict, podsWithoutPv, policyGroupVersion, getPodFn, returnCh)	
"]-->CreateAggregateError

EvictPodsWithoutPV-->CreateAggregateError["
	var errors []error
    errors =
	for i = 0; i <  len(pods); ++i {
		err := <-returnCh
		if err != nil {
			errors = append(errors, err)
		}
	}
	
"]
-->ReturnAggError(("return\nerrors.NewAggregate(errors)"))

```

Notes:
1. [machine-controller-manager/pkg/util/provider/drain.SupportEviction](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/drain/drain.go#L1106) uses Discovery API to find out if the server support eviction subresource and if so return its groupVersion or "" if it doesn't.
   1. [k8s.io/kubectl/pkg/drain.CheckEvictionSupport](https://pkg.go.dev/k8s.io/kubectl/pkg/drain#CheckEvictionSupport) already does this.
2. [attemptEvict boolean](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/drain/drain.go#L400) usage is confusing. Stick to `drain.Options.ForceDeletePods`
3. TODO: GAP? For cordoning a Node we currently just set `Node.Spec.Unschedulable`. But we are also supposed to set the taint. `node.kubernetes.io/unschedulable`. The spec way is supposed to be deprecated.


### drain.Options.evictPodsWithoutPv

drain method that iterates through each given pod and for each pod launches a go-routine that simply delegates to `Options.evictPodsWithoutPv`.

```go
func (o *Options) evictPodsWithoutPv(ctx context.Context, 
    pods []*corev1.Pod,
	policyGroupVersion string, //eviction API's GV
	getPodFn func(namespace, name string) (*corev1.Pod, error),
	returnCh chan error) {
    for _, pod := range pods {
		go o.evictPodWithoutPVInternal(ctx, attemptEvict, pod, policyGroupVersion, getPodFn, returnCh)
	}
	return
}
```




#### drain.Options.evictPodWithoutPVInternal

drian method that  that either evicts or deletes a Pod with retry handling until `Options.MaxEvictRetries` is reached.

```go
func (o *Options) evictPodWithoutPVInternal(
    ctx context.Context, 
    attemptEvict bool, 
    pod *corev1.Pod, 
    policyGroupVersion string, 
    getPodFn func(namespace, name string) (*corev1.Pod, error), 
    returnCh chan error) 

```

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD


RangePod["pod := range pods"]
RangePod-->EvictOrDelPod["go evictPodWithoutPVInternal(attemptEvict bool, pod, policyGroupVersion,pod,getPodFn,returnCh)"]
EvictOrDelPod-->Begin

subgraph "evictPodWithoutPVInternal (evicts or deletes Pod) "
Begin(("Begin"))-->SetRetry["retry := 0"]
SetRetry
-->SetAttemptEvict["if retry >= o.MaxEvictRetries {attemptEvict=false}"]
-->ChkAttemptEvict{"attemptEvict ?"}

ChkAttemptEvict--Yes-->EvictPod["err=o.evictPod(ctx, pod, policyGroupVersion)"]
ChkAttemptEvict--No-->DelPod["err=o.deletePod(ctx, pod)"]

EvictPod-->ChkErr
DelPod-->ChkErr

ChkErr{"Check err"}
ChkErr--"Nil"-->ChkForceDelPods
ChkErr--"IsTooManyRequests(err)"-->GetPdb["
    // Possible case where Pod couldn't be evicted because of PDB violation
    pdbs, err = pdbLister.GetPodPodDisruptionBudgets(pod)
    pdb=pdbs[0] if err !=nil && len(pdbs) > 0
"]
ChkErr--"IsNotFound(err)\n(pod evicted)"-->SendNilChannel-->NilReturn
ChkErr--"OtherErr"-->SendErrChannel
GetPdb-->ChkMisConfiguredPdb{"isMisconfiguredPdb(pdb)?"}
ChkMisConfiguredPdb--Yes-->SetPdbError["err=fmt.Errorf('pdb misconfigured')"]
SetPdbError-->SendErrChannel

ChkMisConfiguredPdb--No-->SleepEvictRetryInterval["time.Sleep(PodEvictionRetryInterval)"]
SleepEvictRetryInterval-->IncRetry["retry+=1"]-->SetAttemptEvict


SendErrChannel-->NilReturn

ChkForceDelPods{"o.ForceDeletePods"}
ChkForceDelPods--"Yes\n(dont wait for\npod disappear)"-->SendNilChannel
ChkForceDelPods--No-->GetPodTermGracePeriod["
    // TODO: discuss this, shouldn't pod grace period override drain ?
    timeout=Min(pod.Spec.TerminationGracePeriodSeconds,o.Timeout)
"]
-->SetBufferPeriod["bufferPeriod := 30 * time.Second"]
-->WaitForDelete["pendingPods=o.waitForDelete(pods, timeout,getPodFn)"]
-->ChkWaitForDelError{err != nil ?}

ChkWaitForDelError--Yes-->SendErrChannel
ChkWaitForDelError--No-->ChkPendingPodsLength{"len(pendingPods) > 0?"}
ChkPendingPodsLength--Yes-->SetTimeoutError["err = fmt.Errorf('pod term timeout')"]
SetTimeoutError-->SendErrChannel

ChkPendingPodsLength--No-->SendNilChannel
end

SendNilChannel["returnCh <- nil"]
SendErrChannel["returnCh <- err"]
NilReturn(("return"))


```

#### isMisconfiguredPdb

TODO: Discuss/Elaborate on why this is considered misconfigured.
```go
func isMisconfiguredPdbV1(pdb *policyv1.PodDisruptionBudget) bool {
	if pdb.ObjectMeta.Generation != pdb.Status.ObservedGeneration {
		return false
	}

	return pdb.Status.ExpectedPods > 0 && 
        pdb.Status.CurrentHealthy >= pdb.Status.ExpectedPods
        && pdb.Status.DisruptionsAllowed == 0
}
```

### drain.Options.evictPodsWithPv

```go
func (o *Options) evictPodsWithPv(ctx context.Context, 
    attemptEvict bool, 
    pods []*corev1.Pod,
	policyGroupVersion string,
	getPodFn func(namespace, name string) (*corev1.Pod, error),
	returnCh chan error)
```
NOTE
- See [drain.Options.evictPodsWithPv](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/drain/drain.go#L580)
- This method basically delegates to `o.evictPodsWithPVInternal` with retry handling
- TODO: Logic of this method can do with some refactoring!

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

Begin((" "))
-->SortPods["
    sortPodsByPriority(pods)
    //Desc priority: pods[i].Spec.Priority > *pods[j].Spec.Priority
"]
-->DoVolumeAccounting["
	podVolumeInfoMap := o.doAccountingOfPvs(ctx, pods)
"]
-->ChkAttemptEvict{attemptEvict ?}

ChkAttemptEvict--Yes-->RetryTillLimit["
	until MaxEvictRetries
"]
-->
InvokeHelper["
	remainingPods, aborted = o.evictPodsWithPVInternal(ctx, attemptEvict, pods, podVolumeInfoMap, policyGroupVersion,  returnCh)
"]
InvokeHelper-->ChkAbort{"
	aborted ||
	len(remainingPods) == 0
"}
ChkAbort--Yes-->RangeRemainingPods
ChkAbort--No-->Sleep["
	pods = remainingPods
	time.Sleep(PodEvictionRetryInterval)
"]
Sleep--loop-->RetryTillLimit

RetryTillLimit--loopend-->ChkRemaining{"len(remainingPods) > 0 && !aborted ?"}
ChkRemaining--Yes-->InvokeHelper1["
// force delete pods
	remainingPods, _ = o.evictPodsWithPVInternal(ctx, false, pods, podVolumeInfoMap, policyGroupVersion, getPodFn, returnCh)
"]

ChkAttemptEvict--No-->InvokeHelper1
InvokeHelper1-->RangeRemainingPods
ChkRemaining--No-->RangeRemainingPods

RangeRemainingPods["pod := range remainingPods"]
RangeRemainingPods--aborted?-->SendNil["returnCh <- nil"]
RangeRemainingPods--attemptEvict?-->SendEvictErr["returnCh <- fmt.Errorf('pod evict error')"]
RangeRemainingPods--else-->SendDelErr["returnCh <- fmt.Errorf('pod delete error')"]


SendNil-->NilReturn
SendEvictErr-->NilReturn
SendDelErr-->NilReturn
NilReturn(("return"))
```



###	drain.Options.evictPodsWithPVInternal

FIXME: name case inconsistency with `evictPodsWithPv`

[drain.Options.evictPodsWithPVInternal](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/drain/drain.go#L646) is a drain helper method that actually evicts/deletes pods and waits for volume detachment. It returns a `remainingPods` slice and a `fastTrack` boolean is meant to abort the pod eviction and exit the calling go-routine. (TODO: should be called `abort` or even better should use custom error here)

```go
func (o *DrainOptions) evictPodsWithPVInternal(ctx context.Context,
    attemptEvict bool, 
    pods []*corev1.Pod, 
    volMap map[string][]string,
	policyGroupVersion string,
	returnCh chan error
    ) (remainingPods []*api.Pod, fastTrack bool)
```

1. Uses `context.Deadline`  passing in `ctx` and a deadline time after the drain timeout to get a sub-context assigned to `mainContext` and a `CancelFunc`. Defer the obtained `CancelFunc`. (So it always invoked when the method terminates)
2. Maintain a pod slice `retryPods` which is initially empty.
3. Iterate through `pods` slice with `i` as index variable 
   1. Apply a select with the one check case:
      1. Check to see if `mainContext` is closed/cancelled. Attempt to read from the Done channel: `<-mainContext.Done()`. If this case matches:
         1. Send `nil` on the return error channel: `returnCh <- nil`
         2. Compute `remainingPods` as `retryPods` slice appended with pods yet to be iterated: `pods[i+1:]...`
         3. Return `remainingPods, true`. (aborted is true)
   2. Initiate the pod eviction start time: `podEvictionStartTime=time.Now()`
   3. Call `volumeAttachmentHandler.AddWorker()` to start tracking `VolumeAttachments`and obtain a `volumeAttachmentEventCh` receive channel that one can use to receive the attached or detached `*.VolumeAttachment`. 
   4. If `attemptEvict` is true, then call [evictPod](#drainoptionsevictpod) else call [deletePod](#drainoptionsdeletepod) helper method. Grab the `err` for eviction/deletion.
   5. eviction/deletion had an error: Analyze the `err`:
      1. If both `attemptEvict` is true and ` apierrors.IsTooManyRequests(err)` is true, then this case is interpreted as an eviction failure due to PDB violation. 
         1. We get the [PodDisruptionBudget](../k8s_facilities.md#pod-disruption-budget) for the pod being iterated.
         2. We check whether it is misconfigured. IF So we send an error on `returnCh` and close the `volumeAttachmentEventCh` using `volumeAttachmentHandler.DeleteWorker` and continue with next loop iteration. ie go to next pod.
      2. If just `apierrors.IsNotFound(err)` is true, this means that Pod is already gone from the node. We send `nil` on `returnCh` and call `volumeAttachmentHandler.DeleteWorker(volumeAttachmentEventCh)` and continue with next pod in iteration.
      3. Otherwise we add the pod to the `retryPod` slice: `retryPods = append(retryPods, pod)`,  call `volumeAttachmentHandler.DeleteWorker(volumeAttachmentEventCh)` and continue with next pod in iteration.
      4. (NOTE: Error handling can be optimized. too much repetition)
   6. Log that the evict/delete was successful.
   7. Get the [PodVolumeInfo](#drainpodvolumeinfo) from `volMap` using the pod key. 
   8. Obtain a context and cancellation function for volume detachment using [context.Timeout](https://pkg.go.dev/context#WithTimeout) passing in the `mainContext` and detach timeout computed as the sum of the termination grace period of the pod (`pod.Spec.TerminationGracePeriodSeconds` if not nil) added to the `PvDetachTimeout` (from the drain options)
   9. Invoke [waitForDetach(ctx, podVolumeInfo, o.nodeName)](#drainoptionswaitfordetach) and grab the `err`.
   10. Invoke the cancel function for detach. NOTE: THIS IS NICHT GUT.  The sub context should be created INSIDE waitForDetach with a defer for the cancelllation.
   11. Analyze the detachment error.
       1.  If `apierrors.IsNotFound(err)` is true this indicates that the node is not found. 
           1.  Send `nil` on `returnCh`
           2.  Call `volumeAttachmentHandler.DeleteWorker(volumeAttachmentEventCh)`
           3.  Compute `remainingPods` as `retryPods` slice appended with pods yet to be iterated: `pods[i+1:]...` 
           4.  Return `remainingPods, true`. (aborted is true)
       2.  For other errors:
           1.  Send the `err` on the `returnCh`.
           2.  Call `volumeAttachmentHandler.DeleteWorker(volumeAttachmentEventCh)`
           3.  Continue with next pod in iteration. 
   12. Obtain a context and cancellation function for volume re-attachment using [context.Timeout](https://pkg.go.dev/context#WithTimeout) passing in the `mainContext` and `drain.Options.PvReattachTimeout`.
   13. Invoke [waitForReattach(ctx, podVolumeInfo, o.nodeName, volumeAttachmentEventCh)](#drainoptionswaitforreattach) and grab the returned `err`. 
   14. Invoke the cancel function for reattach.  NOTE: THIS IS NICHT GUT. The sub context should be created INSIDE `waitForReattach`.
   15. Analyze the re-attachment error.
       1.  If err is a reattachment timeout error just log a warning. TODO: Confused on why we don't return an error on the return channel here.
       2.  Otherwise we Send the `err` on the `returnCh`.
       3.  Call `volumeAttachmentHandler.DeleteWorker(volumeAttachmentEventCh)` and continue with next pod in iteration
   16. YAWN. Someone is very fond of calling this again and again. Call `volumeAttachmentHandler.DeleteWorker(volumeAttachmentEventCh)`
   17. Log the time taken for pod eviction+vol detachment+vol attachment to another node using `time.Since(podEvictionStartTime)`.
   18. Send `nil` on `returnCh`
4. pod iteration loop is done: `return retryPods, false`


