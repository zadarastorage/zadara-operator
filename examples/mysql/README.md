
## Using on-demand Application Snapshots for MySQL

In this example:

- Configure default StorageClass
- install MySQL Deployment using Helm chart
- create on-demand Application Snapshots including Persistent Volumes data and application configuration
- clone created Application Snapshot into a new Namespace

#### For the impatient

```shell script
# Deploy application
STORAGE_CLASS_NAME=$(kubectl get sc -l provisioner=csi.zadara.com | grep nas | awk '{print $1}')
zadara storageclass set-default ${STORAGE_CLASS_NAME}
helm install test-mysql stable/mysql # 'helm install --name test-mysql stable/mysql' for Helm 2
# Configure and create Snapshots
kubectl create -f examples/common/policy-on-demand.yaml
# Wait for automatic application discover
kubectl get appdefinitions.zadara.com default-test-mysql --watch
kubectl create -n zadara -f examples/mysql/snapshot-configuration-od.yaml
kubectl create -n zadara -f examples/mysql/invoker-snapshot-cfg-od.yaml
kubectl get applicationsnapshots.zadara.com --watch -n zadara # wait for application snapshot to be created
# Configure and create Clone of a Snapshot
SNAPSHOT=$(kubectl get applicationsnapshots.zadara.com -n zadara | grep Created | tail -n1 | awk '{print $1}')
sed -i s/SNAPSHOT_NAME_HERE/$SNAPSHOT/ examples/mysql/clone-configuration.yaml
kubectl create -n zadara -f examples/mysql/clone-configuration.yaml
kubectl create -n zadara -f examples/mysql/invoker-clone-cfg.yaml
```

```shell script
# Cleanup Clone and configuration
kubectl delete applicationclones -n zadara -l zadara.com/cloneConfiguration=mysql-test-clone-cfg
kubectl delete -n zadara -f examples/mysql/invoker-clone-cfg.yaml
kubectl delete -n zadara -f examples/mysql/clone-configuration.yaml
# Cleanup Snapshots and configuration
kubectl delete -n zadara -f examples/mysql/invoker-snapshot-cfg-od.yaml
kubectl delete applicationsnapshots -n zadara -l zadara.com/snapshotConfiguration=mysql-on-demand-snapshot-cfg
kubectl delete -n zadara -f examples/mysql/snapshot-configuration-od.yaml
kubectl delete -f examples/common/policy-on-demand.yaml
# Delete application
helm uninstall test-mysql # 'helm delete --purge test-mysql' for Helm 2
```

---

### Prerequisites

Choose a default StorageClass for new Applications.
Default StorageClass is used when PVC does not specify `StorageClassName`.

List available Storage Classes
```shell script
$ kubectl get storageclass
NAME                               PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
example-vpsa-pool-00010003-block   csi.zadara.com   Delete          Immediate           true                   3h7m
example-vpsa-pool-00010003-nas     csi.zadara.com   Delete          Immediate           true                   3h7m
```

