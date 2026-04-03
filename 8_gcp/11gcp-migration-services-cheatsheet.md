# GCP Migration Services — Production Cheatsheet
### Migrate to Virtual Machines · Database Migration Service · Transfer Appliance

---

## PART 1 — MIGRATE TO VIRTUAL MACHINES

Migrate to Virtual Machines (formerly **Velostrata**, then **Migrate for Compute Engine**) is Google Cloud's agent-based, streaming lift-and-shift service that moves workloads from VMware vSphere, physical servers, AWS EC2, and Azure VMs directly to Google Compute Engine. Its core differentiator is **streaming migration** — the migrated VM becomes live and operational on GCP while disk data is still replicating in the background, eliminating the traditional big-bang cutover window and dramatically reducing downtime.

---

### 1.1 OVERVIEW & ARCHITECTURE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SOURCE ENVIRONMENT                                  │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────┐  ┌───────────┐  ┌──────────────┐  │
│  │ VMware vCenter   │  │ Physical     │  │  AWS EC2  │  │  Azure VMs   │  │
│  │ vSphere 6.x–8.x  │  │ Servers      │  │           │  │              │  │
│  └────────┬─────────┘  └──────┬───────┘  └─────┬─────┘  └──────┬───────┘  │
└───────────┼────────────────────┼────────────────┼───────────────┼───────────┘
            │   Replication traffic (TLS 443) via VPN / Interconnect / Internet
            └──────────────────────────┬───────────────────────────┘
                                       │
┌──────────────────────────────────────▼──────────────────────────────────────┐
│                         GOOGLE CLOUD PROJECT                                │
│                                                                             │
│  Management Plane                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Cloud Console  │  gcloud CLI  │  REST API / Terraform              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────────────┐    ┌────────────────────────────────────┐    │
│  │   Migration Manager      │    │  vSphere Plugin (VMware only)      │    │
│  │   (GCP-hosted service)   │    │  Installed in vCenter              │    │
│  └────────────┬─────────────┘    └────────────────────────────────────┘    │
│               │                                                             │
│               ▼                                                             │
│  ┌─────────────────────────┐                                               │
│  │  Cloud Storage          │  ← Replication staging bucket (regional)      │
│  │  Staging Bucket         │    same region as GCE target                  │
│  └────────────┬────────────┘                                               │
│               ▼                                                             │
│  ┌─────────────────────────┐                                               │
│  │  GCE Target VM          │  ← Streams live from staging; serves traffic  │
│  │  (running during sync)  │    before full disk copy completes            │
│  └─────────────────────────┘                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Deployment Models

| Aspect | Google Cloud Console Mode | API / Automation Mode |
|---|---|---|
| Target audience | Migration engineers, manual workflows | Platform/ops teams, large fleets |
| Orchestration | Point-and-click Console UI | `gcloud`, REST API, Terraform |
| Migration groups | Manual grouping via UI | Programmatic group + wave definitions |
| Automation support | Limited; one VM at a time practical | Full scripting, bulk operations |
| Recommended for | < 50 VMs, first migration project | > 50 VMs, repeatable factory migrations |

---

### 1.2 PREREQUISITES & SUPPORTED SOURCES

#### Source Platform Compatibility

| Source Platform | Supported OS | Architecture | Notes |
|---|---|---|---|
| VMware vSphere 6.x–8.x | Windows, Linux (see OS table) | x86_64 | vSphere Plugin required in vCenter; user needs `VirtualMachine.Interact` privilege |
| Physical servers | Windows, Linux | x86_64 | Requires on-prem migration agent installation; ARM not supported |
| AWS EC2 | Windows, Linux | x86_64 | Uses AWS access key; EBS volumes replicated via snapshot chain |
| Azure VMs | Windows, Linux | x86_64 | Uses Azure service principal; managed and unmanaged disks both supported |

#### Supported Guest Operating Systems

| OS Family | Versions | Boot Type | Notes |
|---|---|---|---|
| Windows Server | 2012 R2, 2016, 2019, 2022 | BIOS, UEFI | UEFI requires N2/C3 target machine type |
| RHEL / CentOS | 6, 7, 8, 9 | BIOS, UEFI | CentOS 8 EOL — plan OS upgrade post-migration |
| Ubuntu | 18.04, 20.04, 22.04 | BIOS, UEFI | 18.04 EOL Apr 2023; upgrade recommended post-migration |
| Debian | 9, 10, 11, 12 | BIOS, UEFI | — |
| SLES | 12 SP4+, 15 | BIOS, UEFI | BYOS licensing required; PAYG not available at migration time |
| Oracle Linux | 7, 8, 9 | BIOS, UEFI | UEK and RHCK kernels both supported |

> ⚠️ **NOTE:** UEFI-booted VMs require GCE machine types that support UEFI Secure Boot. **N1 machine types do NOT support UEFI** — use N2, N2D, or C3. Mixing BIOS and UEFI in the same migration wave causes unpredictable boot failures.

#### GCP Prerequisites

```bash
# Enable required APIs
gcloud services enable \
  vmmigration.googleapis.com \
  compute.googleapis.com \
  cloudresourcemanager.googleapis.com \
  iam.googleapis.com \
  secretmanager.googleapis.com \
  storage.googleapis.com

# Grant IAM roles to the migration operator principal
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:migration-engineer@example.com" \
  --role="roles/vmmigration.admin"

gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:migration-engineer@example.com" \
  --role="roles/compute.admin"

gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:migration-engineer@example.com" \
  --role="roles/storage.admin"

# Create replication staging bucket — MUST be in same region as target GCE zone
gcloud storage buckets create gs://my-migration-staging \
  --location=us-central1 \
  --uniform-bucket-level-access \
  --public-access-prevention

# Firewall: allow egress TCP 443 from source network to *.storage.googleapis.com
# Firewall: allow ingress TCP 22/3389 from management CIDRs to target VPC
```

#### Network Bandwidth Requirements

| Factor | Guidance |
|---|---|
| Formula for minimum bandwidth | `Disk size (GB) ÷ Sync window (days) ÷ 86400 × 8 = Mbps required` |
| Example: 500 GB disk, 2-day sync | 500 × 1024 ÷ 2 ÷ 86400 × 8 ≈ **23 Mbps sustained** |
| WAN latency | < 50 ms recommended; > 100 ms increases sync duration non-linearly |
| Continuous replication bandwidth | Proportional to VM write I/O rate; typically 5–20 Mbps |
| Recommended path | Cloud Interconnect (≥ 1 Gbps) or HA VPN for production migrations |

> 💡 **TIP:** Schedule the initial sync (heaviest phase) during off-peak hours. Continuous delta replication is proportional to the VM's write I/O rate, not total disk size — a 10 TB mostly-read disk replicates at near-zero bandwidth post-initial sync.

---

### 1.3 MIGRATION WORKFLOW — STEP BY STEP

#### Migration Lifecycle State Machine

```text
              ┌─────────────┐
              │ NOT_STARTED │  ← Job created; replication not started
              └──────┬──────┘
                     │ start-replication
              ┌──────▼──────┐
              │   PENDING   │  ← Agent connecting; initial handshake
              └──────┬──────┘
                     │ (initial full sync completes)
              ┌──────▼──────┐
              │    READY    │  ← Full sync done; continuous delta sync active
              └──────┬──────┘
         ┌───────────┤ (ongoing delta sync loop)
         │           │
┌────────▼─────────┐ │
│ REPLICATION_CYCLE│─┘  ← Delta replication in progress (returns to READY)
└──────────────────┘

         From READY:
              ├── create-clone-job ──────────────────────────────────────┐
              │                                                           │
              │                                              ┌───────────▼─────────┐
              │                                              │    TEST_CLONING     │
              │                                              └───────────┬─────────┘
              │                                              ┌───────────▼─────────┐
              │                                              │  TEST_CLONE_READY   │
              │                                              │  (validate & delete) │
              │                                              └─────────────────────┘
              │
              └── cut-over ─────────────────────────────────────────────┐
                                                              ┌──────────▼──────────────┐
                                                              │  CUTOVER_IN_PROGRESS    │
                                                              └──────────┬──────────────┘
                                                              ┌──────────▼──────────────┐
                                                              │   CUTOVER_COMPLETE      │
                                                              └──────────┬──────────────┘
                                                                         │ finalize
                                                              ┌──────────▼──────────────┐
                                                              │      COMPLETED          │
                                                              └─────────────────────────┘
```

#### Phase Breakdown

| Phase | Action | Duration Estimate | Reversible? |
|---|---|---|---|
| Source discovery | Source registered; VMs enumerated | Minutes | Yes |
| Add source | Credentials + connectivity validated | Minutes | Yes — delete source |
| Create migration job | Target config defined (zone, machine type, network) | < 1 min | Yes — delete job |
| Initial sync | Full disk copied to GCS staging | Hours to days (size/bandwidth) | Yes — cancel replication |
| Continuous replication | Delta sync; source VM stays live | Ongoing until cutover | Yes — cancel replication |
| Test clone | Ephemeral GCE clone for validation; source unaffected | 5–15 min to create | Yes — delete clone |
| Cutover | Source powered off; GCE VM goes live | 2–10 min | Yes — `cancel-cutover` reverts |
| Finalize | Staging purged; source lock released | < 2 min | **No — irreversible** |

