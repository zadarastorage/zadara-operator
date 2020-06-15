
## Using Application Snapshots and Clones with Cassandra

In this example:

- deploy Cassandra as a StatefulSet with 2 replicas
- create an Application Snapshot including Persistent Volumes data and application configuration
- clone created Application Snapshot into a new Namespace

#### For the impatient

```shell script
# Deploy application
kubectl create -f examples/cassandra/cassandra-service.yaml
STORAGE_CLASS_NAME=$(kubectl get sc -l provisioner=csi.zadara.com | grep nas | awk '{print $1}')
sed -i s/STORAGE_CLASS_NAME_HERE/$STORAGE_CLASS_NAME/ examples/cassandra/cassandra-stateful-set.yaml
kubectl create -f examples/cassandra/cassandra-stateful-set.yaml
# Configure and create Snapshots
kubectl create -f examples/common/policy-every-ten-minutes.yaml
kubectl create -f examples/common/policy-every-two-minutes.yaml
kubectl create -f examples/cassandra/app-definition.yaml
kubectl create -n zadara -f examples/cassandra/snapshot-configuration.yaml
kubectl create -n zadara -f examples/cassandra/invoker-snapshot-cfg.yaml
kubectl get applicationsnapshots.zadara.com -n zadara --watch # wait for application snapshot to be created
# Configure and create Clone of a Snapshot
SNAPSHOT=$(kubectl get applicationsnapshots.zadara.com -n zadara | grep Created | tail -n1 | awk '{print $1}')
sed -i s/SNAPSHOT_NAME_HERE/$SNAPSHOT/ examples/cassandra/clone-configuration.yaml
kubectl create -n zadara -f examples/cassandra/clone-configuration.yaml
kubectl create -n zadara -f examples/cassandra/invoker-clone-cfg.yaml
```

```shell script
# Cleanup Clone and configuration
CLONE=$(kubectl get applicationclones.zadara.com -n zadara | tail -n1 | awk '{print $1}')
kubectl delete applicationclone -n zadara $CLONE
kubectl delete -n zadara -f examples/cassandra/invoker-clone-cfg.yaml
kubectl delete -n zadara -f examples/cassandra/clone-configuration.yaml
# Cleanup Snapshots and configuration
kubectl delete -n zadara -f examples/cassandra/invoker-snapshot-cfg.yaml
kubectl delete applicationsnapshots.zadara.com -n zadara -l zadara.com/snapshotConfiguration=cassandra-test-snapshot-cfg
kubectl delete -n zadara -f examples/cassandra/snapshot-configuration.yaml
kubectl delete -f examples/cassandra/app-definition.yaml
kubectl delete -f examples/common/policy-every-ten-minutes.yaml
kubectl delete -f examples/common/policy-every-two-minutes.yaml
# Delete application
kubectl delete -f examples/cassandra/cassandra-stateful-set.yaml
kubectl delete -f examples/cassandra/cassandra-service.yaml
kubectl delete pvc -l app=cassandra
```

---

### Prerequisites

Install Cassandra client CLI tool
```shell script
$ pip install cqlsh
```

### Deploy stateful application

Create a Service
```shell script
$ kubectl create -f examples/cassandra/cassandra-service.yaml
service/cassandra created
```

Choose StorageClass
```shell script
$ kubectl get sc
NAME                               PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
example-vpsa-pool-00010003-block   csi.zadara.com   Retain          Immediate           true                   14s
example-vpsa-pool-00010003-nas     csi.zadara.com   Retain          Immediate           true                   14s
```
Storage Classes as shown above are created automatically for each VPSA.
You can use one of them, or define a new one.

In this example we will use `example-vpsa-pool-00010003-nas`.
```shell script
$ STORAGE_CLASS_NAME=example-vpsa-pool-00010003-nas
$ sed -i s/STORAGE_CLASS_NAME_HERE/$STORAGE_CLASS_NAME/ examples/cassandra/cassandra-stateful-set.yaml
```

