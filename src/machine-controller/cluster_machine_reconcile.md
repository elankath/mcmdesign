- [Cluster Machine Reconciliation](#cluster-machine-reconciliation)
  - [controller.triggerCreationFlow](#controllertriggercreationflow)
    - [controller.addBootstrapTokenToUserData](#controlleraddbootstraptokentouserdata)
  - [controller.triggerDeletionFlow](#controllertriggerdeletionflow)
    - [controller.getVMStatus](#controllergetvmstatus)
    - [controller.drainNode](#controllerdrainnode)
  - [controller.reconcileMachineHealth](#controllerreconcilemachinehealth)
    - [Health Check FLow Diagram](#health-check-flow-diagram)
    - [Health Check Summary](#health-check-summary)
    - [Health Check Doubts](#health-check-doubts)

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

Apologies for HUMONGOUS flow diagram - all this is in one method - will split into several sections later for clarity!
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
"]-->ShortRetry["retryPeriod=machineutils.ShortRetry"]

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

### controller.addBootstrapTokenToUserData

This method is responsible for adding the bootstrap token for the machine. Bootstrap tokens are used when joining new nodes to a cluster. Bootstrap Tokens are defined with a specific `SecretType`: `bootstrap.kubernetes.io/token` and live in the `kube-system` namespace. These Secrets are then read by the Bootstrap Authenticator in the API Server

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
DeleteMachineFin["retryPeriod=c.deleteMachineFinalizers(machine)\n(dead code?)"]
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
ShortR-->UpdateMachineStatus
LongR-->UpdateMachineStatus
UpdateMachineStatus-->Z
```

### controller.drainNode

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
SetForceDelParams--No-->UpdateNodeTermCond["err=c.UpdateNodeTerminationCondition(ctx, machine)"]
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


## controller.reconcileMachineHealth

[controller.reconcileMachineHealth](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/machinecontroller/machine_util.go#L584) reconciles the machine object with any change in node conditions or VM health.

```go
func (c *controller) reconcileMachineHealth(ctx context.Context, machine *Machine) 
  (machineutils.RetryPeriod, error)
```

NOTES:
1. Reference [controller.isHealth](./mc_helper_methods.md#controllerishealthy) which checks the machine status conditions.

### Health Check FLow Diagram
```mermaid

%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

ShortR(("return machineutils.ShortRetry, err"))
Begin((" "))-->Init["
cloneDirty = false
machineClone = machine.DeepCopy()
"]-->GetNode["
  node, err := c.nodeLister.Get(machine.Status.Node)
"]-->CheckErr{err!-nil}

CheckErr--Yes-->
  ChkIsNotFound{"IsNotFound(err)"}
    --Yes-->ChkMachinePhase{"
    len(machine.Status.Conditions) > 0
    &&
  machine.Status.CurrentStatus.Phase == MachineRunning
"} --"Yes\n(node missing)"
  -->SetUnhealthyNotFound["
    lastOpDesc='machine unhealthy due to node obj missing'
    lastOpState=MachineStateProcessing
    machinePhase: MachineUnknown
    cloneDirty=true
    "]
  -->TODO

ChkIsNotFound--No-->ShortR
ChkMachinePhase--No-->TODO


CheckErr--No-->CheckNodeConditions{"nodeConditionsHaveChanged(
  machine.Status.Conditions, 
  node.Status.Conditions) ? "}

CheckNodeConditions--Yes-->SetChangedConditions["
   machineClone.Status.Conditions = node.Status.Conditions
   cloneDirty = true
"]-->ChkNotHealthyButRunning
CheckNodeConditions--No-->ChkNotHealthyButRunning


ChkNotHealthyButRunning{"!c.isHealth(machine)
&&
machine.Status.CurrentStatus.Phase == MachineRunning
// machine not healthy, and current state running,"}
--Yes-->SetUnhealtyDesc["
  lastOpDesc ='Machine Unealthy due to conditions:' + machine.Status.Condtions
  lastOpState=MachineStateProcessing
  machinePhase: MachineUnknown
  cloneDirty=true"]
-->ChkHealthyButNotRunning{"!c.isHealthy(clone)
  && 
  clone.Status.CurrentStatus.Phase == MachineRunning
  //machine healthy, but curr phase not running
  "}
  -->ChkMachineCreateOk{"
    machine.Status.LastOperation.Type == MachineOperationCreate 
    &&
		machine.Status.LastOperation.State != MachineStateSuccessful 
  "}
  -->ChkLastOpCreateAndOpStateNotMarkedSuccess{"
   machine.Status.LastOperation.Type == MachineOperationCreate 
   &&
	 machine.Status.LastOperation.State != MachineStateSuccessful 
   //create ok but not marked yet as success
  "}
  --Yes-->SetSuccessDesc["
    lastOpDes = 'Machine successfully joined the cluster'
		lastOpType = MachineOperationCreate
  "]-->DeleteBootstrapToken["
    c.deleteBootstrapToken(ctx, Machine.Name)
    // delete bootstrap token created at machine launch
  "]-->SetLastOpStateSuccessAndPhaseRuning["
    lastOpState = MachineStateSuccessful,
    machinePhase: MachineRunning
    cloneDirty=true
  "]

  ChkLastOpCreateAndOpStateNotMarkedSuccess--No-->SetMachineRejoinCluster["
    lastOpDes = 'Machine successfully re-joined the cluster'
		lastOpType = MachineOperationHealthCheck
  "]-->SetLastOpStateSuccessAndPhaseRuning


  SetLastOpStateSuccessAndPhaseRuning-->TODO


style SetUnknown text-align:left

```

### Health Check Summary

1. Gets the `Node` obj associated with the machine. If it IS NOT found, yet the current machine phase is `Running`, change the machine phase to `Unknown`, the last operation state to `Processing`, the last operation type to `HealthCheck`, update the machine status and return with a short retry.  
2. If the `Node` object IS found, then it checks whether the `Machine.Status.Conditions` are different from `Node.Status.Conditions`. If so it sets the machine conditions to the node conditions.
3.  If the machine IS NOT healthy (See [isHealthy](./mc_helper_methods.md#controllerishealthy)) but the current machine phase is `Running`, change the machine phase to `Unknown`, the last operation state to `Processing`, the last operation type to `HealthCheck`, update the machine status and return with a short retry.  
4. If the machine IS healthy but the current machine phase is NOT `Running`,  check whether the last operation type was a `Create`.
   1.  If the last operation type was a `Create` and last operation state is not marked as `Successful`, then delete the bootstrap token associated with the machine. Change the last operation state to `Successful`.
   1. If the last operation type was NOT a `Create`, change the last operation type to `HealthCheck`
   1. Change the machine phase to `Running` and update the machine status and return with a short retry.
5. If the current machine phase is `Pending` (ie machine being created) get the configured machine creation timeout and check.  
   1. If the timoeut HAS NOT expired, enqueue the machine key on the machine work queue after 1m. 
   1. If the timeout HAS expired, then change the last operation state to `Failed` and the machine phase to `Failed`. Update the machine status and return with a short retry.
6. If the current machine phase is `Unknown`, get the effective machine health timeout and check. 
   1. If the timoeut HAS NOT expired, enqueue the machine key on the machine work queue after 1m. 
   2. If the timoeut HAS expired 
      1. Get the machine deployment name `machine.Labels['name']`
      2. Register ONE permit with this name. See [Permit Giver](../mcm_facilities.md#permitspermitgiver)

```
    machineClone.Status.CurrentStatus = CurrentStatus {
      Phase: MachineUnknown,
      LastUpdateTime: Now(),
    };
    machineClone.Status.LastOperation = LastOperation{
        Description:    statusDesc,
        State:          MachineStateProcessing,
        Type:           MachineOperationHealthCheck,
        LastUpdateTime: Now(),
    }
    cloneDirty = true
```
### Health Check Doubts

1. TODO: Why don't we check the machine health using the `Driver.GetMachineStatus` in the reconcile Machine health ? (seems like something obvious to do and would have helped in those meltdown issues where machine was incorrectly marked as failed)
1. TODO: Why do we do `len(machine.Status.Condtions)==0` in the below when ?
1. TODO: why doesn't this code make use of the helper method: `c.machineStatusUpdate` ?
1. TODO: Unclear why `LastOperation.Description` does not use/concatenate one of the predefined constants in `machineutils`
2. TODO: code makes too much use of `cloneDirty` to check whether machine clone obj has changed, when it could easily return early in several branches.
3. TODO: Code directly makes calls to enqueue machine keys on the machine queue and still returns retry periods to caller leanding to un-necessary enqueue of machine keys. (spurious design)