We will set `example-vpsa-pool-00010003-nas` as default.
You can use either `kubectl` command:
```shell script
kubectl patch sc example-vpsa-pool-00010003-nas -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
or use `zadara` CLI tool:
```shell script
$ zadara storageclass set-default example-vpsa-pool-00010003-nas
StorageClass example-vpsa-pool-00010003-nas was successfully set as default
```

Verify that `example-vpsa-pool-00010003-nas` is now default
```shell script
$ kubectl get storageclass
NAME                                       PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
example-vpsa-pool-00010003-block           csi.zadara.com   Delete          Immediate           true                   3h9m
example-vpsa-pool-00010003-nas (default)   csi.zadara.com   Delete          Immediate           true                   3h9m
```

### Deploy an application

We will install the latest mysql Helm chart.
If `helm install` fails, try to `helm repo update`.
```shell script
$ helm install test-mysql stable/mysql
NAME: test-mysql
LAST DEPLOYED: Thu May 14 11:46:32 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
...
```
For Helm 2:
```shell script
$ helm install --name test-mysql stable/mysql
```

Note, that mysql created a PersistentVolumeClaim `test-mysql`.

Let's verify that this PVC indeed belongs to the StorageClass we previously set:
```shell script
$ kubectl get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                     AGE
test-mysql   Bound    pvc-305d8431-049c-49fa-9b5a-1d59e242ef3e   8Gi        RWO            example-vpsa-pool-00010003-nas   63s
```

Wait for MySQL pods to start running
```shell script
$ kubectl get pods --watch
NAME                          READY   STATUS    RESTARTS   AGE
test-mysql-69b7bf88bc-4zhqx   0/1     Running   0          35s
test-mysql-69b7bf88bc-4zhqx   1/1     Running   0          43s
```

## Store data on Persistent Volume created for the application

TODO-dev: instructions to populate DB

## Create Application Snapshot

Create on-demand SnapshotPolicy
```shell script
$ kubectl create -f examples/common/policy-on-demand.yaml
snapshotpolicy.zadara.com/on-demand created
```

Note that the operator also auto-discover the MySql application and creates AppDefinition for it.
Wait for automatic discover to create AppDefinition
```shell script
$ kubectl get appdefinitions.zadara.com default-test-mysql -w
NAME                 AUTODISCOVER   APPNAMESPACE   STATUS    SELECTOR
default-test-mysql   true           default        Running   map[app:test-mysql]
```

Note that you can check which API objects are in the AppDefinition and check if the application is running.
```shell script
$ kubectl describe appdefinitions.zadara.com default-test-mysql
Name:         default-test-mysql
Namespace:
Labels:       zadara.com/autoDiscover=true
Annotations:  <none>
API Version:  zadara.com/v1alpha1
Kind:         AppDefinition
Metadata:
  Creation Timestamp:  2020-05-25T11:58:59Z
  Finalizers:
    zadara.com/in-use-protection
  Generation:        1
  Resource Version:  16466440
  Self Link:         /apis/zadara.com/v1alpha1/namespaces/default/appdefinitions/default-test-mysql
  UID:               649be14b-a6b2-4a3b-a6e9-ed117731e99f
Spec:
  Auto Discover:  true
  Namespace:      default
  Selector:
    Match Labels:
      App:  test-mysql
Status:
  API Objects:
    ConfigMap default/test-mysql-test
    Endpoints default/test-mysql
    PersistentVolumeClaim default/test-mysql
    Pod default/test-mysql-69b7bf88bc-6dpst
    Secret default/test-mysql
    Service default/test-mysql
    Deployment default/test-mysql
    ReplicaSet default/test-mysql-69b7bf88bc
  State:  Running
Events:   <none>
```

Create a SnapshotConfiguration for on-demand policy and MySQL.
```shell script
$ kubectl create -n zadara -f examples/mysql/snapshot-configuration-od.yaml
snapshotconfiguration.zadara.com/cassandra-test-app-definition created
```

Activate SnapshotConfiguration by creating Invoker
```shell script
$ kubectl create -n zadara -f examples/mysql/invoker-snapshot-cfg-od.yaml
invoker.zadara.com/mysql-on-demand-snapshot created
```

Operator will start creating ApplicationSnapshot immediately:
```shell script
$ kubectl get applicationsnapshots --watch -n zadara
  NAME                                                STATUS         SNAPSHOT CONFIGURATION         AGE
  mysql-on-demand-snapshot-cfg-2020-04-16--09-25-19                  mysql-on-demand-snapshot-cfg   0s
  mysql-on-demand-snapshot-cfg-2020-04-16--09-25-19                  mysql-on-demand-snapshot-cfg   0s
  mysql-on-demand-snapshot-cfg-2020-04-16--09-25-19   Pending        mysql-on-demand-snapshot-cfg   0s
  mysql-on-demand-snapshot-cfg-2020-04-16--09-25-19   Pending        mysql-on-demand-snapshot-cfg   2s
  mysql-on-demand-snapshot-cfg-2020-04-16--09-25-19   SuspendingIO   mysql-on-demand-snapshot-cfg   2s
  mysql-on-demand-snapshot-cfg-2020-04-16--09-25-19   Creating       mysql-on-demand-snapshot-cfg   2s
  mysql-on-demand-snapshot-cfg-2020-04-16--09-25-19   Creating       mysql-on-demand-snapshot-cfg   3s
  mysql-on-demand-snapshot-cfg-2020-04-16--09-25-19   Created        mysql-on-demand-snapshot-cfg   3s