Verify that StatefulSet definition `examples/cassandra/cassandra-stateful-set.yaml` has
`storageClassName: example-vpsa-pool-00010003-nas` in `volumeClaimTemplates`
```shell script
$ grep storageClassName examples/cassandra/cassandra-stateful-set.yaml
        storageClassName: example-vpsa-pool-00010003-nas
```

Create StatefulSet
```shell script
$ kubectl create -f examples/cassandra/cassandra-stateful-set.yaml
statefulset.apps/cassandra created
```

Wait for it...
```shell script
$ kubectl get pods --watch
NAME                         READY   STATUS              RESTARTS   AGE
cassandra-0                  0/1     ContainerCreating   0          2s
zoperator-7dc67b756b-gc2gz   1/1     Running             0          38m
cassandra-0                  1/1     Running             0          16s
cassandra-1                  0/1     Pending             0          0s
cassandra-1                  0/1     Pending             0          0s
cassandra-1                  0/1     ContainerCreating   0          1s
cassandra-1                  1/1     Running             0          25s
```

Get Cassandra pod IP (any of them)
```shell script
$ kubectl get po -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
cassandra-0                  1/1     Running   0          6m52s   10.244.0.52   levp-k8s   <none>           <none>
cassandra-1                  1/1     Running   0          6m13s   10.244.0.53   levp-k8s   <none>           <none>
zoperator-7dc67b756b-gc2gz   1/1     Running   0          70m     10.244.0.46   levp-k8s   <none>           <none>
```

Connect to a Database

```shell script
$ cqlsh 10.244.0.52 --cqlversion 3.4.2
Connected to K8Demo at 10.244.0.52:9042.
[cqlsh 5.0.1 | Cassandra 3.9 | CQL spec 3.4.2 | Native protocol v4]
Use HELP for help.
cqlsh>
```

## Store data on Persistent Volume created for the application

TODO-dev: instructions to populate DB

## Create Application Snapshot

Create SnapshotPolicies. We will create two policies, creating snapshots frequently.
```shell script
$ kubectl create -f examples/common/policy-every-ten-minutes.yaml
snapshotpolicy.zadara.com/every-ten-minutes created
$ kubectl create -f examples/common/policy-every-two-minutes.yaml
snapshotpolicy.zadara.com/every-two-minutes created
```

Create AppDefinition to identify Cassandra application.
```shell script
$ kubectl create -f examples/cassandra/app-definition.yaml
appdefinition.zadara.com/cassandra-test-app-definition created
```

Attach snapshot policies to the application by creating SnapshotConfiguration.
```shell script
$ kubectl create -n zadara -f examples/cassandra/snapshot-configuration.yaml
snapshotconfiguration.zadara.com/cassandra-test-snapshot-cfg created
```

Activate SnapshotConfiguration by creating Invoker
```shell script
$ kubectl create -n zadara -f examples/cassandra/invoker-snapshot-cfg.yaml
invoker.zadara.com/cassandra-snapshot created
```

Wait for the first schedule
```shell script
$ kubectl get cronjobs -n zadara --watch
NAME                                            SCHEDULE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cassandra-test-snapshot-cfg-every-ten-minutes   */10 * * * *   False     0        <none>          64s
cassandra-test-snapshot-cfg-every-two-minutes   */2 * * * *    False     0        <none>          64s
cassandra-test-snapshot-cfg-every-two-minutes   */2 * * * *    False     1        9s              67s
```

Verify Application Snapshot creation
```shell script
$ kubectl get applicationsnapshots.zadara.com -n zadara
NAME                                               STATUS    SNAPSHOT CONFIGURATION
cassandra-test-snapshot-cfg-2020-04-01--15-44-09   Created   cassandra-test-snapshot-cfg
```

