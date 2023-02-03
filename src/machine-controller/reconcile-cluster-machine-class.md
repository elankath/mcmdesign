# Reconcile Cluster Machine Class 

## Reconcile Cluster Machine Class Key
`reconcileClusterMachineClassKey` just picks up the machine class key from the machine class queue  and then delegates further. 

```go
func (c *controller) reconcileClusterMachineClassKey(key string) error
```
```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

GetMCName["ns,name=cache.SplitMetanamespacekey(mkey)"]
-->GetMC["
class, err := c.machineClassLister.MachineClasses(c.namespace).Get(name)
if err != nil return err  // basically adds back to the queue after rate limiting
"]
-->RMC["
ctx := context.Background()
reconcileClusterMachineClass(ctx, class)"]
-->CheckErr{"err !=nil"}
--Yes-->ShortR["machineClassQueue.AddAfter(key, machineutils.ShortRetry)"]
CheckErr--No-->LongR["machineClassQueue.AddAfter(key, machineutils.LongRetry)"]

```

## Reconcile Cluster Machine Class

```go
func (c *controller) reconcileClusterMachineClass(ctx context.Context,
 class *v1alpha1.MachineClass) error 
```
Bad design: should ideally return the retry period like other reconcile functions.

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

FindMachineForClass["
machines := Use machineLister and 
match on Machine.Spec.Class.Name == class to 
find machines with matching class"]
-->CheckDelTimeStamp{"
// machines are ref
class.DeletionTimestamp == nil
&& len(machines) > 0
"}

CheckDelTimeStamp--Yes-->AddMCFinalizers["
Add/Update MCM Finalizers to MC 
and use controlMachineClient to update
(why mcm finalizer not mc finalizer?)
'machine.sapcloud.io/machine-controller-manager'
retryPeriod=LongRetry
"]
-->ChkMachineCount{{"len(machines)>0?"}}
--Yes-->EnQMachines["
iterate machines and invoke:
c.machineQueue.Add(machine)
"]
-->End(("End"))

CheckDelTimeStamp--No-->Shortr["
// Seems like over-work here.
retryPeriod=ShortRetry
"]-->ChkMachineCount

ChkMachineCount--No-->DelMCFinalizers["
controller.deleteMachineClassFinalizers(ctx, class)
"]-->End
```



NOTE: Scratch work below. IGNORE.

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

a["ns,name=cache.SplitMetanamespacekey(mkey)"]
getm["machine=machinelister.machines(ns).get(name)"]
valm["validation.validatemachine(machine)"]
valmc["machineclz,secretdata,err=validation.validatemachineclass(machine)"]
longr["retryperiod=machineutils.longretry"]
shortr["retryperiod=machineutils.shortretry"]
enqm["machinequeue.addafter(mkey, retryperiod)"]
checkmdel{"is\nmachine.deletiontimestamp\nset?"}
newdelreq["req=&driver.deletemachinerequest{machine,machineclz,secretdata}"]
delflow["retryperiod=controller.triggerdeletionflow(req)"]
createflow["retryperiod=controller.triggercreationflow(req)"]
hasfin{"hasfinalizer(machine)"}
addfin["addmachinefinalizers(machine)"]
checkmachinenodeexists{"machine.status.node\nexists?"}
reconcilemachinehealth["controller.reconcilemachinehealth(machine)"]
syncnodetemplates["controller.syncnodetemplates(machine)"]
newcreatereq["req=&driver.createmachinerequest{machine,machineclz,secretdata}"]
z(("end"))

a-->getm
enqm-->z
longr-->enqm
shortr-->enqm
getm-->valm
valm-->ok-->valmc
valm--err-->longr
valmc--err-->longr
valmc--ok-->checkmdel
checkmdel--yes-->newdelreq
checkmdel--no-->hasfin
newdelreq-->delflow
hasfin--no-->addfin
hasfin--yes-->shortr
addfin-->checkmachinenodeexists
checkmachinenodeexists--yes-->reconcilemachinehealth
checkmachinenodeexists--no-->newcreatereq
reconcilemachinehealth--ok-->syncnodetemplates
syncnodetemplates--ok-->longr
syncnodetemplates--err-->shortr
delflow-->enqm
newcreatereq-->createflow
createflow-->enqm

```