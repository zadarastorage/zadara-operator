# Installing Zadara Operator using Helm

Due to poor support for Custom Resource Definitions in Helm 2,
Zadara Operator Helm chart supports only [Helm 3](https://helm.sh/docs/intro/install/).

## Install

Install Zadara Operator into `zadara` namespace, using command below.
No `values.yaml` is required to install.

```shell script
$ helm install zoperator --namespace zadara --create-namespace ./helm/zadara-operator/
NAME: zoperator
LAST DEPLOYED: Thu Jun 25 14:28:46 2020
NAMESPACE: zadara
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
##############################################################################
####   Successfully installed Zadara Operator                             ####
##############################################################################

User guide: https://github.com/zadarastorage/zadara-operator

...
```

## Verify installation

Verify that all Custom Resource Definitions are present:
```shell script
$ kubectl api-resources --api-group zadara.com
NAME                     SHORTNAMES   APIGROUP     NAMESPACED   KIND
appdefinitions                        zadara.com   false        AppDefinition
applicationclones                     zadara.com   true         ApplicationClone
applicationsnapshots                  zadara.com   true         ApplicationSnapshot
cloneconfigurations                   zadara.com   true         CloneConfiguration
invokers                              zadara.com   true         Invoker
snapshotconfigurations                zadara.com   true         SnapshotConfiguration
snapshotpolicies                      zadara.com   false        SnapshotPolicy
vpsas                                 zadara.com   false        Vpsa
```

Verify that Operator pod is running:
```shell script
$ kubectl get pods --namespace zadara
NAME                        READY   STATUS    RESTARTS   AGE
zoperator-f46797cbb-fnxgp   1/1     Running   0          39s
```

## Uninstall

```shell script
$ helm uninstall --namespace zadara zoperator
release "zoperator" uninstalled
```

### Notes

Uninstalling the chart _does not remove CRDs_ or associated Custom Resources (Vpsas, ApplicationSnapshots, etc.)

If you want CRDs to be removed, need to do so manually, using the following commands.

**Be careful!** Deleting ApplicationSnapshots and ApplicationClones removes all associated resources,
including cloned application, volume snapshots on VPSA and cloned volumes on VPSA.

```shell script
CRDS="invokers
	applicationclones
	cloneconfigurations
	applicationsnapshots
	snapshotconfigurations
	appdefinitions
	snapshotpolicies
	vpsas"
# Delete all Custom Resoures
for CRD in $CRDS; do
  kubectl delete $CRD --all --all-namespaces
done
# Delete Custom Resource Definitions
for CRD in $CRDS; do
  kubectl delete crd $CRD.zadara.com
done
```