#### Step-by-Step gcloud CLI Workflow (VMware Source)

```bash
# Step 1: Enable the Migrate to VMs API
gcloud services enable vmmigration.googleapis.com

# Step 2: Create a migration source (VMware vCenter)
gcloud migration vms sources create my-vcenter-source \
  --location=us-central1 \
  --type=VMWARE \
  --vmware-vcenter-ip=192.168.1.100 \
  --vmware-username=administrator@vsphere.local \
  --vmware-password-secret=projects/PROJECT/secrets/vcenter-password/versions/latest

# Step 3: List discovered VMs from the source
gcloud migration vms discovered-vms list \
  --source=my-vcenter-source \
  --location=us-central1

# Step 4: Create a migration job
gcloud migration vms migrations create my-migration \
  --source=my-vcenter-source \
  --vm-id=vm-1234 \
  --location=us-central1 \
  --target-project=my-gcp-project \
  --target-zone=us-central1-a \
  --target-machine-type=n2-standard-4 \
  --target-network=projects/PROJECT/global/networks/default \
  --target-subnet=projects/PROJECT/regions/us-central1/subnetworks/default

# Step 5: Start replication (triggers initial full sync)
gcloud migration vms migrations start-replication my-migration \
  --location=us-central1

# Step 6: Check replication status and lag
gcloud migration vms migrations describe my-migration \
  --location=us-central1
# Key fields: state, lastReplicationCycle.endTime, recentCutoverJobs

# Step 7: Create a test clone (non-disruptive — source keeps running)
gcloud migration vms migrations create-clone-job my-migration \
  --location=us-central1
# Validate the clone: boot, SSH/RDP, run smoke tests, then delete before cutover

# Step 8: Initiate cutover (powers off source; promotes GCE VM to live)
gcloud migration vms migrations cut-over my-migration \
  --location=us-central1
# To revert: gcloud migration vms migrations cancel-cutover my-migration --location=us-central1

# Step 9: Finalize — IRREVERSIBLE (purges staging; releases source VM lock)
gcloud migration vms migrations finalize my-migration \
  --location=us-central1
```

> ⚠️ **NOTE:** `finalize` is permanent. Once called, GCS staging data is deleted and the source VM is unlocked for decommission. Never finalize until the GCE VM has passed all validation and completed a burn-in period in production.

---

### 1.4 MIGRATION GROUPS & WAVE PLANNING

Migration waves group VMs by dependency tier, each with its own change window and rollback plan. Proper wave planning is the single most impactful factor in migration success — it contains blast radius, respects application interdependencies, and lets teams course-correct before tackling critical workloads.

#### Dependency Mapping Steps

1. **Network flow analysis** — use VPC Flow Logs or on-prem firewall logs; active TCP connections reveal runtime dependencies
2. **Application tier classification** — tag each VM: `web`, `app`, `cache`, `db`, `batch`, `monitoring`, `infra`
3. **Database isolation** — DBs migrate last, or use DMS in parallel (see Part 2) to avoid data loss during lift-and-shift
4. **Change window alignment** — match each wave's cutover window to the application tier's downtime tolerance

#### Wave Planning Template

| Wave | VMs | Dependencies | Target Zone | Cutover Window | Rollback Plan |
|---|---|---|---|---|---|
| 0 | DNS, NTP, LDAP, monitoring | None | us-central1-a | Sat 02:00–04:00 UTC | DNS re-points to source; cancel-cutover |
| 1 | Stateless web servers (nginx, HAProxy) | Wave 0 complete | us-central1-a/b | Sat 02:00–06:00 UTC | Load balancer targets reverted to source |
| 2 | Application servers (Java, Python) | Wave 1 complete | us-central1-a | Sat 02:00–06:00 UTC | cancel-cutover; source VMs remain intact |
| 3 | Cache layer (Redis, Memcached) | Wave 2 complete | us-central1-a | Same as Wave 2 | Flush cache; restart from source |
| 4 | Databases (MySQL, PostgreSQL) | Waves 1–3; DMS CDC active | us-central1-a | Dedicated window 04:00–08:00 UTC | DMS cancel-promote; repoint app to source DB |

#### Recommended Wave Order

```text
Wave 0  → Shared infra: DNS, NTP, jump hosts, monitoring
Wave 1  → Stateless / standalone: web servers, batch, cron
Wave 2  → Mid-tier application servers
Wave 3  → Cache / session stores (Redis, Memcached, Hazelcast)
Wave 4  → DB-adjacent: connection poolers (pgBouncer, ProxySQL)
Wave 5  → Databases (after DMS CDC replication lag < 30s)
```

> 💡 **TIP:** Never use Migrate to VMs for live transactional database servers. Use **Database Migration Service** (Part 2) for live DB migration with CDC replication. Migrate to VMs is appropriate for the OS/host layer after DMS has completed promotion.

---

### 1.5 COMPUTE ENGINE TARGET CONFIGURATION

#### Machine Type Selection

| Source Profile | Recommended GCE Family | Rationale |
|---|---|---|
| ≤ 8 vCPU, general workload | `e2-standard-*` | Most cost-efficient; burstable available |
| 8–64 vCPU, performance-sensitive | `n2-standard-*` / `n2d-standard-*` | Better single-thread perf than E2 |
| High memory (SAP, in-memory DB) | `m2-ultramem-*` / `m3-*` | Up to 12 TB RAM |
| Compute-intensive (HPC, batch) | `c3-standard-*` | Intel Sapphire Rapids, high frequency |
| GPU workloads | `a2-*` / `g2-standard-*` | NVIDIA A100 / L4 |

#### Disk Migration Behavior

| Disk | Default Behavior | Target GCE Disk Type | Notes |
|---|---|---|---|
| Boot disk | Always migrated | `pd-balanced` | Cannot be excluded |
| Data disks | Migrated by default | `pd-balanced` or `pd-ssd` | Can be excluded via `--exclude-disks` |
| Raw device mappings (RDM) | Not supported | — | Convert to VMDK before migration |
| NFS mounts (mounted paths) | Not captured | — | Re-mount to GCS FUSE, Filestore, or Persistent Disk post-migration |

#### Licensing

```bash
# Windows Server — BYOL (bring existing license; ~30% savings vs PAYG)
gcloud migration vms migrations create my-win-migration \
  --license-type=BYOL \
  --source=my-vcenter-source \
  --vm-id=vm-windows-001 \
  --location=us-central1

# RHEL — after migration, activate BYOS with Red Hat subscription
# ssh into migrated GCE VM:
sudo subscription-manager register \
  --username=YOUR_RH_USERNAME \
  --password=YOUR_RH_PASSWORD
sudo subscription-manager attach --auto
```

> ⚠️ **NOTE:** Static source IPs **cannot be preserved** when migrating across cloud boundaries. Pre-lower DNS TTLs to 60 seconds at least 48 hours before cutover. Use Cloud DNS CNAME updates pointing to new GCE IPs as the cutover mechanism — not IP reservation.

---

### 1.6 CUTOVER STRATEGIES & ROLLBACK

#### Cutover Types

| Strategy | Description | Downtime | When to Use |
|---|---|---|---|
| Test clone | Ephemeral clone spun up for validation; source stays live | Zero | Pre-cutover smoke testing — always do this first |
| Live cutover | Source powered off; GCE promoted immediately | 2–10 min | Standard production cutover after test clone passed |
| Staged cutover | Cutover in a pre-agreed off-peak window; teams standing by | 2–10 min | Most production migrations with change management |
| Emergency rollback | `cancel-cutover` called; source VM revived; DNS reverted | Minutes (+ DNS TTL) | Post-cutover failure discovered within rollback window |

#### Rollback Window Rules

- Window opens: immediately after `cut-over` command succeeds
- Window closes: when `finalize` is called (or manually invoked — no auto-close timer)
- Revert command: `gcloud migration vms migrations cancel-cutover MY_MIGRATION --location=LOCATION`
- Effect: source VM is powered back on; GCE target VM is terminated

#### Pre-Cutover Checklist

- [ ] Replication lag < 15 minutes (check `describe` output: `lastReplicationCycle`)
- [ ] Test clone created, booted successfully, application smoke tests passed
- [ ] DNS TTLs pre-lowered to 60 seconds (done 24–48 hours before cutover)
- [ ] Load balancer or reverse proxy targets updated to GCE IPs (and tested)
- [ ] Cloud Monitoring dashboards and alerting verified on GCE target VM
- [ ] Rollback procedure documented and distributed to all on-call engineers
- [ ] Change freeze window approved by change management board
- [ ] Stakeholders notified; rollback decision criteria agreed in advance
- [ ] Source VM snapshot taken on vSphere/AWS/Azure before initiating cutover

