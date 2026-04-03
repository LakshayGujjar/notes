# ☁️ GCP Cloud Spanner — Comprehensive Cheatsheet

> **Audience:** Developers & Cloud Engineers | **Last updated:** 2025
> A single-reference document covering theory, schema design, transactions, CLI, SDK, and best practices.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Instance Configuration & Compute Capacity](#2-instance-configuration--compute-capacity)
3. [Schema Design & Data Modeling](#3-schema-design--data-modeling)
4. [Reading & Writing Data](#4-reading--writing-data)
5. [Transactions & Consistency](#5-transactions--consistency)
6. [Query & SQL Reference](#6-query--sql-reference)
7. [Change Streams](#7-change-streams)
8. [Backups & Recovery](#8-backups--recovery)
9. [IAM & Security](#9-iam--security)
10. [Performance & Best Practices](#10-performance--best-practices)
11. [Client Library — Python](#11-client-library--python)
12. [gcloud CLI Quick Reference](#12-gcloud-cli-quick-reference)
13. [Pricing Model Summary](#13-pricing-model-summary)
14. [Common Errors & Troubleshooting](#14-common-errors--troubleshooting)

---

## 1. Overview & Key Concepts

**Cloud Spanner** is Google's fully managed, globally distributed, horizontally scalable relational database with external consistency (stronger than serializable), SQL support, and ACID transactions — without the tradeoffs traditionally forced by the CAP theorem.

### Cloud Spanner vs Other Databases

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         Relational Database Landscape                            │
├──────────────┬─────────────┬────────────┬────────────┬─────────────┬────────────┤
│  Spanner     │  Cloud SQL  │  AlloyDB   │  Bigtable  │ CockroachDB │  Vitess    │
├──────────────┼─────────────┼────────────┼────────────┼─────────────┼────────────┤
│ Fully managed│ Managed     │ Managed    │ Managed    │ Self-managed│ Self-managed│
│ Global dist. │ Regional    │ Regional   │ Global     │ Global dist.│ Sharded PG │
│ Horiz. scale │ Vertical    │ Vertical   │ Horiz.     │ Horiz.      │ Horiz.     │
│ SQL + ACID   │ SQL + ACID  │ SQL + ACID │ NoSQL      │ SQL + ACID  │ SQL + ACID │
│ Ext. consist.│ Serializable│Serializable│ Row-level  │ Serializable│Serializable│
│ TrueTime     │ NTP-based   │ NTP-based  │ N/A        │ HLC-based   │ NTP-based  │
│ $$$$–$$$$$   │ $–$$$       │ $$$–$$$$   │ $–$$$      │ $$–$$$      │ $$–$$$     │
└──────────────┴─────────────┴────────────┴────────────┴─────────────┴────────────┘
```

### When to Use Cloud Spanner

- Financial systems requiring **global ACID transactions** with zero data loss
- Applications needing **horizontal read/write scaling** beyond single-instance limits
- Global products requiring **strongly consistent reads** across multiple continents
- Systems where **schema changes** must be non-blocking and online
- Replacing **sharded MySQL/PostgreSQL** that has outgrown its operational complexity

### Core Terminology

| Term | Description |
|---|---|
| **Instance** | Top-level Spanner resource. Contains databases; defines compute (nodes/PUs) and region config. |
| **Database** | A collection of tables, indexes, and schemas within an instance. |
| **Node** | Unit of compute capacity. 1 node = 1,000 Processing Units (PUs). |
| **Processing Unit (PU)** | Granular compute unit. Minimum 100 PUs. Enables cost-efficient small deployments. |
| **Instance Configuration** | Defines where data is stored and replicated (regional, dual-region, multi-region). |
| **Split** | A contiguous range of rows Spanner distributes across servers for load balancing. |
| **Interleaving** | A schema feature that physically co-locates child rows with their parent row. |
| **Mutation** | A set of insert/update/delete operations buffered and committed atomically. |
| **DML** | SQL-based data manipulation (INSERT, UPDATE, DELETE) executed within a transaction. |
| **DDL** | Schema definition language (CREATE TABLE, ALTER TABLE, CREATE INDEX). |
| **Read-Write Transaction** | Serializable transaction that reads and writes; can be aborted and retried. |
| **Read-Only Transaction** | Consistent snapshot read with no locking; never aborts. |
| **Stale Read** | Read from a past timestamp — lower latency, no lock contention. |
| **Blind Write** | Mutation with no preceding read in the same transaction — highest throughput. |
| **TrueTime** | Google's globally synchronised atomic clock API; enables external consistency. |
| **External Consistency** | Stronger than serializability — if Txn B starts after Txn A commits, B sees A's writes. |
| **Change Stream** | A Spanner feature that captures row-level changes (insert/update/delete) over time. |
| **Partitioned DML** | Special DML mode for large-scale updates/deletes across millions of rows. |
| **Version Retention** | Spanner retains old row versions for a configurable window (up to 7 days), enabling stale reads. |
| **Database Role** | Fine-grained, database-level RBAC entities (similar to PostgreSQL roles). |

---

## 2. Instance Configuration & Compute Capacity

### Instance Configuration Types

| Type | Replicas | Geography | Use Case |
|---|---|---|---|
| **Regional** | 3 read-write replicas (1 region, 3 zones) | Single region | Lowest cost; 99.99% SLA |
| **Dual-region** | 4 read-write + 1 witness | 2 specific regions | HA across 2 regions; same continent |
| **Multi-region** | 5+ read-write + witness replicas | Multiple continents | Global apps; 99.999% SLA |

### Named Instance Configurations

| Config Name | Type | Regions Covered | SLA |
|---|---|---|---|
| `regional-us-central1` | Regional | Iowa | 99.99% |
| `regional-europe-west1` | Regional | Belgium | 99.99% |
| `regional-asia-east1` | Regional | Taiwan | 99.99% |
| `nam4` | Dual-region | Northern Virginia + South Carolina | 99.999% |
| `eur3` | Dual-region | Belgium + Netherlands | 99.999% |
| `nam6` | Dual-region | N. Virginia + N. California | 99.999% |
| `nam-eur-asia1` | Multi-region | US + Europe + Asia | 99.999% |
| `nam10` | Multi-region | 5 US regions | 99.999% |
| `eur5` | Multi-region | 5 European regions | 99.999% |
| `asia1` | Dual-region | Tokyo + Osaka | 99.999% |

> **Tip:** Use `gcloud spanner instance-configs list` to see all available configurations for your project and their current status.

### Processing Units vs Nodes

```
100 PU  = smallest billable unit (0.1 node)
1,000 PU = 1 Node
  │
  ├── Up to 2 TB storage per node (regional)
  ├── ~2,000 QPS simple reads per node
  └── ~1,800 QPS simple writes per node
```

### Capacity Planning Guidelines

| Workload | Starting Capacity | Storage per Node |
|---|---|---|
| Dev / prototype | 100 PU | 10 GB (free tier) |
| Small production | 1,000 PU (1 node) | Up to 2 TB |
| Mid-size production | 3,000–5,000 PU | Up to 10 TB |
| High-traffic global | 10,000+ PU or autoscale | Scale linearly |

### Creating and Managing Instances

```bash
# Create a regional instance (1 node)
gcloud spanner instances create my-instance \
  --config=regional-us-central1 \
  --description="Production instance" \
  --nodes=1

# Create with Processing Units (minimum 100 PU — fine-grained)
gcloud spanner instances create my-dev-instance \
  --config=regional-us-central1 \
  --description="Dev instance" \
  --processing-units=100

# Create a multi-region instance
gcloud spanner instances create my-global-instance \
  --config=nam-eur-asia1 \
  --description="Global production" \
  --nodes=3

# Update instance capacity (scale up)
gcloud spanner instances update my-instance --nodes=3

# Update with autoscaler (min/max nodes)
gcloud spanner instances update my-instance \
  --autoscaling-min-nodes=1 \
  --autoscaling-max-nodes=5 \
  --autoscaling-high-priority-cpu-target=65 \
  --autoscaling-storage-target=95

# List instances
gcloud spanner instances list

# Describe instance
gcloud spanner instances describe my-instance

# Delete instance
gcloud spanner instances delete my-instance
```

---

## 3. Schema Design & Data Modeling

### Supported Data Types

| Type | GoogleSQL | PostgreSQL Dialect | Notes |
|---|---|---|---|
| Boolean | `BOOL` | `BOOLEAN` | |
| Integer | `INT64` | `BIGINT` | 64-bit signed |
| Float | `FLOAT64` | `FLOAT8` | IEEE 754 double |
| Exact decimal | `NUMERIC` | `NUMERIC` | 29 digits, 9 scale |
| String | `STRING(N)` | `VARCHAR(N)` | Max 2,621,440 chars; use `MAX` for max |
| Binary | `BYTES(N)` | `BYTEA` | Max 10,485,760 bytes |
| Date | `DATE` | `DATE` | Civil date (no time zone) |
| Timestamp | `TIMESTAMP` | `TIMESTAMPTZ` | Microsecond precision, UTC |
| JSON | `JSON` | `JSONB` | Unordered key-value |
| Array | `ARRAY<T>` | `T[]` | Homogeneous, nullable elements |
| Struct | `STRUCT<...>` | ✗ | Query results only; not storable |

### Primary Key Design — Avoiding Hotspots

> ⚠️ **Warning:** Sequential keys (auto-increment IDs, timestamps as leading PK) create **write hotspots** — all new rows land on the same split, saturating a single server. Spanner cannot split on value boundaries automatically fast enough.

```sql
-- ❌ BAD: Sequential integer creates hotspot
CREATE TABLE Orders (
  OrderId   INT64 NOT NULL,        -- auto-increment — all writes to one split
  CreatedAt TIMESTAMP,
) PRIMARY KEY (OrderId);

-- ✅ GOOD: UUID (random) distributes writes evenly
CREATE TABLE Orders (
  OrderId   STRING(36) NOT NULL,   -- UUID v4 — naturally distributed
  CreatedAt TIMESTAMP,
) PRIMARY KEY (OrderId);

-- ✅ GOOD: Bit-reversed sequence (preserves ordering, avoids hotspot)
-- Use Spanner's built-in bit_reversed_positive sequence
CREATE SEQUENCE order_seq OPTIONS (
  sequence_kind = 'bit_reversed_positive'
);
CREATE TABLE Orders (
  OrderId   INT64 DEFAULT (GET_NEXT_SEQUENCE_VALUE(SEQUENCE order_seq)),
  CreatedAt TIMESTAMP,
) PRIMARY KEY (OrderId);

-- ✅ GOOD: Hash prefix + timestamp
CREATE TABLE Events (
  ShardId   INT64 NOT NULL,        -- hash(entity_id) % N_SHARDS
  EventTime TIMESTAMP NOT NULL,
  EventId   STRING(36) NOT NULL,
) PRIMARY KEY (ShardId, EventTime, EventId);
```

### Interleaved Tables

Interleaving physically co-locates child rows with their parent — eliminates cross-server joins and reduces latency for parent-child queries.

```sql
-- Parent table
CREATE TABLE Customers (
  CustomerId STRING(36) NOT NULL,
  Name       STRING(255),
  Email      STRING(255),
) PRIMARY KEY (CustomerId);

-- Child table INTERLEAVED IN PARENT — must share parent PK prefix
CREATE TABLE Orders (
  CustomerId STRING(36) NOT NULL,   -- must be first, matches parent PK
  OrderId    STRING(36) NOT NULL,
  Amount     NUMERIC,
  CreatedAt  TIMESTAMP,
) PRIMARY KEY (CustomerId, OrderId),
  INTERLEAVE IN PARENT Customers ON DELETE CASCADE;

-- Grandchild — can be interleaved N levels deep (up to 7)
CREATE TABLE OrderItems (
  CustomerId STRING(36) NOT NULL,
  OrderId    STRING(36) NOT NULL,
  ItemId     STRING(36) NOT NULL,
  Quantity   INT64,
) PRIMARY KEY (CustomerId, OrderId, ItemId),
  INTERLEAVE IN PARENT Orders ON DELETE CASCADE;
```

**Interleaving Rules:**
- Child PK must be a proper extension of parent PK (same leading columns)
- `ON DELETE CASCADE` — deleting parent deletes all children
- `ON DELETE NO ACTION` — deleting parent with children fails (default)
- Maximum interleaving depth: 7 levels

### Secondary Indexes

```sql
-- Global secondary index (indexed column not in PK prefix)
CREATE INDEX OrdersByCreatedAt
  ON Orders (CreatedAt DESC);

-- Index with STORING clause (copies columns to avoid index-back reads)
CREATE INDEX OrdersByAmount
  ON Orders (Amount DESC)
  STORING (CreatedAt, CustomerId);

-- NULL_FILTERED index (excludes NULL values — saves index storage)
CREATE NULL_FILTERED INDEX ActiveOrders
  ON Orders (Status)
  WHERE Status IS NOT NULL;

-- Unique index
CREATE UNIQUE INDEX UniqueEmail
  ON Customers (Email);

-- Index on interleaved child — interleave index with parent for locality
CREATE INDEX OrdersByDate
  ON Orders (CustomerId, CreatedAt DESC),
  INTERLEAVE IN Customers;
```

### Foreign Keys

```sql
-- Foreign key constraint (non-interleaved, enforced by Spanner)
CREATE TABLE Shipments (
  ShipmentId STRING(36) NOT NULL,
  OrderId    STRING(36) NOT NULL,
  CONSTRAINT FK_Order FOREIGN KEY (OrderId)
    REFERENCES Orders (OrderId) ON DELETE CASCADE,
) PRIMARY KEY (ShipmentId);
```

> **Tip:** Prefer **interleaving** over foreign keys for parent-child relationships with frequent co-access patterns. Foreign keys enforce referential integrity but add overhead on writes; interleaving does both and enables locality.

### TTL — Row Deletion Policy

```sql
-- Auto-delete rows where DeletedAt is older than 30 days
ALTER TABLE Events
  ADD ROW DELETION POLICY (OLDER_THAN(EventTime, INTERVAL 30 DAY));

-- Remove TTL policy
ALTER TABLE Events DROP ROW DELETION POLICY;
```

### Generated Columns

```sql
CREATE TABLE Products (
  ProductId   STRING(36) NOT NULL,
  PriceCents  INT64,
  TaxRate     FLOAT64,
  -- Computed at write time; stored physically
  TotalCents  INT64 AS (CAST(PriceCents * (1 + TaxRate) AS INT64)) STORED,
) PRIMARY KEY (ProductId);
```

---

## 4. Reading & Writing Data

### DML — GoogleSQL Dialect

```sql
-- INSERT
INSERT INTO Customers (CustomerId, Name, Email)
VALUES ('uuid-1', 'Alice Smith', 'alice@example.com');

-- INSERT OR UPDATE (upsert)
INSERT OR UPDATE INTO Customers (CustomerId, Name, Email)
VALUES ('uuid-1', 'Alice Smith Updated', 'alice@example.com');

-- UPDATE
UPDATE Customers
SET Email = 'newalice@example.com'
WHERE CustomerId = 'uuid-1';

-- DELETE
DELETE FROM Orders
WHERE CreatedAt < TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY);
```

### DML — PostgreSQL Dialect

```sql
-- PostgreSQL dialect uses standard SQL (on PostgreSQL-interface databases)
INSERT INTO customers (customer_id, name, email)
VALUES ('uuid-1', 'Alice Smith', 'alice@example.com')
ON CONFLICT (customer_id) DO UPDATE SET name = EXCLUDED.name;

UPDATE customers SET email = 'new@example.com' WHERE customer_id = 'uuid-1';

DELETE FROM orders WHERE created_at < NOW() - INTERVAL '90 days';
```

### Mutations API (Python)

Mutations are the lowest-level write API — buffered client-side, sent in one RPC at commit.

```python
from google.cloud import spanner

client = spanner.Client(project="my-project")
instance = client.instance("my-instance")
database = instance.database("my-database")

# Insert mutation
with database.batch() as batch:
    batch.insert(
        table="Customers",
        columns=("CustomerId", "Name", "Email"),
        values=[
            ("uuid-1", "Alice Smith", "alice@example.com"),
            ("uuid-2", "Bob Jones",  "bob@example.com"),
        ],
    )

# Insert-or-update (upsert)
with database.batch() as batch:
    batch.insert_or_update(
        table="Customers",
        columns=("CustomerId", "Name", "Email"),
        values=[("uuid-1", "Alice Updated", "alice@example.com")],
    )

# Update mutation
with database.batch() as batch:
    batch.update(
        table="Customers",
        columns=("CustomerId", "Email"),
        values=[("uuid-1", "alice-new@example.com")],
    )

# Delete mutation
with database.batch() as batch:
    batch.delete(
        table="Customers",
        keyset=spanner.KeySet(keys=[["uuid-1"]]),
    )
```

### Read API (Key Lookups & Range Reads)

```python
# Single key lookup
with database.snapshot() as snapshot:
    result = snapshot.read(
        table="Customers",
        columns=["CustomerId", "Name", "Email"],
        keyset=spanner.KeySet(keys=[["uuid-1"]]),
    )
    for row in result:
        print(row)

# Key range read (prefix scan)
from google.cloud.spanner_v1 import KeyRange, KeySet

key_range = KeyRange(
    start_closed=["cust-A"],   # inclusive start
    end_open=["cust-Z"],       # exclusive end
)
with database.snapshot() as snapshot:
    result = snapshot.read(
        table="Customers",
        columns=["CustomerId", "Name"],
        keyset=KeySet(ranges=[key_range]),
    )
```

### Stale Reads

```python
import datetime

# Exact staleness — read data as of 15 seconds ago (lower latency)
with database.snapshot(exact_staleness=datetime.timedelta(seconds=15)) as snapshot:
    result = snapshot.execute_sql("SELECT COUNT(*) FROM Orders")

# Bounded staleness — read data no older than 10 seconds
with database.snapshot(max_staleness=datetime.timedelta(seconds=10)) as snapshot:
    result = snapshot.execute_sql("SELECT * FROM Products")

# Min read timestamp — read at or after a specific time
read_time = datetime.datetime.utcnow() - datetime.timedelta(minutes=5)
with database.snapshot(min_read_timestamp=read_time) as snapshot:
    result = snapshot.execute_sql("SELECT * FROM Events LIMIT 100")
```

### Partitioned DML (Large-Scale Updates)

```python
# Partitioned DML — runs DELETE/UPDATE across millions of rows in parallel
# Does NOT guarantee atomicity across all rows — best for bulk cleanup
row_count = database.execute_partitioned_dml(
    "DELETE FROM AuditLogs WHERE LogTime < TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)"
)
print(f"Deleted approximately {row_count} rows")
```

> ⚠️ **Warning:** Partitioned DML is **not atomic** across the entire table. It processes rows in partitions, each committed independently. Never use it when you need all-or-nothing semantics. Use only for bulk deletes/updates where partial application is acceptable.

---

## 5. Transactions & Consistency

### External Consistency

Spanner's **external consistency** is stronger than serializable isolation:
- If transaction T1 commits **before** T2 starts (in real wall-clock time), then T2 is guaranteed to see T1's writes.
- Achieved via **TrueTime** — GPS + atomic clocks synchronised across all Google data centres, with a known bounded uncertainty (ε typically < 7ms).

### Read-Write Transaction Lifecycle

```
Client                              Spanner
  │                                    │
  │── begin transaction ──────────────►│
  │                                    │ (acquire shared locks on read)
  │── read rows ────────────────────►  │
  │◄─ return rows ─────────────────────│
  │                                    │
  │   (buffer mutations locally)       │
  │                                    │
  │── commit (send mutations) ────────►│
  │                                    │ (acquire write locks)
  │                                    │ (apply TrueTime commit timestamp)
  │◄─ commit timestamp ────────────────│
  │                                    │ (release locks)
```

### Transaction Isolation

| Mode | Isolation Level | Locking | Abortable |
|---|---|---|---|
| Read-write | Serializable + external consistency | Yes (shared read, exclusive write) | Yes (ABORTED on contention) |
| Read-only (strong) | Serializable snapshot at latest commit | No | No |
| Read-only (stale) | Snapshot at past timestamp | No | No |
| Partitioned DML | Best-effort; per-partition atomic | Exclusive per partition | No |

### Retry Loop Pattern (Python)

`ABORTED` errors must always be retried from the beginning of the transaction.

```python
from google.api_core import exceptions as core_exceptions
from google.cloud import spanner
import time

def transfer_funds(database, from_id: str, to_id: str, amount: int):
    """Read-write transaction with exponential backoff retry."""

    def _transfer(transaction):
        # Read current balances
        from_row = transaction.execute_sql(
            "SELECT Balance FROM Accounts WHERE AccountId = @id",
            params={"id": from_id},
            param_types={"id": spanner.param_types.STRING},
        ).one()

        to_row = transaction.execute_sql(
            "SELECT Balance FROM Accounts WHERE AccountId = @id",
            params={"id": to_id},
            param_types={"id": spanner.param_types.STRING},
        ).one()

        if from_row[0] < amount:
            raise ValueError("Insufficient funds")

        # Buffer mutations (not sent until commit)
        transaction.update(
            table="Accounts",
            columns=["AccountId", "Balance"],
            values=[[from_id, from_row[0] - amount]],
        )
        transaction.update(
            table="Accounts",
            columns=["AccountId", "Balance"],
            values=[[to_id, to_row[0] + amount]],
        )

    # run_in_transaction handles ABORTED retries automatically
    database.run_in_transaction(_transfer)
```

> **Tip:** Always use `database.run_in_transaction()` (Python SDK) rather than managing retries manually. It automatically retries on `ABORTED` with exponential backoff and uses the same transaction object, satisfying Spanner's requirement to replay mutations identically on retry.

### Reducing Abort Rates

| Strategy | Why it helps |
|---|---|
| Keep transactions short | Fewer locks held for less time |
| Read only what you need | Fewer shared locks = less contention |
| Use blind writes when possible | No reads = no shared locks = never aborted |
| Use stale reads for non-critical reads | Stale reads don't take locks |
| Avoid reading and writing the same key in a hot loop | High contention on hot rows |
| Batch mutations together | Fewer round-trips, atomic commit |

---

## 6. Query & SQL Reference

### GoogleSQL vs PostgreSQL Dialect

| Feature | GoogleSQL | PostgreSQL Interface |
|---|---|---|
| Dialect | Spanner-native | PostgreSQL-compatible |
| Type for string | `STRING(N)` | `VARCHAR(N)` / `TEXT` |
| Integer | `INT64` | `BIGINT` |
| Upsert syntax | `INSERT OR UPDATE` | `INSERT ... ON CONFLICT` |
| Array literal | `ARRAY[1,2,3]` | `ARRAY[1,2,3]` |
| Timestamp function | `CURRENT_TIMESTAMP()` | `NOW()` |
| Struct type | `STRUCT<...>` | Not supported |
| JSON | `JSON` | `JSONB` |
| Query hints | `@{FORCE_INDEX=...}` | `@{FORCE_INDEX=...}` |
| Driver | `google-cloud-spanner` | `pgx`, `psycopg2` via PGAdapter |

### JOINs and Subqueries

```sql
-- INNER JOIN (prefer interleaved tables for co-located joins)
SELECT c.Name, o.OrderId, o.Amount
FROM Customers c
JOIN Orders o ON c.CustomerId = o.CustomerId
WHERE c.Email LIKE '%@example.com'
ORDER BY o.Amount DESC
LIMIT 100;

-- LEFT JOIN
SELECT c.CustomerId, c.Name, COUNT(o.OrderId) AS OrderCount
FROM Customers c
LEFT JOIN Orders o ON c.CustomerId = o.CustomerId
GROUP BY c.CustomerId, c.Name;

-- Correlated subquery
SELECT * FROM Products p
WHERE p.Price > (
  SELECT AVG(Price) FROM Products
  WHERE Category = p.Category
);
```

### Common Table Expressions (CTEs)

```sql
WITH RecentOrders AS (
  SELECT CustomerId, SUM(Amount) AS TotalSpend
  FROM Orders
  WHERE CreatedAt > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  GROUP BY CustomerId
),
TopCustomers AS (
  SELECT CustomerId FROM RecentOrders WHERE TotalSpend > 10000
)
SELECT c.Name, r.TotalSpend
FROM Customers c
JOIN RecentOrders r ON c.CustomerId = r.CustomerId
WHERE c.CustomerId IN (SELECT CustomerId FROM TopCustomers)
ORDER BY r.TotalSpend DESC;
```

### Window Functions

```sql
-- Running total per customer
SELECT
  CustomerId,
  OrderId,
  Amount,
  SUM(Amount) OVER (
    PARTITION BY CustomerId
    ORDER BY CreatedAt
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS RunningTotal,
  ROW_NUMBER() OVER (PARTITION BY CustomerId ORDER BY Amount DESC) AS RankByAmount
FROM Orders;
```

### UNNEST (Arrays)

```sql
-- Expand array column into rows
SELECT ProductId, Tag
FROM Products,
UNNEST(Tags) AS Tag          -- Tags is an ARRAY<STRING> column
WHERE Tag = 'electronics';

-- Unnest with index
SELECT ProductId, idx, Tag
FROM Products,
UNNEST(Tags) AS Tag WITH OFFSET idx;
```

### Query Optimizer Hints

```sql
-- Force use of a specific secondary index
SELECT * FROM Orders@{FORCE_INDEX=OrdersByCreatedAt}
WHERE CreatedAt > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY);

-- Force a specific JOIN method
SELECT *
FROM Customers c
JOIN@{JOIN_METHOD=HASH_JOIN} Orders o
ON c.CustomerId = o.CustomerId;

-- Disable index use (force full table scan — use with caution)
SELECT * FROM Orders@{FORCE_INDEX=_BASE_TABLE}
WHERE Amount > 1000;

-- Set statement-level timeout (milliseconds)
SELECT * FROM Orders@{STATEMENT_TIMEOUT=5000}
ORDER BY CreatedAt DESC LIMIT 10;
```

### Query Plan & Statistics

```sql
-- Get query execution plan (GoogleSQL)
EXPLAIN SELECT * FROM Orders WHERE CustomerId = 'uuid-1';

-- Get query plan with stats (actual execution statistics)
EXPLAIN ANALYZE SELECT * FROM Orders WHERE CustomerId = 'uuid-1';
```

```bash
# View query statistics via gcloud (top CPU-consuming queries)
gcloud spanner databases execute-sql my-database \
  --instance=my-instance \
  --sql="SELECT text, avg_latency_seconds, execution_count
         FROM SPANNER_SYS.QUERY_STATS_TOP_MINUTE
         ORDER BY avg_cpu_seconds DESC LIMIT 10;"
```

---

## 7. Change Streams

### What Are Change Streams?

Change Streams capture **insert, update, and delete** operations on Spanner tables at the row or column level — providing a durable, ordered log of mutations. They enable CDC (change data capture), event sourcing, audit trails, and real-time replication.

```
Spanner Table Write
        │
        ▼
  Change Stream
  (retention: up to 7 days)
        │
        ├──► Dataflow Pipeline (recommended)
        │         └──► BigQuery / Pub/Sub / Bigtable / etc.
        │
        └──► Spanner API (direct read for low-volume use cases)
```

### Creating Change Streams (DDL)

```sql
-- Track all changes to all tables in the database
CREATE CHANGE STREAM AllTableChanges
  FOR ALL
  OPTIONS (retention_period = '7d');

-- Track specific tables
CREATE CHANGE STREAM OrderChanges
  FOR Orders, OrderItems
  OPTIONS (retention_period = '3d');

-- Track specific columns only
CREATE CHANGE STREAM CustomerEmailChanges
  FOR Customers(Email, UpdatedAt)
  OPTIONS (retention_period = '1d');

-- Capture old values on UPDATE (for diff-based CDC)
CREATE CHANGE STREAM OrderChangesWithOld
  FOR Orders
  OPTIONS (
    retention_period = '7d',
    value_capture_type = 'OLD_AND_NEW_VALUES'
  );
```

### Value Capture Types

| Type | Captures | Use Case |
|---|---|---|
| `NEW_VALUES` | Only new column values (default) | Downstream replication |
| `OLD_AND_NEW_VALUES` | Both old and new column values | Auditing, diff computation |
| `NEW_ROW` | Entire row after change | Snapshot-style CDC |

### Reading Change Stream via API (Python)

```python
from google.cloud import spanner

client = spanner.Client(project="my-project")
instance = client.instance("my-instance")
database = instance.database("my-database")

import datetime

start_time = datetime.datetime.utcnow() - datetime.timedelta(hours=1)
end_time   = datetime.datetime.utcnow()

with database.snapshot() as snapshot:
    results = snapshot.execute_sql(
        """SELECT ChangeRecord
           FROM READ_OrderChanges(
             start_timestamp => @start,
             end_timestamp   => @end,
             partition_token => NULL,
             heartbeat_milliseconds => 10000
           )""",
        params={"start": start_time, "end": end_time},
        param_types={
            "start": spanner.param_types.TIMESTAMP,
            "end":   spanner.param_types.TIMESTAMP,
        },
    )
    for row in results:
        print(row)
```

### Reading via Dataflow (Recommended for Production)

```bash
# Deploy Spanner Change Streams → BigQuery Dataflow template
gcloud dataflow flex-template run change-stream-to-bq \
  --template-file-gcs-location=gs://dataflow-templates/latest/flex/Spanner_Change_Streams_to_BigQuery \
  --region=us-central1 \
  --parameters=\
spannerInstanceId=my-instance,\
spannerDatabase=my-database,\
spannerMetadataInstanceId=my-instance,\
spannerMetadataDatabase=my-metadata-db,\
spannerChangeStreamName=OrderChanges,\
bigQueryDataset=my_dataset
```

### Managing Change Streams

```sql
-- Alter retention period
ALTER CHANGE STREAM OrderChanges
  SET OPTIONS (retention_period = '5d');

-- Add a table to existing change stream
ALTER CHANGE STREAM OrderChanges
  FOR Orders, OrderItems, Shipments;

-- Drop a change stream
DROP CHANGE STREAM OrderChanges;
```

---

## 8. Backups & Recovery

### Managed Backups

Spanner backups are **full, consistent snapshots** taken at a point in time. They are stored within the same instance configuration and replicated accordingly.

```bash
# Create an on-demand backup
gcloud spanner backups create my-backup \
  --instance=my-instance \
  --database=my-database \
  --expiration-date=2025-12-31T23:59:59Z

# Create backup with retention duration (instead of absolute date)
gcloud spanner backups create my-backup \
  --instance=my-instance \
  --database=my-database \
  --retention-period=30d

# Create backup at a specific version time (PITR-style)
gcloud spanner backups create my-pitr-backup \
  --instance=my-instance \
  --database=my-database \
  --version-time=2025-01-15T10:00:00Z \
  --expiration-date=2025-06-01T00:00:00Z

# List backups
gcloud spanner backups list --instance=my-instance

# Describe a backup
gcloud spanner backups describe my-backup --instance=my-instance

# Delete a backup
gcloud spanner backups delete my-backup --instance=my-instance
```

### Scheduled Backups

```bash
# Create a backup schedule (runs daily at 2 AM UTC, keeps for 14 days)
gcloud spanner backup-schedules create daily-backup \
  --instance=my-instance \
  --database=my-database \
  --cron="0 2 * * *" \
  --backup-retention=14d \
  --full-backup

# List backup schedules
gcloud spanner backup-schedules list \
  --instance=my-instance \
  --database=my-database
```

### Restoring from a Backup

```bash
# Restore to a NEW database (cannot overwrite existing)
gcloud spanner databases restore my-restored-db \
  --destination-instance=my-instance \
  --source-backup=projects/my-project/instances/my-instance/backups/my-backup

# Monitor restore operation progress
gcloud spanner operations list \
  --instance=my-instance \
  --database=my-restored-db \
  --filter="@type:RestoreDatabaseMetadata"
```

### Cross-Region Backup Copy

```bash
# Copy backup to a different instance (different region config)
gcloud spanner backups copy \
  --source-instance=my-instance \
  --source-backup=my-backup \
  --destination-instance=my-dr-instance \
  --destination-backup=my-dr-backup \
  --expiration-date=2025-12-31T00:00:00Z
```

### PITR via Version Retention

Spanner retains old row versions for up to 7 days (configurable). Use stale reads to query historical data.

```bash
# Set version retention to 7 days on a database
gcloud spanner databases update my-database \
  --instance=my-instance \
  --version-retention-period=7d
```

```python
import datetime

# Read the database as it was 4 hours ago (within version retention window)
read_time = datetime.datetime.utcnow() - datetime.timedelta(hours=4)
with database.snapshot(read_timestamp=read_time) as snapshot:
    result = snapshot.execute_sql(
        "SELECT * FROM Orders WHERE OrderId = @id",
        params={"id": "uuid-abc"},
        param_types={"id": spanner.param_types.STRING},
    )
    for row in result:
        print(row)
```

### Export / Import via Dataflow (Avro)

```bash
# Export database to Cloud Storage (Avro format)
gcloud dataflow jobs run export-spanner-db \
  --gcs-location=gs://dataflow-templates/latest/Cloud_Spanner_to_GCS_Avro \
  --region=us-central1 \
  --parameters=\
instanceId=my-instance,\
databaseId=my-database,\
outputDir=gs://my-bucket/spanner-exports/

# Import from Cloud Storage (Avro)
gcloud dataflow jobs run import-spanner-db \
  --gcs-location=gs://dataflow-templates/latest/GCS_Avro_to_Cloud_Spanner \
  --region=us-central1 \
  --parameters=\
instanceId=my-instance,\
databaseId=my-database,\
inputDir=gs://my-bucket/spanner-exports/
```

---

## 9. IAM & Security

### IAM Roles Reference

| Role | Permissions |
|---|---|
| `roles/spanner.admin` | Full control of instances, databases, backups |
| `roles/spanner.databaseAdmin` | Create/delete databases, manage schemas and users |
| `roles/spanner.databaseUser` | Read and write data; execute DML |
| `roles/spanner.databaseReader` | Read data and schema; no writes |
| `roles/spanner.viewer` | View instance/database metadata; no data access |
| `roles/spanner.backupAdmin` | Create, restore, delete backups |

```bash
# Grant databaseUser to an application service account
gcloud spanner databases add-iam-policy-binding my-database \
  --instance=my-instance \
  --member="serviceAccount:my-app@my-project.iam.gserviceaccount.com" \
  --role="roles/spanner.databaseUser"

# Grant admin at instance level
gcloud spanner instances add-iam-policy-binding my-instance \
  --member="user:dba@example.com" \
  --role="roles/spanner.admin"
```

### Database Roles (Fine-Grained Access Control)

Database roles provide table- and column-level access within Spanner — useful for multi-tenant schemas or least-privilege application accounts.

```sql
-- Create a database role
CREATE ROLE orders_reader;

-- Grant SELECT on specific tables to the role
GRANT SELECT ON TABLE Orders TO ROLE orders_reader;
GRANT SELECT ON TABLE OrderItems TO ROLE orders_reader;

-- Grant only specific columns
GRANT SELECT (CustomerId, Amount, Status) ON TABLE Orders TO ROLE orders_reader;

-- Grant DML permissions
CREATE ROLE orders_writer;
GRANT INSERT, UPDATE ON TABLE Orders TO ROLE orders_writer;
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE OrderItems TO ROLE orders_writer;

-- Grant a database role to an IAM principal
GRANT ROLE orders_reader TO USER `user@example.com`;
GRANT ROLE orders_writer TO USER `sa@my-project.iam.gserviceaccount.com`;

-- Revoke role
REVOKE ROLE orders_reader FROM USER `user@example.com`;

-- List grants
SELECT * FROM INFORMATION_SCHEMA.ROLE_TABLE_GRANTS;
```

```python
# Connect using a database role (Python SDK)
database = instance.database(
    "my-database",
    database_role="orders_reader"   # restricts to role's permissions
)
```

### CMEK (Customer-Managed Encryption Keys)

```bash
# Create a KMS key for Spanner
gcloud kms keyrings create spanner-keyring --location=us-central1
gcloud kms keys create spanner-key \
  --location=us-central1 \
  --keyring=spanner-keyring \
  --purpose=encryption

# Create a Spanner instance with CMEK
gcloud spanner instances create my-cmek-instance \
  --config=regional-us-central1 \
  --nodes=1 \
  --kms-key=projects/my-project/locations/us-central1/keyRings/spanner-keyring/cryptoKeys/spanner-key
```

> ⚠️ **Warning:** If the CMEK key is destroyed or access is revoked, **the Spanner instance becomes permanently inaccessible**. All data, backups created with that key, and exports become unreadable.

### VPC Service Controls

```bash
# Add Spanner to a VPC-SC service perimeter
gcloud access-context-manager perimeters update my-perimeter \
  --add-restricted-services=spanner.googleapis.com \
  --policy=POLICY_ID
```

### Audit Logging

```bash
# Enable Data Access audit logs for Spanner
# In IAM & Admin > Audit Logs: enable DATA_READ and DATA_WRITE for
# the Cloud Spanner API (spanner.googleapis.com)

# Query audit logs via Cloud Logging
gcloud logging read \
  'resource.type="spanner_instance" AND
   protoPayload.serviceName="spanner.googleapis.com"' \
  --limit=50 \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.methodName)"
```

---

## 10. Performance & Best Practices

### Hotspot Avoidance — Key Design Patterns

| Pattern | Approach | Example |
|---|---|---|
| UUID v4 | Random 128-bit key | `GENERATE_UUID()` |
| Bit-reversed sequence | Monotonic but distributed | `bit_reversed_positive` sequence |
| Hash prefix | `hash(id) % N` as leading key | `(ShardId, Timestamp, Id)` |
| Avoid timestamp leading key | Never use `CreatedAt` as first PK component | Add shard or UUID before timestamp |
| Avoid sequential integers | Use `bit_reversed_positive` sequences | `GET_NEXT_SEQUENCE_VALUE(SEQUENCE s)` |

### CPU Utilisation Targets

| Instance Type | Recommended Max CPU | Action if Exceeded |
|---|---|---|
| Regional | 65% high-priority | Scale up nodes/PUs |
| Multi-region (per replica) | 45% high-priority | Scale up; uneven replica distribution |
| With autoscaler | Set target at 65% / 45% | Autoscaler adds capacity automatically |

```bash
# Check CPU utilisation via Cloud Monitoring
gcloud monitoring metrics list \
  --filter="metric.type=spanner.googleapis.com/instance/cpu/utilization_by_priority"
```

### Optimal Transaction Patterns

```python
# ✅ BEST: Blind writes (no reads) — maximum throughput, never aborted
with database.batch() as batch:
    batch.insert_or_update(
        table="Events",
        columns=["EventId", "EventTime", "Payload"],
        values=[("uuid-1", datetime.datetime.utcnow(), "data")],
    )

# ✅ GOOD: Batch multiple mutations in one transaction
with database.batch() as batch:
    batch.insert("TableA", cols, rows_a)  # all committed atomically
    batch.insert("TableB", cols, rows_b)
    batch.delete("TableC", keyset)

# ⚠️  AVOID: Long-running read-write transactions
# Every second of lock-holding increases abort probability
```

### Batch Writes at Scale

```python
# Write 10,000 rows in batches of 500 mutations each
BATCH_SIZE = 500
rows = [("uuid-" + str(i), f"Name {i}", i * 100) for i in range(10_000)]

for i in range(0, len(rows), BATCH_SIZE):
    chunk = rows[i : i + BATCH_SIZE]
    with database.batch() as batch:
        batch.insert(
            table="Products",
            columns=["ProductId", "Name", "Price"],
            values=chunk,
        )
```

### Session Pool Configuration

```python
from google.cloud.spanner_v1 import Client
from google.cloud.spanner_v1.pool import FixedSizePool, BurstyPool

# FixedSizePool — pre-warms all sessions (low latency, predictable)
client = Client(project="my-project")
pool = FixedSizePool(size=10, default_timeout=30)
database = instance.database("my-database", pool=pool)

# BurstyPool — creates sessions on demand up to max
from google.cloud.spanner_v1.pool import BurstyPool
pool = BurstyPool(target_size=10, max_size=50)
database = instance.database("my-database", pool=pool)
```

> **Tip:** Each session consumes ~1 MB of memory on Spanner servers. For regional instances with 1 node (≈ 10,000 session limit), size your pool to leave headroom. Aim for no more than 100 sessions per 1,000 PU as a rule of thumb.

### Index Design Best Practices

| Practice | Reason |
|---|---|
| Add `STORING` clause for frequently projected columns | Avoids index-back reads (full row fetch after index scan) |
| Use `NULL_FILTERED` for sparse indexes | Reduces index size; avoids indexing NULL values |
| Interleave index with parent table | Improves locality for parent-scoped queries |
| Avoid indexing high-write columns | Every indexed write is 2 writes (table + index row) |
| Limit secondary indexes per table | Each index doubles write amplification; keep to < 5 per table |

### Autoscaler

```bash
# Enable autoscaling (removes need for manual node management)
gcloud spanner instances update my-instance \
  --autoscaling-min-processing-units=1000 \
  --autoscaling-max-processing-units=10000 \
  --autoscaling-high-priority-cpu-target=65 \
  --autoscaling-storage-target=95
```

---

## 11. Client Library — Python

### Installation & Hierarchy

```bash
pip install google-cloud-spanner
```

```
google.cloud.spanner.Client          # Project-level client
    └── Instance                     # Spanner instance
        └── Database                 # A database in the instance
            └── Session (pooled)     # Connection to execute operations
                └── Transaction / Snapshot / Batch
```

### Full CRUD Example

```python
from google.cloud import spanner
from google.cloud.spanner_v1 import param_types
import datetime

# ── Initialise client hierarchy ─────────────────────────────────────
client   = spanner.Client(project="my-project")
instance = client.instance("my-instance")
database = instance.database("my-database")

# ── Create a database with schema ───────────────────────────────────
instance.database(
    "my-new-database",
    ddl_statements=[
        """CREATE TABLE Customers (
             CustomerId STRING(36) NOT NULL,
             Name       STRING(255) NOT NULL,
             Email      STRING(255),
             CreatedAt  TIMESTAMP OPTIONS (allow_commit_timestamp=true),
           ) PRIMARY KEY (CustomerId)"""
    ],
).create().result(timeout=300)

# ── Insert rows with mutations ───────────────────────────────────────
with database.batch() as batch:
    batch.insert(
        table="Customers",
        columns=("CustomerId", "Name", "Email", "CreatedAt"),
        values=[
            ("uuid-1", "Alice", "alice@example.com", spanner.COMMIT_TIMESTAMP),
            ("uuid-2", "Bob",   "bob@example.com",   spanner.COMMIT_TIMESTAMP),
        ],
    )

# ── Read rows by key ─────────────────────────────────────────────────
with database.snapshot() as snapshot:
    keyset = spanner.KeySet(keys=[["uuid-1"]])
    result = snapshot.read(
        table="Customers",
        columns=["CustomerId", "Name", "Email"],
        keyset=keyset,
    )
    for row in result:
        print(f"{row[0]}: {row[1]} <{row[2]}>")

# ── Execute a SQL query ──────────────────────────────────────────────
with database.snapshot() as snapshot:
    results = snapshot.execute_sql(
        "SELECT CustomerId, Name FROM Customers WHERE Email LIKE @domain",
        params={"domain": "%@example.com"},
        param_types={"domain": param_types.STRING},
    )
    for row in results:
        print(row)

# ── Read-write transaction with automatic retry ──────────────────────
def update_email(transaction, customer_id: str, new_email: str):
    row_ct = transaction.execute_update(
        "UPDATE Customers SET Email = @email WHERE CustomerId = @id",
        params={"email": new_email, "id": customer_id},
        param_types={"email": param_types.STRING, "id": param_types.STRING},
    )
    print(f"Updated {row_ct} row(s)")

database.run_in_transaction(update_email, "uuid-1", "alice-new@example.com")

# ── Read-only transaction with stale read ────────────────────────────
with database.snapshot(
    exact_staleness=datetime.timedelta(seconds=30)
) as snapshot:
    results = snapshot.execute_sql("SELECT COUNT(*) AS cnt FROM Customers")
    for row in results:
        print(f"Customer count (30s stale): {row[0]}")

# ── Batch write (high throughput) ────────────────────────────────────
rows = [("uuid-" + str(i), f"User {i}", f"user{i}@example.com") for i in range(1000)]
BATCH_SIZE = 200
for i in range(0, len(rows), BATCH_SIZE):
    with database.batch() as batch:
        batch.insert_or_update(
            table="Customers",
            columns=("CustomerId", "Name", "Email"),
            values=rows[i : i + BATCH_SIZE],
        )

# ── Read a change stream ─────────────────────────────────────────────
start = datetime.datetime.utcnow() - datetime.timedelta(minutes=10)
end   = datetime.datetime.utcnow()

with database.snapshot() as snapshot:
    results = snapshot.execute_sql(
        """SELECT ChangeRecord
           FROM READ_CustomerChanges(
             start_timestamp => @start,
             end_timestamp   => @end,
             partition_token => NULL,
             heartbeat_milliseconds => 5000
           )""",
        params={"start": start, "end": end},
        param_types={"start": param_types.TIMESTAMP, "end": param_types.TIMESTAMP},
    )
    for row in results:
        print(row)
```

---

## 12. gcloud CLI Quick Reference

### Instance Operations

```bash
# CREATE instance
gcloud spanner instances create INSTANCE \
  --config=CONFIG --nodes=N --description="DESC"

# LIST instances
gcloud spanner instances list

# DESCRIBE instance
gcloud spanner instances describe INSTANCE

# UPDATE capacity
gcloud spanner instances update INSTANCE --nodes=N
gcloud spanner instances update INSTANCE --processing-units=N

# ENABLE autoscaling
gcloud spanner instances update INSTANCE \
  --autoscaling-min-nodes=1 \
  --autoscaling-max-nodes=5

# DELETE instance (deletes all databases and backups)
gcloud spanner instances delete INSTANCE

# LIST instance configs
gcloud spanner instance-configs list

# DESCRIBE a config
gcloud spanner instance-configs describe regional-us-central1
```

### Database Operations

```bash
# CREATE database (GoogleSQL)
gcloud spanner databases create DB --instance=INSTANCE

# CREATE database (PostgreSQL dialect)
gcloud spanner databases create DB \
  --instance=INSTANCE \
  --database-dialect=POSTGRESQL

# LIST databases
gcloud spanner databases list --instance=INSTANCE

# DESCRIBE database
gcloud spanner databases describe DB --instance=INSTANCE

# DELETE database
gcloud spanner databases delete DB --instance=INSTANCE

# EXECUTE SQL query
gcloud spanner databases execute-sql DB \
  --instance=INSTANCE \
  --sql="SELECT COUNT(*) FROM Customers"

# APPLY DDL schema update
gcloud spanner databases ddl update DB \
  --instance=INSTANCE \
  --ddl="ALTER TABLE Orders ADD COLUMN Status STRING(20)"

# GET DDL (describe schema)
gcloud spanner databases ddl describe DB --instance=INSTANCE
```

### Backup Operations

```bash
# CREATE backup
gcloud spanner backups create BACKUP \
  --instance=INSTANCE \
  --database=DB \
  --retention-period=30d

# CREATE backup at specific version time
gcloud spanner backups create BACKUP \
  --instance=INSTANCE \
  --database=DB \
  --version-time=2025-01-15T10:00:00Z \
  --expiration-date=2025-06-01T00:00:00Z

# LIST backups
gcloud spanner backups list --instance=INSTANCE

# DESCRIBE backup
gcloud spanner backups describe BACKUP --instance=INSTANCE

# RESTORE database from backup
gcloud spanner databases restore NEW_DB \
  --destination-instance=INSTANCE \
  --source-backup=projects/PROJECT/instances/INSTANCE/backups/BACKUP

# COPY backup cross-region
gcloud spanner backups copy \
  --source-instance=INSTANCE \
  --source-backup=BACKUP \
  --destination-instance=DR_INSTANCE \
  --destination-backup=DR_BACKUP \
  --expiration-date=DATE

# DELETE backup
gcloud spanner backups delete BACKUP --instance=INSTANCE

# CREATE backup schedule
gcloud spanner backup-schedules create SCHEDULE \
  --instance=INSTANCE \
  --database=DB \
  --cron="0 2 * * *" \
  --backup-retention=14d \
  --full-backup

# LIST backup schedules
gcloud spanner backup-schedules list \
  --instance=INSTANCE --database=DB
```

### Operations & Monitoring

```bash
# LIST long-running operations on instance
gcloud spanner operations list --instance=INSTANCE

# LIST operations on a database
gcloud spanner operations list \
  --instance=INSTANCE \
  --database=DB

# DESCRIBE an operation
gcloud spanner operations describe OPERATION_ID \
  --instance=INSTANCE

# CANCEL a long-running operation
gcloud spanner operations cancel OPERATION_ID \
  --instance=INSTANCE
```

---

## 13. Pricing Model Summary

> Prices are approximate US regional rates as of 2025. Multi-region and dual-region are significantly higher. Always verify at [cloud.google.com/spanner/pricing](https://cloud.google.com/spanner/pricing).

### Compute Pricing

| Config Type | Unit | Price per Unit / Hour |
|---|---|---|
| Regional — Node | 1,000 PU | $0.90 |
| Regional — Processing Unit | 100 PU | $0.09 |
| Dual-region — Node | 1,000 PU | $3.00 |
| Dual-region — Processing Unit | 100 PU | $0.30 |
| Multi-region — Node | 1,000 PU | $9.00 |
| Multi-region — Processing Unit | 100 PU | $0.90 |

### Storage Pricing

| Resource | Regional | Multi-Region |
|---|---|---|
| Storage | $0.30 / GB / month | $0.50 / GB / month |
| Backup storage | $0.30 / GB / month | $0.30 / GB / month |

### Free Tier (per month)

| Resource | Free Allocation |
|---|---|
| Compute | 90 PU-hours / month (~10 hours at 100 PU minimum, regional only) |
| Storage | 10 GB |
| Backup | 10 GB |

> **Tip:** The Spanner free tier is intentionally limited — it covers development and small-scale testing only. A 100 PU regional instance running 24/7 costs ~$65/month (minus free tier hours).

### Regional vs Multi-Region Cost Comparison

| Scenario | Regional (1 node) | Multi-Region (1 node) | Ratio |
|---|---|---|---|
| Compute / month | ~$648 | ~$6,480 | 10× |
| Storage / GB / month | $0.30 | $0.50 | 1.67× |
| SLA | 99.99% | 99.999% | +0.009% |

### Cost Optimisation Strategies

| Strategy | Potential Savings |
|---|---|
| Use autoscaler (min/max PU range) | Pay only for capacity actually needed |
| Start with 100 PU for non-production | 10× cheaper than 1 node for dev/test |
| Use stale reads where strong consistency not needed | Reduces CPU usage by 10–30% |
| Use regional config for non-global apps | Up to 10× cheaper than multi-region |
| Batch mutations vs row-by-row DML | Fewer round-trips, less CPU |
| Set TTL / row deletion policies | Auto-delete old data; reduce storage cost |
| Use `STORING` indexes efficiently | Avoids full-row reads; reduces compute |
| Schedule backups with appropriate retention | Don't keep 365-day backups if 30 days suffices |
| Use Committed Use Discounts (1 or 3 year) | Up to 20% discount on compute |

---

## 14. Common Errors & Troubleshooting

| Error / Issue | Cause | Fix |
|---|---|---|
| `ABORTED: Transaction was aborted` | Write-write conflict; lock contention on hot rows | Always retry via `run_in_transaction()`; keep transactions short; use blind writes where possible |
| `DEADLINE_EXCEEDED` | Query or transaction took longer than the client/server timeout | Optimise query (add index, reduce scan); increase client timeout; break large transactions into smaller ones |
| `RESOURCE_EXHAUSTED: Session limit reached` | Session pool exhausted; too many concurrent requests | Increase pool size; ensure sessions are returned to pool (use context managers); check for session leaks |
| `RESOURCE_EXHAUSTED: Quota exceeded` | Per-minute mutation count or node CPU quota hit | Scale up nodes/PUs; reduce write rate; implement client-side rate limiting |
| High CPU on specific splits (hotspot) | Sequential keys funnelling all writes to one server | Redesign primary key: use UUID, bit-reversed sequence, or hash prefix; avoid leading timestamp keys |
| `FAILED_PRECONDITION: Schema change already in progress` | Concurrent DDL operations | Spanner only allows one schema change at a time; poll the operation and retry after it completes |
| Slow queries / full table scan | Missing secondary index; large range scan | Run `EXPLAIN ANALYZE`; add targeted secondary index with `STORING`; use `@{FORCE_INDEX=...}` hint to test |
| Change stream gaps / missed changes | Consumer fell behind retention window (7 days max) | Increase retention period; deploy Dataflow consumer; alert on consumer lag |
| Backup creation failed | Instance CPU too high during backup; transient error | Retry; schedule backups during low-traffic windows; ensure adequate node capacity |
| `INVALID_ARGUMENT: Row size limit exceeded` | A single row exceeds 256 MB | Redesign schema: move large BLOBs to Cloud Storage; store only references in Spanner |
| `NOT_FOUND: Database not found` | Wrong instance or database name | Verify with `gcloud spanner databases list --instance=INSTANCE` |
| Schema migration breaks reads | ALTER TABLE changes a column type incompatibly | Use additive-only migrations: add new column → backfill → switch app → drop old column |
| `PERMISSION_DENIED` on table | IAM role lacks access; database role not granted | Check `roles/spanner.databaseUser`; verify database role grants in `INFORMATION_SCHEMA.ROLE_TABLE_GRANTS` |
| Interleaved child delete fails | `ON DELETE NO ACTION` set on parent-child; parent has children | Use `ON DELETE CASCADE`; or delete children first before parent |
| Replica lag in multi-region | Network partition or regional issue | Use `INFORMATION_SCHEMA.SPANNER_STATISTICS`; Spanner self-heals — monitor and alert on replication lag metric |

### Diagnostic Queries

```sql
-- Top queries by CPU in last minute
SELECT text, avg_latency_seconds, avg_cpu_seconds, execution_count
FROM SPANNER_SYS.QUERY_STATS_TOP_MINUTE
ORDER BY avg_cpu_seconds DESC
LIMIT 10;

-- Active sessions and transactions
SELECT session_name, transaction_tag, read_timestamp, transaction_type
FROM SPANNER_SYS.SESSIONS_STATS_MINUTE
WHERE is_orphaned = FALSE
LIMIT 50;

-- Lock wait information
SELECT
  lock_requester_transaction_tag,
  lock_holder_transaction_tag,
  lock_type
FROM SPANNER_SYS.LOCK_STATS_TOP_MINUTE
LIMIT 20;

-- Storage utilisation per table
SELECT table_name, used_bytes
FROM INFORMATION_SCHEMA.TABLE_SPANNER_STATISTICS
ORDER BY used_bytes DESC;
```

```bash
# Check instance-level CPU metric
gcloud monitoring metrics list \
  --filter="metric.type=spanner.googleapis.com/instance/cpu/utilization"

# List recent errors from Cloud Logging
gcloud logging read \
  'resource.type="spanner_instance" AND severity>=ERROR' \
  --limit=20 \
  --format="table(timestamp,severity,jsonPayload.message)"

# Check long-running operations (schema changes, backups)
gcloud spanner operations list \
  --instance=my-instance \
  --filter="done=false"
```

---

## Quick Reference Card

```
Hierarchy:           Project → Instance → Database → Table
Compute units:       100 PU minimum; 1,000 PU = 1 Node
Key rule:            Never use sequential integers or timestamps as leading PK
Interleaving depth:  Up to 7 levels
Max row size:        256 MB
Max DB size:         Unlimited (scales with nodes)
Storage per node:    Up to 2 TB (regional)
PITR window:         Up to 7 days (version retention)
Backup retention:    Up to 1 year
Change stream retention: Up to 7 days
CPU target (regional):   < 65% high-priority
CPU target (multi-region): < 45% per replica
Transaction ABORTED: Always retry from beginning (run_in_transaction)
Partitioned DML:     Not atomic — use only for bulk cleanup
Stale read staleness: 1 second to 1 hour; reduces latency and CPU
Free tier:           90 PU-hours/month + 10 GB storage (regional)
Dialect options:     GoogleSQL (default) | PostgreSQL
```

---

*Reference: [Cloud Spanner Docs](https://cloud.google.com/spanner/docs) | [Pricing](https://cloud.google.com/spanner/pricing) | [Python Client](https://googleapis.dev/python/spanner/latest/) | [Schema Best Practices](https://cloud.google.com/spanner/docs/schema-design)*
