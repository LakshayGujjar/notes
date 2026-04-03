# 🌐 GCP Dataplex — Comprehensive Cheatsheet

> **Audience:** Data engineers, data platform engineers, data governance teams, and architects managing data lakes and governance on GCP.
> **Last updated:** March 2026 | Covers Universal Catalog, Data Quality, Lineage, Tasks, Data Mesh, and all core Dataplex features.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Lakes, Zones & Assets](#2-lakes-zones--assets)
3. [Dataplex Universal Catalog](#3-dataplex-universal-catalog)
4. [Metadata & Discovery](#4-metadata--discovery)
5. [Data Quality](#5-data-quality)
6. [Data Profiling](#6-data-profiling)
7. [Data Lineage](#7-data-lineage)
8. [IAM, Security & Data Governance](#8-iam-security--data-governance)
9. [Dataplex Tasks](#9-dataplex-tasks)
10. [Dataplex & BigQuery Integration](#10-dataplex--bigquery-integration)
11. [Data Mesh Implementation on GCP](#11-data-mesh-implementation-on-gcp)
12. [gcloud CLI & REST API Quick Reference](#12-gcloud-cli--rest-api-quick-reference)
13. [Monitoring, Observability & Alerting](#13-monitoring-observability--alerting)
14. [Pricing Summary](#14-pricing-summary)
15. [Quick Reference & Comparison Tables](#15-quick-reference--comparison-tables)

---

## 1. Overview & Key Concepts

### What is Dataplex?

**Google Cloud Dataplex** is a unified data governance and intelligent data lake management service. It provides an **active metadata layer** over your existing GCS and BigQuery data — without moving it — enabling discovery, quality, lineage, cataloging, and policy enforcement across your entire data estate.

**Core value proposition:**

| Pillar | Description |
|---|---|
| **Unified governance** | Centralized policies, security, and metadata across distributed data |
| **Data mesh enablement** | Domain-oriented lakes with federated ownership and shared governance |
| **Intelligent discovery** | Automatic schema, partition, and format detection across GCS/BigQuery |
| **Data quality at scale** | Rule-based scanning with BigQuery-powered quality checks |
| **Universal Catalog** | Single search surface for all data assets across the organization |
| **Data lineage** | Automatic and manual tracking of data movement and transformation |

---

### The Data Mesh Paradigm

Dataplex is purpose-built to enable **data mesh** on GCP:

| Data Mesh Principle | Dataplex Implementation |
|---|---|
| Domain ownership | One **Lake** per domain (e.g., `commerce-lake`, `finance-lake`) |
| Data as a product | **Curated zones** as governed, discoverable data products |
| Self-serve platform | Catalog search, quality scans, tasks — available to all teams |
| Federated governance | Org-wide policies + domain-level IAM via zones/lakes |

---

### Dataplex vs. Alternatives

| Dimension | Dataplex | Manual Lake Mgmt | Cloud Data Catalog (legacy) | BigQuery Only |
|---|---|---|---|---|
| **Governance** | ✅ Unified, active | ❌ DIY | ⚠️ Metadata only | ⚠️ BQ-only scope |
| **Discovery** | ✅ Auto GCS + BQ | ❌ Manual | ✅ BQ + limited GCS | ✅ BQ only |
| **Data quality** | ✅ Native scans | ❌ External tools | ❌ | ✅ BQ-only |
| **Lineage** | ✅ Auto + manual | ❌ | ❌ | ⚠️ Column-level only |
| **Data mesh** | ✅ Native | ❌ | ❌ | ❌ |
| **GCS management** | ✅ Full | ❌ | ⚠️ Limited | ❌ |
| **Best for** | Enterprise data governance | Simple single-project | Legacy BQ catalog | SQL-only analytics |

---

### Core Architecture

```
GCP Organization
└── GCP Project
    └── Dataplex Control Plane
        ├── Lakes (domain groupings)
        │   ├── Zones (raw / curated)
        │   │   └── Assets (GCS buckets, BQ datasets)
        │   └── Tasks (Spark/PySpark jobs)
        ├── Universal Catalog (metadata layer)
        │   ├── Entry Groups → Entries → Aspects
        │   └── Business Glossary → Terms
        ├── Data Quality Scans
        ├── Data Profile Scans
        └── Data Lineage (processes → runs → events)

Data Plane (unchanged — data stays where it is):
  ├── GCS Buckets       (Raw zone: gs://my-raw-bucket/)
  ├── BigQuery Datasets (Curated zone: my_project.my_dataset)
  └── Other systems     (Spanner, Bigtable, etc. — via Catalog)
```

> 💡 Dataplex is a **non-invasive overlay** — it never copies, moves, or modifies your data. It manages metadata, policies, and scans over your existing storage.

---

### Universal Catalog vs. Legacy Cloud Data Catalog

| Feature | Universal Catalog (Dataplex) | Legacy Cloud Data Catalog |
|---|---|---|
| **Entries** | Entries with Aspects | Entries with Tags |
| **Metadata model** | Aspect Types (flexible schema) | Tag Templates (rigid schema) |
| **Glossary** | ✅ Native business glossary | ❌ |
| **Lineage** | ✅ Integrated | ❌ (separate product) |
| **GCS discovery** | ✅ Via Dataplex lakes | ⚠️ Manual entry |
| **Search** | ✅ Unified across all types | ✅ BQ-focused |
| **Status** | ✅ Active development | ⚠️ Legacy (migrate to Dataplex) |

> ⚠️ **Migration note:** Cloud Data Catalog Tag Templates migrate to **Aspect Types**. Tags migrate to **Aspects**. Entries remain entries. Google provides an official migration guide.

---

### GCP Integration Map

```
Dataplex
  ├── BigQuery       — auto-catalog tables, column lineage, quality scans, policy tags
  ├── GCS            — discover files, infer schemas, manage as zones/assets
  ├── Dataproc       — run Tasks on managed clusters
  ├── Dataflow       — auto-publish lineage events to Dataplex
  ├── Cloud Data Fusion — auto-publish lineage events
  ├── Vertex AI      — catalog models, pipelines, datasets; publish lineage
  ├── Looker         — discover governed tables via Catalog; glossary-term alignment
  └── Cloud DLP      — scan for PII; tag sensitive columns in Catalog
```

---

### Key Terminology

| Term | Definition |
|---|---|
| **Lake** | Top-level domain container — groups zones, assets, and tasks |
| **Zone** | Sub-domain within a lake — Raw (raw data) or Curated (governed data) |
| **Asset** | A GCS bucket or BigQuery dataset attached to a zone |
| **Entity** | A discovered table or fileset within an asset (auto-registered) |
| **Entry** | A Universal Catalog record for any data resource |
| **Entry Group** | Container organizing related entries |
| **Aspect** | Structured metadata attached to an entry (replaces Tag) |
| **Aspect Type** | Schema definition for an aspect (replaces Tag Template) |
| **Data Scan** | A scheduled or on-demand quality or profile scan |
| **Task** | A managed Spark/PySpark/Notebook job run on Dataplex Dataproc |
| **Lineage Event** | A record of data movement from source(s) to target(s) |
| **Glossary Term** | A business definition linked to entries and columns |

---

### Key Limits & Quotas

| Resource | Limit |
|---|---|
| Lakes per project | 10 |
| Zones per lake | 10 |
| Assets per zone | 100 |
| Data scans per project | 100 |
| Tasks per lake | 100 |
| Entry groups per project | 10,000 |
| Glossary terms per project | 1,000 |
| Lineage events per day | 10,000 (default) |

---

## 2. Lakes, Zones & Assets

### Lake Structure

```
Lake: commerce-lake  (domain = e-commerce)
├── Zone: raw-zone       (type=RAW)    ← unvalidated ingested data
│   ├── Asset: raw-events-bucket     (GCS: gs://raw-events/)
│   └── Asset: raw-orders-bucket     (GCS: gs://raw-orders/)
└── Zone: curated-zone   (type=CURATED) ← validated, governed data products
    ├── Asset: orders-dataset        (BQ: my_proj.orders)
    └── Asset: customers-dataset     (BQ: my_proj.customers)
```

---

### Zone Types

| Feature | Raw Zone | Curated Zone |
|---|---|---|
| **Data state** | Unvalidated, as-ingested | Validated, cleaned, governed |
| **Allowed formats** | Any (CSV, JSON, Parquet, ORC, Avro, binary) | Structured formats only (Parquet, ORC, Avro, BQ tables) |
| **Schema enforcement** | ❌ Flexible | ✅ Enforced on write |
| **IAM defaults** | Restrictive | More accessible |
| **Use case** | Landing zone, data ingestion | Data products, analytics |

---

### gcloud: Lakes, Zones & Assets

```bash
# ── LAKES ─────────────────────────────────────────────────────────────
# Create a lake
gcloud dataplex lakes create commerce-lake \
  --location=us-central1 \
  --display-name="Commerce Domain Lake" \
  --description="All e-commerce data assets" \
  --labels=domain=commerce,env=prod

# List lakes
gcloud dataplex lakes list --location=us-central1 \
  --format="table(name,displayName,state,metastore.service)"

# Describe a lake
gcloud dataplex lakes describe commerce-lake --location=us-central1

# Delete a lake
gcloud dataplex lakes delete commerce-lake --location=us-central1 --quiet

# ── ZONES ─────────────────────────────────────────────────────────────
# Create a raw zone
gcloud dataplex zones create raw-zone \
  --lake=commerce-lake \
  --location=us-central1 \
  --type=RAW \
  --resource-location-type=SINGLE_REGION \
  --display-name="Raw Ingestion Zone" \
  --discovery-enabled \
  --discovery-schedule="0 */6 * * *"    # Every 6 hours

# Create a curated zone
gcloud dataplex zones create curated-zone \
  --lake=commerce-lake \
  --location=us-central1 \
  --type=CURATED \
  --resource-location-type=SINGLE_REGION \
  --display-name="Curated Data Products" \
  --discovery-enabled \
  --discovery-schedule="0 2 * * *"      # Daily at 2am

# List zones in a lake
gcloud dataplex zones list \
  --lake=commerce-lake \
  --location=us-central1

# ── ASSETS ────────────────────────────────────────────────────────────
# Attach a GCS bucket as an asset
gcloud dataplex assets create raw-events-asset \
  --zone=raw-zone \
  --lake=commerce-lake \
  --location=us-central1 \
  --resource-type=STORAGE_BUCKET \
  --resource-name=projects/my-project/buckets/raw-events-bucket \
  --display-name="Raw Events Bucket" \
  --discovery-enabled \
  --discovery-include-patterns="**.parquet,**.json"

# Attach a BigQuery dataset as an asset
gcloud dataplex assets create orders-bq-asset \
  --zone=curated-zone \
  --lake=commerce-lake \
  --location=us-central1 \
  --resource-type=BIGQUERY_DATASET \
  --resource-name=projects/my-project/datasets/orders \
  --display-name="Orders BigQuery Dataset" \
  --discovery-enabled

# List assets
gcloud dataplex assets list \
  --zone=curated-zone --lake=commerce-lake --location=us-central1

# Describe an asset
gcloud dataplex assets describe raw-events-asset \
  --zone=raw-zone --lake=commerce-lake --location=us-central1
```

---

### Terraform: Lake with Zones and Assets

```yaml
# main.tf
resource "google_dataplex_lake" "commerce" {
  name         = "commerce-lake"
  location     = "us-central1"
  display_name = "Commerce Domain Lake"
  description  = "All e-commerce data assets"

  labels = {
    domain = "commerce"
    env    = "prod"
  }
}

resource "google_dataplex_zone" "raw" {
  name         = "raw-zone"
  lake         = google_dataplex_lake.commerce.name
  location     = "us-central1"
  type         = "RAW"
  display_name = "Raw Ingestion Zone"

  resource_spec {
    location_type = "SINGLE_REGION"
  }

  discovery_spec {
    enabled  = true
    schedule = "0 */6 * * *"
    include_patterns = ["**.parquet", "**.json", "**.csv"]
    json_options {
      disable_type_inference = false
    }
    csv_options {
      delimiter          = ","
      header_rows        = 1
      disable_type_inference = false
    }
  }
}

resource "google_dataplex_zone" "curated" {
  name         = "curated-zone"
  lake         = google_dataplex_lake.commerce.name
  location     = "us-central1"
  type         = "CURATED"
  display_name = "Curated Data Products"

  resource_spec {
    location_type = "SINGLE_REGION"
  }

  discovery_spec {
    enabled  = true
    schedule = "0 2 * * *"
  }
}

resource "google_dataplex_asset" "raw_events" {
  name         = "raw-events-asset"
  location     = "us-central1"
  lake         = google_dataplex_lake.commerce.name
  dataplex_zone = google_dataplex_zone.raw.name
  display_name = "Raw Events Bucket"

  resource_spec {
    name = "projects/${var.project}/buckets/raw-events-bucket"
    type = "STORAGE_BUCKET"
  }

  discovery_spec {
    enabled = true
  }
}

resource "google_dataplex_asset" "orders_bq" {
  name         = "orders-bq-asset"
  location     = "us-central1"
  lake         = google_dataplex_lake.commerce.name
  dataplex_zone = google_dataplex_zone.curated.name
  display_name = "Orders BigQuery Dataset"

  resource_spec {
    name = "projects/${var.project}/datasets/orders"
    type = "BIGQUERY_DATASET"
  }

  discovery_spec {
    enabled = true
  }
}
```

---

## 3. Dataplex Universal Catalog

### Catalog Model

```
Universal Catalog
├── Entry Groups (container for related entries)
│   ├── "commerce-entries"  (entries for commerce domain)
│   └── "marketing-entries"
└── Entries (one per data resource)
    ├── Entry: bigquery://projects/proj/datasets/orders/tables/order_facts
    │   ├── System Aspect: schema (auto-populated)
    │   ├── System Aspect: data_profile (from profile scan)
    │   └── Custom Aspect: data_ownership (steward, SLA, domain)
    └── Entry: gs://bucket/events/dt=2026-03-16/
        ├── System Aspect: fileset_spec
        └── Custom Aspect: ingestion_metadata
```

---

### gcloud: Universal Catalog Operations

```bash
# ── ENTRY GROUPS ──────────────────────────────────────────────────────
# Create an entry group
gcloud dataplex entry-groups create commerce-entries \
  --location=us-central1 \
  --display-name="Commerce Domain Entries" \
  --description="All catalog entries for the commerce domain" \
  --labels=domain=commerce

# List entry groups
gcloud dataplex entry-groups list --location=us-central1

# ── ASPECT TYPES ──────────────────────────────────────────────────────
# Create a custom aspect type (defines metadata schema)
gcloud dataplex aspect-types create data-ownership-aspect \
  --location=us-central1 \
  --display-name="Data Ownership" \
  --description="Ownership and SLA metadata for data assets" \
  --metadata-template='{
    "name": "data-ownership",
    "type": "record",
    "fields": [
      {"name": "steward_email", "type": "string"},
      {"name": "team",          "type": "string"},
      {"name": "sla_hours",     "type": "integer"},
      {"name": "pii_flag",      "type": "boolean"},
      {"name": "domain",        "type": "string",
       "enumValues": ["commerce", "finance", "marketing", "platform"]}
    ]
  }'

# ── ENTRIES ────────────────────────────────────────────────────────────
# Create a custom entry (for non-GCP or custom resources)
gcloud dataplex entries create my-kafka-topic \
  --entry-group=commerce-entries \
  --location=us-central1 \
  --entry-type=projects/my-project/locations/us-central1/entryTypes/kafka-topic \
  --display-name="Orders Kafka Topic" \
  --description="Kafka topic receiving order events"

# Add an aspect to an entry
gcloud dataplex entries update my-kafka-topic \
  --entry-group=commerce-entries \
  --location=us-central1 \
  --aspect-keys=projects/my-project/locations/us-central1/aspectTypes/data-ownership-aspect \
  --aspects='{"steward_email":"data-team@company.com","team":"commerce","sla_hours":4,"pii_flag":false,"domain":"commerce"}'

# Search the catalog
gcloud dataplex entries search \
  --location=us-central1 \
  --query="orders type=TABLE" \
  --order-by="relevance" \
  --format="table(name,displayName,entryType,aspects)"

# Full-text search with filters
gcloud dataplex entries search \
  --location=global \
  --query='name:orders AND aspect:data-ownership AND label.domain=commerce'
```

---

### Python: Catalog Operations

```python
from google.cloud import dataplex_v1
from google.cloud.dataplex_v1.types import catalog as catalog_types
import json

# Initialize client
catalog_client = dataplex_v1.CatalogServiceClient()

PROJECT   = "my-project"
LOCATION  = "us-central1"

# ── Search catalog ────────────────────────────────────────────────────
def search_catalog(query: str, order_by: str = "relevance") -> list:
    request = dataplex_v1.SearchEntriesRequest(
        name   = f"projects/{PROJECT}/locations/{LOCATION}",
        query  = query,
        order_by = order_by,
        page_size = 50,
    )
    results = catalog_client.search_entries(request=request)
    return list(results)

entries = search_catalog("orders type=TABLE")
for entry in entries:
    print(f"{entry.dataplex_entry.name} — {entry.dataplex_entry.display_name}")

# ── Get entry details ─────────────────────────────────────────────────
entry = catalog_client.get_entry(
    name=f"projects/{PROJECT}/locations/{LOCATION}/entryGroups/commerce-entries/entries/my-entry"
)
print(f"Entry: {entry.display_name}")
for aspect_key, aspect in entry.aspects.items():
    print(f"  Aspect [{aspect_key}]: {json.dumps(dict(aspect.data), indent=2)}")

# ── Create a business glossary term ───────────────────────────────────
glossary_client = dataplex_v1.CatalogServiceClient()

# Create a glossary first (via REST or SDK)
# Then create terms
term = catalog_client.create_glossary_term(
    parent=f"projects/{PROJECT}/locations/{LOCATION}/glossaries/business-glossary",
    glossary_term_id="revenue",
    glossary_term=dataplex_v1.GlossaryTerm(
        display_name  = "Revenue",
        description   = "Total recognized revenue from completed orders, net of returns and discounts.",
        status        = dataplex_v1.GlossaryTerm.Status.PUBLISHED,
    )
)
print(f"Created term: {term.name}")
```

---

### Business Glossary

```bash
# Create a glossary
gcloud dataplex glossaries create business-glossary \
  --location=us-central1 \
  --display-name="Business Glossary" \
  --description="Canonical business term definitions"

# Create a glossary term
gcloud dataplex glossaries terms create revenue \
  --glossary=business-glossary \
  --location=us-central1 \
  --display-name="Revenue" \
  --description="Total recognized revenue from completed orders, net of returns." \
  --status=PUBLISHED

# Create a child term (hierarchy)
gcloud dataplex glossaries terms create gross-revenue \
  --glossary=business-glossary \
  --location=us-central1 \
  --display-name="Gross Revenue" \
  --parent-term=revenue \
  --status=PUBLISHED

# List terms
gcloud dataplex glossaries terms list \
  --glossary=business-glossary \
  --location=us-central1
```

---

## 4. Metadata & Discovery

### How Discovery Works

```
Dataplex Discovery Process (runs on schedule):
  1. Scans GCS paths under the asset bucket
  2. Detects file formats: Parquet, Avro, ORC, CSV, JSON, Text
  3. Infers schema from file headers and data samples
  4. Detects partitions: Hive-style (key=value/) or date-based
  5. Registers tables and filesets in the entity catalog
  6. Publishes metadata to Universal Catalog as system aspects
  7. Syncs to Cloud Metastore (Hive Metastore) if configured
```

---

### Discovery Configuration

```bash
# Zone-level discovery with include/exclude patterns
gcloud dataplex zones update raw-zone \
  --lake=commerce-lake \
  --location=us-central1 \
  --discovery-enabled \
  --discovery-schedule="0 */4 * * *" \
  --discovery-include-patterns="events/**,orders/**" \
  --discovery-exclude-patterns="**/_tmp/**,**/staging/**"

# Zone-level CSV options
gcloud dataplex zones update raw-zone \
  --lake=commerce-lake \
  --location=us-central1 \
  --discovery-csv-header-rows=1 \
  --discovery-csv-delimiter="," \
  --discovery-csv-disable-type-inference=false

# Zone-level JSON options
gcloud dataplex zones update raw-zone \
  --lake=commerce-lake \
  --location=us-central1 \
  --discovery-json-disable-type-inference=false
```

---

### Partition Detection

Dataplex supports two partition styles:

```
Hive-style (recommended):
gs://bucket/events/
  dt=2026-03-16/
    region=us/
      data.parquet
    region=eu/
      data.parquet
  dt=2026-03-17/
    ...

Non-Hive (date-directory):
gs://bucket/events/
  2026-03-16/
    us/
      data.parquet

Custom keys defined in asset discovery config.
```

---

### Querying Entities (Python)

```python
from google.cloud import dataplex_v1

metadata_client = dataplex_v1.MetadataServiceClient()

PROJECT  = "my-project"
LOCATION = "us-central1"
LAKE     = "commerce-lake"
ZONE     = "raw-zone"

# List all entities in a zone
parent = f"projects/{PROJECT}/locations/{LOCATION}/lakes/{LAKE}/zones/{ZONE}"

entities = metadata_client.list_entities(parent=parent)
for entity in entities:
    print(f"Entity: {entity.id}")
    print(f"  Type:   {entity.type_.name}")
    print(f"  Format: {entity.format_.format_.name}")
    print(f"  Path:   {entity.data_path}")
    if entity.schema_:
        for field in entity.schema_.fields:
            print(f"  Field:  {field.name} ({field.type_.name})")
    print()

# Get a specific entity with full schema
entity = metadata_client.get_entity(
    name=f"{parent}/entities/events_dt"
)

# List partitions of an entity
partitions = metadata_client.list_partitions(
    parent=f"{parent}/entities/events_dt"
)
for p in partitions:
    print(f"Partition: {p.name}, values={p.values}")
```

---

### Best Practices for Discovery

| Practice | Recommendation |
|---|---|
| **Folder structure** | Use `domain/entity_name/partition_key=value/` hierarchy |
| **Hive partitioning** | Always use `key=value` folder names for auto-detection |
| **File formats** | Prefer Parquet or Avro — richer schema inference than CSV |
| **Consistent naming** | Use snake_case for all folder and file names |
| **Exclude patterns** | Always exclude `_tmp`, `staging`, `_checkpoint` directories |
| **Schema consistency** | All files in the same entity path should share the same schema |
| **File size** | Avoid millions of tiny files — impacts discovery performance |

---

## 5. Data Quality

### Data Quality Scan Overview

```
Data Quality Scan:
  Input:  BigQuery table or GCS-based entity
  Rules:  One or more quality rules per column/row
  Output: Pass/fail score per rule, per column, overall
          → Published to Dataplex Catalog
          → Optionally exported to BigQuery table
          → Optionally triggers Cloud Logging alert
```

---

### Rule Types Reference

| Rule Type | Dimension | Description | Example |
|---|---|---|---|
| `NON_NULL_CHECK` | Completeness | Field must not be NULL | `order_id IS NOT NULL` |
| `RANGE_CHECK` | Validity | Value within min/max bounds | `0 <= amount <= 100000` |
| `SET_CHECK` | Validity | Value in allowed set | `status IN ('complete','pending','cancelled')` |
| `REGEX_CHECK` | Validity | Matches regex pattern | `email REGEXP '^[^@]+@[^@]+$'` |
| `UNIQUENESS_CHECK` | Uniqueness | No duplicate values in column | `COUNT(DISTINCT order_id) = COUNT(*)` |
| `SQL_ASSERTION` | Custom | Arbitrary SQL returning row count (0=pass) | `SELECT COUNT(*) FROM t WHERE amount < 0` |
| `ROW_CONDITION` | Validity | Each row must satisfy SQL condition | `amount > 0 AND currency IS NOT NULL` |
| `STATISTIC_RANGE_CHECK` | Accuracy | Aggregate stat within range | `AVG(amount) BETWEEN 50 AND 500` |

---

### gcloud: Data Quality Scans

```bash
# Create a data quality scan on a BigQuery table
gcloud dataplex datascans create orders-quality-scan \
  --location=us-central1 \
  --type=DATA_QUALITY \
  --data-source-resource=//bigquery.googleapis.com/projects/my-project/datasets/orders/tables/order_facts \
  --display-name="Orders Data Quality Scan" \
  --description="Daily quality check on order_facts table" \
  --schedule="0 6 * * *"

# Run a scan immediately (on-demand)
gcloud dataplex datascans run orders-quality-scan \
  --location=us-central1

# Get the latest scan result
gcloud dataplex datascans get-iam-policy orders-quality-scan \
  --location=us-central1

# Describe scan (includes latest result summary)
gcloud dataplex datascans describe orders-quality-scan \
  --location=us-central1

# List all datascans
gcloud dataplex datascans list \
  --location=us-central1 \
  --format="table(name,displayName,type,state,dataQualitySpec.rules.count())"

# List scan jobs (run history)
gcloud dataplex datascans jobs list \
  --datascan=orders-quality-scan \
  --location=us-central1
```

---

### JSON Rule Configuration

```json
{
  "dataQualitySpec": {
    "samplingPercent": 100,
    "rowFilter": "order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)",
    "postScanActions": {
      "bigqueryExport": {
        "resultsTable": "//bigquery.googleapis.com/projects/my-project/datasets/dq_results/tables/orders_quality"
      }
    },
    "rules": [
      {
        "column":    "order_id",
        "name":      "order_id_not_null",
        "dimension": "COMPLETENESS",
        "description": "Order ID must never be null",
        "nonNullExpectation": {},
        "threshold": 1.0
      },
      {
        "column":    "amount",
        "name":      "amount_positive_range",
        "dimension": "VALIDITY",
        "rangeExpectation": {
          "minValue":      "0.01",
          "maxValue":      "100000.00",
          "strictMinEnabled": true,
          "strictMaxEnabled": false
        },
        "threshold": 0.99
      },
      {
        "column": "status",
        "name":   "status_valid_values",
        "dimension": "VALIDITY",
        "setExpectation": {
          "values": ["complete", "pending", "cancelled", "refunded"]
        },
        "threshold": 1.0
      },
      {
        "column": "customer_email",
        "name":   "email_format_check",
        "dimension": "VALIDITY",
        "regexExpectation": {
          "regex": "^[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}$"
        },
        "threshold": 0.95
      },
      {
        "column": "order_id",
        "name":   "order_id_unique",
        "dimension": "UNIQUENESS",
        "uniquenessExpectation": {},
        "threshold": 1.0
      },
      {
        "name":      "no_negative_amounts",
        "dimension": "VALIDITY",
        "rowConditionExpectation": {
          "sqlExpression": "amount > 0 AND amount IS NOT NULL"
        },
        "threshold": 1.0
      },
      {
        "name":      "avg_order_sanity_check",
        "dimension": "ACCURACY",
        "statisticRangeExpectation": {
          "statistic":  "MEAN",
          "column":     "amount",
          "minValue":   "10.0",
          "maxValue":   "1000.0"
        },
        "threshold": 1.0
      },
      {
        "name":      "no_future_dates",
        "dimension": "CONSISTENCY",
        "sqlAssertion": {
          "sqlStatement": "SELECT COUNT(*) FROM `my-project.orders.order_facts` WHERE order_date > CURRENT_DATE()"
        },
        "threshold": 1.0
      }
    ]
  }
}
```

---

### Python: Data Quality Scan

```python
from google.cloud import dataplex_v1
from google.cloud.dataplex_v1.types import datascans as scan_types
from google.protobuf import field_mask_pb2

scan_client = dataplex_v1.DataScanServiceClient()

PROJECT  = "my-project"
LOCATION = "us-central1"

# Create a data quality scan
scan = dataplex_v1.DataScan(
    display_name = "Orders Quality Scan",
    description  = "Daily quality checks on order_facts",
    data = dataplex_v1.DataSource(
        resource = "//bigquery.googleapis.com/projects/my-project/datasets/orders/tables/order_facts"
    ),
    data_quality_spec = dataplex_v1.DataQualitySpec(
        sampling_percent = 100,
        row_filter = "order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)",
        rules = [
            dataplex_v1.DataQualityRule(
                column    = "order_id",
                name      = "order_id_not_null",
                dimension = "COMPLETENESS",
                non_null_expectation = dataplex_v1.DataQualityRule.NonNullExpectation(),
                threshold = 1.0,
            ),
            dataplex_v1.DataQualityRule(
                column    = "amount",
                name      = "amount_valid_range",
                dimension = "VALIDITY",
                range_expectation = dataplex_v1.DataQualityRule.RangeExpectation(
                    min_value = "0.01",
                    max_value = "100000.00",
                ),
                threshold = 0.99,
            ),
        ],
        post_scan_actions = dataplex_v1.DataQualitySpec.PostScanActions(
            bigquery_export = dataplex_v1.DataQualitySpec.PostScanActions.BigQueryExport(
                results_table = "//bigquery.googleapis.com/projects/my-project/datasets/dq_results/tables/orders_quality"
            )
        ),
    ),
    execution_spec = dataplex_v1.DataScan.ExecutionSpec(
        trigger = dataplex_v1.Trigger(
            schedule = dataplex_v1.Trigger.Schedule(cron = "0 6 * * *")
        )
    ),
)

operation = scan_client.create_data_scan(
    parent        = f"projects/{PROJECT}/locations/{LOCATION}",
    data_scan_id  = "orders-quality-scan",
    data_scan     = scan,
)
result = operation.result()
print(f"Created scan: {result.name}")

# Run a scan on demand
run_op = scan_client.run_data_scan(
    name = f"projects/{PROJECT}/locations/{LOCATION}/dataScans/orders-quality-scan"
)
job = run_op.result()
print(f"Scan job: {job.name}, State: {job.state.name}")

# Get scan results
scan_job = scan_client.get_data_scan_job(name=job.name, view="FULL")
if scan_job.data_quality_result:
    result = scan_job.data_quality_result
    print(f"Passed: {result.passed}")
    print(f"Score:  {result.score:.1%}")
    for rule_result in result.rules:
        status = "✅" if rule_result.passed else "❌"
        print(f"  {status} {rule_result.rule.name}: {rule_result.pass_ratio:.1%}")
```

---

## 6. Data Profiling

### What Profile Scans Compute

| Metric | Type | Description |
|---|---|---|
| `null_count` / `null_ratio` | Completeness | Number and percentage of NULLs |
| `distinct_count` / `distinct_ratio` | Uniqueness | Unique value count |
| `min` / `max` | Range | Minimum and maximum values |
| `mean` | Central tendency | Average value (numeric) |
| `std_dev` | Spread | Standard deviation (numeric) |
| `quartiles` (p25, p50, p75) | Distribution | Percentile values |
| `top_n_values` | Frequency | Most common values with counts |
| `column_type` | Schema | Inferred data type |

> 💡 **Profiling vs. Quality:** Profiling is **descriptive** (what does the data look like?). Quality is **prescriptive** (does the data meet rules?). Run a profile scan first to understand your data before writing quality rules.

---

### gcloud & Python: Profile Scans

```bash
# Create a profile scan
gcloud dataplex datascans create orders-profile-scan \
  --location=us-central1 \
  --type=DATA_PROFILE \
  --data-source-resource=//bigquery.googleapis.com/projects/my-project/datasets/orders/tables/order_facts \
  --display-name="Orders Data Profile" \
  --schedule="0 1 * * 0"   # Weekly on Sunday at 1am

# Run on demand
gcloud dataplex datascans run orders-profile-scan \
  --location=us-central1

# Get the latest profile results
gcloud dataplex datascans jobs describe SCAN_JOB_ID \
  --datascan=orders-profile-scan \
  --location=us-central1 \
  --view=FULL \
  --format=json
```

```python
# Create a data profile scan with sampling
from google.cloud import dataplex_v1

scan_client = dataplex_v1.DataScanServiceClient()

profile_scan = dataplex_v1.DataScan(
    display_name = "Orders Profile Scan",
    data = dataplex_v1.DataSource(
        resource = "//bigquery.googleapis.com/projects/my-project/datasets/orders/tables/order_facts"
    ),
    data_profile_spec = dataplex_v1.DataProfileSpec(
        sampling_percent = 30.0,          # Sample 30% to reduce BQ cost
        row_filter = "order_date >= '2026-01-01'",
        include_fields = dataplex_v1.DataProfileSpec.SelectedFields(
            field_names = ["order_id", "amount", "status", "region", "order_date"]
        ),
        post_scan_actions = dataplex_v1.DataProfileSpec.PostScanActions(
            bigquery_export = dataplex_v1.DataProfileSpec.PostScanActions.BigQueryExport(
                results_table = "//bigquery.googleapis.com/projects/my-project/datasets/dq_results/tables/orders_profile"
            )
        ),
    ),
    execution_spec = dataplex_v1.DataScan.ExecutionSpec(
        trigger = dataplex_v1.Trigger(
            schedule = dataplex_v1.Trigger.Schedule(cron = "0 1 * * 0")
        )
    ),
)

operation = scan_client.create_data_scan(
    parent       = "projects/my-project/locations/us-central1",
    data_scan_id = "orders-profile-scan",
    data_scan    = profile_scan,
)
result = operation.result()

# Retrieve profile results
scan_job = scan_client.get_data_scan_job(
    name = f"{result.name}/jobs/latest",
    view = "FULL"
)

if scan_job.data_profile_result:
    for col in scan_job.data_profile_result.profile.fields:
        print(f"\nColumn: {col.name} ({col.type_})")
        stats = col.profile
        if stats.null_ratio is not None:
            print(f"  Null ratio:    {stats.null_ratio:.1%}")
        if stats.distinct_ratio is not None:
            print(f"  Distinct ratio:{stats.distinct_ratio:.1%}")
        if hasattr(stats, 'double_profile'):
            dp = stats.double_profile
            print(f"  Min/Max:       {dp.min:.2f} / {dp.max:.2f}")
            print(f"  Mean/StdDev:   {dp.average:.2f} / {dp.standard_deviation:.2f}")
```

> 💰 **Cost tip:** Set `sampling_percent` to 10–30% for large tables. Profile results are directionally accurate with sampling, and you avoid full-table BQ scans on every run.

---

## 7. Data Lineage

### Lineage Model

```
Lineage Model:
  Process  → represents a recurring job/pipeline
  Run      → a single execution of that process
  Event    → what happened during that run (sources → targets)

Example:
  Process: "orders-daily-etl"
  Run:     "orders-daily-etl/run-20260316"
  Event:   sources=[raw_orders_table]
           targets=[curated_order_facts_table]

Automatic lineage (no code required):
  BigQuery jobs          → column-level lineage for CTAS, INSERT, MERGE
  Dataflow pipelines     → dataset-level lineage
  Cloud Data Fusion      → pipeline-level lineage
  Vertex AI Pipelines    → model training lineage
```

---

### Automatic Lineage Sources

| Service | Lineage Granularity | Trigger |
|---|---|---|
| **BigQuery** | Column-level (CTAS, INSERT, MERGE, COPY) | Automatic on job completion |
| **Dataflow** | Dataset-level | Automatic on pipeline run |
| **Cloud Data Fusion** | Pipeline-level | Automatic per pipeline run |
| **Vertex AI Pipelines** | Dataset/model level | Automatic on pipeline run |
| **Custom (API)** | Any | Manually via Lineage API |

---

### Python: Custom Lineage Events

```python
from google.cloud import datacatalog_lineage_v1
from google.protobuf import timestamp_pb2
import datetime

lineage_client = datacatalog_lineage_v1.LineageClient()

PROJECT  = "my-project"
LOCATION = "us"    # Lineage uses "us" or "eu" not specific regions

# ── Create a Process ──────────────────────────────────────────────────
process = lineage_client.create_process(
    parent = f"projects/{PROJECT}/locations/{LOCATION}",
    process = datacatalog_lineage_v1.Process(
        display_name = "Custom ETL: MySQL → BigQuery",
        attributes   = {
            "tool":    "python-etl",
            "owner":   "data-engineering-team",
            "version": "1.2.0"
        }
    )
)
print(f"Process: {process.name}")

# ── Create a Run ──────────────────────────────────────────────────────
now = datetime.datetime.utcnow()
start_ts = timestamp_pb2.Timestamp()
start_ts.FromDatetime(now - datetime.timedelta(minutes=30))
end_ts = timestamp_pb2.Timestamp()
end_ts.FromDatetime(now)

run = lineage_client.create_run(
    parent = process.name,
    run = datacatalog_lineage_v1.Run(
        display_name = f"ETL run {now.strftime('%Y%m%d-%H%M%S')}",
        state        = datacatalog_lineage_v1.Run.State.COMPLETED,
        start_time   = start_ts,
        end_time     = end_ts,
        attributes   = {"rows_processed": "142500", "status": "success"}
    )
)
print(f"Run: {run.name}")

# ── Create a Lineage Event ────────────────────────────────────────────
event = lineage_client.create_lineage_event(
    parent = run.name,
    lineage_event = datacatalog_lineage_v1.LineageEvent(
        start_time = start_ts,
        end_time   = end_ts,
        links = [
            datacatalog_lineage_v1.EventLink(
                source = datacatalog_lineage_v1.EntityReference(
                    fully_qualified_name = "mysql:my-mysql-host.orders_db.orders"
                ),
                target = datacatalog_lineage_v1.EntityReference(
                    fully_qualified_name = "bigquery:my-project.orders.order_facts"
                ),
            )
        ]
    )
)
print(f"Lineage event: {event.name}")

# ── Query Lineage ─────────────────────────────────────────────────────
# Find all processes that wrote to a BigQuery table
search_results = lineage_client.search_links(
    request = datacatalog_lineage_v1.SearchLinksRequest(
        parent = f"projects/{PROJECT}/locations/{LOCATION}",
        target = datacatalog_lineage_v1.EntityReference(
            fully_qualified_name = "bigquery:my-project.orders.order_facts"
        )
    )
)
print("\nUpstream lineage for order_facts:")
for link in search_results:
    print(f"  Source: {link.source.fully_qualified_name}")
    print(f"  Target: {link.target.fully_qualified_name}")

# Find all tables downstream of a source
downstream = lineage_client.search_links(
    request = datacatalog_lineage_v1.SearchLinksRequest(
        parent = f"projects/{PROJECT}/locations/{LOCATION}",
        source = datacatalog_lineage_v1.EntityReference(
            fully_qualified_name = "bigquery:my-project.orders.order_facts"
        )
    )
)
print("\nDownstream lineage from order_facts:")
for link in downstream:
    print(f"  → {link.target.fully_qualified_name}")
```

---

### REST API: Lineage Events

```bash
TOKEN=$(gcloud auth print-access-token)
BASE="https://datalineage.googleapis.com/v1/projects/my-project/locations/us"

# Create a process
curl -X POST "${BASE}/processes" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"displayName": "My Custom ETL Process",
       "attributes": {"tool": "custom-python"}}'

# Create a run
curl -X POST "${BASE}/processes/PROCESS_ID/runs" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"displayName": "run-20260316",
       "state": "COMPLETED",
       "startTime": "2026-03-16T02:00:00Z",
       "endTime":   "2026-03-16T02:30:00Z"}'

# Create a lineage event
curl -X POST "${BASE}/processes/PROCESS_ID/runs/RUN_ID/lineageEvents" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "startTime": "2026-03-16T02:00:00Z",
    "endTime":   "2026-03-16T02:30:00Z",
    "links": [{
      "source": {"fullyQualifiedName": "bigquery:my-project.raw.orders"},
      "target": {"fullyQualifiedName": "bigquery:my-project.curated.order_facts"}
    }]
  }'
```

---

## 8. IAM, Security & Data Governance

### Dataplex IAM Roles

| Role | Scope | Key Permissions |
|---|---|---|
| `roles/dataplex.admin` | Project | Full control: create/delete lakes, zones, assets, scans |
| `roles/dataplex.editor` | Project | Create/edit all resources; no delete of lakes |
| `roles/dataplex.viewer` | Project | Read all Dataplex resources and metadata |
| `roles/dataplex.dataReader` | Lake/Zone | Read data in assets (GCS read + BQ query) |
| `roles/dataplex.dataWriter` | Lake/Zone | Read + write data in assets |
| `roles/dataplex.metadataReader` | Project | Read Catalog entries, aspects, lineage |
| `roles/dataplex.dataOwner` | Lake/Zone | Full data access including schema changes |

---

### IAM Binding Examples

```bash
# Grant data read access at zone level (not whole project)
gcloud dataplex zones add-iam-policy-binding curated-zone \
  --lake=commerce-lake \
  --location=us-central1 \
  --member="group:analytics-team@company.com" \
  --role="roles/dataplex.dataReader"

# Grant data write access at lake level
gcloud dataplex lakes add-iam-policy-binding commerce-lake \
  --location=us-central1 \
  --member="serviceAccount:etl-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/dataplex.dataWriter"

# Grant catalog metadata read
gcloud projects add-iam-policy-binding my-project \
  --member="group:data-consumers@company.com" \
  --role="roles/dataplex.metadataReader"

# Grant admin at lake level for domain owners
gcloud dataplex lakes add-iam-policy-binding commerce-lake \
  --location=us-central1 \
  --member="user:commerce-data-steward@company.com" \
  --role="roles/dataplex.admin"

# View current IAM policy on a zone
gcloud dataplex zones get-iam-policy curated-zone \
  --lake=commerce-lake \
  --location=us-central1
```

---

### Granular Security Architecture

```
org/
├── project: commerce-domain
│   └── Lake: commerce-lake  ← roles/dataplex.admin → commerce-steward@
│       ├── Zone: raw-zone   ← roles/dataplex.dataWriter → ingestion-sa@
│       └── Zone: curated    ← roles/dataplex.dataReader → analytics-team@
│
└── project: finance-domain
    └── Lake: finance-lake   ← roles/dataplex.admin → finance-steward@
        └── Zone: curated    ← roles/dataplex.dataReader → finance-analysts@
```

> 🔐 IAM at lake/zone level **inherits to all assets**. Granting `dataReader` at the curated zone level gives read access to all GCS buckets and BigQuery datasets in that zone automatically.

---

### Policy Tags (Column-Level Security)

```bash
# Create a policy tag taxonomy
gcloud data-catalog taxonomies create \
  --location=us-central1 \
  --display-name="PII Taxonomy" \
  --description="Policy tags for personally identifiable information"

# Create a policy tag
gcloud data-catalog taxonomies policy-tags create \
  --taxonomy=TAXONOMY_ID \
  --location=us-central1 \
  --display-name="Sensitive PII"

# Apply policy tag to BigQuery column (via BQ schema update)
# Schema JSON:
# { "name": "email", "type": "STRING",
#   "policyTags": {"names": ["POLICY_TAG_RESOURCE_NAME"]} }

# Grant fine-grained access to see masked values
gcloud projects add-iam-policy-binding my-project \
  --member="group:data-scientists@company.com" \
  --role="roles/bigquery.maskedDataViewer"
```

---

### CMEK Configuration

```bash
# Create KMS key for Dataplex
gcloud kms keyrings create dataplex-keyring --location=us-central1

gcloud kms keys create dataplex-key \
  --location=us-central1 \
  --keyring=dataplex-keyring \
  --purpose=encryption

# Grant Dataplex SA access to key
DATAPLEX_SA="service-PROJECT_NUMBER@gcp-sa-dataplex.iam.gserviceaccount.com"

gcloud kms keys add-iam-policy-binding dataplex-key \
  --location=us-central1 \
  --keyring=dataplex-keyring \
  --member="serviceAccount:${DATAPLEX_SA}" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Create lake with CMEK
gcloud dataplex lakes create secure-lake \
  --location=us-central1 \
  --display-name="CMEK-Encrypted Lake" \
  --metastore-service-account="${DATAPLEX_SA}"
```

---

## 9. Dataplex Tasks

### What are Dataplex Tasks?

Dataplex Tasks provide a **managed Spark/PySpark execution environment** using ephemeral Dataproc clusters. They run within the context of a lake, with automatic access to Dataplex-managed assets.

---

### gcloud: Tasks

```bash
# Create a PySpark task
gcloud dataplex tasks create data-transform-task \
  --lake=commerce-lake \
  --location=us-central1 \
  --type=SPARK \
  --trigger-type=ON_DEMAND \
  --display-name="Daily Data Transformation" \
  --description="Transform raw orders into curated fact table" \
  --execution-service-account=dataproc-sa@my-project.iam.gserviceaccount.com \
  --spark-python-script-file=gs://my-scripts/transform_orders.py \
  --spark-file-uris=gs://my-scripts/utils.py,gs://my-scripts/config.yaml \
  --vpc-network-tags=dataplex-worker \
  --execution-args="--date=${RUN_DATE}" \
  --container-image-python-interpreter-path=/usr/bin/python3

# Create a scheduled task (cron)
gcloud dataplex tasks create scheduled-transform \
  --lake=commerce-lake \
  --location=us-central1 \
  --type=SPARK \
  --trigger-type=RECURRING \
  --trigger-schedule="0 3 * * *" \
  --spark-python-script-file=gs://my-scripts/transform_orders.py \
  --display-name="Scheduled Daily Transform"

# Run a task on demand
gcloud dataplex tasks run data-transform-task \
  --lake=commerce-lake \
  --location=us-central1

# List task runs
gcloud dataplex tasks jobs list \
  --task=data-transform-task \
  --lake=commerce-lake \
  --location=us-central1 \
  --format="table(name,uid,state,startTime,endTime)"

# Cancel a running task
gcloud dataplex tasks jobs cancel JOB_ID \
  --task=data-transform-task \
  --lake=commerce-lake \
  --location=us-central1
```

---

### PySpark Task Example

```python
# transform_orders.py — Dataplex Task PySpark script
# Reads from raw zone (GCS), writes to curated zone (BigQuery)

from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from datetime import datetime, timedelta
import argparse
import sys

def main(run_date: str):
    spark = SparkSession.builder \
        .appName(f"OrdersTransform-{run_date}") \
        .config("spark.sql.adaptive.enabled", "true") \
        .getOrCreate()

    spark.sparkContext.setLogLevel("WARN")

    # Read from raw zone (GCS — auto-discovered by Dataplex)
    raw_orders = spark.read.parquet(
        f"gs://raw-orders-bucket/orders/dt={run_date}/"
    )

    # Transform
    curated = (raw_orders
        .filter(F.col("status").isin(["complete", "returned"]))
        .withColumn("order_date",   F.to_date("created_at"))
        .withColumn("revenue",      F.when(F.col("status") == "complete",
                                    F.col("amount")).otherwise(F.lit(0.0)))
        .withColumn("is_returned",  F.col("status") == "returned")
        .withColumn("processed_at", F.current_timestamp())
        .select("order_id", "user_id", "order_date", "amount",
                "revenue", "status", "is_returned", "region", "processed_at")
    )

    # Write to curated zone (BigQuery)
    curated.write \
        .format("bigquery") \
        .option("table", "my-project.orders.order_facts") \
        .option("temporaryGcsBucket", "my-temp-bucket") \
        .option("partitionField",     "order_date") \
        .mode("overwrite") \
        .save()

    print(f"Processed {curated.count()} records for {run_date}")
    spark.stop()

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--date", default=(datetime.today() - timedelta(days=1)).strftime("%Y-%m-%d"))
    args = parser.parse_args()
    main(args.date)
```

---

### Tasks vs. Dataproc Job vs. Cloud Composer

| Dimension | Dataplex Task | Dataproc Job | Cloud Composer |
|---|---|---|---|
| **Cluster mgmt** | Fully managed, ephemeral | Semi-managed | External (Dataproc operator) |
| **Scheduling** | Built-in (cron) | External | ✅ Airflow DAG |
| **Governance** | ✅ Lake-aware, lineage | ❌ | ❌ |
| **Complex DAGs** | ❌ Single-step only | ❌ | ✅ |
| **Notebook support** | ✅ | ❌ | Via operators |
| **Cost** | Dataproc + Dataplex | Dataproc only | Composer + Dataproc |
| **Best for** | Simple lake transforms, governed pipelines | Existing Spark code | Complex multi-step orchestration |

---

## 10. Dataplex & BigQuery Integration

### BigQuery ↔ Dataplex Connection

```bash
# Attach a BigQuery dataset to a curated zone
gcloud dataplex assets create analytics-bq-asset \
  --zone=curated-zone \
  --lake=commerce-lake \
  --location=us-central1 \
  --resource-type=BIGQUERY_DATASET \
  --resource-name=projects/my-project/datasets/analytics \
  --display-name="Analytics BigQuery Dataset" \
  --discovery-enabled

# Once attached:
# - All tables auto-registered as Catalog entries
# - Column-level lineage auto-captured from BQ jobs
# - Quality/profile scans can reference these tables
# - IAM at zone level propagates to BQ dataset access
```

---

### Data Quality on BigQuery (Python + API)

```python
# Create incremental quality scan on BQ table (only scan new data)
incremental_scan = dataplex_v1.DataScan(
    display_name = "Orders Incremental Quality",
    data = dataplex_v1.DataSource(
        resource = "//bigquery.googleapis.com/projects/my-project/datasets/orders/tables/order_facts",
        # Use entity reference for Dataplex-managed assets:
        # entity = "projects/my-project/locations/us-central1/lakes/commerce-lake/zones/curated-zone/entities/order_facts"
    ),
    data_quality_spec = dataplex_v1.DataQualitySpec(
        sampling_percent = 100,
        # Incremental: only scan rows added since last run
        row_filter = "order_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)",
        rules = [
            dataplex_v1.DataQualityRule(
                column = "order_id",
                name   = "not_null",
                dimension = "COMPLETENESS",
                non_null_expectation = dataplex_v1.DataQualityRule.NonNullExpectation(),
                threshold = 1.0,
            )
        ],
    ),
)
```

---

### Export Quality Results to BigQuery

```sql
-- Query the DQ results BigQuery export table
-- (auto-created when postScanActions.bigqueryExport is set)
SELECT
  job_id,
  data_scan_id,
  start_time,
  end_time,
  data_quality_result.passed          AS overall_passed,
  data_quality_result.score           AS quality_score,
  r.name                              AS rule_name,
  r.dimension                         AS dimension,
  r.column                            AS column_name,
  r.passed                            AS rule_passed,
  r.pass_ratio                        AS pass_ratio,
  r.evaluated_count                   AS rows_evaluated,
  r.passed_count                      AS rows_passed,
  r.failed_count                      AS rows_failed
FROM
  `my-project.dq_results.orders_quality`,
  UNNEST(data_quality_result.rules) AS r
WHERE
  DATE(start_time) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
ORDER BY
  start_time DESC, rule_name;
```

---

## 11. Data Mesh Implementation on GCP

### Data Mesh → Dataplex Mapping

| Data Mesh Principle | GCP / Dataplex Implementation |
|---|---|
| **Domain ownership** | One Lake per domain; domain team owns lake IAM |
| **Data as a product** | Curated zones = data products with quality SLAs |
| **Self-serve platform** | Catalog search, data quality, tasks — available to all |
| **Federated governance** | Org-level policies (policy tags, DLP) + domain IAM |

---

### Multi-Domain Data Mesh Reference Architecture

```
GCP Organization
│
├── Project: platform-project (central platform team)
│   ├── Dataplex Catalog (org-wide search + glossary)
│   ├── Data Lineage (org-wide lineage graph)
│   ├── Policy Tag Taxonomies (PII, confidential, etc.)
│   └── Shared Infrastructure (Cloud Composer, Monitoring)
│
├── Project: commerce-domain
│   └── Lake: commerce-lake
│       ├── Zone: raw         ← ingestion-sa@ (dataWriter)
│       │   ├── Asset: raw-events    (GCS)
│       │   └── Asset: raw-orders    (GCS)
│       └── Zone: curated     ← analytics@ (dataReader)
│           ├── Asset: order_facts   (BigQuery)
│           └── Asset: customer_360  (BigQuery)
│               ↑ Data Quality Scans (daily)
│               ↑ Data Profile Scans (weekly)
│
├── Project: finance-domain
│   └── Lake: finance-lake
│       ├── Zone: raw         ← finance-ingest-sa@
│       └── Zone: curated     ← finance-analysts@
│           └── Asset: transactions  (BigQuery)
│
└── Project: marketing-domain
    └── Lake: marketing-lake
        ├── Zone: raw
        └── Zone: curated
            └── Asset: campaign_metrics (BigQuery)

Cross-domain discovery:
  All catalog entries searchable via Universal Catalog
  Lineage graph connects commerce → finance → reporting
  Glossary terms shared across all domains
```

---

### Data Product Pattern (Curated Zone as Product)

```bash
# 1. Create product zone with strict quality gates
gcloud dataplex zones create orders-product-zone \
  --lake=commerce-lake \
  --location=us-central1 \
  --type=CURATED \
  --display-name="Orders Data Product v2"

# 2. Attach the BigQuery dataset (the product)
gcloud dataplex assets create order-facts-product \
  --zone=orders-product-zone \
  --lake=commerce-lake \
  --location=us-central1 \
  --resource-type=BIGQUERY_DATASET \
  --resource-name=projects/my-project/datasets/orders_product

# 3. Create a quality scan = product quality SLA
gcloud dataplex datascans create orders-product-quality \
  --location=us-central1 \
  --type=DATA_QUALITY \
  --data-source-resource=//bigquery.googleapis.com/projects/my-project/datasets/orders_product/tables/order_facts \
  --schedule="0 7 * * *"

# 4. Add product metadata via aspect
gcloud dataplex entries update orders-product-entry \
  --location=us-central1 \
  --aspects='{"sla_hours": 4, "owner": "commerce-team", "version": "2.0"}'

# 5. Grant consumers read access
gcloud dataplex zones add-iam-policy-binding orders-product-zone \
  --lake=commerce-lake \
  --location=us-central1 \
  --member="group:data-consumers@company.com" \
  --role="roles/dataplex.dataReader"
```

---

## 12. gcloud CLI & REST API Quick Reference

### All gcloud dataplex Commands

```bash
# ── LAKES ─────────────────────────────────────────────────────────────
gcloud dataplex lakes create NAME    --location=LOC [--type --display-name --labels]
gcloud dataplex lakes list           --location=LOC
gcloud dataplex lakes describe NAME  --location=LOC
gcloud dataplex lakes update NAME    --location=LOC [--display-name --labels]
gcloud dataplex lakes delete NAME    --location=LOC --quiet
gcloud dataplex lakes add-iam-policy-binding NAME --location=LOC --member=M --role=R
gcloud dataplex lakes get-iam-policy NAME --location=LOC

# ── ZONES ─────────────────────────────────────────────────────────────
gcloud dataplex zones create NAME    --lake=L --location=LOC --type=RAW|CURATED
gcloud dataplex zones list           --lake=L --location=LOC
gcloud dataplex zones describe NAME  --lake=L --location=LOC
gcloud dataplex zones update NAME    --lake=L --location=LOC [--discovery-enabled --discovery-schedule]
gcloud dataplex zones delete NAME    --lake=L --location=LOC --quiet
gcloud dataplex zones add-iam-policy-binding NAME --lake=L --location=LOC --member=M --role=R

# ── ASSETS ────────────────────────────────────────────────────────────
gcloud dataplex assets create NAME   --zone=Z --lake=L --location=LOC --resource-type=TYPE --resource-name=RES
gcloud dataplex assets list          --zone=Z --lake=L --location=LOC
gcloud dataplex assets describe NAME --zone=Z --lake=L --location=LOC
gcloud dataplex assets delete NAME   --zone=Z --lake=L --location=LOC

# ── TASKS ─────────────────────────────────────────────────────────────
gcloud dataplex tasks create NAME    --lake=L --location=LOC --type=SPARK --trigger-type=ON_DEMAND|RECURRING
gcloud dataplex tasks list           --lake=L --location=LOC
gcloud dataplex tasks describe NAME  --lake=L --location=LOC
gcloud dataplex tasks run NAME       --lake=L --location=LOC
gcloud dataplex tasks delete NAME    --lake=L --location=LOC
gcloud dataplex tasks jobs list      --task=T --lake=L --location=LOC
gcloud dataplex tasks jobs cancel JOB --task=T --lake=L --location=LOC

# ── DATA SCANS ────────────────────────────────────────────────────────
gcloud dataplex datascans create NAME  --location=LOC --type=DATA_QUALITY|DATA_PROFILE
gcloud dataplex datascans list         --location=LOC
gcloud dataplex datascans describe NAME --location=LOC
gcloud dataplex datascans run NAME     --location=LOC
gcloud dataplex datascans delete NAME  --location=LOC
gcloud dataplex datascans jobs list    --datascan=S --location=LOC
gcloud dataplex datascans jobs describe JOB --datascan=S --location=LOC --view=FULL

# ── ENTITIES ──────────────────────────────────────────────────────────
gcloud dataplex entities list          --zone=Z --lake=L --location=LOC
gcloud dataplex entities describe NAME --zone=Z --lake=L --location=LOC

# ── CATALOG ───────────────────────────────────────────────────────────
gcloud dataplex entry-groups create NAME  --location=LOC
gcloud dataplex entry-groups list         --location=LOC
gcloud dataplex aspect-types create NAME  --location=LOC --metadata-template=JSON
gcloud dataplex entries search            --location=LOC --query=Q
gcloud dataplex glossaries create NAME    --location=LOC
gcloud dataplex glossaries terms create NAME --glossary=G --location=LOC

# ── LINEAGE ───────────────────────────────────────────────────────────
# Lineage uses the separate Data Lineage API (not dataplex CLI)
# Use: gcloud beta data-lineage ... or Python/REST client
```

---

### REST API Reference

```bash
BASE="https://dataplex.googleapis.com/v1"
TOKEN=$(gcloud auth print-access-token)
PROJECT="my-project"
LOCATION="us-central1"

# Lakes
curl "${BASE}/projects/${PROJECT}/locations/${LOCATION}/lakes"                         -H "Authorization: Bearer ${TOKEN}"  # List
curl -X POST "${BASE}/projects/${PROJECT}/locations/${LOCATION}/lakes?lakeId=my-lake" -H "Authorization: Bearer ${TOKEN}" -H "Content-Type: application/json" -d '{...}'  # Create

# Zones
curl "${BASE}/projects/${PROJECT}/locations/${LOCATION}/lakes/my-lake/zones"           -H "Authorization: Bearer ${TOKEN}"  # List

# Data Scans
curl "${BASE}/projects/${PROJECT}/locations/${LOCATION}/dataScans"                     -H "Authorization: Bearer ${TOKEN}"  # List
curl -X POST "${BASE}/projects/${PROJECT}/locations/${LOCATION}/dataScans/SCAN:run"    -H "Authorization: Bearer ${TOKEN}"  # Run
curl "${BASE}/projects/${PROJECT}/locations/${LOCATION}/dataScans/SCAN/jobs"           -H "Authorization: Bearer ${TOKEN}"  # Job history

# Catalog search
curl "https://dataplex.googleapis.com/v1/projects/${PROJECT}/locations/${LOCATION}:searchEntries?query=orders&orderBy=relevance" \
  -H "Authorization: Bearer ${TOKEN}"

# Lineage
BASE_LIN="https://datalineage.googleapis.com/v1/projects/${PROJECT}/locations/us"
curl "${BASE_LIN}/processes"                              -H "Authorization: Bearer ${TOKEN}"   # List processes
curl -X POST "${BASE_LIN}/processes"                      -H "Authorization: Bearer ${TOKEN}" -H "Content-Type: application/json" -d '{"displayName":"My ETL"}'  # Create process
```

---

### Python Client Cheatsheet

```python
from google.cloud import dataplex_v1
from google.cloud import datacatalog_lineage_v1

# Clients
lake_client    = dataplex_v1.DataplexServiceClient()
scan_client    = dataplex_v1.DataScanServiceClient()
catalog_client = dataplex_v1.CatalogServiceClient()
meta_client    = dataplex_v1.MetadataServiceClient()
lineage_client = datacatalog_lineage_v1.LineageClient()

# Key methods by client
# lake_client:
#   .create_lake()  .get_lake()  .list_lakes()  .delete_lake()
#   .create_zone()  .get_zone()  .list_zones()
#   .create_asset() .get_asset() .list_assets()
#   .create_task()  .run_task()  .list_tasks()

# scan_client:
#   .create_data_scan()  .run_data_scan()  .list_data_scans()
#   .get_data_scan_job() .list_data_scan_jobs()

# catalog_client:
#   .create_entry_group() .create_entry() .get_entry()
#   .create_aspect_type() .search_entries()
#   .create_glossary()    .create_glossary_term()

# meta_client:
#   .list_entities() .get_entity()
#   .list_partitions() .get_partition()

# lineage_client:
#   .create_process()  .create_run()  .create_lineage_event()
#   .search_links()    .get_process()
```

---

## 13. Monitoring, Observability & Alerting

### Key Metrics

| Metric | Description | Alert When |
|---|---|---|
| `dataplex.googleapis.com/datascan/job_status` | Scan job result (1=success, 0=fail) | = 0 (failure) |
| `dataplex.googleapis.com/task/job_status` | Task run status | FAILED state |
| `dataplex.googleapis.com/lake/discovery/runs` | Discovery scan executions | Unexpected drop |
| `dataplex.googleapis.com/datascan/quality_score` | Overall DQ score (0–1) | < threshold (e.g., 0.95) |

---

### Setting Up Quality Failure Alerts

```bash
# Create a Cloud Monitoring alert for DQ scan failure
gcloud alpha monitoring policies create \
  --display-name="Dataplex DQ Scan Failure" \
  --condition-filter='
    metric.type="dataplex.googleapis.com/datascan/job_status"
    metric.labels.scan_id="orders-quality-scan"
    metric.labels.state="FAILED"' \
  --condition-threshold-value=0 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=0s \
  --notification-channels=CHANNEL_ID \
  --project=my-project

# Alert on quality score dropping below 95%
gcloud alpha monitoring policies create \
  --display-name="Dataplex DQ Score Below Threshold" \
  --condition-filter='
    metric.type="dataplex.googleapis.com/datascan/quality_score"
    metric.labels.scan_id="orders-quality-scan"' \
  --condition-threshold-value=0.95 \
  --condition-threshold-comparison=COMPARISON_LT \
  --condition-duration=0s \
  --notification-channels=CHANNEL_ID
```

---

### Cloud Logging Queries

```bash
# Query Dataplex admin activity logs
gcloud logging read \
  'resource.type="dataplex.googleapis.com/Lake"
   protoPayload.methodName="google.cloud.dataplex.v1.DataplexService.CreateLake"' \
  --project=my-project --limit=50

# Query data scan run results
gcloud logging read \
  'resource.type="dataplex.googleapis.com/DataScan"
   jsonPayload.state="FAILED"' \
  --project=my-project --limit=100

# Query discovery events
gcloud logging read \
  'resource.type="dataplex.googleapis.com/Zone"
   logName:"dataplex.googleapis.com/discovery"' \
  --project=my-project --limit=50

# Query task failures
gcloud logging read \
  'resource.type="dataplex.googleapis.com/Task"
   jsonPayload.status="FAILED"
   timestamp>="2026-03-15T00:00:00Z"' \
  --project=my-project
```

---

### Python: Query DQ Results from BigQuery

```python
from google.cloud import bigquery

bq = bigquery.Client(project="my-project")

query = """
SELECT
  DATE(start_time)                    AS scan_date,
  data_scan_id,
  data_quality_result.passed          AS passed,
  data_quality_result.score           AS score,
  COUNTIF(NOT r.passed)               AS failed_rules,
  COUNT(r.name)                       AS total_rules
FROM
  `my-project.dq_results.orders_quality`,
  UNNEST(data_quality_result.rules) AS r
WHERE
  DATE(start_time) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY 1, 2, 3, 4
ORDER BY scan_date DESC
"""

results = bq.query(query).result()
for row in results:
    status = "✅ PASS" if row.passed else "❌ FAIL"
    print(f"{row.scan_date} | {status} | Score: {row.score:.1%} | "
          f"Failed rules: {row.failed_rules}/{row.total_rules}")
```

---

## 14. Pricing Summary

> ⚠️ Prices are approximate as of early 2026. Verify at [cloud.google.com/dataplex/pricing](https://cloud.google.com/dataplex/pricing).

### Dataplex Pricing Components

| Component | Unit | Approx Price | Notes |
|---|---|---|---|
| **Lake** | Per lake per month | ~$0 (metadata mgmt included) | No direct charge |
| **Data Quality Scan** | Per GB scanned (BQ) | BQ on-demand slot cost | Uses BigQuery under the hood |
| **Data Profile Scan** | Per GB scanned (BQ) | BQ on-demand slot cost | Sampling reduces cost significantly |
| **Dataplex Tasks** | Per vCPU-hour (Dataproc) | ~$0.041/vCPU-hr + Dataproc premium | Ephemeral cluster per run |
| **Universal Catalog** | Metadata API calls | ~$0.10 per 10K calls | Search, read, write operations |
| **Data Lineage** | Lineage events stored | ~$0.05 per 1K events/month | Plus API call costs |
| **Discovery** | Included | Free | Runs as part of zone management |

---

### Free Tier

| Operation | Free |
|---|---|
| Lake/zone/asset CRUD | ✅ Always free |
| Metadata discovery | ✅ Always free |
| Catalog search (first N) | ✅ First 10K API calls/month |
| Lineage auto-capture (BQ, Dataflow) | ✅ Always free |
| Catalog entry read/write (first N) | ✅ First 10K API calls/month |

---

### 💰 Cost Optimization Tips

| Tip | Savings | How |
|---|---|---|
| Use `samplingPercent` in scans | 60–90% on DQ/profile | Set to 10–30% for large tables |
| Incremental scans with `rowFilter` | 70–95% vs. full scan | Filter to only new/changed rows |
| Schedule scans off-peak | Indirect | Avoid BQ slot contention |
| Right-size Task clusters | 20–50% | Use autoscaling; add Spot workers |
| Minimize lineage API calls | Direct | Batch lineage events; cache process/run IDs |
| Reuse processes/runs for lineage | Direct | One process per pipeline, not per run |

---

### Monthly Cost Estimate

```
Scenario: 3-domain data mesh, daily quality scans on 5 BQ tables (avg 10 GB each)

Data Quality Scans:
  5 tables × 10 GB × $6.25/TB × 22 days/month = $6.88/month
  (with 30% sampling: $6.88 × 0.30 = $2.06/month)

Data Profile Scans (weekly):
  5 tables × 10 GB × $6.25/TB × 4 weeks = $1.25/month

Dataplex Tasks (daily transform, 4 workers × n2-standard-4, 30 min):
  4 workers × 4 vCPU × $0.041 × 0.5h × 22 days = $7.26/month
  + Dataproc premium: 4 × 4 × $0.010 × 11h = $1.76/month

Universal Catalog (100K API calls/month):
  100K × $0.10/10K = $1.00/month

Data Lineage (5K events/month):
  5K × $0.05/1K = $0.25/month
──────────────────────────────────────────────────────────────
Total with sampling:                              ~$14/month
Total without sampling (full scans):              ~$25/month

BigQuery storage and query costs are SEPARATE from Dataplex costs.
```

---

## 15. Quick Reference & Comparison Tables

### Dataplex Resource Hierarchy

```
GCP Organization
└── GCP Project
    └── Dataplex Lake (domain)
        ├── Zone: Raw (type=RAW)
        │   └── Asset → GCS Bucket
        │       └── Entity (discovered table/fileset)
        │           └── Partitions
        └── Zone: Curated (type=CURATED)
            └── Asset → BigQuery Dataset
                └── Entity (discovered BQ table)
                    └── Schema Fields

Cross-cutting:
├── Universal Catalog Entries (one per resource)
├── Glossary Terms (linked to entries/columns)
├── Data Quality Scans (per table/asset)
├── Data Profile Scans (per table/asset)
├── Data Lineage (processes → runs → events)
└── Tasks (Spark jobs within a lake)
```

---

### Zone Type Comparison

| Feature | Raw Zone | Curated Zone |
|---|---|---|
| **Purpose** | Data ingestion / landing | Governed data products |
| **Data formats** | Any | Structured only (Parquet, ORC, Avro, BQ) |
| **Schema enforcement** | ❌ | ✅ On write |
| **IAM default** | Restrictive (writers only) | Accessible to consumers |
| **Quality scans** | Optional | ✅ Recommended (SLA enforcement) |
| **Discovery** | ✅ Schema inference | ✅ BQ metadata sync |
| **Lineage** | Auto-captured if BQ/Dataflow | Auto-captured |
| **Use case** | Kafka dumps, raw GCS files | Analytics, BI, data products |

---

### Data Quality Rule Types

| Rule | Dimension | SQL Equivalent | Example Config |
|---|---|---|---|
| `NON_NULL_CHECK` | Completeness | `col IS NOT NULL` | `"nonNullExpectation": {}` |
| `RANGE_CHECK` | Validity | `col BETWEEN min AND max` | `"rangeExpectation": {"minValue":"0","maxValue":"100"}` |
| `SET_CHECK` | Validity | `col IN (vals)` | `"setExpectation": {"values":["a","b"]}` |
| `REGEX_CHECK` | Validity | `REGEXP_CONTAINS(col, pattern)` | `"regexExpectation": {"regex":"^\\d{5}$"}` |
| `UNIQUENESS_CHECK` | Uniqueness | `COUNT(DISTINCT col) = COUNT(*)` | `"uniquenessExpectation": {}` |
| `ROW_CONDITION` | Validity | `WHERE NOT (condition)` | `"rowConditionExpectation": {"sqlExpression":"a > b"}` |
| `SQL_ASSERTION` | Custom | Arbitrary SQL → row count | `"sqlAssertion": {"sqlStatement":"SELECT COUNT(*)..."}` |
| `STATISTIC_RANGE_CHECK` | Accuracy | `AVG/MIN/MAX/COUNT BETWEEN range` | `"statisticRangeExpectation": {"statistic":"MEAN","column":"c","minValue":"0"}` |

---

### IAM Roles Summary

| Role | Scope | Can Create | Can Read Data | Can Write Data | Admin |
|---|---|---|---|---|---|
| `dataplex.admin` | Project | ✅ All | ✅ | ✅ | ✅ |
| `dataplex.editor` | Project | ✅ Most | ✅ | ✅ | ❌ |
| `dataplex.viewer` | Project | ❌ | ✅ Metadata | ❌ | ❌ |
| `dataplex.dataReader` | Lake/Zone | ❌ | ✅ Data | ❌ | ❌ |
| `dataplex.dataWriter` | Lake/Zone | ❌ | ✅ | ✅ Data | ❌ |
| `dataplex.metadataReader` | Project | ❌ | ✅ Catalog | ❌ | ❌ |
| `dataplex.dataOwner` | Lake/Zone | ❌ | ✅ | ✅ | ✅ Zone |

---

### Dataplex vs. Legacy Cloud Data Catalog Migration

| Legacy Concept | Dataplex Equivalent | Migration Action |
|---|---|---|
| Tag Template | Aspect Type | Recreate as Aspect Type in Dataplex |
| Tag | Aspect | Migrate tag data to Aspects |
| Entry | Entry | Entries migrate automatically |
| Entry Group | Entry Group | Entry groups migrate automatically |
| Policy Tag | Policy Tag (unchanged) | No migration needed |
| Taxonomy | Taxonomy (unchanged) | No migration needed |
| Data Catalog API | Dataplex Catalog API | Update API endpoints |
| `gcloud data-catalog` | `gcloud dataplex` | Update CLI commands |

---

### Automatic Lineage Sources

| Service | Lineage Type | Granularity | Enablement |
|---|---|---|---|
| **BigQuery** | Job-based | Column-level | Auto (enabled by default) |
| **Dataflow** | Pipeline-based | Dataset-level | Auto (enabled by default) |
| **Cloud Data Fusion** | Pipeline-based | Dataset-level | Auto (Enterprise edition) |
| **Vertex AI Pipelines** | Pipeline-based | Dataset/model | Auto |
| **Vertex AI Datasets** | Data creation | Dataset-level | Auto |
| **Custom (API)** | Manual | Any granularity | Via Lineage API |
| **dbt (via plugin)** | Model-based | Column-level | Via community plugin |

---

### Common Anti-Patterns & Fixes

| ❌ Anti-Pattern | 🔴 Problem | ✅ Fix |
|---|---|---|
| Raw zone with no discovery config | No entities registered; no lineage | Enable discovery with include patterns |
| No quality scans on curated zones | Unknown data quality; silent failures | Create scheduled DQ scans on all curated assets |
| Millions of tiny GCS files | Discovery takes hours; poor performance | Compact files to 100MB–1GB before discovery |
| Missing Hive partition keys | Partition detection fails | Use `key=value/` folder structure consistently |
| One lake for everything | No domain isolation; governance chaos | One lake per domain; separate projects if possible |
| Full-table scans in DQ scans | High BQ cost on large tables | Use `samplingPercent` + `rowFilter` for incremental |
| Skipping glossary terms | Inconsistent metric definitions across teams | Define terms in glossary; link to BQ columns |
| No lineage for custom ETL | Gaps in lineage graph | Use Lineage API for non-GCP pipelines |
| Project-level IAM only | Over-permissive access | Use zone/lake-level IAM for granular control |
| Tasks without error handling | Silent failures, partial data | Add try/except + Cloud Logging in PySpark scripts |

---

### Task vs. Dataproc Job vs. Cloud Composer Decision Guide

| Requirement | Dataplex Task | Dataproc Job | Cloud Composer |
|---|---|---|---|
| Simple lake transform (single step) | ✅ Best | ✅ | Overkill |
| Complex multi-step pipeline | ❌ | ❌ | ✅ Best |
| Built-in scheduling | ✅ | ❌ External | ✅ Airflow |
| Lake-aware execution + lineage | ✅ | ❌ Manual | ❌ Manual |
| Notebook execution | ✅ | ❌ | Via operators |
| Existing Spark codebase | ✅ | ✅ | Via operators |
| Cross-system orchestration | ❌ | ❌ | ✅ Best |
| Lowest operational overhead | ✅ | ✅ | ⚠️ Composer cluster |
| Retry logic + branching | ❌ | ❌ | ✅ |

---

*Generated for GCP Dataplex | Official Docs: [cloud.google.com/dataplex/docs](https://cloud.google.com/dataplex/docs) | Universal Catalog: [cloud.google.com/dataplex/docs/catalog-overview](https://cloud.google.com/dataplex/docs/catalog-overview)*