---

### 1.7 POST-MIGRATION TASKS

After cutover, the migrated VM is functional but not yet optimized for GCE. Install Google-native agents and drivers to unlock full platform capabilities.

#### Install VirtIO Drivers & Guest Packages

```bash
# Debian / Ubuntu
sudo apt-get update && sudo apt-get install -y \
  google-compute-engine \
  google-cloud-ops-agent \
  google-cloud-cli

# RHEL / CentOS / Oracle Linux
sudo yum install -y \
  google-compute-engine \
  google-cloud-ops-agent \
  google-cloud-cli

# Verify VirtIO network driver is loaded (should show virtio_net)
lspci | grep -i virtio
ethtool -i eth0 | grep driver

# Verify Cloud Ops Agent is running
sudo systemctl status google-cloud-ops-agent
```

> 💡 **TIP:** For Windows VMs, install the **Google Cloud VirtIO driver package** post-migration. Without it, network throughput is throttled and disk latency is higher. Download via the serial console using PowerShell if RDP is initially unavailable.

#### Post-Migration Cost Optimization

```bash
# Get automated right-sizing recommendations (available after 8+ days of metrics)
gcloud recommender recommendations list \
  --project=PROJECT \
  --location=us-central1-a \
  --recommender=google.compute.instance.MachineTypeRecommender \
  --format="table(description,primaryImpact.costProjection.cost.units)"

# Apply a 1-year CUD for predictable workloads
gcloud compute commitments create my-commitment \
  --region=us-central1 \
  --resources=vcpu=32,memory=128GB \
  --plan=TWELVE_MONTH \
  --type=GENERAL_PURPOSE_N2

# Attach a snapshot schedule to migrated disks for backup
gcloud compute resource-policies create snapshot-schedule daily-backup \
  --region=us-central1 \
  --max-retention-days=14 \
  --daily-schedule \
  --start-time=03:00

gcloud compute disks add-resource-policies DISK_NAME \
  --resource-policies=daily-backup \
  --zone=us-central1-a
```

---

### 1.8 LIMITS, QUOTAS & TROUBLESHOOTING

#### Key Limits

| Resource | Limit | Notes |
|---|---|---|
| Concurrent replications per project | 50 | Soft limit — file quota increase via Cloud Console |
| Max VM disk size | 64 TB | Per individual disk |
| Max disk count per VM | 60 | Boot + all data disks combined |
| Migration job retention after finalize | 30 days | Metadata only; staging data purged at finalize |
| Test clone lifetime | 24 hours default | Configurable up to 72 hours |
| Max migration sources per project | 100 | Per region |
| Concurrent test clones | 1 per migration job | Cannot run parallel clones for same job |

#### Troubleshooting Decision Trees

```text
── Replication stuck / not progressing ───────────────────────────────────────

├── Egress to GCS blocked?
│   Test: curl -v https://storage.googleapis.com from source VM
│   Fix: open TCP 443 to storage.googleapis.com from source network

├── vCenter credentials expired or revoked?
│   Rotate in Secret Manager → patch source:
│   gcloud migration vms sources patch my-vcenter-source --location=us-central1

├── Disk read errors on source?
│   Linux: fsck -n /dev/sdX   |   Windows: chkdsk C: /scan
│   Fix errors before resuming replication

├── GCS staging bucket full or quota exceeded?
│   Check: gcloud storage du gs://my-migration-staging
│   Delete stale staging data from cancelled/finalized jobs

└── VMware snapshot limit reached?
    VMware max: 31 snapshots per VM; migration uses 1 slot
    Consolidate existing snapshots on source before migrating

── Cutover VM not booting ────────────────────────────────────────────────────

├── GRUB / bootloader issue
│   Open serial console → Console → VM Instance → Serial Console
│   Boot to rescue disk; chroot and run grub-install

├── Missing VirtIO network drivers (Windows)
│   Connect via serial console → install Google VirtIO driver package offline
│   PowerShell: Invoke-WebRequest to download driver from GCS

├── IP address conflict in target VPC
│   GCE assigns new DHCP IP; check:
│   gcloud compute instances describe VM --format="value(networkInterfaces[0].networkIP)"
│   Update application config to new IP

└── GCE firewall blocking SSH (22) or RDP (3389)
    gcloud compute firewall-rules list --filter="direction=INGRESS"
    gcloud compute firewall-rules create allow-mgmt \
      --allow=tcp:22,tcp:3389 \
      --source-ranges=YOUR_MGMT_CIDR
```


---

## PART 2 — DATABASE MIGRATION SERVICE (DMS)

Database Migration Service is Google Cloud's fully managed, serverless service for migrating databases to Cloud SQL, AlloyDB, and other GCP-managed database offerings with minimal downtime. It handles both **homogeneous migrations** (e.g., MySQL → Cloud SQL for MySQL) using native replication protocols, and **heterogeneous migrations** (e.g., Oracle → AlloyDB) using schema conversion workspaces — removing the need to build or manage replication infrastructure.

---

### 2.1 OVERVIEW & ARCHITECTURE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SOURCE DATABASE ENVIRONMENT                              │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  ┌─────────────────┐  │
│  │ MySQL 5.6–8  │  │PostgreSQL    │  │ SQL Server │  │  Oracle 11g–21c │  │
│  │ on-prem/EC2  │  │ 9.4–16       │  │ 2014–2022  │  │  on-prem/EC2    │  │
│  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘  └────────┬────────┘  │
└─────────┼─────────────────┼────────────────┼──────────────────┼────────────┘
          │ Connectivity: IP Allowlist / Cloud VPN / PSC / Reverse SSH Tunnel
          └─────────────────┴────────────────┴──────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────────┐
│                      DMS MIGRATION JOB (serverless)                         │
│                                                                             │
│   ┌────────────────────────────────────────────────────────────────────┐   │
│   │   FULL_DUMP Phase ──► CDC / WAL Replication Phase ──► PROMOTE     │   │
│   └──────────────────────────────┬─────────────────────────────────────┘   │
│                     ┌────────────┴──────────────────────┐                  │
│          ┌──────────▼──────────┐         ┌──────────────▼──────────────┐   │
│          │  Cloud Storage      │         │  Cloud Monitoring            │   │
│          │  (dump staging)     │         │  replication_lag             │   │
│          └─────────────────────┘         │  full_dump_throughput        │   │
│                                          │  cdc_events_rate             │   │
│                                          └─────────────────────────────┘   │
└───────────────────────────────────────────────┬─────────────────────────────┘
                    ┌──────────────────────────┬─┴──────────────────────┐
           ┌────────▼────────┐       ┌─────────▼──────┐      ┌─────────▼───────┐
           │  Cloud SQL      │       │  AlloyDB        │      │  Cloud Spanner  │
           │ MySQL/PG/MSSQL  │       │  (PostgreSQL)   │      │  (preview)      │
           └─────────────────┘       └─────────────────┘      └─────────────────┘
```

#### Migration Types

| Type | Mechanism | Downtime | Replication | Use Case |
|---|---|---|---|---|
| Continuous (CDC) | Full dump → binlog/WAL/CDC streaming | Seconds (promote only) | Yes, until promote | Production; minimal downtime requirement |
| One-time dump | Logical dump → bulk load | Full load duration | No | Dev/test; small non-critical DBs |
| Physical dump + CDC | Physical backup to GCS → restore → CDC | Minutes (promote) | Yes | Very large DBs (> 1 TB) where logical dump is impractical |

---

### 2.2 SUPPORTED SOURCES & DESTINATIONS

| Source DB | Source Version | Destination | Migration Type | Notes |
|---|---|---|---|---|
| MySQL | 5.6, 5.7, 8.0 | Cloud SQL for MySQL | Continuous (CDC) | Binary logging required; `gtid_mode=ON` strongly recommended |
| PostgreSQL | 9.4–16 | Cloud SQL for PostgreSQL | Continuous (WAL) | `wal_level=logical` required |
| PostgreSQL | 9.4–16 | AlloyDB for PostgreSQL | Continuous (WAL) | Identical config to Cloud SQL PG destination |
| SQL Server | 2014, 2016, 2017, 2019, 2022 | Cloud SQL for SQL Server | Continuous (CDC) | Enterprise/Developer edition required for CDC |
| Oracle | 11g R2, 12c, 18c, 19c, 21c | Cloud SQL for PostgreSQL | Heterogeneous | Schema conversion workspace required |
| Oracle | 11g R2–21c | AlloyDB for PostgreSQL | Heterogeneous | Schema conversion workspace required |
| MongoDB | 4.x, 5.x, 6.x | MongoDB Atlas on GCP | Continuous | Preview; uses MongoDB Change Streams |
| Cloud SQL MySQL | 5.7, 8.0 | Cloud SQL MySQL (newer version) | Continuous | Supported in-place major version upgrade path |

> ⚠️ **NOTE:** SQL Server Standard edition does not support CDC on versions prior to SQL Server 2016 SP1. For Standard 2014/2016 RTM, you must use the **one-time dump** type, which requires a full maintenance window.

---

### 2.3 CONNECTIVITY PROFILES

A **Connection Profile** is a named, reusable resource storing the network address, credentials, and TLS configuration for a source or destination database. Multiple migration jobs share connection profiles, centralizing credential management and avoiding duplication.

#### Connection Types

| Type | Description | When to Use |
|---|---|---|
| IP allowlist | DMS public IPs whitelisted on source firewall | Source has public IP; simplest setup |
| Cloud SQL Auth Proxy | Encrypted proxy tunnel via sidecar binary | Cloud SQL-to-Cloud SQL migrations |
| Private Service Connect | Private endpoint in consumer VPC | No internet; highest security |
| Reverse SSH tunnel | SSH tunnel from source host to GCE bastion | Air-gapped on-prem; no inbound VPN |
| VPC Peering | Direct VPC-to-VPC peering | GCP-hosted source DB |
| Cloud VPN / Interconnect | Network-layer encrypted path | Enterprise on-prem, large-scale |

```bash
# ── MySQL Source Connection Profile ────────────────────────────────────────────
gcloud database-migration connection-profiles create my-mysql-source \
  --region=us-central1 \
  --type=MYSQL \
  --host=10.1.2.3 \
  --port=3306 \
  --username=dms_user \
  --password-secret=projects/PROJECT/secrets/mysql-dms-pwd/versions/latest \
  --ssl-mode=REQUIRE \
  --ca-certificate-path=./ca-cert.pem

