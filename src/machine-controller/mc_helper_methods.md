- [Machine Controller Helper Methods](#machine-controller-helper-methods)
	- [controller.addBootstrapTokenToUserData](#controlleraddbootstraptokentouserdata)
	- [controller.addMachineFinalizers](#controlleraddmachinefinalizers)
	- [controller.setMachineTerminationStatus](#controllersetmachineterminationstatus)
	- [controller.machineStatusUpdate](#controllermachinestatusupdate)
	- [controller.UpdateNodeTerminationCondition](#controllerupdatenodeterminationcondition)
	- [controller.isHealthy](#controllerishealthy)
	- [controller.getVMStatus](#controllergetvmstatus)
	- [controller.drainNode](#controllerdrainnode)
	- [controller.deleteVM](#controllerdeletevm)
	- [controller.deleteNodeObject](#controllerdeletenodeobject)
# Machine Controller Helper Methods

## controller.addBootstrapTokenToUserData


This method is responsible for adding the bootstrap token for the machine.

Bootstrap tokens are used when joining new nodes to a cluster. Bootstrap Tokens are defined with a specific `SecretType`: `bootstrap.kubernetes.io/token` and live in the `kube-system` namespace. These Secrets are then read by the Bootstrap Authenticator in the API Server


Reference
- [Bootstrap Tokens](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/)
- [Bootstrap Token Secrets](https://github.com/kubernetes/design-proposals-archive/blob/main/cluster-lifecycle/bootstrap-discovery.md#new-bootstrap-token-secrets)
- [Bootstrap Token Structure](https://github.com/kubernetes/design-proposals-archive/blob/main/cluster-lifecycle/bootstrap-discovery.md#new-bootstrap-token-structure)


```go
func (c *controller) addBootstrapTokenToUserData(ctx context.Context, machineName string, secret *corev1.Secret) error 

```

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

Begin((" "))
-->InitTokenSecret["
tokenID := hex.EncodeToString([]byte(machineName)[len(machineName)-5:])[:6]
// 6 chars length
secretName :='bootstrap-token-' + tokenID
"]
-->GetSecret["
	secret, err = c.targetCoreClient.CoreV1().Secrets('kube-system').Get(ctx, secretName, GetOptions{}) 
"]
-->ChkErr{err!=nil?}

ChkErr--Yes-->ChkNotFound{"IsNotFound(err)"}

ChkNotFound--Yes-->GenToken["
  tokenSecretKey = generateRandomStrOf16Chars
"]-->InitSecretData["
  data := map[string][]byte{
    'token-id': []byte(tokenID),
    'token-secret': []byte(tokenSecretKey),
    'expiration': []byte(c.safetyOptions.MachineCreationTimeout.Duration)
    //..others
 }
"]
-->InitSecret["
  	secret = &corev1.Secret{
				ObjectMeta: metav1.ObjectMeta{
					Name:      secretName,
					Namespace: metav1.NamespaceSystem,
				},
				Type: 'bootstrap.kubernetes.io/token'
				Data: data,
			}
"]
-->CreateSecret["
secret, err =c.targetCoreClient.CoreV1().Secrets('kube-system').Create(ctx, secret, CreateOptions{})
"]
-->ChkErr1{err!=nil?}

ChkErr1--Yes-->ReturnErr(("return err"))
ChkNotFound--No-->ReturnErr

ChkErr1--No-->CreateToken["
token = tokenID + '.' + tokenSecretKey
"]-->InitUserData["
  userDataByes = secret.Data['userData']
  userDataStr = string(userDataBytes)
"]-->ReplaceUserData["
  	userDataS = strings.ReplaceAll(userDataS, 'BOOTSTRAP_TOKEN',placeholder, token)
   	secret.Data['userData'] = []byte(userDataS)
    //discuss this.
"]-->ReturnNil(("return nil"))

style InitSecretData text-align:left
style InitSecret text-align:left
```

## controller.addMachineFinalizers

This method checks for the `MCMFinalizer` Value: `machine.sapcloud.io/machine-controller-manager` and adds it if it is not present. It leverages `k8s.io/apimachinery/pkg/util/sets` package for its work.

This method is regularly called during machine reconciliation, if a machine does not have a deletion timestamp so that all non-deleted machines possess this finalizer.

```go
func (c *controller) addMachineFinalizers(ctx context.Context, machine *v1alpha1.Machine) (machineutils.RetryPeriod, error)
	if finalizers := sets.NewString(machine.Finalizers...); !finalizers.Has(MCMFinalizerName) {
		finalizers.Insert(MCMFinalizerName)
		clone := machine.DeepCopy()
		clone.Finalizers = finalizers.List()
		_, err := c.controlMachineClient.Machines(clone.Namespace).Update(ctx, clone, metav1.UpdateOptions{})
		if err != nil {
			// Keep retrying until update goes through
			klog.Errorf("Failed to add finalizers for machine %q: %s", machine.Name, err)
		} else {
			// Return error even when machine object is updated
			klog.V(2).Infof("Added finalizer to machine %q with providerID %q and backing node %q", machine.Name, getProviderID(machine), getNodeName(machine))
			err = fmt.Errorf("Machine creation in process. Machine finalizers are UPDATED")
		}
	}
	return machineutils.ShortRetry, err

```

##  controller.setMachineTerminationStatus

`setMachineTerminationStatus` set's the machine status to terminating. This is illustrated below. Please note that `Machine.Status.LastOperation` is set an instance of the `LastOperation` struct. (which at times appears to be a command for the next action? Discuss this.) 

```go
func (c *controller) setMachineTerminationStatus(ctx context.Context, dmr *driver.DeleteMachineRequest) (machineutils.RetryPeriod, error)  
```

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

CreateClone["clone := dmr.Machine.DeepCopy()"]
NewCurrStatus["currStatus := &v1alpha1.CurrentStatus{Phase:\n MachineTerminating, LastUpdateTime: time.Now()}"]
SetCurrentStatus["clone.Status.CurrentStatus = currStatus"]
UpdateStatus["c.controlMachineClient.Machines(ns).UpdateStatus(clone)"]
ShortR["retryPeriod=machineUtils.ShortRetry"]
Z(("Return"))

CreateClone-->NewCurrStatus
NewCurrStatus-->SetCurrentStatus
SetCurrentStatus-->UpdateStatus
UpdateStatus-->ShortR
ShortR-->Z
```


## controller.machineStatusUpdate

Updates `machine.Status.LastOperation`, `machine.Status.CurrentStatus` and `machine.Status.LastKnownState`

```go
func (c *controller) machineStatusUpdate(
	ctx context.Context,
	machine *v1alpha1.Machine,
	lastOperation v1alpha1.LastOperation,
	currentStatus v1alpha1.CurrentStatus,
	lastKnownState string) error 
```

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

CreateClone["clone := machine.DeepCopy()"]
-->InitClone["
	clone.Status.LastOperation = lastOperation
	clone.Status.CurrentStatus = currentStatus
	clone.Status.LastKnownState = lastKnownState
"]
-->ChkSimilarStatus{"isMachineStatusSimilar(
	clone.Status,
	machine.Status)"}

ChkSimilarStatus--No-->UpdateStatus["
	err:=c.controlMachineClient
	.Machines(clone.Namespace)
	.UpdateStatus(ctx, clone, metav1.UpdateOptions{})
"]
-->Z1(("return err"))
ChkSimilarStatus--Yes-->Z2(("return nil"))
```

NOTE: [isMachineStatusSimilar](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/machinecontroller/machine_util.go#L544) implementation is quite sad. TODO: we should improve stuff like this when we move to controller-runtime.

## controller.UpdateNodeTerminationCondition

[controller.UpdateNodeTerminationCondition](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/machinecontroller/machine_util.go#L1316) adds or updates the termination condition to the `Node.Status.Conditions` of the node object corresponding to the machine.

```go
func (c *controller) UpdateNodeTerminationCondition(ctx context.Context, machine *v1alpha1.Machine) error 
```

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

Init["
	nodeName := machine.Labels['node']
	newTermCond := v1.NodeCondition{
		Type:               machineutils.NodeTerminationCondition,
		Status:             v1.ConditionTrue,
		LastHeartbeatTime:  Now(),
		LastTransitionTime: Now()}"]
-->GetCond["oldTermCond, err := nodeops.GetNodeCondition(ctx, c.targetCoreClient, nodeName, machineutils.NodeTerminationCondition)"]
-->ChkIfErr{"err != nil ?"}
ChkIfErr--Yes-->ChkNotFound{"apierrors.IsNotFound(err)"}
ChkNotFound--Yes-->ReturnNil(("return nil"))
ChkNotFound--No-->ReturnErr(("return err"))
ChkIfErr--No-->ChkOldTermCondNotNil{"oldTermCond != nil
&& machine.Status.CurrentStatus.Phase 
== MachineTerminating ?"}

ChkOldTermCondNotNil--No-->ChkMachinePhase{"Check\nmachine\n.Status.CurrentStatus\n.Phase?"}
ChkMachinePhase--MachineFailed-->NodeUnhealthy["newTermCond.Reason = machineutils.NodeUnhealthy"]
ChkMachinePhase--"else"-->NodeScaleDown["newTermCond.Reason=machineutils.NodeScaledDown
//assumes scaledown..why?"]
NodeUnhealthy-->UpdateCondOnNode["err=nodeops.AddOrUpdateConditionsOnNode(ctx, c.targetCoreClient, nodeName, newTermCond)"]
NodeScaleDown-->UpdateCondOnNode


ChkOldTermCondNotNil--Yes-->CopyTermReasonAndMessage["
newTermCond.Reason=oldTermCond.Reason
newTermCond.Message=oldTermCond.Message
"]
CopyTermReasonAndMessage-->UpdateCondOnNode


UpdateCondOnNode-->ChkNotFound
```


## controller.isHealthy

Checks if machine is healty by checking its conditions.
```go
func (c *controller) isHealthy(machine *.Machine) bool 
```

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

Begin((" "))
-->Init["
	conditions = machine.Status.Conditions
	badTypes = strings.Split(
	 'KernelDeadlock,ReadonlyFilesystem,DiskPressure,NetworkUnavailable', 
		',')
"]-->ChkCondLen{"len(conditions)==0?"}

ChkCondLen--Yes-->ReturnF(("return false"))
ChkCondLen--No-->IterCond["c:= range conditions"]
IterCond-->ChkNodeReady{"c.Type=='Ready'
&& c.Status != 'True' ?"}--Yes-->ReturnF
ChkNodeReady
--Yes-->IterBadConditions["badType := range badTypes"]
-->ChkType{"badType == c.Type
&&
c.Status != 'False' ?"}
--Yes-->ReturnF

IterBadConditions--loop-->IterCond
ChkType--loop-->IterBadConditions




style Init text-align:left
```
NOTE
1. controller.NodeConditions should be called controller.BadConditionTypes
2. Iterate over `machine.Status.Conditions`
   1. If `Ready` condition inis not `True`, node is determined as un-healty.
   2. If any of the bad condition types are detected, then node is determine as un-healthy


## controller.getVMStatus
(BAD NAME FOR METHOD: should be called `checkMachineExistenceAndEnqueNextOperation`)

```go
func (c *controller) getVMStatus(ctx context.Context, 
    statusReq *driver.GetMachineStatusRequest) (machineutils.RetryPeriod, error)
```

This method is only called for the delete flow. 
1. It attempts to get the machine status
1. If the machine exists, it updates the machine status operation to `InitiateDrain` and returns a `ShortRetry` for the machine work queue. 
1. If attempt to get machine status failed, it will obtain the error code from the error.
   1.  For `Unimplemented`(ie `GetMachineStatus` op was is not implemented), it does the same as `2`. ie: it updates the machine status operation to `InitiateDrain` and returns a `ShortRetry` for the machine work queue. 
   1. If decoding the error code failed, it will update the  machine status operation to `machineutils.GetVMStatus` and returns a `LongRetry` for the machine key into the machine work queue. 
      1. Unsure how we get out of this Loop. TODO: Discuss this.
   2. For `Unknown|DeadlineExceeded|Aborted|Unavailable` it updates the machine status operation to `machineutils.GetVMStatus` status and returns a `ShortRetry` for the machine work queue.  (So that reconcile will run this method again in future)
   3. For `NotFound` code (ie machine is not found), it will enqueue node deletion by updating the machine stauts operation to `machineutils.InitiateNodeDeletion` and returning a `ShortRetry` for the machine work queue.


```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

GetMachineStatus["_,err=driver.GetMachineStatus(statusReq)"]
ChkMachineExists{"err==nil ?\n (ie machine exists)"}
DecodeErrorStatus["errStatus,decodeOk= status.FromError(err)"]
CheckDecodeOk{"decodeOk ?"}

CheckErrStatusCode{"Check errStatus.Code"}

CreateDrainOp["op:=LastOperation{Description: machineutils.InitiateDrain
State: v1alpha1.MachineStateProcessing,
Type: v1alpha1.MachineOperationDelete,
Time: time.Now()}"]

CreateNodeDelOp["op:=LastOperation{Description: machineutils.InitiateNodeDeletion
State: v1alpha1.MachineStateProcessing,
Type: v1alpha1.MachineOperationDelete,
Time: time.Now()}"]

CreateDecodeFailedOp["op:=LastOperation{Description: machineutils.GetVMStatus,
State: v1alpha1.MachineStateFailed,
Type: v1alpha1.MachineOperationDelete,
Time: time.Now()}"]

CreaterRetryVMStatusOp["op:=LastOperation{Description: ma1chineutils.GetVMStatus,
State: v1alpha1.MachineStateFailed,
Type:  v1alpha1.MachineOperationDelete,
Time: time.Now()}"]

ShortR["retryPeriod=machineUtils.ShortRetry"]
LongR["retryPeriod=machineUtils.LongRetry"]
UpdateMachineStatus["c.machineStatusUpdate(machine,op,machine.Status.CurrentStatus, machine.Status.LastKnownState)"]

Z(("End"))

GetMachineStatus-->ChkMachineExists

ChkMachineExists--Yes-->CreateDrainOp
ChkMachineExists--No-->DecodeErrorStatus
DecodeErrorStatus-->CheckDecodeOk
CheckDecodeOk--Yes-->CheckErrStatusCode
CheckDecodeOk--No-->CreateDecodeFailedOp
CreateDecodeFailedOp-->LongR
CheckErrStatusCode--"Unimplemented"-->CreateDrainOp
CheckErrStatusCode--"Unknown|DeadlineExceeded|Aborted|Unavailable"-->CreaterRetryVMStatusOp
CheckErrStatusCode--"NotFound"-->CreateNodeDelOp
CreaterRetryVMStatusOp-->ShortR

CreateDrainOp-->ShortR
CreateNodeDelOp-->ShortR
ShortR-->UpdateMachineStatus
LongR-->UpdateMachineStatus
UpdateMachineStatus-->Z
```

## controller.drainNode

Inside `pkg/util/provider/machinecontroller/machine_util.go`
```go
func (c *controller) drainNode(ctx context.Context, dmr *driver.DeleteMachineRequest) (machineutils.RetryPeriod, error)
```


```mermaid

%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD


Initialize["err = nil
machine = dmr.Machine
nodeName= machine.Labels['node']
drainTimeout=machine.Spec.MachineConfiguration.MachineDrainTimeout || c.safetyOptions.MachineDrainTimeout
maxEvictRetries=machine.Spec.MachineConfiguration.MaxEvictRetries || c.safetyOptions.MaxEvictRetries
skipDrain = false"]
-->GetNodeReadyCond["nodeReadyCond = machine.Status.Conditions contains k8s.io/api/core/v1/NodeReady
readOnlyFSCond=machine.Status.Conditions contains 'ReadonlyFilesystem' 
"]
-->ChkNodeNotReady["skipDrain = (nodeReadyCond.Status == ConditionFalse) && nodeReadyCondition.LastTransitionTime.Time > 5m
or (readOnlyFSCond.Status == ConditionTrue) && readOnlyFSCond.LastTransitionTime.Time > 5m
// discuss this
"]
-->ChkSkipDrain{"skipDrain true?"}
ChkSkipDrain--Yes-->SetOpStateProcessing
ChkSkipDrain--No-->SetHasDrainTimedOut["hasDrainTimedOut = time.Now() > machine.DeletionTimestamp + drainTimeout"]
SetHasDrainTimedOut-->ChkForceDelOrTimedOut{"machine.Labels['force-deletion']
  || hasDrainTimedOut"}

ChkForceDelOrTimedOut--Yes-->SetForceDelParams["
  forceDeletePods=true
  drainTimeout=1m
  maxEvictRetries=1
  "]
SetForceDelParams-->UpdateNodeTermCond["err=c.UpdateNodeTerminationCondition(ctx, machine)"]
ChkForceDelOrTimedOut--No-->UpdateNodeTermCond

UpdateNodeTermCond-->ChkUpdateErr{"err != nil ?"}
ChkUpdateErr--No-->InitDrainOpts["
  // params reduced for brevity
  drainOptions := drain.NewDrainOptions(
    c.targetCoreClient,
    drainTimeout,
    maxEvictRetries,
    c.safetyOptions.PvDetachTimeout.Duration,
    c.safetyOptions.PvReattachTimeout.Duration,
    nodeName,
    forceDeletePods,
    c.driver,
		c.pvcLister,
		c.pvLister,
    c.pdbV1Lister,
		c.nodeLister,
		c.volumeAttachmentHandler)
"]
ChkUpdateErr--"Yes&&forceDelPods"-->InitDrainOpts
ChkUpdateErr--Yes-->SetOpStateFailed["opstate = v1alpha1.MachineStateFailed
  description=machineutils.InitiateDrain
  //drain failed. retry next sync
  "]

InitDrainOpts-->RunDrain["err = drainOptions.RunDrain(ctx)"]
RunDrain-->ChkDrainErr{"err!=nil?"}
ChkDrainErr--No-->SetOpStateProcessing["
  opstate= v1alpha1.MachineStateProcessing
  description=machineutils.InitiateVMDeletion
// proceed with vm deletion"]
ChkDrainErr--"Yes && forceDeletePods"-->SetOpStateProcessing
ChkDrainErr--Yes-->SetOpStateFailed
SetOpStateProcessing-->
InitLastOp["lastOp:=v1alpha1.LastOperation{
			Description:    description,
			State:          state,
			Type:           v1alpha1.MachineOperationDelete,
			LastUpdateTime: metav1.Now(),
		}
  //lastOp is actually the *next* op semantically"]
SetOpStateFailed-->InitLastOp
InitLastOp-->UpdateMachineStatus["c.machineStatusUpdate(ctx,machine,lastOp,machine.Status.CurrentStatus,machine.Status.LastKnownState)"]
-->Return(("machineutils.ShortRetry, err"))
```

Note on above
1. We skip the drain if node is set to ReadonlyFilesystem for over 5 minutes
   1. Check TODO:  `ReadonlyFilesystem` is a MCM condition and not a k8s core node condition. Not sure if we are mis-using this field. TODO: Check this.
2. Check TODO: Why do we check that node is not ready for 5m in order to skip the drain ? Shouldn't we skip the drain if node is simply not ready ? Why wait for 5m here ?/
3. See [Run Drain](./node_drain.md#run-drain)

## controller.deleteVM

Called by `controller.triggerDeletionFlow`

```go
func (c *controller) deleteVM(ctx context.Context, dmReq *driver.DeleteMachineRequest) 
	(machineutils.RetryPeriod, error)

```

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

Begin((" "))
-->Init["
machine = dmr.Machine
"]
-->CallDeleteMachine["
dmResp, err := c.driver.DeleteMachine(ctx, dmReq)
"]
-->ChkDelErr{"err!=nil?"}

ChkDelErr--No-->SetSuccShortRetry["
retryPeriod = machineutils.ShortRetry
description = 'VM deletion was successful.'+ machineutils.InitiateNodeDeletion)
state = MachineStateProcessing
"]
-->InitLastOp["
lastOp := LastOperation{
Description:    description,
State:          state,
Type:           MachineOperationDelete,
LastUpdateTime: Now(),
},
"]
-->SetLastKnownState["
// useless since drivers impls dont set this?. 
// Use machine.Status.LastKnownState instead ?
lastKnownState = dmResp.LastKnownState
"]
-->UpdateMachineStatus["
//Discuss: Introduce finer grained phase for status ?
c.machineStatusUpdate(ctx,machine,lastOp,machine.Status.CurrentStatus,lastKnownState)"]-->Return(("retryPeriod, err"))

ChkDelErr--Yes-->DecodeErrorStatus["
  errStatus,decodeOk= status.FromError(err)
"]
DecodeErrorStatus-->CheckDecodeOk{"decodeOk ?"}
CheckDecodeOk--No-->SetFailed["
	state = MachineStateFailed
	description = 'machine decode error' + machineutils.InitiateVMDeletion
"]-->SetLongRetry["
retryPeriod= machineutils.LongRetry
"]-->InitLastOp

CheckDecodeOk--Yes-->AnalyzeCode{status.Code?}
AnalyzeCode--Unknown, DeadlineExceeded,Aborted,Unavailable-->SetDelFailed["
state = MachineStateFailed
description = VM deletion failed due to
+ err + machineutils.InitiateVMDeletion"]
-->SetShortRetry["
retryPeriod= machineutils.ShortRetry
"]-->InitLastOp

AnalyzeCode--NotFound-->DelSuccess["
// we can proceed with deleting node.
description = 'VM not found. Continuing deletion flow'+ machineutils.InitiateNodeDeletion
state = MachineStateProcessing
"]-->SetShortRetry

AnalyzeCode--default-->DelUnknown["
state = MachineStateFailed
description='VM deletion failed due to' + err 
+ 'Aborting..' + machineutils.InitiateVMDeletion
"]-->SetLongRetry

```

## controller.deleteNodeObject
NOTE: Should have just called this `controller.deleteNode` for naming consistency with other methods.

Called by `triggerDeletionFlow` after successfully deleting the VM.
```go
func (c *controller) deleteNodeObject(ctx context.Context, 
machine *v1alpha1.Machine) 
	(machineutils.RetryPeriod, error) 
```

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

Begin((" "))
-->Init["
nodeName := machine.Labels['node']
"]
-->ChkNodeName{"nodeName != ''" ?}
ChkNodeName--Yes-->DelNode["
err = c.targetCoreClient.CoreV1().Nodes().Delete(ctx, nodeName, metav1.DeleteOptions{})
"]-->ChkDelErr{"err!=nil?"}

ChkNodeName--No-->NodeObjNotFound["
	state = MachineStateProcessing
	description = 'No node object found for' + nodeName + 'Continue'
	+ machineutils.InitiateFinalizerRemoval
"]-->InitLastOp

ChkDelErr--No-->DelSuccess["
	state = MachineStateProcessing
	description = 'Deletion of Node' + nodeName + 'successful'
	 + machineutils.InitiateFinalizerRemoval 
"]-->InitLastOp

ChkDelErr--Yes&&!apierrorsIsNotFound-->FailedNodeDel["
	state = MachineStateFailed
	description = 'Deletion of Node' + 
		nodeName + 'failed due to' + err + machineutils.InitiateNodeDeletion
"]
-->InitLastOp["
lastOp := LastOperation{
Description:    description,
State:          state,
Type:           MachineOperationDelete,
LastUpdateTime: Now(),
},
"]
-->UpdateMachineStatus["
c.machineStatusUpdate(ctx,machine,lastOp,machine.Status.CurrentStatus,machine.Status.LastKnownState)"]
-->Return(("machineutils.ShortRetry, err"))
```
