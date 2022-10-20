
- [MCM Facilities](#mcm-facilities)
	- [Machine Controller Core Types](#machine-controller-core-types)
		- [Machine](#machine)
			- [MachineSpec](#machinespec)
			- [MachineStatus](#machinestatus)
			- [Extended NodeConditionTypes](#extended-nodeconditiontypes)
			- [LastOperation](#lastoperation)
				- [MachineState (should be called MachineOperationState)](#machinestate-should-be-called-machineoperationstate)
				- [MachineOperationType](#machineoperationtype)
			- [CurrentStatus](#currentstatus)
				- [MachinePhase](#machinephase)
			- [Finalizers](#finalizers)
		- [MachineClass](#machineclass)
			- [NodeTemplate](#nodetemplate)
		- [MachineSet](#machineset)
		- [MachineDeployment](#machinedeployment)
	- [Utilities](#utilities)
		- [NodeOps](#nodeops)
			- [nodeops.GetNodeCondition](#nodeopsgetnodecondition)
			- [nodeops.CloneAndAddCondition](#nodeopscloneandaddcondition)
			- [nodeops.AddOrUpdateConditionsOnNode](#nodeopsaddorupdateconditionsonnode)
		- [MachineUtils](#machineutils)
			- [Operation Descriptions](#operation-descriptions)
			- [Retry Periods](#retry-periods)
		- [Misc](#misc)
			- [permits.PermitGiver](#permitspermitgiver)
				- [permits.NewPermitGiver](#permitsnewpermitgiver)
	- [Main Server Structs](#main-server-structs)
		- [MCServer](#mcserver)
			- [MCServer Usage](#mcserver-usage)
			- [MachineControllerConfiguration struct](#machinecontrollerconfiguration-struct)
				- [SafetyOptions](#safetyoptions)
	- [Controller Structs](#controller-structs)
		- [Machine Controller Core Struct](#machine-controller-core-struct)
	- [Driver](#driver)
	- [Codes and (error) Status](#codes-and-error-status)
		- [Code](#code)
		- [Status](#status)

# MCM Facilities

This chapter describes the core types and utilities present in the MCM module and used by the controllers and drivers.

## Machine Controller Core Types

The relevant machine types managed by the MCM controlled reside in [github.com/gardener/machine-controller-manager/pkg/apis/machine/v1alpha1](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/apis/machine/v1alpha1). This follows the standard location for client gen types `<module>/pkg/apis/<group>/<version>`.

Example: `Machine` type is [pkg/apis/machine/v1alpha1.Machine](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/apis/machine/v1alpha1#Machine)


### Machine

[Machine](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/apis/machine/v1alpha1#Machine) is the representation of a physical or virtual machine that corresponds to a front-end k8s node object. An example YAML looks like the below
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

A `Machine` has a `Spec` field represented by [MachineSpec](#machinespec)
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

```mermaid
graph TB
    subgraph Machine
    ObjectMeta
    TypeMeta
    MachineSpec
    MachineStatus
    end
```

#### MachineSpec

[MachineSpec](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/apis/machine/v1alpha1/machine_types.go#L54) represents the specification of a Machine.

```go
type MachineSpec struct {

	// Class is the referrent to the MachineClass. 
	Class ClassSpec 

    // Unique identification of the VM at the cloud provider
	ProviderID string 

	// NodeTemplateSpec describes the data a node should have when created from a template
	NodeTemplateSpec NodeTemplateSpec 

	// Configuration for the machine-controller.  
	*MachineConfiguration 
}
type NodeTemplateSpec struct { // BADLY NAMED!
	metav1.ObjectMeta

	// NodeSpec describes the attributes that a node is created with.
	Spec corev1.NodeSpec
}
```

- `ProviderID` is the unique identification of the VM at the cloud provider. `ProviderID` typically matches with the `node.Spec.ProviderID` on the node object.
- `Class` field is of type `ClassSpec` which is just the (`Kind` and the `Name`) referring to the `MachineClass`. (Ideally the field, type should have been called `ClassReference`) like `OwnerReference`
- `NodeTemplateSpec` describes the data a node should have when created from a template, embeds `ObjectMeta` and holds a [corev1.NodeSpec](https://pkg.go.dev/k8s.io/api/core/v1#NodeSpec) in its `Spec` field.  
  - The `Machine.Spec.NodeTemplateSpec.Spec` mirrors k8s `Node.Spec`
- `MachineSpec` embeds a `MachineConfiguration` which is just a configuration object that is a connection of timeouts, maxEvictRetries and NodeConditions

```mermaid
graph TB
    subgraph MachineSpec
	Class:ClassSpec
	ProviderID:string
	NodeTemplateSpec:NodeTemplateSpec
	MachineConfiguration
    end
```

```go
type MachineConfiguration struct {
	// MachineDraintimeout is the timeout after which machine is forcefully deleted.
	MachineDrainTimeout *Duration

	// MachineHealthTimeout is the timeout after which machine is declared unhealhty/failed.
	MachineHealthTimeout *Duration 

	// MachineCreationTimeout is the timeout after which machinie creation is declared failed.
	MachineCreationTimeout *Duration 

	// MaxEvictRetries is the number of retries that will be attempted while draining the node.
	MaxEvictRetries *int32 

	// NodeConditions are the set of conditions if set to true for MachineHealthTimeOut, machine will be declared failed.
	NodeConditions *string 
}
```


#### MachineStatus

[pkg/apis/machine/v1alpha1.MachineStatus](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/apis/machine/v1alpha1/machine_types.go#L97) represents the most recently observed status of Machine.
```go
type MachineStatus struct {
	// Node string. 
	Node string // TODO: describe me more

	// Conditions of this machine, same as NodeStatus.Conditions
	Conditions []NodeCondition 

	// Last operation refers to the status of the last operation performed. NOTE: this is usually the NextOperation for reconcile!! Discuss!
	LastOperation LastOperation 

	// Current status of the machine object
	CurrentStatus CurrentStatus

	// LastKnownState can store details of the last known state of the VM by the plugins.
	// It can be used by future operation calls to determine current infrastucture state
	LastKnownState string 
}
```
#### Extended NodeConditionTypes

The MCM extends standard k8s [NodeConditionType](./k8s_facilities.md#nodeconditiontype) with several custom conditions. (TODO: isn't this hacky/error prone?)

-  [NodeTerminationCondition]() defined as `	NodeTerminationCondition v1.NodeConditionType = "Terminating"`


NodeTerminationCondition


#### LastOperation

[github.com/machine-controller-manager/pkg/apis/machine/v1alpha1.LastOperation](https://pkg.go.dev/github.com/gardener/machine-controller-manager/pkg/apis/machine/v1alpha1#LastOperation) represents the last operation performed on the object
```go
type LastOperation struct {
	// Description of the operation
	Description string 

	// Last update time of operation
	LastUpdateTime Time 

	// State of operation (bad naming)
	State MachineState 

	// Type of operation
	Type MachineOperationType 
}

```
##### MachineState (should be called MachineOperationState)

NOTE: BADLY NAMED: Should be called `MachineOperationState`

[machine-controller-manager/pkg/apis/machine/v1alpha1.MachineState](https://pkg.go.dev/github.com/gardener/machine-controller-manager/pkg/apis/machine/v1alpha1#MachineState) represents the  current state of a machine operation and is one of `Processing`, `Failed` or `Successful`.

```go
// MachineState is  current state of the machine.
// BAD Name: Should be MachineOperationState
type MachineState string

// These are the valid (operation) states of machines.
const (
	// MachineStatePending means there are operations pending on this machine state
	MachineStateProcessing MachineState = "Processing"

	// MachineStateFailed means operation failed leading to machine status failure
	MachineStateFailed MachineState = "Failed"

	// MachineStateSuccessful indicates that the node is not ready at the moment
	MachineStateSuccessful MachineState = "Successful"
)
```
##### MachineOperationType

[github.com/machine-controller-manager/pkg/apis/machine/v1alpha1.MachineOperationType](https://pkg.go.dev/github.com/gardener/machine-controller-manager/pkg/apis/machine/v1alpha1#MachineOperationType) is a label for the operation performed on a machine object: `Create`/`Update`/`HealthCheck`/`Delete`.

```go
type MachineOperationType string
const (
	// MachineOperationCreate indicates that the operation is a create
	MachineOperationCreate MachineOperationType = "Create"

	// MachineOperationUpdate indicates that the operation is an update
	MachineOperationUpdate MachineOperationType = "Update"

	// MachineOperationHealthCheck indicates that the operation is a create
	MachineOperationHealthCheck MachineOperationType = "HealthCheck"

	// MachineOperationDelete indicates that the operation is a delete
	MachineOperationDelete MachineOperationType = "Delete"
)

```

#### CurrentStatus
[github.com/machine-controller-manager/pkg/apis/machine/v1alpha1.CurrentStatus](https://pkg.go.dev/github.com/gardener/machine-controller-manager/pkg/apis/machine/v1alpha1#CurrentStatus) encapsulates information about the current status of Machine.

```go
type CurrentStatus struct {
	// Phase refers to the Machien (Lifecycle) Phase
	Phase MachinePhase 
	// TimeoutActive when set to true drives the machine controller
	// to check whether	machine failed the configured creation timeout or health check timeout and change machine phase to Failed.
	TimeoutActive bool 
	// Last update time of current status
	LastUpdateTime Time
}
```

##### MachinePhase

`MachinePhase` is a label for the life-cycle phase of a machine at a given time: `Unknown`, `Pending`, `Available`, `Running`, `Terminating`, `Failed`, `CrashLoopBackOff`,.
```go
type MachinePhase string
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

#### Finalizers

See [K8s Finalizers](./k8s_facilities.md#finalizers-and-deletion). The MC defines the following finalizer keys

```go
const (
	MCMFinalizerName = "machine.sapcloud.io/machine-controller-manager"
	MCFinalizerName = "machine.sapcloud.io/machine-controller"
)
```
1. `MCMFinalizerName` is the finalizer used to tag dependecies before deletion
2. `MCFinalizerName` is the finalizer added on `Secret` objects.

### MachineClass

A [MachineClass](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/apis/machine/v1alpha1#MachineClass) is used to used to templatize and re-use provider configuration across multiple Machines or MachineSets or MachineDeployments

<details>
<summary>Example MachineClass YAML for AWS provider</summary>

```yaml
apiVersion: machine.sapcloud.io/v1alpha1
credentialsSecretRef:
  name: cloudprovider
  namespace: shoot--i544024--hana-test
kind: MachineClass
metadata:
  creationTimestamp: "2022-10-20T13:08:07Z"
  finalizers:
  - machine.sapcloud.io/machine-controller-manager
  generation: 1
  labels:
    failure-domain.beta.kubernetes.io/zone: eu-west-1c
  name: shoot--i544024--hana-test-whana-z1-b0f23
  namespace: shoot--i544024--hana-test
  resourceVersion: "38424578"
  uid: 656f863e-5061-420f-8710-96dcc9777be4
nodeTemplate:
  capacity:
    cpu: "4"
    gpu: "1"
    memory: 61Gi
  instanceType: p2.xlarge
  region: eu-west-1
  zone: eu-west-1c
provider: AWS
providerSpec:
  ami: ami-0c3484dbcde4c4d0c
  blockDevices:
  - ebs:
      deleteOnTermination: true
      encrypted: true
      volumeSize: 50
      volumeType: gp3
  iam:
    name: shoot--i544024--hana-test-nodes
  keyName: shoot--i544024--hana-test-ssh-publickey
  machineType: p2.xlarge
  networkInterfaces:
  - securityGroupIDs:
    - sg-0445497aa49ddecb2
    subnetID: subnet-04a338730d20ea601
  region: eu-west-1
  srcAndDstChecksEnabled: false
  tags:
    kubernetes.io/arch: amd64
    kubernetes.io/cluster/shoot--i544024--hana-test: "1"
    kubernetes.io/role/node: "1"
    networking.gardener.cloud/node-local-dns-enabled: "true"
    node.kubernetes.io/role: node
    worker.garden.sapcloud.io/group: whana
    worker.gardener.cloud/cri-name: containerd
    worker.gardener.cloud/pool: whana
    worker.gardener.cloud/system-components: "true"
secretRef:
  name: shoot--i544024--hana-test-whana-z1-b0f23
  namespace: shoot--i544024--hana-test
```
</details>

<br />

Type definition for `MachineClass` shown below
```go
type MachineClass struct {
	metav1.TypeMeta 
	metav1.ObjectMeta 

	ProviderSpec runtime.RawExtension `json:"providerSpec"`
	SecretRef *corev1.SecretReference `json:"secretRef,omitempty"`
	CredentialsSecretRef *corev1.SecretReference
	Provider string 


	NodeTemplate *NodeTemplate 
}
```
Notes
 - As can be seen above , the provider specific configuration to create a node is specified in `MachineClass.ProviderSpec` and this is of extensible custom type [runtime.RawExtension](./k8s_facilities.md#rawextension). This permits instances of different structure types like AWS [aws/api/AWSProviderSpec](https://pkg.go.dev/github.com/gardener/machine-controller-manager-provider-aws@v0.14.0/pkg/aws/apis#AWSProviderSpec) or [azure/apis.AzureProviderSpec](https://github.com/gardener/machine-controller-manager-provider-azure/blob/v0.9.0/pkg/azure/apis/azure_provider_spec.go#L38) to be held within a single type.
 - `MachineClass.SecretRef` stores the necessary secrets such as cloud credentials or userdata (holding bootstrap tokens) 
   - `MachineClass.CredentialsSecretRef` can optionally store the credentials if it is not set in the secret ref. (TODO: Unsure why)
 - `MachineClass.Provider` is specified as the combination of name and location of cloud-specific drivers. (Unsure if this is being actually being followed as for AWS&Azure this is set to simply `AWS` and `Azure` respectively)

#### NodeTemplate

A [NodeTemplate](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/apis/machine/v1alpha1#NodeTemplate) as shown below
1. `Capacity` is of type [k8s.io/api/core/v1.ResourceList](https://pkg.go.dev/k8s.io/api/core/v1#ResourceList) which is effectively a map of (resource name, quantity) pairs. Similar to [Node Capacity](./k8s_facilities.md#capacity), it has keys like `cpu`, `gpu`, `memory`, `storage`, etc and string values that are represented by [resource.Quantity](https://pkg.go.dev/k8s.io/apimachinery/pkg/api/resource#Quantity)
2. `InstanceType` provider specific Instance type of the node belonging to nodeGroup. For AWS this would be the EC2 instance type like `p2.xlarge` for AWS.
3. `Region`: provider specific region name like `eu-west-1` for AWS.
4. `Zone`: provider specified Availability Zone like `eu-west-1c` for AWS.
```go
type NodeTemplate struct {
	Capacity corev1.ResourceList 
	InstanceType string 
	Region string 
	Zone string 
}
```
Example
```yaml
nodeTemplate:
  capacity:
    cpu: "4"
    gpu: "1"
    memory: 61Gi
  instanceType: p2.xlarge
  region: eu-west-1
  zone: eu-west-1c

```
### MachineSet
### MachineDeployment

## Utilities

### NodeOps

#### nodeops.GetNodeCondition
[machine-controller-manager/pkg/util/nodeops.GetNodeCondition](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/nodeops/conditions.go#L72) get the nodes condition matching the specified type 

```go
func GetNodeCondition(ctx context.Context, c clientset.Interface, nodeName string, conditionType v1.NodeConditionType) (*v1.NodeCondition, error) {
	node, err := c.CoreV1().Nodes().Get(ctx, nodeName, metav1.GetOptions{})
	if err != nil {
		return nil, err
	}
	return getNodeCondition(node, conditionType), nil
}
func getNodeCondition(node *v1.Node, conditionType v1.NodeConditionType) *v1.NodeCondition {
	for _, cond := range node.Status.Conditions {
		if cond.Type == conditionType {
			return &cond
		}
	}
	return nil
}
```

#### nodeops.CloneAndAddCondition

[machine-controller-manager/pkg/util/nodeops.CloneAndAddCondition](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/nodeops/conditions.go#L31) adds a condition to the node condition slice. If condition with this type already exists, it updates the `LastTransitionTime`

```go
func CloneAndAddCondition(conditions []v1.NodeCondition, condition v1.NodeCondition) []v1.NodeCondition {
	if condition.Type == "" || condition.Status == "" {
		return conditions
	}
	var newConditions []v1.NodeCondition

	for _, existingCondition := range conditions {
		if existingCondition.Type != condition.Type { 
			newConditions = append(newConditions, existingCondition)
		} else { 
			// condition with this type already exists
			if existingCondition.Status == condition.Status 
			&& existingCondition.Reason == condition.Reason {
				// condition status and reason are  the same, keep existing transition time
				condition.LastTransitionTime = existingCondition.LastTransitionTime
			}
		}
	}
	newConditions = append(newConditions, condition)
	return newConditions
}
```
TODO: Bug ? Logic above will end up adding duplicate node condition if status and reason phrase are different


#### nodeops.AddOrUpdateConditionsOnNode
[machine-controller-manager/pkg/util/nodeops.AddOrUpdateConditionsOnNode](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/nodeops/conditions.go#L81) adds a condition to the node's status, retrying on conflict with backoff.

```go
func AddOrUpdateConditionsOnNode(ctx context.Context, c clientset.Interface, nodeName string, condition v1.NodeCondition) error 
```


```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%
flowchart TD

Init["backoff = wait.Backoff{
	Steps:    5,
	Duration: 100 * time.Millisecond,
	Jitter:   1.0,
}"]
-->retry.RetryOnConflict["retry.RetryOnConflict(backoff, fn)"]
-->GetNode["oldNode, err = c.CoreV1().Nodes().Get(ctx, nodeName, metav1.GetOptions{})"]
subgraph "fn"
    GetNode-->ChkIfErr{"err != nil"}
	ChkIfErr--Yes-->ReturnErr(("return err"))
	ChkIfErr--No-->InitNewNode["newNode := oldNode.DeepCopy()"]
	InitNewNode-->InitConditions["newNode.Status.Conditions:= CloneAndAddCondition(newNode.Status.Conditions, condition)"]
	-->UpdateNewNode["_, err := c.CoreV1().Nodes().UpdateStatus(ctx, newNodeClone, metav1.UpdateOptions{}"]
	-->ReturnErr
end
```


### MachineUtils

#### Operation Descriptions
`machineutils` has a bunch of constants that are descriptions of machine operations that are set into `machine.Status.LastOperation.Description` by the machine controller while performing reconciliation. It also has some [reason phrase](k8s_facilities.md#conditions)

```go
const (
	// GetVMStatus sets machine status to terminating and specifies next step as getting VMs
	GetVMStatus = "Set machine status to termination. Now, getting VM Status"

	// InitiateDrain specifies next step as initiate node drain
	InitiateDrain = "Initiate node drain"

	// InitiateVMDeletion specifies next step as initiate VM deletion
	InitiateVMDeletion = "Initiate VM deletion"

	// InitiateNodeDeletion specifies next step as node object deletion
	InitiateNodeDeletion = "Initiate node object deletion"

	// InitiateFinalizerRemoval specifies next step as machine finalizer removal
	InitiateFinalizerRemoval = "Initiate machine object finalizer removal"

	// LastAppliedALTAnnotation contains the last configuration of annotations, 
	// labels & taints applied on the node object
	LastAppliedALTAnnotation = "node.machine.sapcloud.io/last-applied-anno-labels-taints"

	// MachinePriority is the annotation used to specify priority
	// associated with a machine while deleting it. The less its
	// priority the more likely it is to be deleted first
	// Default priority for a machine is set to 3
	MachinePriority = "machinepriority.machine.sapcloud.io"

	// MachineClassKind is used to identify the machineClassKind for generic machineClasses
	MachineClassKind = "MachineClass"

	// MigratedMachineClass annotation helps in identifying machineClasses who have been migrated by migration controller
	MigratedMachineClass = "machine.sapcloud.io/migrated"

	// NotManagedByMCM annotation helps in identifying the nodes which are not handled by MCM
	NotManagedByMCM = "node.machine.sapcloud.io/not-managed-by-mcm"

	// TriggerDeletionByMCM annotation on the node would trigger the deletion of the corresponding machine object in the control cluster
	TriggerDeletionByMCM = "node.machine.sapcloud.io/trigger-deletion-by-mcm"

	// NodeUnhealthy is a node termination reason for failed machines
	NodeUnhealthy = "Unhealthy"

	// NodeScaledDown is a node termination reason for healthy deleted machines
	NodeScaledDown = "ScaleDown"

	// NodeTerminationCondition describes nodes that are terminating
	NodeTerminationCondition v1.NodeConditionType = "Terminating"
)

```

#### Retry Periods

These are standard retry periods that are internally used by the machine controllers to enqueue keys into the work queue after the specified duration so that reconciliation can be retried afer elapsed duration.

```go
// RetryPeriod is an alias for specifying the retry period
type RetryPeriod time.Duration

// These are the valid values for RetryPeriod
const (
	// ShortRetry tells the controller to retry after a short duration - 15 seconds
	ShortRetry RetryPeriod = RetryPeriod(15 * time.Second)
	// MediumRetry tells the controller to retry after a medium duration - 2 minutes
	MediumRetry RetryPeriod = RetryPeriod(3 * time.Minute)
	// LongRetry tells the controller to retry after a long duration - 10 minutes
	LongRetry RetryPeriod = RetryPeriod(10 * time.Minute)
)

```

###  Misc

#### permits.PermitGiver

`permits.PermitGiver` provides the ability to register, obtain, release and delete permits for a given key. All operations are concurrent safe.

```go
type PermitGiver interface {
	RegisterPermits(key string, numPermits int) //numPermits should be  maxNumPermits
	TryPermit(key string, timeout time.Duration) bool
	ReleasePermit(key string)
	DeletePermits(key string)
	Close()
}
```
The implementation of [github.com/gardener/machine-controller-manager/pkg/util/permits.PermitGiver](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/permits/permits.go#L29) maintains:
 -  a sync map of permit keys mapped to permits, where each permit is a structure comprising a buffered empty struct `struct{}` channel  with buffer size equalling the (max) number of permits. 
    -  `RegisterPermits(key, numPermits)` registers `numPermits` for the given `key`. A `permit` struct is initialized with `permit.c` buffer size as `numPermits`.
 -  entries are deleted if not accessed for a configured time represented by `stalePermitKeyTimeout`. This is done by 'janitor' go-routine associated with the permit giver instance.
 -  When attempting to get a permit, one writes an empty struct to the permit channel within a given timeout. If one can do so within the timeout one has acquired the permit, else not. 
    - `TryPermit(key, timeout)` attempts to get a permit for the given key by sending a `struct{}{}` instance to the buffered `permit.c` channel. 

```go 
type permit struct {
	// lastAcquiredPermitTime is time since last successful TryPermit or new RegisterPermits
	lastAcquiredPermitTime time.Time
	// c is initialized with buffer size N representing num permits
	c                      chan struct{} 
}
type permitGiver struct {
	keyPermitsMap sync.Map // map of string keys to permit struct values
	stopC         chan struct{}
}

```

##### permits.NewPermitGiver

`permits.NewPermitGiver` returns a new `PermitGiver`
```go
func NewPermitGiver(stalePermitKeyTimeout time.Duration, janitorFrequency time.Duration) PermitGiver
```

```mermaid

%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%

flowchart TB
	Begin((" "))
	-->InitStopCh["stopC := make(chan struct{})"]
	-->InitPG["
		pg := permitGiver{
		keyPermitsMap: sync.Map{},
		stopC:         stopC,
	}"]
	-->LaunchCleanUpGoroutine["go cleanup()"]
	LaunchCleanUpGoroutine-->Return(("return &pg"))
	LaunchCleanUpGoroutine--->InitTicker
	subgraph cleanup
	InitTicker["ticker := time.NewTicker(janitorFrequency)"]
	-->caseReadStopCh{"<-stopC ?"}
	caseReadStopCh--Yes-->End(("End"))
	caseReadStopCh--No
		-->ReadTickerCh{"<-ticker.C ?"}
	ReadTickerCh--Yes-->CleanupStale["
	pg.cleanupStalePermitEntries(stalePermitKeyTimeout)
	(Iterates keyPermitsMap, remove entries whose 
	lastAcquiredPermitTime exceeds stalePermitKeyTimeout)
	"]
	ReadTickerCh--No-->caseReadStopCh
	end
```


## Main Server Structs

### MCServer

[machine-controller-manager/pkg/util/provider/app/options.MCServer](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/app/options/options.go#L40)
is the main server context object for the machine controller. It embeds [options.MachineControllerConfiguration](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/options/types.go#L45) and has a `ControlKubeConfig` and `TargetKubeConfig` string fields.


```go
type MCServer struct {
	options.MachineControllerConfiguration

	ControlKubeconfig string
	TargetKubeconfig  string

```

The `MCServer` is constructed and initialized using the  [pkg/util/provider/app/options.NewMCServer](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/app/options/options.go#L48) function which sets most of the default values for fields for the embedded struct.

#### MCServer Usage

Individual providers leverage the MCServer as follows:
```go
  s := options.NewMCServer()
  driver := <providerSpecificDriverInitialization>
  if err := app.Run(s, driver); err != nil {
      fmt.Fprintf(os.Stderr, "%v\n", err)
      os.Exit(1)
  }
```
The MCServer is thus re-used across different providers.

#### MachineControllerConfiguration struct
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

## Controller Structs

### Machine Controller Core Struct

`controller` struct in package `controller` inside go file: `machine-controller-manager/pkg/util/provider/machinecontroller.go` (Bad convention) is the concrete Machine Controller struct that holds state data for the MC and implements the classifical controller `Run(workers int, stopCh <-chan struct{})` method.

The top level `MCServer.Run` method initializes this controller struct and calls its `Run` method

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

## Driver

[github.com/gardener/machine-controller-manager/pkg/util/provider/driver.Driver](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/util/provider/driver#Driver)  is the abstraction facade that decouplesthe machine controller from the cloud-provider specific machine lifecycle details. The MC invokes driver methods while performing reconciliation.

```go
type Driver interface {
	CreateMachine(context.Context, *CreateMachineRequest) (*CreateMachineResponse, error)
	DeleteMachine(context.Context, *DeleteMachineRequest) (*DeleteMachineResponse, error)
	GetMachineStatus(context.Context, *GetMachineStatusRequest) (*GetMachineStatusResponse, error)
	ListMachines(context.Context, *ListMachinesRequest) (*ListMachinesResponse, error)
	GetVolumeIDs(context.Context, *GetVolumeIDsRequest) (*GetVolumeIDsResponse, error)
	GenerateMachineClassForMigration(context.Context, *GenerateMachineClassForMigrationRequest) (*GenerateMachineClassForMigrationResponse, error)
}
```
- `GetVolumeIDs` returns a list volumeIDs for the given list of [PVSpecs](https://pkg.go.dev/k8s.io/api/core/v1#PersistentVolumeSpec)
  - Example: the AWS driver checks if `spec.AWSElasticBlockStore.VolumeID` is not nil and coverts the k8s `spec.AWSElasticBlockStore.VolumeID` to the EBS volume ID. Or if storage is provided by CSI `spec.CSI.Driver="ebs.csi.aws.com"` just gets `spec.CSI.VolumeHandle`
 - `GetMachineStatus` gets the status of the VM backing the machine object on the provider

## Codes and (error) Status

### Code
[github.com/gardener/machine-controller-manager/pkg/util/provider/machinecodes/codes.Code](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/util/provider/machinecodes/codes#Code) is a `uint32` with following error codes. These error codes are contained in errors returned from Driver methods.

Note: Un-happy with current design. It is clear that some error codes overlap each other in the sense that they are supersets of other codes. The right thing to do would have been to make an ErrorCategory.

| Value | Code                | Description              |
| ------| --------------------| ------------------------- 
| 0     | Ok                  | Success     
| 1     | Canceled            | the operation was canceled (by caller)
| 2     | Unknown             | Unknown error (unrecognized code)
| 3     | InvalidArgument     | InvalidArgument indicates client specified an invalid argument.
| 4     | DeadlineExceeded    | DeadlineExceeded means operation expired before completion.  For operations that change the state of the system, this error may be  returned even if the operation has completed successfully. For example, a successful response from a server could have been delayed long enough for the deadline to expire.
| 5     | NotFound            | requested entity not found.
| 6     | AlreadyExists       | an attempt to create an entity failed because one already exists.
| 7     | PermissionDenied    | caller does not have permission to execute the specified operation.
| 8     | ResourceExhausted   | indicates some resource has been exhausted, perhaps a per-user quota, or perhaps the entire file system is out of space.
| 9     | FailedPrecondition  | operation was rejected because the	 system is not in a state required for the operation's execution.
| 10    | Aborted             | operation was aborted and client should retry the full process
| 11    | OutOfRange          | operation was attempted past the valid range. Unlike InvalidArgument, this error indicates a problem that may be fixed if the system state changes.
| 12    | UnImplemented       | operation is not implemented or not supported
| 13    | Internal            | BAD. Some internal invariant broken. 
| 14    | Unavailable         | Service is currently unavailable (transient and op may be tried with backoff)
| 15    | DataLoss            | unrecoverable data loss or corruption.
| 16    | Unauthenticated     | request does not have valid creds for operation. Note: It would have been nice if 7 was called Unauthorized.

### Status

(OPINION: I beleive this is a minor NIH design defect. Ideally one should have re-levaraged the k8s API machinery Status [k8s.io/apimachinery/pkg/apis/meta/v1.Status](https://pkg.go.dev/k8s.io/apimachinery@v0.25.2/pkg/apis/meta/v1#Status)
instead of making custom status object.)

[status](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/util/provider/machinecodes/status) implements errors returned by MachineAPIs. MachineAPIs service handlers should return an error created by this package, and machineAPIs clients should expect a corresponding error to be returned from the RPC call.

[status.Status](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/util/provider/machinecodes/status#Status) implements `error` and encapsulates a `code` which should be onf the codes in [codes.Code](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/util/provider/machinecodes/codes#Code) and a develoer-facing error message in English
```go
type Status struct {
	code int32
	message string
}
// New returns a Status encapsulating code and msg.
func New(code codes.Code, msg string) *Status {
	return &Status{code: int32(code), message: msg}
}
```

NOTE: No ideally why we are doing such hard work as shown in [status.FromError](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/machinecodes/status/status.go#L84) which involves time-consuming regex parsing of an error string into a status. This is actually being used to parse error strings of errors returned by Driver methods. Not good - should be better designed.





