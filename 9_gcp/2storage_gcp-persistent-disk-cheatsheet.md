# 💾 GCP Persistent Disk & Hyperdisk — Comprehensive Cheatsheet

> **Audience:** Developers & Cloud Engineers | **Last updated:** 2025  
> A single-reference document covering theory, configuration, CLI, GKE integration, and best practices.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Disk Types Comparison](#2-disk-types-comparison)
3. [Disk Lifecycle & Attachment](#3-disk-lifecycle--attachment)
4. [Snapshots & Backups](#4-snapshots--backups)
5. [Images](#5-images)
6. [Access Modes & Sharing](#6-access-modes--sharing)
7. [GKE Integration (Persistent Volumes)](#7-gke-integration-persistent-volumes)
8. [Performance Tuning](#8-performance-tuning)
9. [Encryption](#9-encryption)
10. [gcloud CLI Quick Reference](#10-gcloud-cli-quick-reference)
11. [Pricing Model Summary](#11-pricing-model-summary)
12. [Common Errors & Troubleshooting](#12-common-errors--troubleshooting)

---

## 1. Overview & Key Concepts

**Google Cloud Persistent Disk (PD)** is a durable, high-performance **block storage** service designed for use with Google Compute Engine (GCE) VMs and Google Kubernetes Engine (GKE) nodes. **Hyperdisk** is its next-generation evolution, offering independently provisionable IOPS and throughput decoupled from disk size.

### Where Persistent Disk Fits

```
┌─────────────────────────────────────────────────────────┐
│                    GCP Storage Options                  │
├──────────────────┬──────────────────┬───────────────────┤
│  Persistent Disk │   Local SSD      │  Cloud Storage    │
│  (Block)         │  (Ephemeral Blk) │  (Object)         │
├──────────────────┼──────────────────┼───────────────────┤
│ Durable          │ Not durable      │ Durable           │
│ Network-attached │ Physically local │ HTTP API access   │
│ Survives VM stop │ Lost on VM stop  │ No VM required    │
│ Resize online    │ Fixed size       │ Unlimited size    │
│ Snapshots ✓      │ Snapshots ✗      │ Versioning ✓      │
│ ~0.1–1ms lat.    │ ~10–100μs lat.   │ ms–s latency      │
└──────────────────┴──────────────────┴───────────────────┘
```

### Core Terminology

| Term | Description |
|---|---|
| **Volume / Disk** | A block storage resource attached to a VM as a device (e.g., `/dev/sdb`). |
| **Boot Disk** | The primary disk from which a VM boots. Contains the OS. |
| **Data Disk** | Any additional disk attached for application data storage. |
| **IOPS** | Input/Output Operations Per Second — number of read/write ops per second. |
| **Throughput** | Data transfer rate in MB/s — how much data moves per second. |
| **Provisioned Capacity** | The size (in GB) allocated to a disk, billed regardless of usage. |
| **Snapshot** | Incremental, point-in-time backup of a disk stored in Cloud Storage. |
| **Image** | A complete disk image used to create new boot disks or VMs. |
| **Zonal Disk** | Disk in a single zone. Attaches only to VMs in the same zone. |
| **Regional Disk** | Disk replicated across two zones in the same region for HA failover. |
| **Hyperdisk** | Next-gen PD with independently provisionable IOPS and throughput. |
| **Multi-writer** | Mode allowing a disk to be attached to multiple VMs simultaneously (read/write). |
| **Snapshot Policy** | Automated schedule for creating and retaining snapshots. |
| **Image Family** | A group of related images; the family always resolves to the latest non-deprecated image. |
| **CSI Driver** | Container Storage Interface — the GCE PD CSI driver enables PD use in GKE. |

### Key Design Properties

- **Network-attached:** PD is not physically local — it communicates over Google's internal network. Throughput and IOPS are subject to both disk limits and VM network limits.
- **Independent scaling:** Hyperdisk decouples IOPS/throughput from size; Persistent Disk scales both with size.
- **Durability:** Data is automatically replicated within a zone (zonal) or across two zones (regional).
- **Live resize:** Disk size can be increased without stopping the VM (filesystem resize required separately).
- **No in-place shrink:** Disk size can only be increased, never decreased.

---

## 2. Disk Types Comparison

> **Tip:** Use `pd-balanced` as the default for most workloads. Upgrade to `pd-ssd` / `hyperdisk-balanced` for latency-sensitive apps, and `hyperdisk-extreme` for the highest performance databases.

### Persistent Disk Types

| Property | `pd-standard` | `pd-balanced` | `pd-ssd` | `pd-extreme` |
|---|---|---|---|---|
| **Media** | HDD | SSD | SSD | SSD |
| **Min size** | 10 GB | 10 GB | 10 GB | 500 GB |
| **Max size** | 64 TB | 64 TB | 64 TB | 64 TB |
| **Max IOPS (read)** | 3,000 | 15,000 | 15,000 | 120,000 |
| **Max IOPS (write)** | 3,000 | 15,000 | 15,000 | 120,000 |
| **Max throughput** | 400 MB/s | 1,200 MB/s | 1,200 MB/s | 2,400 MB/s |
| **IOPS provisionable** | ✗ (size-based) | ✗ (size-based) | ✗ (size-based) | ✅ Yes |
| **Throughput provisionable** | ✗ | ✗ | ✗ | ✗ |
| **Use case** | Cold storage, backups, batch | General purpose, dev/test | High-performance DBs | Highest-perf Oracle, SAP HANA |
| **Regional support** | ✅ | ✅ | ✅ | ✗ |
| **Multi-writer support** | ✗ | ✗ | ✅ | ✗ |

### Hyperdisk Types

| Property | `hyperdisk-balanced` | `hyperdisk-throughput` | `hyperdisk-extreme` |
|---|---|---|---|
| **Media** | SSD | SSD | SSD |
| **Min size** | 10 GB | 2 TB | 64 GB |
| **Max size** | 64 TB | 32 TB | 64 TB |
| **Max IOPS** | 160,000 | 1,200 | 350,000 |
| **Max throughput** | 2,400 MB/s | 1,200 MB/s | 5,000 MB/s |
| **IOPS provisionable** | ✅ Yes | ✗ | ✅ Yes |
| **Throughput provisionable** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Use case** | General HA, GKE, mid-tier DBs | Data warehouses, Kafka, analytics | Highest-perf OLTP, in-memory DBs |
| **Regional support** | ✅ | ✗ | ✗ |
| **Multi-writer support** | ✅ | ✗ | ✗ |

### IOPS Scaling (Persistent Disk — non-extreme)

For non-extreme PD types, IOPS and throughput scale linearly with disk size up to the type's cap:

| Disk Type | Read IOPS / GB | Write IOPS / GB | Read MB/s / GB | Write MB/s / GB |
|---|---|---|---|---|
| `pd-standard` | 0.75 | 1.5 | 0.12 | 0.12 |
| `pd-balanced` | 6 | 6 | 0.28 | 0.28 |
| `pd-ssd` | 30 | 30 | 0.48 | 0.48 |

> **Example:** A 200 GB `pd-ssd` provides 200 × 30 = **6,000 IOPS** read/write and 200 × 0.48 = **96 MB/s** throughput.

---

## 3. Disk Lifecycle & Attachment

### Creating Disks

```bash
# Create a zonal data disk (pd-ssd, 100 GB)
gcloud compute disks create my-data-disk \
  --type=pd-ssd \
  --size=100GB \
  --zone=us-central1-a

# Create a regional disk (replicated across two zones)
gcloud compute disks create my-regional-disk \
  --type=pd-balanced \
  --size=200GB \
  --region=us-central1 \
  --replica-zones=us-central1-a,us-central1-b

# Create a Hyperdisk with custom IOPS and throughput
gcloud compute disks create my-hyperdisk \
  --type=hyperdisk-balanced \
  --size=500GB \
  --provisioned-iops=10000 \
  --provisioned-throughput=500 \
  --zone=us-central1-a

# Create a disk from a snapshot
gcloud compute disks create my-restored-disk \
  --source-snapshot=my-snapshot \
  --zone=us-central1-a

# Create a boot disk from a public image
gcloud compute disks create my-boot-disk \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --zone=us-central1-a
```

### Attaching Disks to a VM

```bash
# Attach a data disk to a running VM (read-write)
gcloud compute instances attach-disk my-vm \
  --disk=my-data-disk \
  --zone=us-central1-a

# Attach in read-only mode (for multi-VM read access)
gcloud compute instances attach-disk my-vm \
  --disk=my-data-disk \
  --mode=ro \
  --zone=us-central1-a

# Attach at VM creation time
gcloud compute instances create my-vm \
  --zone=us-central1-a \
  --disk=name=my-data-disk,boot=no,auto-delete=no
```

### Formatting and Mounting (Inside VM)

```bash
# List attached disks
lsblk

# Format the new disk (ext4)
sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb

# Create mount point and mount
sudo mkdir -p /mnt/data
sudo mount -o discard,defaults /dev/sdb /mnt/data

# Persist across reboots (add to /etc/fstab)
echo UUID=$(sudo blkid -s UUID -o value /dev/sdb) /mnt/data ext4 discard,defaults 0 2 | sudo tee -a /etc/fstab
```

### Detaching Disks

```bash
# Detach a disk from a running VM
gcloud compute instances detach-disk my-vm \
  --disk=my-data-disk \
  --zone=us-central1-a
```

> ⚠️ **Warning:** Always unmount the filesystem inside the VM **before** detaching the disk to prevent data corruption:  
> `sudo umount /mnt/data`

### Resizing Disks

```bash
# Resize disk (online — VM does not need to be stopped)
gcloud compute disks resize my-data-disk \
  --size=200GB \
  --zone=us-central1-a
```

After resizing the disk, **resize the filesystem inside the VM**:

```bash
# For ext4
sudo resize2fs /dev/sdb

# For xfs (must be mounted)
sudo xfs_growfs /mnt/data

# For a partition-based layout — grow partition first
sudo growpart /dev/sdb 1
sudo resize2fs /dev/sdb1
```

> ⚠️ **Warning:** Disk size can only **increase** — there is no in-place shrink. Plan capacity carefully.

### Deleting Disks

```bash
# Delete a disk (must be detached first)
gcloud compute disks delete my-data-disk --zone=us-central1-a

# Prevent auto-delete of boot disk when VM is deleted
gcloud compute instances set-disk-auto-delete my-vm \
  --no-auto-delete \
  --disk=my-vm \
  --zone=us-central1-a
```

### Zonal vs Regional Disk Behavior

| Behavior | Zonal Disk | Regional Disk |
|---|---|---|
| Replication | Within single zone | Across 2 zones in a region |
| Failover | Manual (snapshot restore) | Force-attach to replica zone |
| Attach restriction | Same zone only | Either replica zone |
| Performance | Full spec | Slightly lower write throughput |
| Use case | Single-VM workloads | HA databases, stateful services |

```bash
# Force-attach a regional disk to the replica zone during failover
gcloud compute instances attach-disk my-standby-vm \
  --disk=my-regional-disk \
  --disk-region=us-central1 \
  --force-attach \
  --zone=us-central1-b
```

---

## 4. Snapshots & Backups

### Snapshot Types

| Type | Description | Storage Location | Cost |
|---|---|---|---|
| **Standard** | Fast restore; stored in Cloud Storage regionally or multi-regionally | Multi-region (default) or region | ~$0.026/GB/mo |
| **Archive** | Lower cost; slower restore (minutes vs seconds); min 7-day retention | Multi-region | ~$0.0130/GB/mo |

### Incremental Behavior

- First snapshot: **full copy** of the disk's used data
- Subsequent snapshots: **only changed blocks** since the last snapshot
- Snapshots form a **chain** — each depends on the previous for restore
- GCS manages chain consolidation transparently; you can safely delete individual snapshots

```
Disk  ──► Snap1 (full) ──► Snap2 (delta) ──► Snap3 (delta)
                ↑ deleting Snap1 is safe — GCS merges data into Snap2
```

### On-Demand Snapshots

```bash
# Create a snapshot of a zonal disk
gcloud compute disks snapshot my-data-disk \
  --snapshot-names=my-snap-$(date +%Y%m%d) \
  --zone=us-central1-a \
  --storage-location=us-central1

# Create an archive snapshot
gcloud compute disks snapshot my-data-disk \
  --snapshot-names=my-archive-snap \
  --zone=us-central1-a \
  --snapshot-type=ARCHIVE

# Create a snapshot of a regional disk
gcloud compute disks snapshot my-regional-disk \
  --snapshot-names=my-regional-snap \
  --region=us-central1 \
  --storage-location=us
```

> **Tip:** You can snapshot a disk while it is attached and running. For database consistency, quiesce writes or use application-level freeze (e.g., `FLUSH TABLES WITH READ LOCK` in MySQL) before snapshotting.

### Scheduled Snapshots (Snapshot Policies)

```bash
# Create a snapshot schedule: daily at 2 AM UTC, keep 7 days
gcloud compute resource-policies create snapshot-schedule daily-backup \
  --region=us-central1 \
  --max-retention-days=7 \
  --start-time=02:00 \
  --daily-schedule

# Attach the policy to a disk
gcloud compute disks add-resource-policies my-data-disk \
  --zone=us-central1-a \
  --resource-policies=daily-backup \
  --region=us-central1

# Remove a snapshot policy from a disk
gcloud compute disks remove-resource-policies my-data-disk \
  --zone=us-central1-a \
  --resource-policies=daily-backup \
  --region=us-central1
```

### Restoring from a Snapshot

```bash
# Create a new disk from a snapshot (same or different zone/region)
gcloud compute disks create my-restored-disk \
  --source-snapshot=my-snap-20250101 \
  --zone=us-east1-b \
  --type=pd-ssd

# Cross-project restore (specify full snapshot URI)
gcloud compute disks create my-restored-disk \
  --source-snapshot=projects/SOURCE_PROJECT/global/snapshots/my-snap \
  --zone=us-central1-a
```

### Cross-Region & Cross-Project Snapshots

```bash
# Store snapshot in a different region from the source disk
gcloud compute disks snapshot my-data-disk \
  --snapshot-names=cross-region-snap \
  --zone=us-central1-a \
  --storage-location=europe-west1  # different region

# Share snapshot with another project via IAM
gcloud compute snapshots add-iam-policy-binding my-snap \
  --member="serviceAccount:sa@other-project.iam.gserviceaccount.com" \
  --role="roles/compute.storageAdmin"
```

### Snapshot Management

```bash
# List all snapshots
gcloud compute snapshots list

# Describe a snapshot (size, creation time, source disk)
gcloud compute snapshots describe my-snap-20250101

# Delete a snapshot
gcloud compute snapshots delete my-snap-20250101
```

---

## 5. Images

### Snapshots vs Images

| Aspect | Snapshot | Image |
|---|---|---|
| **Primary purpose** | Backup & restore of a specific disk | Create new VMs / boot disks |
| **Bootable** | Not directly (must create disk first) | Yes — used directly at VM creation |
| **OS included** | Depends on source disk | Yes (for boot images) |
| **Image family** | ✗ | ✅ Supported |
| **Global scope** | ✗ (project-scoped) | ✅ Can be shared globally |
| **Storage** | Incremental / compressed | Full (compressed) |

### Creating Custom Images

```bash
# Create image from a stopped VM's boot disk
gcloud compute images create my-custom-image \
  --source-disk=my-vm-boot-disk \
  --source-disk-zone=us-central1-a \
  --family=my-app-images \
  --description="App server image v1.2"

# Create image from a snapshot
gcloud compute images create my-image-from-snap \
  --source-snapshot=my-snap-20250101 \
  --family=my-app-images

# Create image from another image (re-package)
gcloud compute images create hardened-debian \
  --source-image=debian-12-bookworm-v20250101 \
  --source-image-project=debian-cloud \
  --family=hardened-images
```

> **Tip:** Stop the VM before creating an image from its boot disk to guarantee filesystem consistency. For live image capture, use a snapshot first, then create the image from the snapshot.

### Image Families

An **image family** always points to the **latest non-deprecated image** in that family. Use families in scripts to avoid hardcoding image names.

```bash
# Create a VM using an image family (resolves to latest)
gcloud compute instances create my-vm \
  --image-family=my-app-images \
  --image-project=my-project \
  --zone=us-central1-a

# Find the latest image in a public family
gcloud compute images describe-from-family debian-12 \
  --project=debian-cloud
```

### Deprecating Images

```bash
# Mark an image as deprecated (users get a warning; it still works)
gcloud compute images deprecate my-custom-image-v1 \
  --state=DEPRECATED \
  --replacement=my-custom-image-v2

# Obsolete — image can no longer be used for new VMs
gcloud compute images deprecate my-custom-image-v1 \
  --state=OBSOLETE
```

### Sharing Images Across Projects

```bash
# Grant another project permission to use your image
gcloud compute images add-iam-policy-binding my-custom-image \
  --member="serviceAccount:sa@consumer-project.iam.gserviceaccount.com" \
  --role="roles/compute.imageUser"

# Grant org-wide access (all authenticated users in the org)
gcloud compute images add-iam-policy-binding my-custom-image \
  --member="domain:example.com" \
  --role="roles/compute.imageUser"
```

### OS Image Types

| Type | Description | Examples |
|---|---|---|
| **Public** | Google-maintained, free to use | `debian-cloud`, `ubuntu-os-cloud`, `cos-cloud` |
| **Premium** | Licensed OS; per-hour license fee added | `windows-cloud`, `rhel-cloud`, `sles-cloud` |
| **Custom** | Your own images built from public/premium base | Stored in your project |
| **Shielded VM** | Hardened images with Secure Boot, vTPM, integrity monitoring | `debian-12-bookworm` with `--shielded-secure-boot` |

```bash
# List all public image families for Debian
gcloud compute images list --project=debian-cloud --no-standard-images

# List your custom images
gcloud compute images list --project=my-project
```

---

## 6. Access Modes & Sharing

### Access Mode Summary

| Mode | Disk Type | VMs | Use Case |
|---|---|---|---|
| **Read-Write Single (RWO)** | All types | 1 VM | Default — standard single-VM usage |
| **Read-Only Multi-attach** | All types | Up to 10 VMs | Shared reference data, config blobs |
| **Read-Write Multi-attach** | `pd-ssd`, `hyperdisk-balanced` only | Up to 8 VMs | Clustered filesystems (GFS2, OCFS2) |
| **Regional PD Failover** | `pd-balanced`, `pd-ssd` (regional) | 1 at a time | HA pairs — active/standby |

### Read-Only Multi-Attach

```bash
# Attach the same disk to multiple VMs in read-only mode
gcloud compute instances attach-disk vm-1 --disk=shared-data --mode=ro --zone=us-central1-a
gcloud compute instances attach-disk vm-2 --disk=shared-data --mode=ro --zone=us-central1-a
gcloud compute instances attach-disk vm-3 --disk=shared-data --mode=ro --zone=us-central1-a
```

**Use cases:** Shared ML model weights, static dataset distribution, read-only config volumes.

### Multi-Writer Mode (Read-Write Multi-Attach)

```bash
# Create a multi-writer disk
gcloud compute disks create my-shared-disk \
  --type=pd-ssd \
  --size=100GB \
  --zone=us-central1-a \
  --multi-writer

# Attach to multiple VMs in read-write mode
gcloud compute instances attach-disk vm-1 \
  --disk=my-shared-disk --zone=us-central1-a
gcloud compute instances attach-disk vm-2 \
  --disk=my-shared-disk --zone=us-central1-a
```

> ⚠️ **Warning:** Multi-writer mode does **not** provide filesystem-level consistency. You must use a **cluster-aware filesystem** (e.g., GFS2, OCFS2) or a distributed locking mechanism. Using standard ext4 or xfs across multiple writers **will corrupt data**.

**Use cases:** Oracle RAC, Hadoop clusters, custom distributed storage, MPI workloads.

### Regional PD for HA Failover

```
Zone A (Primary)                Zone B (Standby)
┌─────────────────┐             ┌─────────────────┐
│   Active VM     │             │   Standby VM    │
│   ┌──────────┐  │  Sync repl  │                 │
│   │ Regional │◄─┼─────────────┼─►               │
│   │   Disk   │  │             │                 │
└───┴──────────┴──┘             └─────────────────┘
                  Failover: force-attach disk to Zone B VM
```

```bash
# Failover: detach from failed VM (or force-attach if VM is unresponsive)
gcloud compute instances attach-disk standby-vm \
  --disk=my-regional-disk \
  --disk-region=us-central1 \
  --force-attach \
  --zone=us-central1-b
```

> **Tip:** Regional PD is the recommended storage backend for stateful HA workloads like PostgreSQL and MySQL when not using Cloud SQL. Combine with a load balancer or keepalived for automatic failover.

---

## 7. GKE Integration (Persistent Volumes)

### GCE PD CSI Driver

GKE clusters version **1.18+** have the **GCE Persistent Disk CSI driver** (`pd.csi.storage.gke.io`) enabled by default. It replaces the legacy in-tree `kubernetes.io/gce-pd` driver.

### StorageClass Definitions

```yaml
# pd-balanced StorageClass (default for most GKE workloads)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: balanced-storage
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-balanced
reclaimPolicy: Retain           # Delete or Retain when PVC is deleted
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer  # Schedules PV to same zone as Pod
---
# pd-ssd StorageClass for latency-sensitive workloads
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ssd-storage
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# Hyperdisk StorageClass with provisioned IOPS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hyperdisk-hi-perf
provisioner: pd.csi.storage.gke.io
parameters:
  type: hyperdisk-balanced
  provisioned-iops-on-create: "10000"
  provisioned-throughput-on-create: "500Mi"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# Regional PD StorageClass for HA workloads
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: regional-balanced
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-balanced
  replication-type: regional-pd
  zones: us-central1-a,us-central1-b
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### PersistentVolumeClaim (PVC)

```yaml
# Standard PVC — dynamically provisions a disk
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce       # RWO — single node read/write
  storageClassName: balanced-storage
  resources:
    requests:
      storage: 100Gi
---
# ReadOnlyMany PVC (attach to multiple pods read-only)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-config
spec:
  accessModes:
    - ReadOnlyMany
  storageClassName: balanced-storage
  resources:
    requests:
      storage: 10Gi
```

### Using PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: my-app:latest
      volumeMounts:
        - mountPath: /data
          name: app-storage
  volumes:
    - name: app-storage
      persistentVolumeClaim:
        claimName: my-app-data
```

### StatefulSet with volumeClaimTemplates

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:                 # Creates one PVC per replica automatically
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: ssd-storage
        resources:
          requests:
            storage: 50Gi
```

### VolumeSnapshot (Disk Backup from GKE)

```yaml
# VolumeSnapshotClass — uses the CSI driver
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: pd-snapshot-class
driver: pd.csi.storage.gke.io
deletionPolicy: Delete
---
# VolumeSnapshot — on-demand backup of a PVC
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-backup-20250101
spec:
  volumeSnapshotClassName: pd-snapshot-class
  source:
    persistentVolumeClaimName: postgres-data-postgres-0
---
# Restore a PVC from a VolumeSnapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-restored
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: ssd-storage
  resources:
    requests:
      storage: 50Gi
  dataSource:
    name: postgres-backup-20250101
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

### Resize a PVC (Online Volume Expansion)

```bash
kubectl patch pvc my-app-data \
  -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# Verify expansion status
kubectl describe pvc my-app-data | grep -A5 Conditions
```

> **Tip:** The StorageClass must have `allowVolumeExpansion: true` for PVC resize to work. The filesystem inside the pod is resized automatically by the CSI driver.

---

## 8. Performance Tuning

### VM-Level Throughput & IOPS Caps

Disk performance is capped at the **lower of disk limits and VM limits**. VM limits scale with vCPU count.

| vCPUs | Max PD Read IOPS | Max PD Write IOPS | Max Throughput (R/W) |
|---|---|---|---|
| 1 | 3,000 | 3,000 | 200 MB/s |
| 4 | 15,000 | 15,000 | 800 MB/s |
| 8 | 25,000 | 25,000 | 1,600 MB/s |
| 16 | 50,000 | 50,000 | 3,200 MB/s |
| 32+ | 100,000+ | 100,000+ | 4,800+ MB/s |

> **Tip:** If you're not hitting expected disk performance, check if the **VM is the bottleneck** — a small VM (2 vCPUs) limits PD performance even if the disk supports higher specs.

### Queue Depth

- PD performs best with **queue depth 32–256** per disk for SSDs
- For HDDs (`pd-standard`), queue depth 4–16 is sufficient
- Hyperdisk extreme: queue depth 256+ recommended for max IOPS

### Benchmarking with `fio`

```bash
# Install fio
sudo apt-get install fio -y

# Sequential write throughput test
sudo fio --name=write-throughput \
  --directory=/mnt/data \
  --numjobs=16 \
  --size=10G \
  --time_based --runtime=60s \
  --ramp_time=2s \
  --ioengine=libaio \
  --direct=1 \
  --verify=0 \
  --bs=1M \
  --iodepth=64 \
  --rw=write \
  --group_reporting

# Random read IOPS test
sudo fio --name=rand-read-iops \
  --directory=/mnt/data \
  --numjobs=8 \
  --size=10G \
  --time_based --runtime=60s \
  --ioengine=libaio \
  --direct=1 \
  --bs=4K \
  --iodepth=256 \
  --rw=randread \
  --group_reporting
```

### Filesystem Selection

| Filesystem | Strengths | Recommended For |
|---|---|---|
| **ext4** | Mature, well-supported, journaling | General workloads, boot disks, databases |
| **xfs** | Better large-file performance, scalable metadata | Big data, analytics, media storage |
| **btrfs** | Copy-on-write, snapshots | Dev environments (not for prod PD) |

```bash
# Optimized ext4 format for SSD (disable unnecessary disk writes)
sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb

# Optimized xfs format
sudo mkfs.xfs -f /dev/sdb
```

### Mount Options for Performance

```bash
# ext4: discard (TRIM) + noatime (skip access time writes)
sudo mount -o discard,defaults,noatime /dev/sdb /mnt/data

# xfs: noatime + largeio + inode64
sudo mount -o noatime,largeio,inode64 /dev/sdb /mnt/data
```

### Recommended Kernel I/O Scheduler

```bash
# Check current scheduler
cat /sys/block/sdb/queue/scheduler

# Set to 'none' (mq-deadline) for SSDs (best for network-attached block)
echo none | sudo tee /sys/block/sdb/queue/scheduler

# Increase queue depth
echo 256 | sudo tee /sys/block/sdb/queue/nr_requests
```

---

## 9. Encryption

### Encryption Options

| Option | Key Management | Key Storage | Use Case |
|---|---|---|---|
| **GMEK** (Google-managed) | Fully automatic | Google-managed | Default — no setup needed |
| **CMEK** | Customer via Cloud KMS | Cloud KMS (Google-hosted) | Compliance, audit trail, key revocation |
| **CSEK** | Customer-supplied per request | Never stored by Google | Maximum control; key required every request |

### GMEK (Default)

All PD data is encrypted at rest with AES-256. No configuration required.

### CMEK with Cloud KMS

```bash
# Step 1: Create a KMS key ring and key
gcloud kms keyrings create my-keyring --location=us-central1
gcloud kms keys create my-disk-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --purpose=encryption

# Step 2: Grant GCE access to use the key
gcloud kms keys add-iam-policy-binding my-disk-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --member="serviceAccount:service-PROJECT_NUMBER@compute-system.iam.gserviceaccount.com" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Step 3: Create a disk with CMEK
gcloud compute disks create my-encrypted-disk \
  --type=pd-ssd \
  --size=100GB \
  --zone=us-central1-a \
  --kms-key=projects/MY_PROJECT/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-disk-key
```

> ⚠️ **Warning:** If the CMEK key is **disabled or destroyed**, the disk becomes **permanently inaccessible**. Snapshots and images derived from CMEK disks also inherit the encryption key requirement.

### CSEK (Customer-Supplied Encryption Key)

CSEK keys are raw AES-256 keys that you generate and supply with each API request. Google never stores them.

```bash
# Generate a 256-bit key
python3 -c "import os,base64; print(base64.b64encode(os.urandom(32)).decode())"
# Example output: 7bxFAWgxvfDrqxG7oTMIYg3nKdq4PpgDmJ8RfKv3ABC=

# Create a disk with CSEK using a key file (RFC 4648 base64)
cat > csek-key.json <<EOF
[
  {
    "uri": "https://www.googleapis.com/compute/v1/projects/MY_PROJECT/zones/us-central1-a/disks/my-csek-disk",
    "key": "7bxFAWgxvfDrqxG7oTMIYg3nKdq4PpgDmJ8RfKv3ABC=",
    "key-type": "raw"
  }
]
EOF

gcloud compute disks create my-csek-disk \
  --type=pd-ssd \
  --size=100GB \
  --zone=us-central1-a \
  --csek-key-file=csek-key.json
```

> ⚠️ **Warning:** If you lose a CSEK key, the data is **permanently unrecoverable**. Always back up CSEK keys securely (e.g., in Secret Manager or an HSM).

### CMEK Key Rotation

```bash
# Create a new key version (automatic rotation can also be enabled)
gcloud kms keys versions create \
  --location=us-central1 \
  --keyring=my-keyring \
  --key=my-disk-key

# Set the new version as primary
gcloud kms keys update my-disk-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --primary-version=2
```

New write operations use the new key version. Existing encrypted blocks use the version they were encrypted with; key rotation does **not** re-encrypt existing disk data.

---

## 10. gcloud CLI Quick Reference

### Disk Operations

```bash
# --- CREATE ---
gcloud compute disks create DISK_NAME \
  --type=pd-ssd --size=100GB --zone=ZONE

# --- LIST ---
gcloud compute disks list
gcloud compute disks list --filter="zone:us-central1-a"

# --- DESCRIBE ---
gcloud compute disks describe DISK_NAME --zone=ZONE

# --- DELETE ---
gcloud compute disks delete DISK_NAME --zone=ZONE

# --- RESIZE ---
gcloud compute disks resize DISK_NAME --size=200GB --zone=ZONE

# --- UPDATE (change type/IOPS for Hyperdisk) ---
gcloud compute disks update DISK_NAME \
  --provisioned-iops=20000 \
  --provisioned-throughput=800 \
  --zone=ZONE
```

### Attach / Detach

```bash
# Attach disk to VM
gcloud compute instances attach-disk VM_NAME \
  --disk=DISK_NAME --zone=ZONE

# Attach read-only
gcloud compute instances attach-disk VM_NAME \
  --disk=DISK_NAME --mode=ro --zone=ZONE

# Detach disk from VM
gcloud compute instances detach-disk VM_NAME \
  --disk=DISK_NAME --zone=ZONE

# Set auto-delete behavior
gcloud compute instances set-disk-auto-delete VM_NAME \
  --auto-delete --disk=DISK_NAME --zone=ZONE
gcloud compute instances set-disk-auto-delete VM_NAME \
  --no-auto-delete --disk=DISK_NAME --zone=ZONE
```

### Snapshots

```bash
# Create snapshot
gcloud compute disks snapshot DISK_NAME \
  --snapshot-names=SNAP_NAME --zone=ZONE

# List snapshots
gcloud compute snapshots list

# Describe snapshot
gcloud compute snapshots describe SNAP_NAME

# Delete snapshot
gcloud compute snapshots delete SNAP_NAME

# Create disk from snapshot
gcloud compute disks create NEW_DISK \
  --source-snapshot=SNAP_NAME --zone=ZONE
```

### Snapshot Policies (Schedules)

```bash
# Create hourly snapshot schedule
gcloud compute resource-policies create snapshot-schedule POLICY_NAME \
  --region=REGION \
  --max-retention-days=3 \
  --start-time=00:00 \
  --hourly-schedule=6    # every 6 hours

# Create weekly schedule (Sundays at 3 AM)
gcloud compute resource-policies create snapshot-schedule POLICY_NAME \
  --region=REGION \
  --max-retention-days=30 \
  --start-time=03:00 \
  --weekly-schedule=sunday

# Attach schedule to disk
gcloud compute disks add-resource-policies DISK_NAME \
  --zone=ZONE \
  --resource-policies=POLICY_NAME \
  --region=REGION

# List policies
gcloud compute resource-policies list --region=REGION

# Delete policy
gcloud compute resource-policies delete POLICY_NAME --region=REGION
```

### Images

```bash
# Create image from disk
gcloud compute images create IMAGE_NAME \
  --source-disk=DISK_NAME \
  --source-disk-zone=ZONE \
  --family=IMAGE_FAMILY

# Create image from snapshot
gcloud compute images create IMAGE_NAME \
  --source-snapshot=SNAP_NAME \
  --family=IMAGE_FAMILY

# List images
gcloud compute images list --project=PROJECT

# Describe image
gcloud compute images describe IMAGE_NAME --project=PROJECT

# Delete image
gcloud compute images delete IMAGE_NAME

# Deprecate image
gcloud compute images deprecate IMAGE_NAME \
  --state=DEPRECATED --replacement=NEW_IMAGE_NAME
```

---

## 11. Pricing Model Summary

> Prices are approximate US region rates as of 2025. Pricing varies by region. Always verify at [cloud.google.com/compute/disks-image-pricing](https://cloud.google.com/compute/disks-image-pricing).

### Persistent Disk Storage (per GB / month)

| Disk Type | US Regions | Europe | Asia-Pacific |
|---|---|---|---|
| `pd-standard` (HDD) | $0.040 | $0.043 | $0.048 |
| `pd-balanced` | $0.100 | $0.110 | $0.120 |
| `pd-ssd` | $0.170 | $0.187 | $0.204 |
| `pd-extreme` | $0.125 (+ IOPS fee) | $0.138 | $0.150 |

### Hyperdisk Storage (per GB / month)

| Disk Type | Base GB Price | Provisioned IOPS | Provisioned Throughput |
|---|---|---|---|
| `hyperdisk-balanced` | $0.100 | $0.004 / IOPS-mo | $0.048 / MB/s-mo |
| `hyperdisk-throughput` | $0.040 | N/A | $0.040 / MB/s-mo |
| `hyperdisk-extreme` | $0.125 | $0.004 / IOPS-mo | $0.048 / MB/s-mo |

### Snapshot & Image Storage

| Resource | Price |
|---|---|
| Standard Snapshot | $0.026 / GB / month (compressed, incremental) |
| Archive Snapshot | $0.013 / GB / month |
| Custom Image | $0.085 / GB / month |

### Cost Optimization Strategies

| Strategy | Savings |
|---|---|
| Right-size disks | Avoid over-provisioning; PD charges for allocated GB, not used GB |
| Use `pd-standard` for cold data | 4× cheaper than `pd-ssd` for infrequently accessed volumes |
| Use Archive snapshots for long-term | ~50% cheaper than standard snapshots for 7+ day retention |
| Set snapshot retention policies | Auto-delete old snapshots; orphaned snapshots accumulate cost |
| Delete unattached disks | Detached disks are still billed; clean up unused volumes |
| Use Hyperdisk for right-sizing perf | Pay only for IOPS you need — avoid over-sizing disk for IOPS |
| Committed Use Discounts (CUDs) | Up to 57% discount for 1- or 3-year commitments on PD |

---

## 12. Common Errors & Troubleshooting

| Error / Issue | Cause | Fix |
|---|---|---|
| `RESOURCE_EXHAUSTED: disk quota exceeded` | Project disk quota (GB or count) exceeded in that region/zone | Request quota increase in IAM & Admin > Quotas; delete unused disks |
| `RESOURCE_IN_USE_BY_ANOTHER_RESOURCE` | Trying to delete a disk that's still attached to a VM | Detach disk first: `gcloud compute instances detach-disk` |
| Filesystem not resized after disk resize | `gcloud compute disks resize` only resizes the block device, not the filesystem | Run `sudo resize2fs /dev/sdb` (ext4) or `sudo xfs_growfs /mnt` (xfs) |
| `IOPS_EXCEEDED` / IO latency spikes | Disk is throttled — requests exceed provisioned or size-based IOPS | Upgrade disk type, increase size, or provision more IOPS (Hyperdisk) |
| VM-level throughput cap hit | VM machine type limits total PD throughput regardless of disk spec | Upgrade to a VM with more vCPUs; check VM I/O limits |
| `403 Permission denied` on snapshot | Service account lacks `roles/compute.storageAdmin` or snapshot is in another project | Grant correct IAM role; check cross-project snapshot IAM |
| Regional disk failover — attach fails | Primary VM still holds the disk lease (not cleanly stopped) | Use `--force-attach` flag to break the lease |
| Snapshot creation fails on busy disk | OS buffers not flushed; application writes during snapshot | Use app-level freeze (e.g., DB flush/lock) or take snapshot of stopped VM |
| `disk size must be a multiple of 1 GB` | Invalid size value passed to resize or create | Use integer GB values (e.g., `--size=100GB`) |
| CSEK disk inaccessible | CSEK key file not provided or key was lost | Ensure key file is available for every operation; key loss = permanent data loss |
| CMEK disk inaccessible | KMS key disabled, destroyed, or IAM binding removed | Restore key version or re-grant `cloudkms.cryptoKeyEncrypterDecrypter` role |
| PVC stuck in `Pending` (GKE) | No matching StorageClass, zone mismatch, or quota exceeded | Check `kubectl describe pvc`; verify StorageClass exists and disk quota |
| PVC stuck in `Terminating` (GKE) | Finalizer not released; PV still bound | Check for PV finalizers: `kubectl patch pv PV_NAME -p '{"metadata":{"finalizers":null}}'` |
| Mount fails: `wrong fs type` | Disk was not formatted before mounting | Format with `mkfs.ext4` or `mkfs.xfs` before mounting |
| `ERROR: (gcloud) argument --size: Bad value` | Size specified without unit or with wrong suffix | Use format like `100GB` or `1TB`, not `100` or `100G` |

### Debugging Commands

```bash
# Check disk attachment status and device name
gcloud compute instances describe my-vm \
  --zone=us-central1-a \
  --format="json(disks)"

# Check disk utilization from within VM
iostat -xz 1   # real-time IOPS and throughput
df -h          # disk space
lsblk          # block device tree

# Check for IO errors in kernel log
sudo dmesg | grep -i "i/o error\|sd[a-z]"

# Verify filesystem integrity (unmounted disk only)
sudo fsck -n /dev/sdb

# Check GKE CSI driver status
kubectl get pods -n kube-system | grep csi
kubectl logs -n kube-system -l app=gcp-compute-persistent-disk-csi-driver
```

---

## Quick Reference Card

```
Block storage types:    pd-standard | pd-balanced | pd-ssd | pd-extreme | hyperdisk-*
Max disk size:          64 TB (all types)
Max object per attach:  1 VM (RWO) | 10 VMs (ROX) | 8 VMs (multi-writer pd-ssd)
Snapshot type:          Incremental (changed blocks only)
Snapshot frequency:     On-demand or scheduled (min 1 hour)
Online resize:          Yes (disk) — filesystem resize required separately
Shrink disk:            NOT supported — increase only
CMEK key loss:          Data permanently inaccessible
Regional PD zones:      Must be in same region, 2 zones
GKE provisioner:        pd.csi.storage.gke.io
CUD discount:           Up to 57% for 1 or 3 year commitment
```

---

*Reference: [Persistent Disk Docs](https://cloud.google.com/compute/docs/disks) | [Hyperdisk Docs](https://cloud.google.com/compute/docs/disks/hyperdisks) | [GKE Storage](https://cloud.google.com/kubernetes-engine/docs/concepts/persistent-volumes) | [Pricing](https://cloud.google.com/compute/disks-image-pricing)*
