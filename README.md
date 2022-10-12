NOTE: This README.md is DEPRECATED

🚧 WIP: MCM Design Book in work at: [MCM Design Book](https://elankath.github.io/mcmdesign/))

(Please wait till 14th Oct to read for full MC/M Controller flows including node drain)

- [Machine Controller Manager](#machine-controller-manager)
	- [K8s Facilities](#k8s-facilities)
		- [k8s apimachinery](#k8s-apimachinery)
			- [Finalizers and Deletion](#finalizers-and-deletion)
			- [wait.Until](#waituntil)
		- [client-go k8s clients.](#client-go-k8s-clients)
		- [client-go Shared Informers.](#client-go-shared-informers)
			- [client-go workqueues](#client-go-workqueues)
		- [Pod Disruptions](#pod-disruptions)
	- [Controller Facilities](#controller-facilities)
		- [Machine Controller Core Types](#machine-controller-core-types)
			- [MachineClass](#machineclass)
			- [Machine](#machine)
				- [MachineSpec](#machinespec)
		- [Controller Client Facades](#controller-client-facades)
		- [Controller Informer Factories.](#controller-informer-factories)
		- [MCServer](#mcserver)
				- [MachineControllerConfiguration struct](#machinecontrollerconfiguration-struct)
				- [SafetyOptions](#safetyoptions)
			- [Controller Core Struct](#controller-core-struct)
		- [Controller Util Types](#controller-util-types)
			- [permits.PermitGiver](#permitspermitgiver)
			- [drain.NewVolumeAttachmentHandler](#drainnewvolumeattachmenthandler)
	- [Machine Phases and Statuses](#machine-phases-and-statuses)
	- [main flow](#main-flow)
	- [mcm provider local launch](#mcm-provider-local-launch)
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
	- [MCM Local Provider](#mcm-local-provider)
	- [Doubts.](#doubts)
		- [Dead Code](#dead-code)
			- [Dead? reconcileClusterNodeKey](#dead-reconcileclusternodekey)
			- [Dead? machine.go | triggerUpdationFlow](#dead-machinego--triggerupdationflow)
		- [Duplicate Initialization of EventRecorder in MC](#duplicate-initialization-of-eventrecorder-in-mc)
		- [Q? Internal to External Scheme Conversion](#q-internal-to-external-scheme-conversion)
	- [TODO](#todo)
# Machine Controller Manager



Machine Controller Manager is a group of cooperative controllers that manage the lifecycle of the worker machines. 

`MachineControllerManager` controller reconciles a set of Custom Resources namely `MachineDeployment`, `MachineSet`.

The `MachineController` impelements the reconciliation loop for Machine objects but delegates creation/updation/deletion of Machines to the Driver facade. Driver implementations for providers (Local/Azure/AWS) are out of tree in `machine-controller-manager-provider-<providerName>` projects. Each provider starts its machine controller independently.

## K8s Facilities

This section describes the types, utils and functions provided by k8s [client-go](https://github.com/kubernetes/client-go) that are used by MCM and MC.


### k8s apimachinery

Read [k8s API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)

#### Finalizers and Deletion

Every k8s object has a `Finalizers []string` field that can be explicitly assigned by a controller. Every k8s object has a `DeletionTimestamp *Time` that is set by API Server when graceful deletion is requested.

These are part of the [k8s.io./apimachinery/pkg/apis/meta/v1.ObjectMeta](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#ObjectMeta) struct type which is embedded in all k8s objects. 

When you tell Kubernetes to delete an object that has finalizers specified for it, the Kubernetes API marks the object for deletion by populating `.metadata.deletionTimestamp` aka `Deletiontimestamp`, and returns a `202` status code (HTTP `Accepted`). The target object remains in a terminating state while the control plane takes the actions defined by the finalizers. After these actions are complete, the controller should removes the relevant finalizers from the target object. When the `metadata.finalizers` field is empty, Kubernetes considers the deletion complete and deletes the object.

#### wait.Until

[k8s.io/apimachinery/pkg/wait.Until](https://github.com/kubernetes/apimachinery/blob/v0.25.0/pkg/util/wait/wait.go#L91) loops until `stop` channel is closed, running `f` every given `period.` 

`func Until(f func(), period time.Duration, stopCh <-chan struct{})`

### client-go k8s clients.

k8s clients have the type  [k8s.io/client-go/kubernetes.ClientSet](https://pkg.go.dev/k8s.io/client-go/kubernetes#Clientset) and is actually a high-level client set facade encapsulating clients for the  `core`, `appsv1`, `discoveryv1`, `eventsv1`, `networkingv1`, `nodev1`, `policyv1`, `storagev1` api groups

```go
// Clientset contains the clients for groups. Each group has exactly one
// version included in a Clientset.
type Clientset struct {
    appsV1                       *appsv1.AppsV1Client
    coreV1                       *corev1.CoreV1Client
    discoveryV1                  *discoveryv1.DiscoveryV1Client
    eventsV1                     *eventsv1.EventsV1Client
    // ...
}
```

Each of these clients associated with api groups expose an interface named _GroupVersionInterface_ that in-turn provides further access a generic REST Interface and access to a typed interface containing getter/setter methods for objects within that API group.

For example [EventsV1Client](https://pkg.go.dev/k8s.io/client-go@v0.25.0/kubernetes/typed/events/v1#EventsV1Client) which is used to interact with features provided by the `events.k8s.io` group implements [EventsV1Interface](https://pkg.go.dev/k8s.io/client-go@v0.25.0/kubernetes/typed/events/v1#EventsV1Interface)

```go
type EventsV1Interface interface {
	RESTClient() rest.Interface // generic REST API access
	EventsGetter // typed interface access
}
// EventsGetter has a method to return a EventInterface.
type EventsGetter interface {
	Events(namespace string) EventInterface
}
```

k8s client-sets are created in 2 steps:
1.  Use `k8s.io/client-go/tools/
2. TODO: describe client creation.

### client-go Shared Informers.

The vital role of a Kubernetes controller is to watch objects for the desired state and the actual state, then send instructions to make the actual state be more like the desired state. The controller thus first needs to retrieve the object's information. Instead of making direct API calls using k8s listers/watchers, client-go controllers should use `SharedInformer`s.

[cache.SharedInformer](https://pkg.go.dev/k8s.io/client-go/tools/cache#SharedInformer) is a primitive exposed by `client-go` lib that maintains a local cache of k8s objects of a particular API group and kind/resource. (restricable by namespace/label/field selectors) which is linked to the authoritative state of the corresponding objects in the API server. 

Informers are used to reduced the load-pressure on the API Server and etcd.

All that is needed to be known at this point is that Informers internally watch for k8s object changes, update an internal indexed store and invoke registered event handlers. Client code must construct event handlers to inject the logic that one would like to execute when an object is Added/Updated/Deleted. 


```go
type SharedInformer interface {
	// AddEventHandler adds an event handler to the shared informer using the shared informer's resync period.  Events to a single handler are delivered sequentially, but there is no coordination between different handlers.
	AddEventHandler(handler cache.ResourceEventHandler)

	// HasSynced returns true if the shared informer's store has been
	// informed by at least one full LIST of the authoritative state
	// of the informer's object collection.  This is unrelated to "resync".
	HasSynced() bool

	// Run starts and runs the shared informer, returning after it stops.
	// The informer will be stopped when stopCh is closed.
	Run(stopCh <-chan struct{})
	//..
}
```

[cache.ResourceEventHandler](https://pkg.go.dev/k8s.io/client-go/tools/cache#ResourceEventHandler) handle notifications for events that happen to a resource.
```go
type ResourceEventHandler interface {
	OnAdd(obj interface{})
	OnUpdate(oldObj, newObj interface{})
	OnDelete(obj interface{})
}
```

[cache.ResourceEventHandlerFuncs](https://pkg.go.dev/k8s.io/client-go/tools/cache#ResourceEventHandlerFuncs) is an adapter to let you easily specify as many or as few of the notification functions as you want while still implementing `ResourceEventHandler`. Nearly all controllers code use instance of this adapter struct to create event handlers to register on shared informers.

```go
type ResourceEventHandlerFuncs struct {
	AddFunc    func(obj interface{})
	UpdateFunc func(oldObj, newObj interface{})
	DeleteFunc func(obj interface{})
}
```

#### client-go workqueues

The basic [workqueue.Interface](https://pkg.go.dev/k8s.io/client-go/util/workqueue#Interface) has the following methods:
```go
type Interface interface {
	Add(item interface{})
	Len() int
	Get() (item interface{}, shutdown bool)
	Done(item interface{})
	ShutDown()
	ShutDownWithDrain()
	ShuttingDown() bool
}
```
This is extended with ability to Add Item as a later time using the [workqueue.DelayingInterface](https://pkg.go.dev/k8s.io/client-go/util/workqueue#DelayingInterface). 
```go
type DelayingInterface interface {
	Interface
	// AddAfter adds an item to the workqueue after the indicated duration has passed
	// Used to requeue items after failueres to avoid ending in hot-loop
	AddAfter(item interface{}, duration time.Duration)
}
```
This is further extended with rate limiting using [workqueue.RateLimiter](https://pkg.go.dev/k8s.io/client-go/util/workqueue#RateLimiter)

```go
type RateLimiter interface {
 	// When gets an item and gets to decide how long that item should wait
	When(item interface{}) time.Duration
	// Forget indicates that an item is finished being retried.  Doesn't matter whether its for perm failing
	// or for success, we'll stop tracking it
	Forget(item interface{})
	// NumRequeues returns back how many failures the item has had
	NumRequeues(item interface{}) int
}

```


TODO: Move me below

The basic contract for a k8s-client controller is to specifiy callback functions that enqueue items on a rate-limited work queue and register these callback functions using a `SharedInformer`. Then the controller `Run` loop then picks up objects from the work queue using `Get` and reconciles them.

The controller implementation functions for `Add|UpdateFunc` usually enqueue the object on a rate limited work queue created using [workqueue.NewNamedRateLimitingQueue](https://pkg.go.dev/k8s.io/client-go/util/workqueue#NewNamedRateLimitingQueue)


TODO: Implementation details covered in appendix.

### Pod Disruptions

TODO: describe me more.
https://innablr.com.au/blog/what-is-pod-disruption-budget-in-k8s-and-why-to-use-it/
https://kubernetes.io/docs/concepts/workloads/pods/disruptions/

A disruption, for the purpose of this blog post, refers to an event when a pod needs to be killed and respawned.  PodDisruptionBudget (pdb) is a useful layer of defence provided by Kubernetes to deal with this kind of issue.


As an application owner, you can create a PodDisruptionBudget (PDB) for each application. A PDB limits the number of Pods of a replicated application that are down simultaneously from voluntary disruptions. For example, a quorum-based application would like to ensure that the number of replicas running is never brought below the number needed for a quorum. A web front end might want to ensure that the number of replicas serving load never falls below a certain percentage of the total.

A pdb defines the budget of voluntary disruption. In essence, a human operator is letting the cluster aware of a minimum threshold in terms of available pods that the cluster needs to guarantee in order to ensure a baseline availability or performance. The word budget is used as in error budget, in the sense that any voluntary disruption within this budget should be acceptable.

A PDB specifies the number of replicas that an application can tolerate having, relative to how many it is intended to have. For example, a Deployment which has a .spec.replicas: 5 is supposed to have 5 pods at any given time. If its PDB allows for there to be 4 at a time, then the Eviction API will allow voluntary disruption of one (but not two) pods at a time.


A typical pdb looks like

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: nginx
```

If we take a closer look at this sample, we will notice

It selects other resources based on labels
It demands that there needs to be at least one pod running.


## Controller Facilities


### Machine Controller Core Types

The MCM and MC API types are present in `pkg/apis/machine(group)/v1alpha1(version)/`.
They include the core k8s object types managed by the controller. (types that embed `pkg/apis/meta/v1.TypeMeta` and `pkg/apis/meta/v1.ObjectMeta`). See [k8s API Go types And Common Machinery](https://iximiuz.com/en/posts/kubernetes-api-go-types-and-common-machinery/) for understanding this.

This includes the top-level structs:
- `MachineClass`
- `Machine` 

#### MachineClass

TODO Describe me with AWS example and local example.

#### Machine

Machine is the representation of a physical or virtual machine that corresponds to a front-end k8s node object. An example YAML looks like the below
```yaml
apiVersion: machine.sapcloud.io/v1alpha1
kind: Machine
metadata:
  name: test-machine
  namespace: default
spec:
  class:
    kind: MachineClass
    name: test-class
```
[pkg/apis/machine/v1alpha1.Machine](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/apis/machine/v1alpha1/machine_types.go#L39)
```go
type Machine struct {
	// ObjectMeta for machine object
	metav1.ObjectMeta 

	// TypeMeta for machine object
	metav1.TypeMeta 

	// Spec contains the specification of the machine
	Spec MachineSpec 

	// Status contains fields depicting the status
	Status MachineStatus 
}

```
##### MachineSpec
MachineSpec is the specification of a Machine.

[pkg/apis/machine/v1alpha1.MachineSpec](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/apis/machine/v1alpha1/machine_types.go#L54)
```go
type MachineSpec struct {

	// Class contains the machineclass attributes of a machine. +optional
	Class ClassSpec 

	// ProviderID represents the provider's unique ID given to a machine. +optional
	ProviderID string 

	// NodeTemplateSpec describes the data a node should have when created from a template
	// +optional
	NodeTemplateSpec NodeTemplateSpec 

	// Configuration for the machine-controller.  // +optional
	*MachineConfiguration 
}
```
- `MachineSpec`, 

The package `pkg/apis/machine/v1alpha1` also contains `register.go` which must contain the following for controller client generation to work properly

- A public package const `GroupName` that declades the group name of the types managed in this package managed by the controller. For MCM this is: `const GroupName = "machine.sapcloud.io"`
- A public package variable `SchemaGroupVersion` of type `k8s.io/apimachinery/pkg/runtime/schema.GroupVersion` which will be used to register the above k8s objects. For MCM this is declared as: `var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: "v1alpha1"}`
- A function `func addKnownTypes(scheme *runtime.Scheme) error` that registers all the above objects using `k8s.io/apimachinery/pkg/runtime.scheme.AddKnownTypes(gv schema.GroupVersion, types ...Object`.
   - Ex: `scheme.AddKnownTypes(SchemeGroupVersion, &AWSMachineClass{}, &MachineClass{}, &Machine{}, &MachineList{},...`
   - Additionally uses `k8s.io/apimachinery/pkg/apis/meta/v1.AddToGroupVersion` to regisgter the `SchemeGroupVersion` as follows: `metav1.AddToGroupVersion(scheme, SchemeGroupVersion)`
 - A public package var `SchemeBuilder` that calls `k8s.io/apimachinery/pkg/runtime.NewSchemeBuilder"` passing in the `addKnownTypes` func: `SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)`



### Controller Client Facades
`pkg/client/clientset/versioned.Interface` which is a [generated](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/generating-clientset.md) client facade prouced using [client-gen](https://itnext.io/how-to-generate-client-codes-for-kubernetes-custom-resource-definitions-crd-b4b9907769ba)

`pkg/client/clientset/versioned.Interface`
```go
package versioned //usually aliased to simply clientset by callers
type Interface interface {
    // Discover() returns  DiscoveryInterface which holds the methods that discover server-supported API groups, versions and resources.
	Discovery() discovery.DiscoveryInterface 
	MachineV1alpha1() machinev1alpha1.MachineV1alpha1Interface
}
```
`MachineV1alpha1Interface` (GroupVersionInterface) is generated by the client-gen and is effectively a high-level facade that is a collection of individual getter interfaces corresponding to each of the k8s types declared in `pkg/apis/machine/v1alpha1(version)/`.

`pkg/client/clientset/versioned/typed/machine/v1alpha1/machine_client.go`
```go
package v1alpha1
type MachineV1alpha1Interface interface {
	RESTClient() rest.Interface
	AWSMachineClassesGetter
	AlicloudMachineClassesGetter
	AzureMachineClassesGetter
	GCPMachineClassesGetter
	MachinesGetter
	MachineClassesGetter
	MachineDeploymentsGetter
	MachineSetsGetter
	OpenStackMachineClassesGetter
	PacketMachineClassesGetter
}

```

`pkg/client/clientset/versioned/typed/machine/v1alpha1/machine.go`
```go
// MachinesGetter has a method to return a MachineInterface.
// A group's client should implement this interface.
type MachinesGetter interface {
	Machines(namespace string) MachineInterface
}
```

The `controller.ClientBuilder` is a client builder that produces these generated controller client facades.

`util/clientbuilder/machine/controller.ClientBuilder`
```go
type ClientBuilder interface {
	// Config returns a new restclient.Config with the given user agent name.
	Config(name string) (*restclient.Config, error)
	// ConfigOrDie return a new restclient.Config with the given user agent
	// name, or logs a fatal error.
	ConfigOrDie(name string) *restclient.Config
	// Client returns a new clientset.Interface with the given user agent
	// name.
	Client(name string) (versioned.Interface, error)
	// ClientOrDie returns a new clientset.Interface with the given user agent
	// name or logs a fatal error, destroying the computer and killing the
	// operator and programmer.
	ClientOrDie(name string) versioned.Interface

```

### Controller Informer Factories.

The client generation also creates informers and informer factories.

`pkg/client/informers/externalversions/v1apha1.Interface`
```go
// Interface provides access to all the informers in this group version.
type Interface interface {
	// AWSMachineClasses returns a AWSMachineClassInformer.
	AWSMachineClasses() AWSMachineClassInformer
	// AlicloudMachineClasses returns a AlicloudMachineClassInformer.
	AlicloudMachineClasses() AlicloudMachineClassInformer
	// AzureMachineClasses returns a AzureMachineClassInformer.
	AzureMachineClasses() AzureMachineClassInformer
	// GCPMachineClasses returns a GCPMachineClassInformer.
	GCPMachineClasses() GCPMachineClassInformer
	// Machines returns a MachineInformer.
	Machines() MachineInformer
	// MachineClasses returns a MachineClassInformer.
	MachineClasses() MachineClassInformer
	// MachineDeployments returns a MachineDeploymentInformer.
	MachineDeployments() MachineDeploymentInformer
	// MachineSets returns a MachineSetInformer.
	MachineSets() MachineSetInformer
	// OpenStackMachineClasses returns a OpenStackMachineClassInformer.
	OpenStackMachineClasses() OpenStackMachineClassInformer
	// PacketMachineClasses returns a PacketMachineClassInformer.
	PacketMachineClasses() PacketMachineClassInformer
}
```

Each Informer returned by this high-level facade provides access to a  `Lister` that allow listing k8s objects of that type using internal shared cache maintained by the informer.

`pkg/client/informers/externalversions/machine/v1alpha1.MachineInformer`
```go
// MachineInformer provides access to a shared informer and lister for
// Machines.
type MachineInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1alpha1.MachineLister
}

```

`pkg/client/listers/machine/v1alpha.MachineLister`
```go
// MachineLister helps list Machines.
// All objects returned here must be treated as read-only.
type MachineLister interface {
	// List lists all Machines in the indexer.
	// Objects returned here must be treated as read-only.
	List(selector labels.Selector) (ret []*v1alpha1.Machine, err error)
	// Machines returns an object that can list and get Machines.
	Machines(namespace string) MachineNamespaceLister
}

```

### MCServer

`machine-controller-manager/pkg/util/provider/app/options.MCServer`
is the main context object for the machine controller. It embeds `options.MachineControllerConfiguration` and has a `ControlKubeConfig` and `TargetKubeConfig` fields.

```go
type MCServer struct {
	options.MachineControllerConfiguration

	ControlKubeconfig string
	TargetKubeconfig  string
}
```

The `MCServer` is initialized using the  `pkg/util/provider/app/options.NewMCServer`  which sets most of the default values for fields for the embedded struct.

##### MachineControllerConfiguration struct
An imnportant struct that represents machine configuration that supports deep-coopying and is embedded within the `MCServer`

`machine-controller-manager/pkg/util/provider/options.MachineControllerConfiguration`
```go
type MachineControllerConfiguration struct {
	metav1.TypeMeta

	// namespace in seed cluster in which controller would look for the resources.
	Namespace string

	// port is the port that the controller-manager's http service runs on.
	Port int32
	// address is the IP address to serve on (set to 0.0.0.0 for all interfaces).
	Address string
	// CloudProvider is the provider for cloud services.
	CloudProvider string
	// ConcurrentNodeSyncs is the number of node objects that are
	// allowed to sync concurrently. Larger number = more responsive nodes,
	// but more CPU (and network) load.
	ConcurrentNodeSyncs int32

	// enableProfiling enables profiling via web interface host:port/debug/pprof/
	EnableProfiling bool
	// enableContentionProfiling enables lock contention profiling, if enableProfiling is true.
	EnableContentionProfiling bool
	// contentType is contentType of requests sent to apiserver.
	ContentType string
	// kubeAPIQPS is the QPS to use while talking with kubernetes apiserver.
	KubeAPIQPS float32
	// kubeAPIBurst is the burst to use while talking with kubernetes apiserver.
	KubeAPIBurst int32
	// leaderElection defines the configuration of leader election client.
	LeaderElection mcmoptions.LeaderElectionConfiguration
	// How long to wait between starting controller managers
	ControllerStartInterval metav1.Duration
	// minResyncPeriod is the resync period in reflectors; will be random between
	// minResyncPeriod and 2*minResyncPeriod.
	MinResyncPeriod metav1.Duration

	// SafetyOptions is the set of options to set to ensure safety of controller
	SafetyOptions SafetyOptions

	//NodeCondition is the string of known NodeConditions. If any of these NodeCondition is set for a timeout period, the machine  will be declared failed and will replaced. Default is "KernelDeadlock,ReadonlyFilesystem,DiskPressure,NetworkUnavailable"
	NodeConditions string

	//BootstrapTokenAuthExtraGroups is a comma-separated string of groups to set bootstrap token's "auth-extra-groups" field to.
	BootstrapTokenAuthExtraGroups string
}

```
##### SafetyOptions

An important struct availablea as the `SafetyOptions` field in `MachineControllerConfiguration` containing several timeouts, retry-limits, etc. Most of these fields are set via [pkg/util/provider/app/options.NewMCServer](https://github.com/gardener/machine-controller-manager/blob/master/pkg/util/provider/app/options/options.go#L48) function

`pkg/util/provider/option.SafetyOptions`
```go
// SafetyOptions are used to configure the upper-limit and lower-limit
// while configuring freezing of machineSet objects
type SafetyOptions struct {
	// Timeout (in durartion) used while creation of
	// a machine before it is declared as failed
	MachineCreationTimeout metav1.Duration
	// Timeout (in durartion) used while health-check of
	// a machine before it is declared as failed
	MachineHealthTimeout metav1.Duration
	// Deprecated. No effect. Timeout (in durartion) used while draining of machine before deletion,
	// beyond which it forcefully deletes machine
	MachineDrainTimeout metav1.Duration
	// Maximum number of times evicts would be attempted on a pod for it is forcibly deleted
	// during draining of a machine.
	MaxEvictRetries int32
	// Timeout (in duration) used while waiting for PV to detach
	PvDetachTimeout metav1.Duration
	// Timeout (in duration) used while waiting for PV to reattach on new node
	PvReattachTimeout metav1.Duration

	// Timeout (in duration) for which the APIServer can be down before
	// declare the machine controller frozen by safety controller
	MachineSafetyAPIServerStatusCheckTimeout metav1.Duration
	// Period (in durartion) used to poll for orphan VMs
	// by safety controller
	MachineSafetyOrphanVMsPeriod metav1.Duration
	// Period (in duration) used to poll for APIServer's health
	// by safety controller
	MachineSafetyAPIServerStatusCheckPeriod metav1.Duration

	// APIserverInactiveStartTime to keep track of the
	// start time of when the APIServers were not reachable
	APIserverInactiveStartTime time.Time
	// MachineControllerFrozen indicates if the machine controller
	// is frozen due to Unreachable APIServers
	MachineControllerFrozen bool
}
```

#### Controller Core Struct

`controller` struct in package `controller` inside go file: `machine-controller-manager/pkg/util/provider/machinecontroller.go` (Bad convention) is the concrete Machine Controller struct that holds state data for the MC and implements the classifical controller `Run(workers int, stopCh <-chan struct{})` method.

```go
package controller
type controller struct {
	namespace                     string // control clustern namespace
	nodeConditions                string // Default: "KernelDeadlock,ReadonlyFilesystem,DiskPressure,NetworkUnavailable"

	controlMachineClient    machineapi.MachineV1alpha1Interface
	controlCoreClient       kubernetes.Interface
	targetCoreClient        kubernetes.Interface
	targetKubernetesVersion *semver.Version

	recorder                record.EventRecorder
	safetyOptions           options.SafetyOptions
	internalExternalScheme  *runtime.Scheme
	driver                  driver.Driver
	volumeAttachmentHandler *drain.VolumeAttachmentHandler
	// permitGiver store two things:
	// - mutex per machinedeployment
	// - lastAcquire time
	// it is used to limit removal of `health timed out` machines
	permitGiver permits.PermitGiver

	// listers
	pvcLister               corelisters.PersistentVolumeClaimLister
	pvLister                corelisters.PersistentVolumeLister
	secretLister            corelisters.SecretLister
	nodeLister              corelisters.NodeLister
	pdbV1beta1Lister        policyv1beta1listers.PodDisruptionBudgetLister
	pdbV1Lister             policyv1listers.PodDisruptionBudgetLister
	volumeAttachementLister storagelisters.VolumeAttachmentLister
	machineClassLister      machinelisters.MachineClassLister
	machineLister           machinelisters.MachineLister
	// queues
    // secretQueue = workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "secret"),
	secretQueue                 workqueue.RateLimitingInterface
	nodeQueue                   workqueue.RateLimitingInterface
	machineClassQueue           workqueue.RateLimitingInterface
	machineQueue                workqueue.RateLimitingInterface
	machineSafetyOrphanVMsQueue workqueue.RateLimitingInterface
	machineSafetyAPIServerQueue workqueue.RateLimitingInterface
	// syncs
	pvcSynced               cache.InformerSynced
	pvSynced                cache.InformerSynced
	secretSynced            cache.InformerSynced
	pdbV1Synced             cache.InformerSynced
	volumeAttachementSynced cache.InformerSynced
	nodeSynced              cache.InformerSynced
	machineClassSynced      cache.InformerSynced
	machineSynced           cache.InformerSynced
}

```


### Controller Util Types

#### permits.PermitGiver
TODO: explain me

`machine-controller-manager/pkg/util/permits.PermitGiver`
```go 
// PermitGiver provides different operations regarding permit for a given key
type PermitGiver interface {
	RegisterPermits(key string, numPermits int)
	TryPermit(key string, timeout time.Duration) bool
	ReleasePermit(key string)
	DeletePermits(key string)
	Close()
}
type permit struct {
	lastAcquiredPermitTime time.Time
	c                      chan struct{}
}
```

#### drain.NewVolumeAttachmentHandler

TODO: explain me

VolumeAttachmentHandler is an handler used to distribute incoming VolumeAttachment requests to all listening workers
`machine-controller-manager/pkg/util/provider/drain.VolumeAttachmentHandler`
```go
type VolumeAttachmentHandler struct {
	sync.Mutex
	workers []chan *storagev1.VolumeAttachment
}
```

## Machine Phases and Statuses


A `Machine`
```go
const (
	// MachinePending means that the machine is being created
	MachinePending MachinePhase = "Pending"

	// MachineAvailable means that machine is present on provider but hasn't joined cluster yet
	MachineAvailable MachinePhase = "Available"

	// MachineRunning means node is ready and running successfully
	MachineRunning MachinePhase = "Running"

	// MachineRunning means node is terminating
	MachineTerminating MachinePhase = "Terminating"

	// MachineUnknown indicates that the node is not ready at the movement
	MachineUnknown MachinePhase = "Unknown"

	// MachineFailed means operation failed leading to machine status failure
	MachineFailed MachinePhase = "Failed"

	// MachineCrashLoopBackOff means creation or deletion of the machine is failing.
	MachineCrashLoopBackOff MachinePhase = "CrashLoopBackOff"
)

```

## main flow
macinedeployment provides a declarative update for machineset and machines. controllers work in a cooperative manner.

entrypoint:
[machine-controller-manager/controller_manager.go](https://github.com/gardener/machine-controller-manager/blob/master/cmd/machine-controller-manager/controller_manager.go)

uses [options.newmcmserver](https://github.com/gardener/machine-controller-manager/blob/master/cmd/machine-controller-manager/app/options/options.go#l48) to create a `mcmserver` which is the main context object for the  controller manager.

`mcmserver` consists of a [machineconfig.machinecontrollermanagerconfiguration](https://github.com/gardener/machine-controller-manager/blob/master/pkg/options/types.go#l44) a `controlkubeconfig` and a `targetkubeconfig`.



## mcm provider local launch

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

## MCM Local Provider 

`cmd/machine-controller/main.go`
Creates `pkg/util/provider/app/options.MCServer`

TODO: Driver CM/DM Diagrams

## Doubts.

### Dead Code

#### Dead? reconcileClusterNodeKey 

This just delegates to `reconcileClusterNode` which does nothing..
```go
func (c *controller) reconcileClusterNode(node *v1.Node) error {
	return nil
}

```

#### Dead? machine.go | triggerUpdationFlow
Can't find usages

### Duplicate Initialization of EventRecorder in MC

`pkg/util/provider/app.createRecorder` already dones this below.
```go
func createRecorder(kubeClient *kubernetes.Clientset) record.EventRecorder {
eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(klog.Infof)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: v1core.New(kubeClient.CoreV1().RESTClient()).Events("")})
	return eventBroadcaster.NewRecorder(kubescheme.Scheme, v1.EventSource{Component: controllerManagerAgentName})
}
```


We get the recorder from this eventBroadcaster and then pass it to the `pkg/util/provider/machinecontroller/controller.NewController` method which again does:
```go
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(klog.Infof)
	eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{Interface: typedcorev1.New(controlCoreClient.CoreV1().RESTClient()).Events(namespace)})
```
The above is useless.

### Q? Internal to External Scheme Conversion

Why do we do this ?
```go
internalClass := &machine.MachineClass{}
	err := c.internalExternalScheme.Convert(class, internalClass, nil)
	if err != nil {
		return err
	}
```


## TODO

- [ ] Migrate to mdbook chapters.
- [ ] Incorporate description of requests from: https://github.com/gardener/machine-controller-manager/blob/master/docs/development/machine_error_codes.md
- [ ] Clearly explain difference between control cluster and targer cluster.
- [ ] describe informers: https://www.cncf.io/blog/2019/10/15/extend-kubernetes-via-a-shared-informer/
  - [ ] https://medium.com/codex/explore-client-go-informer-patterns-4415bb5f1fbd
  - [ ] Execute the pdb-demo
- [ ] Add collapsible sections for large code
- [ ] Understand safety options: safety up and safety down and describe them.
- [ ] Figure out how to get full list of unused code using go staticcheck and cross-modules
- [ ] TODO: Incorporate knowledge of https://cloud.redhat.com/blog/kubernetes-deep-dive-code-generation-customresources into the above including how the informer types are created.
