# 🗂️ GCP Filestore — Comprehensive Cheatsheet

> **Audience:** Developers & Cloud Engineers | **Last updated:** 2025  
> A single-reference document covering theory, configuration, CLI, GKE integration, and best practices.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Service Tiers Comparison](#2-service-tiers-comparison)
3. [Instance Lifecycle & Configuration](#3-instance-lifecycle--configuration)
4. [Networking & Access Control](#4-networking--access-control)
5. [Multi-Share Support](#5-multi-share-support)
6. [Snapshots & Backups](#6-snapshots--backups)
7. [GKE Integration](#7-gke-integration)
8. [Performance Tuning](#8-performance-tuning)
9. [High Availability & Replication](#9-high-availability--replication)
10. [gcloud CLI Quick Reference](#10-gcloud-cli-quick-reference)
11. [Pricing Model Summary](#11-pricing-model-summary)
12. [Common Errors & Troubleshooting](#12-common-errors--troubleshooting)

---

## 1. Overview & Key Concepts

**Google Cloud Filestore** is a fully managed, high-performance **Network File System (NFS)** service on GCP. It provides shared file storage accessible simultaneously by multiple compute instances, GKE pods, or on-premises systems — without managing NFS server infrastructure.

### How Filestore Compares to Other GCP Storage

```
┌──────────────────────────────────────────────────────────────────┐
│                     GCP Storage Landscape                        │
├─────────────────┬──────────────────┬──────────────────┬─────────┤
│   Filestore     │  Persistent Disk │  Cloud Storage   │  BMS    │
│  (File / NFS)   │  (Block)         │  (Object)        │  (NFS+) │
├─────────────────┼──────────────────┼──────────────────┼─────────┤
│ NFS v3 protocol │ iSCSI/virtio-scsi│ HTTP REST API    │ NFS/SMB │
│ Shared access   │ Single VM (RWO)  │ Any client       │ Bare Met │
│ POSIX-compliant │ POSIX-compliant  │ Not POSIX        │ POSIX   │
│ Managed server  │ No server        │ No server        │ Managed │
│ ms latency      │ sub-ms latency   │ ms–s latency     │ μs lat. │
│ Multi-VM mount  │ 1 VM (or ROX)    │ No mount (FUSE)  │ Limited │
└─────────────────┴──────────────────┴──────────────────┴─────────┘
```

### When to Use Filestore

- Applications requiring **shared filesystem access** from multiple VMs or pods simultaneously
- Workloads needing **POSIX compliance** (permissions, symlinks, file locking)
- **Lift-and-shift** of on-premises NAS workloads to GCP
- **GKE workloads** requiring `ReadWriteMany` (RWX) persistent volumes
- **Media rendering**, HPC, ML training pipelines, CMS, ERP systems

### Core Terminology

| Term | Description |
|---|---|
| **Instance** | A managed Filestore NFS server. The top-level resource you create and pay for. |
| **Share** | An NFS export on an instance. Basic tiers have one share; Zonal/Regional/Enterprise support multiple. |
| **Mount Point** | The local directory on a client VM where the NFS share is attached (e.g., `/mnt/filestore`). |
| **NFS Protocol** | Network File System — Filestore uses **NFSv3** by default; Enterprise also supports NFSv4.1. |
| **Capacity** | Provisioned storage size in GB. You pay for allocated capacity, not actual usage. |
| **IOPS** | Read/write operations per second. Scales with capacity for most tiers. |
| **Throughput** | Data transfer rate (MB/s). Also scales with capacity. |
| **Tier** | Service level determining performance, HA, and feature availability. |
| **Zonal Instance** | Single-zone instance; lower cost, no automatic failover. |
| **Regional Instance** | Synchronously replicated across two zones; built-in HA. |
| **Authorized Network** | VPC network permitted to access the Filestore instance. |
| **Connect Mode** | `DIRECT_PEERING` (legacy) or `PRIVATE_SERVICE_ACCESS` (recommended). |
| **Squash** | NFS root squash setting — maps root user on client to anonymous UID on server. |
| **Snapshot** | Point-in-time copy of a share. Zonal/Regional/Enterprise only. |
| **Backup** | Full or incremental copy stored independently from the instance. All tiers. |

---

## 2. Service Tiers Comparison

> **Tip:** Use **Zonal** for most new general-purpose workloads (replaces Basic SSD). Use **Regional** for production HA. Reserve **Enterprise** for legacy NFSv4.1 or AD/LDAP requirements.

### Full Tier Comparison

| Property | Basic HDD | Basic SSD | Zonal | Regional | Enterprise |
|---|---|---|---|---|---|
| **Protocol** | NFSv3 | NFSv3 | NFSv3 | NFSv3 | NFSv3 + NFSv4.1 |
| **Capacity Range** | 1 TB – 63.9 TB | 2.5 TB – 63.9 TB | 1 TB – 100 TB | 1 TB – 100 TB | 1 TB – 10 TB |
| **Max IOPS (read)** | 600 | 60,000 | 160,000 | 160,000 | 120,000 |
| **Max IOPS (write)** | 1,000 | 25,000 | 160,000 | 160,000 | 120,000 |
| **Max Throughput** | 180 MB/s | 1,200 MB/s | 4,800 MB/s | 4,800 MB/s | 2,400 MB/s |
| **Availability SLA** | None | None | 99.9% | 99.99% | 99.99% |
| **Zone Replication** | ✗ | ✗ | ✗ (single zone) | ✅ (2 zones) | ✅ (internal) |
| **Multi-share** | ✗ | ✗ | ✅ (up to 10) | ✅ (up to 10) | ✅ (up to 10) |
| **Snapshots** | ✗ | ✗ | ✅ | ✅ | ✅ |
| **Backups** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **NFSv4.1 / AD/LDAP** | ✗ | ✗ | ✗ | ✗ | ✅ |
| **Relative Cost** | $ | $$ | $$$ | $$$$ | $$$$$ |
| **Best For** | Cold archives, batch | High-perf single zone | General HA workloads | Mission-critical HA | Enterprise apps (SAP, AD) |

### Throughput Scaling by Tier (approximate)

| Tier | Read MB/s per TB | Write MB/s per TB | Notes |
|---|---|---|---|
| Basic HDD | ~8 MB/s | ~8 MB/s | Linear scaling |
| Basic SSD | ~100 MB/s | ~100 MB/s | Linear scaling |
| Zonal | ~100 MB/s | ~100 MB/s | Scales to 4,800 MB/s cap |
| Regional | ~100 MB/s | ~100 MB/s | Scales to 4,800 MB/s cap |
| Enterprise | ~100 MB/s | ~100 MB/s | Max 2,400 MB/s |

---

## 3. Instance Lifecycle & Configuration

### Creating a Filestore Instance

```bash
# Basic HDD instance — 2 TB, default VPC
gcloud filestore instances create my-filestore-hdd \
  --tier=BASIC_HDD \
  --file-share=name=vol1,capacity=2TB \
  --network=name=default \
  --zone=us-central1-a

# Basic SSD instance — 5 TB, custom VPC, Private Services Access
gcloud filestore instances create my-filestore-ssd \
  --tier=BASIC_SSD \
  --file-share=name=vol1,capacity=5TB \
  --network=name=my-vpc,connect-mode=PRIVATE_SERVICE_ACCESS \
  --zone=us-central1-a

# Zonal instance — 10 TB with reserved IP range
gcloud filestore instances create my-zonal-fs \
  --tier=ZONAL \
  --file-share=name=data,capacity=10TB \
  --network=name=my-vpc,connect-mode=PRIVATE_SERVICE_ACCESS,reserved-ip-range=10.100.0.0/29 \
  --zone=us-central1-a \
  --description="Shared data volume for ML pipeline"

# Regional instance — 10 TB, HA across two zones
gcloud filestore instances create my-regional-fs \
  --tier=REGIONAL \
  --file-share=name=data,capacity=10TB \
  --network=name=my-vpc,connect-mode=PRIVATE_SERVICE_ACCESS \
  --region=us-central1

# Enterprise instance with NFSv4.1
gcloud filestore instances create my-enterprise-fs \
  --tier=ENTERPRISE \
  --file-share=name=share1,capacity=5TB \
  --network=name=my-vpc,connect-mode=PRIVATE_SERVICE_ACCESS \
  --region=us-central1
```

### Describing and Listing Instances

```bash
# List all instances in a project
gcloud filestore instances list

# Describe instance (get IP address, status, capacity)
gcloud filestore instances describe my-filestore-ssd --zone=us-central1-a

# Get just the IP address (for NFS mount commands)
gcloud filestore instances describe my-filestore-ssd \
  --zone=us-central1-a \
  --format="value(networks[0].ipAddresses[0])"
```

### Resizing (Scaling Up Capacity)

```bash
# Increase capacity on a Basic SSD instance (size only increases)
gcloud filestore instances update my-filestore-ssd \
  --zone=us-central1-a \
  --file-share=name=vol1,capacity=10TB

# Resize a Zonal instance
gcloud filestore instances update my-zonal-fs \
  --zone=us-central1-a \
  --file-share=name=data,capacity=20TB
```

> ⚠️ **Warning:** Filestore capacity can only be **increased**, never decreased. Plan initial sizing carefully.

### Deleting an Instance

```bash
# Delete a zonal instance
gcloud filestore instances delete my-filestore-ssd --zone=us-central1-a

# Delete a regional instance
gcloud filestore instances delete my-regional-fs --region=us-central1
```

> ⚠️ **Warning:** Deleting an instance **permanently deletes all data** on it, including all shares and snapshots. Take a backup first if the data is needed.

### Mounting from a GCE VM

```bash
# Step 1: Install NFS client
sudo apt-get update && sudo apt-get install nfs-common -y  # Debian/Ubuntu
sudo yum install nfs-utils -y                               # RHEL/CentOS

# Step 2: Create a mount point
sudo mkdir -p /mnt/filestore/data

# Step 3: Mount the NFS share (replace IP with your Filestore IP)
sudo mount -t nfs -o vers=3 10.100.0.2:/data /mnt/filestore/data

# Step 4: Persist across reboots (add to /etc/fstab)
echo "10.100.0.2:/data /mnt/filestore/data nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab

# Verify the mount
df -h | grep filestore
mount | grep filestore
```

### Mounting from GKE (Static)

```bash
# Get Filestore IP for use in YAML
FILESTORE_IP=$(gcloud filestore instances describe my-filestore-ssd \
  --zone=us-central1-a \
  --format="value(networks[0].ipAddresses[0])")
echo $FILESTORE_IP
```

---

## 4. Networking & Access Control

### Connect Modes

| Mode | Description | Recommendation |
|---|---|---|
| `PRIVATE_SERVICE_ACCESS` | Uses Google-managed VPC peering via a reserved IP range | ✅ Recommended for all new instances |
| `DIRECT_PEERING` | Legacy — direct VPC peering, manually managed | Legacy only; use PSA for new deployments |

### Setting Up Private Services Access (PSA)

```bash
# Step 1: Allocate an IP range for PSA in your VPC
gcloud compute addresses create google-managed-services-my-vpc \
  --global \
  --purpose=VPC_PEERING \
  --prefix-length=24 \
  --network=my-vpc

# Step 2: Create the VPC peering connection
gcloud services vpc-peerings connect \
  --service=servicenetworking.googleapis.com \
  --ranges=google-managed-services-my-vpc \
  --network=my-vpc
```

### Firewall Rules for NFS

Filestore uses **TCP port 2049** (NFS) and additional RPC ports. The minimum required rule:

```bash
# Allow NFS traffic from VM subnet to Filestore IP range
gcloud compute firewall-rules create allow-nfs-filestore \
  --network=my-vpc \
  --allow=tcp:2049,udp:2049,tcp:111,udp:111 \
  --source-ranges=10.0.0.0/8 \
  --target-tags=filestore-clients \
  --description="Allow NFS to Filestore instances"
```

> **Tip:** When using `PRIVATE_SERVICE_ACCESS`, the Filestore instance lives in a Google-managed VPC peered to yours. Ensure your VPC's firewall rules permit egress to the Filestore IP range on port 2049.

### Authorized Networks

Filestore access is restricted by the VPC networks specified at instance creation. An instance in VPC `my-vpc` is only reachable from VMs in that VPC (or peered networks).

```bash
# Create instance limited to a specific VPC
gcloud filestore instances create my-fs \
  --tier=ZONAL \
  --file-share=name=data,capacity=5TB \
  --network=name=my-vpc,connect-mode=PRIVATE_SERVICE_ACCESS \
  --zone=us-central1-a
```

### Shared VPC

To use Filestore from a **Shared VPC**:

1. Create the Filestore instance in the **service project** using the **host VPC network**
2. Specify the host project's network with the full resource path

```bash
gcloud filestore instances create my-fs \
  --tier=ZONAL \
  --file-share=name=data,capacity=5TB \
  --network=name=projects/HOST_PROJECT/global/networks/shared-vpc,connect-mode=PRIVATE_SERVICE_ACCESS \
  --zone=us-central1-a
```

### On-Premises Access (Cloud Interconnect / VPN)

```
On-Premises Network
        │
        ▼
  Cloud VPN / Interconnect
        │
        ▼
   GCP VPC (my-vpc)
        │  (VPC peering via PSA)
        ▼
   Filestore Instance
```

Requirements:
- Dedicated Interconnect or Cloud VPN established to GCP VPC
- PSA reserved IP range must be **exported** through the VPN/Interconnect BGP routes
- On-premises clients mount using the Filestore instance's private IP

```bash
# Export PSA routes over VPN (enable export on peering)
gcloud compute networks peerings update servicenetworking-googleapis-com \
  --network=my-vpc \
  --export-custom-routes \
  --import-custom-routes
```

### IP-Based Access Control (NFS Export Options)

Per-share IP restrictions are configured via NFS export options (Zonal/Regional/Enterprise only):

```bash
# Restrict share to a specific CIDR, read-only
gcloud filestore instances update my-zonal-fs \
  --zone=us-central1-a \
  --file-share=name=data,capacity=10TB,nfs-export-options="ip-ranges=10.0.1.0/24,access-mode=READ_ONLY,squash-mode=ROOT_SQUASH"

# Full access from one range, read-only from another
gcloud filestore instances update my-zonal-fs \
  --zone=us-central1-a \
  --file-share='name=data,capacity=10TB,nfs-export-options=[{"ipRanges":["10.0.1.0/24"],"accessMode":"READ_WRITE","squashMode":"NO_ROOT_SQUASH"},{"ipRanges":["10.0.2.0/24"],"accessMode":"READ_ONLY","squashMode":"ROOT_SQUASH"}]'
```

---

## 5. Multi-Share Support

### Overview

**Multi-share** allows a single Filestore instance to host up to **10 independent NFS shares**, each with its own capacity allocation, export options, and access controls. This enables multi-tenancy without provisioning separate instances.

| Feature | Basic HDD | Basic SSD | Zonal | Regional | Enterprise |
|---|---|---|---|---|---|
| Multi-share | ✗ | ✗ | ✅ (up to 10) | ✅ (up to 10) | ✅ (up to 10) |
| Per-share capacity | N/A | N/A | Min 100 GB | Min 100 GB | Min 100 GB |
| Per-share NFS options | ✗ | ✗ | ✅ | ✅ | ✅ |

> **Tip:** The sum of all share capacities must not exceed the instance's total provisioned capacity. You can over-provision shares (thin provisioning) since actual usage is what consumes the pool.

### Creating Shares at Instance Creation

```bash
# Create a Zonal instance with two shares
gcloud filestore instances create my-multi-share-fs \
  --tier=ZONAL \
  --zone=us-central1-a \
  --network=name=my-vpc,connect-mode=PRIVATE_SERVICE_ACCESS \
  --file-share=name=app-data,capacity=3TB \
  --file-share=name=logs,capacity=1TB
```

### Adding a Share to an Existing Instance

```bash
# Add a third share to an existing Zonal instance
gcloud filestore instances update my-multi-share-fs \
  --zone=us-central1-a \
  --file-share=name=backups,capacity=2TB
```

### Per-Share NFS Export Options

| Option | Values | Description |
|---|---|---|
| `access-mode` | `READ_WRITE`, `READ_ONLY` | Permissions for the IP range |
| `squash-mode` | `NO_ROOT_SQUASH`, `ROOT_SQUASH`, `ALL_SQUASH` | UID/GID mapping for NFS root |
| `anon-uid` | integer (default: 65534) | UID mapped to when squashing |
| `anon-gid` | integer (default: 65534) | GID mapped to when squashing |
| `ip-ranges` | CIDR strings | Clients allowed to mount this share |

```bash
# Update share with access restrictions
gcloud filestore instances update my-multi-share-fs \
  --zone=us-central1-a \
  --file-share='name=app-data,capacity=3TB,nfs-export-options=[
    {"ipRanges":["10.0.1.0/24"],"accessMode":"READ_WRITE","squashMode":"NO_ROOT_SQUASH"},
    {"ipRanges":["10.0.2.0/24"],"accessMode":"READ_ONLY","squashMode":"ROOT_SQUASH"}
  ]'
```

### Mounting Multiple Shares from the Same Instance

```bash
FILESTORE_IP="10.100.0.2"

# Mount each share to a different local directory
sudo mkdir -p /mnt/app-data /mnt/logs /mnt/backups

sudo mount -t nfs -o vers=3 ${FILESTORE_IP}:/app-data /mnt/app-data
sudo mount -t nfs -o vers=3 ${FILESTORE_IP}:/logs     /mnt/logs
sudo mount -t nfs -o vers=3 ${FILESTORE_IP}:/backups  /mnt/backups
```

### Multi-Share Use Cases

- **Dev/Staging/Prod isolation:** Separate shares per environment on one instance
- **Team isolation:** One share per team with separate IP-range access controls
- **Application tiers:** Hot data on one share, warm data on another
- **Cost efficiency:** Share a large provisioned instance across multiple workloads

---

## 6. Snapshots & Backups

### Snapshot vs Backup Comparison

| Aspect | Snapshot | Backup |
|---|---|---|
| **Scope** | Per share (point-in-time) | Per instance or share |
| **Tiers supported** | Zonal, Regional, Enterprise only | All tiers |
| **Storage location** | Same instance region | Any supported region |
| **Cross-region** | ✗ | ✅ |
| **Independent lifecycle** | ✗ (deleted with instance) | ✅ (survives instance deletion) |
| **Restore type** | In-place (overwrite share) or new instance | New instance only |
| **Scheduled** | ✅ | ✅ (via Cloud Scheduler + API) |
| **Cost** | Per GB stored | Per GB stored |

### On-Demand Snapshots

```bash
# Create a snapshot of a share on a Zonal instance
gcloud filestore snapshots create my-snapshot-20250101 \
  --instance=my-zonal-fs \
  --instance-zone=us-central1-a \
  --description="Pre-deployment snapshot"

# List snapshots for an instance
gcloud filestore snapshots list \
  --instance=my-zonal-fs \
  --instance-zone=us-central1-a

# Describe a snapshot
gcloud filestore snapshots describe my-snapshot-20250101 \
  --instance=my-zonal-fs \
  --instance-zone=us-central1-a

# Delete a snapshot
gcloud filestore snapshots delete my-snapshot-20250101 \
  --instance=my-zonal-fs \
  --instance-zone=us-central1-a
```

### Restoring from a Snapshot

```bash
# Restore a snapshot IN-PLACE (overwrites current share data — IRREVERSIBLE)
gcloud filestore instances restore my-zonal-fs \
  --zone=us-central1-a \
  --source-snapshot=my-snapshot-20250101 \
  --file-share=data

# Restore snapshot to a NEW instance
gcloud filestore instances create restored-instance \
  --tier=ZONAL \
  --zone=us-central1-a \
  --network=name=my-vpc,connect-mode=PRIVATE_SERVICE_ACCESS \
  --source-snapshot=projects/MY_PROJECT/locations/us-central1-a/instances/my-zonal-fs/snapshots/my-snapshot-20250101 \
  --file-share=name=data,capacity=10TB
```

> ⚠️ **Warning:** In-place snapshot restore **permanently overwrites** the current share contents. This operation cannot be undone. Always verify the snapshot before restoring in-place.

### Scheduled Snapshots

Filestore does not have a built-in snapshot scheduler GUI, but you can automate via **Cloud Scheduler + Cloud Functions**:

```python
# Cloud Function triggered by Cloud Scheduler (Pub/Sub)
import functions_framework
from google.cloud import filestore_v1

@functions_framework.cloud_event
def create_snapshot(cloud_event):
    client = filestore_v1.CloudFilestoreManagerClient()
    instance_name = "projects/MY_PROJECT/locations/us-central1-a/instances/my-zonal-fs"
    snapshot = filestore_v1.Snapshot(description="Scheduled snapshot")
    snapshot_id = f"scheduled-{__import__('datetime').date.today().strftime('%Y%m%d')}"

    operation = client.create_snapshot(
        parent=instance_name,
        snapshot_id=snapshot_id,
        snapshot=snapshot
    )
    operation.result()
    print(f"Snapshot {snapshot_id} created.")
```

### Filestore Backups

Backups are independent of the instance and persist even after the instance is deleted. Supported on all tiers.

```bash
# Create a backup of a share (stored in a specific region)
gcloud filestore backups create my-backup-20250101 \
  --region=us-central1 \
  --source-instance=my-filestore-ssd \
  --source-instance-zone=us-central1-a \
  --source-file-share=vol1

# Cross-region backup (store in a different region)
gcloud filestore backups create my-dr-backup \
  --region=us-east1 \
  --source-instance=my-filestore-ssd \
  --source-instance-zone=us-central1-a \
  --source-file-share=vol1

# List backups
gcloud filestore backups list --region=us-central1

# Describe a backup
gcloud filestore backups describe my-backup-20250101 --region=us-central1

# Restore a backup to a new instance
gcloud filestore instances create restored-from-backup \
  --tier=BASIC_SSD \
  --zone=us-central1-a \
  --network=name=my-vpc,connect-mode=PRIVATE_SERVICE_ACCESS \
  --source-backup=projects/MY_PROJECT/locations/us-central1/backups/my-backup-20250101 \
  --source-backup-region=us-central1 \
  --file-share=name=vol1,capacity=5TB

# Delete a backup
gcloud filestore backups delete my-backup-20250101 --region=us-central1
```

> **Tip:** Use **cross-region backups** as your disaster recovery strategy. Store backups in a region at least 1,000 km away from the source instance for maximum resilience.

---

## 7. GKE Integration

### Filestore CSI Driver

The **Filestore CSI driver** (`filestore.csi.storage.gke.io`) is automatically enabled on GKE clusters running version **1.21+** when the Filestore API is enabled on the project. It supports:
- Dynamic provisioning of Filestore instances
- `ReadWriteMany` (RWX) — multiple pods write simultaneously
- Volume expansion
- Snapshots (Zonal/Regional/Enterprise)

```bash
# Verify CSI driver is running on your GKE cluster
kubectl get pods -n kube-system | grep filestore
kubectl get csidriver filestore.csi.storage.gke.io
```

### StorageClass Definitions

```yaml
# Zonal Filestore StorageClass (standard tier — general purpose)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: filestore-zonal
provisioner: filestore.csi.storage.gke.io
parameters:
  tier: zonal
  network: my-vpc
  connect-mode: PRIVATE_SERVICE_ACCESS
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: Immediate
---
# Regional (HA) Filestore StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: filestore-regional
provisioner: filestore.csi.storage.gke.io
parameters:
  tier: regional
  network: my-vpc
  connect-mode: PRIVATE_SERVICE_ACCESS
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: Immediate
---
# Enterprise StorageClass (NFSv4.1, AD/LDAP)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: filestore-enterprise
provisioner: filestore.csi.storage.gke.io
parameters:
  tier: enterprise
  network: my-vpc
  connect-mode: PRIVATE_SERVICE_ACCESS
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: Immediate
---
# Multi-share StorageClass (single instance, multiple shares)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: filestore-multishare
provisioner: filestore.csi.storage.gke.io
parameters:
  tier: zonal
  multishare: "true"
  network: my-vpc
  connect-mode: PRIVATE_SERVICE_ACCESS
reclaimPolicy: Retain
allowVolumeExpansion: true
```

### PersistentVolumeClaim (Dynamic Provisioning)

```yaml
# Dynamic PVC — automatically creates a new Filestore instance
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteMany       # RWX — the key advantage of Filestore over PD
  storageClassName: filestore-zonal
  resources:
    requests:
      storage: 1Ti        # Minimum 1 TiB for Zonal tier
---
# Multi-share PVC (minimum 100 GiB, shares a pooled instance)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-logs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: filestore-multishare
  resources:
    requests:
      storage: 100Gi
```

### Static Provisioning (Pre-existing Filestore Instance)

```yaml
# PersistentVolume pointing to an existing Filestore instance
apiVersion: v1
kind: PersistentVolume
metadata:
  name: existing-filestore-pv
spec:
  capacity:
    storage: 5Ti
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: filestore.csi.storage.gke.io
    volumeHandle: "modeInstance/us-central1-a/my-filestore-ssd/vol1"
    volumeAttributes:
      ip: "10.100.0.2"
      volume: "vol1"
---
# PVC that binds to the static PV
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: existing-filestore-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""     # Empty string = static binding
  volumeName: existing-filestore-pv
  resources:
    requests:
      storage: 5Ti
```

### Multi-Pod Shared Access (RWX Pattern)

```yaml
# Deployment — all replicas share the same Filestore volume
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 5                 # All 5 pods mount the same NFS share simultaneously
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: nginx:stable
          volumeMounts:
            - name: shared-content
              mountPath: /usr/share/nginx/html
      volumes:
        - name: shared-content
          persistentVolumeClaim:
            claimName: shared-data-pvc
```

### VolumeSnapshot via CSI (Zonal/Regional/Enterprise)

```yaml
# VolumeSnapshotClass for Filestore
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: filestore-snapshot-class
driver: filestore.csi.storage.gke.io
deletionPolicy: Delete
---
# Take a snapshot of a PVC
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: shared-data-snap-20250101
spec:
  volumeSnapshotClassName: filestore-snapshot-class
  source:
    persistentVolumeClaimName: shared-data-pvc
---
# Restore snapshot to a new PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: filestore-zonal
  resources:
    requests:
      storage: 1Ti
  dataSource:
    name: shared-data-snap-20250101
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

### Resize a Filestore PVC

```bash
kubectl patch pvc shared-data-pvc \
  -p '{"spec":{"resources":{"requests":{"storage":"2Ti"}}}}'

kubectl describe pvc shared-data-pvc | grep -A5 Conditions
```

---

## 8. Performance Tuning

### Throughput & IOPS Scaling Summary

| Tier | IOPS formula (read) | Throughput formula | Notes |
|---|---|---|---|
| Basic HDD | ~0.6 IOPS/GB | ~0.09 MB/s/GB | Caps at 600 IOPS / 180 MB/s |
| Basic SSD | ~24 IOPS/GB | ~0.48 MB/s/GB | Caps at 60,000 IOPS / 1,200 MB/s |
| Zonal | ~16 IOPS/GB | ~0.48 MB/s/GB | Caps at 160,000 IOPS / 4,800 MB/s |
| Regional | ~16 IOPS/GB | ~0.48 MB/s/GB | Same caps as Zonal |
| Enterprise | ~12 IOPS/GB | ~0.24 MB/s/GB | Caps at 120,000 IOPS / 2,400 MB/s |

> **Example:** A 10 TB Zonal instance delivers up to 160,000 IOPS and ~4,800 MB/s throughput.

### NFS Mount Options for Performance

```bash
# Optimized mount for high-throughput sequential workloads (ML, media)
sudo mount -t nfs \
  -o vers=3,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=3,async,noatime \
  10.100.0.2:/data /mnt/filestore/data

# Optimized for low-latency IOPS (databases, transaction logs)
sudo mount -t nfs \
  -o vers=3,rsize=65536,wsize=65536,hard,timeo=100,retrans=5,sync,noatime \
  10.100.0.2:/data /mnt/filestore/data
```

### Mount Option Reference

| Option | Recommended Value | Effect |
|---|---|---|
| `vers` | `3` | NFSv3 (default); use `4.1` for Enterprise |
| `rsize` | `1048576` (1 MB) | Read buffer size — larger = better sequential throughput |
| `wsize` | `1048576` (1 MB) | Write buffer size — larger = better sequential throughput |
| `hard` | (flag) | Retry indefinitely on server failure — prevents data corruption |
| `soft` | (flag) | Give up after `timeo` — use only if app handles NFS errors |
| `timeo` | `600` (60s) | Timeout in tenths of a second before retry |
| `retrans` | `3` | Number of retransmissions before error |
| `async` | (flag) | Write buffered (faster) — risk of data loss on crash |
| `sync` | (flag) | Synchronous writes — safer but slower |
| `noatime` | (flag) | Skip access time updates — significant IOPS savings |
| `nconnect` | `4`–`16` | Multiple TCP connections per mount — major throughput boost (kernel 5.3+) |
| `nocto` | (flag) | Disable close-to-open consistency — improves read-heavy workloads |

```bash
# High-concurrency workload: use nconnect for parallel connections
sudo mount -t nfs \
  -o vers=3,rsize=1048576,wsize=1048576,hard,timeo=600,nconnect=8,noatime \
  10.100.0.2:/data /mnt/filestore/data
```

### Kernel NFS Tuning

```bash
# Increase NFS request slots (default 64 — raise for high-concurrency)
echo "options sunrpc tcp_slot_table_entries=128" | sudo tee /etc/modprobe.d/sunrpc.conf
echo "options sunrpc tcp_max_slot_table_entries=128" | sudo tee -a /etc/modprobe.d/sunrpc.conf
sudo sysctl -w sunrpc.tcp_slot_table_entries=128

# Increase dirty page write-back thresholds for large sequential writes
sudo sysctl -w vm.dirty_ratio=40
sudo sysctl -w vm.dirty_background_ratio=10
sudo sysctl -w vm.dirty_expire_centisecs=3000
```

### Benchmarking with fio

```bash
# Sequential write throughput (simulates ML training data writes)
sudo fio --name=seq-write \
  --directory=/mnt/filestore/data \
  --numjobs=8 \
  --size=4G \
  --time_based --runtime=60s \
  --ioengine=libaio \
  --direct=0 \
  --bs=1M \
  --iodepth=64 \
  --rw=write \
  --group_reporting

# Random read IOPS (simulates application metadata reads)
sudo fio --name=rand-read \
  --directory=/mnt/filestore/data \
  --numjobs=8 \
  --size=4G \
  --time_based --runtime=60s \
  --ioengine=libaio \
  --direct=0 \
  --bs=4K \
  --iodepth=128 \
  --rw=randread \
  --group_reporting
```

### Monitoring with nfsstat

```bash
# View NFS client statistics (retransmits, RTT, errors)
nfsstat -c

# View per-mount NFS statistics
nfsstat -m

# Watch NFS operation counts in real time
watch -n 2 nfsstat -c
```

### Common NFS Performance Pitfalls

| Pitfall | Cause | Fix |
|---|---|---|
| Small file overhead | NFSv3 round-trips per file metadata op | Use async mounts, batch operations, or consider a local cache layer |
| Lock contention | Multiple writers competing on same files | Shard workload across multiple shares or directories |
| Low throughput on small VMs | VM network cap lower than Filestore cap | Scale up VM vCPUs; use `nconnect` to parallelize connections |
| High latency on sync mounts | Every write waits for NFS ACK | Switch to `async` for non-critical data; ensure low network RTT |
| Stale file handles | Snapshot or restore invalidated cached inode | Remount the NFS share; clear inode cache |

### Workload-Specific Recommendations

| Workload | Recommended Tier | Mount Options |
|---|---|---|
| ML training (large sequential) | Zonal or Regional | `rsize=1M,wsize=1M,async,noatime,nconnect=8` |
| Web content serving (read-heavy) | Zonal | `rsize=1M,async,noatime,nocto` |
| Database logs (sequential write) | Zonal SSD | `wsize=1M,sync,hard,noatime` |
| General app data | Basic SSD or Zonal | `rsize=1M,wsize=1M,hard,noatime` |
| Cold archive / batch | Basic HDD | `defaults,_netdev` |
| Enterprise apps (SAP, AD) | Enterprise | `vers=4.1,rsize=1M,wsize=1M,hard` |

---

## 9. High Availability & Replication

### HA Model by Tier

| Tier | Zone Replication | Automatic Failover | SLA | RPO | RTO |
|---|---|---|---|---|---|
| Basic HDD | None | ✗ | None | N/A | Manual recovery |
| Basic SSD | None | ✗ | None | N/A | Manual recovery |
| Zonal | Single zone | ✗ | 99.9% | Minutes | Minutes (restart) |
| Regional | 2 zones (sync) | ✅ | 99.99% | Near-zero | < 30 seconds |
| Enterprise | Internal (opaque) | ✅ | 99.99% | Near-zero | < 60 seconds |

### Zonal Tier — Single Zone Architecture

```
Zone A
┌─────────────────────────────────┐
│   Filestore Zonal Instance      │
│   ┌───────────────────────┐     │
│   │ NFS Server (managed)  │     │
│   │ Share: /data          │     │
│   └───────────────────────┘     │
│   GCE VMs / GKE Nodes           │
└─────────────────────────────────┘
         Zone failure → manual failover via backup restore
```

### Regional Tier — Multi-Zone Synchronous Replication

```
Zone A (Primary)                Zone B (Replica)
┌────────────────────┐          ┌────────────────────┐
│ Filestore Regional │◄─Sync───►│ Filestore Regional │
│  Primary Instance  │  Repl.   │  Standby Instance  │
└────────────────────┘          └────────────────────┘
         │                                │
    GKE Nodes A                      GKE Nodes B
         │                                │
         └──────────────┬─────────────────┘
                  Same mount IP
                (Google-managed failover)
```

- Writes are **synchronously committed** to both zones before returning ACK
- On zone failure, Google automatically re-routes the mount IP to the healthy zone
- **Client perspective:** The NFS mount point IP does not change; existing mounts may experience a brief I/O pause during failover (~15–30 seconds)
- Regional replicas are in the **same region** — not cross-region DR

> **Tip:** For GKE workloads, use Regional Filestore with Regional GKE clusters to ensure pods can reschedule to healthy nodes and continue accessing the same Filestore volume after a zone failure.

### Enterprise Tier HA

- HA is built into the Enterprise tier architecture (internal implementation)
- Supports **NFSv4.1** for stronger consistency and session resumption after failover
- Supports **Active Directory and LDAP** integration for identity-based access
- Recommended for **SAP HANA**, Oracle, and other enterprise applications requiring AD/LDAP

### Disaster Recovery Patterns

| Pattern | Description | RTO | RPO | Cost |
|---|---|---|---|---|
| **Cross-region Backup** | Scheduled backups to a secondary region | Hours | Hours | Low |
| **Regional Filestore + Backup** | HA within region + backups for DR | < 30s (zone) / Hours (region) | Near-zero (zone) / Hours (region) | Medium |
| **Dual-Region Deployment** | Separate instances in two regions, app-level sync | Minutes | Minutes–Hours | High |

```bash
# DR runbook: restore from cross-region backup to new region
gcloud filestore instances create dr-instance \
  --tier=ZONAL \
  --zone=us-east1-b \
  --network=name=dr-vpc,connect-mode=PRIVATE_SERVICE_ACCESS \
  --source-backup=projects/MY_PROJECT/locations/us-central1/backups/my-dr-backup \
  --file-share=name=data,capacity=5TB
```

### Multi-Region Architecture Considerations

- Filestore instances are **regional/zonal** — they do not span multiple regions natively
- For multi-region shared storage, use a combination of:
  - **Filestore per region** + application-level replication (rsync, custom sync)
  - **Cloud Storage FUSE** for globally accessible object storage (lower POSIX compliance)
  - **NetApp Cloud Volumes** for cross-region NAS replication on GCP

---

## 10. gcloud CLI Quick Reference

### Instance Operations

```bash
# CREATE
gcloud filestore instances create INSTANCE \
  --tier=ZONAL \
  --file-share=name=SHARE,capacity=SIZE \
  --network=name=VPC,connect-mode=PRIVATE_SERVICE_ACCESS \
  --zone=ZONE

# LIST
gcloud filestore instances list
gcloud filestore instances list --zone=ZONE
gcloud filestore instances list --filter="tier=ZONAL"

# DESCRIBE
gcloud filestore instances describe INSTANCE --zone=ZONE
gcloud filestore instances describe INSTANCE --region=REGION  # for Regional

# DELETE
gcloud filestore instances delete INSTANCE --zone=ZONE
gcloud filestore instances delete INSTANCE --region=REGION

# UPDATE (resize share, update description)
gcloud filestore instances update INSTANCE \
  --zone=ZONE \
  --file-share=name=SHARE,capacity=NEW_SIZE

# UPDATE (add a new share on multi-share instance)
gcloud filestore instances update INSTANCE \
  --zone=ZONE \
  --file-share=name=NEW_SHARE,capacity=SIZE
```

### Snapshot Operations

```bash
# CREATE snapshot
gcloud filestore snapshots create SNAPSHOT_NAME \
  --instance=INSTANCE \
  --instance-zone=ZONE

# LIST snapshots
gcloud filestore snapshots list \
  --instance=INSTANCE \
  --instance-zone=ZONE

# DESCRIBE snapshot
gcloud filestore snapshots describe SNAPSHOT_NAME \
  --instance=INSTANCE \
  --instance-zone=ZONE

# DELETE snapshot
gcloud filestore snapshots delete SNAPSHOT_NAME \
  --instance=INSTANCE \
  --instance-zone=ZONE

# RESTORE in-place from snapshot
gcloud filestore instances restore INSTANCE \
  --zone=ZONE \
  --source-snapshot=SNAPSHOT_NAME \
  --file-share=SHARE_NAME
```

### Backup Operations

```bash
# CREATE backup
gcloud filestore backups create BACKUP_NAME \
  --region=REGION \
  --source-instance=INSTANCE \
  --source-instance-zone=ZONE \
  --source-file-share=SHARE_NAME

# LIST backups
gcloud filestore backups list --region=REGION
gcloud filestore backups list  # all regions

# DESCRIBE backup
gcloud filestore backups describe BACKUP_NAME --region=REGION

# DELETE backup
gcloud filestore backups delete BACKUP_NAME --region=REGION

# RESTORE backup to new instance
gcloud filestore instances create NEW_INSTANCE \
  --tier=TIER \
  --zone=ZONE \
  --network=name=VPC,connect-mode=PRIVATE_SERVICE_ACCESS \
  --source-backup=projects/PROJECT/locations/REGION/backups/BACKUP_NAME \
  --file-share=name=SHARE,capacity=SIZE
```

### NFS Export Options

```bash
# Update NFS export options on a share
gcloud filestore instances update INSTANCE \
  --zone=ZONE \
  --file-share='name=SHARE,capacity=SIZE,nfs-export-options=[
    {"ipRanges":["CIDR"],"accessMode":"READ_WRITE","squashMode":"ROOT_SQUASH"}
  ]'

# Describe instance to see current export options
gcloud filestore instances describe INSTANCE \
  --zone=ZONE \
  --format="json(fileShares)"
```

---

## 11. Pricing Model Summary

> Prices are approximate US region rates as of 2025. Actual pricing varies by region. Always verify at [cloud.google.com/filestore/pricing](https://cloud.google.com/filestore/pricing).

### Instance Storage (per GB / month provisioned)

| Tier | US Regions | Europe | Asia-Pacific |
|---|---|---|---|
| Basic HDD | $0.020 | $0.024 | $0.028 |
| Basic SSD | $0.170 | $0.204 | $0.221 |
| Zonal | $0.110 | $0.130 | $0.150 |
| Regional | $0.220 | $0.265 | $0.300 |
| Enterprise | $0.350 | $0.420 | $0.480 |

### Snapshot & Backup Storage (per GB / month)

| Resource | Price |
|---|---|
| Snapshots (Zonal/Regional/Enterprise) | $0.026 / GB / month |
| Backups (all tiers) | $0.026 / GB / month |

> **Tip:** Snapshots store only changed blocks, so they are typically much smaller than the full provisioned capacity.

### Network Egress

| Destination | Cost |
|---|---|
| Same region (GCE → Filestore) | **Free** |
| Different GCP region | ~$0.01–$0.08 / GB |
| Internet (rare; not typical) | Standard egress pricing |

### Cost Optimization Strategies

| Strategy | Savings |
|---|---|
| Right-size instance capacity | Pay per GB provisioned — start smaller and resize up |
| Use Basic HDD for cold/archival data | 5–8× cheaper than SSD tiers |
| Use Zonal instead of Basic SSD for new deployments | Better value — higher performance at similar cost |
| Use multi-share to consolidate instances | One large instance is cheaper than multiple small ones |
| Delete old snapshots and backups | Orphaned snapshots/backups accumulate cost silently |
| Use Regional only when 99.99% SLA is required | Regional is 2× the cost of Zonal |
| Schedule backups vs continuous snapshots | Backups are full copies — take less frequently for DR only |
| Committed Use Discounts (CUDs) | Available for Filestore — up to 20–30% for 1 or 3 years |

---

## 12. Common Errors & Troubleshooting

| Error / Issue | Cause | Fix |
|---|---|---|
| `mount: connection refused` | Port 2049 blocked by firewall; Filestore IP unreachable | Check firewall rules allow TCP/UDP 2049 from client subnet to Filestore IP |
| `mount: Connection timed out` | VPC routing issue; PSA not configured; wrong IP | Verify PSA setup; check Filestore IP with `gcloud filestore instances describe`; confirm VPC peering routes |
| `mount: access denied by server` | Client IP not in authorized NFS export range; wrong VPC | Check NFS export options IP range; ensure client VM is in the same VPC as Filestore |
| `Stale NFS file handle` | Snapshot restore or Filestore restart invalidated open file handles | Unmount and remount the NFS share: `umount -lf /mnt/filestore && mount ...` |
| `No space left on device` | Share has reached provisioned capacity | Resize the instance: `gcloud filestore instances update --file-share=name=data,capacity=NEW_SIZE` |
| High latency / low throughput | Under-provisioned capacity; VM network bottleneck; `sync` mount | Increase capacity, check VM vCPU count, use `async` mount option, add `nconnect` |
| `Operation not permitted` — snapshot | Tier does not support snapshots (Basic HDD/SSD) | Migrate to Zonal/Regional/Enterprise tier, or use Filestore Backups instead |
| Backup fails with `INVALID_ARGUMENT` | Source share name or instance not found | Verify `--source-file-share` matches exact share name; confirm instance exists |
| GKE PVC stuck in `Pending` | Filestore CSI driver not enabled; capacity too small (min 1 TiB); StorageClass misconfigured | Enable Filestore API; check StorageClass tier matches minimum capacity; check `kubectl describe pvc` |
| GKE PVC stuck in `Terminating` | PV finalizer not released | `kubectl patch pv PV_NAME -p '{"metadata":{"finalizers":null}}'` |
| NFS mount drops / I/O errors | Using `soft` mount option with timeouts | Switch to `hard` mount; investigate network path latency with `ping`, `traceroute` |
| `Permission denied` writing to mount | UID/GID mismatch between client and server; `ROOT_SQUASH` active | Set `NO_ROOT_SQUASH` in NFS export options, or align UID/GID across clients |
| Instance creation fails — quota | Filestore instance count or capacity quota exceeded | Request quota increase via IAM & Admin > Quotas |
| Snapshot restore wipes data | In-place restore was run on wrong instance/share | Always snapshot before restore; prefer `--source-snapshot` to a new instance for safety |
| `nfs: server not responding` | Filestore instance restarting or zone issue | Check instance status: `gcloud filestore instances describe`; use `hard` mounts so clients retry |
| Performance drops after resize | Filesystem not re-balanced (rare for NFS) | No action needed — Filestore manages storage internally; performance recovers within minutes |

### Diagnostic Commands

```bash
# Check if NFS port is reachable from client VM
nc -zv FILESTORE_IP 2049

# View NFS mount statistics and server RTT
nfsstat -m
nfsstat -c

# Check current NFS mounts
mount | grep nfs
cat /proc/mounts | grep nfs

# Check kernel NFS slot table
sysctl sunrpc.tcp_slot_table_entries

# Test NFS performance quickly (without fio)
dd if=/dev/zero of=/mnt/filestore/data/testfile bs=1M count=1024 oflag=direct
dd if=/mnt/filestore/data/testfile of=/dev/null bs=1M count=1024

# Check Filestore instance status
gcloud filestore instances describe my-instance --zone=us-central1-a \
  --format="value(state)"

# Describe GKE CSI driver logs
kubectl logs -n kube-system \
  -l app=filestore-node \
  --container=filestore-driver \
  --tail=50
```

---

## Quick Reference Card

```
Protocol:              NFSv3 (all tiers) | NFSv4.1 (Enterprise only)
Max instance capacity: 63.9 TB (Basic) | 100 TB (Zonal/Regional) | 10 TB (Enterprise)
Max shares/instance:   1 (Basic) | 10 (Zonal/Regional/Enterprise)
Min PVC size (GKE):    1 TiB (dedicated) | 100 GiB (multi-share)
NFS port:              TCP/UDP 2049 (+ port 111 for portmapper)
Connect mode:          PRIVATE_SERVICE_ACCESS (recommended) | DIRECT_PEERING (legacy)
Snapshot support:      Zonal, Regional, Enterprise only
Backup support:        All tiers
Capacity change:       Increase only — no shrink
SLA 99.99%:            Regional and Enterprise tiers
GKE access mode:       ReadWriteMany (RWX) — multiple pods, same volume
CSI driver:            filestore.csi.storage.gke.io
CUD discount:          Available — up to 30% for 1 or 3 year commitment
```

---

*Reference: [Filestore Docs](https://cloud.google.com/filestore/docs) | [Pricing](https://cloud.google.com/filestore/pricing) | [GKE Storage](https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/filestore-file-system) | [CSI Driver](https://github.com/kubernetes-sigs/gcp-filestore-csi-driver)*