# ── PostgreSQL Source Connection Profile ──────────────────────────────────────
gcloud database-migration connection-profiles create my-pg-source \
  --region=us-central1 \
  --type=POSTGRESQL \
  --host=10.1.2.5 \
  --port=5432 \
  --username=dms_user \
  --password-secret=projects/PROJECT/secrets/pg-dms-pwd/versions/latest \
  --ssl-mode=VERIFY_CA \
  --ca-certificate-path=./ca-cert.pem \
  --client-certificate-path=./client-cert.pem \
  --private-key-path=./client-key.pem

# ── Cloud SQL Destination Connection Profile ──────────────────────────────────
gcloud database-migration connection-profiles create my-cloudsql-dest \
  --region=us-central1 \
  --type=CLOUDSQL \
  --cloudsql-instance=projects/PROJECT/locations/us-central1/instances/prod-mysql

# ── AlloyDB Destination Connection Profile ────────────────────────────────────
gcloud database-migration connection-profiles create my-alloydb-dest \
  --region=us-central1 \
  --type=ALLOYDB \
  --alloydb-cluster=projects/PROJECT/locations/us-central1/clusters/prod-alloydb \
  --alloydb-instance=projects/PROJECT/locations/us-central1/clusters/prod-alloydb/instances/primary

# ── List and describe ─────────────────────────────────────────────────────────
gcloud database-migration connection-profiles list --region=us-central1
gcloud database-migration connection-profiles describe my-mysql-source --region=us-central1
```

---

### 2.4 SOURCE DATABASE PREREQUISITES

#### MySQL

```bash
# Required settings in my.cnf / my.ini on source MySQL server:
# [mysqld]
# binlog_format             = ROW      # Required
# binlog_row_image          = FULL     # Required
# expire_logs_days          = 3        # Keep binlogs 3+ days for DMS to consume
# server_id                 = 1        # Must be unique in replication topology
# gtid_mode                 = ON       # Strongly recommended for Cloud SQL
# enforce_gtid_consistency  = ON       # Required when gtid_mode=ON
# log_bin                   = /var/log/mysql/mysql-bin.log
```

```sql
-- Create DMS user on source MySQL
CREATE USER 'dms_user'@'%' IDENTIFIED BY 'STRONG_PASSWORD';
GRANT REPLICATION SLAVE ON *.* TO 'dms_user'@'%';
GRANT REPLICATION CLIENT ON *.* TO 'dms_user'@'%';
GRANT SELECT ON *.* TO 'dms_user'@'%';
GRANT RELOAD ON *.* TO 'dms_user'@'%';
GRANT LOCK TABLES ON *.* TO 'dms_user'@'%';
FLUSH PRIVILEGES;

-- Confirm binary logging is active
SHOW VARIABLES LIKE 'log_bin';          -- Must be: ON
SHOW VARIABLES LIKE 'binlog_format';    -- Must be: ROW
SHOW MASTER STATUS;                     -- Shows current binlog file and position
```

#### PostgreSQL

```bash
# Required settings in postgresql.conf:
# wal_level             = logical    # Required for logical replication
# max_replication_slots = 5          # At least 1 per DMS job + spare slots
# max_wal_senders       = 5          # At least 1 per DMS connection
# wal_keep_size         = 2048       # MB: retain WAL segments during lag spikes
```

```sql
-- Create DMS replication role
CREATE ROLE dms_user WITH
  LOGIN PASSWORD 'STRONG_PASSWORD'
  REPLICATION BYPASSRLS;

GRANT SELECT ON ALL TABLES IN SCHEMA public TO dms_user;
GRANT USAGE ON SCHEMA public TO dms_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO dms_user;

-- Confirm WAL level
SHOW wal_level;                    -- Must return: logical
SELECT count(*) FROM pg_replication_slots;  -- Increases by 1 when job starts
```

```
# pg_hba.conf — allow DMS source IP to connect for replication:
# TYPE  DATABASE    USER      ADDRESS            METHOD
  host  replication dms_user  DMS_IP_CIDR/32     md5
  host  all         dms_user  DMS_IP_CIDR/32     md5
```

#### SQL Server

```sql
-- Enable CDC at the database level
USE MyDatabase;
EXEC sys.sp_cdc_enable_db;

-- Enable CDC per table (repeat for each table to migrate)
EXEC sys.sp_cdc_enable_table
  @source_schema = N'dbo',
  @source_name   = N'orders',
  @role_name     = NULL,
  @supports_net_changes = 1;

-- Create DMS login
CREATE LOGIN dms_user WITH PASSWORD = 'STRONG_PASSWORD';
USE MyDatabase;
CREATE USER dms_user FOR LOGIN dms_user;
ALTER ROLE db_owner ADD MEMBER dms_user;

-- Confirm CDC is enabled
SELECT name, is_cdc_enabled FROM sys.databases WHERE name = 'MyDatabase';
SELECT name, is_tracked_by_cdc FROM sys.tables WHERE is_tracked_by_cdc = 1;
```

> ⚠️ **NOTE:** CDC requires **SQL Server Agent to be running**. Verify: `SELECT status FROM sys.dm_server_services WHERE servicename LIKE 'SQL Server Agent%'`. If Agent is stopped, CDC capture jobs will not execute and DMS will stall during the CDC phase.

---

### 2.5 MIGRATION JOB CREATION & MANAGEMENT

```bash
# ── Create continuous MySQL → Cloud SQL migration job ─────────────────────────
gcloud database-migration migration-jobs create my-mysql-migration \
  --region=us-central1 \
  --type=CONTINUOUS \
  --source=my-mysql-source \
  --destination=my-cloudsql-dest \
  --dump-type=LOGICAL \          # LOGICAL (default) | PHYSICAL (for > 1 TB)
  --dump-parallel-level=OPTIMAL  # MIN | OPTIMAL | MAX

# ── Verify job configuration BEFORE starting ──────────────────────────────────
# Catches connectivity, credential, and DB config issues without starting
gcloud database-migration migration-jobs verify my-mysql-migration \
  --region=us-central1

# ── Start the migration job ───────────────────────────────────────────────────
gcloud database-migration migration-jobs start my-mysql-migration \
  --region=us-central1

# ── Monitor status (poll every 60s during active phases) ─────────────────────
gcloud database-migration migration-jobs describe my-mysql-migration \
  --region=us-central1
# Key fields: state, phase, duration, error.message

# ── List all jobs in a region ─────────────────────────────────────────────────
gcloud database-migration migration-jobs list \
  --region=us-central1 \
  --format="table(name,state,type,source,destination,createTime)"

# ── Promote: stops CDC, promotes destination to primary (the cutover action) ──
gcloud database-migration migration-jobs promote my-mysql-migration \
  --region=us-central1

# ── Restart a failed job ──────────────────────────────────────────────────────
gcloud database-migration migration-jobs restart my-mysql-migration \
  --region=us-central1

# ── Clean up after completion ─────────────────────────────────────────────────
gcloud database-migration migration-jobs delete my-mysql-migration \
  --region=us-central1
