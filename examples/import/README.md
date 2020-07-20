
## Migrate an application to another cluster

In this example:

- Create Application Snapshot using `zadara` CLI
- Export Application Snapshot to file
- Import Application Snapshot into a new Kubernetes cluster and clone it using `zadara` CLI
- Explore additional features and quality-of-life helpers of `zadara` CLI

#### For the impatient

<!--- TODO: rewrite this more automated --->

Source cluster:
```shell script
zadara storageclass set-default example-vpsa-pool-00010003-nas
helm install --namespace mysql --create-namespace test-mysql stable/mysql
kubectl get pods --namespace mysql --watch
zadara snapshot create mysql-test-mysql test-snapshot
zadara snapshot export test-snapshot-2020-07-19--14-18-49 -f mysql-export.tar
```

Destination cluster:
```shell script
zadara snapshot import mysql-export.tar
zadara clone create test-snapshot test-clone --to-namespace mysql
```

---

### Prerequisites

Get `zadara` CLI tool.
We highly recommend enabling autocomplete for `zadara` commands and arguments.
Run `zadara completion --help` for the instructions:
```shell script
$ zadara completion --help
To load completions:

Bash:

$ source <(zadara completion bash)

# To load completions for each session, execute once:
Linux:
  $ zadara completion bash > /etc/bash_completion.d/zadara
...
```

Choose a default StorageClass for new Applications.
Default StorageClass is used when PVC does not specify `StorageClassName`.

Set `example-vpsa-pool-00010003-nas` as default, using `zadara` CLI with autocomplete:
```shell script
$ zadara storageclass set-default  # [tab][tab]
example-vpsa-pool-00010003-block  example-vpsa-pool-00010003-nas
$ zadara storageclass set-default example-vpsa-pool-00010003-nas
StorageClass example-vpsa-pool-00010003-nas was successfully set as default
```
___

Verify that everything is ready for deploying an application. The easiest way is to use `zadara status`, we should see:
 - Operator and CSI pods are ready
 - StorageClass `example-vpsa-pool-00010003-nas` is set as default
 - Vpsa `example-vpsa` in `CsiReady` state
```shell script
$ zadara status
OPERATOR POD                    NODE            READY
zoperator-7db7c5ddf7-vx76j      k8s-base-master true

CSI POD                                                 PROVISIONER     NODE            READY
example-vpsa-csi-zadara-controller-6bfc6ddbb4-m6khj     csi.zadara.com  k8s-base-master true
example-vpsa-csi-zadara-node-smj86                      csi.zadara.com  k8s-base-master true

SC                                         PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
example-vpsa-pool-00010003-block           csi.zadara.com   Retain          Immediate           true                   2d23h
example-vpsa-pool-00010003-nas (default)   csi.zadara.com   Retain          Immediate           true                   2d23h

VPSAS          STATUS     PROVISIONER      HOSTNAME      AUTOEXPAND   AGE
example-vpsa   CsiReady   csi.zadara.com   10.10.10.10   true         2d23h
```

### Deploy an application

Install the latest mysql Helm chart. This time we will install it to a new namespace `mysql`.

```shell script
$ helm install --namespace mysql --create-namespace test-mysql stable/mysql
NAME: test-mysql
LAST DEPLOYED: Sun Jul 19 16:46:52 2020
NAMESPACE: mysql
STATUS: deployed
REVISION: 1
NOTES:
...
```

