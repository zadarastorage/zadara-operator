# Examples

## Before installing Operator

- Setup Kubernetes node
- Clone this repository locally
- Create a VPSA,
make sure it supports SnapshotSet API (TODO-dev: add instructions),
create at least one Storage Pool on VPSA
- [Generate API Token](http://guides.zadarastorage.com/vpsa-guide/1908/managing-access-control.html#managing-user-passwords)
for VPSA, if you don't have one
- Verify that your Kubernetes node has ping to your VPSA

## Common flow

All examples use the same workflow:

1. Install Zadara Operator

2. Add a VPSA

3. Deploy stateful application

    1. Create PVC, or write PVC template, referencing a StorageClass of the VPSA created in (2)

    2. Deploy StatefulSet

4. Store data on Persistent Volume created for the application

5. Create Application Snapshot

    1. Create SnapshotPolicy

    2. Create AppDefinition to identify the application API objects

    3. Attach snapshot policies to the application by creating SnapshotConfiguration

    4. Activate SnapshotConfiguration by creating Invoker

6. Clone Application Snapshot

    1. Create CloneConfiguration for ApplicationSnapshot

    2. Activate CloneConfiguration by creating Invoker

## For the impatient

```shell script
# Replace here:
HOSTNAME=<SET HOSTNAME!>
TOKEN=<SET TOKEN!>
# Create and configure Operator
cd zoperator
sed -i s/HOSTNAME_HERE/$HOSTNAME/ examples/common/vpsa.yaml
sed -i s/TOKEN_HERE/$TOKEN/ examples/common/vpsa.yaml
make kreate
kubectl create -f examples/common/vpsa.yaml
```

```shell script
# Cleanup Operator
kubectl delete -f examples/common/vpsa.yaml
make klean
```

---

## Install Zadara Operator

```shell script
$ cd zoperator
$ make kreate
customresourcedefinition.apiextensions.k8s.io/appdefinitions.zadara.com created
customresourcedefinition.apiextensions.k8s.io/snapshotpolicies.zadara.com created
customresourcedefinition.apiextensions.k8s.io/applicationclones.zadara.com created
customresourcedefinition.apiextensions.k8s.io/invokers.zadara.com created
customresourcedefinition.apiextensions.k8s.io/cloneconfigurations.zadara.com created
customresourcedefinition.apiextensions.k8s.io/applicationsnapshots.zadara.com created
customresourcedefinition.apiextensions.k8s.io/snapshotconfigurations.zadara.com created
clusterrole.rbac.authorization.k8s.io/zoperator created
clusterrolebinding.rbac.authorization.k8s.io/zoperator created
deployment.apps/zoperator created
configmap/zoperator-config-map created
customresourcedefinition.apiextensions.k8s.io/vpsas.zadara.com created
```

TODO-dev: create a Helm chart

## Add a VPSA

Edit `examples/common/vpsa.yaml` and set VPSA credentials: `hostname`, `https` and `token`.

Create VPSA definition in Kubernetes:

```shell script
$ kubectl create -f examples/common/vpsa.yaml
vpsa.zadara.com/example-vpsa created
```

## Creating Snapshots and Clones of stateful Application

This part depends on application, and example scenario.

Before proceeding, make sure you have Operator running, with VPSA in "CsiReady" state.
```shell script
$ kubectl get vpsas -o wide
NAME           STATUS     PROVISIONER      HOSTNAME    AGE
example-vpsa   CsiReady   csi.zadara.com   10.0.8.32   4m27s
```
*Note: it may take a few minutes for CSI to start,
especially the first time, when Kubernetes needs to pull all container images.*

##### Example scenarios:

- [Using Application Snapshots and Clones with Cassandra](cassandra/README.md)

- [Using on-demand Application Snapshots for MySQL](mysql/README.md)


## Cleanup

First, make sure you finished cleanup,
as described in application-specific scenarios (links in a previous section)

Delete VPSA
```shell script
$ kubectl delete -f examples/common/vpsa.yaml
vpsa.zadara.com "example-vpsa" deleted
```

Delete Operator
```shell script
$ make klean
customresourcedefinition.apiextensions.k8s.io "appdefinitions.zadara.com" deleted
customresourcedefinition.apiextensions.k8s.io "snapshotpolicies.zadara.com" deleted
customresourcedefinition.apiextensions.k8s.io "applicationclones.zadara.com" deleted
customresourcedefinition.apiextensions.k8s.io "invokers.zadara.com" deleted
customresourcedefinition.apiextensions.k8s.io "cloneconfigurations.zadara.com" deleted
customresourcedefinition.apiextensions.k8s.io "applicationsnapshots.zadara.com" deleted
customresourcedefinition.apiextensions.k8s.io "snapshotconfigurations.zadara.com" deleted
clusterrole.rbac.authorization.k8s.io "zoperator" deleted
clusterrolebinding.rbac.authorization.k8s.io "zoperator" deleted
deployment.apps "zoperator" deleted
configmap "zoperator-config-map" deleted
customresourcedefinition.apiextensions.k8s.io "vpsas.zadara.com" deleted
```
TODO-dev: delete using Helm chart