```

#### Migration Job Status Reference

| Status | Description | Recommended Action |
|---|---|---|
| `NOT_STARTED` | Job created, not running | Run `start` |
| `FULL_DUMP` | Initial full dump in progress | Wait; monitor `full_dump_throughput` |
| `CDC` | Change data capture streaming active | Monitor `replication_lag`; wait for lag < 10s |
| `PROMOTE_IN_PROGRESS` | Promoting destination to primary | **Do not interrupt** |
| `COMPLETED` | Migration finished | Validate data; delete job |
| `FAILED` | Unrecoverable error | Check `error.message`; fix source; `restart` |

> 💡 **TIP:** Always run `verify` before `start`. It catches ~80% of configuration errors (missing grants, SSL mismatches, firewall blocks) before any data transfer begins, saving hours of debugging a partially-started job.

---

### 2.6 HETEROGENEOUS MIGRATIONS (ORACLE → CLOUD SQL / ALLOYDB)

Heterogeneous migrations require **schema conversion** in addition to data replication — Oracle's type system, PL/SQL, and proprietary constructs don't map directly to PostgreSQL. DMS provides a **Conversion Workspace** that auto-converts as much schema as possible and surfaces everything requiring manual intervention in a categorized report.

#### Conversion Workspace CLI

```bash
# Step 1: Create the conversion workspace
gcloud database-migration conversion-workspaces create my-ora-pg-workspace \
  --region=us-central1 \
  --source-database-engine=ORACLE \
  --destination-database-engine=POSTGRESQL \
  --display-name="Oracle 19c to AlloyDB Migration"

# Step 2: Export DDL from Oracle source (run on source DB with expdp or DBMS_METADATA)
# Then seed the workspace with the exported DDL file:
gcloud database-migration conversion-workspaces seed my-ora-pg-workspace \
  --region=us-central1 \
  --auto-commit \
  --oracle-ddl-file=./oracle-schema.sql

# Step 3: Run automatic schema conversion
gcloud database-migration conversion-workspaces convert my-ora-pg-workspace \
  --region=us-central1

# Step 4: Review conversion results (shows issue counts by severity)
gcloud database-migration conversion-workspaces describe my-ora-pg-workspace \
  --region=us-central1
# Issues categorized as: ACTION_REQUIRED | SUGGESTION | INFO
# Resolve ACTION_REQUIRED items in the DMS Console schema editor before proceeding
```

#### Common Oracle → PostgreSQL Conversion Issues

| Oracle Construct | PostgreSQL Equivalent | Conversion Action |
|---|---|---|
| `NUMBER(p,s)` | `NUMERIC(p,s)` | Automatic |
| `VARCHAR2(n)` | `VARCHAR(n)` | Automatic |
| `DATE` (includes time component) | `TIMESTAMP` | **Manual review** — Oracle DATE ≠ PG DATE |
| `SEQUENCES` | `SEQUENCES` | Automatic (gaps possible) |
| `ROWNUM` | `ROW_NUMBER()` or `LIMIT` | Manual rewrite |
| `CONNECT BY` (hierarchical query) | Recursive CTE (`WITH RECURSIVE`) | Manual rewrite |
| PL/SQL stored procedures | PL/pgSQL | Partial auto-convert; manual for complex logic |
| Oracle-specific packages (`DBMS_*`) | External libraries / custom functions | Manual |
| `SYSDATE` | `NOW()` or `CURRENT_TIMESTAMP` | Automatic |
| `NVL(x, y)` | `COALESCE(x, y)` | Automatic |
| `DECODE()` | `CASE WHEN` | Automatic |

> ⚠️ **NOTE:** Oracle's `DATE` type stores both date AND time with second precision. PostgreSQL's `DATE` stores only the date. DMS converts Oracle `DATE` to PostgreSQL `TIMESTAMP` by default, which is correct — but validate that application code handles the type correctly after migration.

---

### 2.7 MONITORING REPLICATION LAG & HEALTH

```bash
# Query current replication lag via gcloud monitoring
gcloud monitoring read \
  'metric.type="datamigration.googleapis.com/migration_job/replication_lag"
   AND resource.labels.migration_job_id="my-mysql-migration"' \
  --freshness=5m \
  --project=PROJECT

# Key DMS metrics in Cloud Monitoring:
# datamigration.googleapis.com/migration_job/replication_lag      — seconds behind source
# datamigration.googleapis.com/migration_job/full_dump_throughput — bytes/sec during dump
# datamigration.googleapis.com/migration_job/cdc_events_rate      — events/sec during CDC
```

#### Recommended Alert Policies

| Alert | Condition | Severity | Action |
|---|---|---|---|
| Replication lag elevated | `replication_lag > 60s` for 5 min | WARNING | Investigate source write load |
| Replication lag critical | `replication_lag > 300s` for 10 min | CRITICAL | Page on-call; assess cutover impact |
| Job state changed to FAILED | State metric == FAILED | CRITICAL | Immediate response |
| Full dump throughput drop | `full_dump_throughput < 1 MB/s` for 10 min | WARNING | Check source connectivity |

---

### 2.8 CUTOVER PROCEDURE & POST-MIGRATION

#### Cutover Checklist

- [ ] Replication lag < 10 seconds, sustained for 30 minutes
- [ ] All application teams notified; maintenance window active
- [ ] Application connection strings tested and validated against destination DB
- [ ] Read replica failover tested on destination Cloud SQL instance
- [ ] Automated backups enabled on Cloud SQL (`--backup-start-time=03:00`)
- [ ] PITR (point-in-time recovery) enabled on destination instance
- [ ] `promote` command staged and ready; estimated promote duration < 2 minutes

#### Post-Migration Validation

```sql
-- Run on BOTH source and destination — compare row counts
SELECT schemaname, tablename, n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;

-- On source PostgreSQL: confirm DMS replication slot exists before dropping
SELECT slot_name, active, restart_lsn
FROM pg_replication_slots;

-- After successful promote, drop the DMS replication slot on source
-- (orphaned slots prevent WAL cleanup and can fill up source disk)
SELECT pg_drop_replication_slot('dms_slot_name');
```

#### Post-Migration Cloud SQL Hardening

```bash
# Disable public IP if Private Service Connect was the only connectivity method
gcloud sql instances patch prod-mysql \
  --no-assign-ip

# Enable PITR (requires binary logging on Cloud SQL — enabled by default)
gcloud sql instances patch prod-mysql \
  --enable-point-in-time-recovery \
  --backup-start-time=03:00

# Set maintenance window to avoid surprise reboots
gcloud sql instances patch prod-mysql \
  --maintenance-window-day=SUN \
  --maintenance-window-hour=4

# Enable IAM database authentication
gcloud sql instances patch prod-mysql \
  --database-flags=cloudsql_iam_authentication=on
```

---

### 2.9 LIMITS, QUOTAS & TROUBLESHOOTING

#### Key DMS Limits

| Resource | Limit | Notes |
|---|---|---|
| Migration jobs per region per project | 100 | Soft limit; request increase |
| Connection profiles per region | 100 | Shared across all jobs |
| Max table count (logical dump) | 100,000 | Performance degrades noticeably > 10,000 tables |
| Max row size (MySQL logical dump) | 1 GB | Use physical dump for large BLOBs |
| Max DB size (logical mode) | No hard limit | Use physical dump for databases > 1 TB |
| Replication lag guarantee | Best effort only | Depends entirely on source write rate and network |
| Conversion workspace entities | 100,000 | Total DDL objects for heterogeneous migrations |

#### Troubleshooting Decision Tree

```text
── Migration job FAILED ───────────────────────────────────────────────────────
│
│ First step: gcloud database-migration migration-jobs describe JOB --region=REGION
│             Check: error.message, error.code
│
├── "Access denied" / "Permission denied"
│   ├── MySQL: verify REPLICATION SLAVE, RELOAD, SELECT grants on dms_user
│   └── PostgreSQL: verify REPLICATION attribute on role; check pg_hba.conf
│
├── "Cannot connect to source" / "Connection refused"
│   ├── IP Allowlist: add DMS public IPs to source firewall (see Console for IPs)
│   ├── SSL: verify CA cert matches source server's TLS certificate
│   └── Reverse SSH tunnel down → restart tunnel daemon on bastion host
│
├── "Replication slot conflict" (PostgreSQL)
│   Orphaned slot from previous failed job:
│   SELECT pg_drop_replication_slot('slot_name_from_error');
│
├── Replication lag climbing continuously
│   ├── Source write rate exceeds DMS throughput (auto-managed; no manual scale)
│   │   → Temporarily reduce source write load during peak catch-up
│   └── Large bulk operation / batch job on source
│       → Wait for batch to complete; lag recovers automatically afterward
│
└── CDC phase stopped but job shows REPLICATING
    ├── MySQL: binlog retention too short (expire_logs_days)
    │   → If binlogs purged before DMS consumed: job must be restarted from scratch
    │   → Increase expire_logs_days to 7 before restarting
    └── PostgreSQL: WAL segment recycled before DMS consumed
        → wal_keep_size was too low; increase and restart job with fresh dump
