# GCP Compute Engine Cheatsheet

> **TL;DR:** Google Cloud Compute Engine lets you run scalable, high-performance virtual machines on Google's infrastructure — fully configurable from machine type to disk, networking, and lifecycle automation.

---

## Table of Contents

1. [Overview & Core Concepts](#1-overview--core-concepts)
2. [Machine Types & Families](#2-machine-types--families)
3. [Boot Disks & Storage](#3-boot-disks--storage)
4. [Networking Fundamentals](#4-networking-fundamentals)
5. [Instance Lifecycle & Management](#5-instance-lifecycle--management)
6. [Instance Templates & Managed Instance Groups](#6-instance-templates--managed-instance-groups-migs)
7. [Startup & Shutdown Scripts](#7-startup--shutdown-scripts)
8. [IAM & Security](#8-iam--security)
9. [Pricing & Cost Optimization](#9-pricing--cost-optimization)
10. [Monitoring & Logging](#10-monitoring--logging)
11. [gcloud CLI Quick Reference](#11-gcloud-cli-quick-reference)
12. [REST API & Client Library Examples](#12-rest-api--client-library-examples)
13. [Terraform Snippet](#13-terraform-snippet)
14. [Common Patterns & Best Practices](#14-common-patterns--best-practices)
15. [Troubleshooting Quick Reference](#15-troubleshooting-quick-reference)

---

## 1. Overview & Core Concepts

Google Cloud Compute Engine (GCE) is an **Infrastructure-as-a-Service (IaaS)** product that provides virtual machines (VMs) running in Google's data centers, connected to Google's global network.

### Key Terminology

| Term | Description |
|---|---|
| **Instance** | A virtual machine running on Google's infrastructure |
| **Zone** | An isolated deployment area within a region (e.g., `us-central1-a`) |
| **Region** | A geographic location containing multiple zones (e.g., `us-central1`) |
| **Machine Family** | A group of machine types optimized for a workload class (e.g., N2, C2, M3) |
| **vCPU** | Virtual CPU — a hardware hyperthread on the host machine |
| **Machine Type** | A specific combination of vCPUs and memory (e.g., `n2-standard-4`) |
| **Metadata Server** | An internal endpoint (`169.254.169.254`) providing instance config & startup data |
| **Live Migration** | GCE's ability to move running VMs to new hosts during maintenance without downtime |
| **Preemptible / Spot VM** | Short-lived, deeply discounted VMs that can be reclaimed by Google |
| **Instance Template** | A reusable config blueprint for creating instances or MIGs |
| **MIG** | Managed Instance Group — a fleet of identical VMs managed as a unit |
| **Service Account** | A GCP identity used by instances to authenticate to other GCP APIs |
| **Guest OS** | The operating system running inside the VM (Linux or Windows) |
| **Serial Console** | A low-level debugging interface for accessing a VM when SSH is unavailable |

### How GCE Fits in GCP

```
┌─────────────────────────────────────────────┐
│                   GCP Project               │
│  ┌──────────────────────────────────────┐   │
│  │         VPC Network                  │   │
│  │  ┌──────────┐    ┌──────────┐        │   │
│  │  │  Zone A  │    │  Zone B  │        │   │
│  │  │  VM  VM  │    │  VM  VM  │        │   │
│  │  └──────────┘    └──────────┘        │   │
│  └──────────────────────────────────────┘   │
│  Cloud Storage | BigQuery | Cloud SQL ...   │
└─────────────────────────────────────────────┘
```

> **Note:** Always deploy across **multiple zones** within a region for high availability. Cross-region replication is required for full disaster recovery.

---

## 2. Machine Types & Families

### Comparison Table

| Family | Series | Best For | Max vCPUs | Max Memory |
|---|---|---|---|---|
| **General Purpose** | E2 | Cost-optimized workloads | 32 | 128 GB |
| **General Purpose** | N1 | Flexible, legacy workloads | 96 | 624 GB |
| **General Purpose** | N2 | Balanced performance | 128 | 864 GB |
| **General Purpose** | N2D | AMD EPYC, scale-out | 224 | 896 GB |
| **General Purpose** | T2D | ARM (Ampere Altra), scale-out | 60 | 240 GB |
| **Compute Optimized** | C2 | High-frequency single-thread | 60 | 240 GB |
| **Compute Optimized** | C3 | Latest gen Intel, HPC | 176 | 1,408 GB |
| **Memory Optimized** | M1 | In-memory databases (SAP HANA) | 160 | 3,844 GB |
| **Memory Optimized** | M2 | Ultra large in-memory | 416 | 11,776 GB |
| **Memory Optimized** | M3 | Latest gen memory-optimized | 128 | 1,952 GB |
| **Accelerator Optimized** | A2 | NVIDIA A100 GPU workloads | 96 | 1,360 GB |
| **Accelerator Optimized** | G2 | NVIDIA L4 GPU, ML inference | 96 | 384 GB |

### Predefined Machine Type Naming

```
{family}-{type}-{vcpus}

e2-standard-4      →  E2 family, standard, 4 vCPUs
n2-highmem-16      →  N2 family, high-memory, 16 vCPUs
c2-standard-8      →  C2 family, standard, 8 vCPUs
```

### Standard Type Tiers

| Tier | vCPU:Memory Ratio | Example |
|---|---|---|
| `standard` | 1 vCPU : 4 GB | `n2-standard-8` (32 GB) |
| `highmem` | 1 vCPU : 8 GB | `n2-highmem-8` (64 GB) |
| `highcpu` | 1 vCPU : 1 GB | `n2-highcpu-8` (8 GB) |
| `micro` / `small` | Shared vCPU | `e2-micro`, `e2-small` |

### Custom Machine Types

Create instances with any vCPU/memory combo within family limits:

```bash
# Custom: 6 vCPUs, 20 GB RAM on N2
gcloud compute instances create my-vm \
  --machine-type=n2-custom-6-20480 \
  --zone=us-central1-a
```

> **Note:** Custom machine types are billed at a slight premium over predefined equivalents but can save cost by avoiding over-provisioning.

---

## 3. Boot Disks & Storage

### Persistent Disk Types

| Type | gcloud name | IOPS (per GB) | Best For |
|---|---|---|---|
| Standard HDD | `pd-standard` | 0.75 read / 1.5 write | Batch, cold storage |
| Balanced SSD | `pd-balanced` | 6 read / 6 write | General purpose |
| Performance SSD | `pd-ssd` | 30 read / 30 write | Low-latency OLTP |
| Extreme SSD | `pd-extreme` | Provisioned up to 120k | Highest-performance DBs |

### Local SSD

- Physically attached NVMe/SCSI drives (375 GB per disk, up to 24 disks).
- **Ephemeral** — data is lost if the VM is stopped, deleted, or host-migrated.
- ~100x faster IOPS than Persistent Disk.

```bash
# Add local SSD at instance creation
gcloud compute instances create my-vm \
  --local-ssd=interface=NVME \
  --zone=us-central1-a
```

### Disk Operations

```bash
# Create a new persistent disk
gcloud compute disks create my-disk \
  --size=200GB \
  --type=pd-ssd \
  --zone=us-central1-a

# Attach to a running instance
gcloud compute instances attach-disk my-vm \
  --disk=my-disk \
  --zone=us-central1-a

# Resize a disk (online, no restart needed)
gcloud compute disks resize my-disk \
  --size=400GB \
  --zone=us-central1-a
```

> **Note:** You can only **increase** disk size — shrinking a persistent disk is not supported.

### Snapshots

Snapshots are incremental, global backups of persistent disks.

```bash
# Create a snapshot
gcloud compute disks snapshot my-disk \
  --snapshot-names=my-disk-snap-$(date +%Y%m%d) \
  --zone=us-central1-a

# Create a disk from a snapshot
gcloud compute disks create restored-disk \
  --source-snapshot=my-disk-snap-20240101 \
  --zone=us-central1-b
```

### Images & Custom Images

```bash
# List public images for a project
gcloud compute images list --project=debian-cloud

# Create a custom image from a disk
gcloud compute images create my-golden-image \
  --source-disk=my-disk \
  --source-disk-zone=us-central1-a \
  --family=my-app-family

# Create instance from custom image
gcloud compute instances create my-vm \
  --image=my-golden-image \
  --image-project=my-gcp-project \
  --zone=us-central1-a
```

---

## 4. Networking Fundamentals

### VPC Integration

- Every instance lives inside a **VPC subnet**.
- Subnets are **regional** (span all zones in a region).
- VMs in the same VPC can communicate using internal IPs by default.

```bash
# Create a custom VPC and subnet
gcloud compute networks create my-vpc --subnet-mode=custom

gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.0.0/24
```

### Internal vs External IPs

| Type | Description | Persistent? |
|---|---|---|
| **Internal IP** | Private RFC-1918 address within VPC | Yes (via static reservation) |
| **External IP (ephemeral)** | Public IP, released on stop | No |
| **External IP (static)** | Reserved public IP, billed when unused | Yes |

```bash
# Reserve a static external IP
gcloud compute addresses create my-static-ip --region=us-central1

# Assign static IP at instance creation
gcloud compute instances create my-vm \
  --address=my-static-ip \
  --zone=us-central1-a
```

### Firewall Rules

```bash
# Allow SSH (port 22) from any IP to instances tagged "allow-ssh"
gcloud compute firewall-rules create allow-ssh \
  --network=my-vpc \
  --allow=tcp:22 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=allow-ssh

# Allow HTTP/HTTPS from internet
gcloud compute firewall-rules create allow-web \
  --network=my-vpc \
  --allow=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0

# Deny all ingress (lowest priority catch-all)
gcloud compute firewall-rules create deny-all-ingress \
  --network=my-vpc \
  --action=DENY \
  --rules=all \
  --direction=INGRESS \
  --priority=65534
```

### Network Tags

Tags link firewall rules to specific instances without IP-based targeting.

```bash
# Add a tag to an existing instance
gcloud compute instances add-tags my-vm \
  --tags=allow-ssh,http-server \
  --zone=us-central1-a
```

### Cloud NAT

Allows instances **without** external IPs to initiate outbound internet connections.

```bash
gcloud compute routers create my-router \
  --network=my-vpc --region=us-central1

gcloud compute routers nats create my-nat \
  --router=my-router \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges \
  --region=us-central1
```

---

## 5. Instance Lifecycle & Management

### Instance States

```
PROVISIONING → STAGING → RUNNING → STOPPING → TERMINATED
                                 ↘ SUSPENDING → SUSPENDED
```

| State | Description |
|---|---|
| `PROVISIONING` | Resources being allocated |
| `STAGING` | Resources acquired, OS booting |
| `RUNNING` | VM is live and operational |
| `STOPPING` | Shutdown in progress |
| `TERMINATED` | Stopped — disk retained, no compute charge |
| `SUSPENDED` | Memory state preserved (like laptop sleep) |

### Create, Start, Stop, Delete

```bash
# Create a basic Debian instance
gcloud compute instances create my-vm \
  --machine-type=e2-standard-2 \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=50GB \
  --boot-disk-type=pd-balanced \
  --zone=us-central1-a

# Start / Stop / Restart
gcloud compute instances start  my-vm --zone=us-central1-a
gcloud compute instances stop   my-vm --zone=us-central1-a
gcloud compute instances reset  my-vm --zone=us-central1-a   # hard reboot

# Delete (add --delete-disks=all to also remove boot disk)
gcloud compute instances delete my-vm --zone=us-central1-a
```

### SSH Access

```bash
# SSH via gcloud (handles key management automatically)
gcloud compute ssh my-vm --zone=us-central1-a

# SSH with port forwarding (e.g., tunnel a remote DB to localhost:5432)
gcloud compute ssh my-vm --zone=us-central1-a \
  -- -L 5432:localhost:5432
```

### Serial Console Access

Useful when SSH is broken (boot failure, network misconfiguration).

```bash
# Enable interactive serial console
gcloud compute instances add-metadata my-vm \
  --metadata=serial-port-enable=1 \
  --zone=us-central1-a

# Connect to serial console
gcloud compute connect-to-serial-port my-vm --zone=us-central1-a
```

### OS Login

Replaces SSH key metadata with IAM-based access control.

```bash
# Enable OS Login project-wide
gcloud compute project-info add-metadata \
  --metadata=enable-oslogin=TRUE

# Grant OS Login access to a user
gcloud projects add-iam-policy-binding my-project \
  --member=user:dev@example.com \
  --role=roles/compute.osLogin
```

---

## 6. Instance Templates & Managed Instance Groups (MIGs)

### Instance Templates

```bash
# Create an instance template
gcloud compute instance-templates create my-template \
  --machine-type=n2-standard-2 \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=50GB \
  --tags=http-server \
  --metadata=startup-script='#!/bin/bash
    apt-get update -y && apt-get install -y nginx'
```

### Create a MIG from a Template

```bash
gcloud compute instance-groups managed create my-mig \
  --template=my-template \
  --size=3 \
  --zone=us-central1-a
```

### Autoscaling

```bash
gcloud compute instance-groups managed set-autoscaling my-mig \
  --zone=us-central1-a \
  --min-num-replicas=2 \
  --max-num-replicas=10 \
  --target-cpu-utilization=0.6 \
  --cool-down-period=90
```

### Health Checks

```bash
# Create an HTTP health check
gcloud compute health-checks create http my-health-check \
  --port=80 \
  --request-path=/healthz \
  --check-interval=10s \
  --timeout=5s \
  --healthy-threshold=2 \
  --unhealthy-threshold=3

# Attach to MIG
gcloud compute instance-groups managed update my-mig \
  --health-check=my-health-check \
  --initial-delay=60s \
  --zone=us-central1-a
```

### Rolling Updates

```bash
# Update MIG to a new template (rolling, max 1 unavailable at a time)
gcloud compute instance-groups managed rolling-action start-update my-mig \
  --version=template=my-template-v2 \
  --max-unavailable=1 \
  --max-surge=1 \
  --zone=us-central1-a
```

### Canary Deployments

```bash
# Send 20% of traffic to new template version
gcloud compute instance-groups managed rolling-action start-update my-mig \
  --version=template=my-template-v1 \
  --canary-version=template=my-template-v2,target-size=20% \
  --zone=us-central1-a
```

> **Note:** Regional MIGs (multi-zone) are preferred over zonal MIGs for production workloads to survive zone failures.

---

## 7. Startup & Shutdown Scripts

Scripts are passed via **instance metadata** and run automatically by the guest OS.

### Startup Script (inline via gcloud)

```bash
gcloud compute instances create my-vm \
  --metadata=startup-script='#!/bin/bash
set -e
apt-get update -y
apt-get install -y nginx
systemctl enable --now nginx
echo "VM $(hostname) ready" | tee /var/log/startup-done.log' \
  --zone=us-central1-a
```

### Startup Script (from a GCS file)

```bash
# Upload script to GCS
gsutil cp startup.sh gs://my-bucket/scripts/startup.sh

# Reference it via metadata key
gcloud compute instances create my-vm \
  --metadata=startup-script-url=gs://my-bucket/scripts/startup.sh \
  --zone=us-central1-a
```

### Shutdown Script

```bash
gcloud compute instances add-metadata my-vm \
  --metadata=shutdown-script='#!/bin/bash
# Drain application, flush logs, etc.
systemctl stop myapp
sync' \
  --zone=us-central1-a
```

### Reading Metadata from Inside the VM

```bash
# Fetch instance name from metadata server
curl -s "http://metadata.google.internal/computeMetadata/v1/instance/name" \
  -H "Metadata-Flavor: Google"

# Fetch a custom metadata key
curl -s "http://metadata.google.internal/computeMetadata/v1/instance/attributes/my-key" \
  -H "Metadata-Flavor: Google"
```

> **Note:** Always include the `Metadata-Flavor: Google` header — requests without it are rejected to prevent SSRF attacks.

---

## 8. IAM & Security

### Service Accounts

```bash
# Create a dedicated service account for a VM
gcloud iam service-accounts create vm-sa \
  --display-name="My VM Service Account"

# Grant it read access to GCS
gcloud projects add-iam-policy-binding my-project \
  --member=serviceAccount:vm-sa@my-project.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer

# Attach to an instance at creation
gcloud compute instances create my-vm \
  --service-account=vm-sa@my-project.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --zone=us-central1-a
```

### Access Scopes vs IAM Roles

| Concept | Description |
|---|---|
| **Access Scopes** | Legacy mechanism — OAuth 2.0 scopes limiting API access (use `cloud-platform` to rely on IAM only) |
| **IAM Roles** | Modern, fine-grained permission management. Always prefer over narrow scopes |

> **Note:** Use `--scopes=cloud-platform` combined with a least-privilege service account — this delegates all access control to IAM and avoids scope confusion.

### Key IAM Roles for Compute Engine

| Role | Description |
|---|---|
| `roles/compute.admin` | Full control of all Compute resources |
| `roles/compute.instanceAdmin.v1` | Create, modify, delete instances |
| `roles/compute.networkAdmin` | Manage networking resources |
| `roles/compute.storageAdmin` | Manage disks and images |
| `roles/compute.viewer` | Read-only access |
| `roles/compute.osLogin` | SSH as a non-admin user |
| `roles/compute.osAdminLogin` | SSH as a sudo-capable user |

### Shielded VMs

Protects VMs against rootkits and bootkits using Secure Boot, vTPM, and Integrity Monitoring.

```bash
gcloud compute instances create my-shielded-vm \
  --shielded-secure-boot \
  --shielded-vtpm \
  --shielded-integrity-monitoring \
  --zone=us-central1-a
```

### Confidential VMs

Encrypts VM memory in-use using AMD SEV (hardware-based). Ideal for sensitive data.

```bash
gcloud compute instances create my-confidential-vm \
  --confidential-compute \
  --machine-type=n2d-standard-2 \
  --zone=us-central1-a
```

### OS Hardening Tips

- Disable the default `compute` service account; use a custom least-privilege SA.
- Block project-wide SSH keys on sensitive VMs: `--metadata=block-project-ssh-keys=TRUE`
- Restrict SSH with OS Login + 2FA: enable `enable-oslogin-2fa` metadata.
- Use VPC firewall rules to allow SSH only from IAP ranges (`35.235.240.0/20`).
- Enable Shielded VM for all production workloads.

---

## 9. Pricing & Cost Optimization

### Billing Model

- Billed **per second** with a 1-minute minimum.
- Charges apply for: vCPUs, memory, GPUs, disk (GiB/month), external IPs, egress.

### VM Pricing Tiers

| Type | Discount | Notes |
|---|---|---|
| **On-Demand** | None | Standard pricing, no commitment |
| **Spot VM** | Up to 91% | Can be preempted with 30-second notice. No SLA |
| **Committed Use (1yr)** | ~37% | 1-year resource commitment |
| **Committed Use (3yr)** | ~55% | 3-year resource commitment |
| **Sustained Use** | Up to 30% | Auto-applied when instance runs >25% of a month |

### Spot (Preemptible) VMs

```bash
gcloud compute instances create my-spot-vm \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP \
  --machine-type=n2-standard-4 \
  --zone=us-central1-a
```

> **Note:** Use Spot VMs for fault-tolerant batch jobs, CI/CD pipelines, and stateless workloads. Never use them for stateful databases or services requiring continuous availability.

### Committed Use Discounts (CUDs)

```bash
# Purchase a 1-year CUD for 8 vCPUs and 32 GB RAM in us-central1
gcloud compute commitments create my-commitment \
  --plan=12-month \
  --region=us-central1 \
  --resources=vcpu=8,memory=32GB
```

### Cost Optimization Checklist

- Right-size VMs using **Recommender API** suggestions in Cloud Console.
- Delete unattached persistent disks and unused static IPs.
- Set up **budget alerts** in Billing console.
- Use **E2 instances** for dev/test — cheapest general-purpose option.
- Schedule non-production VMs off overnight with Cloud Scheduler + Cloud Functions.
- Prefer regional persistent disks for MIGs — avoids cross-zone I/O charges.

---

## 10. Monitoring & Logging

### Default Metrics (Auto-collected)

| Metric | Path |
|---|---|
| CPU Utilization | `compute.googleapis.com/instance/cpu/utilization` |
| Disk Read/Write Ops | `compute.googleapis.com/instance/disk/read_ops_count` |
| Network In/Out | `compute.googleapis.com/instance/network/received_bytes_count` |
| Memory (with agent) | `agent.googleapis.com/memory/percent_used` |

> **Note:** Memory and disk space metrics require the **Cloud Ops Agent** to be installed inside the VM.

### Install the Cloud Ops Agent

```bash
# On Debian/Ubuntu inside the VM
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install
```

### Create an Uptime Check & Alerting Policy (gcloud)

```bash
# Create uptime check for a GCE instance
gcloud monitoring uptime-checks create http my-check \
  --display-name="My VM HTTP Check" \
  --host=34.x.x.x \
  --path=/healthz \
  --port=80 \
  --check-interval=60s
```

### View Logs

```bash
# Stream serial port output (boot/startup logs)
gcloud compute instances get-serial-port-output my-vm --zone=us-central1-a

# Query Cloud Logging for syslog from a specific instance
gcloud logging read \
  'resource.type="gce_instance" AND resource.labels.instance_id="123456789" AND logName="projects/my-project/logs/syslog"' \
  --limit=50 \
  --format=json
```

### Custom Metric via Ops Agent Config

```yaml
# /etc/google-cloud-ops-agent/config.yaml
metrics:
  receivers:
    nginx_metrics:
      type: nginx
      stub_status_url: http://localhost:80/status
  service:
    pipelines:
      nginx_pipeline:
        receivers: [nginx_metrics]
```

---

## 11. `gcloud` CLI Quick Reference

### Instance Commands

| Command | Description |
|---|---|
| `gcloud compute instances list` | List all instances in the project |
| `gcloud compute instances describe my-vm --zone=ZONE` | Show full config of an instance |
| `gcloud compute instances create ...` | Create a new instance |
| `gcloud compute instances start/stop/reset my-vm` | Change instance state |
| `gcloud compute instances delete my-vm` | Delete an instance |
| `gcloud compute instances update my-vm --machine-type=TYPE` | Change machine type (requires stop) |
| `gcloud compute instances move my-vm --destination-zone=ZONE` | Move instance to another zone |
| `gcloud compute ssh my-vm --zone=ZONE` | SSH into instance |

```bash
# List all running instances across all zones
gcloud compute instances list --filter="status=RUNNING"
```

### Disk Commands

| Command | Description |
|---|---|
| `gcloud compute disks list` | List all disks |
| `gcloud compute disks create NAME --size=SIZE --type=TYPE` | Create a persistent disk |
| `gcloud compute disks resize NAME --size=SIZE` | Grow a disk |
| `gcloud compute disks snapshot NAME` | Create a snapshot |
| `gcloud compute disks delete NAME` | Delete a disk |

```bash
# List all unattached (orphaned) disks
gcloud compute disks list --filter="-users:*" --format="table(name,zone,sizeGb,type)"
```

### Image Commands

| Command | Description |
|---|---|
| `gcloud compute images list` | List available public/custom images |
| `gcloud compute images create NAME --source-disk=DISK` | Create image from disk |
| `gcloud compute images deprecate NAME --state=DEPRECATED` | Deprecate an image version |
| `gcloud compute images delete NAME` | Delete a custom image |

```bash
# List only your project's custom images
gcloud compute images list --project=my-project --no-standard-images
```

### Snapshot Commands

```bash
# Create a scheduled snapshot policy
gcloud compute resource-policies create snapshot-schedule daily-backup \
  --region=us-central1 \
  --max-retention-days=7 \
  --daily-schedule \
  --start-time=03:00

# Attach snapshot schedule to a disk
gcloud compute disks add-resource-policies my-disk \
  --resource-policies=daily-backup \
  --zone=us-central1-a
```

### Firewall Commands

```bash
# List all firewall rules sorted by priority
gcloud compute firewall-rules list --sort-by=PRIORITY

# Update a firewall rule to add a port
gcloud compute firewall-rules update allow-web --allow=tcp:80,tcp:443,tcp:8080
```

### Instance Group Commands

```bash
# List MIGs and their sizes
gcloud compute instance-groups managed list

# Scale a MIG manually
gcloud compute instance-groups managed resize my-mig --size=5 --zone=us-central1-a

# Recreate all instances in a MIG (force update)
gcloud compute instance-groups managed recreate-instances my-mig \
  --instances=my-mig-abc1,my-mig-abc2 \
  --zone=us-central1-a
```

---

## 12. REST API & Client Library Examples

### Create an Instance via REST API (curl)

```bash
curl -X POST \
  "https://compute.googleapis.com/compute/v1/projects/my-project/zones/us-central1-a/instances" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-api-vm",
    "machineType": "zones/us-central1-a/machineTypes/e2-standard-2",
    "disks": [{
      "boot": true,
      "autoDelete": true,
      "initializeParams": {
        "sourceImage": "projects/debian-cloud/global/images/family/debian-12",
        "diskSizeGb": "50",
        "diskType": "zones/us-central1-a/diskTypes/pd-balanced"
      }
    }],
    "networkInterfaces": [{
      "network": "global/networks/default",
      "accessConfigs": [{
        "type": "ONE_TO_ONE_NAT",
        "name": "External NAT"
      }]
    }],
    "serviceAccounts": [{
      "email": "vm-sa@my-project.iam.gserviceaccount.com",
      "scopes": ["https://www.googleapis.com/auth/cloud-platform"]
    }]
  }'
```

### Create an Instance via Python Client Library

```python
from google.cloud import compute_v1

def create_instance(project: str, zone: str, name: str) -> compute_v1.Instance:
    client = compute_v1.InstancesClient()

    instance = compute_v1.Instance()
    instance.name = name
    instance.machine_type = f"zones/{zone}/machineTypes/e2-standard-2"

    # Boot disk
    disk = compute_v1.AttachedDisk()
    disk.boot = True
    disk.auto_delete = True
    initialize_params = compute_v1.AttachedDiskInitializeParams()
    initialize_params.source_image = (
        "projects/debian-cloud/global/images/family/debian-12"
    )
    initialize_params.disk_size_gb = 50
    initialize_params.disk_type = f"zones/{zone}/diskTypes/pd-balanced"
    disk.initialize_params = initialize_params
    instance.disks = [disk]

    # Network interface with external IP
    network_interface = compute_v1.NetworkInterface()
    network_interface.network = "global/networks/default"
    access_config = compute_v1.AccessConfig()
    access_config.type_ = compute_v1.AccessConfig.Type.ONE_TO_ONE_NAT
    access_config.name = "External NAT"
    network_interface.access_configs = [access_config]
    instance.network_interfaces = [network_interface]

    # Create the instance
    operation = client.insert(project=project, zone=zone, instance_resource=instance)
    operation.result(timeout=300)  # Wait for completion

    return client.get(project=project, zone=zone, instance=name)


if __name__ == "__main__":
    vm = create_instance("my-project", "us-central1-a", "my-python-vm")
    print(f"Created: {vm.name} | Status: {vm.status}")
```

> **Note:** Install the library with `pip install google-cloud-compute`. Authentication uses Application Default Credentials — run `gcloud auth application-default login` locally.

---

## 13. Terraform Snippet

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = "us-central1"
}

variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

# ── Data disk ──────────────────────────────────────────────
resource "google_compute_disk" "data_disk" {
  name  = "my-vm-data-disk"
  type  = "pd-ssd"
  zone  = "us-central1-a"
  size  = 100
}

# ── VM Instance ────────────────────────────────────────────
resource "google_compute_instance" "vm" {
  name         = "my-terraform-vm"
  machine_type = "n2-standard-2"
  zone         = "us-central1-a"

  tags = ["http-server", "allow-ssh"]

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      size  = 50
      type  = "pd-balanced"
    }
  }

  # Attach the data disk
  attached_disk {
    source      = google_compute_disk.data_disk.self_link
    device_name = "data-disk"
    mode        = "READ_WRITE"
  }

  network_interface {
    network    = "default"
    subnetwork = "default"

    # Remove this block to create a private-only instance
    access_config {}
  }

  service_account {
    email  = "vm-sa@${var.project_id}.iam.gserviceaccount.com"
    scopes = ["cloud-platform"]
  }

  metadata = {
    enable-oslogin = "TRUE"
    startup-script = <<-EOF
      #!/bin/bash
      apt-get update -y && apt-get install -y nginx
    EOF
  }

  shielded_instance_config {
    enable_secure_boot          = true
    enable_vtpm                 = true
    enable_integrity_monitoring = true
  }

  lifecycle {
    # Prevent accidental destruction of the VM
    prevent_destroy = false
    # Ignore external changes to startup-script
    ignore_changes = [metadata["startup-script"]]
  }
}

# ── Outputs ────────────────────────────────────────────────
output "instance_internal_ip" {
  value = google_compute_instance.vm.network_interface[0].network_ip
}

output "instance_external_ip" {
  value = try(google_compute_instance.vm.network_interface[0].access_config[0].nat_ip, "none")
}
```

```bash
# Deploy
terraform init
terraform plan -var="project_id=my-project"
terraform apply -var="project_id=my-project"
```

---

## 14. Common Patterns & Best Practices

### Immutable Infrastructure

Never patch running VMs. Instead:
1. Build a new **custom image** with the updated OS/app.
2. Update the **instance template** to reference the new image.
3. Perform a **rolling update** of the MIG.
4. Deprecate or delete the old image.

```bash
# Build → tag → deploy workflow
gcloud compute images create app-v2-$(date +%Y%m%d) \
  --source-disk=build-disk --source-disk-zone=us-central1-a \
  --family=my-app

gcloud compute instance-templates create tmpl-v2 \
  --image-family=my-app --image-project=my-project ...

gcloud compute instance-groups managed rolling-action start-update my-mig \
  --version=template=tmpl-v2 --zone=us-central1-a
```

### Golden Images

Maintain a curated base image with:
- Hardened OS (CIS benchmarks applied)
- Monitoring/logging agents pre-installed
- Required packages baked in (no runtime `apt-get`)
- Tagged with an image **family** for easy versioning

### Least-Privilege Service Accounts

```bash
# One dedicated SA per application role — never share
gcloud iam service-accounts create sa-web-frontend
gcloud iam service-accounts create sa-backend-api
gcloud iam service-accounts create sa-data-pipeline

# Grant only what's needed
gcloud projects add-iam-policy-binding my-project \
  --member=serviceAccount:sa-web-frontend@my-project.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer
```

### Automated Snapshot Schedules

```bash
# Attach an hourly snapshot schedule to production disks
gcloud compute resource-policies create snapshot-schedule prod-hourly \
  --region=us-central1 \
  --max-retention-days=3 \
  --hourly-schedule=1 \
  --start-time=00:00

gcloud compute disks add-resource-policies prod-disk \
  --resource-policies=prod-hourly \
  --zone=us-central1-a
```

### Multi-Zone Redundancy

```bash
# Create a REGIONAL MIG spanning all zones in us-central1
gcloud compute instance-groups managed create my-regional-mig \
  --template=my-template \
  --size=6 \
  --region=us-central1
```

### Additional Best Practices

- **Use labels** consistently for cost attribution: `env=prod`, `team=backend`, `app=api`.
- **Never use the default service account** (`PROJECT_NUMBER-compute@developer.gserviceaccount.com`) for production VMs — it has broad project-editor permissions.
- **Enable deletion protection** on critical VMs: `gcloud compute instances update my-vm --deletion-protection`.
- **Store startup scripts in GCS**, not inline metadata, for version control and auditability.
- **Test startup scripts** with Cloud Shell before deploying to production.

---

## 15. Troubleshooting Quick Reference

| Issue | Likely Cause | Fix |
|---|---|---|
| `QUOTA_EXCEEDED` on instance create | Project or regional CPU/GPU quota exhausted | Request quota increase in IAM & Admin → Quotas, or use a different region |
| SSH connection timeout | No firewall rule for TCP:22, or instance has no external IP | Add `allow-ssh` firewall rule targeting instance tag; verify IP assignment |
| SSH `Permission denied (publickey)` | SSH key not in instance metadata or OS Login misconfiguration | Run `gcloud compute ssh` (auto-manages keys) or check OS Login IAM roles |
| Instance stuck in `STAGING` | Boot disk image issue or quota problem | Check serial console output: `gcloud compute instances get-serial-port-output` |
| Startup script not running | Script syntax error, wrong metadata key, or GCS permissions | Verify key is `startup-script` (not `startup_script`); check `syslog` via Cloud Logging |
| Disk attachment fails | Disk in a different zone than instance | Disks must be in the **same zone** as the instance; create a snapshot and restore in the correct zone |
| Cannot resize disk online | Filesystem not extended after disk resize | Resize is two steps: `gcloud compute disks resize` (API) then `resize2fs /dev/sda` (inside VM) |
| `RESOURCE_ALREADY_EXISTS` | Name conflict | Use `gcloud compute instances list` to check; choose a unique name |
| MIG autoscaling not triggering | Cool-down period active or metric not flowing | Wait for cool-down; verify Cloud Ops Agent is running and sending metrics |
| High network egress costs | Traffic going to external IPs unnecessarily | Route traffic through internal IPs; use Private Google Access; check for data exfiltration |
| Serial console access denied | `serial-port-enable` metadata not set | Add `serial-port-enable=1` to instance metadata; ensure IAM allows `compute.instances.getSerialPortOutput` |
| `CONDITION_NOT_MET` on delete | Deletion protection is enabled | Run `gcloud compute instances update my-vm --no-deletion-protection` first |
| GPU instance creation fails | GPU quota or zone availability | Check GPU quotas; try a different zone; verify GPU is available in that zone with `gcloud compute accelerator-types list` |

---

*Last updated: 2025 | Covers GCE GA features as of this date.*
*Refer to [cloud.google.com/compute/docs](https://cloud.google.com/compute/docs) for the latest information.*
