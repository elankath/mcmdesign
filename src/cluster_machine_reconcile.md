- [Cluster Machine Reconciliation](#cluster-machine-reconciliation)
  - [controller.reconcileClusterMachineKey](#controllerreconcileclustermachinekey)
    - [triggerDeletionFlow](#triggerdeletionflow)
  - [controller.reconcileMachineHealth](#controllerreconcilemachinehealth)
  - [controller.syncMachineNodeTemplates](#controllersyncmachinenodetemplates)

# Cluster Machine Reconciliation

##  controller.reconcileClusterMachineKey

The primary reconcile function for the machine that analyzes machine status and triggers lifecalls calls to the driver. TODO: Provide descriptive summary

```go
func (c *controller) reconcileClusterMachineKey(key string) error
```

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

A["ns,name=cache.SplitMetaNamespaceKey(mKey)"]
GetM["machine=machineLister.Machines(ns).Get(name)"]
ValM["validation.ValidateMachine(machine)"]
ValMC["machineClz,secretData,err=validation.ValidateMachineClass(machine)"]
LongR["retryPeriod=machineUtils.LongRetry"]
ShortR["retryPeriod=machineUtils.ShortRetry"]
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
### triggerDeletionFlow

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
LongR1["retryPeriod=machineUtils.LongRetry"]
ShortR["retryPeriod=machineUtils.ShortRetry"]
ChkMachineTerm{"machine.Status.CurrentStatus.Phase\n==MachineTerminating ?"}
CheckMachineOperation{"Check\nmachine.Status.LastOperation.Description"}
DrainNode["retryPeriod=c.drainNode(dmr)"]
DeleteVM["retryPeriod=c.deleteVM(dmr)"]
DeleteNode["retryPeriod=c.deleteNodeObject(dmr)"]
DeleteMachineFin["retryPeriod=c.deleteMachineFinalizers(machine)\n(dead code?)"]
SetMachineTermStatus["c.setMachineTerminationStatus(dmr)"]

CreateMachineStatusRequest["statusReq=&driver.GetMachineStatusRequest{machine, machineClass,secret}"]
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

CreateDecodeFailedOp["op:=LastOperation{Description: 'FailedDecode'
State: v1alpha1.MachineStateProcessing,
Type: v1alpha1.MachineOperationDelete,
Time: time.Now()}"]

CreaterRetryVMStatusOp["op:=LastOperation{Description: machineutils.GetVMStatus,
State: v1alpha1.MachineStateFailed,
Type:  v1alpha1.MachineOperationDelete,
Time: time.Now()}"]

ShortR1["retryPeriod=machineUtils.ShortRetry"]
MachineStatusUpdate["c.machineStatusUpdate(machine,op,machine.Status.CurrentStatus, machine.Status.LastKnownState)"]

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
DrainNode-->Z
CreateMachineStatusRequest-->GetMachineStatus
GetMachineStatus-->ChkMachineExists

ChkMachineExists--Yes-->CreateDrainOp
ChkMachineExists--No-->DecodeErrorStatus
DecodeErrorStatus-->CheckDecodeOk
CheckDecodeOk--Yes-->CheckErrStatusCode
CheckDecodeOk--No-->CreateDecodeFailedOp
CreateDecodeFailedOp-->LongR1
CheckErrStatusCode--"Unimplemented"-->CreateDrainOp
CheckErrStatusCode--"Unknown|DeadlineExceeded|Aborted|Unavailable"-->CreaterRetryVMStatusOp
CheckErrStatusCode--"NotFound"-->CreateNodeDelOp
CreaterRetryVMStatusOp-->ShortR1

CreateDrainOp-->ShortR1
CreateNodeDelOp-->ShortR1
ShortR1-->MachineStatusUpdate
LongR1-->MachineStatusUpdate
MachineStatusUpdate-->Z


ShortR-->Z

```

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