Verify that all API objects (Pods, PVCs, etc) are included. *API Objects not fully shown in this example, for readability*
```shell script
$ kubectl describe applicationsnapshots.zadara.com -n zadara cassandra-test-snapshot-cfg-2020-04-01--15-44-09
Name:         cassandra-test-snapshot-cfg-2020-04-01--15-44-09
Namespace:    zadara
Labels:       zadara.com/snapshotConfiguration=cassandra-test-snapshot-cfg
              zadara.com/snapshotPolicy=every-two-minutes
Annotations:  <none>
API Version:  zadara.com/v1alpha1
Kind:         ApplicationSnapshot
Metadata:
  Creation Timestamp:  2020-04-01T15:44:21Z
  Finalizers:
    zadara.com/appsnapshot-cleanup
  Generation:        3
  Resource Version:  5391175
  Self Link:         /apis/zadara.com/v1alpha1/namespaces/default/applicationsnapshots/cassandra-test-snapshot-cfg-2020-04-01--15-44-09
  UID:               2c859dd7-a81e-405e-9dd7-6f49f7fa3766
Spec:
  API Objects:
    {"apiVersion":"v1","kind":"Endpoints", ...}
    {"apiVersion":"v1","kind":"PersistentVolumeClaim", ...}
    {"apiVersion":"v1","kind":"PersistentVolumeClaim", ...}
    {"apiVersion":"v1","kind":"Pod", ...}
    {"apiVersion":"v1","kind":"Pod", ...}
    {"apiVersion":"v1","kind":"Service", ...}
    {"apiVersion":"v1","kind":"PersistentVolume", ...}
    {"apiVersion":"v1","kind":"PersistentVolume", ...}

  Snapshot Configuration:  cassandra-test-snapshot-cfg
  Vpsas:
    Name:         example-vpsa
    Provisioner:  csi.zadara.com
    Snapshots:
      Cg:  cg-00000003
      Id:  snap-00000009
      Cg:  cg-00000004
      Id:  snap-0000000a
Status:
  State:  Created
Events:   <none>
```

## Clone Application Snapshot

Choose Application Snapshot you want to clone
```shell script
$ kubectl get applicationsnapshots -n zadara
NAME                                               STATUS    SNAPSHOT CONFIGURATION
cassandra-test-snapshot-cfg-2020-04-02--13-16-01   Created   cassandra-test-snapshot-cfg
cassandra-test-snapshot-cfg-2020-04-02--13-18-02   Created   cassandra-test-snapshot-cfg
cassandra-test-snapshot-cfg-2020-04-02--13-20-03   Created   cassandra-test-snapshot-cfg
cassandra-test-snapshot-cfg-2020-04-02--13-20-04   Created   cassandra-test-snapshot-cfg
```

Edit `examples/cassandra/clone-configuration.yaml`, and set `appSnapshot.name`.
For example:
```shell script
$ SNAPSHOT=cassandra-test-snapshot-cfg-2020-04-02--13-20-04
$ sed -i s/SNAPSHOT_NAME_HERE/$SNAPSHOT/ examples/cassandra/clone-configuration.yaml
```

Create CloneConfiguration for ApplicationSnapshot
```shell script
$ kubectl create -n zadara -f examples/cassandra/clone-configuration.yaml
cloneconfiguration.zadara.com/cassandra-test-clone-cfg created
```

Activate CloneConfiguration and start clone process by creating Invoker
```shell script
$ kubectl create -n zadara -f examples/cassandra/invoker-clone-cfg.yaml
invoker.zadara.com/cassandra-clone created
```

In `examples/cassandra/clone-configuration.yaml` we defined the following:
```yaml
  targetNamespace: "cloned"
```
That is, resources from namespace `default` will be cloned into namespace `cloned`.

