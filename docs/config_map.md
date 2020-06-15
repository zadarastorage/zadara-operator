# Zadara Operator Configuration

### Example config file

```yaml
logLevel:
  app-definition: "info"
  app-clone: "info"
  app-snapshot: "info"
  cronjob: "info"
  csi: "info"
  general: "info"
  invoker: "info"
  policy: "info"
  snapshot-cfg: "info"
  vpsa: "info"
  zoperator: "info"
useLogColors: "false"
defaultStorageClass:
  reclaimPolicy: "Retain"
csi:
  repository: quay.io/zadara/csi-driver
  tag: 1.2.1
  pullPolicy: IfNotPresent
```

 Value | Description |
 ------|-------------|
`logLevel` | Verbosity levels for logs. Can be set for each log tag.
`logLevel.<tag>` | Allowed values: `panic`, `fatal`, `error`, `warn` or `warning`, `info`, `debug`.
`useLogColors` | Use colored output in logs. Does not auto-detect pipes, redirection, or other non-interactive outputs.
`defaultStorageClass` | Parameters for sample StorageClasses Operator creates for each VPSA.
`defaultStorageClass.reclaimPolicy` | See https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy
`csi` | Configure CSI Driver image
`csi.repository` | `zadara/csi-driver` repository contains Ubuntu-based image, `quay.io/zadara/csi-driver` contains an image based on RedHat UBI
`csi.tag` | CSI driver version
`csi.pullPolicy` | Image pull policy. Allowed values: `Always`, `Never`, `IfNotPresent`.

#### Log tags

Log tags appear in operator logs right after the timestamp: `general`, `vpsa`.
For all available log tags see [example above](#example-config-file).
```
Apr 23 07:46:19.725551 [general] [INFO]                                      main.main[ 112] Starting Zadara Operator.
Apr 23 07:49:24.368952 [vpsa]    [INFO]            common.(*ReconcileBase).ReconcileCR[  56] Reconcile *v1alpha1.Vpsa instance 'default/example-vpsa'
```

### Using config map

Local config files are easy to maintain, but for Kubernetes, we must make the config available on all cluster nodes.
To convert this config file into Kubernetes ConfigMap, use helper script (TODO-dev: this should be one of features of our CLI)
`hack/configmap_gen.sh`:

1. Edit [local config file](../zadara-operator-config.yaml) `zadara-operator-config.yaml`

2. Apply config file, as a ConfigMap to your cluster
   ```shell script
   hack/configmap_gen.sh apply
   ```
   Local config file can be specified explicitly:
   ```shell script
   hack/configmap_gen.sh apply -i /path/to/local-config.yaml
   ```
   For more options:
   ```shell script
   hack/configmap_gen.sh -h
   ```

3. It may take about 1 minute for ConfigMap to be updated in all pods.
