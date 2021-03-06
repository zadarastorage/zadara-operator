## zadara logs csi

Print logs of Zadara CSI

### Synopsis

Helper command to get logs of the CSI.
You may change the verbose level of the logs using config command. PROVISIONER is the name of CSIDriver (kubectl get csidrivers)

```
zadara logs csi [PROVISIONER] {--node NODE_NAME | --controller} [flags]
```

### Options

```
      --controller    Controller pod of the running CSI
  -h, --help          help for csi
      --node string   Node pod of the running CSI
```

### Options inherited from parent commands

```
      --config string   config file (default is $HOME/.cli.yaml)
      --follow          Output appended data as the logs grows
      --lines int       Output the last NUM lines (default 50)
  -p, --previous        Output previous instance log
      --retry           Retry fetch logs on failure
```

### SEE ALSO

* [zadara logs](zadara_logs.md)	 - Print logs of Zadara Operator/CSI

###### Auto generated by spf13/cobra on 26-Aug-2020
