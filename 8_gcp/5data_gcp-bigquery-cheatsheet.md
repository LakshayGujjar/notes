# 📊 GCP BigQuery — Comprehensive Cheatsheet

> **Audience:** Data engineers, analysts, and architects working with GCP BigQuery.
> **Last updated:** March 2026 | Covers GoogleSQL, BigQuery editions, BQML, and all core features.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Data Organization & Schema Design](#2-data-organization--schema-design)
3. [Querying & SQL Reference](#3-querying--sql-reference)
4. [Loading & Exporting Data](#4-loading--exporting-data)
5. [Performance Optimization](#5-performance-optimization)
6. [Access Control & Security](#6-access-control--security)
7. [Cost Management](#7-cost-management)
8. [Advanced Features](#8-advanced-features)
9. [bq CLI & API Quick Reference](#9-bq-cli--api-quick-reference)
10. [Monitoring, Auditing & Observability](#10-monitoring-auditing--observability)
11. [Pricing Summary](#11-pricing-summary)
12. [Quick Reference & Cheatsheet Tables](#12-quick-reference--cheatsheet-tables)

---

## 1. Overview & Key Concepts

### What is BigQuery?

BigQuery is GCP's **fully managed, serverless, petabyte-scale data warehouse**. It separates compute from storage, enabling independent scaling of each, with near-zero infrastructure management.

**Core architectural pillars:**

| Component | Technology | Role |
|---|---|---|
| **Query Engine** | Dremel | Massively parallel SQL execution |
| **Storage Format** | Capacitor (columnar) | Compressed, encoded column storage |
| **Distributed Storage** | Colossus (GFS successor) | Durable, geo-replicated object store |
| **Compute Orchestration** | Borg | Slot scheduling and job execution |
| **Shuffle** | Jupiter network | High-speed inter-node data exchange |

---

### Key Components

```
GCP Project
├── Dataset (logical namespace + access boundary)
│   ├── Table (native, external, snapshot, clone)
│   ├── View (virtual table = saved SQL query)
│   ├── Materialized View (precomputed + cached)
│   └── Routine (UDF, stored procedure, remote function)
└── Jobs (query, load, export, copy — async execution units)
```

---

### Storage vs. Compute Separation

```
┌─────────────────────────────────────────────────────────┐
│                    COMPUTE LAYER                         │
│   Dremel workers (slots) — scale independently          │
│   Reservations / On-demand / Autoscaling                │
└────────────────────────┬────────────────────────────────┘
                         │  Jupiter (fast shuffle)
┌────────────────────────▼────────────────────────────────┐
│                    STORAGE LAYER                         │
│   Colossus — Capacitor columnar files                    │
│   Shared across projects; billed separately              │
└─────────────────────────────────────────────────────────┘
```

> 💡 Because storage and compute are decoupled, you pay for storage 24/7 but only pay for compute **when queries run** (on-demand) or via reserved slot capacity.

---

### BigQuery Editions

| Edition | Target | Key Features |
|---|---|---|
| **Standard** | Exploration, ad-hoc | On-demand or reservations, no advanced features |
| **Enterprise** | Production workloads | + CMEK, BI Engine, column-level security, Reservations |
| **Enterprise Plus** | Mission-critical | + 1-year/3-year commitments, max autoscaling, priority support |

---

### Slot-Based Execution Model

A **slot** = 1 unit of BigQuery compute (CPU + memory + networking).

- Query submitted → BigQuery optimizer creates a **query plan** (DAG of stages)
- Each stage runs as **parallel workers** consuming slots
- **On-demand:** slots allocated dynamically per query (shared pool, up to 2,000 slots/project by default)
- **Capacity:** you reserve a fixed or auto-scaled slot pool (Reservations)

```
Query Plan:
  Stage 1: Read + Filter (1,000 slots × 0.1s)
       ↓
  Stage 2: Aggregate (500 slots × 0.05s)
       ↓
  Stage 3: Join + Sort (200 slots × 0.02s)
       ↓
  Stage 4: Output
```

> 💡 More slots = faster wall-clock time for large queries. But slots don't reduce bytes scanned (that's about query writing, not capacity).

---

### On-Demand vs. Capacity Pricing

| Model | Best For | Billing Unit | Predictability |
|---|---|---|---|
| **On-demand** | Sporadic, variable workloads | $ per TB scanned | Low (surprise bills possible) |
| **Capacity (Reservations)** | Steady, predictable workloads | $ per slot-hour | High (fixed cost) |
| **Autoscaling** | Spiky but frequent workloads | Baseline + burst slots | Medium |

> 💡 Rule of thumb: if your monthly on-demand bill exceeds the cost of ~2,000 reserved slots, switch to capacity pricing.

---

## 2. Data Organization & Schema Design

### Datasets

```bash
# Create a regional dataset (US)
bq mk \
  --dataset \
  --location=US \
  --description="Production analytics dataset" \
  --default_table_expiration=7776000 \   # 90 days in seconds
  MY_PROJECT:MY_DATASET

# Create in EU multi-region
bq mk --dataset --location=EU MY_PROJECT:eu_dataset

# Set dataset-level access (add a reader)
bq update --set_label env:prod MY_PROJECT:MY_DATASET
```

| Location Type | Examples | Use Case |
|---|---|---|
| **Regional** | `us-central1`, `europe-west1` | Data residency requirements |
| **Multi-regional** | `US`, `EU` | Highest availability, no region pinning |

> ⚠️ Dataset location is **immutable** after creation. Plan carefully — cross-region data movement requires export → reimport.

---

### Table Types

| Type | Description | Use Case |
|---|---|---|
| **Native** | Managed BigQuery storage (Capacitor) | Default — best performance |
| **External** | Federated query over GCS, Bigtable, Drive, Spanner | Raw data lake, avoid duplication |
| **View** | Virtual table — SQL query evaluated at read time | Logical abstraction, access control |
| **Materialized View** | Precomputed query result, auto-refreshed | Accelerate repeated aggregation queries |
| **Snapshot** | Point-in-time read-only copy | Backup, audit, before/after comparison |
| **Clone** | Writable copy (zero-storage until divergence) | Dev/test environments, branch datasets |

---

### Data Types Reference

| Type | Description | Example Value |
|---|---|---|
| `STRING` | UTF-8 text, up to 16 MB | `'hello'` |
| `BYTES` | Binary data, up to 16 MB | `B'\x00\xFF'` |
| `INT64` / `INTEGER` | 64-bit signed integer | `42` |
| `FLOAT64` / `FLOAT` | IEEE 754 double | `3.14` |
| `NUMERIC` | 38 digits, 9 decimal places | `NUMERIC '1234.56'` |
| `BIGNUMERIC` | 76 digits, 38 decimal places | Financial precision |
| `BOOL` / `BOOLEAN` | `TRUE` or `FALSE` | `TRUE` |
| `DATE` | Calendar date | `'2026-03-16'` |
| `TIME` | Time of day (no timezone) | `'14:30:00'` |
| `DATETIME` | Date + time (no timezone) | `'2026-03-16T14:30:00'` |
| `TIMESTAMP` | Date + time + UTC offset | `'2026-03-16 14:30:00 UTC'` |
| `GEOGRAPHY` | WGS84 geographic point/shape | `ST_GEOGPOINT(-122.4, 37.7)` |
| `JSON` | Semi-structured JSON | `JSON '{"key": "value"}'` |
| `STRUCT` / `RECORD` | Named fields (nested object) | `STRUCT('Alice' AS name, 30 AS age)` |
| `ARRAY` | Ordered list of same-type elements | `[1, 2, 3]` |

**Field modes:**

| Mode | Description |
|---|---|
| `NULLABLE` | Field can be NULL (default) |
| `REQUIRED` | Field must have a value; NULLs rejected at load |
| `REPEATED` | Field is an array (0 or more values) |

---

### Nested & Repeated Fields

```sql
-- Schema with STRUCT and ARRAY
CREATE TABLE orders (
  order_id      STRING    NOT NULL,
  created_at    TIMESTAMP NOT NULL,
  customer      STRUCT<                      -- nested STRUCT
    id          STRING,
    name        STRING,
    email       STRING
  >,
  line_items    ARRAY<STRUCT<                -- repeated STRUCT (array of structs)
    product_id  STRING,
    quantity    INT64,
    unit_price  NUMERIC
  >>
);

-- Query nested fields
SELECT
  order_id,
  customer.name,
  customer.email
FROM orders;

-- UNNEST array into rows
SELECT
  o.order_id,
  item.product_id,
  item.quantity,
  item.unit_price
FROM orders o,
UNNEST(o.line_items) AS item;          -- each item becomes a row
```

> 💡 Nested/repeated fields are BigQuery's native denormalization. They eliminate expensive JOINs and reduce storage by keeping related data co-located in the same row.

---

### Schema Auto-Detection vs. Explicit

```bash
# Auto-detect schema from CSV (convenient but imprecise)
bq load \
  --autodetect \
  --source_format=CSV \
  MY_PROJECT:MY_DATASET.my_table \
  gs://my-bucket/data.csv

# Explicit schema from JSON file (recommended for production)
bq load \
  --source_format=NEWLINE_DELIMITED_JSON \
  --schema=schema.json \
  MY_PROJECT:MY_DATASET.my_table \
  gs://my-bucket/data.ndjson
```

`schema.json` example:
```json
[
  { "name": "user_id",    "type": "STRING",    "mode": "REQUIRED" },
  { "name": "event_ts",   "type": "TIMESTAMP", "mode": "REQUIRED" },
  { "name": "amount",     "type": "NUMERIC",   "mode": "NULLABLE" },
  { "name": "tags",       "type": "STRING",    "mode": "REPEATED" }
]
```

---

### Schema Evolution

```bash
# Add a new NULLABLE column (always safe)
bq update --schema new_schema.json MY_PROJECT:MY_DATASET.my_table

# Relax a column from REQUIRED to NULLABLE
bq update \
  --schema '[{"name":"amount","type":"NUMERIC","mode":"NULLABLE"}]' \
  MY_PROJECT:MY_DATASET.my_table
```

| Change | Supported? |
|---|---|
| Add NULLABLE column | ✅ Always allowed |
| Add REQUIRED column | ❌ Not allowed (would break existing rows) |
| Relax REQUIRED → NULLABLE | ✅ Allowed |
| Change data type | ❌ Not allowed (must recreate) |
| Rename column | ❌ Not allowed natively (use ALTER COLUMN in preview) |
| Delete column | ❌ Not allowed (DROP COLUMN available in preview) |

---

### Table Partitioning

```sql
-- Ingestion-time partitioning (auto-partition by load date)
CREATE TABLE events_ingestion
PARTITION BY _PARTITIONDATE
OPTIONS (partition_expiration_days = 365)
AS SELECT * FROM source_table;

-- Column-based DATE partitioning
CREATE TABLE events_by_date (
  event_id   STRING,
  event_date DATE,
  payload    JSON
)
PARTITION BY event_date
OPTIONS (partition_expiration_days = 90);

-- TIMESTAMP column partitioning (hourly granularity)
CREATE TABLE events_hourly
PARTITION BY TIMESTAMP_TRUNC(event_ts, HOUR)
AS SELECT * FROM source;

-- INTEGER RANGE partitioning
CREATE TABLE sales_by_region (
  region_id  INT64,
  sale_amt   NUMERIC
)
PARTITION BY RANGE_BUCKET(region_id, GENERATE_ARRAY(0, 100, 10));
```

| Partition Type | Column Type | Granularity |
|---|---|---|
| Ingestion-time | `_PARTITIONDATE` (auto) | Day |
| Column DATE | DATE | Day |
| Column TIMESTAMP | TIMESTAMP/DATETIME | Hour, Day, Month, Year |
| Integer Range | INT64 | Custom ranges |

> 💡 A table can have **up to 10,000 partitions**. Use partition expiration to auto-drop old data and control storage costs.

---

### Table Clustering

```sql
-- Partition + Cluster (most powerful combo)
CREATE TABLE user_events (
  user_id    STRING,
  event_ts   TIMESTAMP,
  event_type STRING,
  country    STRING,
  platform   STRING
)
PARTITION BY DATE(event_ts)
CLUSTER BY user_id, event_type, country;   -- up to 4 cluster columns
```

**How clustering works:** BigQuery sorts data within each partition by cluster columns, creating co-located blocks. Queries filtering on cluster columns skip irrelevant blocks → fewer bytes scanned.

> 💡 Put the **most selective** filter column first in the CLUSTER BY list. Clustering benefits queries that filter or aggregate on those columns.

---

## 3. Querying & SQL Reference

### GoogleSQL vs. Legacy SQL

| Feature | GoogleSQL (Default) | Legacy SQL |
|---|---|---|
| Standard | ANSI SQL compliant | BigQuery-specific |
| Backtick quoting | `` `project.dataset.table` `` | `[project:dataset.table]` |
| Subqueries | ✅ Full support | ⚠️ Limited |
| ARRAY/STRUCT | ✅ Native | ❌ Not supported |
| Recommended | ✅ **Always use this** | ❌ Avoid |

```bash
# Force GoogleSQL (default since 2023, but explicit is clear)
bq query --use_legacy_sql=false 'SELECT 1'
```

---

### Built-in Functions Quick Reference

**String Functions:**
```sql
CONCAT(s1, s2)                  -- 'hello' || ' world'
SUBSTR(str, pos, len)           -- SUBSTR('hello', 1, 3) → 'hel'
UPPER(str) / LOWER(str)
TRIM(str) / LTRIM / RTRIM
REPLACE(str, old, new)
REGEXP_EXTRACT(str, pattern)    -- Extract first match
REGEXP_REPLACE(str, pattern, r) -- Regex substitution
SPLIT(str, delimiter)           -- Returns ARRAY<STRING>
STRING_AGG(str, sep)            -- Aggregate strings
FORMAT('%s=%d', 'x', 42)        -- Printf-style formatting
```

**Date/Time Functions:**
```sql
CURRENT_DATE() / CURRENT_TIMESTAMP() / CURRENT_DATETIME()
DATE(timestamp_expr)            -- Extract DATE part
DATE_ADD(date, INTERVAL n DAY)  -- Add time interval
DATE_DIFF(date1, date2, DAY)    -- Difference in units
DATE_TRUNC(date, MONTH)         -- Truncate to period
TIMESTAMP_TRUNC(ts, HOUR)       -- Truncate timestamp
FORMAT_DATE('%Y-%m', date)      -- Format as string
PARSE_DATE('%Y%m%d', '20260316')-- Parse from string
EXTRACT(YEAR FROM date)         -- Extract component
UNIX_SECONDS(ts)                -- To Unix epoch
TIMESTAMP_SECONDS(epoch)        -- From Unix epoch
```

**Math Functions:**
```sql
ROUND(x, n) / CEIL(x) / FLOOR(x)
ABS(x) / MOD(x, y) / DIV(x, y)
SAFE_DIVIDE(x, y)               -- Returns NULL instead of divide-by-zero
POW(x, n) / SQRT(x) / LN(x) / LOG(x, base)
GREATEST(a, b, ...) / LEAST(a, b, ...)
RAND()                          -- 0.0 to 1.0 random
```

**Conditional Functions:**
```sql
IF(condition, true_val, false_val)
IFNULL(expr, default_val)       -- Replace NULL
NULLIF(expr, value)             -- Return NULL if equal
COALESCE(a, b, c)               -- First non-NULL
CASE WHEN x THEN y ELSE z END
```

**JSON Functions:**
```sql
JSON_VALUE(json_col, '$.key')           -- Extract scalar
JSON_QUERY(json_col, '$.nested.obj')    -- Extract JSON object
JSON_VALUE_ARRAY(json_col, '$.arr')     -- Extract array of scalars
TO_JSON_STRING(value)                   -- Convert to JSON string
PARSE_JSON('{"key":"val"}')             -- Parse string to JSON
```

---

### Window Functions

```sql
SELECT
  user_id,
  event_ts,
  amount,
  -- Running total
  SUM(amount)    OVER (PARTITION BY user_id ORDER BY event_ts
                       ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
  -- Rank within partition
  RANK()         OVER (PARTITION BY user_id ORDER BY amount DESC)        AS rank_by_amount,
  DENSE_RANK()   OVER (PARTITION BY user_id ORDER BY amount DESC)        AS dense_rank,
  ROW_NUMBER()   OVER (PARTITION BY user_id ORDER BY event_ts)           AS row_num,
  -- Lag/lead
  LAG(amount, 1) OVER (PARTITION BY user_id ORDER BY event_ts)           AS prev_amount,
  LEAD(amount,1) OVER (PARTITION BY user_id ORDER BY event_ts)           AS next_amount,
  -- Moving average (3-row window)
  AVG(amount)    OVER (PARTITION BY user_id ORDER BY event_ts
                       ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)         AS moving_avg_3,
  -- Percentile
  PERCENT_RANK() OVER (PARTITION BY user_id ORDER BY amount)             AS pct_rank,
  -- First/last value in window
  FIRST_VALUE(amount) OVER (PARTITION BY user_id ORDER BY event_ts
                            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS first_amount
FROM events;
```

---

### CTEs and Recursive CTEs

```sql
-- Standard CTE
WITH
daily_totals AS (
  SELECT DATE(event_ts) AS day, SUM(amount) AS total
  FROM events
  GROUP BY 1
),
ranked AS (
  SELECT *, RANK() OVER (ORDER BY total DESC) AS rnk
  FROM daily_totals
)
SELECT * FROM ranked WHERE rnk <= 10;

-- Recursive CTE (e.g., org hierarchy traversal)
WITH RECURSIVE org_tree AS (
  -- Anchor: start from root
  SELECT employee_id, manager_id, name, 0 AS depth
  FROM employees
  WHERE manager_id IS NULL

  UNION ALL

  -- Recursive: join children to parents
  SELECT e.employee_id, e.manager_id, e.name, t.depth + 1
  FROM employees e
  JOIN org_tree t ON e.manager_id = t.employee_id
)
SELECT * FROM org_tree ORDER BY depth, name;
```

---

### ARRAY & STRUCT Operations

```sql
-- UNNEST array → rows
SELECT order_id, item
FROM orders, UNNEST(line_items) AS item;

-- UNNEST with offset (index)
SELECT order_id, idx, item.product_id
FROM orders, UNNEST(line_items) AS item WITH OFFSET AS idx;

-- ARRAY_AGG — aggregate rows into array
SELECT
  user_id,
  ARRAY_AGG(product_id ORDER BY purchase_ts LIMIT 10) AS recent_products
FROM purchases
GROUP BY user_id;

-- ARRAY_LENGTH, IN UNNEST
SELECT * FROM t WHERE 'electronics' IN UNNEST(t.categories);

-- STRUCT construction
SELECT STRUCT('Alice' AS name, 30 AS age) AS person;

-- Access nested STRUCT field
SELECT order.customer.name FROM orders;

-- ARRAY of STRUCTs
SELECT
  user_id,
  ARRAY_AGG(STRUCT(product_id, amount) ORDER BY amount DESC) AS purchases
FROM events
GROUP BY user_id;
```

---

### DML: INSERT, UPDATE, DELETE, MERGE

```sql
-- INSERT
INSERT INTO my_table (col1, col2) VALUES ('a', 1), ('b', 2);
INSERT INTO my_table SELECT * FROM staging_table;

-- UPDATE
UPDATE my_table
SET status = 'inactive'
WHERE last_login < DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY);

-- DELETE
DELETE FROM my_table
WHERE created_at < TIMESTAMP('2020-01-01');

-- MERGE (upsert pattern)
MERGE target_table T
USING staging_table S
ON T.user_id = S.user_id
WHEN MATCHED THEN
  UPDATE SET
    T.name    = S.name,
    T.email   = S.email,
    T.updated = CURRENT_TIMESTAMP()
WHEN NOT MATCHED BY TARGET THEN
  INSERT (user_id, name, email, updated)
  VALUES (S.user_id, S.name, S.email, CURRENT_TIMESTAMP())
WHEN NOT MATCHED BY SOURCE THEN
  DELETE;
```

> ⚠️ BigQuery DML uses **snapshot isolation** — large DML statements can be expensive. Prefer `INSERT OVERWRITE` (WRITE_TRUNCATE) via load jobs for bulk operations.

---

### DDL

```sql
-- CREATE TABLE with options
CREATE TABLE IF NOT EXISTS `proj.dataset.sales` (
  sale_id   STRING NOT NULL,
  sale_date DATE   NOT NULL,
  amount    NUMERIC,
  region    STRING
)
PARTITION BY sale_date
CLUSTER BY region
OPTIONS (
  description        = 'Daily sales facts',
  partition_expiration_days = 365,
  require_partition_filter  = TRUE    -- Force partition filter in queries
);

-- CREATE VIEW
CREATE OR REPLACE VIEW `proj.dataset.v_active_users` AS
SELECT user_id, email FROM users WHERE status = 'active';

-- CREATE MATERIALIZED VIEW
CREATE MATERIALIZED VIEW `proj.dataset.mv_daily_revenue`
PARTITION BY sale_date
OPTIONS (enable_refresh = TRUE, refresh_interval_minutes = 60)
AS
SELECT sale_date, SUM(amount) AS revenue
FROM `proj.dataset.sales`
GROUP BY sale_date;

-- CREATE SCALAR UDF (JavaScript)
CREATE OR REPLACE FUNCTION `proj.dataset.mask_email`(email STRING)
RETURNS STRING
LANGUAGE js AS '''
  if (!email) return null;
  const [user, domain] = email.split('@');
  return user.slice(0,2) + '***@' + domain;
''';

-- CREATE SQL UDF
CREATE OR REPLACE FUNCTION `proj.dataset.days_ago`(n INT64)
RETURNS DATE AS (DATE_SUB(CURRENT_DATE(), INTERVAL n DAY));

-- ALTER TABLE
ALTER TABLE `proj.dataset.sales`
  ADD COLUMN IF NOT EXISTS discount NUMERIC,
  SET OPTIONS (description = 'Updated sales facts');

-- DROP
DROP TABLE IF EXISTS `proj.dataset.temp_table`;
DROP VIEW  IF EXISTS `proj.dataset.v_old`;
```

---

### Scripting (Stored Procedures & Control Flow)

```sql
-- Stored procedure with control flow
CREATE OR REPLACE PROCEDURE `proj.dataset.process_events`(
  IN  start_date DATE,
  OUT row_count  INT64
)
BEGIN
  DECLARE v_ts TIMESTAMP;
  SET v_ts = CURRENT_TIMESTAMP();

  -- IF/ELSE
  IF start_date IS NULL THEN
    SET start_date = DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY);
  END IF;

  -- DML inside procedure
  DELETE FROM `proj.dataset.staging`
  WHERE event_date < start_date;

  -- LOOP with counter
  DECLARE i INT64 DEFAULT 0;
  LOOP
    SET i = i + 1;
    IF i >= 3 THEN LEAVE; END IF;
  END LOOP;

  -- WHILE loop
  WHILE i < 10 DO
    SET i = i + 1;
  END WHILE;

  -- Return output
  SET row_count = @@row_count;

  -- Exception handling
  EXCEPTION WHEN ERROR THEN
    SELECT @@error.message, @@error.statement_text;
END;

-- CALL the procedure
CALL `proj.dataset.process_events`(DATE '2026-01-01', NULL);
```

---

### Query Parameters

```python
# Named parameters (@param) — prevent SQL injection
from google.cloud import bigquery

client = bigquery.Client()

query = """
    SELECT * FROM `proj.dataset.orders`
    WHERE user_id = @user_id
      AND order_date BETWEEN @start_date AND @end_date
"""

job_config = bigquery.QueryJobConfig(
    query_parameters=[
        bigquery.ScalarQueryParameter("user_id",    "STRING", "user_123"),
        bigquery.ScalarQueryParameter("start_date", "DATE",   "2026-01-01"),
        bigquery.ScalarQueryParameter("end_date",   "DATE",   "2026-03-31"),
    ]
)

results = client.query(query, job_config=job_config).result()
```

---

### INFORMATION_SCHEMA Queries

```sql
-- List all tables in a dataset
SELECT table_name, table_type, creation_time, row_count, size_bytes
FROM `proj.dataset.INFORMATION_SCHEMA.TABLES`;

-- List columns with types
SELECT table_name, column_name, data_type, is_nullable
FROM `proj.dataset.INFORMATION_SCHEMA.COLUMNS`
WHERE table_name = 'my_table';

-- List partition metadata
SELECT partition_id, total_rows, total_logical_bytes, last_modified_time
FROM `proj.dataset.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = 'events';

-- Recent jobs (last 24h)
SELECT
  job_id, user_email, query,
  total_bytes_billed / POW(2,40) AS tb_billed,
  total_slot_ms / 1000.0 AS slot_seconds,
  TIMESTAMP_DIFF(end_time, start_time, SECOND) AS duration_sec,
  state, error_result
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
ORDER BY total_bytes_billed DESC
LIMIT 50;
```

---

### bq query CLI Examples

```bash
# Run a query (on-demand)
bq query --use_legacy_sql=false \
  'SELECT COUNT(*) FROM `proj.dataset.events`'

# Dry run — estimate bytes without executing
bq query --use_legacy_sql=false --dry_run \
  'SELECT * FROM `proj.dataset.events`'

# Write results to a destination table
bq query --use_legacy_sql=false \
  --destination_table=proj:dataset.results \
  --replace \
  'SELECT user_id, COUNT(*) cnt FROM `proj.dataset.events` GROUP BY 1'

# Use a parameter
bq query --use_legacy_sql=false \
  --parameter='min_amount:FLOAT64:100.0' \
  'SELECT * FROM `proj.dataset.sales` WHERE amount > @min_amount'

# Set query timeout (ms)
bq query --use_legacy_sql=false \
  --job_timeout_ms=60000 \
  'SELECT 1'
```

---

## 4. Loading & Exporting Data

### Supported Load Formats

| Format | Schema Required | Compression | Best For |
|---|---|---|---|
| **CSV** | Optional (autodetect) | gzip | Simple flat data |
| **Newline-delimited JSON** | Optional | gzip | Semi-structured, schema flexibility |
| **Avro** | Embedded in file | deflate, snappy | Schema evolution, complex types |
| **Parquet** | Embedded in file | snappy, gzip, zstd | Columnar — fastest load, best compression |
| **ORC** | Embedded in file | zlib, snappy | Hadoop ecosystem |

> 💡 **Parquet is preferred** for load jobs — it preserves types exactly, compresses well, and loads faster than CSV/JSON.

---

### bq load CLI Examples

```bash
# Load CSV with explicit schema
bq load \
  --source_format=CSV \
  --schema=schema.json \
  --skip_leading_rows=1 \
  --write_disposition=WRITE_APPEND \
  proj:dataset.my_table \
  gs://my-bucket/data/*.csv

# Load Parquet (schema from file)
bq load \
  --source_format=PARQUET \
  --parquet_enable_list_inference=true \
  proj:dataset.my_table \
  gs://my-bucket/data/*.parquet

# Load NDJSON into partitioned table (by field)
bq load \
  --source_format=NEWLINE_DELIMITED_JSON \
  --schema=schema.json \
  --time_partitioning_field=event_date \
  --time_partitioning_type=DAY \
  --clustering_fields=user_id,event_type \
  proj:dataset.events \
  gs://my-bucket/events/2026-03-16/*.json

# Write dispositions
--write_disposition=WRITE_APPEND     # Add rows to existing table
--write_disposition=WRITE_TRUNCATE   # Replace all data
--write_disposition=WRITE_EMPTY      # Fail if table not empty
```

---

### Streaming Inserts (Legacy)

```python
from google.cloud import bigquery

client = bigquery.Client()
table_id = "proj.dataset.events"

rows = [
    {"user_id": "u1", "event_ts": "2026-03-16T10:00:00Z", "amount": 99.99},
    {"user_id": "u2", "event_ts": "2026-03-16T10:01:00Z", "amount": 49.50},
]

errors = client.insert_rows_json(table_id, rows)
if errors:
    print(f"Streaming insert errors: {errors}")
```

| Streaming Inserts | Notes |
|---|---|
| Latency | Data available for queries in seconds |
| Quota | 1 GB/s per project (default) |
| Deduplication | Best-effort using `insertId` (not guaranteed) |
| Cost | $0.01 per 200 MB inserted |
| Limitation | Cannot stream into partitioned tables older than 31 days |

> ⚠️ For new workloads, prefer the **Storage Write API** over legacy streaming inserts — it offers exactly-once semantics and lower cost.

---

### Storage Write API (Recommended)

| Stream Type | Semantics | Use Case |
|---|---|---|
| **Default** | At-least-once | Simple append, no dedup needed |
| **Committed** | Exactly-once, immediate visibility | Financial data, audit logs |
| **Buffered** | Exactly-once, controlled flush | Batch-like with low latency |
| **Pending** | Exactly-once, commit on close | Atomic batch loads |

```python
from google.cloud.bigquery_storage_v1 import BigQueryWriteClient, types
from google.protobuf import descriptor_pb2

# Simplified example — use the bigquery-storage library
# pip install google-cloud-bigquery-storage

client = BigQueryWriteClient()
parent = client.table_path("proj", "dataset", "my_table")

write_stream = client.create_write_stream(
    parent=parent,
    write_stream=types.WriteStream(type_=types.WriteStream.Type.COMMITTED)
)
# ... append rows using proto serialization
```

---

### Data Transfer Service

```bash
# List available transfer sources
bq ls --transfer_config --transfer_location=us

# Create a scheduled transfer (Google Ads → BigQuery)
bq mk --transfer_config \
  --target_dataset=MY_DATASET \
  --display_name="Google Ads Transfer" \
  --data_source=google_ads \
  --params='{"customer_id":"123-456-7890"}' \
  --schedule="every 24 hours"
```

---

### Export (bq extract)

```bash
# Export to CSV (single file)
bq extract \
  --destination_format=CSV \
  --compression=GZIP \
  proj:dataset.my_table \
  gs://my-bucket/export/data.csv.gz

# Export to Parquet (multiple shards for large tables)
bq extract \
  --destination_format=PARQUET \
  proj:dataset.my_table \
  'gs://my-bucket/export/shard_*.parquet'

# Export a query result (not just a table)
bq query --use_legacy_sql=false \
  --destination_table=proj:dataset.temp_export \
  --replace \
  'SELECT * FROM dataset.events WHERE DATE(ts) = "2026-03-16"'

bq extract proj:dataset.temp_export gs://bucket/2026-03-16_*.parquet
```

> 💡 Use **wildcard URIs** (`shard_*.parquet`) for tables >1 GB — BigQuery auto-shards the export into multiple files.

---

## 5. Performance Optimization

### 🚀 Dry Run Before Executing

```bash
# Estimate cost before running (FREE — no bytes billed)
bq query --use_legacy_sql=false --dry_run \
  'SELECT * FROM `proj.dataset.events`'
# Output: Query successfully validated. Assuming the tables are not modified,
#         running this query will process 4271890432 bytes (3.98 GB).
```

---

### Partition Pruning

```sql
-- ❌ BAD: Full table scan — no partition filter
SELECT * FROM `proj.dataset.events`;

-- ✅ GOOD: Partition pruned — only reads matching partitions
SELECT * FROM `proj.dataset.events`
WHERE event_date = '2026-03-16';                    -- DATE partition column

-- ✅ GOOD: Range of partitions
SELECT * FROM `proj.dataset.events`
WHERE event_date BETWEEN '2026-03-01' AND '2026-03-16';

-- ❌ BAD: Wrapping partition column in function prevents pruning
WHERE DATE_TRUNC(event_date, MONTH) = '2026-03-01'  -- Can't prune!

-- ✅ GOOD: Use range instead
WHERE event_date BETWEEN '2026-03-01' AND '2026-03-31'
```

> 💡 Set `require_partition_filter = TRUE` on high-cost tables to enforce partition filters at query time.

---

### Cluster Pruning

```sql
-- Table: PARTITION BY event_date CLUSTER BY user_id, event_type

-- ✅ GOOD: Uses cluster order — BigQuery skips non-matching blocks
SELECT * FROM `proj.dataset.events`
WHERE event_date = '2026-03-16'
  AND user_id = 'u_12345'
  AND event_type = 'purchase';

-- ⚠️ PARTIAL: Only first cluster column used
WHERE event_date = '2026-03-16'
  AND event_type = 'purchase';   -- Skips user_id (first col) — less effective
```

---

### Approximate Aggregations (10–100x faster)

```sql
-- Exact (expensive for huge datasets)
SELECT COUNT(DISTINCT user_id) FROM events;       -- ~$$$

-- Approximate (HLL++ algorithm, 1-2% error, much faster)
SELECT APPROX_COUNT_DISTINCT(user_id) FROM events; -- ~$

-- Approximate top-N
SELECT APPROX_TOP_COUNT(country, 10) FROM events;

-- Approximate quantiles (array of 100 percentile values)
SELECT APPROX_QUANTILES(amount, 100) FROM sales;
-- Access p95: APPROX_QUANTILES(amount, 100)[OFFSET(95)]
```

---

### Materialized Views

```sql
-- Create a materialized view with auto-refresh
CREATE MATERIALIZED VIEW `proj.dataset.mv_hourly_revenue`
PARTITION BY DATE(hour)
CLUSTER BY region
OPTIONS (
  enable_refresh          = TRUE,
  refresh_interval_minutes = 30,
  max_staleness           = INTERVAL "4:0:0" HOUR TO SECOND
)
AS
SELECT
  TIMESTAMP_TRUNC(sale_ts, HOUR) AS hour,
  region,
  SUM(amount) AS revenue,
  COUNT(*)    AS txn_count
FROM `proj.dataset.sales`
GROUP BY 1, 2;
```

> 💡 BigQuery **automatically rewrites queries** to use materialized views when the query matches the MV's definition — even if you query the base table. Zero code changes needed.

---

### Anti-Patterns

| ❌ Anti-Pattern | 🔴 Problem | ✅ Fix |
|---|---|---|
| `SELECT *` on large tables | Reads all columns = max bytes | Project only needed columns |
| No partition filter | Full table scan | Add `WHERE partition_col = ...` |
| Self-join on large table | Shuffles entire table twice | Use window functions instead |
| Many small streaming inserts | High overhead per-request | Batch rows, use Storage Write API |
| CROSS JOIN without condition | Cartesian product — can OOM | Always add join condition |
| UDF on every row (JS) | Slow interpreter overhead | Use native SQL functions when possible |
| `COUNT(DISTINCT x)` on billions | Expensive exact count | Use `APPROX_COUNT_DISTINCT` |
| Repeated subquery in SELECT | Evaluated per-row | Move to CTE or JOIN |
| Joining before filtering | Large intermediate shuffle | Filter BEFORE joining |
| Storing JSON as STRING | No pushdown or pruning | Use native JSON type or STRUCT |

---

### BI Engine

```bash
# Reserve BI Engine capacity for a project (in-memory acceleration)
bq update --bi_reservation_size=10 \
  --location=US \
  --project_id=MY_PROJECT
# Sizes in GB: 1, 2, 5, 10, 20, 50, 100, 150, 200, 250, 300
```

> 💡 BI Engine accelerates Looker Studio, Looker, and direct SQL queries on **small to medium** aggregation result sets (<10 GB). It does **not** replace slot reservations for large queries.

---

## 6. Access Control & Security

### IAM Roles

| Role | Scope | Grants |
|---|---|---|
| `roles/bigquery.admin` | Project | Full control: all datasets, jobs, config |
| `roles/bigquery.dataOwner` | Dataset | Read/write/delete tables in dataset; manage ACLs |
| `roles/bigquery.dataEditor` | Dataset | Read/write tables; cannot manage ACLs |
| `roles/bigquery.dataViewer` | Dataset | Read tables and metadata; no query jobs |
| `roles/bigquery.jobUser` | Project | Run query/load/export jobs; no data access |
| `roles/bigquery.user` | Project | Run jobs + read public datasets; limited metadata |
| `roles/bigquery.readSessionUser` | Project | Use BigQuery Storage Read API |
| `roles/bigquery.filteredDataViewer` | Table | Access rows matching their row access policy |
| `roles/bigquery.maskedDataViewer` | Column | See masked (obfuscated) column values |

> 💡 Common pattern: grant `bigquery.jobUser` at **project** level + `bigquery.dataViewer` at **dataset** level. This is the minimum to query data.

---

### Dataset ACLs vs. IAM Policies

| | Dataset ACL | IAM Policy |
|---|---|---|
| **Scope** | Single dataset | Project / folder / org |
| **Granularity** | Table-level possible | Resource hierarchy |
| **Special entities** | `allAuthenticatedUsers`, groups, service accounts | Same |
| **Precedence** | Dataset ACL checked first for data access | IAM for job submission |
| **Recommended** | For dataset-level sharing | For org-wide access |

```bash
# Grant dataset-level access via bq CLI
bq update --set_label=team:analytics \
  proj:MY_DATASET

# Add a reader to a dataset
bq update \
  --add_view_access \
  'proj:dataset.authorized_view' \
  proj:MY_DATASET
```

---

### Row-Level Security

```sql
-- Create a row access policy
CREATE ROW ACCESS POLICY filter_by_region
ON `proj.dataset.sales`
GRANT TO ("group:emea-team@company.com")
FILTER USING (region = 'EMEA');

CREATE ROW ACCESS POLICY filter_by_region_apac
ON `proj.dataset.sales`
GRANT TO ("group:apac-team@company.com")
FILTER USING (region = 'APAC');

-- Users not covered by ANY policy see NO rows
-- Use a catch-all policy for default access
CREATE ROW ACCESS POLICY all_rows
ON `proj.dataset.sales`
GRANT TO ("group:data-admin@company.com")
FILTER USING (TRUE);

-- List row access policies
SELECT * FROM `proj.dataset.INFORMATION_SCHEMA.ROW_ACCESS_POLICIES`;

-- Drop a policy
DROP ALL ROW ACCESS POLICIES ON `proj.dataset.sales`;
```

---

### Column-Level Security (Data Masking)

```bash
# 1. Create a taxonomy and policy tag in Data Catalog
gcloud data-catalog taxonomies create \
  --location=us \
  --display-name="PII Taxonomy" \
  --description="Tags for personally identifiable information"

# 2. Create a policy tag
gcloud data-catalog taxonomies policy-tags create \
  --taxonomy=TAXONOMY_ID \
  --location=us \
  --display-name="Email"

# 3. Attach tag to column (via schema JSON or Console)
# schema.json:
# { "name": "email", "type": "STRING", "policyTags": {"names":["POLICY_TAG_RESOURCE_NAME"]} }

# 4. Grant fine-grained access
gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="user:analyst@company.com" \
  --role="roles/bigquery.maskedDataViewer"
```

```sql
-- Column masking rule (Data Policy API)
-- Set via Console: IAM & Admin → Data Policies
-- Example masking types: SHA256, ALWAYS_NULL, DEFAULT_MASKING_VALUE, LAST_FOUR_CHARACTERS
```

---

### Authorized Views & Datasets

```sql
-- Authorized view: allows view to access base table
-- even if the querying user has no direct table access
-- 1. Create the view
CREATE VIEW `proj.reporting.v_user_summary` AS
SELECT user_id, CONCAT(first_name, ' ', last_name) AS name
FROM `proj.raw.users`;

-- 2. Authorize the view to access proj.raw dataset
-- (Done via Console or API — not SQL)
-- bq update --add_view_access 'proj:reporting.v_user_summary' proj:raw
```

---

### CMEK with Cloud KMS

```bash
# Create a KMS key
gcloud kms keyrings create bq-keyring \
  --location=us-central1

gcloud kms keys create bq-table-key \
  --location=us-central1 \
  --keyring=bq-keyring \
  --purpose=encryption

# Create CMEK-encrypted table
bq mk --table \
  --schema=schema.json \
  --kms_key=projects/MY_PROJECT/locations/us-central1/keyRings/bq-keyring/cryptoKeys/bq-table-key \
  proj:dataset.encrypted_table
```

---

### VPC Service Controls

```bash
# Restrict BigQuery to a VPC Service Control perimeter
gcloud access-context-manager perimeters create bq-perimeter \
  --title="BigQuery Perimeter" \
  --resources="projects/MY_PROJECT_NUMBER" \
  --restricted-services="bigquery.googleapis.com" \
  --policy=MY_POLICY_ID
```

---

## 7. Cost Management

### 💰 On-Demand Cost Estimation

```sql
-- Estimate cost of a query before running (via DRY RUN)
-- $6.25 per TB scanned (on-demand, us region)
-- bytes_processed / 1e12 * $6.25 = estimated cost

-- After running: check actual bytes billed
SELECT
  job_id,
  total_bytes_billed / POW(2, 40) AS tb_billed,
  total_bytes_billed / POW(2, 40) * 6.25 AS estimated_cost_usd
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE job_id = 'MY_JOB_ID';
```

---

### Top Query Cost Analysis

```sql
-- Top 20 most expensive queries (last 7 days)
SELECT
  user_email,
  SUBSTR(query, 0, 100)                                    AS query_preview,
  ROUND(total_bytes_billed / POW(2,40), 4)                 AS tb_billed,
  ROUND(total_bytes_billed / POW(2,40) * 6.25, 4)          AS cost_usd,
  ROUND(total_slot_ms / 1000.0, 1)                         AS slot_seconds,
  TIMESTAMP_DIFF(end_time, start_time, SECOND)             AS duration_sec,
  creation_time
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND job_type = 'QUERY'
  AND state = 'DONE'
  AND error_result IS NULL
ORDER BY total_bytes_billed DESC
LIMIT 20;

-- Daily cost by user
SELECT
  DATE(creation_time)                              AS day,
  user_email,
  ROUND(SUM(total_bytes_billed)/POW(2,40), 3)      AS total_tb,
  ROUND(SUM(total_bytes_billed)/POW(2,40)*6.25, 2) AS total_cost_usd,
  COUNT(*)                                         AS job_count
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY 1, 2
ORDER BY 1 DESC, 4 DESC;
```

---

### Custom Cost Quotas

```bash
# Set per-project daily quota (bytes processed)
# Via Console: BigQuery → Admin → Quotas
# Or via Cloud Quotas API:
gcloud services quota update \
  --service=bigquery.googleapis.com \
  --consumer=project:MY_PROJECT \
  --metric=bigquery.googleapis.com/quota/query/usage \
  --unit=1/d/{project} \
  --value=1000000000000    # 1 TB per day
```

---

### Storage Cost Management

```sql
-- Find tables consuming most storage
SELECT
  table_schema,
  table_name,
  ROUND(total_logical_bytes / POW(2,30), 2)         AS logical_gb,
  ROUND(total_physical_bytes / POW(2,30), 2)        AS physical_gb,
  ROUND(active_logical_bytes / POW(2,30), 2)        AS active_gb,
  ROUND(long_term_logical_bytes / POW(2,30), 2)     AS longterm_gb,
  last_modified_time
FROM `proj.region-us`.INFORMATION_SCHEMA.TABLE_STORAGE
ORDER BY total_logical_bytes DESC
LIMIT 20;
```

```bash
# Set table expiration (auto-delete after N days)
bq update \
  --expiration=2592000 \    # 30 days in seconds
  proj:dataset.temp_table

# Set dataset default expiration for all new tables
bq update \
  --default_table_expiration=604800 \   # 7 days
  proj:MY_DATASET
```

| Storage Tier | Condition | Cost (approx) |
|---|---|---|
| **Active** | Modified in last 90 days | ~$0.02/GB/month |
| **Long-term** | Not modified for 90+ days | ~$0.01/GB/month (50% off) |

---

## 8. Advanced Features

### BigQuery ML (BQML)

```sql
-- Train a logistic regression model
CREATE OR REPLACE MODEL `proj.dataset.churn_model`
OPTIONS (
  model_type           = 'LOGISTIC_REG',
  input_label_cols     = ['churned'],
  max_iterations       = 20,
  l1_reg               = 0.1,
  data_split_method    = 'AUTO_SPLIT'
)
AS
SELECT
  days_since_login,
  purchase_count,
  avg_order_value,
  support_tickets,
  churned
FROM `proj.dataset.training_data`;

-- Evaluate model
SELECT * FROM ML.EVALUATE(MODEL `proj.dataset.churn_model`);

-- Predict on new data
SELECT
  user_id,
  predicted_churned,
  predicted_churned_probs
FROM ML.PREDICT(
  MODEL `proj.dataset.churn_model`,
  (SELECT * FROM `proj.dataset.new_users`)
);

-- Feature importance
SELECT * FROM ML.FEATURE_IMPORTANCE(MODEL `proj.dataset.churn_model`);
```

**Supported BQML model types:**

| Model Type | `model_type` Value |
|---|---|
| Logistic Regression | `LOGISTIC_REG` |
| Linear Regression | `LINEAR_REG` |
| K-Means Clustering | `KMEANS` |
| Matrix Factorization | `MATRIX_FACTORIZATION` |
| Random Forest (classification) | `RANDOM_FOREST_CLASSIFIER` |
| XGBoost | `BOOSTED_TREE_CLASSIFIER` / `BOOSTED_TREE_REGRESSOR` |
| AutoML Tables | `AUTOML_CLASSIFIER` / `AUTOML_REGRESSOR` |
| TensorFlow (import) | `TENSORFLOW` |
| Time Series (ARIMA+) | `ARIMA_PLUS` |
| LLM (remote, Vertex AI) | `LLAMA2`, `TEXT_GENERATE` (remote) |

---

### Remote Functions (Cloud Run / Cloud Functions)

```sql
-- Create a remote function connection
-- (Create connection via Console or bq CLI first)

-- Register the remote function
CREATE FUNCTION `proj.dataset.call_geocoder`(address STRING)
RETURNS JSON
REMOTE WITH CONNECTION `proj.us.my-cloud-run-connection`
OPTIONS (endpoint = 'https://geocoder-abc123-uc.a.run.app/geocode');

-- Use it in SQL like any UDF
SELECT
  address,
  call_geocoder(address) AS geo_result
FROM `proj.dataset.addresses`;
```

---

### BigQuery Omni (Cross-Cloud)

```sql
-- Query AWS S3 data from BigQuery (via BigQuery Omni on AWS)
-- Requires dataset in aws-us-east-1 region

CREATE EXTERNAL TABLE `proj.aws_dataset.s3_logs`
OPTIONS (
  format      = 'PARQUET',
  uris        = ['s3://my-aws-bucket/logs/*.parquet'],
  connection  = 'proj.aws-us-east-1.my-aws-connection'
);

SELECT * FROM `proj.aws_dataset.s3_logs` LIMIT 100;
```

---

### Time Travel & Fail-Safe

```sql
-- Query table as it existed 24 hours ago (time travel window: 7 days default)
SELECT * FROM `proj.dataset.orders`
FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR);

-- Restore a deleted/corrupted table from a specific timestamp
CREATE TABLE `proj.dataset.orders_restored`
AS
SELECT * FROM `proj.dataset.orders`
FOR SYSTEM_TIME AS OF '2026-03-15 12:00:00 UTC';
```

| Feature | Window | Purpose |
|---|---|---|
| **Time Travel** | Up to 7 days (configurable) | Query historical snapshots, restore tables |
| **Fail-safe** | 7 days after time-travel window | Google-managed recovery (no user query) |

```bash
# Set time travel window (1–7 days)
bq update \
  --time_travel_hours=168 \   # 7 days
  proj:dataset.important_table
```

---

### Analytics Hub

```bash
# Create a data exchange (publisher side)
bq mk --data_exchange \
  --display_name="Public Analytics Exchange" \
  --description="Curated public datasets" \
  --location=US \
  proj:MY_EXCHANGE

# Create a listing (add a dataset to exchange)
bq mk --listing \
  --exchange=MY_EXCHANGE \
  --display_name="Web Traffic Dataset" \
  --description="Daily web traffic aggregates" \
  --linked_dataset=proj:web_traffic_dataset \
  --location=US \
  proj:MY_LISTING

# Subscribe to a listing (subscriber side)
bq mk --subscription \
  --listing=projects/PUBLISHER_PROJECT/locations/US/dataExchanges/EXCHANGE/listings/LISTING \
  --destination_dataset=proj:my_subscribed_data
```

---

### Dataform (SQL Workflow Orchestration)

Dataform is BigQuery-native SQL pipeline orchestration (similar to dbt):

```sql
-- definitions/orders.sqlx
config {
  type: "table",
  schema: "analytics",
  description: "Cleaned orders fact table",
  tags: ["daily"],
  bigquery: {
    partitionBy: "DATE(order_ts)",
    clusterBy: ["region", "product_id"]
  }
}

SELECT
  order_id,
  order_ts,
  region,
  product_id,
  SAFE_CAST(amount AS NUMERIC) AS amount
FROM ${ref("raw_orders")}
WHERE order_id IS NOT NULL
```

---

## 9. bq CLI & API Quick Reference

### Essential bq Commands

```bash
# ── DATASET ──────────────────────────────────────────────────────────
bq mk --dataset --location=US proj:MY_DATASET         # Create dataset
bq ls proj:                                            # List datasets
bq show proj:MY_DATASET                               # Describe dataset
bq rm --dataset --recursive proj:MY_DATASET           # Delete dataset + all tables

# ── TABLE ─────────────────────────────────────────────────────────────
bq mk --table proj:dataset.my_table schema.json       # Create table from schema
bq show proj:dataset.my_table                         # Describe table (schema, partitioning)
bq ls proj:MY_DATASET                                 # List tables in dataset
bq rm proj:dataset.my_table                           # Delete table
bq cp proj:dataset.src proj:dataset.dst               # Copy table
bq update --expiration=86400 proj:dataset.my_table    # Set 1-day expiration

# ── QUERY ─────────────────────────────────────────────────────────────
bq query --use_legacy_sql=false 'SQL'                 # Run query
bq query --dry_run 'SQL'                              # Estimate bytes (no execution)
bq query --destination_table=p:d.t --replace 'SQL'   # Write to table
bq query --nouse_cache 'SQL'                          # Bypass query cache
bq query --maximum_bytes_billed=1000000000 'SQL'      # Set hard cost cap (1 GB)

# ── LOAD ──────────────────────────────────────────────────────────────
bq load --source_format=CSV p:d.t gs://bucket/f.csv  # Load from GCS
bq load --autodetect --source_format=CSV p:d.t f.csv # Load local file + autodetect

# ── EXTRACT ──────────────────────────────────────────────────────────
bq extract p:d.t gs://bucket/out_*.csv                # Export (wildcard for sharding)
bq extract --destination_format=PARQUET p:d.t gs://   # Export as Parquet

# ── JOBS ──────────────────────────────────────────────────────────────
bq ls --jobs --all --max_results=10                   # List recent jobs
bq show --job JOB_ID                                  # Describe a job
bq cancel JOB_ID                                      # Cancel running job

# ── FORMATTING ────────────────────────────────────────────────────────
--format=json           # JSON output
--format=prettyjson     # Pretty-printed JSON
--format=csv            # CSV output
--format=sparse         # Human-readable table (default)
--format=none           # No output (for side-effect queries)
```

---

### Python Client Library

```python
from google.cloud import bigquery
import pandas as pd

client = bigquery.Client(project="MY_PROJECT")

# ── Run a query ───────────────────────────────────────────────────────
query = "SELECT user_id, COUNT(*) AS cnt FROM `proj.dataset.events` GROUP BY 1"
df = client.query(query).to_dataframe()
print(df.head())

# ── Load DataFrame to BigQuery ────────────────────────────────────────
df = pd.DataFrame({"user_id": ["u1","u2"], "score": [0.9, 0.7]})

job_config = bigquery.LoadJobConfig(
    write_disposition=bigquery.WriteDisposition.WRITE_TRUNCATE,
    schema=[
        bigquery.SchemaField("user_id", "STRING"),
        bigquery.SchemaField("score",   "FLOAT64"),
    ],
)

job = client.load_table_from_dataframe(df, "proj.dataset.scores", job_config=job_config)
job.result()  # Wait for completion
print(f"Loaded {job.output_rows} rows")

# ── Stream rows ───────────────────────────────────────────────────────
rows = [{"user_id": "u3", "score": 0.85}]
errors = client.insert_rows_json("proj.dataset.scores", rows)
if errors:
    raise RuntimeError(errors)

# ── Create a table from DDL ───────────────────────────────────────────
ddl = """
CREATE TABLE IF NOT EXISTS `proj.dataset.new_table` (
  id STRING, value FLOAT64
)
"""
client.query(ddl).result()

# ── Copy a table ──────────────────────────────────────────────────────
copy_job = client.copy_table("proj.dataset.src", "proj.dataset.dst")
copy_job.result()
```

---

### REST API Quick Reference

```bash
# Run a synchronous query (returns immediately for fast queries)
curl -X POST \
  "https://bigquery.googleapis.com/bigquery/v2/projects/MY_PROJECT/queries" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "SELECT COUNT(*) FROM `proj.dataset.events`",
    "useLegacySql": false,
    "timeoutMs": 30000
  }'

# Insert rows via streaming insertAll
curl -X POST \
  "https://bigquery.googleapis.com/bigquery/v2/projects/MY_PROJECT/datasets/MY_DATASET/tables/MY_TABLE/insertAll" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "rows": [
      {"insertId": "row1", "json": {"user_id": "u1", "amount": 99.99}},
      {"insertId": "row2", "json": {"user_id": "u2", "amount": 49.50}}
    ]
  }'
```

---

## 10. Monitoring, Auditing & Observability

### Cloud Monitoring Metrics

| Metric | Description |
|---|---|
| `bigquery.googleapis.com/storage/table_count` | Number of tables per project |
| `bigquery.googleapis.com/storage/stored_bytes` | Total bytes stored |
| `bigquery.googleapis.com/slots/total_available` | Total reserved slots |
| `bigquery.googleapis.com/slots/allocated_for_project` | Slots used by project |
| `bigquery.googleapis.com/job_count` | Jobs by status (DONE, FAILED, PENDING) |
| `bigquery.googleapis.com/quota/query_scanned_bytes/usage` | Bytes scanned (quota monitoring) |

---

### INFORMATION_SCHEMA Monitoring Queries

```sql
-- ── Slow queries (last 24h, top 20) ──────────────────────────────────
SELECT
  job_id,
  user_email,
  ROUND(TIMESTAMP_DIFF(end_time, start_time, SECOND), 1)  AS duration_sec,
  ROUND(total_bytes_billed / POW(2,40), 4)                AS tb_billed,
  SUBSTR(query, 0, 200)                                   AS query_preview
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
  AND state = 'DONE'
  AND job_type = 'QUERY'
ORDER BY duration_sec DESC
LIMIT 20;

-- ── Failed jobs ────────────────────────────────────────────────────────
SELECT
  job_id, user_email, creation_time,
  error_result.reason,
  error_result.message
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
  AND error_result IS NOT NULL
ORDER BY creation_time DESC;

-- ── Slot utilization by reservation ────────────────────────────────────
SELECT
  reservation_id,
  SUM(total_slot_ms) / 1000.0 AS total_slot_seconds,
  COUNT(*) AS job_count
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
GROUP BY 1 ORDER BY 2 DESC;

-- ── Table storage breakdown ────────────────────────────────────────────
SELECT
  table_schema,
  table_name,
  ROUND(total_logical_bytes / POW(2,30), 2) AS logical_gb,
  ROUND(total_physical_bytes / POW(2,30), 2) AS physical_gb,
  last_modified_time
FROM `region-us`.INFORMATION_SCHEMA.TABLE_STORAGE
WHERE total_logical_bytes > 1e9   -- > 1 GB
ORDER BY total_logical_bytes DESC
LIMIT 20;

-- ── Partition metadata ────────────────────────────────────────────────
SELECT
  partition_id,
  total_rows,
  ROUND(total_logical_bytes / POW(2,20), 1) AS logical_mb,
  last_modified_time
FROM `proj.dataset.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = 'events'
ORDER BY partition_id DESC
LIMIT 30;
```

---

### Cloud Audit Logs

| Log Type | What It Captures |
|---|---|
| **Admin Activity** | DDL (CREATE, DROP, ALTER), IAM changes, dataset creation |
| **Data Access** | SELECT queries, DML, reads (requires explicit enable) |
| **System Event** | Automatic BigQuery maintenance operations |

```bash
# Query Data Access audit logs (requires Data Access logging enabled)
gcloud logging read \
  'resource.type="bigquery_resource"
   protoPayload.methodName="jobservice.query"
   protoPayload.authenticationInfo.principalEmail="analyst@company.com"' \
  --project=MY_PROJECT \
  --limit=50 \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.resourceName)"

# Enable Data Access audit logging for BigQuery
gcloud projects set-iam-policy MY_PROJECT policy.json
# policy.json must include auditConfigs for bigquery.googleapis.com
```

---

### Gemini in BigQuery (AI-Assisted SQL)

BigQuery now includes **Gemini** integration for AI-assisted query authoring:

- **SQL generation** from natural language in the BigQuery Studio editor
- **SQL explanation** — describe what a query does in plain English
- **Auto-complete** for SQL as you type
- **Data insights** — ask questions about tables in natural language

> 💡 Enable via: BigQuery Studio → Enable Gemini features. Requires Gemini for Google Cloud subscription.

---

## 11. Pricing Summary

> ⚠️ Prices below are approximate as of early 2026 (US region). Always verify at [cloud.google.com/bigquery/pricing](https://cloud.google.com/bigquery/pricing).

### Storage Pricing

| Tier | Condition | Price (per GB/month) |
|---|---|---|
| **Active logical** | Modified in last 90 days | ~$0.02 |
| **Long-term logical** | Not modified for 90+ days | ~$0.01 |
| **Active physical** | Compressed, with time travel | ~$0.04 |
| **Long-term physical** | Compressed, no modification | ~$0.02 |

> 💡 Use **physical billing** (enabled per dataset) if your data compresses well — Capacitor compression typically achieves 3–5x, making physical billing cheaper.

---

### Compute Pricing

| Model | Price | Notes |
|---|---|---|
| **On-demand** | ~$6.25 / TB scanned | First 1 TB/month free |
| **Standard edition slots** | ~$0.04 / slot-hour | Autoscaling, no commitment |
| **Enterprise edition slots** | ~$0.06 / slot-hour | + CMEK, BI Engine, column security |
| **Enterprise Plus (1-year)** | ~$0.052 / slot-hour | ~13% discount vs. Enterprise on-demand |
| **Enterprise Plus (3-year)** | ~$0.039 / slot-hour | ~35% discount |

---

### Free Tier

| Resource | Free Amount | Notes |
|---|---|---|
| Storage | **10 GB/month** | Per project |
| Queries | **1 TB/month** | On-demand scanned bytes |
| Streaming inserts | ❌ Not free | $0.01/200MB |
| Metadata operations | ✅ Always free | `SHOW`, `DESCRIBE`, `LIST` |
| Cached query results | ✅ Free | 24-hour cache; same query + same table = no charge |
| DML on partitioned tables | ✅ Free for partitioned | Only non-partitioned DML is charged |

---

## 12. Quick Reference & Cheatsheet Tables

### Data Type Decision Guide

| If you need... | Use |
|---|---|
| Text (any length) | `STRING` |
| Whole numbers | `INT64` |
| Floating point (approx) | `FLOAT64` |
| Exact decimal (financial) | `NUMERIC` (38 digits) or `BIGNUMERIC` |
| True/False | `BOOL` |
| Calendar date only | `DATE` |
| Timestamp with timezone | `TIMESTAMP` |
| Timestamp without timezone | `DATETIME` |
| Coordinates / shapes | `GEOGRAPHY` |
| Semi-structured data | `JSON` |
| Nested object | `STRUCT<field1 TYPE, field2 TYPE>` |
| List of values | `ARRAY<TYPE>` |
| Binary blob | `BYTES` |

---

### Partitioning vs. Clustering Decision Guide

| Scenario | Recommendation |
|---|---|
| Filter by date/time in most queries | **Partition by DATE/TIMESTAMP** |
| Filter by integer ranges (user_id segments) | **Partition by INTEGER RANGE** |
| Filter by string column (region, status) | **Cluster by that column** |
| Filter on multiple columns | **Cluster by multiple columns** (up to 4) |
| Need partition expiration | **Partition** (clustering has no expiration) |
| Both date AND string filters | **Partition by date + Cluster by string** |
| Table < 1 GB | Skip both — overhead not worth it |
| Table > 1 TB with no time dimension | **Cluster only** |

---

### Load Format Comparison

| Format | Schema | Types Preserved | Compression | Speed | Best For |
|---|---|---|---|---|---|
| CSV | External | ❌ (all strings) | gzip | Medium | Simple flat data |
| NDJSON | External / auto | ⚠️ Partial | gzip | Slow | Flexible schema |
| Avro | Embedded | ✅ All types | deflate/snappy | Fast | Schema evolution |
| Parquet | Embedded | ✅ All types + nested | snappy/zstd | **Fastest** | **Preferred format** |
| ORC | Embedded | ✅ All types | zlib/snappy | Fast | Hadoop workloads |

---

### IAM Roles Summary

| Role | Data Access | Job Submission | Admin |
|---|---|---|---|
| `bigquery.admin` | ✅ All | ✅ Yes | ✅ Full |
| `bigquery.dataOwner` | ✅ Read/Write | ❌ No | ✅ Dataset ACLs |
| `bigquery.dataEditor` | ✅ Read/Write | ❌ No | ❌ No |
| `bigquery.dataViewer` | ✅ Read only | ❌ No | ❌ No |
| `bigquery.jobUser` | ❌ No | ✅ Yes | ❌ No |
| `bigquery.user` | ⚠️ Limited | ✅ Yes | ❌ No |

> 💡 **Standard pattern:** `bigquery.jobUser` (project) + `bigquery.dataViewer` (dataset) = minimum read access.

---

### Common SQL Patterns

**Deduplication (keep latest row per key):**
```sql
SELECT * EXCEPT (row_num)
FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY updated_at DESC) AS row_num
  FROM users
)
WHERE row_num = 1;
```

**Sessionization (group events into sessions, 30-min gap):**
```sql
WITH gaps AS (
  SELECT *,
    TIMESTAMP_DIFF(event_ts,
      LAG(event_ts) OVER (PARTITION BY user_id ORDER BY event_ts),
      MINUTE) AS gap_min
  FROM events
),
sessions AS (
  SELECT *,
    SUM(IF(gap_min IS NULL OR gap_min > 30, 1, 0))
      OVER (PARTITION BY user_id ORDER BY event_ts) AS session_id
  FROM gaps
)
SELECT user_id, session_id, MIN(event_ts) AS session_start,
       MAX(event_ts) AS session_end, COUNT(*) AS events
FROM sessions
GROUP BY 1, 2;
```

**SCD Type 2 (track historical changes):**
```sql
MERGE target T
USING (SELECT *, CURRENT_TIMESTAMP() AS eff_start FROM staging) S
ON T.user_id = S.user_id AND T.is_current = TRUE

WHEN MATCHED AND (T.email != S.email OR T.name != S.name) THEN
  UPDATE SET T.is_current = FALSE, T.eff_end = S.eff_start

WHEN NOT MATCHED BY TARGET THEN
  INSERT (user_id, name, email, eff_start, eff_end, is_current)
  VALUES (S.user_id, S.name, S.email, S.eff_start, NULL, TRUE);
```

**PIVOT (rows to columns):**
```sql
SELECT *
FROM (SELECT user_id, product_category, amount FROM sales)
PIVOT (SUM(amount) FOR product_category IN ('electronics', 'clothing', 'food'));
```

**UNPIVOT (columns to rows):**
```sql
SELECT user_id, quarter, revenue
FROM quarterly_sales
UNPIVOT (revenue FOR quarter IN (q1, q2, q3, q4));
```

---

### Key bq CLI One-Liners

```bash
bq query --use_legacy_sql=false --dry_run 'SQL'          # Estimate cost
bq load --autodetect --source_format=CSV p:d.t gs://f    # Quick load
bq extract p:d.t 'gs://bucket/shard_*.parquet'           # Export
bq show --schema --format=prettyjson p:d.t               # Print schema
bq head -n 10 p:d.t                                      # Preview 10 rows
bq ls --format=prettyjson p:                             # List datasets (JSON)
bq update --expiration=86400 p:d.t                       # Set 1-day TTL
bq rm --force --table p:d.t                              # Force delete table
bq cp --sync=false p:d.src p:d.dst                       # Async table copy
bq query --maximum_bytes_billed=5000000000 'SQL'         # Hard cap at 5 GB
```

---

*Generated for GCP BigQuery | Official Docs: [cloud.google.com/bigquery/docs](https://cloud.google.com/bigquery/docs)*
