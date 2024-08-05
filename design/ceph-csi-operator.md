---
title: Integrate Ceph-CSI operator with Rook
target-version: release-1.15
---

# Integrate Ceph-CSI operator with Rook

## Overview

To integrate the new Ceph-CSI operator with Rook and remove the existing CSI deployment related code, RBAC's and dead code in later releases. Also before moving forward please read the ceph-CSI-operator design doc [here](https://github.com/ceph/ceph-csi-operator/blob/main/docs/design/operator.md).

**Note**: This feature is experimental in v1.15 and will not come with multus support in v1.15 so, users who wish to use multus or already have multus based cluster should avoid using this feature.

## Rook Deploying Ceph-CSI-Operator

There are two scenarios rook needs to manage while deploying of the new CSI-operator.

1. In v1.15, where the CSI-operator is experimental, we'll support:
    1. Manifest-based deployment
    2. Manual steps are ok for enabling the csi operator in upgraded clusters
2. In Future release, we'll add support for:
    1. Deployment in new cluster i.e [Fresh deployment](#fresh-deployment)
        1. Manifest Deployment
        2. Helm Deployment
    2. Deployment in upgrade cluster i.e [Upgrade clusters](#upgraded-clusters) cluster.
        1. Manifest Upgrade
        2. Helm Upgrade

We need to have some bash script which will be called by some make command to automatically update the RBAC, CRDs files in rook repo from the ceph-CSI-operator repo.

**Note**: If user enables this operator and also enables multus networking in that case we'll ignore deploying ceph-CSI operator.

## Fresh deployment

Rook will define the new CSI-operator defaults settings in [CRDs](#new-ceph-csi-operator-crds) in operator.yaml or operator chart `rook-ceph` `value.yaml`, and can be customizable by users. Rook will define the CSI operator and related CRs such as it is deployed by default and automatically with Rook.

- **Manifests**:
    - operator.yaml: Include the CSI operator deployment, and default CephCSI CRs
    - crds.yaml: Include the CephCSI CRDs
    - common.yaml: Include the RBAC needed for the CSI operator
    - Consider whether to have separate manifests, particularly in the first release if the CSI operator is experimental
- **Helm chart(will be supported in future releases)**: Add new settings in the operator and cluster chart and all others CRDs, RBAC will be added similarly to the manifest

In fresh clusters, we will introduce a new setting in `operator.yaml` configmap `rook-ceph-operator-config` name `ROOK_USE_CSI_OPERATOR: true` and if `USE_CSI_OPERATOR:` is set to `true` it will also disable the CSI `ROOK_CSI_DISABLE_DRIVER`. Setting `ROOK_CSI_DISABLE_DRIVER: true` will skip the current CSI driver reconciliation and the settings configured in `operator.yaml` for the new CRs will be taken into consideration.

From here, user/admin creates/modify the default manifest itself present in rook repository.

## Upgraded clusters

For upgrade clusters, we'll fetch the CSI settings and cluster connections details from configmaps like `rook-ceph-operator-config` and `rook-ceph-csi-config`. After fetching the CSI settings and cluster connections
details we can fill the new Ceph-CSI operator cr `CephCSICephConfig` with connection details and other CR's. Once we have the settings we can set `USE_CSI_OPERATOR:` is to `true` it will also disable the CSI `ROOK_CSI_DISABLE_DRIVER`.

**Note:** Not much changes are anticipated for external mode clusters.

## New Ceph-CSI operator CRDs

### Ceph-CSI-Operator configuration

The CSI operator brings new CRDS and API for users to configure settings. These CRDs are:

### CephCSICephCluster CRD

[This CRD](https://github.com/ceph/ceph-csi-operator/blob/main/docs/design/operator.md#cephCSIcephcluster-crd) stores connection and configuration details for a single Ceph cluster and provide the information to be used by multiple CSI drivers.

Any setting in this CR either needs to be generated by Rook, or wrapped by CephCluster CR under `spec.csi` or `rook-ceph-cluster` helm chart `values.yaml`.

### CephCSIConfig CRD

[This CRD](https://github.com/ceph/ceph-CSI-operator/blob/main/docs/design/operator.md#cephCSIconfig-crd) contains details about CephFS, RBD, and NFS configuration to be used when communicating with Ceph. Include a reference to a CephCSICephCluster holding the connection information for the target Ceph cluster.

The Rook operator would generate an instance of these CR for each of CephFilesystemSubVolumeGroup and CephBlockPoolRadosNamespace CRs.

### CephCSIOperatorConfig CRD

[This CRD](https://github.com/ceph/ceph-CSI-operator/blob/main/docs/design/operator.md#cephCSIoperatorconfig-crd)  manages operator-level configurations and offers a place to overwrite settings for CSI drivers. This CRD is a namespace-scoped CRD and a single CR named ceph-CSI-operator-config can be created by an user inside the operator's namespace.

### CephCSIDriver CRD

Rook will define three instances of [this CRD](https://github.com/ceph/ceph-CSI-operator/blob/main/docs/design/operator.md#cephCSIdriver-crd), one each for RBD, CephFS, and NFS for managing the installation, lifecycle management, and configuration for CephFS, RBD, and NFS CSI drivers.
The CRs will be empty (all the settings are defined as defaults in the previous CR), or else some small set of default driver-specific settings.

### CephCSIConfigMapping

[This CRD](https://github.com/ceph/ceph-CSI-operator/blob/main/docs/design/operator.md#cephCSIconfigmapping) contains a mapping between local and remote Ceph cluster configurations.It also provides a mapping between the CephCSIConfigurations.
Rook will define these mappings when the rbd mirroring is enabled.