```

We can create another ApplicationSnapshot by deleting Invoker and creating it again:
```shell script
$ kubectl delete -n zadara -f examples/mysql/invoker-snapshot-cfg-od.yaml
invoker.zadara.com "mysql-on-demand-snapshot" deleted
$ kubectl create -n zadara -f examples/mysql/invoker-snapshot-cfg-od.yaml
invoker.zadara.com/mysql-on-demand-snapshot created
```

We can see that a new ApplicationSnapshot has been created for the same SnapshotConfiguration:
```shell script
$ kubectl get applicationsnapshots --watch -n zadara
NAME                                                STATUS    SNAPSHOT CONFIGURATION         AGE
mysql-on-demand-snapshot-cfg-2020-04-16--09-25-19   Created   mysql-on-demand-snapshot-cfg   88s
mysql-on-demand-snapshot-cfg-2020-04-16--09-26-49   Created   mysql-on-demand-snapshot-cfg   5s
```

Verify that all API objects (Pods, PVCs, etc) are included.
The list should contain all `RESOURCES` we seen after installing Helm Chart (run `helm status test-mysql` to show it again).
*API Objects not fully shown in this example, for readability*
```
$ kubectl describe applicationsnapshots mysql-on-demand-snapshot-cfg-2020-04-16--09-26-49 | cut -c1-120
Name:         mysql-on-demand-snapshot-cfg-2020-04-16--09-26-49
Namespace:    default
Labels:       zadara.com/snapshotConfiguration=mysql-on-demand-snapshot-cfg
              zadara.com/snapshotPolicy=on-demand
Annotations:  <none>
API Version:  zadara.com/v1alpha1
Kind:         ApplicationSnapshot
Metadata:
  Creation Timestamp:  2020-04-16T09:26:54Z
  Finalizers:
    zadara.com/appsnapshot-cleanup
  Generation:        3
  Resource Version:  8056629
  Self Link:         /apis/zadara.com/v1alpha1/namespaces/default/applicationsnapshots/mysql-on-demand-snapshot-cfg-2020
  UID:               72b9090a-991a-4126-9a19-20a079668dfb
