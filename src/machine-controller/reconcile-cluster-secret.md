# Reconcile Cluster Secret

`reconcileClusterSecretKey` reconciles an secret due to controller resync
or an event on the secret
```go
func (c *controller) reconcileClusterSecretKey(key string) error 
// which looks up secret and delegates to
func (c *controller) reconcileClusterSecret(ctx context.Context, secret *corev1.Secret) error 
```
## Usage
Worker go-routines are created for this as below

```go
createWorker(c.secretQueue, 
    "ClusterSecret", 
    maxRetries, 
    true, 
    c.reconcileClusterSecretKey,
     stopCh, 
     &waitGroup)
```
## Flow

[controller.reconcileClusterSecretkey](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/pkg/util/provider/machinecontroller/secret.go#L37)
 basically adds the [MCFinalizerName](../mcm_facilities.md#finalizers)  (value: `machine.sapcloud.io/machine-controller`) to the list of `secret.finalizers` for all secrets that are referenced by machine classes within the same namespace.

```mermaid
%%{init: {'themeVariables': { 'fontSize': '10px'}, "flowchart": {"useMaxWidth": false }}}%%

flowchart TD

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