```


---

## PART 3 — TRANSFER APPLIANCE

Transfer Appliance is a physical, ruggedized, rack-mountable storage device that Google ships to your datacenter for offline bulk data migration into GCP. It is purpose-built for scenarios where the data volume is so large — or available internet bandwidth so limited — that online transfer would take weeks or be prohibitively expensive. All data is AES-256 encrypted on-device before shipment, and Google destroys residual data on the appliance after ingestion.

---

### 3.1 OVERVIEW & ARCHITECTURE

#### When to Use Transfer Appliance vs Online Transfer

| Data Size | Available Bandwidth | Online Transfer Time | Recommendation |
|---|---|---|---|
| < 10 TB | Any | < 10 days | Online via Storage Transfer Service |
| 10–100 TB | > 1 Gbps | 1–10 days | Evaluate cost both ways; appliance may win on speed |
| 10–100 TB | < 100 Mbps | > 10 days | **Transfer Appliance** |
| > 100 TB | Any | > 10 days | **Transfer Appliance** |
| > 1 PB | Any | Impractical online | **Multiple Transfer Appliances** (staggered) |

#### Appliance SKU Comparison

| Model | Usable Capacity | Form Factor | Max Network Throughput | Encryption |
|---|---|---|---|---|
| TA40 | 40 TB | 2U rack | 1 Gbps | AES-256 at-rest + TLS in-transit |
| TA300 | 300 TB | 8U rack | 10 Gbps | AES-256 at-rest + TLS in-transit |

#### Transfer Appliance Lifecycle

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                      TRANSFER APPLIANCE LIFECYCLE                            │
│                                                                              │
│  ① ORDER              ② SHIP TO YOU        ③ COPY DATA                     │
│  ─────────────────    ─────────────────    ─────────────────                │
│  Cloud Console →      Google ships         Rack appliance                   │
│  Create job           TA40 or TA300         Connect to network               │
│  Choose model         to your address       Mount via NFS/iSCSI/SMB          │
│  Set GCS bucket       5–7 business days     rsync / parallel copy            │
│  Receive passphrase   (US/EU regions)       Checksum verify                  │
│  → Store in           Verify tamper seal    Encrypt at rest (auto)           │
│    Secret Manager     on arrival                                             │
│                                                                              │
│  ④ LOCK & SHIP        ⑤ GOOGLE INGESTS     ⑥ VERIFY & CONFIRM              │
│  ─────────────────    ─────────────────    ─────────────────                │
│  ta-admin             Google receives      Email notification                │
│  lock-for-shipping    Unlocks with         Check GCS object count            │
│  Apply tamper seal    your passphrase      Validate manifest                 │
│  Ship with Google     Uploads to GCS       checksums                        │
│  -provided label      5–10 biz days        Drop replication slot             │
│  Insure appliance     Wipes device         Request Cert of Destruction       │
│                       (DoD 5220.22-M)      if required                      │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

### 3.2 ORDERING & SETUP

#### Prerequisites

- GCS destination bucket must exist **before** ordering; must be in the same continent as the Google ingestion facility (US facility → US region bucket; EU facility → EU region bucket)
- Required IAM role for ordering: `roles/storagetransfer.admin` on the project
- Datacenter requirements:
  - TA40: 2U rack slot, standard 15A PDU, 1GbE or 10GbE network port
  - TA300: 8U rack slot, 30A PDU (240V recommended), 10GbE network port
  - Physical security: locked cage or secure rack in access-controlled area

> ⚠️ **NOTE:** The **encryption passphrase** is generated once during order creation and displayed only at that moment. Store it **immediately** in Secret Manager or a hardware security module. If the passphrase is lost, the data on the appliance is **permanently unrecoverable** — Google cannot decrypt it.

#### Order Flow (Console Only — No gcloud CLI for Ordering)

```
Cloud Console → Storage → Transfer Appliance → Create Transfer
  ↓
Specify: GCS destination bucket
         Appliance model (TA40 or TA300)
         Shipping address and contact
         Data description (NFS shares, directories, estimated volume)
  ↓
Receive: Appliance serial number + encryption passphrase
         Google-provided shipping label (for return shipment)
         Order tracking number
```

#### Configuring the Appliance After Arrival

```bash
# Connect to appliance management UI via browser (use credentials on first-boot label):
# https://APPLIANCE_IP:8443

# Configure network via CLI (SSH to management port 22):
ta-config network set-ip \
  --interface=eth0 \
  --ip=10.0.1.50 \
  --netmask=255.255.255.0 \
  --gateway=10.0.1.1 \
  --dns=8.8.8.8

# Verify connectivity from appliance to your NAS / source data servers
ta-config network test --target=YOUR_NAS_IP

# Check appliance health and storage status
ta-admin status
ta-admin storage-status
```

---

### 3.3 DATA COPY METHODS

#### Supported Ingestion Protocols

| Protocol | Max Speed | Use Case | Notes |
|---|---|---|---|
| NFS v3/v4 | Up to 10 Gbps | Linux/Unix NAS, file servers | Appliance acts as NFS server; most common method |
| iSCSI | Up to 10 Gbps | Block-level raw disk images | For database files or disk images requiring block access |
| SMB / CIFS | Up to 1 Gbps | Windows file shares | Appliance exposes SMB share; limited to 1 Gbps |
| Direct rsync over NFS | Up to 10 Gbps | Any POSIX file source | NFS mount + rsync; supports resume and checksum verification |

#### Mounting and Copying Data

```bash
# Mount the appliance as an NFS target on your Linux server
sudo mount -t nfs APPLIANCE_IP:/data /mnt/appliance \
  -o rw,hard,intr,rsize=1048576,wsize=1048576,nfsvers=3,timeo=300

# Basic rsync copy (preserves permissions, supports resume on interruption)
rsync -avz --progress --checksum \
  /source/data/ \
  /mnt/appliance/destination-folder/

# Maximum throughput — parallel rsync across subdirectories using GNU parallel
find /source/data -mindepth 1 -maxdepth 1 -type d | \
  parallel --jobs 8 \
  rsync -avz --progress {} /mnt/appliance/

# Verify copy integrity before shipping — compare MD5 checksums
find /source/data -type f -exec md5sum {} \; \
  | sort > /tmp/source-checksums.txt

find /mnt/appliance/destination-folder -type f -exec md5sum {} \; \
  | sort > /tmp/appliance-checksums.txt

diff /tmp/source-checksums.txt /tmp/appliance-checksums.txt
# Zero output = perfect match; ship only when checksums match
```

> 💡 **TIP:** Enable jumbo frames (MTU 9000) on both the client server's NIC and the appliance data port for maximum throughput. A mismatched MTU is the single most common cause of sub-100 MB/s copy speeds on 10 GbE networks. Verify with: `ip link show eth0 | grep mtu`.

#### Encryption (Automatic — No User Action Required)

All data written to the appliance is automatically encrypted with AES-256 full-disk encryption (LUKS) using the passphrase established during ordering. There is no unencrypted write mode. The passphrase is required by Google's ingestion facility and must be entered via the Cloud Console job interface — it is never transmitted automatically.

---

### 3.4 SHIPPING, INGESTION & VALIDATION

#### Preparing for Shipping

```bash
# Lock the appliance before handing to shipping carrier
# This seals the encryption and activates tamper-evident protection
ta-admin lock-for-shipping \
  --confirm \
  --passphrase-hash=SHA256_OF_YOUR_PASSPHRASE

# Confirm locked and sealed status before handoff
ta-admin status | grep -E "locked|sealed|tamper"
# Expected output: locked: true, tamper-seal: active
```

#### Shipping Logistics

- Use the **Google-provided shipping label** (printed from Console during order creation)
- Ship to the Google ingestion facility address shown in the job details page
- Insure the appliance for its replacement value; Google assumes liability after confirmed receipt
- Track shipment progress via: Console → Transfer Appliance → your job → **Shipment tracking** tab
- Verify the **tamper-evident seal serial number** matches what's recorded in the Console before handing to carrier

#### Post-Ingestion Validation

```bash
# After Google emails "Ingestion Complete" notification, verify GCS destination
gcloud storage ls gs://YOUR-DESTINATION-BUCKET/ --recursive | head -100

# Compare object count (should match your source file count)
gcloud storage ls gs://YOUR-DESTINATION-BUCKET/ --recursive | wc -l

# Download the ingestion manifest that Google uploads to GCS
gcloud storage cp \
  gs://YOUR-DESTINATION-BUCKET/transfer_appliance_manifest.json \
  ./manifest.json

# Inspect the manifest structure
cat manifest.json | python3 -m json.tool | head -40
```

```python
import json, hashlib, pathlib, sys

# Validate GCS objects against Transfer Appliance ingestion manifest
with open("manifest.json") as f:
    manifest = json.load(f)

errors = []
matched = 0

for entry in manifest.get("files", []):
    local_path = pathlib.Path(entry["local_path"])
    expected_md5 = entry.get("md5_checksum", "")

    if not local_path.exists():
        errors.append(f"MISSING: {local_path}")
        continue

    actual_md5 = hashlib.md5(local_path.read_bytes()).hexdigest()
    if actual_md5 != expected_md5:
        errors.append(f"MISMATCH: {local_path} (expected={expected_md5}, got={actual_md5})")
    else:
        matched += 1

