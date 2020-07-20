# Examples

## Before installing Operator

- Setup Kubernetes node
- Clone this repository locally
- Create a VPSA,
make sure it supports SnapshotSet API (available since version 20.11),
create at least one Storage Pool on VPSA
- [Generate API Token](http://guides.zadarastorage.com/vpsa-guide/1908/managing-access-control.html#managing-user-passwords)
for VPSA, if you don't have one
- Verify that your Kubernetes nodes have ping to your VPSA

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
# Create Operator in "zadara" namespace
helm install zoperator --namespace zadara --create-namespace ./helm/zadara-operator/
# Set credentials and create a Vpsa custom resource
HOSTNAME=<SET HOSTNAME!>
TOKEN=<SET TOKEN!>
sed -i s/HOSTNAME_HERE/$HOSTNAME/ examples/common/vpsa.yaml
sed -i s/TOKEN_HERE/$TOKEN/ examples/common/vpsa.yaml
kubectl create -f examples/common/vpsa.yaml
```

```shell script
# Cleanup Operator
helm uninstall --namespace zadara zoperator
```

---

## Install Zadara Operator

Follow the instructions in [Helm installation manual](../docs/install_helm.md)

## Add a VPSA

Edit `examples/common/vpsa.yaml` and set VPSA credentials: `hostname`, `https` and `token`.

Typically, `https: true` is used when VPSA hostname is a DNS name like `vsa-00000042-zadara-test-01.zadaravpsa.com`, and `https: false` is used with IP address hostname.

Create VPSA Custom Resource in Kubernetes:

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

- [Using Application Snapshots and Clones with Cassandra](cassandra/README.md).
  This scenario demonstrates creating of snapshots and clones of a 2-replicas
  Cassandra StatefulSet, working with multiple recurring Snapshot Policies.

- [Using on-demand Application Snapshots for MySQL](mysql/README.md).
  This scenario features installing 1-replica MySQL from Helm charts repository,
  configuring default storage class, and creating on-demand snapshots and clones.

- [Migrate MySQL to another cluster](import/README.md).
  This scenario demonstrates migration of MySQL deployment, where the new deployment uses cloned volumes of the same VPSA (i.e compute resources migration).
  Also, this includes multiple examples of `zadara` CLI usage. 
  
## Cleanup

First, make sure you finished cleanup,
as described in application-specific scenarios (links in a previous section).

You can run `zadara status` to see all Zadara Custom Resources and related objects.
Output should be similar to the example below: Operator and CSI pods, Storage Classes (SC) and VPSAs (no ApplicationSnapshots, ApplicationClones, etc).
```shell script
$ zadara status
OPERATOR POD                    NODE            READY
zoperator-7db7c5ddf7-vx76j      k8s-base-master true

CSI POD                                                 PROVISIONER     NODE            READY
example-vpsa-csi-zadara-controller-6bfc6ddbb4-m6khj     csi.zadara.com  k8s-base-master true
example-vpsa-csi-zadara-node-smj86                      csi.zadara.com  k8s-base-master true

SC                                 PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
example-vpsa-pool-00010003-block   csi.zadara.com   Retain          Immediate           true                   2d19h
example-vpsa-pool-00010003-nas     csi.zadara.com   Retain          Immediate           true                   2d19h

VPSAS          STATUS     PROVISIONER      HOSTNAME      AUTOEXPAND   AGE
example-vpsa   CsiReady   csi.zadara.com   10.10.10.10   true         2d19h
```

Delete VPSA
```shell script
$ kubectl delete -f examples/common/vpsa.yaml
vpsa.zadara.com "example-vpsa" deleted
```

To delete Operator follow the instructions in [Helm installation manual](../docs/install_helm.md#uninstall)
