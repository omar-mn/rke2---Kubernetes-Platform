# Longhorn Storage

official docs: **[Install with Helm](https://longhorn.io/docs/1.12.1/deploy/install/install-with-helm/)**

## What is Longhorn?

Longhorn is a distributed block storage system for Kubernetes. It provides persistent storage for workloads through Kubernetes `PersistentVolume` (PV), `PersistentVolumeClaim` (PVC), and `StorageClass` resources.

It also provides storage replication, snapshots, and backups for persistent volumes.

---

# Installing Longhorn

Longhorn was installed using Helm following the official documentation.

### Add the Helm repository

```bash
helm repo add longhorn https://charts.longhorn.io
```

### Update Helm repositories

```bash
helm repo update
```

### Install Longhorn

```bash
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --version 1.12.1
```

### Verify the installation

```bash
kubectl -n longhorn-system get pod
```

Longhorn components should be in the `Running` state after the installation completes.

---

# Dynamic vs Static Provisioning

## Dynamic Provisioning

With dynamic provisioning, we only create a `PVC` and specify the desired `StorageClass`.

Kubernetes and Longhorn automatically create the corresponding PV and Longhorn volume.

In our application:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres
  namespace: mates-app
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
```

The flow is:

```text
PVC
 │
 ▼
StorageClass: longhorn
 │
 ▼
Longhorn
 │
 ├── creates Volume
 └── creates/binds PV
```

The Longhorn StorageClass defines how the volume should be provisioned.

## Static Provisioning

With static provisioning, the storage volume already exists before the Kubernetes workload requests it.

The PV and PVC are then created manually and bound to the existing volume.

```text
Existing Volume
      │
      ▼
     PV
      │
      ▼
     PVC
      │
      ▼
     Pod
```

This is useful when binding Kubernetes workloads to an already-existing storage volume.

---

# Volume Replication

Longhorn can maintain multiple replicas of the same volume across different nodes.

The default `longhorn` StorageClass uses **3 replicas** unless configured otherwise. Longhorn schedules these replicas across available nodes to provide storage high availability.

```text
              Longhorn Volume
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Replica 1  Replica 2  Replica 3
          │          │          │
       Node 1      Node 2      Node 3
```

This allows a workload to access the same persistent data even when the Pod is rescheduled onto another node, provided the volume can be attached and the replicas remain available.

---

# Snapshots

Longhorn supports creating snapshots of volumes at any time.

A snapshot represents the volume's state at a specific point in time and can be used for recovery or creating another volume.

```text
Volume
  │
  ├── Snapshot 1
  ├── Snapshot 2
  └── Snapshot 3
```

---

# Backups

Longhorn can also back up volumes to an external backup target.

Instead of keeping the data only inside the cluster, backups can be stored on an external storage system such as an S3-compatible object store or another supported backup target.

```text
Longhorn Volume
      │
      ▼
   Snapshot
      │
      ▼
   Longhorn Backup
      │
      ▼
External Backup Target
```

This provides an additional recovery mechanism in case the cluster or its local storage becomes unavailable. Longhorn supports configuring a backup target and creating backups from volume snapshots.
