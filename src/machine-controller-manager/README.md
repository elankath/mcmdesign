- [Machine Controller Manager](#machine-controller-manager)
  - [MCM Launch](#mcm-launch)

# Machine Controller Manager

The Machine Controller Manager handles reconciliation of [MachineDeployment](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/apis/machine/v1alpha1#MachineDeployment) and [MachineSet](https://pkg.go.dev/github.com/gardener/machine-controller-manager@v0.47.0/pkg/apis/machine/v1alpha1#MachineSet) objects.

Ideally this should be the called `machine-deployment-controller` but the current name is a legacy holdover when all controllers were in one project module.

The Machine Controller Manager Entry Point is at [github.com/gardener/machine-controller-manager/cmd/machine-controller-manager/controller_manager.go](https://github.com/gardener/machine-controller-manager/blob/v0.47.0/cmd/machine-controller-manager/controller_manager.go#L40)


## MCM Launch








