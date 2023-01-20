- [Machine Controller](#machine-controller)
	- [MC Launch](#mc-launch)
		- [Dev](#dev)
			- [Build](#build)
			- [Launch](#launch)
		- [Prod](#prod)
			- [Build](#build-1)
		- [Launch Flow](#launch-flow)
			- [Summary](#summary)
	- [Machine Controller Loop](#machine-controller-loop)
		- [app.Run](#apprun)
			- [Summary](#summary-1)
		- [app.StartControllers](#appstartcontrollers)
		- [controller initialization](#controller-initialization)
			- [1. newcontroller factory func](#1-newcontroller-factory-func)
			- [1.1 create controller struct](#11-create-controller-struct)
			- [1.2 assign listers and hassynced funcs to controller struct](#12-assign-listers-and-hassynced-funcs-to-controller-struct)
			- [1.3 register controller event handlers on informers.](#13-register-controller-event-handlers-on-informers)
			- [1.4 event handler functions](#14-event-handler-functions)
				- [1.4.1 adding secrets keys to secretsqueue](#141-adding-secrets-keys-to-secretsqueue)
				- [1.4.2 adding machine class names and keys to machineclassqueue](#142-adding-machine-class-names-and-keys-to-machineclassqueue)
				- [1.4.3 adding machine keys to machinequeue](#143-adding-machine-keys-to-machinequeue)
		- [machinecontroller.run](#machinecontrollerrun)
			- [1. wait for informer caches to sync](#1-wait-for-informer-caches-to-sync)
			- [2. register metrics](#2-register-metrics)
				- [2.1 controller.describe](#21-controllerdescribe)
				- [2.1 controller.collect](#21-controllercollect)
			- [3. create controller worker go-routines applying reconciliations](#3-create-controller-worker-go-routines-applying-reconciliations)
				- [3.1 createworker](#31-createworker)
			- [4. reconciliation functions executed by worker](#4-reconciliation-functions-executed-by-worker)
# Machine Controller

The [Machine Controller]() handles reconciliation of [Machine](./../mcm_facilities.md#machine) and [MachineClass](./../mcm_facilities.md#machineclass) objects. 

The Machine Controller Entry Point for any provider is at 
`machine-controller-manager-provider-<name>/cmd/machine-controller/main.go`

## MC Launch

### Dev 

#### Build
A `Makefile` in the root of `machine-controller-manager-provider-<name>` builds the provider specific machine controller  for linux with CGO enabled. The `make build` target invokes the shell script [.ci/build](https://github.com/gardener/machine-controller-manager-provider-aws/blob/master/.ci/build) to do this.

```
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
  -a \
  -v \
  -o ${BINARY_PATH}/rel/machine-controller \
  cmd/machine-controller/main.goo
```

#### Launch
Assuming one has initialized the variables using `make download-kubeconfigs`, one can then use `make start` target which launches the MC with flags as shown below.
Most of these timeout flags are redundant since exact same values are 
given in [machine-controller-manager/pkg/util/provider/app/options.NewMCServer]()
```
go run -mod=vendor 
    cmd/machine-controller/main.go 
    --control-kubeconfig=$(CONTROL_KUBECONFIG) 
    --target-kubeconfig=$(TARGET_KUBECONFIG) 
    --namespace=$(CONTROL_NAMESPACE) 
    --machine-creation-timeout=20m 
    --machine-drain-timeout=5m 
    --machine-health-timeout=10m 
    --machine-pv-detach-timeout=2m 
    --machine-safety-apiserver-statuscheck-timeout=30s 
    --machine-safety-apiserver-statuscheck-period=1m 
    --machine-safety-orphan-vms-period=30m 
    --leader-elect=$(LEADER_ELECT) 
    --v=3
```
### Prod 

#### Build
A `Dockerfile` builds the provider specific machine controller and launches it directly with no CLI arguments. Hence uses coded defaults

```Dockerfile
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH \
      go build \
      -mod=vendor \
      -o /usr/local/bin/machine-controller \
      cmd/machine-controller/main.go
COPY --from=builder /usr/local/bin/machine-controller /machine-controller
ENTRYPOINT ["/machine-controller"]
```

The `machine-controller-manager` deployment usually launches both the MC in a Pod with following arguments
```
./machine-controller
         --control-kubeconfig=inClusterConfig
         --machine-creation-timeout=20m
         --machine-drain-timeout=2h
         --machine-health-timeout=10m
         --namespace=shoot--i034796--tre
         --port=10259
         --target-kubeconfig=/var/run/secrets/gardener.cloud/shoot/generic-kubeconfig/kubeconfig
         --v=3
```

### Launch Flow

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TB

Begin(("cmd/
machine-controller/
main.go"))
-->NewMCServer["mc=options.NewMCServer"]
-->AddFlaogs["mc.AddFlags(pflag.CommandLine)"]
-->LogOptions["options := k8s.io/component/base/logs.NewOptions()
	options.AddFlags(pflag.CommandLine)"]
-->InitFlags["flag.InitFlags"]
InitFlags--local-->NewLocalDriver["
	driver, err := local.NewDriver(s.ControlKubeconfig)
	if err exit
"]
InitFlags--aws-->NewPlatformDriver["
	driver := aws.NewAWSDriver(&spi.PluginSPIImpl{}))
	OR
	driver := cp.NewAzureDriver(&spi.PluginSPIImpl{})
	//etc
"]

NewLocalDriver-->AppRun["
	err := app.Run(mc, driver)
"]
NewPlatformDriver-->AppRun
AppRun-->End(("if err != nil 
os.Exit(1)"))
```
#### Summary
1. Creates [machine-controller-manager/pkg/util/provider/app/options.MCServer](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/util/provider/app/options#MCServer) using `options.NewMCServer` which is the main context object for the machinecontroller that embeds a
[options.MachineControllerConfiguration](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/options#MachineControllerManagerConfiguration).

   `options.NewMCServer` initializes `options.MCServer` struct with default values for 
    - `Port: 10258`, 
    - `Namespace: default`, 
    - `ConcurrentNodeSyncs: 50`: number of worker go-routines that are used to process items from a work queue. See [Worker](#31-createworker) below
    - `NodeConditions: "KernelDeadLock,ReadonlyFilesystem,DiskPressure,NetworkUnavailable"` (failure node conditions that indicate that a machine is un-healthy)
    - `MinResyncPeriod: 12 hours`, `KubeAPIQPS: 20`, `KubeAPIBurst:30`: config params for k8s clients. See [rest.Config](https://pkg.go.dev/k8s.io/client-go@v0.25.3/rest#Config)

1. calls `MCServer.AddFlags` which defines all parsing flags for the machine controller into fields of `MCServer` instance created in the last step.
1. calls `k8s.io/component-base/logs.NewOptions` and then `options.AddFlags` for logging options. 
  TODO: Should get rid of this when moving to `logr`.) 
    - See [Logging In Gardener Components](https://github.com/gardener/gardener/blob/master/docs/development/logging.md). 
    - Then use the [logcheck](https://github.com/gardener/gardener/tree/master/hack/tools/logcheck)tool.
1. Driver initialization code varies according to the provider type.
	- Local Driver
      - calls `NewDriver` with control kube config that creates a controller runtime client (`sigs.k8s.io/controller-runtime/pkg/client`) which then calls `pkg/local/driver.NewDriver` passing the controlloer-runtime client which constructs a `localdriver` encapsulating the passed in client.
      - `driver := local.NewDriver(c)`
      - the `localdriver` implements [Driver](https://github.com/gardener/machine-controller-manager/blob/master/pkg/util/provider/driver/driver.go#l28) is the facade for creation/deletion of vm's
    - Provider Specific Driver Example
      - `driver := aws.NewAWSDriver(&spi.PluginSPIImpl{})`
      - `driver := cp.NewAzureDriver(&spi.PluginSPIImpl{})`
      - `spi.PluginSPIImpl` is a struct that implements a provider specific interface that initializes a provider session.
2. calls [app.Run](https://github.com/gardener/machine-controller-manager/blob/master/pkg/util/provider/app/app.go#l77) passing in the previously created `MCServer` and `Driver` instances.

## Machine Controller Loop

### app.Run

`app.Run` is the function that setups the main control loop of the machine controller server. 


#### Summary
1. [app.Run(options *options.MCServer, driver driver.Driver)](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/app/app.go#L77) is the common run loop for all provider Machine Controllers.
1. Creates `targetkubeconfig` and `controlkubeconfig` of type `k8s.io/client-go/rest.Config` from the target kube config path using `clientcmd.BuildConfigFromFlags`
1. Set fields such as `config.QPS` and `config.Burst`  in both `targetkubeconfig` and `controlkubeconfig` from the passed in `options.MCServer`
1. Create `kubeClientControl` from the `controlkubeconfig` using the standard client-go client factory metohd: `kubernetes.NewForConfig` that returns a `client-go/kubernetes.Clientset`
1. Similarly create another `Clientset` named `leaderElectionClient` using `controlkubeconfig`
1. Start a go routine using the function `startHTTP` that registers a bunch of http handlers for the go profiler, prometheus metrics and the health check.
1. Call `createRecorder` passing the `kubeClientControl` client set instance that returns a [client-go/tools/record.EventRecorder](https://github.com/kubernetes/client-go/blob/master/tools/record/event.go#L91)
	1. Creates a new `eventBroadcaster` of type [event.EventBroadcaster](https://github.com/kubernetes/client-go/blob/master/tools/record/event.go#l113)
	1. Set the logging function of the broadcaster to `klog.Infof`.
	1. Sets the event sink using `eventBroadcaster.StartRecordingToSink` passing the event interface as `kubeClient.CoreV1().RESTClient()).Events("")`. Effectively events will be published remotely.
	1. Returns the `record.EventRecorder` associated with the `eventBroadcaster` using `eventBroadcaster.NewRecorder` 
1. Constructs an anonymous function assigned to `run` variable which does the following:
	1. initializes a `stop` receive channel.
	1. creates a `controlMachineClientBuilder` using `machineclientbuilder.SimpleClientBuilder` using the `controlkubeconfig`.
	1. creates a `controlCoreClientBuidler` using `coreclientbuilder.SimpleControllerClientBuilder` wrapping `controlkubeconfig`.
	1. creates `targetCoreClientBuilder` using `coreclientbuilder.SimpleControllerClientBuilder` wrapping `controlkubeconfig`.
	1. call the `app.StartControllers` function passing the `options`, `driver`, `controlkubeconfig`, `targetkubeconfig`, `controlMachineClientBuilder`, `controlCoreClientBuilder`, `targetCoreClientBuilder`, `recorder` and `stop` channel.
    	- // Q: if you are going to pass the controlkubeconfig and targetkubeconfig - why not create the client builders inside the startcontrollers ?
	1.  if `app.StartcOntrollers` return an error panic and exit `run`.
  1. use [leaderelection.RunOrDie](https://github.com/kubernetes/client-go/blob/master/tools/leaderelection/leaderelection.go#L218) to start a leader election and pass the previously created `run` function to as the callback for `OnStartedLeading`. `OnStartedLeading` callback is invoked when a leaderelector client starts leading.

### app.StartControllers
[app.StartControllers](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/app/app.go#L202) starts all controller loops which are part of the machine controller. 

```go
func StartControllers(s *options.MCServer,
	controlCoreKubeconfig *rest.Config,
	targetCoreKubeconfig *rest.Config,
	controlMachineClientBuilder machineclientbuilder.ClientBuilder,
	controlCoreClientBuilder coreclientbuilder.ClientBuilder,
	targetCoreClientBuilder coreclientbuilder.ClientBuilder,
	driver driver.Driver,
	recorder record.EventRecorder,
	stop <-chan struct{}) error
```
1. calls `getAvailableResources` using the `controlCoreClientBuilder` that returns a `map[schema.GroupVersionResource]bool` assigned to `availableresources`
	- `getAvailableResources` waits till the api server is running by checking its `/healthz` using `wait.PollImmediate`. keeps re-creating the client using `clientbuilder.Client` method. 
	- then uses `client.Discovery().ServerResources` which returns returns the supported resources for all groups and versions as a slice of [*metav1.APIResourceList](https://github.com/kubernetes/apimachinery/blob/373a5f752d44989b9829888460844849878e1b6e/pkg/apis/meta/v1/types.go#L1131) (which encapsulates a [[]APIResource](https://github.com/kubernetes/apimachinery/blob/v0.26.1/pkg/apis/meta/v1/types.go#L1081)) and then converts that to a `map[schema.GroupVersionResource]bool` `
- creates a `controlMachineClient` which is a client for the controller crd types (type: `versioned.interface`) using `controlmachineclientbuilder.clientordie` using `machine-controller` as client name. this client targets the control cluster - ie the cluster holding the machine cr's.
- creates a `controlcoreclient` (of type: `kubernetes.clientset`) which is the standard k8s client-go client for accessing the k8s control cluster.
- creates a `targetcoreclient` (of type: `kubernetes.clientset`) which is the standard k8s client-go client for accessing the k8s target cluster. 
- obtain the target cluster k8s version using the discovery interface and preserve it in `targetkubernetesversion`
- if the `availableresources` does not contain the `machinegvr` exit this loop with error.
- creates the following informer factories:
  -  `controlmachineinformerfactory` using the generated `client/informers/externalversions.newfilteredsharedinformerfactory` passing the conrol machine client, the configured min resync period and control namespace obtained from mcserver.
  -  `controlcoreinformerfactory` using the standard k8s client-go `k8s.io/client-go/informers.newfilteredsharedinformerfactory` method passing in the k8s client for the , the control cluster, configured min resync period and control namespace obtained from mcserver.
  -  `targetcoreinformerfactory` using the standard k8s client-go `k8s.io/client-go/informers.newfilteredsharedinformerfactory` method passing in the k8s core client for the target cluster, the configured min resync period and control namespace obtained from mcserver.
  -   get the controller's informer access facade `v1alpha1.interface` using `controlmachineinformerfactory.machine().v1alpha1()` and assign to `machinesharedinformers`
  -  now create the `machinecontroller` using `pkg/util/provider/machinecontroller/controller.newcontroller` factory function, passing the below:
     -  control namespace from `mcserver.namespace`
     -  `safetyoptions` from `mcserver.safetyoptions`
     -  `nodeconditions` from `mcserver.nodeconditions`. (by default these would be : "kerneldeadlock,readonlyfilesystem,diskpressure,networkunavailable")
     -  clients: `controlmachineclient`, `controlcoreclient`, `targetcoreclient`
     -  the `driver` 
     -  target cluster informers: `nodeinformer`, `persistentvolumeclaimsinformer`, `persistentvolumeinformer`, `volumeattachmentinformer` and `poddisruptionbudgetinformer` obtained from `targetcoreinformerfactory`
     -  control cluster informers: 
        - `secretinformer` from `controlcoreinformerfactory`
        - `machineclassinformer`, `machineinformer` from `machinesharedinformers`
     -  event recorder
     -  `targetkubernetesversion`
  -  start all informers by calling `sharedinformerfactory.start(stopch <-chan struct{})` for `controlmachineinformerfactory`, `controlcoreinformerfactory` and `targetcoreinformerfactory` passing teh `stop` channel
  -  launches the `machinecontroller` in new go-routine by invoking: `go machinecontroller.run(mcserver.concurrentnodesyncs, stop)`
  
### controller initialization
 the machine controller is constructed using
`machine-controller-manager/pkg/util/provider/machinecontroller/controller.newcontroller` factory function which initializes the `controller` struct.


#### 1. newcontroller factory func

mc is constructed using the factory function below:
```go
func NewController(
	namespace string,
	controlmachineclient machineapi.machinev1alpha1interface,
	controlcoreclient kubernetes.interface,
	targetcoreclient kubernetes.interface,
	driver driver.driver,
	pvcinformer coreinformers.persistentvolumeclaiminformer,
	pvinformer coreinformers.persistentvolumeinformer,
	secretinformer coreinformers.secretinformer,
	nodeinformer coreinformers.nodeinformer,
	pdbinformer policyinformers.poddisruptionbudgetinformer,
	volumeattachmentinformer storageinformers.volumeattachmentinformer,
	machineclassinformer machineinformers.machineclassinformer,
	machineinformer machineinformers.machineinformer,
	recorder record.eventrecorder,
	safetyoptions options.safetyoptions,
	nodeconditions string,
	bootstraptokenauthextragroups string,
) (controller, error) {
	// etc

}
```

#### 1.1 create controller struct

create the controller struct initializing rate-limiting work queues for all relevant resources

```go
	controller := &controller{
		namespace:                     namespace,
		controlmachineclient:          controlmachineclient,
		controlcoreclient:             controlcoreclient,
		targetcoreclient:              targetcoreclient,
		recorder:                      recorder,
		secretqueue:                   workqueue.newnamedratelimitingqueue(workqueue.defaultcontrollerratelimiter(), "secret"),
		nodequeue:                     workqueue.newnamedratelimitingqueue(workqueue.defaultcontrollerratelimiter(), "node"),
		machineclassqueue:             workqueue.newnamedratelimitingqueue(workqueue.defaultcontrollerratelimiter(), "machineclass"),
		machinequeue:                  workqueue.newnamedratelimitingqueue(workqueue.defaultcontrollerratelimiter(), "machine"),
		machinesafetyorphanvmsqueue:   workqueue.newnamedratelimitingqueue(workqueue.defaultcontrollerratelimiter(), "machinesafetyorphanvms"),
		machinesafetyapiserverqueue:   workqueue.newnamedratelimitingqueue(workqueue.defaultcontrollerratelimiter(), "machinesafetyapiserver"),
		safetyoptions:                 safetyoptions,
		nodeconditions:                nodeconditions,
		driver:                        driver,
		bootstraptokenauthextragroups: bootstraptokenauthextragroups,
		volumeattachmenthandler:       nil,
		permitgiver:                   permits.newpermitgiver(permitgiverstaleentrytimeout, janitorfreq),
	}


```
#### 1.2 assign listers and hassynced funcs to controller struct

```go
	// initialize controller listers from the passed-in shared informers (8 listers)
	controller.pvclister = pvcinformer
	controller.pvlister = pvinformer.lister()
    controller.machinelister = machineinformer.lister()
	// etc
	// assign the hassynced function from the passed-in shared informers
	controller.pvcsynced = pvcinformer.informer().hassynced
	controller.pvsynced = pvinformer.informer().hassynced
    controller.machinesynced = machineinformer.informer().hassynced
```

#### 1.3 register controller event handlers on informers.

an informer invokes registered event handler when a k8s object changes. event handlers are registered using `<type>informer().addeventhandler(resourceeventhandler)` function. the controller initialization registers event handlers: on control-cluster `secretinformer`, control-cluster `machineclassinformer`, control-cluster `machineinformer` and target-cluster `nodeinformer`.

```go
   // event handlers added for control-cluster secret add/delete
	secretinformer.informer().addeventhandler(cache.resourceeventhandlerfuncs{
		addfunc:    controller.secretadd,
		deletefunc: controller.secretdelete,
	})

   // 2 event handlers added for controlcluster machineclass add/update/delete
	machineclassinformer.informer().addeventhandler(cache.resourceeventhandlerfuncs{
		addfunc:    controller.machineclasstosecretadd,
		updatefunc: controller.machineclasstosecretupdate,
		deletefunc: controller.machineclasstosecretdelete,
	})

	machineclassinformer.informer().addeventhandler(cache.resourceeventhandlerfuncs{
		addfunc:    controller.machineclassadd,
		updatefunc: controller.machineclassupdate,
		deletefunc: controller.machineclassdelete,
	})

   // 3 event handlers added for control-cluster machine add/update/delete
	machineinformer.informer().addeventhandler(cache.resourceeventhandlerfuncs{
		addfunc:    controller.machinetomachineclassadd,
		updatefunc: controller.machinetomachineclassupdate,
		deletefunc: controller.machinetomachineclassdelete,
	})
	machineinformer.informer().addeventhandler(cache.resourceeventhandlerfuncs{
		addfunc:    controller.addmachine,
		updatefunc: controller.updatemachine,
		deletefunc: controller.deletemachine,
	})

	machineinformer.informer().addeventhandler(cache.resourceeventhandlerfuncs{
		// updatemachinetosafety makes sure that orphan vm handler is invoked on some specific machine obj updates
		updatefunc: controller.updatemachinetosafety,
		// deletemachinetosafety makes sure that orphan vm handler is invoked on any machine deletion
		deletefunc: controller.deletemachinetosafety,
	})

    // event handler registered for target-cluster node add/update/delete
	nodeinformer.informer().addeventhandler(cache.resourceeventhandlerfuncs{
		addfunc:    controller.addnodetomachine,
		updatefunc: controller.updatenodetomachine,
		deletefunc: controller.deletenodetomachine,
	}		

	// event handler added for target-cluster volume attachments is handled by
	// utility volumeattachmenthandler that can distribute incoming volueattachment 
	// add/updates to a bunch of workers.
	controller.volumeattachmenthandler = drain.newvolumeattachmenthandler()
	volumeattachmentinformer.informer().addeventhandler(cache.resourceeventhandlerfuncs{
		addfunc:    controller.volumeattachmenthandler.addvolumeattachment,
		updatefunc: controller.volumeattachmenthandler.updatevolumeattachment,
	})
```

#### 1.4 event handler functions

controller informer event handlers generally add the object keys to the appropriate work queues which are later picked up and reconciled in processing in `controller.run`.

the work queue is used to separate the delivery of the object from its processing. resource event handler functions extract the key of the delivered object and add it to the relevant work queue for future processing. (in `controller.run`) 

##### 1.4.1 adding secrets keys to secretsqueue 

`controller.secretadd(obj interfade{})` which is the `addfunc` callback registered for the `secretinformer` extracts the secret `key` using `cache.deletionhandlingmetanamespacekeyfunc(obj)` and then adds this `key` to the `secretqueue`.

```go
func (c *controller) secretadd(obj interface{}) {
// confusing. should just use metanamespacekeyfunc(obj)
	key, err := cache.deletionhandlingmetanamespacekeyfunc(obj) 
	if err != nil {
		c.secretqueue.add(key)
	}
}
func (c *controller) secretdelete(obj interface{}) {
	c.secretadd(obj) // dont like this delegation
}
```
in the case of `controller.secretdelete`, we must check for the [deletedfinalstateunknown](https://pkg.go.dev/k8s.io/client-go/tools/cache#deletedfinalstateunknown) state of that secret in the cache before enqueuing its key. the `deletedfinalstateunknown` state means that the object has been deleted but that the watch deletion event was missed while disconnected from apiserver and the controller didn't react accordingly, hence we need to add to the secretqueue for processing.

`controller.machineclasstosecretadd`  is the `addfunc` callback registered for the `machineclassinformer` adds the key (namespace/name) of the secret obtained from the newly added `machineclass` object.

```go
func (c *controller) machineclasstosecretadd(obj interface{}) {
	machineclass, ok := obj.(*v1alpha1.machineclass)
	// if error or nil ref return
	secretrefs := []*corev1.secretreference{machineclass.secretref, machineclass.credentialssecretref}
	for _, secretref := range secretrefs {
		if secretref != nil {
			queue.add(secretref.namespace + "/" + secretref.name)
		}
	}
}
```

`controller.machineclasstosecretupdate` is the `updatefunc` callback registered for the `machine classinformer` which adds the key (namespace/name) of the secret obtained from the old/new machine class if its `secretref` is not nil.

```go
func (c *controller) machineclasstosecretupdate(oldobj interface{}, newobj interface{}) {
	oldmachineclass, ok := oldobj.(*v1alpha1.machineclass)
	newmachineclass, ok := newobj.(*v1alpha1.machineclass)
	// if error or nil refs return
	secretqueue.add(oldmachineclass.secretref.namespace + "/" + oldmachineclass.secretref.name)
	secretqueue.add(newmachineclass.secretref.namespace + "/" + newmachineclass.secretref.name)
}
```

##### 1.4.2 adding machine class names and keys to machineclassqueue

`controller.machineclassadd`is specified as both the `addfunc` and `deletefunc` callback registered on the `machineclassinformer`. it gets the object key `namespace/name` for the machine class `obj` and adds the key to the `machineclassqueue`

`controller.machineclassupdate` just delegates to `controller.machineclassadd(newobj)`.

```go
func (c *controller) machineclassadd(obj interface{}) {
	key, err := cache.deletionhandlingmetanamespacekeyfunc(obj)
	// log & return on err != nil
	c.machineclassqueue.add(key)
}
func (c *controller) machineclassupdate(oldobj, newobj interface{}) {
	old, ok := oldobj.(*v1alpha1.machineclass)
	if old == nil || !ok {
		return
	}
	new, ok := newobj.(*v1alpha1.machineclass)
	if new == nil || !ok {
		return
	}

	c.machineclassadd(newobj)
}
```

`controller.machinetomachineclassadd` is both an `addfunc` and `deletefunc` callback registered on the `machineinfomer` . this simply adds the machine spec class name to the `machineclassqueue`. 
```go

func (c *controller) machinetomachineclassadd(obj interface{}) {
	c.machineclassqueue.add(machine.spec.class.name)
}
```
`controller.machinetomachineclassupdate` is the `updatefunc` callback registered on the `machineinformer`.  it enqueues the `newmacine` spec class name if there are no changes in the machine class name, otherwise it adds both the old and new machine classnames to the `machineclassqueue'.

```go
func (c *controller) machinetomachineclassupdate(oldobj, newobj interface{}) {
	oldmachine, ok := oldobj.(*v1alpha1.machine)
	newmachine, ok := newobj.(*v1alpha1.machine)
	// return if nil or not ok
	if oldmachine.spec.class.kind == newmachine.spec.class.kind {
		c.machineclassqueue.add(newmachine.spec.class.name)
	} else {
		c.machineclassqueue.add(oldmachine.spec.class.name)
		c.machineclassqueue.add(newmachine.spec.class.name)
	}
}
```

so, the `machineclassqueue` effectively holds machine-class keys (ns/name) and plain class names (name)
for added/updated/delete machine classes.

##### 1.4.3 adding machine keys to machinequeue

`controller.addmachine`, `controller.updatemachine` and `controller.deletemachine` all just deletgate the newly added/newly updated and newly delete machine obj to `controller.enqueuemachine` which simpy gets the object key for the machine and adds it to the `machinequeue`.

```go
func (c *controller) enqueuemachine(obj interface{}) {
	key, err := cache.metanamespacekeyfunc(obj)
	// return if err
	c.machinequeue.add(key)
}
```

`controller.addnodetomachine` is specified as the `addfunc` registered for the `nodeinformer`.  basically, when a new node is created, the corresponding `machine` obj is obtained for the `node` and if the `machine.status.conditions`  does not match the `nodestatusconditions` then the machine `key` is added to the `machinequeue` for reconciliation.

snippet shown be below with error handling+logging omitted.
```go
func (c *controller) addnodetomachine(obj interface{}) {
	// if err != nil log err and return
	node := obj.(*corev1.node)
	key, err := cache.deletionhandlingmetanamespacekeyfunc(obj)
	machine, err := c.getmachinefromnode(key) // leverages machinelister and gets machine whose 'node' label matches key
	if machine.status.currentstatus.phase != v1alpha1.machinecrashloopbackoff && nodeconditionshavechanged(machine.status.conditions, node.status.conditions) {
		macinekey, err = cache.metanamespacekeyfunc(obj)
		c.machinequeue.add(key)
	}
}

```

`controller.updatenodetomachine` is specified as `updatefunc` registered for the `nodeinformer`. in a nutshell, it simply delegates to `addnodetomachine(newobj)` except if the node has the annotation `machineutils.triggerdeletionbymcm` (value: `node.machine.sapcloud.io/trigger-deletion-by-mcm`), in which case get the `machine` obj corresponding to the node and then leverages the `controlmachineclient` to delete the machine object.

todo: who sets this annotation ?? can't find the guy.

snippet shown be below without error handling+logging omitted.
```go
func (c *controller) updatenodetomachine(oldobj, newobj interface{}) {
	node := newobj.(*corev1.node)
	// check for the triggerdeletionbymcm annotation on the node object
	// if it is present then mark the machine object for deletion
	if value, ok := node.annotations[machineutils.triggerdeletionbymcm]; ok && value == "true" {
		machine, err := c.getmachinefromnode(node.name)
		if machine.deletiontimestamp == nil {
			c.controlmachineclient
			.machines(c.namespace)
			.delete(context.background(), machine.name, metav1.deleteoptions{});		
		} 
	}  else {
		c.addnodetomachine(newobj)
	}
}
```
`controller.deletenodetomachine` is specified as `deletefunc` registered for the `nodeinformer` and is quite simple. it just finds the corresponding machine object and adds its key to the `machinequeue`

snippet shown be below without error handling+logging omitted.
```go
func (c *controller) deletenodetomachine(obj interface{}) {
	// if err != nil log error and return for statements below
	key, err := cache.deletionhandlingmetanamespacekeyfunc(obj)
	machine, err := c.getmachinefromnode(key) // leverages machinelister and gets machine whose 'node' label matches key
	macinekey, err = cache.metanamespacekeyfunc(obj)
	c.machinequeue.add(key)
}

```
### machinecontroller.run

```go
func (c *controller) run(workers int, stopch <-chan struct{}) {
	// ...
}

```
#### 1. wait for informer caches to sync

when an informer starts, it will build a cache of all resources it currently watches which is lost when the application
restarts. this means that on startup, each of your handler functions will be invoked as the initial state is built. if this
is not desirable for your use case, you can wait until the caches are synced before performing any updates using the
`cache.waitforcachesync` function:

```go
if !cache.waitforcachesync(stopch, c.secretsynced, c.pvcsynced, c.pvsynced, c.pdbsynced, c.volumeattachementsynced, c.nodesynced, c.machineclasssynced, c.machinesynced) {
		runtimeutil.handleerror(fmt.errorf("timed out waiting for caches to sync"))
		return
}
```
#### 2. register metrics

the controller struct implements the [prometheus.collector](https://pkg.go.dev/github.com/prometheus/client_golang@v1.13.0/prometheus#collector) interface and can therefore
 be registered on prometheus metrics registry. 
 
collectors which are added to the registry will collect metrics to expose them via the metrics endpoint of the mcm every time when the endpoint is called.
```go
prometheus.mustregister(c)
```

##### 2.1 controller.describe

all [promethueus.metric](https://pkg.go.dev/github.com/prometheus/client_golang@v1.13.0/prometheus#metric) that are collected must first be described using a [prometheus.desc](https://pkg.go.dev/github.com/prometheus/client_golang@v1.13.0/prometheus#desc) which is the _meta-data_ about a metric.  

as can be seen below the machine controller sends a description of `metrics.machinecountdesc` to prometheus. this is `mcm_machine_items_total` which is the count of machines managed by controller. doubt: we currently appear to have  only have one metric for the mc ?
```go
var machinecountdesc = prometheus.newdesc("mcm_machine_items_total", "count of machines currently managed by the mcm.", nil, nil)

func (c *controller) describe(ch chan<- *prometheus.desc) {
	ch <- metrics.machinecountdesc
}
```
##### 2.1 controller.collect

`collect` is called by the prometheus registry when collecting
 metrics. the implementation sends each collected metric via the
 provided channel and returns once the last metric has been sent. the
descriptor of each sent metric is one of those returned by `describe`

todo: describe each of the collect methods.
```go
// collect is method required to implement the prometheus.collect interface.
func (c *controller) collect(ch chan<- prometheus.metric) {
	c.collectmachinemetrics(ch)
	//c.collectmachinesetmetrics(ch)
	//c.collectmachinedeploymentmetrics(ch)
	c.collectmachinecontrollerfrozenstatus(ch)
}
```
#### 3. create controller worker go-routines applying reconciliations

```go
func (c *controller) run(workers int, stopch <-chan struct{}) {
	//.. 3
	waitgroup sync.waitgroup
	for i := 0; i < workers; i++ {
		createworker(c.secretqueue, "clustersecret", maxretries, true, c.reconcileclustersecretkey, stopch, &waitgroup)
		createworker(c.machineclassqueue, "clustermachineclass", maxretries, true, c.reconcileclustermachineclasskey, stopch, &waitgroup)
		createworker(c.nodequeue, "clusternode", maxretries, true, c.reconcileclusternodekey, stopch, &waitgroup)
		createworker(c.machinequeue, "clustermachine", maxretries, true, c.reconcileclustermachinekey, stopch, &waitgroup
		createworker(c.machinesafetyorphanvmsqueue, "clustermachinesafetyorphanvms", maxretries, true, c.reconcileclustermachinesafetyorphanvms, stopch, &waitgroup)
		createworker(c.machinesafetyapiserverqueue, "clustermachineapiserver", maxretries, true, c.reconcileclustermachinesafetyapiserver, stopch, &waitgroup)
	}
	<-stopch
	waitgroup.wait()
}

```
##### 3.1 createworker

`createworker` creates and runs a go-routine that just processes items in the
specified `queue`. the worker will run until `stopch` is closed. the worker will be
 added to the wait group when started and marked done when finished.

```go
func createworker(queue workqueue.ratelimitinginterface, resourcetype string, maxretries int, forgetaftersuccess bool, reconciler func(key string) error, stopch <-chan struct{}, waitgroup *sync.waitgroup) {
	waitgroup.add(1)
	go func() {
		wait.until(worker(queue, resourcetype, maxretries, forgetaftersuccess, reconciler), time.second, stopch)
		waitgroup.done()
	}()
}
```

[worker](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/machinecontroller/controller.go#l369) returns a function that 
1. de-queues items (keys) from the work `queue`. the `key`s that are obtained using work `queue.get` to be strings of the form `namespace/name` of the resource. 
2. processes them by invoking the `reconciler(key)` function 
   1. the purpose of the `reconciler` is to compares the actual state with the desired state, and attempts to converge the two. it should then update the `status` block of the resource.
   2. if `reconciler` returns an error, requeue the item up to `maxretries` before giving up.
3. marks items as done.  

then we execute the `reconciler`. 


```go
func worker(queue workqueue.ratelimitinginterface, resourcetype string, maxretries int, forgetaftersuccess bool, reconciler func(key string) error) func() {
	return func() {
		exit := false
		for !exit {
			exit = func() bool {
				key, quit := queue.get()
				if quit {
					return true
				}
				defer queue.done(key)

				err := reconciler(key.(string))
				if err == nil {
					if forgetaftersuccess { // always true for mc
						queue.forget(key)
					}
					return false
				}

				if queue.numrequeues(key) < maxretries {
					queue.addratelimited(key)
					return false
				}

				queue.forget(key)
				return false
			}()
		}
	}
}
```
#### 4. reconciliation functions executed by worker

the controller starts worker go-routines that pop out keys from the relevant workqueue and execute the reconcile function.

See reconcile chapters.

