# Zadara Operator Configuration

### Example config file

```yaml
# Verbosity level for logs.
# Allowed values: fatal, error, warn or warning, info, debug
logLevel:
  app-definition: "info"
  app-clone: "info"
  app-snapshot: "info"
  # CronJobs owned by operator for ApplicationSnapshots management
  cronjob: "info"
  csi: "info"
  general: "info"
  invoker: "info"
  policy: "info"
  snapshot-cfg: "info"
  vpsa: "info"
  zoperator: "info"
# Use colored output in logs. Does not auto-detect pipes, redirection, or other non-interactive outputs.
useLogColors: "true"
# Parameters for default storage classes Operator creates for each VPSA
# See https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy
defaultStorageClass:
  reclaimPolicy: "Retain"
appDiscoverExcludedNamespaces:
  - "kube-system"
  - "openshift"
csi:
  repository: zadara/csi-driver
  tag: 1.2.6
  pullPolicy: IfNotPresent
```

 Value | Description |
 ------|-------------|
`logLevel` | Verbosity levels for logs. Can be set for each log tag.
`logLevel.<tag>` | Allowed values: `fatal`, `error`, `warn` or `warning`, `info`, `debug`.
`useLogColors` | Use colored output in logs. Does not auto-detect pipes, redirection, or other non-interactive outputs.
`defaultStorageClass` | Parameters for sample StorageClasses Operator creates for each VPSA.
`defaultStorageClass.reclaimPolicy` | See https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy
`appDiscoverExcludedNamespaces` | Disable application auto-discovery for namespaces, containing specified string in their name (e.g `openshift` will exclude all namespaces such as `openshift-markeplace`, `openshift-operators`, etc). Typically used for system namespaces.
`csi` | Configure CSI Driver image
`csi.repository` | Zadara CSI Driver is available on DockerHub `zadara/csi-driver` or in RedHat certified registry `registry.connect.redhat.com/zadara/csi`
`csi.tag` | CSI driver version
`csi.pullPolicy` | Image pull policy. Allowed values: `Always`, `Never`, `IfNotPresent`.

#### Log tags

Log tags appear in operator logs right after the timestamp: `general`, `vpsa`.
For all available log tags see [example above](#example-config-file).
```
Apr 23 07:46:19.725551 [general] [INFO]                                      main.main[ 112] Starting Zadara Operator.
Apr 23 07:49:24.368952 [vpsa]    [INFO]            common.(*ReconcileBase).ReconcileCR[  56] Reconcile *v1alpha1.Vpsa instance 'default/example-vpsa'
```

### Updating Operator config map

There are two ways to update ConfigMap values:
- edit ConfigMap `zoperator-config-map`, residing in the same namespace as Operator, using `kubectl`
- use `zadara` [CLI helper tool](https://github.com/zadarastorage/zadara-operator#cli).

Note that it may take about 1 minute for ConfigMap to be updated in all pods.

#### Using kubectl

Same instructions apply for `oc`.

Assuming zoperator runs in `zadara` namespace, run:
```shell script
$ kubectl edit configmap --namespace zadara zoperator-config-map
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  zadara-operator-config.yaml: |
    # Verbosity level for logs.
    # Allowed values: fatal, error, warn or warning, info, debug
    logLevel:
...
```

#### Using CLI helper

Show the current config:

```shell script
$ zadara config show
# Verbosity level for logs.
# Allowed values: panic, fatal, error, warn or warning, info, debug
logLevel:
...
```

Update config values (tip: tap [tab][tab] to auto-complete ConfigMap keys):
```shell script
$ zadara config set logLevel. # [tab][tab]
logLevel.app-clone       logLevel.app-snapshot    logLevel.csi             logLevel.invoker         logLevel.snapshot-cfg    logLevel.zoperator
logLevel.app-definition  logLevel.cronjob         logLevel.general         logLevel.policy          logLevel.vpsa
$ zadara config set logLevel.vpsa --value=debug
Config entry logLevel.vpsa was successfully updated to debug
```
