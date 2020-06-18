![Image](zadara.png)

# zOperator - Zadara Storage Kubernetes Operator

Zadara Storage Operator: automatic CSI driver management and Data Protection services

---
## Table of Contents

[Overview](#overview)

[Installation](#installation)

[Zadara Custom Resources](#zadara-custom-resources)

[Operator Configuration](#operator-configuration)

[Usage Examples](examples/README.md)

[CLI](#cli)

[CSI Driver](#csi-driver)

[Troubleshooting](docs/troubleshooting.md)

[Contact us](#contact-us)

[About](#about)

---
### Overview

Zadara company supply Fully-managed, enterprise storage-as-a-service (STaaS) solutions, available on premises or in the cloud.
As part of Zadara solutions, Zadara provides an Operator for Kubernetes called zOperator.

#### zOperator provides:

* Automatically discover installed applications on the cluster.
* Manage Zadara CSI drivers.
* Schedule crash consistent snapshots on an application.
* Clone application from a snapshot.
* CLI to easily control the operator and manage its resources.

---
### Installation

[Install using Helm Chart](docs/install_helm.md)

---
### Zadara Custom Resources

[Custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#custom-resources) are extensions of the Kubernetes API. This page discusses when to add a custom resource to your Kubernetes cluster and when to use a standalone service. It describes the two methods for adding custom resources and how to choose between them.

Here are [Zadara Custom Resources](docs/custom_resources.md).

---
### Operator Configuration

zOperator can be configured using [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap).

Here is [Zadara Config Map](docs/config_map.md).

---
### CLI

Zadara CLI helps manages Zadara Operator/CSI Kubernetes cluster.
[CLI Download](cli)
[Documentation](cli/docs/README.md)

---
### CSI Driver

Zadara supply a CSI driver.
More information at https://github.com/zadarastorage/zadara-csi

---
### Contact Us

Feel free to contact us at:
**k8s@zadara.com**

---
### About

More information at https://www.zadara.com/

---

