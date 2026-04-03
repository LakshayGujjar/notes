# 🔄 GCP Cloud Data Fusion — Comprehensive Cheatsheet

> **Audience:** Data engineers, ETL developers, platform engineers, and architects building data pipelines on GCP.
> **Last updated:** March 2026 | Covers Pipeline Studio, Wrangler, Replication, REST API, and all core features.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Instance Setup & Configuration](#2-instance-setup--configuration)
3. [Pipeline Studio & Pipeline Design](#3-pipeline-studio--pipeline-design)
4. [Sources, Sinks & Connectors](#4-sources-sinks--connectors)
5. [Transformations & Data Processing](#5-transformations--data-processing)
6. [Wrangler (Data Preparation)](#6-wrangler-data-preparation)
7. [Replication (CDC)](#7-replication-cdc)
8. [Scheduling & Orchestration](#8-scheduling--orchestration)
9. [Metadata, Lineage & Data Catalog](#9-metadata-lineage--data-catalog)
10. [IAM, Security & Networking](#10-iam-security--networking)
11. [Compute Profiles & Performance Tuning](#11-compute-profiles--performance-tuning)
12. [REST API & Automation](#12-rest-api--automation)
13. [Monitoring, Logging & Debugging](#13-monitoring-logging--debugging)
14. [gcloud CLI & REST API Quick Reference](#14-gcloud-cli--rest-api-quick-reference)
15. [Pricing Summary](#15-pricing-summary)
16. [Quick Reference & Comparison Tables](#16-quick-reference--comparison-tables)

---

## 1. Overview & Key Concepts

### What is Cloud Data Fusion?

**Google Cloud Data Fusion** is a fully managed, cloud-native data integration service built on the open-source **CDAP (Cask Data Application Platform)**. It provides a visual, drag-and-drop pipeline builder for building ETL/ELT pipelines — no distributed systems code required. Pipelines are compiled to Apache Spark jobs that run on ephemeral Dataproc clusters.

**Core value proposition:**

| Pillar | Description |
|---|---|
| **Code-free ETL/ELT** | Visual drag-and-drop pipeline builder for all skill levels |
| **Open-source foundation** | Built on Apache CDAP — portable, extensible |
| **Governed integration** | Built-in lineage, metadata, and data quality |
| **Extensible** | 150+ pre-built connectors; custom plugin framework |
| **Managed execution** | Auto-provisions Dataproc clusters per run; zero infra management |

---

### Cloud Data Fusion vs. Alternatives

| Dimension | Cloud Data Fusion | Dataflow (Beam) | Dataproc (Spark) | Data Transfer Service |
|---|---|---|---|---|
| **Interface** | Visual GUI + code | Code only (Java/Python) | Code only (PySpark/Scala) | Console / CLI |
| **Skill required** | Low (GUI-first) | High (Beam SDK) | High (Spark) | Low |
| **Streaming** | ✅ (limited) | ✅ Native, advanced | ✅ Spark Streaming | ❌ |
| **Custom logic** | ✅ Plugins + Python/JS | ✅ Full Beam | ✅ Full Spark | ❌ |
| **CDC / Replication** | ✅ Built-in | ❌ | ❌ | ❌ |
| **Data lineage** | ✅ Built-in | ❌ | ❌ | ❌ |
| **Cost model** | Instance + Dataproc | vCPU-hr | Cluster-hr | Per TB |
| **Best for** | ETL teams, mixed-skill orgs, governed pipelines | Advanced streaming, custom transforms | Existing Spark code, ML pipelines | Simple BQ/GCS transfers |

> 💡 **Choose Cloud Data Fusion when:** your team includes non-engineers, you need built-in lineage/governance, or you want reusable visual pipelines. Choose **Dataflow** for complex custom streaming. Choose **Dataproc** for existing Spark workloads.

---

### Core Architecture

```
Cloud Data Fusion Instance
├── CDAP Engine (pipeline runtime + metadata)
│   ├── Namespaces (logical isolation)
│   │   ├── Pipelines (batch + streaming)
│   │   ├── Connections (reusable data source configs)
│   │   ├── Datasets (logical data references)
│   │   └── Artifacts (plugin JARs)
│   └── Metadata Store (lineage, tags, annotations)
├── Pipeline Studio (visual designer UI)
├── Wrangler (interactive data prep)
├── Hub (plugin marketplace)
├── Replication (CDC engine)
└── REST API (all operations programmatic)

Execution Flow:
  Pipeline Design (GUI) → Compiled to Spark DAG
       → Submitted to ephemeral Dataproc cluster
       → Results written to sink (BigQuery, GCS, etc.)
       → Cluster auto-deleted after run
```

---

### Editions Comparison

| Feature | Developer | Basic | Enterprise |
|---|---|---|---|
| **SLA** | No SLA | 99.9% | 99.9% |
| **Namespaces** | 1 | 1 | ✅ Multiple |
| **Replication (CDC)** | ❌ | ❌ | ✅ |
| **Private IP** | ❌ | ✅ | ✅ |
| **Streaming pipelines** | ❌ | ✅ | ✅ |
| **CMEK** | ❌ | ✅ | ✅ |
| **Support** | Community | Business | Premium |
| **Use case** | Dev / testing | Production ETL | Enterprise governance |

> ⚠️ **Developer edition** has no SLA, no private IP, and no streaming. Use it exclusively for development and testing. **Never use Developer edition for production workloads.**

---

### Pipeline Types

| Type | Execution Engine | Trigger | Data Model | Use Case |
|---|---|---|---|---|
| **Batch** | Apache Spark | On-demand / scheduled | Bounded (files, tables) | ETL, data warehousing, backfill |
| **Real-time** | Spark Streaming | Continuous | Unbounded (streams) | Pub/Sub ingestion, CDC, event processing |

---

### Execution Model

```
Visual Pipeline (JSON spec)
        │
        ▼
CDAP Runtime compiles to Spark DAG
        │
        ▼
Dataproc cluster provisioned (ephemeral, per-run)
   ├── Master node: YARN ResourceManager + Spark driver
   └── Worker nodes: Spark executors
        │
        ▼
Spark job executes → Stages run in parallel
        │
        ▼
Results written to sink → Cluster auto-deleted
```

> 💡 Each pipeline run creates a **fresh ephemeral Dataproc cluster** (unless using a pre-provisioned static cluster). Startup time is ~2–3 minutes. Account for this in latency-sensitive workflows.

---

### Key Limits & Quotas

| Resource | Limit | Notes |
|---|---|---|
| Max pipeline stages | 100 | Per pipeline |
| Max concurrent pipeline runs | 50 (Enterprise) / 20 (Basic) | Per instance |
| Max pipelines per namespace | 1,000 | |
| Max namespaces | Varies by edition | Enterprise: multiple |
| Instance per project per region | 1 | Request increase if needed |
| Replication jobs | Enterprise only | Up to 10 concurrent |

---

## 2. Instance Setup & Configuration

### Creating an Instance

```bash
# Create a Basic edition instance (public IP)
gcloud data-fusion instances create my-cdf-instance \
  --location=us-central1 \
  --type=BASIC \
  --version=6.9.2

# Create an Enterprise edition instance with private IP
gcloud data-fusion instances create my-enterprise-instance \
  --location=us-central1 \
  --type=ENTERPRISE \
  --version=6.9.2 \
  --enable-private-ip \
  --no-enable-public-ip \
  --network=projects/MY_PROJECT/global/networks/my-vpc

# Describe an instance (get service account, API endpoint)
gcloud data-fusion instances describe my-cdf-instance \
  --location=us-central1 \
  --format="json(serviceAccount,apiEndpoint,state,type)"

# List all instances
gcloud data-fusion instances list --location=us-central1

# Delete an instance
gcloud data-fusion instances delete my-cdf-instance \
  --location=us-central1 --quiet
```

---

### Service Account Configuration

```bash
# Get the Cloud Data Fusion service account (auto-created per instance)
CDF_SA=$(gcloud data-fusion instances describe my-cdf-instance \
  --location=us-central1 \
  --format="value(serviceAccount)")

echo "CDF Service Account: $CDF_SA"
# e.g., service-PROJECT_NUMBER@gcp-sa-datafusion.iam.gserviceaccount.com

# Grant CDF SA the roles it needs to run Dataproc jobs
gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="serviceAccount:${CDF_SA}" \
  --role="roles/dataproc.admin"

gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="serviceAccount:${CDF_SA}" \
  --role="roles/bigquery.dataEditor"

gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="serviceAccount:${CDF_SA}" \
  --role="roles/storage.objectAdmin"

# Dataproc workers use the Compute Engine default SA or a custom SA
# Grant it Storage and BigQuery access too
DATAPROC_SA="PROJECT_NUMBER-compute@developer.gserviceaccount.com"

gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="serviceAccount:${DATAPROC_SA}" \
  --role="roles/bigquery.dataEditor"

gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="serviceAccount:${DATAPROC_SA}" \
  --role="roles/storage.objectAdmin"
```

---

### Private IP Configuration

```bash
# 1. Enable Private Services Access on your VPC
gcloud compute addresses create google-managed-services-my-vpc \
  --global \
  --purpose=VPC_PEERING \
  --prefix-length=22 \
  --network=my-vpc

gcloud services vpc-peerings connect \
  --service=servicenetworking.googleapis.com \
  --ranges=google-managed-services-my-vpc \
  --network=my-vpc

# 2. Create firewall rule to allow CDF → Dataproc communication
gcloud compute firewall-rules create allow-cdf-to-dataproc \
  --network=my-vpc \
  --allow=tcp:0-65535,udp:0-65535,icmp \
  --source-ranges=10.0.0.0/8 \
  --target-tags=dataproc

# 3. Create the private IP CDF instance
gcloud data-fusion instances create my-private-cdf \
  --location=us-central1 \
  --type=ENTERPRISE \
  --enable-private-ip \
  --no-enable-public-ip \
  --network=projects/MY_PROJECT/global/networks/my-vpc
```

---

### Terraform: Provisioning Cloud Data Fusion

```yaml
# main.tf
resource "google_data_fusion_instance" "cdf" {
  name                          = "my-cdf-instance"
  region                        = "us-central1"
  type                          = "BASIC"
  display_name                  = "My Data Fusion Instance"
  enable_stackdriver_logging    = true
  enable_stackdriver_monitoring = true

  network_config {
    network       = "projects/${var.project}/global/networks/${var.network}"
    ip_allocation = "10.89.48.0/22"    # Must not overlap existing ranges
  }

  labels = {
    env  = "prod"
    team = "data-eng"
  }
}

# Grant CDF SA necessary roles
resource "google_project_iam_member" "cdf_dataproc_admin" {
  project = var.project
  role    = "roles/dataproc.admin"
  member  = "serviceAccount:${google_data_fusion_instance.cdf.service_account}"
}

resource "google_project_iam_member" "cdf_bq_editor" {
  project = var.project
  role    = "roles/bigquery.dataEditor"
  member  = "serviceAccount:${google_data_fusion_instance.cdf.service_account}"
}

output "cdf_api_endpoint" {
  value = google_data_fusion_instance.cdf.service_endpoint
}

output "cdf_service_account" {
  value = google_data_fusion_instance.cdf.service_account
}
```

---

### Compute Profile Configuration (UI → JSON)

```json
{
  "name": "custom-dataproc-profile",
  "label": "Large Cluster Profile",
  "description": "High-memory Dataproc cluster for large ETL jobs",
  "provisioner": {
    "name": "gcp-dataproc",
    "properties": [
      {"name": "projectId",          "value": "my-project"},
      {"name": "region",             "value": "us-central1"},
      {"name": "network",            "value": "my-vpc"},
      {"name": "masterMachineType",  "value": "n2-standard-4"},
      {"name": "workerMachineType",  "value": "n2-highmem-8"},
      {"name": "numberOfWorkers",    "value": "4"},
      {"name": "workerBootDiskSize", "value": "200"},
      {"name": "imageVersion",       "value": "2.1"},
      {"name": "secondaryWorkerCount","value": "4"},
      {"name": "secondaryWorkerType","value": "SPOT"},
      {"name": "serviceAccount",     "value": "dataproc-sa@my-project.iam.gserviceaccount.com"}
    ]
  }
}
```

---

## 3. Pipeline Studio & Pipeline Design

### Pipeline Anatomy

```
Source Stage(s)
    │
    ▼
Transform Stage(s) ──► Error Output ──► Error Sink
    │
    ▼
Sink Stage(s)

Stage types:
  ● Source     — reads data (BigQuery, GCS, JDBC, Pub/Sub...)
  ● Transform  — modifies data (Wrangler, Joiner, Group By...)
  ● Sink       — writes data (BigQuery, GCS, JDBC...)
  ● Action     — side effects (delete file, run SQL, HTTP call)
  ● Condition  — branching logic (if/else routing)
  ● Alert      — notifications (email, webhook)
  ● Error      — receives error records from other stages
```

---

### Macros & Runtime Arguments

```
Macro syntax: ${macro_name}
Default value: ${macro_name:default_value}

Example plugin config using macros:
  Table: ${bq_dataset}.${bq_table}
  Date:  ${run_date:2026-03-16}
  Path:  gs://${bucket}/data/${run_date}/

Providing runtime arguments at trigger time:
  CLI:  --runtime-args="bq_table=orders,run_date=2026-03-16"
  API:  POST body: {"runtimeArgs": {"bq_table": "orders", "run_date": "2026-03-16"}}
  UI:   Pipeline → Run → Set Runtime Arguments
```

---

### Pipeline JSON Structure (Export Format)

```json
{
  "name": "orders-etl-pipeline",
  "description": "Daily orders ETL from MySQL to BigQuery",
  "artifact": {
    "name": "cdap-data-pipeline",
    "version": "6.9.2",
    "scope": "SYSTEM"
  },
  "config": {
    "resources": {
      "memoryMB": 2048,
      "virtualCores": 1
    },
    "driverResources": {
      "memoryMB": 2048,
      "virtualCores": 1
    },
    "connections": [
      {"from": "MySQL-Source", "to": "Wrangler-Transform"},
      {"from": "Wrangler-Transform", "to": "BigQuery-Sink"},
      {"from": "Wrangler-Transform", "to": "Error-Collector"}
    ],
    "stages": [
      {
        "name": "MySQL-Source",
        "plugin": {
          "name": "Database",
          "type": "batchsource",
          "artifact": {"name": "database-plugins", "version": "2.9.0"},
          "properties": {
            "jdbcPluginName":  "mysql",
            "connectionString":"jdbc:mysql://${db_host}:3306/${db_name}",
            "user":            "${db_user}",
            "password":        "${secure(db_password)}",
            "tableName":       "orders",
            "splitBy":         "order_id",
            "numSplits":       "4",
            "schema": "{\"type\":\"record\",\"name\":\"etlSchemaBody\",\"fields\":[{\"name\":\"order_id\",\"type\":\"int\"},{\"name\":\"amount\",\"type\":\"double\"},{\"name\":\"created_at\",\"type\":\"string\"}]}"
          }
        }
      },
      {
        "name": "BigQuery-Sink",
        "plugin": {
          "name": "BigQueryTable",
          "type": "batchsink",
          "artifact": {"name": "google-cloud", "version": "0.21.0"},
          "properties": {
            "project":           "my-project",
            "dataset":           "${bq_dataset}",
            "table":             "orders",
            "operation":         "INSERT",
            "truncateTable":     "false",
            "createPartitionedTable": "true",
            "partitioningType":  "TIME",
            "partitionByField":  "created_at"
          }
        }
      }
    ],
    "schedule": "0 2 * * *",
    "engine": "spark",
    "numOfRecordsPreview": 100
  }
}
```

---

### Deploying & Triggering via REST API

```bash
# Get access token
TOKEN=$(gcloud auth print-access-token)

# CDF instance API endpoint (from gcloud describe)
CDF_URL="https://my-cdf-instance-PROJECT_NUMBER-dot-uscentral1.datafusion.googleusercontent.com"
NAMESPACE="default"
PIPELINE="orders-etl-pipeline"

# Deploy a pipeline from JSON file
curl -X PUT \
  "${CDF_URL}/api/v3/namespaces/${NAMESPACE}/apps/${PIPELINE}" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d @pipeline.json

# Start a pipeline run with runtime arguments
curl -X POST \
  "${CDF_URL}/api/v3/namespaces/${NAMESPACE}/apps/${PIPELINE}/workflows/DataPipelineWorkflow/start" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"runtimeArgs": {"bq_dataset": "analytics", "run_date": "2026-03-16"}}'

# Get run status
curl \
  "${CDF_URL}/api/v3/namespaces/${NAMESPACE}/apps/${PIPELINE}/workflows/DataPipelineWorkflow/runs?limit=1" \
  -H "Authorization: Bearer ${TOKEN}"
```

---

### Error Handling Configuration

```json
{
  "name": "Wrangler-Transform",
  "plugin": {
    "name": "Wrangler",
    "type": "transform",
    "properties": {
      "directives": "parse-as-csv :body ','\nset-type :amount double\nsend-to-error !dq:isNumber(amount)",
      "on-error":   "send-to-error-port"
    }
  },
  "outputSchema": "...",
  "errorDatasetName": "error-records"
}
```

```
Error routing in pipeline:
  Wrangler Transform ──[error output]──► GCS Error Sink
                    └──[main output]──► BigQuery Sink

Error records include:
  - Original record data
  - Error message
  - Stage name that produced the error
  - Error code
```

---

## 4. Sources, Sinks & Connectors

### Built-in Plugin Reference

| Plugin | Type | Formats / Systems |
|---|---|---|
| **BigQuery** | Source + Sink | Full table, SQL query, partitioned |
| **GCS** | Source + Sink | CSV, JSON, Avro, Parquet, ORC, Delimited, Text |
| **Cloud Spanner** | Source + Sink | Table reads, SQL query |
| **Cloud Bigtable** | Source + Sink | Row key + column families |
| **Cloud Pub/Sub** | Source (streaming) + Sink | Message body + attributes |
| **Database (JDBC)** | Source + Sink | MySQL, PostgreSQL, SQL Server, Oracle, DB2 |
| **Salesforce** | Source + Sink | SOQL, Bulk API, incremental |
| **SAP** | Source | ODP, Table Reader |
| **Kafka** | Source + Sink | Topic, consumer group, SASL auth |
| **HTTP** | Source + Sink | REST APIs, pagination |
| **Elasticsearch** | Sink | Index, document type |
| **Azure Blob** | Source + Sink | CSV, Avro, Parquet |
| **Amazon S3** | Source + Sink | CSV, Avro, Parquet |

---

### BigQuery Source Configuration

```json
{
  "name": "BigQuery-Source",
  "plugin": {
    "name": "BigQueryTable",
    "type": "batchsource",
    "properties": {
      "project":           "my-project",
      "dataset":           "analytics",
      "table":             "events",
      "filter":            "event_date = '${run_date}'",
      "enableQueryingView": "false",
      "query":             "SELECT user_id, event_type, amount FROM `my-project.analytics.events` WHERE event_date = '${run_date}'",
      "partitionFrom":     "${run_date}",
      "partitionTo":       "${run_date}",
      "schema": "{\"type\":\"record\",\"name\":\"etlSchemaBody\",\"fields\":[{\"name\":\"user_id\",\"type\":\"string\"},{\"name\":\"event_type\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"}]}"
    }
  }
}
```

---

### BigQuery Sink Configuration

```json
{
  "name": "BigQuery-Sink",
  "plugin": {
    "name": "BigQueryTable",
    "type": "batchsink",
    "properties": {
      "project":                "my-project",
      "dataset":                "${output_dataset}",
      "table":                  "orders_processed",
      "operation":              "INSERT",
      "truncateTable":          "false",
      "allowSchemaRelaxation":  "true",
      "createPartitionedTable": "true",
      "partitioningType":       "TIME",
      "partitionByField":       "order_date",
      "clusteringOrder":        "region,status",
      "location":               "US",
      "bigQueryJobLabels":       "pipeline=orders-etl,env=prod"
    }
  }
}
```

**BigQuery write dispositions:**

| Operation | Behavior |
|---|---|
| `INSERT` | Append rows to table (create if not exists) |
| `UPDATE` | Update matching rows (requires `dedupeBy`) |
| `UPSERT` | Insert or update by key |
| `TRUNCATE` | Truncate + reload (full overwrite) |

---

### Cloud Storage Source/Sink

```json
{
  "name": "GCS-Source",
  "plugin": {
    "name": "File",
    "type": "batchsource",
    "properties": {
      "path":            "gs://my-bucket/data/${run_date}/*.parquet",
      "format":          "parquet",
      "recursive":       "true",
      "ignoreMissingFiles": "true",
      "filenameOnly":    "false",
      "copyHeader":      "true"
    }
  }
},
{
  "name": "GCS-Sink",
  "plugin": {
    "name": "File",
    "type": "batchsink",
    "properties": {
      "path":            "gs://my-bucket/output/${run_date}",
      "format":          "parquet",
      "suffix":          "yyyy-MM-dd-HH-mm",
      "codec":           "snappy",
      "writeHeader":     "false",
      "numPartitions":   "10"
    }
  }
}
```

---

### JDBC Source Configuration

```json
{
  "name": "MySQL-Source",
  "plugin": {
    "name": "Database",
    "type": "batchsource",
    "properties": {
      "jdbcPluginName":   "mysql",
      "connectionString": "jdbc:mysql://${host}:3306/${database}?useSSL=true",
      "user":             "${db_user}",
      "password":         "${secure(db_password)}",
      "importQuery":      "SELECT * FROM orders WHERE updated_at >= '${last_run}'",
      "numSplits":        "8",
      "splitBy":          "order_id",
      "fetchSize":        "5000",
      "connectionArguments": "connectTimeout=30000;socketTimeout=300000"
    }
  }
}
```

> 💡 The `splitBy` and `numSplits` parameters enable **parallel reads** — Dataproc workers read different ID ranges simultaneously. Always set these for large tables.

---

### Secure Credential Storage

```bash
# Store a secret in CDF Secure Store (encrypted at rest)
# Via REST API:
curl -X PUT \
  "${CDF_URL}/api/v3/namespaces/default/securekeys/db_password" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"data": "my_secret_password", "description": "MySQL DB password"}'

# Reference in pipeline config:
# "password": "${secure(db_password)}"
```

---

## 5. Transformations & Data Processing

### Core Transform Plugins

| Plugin | Purpose | Key Config |
|---|---|---|
| **Wrangler** | Interactive data prep, directive-based transforms | Directives script, schema |
| **Joiner** | Join 2+ datasets | Join type, join keys, selected fields |
| **Group By** | Aggregate records | Group keys, aggregate functions |
| **Deduplication** | Remove duplicate records | Unique fields, dedup strategy |
| **Pivot** | Rows → columns | Key field, pivot column, aggregate |
| **Unpivot** | Columns → rows | Field to unpivot, output field name |
| **Value Mapper** | Map values using a lookup | Input field, mapping table |
| **Field Renamer** | Rename/reorder fields | Old name → new name pairs |
| **Python Evaluator** | Custom Python 3 logic | Python script, I/O schema |
| **JavaScript Evaluator** | Custom JS logic | JS function body |
| **Switch** | Conditional routing | Conditions → port mappings |
| **CSV Parser** | Parse CSV strings | Delimiter, has header |
| **JSON Parser** | Parse JSON strings | Field to parse, output schema |
| **XML Parser** | Parse XML strings | XPath expressions |
| **Regex Extractor** | Extract via regex | Pattern, output groups |

---

### Joiner Plugin Configuration

```json
{
  "name": "Order-Customer-Join",
  "plugin": {
    "name": "Joiner",
    "type": "batchjoiner",
    "properties": {
      "joinType":          "left",
      "joinCondition":     "orders.user_id = users.id",
      "requiredInputs":    "orders",
      "selectedFields":    "orders.order_id,orders.amount,orders.created_at,users.name,users.email,users.country",
      "numPartitions":     "200",
      "joinNullSafe":      "false"
    }
  }
}
```

| Join Type | Behavior |
|---|---|
| `inner` | Only matching records from both inputs |
| `outer` | All records from both; NULLs where no match |
| `left` | All records from first input; NULLs from second if no match |

---

### Group By Configuration

```json
{
  "name": "Revenue-By-Region",
  "plugin": {
    "name": "GroupByAggregate",
    "type": "batchaggregator",
    "properties": {
      "groupByFields":   "region,order_date",
      "aggregates":      "total_revenue:sum(amount),order_count:count(*),avg_order:avg(amount),max_order:max(amount),unique_customers:countDistinct(user_id)"
    }
  }
}
```

**Supported aggregate functions:**

| Function | Description |
|---|---|
| `count(*)` | Count all records |
| `count(field)` | Count non-null values |
| `countDistinct(field)` | Count unique values |
| `sum(field)` | Sum of numeric field |
| `avg(field)` | Average of numeric field |
| `min(field)` / `max(field)` | Min/max value |
| `stddev(field)` | Standard deviation |
| `collectList(field)` | Aggregate into an array |
| `first(field)` / `last(field)` | First/last value in group |

---

### Python Evaluator

```json
{
  "name": "PII-Masker",
  "plugin": {
    "name": "PythonEvaluator",
    "type": "transform",
    "properties": {
      "script": "import hashlib\n\ndef transform(record, emitter, context):\n    try:\n        # Mask email: keep domain, hash local part\n        email = record['email'] or ''\n        if '@' in email:\n            local, domain = email.split('@', 1)\n            hashed = hashlib.sha256(local.encode()).hexdigest()[:8]\n            record['email_masked'] = f'{hashed}@{domain}'\n        else:\n            record['email_masked'] = 'invalid'\n        \n        # Categorize order value\n        amount = float(record['amount'] or 0)\n        if amount >= 1000:\n            record['tier'] = 'premium'\n        elif amount >= 100:\n            record['tier'] = 'standard'\n        else:\n            record['tier'] = 'basic'\n        \n        emitter.emit(record)\n    except Exception as e:\n        emitter.emitError({'errorCode': 1, 'errorMsg': str(e), 'invalidRecord': str(record)})",
      "pythonVersion": "3",
      "schema": "{\"type\":\"record\",\"name\":\"etlSchemaBody\",\"fields\":[{\"name\":\"order_id\",\"type\":\"int\"},{\"name\":\"amount\",\"type\":\"double\"},{\"name\":\"email_masked\",\"type\":\"string\"},{\"name\":\"tier\",\"type\":\"string\"}]}"
    }
  }
}
```

---

### JavaScript Evaluator

```json
{
  "name": "JS-Transform",
  "plugin": {
    "name": "JavaScript",
    "type": "transform",
    "properties": {
      "script": "function transform(record, emitter, context) {\n  var amount = parseFloat(record.amount || 0);\n  var tax_rate = record.country === 'US' ? 0.08 : 0.20;\n  record.tax = (amount * tax_rate).toFixed(2);\n  record.total = (amount + parseFloat(record.tax)).toFixed(2);\n  emitter.emit(record);\n}",
      "schema": "..."
    }
  }
}
```

---

### Deduplication Plugin

```json
{
  "name": "Dedup-Orders",
  "plugin": {
    "name": "Deduplicate",
    "type": "batchaggregator",
    "properties": {
      "uniqueFields":   "order_id",
      "filterOperation":"last(updated_at)"
    }
  }
}
```

| Strategy | Behavior |
|---|---|
| `any` | Keep any one record per key (fastest) |
| `last(field)` | Keep record with latest value of field |
| `first(field)` | Keep record with earliest value of field |
| `max(field)` | Keep record with max value of field |
| `min(field)` | Keep record with min value of field |

---

## 6. Wrangler (Data Preparation)

### Wrangler Workflow

```
1. Open Wrangler UI (Data Preparation menu)
2. Connect to data source:
   - Upload file (CSV, JSON, Excel, Avro)
   - Connect to BigQuery table
   - Connect to GCS path
   - Connect to database (JDBC)
3. Browse sampled records (first N rows)
4. Apply directives interactively:
   - Click column header for auto-suggestions
   - Type directives manually
   - Preview results after each directive
5. Validate output schema
6. Send to Pipeline (exports as Wrangler stage in Pipeline Studio)
```

---

### Wrangler Directives Reference

| Directive | Syntax | Example | Description |
|---|---|---|---|
| `parse-as-csv` | `parse-as-csv :col ','` | `parse-as-csv :body ','` | Parse CSV string into columns |
| `parse-as-json` | `parse-as-json :col [depth]` | `parse-as-json :payload 1` | Parse JSON string |
| `parse-as-xml` | `parse-as-xml :col` | `parse-as-xml :doc` | Parse XML string |
| `set-column` | `set-column :col expr` | `set-column :total amount * 1.08` | Set column value |
| `drop` | `drop :col1,:col2` | `drop :_id,:raw` | Remove columns |
| `rename` | `rename :old :new` | `rename :usr_id :user_id` | Rename column |
| `set-type` | `set-type :col type` | `set-type :amount double` | Cast column type |
| `fill-null-or-empty` | `fill-null-or-empty :col 'val'` | `fill-null-or-empty :region 'UNKNOWN'` | Replace nulls |
| `filter-rows-on` | `filter-rows-on condition` | `filter-rows-on empty(order_id)` | Filter bad rows |
| `send-to-error` | `send-to-error condition` | `send-to-error !dq:isDate(created_at,'yyyy-MM-dd')` | Route to error |
| `split-to-columns` | `split-to-columns :col 'sep'` | `split-to-columns :tags ','` | Split into multiple cols |
| `merge` | `merge :col1 :col2 :out 'sep'` | `merge :first :last :name ' '` | Merge columns |
| `format-date` | `format-date :col 'fmt'` | `format-date :dt 'yyyy-MM-dd'` | Format date string |
| `hash` | `hash :col alg` | `hash :email SHA-256` | Hash column value |
| `mask-number` | `mask-number :col pattern` | `mask-number :phone #-####-####` | Mask numeric field |
| `lowercase` | `lowercase :col` | `lowercase :email` | Convert to lowercase |
| `uppercase` | `uppercase :col` | `uppercase :country` | Convert to uppercase |
| `trim` | `trim :col` | `trim :name` | Trim whitespace |
| `ltrim` / `rtrim` | `ltrim :col 'chars'` | `ltrim :code '0'` | Trim left/right |
| `find-and-replace` | `find-and-replace :col /regex/replace/` | `find-and-replace :phone /[^0-9]//g` | Regex replace |
| `extract-regex-groups` | `extract-regex-groups :col 'regex'` | `extract-regex-groups :url '(https?)://(.+)'` | Extract groups |
| `columns-replace` | `columns-replace /regex/replace/` | `columns-replace /^_//` | Rename columns by regex |
| `swap` | `swap :col1 :col2` | `swap :first_name :last_name` | Swap column values |
| `write-as-json-map` | `write-as-json-map :col k1:v1,k2:v2` | — | Serialize to JSON map |

---

### Complete Wrangler Recipe Example

```
# orders_cleanup.recipe
# Parse raw CSV line
parse-as-csv :body ',' true

# Remove raw body column
drop :body

# Rename columns to snake_case
rename :col1 :order_id
rename :col2 :customer_email
rename :col3 :amount
rename :col4 :created_date
rename :col5 :status
rename :col6 :region

# Set types
set-type :order_id integer
set-type :amount double

# Clean strings
trim :customer_email
lowercase :customer_email
trim :status
uppercase :region

# Handle nulls
fill-null-or-empty :region 'UNKNOWN'
fill-null-or-empty :status 'pending'

# Format date
format-date :created_date 'yyyy-MM-dd'

# Validate and route bad records
send-to-error !dq:isDate(created_date,'yyyy-MM-dd')
send-to-error empty(order_id)
send-to-error !dq:isNumber(amount)
send-to-error amount < 0

# Hash PII
hash :customer_email SHA-256

# Add derived column
set-column :is_high_value amount >= 500
```

---

## 7. Replication (CDC)

### What is Cloud Data Fusion Replication?

Replication provides **CDC (Change Data Capture)**-based real-time replication from operational databases to BigQuery. It captures INSERT, UPDATE, DELETE operations from database transaction logs (binlog for MySQL, WAL for PostgreSQL) using a **Debezium-based engine**.

---

### Supported Sources

| Database | Protocol | Requirements |
|---|---|---|
| **MySQL** | Binary log (binlog) | binlog_format=ROW, binlog_row_image=FULL |
| **PostgreSQL** | WAL (pglogical) | wal_level=logical, max_replication_slots>0 |
| **SQL Server** | CDC feature | SQL Server CDC enabled on tables |
| **Oracle** | LogMiner | ARCHIVELOG mode, supplemental logging |

---

### Replication Setup

```bash
# Create a replication job via REST API
curl -X POST \
  "${CDF_URL}/api/v3/namespaces/default/replication/instances" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "mysql-to-bq-replication",
    "description": "Replicate orders DB to BigQuery",
    "source": {
      "name": "MySQL",
      "properties": {
        "host":     "10.0.0.5",
        "port":     "3306",
        "database": "orders_db",
        "user":     "repl_user",
        "password": "${secure(mysql_repl_password)}"
      }
    },
    "target": {
      "name": "BigQuery",
      "properties": {
        "project":  "my-project",
        "dataset":  "replicated_orders",
        "location": "US"
      }
    },
    "tables": [
      {
        "database": "orders_db",
        "table":    "orders",
        "columns":  ["order_id", "user_id", "amount", "status", "created_at"],
        "excludeColumns": []
      },
      {
        "database": "orders_db",
        "table":    "order_items"
      }
    ],
    "offsetBasedConsumption": false,
    "startingPosition": "LATEST"
  }'
```

---

### Replication Configuration Options

| Option | Description | Default |
|---|---|---|
| `startingPosition` | `LATEST` (CDC-only) or `EARLIEST` (full snapshot + CDC) | `LATEST` |
| `tables` | Array of table configs to include | All tables |
| `excludeColumns` | Columns to exclude from replication | None |
| `allowTableColumnAddition` | Handle `ADD COLUMN` DDL events | `true` |
| `allowTableColumnDrop` | Handle `DROP COLUMN` DDL events | `false` |
| `maxRetrySeconds` | Max retry on source connectivity loss | `300` |
| `offsetBasedConsumption` | Resume from saved offset after restart | `true` |

---

### Schema Change Handling

| DDL Event | Default Behavior | Configuration |
|---|---|---|
| `ADD COLUMN` | ✅ Adds column to BQ table | `allowTableColumnAddition: true` |
| `DROP COLUMN` | ⚠️ Column remains in BQ (nulled) | `allowTableColumnDrop: false` |
| `RENAME COLUMN` | ❌ Creates new column; old stays | Manual intervention needed |
| `CHANGE TYPE` | ⚠️ Best-effort cast | May require pipeline pause |
| `ADD TABLE` | Ignored (not in config) | Update replication config |

---

### Replication vs. Datastream

| Feature | CDF Replication | Datastream |
|---|---|---|
| **UI** | Visual (no-code) | Code / Console |
| **Sources** | MySQL, PG, SQL Server, Oracle | MySQL, PG, Oracle, SQL Server, AlloyDB |
| **Targets** | BigQuery | BigQuery, GCS, Pub/Sub |
| **Transformation** | ❌ (raw CDC) | ❌ (raw CDC) |
| **Schema evolution** | ✅ Basic | ✅ Advanced |
| **Latency** | Seconds | Seconds |
| **Backfill** | ✅ Snapshot mode | ✅ Snapshot mode |
| **Monitoring** | CDF UI | Datastream UI + Cloud Monitoring |
| **Best for** | CDF-heavy teams, visual management | Standalone CDC, multi-target |

---

## 8. Scheduling & Orchestration

### Built-in Scheduler

```bash
# Enable schedule on a pipeline via REST API
curl -X POST \
  "${CDF_URL}/api/v3/namespaces/default/apps/orders-etl-pipeline/schedules/pipeline-schedule/enable" \
  -H "Authorization: Bearer ${TOKEN}"

# Create a schedule (cron: daily at 2am UTC)
curl -X PUT \
  "${CDF_URL}/api/v3/namespaces/default/apps/orders-etl-pipeline/schedules/pipeline-schedule" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "pipeline-schedule",
    "description": "Run daily at 2am UTC",
    "program": {
      "programName": "DataPipelineWorkflow",
      "programType": "WORKFLOW"
    },
    "properties": {
      "run_date": "${runtime.start_date:yyyy-MM-dd}"
    },
    "schedule": {
      "cronExpression": "0 2 * * *"
    }
  }'
```

---

### Cloud Composer (Airflow) Integration

```python
# Airflow DAG: trigger CDF pipeline, wait for completion
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.google.cloud.operators.datafusion import (
    CloudDataFusionStartPipelineOperator,
    CloudDataFusionDeletePipelineOperator,
)
from airflow.providers.google.cloud.sensors.datafusion import (
    CloudDataFusionPipelineStateSensor,
)

PROJECT_ID    = "my-project"
LOCATION      = "us-central1"
INSTANCE_NAME = "my-cdf-instance"
NAMESPACE     = "default"
PIPELINE_NAME = "orders-etl-pipeline"

default_args = {
    "owner":            "data-team",
    "retries":          2,
    "retry_delay":      timedelta(minutes=5),
    "email_on_failure": True,
    "email":            ["alerts@company.com"],
}

with DAG(
    dag_id="cdf_orders_etl",
    schedule_interval="0 2 * * *",
    start_date=datetime(2026, 1, 1),
    catchup=False,
    default_args=default_args,
    tags=["data-fusion", "orders"],
) as dag:

    start_pipeline = CloudDataFusionStartPipelineOperator(
        task_id="start_cdf_pipeline",
        project_id=PROJECT_ID,
        location=LOCATION,
        instance_name=INSTANCE_NAME,
        namespace=NAMESPACE,
        pipeline_name=PIPELINE_NAME,
        runtime_args={
            "run_date":    "{{ ds }}",
            "bq_dataset":  "analytics",
            "bq_table":    "orders_processed",
        },
        success_states=["RUNNING"],   # Don't wait here; use sensor
        pipeline_timeout=3600,        # Max 1 hour
    )

    wait_for_completion = CloudDataFusionPipelineStateSensor(
        task_id="wait_for_pipeline",
        project_id=PROJECT_ID,
        location=LOCATION,
        instance_name=INSTANCE_NAME,
        namespace=NAMESPACE,
        pipeline_name=PIPELINE_NAME,
        pipeline_id="{{ task_instance.xcom_pull('start_cdf_pipeline') }}",
        expected_statuses=["COMPLETED"],
        failure_statuses=["FAILED", "KILLED"],
        mode="poke",
        poke_interval=60,   # Check every 60 seconds
        timeout=3600,
    )

    start_pipeline >> wait_for_completion
```

---

### Event-Driven Trigger (Cloud Functions → CDF)

```python
# Cloud Function triggered on GCS file arrival → start CDF pipeline
import functions_framework
import requests
import google.auth
import google.auth.transport.requests
import json

INSTANCE_URL = "https://my-cdf-instance-PROJ_NUM-dot-uscentral1.datafusion.googleusercontent.com"
NAMESPACE    = "default"
PIPELINE     = "gcs-to-bq-pipeline"

@functions_framework.cloud_event
def trigger_cdf_pipeline(cloud_event):
    data = cloud_event.data
    bucket = data["bucket"]
    file_path = data["name"]

    print(f"File arrived: gs://{bucket}/{file_path}")

    # Get access token
    credentials, _ = google.auth.default(
        scopes=["https://www.googleapis.com/auth/cloud-platform"]
    )
    auth_req = google.auth.transport.requests.Request()
    credentials.refresh(auth_req)
    token = credentials.token

    # Start CDF pipeline
    url = f"{INSTANCE_URL}/api/v3/namespaces/{NAMESPACE}/apps/{PIPELINE}/workflows/DataPipelineWorkflow/start"
    payload = {
        "runtimeArgs": {
            "input_path": f"gs://{bucket}/{file_path}",
            "run_date":   file_path.split('/')[0]  # Extract date from path
        }
    }

    response = requests.post(
        url,
        json=payload,
        headers={"Authorization": f"Bearer {token}"},
        timeout=30
    )

    if response.status_code == 200:
        run_id = response.json().get("runId")
        print(f"Pipeline started: run_id={run_id}")
    else:
        raise RuntimeError(f"Failed to start pipeline: {response.text}")
```

---

### Cloud Workflows Integration

```yaml
# cloud_workflow.yaml — Orchestrate CDF pipeline via Cloud Workflows
main:
  params: [args]
  steps:
    - init:
        assign:
          - project_id: $${sys.get_env("GOOGLE_CLOUD_PROJECT_ID")}
          - cdf_url: "https://my-cdf-instance-PROJNUM-dot-uscentral1.datafusion.googleusercontent.com"
          - namespace: "default"
          - pipeline: "orders-etl-pipeline"
          - run_date: $${args.run_date}

    - start_pipeline:
        call: http.post
        args:
          url: $${"${cdf_url}/api/v3/namespaces/${namespace}/apps/${pipeline}/workflows/DataPipelineWorkflow/start"}
          auth:
            type: OAuth2
          body:
            runtimeArgs:
              run_date: $${run_date}
        result: start_response

    - extract_run_id:
        assign:
          - run_id: $${start_response.body.runId}

    - poll_status:
        steps:
          - check_status:
              call: http.get
              args:
                url: $${"${cdf_url}/api/v3/namespaces/${namespace}/apps/${pipeline}/workflows/DataPipelineWorkflow/runs/${run_id}"}
                auth:
                  type: OAuth2
              result: status_response
          - evaluate:
              switch:
                - condition: $${status_response.body.status == "COMPLETED"}
                  return: "SUCCESS"
                - condition: $${status_response.body.status == "FAILED"}
                  raise: $${status_response.body}
                - condition: true
                  steps:
                    - wait:
                        call: sys.sleep
                        args:
                          seconds: 30
                    - continue:
                        next: check_status
```

---

## 9. Metadata, Lineage & Data Catalog

### Metadata Entities

Cloud Data Fusion automatically tracks:

| Entity Type | Examples | Metadata Tracked |
|---|---|---|
| **Dataset** | BigQuery table, GCS path, JDBC table | Name, schema, tags, lineage |
| **Program** | Pipeline, Wrangler workspace | Run history, config, lineage |
| **Field** | Column within a dataset | Type, lineage (source → target) |
| **Artifact** | Plugin JARs | Version, properties |
| **Namespace** | Logical isolation unit | Pipelines, datasets within |

---

### Data Lineage

```
Field-Level Lineage Example:
  MySQL.orders.amount
      │
      ▼ (Wrangler: set-type double)
  Wrangler.amount (double)
      │
      ▼ (Group By: sum(amount))
  GroupBy.total_revenue
      │
      ▼
  BigQuery.analytics.daily_revenue.total_revenue

Dataset-Level Lineage:
  MySQL:orders_db.orders ──► [Pipeline: orders-etl] ──► BigQuery:analytics.orders
```

---

### REST API: Querying Lineage

```bash
# Get field-level lineage for a BigQuery table field
curl \
  "${CDF_URL}/api/v3/namespaces/default/metadata/lineage?entityId=BigQuery:my-project.analytics.orders.user_id&levels=3&direction=both" \
  -H "Authorization: Bearer ${TOKEN}"

# Get dataset-level lineage
curl \
  "${CDF_URL}/api/v3/namespaces/default/datasets/BigQuery:my-project.analytics.orders/lineage?start=1700000000&end=1800000000" \
  -H "Authorization: Bearer ${TOKEN}"

# Add a business metadata tag to a dataset
curl -X POST \
  "${CDF_URL}/api/v3/namespaces/default/metadata/tags" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "entityId": "BigQuery:my-project.analytics.orders",
    "tags": ["PII", "financial", "production"]
  }'

# Add custom key-value properties (business metadata)
curl -X POST \
  "${CDF_URL}/api/v3/namespaces/default/metadata/properties" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "entityId": "BigQuery:my-project.analytics.orders",
    "properties": {
      "owner":         "data-engineering-team",
      "sla":           "daily-by-6am",
      "pii_fields":    "customer_email",
      "data_domain":   "commerce"
    }
  }'
```

---

### Data Catalog Integration

Cloud Data Fusion automatically syncs metadata to **Google Cloud Data Catalog**:

- Pipelines appear as **Entries** in Data Catalog
- Datasets (BigQuery tables, GCS paths) are linked with **lineage edges**
- Tags applied in CDF propagate to Data Catalog
- Data Catalog search queries surface CDF-managed assets

> 💡 Enable the **Data Catalog API** (`datacatalog.googleapis.com`) and grant the CDF SA `roles/datacatalog.entryGroupOwner` for automatic sync.

---

## 10. IAM, Security & Networking

### IAM Roles

| Role | Grants |
|---|---|
| `roles/datafusion.admin` | Full control: create/delete instances, manage all resources |
| `roles/datafusion.developer` | Create/edit/run pipelines; no instance management |
| `roles/datafusion.viewer` | Read-only: view pipelines, runs, and metadata |
| `roles/datafusion.runner` | Start/stop pipeline runs only (no edit access) |

```bash
# Grant developer access to a specific instance (not whole project)
gcloud data-fusion instances add-iam-policy-binding my-cdf-instance \
  --location=us-central1 \
  --member="user:engineer@company.com" \
  --role="roles/datafusion.developer"

# Grant runner access (for CI/CD service accounts)
gcloud data-fusion instances add-iam-policy-binding my-cdf-instance \
  --location=us-central1 \
  --member="serviceAccount:cicd-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/datafusion.runner"

# Get current IAM policy
gcloud data-fusion instances get-iam-policy my-cdf-instance \
  --location=us-central1
```

---

### Service Account Architecture

```
User submits pipeline run
       │
       ▼
Cloud Data Fusion SA (service-PROJECT_NUMBER@gcp-sa-datafusion.iam.gserviceaccount.com)
  Needs:
    roles/dataproc.admin             (create/delete Dataproc clusters)
    roles/iam.serviceAccountUser     (use Dataproc SA)
    roles/storage.objectAdmin        (staging GCS bucket)
       │
       ▼ Creates Dataproc cluster
Dataproc Worker SA (custom or Compute Engine default SA)
  Needs:
    roles/bigquery.dataEditor        (write to BQ)
    roles/storage.objectAdmin        (read/write GCS)
    roles/spanner.databaseUser       (if using Spanner)
    roles/bigtable.user              (if using Bigtable)
    roles/pubsub.subscriber          (if reading Pub/Sub)
    roles/pubsub.publisher           (if writing Pub/Sub)
```

---

### CMEK Configuration

```bash
# Create a KMS keyring and key for CDF
gcloud kms keyrings create cdf-keyring --location=us-central1

gcloud kms keys create cdf-key \
  --location=us-central1 \
  --keyring=cdf-keyring \
  --purpose=encryption

# Grant CDF SA and Dataproc SA access to the key
CDF_SA="service-PROJECT_NUMBER@gcp-sa-datafusion.iam.gserviceaccount.com"

gcloud kms keys add-iam-policy-binding cdf-key \
  --location=us-central1 \
  --keyring=cdf-keyring \
  --member="serviceAccount:${CDF_SA}" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Create CMEK-enabled CDF instance
gcloud data-fusion instances create my-cmek-instance \
  --location=us-central1 \
  --type=ENTERPRISE \
  --kms-key=projects/my-project/locations/us-central1/keyRings/cdf-keyring/cryptoKeys/cdf-key
```

---

### Enabling Audit Logging

```bash
# Enable Data Access audit logs for Cloud Data Fusion
cat > audit-policy.yaml << 'EOF'
auditConfigs:
- service: datafusion.googleapis.com
  auditLogConfigs:
  - logType: ADMIN_READ
  - logType: DATA_READ
  - logType: DATA_WRITE
EOF

gcloud projects set-iam-policy my-project audit-policy.yaml
```

---

## 11. Compute Profiles & Performance Tuning

### Compute Profile Parameters

| Parameter | Description | Recommended |
|---|---|---|
| `masterMachineType` | Master VM machine type | `n2-standard-4` |
| `workerMachineType` | Worker VM machine type | `n2-highmem-8` (Spark) |
| `numberOfWorkers` | Primary worker count | 2–10 depending on data size |
| `maxNumberOfWorkers` | Max for autoscaling | Set budget ceiling |
| `secondaryWorkerCount` | Spot/preemptible workers | 50–75% of primary count |
| `secondaryWorkerType` | `SPOT` or `PREEMPTIBLE` | `SPOT` for cost savings |
| `imageVersion` | Dataproc image | `2.1` (Spark 3.3) |
| `masterBootDiskSize` | Master disk (GB) | 100 |
| `workerBootDiskSize` | Worker disk (GB) | 200 (for shuffle) |
| `numWorkerLocalSSDs` | Local SSD count | 1–2 for heavy shuffle |
| `initializationActions` | Custom init scripts | GCS URI |

---

### Spark Configuration Overrides

```json
{
  "name": "orders-etl-pipeline",
  "config": {
    "engine": "spark",
    "properties": {
      "system.spark.executor.memory":             "8g",
      "system.spark.executor.cores":              "4",
      "system.spark.driver.memory":               "4g",
      "system.spark.sql.shuffle.partitions":      "400",
      "system.spark.sql.adaptive.enabled":        "true",
      "system.spark.sql.adaptive.coalescePartitions.enabled": "true",
      "system.spark.sql.adaptive.skewJoin.enabled": "true",
      "system.spark.dynamicAllocation.enabled":   "true",
      "system.spark.dynamicAllocation.minExecutors": "2",
      "system.spark.dynamicAllocation.maxExecutors": "20"
    }
  }
}
```

---

### Performance Optimization Strategies

| Strategy | How | Impact |
|---|---|---|
| **Parallel JDBC reads** | Set `splitBy` + `numSplits` on JDBC source | 🚀 Major for large DBs |
| **BQ partition pushdown** | Set `partitionFrom`/`partitionTo` on BQ source | 🚀 Major BQ cost + speed |
| **Increase shuffle partitions** | `spark.sql.shuffle.partitions=400` | 🚀 For large aggregations |
| **Enable AQE** | `spark.sql.adaptive.enabled=true` | 🚀 Auto-optimizes query plan |
| **Spot secondary workers** | Set `secondaryWorkerType=SPOT` | 💰 60–90% compute savings |
| **Increase batch size** | BigQuery sink `batchSize: 10000` | ✅ Fewer API calls |
| **Local SSDs for shuffle** | `numWorkerLocalSSDs: 2` | 🚀 Faster shuffle |
| **Pushdown optimization** | Enable in JDBC source: `enableAutoCommit` | 🚀 Reduces data transfer |
| **Autoscaling profile** | Set min/max workers | ✅ Right-sizes to workload |
| **Right-size memory** | Match `executor.memory` to worker RAM | ✅ Fewer OOM failures |

---

### Performance Anti-Patterns

| ❌ Anti-Pattern | 🔴 Problem | ✅ Fix |
|---|---|---|
| JDBC source without `splitBy` | Single-threaded read | Add `splitBy=id, numSplits=8` |
| BQ source without partition filter | Full table scan (expensive) | Add `partitionFrom`/`partitionTo` |
| Default `shuffle.partitions=200` on large data | Too few partitions; OOM | Set to 2–3× total executor cores |
| No Spot secondary workers | High compute cost | Add 50% Spot workers |
| Wrangler doing complex joins | Spark shuffle overhead | Use Joiner plugin instead |
| Storing secrets in plaintext | Security risk | Use `${secure(key)}` references |
| Large `numberOfWorkers` for small data | Cluster startup dominates | Use Developer profile for small jobs |
| No autoscaling on variable workloads | Over/under-provisioned | Set min/maxNumberOfWorkers |
| Python Evaluator for every record | Interpreter overhead per row | Batch operations; prefer native transforms |
| No error outputs on transforms | Silent data loss | Add error outputs to all transforms |

---

## 12. REST API & Automation

### API Authentication & Base URL

```bash
# Base URL format
CDF_URL="https://{INSTANCE_NAME}-{PROJECT_NUMBER}-dot-{REGION}.datafusion.googleusercontent.com"
# Get from: gcloud data-fusion instances describe INSTANCE --location=REGION --format="value(apiEndpoint)"

# Authenticate with ADC
TOKEN=$(gcloud auth print-access-token)

# All requests use:
# Authorization: Bearer ${TOKEN}
# Content-Type: application/json
```

---

### Pipeline Management REST Calls

```bash
# List all pipelines in a namespace
curl "${CDF_URL}/api/v3/namespaces/default/apps" \
  -H "Authorization: Bearer ${TOKEN}"

# Get pipeline details
curl "${CDF_URL}/api/v3/namespaces/default/apps/orders-etl-pipeline" \
  -H "Authorization: Bearer ${TOKEN}"

# Deploy (create/update) a pipeline
curl -X PUT \
  "${CDF_URL}/api/v3/namespaces/default/apps/orders-etl-pipeline" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d @pipeline.json

# Delete a pipeline
curl -X DELETE \
  "${CDF_URL}/api/v3/namespaces/default/apps/orders-etl-pipeline" \
  -H "Authorization: Bearer ${TOKEN}"
```

---

### Pipeline Execution REST Calls

```bash
# Start a pipeline run
curl -X POST \
  "${CDF_URL}/api/v3/namespaces/default/apps/orders-etl-pipeline/workflows/DataPipelineWorkflow/start" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"runtimeArgs": {"run_date": "2026-03-16", "bq_dataset": "analytics"}}'

# Stop a running pipeline
curl -X POST \
  "${CDF_URL}/api/v3/namespaces/default/apps/orders-etl-pipeline/workflows/DataPipelineWorkflow/runs/${RUN_ID}/stop" \
  -H "Authorization: Bearer ${TOKEN}"

# Get run status
curl "${CDF_URL}/api/v3/namespaces/default/apps/orders-etl-pipeline/workflows/DataPipelineWorkflow/runs?limit=5" \
  -H "Authorization: Bearer ${TOKEN}"

# Get detailed run metrics
curl "${CDF_URL}/api/v3/namespaces/default/apps/orders-etl-pipeline/workflows/DataPipelineWorkflow/runs/${RUN_ID}/statistics" \
  -H "Authorization: Bearer ${TOKEN}"
```

---

### Python: Full Automation Pattern

```python
import requests
import time
import subprocess
import json
from typing import Optional

def get_access_token() -> str:
    result = subprocess.run(
        ["gcloud", "auth", "print-access-token"],
        capture_output=True, text=True, check=True
    )
    return result.stdout.strip()

class CDFClient:
    def __init__(self, instance_url: str, namespace: str = "default"):
        self.base_url = f"{instance_url}/api/v3/namespaces/{namespace}"
        self.namespace = namespace

    def _headers(self) -> dict:
        return {
            "Authorization": f"Bearer {get_access_token()}",
            "Content-Type": "application/json"
        }

    def deploy_pipeline(self, pipeline_name: str, pipeline_json_path: str) -> bool:
        with open(pipeline_json_path) as f:
            pipeline_config = json.load(f)
        url = f"{self.base_url}/apps/{pipeline_name}"
        resp = requests.put(url, json=pipeline_config, headers=self._headers(), timeout=60)
        resp.raise_for_status()
        print(f"✅ Deployed pipeline: {pipeline_name}")
        return True

    def start_pipeline(self, pipeline_name: str, runtime_args: dict) -> str:
        url = f"{self.base_url}/apps/{pipeline_name}/workflows/DataPipelineWorkflow/start"
        resp = requests.post(url, json={"runtimeArgs": runtime_args},
                            headers=self._headers(), timeout=30)
        resp.raise_for_status()
        run_id = resp.json().get("runId")
        print(f"🚀 Started pipeline: {pipeline_name} (run_id={run_id})")
        return run_id

    def get_run_status(self, pipeline_name: str, run_id: str) -> dict:
        url = f"{self.base_url}/apps/{pipeline_name}/workflows/DataPipelineWorkflow/runs/{run_id}"
        resp = requests.get(url, headers=self._headers(), timeout=30)
        resp.raise_for_status()
        return resp.json()

    def wait_for_completion(self, pipeline_name: str, run_id: str,
                            timeout_seconds: int = 3600,
                            poll_interval: int = 30) -> str:
        terminal_states = {"COMPLETED", "FAILED", "KILLED", "STOPPED"}
        elapsed = 0
        while elapsed < timeout_seconds:
            status = self.get_run_status(pipeline_name, run_id)
            state = status.get("status", "UNKNOWN")
            print(f"⏳ Pipeline {pipeline_name} [{run_id}]: {state} ({elapsed}s elapsed)")

            if state in terminal_states:
                if state == "COMPLETED":
                    print(f"✅ Pipeline completed successfully")
                else:
                    error = status.get("properties", {}).get("failureInfo", "Unknown error")
                    raise RuntimeError(f"❌ Pipeline {state}: {error}")
                return state

            time.sleep(poll_interval)
            elapsed += poll_interval

        raise TimeoutError(f"Pipeline did not complete within {timeout_seconds}s")

    def run_and_wait(self, pipeline_name: str, runtime_args: dict,
                     timeout_seconds: int = 3600) -> str:
        run_id = self.start_pipeline(pipeline_name, runtime_args)
        return self.wait_for_completion(pipeline_name, run_id, timeout_seconds)


# Usage
if __name__ == "__main__":
    cdf = CDFClient(
        instance_url="https://my-cdf-PROJNUM-dot-uscentral1.datafusion.googleusercontent.com"
    )

    # Deploy from file
    cdf.deploy_pipeline("orders-etl-pipeline", "./pipelines/orders-etl.json")

    # Run and wait
    final_state = cdf.run_and_wait(
        pipeline_name="orders-etl-pipeline",
        runtime_args={"run_date": "2026-03-16", "bq_dataset": "analytics"},
        timeout_seconds=1800
    )
    print(f"Final state: {final_state}")
```

---

### CI/CD: GitHub Actions Workflow

```yaml
# .github/workflows/cdf-deploy.yml
name: Deploy Cloud Data Fusion Pipelines

on:
  push:
    branches: [main]
    paths: ['pipelines/**/*.json']

env:
  PROJECT_ID:    ${{ secrets.GCP_PROJECT_ID }}
  REGION:        us-central1
  INSTANCE_NAME: my-cdf-instance
  NAMESPACE:     default

jobs:
  validate-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up gcloud
        uses: google-github-actions/setup-gcloud@v2

      - name: Get CDF API Endpoint
        run: |
          CDF_URL=$(gcloud data-fusion instances describe $INSTANCE_NAME \
            --location=$REGION \
            --format="value(apiEndpoint)")
          echo "CDF_URL=${CDF_URL}" >> $GITHUB_ENV

      - name: Validate pipeline JSON files
        run: |
          for f in pipelines/*.json; do
            echo "Validating: $f"
            python3 -c "import json; json.load(open('$f'))" || exit 1
            echo "✅ Valid JSON: $f"
          done

      - name: Deploy changed pipelines
        run: |
          TOKEN=$(gcloud auth print-access-token)
          for f in $(git diff --name-only HEAD~1 HEAD -- 'pipelines/*.json'); do
            PIPELINE_NAME=$(python3 -c "import json,sys; d=json.load(open('$f')); print(d['name'])")
            echo "Deploying: $PIPELINE_NAME"
            HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
              -X PUT \
              "${CDF_URL}/api/v3/namespaces/${NAMESPACE}/apps/${PIPELINE_NAME}" \
              -H "Authorization: Bearer ${TOKEN}" \
              -H "Content-Type: application/json" \
              -d @$f)
            if [ "$HTTP_CODE" != "200" ]; then
              echo "❌ Deploy failed for $PIPELINE_NAME (HTTP $HTTP_CODE)"
              exit 1
            fi
            echo "✅ Deployed: $PIPELINE_NAME"
          done
```

---

## 13. Monitoring, Logging & Debugging

### Key Metrics to Monitor

| Metric | Description | Alert When |
|---|---|---|
| Pipeline run status | COMPLETED / FAILED / KILLED | Any FAILED or KILLED |
| Run duration | Wall-clock time per run | Exceeds SLA threshold |
| Records in / out per stage | Throughput per stage | Unexpected drop to 0 |
| Error records per stage | Bad records routed to error output | > acceptable threshold |
| Dataproc cluster startup | Time to provision cluster | > 5 minutes |
| YARN memory utilization | Memory pressure on workers | > 85% |
| GCS read/write throughput | I/O bottleneck detection | Sustained low throughput |

---

### Cloud Logging Queries

```bash
# All pipeline run logs for a specific pipeline
gcloud logging read \
  'resource.type="cloud_datafusion_instance"
   labels."pipeline_name"="orders-etl-pipeline"
   severity>=WARNING' \
  --project=my-project \
  --limit=100 \
  --format="table(timestamp,severity,textPayload)"

# Dataproc worker logs for a CDF-triggered cluster
gcloud logging read \
  'resource.type="cloud_dataproc_cluster"
   labels.run_id="RUN_ID_HERE"
   severity>=ERROR' \
  --project=my-project \
  --limit=50

# Failed pipeline runs in the last 24 hours
gcloud logging read \
  'resource.type="cloud_datafusion_instance"
   jsonPayload.status="FAILED"
   timestamp>="2026-03-15T00:00:00Z"' \
  --project=my-project

# OOM errors on Dataproc workers
gcloud logging read \
  'resource.type="cloud_dataproc_cluster"
   textPayload:"OutOfMemoryError"' \
  --project=my-project --limit=20
```

---

### Setting Up Alerting

```bash
# Alert on pipeline failure
gcloud alpha monitoring policies create \
  --display-name="CDF Pipeline Failure" \
  --condition-filter='
    metric.type="datafusion.googleapis.com/pipeline/run_status"
    metric.labels.status="FAILED"' \
  --condition-threshold-value=1 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=0s \
  --notification-channels=CHANNEL_ID \
  --project=my-project
```

---

### Common Failure Causes & Fixes

| Failure | Symptoms | Fix |
|---|---|---|
| **Schema mismatch** | "Field X not found in schema" | Check output schema of previous stage; enable schema propagation |
| **OOM on workers** | `java.lang.OutOfMemoryError` in Dataproc logs | Increase `spark.executor.memory`; use larger machine type |
| **Permission error** | `Access denied` to BigQuery/GCS | Grant Dataproc SA required roles |
| **Macro not resolved** | `${macro_name} unresolved` | Provide runtime argument or default value |
| **JDBC connection timeout** | `Communications link failure` | Check VPC firewall rules; verify DB host reachability |
| **GCS path not found** | `No files found matching pattern` | Verify path; enable `ignoreMissingFiles` if expected |
| **BQ table not found** | `Table not found` | Enable `CREATE_IF_NOT_EXISTS`; verify dataset/project |
| **Wrangler type cast** | `Cannot convert X to type Y` | Add `set-type` before the failing stage |
| **Cluster startup timeout** | Run stuck in `STARTING` | Check Dataproc quotas; verify VPC peering for private IP |
| **Plugin not found** | `Artifact not found` | Install plugin from Hub; verify version compatibility |

---

## 14. gcloud CLI & REST API Quick Reference

### gcloud data-fusion Commands

```bash
# ── INSTANCES ─────────────────────────────────────────────────────────
# Create
gcloud data-fusion instances create NAME \
  --location=REGION --type=BASIC|ENTERPRISE|DEVELOPER

# List
gcloud data-fusion instances list --location=REGION \
  --format="table(name,state,type,apiEndpoint)"

# Describe (get SA, endpoint, state)
gcloud data-fusion instances describe NAME \
  --location=REGION

# Delete
gcloud data-fusion instances delete NAME \
  --location=REGION --quiet

# Get IAM policy
gcloud data-fusion instances get-iam-policy NAME \
  --location=REGION

# Add IAM binding
gcloud data-fusion instances add-iam-policy-binding NAME \
  --location=REGION \
  --member="user:EMAIL" \
  --role="roles/datafusion.developer"

# Restart instance
gcloud data-fusion instances restart NAME \
  --location=REGION

# Upgrade instance
gcloud data-fusion instances update NAME \
  --location=REGION \
  --version=6.9.2
```

---

### REST API Quick Reference

| Operation | Method | URL Path |
|---|---|---|
| **List pipelines** | GET | `/v3/namespaces/{ns}/apps` |
| **Get pipeline** | GET | `/v3/namespaces/{ns}/apps/{name}` |
| **Deploy pipeline** | PUT | `/v3/namespaces/{ns}/apps/{name}` |
| **Delete pipeline** | DELETE | `/v3/namespaces/{ns}/apps/{name}` |
| **Start run** | POST | `/v3/namespaces/{ns}/apps/{name}/workflows/DataPipelineWorkflow/start` |
| **Stop run** | POST | `/v3/namespaces/{ns}/apps/{name}/workflows/DataPipelineWorkflow/runs/{runId}/stop` |
| **Get run status** | GET | `/v3/namespaces/{ns}/apps/{name}/workflows/DataPipelineWorkflow/runs/{runId}` |
| **List runs** | GET | `/v3/namespaces/{ns}/apps/{name}/workflows/DataPipelineWorkflow/runs?limit=N` |
| **List schedules** | GET | `/v3/namespaces/{ns}/apps/{name}/schedules` |
| **Enable schedule** | POST | `/v3/namespaces/{ns}/apps/{name}/schedules/{schedName}/enable` |
| **List namespaces** | GET | `/v3/namespaces` |
| **List secure keys** | GET | `/v3/namespaces/{ns}/securekeys` |
| **Set secure key** | PUT | `/v3/namespaces/{ns}/securekeys/{key}` |
| **Get lineage** | GET | `/v3/namespaces/{ns}/datasets/{dataset}/lineage` |

---

### Output Formatting

```bash
# JSON output
gcloud data-fusion instances list --format=json

# Custom table
gcloud data-fusion instances list \
  --format="table(name,state,type,createTime.date('%Y-%m-%d'))"

# Extract single value
gcloud data-fusion instances describe my-cdf \
  --location=us-central1 \
  --format="value(apiEndpoint)"

# Filter by state
gcloud data-fusion instances list \
  --filter="state=ACTIVE" \
  --format="table(name,type,state)"
```

---

## 15. Pricing Summary

> ⚠️ Prices are approximate as of early 2026. Verify at [cloud.google.com/data-fusion/pricing](https://cloud.google.com/data-fusion/pricing).

### Instance Pricing

| Edition | Per-Hour Cost | Notes |
|---|---|---|
| **Developer** | ~$0.35/hr | ~$255/month; no SLA; dev/test only |
| **Basic** | ~$2.00/hr | ~$1,460/month; production-ready |
| **Enterprise** | ~$4.00/hr | ~$2,920/month; multi-namespace, CDC, CMEK |

> ⚠️ Instance cost is **always running** while instance exists — even when no pipelines run. **Delete unused instances.**

---

### Dataproc Compute Costs (Separate)

Each pipeline run provisions a Dataproc cluster:

| Component | Cost |
|---|---|
| GCE VMs (master + workers) | Standard GCE per-vCPU-hour |
| Dataproc premium | +$0.010/vCPU-hour |
| Spot/preemptible workers | 60–90% discount on VMs |
| Cluster startup overhead | ~2–3 min × cluster cost |

---

### 💰 Cost Optimization Tips

| Tip | Savings | How |
|---|---|---|
| Use Developer edition for dev/test | ~83% vs Basic | `--type=DEVELOPER` |
| Delete idle instances | 100% idle cost | `gcloud data-fusion instances delete` |
| Use Spot secondary workers | 60–90% on workers | `secondaryWorkerType: SPOT` |
| Minimize cluster size with autoscaling | 20–50% | Set `min/maxNumberOfWorkers` |
| Schedule off-peak | Indirect (no savings) | Reduce latency for users |
| Right-size master machine | 10–30% | Use `n2-standard-2` for small jobs |
| Use ephemeral clusters | Inherent (default) | Default behavior; no persistent cluster |
| Stop pipelines during maintenance | Direct | Disable schedules temporarily |

---

### Monthly Cost Estimate

```
Scenario: Basic edition, 1 daily ETL pipeline
  - 4 workers × n2-highmem-8 (8 vCPU, 64 GB) + 1 master
  - 30-minute runtime per day, 22 business days/month

Instance cost:  $2.00 × 730 hr          = $1,460/month
Dataproc VMs:   (4+1) × 8 vCPU × $0.041 × (0.5h × 22) = $180/month
Dataproc prem:  (4+1) × 8 × $0.010 × 11h               = $4.40/month
──────────────────────────────────────────────────────────────────────
Total:                                                    ~$1,644/month

With Spot secondary workers (replace 3 of 4 workers):
  Worker savings: 3 workers × ~75% discount              = -$100/month
  Total with Spot:                                       ~$1,544/month

Developer edition (dev/test, same pipeline):
  Instance cost: $0.35 × 730h                            = $255/month
  Dataproc:                                              = $184/month
  Total dev:                                             ~$439/month
```

---

## 16. Quick Reference & Comparison Tables

### Edition Comparison

| Feature | Developer | Basic | Enterprise |
|---|---|---|---|
| **SLA** | ❌ | 99.9% | 99.9% |
| **Private IP** | ❌ | ✅ | ✅ |
| **Streaming pipelines** | ❌ | ✅ | ✅ |
| **Multiple namespaces** | ❌ | ❌ | ✅ |
| **Replication (CDC)** | ❌ | ❌ | ✅ |
| **CMEK** | ❌ | ✅ | ✅ |
| **VPC SC support** | ❌ | ✅ | ✅ |
| **Max concurrent runs** | 5 | 20 | 50 |
| **Price/hour** | ~$0.35 | ~$2.00 | ~$4.00 |

---

### Cloud Data Fusion vs. Dataflow vs. Dataproc

| Dimension | Cloud Data Fusion | Dataflow | Dataproc |
|---|---|---|---|
| **Interface** | Visual GUI | Code (Java/Python) | Code (Spark/Hive) |
| **Skill level** | Low | High | High |
| **Streaming** | ✅ Basic | ✅ Advanced | ✅ Spark Streaming |
| **Batch ETL** | ✅ Excellent | ✅ | ✅ |
| **CDC / Replication** | ✅ Native | ❌ | ❌ |
| **Data lineage** | ✅ Native | ❌ | ❌ |
| **Arbitrary code** | ✅ Python/JS plugins | ✅ Full Beam SDK | ✅ Full Spark |
| **Execution engine** | Spark (via Dataproc) | Dataflow service | Spark/Hadoop |
| **Startup time** | ~3 min (cluster) | ~2–5 min | ~90s (cluster) |
| **Best for** | Mixed teams, governed ETL | Complex streaming | Spark workloads, ML |

---

### Pipeline Lifecycle States

| State | Description | Transitions To |
|---|---|---|
| **DRAFT** | Pipeline created but not published | → PUBLISHED |
| **PUBLISHED** | Published and runnable | → RUNNING, → DRAFT |
| **PROVISIONING** | Dataproc cluster being created | → STARTING |
| **STARTING** | Spark job being submitted | → RUNNING |
| **RUNNING** | Pipeline actively executing | → COMPLETED, FAILED, KILLED, STOPPED |
| **COMPLETED** | Finished successfully | Terminal |
| **FAILED** | Finished with error | Terminal |
| **KILLED** | Manually stopped mid-run | Terminal |
| **STOPPED** | Graceful stop requested | Terminal |

---

### Wrangler Directives Quick Reference

| Directive | Syntax | Use Case |
|---|---|---|
| `parse-as-csv` | `parse-as-csv :col ',' true` | Parse CSV line |
| `parse-as-json` | `parse-as-json :col 1` | Parse JSON string |
| `set-column` | `set-column :col expression` | Calculate new column |
| `set-type` | `set-type :col double` | Cast column type |
| `rename` | `rename :old :new` | Rename column |
| `drop` | `drop :col1,:col2` | Remove columns |
| `fill-null-or-empty` | `fill-null-or-empty :col 'val'` | Handle nulls |
| `filter-rows-on` | `filter-rows-on empty(:col)` | Remove rows |
| `send-to-error` | `send-to-error condition` | Route bad records |
| `hash` | `hash :col SHA-256` | Hash PII fields |
| `lowercase` / `uppercase` | `lowercase :col` | Case conversion |
| `trim` | `trim :col` | Strip whitespace |
| `format-date` | `format-date :col 'yyyy-MM-dd'` | Standardize dates |
| `find-and-replace` | `find-and-replace :col /regex/rep/` | Regex replace |
| `merge` | `merge :a :b :out ' '` | Concatenate columns |
| `split-to-columns` | `split-to-columns :col ','` | Split by delimiter |

---

### Compute Profile Parameter Reference

| Parameter | Type | Default | Description |
|---|---|---|---|
| `masterMachineType` | String | `n1-standard-4` | Master VM type |
| `workerMachineType` | String | `n1-standard-4` | Worker VM type |
| `numberOfWorkers` | Integer | `2` | Primary worker count |
| `maxNumberOfWorkers` | Integer | `0` (disabled) | Max for autoscaling |
| `secondaryWorkerCount` | Integer | `0` | Spot/preemptible count |
| `secondaryWorkerType` | Enum | `PREEMPTIBLE` | `SPOT` or `PREEMPTIBLE` |
| `imageVersion` | String | `2.0` | Dataproc image version |
| `masterBootDiskSize` | Integer (GB) | `100` | Master disk size |
| `workerBootDiskSize` | Integer (GB) | `100` | Worker disk size |
| `numWorkerLocalSSDs` | Integer | `0` | Local SSD count per worker |
| `zone` | String | Auto | GCP zone |
| `region` | String | Instance region | Dataproc region |
| `network` | String | Default | VPC network |
| `serviceAccount` | String | Compute default | Dataproc SA |
| `initializationActions` | String | — | GCS URI of init script |

---

### Key REST API Endpoints One-Liner Reference

```bash
BASE="${CDF_URL}/api/v3/namespaces/default"
H="Authorization: Bearer $(gcloud auth print-access-token)"

# CRUD
curl "$BASE/apps"                                                -H "$H"          # List pipelines
curl "$BASE/apps/PIPELINE"                                       -H "$H"          # Get pipeline
curl -X PUT "$BASE/apps/PIPELINE" -H "$H" -H "Content-Type: application/json" -d @p.json  # Deploy
curl -X DELETE "$BASE/apps/PIPELINE"                             -H "$H"          # Delete

# Execution
curl -X POST "$BASE/apps/PIPELINE/workflows/DataPipelineWorkflow/start" -H "$H" -d '{}'  # Start
curl -X POST "$BASE/apps/PIPELINE/workflows/DataPipelineWorkflow/runs/RUN/stop"    -H "$H"  # Stop
curl "$BASE/apps/PIPELINE/workflows/DataPipelineWorkflow/runs?limit=5"             -H "$H"  # List runs
curl "$BASE/apps/PIPELINE/workflows/DataPipelineWorkflow/runs/RUN"                 -H "$H"  # Get run

# Scheduling
curl "$BASE/apps/PIPELINE/schedules"                             -H "$H"          # List schedules
curl -X POST "$BASE/apps/PIPELINE/schedules/SCHED/enable"        -H "$H"          # Enable schedule
curl -X POST "$BASE/apps/PIPELINE/schedules/SCHED/disable"       -H "$H"          # Disable schedule

# Security
curl -X PUT "$BASE/securekeys/KEY" -H "$H" -d '{"data":"value"}' -H "Content-Type: application/json" # Set secret
```

---

*Generated for GCP Cloud Data Fusion | Official Docs: [cloud.google.com/data-fusion/docs](https://cloud.google.com/data-fusion/docs)*