When MySQL application has been deployed, Operator will automatically create an [AppDefinition](../../docs/custom_resources.md#AppDefinition) object,
which represents an application in your cluster (in other words, which K8s objects comprise the application).

Let's check on `mysql` deployment:
```shell script
$ zadara app info
NAME               AUTODISCOVER   APPNAMESPACE   STATUS    SELECTOR
mysql-test-mysql   true           mysql          Running   map[app:test-mysql]
```
That is, `mysql-test-mysql` (the default format is `<namespace>-<app name>`) app was auto-discovered by zOperator,
in namespace `mysql`, having labels `app: test-mysql`. AppDefinition is in `Running` state, meaning there is at least one running Pod.

We can see a list of K8s objects of `mysql-test-mysql`, using `zadara app` with `-o yaml` flag (similar to how it works in `kubectl get`).
See `status.apiObjects` field in example below. These are K8s objects which will be included in ApplicationSnapshot.

_Some fields omitted for readability._
```shell script
$ zadara app info -o yaml
apiVersion: v1
items:
- apiVersion: zadara.com/v1alpha1
  kind: AppDefinition
  metadata:
    creationTimestamp: "2020-07-19T13:46:52Z"
    finalizers:
    - zadara.com/in-use-protection
    labels:
      zadara.com/autoDiscover: "true"
    name: mysql-test-mysql
  spec:
    autoDiscover: true
    namespace: mysql
    selector:
      matchLabels:
        app: test-mysql
  status:
    apiObjects:
    - PersistentVolumeClaim mysql/test-mysql
    - Service mysql/test-mysql
    - Secret mysql/test-mysql
    - Pod mysql/test-mysql-6d7547b84c-hqclt
    - ConfigMap mysql/test-mysql-test
    - Endpoints mysql/test-mysql
    - ReplicaSet mysql/test-mysql-6d7547b84c
    - Deployment mysql/test-mysql
    state: Running
kind: List
metadata: {}
```

## Store data on Persistent Volume created for the application

TODO-dev: instructions to populate DB

## Create Application Snapshot

Application Snapshot can be created using a single `zadara` CLI command:
```
zadara snapshot create APP SNAPSHOT
```
Where `APP` is AppDefinition name, as appears in `zadara app info`, and `SNAPSHOT` is the name for the new snapshot.
In this example, we will create an ApplicationSnapshot name `test-snapshot`:
```shell script
$ zadara snapshot create mysql-test-mysql test-snapshot
Snapshot 'test-snapshot' for 'mysql-test-mysql' application was created successfully
```

Behind the scenes, `zadara` CLI and Operator will create
[SnapshotPolicy](../../docs/custom_resources.md#SnapshotPolicy) (with on-demand schedule),
[SnapshotConfiguration](../../docs/custom_resources.md#SnapshotConfiguration),
[Invoker](../../docs/custom_resources.md#Invoker) and
[ApplicationSnapshot](../../docs/custom_resources.md#ApplicationSnapshot).
These new objects can be seen in `zadara status` output:
```shell script
$ zadara status
OPERATOR POD                    NODE            READY
zoperator-7db7c5ddf7-vx76j      k8s-base-master true

CSI POD                                                 PROVISIONER     NODE            READY
example-vpsa-csi-zadara-controller-6bfc6ddbb4-m6khj     csi.zadara.com  k8s-base-master true
example-vpsa-csi-zadara-node-smj86                      csi.zadara.com  k8s-base-master true

NAMESPACE   PVC          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                     AGE   VOLUMEMODE
mysql       test-mysql   Bound    pvc-db5dd47f-714a-49c9-9dfb-2a95ebbb8f34   8Gi        RWO            example-vpsa-pool-00010003-nas   36m   Filesystem

SC                                         PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
example-vpsa-pool-00010003-block           csi.zadara.com   Retain          Immediate           true                   2d23h
example-vpsa-pool-00010003-nas (default)   csi.zadara.com   Retain          Immediate           true                   2d23h

VPSAS          STATUS     PROVISIONER      HOSTNAME    AUTOEXPAND   AGE
example-vpsa   CsiReady   csi.zadara.com   10.2.8.22   true         2d23h

APPDEFINITIONS     AUTODISCOVER   APPNAMESPACE   STATUS    SELECTOR
mysql-test-mysql   true           mysql          Running   map[app:test-mysql]

SNAPSHOTPOLICIES   STATUS   SCHEDULE    KEEP LAST
on-demand          Ready    on-demand   1

NAMESPACE   SNAPSHOTCONFIGURATIONS   STATUS   APP
zadara      test-snapshot            Active   mysql-test-mysql

NAMESPACE   INVOKERS        STATUS   SNAPSHOT CONFIGURATION   CLONE CONFIGURATION
zadara      test-snapshot   Ready    test-snapshot

NAMESPACE   APPLICATIONSNAPSHOTS                 STATUS    SNAPSHOT CONFIGURATION   AGE
zadara      test-snapshot-2020-07-19--14-18-49   Created   test-snapshot            4m15s
```

The name `test-snapshot` was given to SnapshotConfiguration and Invoker,
and the ApplicationSnapshot will have a name in `<snapshot name>-<date>--<time>` format: `test-snapshot-2020-07-19--14-18-49` in above example.

## Export Application Snapshot

Now, we will export ApplicationSnapshot with all its dependencies into a file, using command:
```
zadara snapshot export NAME [-f FILENAME]
```
Where `NAME` is the ApplicationSnapshot name, and `FILENAME` is an optional name for resulting file (a tar archive containing all required objects),
by default filename matches the ApplicationSnapshot name.

In this example we will export snapshot `test-snapshot-2020-07-19--14-18-49` to `mysql-export.tar` in current working directory.
```shell script
$ zadara snapshot export test-snapshot-2020-07-19--14-18-49 -f mysql-export.tar
Export ApplicationSnapshot zadara/test-snapshot-2020-07-19--14-18-49
Export SnapshotConfiguration zadara/test-snapshot
Export AppDefinition mysql-test-mysql
Export SnapshotPolicy on-demand
Export Vpsa example-vpsa
Application Snapshot was exported successfully
$ ls -lh mysql-export.tar
-rw-rw-r-- 1 user user 43K Jul 19 17:33 mysql-export.tar
```

## Import Application Snapshot

At this point ApplicationSnapshot can be imported into a new cluster.

Requirements for the destination cluster:
- zOperator is installed and running
- all Nodes have connectivity to the VPSA

*Vpsa custom resources and CSI drivers will be automatically created during the process of import.*

Check that zOperator is running (note that no other Zadara resources are present):
```shell script
$ zadara status
OPERATOR POD                    NODE            READY
zoperator-7db7c5ddf7-s26s4      k8s-base-master true
```

Copy exported archive `mysql-export.tar` to the current working directory in the new cluster, and import using command:

```shell script
$ zadara snapshot import mysql-export.tar
Import AppDefinition mysql-test-mysql
Import SnapshotPolicy on-demand
Import Vpsa example-vpsa
Import SnapshotConfiguration zadara/test-snapshot
Import ApplicationSnapshot zadara/test-snapshot-2020-07-19--14-18-49
Application Snapshot was imported successfully
```

Verify that snapshot has been imported. Make sure that the Vpsa is in `CsiReady` state.
Overall, `zadara status` output should look as if we have just created an ApplicationSnapshot *in the source cluster*.
Notable difference: AppDefinition is not auto-discovered anymore, and it has changed its state to `DoesNotExist`.
This state means no objects matching AppDefinition's namespace and selector are present in the destination cluster.
```shell script
$ zadara status
OPERATOR POD                    NODE            READY
zoperator-7db7c5ddf7-sl9g9      k8s-base-master true

CSI POD                                                 PROVISIONER     NODE            READY
example-vpsa-csi-zadara-controller-6bfc6ddbb4-pwzgr     csi.zadara.com  k8s-base-master true
example-vpsa-csi-zadara-node-tvcxl                      csi.zadara.com  k8s-base-master true

SC                                 PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
example-vpsa-pool-00010003-block   csi.zadara.com   Retain          Immediate           true                   23s
example-vpsa-pool-00010003-nas     csi.zadara.com   Retain          Immediate           true                   23s

VPSAS          STATUS     PROVISIONER      HOSTNAME      AUTOEXPAND   AGE
example-vpsa   CsiReady   csi.zadara.com   10.10.10.10   true         30s

APPDEFINITIONS     AUTODISCOVER   APPNAMESPACE   STATUS         SELECTOR
mysql-test-mysql                  mysql          DoesNotExist   map[app:test-mysql]

SNAPSHOTPOLICIES   STATUS   SCHEDULE    KEEP LAST
on-demand          Ready    on-demand   1

NAMESPACE   SNAPSHOTCONFIGURATIONS   STATUS   APP
zadara      test-snapshot            Active   mysql-test-mysql

NAMESPACE   APPLICATIONSNAPSHOTS                 STATUS    SNAPSHOT CONFIGURATION   AGE
zadara      test-snapshot-2020-07-19--14-18-49   Created   test-snapshot            29s
```

## Clone imported Application Snapshot

An imported snapshot can be cloned in exactly the same way as local ApplicationSnapshot.

Like before, clone can be created with a one-liner:
```shell script
zadara clone create SNAPSHOT CLONE --to-namespace NAMESPACE [--app-snapshot APP-SNAPSHOT]
```
Where `SNAPSHOT` is snapshot name as we used in `zadara snapshot create` (i.e SnapshotConfiguration name),
`CLONE` is a name for the new clone,
`NAMESPACE` is the namespace where cloned MySQL will run (will be created if not exists)
and `APP-SNAPSHOT` allows to choose a specific point-in-time (i.e ApplicationSnapshot) to clone.
In our case we only have a single ApplicationSnapshot, so `APP-SNAPSHOT` can be omitted.

Create a clone of `test-snapshot`, named `test-clone` in namespace `mysql`:
```shell script
$ zadara clone create test-snapshot test-clone --to-namespace mysql
Clone 'test-clone' was created successfully from snapshot 'test-snapshot'
```

Verify:
```shell script
$ zadara status
OPERATOR POD                    NODE            READY
zoperator-7db7c5ddf7-6wx9v      k8s-base-master true

CSI POD                                                 PROVISIONER     NODE            READY
example-vpsa-csi-zadara-controller-6bfc6ddbb4-nrjcn     csi.zadara.com  k8s-base-master true
example-vpsa-csi-zadara-node-zp2fg                      csi.zadara.com  k8s-base-master true

NAMESPACE   PVC          STATUS   VOLUME                                                           CAPACITY   ACCESS MODES   STORAGECLASS                     AGE   VOLUMEMODE
mysql       test-mysql   Bound    clone-pvc-2ab19cc6-fb25-46e1-aa95-b5b597e415ee52524f03882a73a3   8Gi        RWO            example-vpsa-pool-00010003-nas   13s   Filesystem

SC                                 PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
example-vpsa-pool-00010003-block   csi.zadara.com   Retain          Immediate           true                   93s
example-vpsa-pool-00010003-nas     csi.zadara.com   Retain          Immediate           true                   93s

VPSAS          STATUS     PROVISIONER      HOSTNAME      AUTOEXPAND   AGE
example-vpsa   CsiReady   csi.zadara.com   10.10.10.10   true         98s

APPDEFINITIONS     AUTODISCOVER   APPNAMESPACE   STATUS    SELECTOR
mysql-test-mysql                  mysql          Running   map[app:test-mysql]

SNAPSHOTPOLICIES   STATUS   SCHEDULE    KEEP LAST
on-demand          Ready    on-demand   1

NAMESPACE   SNAPSHOTCONFIGURATIONS   STATUS   APP
zadara      test-snapshot            Idle     mysql-test-mysql

NAMESPACE   CLONECONFIGURATIONS   STATE   SNAPSHOT                              CLONE         DRYRUN
zadara      test-clone            Ready   test-snapshot-2020-07-19--14-18-49    test-clone

NAMESPACE   INVOKERS      STATUS   SNAPSHOT CONFIGURATION   CLONE CONFIGURATION
zadara      test-clone    Ready                             test-clone

NAMESPACE   APPLICATIONCLONES   STATUS    CLONE CONFIGURATION   AGE
zadara      test-clone          Created   test-clone            14s

NAMESPACE   APPLICATIONSNAPSHOTS                  STATUS    SNAPSHOT CONFIGURATION   AGE
zadara      test-snapshot-2020-07-19--14-18-49    Created   test-snapshot            99
```
You can see that behind the scenes, `zadara` CLI and Operator have created
[CloneConfiguration](../../docs/custom_resources.md#CloneConfiguration),
[Invoker](../../docs/custom_resources.md#Invoker) and
[ApplicationClone](../../docs/custom_resources.md#ApplicationClone).

<!--- TODO: cleanup - both clusters, and in one of them we may need to force-delete --->