total = len(manifest.get("files", []))
print(f"Checked: {total} files | Matched: {matched} | Errors: {len(errors)}")

if errors:
    print("\nValidation failures:")
    for e in errors:
        print(f"  {e}")
    sys.exit(1)
else:
    print("All checksums verified successfully.")
```

---

### 3.5 ENCRYPTION & SECURITY

#### Encryption Model

| Layer | Mechanism | Key Holder | Notes |
|---|---|---|---|
| At-rest (appliance disks) | AES-256 LUKS full-disk encryption | You (passphrase) | Always active; no plaintext mode |
| In-transit (appliance → GCS) | TLS 1.3 | Google + You | Active during Google ingestion |
| At-rest (GCS) | Google-managed keys (GMEK) or CMEK | Google / You | Configure CMEK on bucket before ordering |

#### Chain of Custody

- **On receipt:** verify tamper-evident seal serial number against Console records; report any seal breach to Google immediately
- **During use:** store appliance in locked, access-controlled area; restrict physical access to authorized personnel only
- **On return:** Google issues a **Certificate of Destruction** after DoD 5220.22-M wipe — request via Console for compliance documentation
- **Regulated industries:** HIPAA, PCI-DSS, FedRAMP customers should request Certificate of Destruction proactively and retain for audit records

> ⚠️ **NOTE:** For PCI-DSS compliance, ensure the GCS destination bucket uses **CMEK (Customer-Managed Encryption Keys)** via Cloud KMS, and restrict bucket IAM to only the service accounts that require access to the migrated data. Transfer Appliance itself meets FedRAMP Moderate requirements.

---

### 3.6 POST-TRANSFER — ORGANIZING DATA IN GCS

Data ingested via Transfer Appliance lands in GCS with the same directory hierarchy as it was copied to the appliance. Post-ingestion reorganization can be done using three tools depending on the scale and transformation needed.

| Tool | Use Case | Scale |
|---|---|---|
| `gcloud storage mv` | Prefix renames; small-to-medium object moves | Millions of objects |
| Storage Transfer Service | GCS-to-GCS copies with filtering, scheduling, and manifest support | Billions of objects |
| Cloud Dataflow | Format conversion (CSV → Parquet), transformations, enrichment | Any scale |

```bash
# Reorganize objects from flat appliance layout to date-partitioned structure
gcloud storage mv \
  "gs://BUCKET/raw-appliance-data/**" \
  "gs://BUCKET/organized/2025-01/" \
  --recursive

# Apply a tiered lifecycle policy to automatically move aging data to cheaper storage
gcloud storage buckets update gs://BUCKET \
  --lifecycle-file=lifecycle.json
```

```json
{
  "rule": [
    {
      "action": { "type": "SetStorageClass", "storageClass": "NEARLINE" },
      "condition": {
        "age": 30,
        "matchesStorageClass": ["STANDARD"]
      }
    },
    {
      "action": { "type": "SetStorageClass", "storageClass": "COLDLINE" },
      "condition": {
        "age": 90,
        "matchesStorageClass": ["NEARLINE"]
      }
    },
    {
      "action": { "type": "SetStorageClass", "storageClass": "ARCHIVE" },
      "condition": {
        "age": 365,
        "matchesStorageClass": ["COLDLINE"]
      }
    }
  ]
}
```

> 💡 **TIP:** If you need to run the same data through multiple GCP services (BigQuery, Dataproc, Vertex AI), land it once in a **STANDARD** GCS bucket and let the lifecycle policy tier it down over time. Avoid copying the same dataset across multiple buckets — use IAM and bucket-level access to control which services can read it.

---

### 3.7 LIMITS, PRICING & TROUBLESHOOTING

#### Limits and Logistics Reference

| Parameter | Value | Notes |
|---|---|---|
| Minimum order volume | 10 TB | Below 10 TB, use Storage Transfer Service online |
| Appliance loan period | 60 days (TA40) / 90 days (TA300) | Overage fees charged to billing account after deadline |
| Overdue daily fee | Varies by model | Typically $300–$500/day; check current pricing |
| Countries supported | US, UK, select EU | Check appliance availability page at order time |
| Google ingestion time | 5–10 business days after receipt | Not SLA-backed; plan buffer in migration timeline |
| GCS destination bucket region | Must be in same continent as ingestion facility | US facility → US bucket; EU facility → EU bucket |
| Max parallel copy sessions | 8 (TA40) / 16 (TA300) | Per appliance; more parallel rsync = faster copy |

#### Pricing Model

| Cost Component | Who Pays | Notes |
|---|---|---|
| Device rental fee | You | Per-day from Google shipment to Google receipt confirmation |
| Ingestion processing fee | You | Per TB ingested into GCS |
| GCS ingress | Free | No GCS ingress charges for Transfer Appliance |
| Outbound shipping | You | To Google facility; use Google-provided label for correct address |
| Return shipping | Google | Google pays return shipping after ingestion |

#### Troubleshooting Table

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Cannot mount NFS share from appliance | Firewall on appliance or client NIC | Run `ta-config firewall show`; allow NFS port 2049/tcp from client CIDR |
| Copy speed consistently < 100 MB/s | MTU mismatch or NIC duplex negotiation failure | Set `MTU=9000` on both client and appliance data interface; verify auto-negotiation |
| Appliance web UI (8443) unreachable | Wrong management IP or firewall | SSH to management port; run `ta-config network show` to confirm IP |
| Data partially missing after GCS ingestion | Checksum mismatch during Google ingestion | Review manifest JSON for failed files; reship missing files online via Storage Transfer Service |
| Appliance locked prematurely before copy complete | `lock-for-shipping` called accidentally | Contact Google Cloud Support immediately — appliance cannot self-unlock |
| Tamper seal broken on arrival | Damage in transit or tampering | Photograph and report to Google; do NOT load data; request replacement |


---

## PART 4 — CROSS-SERVICE MIGRATION PATTERNS & DECISION GUIDE

The three migration services covered in this cheatsheet are complementary, not mutually exclusive — most real-world enterprise migrations combine all three in parallel workstreams with careful sequencing. Understanding how they fit together is as important as knowing how to operate each one individually.

---

### 4.1 CHOOSING THE RIGHT MIGRATION SERVICE

| Scenario | Primary Service | Supporting Services |
|---|---|---|
| VMware VM lift-and-shift to GCE | Migrate to Virtual Machines | Cloud VPN / Interconnect, Cloud DNS, Secret Manager |
| AWS EC2 / Azure VM lift-and-shift to GCE | Migrate to Virtual Machines | Cloud Interconnect equivalent, Cloud DNS |
| MySQL / PostgreSQL on-prem → Cloud SQL, minimal downtime | DMS (CONTINUOUS type) | Secret Manager, VPC Peering / PSC, Cloud Monitoring |
| Oracle → AlloyDB (heterogeneous) | DMS + Conversion Workspace | Cloud SQL Auth Proxy, Schema review tooling |
| Petabyte-scale file data, limited bandwidth | Transfer Appliance (TA300) | GCS, Storage Transfer Service (post-ingest reorg) |
| < 1 TB data, good bandwidth | Storage Transfer Service (online) | GCS, Cloud Scheduler |
| Large DB (> 1 TB) + bandwidth constraints | DMS (PHYSICAL dump type) + Transfer Appliance | GCS staging, Cloud SQL for import |
| Application VMs + live database | DMS (DB) in parallel with Migrate to VMs (app) | Wave coordination, DNS cutover tooling |
| Edge datacenter with no internet | Transfer Appliance → GCS → App migration | GCS, Anthos Bare Metal on-prem |

---

### 4.2 COMBINED MIGRATION WAVE PATTERN

A typical 3-tier enterprise application migration coordinates all three services simultaneously across four waves. The key principle is **data migration leads application migration** — DMS replication must be stable and databases promoted before application VM cutover.

```text
──────────────────── MIGRATION TIMELINE ────────────────────────────────────────

 WEEK -4         WEEK -2         WEEK -1         CUTOVER WEEK
 ─────────────   ─────────────   ─────────────   ─────────────────────────────

 Wave 0: Infrastructure & Networking
 ├── Cloud VPN / Interconnect provisioning (Day -28)
 ├── GCP VPC, subnets, firewall rules, NAT config
 ├── Cloud DNS zones mirrored from on-prem DNS
 └── GCS buckets, IAM, Secret Manager secrets created

 Wave 1: Bulk Historical Data (Transfer Appliance)        ←── runs in background
 ├── TA300 ordered (Day -28)
 ├── TA300 arrives at datacenter (Day -21)
 ├── Data copied to appliance (Days -21 to -14)
 ├── Appliance shipped to Google (Day -14)
 └── Google ingests data into GCS (Days -9 to -4)         ←── verify on Day -3

 Wave 2: Live Database Replication (DMS)                  ←── parallel with Wave 1
 ├── Connection profiles created (Day -21)
 ├── DMS migration jobs created + verified (Day -20)
 ├── DMS FULL_DUMP phase (Day -20 to -15)
 ├── DMS CDC replication active (Day -15 onward)
 └── Monitor lag daily; target < 30s by cutover week

 Wave 3: Application VM Replication (Migrate to VMs)      ←── starts after VPN ready
 ├── Sources registered (Day -14)
 ├── Migration jobs created (Day -14)
 ├── Initial sync (Day -14 to -7)
 ├── Continuous replication + test clones (Days -7 to -2)
 └── All test clones validated by Day -1

 Wave 4: Cutover Window (Coordinated)
 ├── T-24h: DNS TTLs lowered to 60 seconds
 ├── T-4h:  DMS replication lag confirmed < 30s
 ├── T-0:   Maintenance window starts; apps write-quiesced
 ├── T+5m:  DMS `promote` executed → Cloud SQL/AlloyDB live
 ├── T+7m:  Migrate to VMs `cut-over` executed → GCE VMs live
 ├── T+10m: DNS A/CNAME records updated → new IPs
 ├── T+15m: Smoke tests run → go/no-go decision
 └── T+72h: Migrate to VMs `finalize` after successful burn-in

