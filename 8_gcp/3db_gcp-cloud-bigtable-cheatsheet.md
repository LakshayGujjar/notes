# 🗄️ GCP Cloud Bigtable — Comprehensive Cheatsheet

> **Audience:** Developers & Cloud Engineers | **Last updated:** 2025
> A single-reference document covering theory, schema design, row key patterns, filters, replication, SDK, and best practices.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Instance & Cluster Architecture](#2-instance--cluster-architecture)
3. [Data Model & Schema Design](#3-data-model--schema-design)
4. [Row Key Design — Deep Dive](#4-row-key-design--deep-dive)
5. [Reading Data](#5-reading-data)
6. [Writing Data](#6-writing-data)
7. [Filters](#7-filters)
8. [Garbage Collection (GC) Policies](#8-garbage-collection-gc-policies)
9. [Replication & App Profiles](#9-replication--app-profiles)
10. [Performance & Best Practices](#10-performance--best-practices)
11. [Key Visualizer](#11-key-visualizer)
12. [IAM & Security](#12-iam--security)
13. [Import, Export & Backups](#13-import-export--backups)
14. [Monitoring & Observability](#14-monitoring--observability)
15. [Client Library — Python](#15-client-library--python)
16. [cbt CLI Quick Reference](#16-cbt-cli-quick-reference)
17. [Pricing Model Summary](#17-pricing-model-summary)
18. [Common Errors & Troubleshooting](#18-common-errors--troubleshooting)

---

## 1. Overview & Key Concepts

**Cloud Bigtable** is Google's fully managed, petabyte-scale, wide-column NoSQL database designed for single-digit millisecond latency at massive throughput. It powers Google Search, Maps, and Analytics, and is the backbone for time-series, IoT, financial, and ML feature-store workloads.

### Cloud Bigtable vs Other Databases

```
┌───────────────────────────────────────────────────────────────────────────────────────┐
│                              Database Landscape                                       │
├─────────────┬────────────┬───────────┬────────────┬─────────────┬────────────────────┤
│  Bigtable   │  Firestore │  Spanner  │  Cloud SQL │    HBase    │ Cassandra/DynamoDB  │
├─────────────┼────────────┼───────────┼────────────┼─────────────┼────────────────────┤
│ Wide-column │ Document   │ Relational│ Relational │ Wide-column │ Wide-column        │
│ NoSQL       │ NoSQL      │ NewSQL    │ SQL        │ NoSQL       │ NoSQL              │
│ Manual ops  │ Serverless │ Managed   │ Managed    │ Self-managed│ Self/managed       │
│ No SQL      │ Rich query │ Full SQL  │ Full SQL   │ No SQL      │ Limited CQL        │
│ Row key only│ Indexes    │ Indexes   │ Indexes    │ Row key only│ Partition key      │
│ PB-scale    │ TB-scale   │ PB-scale  │ TB-scale   │ PB-scale    │ PB-scale           │
│ <1ms–10ms   │ <10ms      │ <10ms     │ <10ms      │ 1ms–100ms   │ 1ms–10ms           │
│ HBase API   │ Firestore  │ JDBC      │ JDBC       │ HBase API   │ Cassandra/DynamoDB │
│ $$–$$$      │ $–$$       │ $$$$      │ $–$$$      │ $$          │ $$–$$$             │
└─────────────┴────────────┴───────────┴────────────┴─────────────┴────────────────────┘
```

### When to Use Cloud Bigtable

- **Time-series** — IoT sensor data, metrics, stock ticks, logs at billions of events/day
- **AdTech** — user activity streams, click tracking, real-time bidding feature stores
- **Financial** — trade history, fraud signals, transaction aggregates
- **ML feature stores** — pre-computed feature vectors keyed by entity ID
- **Graph adjacency** — large-scale graph edge storage with composite row keys
- **Replacing HBase** — lift-and-shift with compatible API

### Core Terminology

| Term | Description |
|---|---|
| **Instance** | Top-level Bigtable resource. Contains one or more clusters and all tables. |
| **Cluster** | A set of nodes in a single zone. Stores data on Colossus (Google's distributed FS). |
| **Node** | A compute unit within a cluster. Does not store data directly; serves reads/writes. |
| **Table** | A collection of rows, sorted lexicographically by row key. No fixed schema. |
| **Row** | A single record identified by a unique row key. Max 256 MB recommended. |
| **Row Key** | Binary-safe string (max 4 KB) that uniquely identifies a row. Determines data locality. |
| **Column Family** | A named group of columns. GC policy is defined per family. Max 100 per table. |
| **Column Qualifier** | A named column within a family. Can be dynamic — no schema required. |
| **Cell** | The intersection of a row key + column family + column qualifier + timestamp. |
| **Timestamp** | 64-bit integer (microseconds by default). Multiple versions per cell allowed. |
| **Tablet** | A contiguous range of rows managed by one node. Bigtable auto-splits tablets. |
| **Compaction** | Background process that merges SSTables and applies GC policies. |
| **Garbage Collection (GC)** | Policy per column family that removes old cell versions asynchronously. |
| **App Profile** | Routing configuration — controls which cluster(s) receive requests and HA behaviour. |
| **Mutation** | A write operation: SetCell, DeleteFromColumn, DeleteFromRow, DeleteFromColumnFamily. |
| **Filter** | Server-side predicate applied during reads to select rows, columns, or values. |
| **Replication** | Async cross-cluster data sync for HA and workload isolation. |
| **Key Visualizer** | GCP Console tool that shows read/write heatmaps over row key space over time. |

---

## 2. Instance & Cluster Architecture

### Instance Types

| Type | Nodes | Replication | Use Case | SLA |
|---|---|---|---|---|
| **Production** | 3+ per cluster | Multi-cluster supported | Production workloads | 99.99% (multi-cluster) |
| **Development** | 1 (fixed) | Single cluster only | Dev/test (non-HA) | No SLA |

> ⚠️ **Warning:** Development instances have **no SLA**, cannot be resized, and cannot be converted to production. Never use them for production workloads.

### Cluster Configuration

| Parameter | Options | Notes |
|---|---|---|
| **Zone** | Any GCP zone | Choose zone closest to application; one cluster per zone |
| **Node count** | 1–n (production min 3) | Determines CPU, memory, and network throughput |
| **Storage type** | SSD or HDD | SSD: low-latency random access; HDD: high-throughput sequential, 3× cheaper |
| **Storage capacity** | ~16 TB/node (SSD), ~64 TB/node (HDD) | Managed by Colossus; nodes are compute-only |

### Storage & Throughput Per Node (Estimates)

| Storage Type | Storage per Node | Write Throughput | Read Throughput | Latency |
|---|---|---|---|---|
| **SSD** | ~16 TB | ~10,000 rows/s | ~10,000 rows/s | <1ms p50 |
| **HDD** | ~64 TB | ~500 rows/s | ~1,500 rows/s | <200ms p50 |

> **Tip:** These are per-node estimates for small rows (~1 KB). Large rows or heavy scan workloads significantly reduce effective throughput. Always load-test before sizing.

### Instance & Cluster CLI

```bash
# ── CREATE instance (production, SSD, 3 nodes) ──────────────────────
gcloud bigtable instances create my-instance \
  --display-name="My Production Instance" \
  --cluster=my-cluster-us-central1-a \
  --cluster-zone=us-central1-a \
  --cluster-num-nodes=3 \
  --cluster-storage-type=SSD

# ── CREATE development instance ─────────────────────────────────────
gcloud bigtable instances create my-dev-instance \
  --display-name="Dev Instance" \
  --cluster=dev-cluster \
  --cluster-zone=us-central1-a \
  --instance-type=DEVELOPMENT

# ── LIST instances ───────────────────────────────────────────────────
gcloud bigtable instances list

# ── DESCRIBE instance ────────────────────────────────────────────────
gcloud bigtable instances describe my-instance

# ── DELETE instance ──────────────────────────────────────────────────
gcloud bigtable instances delete my-instance

# ── ADD a second cluster (for replication) ──────────────────────────
gcloud bigtable clusters create my-cluster-us-east1-b \
  --instance=my-instance \
  --zone=us-east1-b \
  --num-nodes=3

# ── RESIZE a cluster ─────────────────────────────────────────────────
gcloud bigtable clusters update my-cluster-us-central1-a \
  --instance=my-instance \
  --num-nodes=5

# ── ENABLE autoscaling on a cluster ─────────────────────────────────
gcloud bigtable clusters update my-cluster-us-central1-a \
  --instance=my-instance \
  --autoscaling-min-nodes=3 \
  --autoscaling-max-nodes=10 \
  --autoscaling-cpu-target=60 \
  --autoscaling-storage-target=70

# ── LIST clusters ────────────────────────────────────────────────────
gcloud bigtable clusters list --instances=my-instance

# ── DESCRIBE cluster ─────────────────────────────────────────────────
gcloud bigtable clusters describe my-cluster-us-central1-a \
  --instance=my-instance
```

---

## 3. Data Model & Schema Design

### Wide-Column Structure

```
Table: "metrics"
─────────────────────────────────────────────────────────────────────────────────────
Row Key (sorted)          │ cf:temp       │ cf:humidity   │ cf:pressure
                          │ (col family)  │ (col family)  │ (col family)
──────────────────────────┼───────────────┼───────────────┼────────────────────────
sensor-A#~1706918400      │ t1: "22.5"    │ t1: "65"      │ t1: "1013.2"
                          │ t2: "22.3"    │               │                        ← multiple
                          │ t3: "22.1"    │               │                          cell versions
──────────────────────────┼───────────────┼───────────────┼────────────────────────
sensor-A#~1706918460      │ t1: "22.8"    │ t1: "64"      │ t1: "1013.0"
──────────────────────────┼───────────────┼───────────────┼────────────────────────
sensor-B#~1706918400      │ t1: "19.0"    │ t1: "70"      │ t1: "1012.5"
─────────────────────────────────────────────────────────────────────────────────────
                            ↑ column qualifier = "temp"
                              column family = "cf"
```

**Key properties:**
- Rows are **sorted lexicographically** by row key — this is the only index
- Column families must be defined at table creation; column qualifiers are dynamic
- Each cell can have **multiple versions** (timestamps) — GC policy controls retention
- Sparse columns — rows don't need to populate every qualifier

### Tall vs Wide Table Patterns

| Pattern | Structure | Best For | Avoid When |
|---|---|---|---|
| **Tall (narrow)** | Many rows, few columns per row | Time-series, event streams, logs | Need to scan many qualifiers per row |
| **Wide** | Few rows, many column qualifiers | User profiles, sparse feature stores | Row exceeds 100 MB |
| **Hybrid** | Rows grouped by time bucket + qualifier per metric | Aggregated metrics, dashboards | Access patterns span many buckets |

### Column Family Design

```
Best practices for column families:
  ✅  Group columns accessed together in the same family
  ✅  Keep families to 2–5 per table (max 100 but < 5 recommended)
  ✅  Assign a GC policy per family based on retention needs
  ✅  Use short family names (1–2 chars) to minimise row overhead
  ❌  Don't create a family per column qualifier
  ❌  Don't mix hot (frequent) and cold (rare) data in the same family
```

### Time-Series Schema Patterns

```
Pattern 1 — One metric per row (tall table, recommended for most IoT)
  Row key:  sensor-001#~1706918400   (~ prefix before reversed timestamp)
  cf:data   temp=22.5, humidity=65, pressure=1013.2

Pattern 2 — One column qualifier per metric (wide table, good for many metrics per device)
  Row key:  sensor-001#20240203      (daily bucket)
  cf:temp   1706918400=22.5, 1706918460=22.8, ...  (timestamp as qualifier)
  cf:hum    1706918400=65,   1706918460=64, ...
```

### Schema Anti-Patterns to Avoid

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Sequential row keys (`0001`, `0002`) | Write hotspot on one tablet | Hash prefix, reverse timestamp, or UUID |
| Timestamp as leading key component | All new rows hit the same tablet | Reverse or prefix with entity ID |
| Rows > 100 MB | Slow reads, tablet split issues | Distribute data across multiple rows |
| Too many column families (> 5) | Increased metadata overhead; slower scans | Consolidate into fewer families |
| Too many column qualifiers (> 100K per row) | Row becomes too wide; memory pressure | Bucket into separate rows |
| Storing blobs > 10 MB in cells | Network and memory overhead | Store blob in Cloud Storage; keep reference in Bigtable |
| Using Bigtable as a key-value store with tiny rows | High overhead per row vs data ratio | Consider Memorystore or Firestore |

---

## 4. Row Key Design — Deep Dive

> ⚠️ **Critical:** Row key design is the **single most important decision** in Bigtable schema design. A bad row key creates hotspots, limits throughput, and cannot be changed without rewriting the entire table.

### The Hotspot Problem

```
❌ Sequential keys — ALL new writes land on the SAME tablet (trailing edge hotspot)

Time →  t1      t2      t3      t4
Keys:   0001    0002    0003    0004   ← always writing to the "latest" tablet
                                        one node handles 100% of writes

✅ Hashed / reversed keys — writes DISTRIBUTE across all tablets

Time →  t1         t2         t3         t4
Keys:   a4f2...    91bc...    3e7d...    c82a...   ← random distribution
                                                     all nodes share the load
```

### Row Key Design Strategies

| Strategy | Format | Pros | Cons | Use Case |
|---|---|---|---|---|
| **Sequential ID** | `0001`, `0002` | Human-readable | Write hotspot | ❌ Never use as-is |
| **Hash prefix** | `md5(id)[:4]#id` | Even distribution | No range scans by entity | High-write entity IDs |
| **Reversed timestamp** | `MAX_TS - ts` | Latest-first scans | Non-intuitive | Time-series (latest N) |
| **Salted prefix** | `shard%N#id#ts` | Distributes load | Must fan-out reads | High-cardinality writes |
| **Composite key** | `tenantId#entityId#~ts` | Range scans per entity | Key design complexity | Multi-tenant, time-series |
| **Field-reversed** | `com.example.www` | Groups by domain suffix | Requires reversal logic | Domain/IP hierarchies |
| **UUID v4** | random 128-bit | Perfect distribution | No locality, no range scan | Pure key-value lookups |

### Reversed Timestamp Pattern (Latest-First)

```python
import struct, time

MAX_INT64 = 0x7FFFFFFFFFFFFFFF

def reversed_ts(ts_micros: int) -> bytes:
    """Encode timestamp so that lexicographic sort = reverse chronological."""
    return struct.pack(">q", MAX_INT64 - ts_micros)

# Current time in microseconds
now_micros = int(time.time() * 1_000_000)
rev_ts = reversed_ts(now_micros)

# Row key: "sensor-A#" + 8-byte reversed timestamp
row_key = b"sensor-A#" + rev_ts

# Scanning from the start of "sensor-A#" gives LATEST readings first
# Row key prefix scan on b"sensor-A#" returns newest → oldest
```

### Composite Key Patterns

```
Time-series (IoT sensor):
  metric#deviceId#~reversedTimestamp
  → "temperature#sensor-001#\xf8\x96..."
  → Range scan on "temperature#sensor-001#" gives latest readings first

Multi-tenant:
  tenantId#resourceType#resourceId
  → "acme#order#ord-9821"
  → All ACME orders are co-located; scan by tenant is efficient

User activity:
  userId#eventType#timestamp
  → "user-abc#purchase#1706918400"
  → Scan all purchases for user-abc with a prefix scan

Reverse domain (web crawl):
  reversed.domain#path#timestamp
  → "com.example.www#/products/abc#1706918400"
  → All pages from example.com are co-located
```

### Row Key Best Practices

| Rule | Guideline |
|---|---|
| **Max size** | 4 KB hard limit; keep under **64 bytes** for performance |
| **Separator character** | Use `#` or `\x00` (null byte) as field delimiter |
| **Binary-safe** | Row keys are raw bytes; use `struct.pack` for numeric fields |
| **No spaces** | Spaces in row keys cause issues with `cbt` CLI |
| **Reversed timestamps** | Always use `MAX_INT64 - timestamp` for latest-first access |
| **Avoid hot prefixes** | Ensure many distinct leading bytes across active keys |
| **Co-locate related rows** | Put frequently co-accessed rows adjacent (same prefix) |
| **Test with Key Visualizer** | Load test with production-like data before going live |

---

## 5. Reading Data

### Point Lookup (Single Row)

```python
from google.cloud import bigtable

client   = bigtable.Client(project="my-project", admin=True)
instance = client.instance("my-instance")
table    = instance.table("metrics")

# Read a single row by exact key
row = table.read_row(b"sensor-A#\xf8\x96\x00\x00\x00\x00\x00\x00")
if row:
    for cf, cols in row.cells.items():
        for qualifier, cells in cols.items():
            print(f"{cf}:{qualifier} = {cells[0].value} @ {cells[0].timestamp}")
```

```bash
# cbt point lookup
cbt -project=my-project -instance=my-instance \
  lookup metrics "sensor-A#somekey"
```

### Row Range Scan

```python
from google.cloud.bigtable.row_set import RowSet, RowRange

# Scan a contiguous range of rows
row_set = RowSet()
row_set.add_row_range(RowRange(
    start_key=b"sensor-A#",          # inclusive start
    end_key=b"sensor-B#",            # exclusive end
))

rows = table.read_rows(row_set=row_set, limit=1000)
rows.consume_all()

for row in rows.rows.values():
    print(row.row_key)
```

### Prefix Scan

```python
from google.cloud.bigtable.row_set import RowSet

# All rows starting with "sensor-A#"
prefix = b"sensor-A#"
end_key = prefix[:-1] + bytes([prefix[-1] + 1])  # prefix + 1

row_set = RowSet()
row_set.add_row_range_from_keys(start_key=prefix, end_key=end_key)

rows = table.read_rows(row_set=row_set, limit=100)
rows.consume_all()
```

```bash
# cbt prefix scan
cbt -project=my-project -instance=my-instance \
  read metrics prefix="sensor-A#" count=100
```

### Reading Specific Column Families / Qualifiers

```python
from google.cloud.bigtable.row_filters import (
    FamilyNameRegexFilter,
    ColumnQualifierRegexFilter,
    CellsPerColumnLimitFilter,
    RowFilterChain,
)

# Read only the "cf" column family
family_filter = FamilyNameRegexFilter("cf")

# Read only the latest version of each cell
latest_filter = CellsPerColumnLimitFilter(1)

# Chain filters (AND — both must match)
chain = RowFilterChain(filters=[family_filter, latest_filter])

rows = table.read_rows(filter_=chain, limit=50)
rows.consume_all()
```

### Reading Multiple Disjoint Rows

```python
from google.cloud.bigtable.row_set import RowSet

row_set = RowSet()
row_set.add_row_key(b"sensor-A#key1")
row_set.add_row_key(b"sensor-B#key2")
row_set.add_row_key(b"sensor-C#key3")

rows = table.read_rows(row_set=row_set)
rows.consume_all()
```

### Reading Latest N Versions

```python
from google.cloud.bigtable.row_filters import CellsPerColumnLimitFilter

# Read only the 3 most recent versions per cell
rows = table.read_rows(filter_=CellsPerColumnLimitFilter(3))
rows.consume_all()
```

### Reading with Timestamp Range

```python
from google.cloud.bigtable.row_filters import TimestampRangeFilter
import datetime

start = datetime.datetime(2025, 1, 1, tzinfo=datetime.timezone.utc)
end   = datetime.datetime(2025, 2, 1, tzinfo=datetime.timezone.utc)

ts_filter = TimestampRangeFilter(start=start, end=end)
rows = table.read_rows(filter_=ts_filter, limit=500)
rows.consume_all()
```

---

## 6. Writing Data

### Single Row Mutation

```python
from google.cloud.bigtable.row import DirectRow
import time

# Create a direct row and apply mutations
row = table.direct_row(b"sensor-A#key001")

# SetCell — write a value with explicit timestamp (microseconds)
ts_micros = int(time.time() * 1_000_000)
row.set_cell(
    column_family_id="cf",
    column="temp",
    value=b"22.5",
    timestamp_micros=ts_micros,
)
row.set_cell("cf", "humidity", b"65", timestamp_micros=ts_micros)

# Commit the mutation
row.commit()
```

```bash
# cbt write a single cell
cbt -project=my-project -instance=my-instance \
  set metrics "sensor-A#key001" cf:temp=22.5
```

### Bulk Mutations (MutateRows — Batch Write)

```python
from google.cloud.bigtable.row import DirectRow
import time

# Build a list of rows to write in one batched RPC
rows = []
ts = int(time.time() * 1_000_000)

for i in range(1000):
    row = table.direct_row(f"sensor-A#key{i:06d}".encode())
    row.set_cell("cf", "temp",     str(20 + i % 10).encode(), timestamp_micros=ts)
    row.set_cell("cf", "humidity", str(60 + i % 5).encode(),  timestamp_micros=ts)
    rows.append(row)

# Send all rows in one batch (max 100,000 mutations per batch)
errors = table.mutate_rows(rows)
for error in errors:
    print(f"Row {error.index} failed: {error.message}")
```

> **Tip:** Each `mutate_rows` call can contain up to **100,000 mutations** (individual SetCell/Delete operations). For highest throughput, batch 1,000–10,000 rows per call and parallelise across multiple threads.

### Conditional Mutation (CheckAndMutate)

```python
from google.cloud.bigtable.row import ConditionalRow
from google.cloud.bigtable.row_filters import ValueRegexFilter

# Apply true_mutations only if cell matches filter; false_mutations otherwise
cond_row = table.conditional_row(b"sensor-A#key001")

value_filter = ValueRegexFilter(b"22\\..*")  # matches values starting with "22."

# If condition is TRUE → apply these mutations
cond_row.set_cell("cf", "status", b"normal",  timestamp_micros=ts, state=True)

# If condition is FALSE → apply these mutations
cond_row.set_cell("cf", "status", b"abnormal", timestamp_micros=ts, state=False)

result = cond_row.commit()
print("Condition matched:", result)  # True if predicate was satisfied
```

### Read-Modify-Write (Append / Increment)

```python
from google.cloud.bigtable.row import AppendRow

# Atomically append bytes to a cell value
append_row = table.append_row(b"sensor-A#counter")
append_row.append_cell_value("cf", "log", b"| new-event")
append_row.commit()

# Atomically increment a 64-bit integer cell
from google.cloud.bigtable.row import DirectRow
# Use append_row.increment_cell_value for integer counters
append_row2 = table.append_row(b"sensor-A#counter")
append_row2.increment_cell_value("cf", "count", 1)
append_row2.commit()
```

### Delete Operations

```python
row = table.direct_row(b"sensor-A#key001")

# Delete a specific cell (all versions)
row.delete_cell("cf", "temp")

# Delete a specific cell at a specific timestamp
row.delete_cell("cf", "temp", time_range=TimestampRange(start=start, end=end))

# Delete all cells in a column family
row.delete_cells("cf", row.ALL_COLUMNS)

# Delete the entire row
row.delete()

row.commit()
```

```bash
# cbt delete a row
cbt -project=my-project -instance=my-instance \
  deleterow metrics "sensor-A#key001"

# cbt delete a specific cell
cbt -project=my-project -instance=my-instance \
  delete metrics "sensor-A#key001" cf:temp
```

---

## 7. Filters

> Filters are applied **server-side** — they reduce network bandwidth and client-side processing. Filters do not change what is stored; they select what is returned.

### Filter Types Reference

| Filter | Class | Description | Use Case |
|---|---|---|---|
| **RowKeyRegexFilter** | `RowKeyRegexFilter(regex)` | Include rows whose key matches regex | Prefix/pattern scans |
| **FamilyNameRegexFilter** | `FamilyNameRegexFilter(regex)` | Include only matching column families | Limit columns returned |
| **ColumnQualifierRegexFilter** | `ColumnQualifierRegexFilter(regex)` | Include columns matching regex | Select specific metrics |
| **ColumnRangeFilter** | `ColumnRangeFilter(family, start, end)` | Include columns in a qualifier range | Range of timestamps-as-qualifiers |
| **ValueRegexFilter** | `ValueRegexFilter(regex)` | Include cells whose value matches regex | Filter by value content |
| **ValueRangeFilter** | `ValueRangeFilter(start, end)` | Include cells in a value range | Numeric threshold queries |
| **TimestampRangeFilter** | `TimestampRangeFilter(start, end)` | Include cells in a time range | Time-windowed reads |
| **CellsPerRowLimitFilter** | `CellsPerRowLimitFilter(n)` | Return at most N cells per row | Row sampling |
| **CellsPerColumnLimitFilter** | `CellsPerColumnLimitFilter(n)` | Return at most N versions per column | Latest-N reads |
| **RowSampleFilter** | `RowSampleFilter(fraction)` | Return random sample of rows (0.0–1.0) | Approximate counts |
| **StripValueTransformer** | `StripValueTransformer()` | Return cells with empty values (key-only) | Key existence checks |
| **PassAllFilter** | `PassAllFilter()` | Pass all cells (no filtering) | Default / placeholder |
| **BlockAllFilter** | `BlockAllFilter()` | Block all cells | Condition filter false branch |

### Filter Composition

```python
from google.cloud.bigtable.row_filters import (
    RowFilterChain,        # AND — all filters must pass
    RowFilterUnion,        # OR — any filter can pass
    ConditionalRowFilter,  # IF predicate THEN true_filter ELSE false_filter
    CellsPerColumnLimitFilter,
    FamilyNameRegexFilter,
    TimestampRangeFilter,
    ValueRegexFilter,
    StripValueTransformer,
)
import datetime

# ── AND chain: family="cf" AND latest 1 version ────────────────────
chain_filter = RowFilterChain(filters=[
    FamilyNameRegexFilter("cf"),
    CellsPerColumnLimitFilter(1),
])

# ── OR union: return cells from "cf" OR "meta" families ────────────
union_filter = RowFilterUnion(filters=[
    FamilyNameRegexFilter("cf"),
    FamilyNameRegexFilter("meta"),
])

# ── Timestamp range AND latest version ─────────────────────────────
start = datetime.datetime(2025, 1, 1, tzinfo=datetime.timezone.utc)
end   = datetime.datetime(2025, 2, 1, tzinfo=datetime.timezone.utc)

time_and_latest = RowFilterChain(filters=[
    TimestampRangeFilter(start=start, end=end),
    CellsPerColumnLimitFilter(1),
])

# ── Conditional filter: if value matches "alarm" → strip value (key-only)
#    else → return cell normally ──────────────────────────────────────
cond_filter = ConditionalRowFilter(
    base_filter=ValueRegexFilter(b"alarm"),
    true_filter=StripValueTransformer(),
    false_filter=CellsPerColumnLimitFilter(1),
)

# Apply filter to a read
rows = table.read_rows(filter_=chain_filter, limit=100)
rows.consume_all()
```

---

## 8. Garbage Collection (GC) Policies

### What Are GC Policies?

GC policies control **how many versions** or **how long** cell data is retained per column family. Bigtable applies GC **asynchronously during compaction** — deleted versions are not immediately removed from storage but are excluded from reads.

### GC Policy Types

| Policy | Class | Behaviour |
|---|---|---|
| **MaxVersionsGCPolicy** | `MaxVersionsGCPolicy(n)` | Keep only the N most recent versions per cell |
| **MaxAgeGCPolicy** | `MaxAgeGCPolicy(duration)` | Delete cells older than the specified duration |
| **UnionGCPolicy** | `GCRuleUnion([p1, p2])` | Keep cell if it satisfies **either** policy (more permissive) |
| **IntersectionGCPolicy** | `GCRuleIntersection([p1, p2])` | Keep cell only if it satisfies **both** policies (more aggressive) |

### GC Policy Examples

```python
import datetime
from google.cloud.bigtable.column_family import (
    MaxVersionsGCRule,
    MaxAgeGCRule,
    GCRuleUnion,
    GCRuleIntersection,
)

# ── Keep only the 1 most recent version (common default) ────────────
cf_latest = table.column_family("cf", gc_rule=MaxVersionsGCRule(1))
cf_latest.create()

# ── Keep data for 30 days (time-series TTL) ──────────────────────────
cf_timeseries = table.column_family(
    "ts",
    gc_rule=MaxAgeGCRule(datetime.timedelta(days=30)),
)
cf_timeseries.create()

# ── Keep 3 versions OR data younger than 7 days (UNION — keep more) ─
cf_audit = table.column_family(
    "audit",
    gc_rule=GCRuleUnion(rules=[
        MaxVersionsGCRule(3),
        MaxAgeGCRule(datetime.timedelta(days=7)),
    ]),
)
cf_audit.create()

# ── Keep only if BOTH: ≤ 5 versions AND younger than 30 days ────────
# (INTERSECTION — keep less = most aggressive deletion)
cf_cache = table.column_family(
    "cache",
    gc_rule=GCRuleIntersection(rules=[
        MaxVersionsGCRule(5),
        MaxAgeGCRule(datetime.timedelta(days=30)),
    ]),
)
cf_cache.create()

# ── Update GC policy on existing column family ───────────────────────
existing_cf = table.column_family("cf", gc_rule=MaxVersionsGCRule(1))
existing_cf.update()
```

```bash
# cbt set GC policy — keep 1 version
cbt -project=my-project -instance=my-instance \
  setgcpolicy metrics cf maxversions=1

# cbt set GC policy — keep 30 days
cbt -project=my-project -instance=my-instance \
  setgcpolicy metrics ts maxage=30d

# cbt set GC policy — union: 3 versions OR 7 days
cbt -project=my-project -instance=my-instance \
  setgcpolicy metrics audit "(maxversions=3 || maxage=7d)"

# cbt set GC policy — intersection: 5 versions AND 30 days
cbt -project=my-project -instance=my-instance \
  setgcpolicy metrics cache "(maxversions=5 && maxage=30d)"
```

### GC Policy Design Guide

| Use Case | Recommended Policy | Rationale |
|---|---|---|
| Time-series (IoT, metrics) | `MaxAgeGCRule(30d)` | Keep rolling 30-day window |
| Audit log | `GCRuleUnion([MaxVersionsGCRule(3), MaxAgeGCRule(7d)])` | Keep at least 3 versions, up to 7 days |
| Cache / session | `GCRuleIntersection([MaxVersionsGCRule(1), MaxAgeGCRule(1d)])` | Expire aggressively |
| ML feature store | `MaxVersionsGCRule(1)` | Only latest feature vector needed |
| Event sourcing | `MaxAgeGCRule(365d)` | Keep 1 year of history |
| Write-once archive | No GC policy | Retain indefinitely |

> ⚠️ **Warning:** GC is applied **eventually** — storage is not reclaimed immediately. During heavy writes, storage may temporarily exceed expected bounds until compaction runs. Set GC policies on all column families; the default is to retain **all versions forever**.

---

## 9. Replication & App Profiles

### How Replication Works

```
Multi-cluster Bigtable Instance
┌─────────────────────────────────────────────────────┐
│                   Instance                          │
│                                                     │
│  ┌──────────────────┐    async    ┌───────────────┐ │
│  │ Cluster A        │ ─────────► │ Cluster B     │ │
│  │ us-central1-a    │ ◄───────── │ us-east1-b    │ │
│  │ 3 nodes / SSD    │    repl.   │ 3 nodes / SSD │ │
│  └──────────────────┘            └───────────────┘ │
│                                                     │
│  Replication: eventual consistency, async           │
│  Typical lag: < 1 minute (usually milliseconds)     │
└─────────────────────────────────────────────────────┘
```

### Replication Characteristics

| Property | Value |
|---|---|
| Consistency model | Eventual consistency (async replication) |
| Typical replication lag | Milliseconds to seconds; up to minutes under load |
| Conflict resolution | Last-writer-wins per cell (based on wall-clock timestamp) |
| Read-your-writes | Guaranteed only within same cluster; use single-cluster app profile |
| Failover | Automatic with multi-cluster routing app profile |
| Max clusters per instance | 8 |

### App Profile Types

| Profile Type | Routing | Read-Your-Writes | Use Case |
|---|---|---|---|
| **Single-cluster routing** | Sends all requests to one specific cluster | ✅ Yes | OLTP, transactional workloads, strong consistency needs |
| **Multi-cluster routing** | Automatically routes to nearest/healthy cluster | ✗ Not guaranteed | HA reads, analytics, non-transactional workloads |
| **Single-cluster + failover** | Primary cluster with fallback to secondary | ✅ On primary | Warm standby DR pattern |

### Creating & Managing App Profiles

```bash
# Create a single-cluster routing app profile (OLTP / transactional)
gcloud bigtable app-profiles create oltp-profile \
  --instance=my-instance \
  --route-to=my-cluster-us-central1-a \
  --restrict-to=my-cluster-us-central1-a \
  --no-failover-radius

# Create a multi-cluster routing app profile (HA reads / analytics)
gcloud bigtable app-profiles create analytics-profile \
  --instance=my-instance \
  --route-any

# Create a single-cluster with allowed transactional writes
gcloud bigtable app-profiles create txn-profile \
  --instance=my-instance \
  --route-to=my-cluster-us-central1-a \
  --transactional-writes

# List app profiles
gcloud bigtable app-profiles list --instance=my-instance

# Describe an app profile
gcloud bigtable app-profiles describe oltp-profile --instance=my-instance

# Delete an app profile
gcloud bigtable app-profiles delete oltp-profile --instance=my-instance
```

### Using App Profiles in Python SDK

```python
from google.cloud import bigtable

# Connect using a specific app profile
client = bigtable.Client(project="my-project", admin=False)
instance = client.instance("my-instance")
table = instance.table("metrics", app_profile_id="analytics-profile")

# All reads/writes on this table object use the analytics-profile routing
rows = table.read_rows(limit=100)
rows.consume_all()
```

### Consistency Guarantees

| Scenario | Guarantee |
|---|---|
| Write + read on same cluster (single-cluster profile) | ✅ Read-your-writes |
| Write on cluster A, read on cluster B | ✗ Eventual — may not see latest write |
| Two writes to same cell from different clusters | Last writer wins (wall-clock time) |
| Multi-cluster routing profile | No read-your-writes guarantee |
| Conditional mutations (CheckAndMutate) | Atomic within a single cluster only |

---

## 10. Performance & Best Practices

### CPU & Storage Utilisation Targets

| Metric | Target | Action if Exceeded |
|---|---|---|
| CPU utilisation | < 70% for SSD; < 35% for HDD | Add nodes or enable autoscaling |
| Storage per node (SSD) | < 70% of 16 TB/node | Add nodes before limit |
| Storage per node (HDD) | < 70% of 64 TB/node | Add nodes before limit |
| Replication lag | < 1 minute | Check source cluster CPU; add nodes |

### Node Sizing Formula

```
Nodes required = MAX(
    ceil(peak_write_QPS    / 10,000),   # SSD write throughput limit
    ceil(peak_read_QPS     / 10,000),   # SSD read throughput limit
    ceil(total_storage_GiB / (16,384 * 0.70)),  # 70% storage utilisation
    3                                   # production minimum
)
```

### Throughput Targets (SSD, ~1 KB rows)

| Operation | Per Node | Notes |
|---|---|---|
| Writes (SetCell) | ~10,000 rows/s | Decreases with larger rows or many mutations per row |
| Point reads | ~10,000 rows/s | Consistent low latency |
| Scan throughput | ~220 MB/s | Increases with larger rows; filter server-side |
| HDD writes | ~500 rows/s | Best for sequential bulk ingest, not random writes |

### Batch Write Best Practices

```python
import concurrent.futures

def write_batch(table, rows):
    errors = table.mutate_rows(rows)
    return errors

# Parallelise batch writes across threads for maximum throughput
BATCH_SIZE = 1000
PARALLELISM = 10

all_rows = [...]  # list of DirectRow objects

batches = [all_rows[i:i+BATCH_SIZE] for i in range(0, len(all_rows), BATCH_SIZE)]

with concurrent.futures.ThreadPoolExecutor(max_workers=PARALLELISM) as executor:
    futures = [executor.submit(write_batch, table, batch) for batch in batches]
    for f in concurrent.futures.as_completed(futures):
        errors = f.result()
        if errors:
            print(f"Errors: {errors}")
```

### Avoiding Hotspots Checklist

```
✅  Row keys start with high-cardinality, evenly distributed prefix
✅  No sequential integers or timestamps as leading key component
✅  Use reversed timestamps for time-series (MAX_INT64 - ts)
✅  Composite keys: entity_id#reversed_ts, NOT ts#entity_id
✅  Key Visualizer shows uniform distribution (no streaks)
✅  Multiple active key prefixes (many active tablets)
✅  Pre-split tables before bulk ingestion
❌  Never use row keys that sort monotonically under write load
❌  Never use UNIX timestamp as the leading key for append-only tables
❌  Avoid "hot" rows written > 1,000 times/second (single row = single tablet)
```

### Scan Optimisation

```python
# ✅ Always use row key prefix to limit scan range
# ✅ Use FamilyNameRegexFilter to fetch only needed families
# ✅ Use CellsPerColumnLimitFilter(1) to get latest version only
# ✅ Set limit on read_rows to avoid unbounded scans
# ✅ Use server-side filters instead of client-side filtering

from google.cloud.bigtable.row_filters import (
    RowFilterChain, FamilyNameRegexFilter, CellsPerColumnLimitFilter
)

optimised_filter = RowFilterChain(filters=[
    FamilyNameRegexFilter("cf"),         # only fetch "cf" family
    CellsPerColumnLimitFilter(1),         # only latest version
])

rows = table.read_rows(
    filter_=optimised_filter,
    limit=1000,
    # Constrain key range via RowSet prefix
)
```

### Connection Management (Singleton Pattern)

```python
# ✅ Create ONE client per process — reuse it across requests
# Each client maintains a gRPC channel pool (default: 4 channels)
_bigtable_client = None

def get_bigtable_table(project, instance_id, table_id):
    global _bigtable_client
    if _bigtable_client is None:
        _bigtable_client = bigtable.Client(project=project, admin=False)
    instance = _bigtable_client.instance(instance_id)
    return instance.table(table_id)
```

> ⚠️ **Warning:** Creating a new `bigtable.Client()` per request is a **severe performance anti-pattern** — each client creates new gRPC channels and performs a new auth handshake. Always use a singleton client per process.

### Autoscaler Configuration

```bash
# Enable autoscaling (recommended for variable workloads)
gcloud bigtable clusters update my-cluster-us-central1-a \
  --instance=my-instance \
  --autoscaling-min-nodes=3 \
  --autoscaling-max-nodes=20 \
  --autoscaling-cpu-target=60 \      # scale up when CPU > 60%
  --autoscaling-storage-target=70    # scale up when storage > 70% per node

# Disable autoscaling (return to manual)
gcloud bigtable clusters update my-cluster-us-central1-a \
  --instance=my-instance \
  --no-autoscaling \
  --num-nodes=5
```

---

## 11. Key Visualizer

### What Is Key Visualizer?

Key Visualizer is a GCP Console tool that shows a **heatmap** of read and write activity across your Bigtable row key space over time. It is the primary diagnostic tool for schema and hotspot problems.

```
Key Visualizer Heatmap Layout:
┌─────────────────────────────────────────────────────────────┐
│  Y-axis: Row key space (top = lexicographically first key)  │
│  X-axis: Time (left = older, right = most recent)           │
│  Colour: Activity intensity (dark = low, bright = high)     │
└─────────────────────────────────────────────────────────────┘
```

### Reading Heatmap Patterns

| Pattern | Appearance | Meaning | Action |
|---|---|---|---|
| **Uniform distribution** ✅ | Even colour across all rows and time | Writes/reads well distributed | No action needed |
| **Leading-edge hotspot** ❌ | Bright streak at the bottom (latest keys) | Sequential/timestamp keys — all writes at the "end" | Redesign row key (reverse timestamp, hash prefix) |
| **Single hot row** ❌ | Thin, persistent horizontal bright line | One row receiving disproportionate traffic | Split logic across multiple rows |
| **Vertical bright bands** ⚠️ | Bright columns across all rows at a time | Traffic spike / batch job | Expected if scheduled; investigate if unexpected |
| **Gradual spread over time** ✅ | Active region slowly moves down Y-axis | Healthy time-series append pattern | No action needed |
| **Diagonal bright line** ❌ | Bright diagonal streak | Correlated key + time monotonic writes | Reverse timestamp in key |
| **Clustered top region only** ❌ | Only the top 10% of key space is active | Most data is cold; new data concentrated | Consider pre-splitting or rebalancing |

### Accessing Key Visualizer

1. GCP Console → Bigtable → your instance → **Key Visualizer** tab
2. Select a **table** and **time range** (available for tables with > 1 GB data accessed in last 7 days)
3. Toggle between **Reads** and **Writes** heatmaps
4. Hover over regions to see row key ranges and operation counts
5. Click a region to zoom in on a narrower key range

> **Tip:** Key Visualizer requires the table to have received at least **1 GB of read or write traffic** in the last 7 days to generate a scan. For low-traffic dev tables, generate load with a test harness before using Key Visualizer.

---

## 12. IAM & Security

### IAM Roles Reference

| Role | Permissions | Use Case |
|---|---|---|
| `roles/bigtable.admin` | Full control: instances, clusters, tables, data | DBAs, infrastructure automation |
| `roles/bigtable.user` | Read + write data; no admin | Application service accounts |
| `roles/bigtable.reader` | Read data only | Reporting, analytics, read-only services |
| `roles/bigtable.viewer` | View instance/cluster metadata; no data | Monitoring, dashboards, ops visibility |

```bash
# Grant bigtable.user to an application service account
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-app@my-project.iam.gserviceaccount.com" \
  --role="roles/bigtable.user"

# Grant bigtable.reader for a reporting service account
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:reporter@my-project.iam.gserviceaccount.com" \
  --role="roles/bigtable.reader"

# Grant bigtable.admin to a specific user
gcloud projects add-iam-policy-binding my-project \
  --member="user:dba@example.com" \
  --role="roles/bigtable.admin"
```

### Encryption

| Type | Managed By | Configuration |
|---|---|---|
| **Google-managed (default)** | Google | Automatic; no configuration needed |
| **CMEK** | Customer KMS key | Specified at cluster creation; applies to all data at rest |

```bash
# Create a cluster with CMEK
gcloud bigtable clusters create my-cmek-cluster \
  --instance=my-instance \
  --zone=us-central1-a \
  --num-nodes=3 \
  --kms-key=projects/my-project/locations/us-central1/keyRings/my-ring/cryptoKeys/my-key
```

> ⚠️ **Warning:** CMEK applies at the **cluster** level, not the instance level. Each cluster in a multi-cluster instance can have a different KMS key. If the key is disabled or destroyed, the cluster data becomes **permanently inaccessible**.

### In-Transit Encryption

All Bigtable API calls use **TLS 1.2+** enforced by Google. No configuration required. The Python client library and all official SDKs use TLS by default. `cbt` uses gRPC over TLS.

### Audit Logging

```bash
# Enable Data Access audit logs for Bigtable
# In IAM & Admin > Audit Logs: enable DATA_READ and DATA_WRITE for
# bigtable.googleapis.com

# Query audit logs via Cloud Logging
gcloud logging read \
  'resource.type="bigtable_instance" AND
   protoPayload.serviceName="bigtable.googleapis.com"' \
  --limit=20 \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.methodName)"
```

---

## 13. Import, Export & Backups

### Export to Cloud Storage via Dataflow

```bash
# Export Bigtable table to Cloud Storage (Avro format)
gcloud dataflow jobs run export-bigtable-avro \
  --gcs-location=gs://dataflow-templates/latest/Cloud_Bigtable_to_GCS_Avro \
  --region=us-central1 \
  --parameters=\
bigtableProjectId=my-project,\
bigtableInstanceId=my-instance,\
bigtableTableId=metrics,\
outputDirectory=gs://my-bucket/bigtable-exports/,\
filenamePrefix=metrics-export-

# Export to Parquet format
gcloud dataflow jobs run export-bigtable-parquet \
  --gcs-location=gs://dataflow-templates/latest/Cloud_Bigtable_to_GCS_Parquet \
  --region=us-central1 \
  --parameters=\
bigtableProjectId=my-project,\
bigtableInstanceId=my-instance,\
bigtableTableId=metrics,\
outputDirectory=gs://my-bucket/bigtable-exports-parquet/,\
numShards=10
```

### Import from Cloud Storage via Dataflow

```bash
# Import Avro files from Cloud Storage into Bigtable
gcloud dataflow jobs run import-bigtable-avro \
  --gcs-location=gs://dataflow-templates/latest/GCS_Avro_to_Cloud_Bigtable \
  --region=us-central1 \
  --parameters=\
bigtableProjectId=my-project,\
bigtableInstanceId=my-instance,\
bigtableTableId=metrics,\
inputFilePattern=gs://my-bucket/bigtable-exports/metrics-export-*.avro
```

### Managed Backups

```bash
# Create an on-demand backup (source table → backup within same cluster)
gcloud bigtable backups create my-backup \
  --instance=my-instance \
  --cluster=my-cluster-us-central1-a \
  --table=metrics \
  --expiration-date=2025-12-31T00:00:00Z

# Create backup with retention duration
gcloud bigtable backups create my-backup-90d \
  --instance=my-instance \
  --cluster=my-cluster-us-central1-a \
  --table=metrics \
  --retention-period=90d

# List backups for a cluster
gcloud bigtable backups list \
  --instance=my-instance \
  --cluster=my-cluster-us-central1-a

# Describe a backup
gcloud bigtable backups describe my-backup \
  --instance=my-instance \
  --cluster=my-cluster-us-central1-a

# Restore backup to a NEW table (cannot overwrite existing)
gcloud bigtable instances tables restore \
  --source=projects/my-project/instances/my-instance/clusters/my-cluster-us-central1-a/backups/my-backup \
  --destination=projects/my-project/instances/my-instance/tables/metrics-restored \
  --destination-instance=my-instance

# Copy backup to another cluster (cross-region DR)
gcloud bigtable backups copy \
  --source-instance=my-instance \
  --source-cluster=my-cluster-us-central1-a \
  --source-backup=my-backup \
  --destination-instance=my-instance \
  --destination-cluster=my-cluster-us-east1-b \
  --destination-backup=my-backup-dr \
  --expiration-date=2025-12-31T00:00:00Z

# Delete a backup
gcloud bigtable backups delete my-backup \
  --instance=my-instance \
  --cluster=my-cluster-us-central1-a
```

### Backup Limits

| Resource | Limit |
|---|---|
| Max backup retention | 90 days |
| Max backups per table | 150 |
| Max backups per cluster | 150 total across all tables |
| Backup storage billing | Charged per GiB/month (same rate as data storage) |

---

## 14. Monitoring & Observability

### Key Cloud Monitoring Metrics

| Metric | Full Name | Alert Threshold | Description |
|---|---|---|---|
| **CPU load** | `bigtable.googleapis.com/cluster/cpu_load` | > 0.70 | Fractional cluster CPU (0–1); add nodes if sustained > 70% |
| **Storage used** | `bigtable.googleapis.com/cluster/disk_load` | > 0.70 | Fractional disk usage per node |
| **Server latency** | `bigtable.googleapis.com/server/latencies` | p99 > 100ms | End-to-end server-side latency by method |
| **Request count** | `bigtable.googleapis.com/server/request_count` | — | QPS by method (ReadRows, MutateRows, etc.) |
| **Error count** | `bigtable.googleapis.com/server/error_count` | > 0 | Errors by status code |
| **Replication lag** | `bigtable.googleapis.com/replication/max_delay` | > 60s | Max replication delay (seconds) per cluster |
| **Node count** | `bigtable.googleapis.com/cluster/node_count` | — | Current node count (useful with autoscaler) |

### Setting Up Alerting

```bash
# Create an alerting policy for CPU > 70% via gcloud (or use Console)
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="Bigtable CPU High" \
  --condition-display-name="CPU > 70%" \
  --condition-filter='resource.type="bigtable_cluster" AND
    metric.type="bigtable.googleapis.com/cluster/cpu_load"' \
  --condition-threshold-value=0.70 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=300s
```

### Useful Monitoring Queries (MQL)

```bash
# View CPU utilisation for a specific instance
gcloud monitoring metrics list \
  --filter="metric.type=bigtable.googleapis.com/cluster/cpu_load"

# View server latency percentiles
gcloud logging read \
  'resource.type="bigtable_cluster" AND
   metric.type="bigtable.googleapis.com/server/latencies"' \
  --limit=10
```

### Observability Checklist

```
Dashboard: Monitor CPU, storage, request rate, and error rate per cluster
Alerting:  CPU > 70%; replication lag > 60s; error rate spike
Key Visualizer: Check weekly for hotspot development or schema drift
Audit Logs: Enable DATA_READ for security-sensitive tables
Latency:   Watch p99 latency; spikes indicate hotspots or missing filters
Node Count: Graph node count vs QPS to validate autoscaler behaviour
```

---

## 15. Client Library — Python

### Installation & Hierarchy

```bash
pip install google-cloud-bigtable
```

```
google.cloud.bigtable.Client          # Project-level client (singleton)
    └── Instance                      # client.instance("my-instance")
        └── Table                     # instance.table("my-table")
            ├── DirectRow             # table.direct_row(row_key)
            ├── ConditionalRow        # table.conditional_row(row_key)
            ├── AppendRow             # table.append_row(row_key)
            └── ColumnFamily          # table.column_family("cf", gc_rule)
```

### Full Reference Example

```python
from google.cloud import bigtable
from google.cloud.bigtable.column_family import MaxVersionsGCRule, MaxAgeGCRule
from google.cloud.bigtable.row_filters import (
    RowFilterChain, FamilyNameRegexFilter, CellsPerColumnLimitFilter,
    TimestampRangeFilter, ValueRegexFilter,
)
from google.cloud.bigtable.row_set import RowSet, RowRange
import datetime, struct, time

# ── Initialise singleton client ──────────────────────────────────────
client   = bigtable.Client(project="my-project", admin=True)
instance = client.instance("my-instance")

# ── Create a table with column families ──────────────────────────────
table = instance.table("sensors")
table.create()

# Column family with GC: latest version only
cf_data = table.column_family("data", gc_rule=MaxVersionsGCRule(1))
cf_data.create()

# Column family with GC: keep 30 days
cf_hist = table.column_family("history",
    gc_rule=MaxAgeGCRule(datetime.timedelta(days=30)))
cf_hist.create()

# ── Write a single row ───────────────────────────────────────────────
MAX_INT64 = 0x7FFFFFFFFFFFFFFF
now_us    = int(time.time() * 1_000_000)
rev_ts    = struct.pack(">q", MAX_INT64 - now_us)
row_key   = b"sensor-A#" + rev_ts

row = table.direct_row(row_key)
row.set_cell("data", "temp",     b"22.5", timestamp_micros=now_us)
row.set_cell("data", "humidity", b"65",   timestamp_micros=now_us)
row.commit()

# ── Batch write 1000 rows ────────────────────────────────────────────
rows = []
for i in range(1000):
    ts = int(time.time() * 1_000_000)
    rk = f"sensor-B#".encode() + struct.pack(">q", MAX_INT64 - ts - i)
    r  = table.direct_row(rk)
    r.set_cell("data", "temp", str(20 + i % 5).encode(), timestamp_micros=ts)
    rows.append(r)

errors = table.mutate_rows(rows)
if errors:
    print(f"{len(errors)} rows failed")

# ── Read a single row ────────────────────────────────────────────────
fetched = table.read_row(row_key)
if fetched:
    temp = fetched.cells["data"][b"temp"][0].value
    print(f"temp = {temp.decode()}")

# ── Scan a row range with filters ────────────────────────────────────
prefix = b"sensor-A#"
end_key = prefix[:-1] + bytes([prefix[-1] + 1])

row_set = RowSet()
row_set.add_row_range(RowRange(start_key=prefix, end_key=end_key))

scan_filter = RowFilterChain(filters=[
    FamilyNameRegexFilter("data"),
    CellsPerColumnLimitFilter(1),     # latest version only
])

result = table.read_rows(row_set=row_set, filter_=scan_filter, limit=100)
result.consume_all()
for rk, row_data in result.rows.items():
    print(rk, row_data.cells)

# ── Delete a row ─────────────────────────────────────────────────────
del_row = table.direct_row(row_key)
del_row.delete()
del_row.commit()

# ── Conditional mutation (CheckAndMutate) ────────────────────────────
cond_row = table.conditional_row(b"sensor-A#status")
pred = ValueRegexFilter(b"alarm")
cond_row.set_cell("data", "ack", b"1", timestamp_micros=now_us, state=True)
cond_row.set_cell("data", "ack", b"0", timestamp_micros=now_us, state=False)
matched = cond_row.commit()
print("Condition matched:", matched)

# ── Use a specific app profile ────────────────────────────────────────
table_analytics = instance.table("sensors", app_profile_id="analytics-profile")
result2 = table_analytics.read_rows(limit=10)
result2.consume_all()

# ── Time-series scan: latest 10 readings for sensor-A ────────────────
prefix = b"sensor-A#"
end_key = prefix[:-1] + bytes([prefix[-1] + 1])
ts_row_set = RowSet()
ts_row_set.add_row_range(RowRange(start_key=prefix, end_key=end_key))

ts_result = table.read_rows(
    row_set=ts_row_set,
    filter_=CellsPerColumnLimitFilter(1),
    limit=10,    # reversed timestamp → first 10 = latest 10 readings
)
ts_result.consume_all()
for rk in sorted(ts_result.rows.keys()):
    print(rk)
```

---

## 16. cbt CLI Quick Reference

### Installation & Setup

```bash
# Install cbt
gcloud components install cbt

# Configure default project and instance
echo "project = my-project"  >> ~/.cbtrc
echo "instance = my-instance" >> ~/.cbtrc

# Override per command
cbt -project=other-project -instance=other-instance ls
```

### Instance & Cluster Operations

```bash
# List instances
cbt listinstances

# Create table
cbt createtable my-table

# List tables
cbt ls

# Describe table (column families, GC policies)
cbt ls my-table

# Delete table
cbt deletetable my-table
```

### Column Family Operations

```bash
# Create column family
cbt createfamily my-table cf

# Set GC policy
cbt setgcpolicy my-table cf maxversions=1
cbt setgcpolicy my-table ts maxage=30d
cbt setgcpolicy my-table audit "(maxversions=3 || maxage=7d)"

# Delete column family
cbt deletefamily my-table cf
```

### Reading Data

```bash
# Point lookup (single row)
cbt lookup my-table "sensor-A#key001"

# Lookup with specific columns
cbt lookup my-table "sensor-A#key001" columns=cf:temp,cf:humidity

# Scan all rows (dangerous on large tables — add count or prefix)
cbt read my-table count=10

# Scan with prefix
cbt read my-table prefix="sensor-A#" count=100

# Scan a key range
cbt read my-table start="sensor-A#" end="sensor-B#"

# Scan with column filter
cbt read my-table prefix="sensor-A#" columns=data:temp count=50

# Read N most recent versions per cell
cbt read my-table prefix="sensor-A#" cells-per-column=3

# Read specific column family only
cbt read my-table prefix="sensor-A#" families=data
```

### Writing Data

```bash
# Write a single cell
cbt set my-table "sensor-A#key001" cf:temp=22.5

# Write multiple cells in one row
cbt set my-table "sensor-A#key001" \
  cf:temp=22.5 cf:humidity=65 cf:pressure=1013.2

# Write with explicit timestamp (Unix microseconds)
cbt set my-table "sensor-A#key001" cf:temp=22.5@1706918400000000
```

### Delete Operations

```bash
# Delete a single row
cbt deleterow my-table "sensor-A#key001"

# Delete a specific cell (all versions)
cbt delete my-table "sensor-A#key001" cf:temp

# Delete cells in a time range
cbt delete my-table "sensor-A#key001" cf:temp \
  start=2025-01-01T00:00:00Z end=2025-02-01T00:00:00Z
```

### App Profile Operations

```bash
# List app profiles
cbt listappprofiles

# Using a specific app profile
cbt -app-profile=analytics-profile read my-table prefix="sensor-A#" count=10
```

### Backup Operations

```bash
# List backups for a cluster
cbt listbackups -cluster=my-cluster-us-central1-a

# Create a backup
# (use gcloud for backup management — cbt has limited backup support)
gcloud bigtable backups create my-backup \
  --instance=my-instance \
  --cluster=my-cluster-us-central1-a \
  --table=my-table \
  --retention-period=30d
```

---

## 17. Pricing Model Summary

> Prices are approximate US regional rates as of 2025. Multi-region replication incurs additional charges. Always verify at [cloud.google.com/bigtable/pricing](https://cloud.google.com/bigtable/pricing).

### Node Pricing (per node per hour)

| Storage Type | Region Type | Price / Node / Hour |
|---|---|---|
| SSD | US regional | $0.65 |
| SSD | Non-US regional | $0.78 |
| HDD | US regional | $0.16 |
| HDD | Non-US regional | $0.19 |

### Storage Pricing (per GiB per month)

| Storage Type | Price / GiB / Month |
|---|---|
| SSD | $0.17 |
| HDD | $0.026 |
| Backup storage | Same as source storage type |

### Replication Pricing

| Charge | Description |
|---|---|
| Node replication | Node cost per destination cluster × node count |
| Write replication | Each mutated row is charged once per additional cluster |
| Storage replication | Full data replicated; storage cost × number of clusters |

> ⚠️ **Warning:** Multi-cluster replication **multiplies** both node and storage costs by the number of clusters. A 3-cluster instance is ~3× the cost of a single-cluster instance.

### Free Tier

Bigtable has **no free tier** — charges begin from the first node-hour.

### Cost Comparison

| Configuration | Nodes | Storage | Monthly Estimate (100 GB SSD) |
|---|---|---|---|
| Single cluster, 3 nodes, SSD | 3 | 100 GB | ~$1,428 nodes + ~$17 storage |
| Single cluster, 3 nodes, HDD | 3 | 100 GB | ~$350 nodes + ~$2.60 storage |
| 2 clusters, 3 nodes each, SSD | 6 | 200 GB | ~$2,856 nodes + ~$34 storage |
| Development, 1 node, SSD | 1 | 10 GB | ~$476 nodes + ~$1.70 storage |

### Cost Optimisation Strategies

| Strategy | Savings |
|---|---|
| Use autoscaler with tight CPU target | Pay only for capacity actually needed |
| Use HDD for sequential-scan / cold-data workloads | ~6× cheaper storage, ~4× cheaper nodes |
| Set aggressive GC policies | Reduce storage by eliminating old cell versions |
| Use development instance for non-production | Lower node cost; no SLA |
| Avoid cross-region replication unless required | Prevents 2–3× cost multiplier |
| Use SSD only when low-latency reads required | HDD is fine for batch ETL and analytics |
| Pre-size nodes before bulk ingestion, scale down after | Avoid paying for peak capacity permanently |
| Consolidate multiple small tables | Reduces per-table overhead and metadata cost |

---

## 18. Common Errors & Troubleshooting

| Error / Issue | Cause | Fix |
|---|---|---|
| `RESOURCE_EXHAUSTED: quota exceeded` | Cluster CPU > 100%; too few nodes for QPS | Scale up nodes immediately; enable autoscaler; reduce write rate; use `cbt` to check node count |
| `DEADLINE_EXCEEDED` | Full table scan (no prefix); large rows; slow filter | Add row key prefix to all scans; use `CellsPerColumnLimitFilter(1)`; add `FamilyNameRegexFilter`; reduce scan range with `limit` |
| Hotspot rows (Key Visualizer streak) | Sequential row keys; timestamp as leading key | Redesign row key: reverse timestamp, add hash/salt prefix; verify with Key Visualizer |
| Large rows causing high latency | Single row > 100 MB; too many column qualifiers per row | Split into multiple rows by time bucket; reduce qualifiers per row; use column families to separate hot/cold data |
| `NOT_FOUND: table not found` | Wrong table name, instance, or project | Run `cbt ls`; verify `-instance` and `-project` flags; check app profile cluster routing |
| `NOT_FOUND: instance not found` | Wrong instance ID or project | Run `gcloud bigtable instances list`; verify project ID in `~/.cbtrc` |
| Replication lag too high | Source cluster CPU saturated; network issue | Add nodes to source cluster; monitor `replication/max_delay` metric; check for write storms |
| GC not reclaiming storage | GC policy too lenient; compaction not triggered | Verify GC policy via `cbt ls TABLE`; manually trigger compaction is not possible — wait for background compaction; consider stronger GC policy |
| `PERMISSION_DENIED` | Service account missing IAM role | Grant `roles/bigtable.user` (write) or `roles/bigtable.reader` (read); verify ADC credentials |
| Batch mutation errors | Individual rows in `mutate_rows` can fail independently | Check `errors` return from `mutate_rows`; retry failed rows with exponential backoff |
| `invalid argument: row key` | Row key contains `\n` or exceeds 4 KB | Sanitise row keys; keep under 64 bytes; never include newlines |
| `cbt read` hangs / very slow | No prefix or range limit; full table scan | Always pass `prefix=`, `start=`/`end=`, or `count=` to `cbt read` |
| Writes succeeding but data not visible | Reading from wrong cluster (multi-cluster instance) | Use single-cluster app profile for read-your-writes; verify app profile cluster routing |
| Storage growing despite GC policy | GC applies lazily during compaction; no forced compaction | Correct behaviour — storage reclaimed asynchronously; verify GC policy is set correctly with `cbt ls TABLE` |
| High p99 latency on point reads | Tablet not in memory (cold read); slow SSTable lookups | Warm up with pre-read scan; check Key Visualizer for hotspots; verify row key doesn't require wide scans |

### Diagnostic Commands

```bash
# Check instance and cluster status
gcloud bigtable instances describe my-instance
gcloud bigtable clusters list --instances=my-instance

# List tables and their column families/GC policies
cbt -project=my-project -instance=my-instance ls
cbt -project=my-project -instance=my-instance ls my-table

# Check current node count
gcloud bigtable clusters describe my-cluster-us-central1-a \
  --instance=my-instance \
  --format="value(serveNodes)"

# Check CPU metric via Cloud Monitoring
gcloud monitoring metrics list \
  --filter="metric.type=bigtable.googleapis.com/cluster/cpu_load"

# List long-running operations (backup, restore)
gcloud bigtable operations list --instance=my-instance

# Verify app profiles
gcloud bigtable app-profiles list --instance=my-instance
```

---

## Quick Reference Card

```
Hierarchy:            Project → Instance → Cluster(s) → Table → Row → Cell
Storage:              Colossus (not on nodes) — nodes are compute only
Row key max:          4 KB hard; keep under 64 bytes
Leading key rule:     NEVER use sequential IDs or raw timestamps as leading key
Reversed timestamp:   MAX_INT64 - timestamp_micros (latest-first scans)
Column families:      Max 100; recommended < 5; GC policy per family
Cell versions:        Multiple per cell; controlled by GC policy
GC timing:            Asynchronous (applied during compaction)
Write throughput:     ~10,000 rows/node/sec (SSD, ~1 KB rows)
CPU target:           < 70% SSD; < 35% HDD
Singleton client:     Create ONE bigtable.Client per process — never per request
Filters:              Applied server-side — always filter to reduce network cost
Replication:          Async, eventual consistency; last-writer-wins per cell
App profiles:         Single-cluster = read-your-writes; Multi-cluster = HA
Backups:              Max 90 days retention; 150 backups per cluster
No free tier:         Charges start from first node-hour
Key Visualizer:       Available for tables with > 1 GB traffic in last 7 days
HBase API:            Bigtable is HBase-compatible (use HBase client with emulator)
```

---

*Reference: [Cloud Bigtable Docs](https://cloud.google.com/bigtable/docs) | [Schema Design](https://cloud.google.com/bigtable/docs/schema-design) | [Python SDK](https://googleapis.dev/python/bigtable/latest/) | [Pricing](https://cloud.google.com/bigtable/pricing) | [Key Visualizer](https://cloud.google.com/bigtable/docs/keyvis-overview)*
