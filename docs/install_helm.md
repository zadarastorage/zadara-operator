# Installing Zadara Operator using Helm

Due to poor support for Custom Resource Definitions in Helm 2,
Zadara Operator Helm chart supports only [Helm 3](https://helm.sh/docs/intro/install/).

## Install

Install Zadara Operator into `zadara` namespace:

```shell script
$ helm install zoperator --namespace zadara --create-namespace ./helm/zadara-operator/
NAME: zoperator
LAST DEPLOYED: Thu Jun 18 11:08:10 2020
NAMESPACE: zadara
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
##############################################################################
####   Successfully installed Zadara Operator                             ####
##############################################################################

User guide: https://github.com/zadarastorage/zadara-operator

For your convenience, we suggest downloading CLI helper tool, available at:
https://github.com/zadarastorage/zadara-operator/tree/release/cli

Start with creating Vpsa:
...
```

No `values.yaml` is required to install.

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