──────────────────────────────────────────────────────────────────────────────────
```

---

### 4.3 IAM ROLES SUMMARY

| Service | Role | Description |
|---|---|---|
| Migrate to VMs | `roles/vmmigration.admin` | Full migration lifecycle management |
| Migrate to VMs | `roles/vmmigration.viewer` | Read-only migration monitoring |
| Migrate to VMs | `roles/compute.admin` | Required to create GCE target VM resources |
| Migrate to VMs | `roles/storage.admin` | Required for GCS staging bucket access |
| DMS | `roles/datamigration.admin` | Full DMS job and profile management |
| DMS | `roles/datamigration.viewer` | Read-only view of jobs and profiles |
| DMS | `roles/cloudsql.admin` | Required to create Cloud SQL destination instances |
| DMS | `roles/alloydb.admin` | Required to create AlloyDB destination clusters |
| Transfer Appliance | `roles/storagetransfer.admin` | Order appliances; manage transfer jobs |
| Transfer Appliance | `roles/storage.admin` | Write permission on GCS destination bucket |
| All services | `roles/secretmanager.secretAccessor` | Access credentials stored in Secret Manager |
| All services | `roles/monitoring.viewer` | View Cloud Monitoring dashboards and metrics |

---

### 4.4 COST ESTIMATION GUIDE

#### Migrate to Virtual Machines

| Cost Component | Notes |
|---|---|
| Migrate to VMs service | **Free** — no charge for the migration service itself |
| Cloud VPN tunnel hours | ~$0.05/hour per tunnel; HA VPN = 2 tunnels minimum |
| GCS staging bucket | Temporary; Standard class; ~$0.02/GB/month while migration active |
| GCE instances | Full Compute Engine pricing once VMs are live post-cutover |
| Network egress | Source-side egress costs may apply (especially from AWS/Azure) |

#### Database Migration Service

| Cost Component | Notes |
|---|---|
| DMS migration job | **Free** for homogeneous migrations to Cloud SQL / AlloyDB |
| Conversion workspace | **Free** for heterogeneous schema conversion |
| Cloud SQL destination instance | Full Cloud SQL instance pricing applies during and after migration |
| AlloyDB destination | Full AlloyDB cluster pricing applies during and after migration |
| Network egress | Source-side egress from AWS/Azure during replication |

> 💡 **TIP:** To minimize DMS costs, right-size the destination Cloud SQL instance during migration (use a smaller tier during the dump/CDC phase) and scale up after cutover when actual load is known. Cloud SQL supports live vertical scaling with minimal downtime.

#### Transfer Appliance

| Cost Component | Notes |
|---|---|
| Device rental fee | Per-day from Google shipment date to confirmed Google receipt |
| Ingestion processing fee | Per TB of data successfully ingested into GCS |
| GCS ingress | **Free** — no GCS ingress charges for Transfer Appliance data |
| Outbound shipping | Customer responsibility (use Google-provided label for correct address) |
| Return shipping | Google responsibility after ingestion confirmed |

#### Online vs Appliance Cost Decision Formula

```text
Online Transfer Total Cost =
    (Source cloud egress $/GB × data volume in GB)
  + (Storage Transfer Service fee $/GB × data volume in GB)
  + (Network link cost: VPN/Interconnect hours × hourly rate)
  + (Operational cost: engineering days × day rate)

Appliance Transfer Total Cost =
    (Device rental $/day × estimated loan days)
  + (Ingestion fee $/TB × data volume in TB)
  + (Outbound shipping cost)

Decision rule:
  → If online cost < appliance cost AND transfer completes in < 7 days:  USE ONLINE
  → If data > 100 TB OR online transfer > 14 days:                       USE APPLIANCE
  → If both methods are similar cost:                                     USE APPLIANCE
    (Appliance is more predictable; eliminates bandwidth contention)
```

---

### 4.5 QUICK REFERENCE COMMAND CHEATSHEET

```bash
# ═══ MIGRATE TO VIRTUAL MACHINES ════════════════════════════════════════════

# API and source management
gcloud services enable vmmigration.googleapis.com
gcloud migration vms sources list --location=REGION
gcloud migration vms sources describe SOURCE_NAME --location=REGION
gcloud migration vms discovered-vms list --source=SOURCE --location=REGION

# Migration job lifecycle
gcloud migration vms migrations list --location=REGION
gcloud migration vms migrations describe MIGRATION --location=REGION
gcloud migration vms migrations start-replication MIGRATION --location=REGION
gcloud migration vms migrations create-clone-job MIGRATION --location=REGION
gcloud migration vms migrations cut-over MIGRATION --location=REGION
gcloud migration vms migrations cancel-cutover MIGRATION --location=REGION
gcloud migration vms migrations finalize MIGRATION --location=REGION
gcloud migration vms migrations delete MIGRATION --location=REGION

# ═══ DATABASE MIGRATION SERVICE ══════════════════════════════════════════════

# Connection profiles
gcloud database-migration connection-profiles list --region=REGION
gcloud database-migration connection-profiles describe NAME --region=REGION
gcloud database-migration connection-profiles delete NAME --region=REGION

# Migration jobs
gcloud database-migration migration-jobs list --region=REGION
gcloud database-migration migration-jobs describe JOB --region=REGION
gcloud database-migration migration-jobs verify JOB --region=REGION
gcloud database-migration migration-jobs start JOB --region=REGION
gcloud database-migration migration-jobs promote JOB --region=REGION
gcloud database-migration migration-jobs restart JOB --region=REGION
gcloud database-migration migration-jobs stop JOB --region=REGION
gcloud database-migration migration-jobs delete JOB --region=REGION

# Conversion workspaces (heterogeneous migrations)
gcloud database-migration conversion-workspaces list --region=REGION
gcloud database-migration conversion-workspaces describe WS --region=REGION
gcloud database-migration conversion-workspaces convert WS --region=REGION
gcloud database-migration conversion-workspaces seed WS \
  --oracle-ddl-file=FILE --region=REGION
gcloud database-migration conversion-workspaces delete WS --region=REGION

# Monitoring replication lag
gcloud monitoring read \
  'metric.type="datamigration.googleapis.com/migration_job/replication_lag"
   AND resource.labels.migration_job_id="JOB_ID"' \
  --freshness=5m --project=PROJECT

# ═══ TRANSFER APPLIANCE ══════════════════════════════════════════════════════
# Note: Ordering is Console-only; CLI is for post-arrival appliance management

# GCS bucket prep and validation
gcloud storage buckets create gs://BUCKET --location=REGION \
  --uniform-bucket-level-access
gcloud storage ls gs://BUCKET/ --recursive | wc -l
gcloud storage cp gs://BUCKET/transfer_appliance_manifest.json ./manifest.json

# Post-ingestion object reorganization
gcloud storage mv "gs://BUCKET/raw/**" "gs://BUCKET/organized/" --recursive
gcloud storage buckets update gs://BUCKET --lifecycle-file=lifecycle.json

# ═══ SHARED ACROSS SERVICES ══════════════════════════════════════════════════

# Store and retrieve database credentials in Secret Manager
gcloud secrets create db-password --data-file=-
echo -n "STRONG_PASSWORD" | gcloud secrets versions add db-password --data-file=-
gcloud secrets versions access latest --secret=db-password

# Monitor all migration-related operations in Cloud Logging
gcloud logging read \
  'resource.type="audited_resource" AND
   protoPayload.serviceName="vmmigration.googleapis.com"' \
  --freshness=1h --project=PROJECT --format=json | jq '.[] | .protoPayload.methodName'

gcloud logging read \
  'resource.type="audited_resource" AND
   protoPayload.serviceName="datamigration.googleapis.com"' \
  --freshness=1h --project=PROJECT
```

