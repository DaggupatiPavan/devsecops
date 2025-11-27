# 09. Storage Deep Dive

## 1. Storage Architecture

Kubernetes decouples storage provision from usage.

### PersistentVolume (PV)
A piece of storage in the cluster. It is a cluster-level resource (not namespaced).
- **Static Provisioning**: Admin creates PVs manually.
- **Dynamic Provisioning**: Cluster creates PVs automatically via StorageClasses.

### PersistentVolumeClaim (PVC)
A request for storage by a user. It is namespaced.
- Pods use PVCs as volumes.
- PVCs bind to PVs that satisfy their requirements (size, access mode).

### StorageClass (SC)
Defines "classes" of storage (e.g., "fast-ssd", "cheap-hdd").
- **Provisioner**: The plugin that creates the volume (e.g., `ebs.csi.aws.com`).
- **ReclaimPolicy**: What happens to PV when PVC is deleted?
    - `Delete`: Delete the underlying cloud volume (Default).
    - `Retain`: Keep the volume for manual recovery.

### Access Modes
- **ReadWriteOnce (RWO)**: Mounted by single node as R/W. (Block storage like EBS).
- **ReadOnlyMany (ROX)**: Mounted by multiple nodes as Read-only.
- **ReadWriteMany (RWX)**: Mounted by multiple nodes as R/W. (File storage like NFS/EFS).

## 2. Container Storage Interface (CSI)

CSI is the standard for exposing block and file storage systems to container orchestration systems.

### Architecture
- **Controller Plugin**: Runs as a Deployment (usually 1 replica). Talks to Cloud API to Create/Delete volumes.
- **Node Plugin**: Runs as a DaemonSet on every node. Mounts/Unmounts volumes to the Pod directory.

### Sidecars
CSI drivers use helper sidecars provided by Kubernetes:
- **external-provisioner**: Watches PVCs and calls `CreateVolume`.
- **external-attacher**: Calls `ControllerPublish` (attach disk to node).
- **node-driver-registrar**: Registers the driver with Kubelet.

## 3. Advanced Storage Features

### Volume Snapshots
Standard way to trigger snapshots of volume contents.
1.  **VolumeSnapshotClass**: Defines the driver and parameters.
2.  **VolumeSnapshot**: The request to take a snapshot of a specific PVC.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: new-snapshot-demo
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: pvc-test
```

### Volume Expansion
Allow resizing PVCs without downtime.
- `allowVolumeExpansion: true` in StorageClass.
- Edit PVC `spec.resources.requests.storage`.
- File system resizing happens automatically (online) or on next pod restart (offline).

### Volume Cloning
Create a new PVC populated with data from an existing PVC.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  dataSource:
    name: source-pvc
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```
