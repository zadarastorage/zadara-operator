
<!--- This file is auto-generated. Do not edit. -->
<!--- Some formatting options (e.g. '|' escaping) are not supported by all Markdown viewers. -->
<!--- Best viewed on Github, or https://dillinger.io/ -->

# Zadara Operator Custom Resources
- [Vpsa](#Vpsa)
- [AppDefinition](#AppDefinition)
- [SnapshotPolicy](#SnapshotPolicy)
- [SnapshotConfiguration](#SnapshotConfiguration)
- [CloneConfiguration](#CloneConfiguration)
- [Invoker](#Invoker)
- [ApplicationClone](#ApplicationClone)
- [ApplicationSnapshot](#ApplicationSnapshot)
---

##  Vpsa

VPSA allows your Kubernetes cluster to connect to VPSA. For each Vpsa having `spec.csi` specified, Operator will automatically deploy a CSI Driver, and create sample Storage Classes (Block and NAS for each Storage Pool on that VPSA).
```shell script
kubectl get vpsas
```

#### Example YAML
```yaml
apiVersion: zadara.com/v1alpha1
kind: Vpsa
metadata:
  name: example-vpsa
spec:
  hostname: "example.vpsas.zadara.com"
  https: true
  token: "EXAMPLETOKEN-1234"
  # csi: if defined, run CSI driver and connect to VPSA
  csi:
    provisioner: "csi.zadara.com"
    iscsiMode: "rootfs"
    # Support VPSA Volume auto-expand feature
    autoExpandSupport: true
    # livenessProbe: k8s built-in, some fields omitted
    livenessProbe:
      periodSeconds: 5
      httpGet:
        port: 9808

```

#### Spec
Field | Description | Notes
------|-------------|------
`spec` | VpsaSpec defines the desired state of Vpsa |
`spec.hostname` | Hostname of the VPSA | Required
`spec.https` | Use https connection to the VPSA (default: http) | Required
`spec.token` | Token of the VPSA | Required
`spec.csi` | CSI provisioner |
`spec.csi.autoExpandSupport` | Support VPSA NAS Volume auto-expand feature. Enabled by default. To enable auto-expand for CSI Volumes, you need to configure Storage Class parameters.volumeOptions https://github.com/zadarastorage/zadara-csi/#storage-class. When autoExpandSupport is enabled, periodical sync will be handled by a CronJob, running in the same namespace as CSI driver. |
`spec.csi.iscsiMode` | A way for the CSI plugin to reach iscsiadm on host https://github.com/zadarastorage/zadara-csi#node-iscsi-connectivity | Required. Allowed values: `"rootfs"`, `"client-server"`
`spec.csi.provisioner` | CSI provisioner to be used in a StorageClass, for example `csi.zadara.com`. More: https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner |
`spec.csi.livenessProbe` | Liveness probe configuration for CSI driver. Default livenessProbe.handler is httpGet: {path:"/healthz", port:9808}. Note: livenessProbe.handler MUST be httpGet, with path "/healthz". Other options not allowed. livenessProbe.handler.port is configurable (default 9808) and MUST be unique for each CSI instance. More: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#configure-probes |
`spec.csi.livenessProbe.failureThreshold` | Minimum consecutive failures for the probe to be considered failed after having succeeded. Defaults to 3. Minimum value is 1. |
`spec.csi.livenessProbe.initialDelaySeconds` | Number of seconds after the container has started before liveness probes are initiated. More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes |
`spec.csi.livenessProbe.periodSeconds` | How often (in seconds) to perform the probe. Default to 10 seconds. Minimum value is 1. |
`spec.csi.livenessProbe.successThreshold` | Minimum consecutive successes for the probe to be considered successful after having failed. Defaults to 1. Must be 1 for liveness and startup. Minimum value is 1. |
`spec.csi.livenessProbe.timeoutSeconds` | Number of seconds after which the probe times out. Defaults to 1 second. Minimum value is 1. More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes |
`spec.csi.livenessProbe.exec` | One and only one of the following should be specified. Exec specifies the action to take. |
`spec.csi.livenessProbe.exec.command` | Command is the command line to execute inside the container, the working directory for the command  is root ('/') in the container's filesystem. The command is simply exec'd, it is not run inside a shell, so traditional shell instructions ('\|', etc) won't work. To use a shell, you need to explicitly call out to that shell. Exit status of 0 is treated as live/healthy and non-zero is unhealthy. |
`spec.csi.livenessProbe.httpGet` | HTTPGet specifies the http request to perform. |
`spec.csi.livenessProbe.httpGet.host` | Host name to connect to, defaults to the pod IP. You probably want to set "Host" in httpHeaders instead. |
`spec.csi.livenessProbe.httpGet.httpHeaders` | Custom headers to set in the request. HTTP allows repeated headers. |
`spec.csi.livenessProbe.httpGet.path` | Path to access on the HTTP server. |
`spec.csi.livenessProbe.httpGet.port` | Name or number of the port to access on the container. Number must be in the range 1 to 65535. Name must be an IANA_SVC_NAME. | Required
`spec.csi.livenessProbe.httpGet.scheme` | Scheme to use for connecting to the host. Defaults to HTTP. |
`spec.csi.livenessProbe.tcpSocket` | TCPSocket specifies an action involving a TCP port. TCP hooks not yet supported TODO: implement a realistic TCP lifecycle hook |
`spec.csi.livenessProbe.tcpSocket.host` | Optional: Host name to connect to, defaults to the pod IP. |
`spec.csi.livenessProbe.tcpSocket.port` | Number or name of the port to access on the container. Number must be in the range 1 to 65535. Name must be an IANA_SVC_NAME. | Required

#### Status
Field | Description | Notes
------|-------------|------
`status` | VpsaStatus defines the observed state of Vpsa |
`status.state` |  | Required. Allowed values: `"Pending"`, `"CsiUnknown"`, `"CsiCreating"`, `"CsiReady"`, `"CsiDeleting"`, `"CsiFailed"`
##  AppDefinition

AppDefinition allows to define which Kubernetes resources compose an Application, as a logical unit. This is done by means of namespaces and label-based selectors. The result of the query should include workloads (e.g Deployments, StatefulSets), RBAC, configuration (Secrets, ConfigMaps), storage, and all other resources required for the application to run.
```shell script
kubectl get appdefinitions
```

#### Example YAML
```yaml
apiVersion: zadara.com/v1alpha1
kind: AppDefinition
metadata:
  name: cassandra-test
spec:
  namespace: "default"
  selector:
    matchLabels:
      app: "cassandra"

```

#### Spec
Field | Description | Notes
------|-------------|------
`spec` | AppDefinitionSpec defines the desired state of AppDefinition |
`spec.autoDiscover` | Application Definition was automatically created by Zadara Operator |
`spec.namespace` | Namespace to include in ApplicationSnapshots | Required
`spec.selector` | Selector for API Resources (PVCs, Deployments, etc.) to include in as Application |
`spec.selector.matchExpressions` | matchExpressions is a list of label selector requirements. The requirements are ANDed. |
`spec.selector.matchLabels` | matchLabels is a map of {key,value} pairs. A single {key,value} in the matchLabels map is equivalent to an element of matchExpressions, whose key field is "key", the operator is "In", and the values array contains only "value". The requirements are ANDed. |

#### Status
Field | Description | Notes
------|-------------|------
`status` | AppDefinitionStatus defines the observed state of AppDefinition |
`status.apiObjects` | API objects (Deployments, StatefulSets, ConfigMaps, etc.) matching namespace and selector defined in the Spec. These objects will be included in ApplicationSnapshot. |
`status.state` | State of the application snapshot is one of the following: Pending - Application definition is waiting to Operator initialization. DoesNotExist - Application is not installed on cluster. NotRunning - Application exists on cluster but not running (no active pods running). Running - Application is running on cluster. | Required. Allowed values: `"Pending"`, `"DoesNotExist"`, `"NotRunning"`, `"Running"`
##  SnapshotPolicy

SnapshotPolicy defines frequency for ApplicationSnapshots to be taken, and retention policy for cleaning up old ApplicationSnapshots.
```shell script
kubectl get snapshotpolicies
```

#### Example YAML
```yaml
apiVersion: zadara.com/v1alpha1
kind: SnapshotPolicy
metadata:
  name: daily-snapshot-for-month
spec:
  # schedule: either cron expression or "on-demand"
  # various extended cron syntax options are supported:
  #  slash notation: */10, */5
  #  predefined schedules: @daily, @hourly, ...
  #  weekdays and months: SUN,MON,TUE,... JAN,FEB,MAR,...
  schedule: "0 0 * * *"
  keepLast: 7

```

#### Spec
Field | Description | Notes
------|-------------|------
`spec` | SnapshotPolicySpec defines the desired state of SnapshotPolicy |
`spec.keepLast` | Retention policy for snapshots. Oldest snapshots are deleted. | Required
`spec.schedule` | Either schedule in Cron format (see https://en.wikipedia.org/wiki/Cron) or "on-demand" | Required

#### Status
Field | Description | Notes
------|-------------|------
`status` | SnapshotPolicyStatus defines the observed state of SnapshotPolicy |
`status.state` | State of the Snapshot Policy can be one of the following: Pending - Snapshot Policy is waiting to Operator initialization. Ready - Snapshot Policy is valid and ready for use. | Required. Allowed values: `"Pending"`, `"Ready"`
##  SnapshotConfiguration

SnapshotConfiguration allows to select application resources which must be included in ApplicationSnapshot (using namespace and labels), and to attach Snapshot Policies to the application (including Persistent Volumes on VPSA). Additionally, it provides information about all Application Snapshot created for this SnapshotConfiguration.
```shell script
kubectl get snapshotconfigurations
```

#### Example YAML
```yaml
apiVersion: zadara.com/v1alpha1
kind: SnapshotConfiguration
metadata:
  name: dr-us-west-snapshot
spec:
  appDefinition: cassandra-test
  policies:
    - "daily-snapshot-for-month"
    - "weekly-snapshot-for-year"

```

#### Spec
Field | Description | Notes
------|-------------|------
`spec` |  |
`spec.appDefinition` | Application Definition for the snapshot | Required
`spec.policies` | Snapshot policies, defining when ApplicationSnapshots should be taken | Required

#### Status
Field | Description | Notes
------|-------------|------
`status` |  |
`status.policies` | All policies mentioned in Spec with lists of all the related ApplicationSnapshots |
`status.state` | State of the Snapshot Configuration can be one of the following: Pending - Snapshot configuration is waiting for required objects to be ready to use. Idle - No Application Snapshots using this configuration were created. Active - All Application Snapshots using this configuration were created successfully. Failed - Some Application Snapshots using this configuration are in failed state. | Required. Allowed values: `"Pending"`, `"Idle"`, `"Active"`, `"Failed"`
##  CloneConfiguration

CloneConfiguration allows to specify which ApplicationSnapshot must be restored. Additionally, it allows to set destination namespace and renaming pattern for restored resources.
```shell script
kubectl get cloneconfigurations
```

#### Example YAML
```yaml
apiVersion: zadara.com/v1alpha1
kind: CloneConfiguration
metadata:
  name: dr-us-west-clone
spec:
  appSnapshot:
    # references to metadata.name of ApplicationSnapshot
    name: "mysql-snap-2020-01-26"
  dryRun: false
  prefix: "clone-"
  targetNamespace: "test"

```

#### Spec
Field | Description | Notes
------|-------------|------
`spec` |  |
`spec.dryRun` | Run clone configuration checks only (default: false) |
`spec.prefix` | Prefix to add to cloned resources (default: "clone-") |
`spec.targetNamespace` | Target namespaces to clone to | Required
`spec.appClone` | Application Clone name (default: Prefix + AppSnapshot.Name + Suffix) |
`spec.appClone.name` | Name of the Application Clone | Required
`spec.appSnapshot` | Application Snapshot to clone | Required
`spec.appSnapshot.name` | Name of the Application Snapshot | Required

#### Status
Field | Description | Notes
------|-------------|------
`status` |  |
`status.state` | State of the Clone Configuration can be one of the following: Pending - Clone configuration is waiting for required objects to be ready to use. Ready - Clone configuration is valid and ready for use. | Required. Allowed values: `"Pending"`, `"Ready"`
##  Invoker

Invoker activates SnapshotConfiguration or CloneConfiguration. When SnapshotConfiguration is invoked, Operator will create a CronJob (for recurring Snapshot Policies), or a Job (for on-demand policy), which will create a new ApplicationSnapshot. When CloneConfiguration is invoked, Operator will create a new ApplicationClone object, eventually causing ApplicationSnapshot to be cloned. Deleting Invoker is an equivalent to stopping a SnapshotPolicy, or a cloning process (Application Snapshots or Clones will not be deleted).
```shell script
kubectl get invokers
```

#### Example YAML
```yaml
apiVersion: zadara.com/v1alpha1
kind: Invoker
metadata:
  name: mysql-daily-backup
spec:
  # one-of: snapshotConfiguration | cloneConfiguration
  snapshotConfiguration:
    name: "dr-us-west-snapshot"

```

#### Spec
Field | Description | Notes
------|-------------|------
`spec` | InvokerSpec defines the desired state of Invoker Exactly one of snapshotConfiguration\|cloneConfiguration must be specified |
`spec.cloneConfiguration` | Reference to a CloneConfiguration to Invoke |
`spec.cloneConfiguration.name` | Reference to Clone configuration name | Required
`spec.snapshotConfiguration` | Reference to a SnapshotConfiguration to Invoke |
`spec.snapshotConfiguration.name` | Reference to Snapshot configuration name | Required

#### Status
Field | Description | Notes
------|-------------|------
`status` | InvokerStatus defines the observed state of Invoker |
`status.state` | State of the Invoker can be one of the following: Pending - Invoker is waiting for required objects to be ready to use. Ready - Invoker is valid and ready for use. | Required. Allowed values: `"Pending"`, `"Ready"`
##  ApplicationClone

ApplicationClone is an internal resource, created by Operator. It includes a list of all restored Kubernetes resources and VPSA Volumes, and provides information about restore status for each item. When ApplicationClone is created, Operator starts cloning VPSA Volumes, and restoring Kubernetes resources, from the ApplicationSnapshot.
```shell script
kubectl get applicationclones
```

#### Example YAML
```yaml
apiVersion: zadara.com/v1alpha1
kind: ApplicationClone
metadata:
  name: cassandra-2009-02-09--19-52-14-clone
spec:
  # reference to metadata.name of CloneConfiguration
  cloneConfiguration: "dr-us-west-clone"

```

#### Spec
Field | Description | Notes
------|-------------|------
`spec` |  |
`spec.cloneConfiguration` | Reference For Clone Configuration | Required

#### Status
Field | Description | Notes
------|-------------|------
`status` |  |
`status.pvcs` | PVCs of the Application |
`status.pvs` | PVs of the Application |
`status.resourceKinds` | Resource Kinds (Group, Version, Resource) of the Application (including the APIObjects) |
`status.state` | State of the ApplicationClone | Required. Allowed values: `"Pending"`, `"Initialized"`, `"Creating"`, `"Created"`, `"Failed"`, `"Deleting"`
##  ApplicationSnapshot

ApplicationSnapshot is an internal resource, created by Operator. It includes all Application metadata (such as Deployment, StatefulSet, ConfigMap, PersistentVolumeClaim, and other Kubernetes resources) and Snapshots of Data, stored on VPSA. When ApplicationSnapshot is created, Operator creates snapshots of VPSA Volumes, and stores serialized Kubernetes resources in the ApplicationSnapshot. ApplicationSnapshot can appear only in one CloneConfiguration at the same time (User can't clone the same ApplicationSnapshot twice).
```shell script
kubectl get applicationsnapshots
```

#### Example YAML
```yaml
apiVersion: zadara.com/v1alpha1
kind: ApplicationSnapshot
metadata:
  name: cassandra-2009-02-09--19-52-14
spec:
  vpsas:
    - name: "example-vpsa"
      provisioner: "csi.zadara.com"
      snapshots:
        - cg: "vol-0000001"
  # JSON (or YAML) of all app's API objects
  apiObjects:
    - "{\"apiVersion\": \"v1\",\"items\": [{\"apiVersion\": \"v1\",\"kind\": \"ConfigMap\",\"metadata\": {..."
    - "{\"apiVersion\": \"v1\",\"items\": [{\"apiVersion\": \"apps\v1\",\"kind\": \"Deployment\",\"metadata\": {..."
  # SnapshotConfiguration parameters (not necessary, helps for readability and debug)
  snapshotConfiguration: "dr-us-west-snapshot"

```

#### Spec
Field | Description | Notes
------|-------------|------
`spec` | ApplicationSnapshotSpec defines the desired state of ApplicationSnapshot |
`spec.apiObjects` | Included API objects by the application snapshot |
`spec.snapshotConfiguration` | Reference to a SnapshotConfiguration to snapshot | Required
`spec.vpsas` | VPSAs that provides the volumes of the Application |

#### Status
Field | Description | Notes
------|-------------|------
`status` | ApplicationSnapshotStatus defines the observed state of ApplicationSnapshot |
`status.state` | State of the application snapshot is one of the following: Pending - Snapshot is waiting for required objects to be ready to use. Initialized - Snapshot was initialized. SuspendingIO - Suspending IO of VPSAs volumes to be crash-consistent. Creating - Snapshot is being created by the VPSAs. Created - Snapshot is created and ready to be cloned. Failed - Snapshot creation failed, can't cloned from this snapshot. Deleting - Snapshot is being deleted in the VPSAs. | Required. Allowed values: `"Pending"`, `"Initialized"`, `"SuspendingIO"`, `"Creating"`, `"Failed"`, `"Created"`, `"Deleting"`
