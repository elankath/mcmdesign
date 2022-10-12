- [Machine Controller](#machine-controller)
  - [MC Launch](#mc-launch)
  - [machine controller loop](#machine-controller-loop)
    - [app.run](#apprun)
    - [app.startcontrollers](#appstartcontrollers)
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
        - [4.1  reconcileclustersecretkey](#41--reconcileclustersecretkey)
        - [4.2  reconcileclustermachineclasskey](#42--reconcileclustermachineclasskey)
        - [4.3  reconcileclustermachinekey](#43--reconcileclustermachinekey)
          - [4.3.1 controller.triggercreationflow](#431-controllertriggercreationflow)
        - [4.4  reconcileclustermachinesafetyorphanvms](#44--reconcileclustermachinesafetyorphanvms)
        - [4.5  reconcileclustermachinesafetyapiserver](#45--reconcileclustermachinesafetyapiserver)
# Machine Controller

## MC Launch

most of these timeout flags are redundant since exact same values are 
given in `machine-controller-manager/pkg/util/provider/app/options.newmcserver`
```
go run -mod=vendor cmd/machine-controller/main.go \
			--control-kubeconfig=$(control_kubeconfig) \
			--target-kubeconfig=$(target_kubeconfig) \
			--namespace=$(control_namespace) \
			--machine-creation-timeout=20m \
			--machine-drain-timeout=5m \
			--machine-health-timeout=10m \
			--machine-pv-detach-timeout=2m \
			--machine-safety-apiserver-statuscheck-timeout=30s \
			--machine-safety-apiserver-statuscheck-period=1m \
			--machine-safety-orphan-vms-period=30m \
			--leader-elect=$(leader_elect) \
			--v=3

```
entrypoint: 
`cmd/machine-controller/main.go`

- creates `machine-controller-manager/pkg/util/provider/app/options.mcserver` using `options.newmcserver` which is the main context object for the machinecontroller that embeds a
`machineconfig.machinecontrollerconfiguration`,   the `options.newmcserver` initializes `options.mcserver` with default values for `port: 10258`, `namespace: default`, `concurrentnodesyncs: 50`, `nodeconditions: "kerneldeadlock,readonlyfilesystem,diskpressure,networkunavailable"`, `minresyncperiod: 12 hours`, `kubeapiqps: 20`, `kubeapiburst:30` 

- calls `mcserver.addflags` which defines all parsing flags for the machine controller into fields of `mcserver` instance created in the last step.

- calls `k8s.io/component-base/logs.newoptions` and then `options.addflags` for logging options. probably get rid of this when moving to `logr`. see [logging in gardener components](https://github.com/gardener/gardener/blob/master/docs/development/logging.md). then use the [logcheck](https://github.com/gardener/gardener/tree/master/hack/tools/logcheck)tool.
- calls `newdriver` with control kube config that creates a controller runtime client (`sigs.k8s.io/controller-runtime/pkg/client`) which then calls `pkg/local/driver.newdriver` passing the controlloer-runtime client which constructs a `localdriver` encapsulating the passed in client.
  - the `localdriver` implements [driver](https://github.com/gardener/machine-controller-manager/blob/master/pkg/util/provider/driver/driver.go#l28) is the facade for creation/deletion of vm's
- calls [app.run](https://github.com/gardener/machine-controller-manager/blob/master/pkg/util/provider/app/app.go#l77) passing in the previously created `mcserver` and `driver` instances.
- 

## machine controller loop

### app.run

`app.run` is the function that setups the main control loop of the machine controller server. 

- [app.run(s *options.mcserver, driver driver.driger)](https://github.com/gardener/machine-controller-manager/blob/master/pkg/util/provider/app/app.go#l77) is the common run loop for all machine controllers
- creates `targetkubeconfig` and `controlkubeconfig` of type `k8s.io/client-go/rest.config` from the target kube config path using `clientcmd.buildconfigfromflags`
- set fields such as `config.qps` and `config.burst`  in both `targetkubeconfig` and `controlkubeconfig` from the `options.newmcserver`
- create `kubeclientcontrol` from the `controlkubeconfig` using the standard client-go client factory metohd: `kubernetes.newforconfig` that returns a `client-go/kubernetes.clientset`
- similarly create another `clientset` named `leaderelectionclient` using `controlkubeconfig`
- start a go routine using the function `starthttp` that registers a bunch of http handlers for the go profiler, prometheus metrics and the health check.
- call `createrecorder` passing the `kubeclientcontrol` client set instance that returns a [client-go/tools/record.eventrecorder](https://github.com/kubernetes/client-go/blob/master/tools/record/event.go#l91)
  - creates a new `eventbroadcaster` of type [event.eventbroadcaster](https://github.com/kubernetes/client-go/blob/master/tools/record/event.go#l113)
  - set the logging function of the broadcaster to `klog.infof`.
  - sets the event slink using `startrecordingtoslink` passing the event interface as `kubeclient.corev1().restclient()).events("")`. effectively events will be published remotely.
  - returns the `record.eventrecorder` associated with the `eventbroadcaster` using `eventbroadcaster.newrecorder`
  - constructs the `run` anonmous function assigned to `run` variable which does the following:
    - initializes a `stop` receive channel.
    - creates a `controlmachineclientbuilder` using `machineclientbuilder.simpleclientbuilder` using the `controlkubeconfig`.
    - creates a `controlcoreclientbuidler` using `coreclientbuilder.simplecontrollerclientbuilder` wrapping `controlkubeconfig`.
    - creates `targetcoreclientbuilder` using `coreclientbuilder.simplecontrollerclientbuilder` wrapping `controlkubeconfig`.
    - // imho far too many clients created
    - call the `startcontrollers` function passing the `mcserver`, `driver`, `controlkubeconfig`, `targetkubeconfig`, `controlmachineclientbuilder`, `controlcoreclientbuilder`, `targetcoreclientbuilder`, `recorder` and `stop` channel.
      - // ?: if you are going to pass the controlkubeconfig and targetkubeconfig - why not create the client builders inside the startcontrollers ?
      - if `startcontrollers` return an error panic and exit `run`.
  - use [leaderelection.runordie](https://github.com/kubernetes/client-go/blob/master/tools/leaderelection/leaderelection.go#l218) to start a leader election and pass the previously created `run` function to the leader callbacks for `onstartedleading`. `onstartedleading` is called when a leaderelector client starts leading.

### app.startcontrollers
[app.startcontrollers](https://github.com/gardener/machine-controller-manager/blob/master/pkg/util/provider/app/app.go#l202) starts all controllers which are part fo machine controller. currently there is only one controller: the machine controller started.
- calls `getavailableresources` using the `controlcoreclientbuilder` that returns a `map[schema.groupversionresource]bool]` assigned to `availableresources`
  - `getavailableresources` waits till the api server is running by checking its `/healthz` using `wait.pollimmediate`. keeps re-creating the client using `clientbuilder.client` method. 
  - then uses `clientset.interface.discovery.serverresources` to get a `[]*metav1.apiresourcelist` (which encapsulates a `[]apiresources`) and then converts that to a `map[schema.groupversionresource]bool]` `
- creates a `controlmachineclient` which is a client for the controller crd types (type: `versioned.interface`) using `controlmachineclientbuilder.clientordie` using `machine-controller` as client name. this client targets the control cluster - ie the cluster holding the machine cr's.
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
func newcontroller(
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

##### 4.1  reconcileclustersecretkey

`reconcileclustersecretkey` basically adds the `mcfinalizer` (value: `machine.sapcloud.io/machine-controller`) to the list of `secret.finalizers` for all secrets that are referenced by machine classes within the same namespace.

```mermaid
%%{init: {'themevariables': { 'fontsize': '10px'}}}%%

flowchart td

a[ns, name = cache.splitmetanamespacekey]
b["sec=secretlister.secrets(ns).get(name)"]
c["machineclasses=findmachineclassforsecret(name)"]
d{machineclasses empty?}
e["updatesecretfinalizers(sec)"] 
f["secretq.addafter(key,10min)"]
z(("end"))
a-->b
b-->c
c-->d
d--yes-->z
d--no-->e
e--err-->f
e--success-->z
f-->z
```

##### 4.2  reconcileclustermachineclasskey 

`reconcileclustermachineclasskey` re-queues after 10seconds on failure and 10mins on success. ie the machine class key is regularly re-queued.  bad: we are not using the `machineutils.longretry|shortretry` constants here.

```mermaid
%%{init: {'themevariables': { 'fontsize': '10px'}}}%%

flowchart td

rok["machineclassqueue.addafter(clzkey, 10m)"]
rerr["machineclassqueue.addafter(clzkey, 10s)"]
a["ns,name=cache.splitmetanamespacekey(clzkey)"]
gmc["clz=machineclasslister.machineclasses(ns).get(name)"]
fm["machines=findmachinesforclass(clz)"]
checknildel{"clz.deletiontimestamp==nil?"}
checklen{"len(machines)>0"}
hasfin{"clz.hasfinalizer"}
addfin["addmachineclassfinalizers(clz)"]
delfin["deletemachineclassfinalizers(clz)"]
eqm1["machinequeue.add(machinekeys)"]
eqm2["machinequeue.add(machinekeys)"]
z(("end"))

rok-->z
rerr-->z
a-->gmc
gmc-->fm
fm--succ-->checklen
fm--err-->rerr
checklen--yes-->checknildel
checklen--no-->delfin
checknildel--yes-->hasfin
checknildel--no-->eqm2
hasfin--no-->addfin
hasfin--yes-->rok
addfin-->eqm1
eqm1-->rok
eqm2-->rerr
delfin-->rok
```
##### 4.3  reconcileclustermachinekey 


```mermaid
%%{init: {'themevariables': { 'fontsize': '10px'}}}%%

flowchart td


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

###### 4.3.1 controller.triggercreationflow

the creation flow adds policy and delegates to the `driver.createmachine`.



##### 4.4  reconcileclustermachinesafetyorphanvms
tbd

##### 4.5  reconcileclustermachinesafetyapiserver 
tbd



