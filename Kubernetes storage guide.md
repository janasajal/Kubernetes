# Kubernetes Storage

**Author:** Sajal Jana

A production-focused guide for managing persistent storage in Kubernetes clusters.

---

## Table of Contents

- [1. Persistent Data](#1-persistent-data)
  - [1.1 Storage Architecture Overview](#11-storage-architecture-overview)
  - [1.2 PersistentVolumes (PV)](#12-persistentvolumes-pv)
    - [Creating PersistentVolumes](#creating-persistentvolumes)
    - [PV Lifecycle](#pv-lifecycle)
    - [PV Types](#pv-types)
  - [1.3 PersistentVolumeClaims (PVC)](#13-persistentvolumeclaims-pvc)
    - [Creating PVCs](#creating-pvcs)
    - [Using PVCs in Pods](#using-pvcs-in-pods)
  - [1.4 PV and PVC Binding](#14-pv-and-pvc-binding)
    - [Binding Process](#binding-process)
    - [Binding Modes](#binding-modes)
    - [Selector-Based Binding](#selector-based-binding)
  - [1.5 Volume Expansion](#15-volume-expansion)
  - [1.6 Volume Snapshots](#16-volume-snapshots)
- [2. Storage Classes](#2-storage-classes)
  - [2.1 Understanding StorageClasses](#21-understanding-storageclasses)
  - [2.2 Dynamic Provisioning](#22-dynamic-provisioning)
  - [2.3 Storage Class Configuration](#23-storage-class-configuration)
    - [AWS EBS StorageClass](#aws-ebs-storageclass)
    - [GCP Persistent Disk StorageClass](#gcp-persistent-disk-storageclass)
    - [Azure Disk StorageClass](#azure-disk-storageclass)
    - [NFS StorageClass](#nfs-storageclass)
    - [Local Storage StorageClass](#local-storage-storageclass)
  - [2.4 Default StorageClass](#24-default-storageclass)
  - [2.5 Volume Binding Modes](#25-volume-binding-modes)
  - [2.6 Storage Class Parameters](#26-storage-class-parameters)
- [3. Volume Operations](#3-volume-operations)
  - [3.1 Access Modes](#31-access-modes)
    - [Access Mode Combinations](#access-mode-combinations)
    - [Access Mode Support by Volume Type](#access-mode-support-by-volume-type)
  - [3.2 Reclaim Policies](#32-reclaim-policies)
    - [Retain](#retain)
    - [Delete](#delete)
    - [Recycle (Deprecated)](#recycle-deprecated)
  - [3.3 Volume Types](#33-volume-types)
    - [EmptyDir](#emptydir)
    - [HostPath](#hostpath)
    - [ConfigMap and Secret Volumes](#configmap-and-secret-volumes)
    - [PersistentVolumeClaim](#persistentvolumeclaim)
    - [Cloud Provider Volumes](#cloud-provider-volumes)
    - [Network Storage](#network-storage)
  - [3.4 Volume Modes](#34-volume-modes)
  - [3.5 Mount Options](#35-mount-options)
  - [3.6 Storage Quotas and Limits](#36-storage-quotas-and-limits)
- [4. Common Patterns and Best Practices](#4-common-patterns-and-best-practices)
  - [4.1 StatefulSet with Persistent Storage](#41-statefulset-with-persistent-storage)
  - [4.2 Shared Storage Pattern](#42-shared-storage-pattern)
  - [4.3 Backup and Restore](#43-backup-and-restore)
  - [4.4 Storage Best Practices](#44-storage-best-practices)
- [5. Troubleshooting](#5-troubleshooting)
  - [5.1 PVC Stuck in Pending](#51-pvc-stuck-in-pending)
  - [5.2 Pod Stuck in ContainerCreating](#52-pod-stuck-in-containercreating)
  - [5.3 Volume Mount Errors](#53-volume-mount-errors)
  - [5.4 Storage Performance Issues](#54-storage-performance-issues)
- [6. Quick Reference](#6-quick-reference)
  - [6.1 Essential Commands](#61-essential-commands)
  - [6.2 Common Scenarios](#62-common-scenarios)
- [7. Production Checklist](#7-production-checklist)
- [8. Further Reading](#8-further-reading)

---

## 1. Persistent Data

Kubernetes provides persistent storage that survives pod restarts and rescheduling.

### 1.1 Storage Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Storage Architecture                     │
└─────────────────────────────────────────────────────────────┘

Developer/User               Cluster Admin
     │                             │
     │ 1. Request storage          │ 2. Provision storage
     │    via PVC                  │    via PV or StorageClass
     │                             │
     ▼                             ▼
┌─────────────────┐         ┌──────────────┐
│      PVC        │◄────────┤      PV      │
│  (Claim/Request)│ Binding │  (Resource)  │
└────────┬────────┘         └──────┬───────┘
         │                         │
         │ 3. Mount volume         │
         ▼                         ▼
    ┌─────────┐            ┌──────────────┐
    │   Pod   │            │   Storage    │
    └─────────┘            │   Backend    │
                           │ (EBS/NFS/etc)│
                           └──────────────┘
```

**Key Concepts:**

- **PersistentVolume (PV):** Cluster resource representing physical storage
- **PersistentVolumeClaim (PVC):** User request for storage
- **StorageClass:** Template for dynamic provisioning
- **Volume:** Directory accessible to containers in a pod

### 1.2 PersistentVolumes (PV)

A cluster-wide resource representing physical storage provisioned by an administrator or dynamically via StorageClass.

#### Creating PersistentVolumes

**Static NFS PersistentVolume:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    server: nfs-server.example.com
    path: "/exported/path"
```

**Static HostPath PersistentVolume (for testing only):**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: "/mnt/data"
    type: DirectoryOrCreate
```

**Cloud Provider PV (AWS EBS - usually dynamic):**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: gp3
  awsElasticBlockStore:
    volumeID: vol-0abc123def456
    fsType: ext4
```

```bash
# Create PV
kubectl apply -f pv.yaml

# List PVs
kubectl get pv

# Describe PV
kubectl describe pv nfs-pv

# Delete PV
kubectl delete pv nfs-pv
```

#### PV Lifecycle

```
Available → Bound → Released → (Reclaimed) → Available
```

**States:**

1. **Available:** Ready for binding to a PVC
2. **Bound:** Bound to a PVC
3. **Released:** PVC deleted, but resource not yet reclaimed
4. **Failed:** Automatic reclamation failed

```bash
# Check PV status
kubectl get pv nfs-pv -o jsonpath='{.status.phase}'
```

#### PV Types

**Common PV Types:**

| Type | Use Case | Access Modes |
|------|----------|--------------|
| **awsElasticBlockStore** | AWS EBS volumes | RWO |
| **gcePersistentDisk** | GCP Persistent Disks | RWO, ROX |
| **azureDisk** | Azure Disk Storage | RWO |
| **azureFile** | Azure File Storage | RWO, ROX, RWX |
| **nfs** | NFS shares | RWO, ROX, RWX |
| **cephfs** | Ceph File System | RWO, ROX, RWX |
| **glusterfs** | GlusterFS volumes | RWO, ROX, RWX |
| **iscsi** | iSCSI storage | RWO, ROX |
| **local** | Local node storage | RWO |
| **hostPath** | Node filesystem (testing only) | RWO |

### 1.3 PersistentVolumeClaims (PVC)

A request for storage by a user. Claims consume PV resources.

#### Creating PVCs

**Basic PVC:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

**PVC with Specific PV Selector:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
  selector:
    matchLabels:
      environment: production
      tier: database
```

**PVC for Shared Storage:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data-pvc
spec:
  accessModes:
    - ReadWriteMany  # Multiple pods can read/write
  resources:
    requests:
      storage: 100Gi
  storageClassName: nfs
```

```bash
# Create PVC
kubectl apply -f pvc.yaml

# List PVCs
kubectl get pvc

# Describe PVC
kubectl describe pvc mysql-pvc

# Check PVC status
kubectl get pvc mysql-pvc -o jsonpath='{.status.phase}'

# Delete PVC
kubectl delete pvc mysql-pvc
```

#### Using PVCs in Pods

**Single Container Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: password
    volumeMounts:
    - name: mysql-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-storage
    persistentVolumeClaim:
      claimName: mysql-pvc
```

**Deployment with PVC:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:latest
        volumeMounts:
        - name: webapp-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: webapp-storage
        persistentVolumeClaim:
          claimName: webapp-pvc
```

**Multiple Containers Sharing Volume:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: content-generator
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do date > /data/index.html; sleep 60; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    persistentVolumeClaim:
      claimName: shared-pvc
```

### 1.4 PV and PVC Binding

#### Binding Process

**How PV and PVC Bind:**

1. User creates PVC with requirements (size, access mode, StorageClass)
2. Kubernetes finds suitable PV matching requirements
3. PV and PVC are bound (1:1 relationship)
4. PVC can be used by pods

**Binding Criteria:**

- **Storage Size:** PV must have >= requested size
- **Access Modes:** PV must support requested access mode
- **StorageClass:** Must match (or both empty)
- **Selector:** If PVC has selector, PV must match labels

**Example:**

```yaml
# PV with labels
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-fast
  labels:
    type: ssd
    environment: production
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  awsElasticBlockStore:
    volumeID: vol-12345
---
# PVC requesting labeled PV
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-fast
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: fast
  selector:
    matchLabels:
      type: ssd
      environment: production
```

#### Binding Modes

**Immediate Binding (Default):**
- PVC binds to PV as soon as PVC is created
- Volume provisioning happens immediately

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
volumeBindingMode: Immediate  # Default
```

**WaitForFirstConsumer:**
- Delays binding until pod using PVC is scheduled
- Ensures volume is created in correct availability zone/region

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-delayed
provisioner: kubernetes.io/aws-ebs
volumeBindingMode: WaitForFirstConsumer
```

**Why use WaitForFirstConsumer?**

```yaml
# Problem with Immediate binding:
# 1. PVC created
# 2. Volume provisioned in zone us-east-1a
# 3. Pod scheduled to node in zone us-east-1b
# 4. Volume can't attach (wrong zone)

# Solution: WaitForFirstConsumer
# 1. PVC created (no volume yet)
# 2. Pod scheduled to node in us-east-1b
# 3. Volume provisioned in us-east-1b (same zone as pod)
# 4. Volume successfully attaches
```

#### Selector-Based Binding

**Label-based Selection:**

```yaml
# PV with custom labels
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-database
  labels:
    usage: database
    performance: high
    backup: enabled
spec:
  capacity:
    storage: 200Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: premium
---
# PVC selecting specific PV
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-database
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
  storageClassName: premium
  selector:
    matchLabels:
      usage: database
      performance: high
    matchExpressions:
    - key: backup
      operator: In
      values:
      - enabled
```

### 1.5 Volume Expansion

Increase PVC size without recreating volumes.

**Prerequisites:**
- StorageClass must allow expansion: `allowVolumeExpansion: true`
- Volume plugin must support expansion
- PVC must be in Bound state

**StorageClass with Expansion:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable
provisioner: kubernetes.io/aws-ebs
allowVolumeExpansion: true
parameters:
  type: gp3
```

**Expanding a PVC:**

```yaml
# Original PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi  # Original size
  storageClassName: expandable
```

```bash
# Method 1: Edit PVC
kubectl edit pvc mysql-pvc
# Change: storage: 10Gi → storage: 20Gi

# Method 2: Patch PVC
kubectl patch pvc mysql-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Check expansion status
kubectl describe pvc mysql-pvc
kubectl get pvc mysql-pvc -o jsonpath='{.status.capacity.storage}'
```

**Online vs Offline Expansion:**

**Online Expansion (preferred):**
- Volume expanded while pod is running
- Supported by most cloud providers (AWS EBS, GCP PD, Azure Disk)

**Offline Expansion:**
- Requires pod restart
- File system expansion happens during pod restart

```yaml
# Check if resize is needed
kubectl get pvc mysql-pvc -o yaml | grep -A 5 conditions

# If condition shows "FileSystemResizePending":
# Restart pod to complete expansion
kubectl delete pod mysql-pod
```

**Important Notes:**

- ✅ Can only increase size (never decrease)
- ✅ Online expansion requires CSI driver support
- ⚠️ Backup data before expansion
- ⚠️ Monitor disk usage after expansion

### 1.6 Volume Snapshots

Create point-in-time copies of volumes.

**Prerequisites:**
- CSI driver with snapshot support
- VolumeSnapshotClass configured

**VolumeSnapshotClass:**

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Delete  # or Retain
```

**Creating a Snapshot:**

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: mysql-pvc
```

```bash
# Create snapshot
kubectl apply -f snapshot.yaml

# List snapshots
kubectl get volumesnapshots

# Describe snapshot
kubectl describe volumesnapshot mysql-snapshot
```

**Restoring from Snapshot:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc-restored
spec:
  dataSource:
    name: mysql-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: expandable
```

---

## 2. Storage Classes

StorageClasses enable dynamic provisioning of storage.

### 2.1 Understanding StorageClasses

**Purpose:**
- Abstract storage implementation details
- Enable dynamic provisioning
- Define storage characteristics (performance, availability)
- Set default behaviors (reclaim policy, binding mode)

**Architecture:**

```
User creates PVC → References StorageClass → 
  Provisioner creates PV → PV bound to PVC
```

```bash
# List StorageClasses
kubectl get storageclasses
kubectl get sc  # Short form

# Describe StorageClass
kubectl describe sc standard

# View StorageClass YAML
kubectl get sc standard -o yaml
```

### 2.2 Dynamic Provisioning

Automatically create PVs when PVCs are created.

**Without Dynamic Provisioning (Static):**

```yaml
# Admin manually creates PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-001
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
---
# User creates PVC, waits for manual PV
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**With Dynamic Provisioning:**

```yaml
# Admin creates StorageClass (once)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
---
# User creates PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast  # PV auto-created
```

### 2.3 Storage Class Configuration

#### AWS EBS StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com  # CSI driver
parameters:
  type: gp3                    # Volume type
  iops: "3000"                 # IOPS (gp3, io1, io2)
  throughput: "125"            # Throughput MB/s (gp3)
  encrypted: "true"            # Enable encryption
  kmsKeyId: "arn:aws:kms:..."  # KMS key for encryption
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
```

**EBS Volume Types:**

```yaml
# General Purpose SSD (gp3) - balanced
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"

# General Purpose SSD (gp2) - older
parameters:
  type: gp2

# Provisioned IOPS SSD (io2) - high performance
parameters:
  type: io2
  iops: "10000"

# Throughput Optimized HDD (st1) - big data
parameters:
  type: st1

# Cold HDD (sc1) - archival
parameters:
  type: sc1
```

#### GCP Persistent Disk StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd               # pd-standard, pd-ssd, pd-balanced
  replication-type: none     # none, regional-pd
  disk-encryption-kms-key: "projects/..."
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
```

**GCP Disk Types:**

```yaml
# SSD Persistent Disk - high performance
parameters:
  type: pd-ssd

# Balanced Persistent Disk - balanced
parameters:
  type: pd-balanced

# Standard Persistent Disk - cost-effective
parameters:
  type: pd-standard

# Regional Persistent Disk - replicated
parameters:
  type: pd-ssd
  replication-type: regional-pd
```

#### Azure Disk StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium
provisioner: disk.csi.azure.com
parameters:
  storageaccounttype: Premium_LRS  # Disk type
  kind: Managed                     # Managed or Dedicated
  cachingMode: ReadOnly             # None, ReadOnly, ReadWrite
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
```

**Azure Disk Types:**

```yaml
# Premium SSD
parameters:
  storageaccounttype: Premium_LRS

# Standard SSD
parameters:
  storageaccounttype: StandardSSD_LRS

# Standard HDD
parameters:
  storageaccounttype: Standard_LRS

# Ultra SSD
parameters:
  storageaccounttype: UltraSSD_LRS
```

#### NFS StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.example.com
  share: /exported/path
  mountOptions:
    - hard
    - nfsvers=4.1
volumeBindingMode: Immediate
allowVolumeExpansion: false
reclaimPolicy: Delete
```

#### Local Storage StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

**Local PV (used with local StorageClass):**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node1
```

### 2.4 Default StorageClass

Set a default StorageClass for PVCs that don't specify one.

**Mark StorageClass as Default:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
```

```bash
# Set default StorageClass
kubectl annotate storageclass standard \
  storageclass.kubernetes.io/is-default-class=true

# Remove default annotation
kubectl annotate storageclass standard \
  storageclass.kubernetes.io/is-default-class-

# List default StorageClass
kubectl get sc
# Look for "(default)" in output
```

**Using Default StorageClass:**

```yaml
# PVC without storageClassName uses default
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  # storageClassName not specified - uses default
```

**Disable Default StorageClass:**

```yaml
# Explicitly set to empty string
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: ""  # Don't use default
```

### 2.5 Volume Binding Modes

**Immediate:**
- Volume provisioned when PVC created
- May cause zone/region conflicts

```yaml
volumeBindingMode: Immediate
```

**WaitForFirstConsumer (Recommended):**
- Delays provisioning until pod scheduled
- Ensures volume in same zone as pod

```yaml
volumeBindingMode: WaitForFirstConsumer
```

**Example Scenario:**

```yaml
# Cluster with nodes in multiple zones
# us-east-1a, us-east-1b, us-east-1c

# StorageClass with WaitForFirstConsumer
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-zonal
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: topology.ebs.csi.aws.com/zone
    values:
    - us-east-1a
    - us-east-1b
    - us-east-1c
```

### 2.6 Storage Class Parameters

**Common Parameters:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: custom-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3                    # Provisioner-specific
  encrypted: "true"            # Enable encryption
  iops: "3000"                 # IOPS setting
  fsType: ext4                 # Filesystem type
mountOptions:                  # Mount options
  - discard
  - noatime
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
```

**Filesystem Types:**

```yaml
parameters:
  fsType: ext4     # Default for most Linux
  # fsType: ext3   # Older Linux
  # fsType: xfs    # Better for large files
  # fsType: btrfs  # Advanced features
```

**Mount Options:**

```yaml
mountOptions:
  - discard        # Enable TRIM for SSD
  - noatime        # Don't update access time
  - nodiratime     # Don't update directory access time
  - barrier=0      # Disable write barriers (performance)
```

---

## 3. Volume Operations

### 3.1 Access Modes

Define how volumes can be mounted by nodes.

**Three Access Modes:**

1. **ReadWriteOnce (RWO)**
   - Volume mounted read-write by single node
   - Most common for block storage

2. **ReadOnlyMany (ROX)**
   - Volume mounted read-only by many nodes
   - Used for shared read-only data

3. **ReadWriteMany (RWX)**
   - Volume mounted read-write by many nodes
   - Required for shared storage (NFS, CephFS)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteOnce  # Can specify multiple
    - ReadOnlyMany
  resources:
    requests:
      storage: 10Gi
```

#### Access Mode Combinations

**Single Access Mode:**

```yaml
# RWO only - database, single instance apps
accessModes:
  - ReadWriteOnce

# RWX only - shared file storage
accessModes:
  - ReadWriteMany

# ROX only - shared read-only data
accessModes:
  - ReadOnlyMany
```

**Multiple Access Modes:**

```yaml
# Support both RWO and ROX
# PV can be bound in either mode
accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
```

#### Access Mode Support by Volume Type

| Volume Type | RWO | ROX | RWX |
|-------------|-----|-----|-----|
| **AWS EBS** | ✅ | ❌ | ❌ |
| **GCP PD** | ✅ | ✅ | ❌ |
| **Azure Disk** | ✅ | ❌ | ❌ |
| **Azure File** | ✅ | ✅ | ✅ |
| **NFS** | ✅ | ✅ | ✅ |
| **CephFS** | ✅ | ✅ | ✅ |
| **GlusterFS** | ✅ | ✅ | ✅ |
| **HostPath** | ✅ | ❌ | ❌ |
| **Local** | ✅ | ❌ | ❌ |

**Use Cases:**

```yaml
# Database (RWO) - single writer
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast

---
# Shared content (RWX) - multiple writers
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-content-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
  storageClassName: nfs

---
# Static assets (ROX) - multiple readers
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: static-assets-pvc
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: nfs
```

### 3.2 Reclaim Policies

Defines what happens to PV after PVC is deleted.

#### Retain

**Behavior:**
- PV remains after PVC deleted
- Data preserved
- Manual cleanup required
- PV status changes to "Released"

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-important-data
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
```

**When to Use:**
- Production databases
- Important data requiring backup
- Compliance requirements

**Cleanup Process:**

```bash
# 1. Delete PVC
kubectl delete pvc my-pvc

# 2. PV status becomes "Released"
kubectl get pv

# 3. Manually backup data
# 4. Delete PV
kubectl delete pv my-pv

# 5. Delete underlying storage (if needed)
# aws ec2 delete-volume --volume-id vol-xxx
```

#### Delete

**Behavior:**
- PV and underlying storage deleted when PVC deleted
- Automatic cleanup
- **Data permanently lost**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-temp-data
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: standard
```

**When to Use:**
- Temporary data
- Development/testing environments
- Stateless applications
- Cost optimization (auto-cleanup)

**⚠️ Warning:** Default for most dynamic provisioning

#### Recycle (Deprecated)

**Behavior:**
- Basic scrub (rm -rf /volume/*)
- PV available for new claim
- **Deprecated** - don't use

**Alternative:** Use Delete policy or manual cleanup

### 3.3 Volume Types

#### EmptyDir

Temporary storage that exists as long as pod exists.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-pod
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir: {}  # Deleted when pod deleted
```

**EmptyDir with Memory:**

```yaml
volumes:
- name: cache
  emptyDir:
    medium: Memory  # tmpfs - fast but uses node RAM
    sizeLimit: 1Gi
```

**Use Cases:**
- Temporary cache
- Scratch space
- Sharing files between containers in pod

#### HostPath

Mount directory from node's filesystem.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: host-data
      mountPath: /data
  volumes:
  - name: host-data
    hostPath:
      path: /mnt/data
      type: DirectoryOrCreate
```

**HostPath Types:**

```yaml
# Directory must exist
hostPath:
  path: /mnt/data
  type: Directory

# Create directory if doesn't exist
hostPath:
  path: /mnt/data
  type: DirectoryOrCreate

# File must exist
hostPath:
  path: /mnt/data/config.yaml
  type: File

# Socket must exist
hostPath:
  path: /var/run/docker.sock
  type: Socket
```

**Use Cases:**
- Access Docker socket for DinD
- Node monitoring agents
- Log collection

**⚠️ Security Risks:**
- Pod can access node filesystem
- Only use for trusted workloads
- Never in multi-tenant clusters

#### ConfigMap and Secret Volumes

Mount ConfigMaps and Secrets as files.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: config
      mountPath: /etc/config
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
  - name: secrets
    secret:
      secretName: app-secrets
```

#### PersistentVolumeClaim

Mount persistent storage.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    volumeMounts:
    - name: data
      mountPath: /var/lib/mysql
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: mysql-pvc
```

#### Cloud Provider Volumes

**AWS EBS (direct mount - not recommended):**

```yaml
volumes:
- name: data
  awsElasticBlockStore:
    volumeID: vol-0abc123
    fsType: ext4
```

**GCE Persistent Disk:**

```yaml
volumes:
- name: data
  gcePersistentDisk:
    pdName: my-disk
    fsType: ext4
```

**Azure Disk:**

```yaml
volumes:
- name: data
  azureDisk:
    diskName: my-disk
    diskURI: /subscriptions/.../disks/my-disk
```

**⚠️ Recommendation:** Use PVC + StorageClass instead

#### Network Storage

**NFS:**

```yaml
volumes:
- name: nfs
  nfs:
    server: nfs-server.example.com
    path: /exported/path
    readOnly: false
```

**CephFS:**

```yaml
volumes:
- name: cephfs
  cephfs:
    monitors:
    - 10.0.0.1:6789
    - 10.0.0.2:6789
    user: admin
    secretRef:
      name: ceph-secret
    path: /
```

**GlusterFS:**

```yaml
volumes:
- name: gluster
  glusterfs:
    endpoints: glusterfs-cluster
    path: my-volume
    readOnly: false
```

### 3.4 Volume Modes

Define how volume is consumed: filesystem or raw block.

**Filesystem (Default):**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-fs
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem  # Default
  resources:
    requests:
      storage: 10Gi
```

**Block:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-block
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block  # Raw block device
  resources:
    requests:
      storage: 10Gi
```

**Using Block Volume:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: block-consumer
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeDevices:  # Not volumeMounts
    - name: data
      devicePath: /dev/xvda  # Block device path
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-block
```

**Use Cases:**
- Databases needing raw block device
- Custom filesystems
- Direct disk access

### 3.5 Mount Options

Additional options for mounting volumes.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: custom-mount
provisioner: kubernetes.io/aws-ebs
mountOptions:
  - discard       # Enable TRIM
  - noatime       # Don't update access time
  - nodiratime    # Don't update dir access time
  - barrier=0     # Disable write barriers
parameters:
  type: gp3
```

**NFS Mount Options:**

```yaml
mountOptions:
  - hard          # Hard mount (recommended)
  - nfsvers=4.1   # NFS version
  - noatime       # Performance
  - nodiratime    # Performance
  - rsize=1048576 # Read buffer size
  - wsize=1048576 # Write buffer size
```

**Common Mount Options:**

```yaml
# Performance
mountOptions:
  - noatime       # Don't update access time
  - nodiratime    # Don't update directory access time

# Reliability (NFS)
mountOptions:
  - hard          # Retry forever
  - intr          # Allow interrupts

# Security
mountOptions:
  - ro            # Read-only
  - noexec        # No execute
  - nosuid        # No setuid
```

### 3.6 Storage Quotas and Limits

Limit storage consumption per namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: dev
spec:
  hard:
    # Total storage across all PVCs
    requests.storage: "100Gi"
    
    # Number of PVCs
    persistentvolumeclaims: "10"
    
    # Storage by StorageClass
    standard.storageclass.storage.k8s.io/requests.storage: "50Gi"
    fast.storageclass.storage.k8s.io/requests.storage: "20Gi"
    
    # Number of PVCs by StorageClass
    standard.storageclass.storage.k8s.io/persistentvolumeclaims: "5"
    fast.storageclass.storage.k8s.io/persistentvolumeclaims: "2"
```

```bash
# Create quota
kubectl apply -f storage-quota.yaml

# View quota
kubectl get resourcequota storage-quota -n dev

# Describe quota (shows usage)
kubectl describe resourcequota storage-quota -n dev
```

**LimitRange for PVCs:**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pvc-limit
  namespace: dev
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 50Gi  # Max per PVC
    min:
      storage: 1Gi   # Min per PVC
```

---

## 4. Common Patterns and Best Practices

### 4.1 StatefulSet with Persistent Storage

StatefulSets automatically create PVCs for each pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None  # Headless
  selector:
    app: mysql
  ports:
  - port: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  # VolumeClaimTemplates - automatic PVC per pod
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: fast
      resources:
        requests:
          storage: 100Gi
```

**How it works:**
- Creates PVC: `data-mysql-0`, `data-mysql-1`, `data-mysql-2`
- Each pod gets dedicated storage
- PVCs persist even if StatefulSet deleted

```bash
# View StatefulSet PVCs
kubectl get pvc -l app=mysql

# Scale StatefulSet (creates more PVCs)
kubectl scale statefulset mysql --replicas=5

# Delete StatefulSet (PVCs remain)
kubectl delete statefulset mysql

# Delete PVCs manually
kubectl delete pvc data-mysql-0 data-mysql-1 data-mysql-2
```

### 4.2 Shared Storage Pattern

Multiple pods accessing same volume.

**ReadWriteMany Example:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage
spec:
  accessModes:
    - ReadWriteMany  # RWX required
  resources:
    requests:
      storage: 100Gi
  storageClassName: nfs  # Must support RWX
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 5  # Multiple pods
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx
        volumeMounts:
        - name: shared
          mountPath: /usr/share/nginx/html
      volumes:
      - name: shared
        persistentVolumeClaim:
          claimName: shared-storage  # Same PVC
```

**ReadOnlyMany Example:**

```yaml
# Upload pod - writes data
apiVersion: v1
kind: Pod
metadata:
  name: content-uploader
spec:
  containers:
  - name: uploader
    image: uploader:v1
    volumeMounts:
    - name: content
      mountPath: /data
  volumes:
  - name: content
    persistentVolumeClaim:
      claimName: static-content
---
# Web servers - read data
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 10
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: content
          mountPath: /usr/share/nginx/html
          readOnly: true  # Read-only
      volumes:
      - name: content
        persistentVolumeClaim:
          claimName: static-content
```

### 4.3 Backup and Restore

**Backup Strategies:**

**1. Volume Snapshots (Recommended):**

```yaml
# Create snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-backup-20240215
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: mysql-pvc
```

**2. Application-Level Backup:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
spec:
  schedule: "0 2 * * *"  # Daily 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: mysql:8.0
            command:
            - /bin/sh
            - -c
            - |
              mysqldump -h mysql -u root -p$MYSQL_ROOT_PASSWORD \
                --all-databases > /backup/backup-$(date +%Y%m%d).sql
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

**3. Copy to External Storage:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-to-s3
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: amazon/aws-cli
            command:
            - /bin/sh
            - -c
            - |
              aws s3 sync /data s3://my-backup-bucket/$(date +%Y%m%d)/
            volumeMounts:
            - name: data
              mountPath: /data
              readOnly: true
            env:
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: access-key
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: secret-key
          volumes:
          - name: data
            persistentVolumeClaim:
              claimName: app-data-pvc
          restartPolicy: OnFailure
```

**Restore from Snapshot:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc-restored
spec:
  dataSource:
    name: mysql-backup-20240215
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast
```

### 4.4 Storage Best Practices

**1. Always Use StorageClasses:**

```yaml
# ✅ Good - dynamic provisioning
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  storageClassName: fast
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi

# ❌ Bad - static provisioning (manual)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-001
spec:
  capacity:
    storage: 100Gi
```

**2. Use WaitForFirstConsumer:**

```yaml
# ✅ Good - volume in same zone as pod
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer

# ❌ Risky - may provision in wrong zone
volumeBindingMode: Immediate
```

**3. Enable Volume Expansion:**

```yaml
# ✅ Good - can grow volumes
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true

# ❌ Bad - volumes can't grow
allowVolumeExpansion: false
```

**4. Set Appropriate Reclaim Policy:**

```yaml
# Production - Retain
persistentVolumeReclaimPolicy: Retain

# Development - Delete
persistentVolumeReclaimPolicy: Delete
```

**5. Use Appropriate Access Modes:**

```yaml
# Database - RWO
accessModes:
  - ReadWriteOnce

# Shared files - RWX (requires NFS/CephFS)
accessModes:
  - ReadWriteMany

# Static content - ROX
accessModes:
  - ReadOnlyMany
```

**6. Monitor Storage Usage:**

```bash
# Check PVC usage
kubectl get pvc

# Check node storage
kubectl top nodes

# Describe PVC for details
kubectl describe pvc mysql-pvc
```

**7. Implement Backups:**

```yaml
# Regular snapshots
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: daily-backup
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: important-data
```

**8. Set Resource Quotas:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
spec:
  hard:
    requests.storage: "1Ti"
    persistentvolumeclaims: "20"
```

---

## 5. Troubleshooting

### 5.1 PVC Stuck in Pending

**Symptoms:**
```bash
kubectl get pvc
# NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
# mysql-pvc   Pending                                      fast
```

**Common Causes:**

**1. No matching PV (static provisioning):**

```bash
# Check for available PVs
kubectl get pv

# Solution: Create matching PV
```

**2. No StorageClass:**

```bash
# Check if StorageClass exists
kubectl get storageclass fast

# Solution: Create StorageClass or use existing one
```

**3. Insufficient resources:**

```bash
# Describe PVC for events
kubectl describe pvc mysql-pvc

# Look for errors like:
# - No nodes available
# - Insufficient capacity
```

**4. Provisioner issues:**

```bash
# Check provisioner pods
kubectl get pods -n kube-system | grep provisioner

# Check provisioner logs
kubectl logs -n kube-system <provisioner-pod>
```

**5. Zone constraints:**

```bash
# Pod scheduled to zone without storage
# Solution: Use WaitForFirstConsumer

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

### 5.2 Pod Stuck in ContainerCreating

**Symptoms:**
```bash
kubectl get pods
# NAME    READY   STATUS              RESTARTS   AGE
# mysql   0/1     ContainerCreating   0          5m
```

**Common Causes:**

**1. Volume mount failure:**

```bash
# Describe pod for events
kubectl describe pod mysql

# Look for errors like:
# - Unable to attach volume
# - Failed to mount volume
# - Volume already attached to another node
```

**2. Node affinity mismatch:**

```bash
# PV has node affinity, pod scheduled elsewhere
# Check PV node affinity
kubectl get pv <pv-name> -o yaml | grep nodeAffinity
```

**3. Volume already attached:**

```bash
# EBS volumes can only attach to one node
# Force detach (dangerous):
# aws ec2 detach-volume --volume-id vol-xxx --force
```

**4. CSI driver issues:**

```bash
# Check CSI driver pods
kubectl get pods -n kube-system | grep csi

# Check CSI driver logs
kubectl logs -n kube-system <csi-pod>
```

### 5.3 Volume Mount Errors

**Permission Denied:**

```bash
# Check fsGroup and runAsUser
spec:
  securityContext:
    fsGroup: 1000
    runAsUser: 1000
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /data
```

**Read-only Filesystem:**

```bash
# Check if mounted as read-only
kubectl exec -it <pod> -- mount | grep <mountpath>

# Fix: Remove readOnly flag
volumeMounts:
- name: data
  mountPath: /data
  readOnly: false  # Remove or set to false
```

**Volume Not Found:**

```bash
# Check PVC exists and is bound
kubectl get pvc <pvc-name>

# Check PVC referenced correctly in pod
kubectl get pod <pod-name> -o yaml | grep persistentVolumeClaim
```

### 5.4 Storage Performance Issues

**Slow I/O:**

**1. Check volume type:**

```bash
# Use faster storage (gp3 vs gp2, SSD vs HDD)
parameters:
  type: gp3
  iops: "10000"
  throughput: "500"
```

**2. Check mount options:**

```bash
# Add performance mount options
mountOptions:
  - noatime
  - nodiratime
  - discard
```

**3. Monitor IOPS:**

```bash
# AWS CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions Name=VolumeId,Value=vol-xxx

# Inside pod
iostat -x 1
```

**High Latency:**

**1. Check node-volume proximity:**

```bash
# Ensure volume in same zone as pod
# Use WaitForFirstConsumer binding mode
```

**2. Check network:**

```bash
# For NFS/network storage
# Test network latency
kubectl exec -it <pod> -- ping nfs-server.example.com
```

---

## 6. Quick Reference

### 6.1 Essential Commands

**PersistentVolumes:**

```bash
# List PVs
kubectl get pv

# Describe PV
kubectl describe pv <pv-name>

# View PV details
kubectl get pv <pv-name> -o yaml

# Delete PV
kubectl delete pv <pv-name>

# Watch PV status
kubectl get pv -w
```

**PersistentVolumeClaims:**

```bash
# Create PVC
kubectl apply -f pvc.yaml

# List PVCs
kubectl get pvc

# List PVCs in namespace
kubectl get pvc -n <namespace>

# Describe PVC
kubectl describe pvc <pvc-name>

# Delete PVC
kubectl delete pvc <pvc-name>

# Check PVC capacity
kubectl get pvc <pvc-name> -o jsonpath='{.status.capacity.storage}'
```

**StorageClasses:**

```bash
# List StorageClasses
kubectl get storageclasses
kubectl get sc

# Describe StorageClass
kubectl describe sc <sc-name>

# View StorageClass YAML
kubectl get sc <sc-name> -o yaml

# Set default StorageClass
kubectl annotate storageclass <sc-name> \
  storageclass.kubernetes.io/is-default-class=true

# Remove default annotation
kubectl annotate storageclass <sc-name> \
  storageclass.kubernetes.io/is-default-class-
```

**Volume Snapshots:**

```bash
# List VolumeSnapshots
kubectl get volumesnapshots

# Describe snapshot
kubectl describe volumesnapshot <snapshot-name>

# Create snapshot
kubectl apply -f snapshot.yaml

# Delete snapshot
kubectl delete volumesnapshot <snapshot-name>
```

**Troubleshooting:**

```bash
# Check events
kubectl get events --sort-by=.metadata.creationTimestamp

# Describe pod (volume issues)
kubectl describe pod <pod-name>

# Check PVC binding
kubectl get pvc <pvc-name> -o jsonpath='{.status.phase}'

# Check volume mounts in pod
kubectl exec -it <pod-name> -- df -h

# Check CSI driver
kubectl get pods -n kube-system | grep csi

# View CSI driver logs
kubectl logs -n kube-system <csi-pod>
```

### 6.2 Common Scenarios

**Scenario 1: Create PVC with default StorageClass**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF
```

**Scenario 2: Create PVC with specific StorageClass**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 100Gi
EOF
```

**Scenario 3: Expand existing PVC**

```bash
# Patch PVC to increase size
kubectl patch pvc mysql-pvc \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Restart pod to complete expansion (if needed)
kubectl delete pod mysql-pod
```

**Scenario 4: Create snapshot and restore**

```bash
# Create snapshot
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: backup-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: data-pvc
EOF

# Restore from snapshot
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  dataSource:
    name: backup-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast
EOF
```

---

## 7. Production Checklist

### Storage Infrastructure

- [ ] **StorageClasses Configured**
  - [ ] Default StorageClass set
  - [ ] Multiple StorageClasses for different tiers (fast, standard, cheap)
  - [ ] WaitForFirstConsumer binding mode enabled
  - [ ] Volume expansion enabled
  - [ ] Appropriate reclaim policies set

- [ ] **CSI Drivers Installed**
  - [ ] Cloud provider CSI driver deployed
  - [ ] CSI driver version compatible with Kubernetes version
  - [ ] CSI snapshotter deployed (for snapshots)
  - [ ] CSI driver monitoring enabled

### Persistent Volumes

- [ ] **PV Configuration**
  - [ ] Appropriate access modes set
  - [ ] Correct reclaim policy (Retain for production)
  - [ ] Node affinity configured (for local volumes)
  - [ ] Labels applied for organization

- [ ] **PVC Best Practices**
  - [ ] Resource requests appropriate for workload
  - [ ] StorageClass specified explicitly
  - [ ] Access modes match volume capabilities
  - [ ] Selectors used for specific PV requirements

### Backup and Recovery

- [ ] **Backup Strategy**
  - [ ] VolumeSnapshotClass configured
  - [ ] Regular snapshots scheduled (CronJob)
  - [ ] Snapshot retention policy defined
  - [ ] Restore procedure documented and tested
  - [ ] Off-cluster backups for critical data

- [ ] **Disaster Recovery**
  - [ ] Multi-zone replication configured (if supported)
  - [ ] Backup verification automated
  - [ ] Recovery Time Objective (RTO) defined
  - [ ] Recovery Point Objective (RPO) defined

### Monitoring and Alerting

- [ ] **Storage Monitoring**
  - [ ] Volume usage monitored
  - [ ] IOPS and throughput monitored
  - [ ] PVC pending alerts configured
  - [ ] Volume attachment failures alerted
  - [ ] Storage quota usage tracked

### Security

- [ ] **Encryption**
  - [ ] Encryption at rest enabled
  - [ ] KMS keys rotated regularly
  - [ ] Encryption in transit enabled (if supported)

- [ ] **Access Control**
  - [ ] RBAC policies restrict PVC creation
  - [ ] StorageClass permissions controlled
  - [ ] Volume snapshot access restricted
  - [ ] Resource quotas enforced per namespace

### Cost Optimization

- [ ] **Cost Management**
  - [ ] Unused PVs identified and deleted
  - [ ] Volume types optimized (SSD vs HDD)
  - [ ] Storage quotas prevent overspending
  - [ ] Old snapshots cleaned up automatically

---

## 8. Further Reading

### Official Documentation

- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [Volume Plugins](https://kubernetes.io/docs/concepts/storage/volumes/)

### CSI Drivers

- [AWS EBS CSI Driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)
- [GCP PD CSI Driver](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver)
- [Azure Disk CSI Driver](https://github.com/kubernetes-sigs/azuredisk-csi-driver)
- [NFS CSI Driver](https://github.com/kubernetes-csi/csi-driver-nfs)
- [CSI Driver List](https://kubernetes-csi.github.io/docs/drivers.html)

### Best Practices

- [Storage Best Practices](https://kubernetes.io/docs/concepts/storage/storage-classes/#best-practices)
- [StatefulSet Best Practices](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)
- [Volume Security](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

### Tools

- [Velero](https://velero.io/) - Backup and disaster recovery
- [Stash](https://stash.run/) - Backup and recovery
- [Rook](https://rook.io/) - Cloud-native storage orchestration
- [OpenEBS](https://openebs.io/) - Container-attached storage

---

**Document Version:** 1.0  
**Last Updated:** February 15, 2026  
**Author:** Sajal Jana

**License:** This document is provided as-is for educational purposes.
