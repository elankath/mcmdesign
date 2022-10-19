# Reconcile Cluster Machine Class Key

`reconcileClusterMachineClassKey` reconciles an machineClass due to controller resync or an event on the machineClass.

```go
func (c *controller) reconcileClusterMachineClassKey(key string) error
```

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD


a["ns,name=cache.splitmetanamespacekey(mkey)"]
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