Verify that PVCs are created (note the namespace parameter: `-n cloned`)
```shell script
$ kubectl get pvc -n cloned
NAME                               STATUS   VOLUME                                           CAPACITY   ACCESS MODES   STORAGECLASS     AGE
clone-cassandra-data-cassandra-0   Bound    clone-pvc-ca54f491-0058-47b6-9328-30b89bfab929   100Gi      RWO,RWX        csi-zadara-nas   29s
clone-cassandra-data-cassandra-1   Bound    clone-pvc-047ee3a9-bd30-4a4d-b22a-86b51c726d89   100Gi      RWO,RWX        csi-zadara-nas   29s
```

Verify that Pods are created
```shell script
$ kubectl get pods -n cloned
NAME                READY   STATUS              RESTARTS   AGE
clone-cassandra-0   1/1     Running                    0   4m22s
clone-cassandra-1   0/1     ContainerCreating          0   4m22s
```

## Cleanup

#### Clones

Get the name of Application Clone
```shell script
$ kubectl get applicationclones.zadara.com -n zadara
NAME                                                     AGE
clone-cassandra-test-snapshot-cfg-2020-04-02--13-18-02   50m
```
Delete Application clone including all cloned resources (Pods, PVCs, etc.)
```shell script
$ kubectl delete applicationclone -n zadara clone-cassandra-test-snapshot-cfg-2020-04-02--13-18-02
applicationclone.zadara.com "clone-cassandra-test-snapshot-cfg-2020-04-02--13-18-02" deleted
```
You can see that `cloned` namespace with all resources has been deleted
```shell script
$ kubectl get namespace cloned
Error from server (NotFound): namespaces "cloned" not found
```

Delete Clone Configuration and Invoker
```shell script
$ kubectl delete -n zadara -f examples/cassandra/invoker-clone-cfg.yaml
invoker.zadara.com "cassandra-clone" deleted
$ kubectl delete -n zadara -f examples/cassandra/clone-configuration.yaml
cloneconfiguration.zadara.com "cassandra-test-clone-cfg" deleted
```

#### Snapshots

Deactivate Snapshots Configration by deleting Invoker
```shell script
$ kubectl delete -n zadara -f examples/cassandra/invoker-snapshot-cfg.yaml
invoker.zadara.com "cassandra-snapshot" deleted
```

Delete Application Snapshots created by Snapshot Configuration `cassandra-test-snapshot-cfg`
(all snapshots have a corresponding label).
This may take a few moments to complete.
```shell script
$ kubectl delete applicationsnapshots.zadara.com -n zadara -l zadara.com/snapshotConfiguration=cassandra-test-snapshot-cfg
applicationsnapshot.zadara.com "cassandra-test-snapshot-cfg-2020-04-02--12-30-03" deleted
applicationsnapshot.zadara.com "cassandra-test-snapshot-cfg-2020-04-02--12-30-04" deleted
```

Delete Snapshot Configuration
```shell script
$ kubectl delete -n zadara -f examples/cassandra/snapshot-configuration.yaml
snapshotconfiguration.zadara.com "cassandra-test-snapshot-cfg" deleted
```

Delete Application Definition
```shell script
$ kubectl delete -f examples/cassandra/app-definition.yaml
appdefinition.zadara.com/cassandra-test-app-definition deleted
```

Delete Snapshot Policies
```shell script
$ kubectl delete -f examples/common/policy-every-ten-minutes.yaml
snapshotpolicy.zadara.com "every-ten-minutes" deleted
$ kubectl delete -f examples/common/policy-every-two-minutes.yaml
snapshotpolicy.zadara.com "every-two-minutes" deleted
```

#### Application

Delete Cassandra, including all resources.
Note, that deleting StatefulSet does not delete its' PVCs - we do this explicitly, using `app=cassandra` label.
```shell script
$ kubectl delete -f examples/cassandra/cassandra-stateful-set.yaml
statefulset.apps "cassandra" deleted
$ kubectl delete -f examples/cassandra/cassandra-service.yaml
service "cassandra" deleted
$ kubectl delete pvc -l app=cassandra
persistentvolumeclaim "cassandra-data-cassandra-0" deleted
persistentvolumeclaim "cassandra-data-cassandra-1" deleted
```
