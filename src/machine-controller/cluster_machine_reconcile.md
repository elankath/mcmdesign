- [Cluster Machine Reconciliation](#cluster-machine-reconciliation)
  - [controller.triggerCreationFlow](#controllertriggercreationflow)
  - [controller.triggerDeletionFlow](#controllertriggerdeletionflow)
  - [controller.reconcileMachineHealth](#controllerreconcilemachinehealth)
    - [Health Check Flow Diagram](#health-check-flow-diagram)
    - [Health Check Summary](#health-check-summary)
    - [Health Check Doubts](#health-check-doubts)
  - [controller.triggerUpdationFlow](#controllertriggerupdationflow)

While perusing the below, you might need to reference [Machine Controller Helper Functions](./mc_helper_funcs.md)  as several reconcile functions delegate to helper methods defined on the machine controller struct.

# Cluster Machine Reconciliation

```go
func (c *controller) reconcileClusterMachineKey(key string) error
```
The top-level reconcile function for the machine that analyzes machine status and delegates to the individual reconcile functions for machine-creation, machine-deletion and machine-health-check flows. 


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

Begin((" "))-->A
A-->GetM
EnqM-->Z
LongR-->EnqM
ShortR-->EnqM
GetM-->ValM
ValM--Ok-->ValMC
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

## controller.triggerCreationFlow

[Controller Method](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/machinecontroller/machine.go#L326) that orchestraes the call to the [Driver.CreateMachine](../mcm_facilities.md#driver)

This method badly requires to be split into several functions. It is too long. 
```go
func (c *controller) triggerCreationFlow(ctx context.Context, 
cmr *driver.CreateMachineRequest) 
  (machineutils.RetryPeriod, error) 
```

Apologies for HUMONGOUS flow diagram - ideally code should have been split here into small functions.

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

ShortP["retryPeriod=machineutils.ShortRetry"]
MediumP["retryPeriod=machineutils.MediumRetry"]
Return(("return retryPeriod, err"))


Begin((" "))-->Init["
  machine     = cmr.Machine
	machineName = cmr.Machine.Name
  secretCopy := cmr.Secret.DeepCopy() //NOTE: seems Un-necessary?
"]
-->AddBootStrapToken["
 err = c.addBootstrapTokenToUserData(ctx, machine.Name, secretCopy)
//  get/create bootstrap token and populate inside secretCopy['userData']
"]
-->ChkErr{err != nil?}

ChkErr--Yes-->ShortP-->Return

ChkErr--No-->CreateMachineStatusReq["
  statusReq = & driver.GetMachineStatusRequest{
			Machine:      machine,
			MachineClass: cmr.MachineClass,
			Secret:       cmr.Secret,
		},
"]-->GetMachineStatus["
  statusResp, err := c.driver.GetMachineStatus(ctx, statusReq)
  //check if VM already exists
"]-->ChkStatusErr{err!=nil}

ChkStatusErr--No-->InitNodeNameFromStatusResp["
   nodeName = statusResp.NodeName
  providerID = statusResp.ProviderID
"]

ChkStatusErr--Yes-->DecodeErrorStatus["
  errStatus,decodeOk= status.FromError(err)
"]
DecodeErrorStatus-->CheckDecodeOk{"decodeOk ?"}

CheckDecodeOk--No-->MediumP-->Return
CheckDecodeOk--Yes-->AnalyzeCode{status.Code?}


AnalyzeCode--NotFound,Unimplemented-->ChkNodeLabel{"machine.Labels['node']?"}

ChkNodeLabel--No-->CreateMachine["
// node label is not present -> no machine
 resp, err := c.driver.CreateMachine(ctx, cmr)
"]-->ChkCreateError{err!=nil?}

ChkNodeLabel--Yes-->InitNodeNameFromMachine["
  nodeName = machine.Labels['node']
"]


AnalyzeCode--Unknown,DeadlineExceeded,Aborted,Unavailable-->ShortRetry["
retryPeriod=machineutils.ShortRetry
"]-->GetLastKnownState["
  lastKnownState := machine.Status.LastKnownState
"]-->InitFailedOp["
 lastOp := LastOperation{
    Description: err.Error(),
    State: MachineStateFailed,
    Type: MachineOperationCreatea,
    LastUpdateTime: Now(),
 };
 currStatus := CurrentStatus {
    Phase: MachineCrashLoopBackOff || MachineFailed (on create timeout)
    LastUpdateTime: Now()
 }
"]-->UpdateMachineStatus["
c.machineStatusUpdate(ctx,machine,lastOp,currStatus,lastKnownState)
"]-->Return


ChkCreateError--Yes-->SetLastKnownState["
  	lastKnownState = resp.LastKnownState
"]-->InitFailedOp

ChkCreateError--No-->InitNodeNameFromCreateResponse["
  nodeName = resp.NodeName
  providerID = resp.ProviderID
"]-->ChkStaleNode{"
// check stale node
nodeName != machineName 
&& nodeLister.Get(nodeName) exists"}


InitNodeNameFromStatusResp-->ChkNodeLabelAnnotPresent{"
cmr.Machine.Labels['node']
&& cmr.Machine.Annotations[MachinePriority] ?
"}
InitNodeNameFromMachine-->ChkNodeLabelAnnotPresent

ChkNodeLabelAnnotPresent--No-->CloneMachine["
  clone := machine.DeepCopy;
  clone.Labels['node'] = nodeName
  clone.Annotations[machineutils.MachinePriority] = '3'
  clone.Spec.ProviderID = providerID
"]-->UpdateMachine["
  _, err := c.controlMachineClient.Machines(clone.Namespace).Update(ctx, clone, UpdateOptions{})
"]-->ShortP




ChkStaleNode--No-->CloneMachine
ChkStaleNode--Yes-->CreateDMR["
  dmr := &driver.DeleteMachineRequest{
						Machine: &Machine{
							ObjectMeta: machine.ObjectMeta,
							Spec: MachineSpec{
								ProviderID: providerID,
							},
						},
						MachineClass: createMachineRequest.MachineClass,
						Secret:       secretCopy,
					}
"]-->DeleteMachine["
  _, err := c.driver.DeleteMachine(ctx, deleteMachineRequest)
  // discuss stale node case
  retryPeriod=machineutils.ShortRetry
"]-->InitFailedOp1["
 lastOp := LastOperation{
    Description: 'VM using old node obj',
    State: MachineStateFailed,
    Type: MachineOperationCreate, //seems wrong
    LastUpdateTime: Now(),
 };
 currStatus := CurrentStatus {
    Phase: MachineFailed (on create timeout)
    LastUpdateTime: Now()
 }
"]-->UpdateMachineStatus

ChkNodeLabelAnnotPresent--Yes-->ChkMachineStatus{"machine.Status.Node != nodeName
  || machine.Status.CurrentStatus.Phase == ''"}

ChkMachineStatus--No-->LongP["retryPeriod = machineutils.LongRetry"]-->Return

ChkMachineStatus--Yes-->CloneMachine1["
  clone := machine.DeepCopy()
  clone.Status.Node = nodeName
"]-->SetLastOp["
 lastOp := LastOperation{
    Description: 'Creating Machine on Provider',
    State: MachineStateProcessing,
    Type: MachineOperationCreate,
    LastUpdateTime: Now(),
 };
 currStatus := CurrentStatus {
    Phase: MachinePending,
    TimeoutActive:  true,
    LastUpdateTime: Now()
 }
 lastKnownState = clone.Status.LastKnownState
"]-->UpdateMachineStatus

style InitFailedOp text-align:left

```


## controller.triggerDeletionFlow

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
DeleteMachineFin["retryPeriod=c.deleteMachineFinalizers(machine)"]
SetMachineTermStatus["c.setMachineTerminationStatus(dmr)"]

CreateMachineStatusRequest["statusReq=&driver.GetMachineStatusRequest{machine, machineClass,secret}"]
GetVMStatus["retryPeriod=c.getVMStatus(statusReq)"]



Z(("End"))

Begin((" "))-->HasFin
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


## controller.reconcileMachineHealth

[controller.reconcileMachineHealth](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/machinecontroller/machine_util.go#L584) reconciles the machine object with any change in node conditions or VM health.

```go
func (c *controller) reconcileMachineHealth(ctx context.Context, machine *Machine) 
  (machineutils.RetryPeriod, error)
```

NOTES:
1. Reference [controller.isHealth](./mc_helper_methods.md#controllerishealthy) which checks the machine status conditions.

### Health Check Flow Diagram

See [What are the different phases of a Machine](https://github.com/gardener/machine-controller-manager/blob/master/docs/FAQ.md#what-are-the-different-phases-of-a-machine)

### Health Check Summary

1. Gets the `Node` obj associated with the machine. If it IS NOT found, yet the current machine phase is `Running`, change the machine phase to `Unknown`, the last operation state to `Processing`, the last operation type to `HealthCheck`, update the machine status and return with a short retry.  
2. If the `Node` object IS found, then it checks whether the `Machine.Status.Conditions` are different from `Node.Status.Conditions`. If so it sets the machine conditions to the node conditions.
3.  If the machine IS NOT healthy (See [isHealthy](./mc_helper_methods.md#controllerishealthy)) but the current machine phase is `Running`, change the machine phase to `Unknown`, the last operation state to `Processing`, the last operation type to `HealthCheck`, update the machine status and return with a short retry.  
4. If the machine IS healthy but the current machine phase is NOT `Running` and the machine's node does not have the `node.gardener.cloud/critical-components-not-ready` taint,  check whether the last operation type was a `Create`.
   1.  If the last operation type was a `Create` and last operation state is NOT marked as `Successful`, then delete the bootstrap token associated with the machine. Change the last operation state to `Successful`. Let the last operation type continue to remain as `Create`.
   1. If the last operation type was NOT a `Create`, change the last operation type to `HealthCheck`
   1. Change the machine phase to `Running` and update the machine status and return with a short retry.
   2. (The above 2 cases take care of a newly created machine and a machine that became OK after ome temporary issue)
5. If the current machine phase is `Pending` (ie machine being created: see `triggerCreationFlow`) get the configured machine creation timeout and check.  
   1. If the timoeut HAS NOT expired, enqueue the machine key on the machine work queue after 1m. 
   2. If the timeout HAS expired, then change the last operation state to `Failed` and the machine phase to `Failed`. Update the machine status and return with a short retry.
6. If the current machine phase is `Unknown`, get the effective machine health timeout and check. 
   1. If the timeout HAS NOT expired, enqueue the machine key on the machine work queue after 1m. 
   2. If the timeout HAS expired 
      1. Get the machine deployment name `machineDeployName := machine.Labels['name']` corresponding to this machine
      2. Register ONE permit with this with `machineDeployName`. See [Permit Giver](../mcm_facilities.md#permitspermitgiver). Q: Background of this change ? Couldn't we find a better way to throttle via work-queues instead of complicated `PermitGiver` and go-routines? Even simple lock would be OK here right ? 
      3. Attempt to get ONE permit for `machineDeployName` using a `lockAcquireTimeout` of 1s
         1. Throttle to check whether machine CAN be marked as `Failed` using `markable, err := controller.canMarkMachineFailed`. 
         2. If machine can be marked, change the last operation state (ie the health check) to `Failed`, preserve the last operation type, change machine phase to `Failed`. Update the machine status. See `c.updateMachineToFailedState`
         3. Then use `wait.Poll` using 100ms as `pollInterval` and 1s as `cacheUpdateTimeout` using the following poll condition function:
            1. Get the `machine` from the `machineLister` (which uses the cache of the shared informer)
            2. Return true if `machine.Status.CurrentStatus.Phase` is `Failed` or `Terminating` or the `machine` is not found
            3. Return false otherwise.

### Health Check Doubts

1. TODO: Why don't we check the machine health using the `Driver.GetMachineStatus` in the reconcile Machine health ? (seems like something obvious to do and would have helped in those meltdown issues where machine was incorrectly marked as failed)
1. TODO: why doesn't this code make use of the helper method: `c.machineStatusUpdate` ?
1. TODO: Unclear why `LastOperation.Description` does not use/concatenate one of the predefined constants in `machineutils`
2. TODO: code makes too much use of `cloneDirty` to check whether machine clone obj has changed, when it could easily return early in several branches.
3. TODO: Code directly makes calls to enqueue machine keys on the machine queue and still returns retry periods to caller leanding to un-necessary enqueue of machine keys. (spurious design)


## controller.triggerUpdationFlow

Doesn't seem to be used ? Possibly dead code ?