Spec:
  API Objects:
    {"apiVersion":"v1","kind":"ConfigMap",...}
    {"apiVersion":"v1","kind":"Endpoints",...}
    {"apiVersion":"v1","kind":"Pod",...}
    {"apiVersion":"v1","kind":"Secret,...}
    {"apiVersion":"v1","kind":"Service",...}
    {"apiVersion":"apps/v1","kind":"Deployment",...}
    {"apiVersion":"apps/v1","kind":"ReplicaSet",...}
    {"apiVersion":"v1","kind":"PersistentVolume",...}
    {"apiVersion":"v1","kind":"PersistentVolumeClaim",...}

  Snapshot Configuration:  mysql-on-demand-snapshot-cfg
  Vpsas:
    Name:         example-vpsa
    Provisioner:  csi.zadara.com
    Snapshots:
      Cg:  cg-00000044
      Id:  snap-0000011b
Status:
  State:  Created
Events:   <none>
```

## Clone Application Snapshot

Choose Application Snapshot you want to clone
```shell script
$ kubectl get applicationsnapshots -n zadara
NAME                                                STATUS    SNAPSHOT CONFIGURATION         AGE
mysql-on-demand-snapshot-cfg-2020-04-16--09-25-19   Created   mysql-on-demand-snapshot-cfg   20m
mysql-on-demand-snapshot-cfg-2020-04-16--09-26-49   Created   mysql-on-demand-snapshot-cfg   19m
```

Edit `examples/mysql/clone-configuration.yaml`, and set `appSnapshot.name`.
For example:
```shell script
$ SNAPSHOT=mysql-on-demand-snapshot-cfg-2020-04-16--09-26-49
$ sed -i s/SNAPSHOT_NAME_HERE/$SNAPSHOT/ examples/mysql/clone-configuration.yaml
```

Create CloneConfiguration for ApplicationSnapshot
```shell script
$ kubectl create -n zadara -f examples/mysql/clone-configuration.yaml
cloneconfiguration.zadara.com/mysql-test-clone-cfg created
```

Activate CloneConfiguration and start clone process by creating Invoker
```shell script
$ kubectl create -n zadara -f examples/mysql/invoker-clone-cfg.yaml
invoker.zadara.com/mysql-clone created
```

In `examples/mysql/clone-configuration.yaml` we defined the following:
```yaml
  targetNamespace: "cloned"
```
That is, resources from namespace `default` will be cloned into namespace `cloned`.

Verify that cloned resources are successfully created (note the namespace parameter: `-n cloned`)
```shell script
$ kubectl get all -n cloned
NAME                              READY   STATUS    RESTARTS   AGE
pod/test-mysql-5b85f868bb-jj64x   1/1     Running   0          4m56s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/test-mysql   ClusterIP   10.100.195.162   <none>        3306/TCP   4m56s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/test-mysql   1/1     1            1           4m56s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/test-mysql-5b85f868bb   1         1         1       4m56s
```

## Cleanup

#### Clones

Delete Application clone including all cloned resources (Pods, PVCs, etc.)
```shell script
$ kubectl delete applicationclones -n zadara -l zadara.com/cloneConfiguration=mysql-test-clone-cfg
applicationclone.zadara.com "clone-mysql-on-demand-snapshot-cfg-2020-04-16--09-26-49" deleted
```

You can see that `cloned` namespace with all resources has been deleted
```shell script
$ kubectl get namespace cloned
Error from server (NotFound): namespaces "cloned" not found
```

Delete Clone Configuration and Invoker
```shell script
$ kubectl delete -n zadara -f examples/mysql/invoker-clone-cfg.yaml
invoker.zadara.com "mysql-clone" deleted
$ kubectl delete -n zadara -f examples/mysql/clone-configuration.yaml
cloneconfiguration.zadara.com "mysql-test-clone-cfg" deleted
```

#### Snapshots

Delete the  Invoker of Snapshots Configuration
```shell script
$ kubectl delete -n zadara -f examples/mysql/invoker-snapshot-cfg-od.yaml
invoker.zadara.com "mysql-on-demand-snapshot" deleted
```

Delete Application Snapshots created by Snapshot Configuration `mysql-test-snapshot-cfg`
(all snapshots have a corresponding label).
This may take a few moments to complete.
```shell script
$ kubectl delete applicationsnapshots -n zadara -l zadara.com/snapshotConfiguration=mysql-on-demand-snapshot-cfg
applicationsnapshot.zadara.com "mysql-on-demand-snapshot-cfg-2020-04-16--09-25-19" deleted
applicationsnapshot.zadara.com "mysql-on-demand-snapshot-cfg-2020-04-16--09-26-49" deleted
```

Delete Snapshot Configuration
```shell script
$ kubectl delete -n zadara -f examples/mysql/snapshot-configuration-od.yaml
snapshotconfiguration.zadara.com "mysql-on-demand-snapshot-cfg" deleted
```

Delete Snapshot Policy
```shell script
$ kubectl delete -f examples/common/policy-on-demand.yaml
snapshotpolicy.zadara.com "on-demand" deleted
```

#### Application

Delete mysql, including all resources.

Helm 3:
```shell script
$ helm uninstall test-mysql
```

Helm 2:
```shell script
$ helm delete --purge test-mysql
```
