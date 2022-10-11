- [Cluster Machine Reconcile](#cluster-machine-reconcile)
  - [triggerDeletionFlow](#triggerdeletionflow)
    - [controller.getVMStatus](#controllergetvmstatus)
    - [controller.drainNode](#controllerdrainnode)
  - [controller.reconcileMachineHealth](#controllerreconcilemachinehealth)
  - [controller.syncMachineNodeTemplates](#controllersyncmachinenodetemplates)

While perusing the below, you might need to reference [Machine Controller Helper Functions](./mc_helper_funcs.md)  as several reconcile functions delegate to helper methods defined on the machine controller struct.

# Cluster Machine Reconcile 

```go
func (c *controller) reconcileClusterMachineKey(key string) error
```
The top-level reconcile function for the machine that analyzes machine status and delegates to the reconcile functions for creation and deletion flows. TODO: Provide descriptive summary


```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

A["ns,name=cache.SplitMetaNamespaceKey(mKey)"]
GetM["machine=machineLister.Machines(ns).Get(name)"]
ValM["validation.ValidateMachine(machine)"]
ValMC["machineClz,secretData,err=validation.ValidateMachineClass(machine)"]
LongR["retryPeriod=machineutils.LongRetry"]
ShortR["retryPeriod=machineutils.ShortRetry"]
EnqM["machineQueue.AddAfter(mKey, retryPeriod)"]
CheckMDel{"Is\nmachine.DeletionTimestamp\nSet?"}
NewDelReq["req=&driver.DeleteMachineRequest{machine,machineClz,secretData}"]
DelFlow["retryPeriod=controller.triggerDeletionFlow(req)"]
CreateFlow["retryPeriod=controller.triggerCreationFlow(req)"]
HasFin{"HasFinalizer(machine)"}
AddFin["addMachineFinalizers(machine)"]
CheckMachineNodeExists{"machine.Status.Node\nExists?"}
ReconcileMachineHealth["controller.reconcileMachineHealth(machine)"]
SyncNodeTemplates["controller.syncNodeTemplates(machine)"]
NewCreateReq["req=&driver.CreateMachineRequest{machine,machineClz,secretData}"]
Z(("End"))

A-->GetM
EnqM-->Z
LongR-->EnqM
ShortR-->EnqM
GetM-->ValM
ValM-->Ok-->ValMC
ValM--Err-->LongR
ValMC--Err-->LongR
ValMC--Ok-->CheckMDel
CheckMDel--Yes-->NewDelReq
CheckMDel--No-->HasFin
NewDelReq-->DelFlow
HasFin--No-->AddFin
HasFin--Yes-->ShortR
AddFin-->CheckMachineNodeExists
CheckMachineNodeExists--Yes-->ReconcileMachineHealth
CheckMachineNodeExists--No-->NewCreateReq
ReconcileMachineHealth--Ok-->SyncNodeTemplates
SyncNodeTemplates--Ok-->LongR
SyncNodeTemplates--Err-->ShortR
DelFlow-->EnqM
NewCreateReq-->CreateFlow
CreateFlow-->EnqM

```
## triggerDeletionFlow

```go
func (c *controller) triggerDeletionFlow(ctx context.Context, dmr *driver.DeleteMachineRequest) (machineutils.RetryPeriod, error) 

```
Please note that there is sad use of `machine.Status.LastOperation`  as semantically the _next_ requested operation. This is confusing. TODO: DIscuss This.

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

GM["machine=dmr.Machine\n
machineClass=dmr.MachineClass\n
secret=dmr.Secret"]
HasFin{"HasFinalizer(machine)"}
LongR["retryPeriod=machineUtils.LongRetry"]
ShortR["retryPeriod=machineUtils.ShortRetry"]
ChkMachineTerm{"machine.Status.CurrentStatus.Phase\n==MachineTerminating ?"}
CheckMachineOperation{"Check\nmachine.Status.LastOperation.Description"}
DrainNode["retryPeriod=c.drainNode(dmr)"]
DeleteVM["retryPeriod=c.deleteVM(dmr)"]
DeleteNode["retryPeriod=c.deleteNodeObject(dmr)"]
DeleteMachineFin["retryPeriod=c.deleteMachineFinalizers(machine)\n(dead code?)"]
SetMachineTermStatus["c.setMachineTerminationStatus(dmr)"]

CreateMachineStatusRequest["statusReq=&driver.GetMachineStatusRequest{machine, machineClass,secret}"]
GetVMStatus["retryPeriod=c.getVMStatus(statusReq)"]



Z(("End"))

HasFin--Yes-->GM
HasFin--No-->LongR
LongR-->Z
GM-->ChkMachineTerm
ChkMachineTerm--No-->SetMachineTermStatus
ChkMachineTerm--Yes-->CheckMachineOperation
SetMachineTermStatus-->ShortR
CheckMachineOperation--GetVMStatus-->CreateMachineStatusRequest
CheckMachineOperation--InitiateDrain-->DrainNode
CheckMachineOperation--InitiateVMDeletion-->DeleteVM
CheckMachineOperation--InitiateNodeDeletion-->DeleteNode
CheckMachineOperation--InitiateFinalizerRemoval-->DeleteMachineFin
CreateMachineStatusRequest-->GetVMStatus
GetVMStatus-->Z


DrainNode-->Z
DeleteVM-->Z
DeleteNode-->Z
DeleteMachineFin-->Z
ShortR-->Z

```

### controller.getVMStatus
(BAD NAME FOR METHOD: should be called `checkMachineExistenceAndEnqueNextOperation`)

```go
func (c *controller) getVMStatus(ctx context.Context, 
    statusReq *driver.GetMachineStatusRequest) (machineutils.RetryPeriod, error)
```

This method is only called for the delete flow. 
1. It attempts to get the machine status
1. If the machine exists, it updates the machine status operation to `InitiateDrain` and returns a `ShortRetry` for the machine work queue. 
1. If attempt to get machine status failed, it will obtain the error code from the error.
   1. If decoding the error code failed, it will update the  machine status operation to `machineutils.GetVMStatus`returns a `LongRetry` for the machine work queue. 
      1. Unsure how we get out of this Loop. TODO: Discuss this. Is this dead code?
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

CreaterRetryVMStatusOp["op:=LastOperation{Description: machineutils.GetVMStatus,
State: v1alpha1.MachineStateFailed,
Type:  v1alpha1.MachineOperationDelete,
Time: time.Now()}"]

ShortR["retryPeriod=machineUtils.ShortRetry"]
LongR["retryPeriod=machineUtils.LongRetry"]
MachineStatusUpdate["c.machineStatusUpdate(machine,op,machine.Status.CurrentStatus, machine.Status.LastKnownState)"]

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
ShortR-->MachineStatusUpdate
LongR-->MachineStatusUpdate
MachineStatusUpdate-->Z
```

### controller.drainNode

Inside `pkg/util/provider/machinecontroller/machine_util.go`
```go
func (c *controller) drainNode(ctx context.Context, dmr *driver.DeleteMachineRequest) (machineutils.RetryPeriod, error)
```


```mermaid

%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

Initialize["machine = dmr.Machine
nodeName= machine.Labels['node']
drainTimeout=machine.Spec.MachineConfiguration.MachineDrainTimeout || c.safetyOptions.MachineDrainTimeout
forceDeleteLabelPresent = machine.Labels['force-deletion'] == 'True'
skipDrain = false"]
-->GetNodeReadyCond["nodeReadyCond = machine.Status.Conditions contains k8s.io/api/core/v1/NodeReady
readOnlyFSCond=machine.Status.Conditions contains 'ReadonlyFilesystem' 
"]
-->ChkNodeNotReady["skipDrain = (nodeReadyCond.Status == ConditionFalse) && nodeReadyCondition.LastTransitionTime.Time > 5m"]
-->ChkReadOnlyFS["skipDrain = (readOnlyFSCond.Status == ConditionTrue) && readOnlyFSCond.LastTransitionTime.Time > 5m"]
-->ChkSkipDrain1{"skipDrain true?"}
ChkSkipDrain1--No-->
ChkSkipDrain1--Yes-->SetOpState["opState=MachineStateProcessing"]
Z(("End"))
```

Note on above
1. We skip the drain if node is set to ReadonlyFilesystem for over 5 minutes
   1. Check TODO:  `ReadonlyFilesystem` is a MCM condition and not a k8s core node condition. Not sure if we are mis-using this field. TODO: Check this.
2. Check TODO: Why do we check that node is not ready for 5m in order to skip the drain ? Shouldn't we skip the drain if node is simply not ready ? Why wait for 5m here ?

btmp
```
```
etmp

## controller.reconcileMachineHealth

```go
func (c *controller) reconcileMachineHealth(ctx context.Context, machine *v1alpha1.Machine) (machineutils.RetryPeriod, error)
```
TODO: illustrate me

## controller.syncMachineNodeTemplates

```go
func (c *controller) syncMachineNodeTemplates(ctx context.Context, machine *v1alpha1.Machine) (machineutils.RetryPeriod, error) 
```
TODO: illustrate me