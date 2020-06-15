
# Helm chart notes for developers

Few important notes:
- Operator Helm Chart is compatible with Helm 3 only. Helm 2 has poor support for CRDs.
- CRDs are not uninstalled automatically, when the chart is uninstalled (which makes sense for the customers: they may re-install the operator without losing Custom Resources)
- Helm 3 [does not show installed objects](https://github.com/helm/helm/issues/5952) in `helm status`.
There's a nice plugin https://github.com/marckhouzam/helm-fullstatus (shows all but CRDs)

Complete guide to chart directory structure and `Chart.yaml` available [here](https://helm.sh/docs/topics/charts/)

Below are the most important notes from the official guide.

## Structure

```
zadara-operator/
  Chart.yaml          # A YAML file containing information about the chart
  LICENSE             # OPTIONAL: A plain text file containing the license for the chart
  README.md           # OPTIONAL: A human-readable README file
  values.yaml         # The default configuration values for this chart
  values.schema.json  # OPTIONAL: A JSON Schema for imposing a structure on the values.yaml file
  charts/             # A directory containing any charts upon which this chart depends.
  crds/               # Custom Resource Definitions
  templates/          # A directory of templates that, when combined with values,
                      # will generate valid Kubernetes manifest files.
  templates/tests     # OPTIONAL: K8s Job to verify installation https://helm.sh/docs/topics/chart_tests/
  templates/NOTES.txt # OPTIONAL: A plain text file (templated) containing short usage notes
```

### README.md

A README for a chart should be formatted in Markdown (README.md), and should generally contain:

- A description of the application or service the chart provides
- Any prerequisites or requirements to run the chart
- Descriptions of options in values.yaml and default values
- Any other information that may be relevant to the installation or configuration of the chart

### NOTES.txt

The chart can also contain a short plain text `templates/NOTES.txt` file that will be printed out after installation,
and when viewing the status of a release.

This file is *evaluated as a template*, and can be used to display usage notes, next steps,
or any other information relevant to a release of the chart.

For example, instructions could be provided for connecting to a database, or accessing a web UI.
Since this file is printed to STDOUT when running helm install or helm status,
it is recommended to keep the content brief and point to the README for greater detail.

### Predefined values in templates

The following values are pre-defined, are available to every template, and cannot be overridden. As with all values, the names are case sensitive.

- `Release.Name`: The name of the release (not the chart)
- `Release.Namespace`: The namespace the chart was released to.
- `Release.Service`: The service that conducted the release.
- `Release.IsUpgrade`: This is set to true if the current operation is an upgrade or rollback.
- `Release.IsInstall`: This is set to true if the current operation is an install.
- `Chart`: The contents of the `Chart.yaml`. Thus, the chart version is obtainable as `Chart.Version` and the maintainers are in `Chart.Maintainers`.
- `Files`: A map-like object containing all non-special files in the chart.
This will not give you access to templates, but will give you access to additional files that are present
(unless they are excluded using `.helmignore`). Files can be accessed using `{{ index .Files "file.name" }} `
or using the `{{ .Files.Get name }}` function. You can also access the contents of the file as `[]byte` using `{{ .Files.GetBytes }}`
- `Capabilities`: A map-like object that contains information about the versions of Kubernetes
(`{{ .Capabilities.KubeVersion }}` and the supported Kubernetes API versions (`{{ .Capabilities.APIVersions.Has "batch/v1" }}`)
