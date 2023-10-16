# Orphan / Safety Jobs

Read MCM FAQ: [What is Safety Controller in MCM](https://github.com/gardener/machine-controller-manager/blob/master/docs/FAQ.md#what-is-safety-controller-in-mcm)

These are jobs that periodically run by pushing dummy keys onto their respective work-queues. The worker then picks up and dispatches to the reconcile functions.

```go
    worker.Run(c.machineSafetyOrphanVMsQueue, "ClusterMachineSafetyOrphanVMs", 
    worker.DefaultMaxRetries, 
    true, c.reconcileClusterMachineSafetyOrphanVMs, stopCh, &waitGroup)

    worker.Run(c.machineSafetyAPIServerQueue, "ClusterMachineAPIServer", 
    worker.DefaultMaxRetries, true, c.reconcileClusterMachineSafetyAPIServer, 
    stopCh, &waitGroup)

```

## reconcileClusterMachineSafetyOrphanVMs

This techinically isn't a reconcilation loop. It is effectively just a job.


```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

Begin((" "))
-->RescheduleJob["
// Schedule Rerun
defer c.machineSafetyOrphanVMsQueue.AddAfter('', c.safetyOptions.MachineSafetyOrphanVMsPeriod.Duration)
"]-->
GetMC["Get Machine Classes and iterate"]
-->forEach["
asdf
"]
-->ListDriverMachines["
listMachineResp := driver.ListMachines(...)
"]

ListDriverMachines-->MachinesExist{
    #listMachineResp.MachineList > 0 ?
}
MachinesExist-->|yes| SyncCache["cache.WaitForCacheSync(stopCh, machineInformer.Informer().HasSynced)"]

SyncCache-->IterMachine["Iterate machineID,machineName in listMachineResp.MachineList"]
-->GetMachine["machine, err := Get Machine from Lister"]
-->ChkErr{Check err}

ChkErr-->|err NotFound|DelMachine
ChkErr-->|err nil|ChkMachineId

ChkMachineId{
    machine.Spec.ProviderID == machineID 
    OR
    machinePhase is empty or CrashLoopBackoff ?}
ChkMachineId-->|yes|IterMachine
ChkMachineId-->|no|DelMachine["
driver.DeleteMachine(ctx, &driver.DeleteMachineRequest{
Machine:      machine, secretData...})
"]

DelMachine--iterDone-->ScheduleRerun
MachinesExist-->|no| ReturnLongRetry["retryPeriod := machineutils.LongRetry"]

-->ScheduleRerun["
c.machineSafetyOrphanVMsQueue.AddAfter('', time.Duration(retryPeriod))
"]
```
