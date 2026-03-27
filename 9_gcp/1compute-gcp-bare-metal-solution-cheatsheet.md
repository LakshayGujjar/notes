# GCP Bare Metal Solution (BMS) Cheatsheet

> **TL;DR:** Bare Metal Solution is GCP's dedicated single-tenant physical server offering — purpose-built for specialized enterprise workloads (Oracle Database, SAP HANA, HPC) that cannot run on virtual machines, co-located in Google data centers and connected to GCP services via low-latency Cloud Interconnect.

---

## Table of Contents

1. [Overview & Core Concepts](#1-overview--core-concepts)
2. [Architecture & Infrastructure Model](#2-architecture--infrastructure-model)
3. [Server Types & Hardware Configurations](#3-server-types--hardware-configurations)
4. [Networking Deep Dive](#4-networking-deep-dive)
5. [Storage Architecture](#5-storage-architecture)
6. [Provisioning & Lifecycle Management](#6-provisioning--lifecycle-management)
7. [Operating System & Software Management](#7-operating-system--software-management)
8. [Connecting BMS to GCP Services](#8-connecting-bms-to-gcp-services)
9. [Oracle on Bare Metal Solution](#9-oracle-on-bare-metal-solution)
10. [SAP on Bare Metal Solution](#10-sap-on-bare-metal-solution)
11. [IAM & Security](#11-iam--security)
12. [Monitoring, Logging & Observability](#12-monitoring-logging--observability)
13. [Backup, DR & High Availability](#13-backup-dr--high-availability)
14. [gcloud & API Quick Reference](#14-gcloud--api-quick-reference)
15. [Terraform Snippet](#15-terraform-snippet)
16. [Cost Model & Optimization](#16-cost-model--optimization)
17. [Migration to BMS](#17-migration-to-bms)
18. [Troubleshooting Quick Reference](#18-troubleshooting-quick-reference)

---

## 1. Overview & Core Concepts

**Bare Metal Solution (BMS)** provides dedicated, single-tenant physical servers housed in Google-managed data center space that is co-located with or adjacent to GCP regions. It is designed for specialized enterprise workloads with strict requirements around hardware access, software licensing, or performance that prevent virtualization.

### The Problem BMS Solves

| Requirement | Why VMs Can't Satisfy It | BMS Solution |
|---|---|---|
| Oracle Database licensing | Oracle charges per physical core on VMs | Dedicated physical cores; Oracle counts only BMS cores |
| SAP HANA TDI certification | SAP certifies specific hardware; VMs are not always certified | Google-certified BMS profiles for SAP HANA |
| Non-virtualized workloads | Some legacy apps require direct hardware access | Full hardware access; no hypervisor layer |
| Ultra-low latency | Hypervisor adds microsecond overhead | Direct hardware; no virtualization overhead |
| Hardware Security Modules | Some HSM requirements forbid shared hardware | Dedicated physical server |

### Key Terminology

| Term | Description |
|---|---|
| **Bare Metal Server** | A dedicated, single-tenant physical server provisioned for one customer |
| **Pod** | A logical grouping of BMS infrastructure within a region; defines the network/storage boundary |
| **Region** | The GCP region where BMS infrastructure is located (must match your GCP project region) |
| **VLAN** | Virtual LAN — the Layer 2 network segment connecting BMS servers to each other and to GCP |
| **VLAN Attachment** | A logical connection between a Cloud Interconnect circuit and a GCP VPC network |
| **LUN** | Logical Unit Number — a block storage device presented to a BMS server via iSCSI |
| **NFS Share** | A network file system share provisioned from BMS storage and mounted on BMS servers |
| **Client Project** | The GCP project that owns the BMS resources and billing |
| **BMS Project** | Google's internal project that manages the physical BMS infrastructure |
| **Cloud Interconnect** | Dedicated or Partner high-bandwidth network connection between BMS and GCP VPC |
| **Jump Server** | A hardened bastion host (GCE VM) used to SSH into BMS servers from the internet |
| **Private Google Access** | Allows BMS servers to reach Google APIs (GCS, Secret Manager, etc.) without public IPs |
| **iSCSI** | Internet Small Computer Systems Interface — protocol used to attach BMS block storage (LUNs) |
| **MPIO** | Multipath I/O — Linux configuration for redundant iSCSI paths to storage |
| **Hyperthreading** | Intel SMT technology presenting 2 logical CPUs per physical core (configurable on BMS) |

### BMS vs. Other GCP Compute Options

| Feature | Bare Metal Solution | GCE (VMs) | GKE | VMware Engine |
|---|---|---|---|---|
| **Hardware model** | Dedicated physical servers | Shared (multi-tenant VMs) | Shared (containerized) | Dedicated VMware cluster |
| **Hypervisor** | None | KVM | KVM (nodes) | VMware ESXi |
| **Oracle licensing** | Favorable (physical cores) | Expensive (all vCPUs count) | Not recommended | Supported |
| **SAP HANA TDI** | ✅ Certified profiles | ⚠️ Some certified | ❌ | ✅ |
| **Scale to zero** | ❌ Always provisioned | ✅ | ✅ | ❌ |
| **Boot time** | Minutes–hours (provisioning) | Seconds | Seconds | Minutes |
| **Direct hardware access** | ✅ Full (BIOS, NIC, disk) | ❌ | ❌ | Partial |
| **Max RAM** | 12+ TB | 12 TB (m2-ultramem) | Per node | Per node |
| **Network to GCP VPC** | Cloud Interconnect (VLAN) | Direct (same fabric) | Direct | VPN/Interconnect |
| **Pricing model** | Monthly reservation | Per second | Per node-hour | Per node-hour |
| **Best for** | Oracle, SAP HANA, HPC | General purpose workloads | Containerized apps | VMware migration/lift |

> **Note:** BMS is **not** a replacement for GCE. It is a highly specialized offering for workloads with requirements that cannot be met by virtualization. For most workloads, GCE provides better flexibility, cost, and scale.

---

## 2. Architecture & Infrastructure Model

### Physical & Logical Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GCP Region (e.g., us-central1)               │
│                                                                     │
│  ┌──────────────────────────────┐   ┌───────────────────────────┐  │
│  │      GCP Project (Client)    │   │    BMS Pod (Google-mgd)   │  │
│  │                              │   │                           │  │
│  │  ┌─────────────────────┐     │   │  ┌─────────────────────┐  │  │
│  │  │   VPC Network       │     │   │  │  Bare Metal Server  │  │  │
│  │  │  ┌──────────────┐   │     │   │  │  (Oracle / SAP)     │  │  │
│  │  │  │  GCE VMs     │   │     │   │  └────────┬────────────┘  │  │
│  │  │  │  (App Layer) │   │     │   │           │ iSCSI / NFS   │  │
│  │  │  └──────────────┘   │     │   │  ┌────────▼────────────┐  │  │
│  │  └──────────┬──────────┘     │   │  │  Storage Array      │  │  │
│  │             │                │   │  │  (LUNs / NFS)       │  │  │
│  │             │ VLAN Attachment│   │  └─────────────────────┘  │  │
│  │             │ (Interconnect) │   │                           │  │
│  └─────────────┼────────────────┘   └───────────┬───────────────┘  │
│                │                                │                   │
│                └────────── VLAN (802.1Q) ───────┘                   │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              Cloud Services (reachable via PGA)               │  │
│  │   Cloud Storage │ Secret Manager │ Cloud Logging │ Monitoring │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Regional Availability

BMS is available in select GCP regions. Always verify current availability:

```bash
gcloud bms locations list --project=my-project
```

### BMS Project vs. Client Project Separation

| Aspect | BMS Project (Google-managed) | Client Project (Customer-managed) |
|---|---|---|
| **Owner** | Google | Customer |
| **Contains** | Physical servers, storage arrays, network switches | GCP resources, VPC, IAM, billing |
| **Customer access** | Via BMS API only | Full GCP Console + APIs |
| **Networking** | BMS VLAN fabric | VPC subnets |
| **Billing** | Rolls up to Client Project | Customer's billing account |

### Network Architecture

BMS uses **Cloud Interconnect** (Dedicated or Partner) with **VLAN attachments** to bridge the BMS pod network into the customer's GCP VPC:

```
On-Premises / BMS Pod                    GCP VPC
─────────────────────     Interconnect   ──────────────────────
BMS Server                ─────────────► VLAN Attachment
  │                       (802.1Q VLAN)  Cloud Router
  │ iSCSI                               VPC Subnet
  ▼                                      GCE VMs
Storage Array                            Cloud Load Balancer
```

### Storage Architecture

| Storage Type | Protocol | Use Case | Performance |
|---|---|---|---|
| **SSD LUN** | iSCSI | Oracle data files, SAP HANA data/log | High IOPS, low latency |
| **HDD LUN** | iSCSI | Oracle archives, backup staging | High capacity, lower cost |
| **NVMe** | Direct-attach | Ultra-low-latency temp storage | Fastest available |
| **NFS Share** | NFS v3/v4 | Shared Oracle homes, SAP transport dirs | Shared access |

### Private Google Access for BMS

BMS servers reach Google APIs (Cloud Storage, Secret Manager, etc.) without public IPs via Private Google Access configured on the VLAN attachment subnet.

---

## 3. Server Types & Hardware Configurations

### BMS Server Profile Categories

| Category | RAM Range | vCPU / Physical Cores | Primary Use Case |
|---|---|---|---|
| **High-Memory** | 384 GB – 12 TB | 56 – 448 cores | SAP HANA scale-up, Oracle large SGA |
| **High-CPU** | 192 GB – 768 GB | 112 – 448 cores | Oracle RAC, HPC, parallel processing |
| **Standard** | 192 GB – 384 GB | 56 – 112 cores | Oracle single-instance, mid-size apps |
| **GPU-enabled** | 384 GB – 768 GB | 112 cores + GPUs | AI/ML inference, HPC simulation |

### Representative Server Profiles

| Profile Name | Physical Cores | RAM | Local Storage | Typical Workload |
|---|---|---|---|---|
| `o2-highcpu-112` | 112 | 384 GB | 2× 960 GB SSD | Oracle RAC node |
| `o2-standard-32` | 32 | 192 GB | 2× 480 GB SSD | Oracle single instance |
| `o2-highmem-224` | 224 | 3 TB | 2× 960 GB SSD | Oracle Data Warehouse |
| `s4-highmem-192` | 192 | 6 TB | 2× 960 GB SSD | SAP HANA scale-up (6 TB) |
| `s4-highmem-448` | 448 | 12 TB | 4× 960 GB SSD | SAP HANA max scale-up |
| `n2-highmem-128` | 128 | 864 GB | 2× 960 GB SSD | General enterprise |

> **Note:** Profile names and availability change. Always verify current profiles at [cloud.google.com/bare-metal/docs/bms-planning](https://cloud.google.com/bare-metal/docs/bms-planning) or by running `gcloud bms instances list --location=REGION`.

### Hardware Selection Guide

| Workload | Key Requirement | Recommended Profile Type | Notes |
|---|---|---|---|
| **Oracle RAC** | Many physical cores, fast inter-node network | High-CPU | Use InfiniBand or 25GbE bonded NIC for RAC interconnect |
| **Oracle single instance** | Large SGA memory | High-Memory | Over-provision RAM for buffer cache |
| **Oracle Data Guard** | Matching primary spec | Same as primary | Standby must match primary exactly |
| **SAP HANA scale-up** | RAM ≥ database size | High-Memory | Choose certified SAP HANA TDI profile |
| **SAP HANA scale-out** | Distributed RAM across nodes | Multiple Standard | All nodes same profile; shared NFS for /hana/shared |
| **HPC** | Core count + low-latency network | High-CPU | Consider InfiniBand NIC option |

### Hardware Configuration Options

```
Hyperthreading:
  Enabled  → 2 logical CPUs per physical core (default for most workloads)
  Disabled → 1:1 logical:physical (Oracle licensing optimization)

NIC Bonding:
  Active-passive (802.3ad LACP) → Redundancy
  LACP bonding                  → Redundancy + bandwidth aggregation

RAID:
  RAID 1   → Boot disk mirroring (default)
  RAID 10  → Data disks: performance + redundancy
  (LUNs from storage array are RAID-protected at array level)
```

> **Note:** Disabling hyperthreading is a common Oracle licensing optimization — it reduces the logical CPU count that Oracle Database Enterprise Edition must be licensed for. Confirm with your Oracle licensing team before disabling.

---

## 4. Networking Deep Dive

### VLAN Structure

Each BMS pod uses VLANs to segment traffic:

| VLAN Type | Purpose | Routed to GCP VPC? |
|---|---|---|
| **Client network** | Primary workload traffic; connects BMS to GCP VPC | ✅ Yes (via VLAN attachment) |
| **Private network** | BMS-to-BMS traffic within the pod (e.g., Oracle RAC interconnect, SAP HANA HSR) | ❌ Pod-internal only |
| **OOB (Out-of-Band)** | BMC/IPMI access for hardware management | ❌ Google-managed only |
| **Storage network** | iSCSI/NFS traffic to the storage array | ❌ Pod-internal only |

### Interconnect Types

| Type | Bandwidth | Latency | SLA | Use Case |
|---|---|---|---|---|
| **Dedicated Interconnect** | 10 Gbps – 200 Gbps | ~1ms | 99.99% (redundant) | Production; high-throughput |
| **Partner Interconnect** | 50 Mbps – 10 Gbps | Slightly higher | 99.9% – 99.99% | Smaller bandwidth; access via partner |

```bash
# List VLAN attachments in a region
gcloud compute interconnects attachments list \
  --region=us-central1 \
  --project=my-project

# Describe a specific VLAN attachment
gcloud compute interconnects attachments describe my-vlan-attachment \
  --region=us-central1
```

### IP Address Management

BMS does **not** use DHCP — all IP addresses are statically assigned by Google during provisioning and must be managed manually or via the BMS API.

```bash
# View assigned IP addresses for a BMS instance
gcloud bms instances describe my-bms-server \
  --location=us-central1 \
  --project=my-project \
  --format="yaml(networks)"
```

### DNS Configuration

BMS servers do not automatically register in Cloud DNS. Configure manually:

```bash
# /etc/resolv.conf on BMS server — use GCP's internal DNS resolver
nameserver 169.254.169.254   # Metadata server (also provides DNS on GCP)
# Or use your on-premises / Cloud DNS resolver IP
search my-project.internal c.my-project.internal google.internal
```

```bash
# Add a DNS A record for a BMS server in Cloud DNS
gcloud dns record-sets create bms-oracle-01.internal.mycompany.com. \
  --type=A \
  --ttl=300 \
  --rrdatas=10.100.0.5 \
  --zone=internal-zone \
  --project=my-project
```

### Firewall Considerations

> **Note:** BMS servers have **no GCP VPC firewall**. All network security must be enforced at the **OS level** using `iptables` (RHEL/OEL 7) or `firewalld` (RHEL/OEL 8+). There is no equivalent of GCP security groups for BMS servers.

```bash
# firewalld — allow Oracle listener port from GCP VPC subnet
firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  source address="10.0.0.0/8"
  port protocol="tcp" port="1521"
  accept'
firewall-cmd --reload

# iptables — allow SSH from jump server only
iptables -A INPUT -p tcp --dport 22 -s 10.100.0.10 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```

### Jump Server / Bastion Pattern

```
Internet ──► Cloud NAT ──► GCE Jump Server ──► BMS Server
                          (in GCP VPC)       (SSH key auth only)
                          10.0.0.10          10.100.0.5
```

```bash
# SSH to BMS via jump server (ProxyJump)
ssh -J jump-user@<JUMP_SERVER_EXTERNAL_IP> \
    bmsadmin@10.100.0.5

# Or configure in ~/.ssh/config
# Host bms-oracle-01
#   HostName 10.100.0.5
#   User bmsadmin
#   ProxyJump jump-user@<JUMP_SERVER_IP>
#   IdentityFile ~/.ssh/bms_rsa
```

### Routing Between BMS and GCP VPC

```bash
# On BMS server — add static route to GCP VPC subnet via gateway
ip route add 10.0.0.0/8 via 10.100.0.1

# Make persistent (RHEL/OEL — /etc/sysconfig/network-scripts/)
cat >> /etc/sysconfig/network-scripts/route-bond0 << 'EOF'
10.0.0.0/8 via 10.100.0.1
EOF
```

---

## 5. Storage Architecture

### LUN Provisioning and iSCSI Attachment

LUNs are provisioned via the GCP Console or BMS API and presented to BMS servers via iSCSI over the dedicated storage network.

```bash
# Step 1: Discover iSCSI targets (run on BMS server after LUN is provisioned)
iscsiadm -m discovery -t sendtargets -p <STORAGE_IP>:3260

# Step 2: Log in to all discovered targets
iscsiadm -m node --loginall=all

# Step 3: Verify iSCSI sessions are established
iscsiadm -m session

# Step 4: Rescan SCSI bus to detect new LUNs
rescan-scsi-bus.sh
# Or manually:
echo "- - -" > /sys/class/scsi_host/host0/scan

# Step 5: Verify disks are visible
lsblk
fdisk -l | grep "Disk /dev/sd"
```

### Multipath I/O (MPIO) Configuration

MPIO provides redundant paths to storage LUNs — essential for production workloads.

```bash
# Install multipath tools (RHEL/OEL)
yum install -y device-mapper-multipath

# Enable and start multipathd
systemctl enable --now multipathd

# Configure /etc/multipath.conf
cat > /etc/multipath.conf << 'EOF'
defaults {
    user_friendly_names   yes
    path_grouping_policy  multibus
    failback              immediate
    no_path_retry         fail
}

blacklist {
    devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
    devnode "^hd[a-z]"
}

devices {
    device {
        vendor                "PURE"
        product               "FlashArray"
        path_selector         "queue-length 0"
        path_grouping_policy  multibus
        path_checker          tur
        fast_io_fail_tmo      10
        features              "0"
        hardware_handler      "0"
        rr_weight             uniform
        rr_min_io             1
    }
}
EOF

# Reload multipath configuration
multipath -r

# View multipath topology
multipath -ll
```

### NFS Share Mounting

```bash
# Install NFS utilities
yum install -y nfs-utils

# Create mount point
mkdir -p /hana/shared

# Mount NFS share (add to /etc/fstab for persistence)
mount -t nfs -o rw,hard,intr,rsize=1048576,wsize=1048576,timeo=600 \
  <NFS_SERVER_IP>:/vol/hana-shared /hana/shared

# Persistent mount in /etc/fstab
echo "<NFS_SERVER_IP>:/vol/hana-shared /hana/shared nfs rw,hard,intr,rsize=1048576,wsize=1048576,timeo=600 0 0" \
  >> /etc/fstab

# Verify mount
df -h /hana/shared
mount | grep nfs
```

### Oracle ASM with BMS LUNs

```bash
# After MPIO is configured, identify multipath device names
ls -la /dev/mapper/ | grep mpath

# Create udev rules for consistent ASM disk naming
cat > /etc/udev/rules.d/99-oracle-asmdevices.rules << 'EOF'
KERNEL=="dm-*", ENV{DM_NAME}=="mpatha", SYMLINK+="oracleasm/asm-disk1", \
  OWNER="grid", GROUP="asmadmin", MODE="0660"
KERNEL=="dm-*", ENV{DM_NAME}=="mpathb", SYMLINK+="oracleasm/asm-disk2", \
  OWNER="grid", GROUP="asmadmin", MODE="0660"
EOF

udevadm control --reload-rules && udevadm trigger
```

### Expanding a LUN

LUN expansion is initiated via the GCP Console or BMS API, then recognized at the OS level:

```bash
# Step 1: Expand the LUN via BMS API (see Section 14)
# Step 2: Rescan iSCSI sessions to detect new size
iscsiadm -m session --rescan

# Step 3: Rescan SCSI devices
echo 1 > /sys/block/sdb/device/rescan

# Step 4: If using multipath, reload
multipath -r

# Step 5: Resize the filesystem (ext4 example)
resize2fs /dev/mapper/mpatha

# Step 5 alt: For Oracle ASM, use ASMCMD to resize disk group
# asmcmd> ALTER DISKGROUP DATA RESIZE ALL MAXSIZE UNLIMITED;
```

### Storage Performance Tuning

```bash
# Set I/O scheduler to 'deadline' for iSCSI block devices (RHEL 7)
echo deadline > /sys/block/sdb/queue/scheduler

# Persistent via udev rule
cat > /etc/udev/rules.d/60-ioscheduler.rules << 'EOF'
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", \
  ATTR{queue/scheduler}="deadline"
EOF

# For Oracle workloads: disable atime on data filesystems
mount -o remount,noatime,nodiratime /oradata

# Increase read-ahead for HDD LUNs
blockdev --setra 4096 /dev/sdb
```

---

## 6. Provisioning & Lifecycle Management

### Provisioning Lifecycle

```
Step 1: Order (GCP Console / API / Terraform)
   ├── Specify: region, pod, server profile, OS image
   ├── Configure: VLANs, LUNs, NFS shares, SSH keys
   └── Submit provisioning request

Step 2: Fulfillment (Google-managed, ~1-5 business days)
   ├── Physical hardware racked and cabled
   ├── Network configured (VLANs, routing)
   └── Storage provisioned and attached

Step 3: OS Imaging (automated, ~1-2 hours)
   ├── Selected OS image applied
   ├── SSH keys injected
   └── Initial network configuration applied

Step 4: Handoff
   ├── BMS server accessible via SSH from jump server
   ├── Customer confirms connectivity
   └── Customer takes over configuration
```

> **Note:** BMS provisioning is **not instant**. Plan for 3–7 business days for new servers. Reprovisioning (reimage) of existing servers takes 1–4 hours.

### Starting and Stopping Servers

```bash
# Stop a BMS server (graceful OS shutdown must happen first!)
# 1. SSH in and shut down the OS
ssh bmsadmin@10.100.0.5 "sudo shutdown -h now"

# 2. Then stop via BMS API (prevents auto-restart)
gcloud bms instances stop my-bms-server \
  --location=us-central1 \
  --project=my-project

# Start a stopped BMS server
gcloud bms instances start my-bms-server \
  --location=us-central1 \
  --project=my-project

# Check server state
gcloud bms instances describe my-bms-server \
  --location=us-central1 \
  --project=my-project \
  --format="yaml(state)"
```

### Updating Server Metadata

```bash
# Add SSH keys to a running BMS server
gcloud bms instances patch my-bms-server \
  --location=us-central1 \
  --project=my-project \
  --ssh-keys="bmsadmin:$(cat ~/.ssh/id_rsa.pub)"

# Update OS image (requires reimage — destructive!)
gcloud bms instances reimage my-bms-server \
  --location=us-central1 \
  --project=my-project \
  --os-image=projects/my-project/global/images/rhel-8-custom
```

### BMS REST API (Direct)

```bash
# Get BMS server details via REST API
curl -X GET \
  "https://baremetalsolution.googleapis.com/v2/projects/my-project/locations/us-central1/instances/my-bms-server" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json"

# List all BMS instances
curl -X GET \
  "https://baremetalsolution.googleapis.com/v2/projects/my-project/locations/us-central1/instances" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)"
```

---

## 7. Operating System & Software Management

### Supported OS Images

| OS | Versions Supported | Oracle Certified | SAP Certified |
|---|---|---|---|
| **RHEL** (Red Hat Enterprise Linux) | 7.x, 8.x, 9.x | ✅ | ✅ |
| **OEL** (Oracle Enterprise Linux) | 7.x, 8.x | ✅ | ⚠️ Limited |
| **SLES** (SUSE Linux Enterprise Server) | 12 SP5, 15 SP3+ | ✅ | ✅ |
| **Windows Server** | 2019, 2022 | ✅ (limited) | ❌ |
| **Custom image** | Customer-provided | Customer responsibility | Customer responsibility |

> **Note:** Google provides base OS images. The customer is responsible for registering with Red Hat / SUSE for updates and support. Google does **not** provide OS-level support — only hardware and network support.

### SSH Key Management

```bash
# BMS uses OS-level SSH key management — keys are in /home/USER/.ssh/authorized_keys

# Add SSH key for a new admin user
useradd -m -s /bin/bash bmsadmin
mkdir -p /home/bmsadmin/.ssh
echo "ssh-rsa AAAA...your-key..." >> /home/bmsadmin/.ssh/authorized_keys
chmod 700 /home/bmsadmin/.ssh
chmod 600 /home/bmsadmin/.ssh/authorized_keys
chown -R bmsadmin:bmsadmin /home/bmsadmin/.ssh

# Inject SSH key via BMS API at provision time (also via Console)
gcloud bms ssh-keys create my-key \
  --public-key="$(cat ~/.ssh/id_rsa.pub)" \
  --project=my-project

# List provisioned SSH keys
gcloud bms ssh-keys list --project=my-project
```

### OS Patching Strategy

Since Google does not patch BMS OS, the customer is fully responsible:

```bash
# RHEL/OEL — register with subscription manager (required for yum updates)
subscription-manager register \
  --username=<RHN_USER> --password=<RHN_PASS> \
  --auto-attach

# Update all packages
yum update -y

# Apply security patches only
yum update --security -y

# Check for available updates
yum check-update

# SLES — use zypper
zypper refresh && zypper update -y
```

### Oracle Database Installation Prerequisites

```bash
# 1. Install required OS packages
yum install -y oracle-database-preinstall-19c   # OEL/RHEL — sets all Oracle prereqs

# 2. Configure kernel parameters (if not using preinstall RPM)
cat >> /etc/sysctl.conf << 'EOF'
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 2097152
kernel.shmmax = 4294967295
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.core.rmem_max = 4194304
net.core.wmem_max = 1048576
EOF
sysctl -p

# 3. Create Oracle user and groups
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 asmadmin
useradd -u 54321 -g oinstall -G dba,asmadmin oracle

# 4. Create Oracle directories
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1
chown -R oracle:oinstall /u01
chmod -R 775 /u01

# 5. Continue with Oracle installer (runInstaller)
su - oracle
./runInstaller -silent -responseFile /tmp/db_install.rsp
```

### SAP HANA Installation Prerequisites

```bash
# Check BMS hardware compatibility (SAP Note 2235581)
# Verify RAM, CPU, storage layout before installation

# Configure OS for SAP HANA (RHEL 8 example)
# 1. Apply SAP HANA specific RHEL tuning profile
tuned-adm profile sap-hana

# 2. Disable transparent huge pages
cat >> /etc/rc.d/rc.local << 'EOF'
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
EOF
chmod +x /etc/rc.d/rc.local

# 3. Disable NUMA balancing
echo 0 > /proc/sys/kernel/numa_balancing

# 4. Set swappiness
sysctl vm.swappiness=10

# 5. Create SAP HANA filesystem structure on BMS LUNs
mkdir -p /hana/data /hana/log /hana/shared /hana/backup /usr/sap
# Mount each on separate LUNs (see Section 10 for layout details)
```

---

## 8. Connecting BMS to GCP Services

### Cloud Storage Access (via Private Google Access)

```bash
# Step 1: Ensure Private Google Access is enabled on the VPC subnet
gcloud compute networks subnets update my-bms-subnet \
  --enable-private-ip-google-access \
  --region=us-central1

# Step 2: Add route for Google APIs on BMS server
ip route add 199.36.153.8/30 via <VLAN_GATEWAY>    # restricted.googleapis.com
ip route add 199.36.153.4/30 via <VLAN_GATEWAY>    # private.googleapis.com

# Step 3: Configure DNS to resolve Google APIs to private IPs
# /etc/hosts entries (or Cloud DNS private zone):
echo "199.36.153.8  storage.googleapis.com" >> /etc/hosts
echo "199.36.153.8  secretmanager.googleapis.com" >> /etc/hosts

# Step 4: Install Google Cloud SDK on BMS
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init

# Step 5: Test Cloud Storage access
gsutil ls gs://my-backup-bucket/
gsutil cp /backup/oracle.dmp gs://my-backup-bucket/oracle/
```

### Secret Manager Integration

```python
# Access secrets from BMS server (Python)
# Install: pip install google-cloud-secret-manager

from google.cloud import secretmanager

def get_secret(project_id: str, secret_id: str) -> str:
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("utf-8")

# Usage — retrieve Oracle SYS password
sys_password = get_secret("my-project", "oracle-sys-password")
```

```bash
# Shell script usage — retrieve secret via gcloud
DB_PASSWORD=$(gcloud secrets versions access latest \
  --secret=oracle-sys-password \
  --project=my-project)
```

### Ops Agent Installation (Metrics & Logs to Cloud Monitoring)

```bash
# Install Google Cloud Ops Agent on BMS server (RHEL/OEL/SLES)
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install

# Enable and start
systemctl enable --now google-cloud-ops-agent

# Verify metrics are flowing
gcloud monitoring metrics list \
  --filter="metric.type=starts_with('agent.googleapis.com')" \
  --project=my-project | head -20
```

### Connecting to Cloud SQL from BMS

```bash
# Install Cloud SQL Auth Proxy on BMS server
curl -o cloud-sql-proxy \
  https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.0.0/cloud-sql-proxy.linux.amd64
chmod +x cloud-sql-proxy

# Run the proxy (connects via Cloud SQL API — no VPC tunnel needed)
./cloud-sql-proxy my-project:us-central1:my-sql-instance &

# Connect to PostgreSQL via proxy
psql -h 127.0.0.1 -U myuser -d mydb
```

### Cloud VPN as Backup Path

```bash
# Create a Cloud VPN tunnel as backup for Cloud Interconnect
gcloud compute vpn-gateways create my-vpn-gateway \
  --network=my-vpc \
  --region=us-central1

gcloud compute vpn-tunnels create bms-backup-tunnel \
  --peer-address=<BMS_SIDE_VPN_IP> \
  --vpn-gateway=my-vpn-gateway \
  --ike-version=2 \
  --shared-secret=my-secret \
  --router=my-cloud-router \
  --region=us-central1
```

---

## 9. Oracle on Bare Metal Solution

### Oracle Architecture Options on BMS

| Architecture | Servers Needed | HA Level | Licensing |
|---|---|---|---|
| **Single Instance** | 1 BMS server | Low (no automatic failover) | 1× server core count |
| **RAC (Real Application Clusters)** | 2+ BMS servers | High (active-active) | All nodes' core counts |
| **Data Guard (Primary + Standby)** | 2 BMS servers (separate pods/regions) | High (active-passive) | Primary only (standby is passive) |
| **RAC + Data Guard** | 4+ servers | Very High | RAC nodes only |

### Oracle License Considerations on BMS

```
BMS Physical Core Count:    56 cores, hyperthreading DISABLED
Oracle EE License Required: 56 cores × 0.5 (Intel factor) = 28 processor licenses

BMS Physical Core Count:    56 cores, hyperthreading ENABLED
Oracle EE License Required: 112 vCPUs × 0.5 (Intel factor) = 56 processor licenses
```

> **Note:** Disabling hyperthreading on BMS is the **single most impactful Oracle cost optimization** available. It halves the number of Oracle Database licenses required. Confirm with Oracle licensing before changing.

### Oracle ASM Disk Group Setup

```bash
# After LUNs are visible via multipath (see Section 5):

# Create ASM disk groups using ASMCA or sqlplus as grid user
su - grid

# Create DATA disk group (for Oracle data files)
asmca -silent -createDiskGroup \
  -diskGroupName DATA \
  -diskList /dev/oracleasm/asm-disk1,/dev/oracleasm/asm-disk2 \
  -redundancy EXTERNAL \
  -au_size 4

# Create RECO disk group (for redo logs and archive logs)
asmca -silent -createDiskGroup \
  -diskGroupName RECO \
  -diskList /dev/oracleasm/asm-disk3 \
  -redundancy EXTERNAL \
  -au_size 4

# Verify disk groups
su - oracle
sqlplus / as sysdba
```

```sql
-- Verify ASM disk groups from Oracle
SELECT name, state, total_mb, free_mb, type
FROM v$asm_diskgroup;

-- Check ASM disks
SELECT path, name, group_number, disk_number, state, total_mb
FROM v$asm_disk
ORDER BY group_number, disk_number;
```

### Oracle Data Guard Configuration (HA/DR)

```sql
-- On PRIMARY: enable archivelog and force logging
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE FORCE LOGGING;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;

-- Configure log archive destinations
ALTER SYSTEM SET log_archive_dest_1='LOCATION=+RECO VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=ORCL_PRIMARY';
ALTER SYSTEM SET log_archive_dest_2='SERVICE=ORCL_STANDBY LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=ORCL_STANDBY';
ALTER SYSTEM SET log_archive_dest_state_2=ENABLE;
ALTER SYSTEM SET fal_server=ORCL_STANDBY;
ALTER SYSTEM SET db_unique_name=ORCL_PRIMARY SCOPE=SPFILE;
ALTER SYSTEM SET log_archive_config='DG_CONFIG=(ORCL_PRIMARY,ORCL_STANDBY)';
```

```ini
# tnsnames.ora — Oracle Net configuration (on both primary and standby)
ORCL_PRIMARY =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = bms-oracle-primary)(PORT = 1521))
    (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = ORCL)(UR = A)))

ORCL_STANDBY =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = bms-oracle-standby)(PORT = 1521))
    (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = ORCL)(UR = A)))
```

### Oracle RMAN Backup to Cloud Storage

```bash
# Mount Cloud Storage bucket via gcsfuse on BMS (for RMAN backup target)
yum install -y gcsfuse
mkdir /rman-backup
gcsfuse --implicit-dirs my-oracle-backup-bucket /rman-backup
```

```bash
# RMAN backup script targeting GCS-mounted directory
rman target / << 'EOF'
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
CONFIGURE BACKUP OPTIMIZATION ON;
CONFIGURE COMPRESSION ALGORITHM 'MEDIUM';

BACKUP AS COMPRESSED BACKUPSET
  INCREMENTAL LEVEL 0
  DATABASE FORMAT '/rman-backup/full_%d_%T_%U.bkp'
  INCLUDE CURRENT CONTROLFILE;

BACKUP ARCHIVELOG ALL
  FORMAT '/rman-backup/arch_%d_%T_%U.bkp'
  DELETE INPUT;

CROSSCHECK BACKUP;
DELETE NOPROMPT OBSOLETE;
EOF
```

### Oracle Performance Tuning on BMS

```sql
-- Check and set key Oracle parameters for BMS (large memory servers)
ALTER SYSTEM SET sga_target=<80%_of_RAM>G SCOPE=SPFILE;
ALTER SYSTEM SET pga_aggregate_target=<10%_of_RAM>G SCOPE=SPFILE;
ALTER SYSTEM SET db_cache_size=0 SCOPE=SPFILE;   -- Let SGA_TARGET manage

-- Enable NUMA-aware memory allocation (if hyperthreading disabled)
ALTER SYSTEM SET "_enable_NUMA_support"=TRUE SCOPE=SPFILE;

-- Set parallel degree for large BMS core counts
ALTER SYSTEM SET parallel_max_servers=<CORE_COUNT*2> SCOPE=SPFILE;
ALTER SYSTEM SET cpu_count=<PHYSICAL_CORE_COUNT> SCOPE=SPFILE;
```

---

## 10. SAP on Bare Metal Solution

### SAP HANA Architecture Options on BMS

| Architecture | Description | Min Servers | Max RAM |
|---|---|---|---|
| **Scale-Up** | Single large server; entire HANA DB in RAM | 1 | 12 TB (single BMS) |
| **Scale-Out** | Multiple servers; DB distributed across nodes | 2–16 | 120+ TB |
| **Scale-Out + Standby** | Scale-out with a warm standby node | 3–17 | 120+ TB |
| **HSR (System Replication)** | Primary + secondary for HA/DR | 2 | Per primary |

### BMS SAP HANA TDI Certification

BMS profiles that carry the SAP HANA TDI (Tailored Data Center Integration) certification:
- Must pass SAP's storage, network, CPU, and memory validation tests
- Google certifies specific BMS profiles — check [SAP Note 2235581](https://launchpad.support.sap.com/#/notes/2235581)
- TDI compliance is required for SAP support agreements on customer-managed hardware

### SAP HANA Storage Layout on BMS LUNs

```
Recommended filesystem layout (mount each on a dedicated LUN):

/hana/data     → Data volume     (SSD LUN, 3× HANA RAM size minimum)
/hana/log      → Log volume      (SSD LUN, 512 GB – 1× HANA RAM, ultra-low latency)
/hana/shared   → Shared volume   (NFS share for scale-out; SSD LUN for scale-up)
/hana/backup   → Backup volume   (HDD LUN or NFS; 2-3× /hana/data size)
/usr/sap       → SAP binaries    (SSD LUN, 100–200 GB)
```

```bash
# Format and mount /hana/data (XFS recommended for SAP HANA)
mkfs.xfs -K /dev/mapper/mpatha
mkdir -p /hana/data
mount -t xfs -o relatime,inode64 /dev/mapper/mpatha /hana/data

# /etc/fstab entry
echo "/dev/mapper/mpatha /hana/data xfs relatime,inode64 0 2" >> /etc/fstab

# Set permissions for HANA admin user
chown -R <sid>adm:sapsys /hana/data
chmod 755 /hana/data
```

### SAP HANA System Replication (HSR) Configuration

```bash
# On PRIMARY node — enable system replication
# Run as <SID>adm user
hdbnsutil -sr_enable --name=PRIMARY

# Backup primary before configuring secondary
hdbsql -i <INSTANCE_NUM> -u SYSTEM -p <PASSWORD> \
  "BACKUP DATA USING FILE ('/hana/backup/INITIAL')"

# On SECONDARY node — register with primary
hdbnsutil -sr_register \
  --name=SECONDARY \
  --remoteHost=bms-hana-primary \
  --remoteInstance=<INSTANCE_NUM> \
  --replicationMode=sync \
  --operationMode=logreplay

# Check replication status (run as <SID>adm on primary)
hdbnsutil -sr_state
```

### Network Topology for SAP on GCP + BMS

```
Internet
    │
    ▼
GCP Load Balancer
    │
    ▼
GCE VMs (SAP Application Layer — ABAP/Java/Web Dispatcher)
    │  VPC subnet 10.0.0.0/24
    │  ← VLAN Attachment / Cloud Interconnect →
    │  BMS pod network 10.100.0.0/24
    ▼
BMS Servers (SAP HANA Database Layer)
    │
    ▼
BMS Storage Array (SAP HANA LUNs)
```

```ini
# SAP HANA global.ini — network configuration for replication
[communication]
listeninterface = .global

[system_replication_communication]
listeninterface = .global

[system_replication_hostname_resolution]
bms-hana-primary   = 10.100.0.5
bms-hana-secondary = 10.100.0.6
```

---

## 11. IAM & Security

### BMS-Specific IAM Roles

| Role | Description | Grants Access To |
|---|---|---|
| `roles/baremetalsolution.admin` | Full BMS resource management | All BMS API operations |
| `roles/baremetalsolution.editor` | Create/modify BMS resources | Instances, volumes, networks |
| `roles/baremetalsolution.viewer` | Read-only view of BMS resources | List and describe operations |
| `roles/baremetalsolution.operationsViewer` | View long-running operations | Operation status only |

```bash
# Grant BMS admin role to an infrastructure team
gcloud projects add-iam-policy-binding my-project \
  --member=group:infra-team@mycompany.com \
  --role=roles/baremetalsolution.admin

# Grant viewer role to a monitoring team
gcloud projects add-iam-policy-binding my-project \
  --member=serviceAccount:monitoring-sa@my-project.iam.gserviceaccount.com \
  --role=roles/baremetalsolution.viewer
```

### Access Control Model

```
BMS API Access (IAM)              OS-Level Access (SSH)
────────────────────              ─────────────────────────
roles/baremetalsolution.*    →    SSH keys in authorized_keys
Controls: provision, list,        Controls: shell access, sudo
  update, delete servers          Managed by: customer

Note: IAM does NOT grant SSH.     Note: OS access is INDEPENDENT
IAM controls the GCP API only.    of BMS API / IAM permissions.
```

### OS-Level Security Hardening

```bash
# Disable root SSH login
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config

# Disable password authentication (keys only)
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^#\?ChallengeResponseAuthentication.*/ChallengeResponseAuthentication no/' /etc/ssh/sshd_config
systemctl reload sshd

# Configure sudo for Oracle/SAP users (limited commands only)
cat > /etc/sudoers.d/oracle << 'EOF'
oracle ALL=(root) NOPASSWD: /sbin/multipath, /usr/sbin/iscsiadm
grid   ALL=(root) NOPASSWD: /usr/sbin/oracleasm
EOF

# Enable auditd for comprehensive audit logging
systemctl enable --now auditd
auditctl -w /etc/passwd -p wa -k passwd_changes
auditctl -w /etc/ssh/sshd_config -p wa -k sshd_config

# Configure firewalld (RHEL 8+)
systemctl enable --now firewalld
firewall-cmd --set-default-zone=drop
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" service name="ssh" accept'
firewall-cmd --reload
```

### Encryption at Rest

> **Note:** BMS storage (LUNs and NFS shares) is encrypted at rest using **Google-managed encryption keys** by default. This is applied at the storage array level and is transparent to the OS. Customer-managed encryption keys (CMEK) for BMS storage are not currently supported — use OS-level encryption (dm-crypt/LUKS) if CMEK is required.

### Audit Logging for BMS API Calls

BMS API calls are automatically captured in **Cloud Audit Logs**:

```bash
# View BMS admin activity audit logs
gcloud logging read \
  'resource.type="baremetalsolution.googleapis.com/Instance"
   AND logName="projects/my-project/logs/cloudaudit.googleapis.com%2Factivity"' \
  --project=my-project \
  --limit=50 \
  --format=json
```

---

## 12. Monitoring, Logging & Observability

### Ops Agent Configuration on BMS

```yaml
# /etc/google-cloud-ops-agent/config.yaml
# Collect system metrics and application logs

metrics:
  receivers:
    hostmetrics:
      type: hostmetrics
      collection_interval: 60s
    oracle_db:
      type: oracle_db
      connection_string: "oracle/oracle@localhost:1521/ORCL"
      collection_interval: 300s
  service:
    pipelines:
      default_pipeline:
        receivers: [hostmetrics]
      oracle_pipeline:
        receivers: [oracle_db]

logging:
  receivers:
    oracle_alert:
      type: files
      include_paths:
        - /u01/app/oracle/diag/rdbms/*/*/trace/alert_*.log
    syslog:
      type: files
      include_paths:
        - /var/log/messages
        - /var/log/syslog
  service:
    pipelines:
      default_pipeline:
        receivers: [syslog, oracle_alert]
```

```bash
# Restart Ops Agent after config change
systemctl restart google-cloud-ops-agent

# Verify agent is running and collecting
systemctl status google-cloud-ops-agent
journalctl -u google-cloud-ops-agent -f
```

### Default Metrics Collected by Ops Agent

| Metric Category | Metric Examples |
|---|---|
| **CPU** | `agent.googleapis.com/cpu/utilization`, `cpu/load_1m` |
| **Memory** | `agent.googleapis.com/memory/percent_used`, `memory/bytes_used` |
| **Disk I/O** | `agent.googleapis.com/disk/read_bytes_count`, `disk/io_time` |
| **Network** | `agent.googleapis.com/interface/traffic`, `interface/errors` |
| **Filesystem** | `agent.googleapis.com/disk/percent_used` |
| **Process** | `agent.googleapis.com/processes/count` |

### Useful Cloud Logging Filters for BMS

```bash
# All logs from a specific BMS server (by hostname)
resource.type="gce_instance"
labels."compute.googleapis.com/resource_name"="bms-oracle-01"

# Oracle alert log errors
resource.type="gce_instance"
logName="projects/my-project/logs/oracle_alert"
severity>=ERROR

# BMS API operations (provisioning, start, stop)
resource.type="baremetalsolution.googleapis.com/Instance"
logName="projects/my-project/logs/cloudaudit.googleapis.com%2Factivity"

# SSH login events from BMS servers
resource.type="gce_instance"
jsonPayload.message:"Accepted publickey"

# Kernel errors (disk/network issues)
resource.type="gce_instance"
jsonPayload.MESSAGE:"kernel:"
severity=CRITICAL OR severity=ALERT OR severity=EMERGENCY
```

### Setting Up Alerting Policies

```bash
# Create CPU utilization alert for BMS servers
gcloud monitoring policies create \
  --policy-from-file=cpu-alert-policy.yaml \
  --project=my-project
```

```yaml
# cpu-alert-policy.yaml
displayName: "BMS Server High CPU"
conditions:
  - displayName: "CPU utilization > 90% for 5 minutes"
    conditionThreshold:
      filter: 'resource.type="gce_instance" AND metric.type="agent.googleapis.com/cpu/utilization"'
      comparison: COMPARISON_GT
      thresholdValue: 0.9
      duration: 300s
      aggregations:
        - alignmentPeriod: 60s
          perSeriesAligner: ALIGN_MEAN
notificationChannels:
  - projects/my-project/notificationChannels/my-email-channel
```

---

## 13. Backup, DR & High Availability

### LUN Snapshot Strategy

```bash
# Create a LUN snapshot via BMS API
curl -X POST \
  "https://baremetalsolution.googleapis.com/v2/projects/my-project/locations/us-central1/volumes/my-lun/snapshots" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "daily-snapshot-20240101",
    "description": "Daily snapshot before patching"
  }'

# List snapshots for a volume
gcloud bms volumes snapshots list \
  --volume=my-lun \
  --location=us-central1 \
  --project=my-project

# Restore from snapshot (creates a new volume — original is unchanged)
gcloud bms volumes snapshots restore my-snapshot \
  --volume=my-lun \
  --location=us-central1 \
  --project=my-project
```

> **Note:** LUN snapshots are **crash-consistent**, not application-consistent. For Oracle and SAP HANA, always quiesce the application (Oracle hot backup mode or HANA freeze) before taking a snapshot, or use application-level backup tools (RMAN / HANA Backint) for guaranteed consistency.

### Oracle HA/DR Architecture

| Scenario | Technology | RTO | RPO | Notes |
|---|---|---|---|---|
| **Local HA** | Oracle RAC | Near-zero | Zero | Active-active; requires shared storage |
| **Regional DR** | Data Guard sync | Minutes | Zero | Synchronous redo shipping |
| **Regional DR** | Data Guard async | 30–60 min | Minutes | Asynchronous; lower impact on primary |
| **Multi-region DR** | Data Guard Far Sync | Hours | Minutes–hours | Far Sync instance proxies redo |

### SAP HANA HA/DR Architecture

| Scenario | Technology | RTO | RPO | Notes |
|---|---|---|---|---|
| **Same-pod HA** | HSR sync (tier 1) | < 1 min | Zero | Automatic failover with Pacemaker |
| **Cross-pod DR** | HSR async (tier 2) | 10–30 min | Minutes | Manual or semi-auto failover |
| **Cross-region DR** | HANA backup/restore | Hours | Hours | Restore from Cloud Storage |

### Manual Failover Procedure (Oracle Data Guard)

```bash
# Step 1: On STANDBY — verify it's in sync with primary
sqlplus / as sysdba
```

```sql
-- Check Data Guard status on standby
SELECT name, db_unique_name, open_mode, protection_mode, database_role
FROM v$database;

SELECT process, status, sequence#, block#
FROM v$managed_standby
WHERE process LIKE 'MRP%';

-- Perform switchover (planned failover — primary is still up)
ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN;

-- OR perform failover (primary is DOWN)
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE FINISH;
ALTER DATABASE ACTIVATE STANDBY DATABASE;
```

### RTO/RPO Planning for BMS

| Consideration | Impact | Mitigation |
|---|---|---|
| BMS is **regional only** | No cross-region auto-failover | Manual DR runbook; Data Guard to BMS in another region |
| BMS provisioning takes days | New server cannot be ordered during incident | Pre-provision standby servers |
| LUN snapshots are crash-consistent | Data loss possible | Use app-level backup (RMAN/Backint) |
| Cloud Interconnect failure | BMS isolated from GCP services | Cloud VPN backup path (see Section 8) |

---

## 14. `gcloud` & API Quick Reference

### Instance Commands

| Command | Description |
|---|---|
| `gcloud bms instances list --location=REGION` | List all BMS instances in a region |
| `gcloud bms instances describe NAME --location=REGION` | Show full instance configuration |
| `gcloud bms instances start NAME --location=REGION` | Power on a stopped BMS server |
| `gcloud bms instances stop NAME --location=REGION` | Power off a BMS server |
| `gcloud bms instances reset NAME --location=REGION` | Hard reset (like pressing the reset button) |
| `gcloud bms instances patch NAME --location=REGION [FLAGS]` | Update instance metadata (SSH keys, labels) |

```bash
# List all BMS instances with state and IP addresses
gcloud bms instances list \
  --location=us-central1 \
  --project=my-project \
  --format="table(name, state, networks[0].ipAddress, machineType)"

# Describe instance — get all details including storage and network
gcloud bms instances describe my-bms-server \
  --location=us-central1 \
  --project=my-project \
  --format=yaml
```

### Volume (LUN) Commands

```bash
# List all volumes (LUNs)
gcloud bms volumes list \
  --location=us-central1 \
  --project=my-project

# Describe a specific volume
gcloud bms volumes describe my-lun \
  --location=us-central1 \
  --project=my-project

# Resize a volume (increase size only)
gcloud bms volumes patch my-lun \
  --location=us-central1 \
  --project=my-project \
  --size-gib=2048

# List volume snapshots
gcloud bms volumes snapshots list \
  --volume=my-lun \
  --location=us-central1 \
  --project=my-project
```

### Network Commands

```bash
# List BMS networks
gcloud bms networks list \
  --location=us-central1 \
  --project=my-project

# Describe a BMS network (VLAN config, IP ranges)
gcloud bms networks describe my-bms-network \
  --location=us-central1 \
  --project=my-project

# List VLAN attachments
gcloud bms networks list-network-usage \
  --location=us-central1 \
  --project=my-project
```

### NFS Share Commands

```bash
# List NFS shares
gcloud bms nfs-shares list \
  --location=us-central1 \
  --project=my-project

# Describe an NFS share (get mount path and allowed clients)
gcloud bms nfs-shares describe my-nfs-share \
  --location=us-central1 \
  --project=my-project

# Create an NFS share (provisioning — may take time)
gcloud bms nfs-shares create my-new-nfs \
  --location=us-central1 \
  --project=my-project \
  --size-gib=1024 \
  --storage-type=SSD \
  --allowed-client=allowed-clients-config.json
```

### SSH Key Commands

```bash
# Create/register an SSH public key for BMS provisioning
gcloud bms ssh-keys create my-key \
  --public-key="$(cat ~/.ssh/id_rsa.pub)" \
  --project=my-project

# List registered SSH keys
gcloud bms ssh-keys list --project=my-project

# Delete an SSH key
gcloud bms ssh-keys delete my-key --project=my-project
```

### Operations Commands

```bash
# List long-running BMS operations (provisioning, reimage, etc.)
gcloud bms operations list \
  --location=us-central1 \
  --project=my-project

# Describe an operation to check progress
gcloud bms operations describe OPERATION_ID \
  --location=us-central1 \
  --project=my-project
```

---

## 15. Terraform Snippet

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

variable "bms_ssh_public_key" {
  description = "SSH public key for BMS server access"
  type        = string
  sensitive   = true
}

# ── SSH Key Registration ───────────────────────────────────
resource "google_bare_metal_solution_ssh_key" "bms_key" {
  project    = var.project_id
  name       = "bms-admin-key"
  public_key = var.bms_ssh_public_key
}

# ── BMS Network (VLAN) ────────────────────────────────────
# Note: BMS networks are typically pre-created or provisioned
# together with the server. Reference an existing network:
data "google_bare_metal_solution_network" "bms_network" {
  project  = var.project_id
  name     = "at-1234567-vlan001"
  location = "us-central1"
}

# ── Storage Volume (LUN) ──────────────────────────────────
resource "google_bare_metal_solution_volume" "oracle_data" {
  project     = var.project_id
  location    = "us-central1"
  name        = "oracle-data-vol"
  size_gib    = 2048
  type        = "SSD"
  snapshot_auto_delete_behavior = "NEWEST_FIRST"

  snapshot_schedule_policy {
    name = "daily-snapshot-policy"
  }
}

resource "google_bare_metal_solution_volume" "oracle_reco" {
  project  = var.project_id
  location = "us-central1"
  name     = "oracle-reco-vol"
  size_gib = 1024
  type     = "SSD"
}

# ── NFS Share ─────────────────────────────────────────────
resource "google_bare_metal_solution_nfs_share" "oracle_home" {
  project  = var.project_id
  location = "us-central1"
  name     = "oracle-home-nfs"

  allowed_clients {
    network         = data.google_bare_metal_solution_network.bms_network.id
    allowed_clients_cidr = "10.100.0.0/24"
    mount_permissions    = "READ_WRITE"
    allow_dev            = true
    allow_suid           = true
    no_root_squash       = true
  }
}

# ── BMS Server Instance ───────────────────────────────────
resource "google_bare_metal_solution_instance" "oracle_server" {
  project  = var.project_id
  location = "us-central1"
  name     = "bms-oracle-01"

  # Machine type — confirm availability with Google
  machine_type = "o2-standard-32"

  # OS Image
  os_image = "projects/${var.project_id}/global/images/rhel-8-6-minimal"

  # SSH keys for initial access
  ssh_keys = [google_bare_metal_solution_ssh_key.bms_key.name]

  # Hyperthreading: false = disabled (Oracle license optimization)
  hyperthreading_enabled = false

  # Network attachments
  networks {
    network     = data.google_bare_metal_solution_network.bms_network.id
    type        = "CLIENT"
    network_ip  = "10.100.0.5"
  }

  # LUN attachments
  luns {
    name            = google_bare_metal_solution_volume.oracle_data.name
    boot_volume     = false
    size_gb         = 2048
    state           = "READY"
    storage_type    = "SSD"
  }

  luns {
    name            = google_bare_metal_solution_volume.oracle_reco.name
    boot_volume     = false
    size_gb         = 1024
    state           = "READY"
    storage_type    = "SSD"
  }

  # Labels for cost attribution and operations
  labels = {
    env         = "production"
    workload    = "oracle-db"
    team        = "database"
    cost-center = "finance"
  }

  depends_on = [
    google_bare_metal_solution_ssh_key.bms_key,
    google_bare_metal_solution_volume.oracle_data,
    google_bare_metal_solution_volume.oracle_reco,
  ]
}

# ── Cloud Monitoring Alert for CPU ────────────────────────
resource "google_monitoring_alert_policy" "bms_cpu_alert" {
  project      = var.project_id
  display_name = "BMS Oracle Server High CPU"

  conditions {
    display_name = "CPU > 90% for 5 minutes"
    condition_threshold {
      filter          = "resource.type=\"gce_instance\" AND metric.type=\"agent.googleapis.com/cpu/utilization\""
      comparison      = "COMPARISON_GT"
      threshold_value = 0.9
      duration        = "300s"
      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MEAN"
      }
    }
  }

  notification_channels = []   # Add your channel IDs here
  combiner              = "OR"
}

# ── Outputs ────────────────────────────────────────────────
output "bms_server_name" {
  value       = google_bare_metal_solution_instance.oracle_server.name
  description = "The name of the provisioned BMS server"
}

output "bms_server_id" {
  value = google_bare_metal_solution_instance.oracle_server.id
}

output "nfs_share_mount_point" {
  value       = google_bare_metal_solution_nfs_share.oracle_home.id
  description = "Reference ID of the NFS share"
}
```

```bash
terraform init
terraform plan  -var="project_id=my-project" -var="bms_ssh_public_key=$(cat ~/.ssh/id_rsa.pub)"
terraform apply -var="project_id=my-project" -var="bms_ssh_public_key=$(cat ~/.ssh/id_rsa.pub)"
```

> **Note:** BMS Terraform resources may not be fully idempotent — some changes (machine type, hyperthreading) require reprovisioning. Always test in non-production and review the plan carefully before applying.

---

## 16. Cost Model & Optimization

### BMS Pricing Components

| Component | Billing Model | Notes |
|---|---|---|
| **Server** | Monthly reservation | Fixed monthly fee per server profile; no per-second billing |
| **Storage (SSD LUN)** | Per GiB-month | Billed for provisioned size, not used size |
| **Storage (HDD LUN)** | Per GiB-month | Lower cost per GiB than SSD |
| **NFS Share** | Per GiB-month | SSD or HDD-backed |
| **Cloud Interconnect** | Port + attachment fee | Dedicated: ~$1,700/month per 10G port + per-Gbps attachment |
| **Network egress** | Per GiB | Standard GCP egress rates apply |
| **Snapshots** | Per GiB-month | Only billed for changed data (incremental) |

### BMS vs. GCE Cost Comparison

| Workload | BMS Option | Approx. BMS/month | GCE Equivalent | Approx. GCE/month | Notes |
|---|---|---|---|---|---|
| Oracle DB (56 cores, 384 GB) | o2-standard-32 | ~$8,000 | m2-standard-110 | ~$18,000 | BMS Oracle license is ~50% fewer licenses (HT off) |
| SAP HANA 6TB | s4-highmem-192 | ~$25,000 | m2-megamem-416 | ~$60,000 | SAP certification + memory density |
| Oracle RAC (2×56 core) | 2× o2-standard-32 | ~$16,000 | 2× m2-standard-110 | ~$36,000 | Plus interconnect cost |

> **Note:** The BMS server cost does **not** include Oracle Database licenses. Dedicated BMS servers typically require 50% fewer Oracle licenses than equivalent GCE VM configurations (when hyperthreading is disabled), which can save hundreds of thousands of dollars annually.

### Cost Optimization Strategies

```bash
# 1. Disable hyperthreading to halve Oracle license count
gcloud bms instances patch my-bms-server \
  --location=us-central1 \
  --project=my-project \
  --hyperthreading-enabled=false
# (Requires server restart)

# 2. Right-size storage — identify over-provisioned LUNs
gcloud bms volumes list \
  --location=us-central1 \
  --project=my-project \
  --format="table(name, sizeGib, currentSizeGib)"

# 3. Use HDD LUNs for backup and archive data (vs SSD)
# HDD: ~$0.05/GiB/month vs SSD: ~$0.17/GiB/month

# 4. Delete unused snapshots
gcloud bms volumes snapshots delete old-snapshot \
  --volume=my-lun \
  --location=us-central1 \
  --project=my-project

# 5. Shut down non-production servers on schedule (maintenance windows)
gcloud bms instances stop my-dev-server \
  --location=us-central1 \
  --project=my-project
```

### FinOps Recommendations for BMS

- Use **labels** consistently on all BMS resources for cost allocation: `env`, `workload`, `team`, `cost-center`.
- Set up **Budget Alerts** in Cloud Billing for BMS costs.
- Review **Cloud Billing reports** monthly — filter by `Service: Bare Metal Solution`.
- **Consolidate non-production** environments — run dev/test on GCE VMs when Oracle/SAP workloads can tolerate virtualization.
- Use **Committed Use Discounts** (CUDs) for Cloud Interconnect ports that are in continuous use.
- **Archive cold storage** to Cloud Storage (Nearline/Coldline) instead of HDD LUNs.

---

## 17. Migration to BMS

### Pre-Migration Checklist

```
□ Hardware requirements validated (core count, RAM, storage IOPS)
□ BMS profile selected and certified for workload (SAP TDI if applicable)
□ Network architecture designed (VLAN, Interconnect bandwidth)
□ Storage layout designed (LUN count, size, type per mount point)
□ Oracle/SAP license review completed
□ Firewall/security rules documented
□ DNS entries planned for BMS servers
□ Jump server provisioned in GCP VPC
□ Cloud Interconnect ordered (lead time: 4–10 weeks for Dedicated)
□ Backup strategy defined (RMAN / HANA Backint to Cloud Storage)
□ Rollback plan documented
□ Maintenance window agreed with business stakeholders
```

### Oracle Database Migration (Lift and Shift)

```bash
# Method 1: Oracle Data Pump (logical export — recommended for schema migration)
# On source (on-premises):
expdp system/password@ORCL \
  FULL=Y \
  DIRECTORY=DATA_PUMP_DIR \
  DUMPFILE=full_export_%U.dmp \
  LOGFILE=full_export.log \
  PARALLEL=8 \
  COMPRESSION=ALL

# Transfer dump files to GCS
gsutil -m cp /oracle/dpdump/full_export*.dmp gs://my-migration-bucket/oracle/

# On target (BMS) — copy from GCS and import
gsutil -m cp gs://my-migration-bucket/oracle/full_export*.dmp /oracle/dpdump/
impdp system/password@ORCL_NEW \
  FULL=Y \
  DIRECTORY=DATA_PUMP_DIR \
  DUMPFILE=full_export_%U.dmp \
  LOGFILE=full_import.log \
  PARALLEL=8
```

```bash
# Method 2: Oracle RMAN Duplicate (for large databases — minimal downtime)
# On target BMS — duplicate from active primary (online copy)
rman target sys/password@SOURCE auxiliary sys/password@TARGET_AUX << 'EOF'
DUPLICATE TARGET DATABASE TO ORCL_NEW
  FROM ACTIVE DATABASE
  USING COMPRESSED BACKUPSET
  SECTION SIZE 10G
  SPFILE
    SET db_unique_name='ORCL_NEW'
    SET log_archive_dest_1='LOCATION=+RECO'
  NOFILENAMECHECK;
EOF
```

### SAP HANA Migration (Backup/Restore)

```bash
# Step 1: Full backup of source HANA system
hdbsql -i <INSTANCE> -u SYSTEM -p <PASSWORD> \
  "BACKUP DATA USING BACKINT ('FULL_MIGRATION_BACKUP')"

# Step 2: Copy backup to Cloud Storage
gsutil -m rsync -r /hana/backup/ gs://my-migration-bucket/hana-backup/

# Step 3: On target BMS — restore from GCS backup
gsutil -m rsync -r gs://my-migration-bucket/hana-backup/ /hana/backup/

# Step 4: Recover the system on BMS
hdbrecover -i <INSTANCE> \
  --recover-until-now \
  --using-backup-catalog=/hana/backup \
  --target-system-id=<SID>
```

### Migration Decision Matrix

| Migration Method | Database Size | Downtime | Complexity | Best For |
|---|---|---|---|---|
| **Oracle Data Pump** | < 2 TB | Hours–days | Low | Schema migration, version upgrade |
| **RMAN Duplicate (Active)** | Any size | Minutes | Medium | Same version, minimal downtime |
| **RMAN Backup/Restore** | Any size | Hours | Low | Offline migration window acceptable |
| **Oracle Data Guard** | Any size | Near-zero | High | Zero-downtime production migration |
| **SAP HANA Backup/Restore** | < 5 TB | Hours | Low | Offline migration acceptable |
| **SAP HANA System Replication** | Any size | Minutes | High | Near-zero downtime migration |

### Post-Migration Validation

```sql
-- Oracle: Validate data integrity after migration
SELECT COUNT(*) FROM dba_tables;
SELECT owner, table_name, num_rows FROM dba_tables ORDER BY num_rows DESC;

-- Check for invalid objects
SELECT owner, object_name, object_type, status
FROM dba_objects WHERE status = 'INVALID'
ORDER BY owner, object_type;

-- Validate tablespace usage
SELECT tablespace_name, used_space*8192/1024/1024 used_mb,
       free_space*8192/1024/1024 free_mb
FROM dba_tablespace_usage_metrics;
```

```bash
# SAP HANA: Post-migration health check
hdbnsutil -checkSystem
hdbsql -i <INSTANCE> -u SYSTEM -p <PASSWORD> \
  "SELECT * FROM SYS.M_DATABASE"
```

---

## 18. Troubleshooting Quick Reference

| Issue | Likely Cause | Fix |
|---|---|---|
| **SSH connection refused / timeout** | firewalld/iptables blocking SSH; wrong IP; jump server not configured | Verify OS firewall allows SSH from jump server IP; confirm BMS IP via `gcloud bms instances describe`; test from jump server within VPC |
| **LUN not visible after attachment** | iSCSI session not refreshed; udev rules not triggered; multipath not reloaded | Run `iscsiadm -m session --rescan`; `rescan-scsi-bus.sh`; `multipath -r`; verify with `lsblk` |
| **NFS mount fails: "Connection refused"** | NFS client package missing; network route missing; NFS server IP wrong | `yum install -y nfs-utils`; verify route to NFS server; check mount path in NFS share config |
| **NFS mount fails: "Permission denied"** | Client IP not in NFS allowed clients list | Update NFS share allowed clients list via BMS Console/API to include BMS server IP |
| **iSCSI session drops intermittently** | Network instability; iSCSI timeout too low; MTU mismatch | Increase `node.session.timeo.replacement_timeout`; verify MTU matches on all hops (jumbo frames 9000 recommended for storage network) |
| **OS not booting after reimage** | Reimage failed partway; boot LUN not detected | Contact Google Cloud Support; check BMS console for reimage operation status |
| **Ops Agent not sending metrics** | Private Google Access not enabled; DNS resolution failing for `googleapis.com`; Agent not installed/running | Enable PGA on subnet; add `199.36.153.8` route; `systemctl status google-cloud-ops-agent`; check `/var/log/google-cloud-ops-agent/` |
| **Cloud Storage access denied from BMS** | Service account lacks `storage.objectViewer`; PGA not configured; wrong DNS for `storage.googleapis.com` | Grant SA role; verify `199.36.153.8` route exists; add `storage.googleapis.com` to `/etc/hosts` |
| **High network latency spikes** | Cloud Interconnect congestion; VLAN misconfiguration; MTU fragmentation | Check Interconnect monitoring in Cloud Console; verify MTU is consistent (9000 recommended); check QoS tagging |
| **Oracle ASM disk discovery fails** | udev rules not applied; wrong permissions; multipath device not available | `udevadm trigger`; verify `/dev/oracleasm/` symlinks exist; check `ls -la /dev/mapper/`; run `kpartx -a` |
| **BMS API permission error: 403** | Missing `roles/baremetalsolution.admin` or `editor` on project | `gcloud projects add-iam-policy-binding` with appropriate BMS role |
| **BMS provisioning stuck in PROVISIONING state** | Normal — BMS provisioning takes 3–7 business days | Check `gcloud bms operations list` for status; contact Google Cloud Support if > 10 business days |
| **Oracle listener not accessible from GCP VPC** | Port 1521 blocked by firewalld; listener bound to wrong IP | `firewall-cmd --add-port=1521/tcp --permanent && firewall-cmd --reload`; verify `lsnrctl status` shows correct IP |
| **SAP HANA System Replication failing** | Network latency too high between BMS nodes; HSR port blocked; log shipping behind | Check latency between BMS nodes on private VLAN; open ports 4001x–4007x; run `hdbnsutil -sr_state` |
| **Kernel OOM killer triggered** | Workload exceeding available RAM; HugePages not configured for Oracle/HANA | Increase `--memory` (select larger BMS profile); configure HugePages for Oracle SGA; verify no memory overcommit |
| **Disk I/O latency spikes on SSD LUN** | iSCSI path degraded; wrong I/O scheduler; read-ahead too high | `multipath -ll` to check paths; set I/O scheduler to `deadline`/`mq-deadline`; run `iostat -xz 1` to diagnose |

---

*Last updated: 2025 | Covers Bare Metal Solution GA features as of this date.*
*Refer to [cloud.google.com/bare-metal/docs](https://cloud.google.com/bare-metal/docs) for the latest information.*
