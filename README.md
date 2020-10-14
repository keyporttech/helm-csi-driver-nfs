# helm-csi-driver-nfs

![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 2.0.0](https://img.shields.io/badge/AppVersion-2.0.0-informational?style=flat-square)

A Kubnetes Helm chart that mounts NFS volumes hosted on a remote server.

**Homepage:** <https://github.com/keyporttech/helm-charts>

## Introduction

This Kubernetes helm chart allows the management of Network File System (NFS) storage.  It does this by deploying, to each node in the cluster, a pod running both the driver and CSI Node Driver Registrar sidecar containers.  Applications can access existing NFS shares by simply specifying a Persistent Volume (PV) and Persistent Volume Claim (PVC).

This chart was developed and tested on kubernetes version 1.19, but should work on earlier or later versions.

## Requirements

Kubernetes: `>=1.14`

* helm 3 installation
* Nodes have enought resources to run the chart pod.
* Any one namespace to deploy into.
* NFS Client installed on each node in the cluster.
* An NFS server and existing accessible shares (confirm that the shares are accessible from one or more of the cluster nodes).

Images:

* Node Driver Registrar: https://github.com/kubernetes-csi/node-driver-registrar
* CSI NFS driver: https://github.com/kubernetes-csi/csi-driver-nfs

## Installing the Chart

There are two methods that can be used to install the chart.  The first is to use the chart from the Keyporttech Helm Repository.

```console
helm repo add keyporttech https://keyporttech.github.io/helm-charts/
helm install my-release keyporttech/helm-csi-driver-nfs -n my-namespace
```
or clone this repo and install from the local file system.

```console
$ helm install my-release . -n my-namespace
```
> **Tip**: There should not be a need to modify any of the chart values.  Use the default [values.yaml](values.yaml) file.
>

## Uninstalling the Chart

To uninstall/delete the `my-release` deployment:

```console
$ helm uninstall my-release -n my-namespace
```

All resources associated with the last release of the chart as well as its release history will be deleted.

## Storage

This chart enables applications to access existing NFS shares through SCs, PVs and PVCs.  By default no shares will be deleted when the chart is uninstalled.

In the repository root folder there is a examples subfolder containing the nginx web server.  It shows how to create a sc, pv and pvc outside the chart so that an application can mount an existing NFS share.

Applications do not have to be installed into the same namespace as this chart.  Already deployed applications can remain in their current namespace and be updated to use the driver.

See nginx pod example below.  The following may need to change:
* SC, PV and PVC name
* PV and PVC storage size
* PV mountOptions and volumeAttributes

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-driver-nfs-nodeplugin-nginx-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: nfs.csi.k8s.io
reclaimPolicy: Retain
volumeBindingMode: Immediate
parameters:
  storagepolicyname: "NFS CSI Driver"  # Optional Parameter
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: csi-driver-nfs-nodeplugin-nginx-pv
spec:
  #claimRef:
  #  name: csi-driver-nfs-nodeplugin-pvc
  #  namespace: apps
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  storageClassName: csi-driver-nfs-nodeplugin-nginx-sc
  mountOptions:
    - nfsvers=3
    - nolock
    #- hard
  csi:
    driver: nfs.csi.k8s.io
    volumeHandle: nginx-data-id
    volumeAttributes:
      # NFS server and mount path i.e. share or its subdirectory
      # server: IP or FQDN i.e. host.example.com
      server: 10.10.1.12
      share: mnt/Pool2/Virtualization/Kubernetes/Applications/csi-dir
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-driver-nfs-nodeplugin-nginx-pvc
spec:
  storageClassName: csi-driver-nfs-nodeplugin-nginx-sc
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  #serviceAccount: csi-driver-nfs-nodeplugin
  containers:
  - image: maersk/nginx
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
    volumeMounts:
      - mountPath: /var/www
        name: nginx-data-nfs-nodeplugin
  volumes:
  - name: nginx-data-nfs-nodeplugin
    persistentVolumeClaim:
      claimName: csi-driver-nfs-nodeplugin-nginx-pvc
```
This will enable direct communication between the NFS server and the CSI Driver running in the cluster.

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` | List of rules for which nodes the pod is eligible to be scheduled.  This method relies on node labels. |
| csiDriver.image.pullPolicy | string | `"IfNotPresent"` | When the image is pulled |
| csiDriver.image.repository | string | `"registry.keyporttech.com/csi-driver-nfs"` | Repository for the NFS CSI Driver |
| csiDriver.image.tag | string | `"2.0.0"` | Version tag for the image |
| csiDriver.name | string | `"nfs.csi.k8s.io"` | NFS CSI Driver name |
| csiDriver.securityContext.allowPrivilegeEscalation | bool | `true` | Can the current user context of the container be changed |
| csiDriver.securityContext.capabilities.add | list | `["SYS_ADMIN"]` | System admininstrator capability |
| csiDriver.securityContext.privileged | bool | `true` | Does the driver have operating system administrative capabilities |
| fullnameOverride | string | `"csi-driver-nfs-nodeplugin"` | If not empty, replaces the generated name for the deployment |
| imagePullSecrets | list | `[]` | List of secrets used to pull a private images for the pod |
| nameOverride | string | `""` | If not empty, replaces the name of the chart |
| nodeSelector | object | `{}` | List of key-value pairs used to select a node for pod deployment.  In order for the node to be eligible it must have each of the specified key-value pairs as labels. |
| podAnnotations | object | `{}` | A list of annotations for the pod |
| podSecurityContext | object | `{}` | Specifies the privilege and access control settings of the pod |
| rbac.enable | bool | `true` | Specfies whether ClusterRole and ClusterRoleBinding will be enabled for ServiceAccount |
| replicaCount | int | `1` | Number of pods to load balance between |
| resources | object | `{}` | Specifies the cpu and memory to be allocated for the pod |
| serviceAccount.annotations | object | `{}` | List of annotations for the account |
| serviceAccount.create | bool | `true` | Specifies whether a service account should be created |
| serviceAccount.name | string | `""` | The name of the service account.  If empty, a name is generated. |
| sidecar.image.pullPolicy | string | `"IfNotPresent"` | When the image is pulled |
| sidecar.image.repository | string | `"k8s.gcr.io/sig-storage/csi-node-driver-registrar"` | Repository for the Registrar CSI Driver |
| sidecar.image.tag | string | `"v1.3.0"` | Version tag for the image |
| sidecar.name | string | `"csi-driver-registrar"` | Registrar CSI Driver name |
| sidecar.securityContext | object | `{}` | Specifies the privilege and access control settings of the container |
| tolerations | list | `[]` | List of tolerations which allow the pod to be scheduled onto nodes with matching taints |

## Source Code

* <https://github.com/keyporttech/helm-csi-driver-nfs>

## Maintainers

| Name | Email | Url |
| ---- | ------ | --- |
| Keyport Technologies, Inc. | support@keyporttech.com | https://keyporttech.github.io/ |

## Contributing

Please see [keyporttech charts contribution guidelines](https://github.com/keyporttech/helm-charts/blob/master/CONTRIBUTING.md)

