# NFS CSI Driver Helm chart
[NFS CSI Driver](https://github.com/kubernetes-csi/csi-driver-nfs) implements the [Container Storage Interface (CSI) specification](https://github.com/container-storage-interface/spec/blob/master/spec.md) to enable the mounting of existing NFS shares into applications.

## Introduction

This Kubernetes helm chart allows the management of Network File System (NFS) storage.  It does this by deploying, to each node in the cluster, a pod running both the driver and CSI Node Driver Registrar sidecar containers.  Applications can access existing NFS shares by simply specifying a Persistent Volume (PV) and Persistent Volume Claim (PVC).

This chart was developed and tested on kubernetes version 1.19, but should work on earlier or later versions.

## Prerequisites

- A kubernetes cluster 1.14 or later ( most recent release recommended)
- helm 3 installation
- Nodes have enought resources to run the NFS pods.
- An NFS server and existing accessible shares (confirm that the shares are accessible from one or more of the cluster nodes).

## Installing the Chart

There are two methods that can be used to install the chart.  The first is to use the chart from the Keyporttech Helm Repository.

```bash
helm repo add keyporttech https://keyporttech.github.io/helm-charts/
helm install my-release keyporttech/helm-csi-driver-nfs -n my-namespace
```
or clone this repo and install from the local file system.

```bash
$ helm install my-release . -n my-namespace
```
> **Tip**: There should not be a need to modify any of the chart values.  Use the default [values.yaml](values.yaml) file.
>

### Values File Configuration
By default the chart will use the default values.yaml file unless a custom values file is specified.

**Note:** The image in the quay.io/k8 registry was not accessible.  The driver image consequently had to be built using the dockerfile in the NFS CSI Driver git repository and hosted in the keyporttech image registry.

```yaml
csiDriver:
  name: nfs.csi.k8s.io
  # Build and push to image registry as pulling
  #   docker pull quay.io/k8scsi/csi-driver-nfs:v2.0.0
  # is failing.
  image:
    repository: registry.keyporttech.com:30243/csi-driver-nfs
    pullPolicy: IfNotPresent
    # Overrides the image tag whose default is the chart appVersion.
    tag: v2.0.0

```


## Uninstalling the Chart

To uninstall/delete the `my-release` deployment:

```bash
$ helm uninstall my-release -n my-namespace
```

All resources associated with the last release of the chart as well as its release history will be deleted.

## Data Management

This chart enables applications to access existing NFS shares through PVs and PVCs.  By default no shares will be deleted when the chart is uninstalled.

In the repository root folder there is a examples subfolder containing the nginx web server.  It shows how to create a pv and pvc outside the chart so that an application can mount an existing NFS share.

Applications do not have to be installed into the same namespace as this chart.  Already deployed applications can remain in their current namespace and be updated to use the driver.

See nginx pod example below.  The following may need to change:
* PV and PVC name
* PV and PVC storage size
* PV mountOptions and volumeAttributes


```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: csi-driver-nfs-nodeplugin-pv
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
  storageClassName: csi-driver-nfs-nodeplugin
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
      server: 192.168.x.x
      share: /mnt/Pool/share/csi-dir
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-driver-nfs-nodeplugin-pvc
spec:
  storageClassName: csi-driver-nfs-nodeplugin
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
      claimName: csi-driver-nfs-nodeplugin-pvc
```
This will enable direct communication between the NFS server and the CSI Driver running in the cluster.


## Configuration

Refer to [values.yaml](values.yaml) for the full run-down on defaults.

The following table lists the configurable parameters of this chart and their default values.

| Parameter                  | Description                                     | Default                                                    |
| -----------------------    | ---------------------------------------------   | ---------------------------------------------------------- |
| `sidecar.name`                 | CSI Driver Registrar name                            | `csi-driver-registrar`                                                    |
| `sidecar.image.repository`                    | CSI Driver Registrar image registry                     | `k8s.gcr.io/sig-storage/csi-node-driver-registrar`                                                 |
| `sidecar.image.tag`                    | CSI Driver Registrar image version                     | `v1.3.0`                                                 |
| `csiDriver.name`                 | CSI Driver name                            | `nfs.csi.k8s.io`                                                    |
| `csiDriver.image.repository`                    | CSI Driver image registry                     | `registry.keyporttech.com:30243/csi-driver-nfs:v2.0.0`                                                 |
| `csiDriver.image.tag`                    | CSI Driver image version                     | `v2.0.0`                                                 |
| `fullnameOverride`                 | Chart and deployment name                            | `csi-driver-nfs-nodeplugin`                                                    |
| `serviceAccount.name`                 | ServiceAccount name                            | `csi-driver-nfs-nodeplugin`                                                    |
| `storageClass.name`                 | StorageClass name                            | `csi-driver-nfs-nodeplugin`                                                    |

## Resources

* Node Driver Registrar: https://github.com/kubernetes-csi/node-driver-registrar
* CSI NFS driver: https://github.com/kubernetes-csi/csi-driver-nfs

## Contributing

Please see [keyporttech charts contribution guidelines](https://github.com/keyporttech/helm-charts/blob/master/CONTRIBUTING.md)
