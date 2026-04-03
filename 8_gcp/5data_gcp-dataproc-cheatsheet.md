# 🚀 GCP Dataproc — Comprehensive Cheatsheet

> **Audience:** Data engineers, platform engineers, and architects running Spark, Hadoop, Hive, Presto, and Flink workloads on GCP.
> **Last updated:** March 2026 | Covers Dataproc on GCE, Dataproc on GKE, and Dataproc Serverless.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Cluster Architecture & Configuration](#2-cluster-architecture--configuration)
3. [Running Jobs](#3-running-jobs)
4. [Dataproc Serverless (Spark)](#4-dataproc-serverless-spark)
5. [Storage & Data Access Patterns](#5-storage--data-access-patterns)
6. [Autoscaling & Cost Optimization](#6-autoscaling--cost-optimization)
7. [Workflow Templates & Orchestration](#7-workflow-templates--orchestration)
8. [Spark on Dataproc — Deep Dive](#8-spark-on-dataproc--deep-dive)
9. [Security & Networking](#9-security--networking)
10. [Monitoring, Logging & Debugging](#10-monitoring-logging--debugging)
11. [gcloud CLI & API Quick Reference](#11-gcloud-cli--api-quick-reference)
12. [Pricing Summary](#12-pricing-summary)
13. [Quick Reference & Comparison Tables](#13-quick-reference--comparison-tables)

---

## 1. Overview & Key Concepts

### What is Dataproc?

**Google Cloud Dataproc** is a fully managed service for running Apache Spark, Hadoop, Hive, Presto/Trino, Flink, and other open-source data processing frameworks on GCP. Clusters provision in **~90 seconds**, integrate natively with GCS, BigQuery, and Bigtable, and support ephemeral (per-job) or persistent cluster patterns.

**Core value proposition:**

| Pillar | Description |
|---|---|
| **Fast provisioning** | Clusters ready in ~90s vs. hours for self-managed |
| **Deep GCP integration** | Native connectors for GCS, BigQuery, Bigtable, Pub/Sub |
| **Open-source compatible** | Run existing Spark/Hadoop code with zero changes |
| **Flexible billing** | Ephemeral clusters, Spot workers, autoscaling |
| **Managed components** | Automatic version updates, security patches |

---

### Dataproc vs. Alternatives

| Dimension | Dataproc | Self-managed Spark | Dataflow | BigQuery |
|---|---|---|---|---|
| **Framework** | Spark / Hadoop / Hive | Any | Apache Beam | SQL only |
| **Management** | Managed cluster | Full DIY | Fully serverless | Fully serverless |
| **Custom code** | ✅ Any Spark/Hadoop API | ✅ Full control | ✅ Beam DoFn | ❌ SQL + UDFs |
| **Streaming** | ✅ Spark Structured Streaming | ✅ | ✅ Native | ⚠️ Limited |
| **Startup time** | ~90 seconds | 30–60 min | ~2–5 min | Instant |
| **Port existing code** | ✅ Zero changes | ✅ | ❌ Rewrite needed | ❌ Rewrite needed |
| **Best for** | Spark/Hadoop migrations, Hive, ML | Full control needed | New streaming ETL | SQL analytics |

> 💡 **Choose Dataproc when:** you have existing Spark/Hadoop code, need Hive/HBase/Presto, or require open-source ecosystem flexibility. Choose Dataflow for new streaming pipelines. Choose BigQuery for SQL analytics.

---

### Core Components

| Component | Description |
|---|---|
| **Cluster** | Managed group of GCE VMs running Spark/Hadoop daemons |
| **Job** | A single Spark/Hive/Pig/etc. execution submitted to a cluster |
| **Workflow Template** | DAG of jobs with cluster lifecycle management |
| **Autoscaling Policy** | YAML rules for scaling worker count based on YARN metrics |
| **Dataproc Metastore** | Managed Apache Hive Metastore (shared across clusters) |
| **Initialization Action** | Shell scripts run on every cluster node at creation time |

---

### Cluster Types

| Type | Masters | Workers | Use Case |
|---|---|---|---|
| **Standard** | 1 | 2+ | Default; development and production batch |
| **Single Node** | 1 (also worker) | 0 | Development, testing, small jobs |
| **High Availability (HA)** | 3 | 2+ | Production; survives master failure |

> ⚠️ **HA clusters** require 3 masters running Zookeeper for leader election. HDFS NameNode is active/standby. Use for long-running or mission-critical clusters.

---

### Dataproc Variants

| Variant | Cluster | Management | Storage | Best For |
|---|---|---|---|---|
| **Dataproc on GCE** | GCE VMs | Managed VMs | GCS or HDFS | Most workloads |
| **Dataproc on GKE** | GKE pods | K8s-native | GCS | Containerized workloads, multi-tenant |
| **Dataproc Serverless** | None (auto) | Fully serverless | GCS | Ad-hoc batch, no cluster overhead |

---

### Dataproc Serverless

- **No cluster to create or manage** — submit a job, infrastructure auto-provisions
- Supports **PySpark** and **Spark (Scala/Java)** batch workloads
- Billed per **DCU-hour** (Dataproc Compute Unit) — no idle cluster cost
- Interactive sessions available via **Vertex AI Workbench** integration

---

### Supported Open-Source Components

| Component | Purpose | Notes |
|---|---|---|
| **Apache Spark** | Batch + streaming data processing | Core component, always available |
| **Apache Hadoop** | MapReduce + HDFS | Always available |
| **Apache Hive** | SQL on Hadoop/GCS | Included; use Metastore for persistence |
| **Apache Pig** | Data flow scripting | Legacy; prefer Spark |
| **Apache Tez** | Hive execution engine | DAG-based, faster than MapReduce |
| **Presto / Trino** | Interactive SQL | Optional component |
| **Apache Flink** | Stream processing | Optional component |
| **Jupyter Notebook** | Interactive development | Optional; via Component Gateway |
| **Apache Zeppelin** | Notebook for Spark/SQL | Optional |

---

### Image Versions

Dataproc image versions tie together all component versions:

| Image Version | Spark | Hadoop | Hive | Python |
|---|---|---|---|---|
| `2.2` | 3.5.x | 3.3.x | 3.1.x | 3.11 |
| `2.1` | 3.3.x | 3.3.x | 3.1.x | 3.10 |
| `2.0` | 3.1.x | 3.2.x | 3.1.x | 3.8 |
| `1.5` (legacy) | 2.4.x | 2.10.x | 2.3.x | 3.7 |

```bash
# List available image versions
gcloud dataproc clusters list-supported-images \
  --region=us-central1

# Specify image version at cluster creation
gcloud dataproc clusters create my-cluster \
  --image-version=2.2 \
  --region=us-central1
```

---

## 2. Cluster Architecture & Configuration

### Cluster Anatomy

```
HA Cluster (3 masters + N workers):
┌─────────────────────────────────────────────────────────┐
│  Master 0 (Active)      Master 1         Master 2        │
│  - HDFS NameNode       - NameNode        - Zookeeper     │
│  - YARN ResourceMgr    - Standby         - Zookeeper     │
│  - Spark History Srv   - Zookeeper       - ResourceMgr   │
└─────────────────────────────────────────────────────────┘
┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────┐
│  Worker 0  │ │  Worker 1  │ │  Worker N  │ │ Preempt. │
│  HDFS DN   │ │  HDFS DN   │ │  HDFS DN   │ │ Worker   │
│  YARN NM   │ │  YARN NM   │ │  YARN NM   │ │ YARN NM  │
│  Executors │ │  Executors │ │  Executors │ │ (no HDFS)│
└────────────┘ └────────────┘ └────────────┘ └──────────┘
```

**Node roles:**

| Role | Node | Component |
|---|---|---|
| HDFS NameNode | Master | Manages HDFS namespace |
| YARN ResourceManager | Master | Allocates cluster resources to jobs |
| HDFS DataNode | Worker | Stores HDFS blocks |
| YARN NodeManager | Worker | Runs containers/executors on node |
| Spark Driver | Master (client) or Worker (cluster) | Coordinates Spark job |
| Spark Executor | Workers | Executes Spark tasks |

---

### GCS vs. HDFS Storage

| Feature | HDFS | GCS (Recommended) |
|---|---|---|
| **Persistence** | Lost when cluster deleted | ✅ Persists independently |
| **Cluster resize** | Rebalancing required | ✅ No impact |
| **Cost** | Tied to cluster VMs | ✅ Cheap object storage |
| **Sharing** | Per-cluster | ✅ Share across clusters |
| **Performance** | Low latency local | ~same for large files |
| **Operations** | Must manage HDFS | ✅ Fully managed |

> 💡 **Always use GCS as primary storage** (`gs://` URIs) and avoid HDFS for data persistence. Use HDFS only for local shuffle intermediates (automatic).

---

### Cluster Properties

Configure Spark, Hadoop, YARN, and Hive settings via `--properties`:

```bash
gcloud dataproc clusters create my-cluster \
  --region=us-central1 \
  --properties=\
"spark:spark.executor.memory=4g,\
spark:spark.executor.cores=2,\
spark:spark.dynamicAllocation.enabled=true,\
spark:spark.sql.shuffle.partitions=200,\
yarn:yarn.nodemanager.resource.memory-mb=14336,\
yarn:yarn.nodemanager.resource.cpu-vcores=4,\
hive:hive.execution.engine=tez,\
hdfs:dfs.replication=2,\
mapred:mapreduce.map.memory.mb=2048"
```

**Property prefix mapping:**

| Prefix | Config File |
|---|---|
| `spark:` | `spark-defaults.conf` |
| `yarn:` | `yarn-site.xml` |
| `hdfs:` | `hdfs-site.xml` |
| `hive:` | `hive-site.xml` |
| `mapred:` | `mapred-site.xml` |
| `core:` | `core-site.xml` |
| `dataproc:` | Dataproc-specific settings |

---

### Full Cluster Create Command

```bash
gcloud dataproc clusters create MY_CLUSTER \
  --region=us-central1 \
  --zone=us-central1-a \                          # Omit for auto-zone selection
  --image-version=2.2 \
  --master-machine-type=n2-standard-8 \           # Master VM type
  --master-boot-disk-size=100GB \
  --master-boot-disk-type=pd-ssd \
  --num-workers=4 \
  --worker-machine-type=n2-highmem-8 \            # Worker VM type
  --worker-boot-disk-size=200GB \
  --num-secondary-workers=4 \                     # Preemptible/Spot workers
  --secondary-worker-type=spot \                  # or preemptible
  --enable-component-gateway \                    # Secure UI access
  --optional-components=JUPYTER,PRESTO \          # Optional components
  --initialization-actions=gs://bucket/init.sh \ # Init scripts
  --metadata=key1=val1,key2=val2 \               # Node metadata
  --labels=env=prod,team=data-eng \              # Cluster labels
  --service-account=sa@proj.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --subnet=projects/proj/regions/us-central1/subnetworks/my-subnet \
  --no-address \                                  # Private IP (no external)
  --max-idle=2h \                                # Auto-delete if idle 2h
  --max-age=12h \                                # Auto-delete after 12h
  --properties=spark:spark.executor.memory=6g \
  --num-master-local-ssds=1 \                    # Local SSD for shuffle
  --num-worker-local-ssds=2 \
  --high-availability                            # 3-master HA mode
```

---

### Initialization Actions

Init actions are shell scripts run on **every node** at cluster creation, as `root`.

```bash
# Use a built-in init action (from the public GCS bucket)
gcloud dataproc clusters create my-cluster \
  --initialization-actions=\
    gs://goog-dataproc-initialization-actions-us-central1/connectors/connectors.sh,\
    gs://my-bucket/custom-init.sh \
  --metadata=\
    spark-bigquery-connector-version=0.34.0,\
    custom-setting=my-value

# Custom init action example
cat > init.sh << 'EOF'
#!/bin/bash
set -euxo pipefail

# Install Python libraries on all nodes
pip install pandas==2.0.3 pyarrow==14.0.0 scikit-learn==1.3.0

# Install only on master
ROLE=$(/usr/share/google/get_metadata_value attributes/dataproc-role)
if [[ "${ROLE}" == 'Master' ]]; then
  pip install jupyter-contrib-nbextensions
fi
EOF

gsutil cp init.sh gs://my-bucket/init.sh
```

> ⚠️ Init actions **block cluster creation** until they complete (or time out at 10 min). Keep them fast. Failures cause cluster creation to fail. Use `--initialization-action-timeout` to override.

---

### Optional Components

```bash
# Enable optional components
gcloud dataproc clusters create my-cluster \
  --optional-components=JUPYTER,PRESTO,FLINK,ZEPPELIN,DOCKER \
  --enable-component-gateway \
  --region=us-central1
```

| Component | Flag Value | Access Via |
|---|---|---|
| Jupyter Notebook | `JUPYTER` | Component Gateway |
| Apache Zeppelin | `ZEPPELIN` | Component Gateway |
| Presto / Trino | `PRESTO` | Component Gateway / SSH |
| Apache Flink | `FLINK` | YARN / Component Gateway |
| Apache Ranger | `RANGER` | SSH |
| Docker | `DOCKER` | SSH |
| Solr | `SOLR` | SSH |

---

## 3. Running Jobs

### Supported Job Types

| Job Type | CLI Subcommand | Language | Use Case |
|---|---|---|---|
| **Spark** | `spark` | Scala / Java | DataFrame/RDD processing |
| **PySpark** | `pyspark` | Python | Python-based Spark jobs |
| **SparkR** | `spark-r` | R | R-based Spark jobs |
| **SparkSQL** | `spark-sql` | SQL | SQL-first workloads |
| **Hive** | `hive` | HiveQL | SQL on large datasets |
| **Pig** | `pig` | Pig Latin | Legacy data flows |
| **Hadoop MapReduce** | `hadoop` | Java | Legacy MapReduce jobs |
| **Presto** | `presto` | SQL | Interactive query |
| **Flink** | `flink` | Java / Scala | Stateful stream processing |

---

### gcloud dataproc jobs submit Examples

```bash
# ── PySpark job ───────────────────────────────────────────────────────
gcloud dataproc jobs submit pyspark gs://bucket/my_job.py \
  --cluster=MY_CLUSTER \
  --region=us-central1 \
  --py-files=gs://bucket/utils.py,gs://bucket/deps.zip \
  --files=gs://bucket/config.yaml \
  --properties=spark.executor.memory=4g,spark.executor.cores=2 \
  --driver-log-levels=root=INFO \
  -- \                                              # Job arguments follow
  --input=gs://bucket/input/ \
  --output=gs://bucket/output/ \
  --date=2026-03-16

# ── Spark (JAR) job ───────────────────────────────────────────────────
gcloud dataproc jobs submit spark \
  --cluster=MY_CLUSTER \
  --region=us-central1 \
  --class=com.example.MyMainClass \
  --jars=gs://bucket/my-job-1.0.jar,gs://bucket/deps.jar \
  --properties=spark.executor.memory=8g \
  -- \
  --input=gs://bucket/data/ \
  --output=gs://bucket/results/

# ── SparkSQL job ──────────────────────────────────────────────────────
gcloud dataproc jobs submit spark-sql \
  --cluster=MY_CLUSTER \
  --region=us-central1 \
  --file=gs://bucket/queries/my_query.sql \
  --properties=spark.sql.shuffle.partitions=400

# ── Hive job ──────────────────────────────────────────────────────────
gcloud dataproc jobs submit hive \
  --cluster=MY_CLUSTER \
  --region=us-central1 \
  --file=gs://bucket/hive/my_script.hql \
  --hive-variables=dt=2026-03-16,env=prod

# ── Hadoop MapReduce job ──────────────────────────────────────────────
gcloud dataproc jobs submit hadoop \
  --cluster=MY_CLUSTER \
  --region=us-central1 \
  --class=org.apache.hadoop.examples.WordCount \
  --jars=file:///usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar \
  -- \
  gs://bucket/input/ gs://bucket/output/

# ── List/describe/cancel jobs ─────────────────────────────────────────
gcloud dataproc jobs list --cluster=MY_CLUSTER --region=us-central1
gcloud dataproc jobs describe JOB_ID --region=us-central1
gcloud dataproc jobs cancel JOB_ID --region=us-central1
```

---

### Job Lifecycle

```
PENDING → SETUP_DONE → RUNNING → DONE
                              ↘ CANCELLED
                              ↘ ERROR
```

| State | Description |
|---|---|
| `PENDING` | Job queued; cluster acquiring resources |
| `SETUP_DONE` | Driver launched; executors allocating |
| `RUNNING` | Actively processing data |
| `DONE` | Completed successfully |
| `ERROR` | Failed; check driver logs |
| `CANCELLED` | Manually cancelled |

---

### Complete PySpark Job Example

```python
# my_etl_job.py — Submit with: gcloud dataproc jobs submit pyspark
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
import argparse

def main(input_path: str, output_path: str, date: str):
    spark = SparkSession.builder \
        .appName("DataprocETLJob") \
        .config("spark.sql.adaptive.enabled", "true") \
        .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
        .getOrCreate()

    spark.sparkContext.setLogLevel("WARN")

    # Read from GCS
    df = spark.read.parquet(input_path)

    # Transform
    result = (df
        .filter(F.col("event_date") == date)
        .groupBy("user_id", "product_category")
        .agg(
            F.sum("amount").alias("total_amount"),
            F.count("*").alias("txn_count"),
            F.avg("amount").alias("avg_amount")
        )
        .withColumn("processed_at", F.current_timestamp())
    )

    # Write back to GCS as Parquet (partitioned)
    result.write \
        .mode("overwrite") \
        .partitionBy("event_date") \
        .parquet(output_path)

    print(f"Written {result.count()} rows to {output_path}")
    spark.stop()

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--input",  required=True)
    parser.add_argument("--output", required=True)
    parser.add_argument("--date",   required=True)
    args = parser.parse_args()
    main(args.input, args.output, args.date)
```

---

### HiveQL Job Example

```sql
-- my_query.hql — Submit with: gcloud dataproc jobs submit hive --file=...
-- Variable substitution: --hive-variables dt=2026-03-16

SET hive.execution.engine=tez;
SET hive.tez.auto.reducer.parallelism=true;

CREATE EXTERNAL TABLE IF NOT EXISTS raw_events (
  user_id       STRING,
  event_type    STRING,
  amount        DOUBLE,
  event_ts      TIMESTAMP
)
STORED AS PARQUET
LOCATION 'gs://my-bucket/raw/events/'
TBLPROPERTIES ("parquet.compress"="SNAPPY");

-- Aggregate and write to output table
INSERT OVERWRITE TABLE daily_summary
PARTITION (dt = '${dt}')
SELECT
  user_id,
  event_type,
  SUM(amount)  AS total_amount,
  COUNT(*)     AS event_count
FROM raw_events
WHERE DATE(event_ts) = '${dt}'
GROUP BY user_id, event_type;
```

---

### Accessing Logs

```bash
# Stream driver output during job
gcloud dataproc jobs wait JOB_ID --region=us-central1

# Get job driver log from GCS (after completion)
gsutil cat gs://dataproc-staging-us-central1-PROJ/google-cloud-dataproc-metainfo/*/jobs/JOB_ID/driveroutput*

# Query job logs in Cloud Logging
gcloud logging read \
  'resource.type="cloud_dataproc_job"
   resource.labels.job_id="JOB_ID"
   severity>=WARNING' \
  --project=MY_PROJECT \
  --limit=100

# YARN logs via SSH (for deeper debugging)
gcloud compute ssh CLUSTER-m --zone=us-central1-a --command=\
  "yarn logs -applicationId APPLICATION_ID 2>&1 | head -500"
```

---

## 4. Dataproc Serverless (Spark)

### What is Dataproc Serverless?

Dataproc Serverless provisions compute **on-demand per batch job** — no cluster creation, no idle cost, no capacity planning. You submit a PySpark or Spark job and pay only for execution time.

---

### Submitting Serverless Batch Jobs

```bash
# ── PySpark serverless batch ──────────────────────────────────────────
gcloud dataproc batches submit pyspark gs://bucket/my_job.py \
  --region=us-central1 \
  --batch-id=my-batch-001 \                        # Optional custom ID
  --subnet=projects/proj/regions/us-central1/subnetworks/default \
  --version=2.1 \                                  # Runtime version
  --container-image=gcr.io/proj/my-custom-image \  # Optional custom container
  --service-account=sa@proj.iam.gserviceaccount.com \
  --deps-bucket=gs://bucket/deps \                 # For staging dependencies
  --py-files=gs://bucket/utils.py \
  --properties=spark.executor.cores=4,spark.executor.memory=8g \
  --labels=env=prod,job-type=etl \
  -- \
  --input=gs://bucket/input \
  --output=gs://bucket/output

# ── Spark (JAR) serverless batch ─────────────────────────────────────
gcloud dataproc batches submit spark \
  --region=us-central1 \
  --class=com.example.MainClass \
  --jars=gs://bucket/my-job.jar \
  --properties=spark.executor.memory=16g \
  -- arg1 arg2

# ── List and describe batches ─────────────────────────────────────────
gcloud dataproc batches list --region=us-central1
gcloud dataproc batches describe BATCH_ID --region=us-central1
gcloud dataproc batches cancel BATCH_ID --region=us-central1
gcloud dataproc batches delete BATCH_ID --region=us-central1
```

---

### Custom Container for Serverless

```dockerfile
# Dockerfile — extend the Dataproc Serverless base image
FROM gcr.io/dataproc-batches/dataproc-pyspark:2.1-debian11

# Install additional Python packages
RUN pip install --no-cache-dir \
    pandas==2.0.3 \
    pyarrow==14.0.0 \
    scikit-learn==1.3.0 \
    google-cloud-bigquery==3.14.0

# Copy custom modules
COPY my_utils/ /opt/custom/my_utils/
ENV PYTHONPATH="/opt/custom:${PYTHONPATH}"
```

```bash
# Build and push
docker build -t gcr.io/my-project/my-pyspark:latest .
docker push gcr.io/my-project/my-pyspark:latest

# Use custom container in batch submission
gcloud dataproc batches submit pyspark gs://bucket/job.py \
  --container-image=gcr.io/my-project/my-pyspark:latest \
  --region=us-central1
```

---

### Serverless Networking Requirements

```bash
# Required: Private Google Access must be enabled on the subnet
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-private-ip-google-access

# Required: No external IPs on batch nodes (serverless is always private)
# Serverless batch uses the subnet's secondary IP ranges for pods

# Firewall rule to allow serverless batch internal communication
gcloud compute firewall-rules create allow-dataproc-serverless \
  --network=my-vpc \
  --allow=tcp,udp,icmp \
  --source-ranges=10.0.0.0/8 \
  --target-tags=dataproc
```

---

### Serverless vs. Cluster-Based Comparison

| Dimension | Cluster-Based | Dataproc Serverless |
|---|---|---|
| **Startup time** | ~90 seconds | ~2–4 minutes |
| **Management** | Cluster lifecycle | None |
| **Idle cost** | ⚠️ Billed when idle | ✅ Zero idle cost |
| **HDFS** | ✅ Available | ❌ Not available |
| **Persistent state** | ✅ Between jobs | ❌ Per-job only |
| **Interactive** | ✅ SSH, Jupyter | ✅ Vertex AI Workbench |
| **Custom deps** | Init actions | Custom container |
| **Hive/Presto** | ✅ Supported | ❌ Not supported |
| **Billing unit** | vCPU-hr + GCE | DCU-hr + shuffle |
| **Best for** | Long-running, Hive, interactive | Ad-hoc batch, infrequent jobs |

> 💡 **Choose Serverless when:** jobs run infrequently (< 8h/day), you don't need HDFS or Hive, and you want zero cluster management. For continuous/daily pipelines, ephemeral clusters are often cheaper.

---

## 5. Storage & Data Access Patterns

### GCS Connector

Dataproc clusters include the **GCS connector** (`gcs-connector`), which maps `gs://` URIs to GCS objects transparently in Spark, Hive, and Hadoop.

```python
# Spark reads GCS natively — no extra config needed
df = spark.read.parquet("gs://my-bucket/data/events/dt=2026-03-16/")

# Write back to GCS
df.write.mode("overwrite").parquet("gs://my-bucket/output/results/")

# Configure GCS connector behavior (in cluster properties or job properties)
# spark:spark.hadoop.fs.gs.implicit.dir.repair.enable=false
# spark:spark.hadoop.fs.gs.reporting.metrics.enable=true
```

---

### Reading/Writing Common Formats

```python
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("FormatExamples").getOrCreate()

GCS = "gs://my-bucket"

# Parquet (preferred — columnar, compressed, schema-embedded)
df = spark.read.parquet(f"{GCS}/data/*.parquet")
df.write.mode("overwrite").snappy.parquet(f"{GCS}/output/")

# Avro (good for schema evolution)
df = spark.read.format("avro").load(f"{GCS}/data/")
df.write.format("avro").mode("append").save(f"{GCS}/output/")

# ORC (Hive-optimized)
df = spark.read.orc(f"{GCS}/data/")
df.write.orc(f"{GCS}/output/")

# CSV
df = spark.read.csv(f"{GCS}/data/*.csv", header=True, inferSchema=True)
df.write.csv(f"{GCS}/output/", header=True, mode="overwrite")

# JSON (newline-delimited)
df = spark.read.json(f"{GCS}/data/*.json")
df.write.json(f"{GCS}/output/", mode="overwrite")

# Delta Lake (requires delta-core JAR)
df = spark.read.format("delta").load(f"{GCS}/delta/events/")
df.write.format("delta").mode("append").save(f"{GCS}/delta/events/")

# Apache Iceberg (requires iceberg-spark runtime JAR)
spark.sql("""
  CREATE TABLE IF NOT EXISTS my_catalog.my_db.events
  USING iceberg
  LOCATION 'gs://my-bucket/iceberg/events'
  AS SELECT * FROM staging_events
""")
df = spark.table("my_catalog.my_db.events")
```

---

### BigQuery Connector

```python
# Read from BigQuery — uses Storage Read API for high-throughput
df = spark.read.format("bigquery") \
    .option("table", "my-project.dataset.my_table") \
    .option("filter", "dt = '2026-03-16'") \   # Push-down filter
    .load()

# Alternative: read via SQL
df = spark.read.format("bigquery") \
    .option("query", """
        SELECT user_id, SUM(amount) AS total
        FROM `my-project.dataset.sales`
        WHERE dt >= '2026-01-01'
        GROUP BY user_id
    """) \
    .option("materializationDataset", "my_temp_dataset") \
    .load()

# Write to BigQuery
df.write.format("bigquery") \
    .option("table", "my-project.dataset.output") \
    .option("temporaryGcsBucket", "my-temp-bucket") \
    .option("writeMethod", "indirect") \              # or "direct" (Storage Write API)
    .mode("overwrite") \
    .save()
```

```bash
# Enable BigQuery connector via init action at cluster creation
gcloud dataproc clusters create my-cluster \
  --initialization-actions=\
    gs://goog-dataproc-initialization-actions-us-central1/connectors/connectors.sh \
  --metadata=bigquery-connector-version=1.3.0,\
             spark-bigquery-connector-version=0.34.0
```

---

### Dataproc Metastore

**Dataproc Metastore** is a managed Hive Metastore service that persists table metadata independently of cluster lifecycle.

```bash
# Create a Dataproc Metastore service
gcloud metastore services create my-metastore \
  --location=us-central1 \
  --tier=DEVELOPER \                             # or ENTERPRISE for HA
  --hive-metastore-version=3.1.2 \
  --network=my-vpc

# Attach cluster to Metastore
gcloud dataproc clusters create my-cluster \
  --region=us-central1 \
  --dataproc-metastore=projects/proj/locations/us-central1/services/my-metastore

# Now Hive tables persist after cluster deletion
# BigQuery and Dataflow can also query the same Metastore
```

---

### Apache Iceberg on Dataproc

```bash
# Enable Iceberg at cluster creation
gcloud dataproc clusters create my-cluster \
  --properties=\
"spark:spark.jars.packages=org.apache.iceberg:iceberg-spark-runtime-3.5_2.12:1.4.3,\
spark:spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions,\
spark:spark.sql.catalog.my_catalog=org.apache.iceberg.spark.SparkCatalog,\
spark:spark.sql.catalog.my_catalog.type=hive,\
spark:spark.sql.catalog.my_catalog.uri=thrift://METASTORE_HOST:9083,\
spark:spark.sql.catalog.my_catalog.warehouse=gs://my-bucket/iceberg-warehouse"
```

```python
# Iceberg table operations
spark.sql("""
    CREATE TABLE my_catalog.db.events (
        event_id   STRING,
        user_id    STRING,
        amount     DOUBLE,
        event_date DATE
    )
    USING iceberg
    PARTITIONED BY (days(event_date))
    TBLPROPERTIES ('write.format.default' = 'parquet')
""")

# Time travel
spark.read.option("as-of-timestamp", "2026-03-01 00:00:00") \
    .format("iceberg") \
    .load("my_catalog.db.events")

# Schema evolution
spark.sql("ALTER TABLE my_catalog.db.events ADD COLUMN region STRING")

# Compaction (merge small files)
spark.sql("CALL my_catalog.system.rewrite_data_files('db.events')")
```

---

## 6. Autoscaling & Cost Optimization

### Autoscaling Policies

Dataproc autoscaling uses **YARN pending memory** to decide when to add/remove workers.

```yaml
# autoscaling-policy.yaml
workerConfig:
  minInstances: 2
  maxInstances: 50
  weight: 1
secondaryWorkerConfig:
  minInstances: 0
  maxInstances: 100
  weight: 1
basicAlgorithm:
  yarnConfig:
    scaleUpFactor: 1.0          # Add 100% of needed workers at once
    scaleDownFactor: 1.0        # Remove 100% of excess workers at once
    scaleUpMinWorkerFraction: 0.0
    scaleDownMinWorkerFraction: 0.0
    gracefulDecommissionTimeout: 1h   # Wait for tasks to finish before removing
  cooldownPeriod: 5m            # Minimum time between scaling events
```

```bash
# Create autoscaling policy
gcloud dataproc autoscaling-policies import my-policy \
  --source=autoscaling-policy.yaml \
  --region=us-central1

# Attach to cluster
gcloud dataproc clusters create my-cluster \
  --autoscaling-policy=my-policy \
  --region=us-central1

# Update policy
gcloud dataproc autoscaling-policies export my-policy \
  --destination=policy.yaml \
  --region=us-central1
# Edit policy.yaml ...
gcloud dataproc autoscaling-policies import my-policy \
  --source=policy.yaml \
  --region=us-central1
```

**Autoscaling policy fields:**

| Field | Description | Recommended |
|---|---|---|
| `scaleUpFactor` | Fraction of needed workers to add | 1.0 (aggressive) |
| `scaleDownFactor` | Fraction of excess workers to remove | 1.0 |
| `minInstances` | Minimum worker count | ≥ 2 |
| `maxInstances` | Maximum worker count | Set budget ceiling |
| `cooldownPeriod` | Min time between scaling events | 5–10m |
| `gracefulDecommissionTimeout` | Wait for task completion before removing | 1h |

---

### Secondary Workers (Preemptible / Spot)

```bash
# Add Spot secondary workers (60–91% cheaper than regular VMs)
gcloud dataproc clusters create my-cluster \
  --num-workers=4 \                              # Always primary workers
  --num-secondary-workers=16 \                   # Spot/preemptible workers
  --secondary-worker-type=spot \                 # spot or preemptible

# Secondary worker limitations:
# ❌ No HDFS DataNode role (data on GCS is fine)
# ❌ Cannot be sole workers (always need primary workers)
# ⚠️ Can be preempted at any time — use with checkpointing
```

> ⚠️ Secondary workers are **not counted toward primary worker HDFS** — always use GCS for data storage when using secondary workers.

---

### Ephemeral Cluster Pattern (Most Cost-Efficient for Batch)

```bash
#!/bin/bash
# ephemeral-etl.sh — Create cluster, run job, delete cluster

PROJECT=my-project
REGION=us-central1
CLUSTER=etl-$(date +%Y%m%d%H%M%S)
BUCKET=gs://my-bucket

# 1. Create cluster
gcloud dataproc clusters create $CLUSTER \
  --region=$REGION \
  --single-node \                               # For small jobs
  --image-version=2.2 \
  --num-secondary-workers=8 \
  --secondary-worker-type=spot \
  --max-idle=30m \
  --project=$PROJECT

# 2. Submit job
JOB_ID=$(gcloud dataproc jobs submit pyspark $BUCKET/my_job.py \
  --cluster=$CLUSTER \
  --region=$REGION \
  --format="value(reference.jobId)" \
  -- --input=$BUCKET/input --output=$BUCKET/output)

# 3. Wait for job
gcloud dataproc jobs wait $JOB_ID --region=$REGION

# 4. Delete cluster
gcloud dataproc clusters delete $CLUSTER --region=$REGION --quiet
```

---

### Cluster Auto-Deletion

```bash
# Auto-delete cluster after 2 hours of inactivity
gcloud dataproc clusters create my-cluster \
  --max-idle=2h \
  --region=us-central1

# Auto-delete cluster after maximum 8 hours regardless of activity
gcloud dataproc clusters create my-cluster \
  --max-age=8h \
  --region=us-central1

# Both combined
gcloud dataproc clusters create my-cluster \
  --max-idle=1h \
  --max-age=6h \
  --region=us-central1
```

---

### Machine Type Selection

| Workload | Recommended Machine | Reason |
|---|---|---|
| **Spark (general)** | `n2-standard-8` / `n2-highmem-8` | Balance CPU/memory |
| **Spark (memory-intensive)** | `n2-highmem-16` / `n2-highmem-32` | Large in-memory datasets |
| **Presto / Trino** | `n2-highcpu-16` | Query-intensive, less memory |
| **ML training (GPU)** | `n1-standard-8` + `nvidia-tesla-t4` | GPU acceleration |
| **Master (Standard)** | `n2-standard-4` | Lightweight coordination |
| **Master (HA)** | `n2-standard-8` | Handles ZK + NameNode |

---

### 💰 Cost Comparison

| Strategy | Monthly Cost | Tradeoffs |
|---|---|---|
| **Persistent cluster (4 workers, 24/7)** | ~$800–1,200 | Always available; always billed |
| **Ephemeral cluster (8h/day, 22 days)** | ~$250–400 | Only billed when running |
| **Ephemeral + Spot workers (75% spot)** | ~$80–150 | Cheapest; preemption risk |
| **Dataproc Serverless (infrequent jobs)** | Pay per job (~$0.10/DCU-hr) | Best for < 1h/day total usage |

---

## 7. Workflow Templates & Orchestration

### Workflow Templates Overview

Workflow templates define a **DAG of Dataproc jobs** that run on a managed (auto-created) or existing cluster. They handle cluster lifecycle automatically.

```bash
# Create a workflow template
gcloud dataproc workflow-templates create my-etl-workflow \
  --region=us-central1

# Set managed cluster config (cluster created/deleted per workflow run)
gcloud dataproc workflow-templates set-managed-cluster my-etl-workflow \
  --region=us-central1 \
  --cluster-name=wf-cluster \
  --single-node \
  --master-machine-type=n2-standard-8 \
  --image-version=2.2

# Add jobs to the workflow
gcloud dataproc workflow-templates add-job pyspark \
  --workflow-template=my-etl-workflow \
  --region=us-central1 \
  --step-id=extract \
  --file=gs://bucket/extract.py \
  -- --input=gs://bucket/raw --output=gs://bucket/staging

gcloud dataproc workflow-templates add-job pyspark \
  --workflow-template=my-etl-workflow \
  --region=us-central1 \
  --step-id=transform \
  --start-after=extract \                        # Depends on extract step
  --file=gs://bucket/transform.py \
  -- --input=gs://bucket/staging --output=gs://bucket/processed

gcloud dataproc workflow-templates add-job hive \
  --workflow-template=my-etl-workflow \
  --region=us-central1 \
  --step-id=load \
  --start-after=transform \
  --file=gs://bucket/load.hql

# Instantiate (run) the workflow
gcloud dataproc workflow-templates instantiate my-etl-workflow \
  --region=us-central1

# Instantiate with parameter overrides
gcloud dataproc workflow-templates instantiate my-etl-workflow \
  --region=us-central1 \
  --parameters=INPUT_DATE=2026-03-16,ENV=prod
```

---

### Parameterized Workflow Templates (YAML)

```yaml
# workflow-template.yaml
id: parameterized-etl
placement:
  managedCluster:
    clusterName: wf-cluster-${date}
    config:
      masterConfig:
        numInstances: 1
        machineTypeUri: n2-standard-8
      workerConfig:
        numInstances: 4
        machineTypeUri: n2-highmem-8
      softwareConfig:
        imageVersion: "2.2"
        properties:
          spark:spark.sql.shuffle.partitions: "400"
jobs:
  - stepId: extract
    pysparkJob:
      mainPythonFileUri: gs://bucket/extract.py
      args:
        - --input=gs://bucket/raw/${date}/
        - --output=gs://bucket/staging/${date}/
  - stepId: transform
    prerequisiteStepIds: [extract]
    pysparkJob:
      mainPythonFileUri: gs://bucket/transform.py
      args:
        - --input=gs://bucket/staging/${date}/
        - --output=gs://bucket/processed/dt=${date}/
parameters:
  - name: date
    fields: [jobs[0].pysparkJob.args[0], jobs[0].pysparkJob.args[1],
             jobs[1].pysparkJob.args[0], jobs[1].pysparkJob.args[1]]
    description: Processing date (YYYY-MM-DD)
```

```bash
# Import and run the parameterized template
gcloud dataproc workflow-templates import my-etl \
  --source=workflow-template.yaml \
  --region=us-central1

gcloud dataproc workflow-templates instantiate my-etl \
  --region=us-central1 \
  --parameters=date=2026-03-16
```

---

### Cloud Composer (Airflow) Integration

```python
# Airflow DAG with Dataproc operators
from airflow import DAG
from airflow.providers.google.cloud.operators.dataproc import (
    DataprocCreateClusterOperator,
    DataprocSubmitJobOperator,
    DataprocDeleteClusterOperator,
)
from airflow.utils.dates import days_ago
from datetime import timedelta

CLUSTER_CONFIG = {
    "master_config": {"num_instances": 1, "machine_type_uri": "n2-standard-8"},
    "worker_config": {"num_instances": 4, "machine_type_uri": "n2-highmem-8"},
    "software_config": {"image_version": "2.2"},
}

PYSPARK_JOB = {
    "reference": {"project_id": "my-project"},
    "placement": {"cluster_name": "airflow-cluster-{{ ds_nodash }}"},
    "pyspark_job": {
        "main_python_file_uri": "gs://bucket/my_job.py",
        "args": ["--date={{ ds }}", "--output=gs://bucket/output/"],
    },
}

with DAG(
    dag_id="dataproc_etl",
    schedule_interval="0 6 * * *",
    start_date=days_ago(1),
    catchup=False,
    default_args={"retries": 2, "retry_delay": timedelta(minutes=5)},
) as dag:

    create_cluster = DataprocCreateClusterOperator(
        task_id="create_cluster",
        project_id="my-project",
        cluster_config=CLUSTER_CONFIG,
        region="us-central1",
        cluster_name="airflow-cluster-{{ ds_nodash }}",
    )

    submit_job = DataprocSubmitJobOperator(
        task_id="run_pyspark",
        job=PYSPARK_JOB,
        region="us-central1",
        project_id="my-project",
    )

    delete_cluster = DataprocDeleteClusterOperator(
        task_id="delete_cluster",
        project_id="my-project",
        cluster_name="airflow-cluster-{{ ds_nodash }}",
        region="us-central1",
        trigger_rule="all_done",  # Delete even if job failed
    )

    create_cluster >> submit_job >> delete_cluster
```

---

## 8. Spark on Dataproc — Deep Dive

### Driver Placement: YARN Modes

| Mode | Driver Location | Use Case |
|---|---|---|
| **YARN client** (default) | Local machine / master node | Interactive, debugging (logs stream to client) |
| **YARN cluster** | Worker node | Production batch (driver resilient to client failure) |

```bash
# YARN cluster mode (recommended for production)
gcloud dataproc jobs submit pyspark gs://bucket/job.py \
  --cluster=my-cluster \
  --region=us-central1 \
  --properties=spark.submit.deployMode=cluster
```

---

### Key Spark Configurations for Dataproc

```bash
# Recommended Spark properties for n2-highmem-8 workers (8 vCPU, 64 GB RAM)
gcloud dataproc clusters create my-cluster \
  --worker-machine-type=n2-highmem-8 \
  --properties=\
"spark:spark.executor.cores=4,\
spark:spark.executor.memory=24g,\
spark:spark.executor.memoryOverhead=4g,\
spark:spark.driver.memory=8g,\
spark:spark.driver.memoryOverhead=2g,\
spark:spark.dynamicAllocation.enabled=true,\
spark:spark.dynamicAllocation.minExecutors=1,\
spark:spark.dynamicAllocation.maxExecutors=100,\
spark:spark.dynamicAllocation.initialExecutors=4,\
spark:spark.sql.shuffle.partitions=400,\
spark:spark.sql.adaptive.enabled=true,\
spark:spark.sql.adaptive.coalescePartitions.enabled=true,\
spark:spark.sql.adaptive.skewJoin.enabled=true,\
yarn:yarn.nodemanager.resource.memory-mb=58368,\
yarn:yarn.nodemanager.resource.cpu-vcores=7"
```

**Key configuration parameters:**

| Parameter | Description | Recommended Value |
|---|---|---|
| `spark.executor.memory` | Heap memory per executor | 70–75% of worker RAM / executors |
| `spark.executor.memoryOverhead` | Off-heap memory (JVM overhead) | 10–15% of executor memory |
| `spark.executor.cores` | vCPUs per executor | 4 (sweet spot for parallelism vs. HDFS) |
| `spark.dynamicAllocation.enabled` | Auto-scale executor count | `true` |
| `spark.sql.shuffle.partitions` | Partitions for joins/aggregations | 2–3× worker vCPU count |
| `spark.sql.adaptive.enabled` | Adaptive Query Execution | `true` (default in Spark 3.x) |
| `spark.memory.fraction` | JVM heap for execution + storage | `0.6` (default) |
| `spark.memory.storageFraction` | Fraction of above for caching | `0.5` (default) |

---

### PySpark DataFrame API Examples

```python
from pyspark.sql import SparkSession, Window
from pyspark.sql import functions as F
from pyspark.sql.types import StructType, StructField, StringType, DoubleType

spark = SparkSession.builder \
    .appName("SparkDeepDive") \
    .config("spark.sql.adaptive.enabled", "true") \
    .getOrCreate()

# ── Read with explicit schema ─────────────────────────────────────────
schema = StructType([
    StructField("user_id",    StringType(), False),
    StructField("amount",     DoubleType(), True),
    StructField("event_date", StringType(), True),
])

df = spark.read.schema(schema).parquet("gs://bucket/events/")

# ── Broadcast join (small table → large table) ────────────────────────
users = spark.read.parquet("gs://bucket/users/")       # Small table
events = spark.read.parquet("gs://bucket/events/")     # Large table

enriched = events.join(F.broadcast(users), "user_id")  # Avoids shuffle

# ── Window functions ──────────────────────────────────────────────────
window = Window.partitionBy("user_id").orderBy("event_date")
result = df.withColumn("cumulative_spend", F.sum("amount").over(window)) \
           .withColumn("rank",             F.rank().over(window))

# ── Aggregation with multiple aggs ───────────────────────────────────
summary = df.groupBy("user_id", "event_date").agg(
    F.sum("amount").alias("total"),
    F.count("*").alias("txn_count"),
    F.max("amount").alias("max_amt"),
    F.approx_count_distinct("product_id").alias("unique_products")
)

# ── Repartition before write to avoid small files ────────────────────
result \
    .repartition(100, "event_date") \
    .write \
    .partitionBy("event_date") \
    .mode("overwrite") \
    .parquet("gs://bucket/output/")

# ── Cache a reused dataset ────────────────────────────────────────────
df.cache()
count = df.count()           # Triggers caching
filtered = df.filter(...)    # Uses cache
df.unpersist()               # Release when done
```

---

### SparkSQL Examples

```sql
-- Register temp view
-- spark.sql("CREATE OR REPLACE TEMP VIEW events AS SELECT * FROM parquet.`gs://bucket/events/`")

-- CTE with window function
WITH ranked_users AS (
  SELECT
    user_id,
    SUM(amount)                                        AS total_spend,
    COUNT(*)                                           AS txn_count,
    RANK() OVER (ORDER BY SUM(amount) DESC)            AS spend_rank,
    PERCENT_RANK() OVER (ORDER BY SUM(amount))         AS percentile
  FROM events
  WHERE event_date >= '2026-01-01'
  GROUP BY user_id
)
SELECT *
FROM ranked_users
WHERE spend_rank <= 100
ORDER BY spend_rank;

-- CREATE TABLE backed by GCS
CREATE TABLE IF NOT EXISTS analytics.daily_summary
USING PARQUET
PARTITIONED BY (event_date)
LOCATION 'gs://my-bucket/tables/daily_summary/'
AS
SELECT
  event_date,
  user_id,
  SUM(amount)  AS revenue,
  COUNT(*)     AS orders
FROM events
GROUP BY event_date, user_id;
```

---

### Spark Structured Streaming on Dataproc

```python
# Stream from Pub/Sub → GCS
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = SparkSession.builder \
    .appName("PubSubStreaming") \
    .config("spark.jars", "gs://spark-lib/pubsub/spark-pubsub-connector.jar") \
    .getOrCreate()

# Read from Pub/Sub
df = spark.readStream \
    .format("pubsub") \
    .option("pubsub.project.id", "my-project") \
    .option("pubsub.subscription.id", "my-sub") \
    .load()

# Parse JSON payload
parsed = df.select(
    F.from_json(F.col("payload").cast("string"),
                "user_id STRING, amount DOUBLE, ts TIMESTAMP").alias("data")
).select("data.*")

# Windowed aggregation
windowed = parsed \
    .withWatermark("ts", "5 minutes") \
    .groupBy(F.window("ts", "1 minute"), "user_id") \
    .agg(F.sum("amount").alias("total"))

# Write to GCS (checkpoint required for exactly-once)
query = windowed.writeStream \
    .format("parquet") \
    .option("path", "gs://bucket/streaming-output/") \
    .option("checkpointLocation", "gs://bucket/checkpoints/pubsub-job/") \
    .partitionBy("window") \
    .trigger(processingTime="30 seconds") \
    .start()

query.awaitTermination()
```

---

### Performance Tuning: AQE and Skew

```python
# Enable Adaptive Query Execution (Spark 3.x, default on Dataproc 2.x)
spark.conf.set("spark.sql.adaptive.enabled",                      "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled",   "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled",             "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor","5")
spark.conf.set("spark.sql.adaptive.advisoryPartitionSizeInBytes", "128m")

# Handle data skew manually with salting
from pyspark.sql import functions as F
import random

NUM_SALTS = 50

# Salt the skewed key on the left side
left_salted = large_df.withColumn(
    "salt_key",
    F.concat(F.col("skewed_key"), F.lit("_"), (F.rand() * NUM_SALTS).cast("int").cast("string"))
)

# Explode the right side to match all salt values
right_exploded = small_df.withColumn("salt", F.explode(F.array([F.lit(i) for i in range(NUM_SALTS)]))) \
    .withColumn("salt_key", F.concat(F.col("skewed_key"), F.lit("_"), F.col("salt").cast("string")))

result = left_salted.join(right_exploded, "salt_key")
```

---

## 9. Security & Networking

### IAM Roles

| Role | Who Needs It | Grants |
|---|---|---|
| `roles/dataproc.admin` | Ops / platform team | Create, delete, update clusters and jobs |
| `roles/dataproc.editor` | Data engineers | Submit jobs, manage clusters (no delete) |
| `roles/dataproc.viewer` | Analysts / auditors | View cluster/job status and logs |
| `roles/dataproc.worker` | **Worker SA only** | Required on the VM service account |

```bash
# Grant data engineer access
gcloud projects add-iam-policy-binding my-project \
  --member="user:engineer@company.com" \
  --role="roles/dataproc.editor"

# Grant worker SA the required roles
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:worker-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/dataproc.worker"

gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:worker-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataEditor"

gsutil iam ch \
  serviceAccount:worker-sa@my-project.iam.gserviceaccount.com:roles/storage.objectAdmin \
  gs://my-data-bucket
```

---

### Service Account Architecture

```
JOB SUBMISSION                   CLUSTER NODES
      │                               │
      ▼                               ▼
  User / SA                       Worker SA
  (submits via gcloud/API)      (runs on GCE VMs)
      │                               │
  Needs:                          Needs:
  dataproc.editor                   dataproc.worker
  iam.serviceAccountUser            storage.objectAdmin (GCS)
  (on worker SA)                    bigquery.dataEditor (BQ)
                                    pubsub.subscriber (Pub/Sub)
                                    bigtable.user (Bigtable)
```

---

### Private Cluster (No External IPs)

```bash
# Create cluster with private IPs (no external addresses)
gcloud dataproc clusters create my-private-cluster \
  --region=us-central1 \
  --subnet=projects/proj/regions/us-central1/subnetworks/my-private-subnet \
  --no-address \                                   # No external IPs
  --enable-component-gateway \                     # Secure web UI access
  --service-account=worker-sa@proj.iam.gserviceaccount.com

# Required: Private Google Access on subnet
gcloud compute networks subnets update my-private-subnet \
  --region=us-central1 \
  --enable-private-ip-google-access

# Required: Firewall rules for inter-node communication
gcloud compute firewall-rules create dataproc-internal \
  --network=my-vpc \
  --allow=tcp:0-65535,udp:0-65535,icmp \
  --source-ranges=10.0.0.0/8 \
  --target-tags=dataproc
```

---

### CMEK with Cloud KMS

```bash
# Create a KMS keyring and key
gcloud kms keyrings create dataproc-keyring \
  --location=us-central1

gcloud kms keys create dataproc-key \
  --location=us-central1 \
  --keyring=dataproc-keyring \
  --purpose=encryption

# Grant Dataproc service account access to key
gcloud kms keys add-iam-policy-binding dataproc-key \
  --location=us-central1 \
  --keyring=dataproc-keyring \
  --member="serviceAccount:service-PROJECT_NUMBER@dataproc-accounts.iam.gserviceaccount.com" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Create CMEK-encrypted cluster
gcloud dataproc clusters create my-encrypted-cluster \
  --region=us-central1 \
  --gce-pd-kms-key=projects/my-project/locations/us-central1/keyRings/dataproc-keyring/cryptoKeys/dataproc-key
```

---

### Kerberos (Enterprise Security)

```bash
# Enable Kerberos on cluster creation
gcloud dataproc clusters create my-kerberos-cluster \
  --region=us-central1 \
  --kerberos-config-file=gs://bucket/kerberos-config.yaml

# kerberos-config.yaml
# kerberosConfig:
#   enableKerberos: true
#   rootPrincipalPasswordUri: gs://bucket/kerberos-root-password.encrypted
#   kmsKeyUri: projects/proj/locations/us/keyRings/kr/cryptoKeys/key
#   realm: DATAPROC.EXAMPLE.COM
#   kdcDbKeyUri: gs://bucket/kdc-db-key.encrypted
#   tgtLifetimeHours: 24
```

---

### Component Gateway (Secure Web UIs)

```bash
# Enable Component Gateway (no SSH tunnels needed)
gcloud dataproc clusters create my-cluster \
  --enable-component-gateway \
  --optional-components=JUPYTER,PRESTO \
  --region=us-central1

# Get Component Gateway URLs
gcloud dataproc clusters describe my-cluster \
  --region=us-central1 \
  --format="json(config.endpointConfig)"

# Outputs:
# Spark History Server:    https://...dataproc.googleusercontent.com/gateway/...
# YARN ResourceManager:   https://...
# Jupyter Notebook:       https://...
# Zeppelin:               https://...
```

---

## 10. Monitoring, Logging & Debugging

### Key Cloud Monitoring Metrics

| Metric | Description | Alert On |
|---|---|---|
| `dataproc.googleapis.com/cluster/yarn/memory_size` | YARN total memory (MB) | — |
| `dataproc.googleapis.com/cluster/yarn/memory_mb_available` | Available YARN memory | < 10% |
| `dataproc.googleapis.com/cluster/yarn/nodemanagers_active` | Active YARN NodeManagers | < min_workers |
| `dataproc.googleapis.com/cluster/yarn/apps_running` | Running YARN applications | — |
| `dataproc.googleapis.com/cluster/yarn/apps_failed` | Failed YARN applications | > 0 |
| `dataproc.googleapis.com/cluster/hdfs/capacity_used_percentage` | HDFS usage % | > 80% |
| `dataproc.googleapis.com/job/state` | Job state (1=running, 0=not) | Unexpected state changes |

```bash
# Create alert for YARN memory pressure
gcloud alpha monitoring policies create \
  --display-name="Dataproc Low YARN Memory" \
  --condition-filter='
    metric.type="dataproc.googleapis.com/cluster/yarn/memory_mb_available"
    resource.labels.cluster_name="my-cluster"' \
  --condition-threshold-value=4096 \
  --condition-threshold-comparison=COMPARISON_LT \
  --condition-duration=300s \
  --notification-channels=CHANNEL_ID
```

---

### Cloud Logging Queries

```bash
# Job driver logs (all)
gcloud logging read \
  'resource.type="cloud_dataproc_job"
   resource.labels.job_id="JOB_ID"' \
  --project=MY_PROJECT --limit=200

# Cluster-level logs (YARN, HDFS)
gcloud logging read \
  'resource.type="cloud_dataproc_cluster"
   resource.labels.cluster_name="MY_CLUSTER"
   severity>=WARNING' \
  --project=MY_PROJECT --limit=100

# Find OOM errors
gcloud logging read \
  'resource.type="cloud_dataproc_job"
   textPayload:"OutOfMemoryError"' \
  --project=MY_PROJECT --limit=50

# GCS throttling errors
gcloud logging read \
  'resource.type="cloud_dataproc_job"
   textPayload:"rateLimitExceeded"' \
  --project=MY_PROJECT --limit=50

# All errors in last 24h
gcloud logging read \
  'resource.type="cloud_dataproc_job"
   severity=ERROR
   timestamp>="2026-03-15T00:00:00Z"' \
  --project=MY_PROJECT
```

---

### Common Failure Modes & Fixes

| Failure | Symptoms | Fix |
|---|---|---|
| **Executor OOM** | `java.lang.OutOfMemoryError: Java heap space` | Increase `spark.executor.memory`; reduce `spark.executor.cores` |
| **Driver OOM** | Driver killed; job fails with OOM | Increase `spark.driver.memory`; avoid `collect()` on large datasets |
| **Shuffle OOM** | `ExternalAppendOnlyMap` errors | Increase `spark.executor.memoryOverhead`; enable `spark.sql.adaptive` |
| **GCS throttling** | `rateLimitExceeded` in logs | Reduce parallelism; use exponential backoff; spread writes |
| **Too many small files** | Slow reads; excessive GCS API calls | Coalesce before write; use `repartition(N)` |
| **Data skew** | Most tasks fast, a few extremely slow | Enable AQE skew join; salt keys manually |
| **YARN preemption** | Spot workers lost mid-job | Enable checkpointing; use `--gracefulDecommissionTimeout` |
| **Shuffle failures** | `FetchFailed` exception | Increase `spark.reducer.maxReqsInFlight`; add local SSDs for shuffle |

---

### Debugging OOM — Decision Tree

```
Job fails with OOM?
    │
    ├── Driver OOM?
    │       └── Fix: increase spark.driver.memory; avoid collect()
    │
    └── Executor OOM?
            │
            ├── Heap space?
            │       └── Fix: increase spark.executor.memory
            │
            ├── Off-heap / GC overhead?
            │       └── Fix: increase spark.executor.memoryOverhead (min 10% of executor memory)
            │
            └── Shuffle / sort OOM?
                    └── Fix: enable AQE; increase spark.sql.shuffle.partitions;
                             add local SSDs for faster shuffle
```

---

## 11. gcloud CLI & API Quick Reference

### Cluster Commands

```bash
# Create
gcloud dataproc clusters create CLUSTER [flags]

# List
gcloud dataproc clusters list \
  --region=us-central1 \
  --format="table(clusterName,status.state,config.workerConfig.numInstances)"

# Describe
gcloud dataproc clusters describe CLUSTER --region=us-central1

# Update (resize)
gcloud dataproc clusters update CLUSTER \
  --region=us-central1 \
  --num-workers=8 \
  --num-secondary-workers=16

# Delete
gcloud dataproc clusters delete CLUSTER --region=us-central1 --quiet

# Start/Stop (save VM costs, keep cluster config)
gcloud dataproc clusters start  CLUSTER --region=us-central1
gcloud dataproc clusters stop   CLUSTER --region=us-central1

# Diagnose (collect logs for support)
gcloud dataproc clusters diagnose CLUSTER \
  --region=us-central1 \
  --tarball-gcs-dir=gs://bucket/diagnose/
```

---

### Job Commands

```bash
# Submit PySpark
gcloud dataproc jobs submit pyspark FILE \
  --cluster=CLUSTER --region=us-central1 [flags]

# Submit Spark JAR
gcloud dataproc jobs submit spark \
  --cluster=CLUSTER --region=us-central1 \
  --class=com.example.Main --jars=gs://bucket/job.jar

# Submit Hive
gcloud dataproc jobs submit hive \
  --cluster=CLUSTER --region=us-central1 \
  --file=gs://bucket/script.hql

# List jobs
gcloud dataproc jobs list \
  --cluster=CLUSTER --region=us-central1 \
  --state-filter=active \
  --format="table(reference.jobId,status.state,statusHistory[0].stateStartTime)"

# Describe a job
gcloud dataproc jobs describe JOB_ID --region=us-central1

# Wait for job completion (stream logs)
gcloud dataproc jobs wait JOB_ID --region=us-central1

# Cancel a job
gcloud dataproc jobs cancel JOB_ID --region=us-central1
```

---

### Serverless Batch Commands

```bash
# Submit PySpark batch
gcloud dataproc batches submit pyspark FILE \
  --region=us-central1 \
  --version=2.1 \
  --subnet=SUBNET [flags]

# List batches
gcloud dataproc batches list --region=us-central1

# Describe batch
gcloud dataproc batches describe BATCH_ID --region=us-central1

# Cancel batch
gcloud dataproc batches cancel BATCH_ID --region=us-central1

# Delete completed batch record
gcloud dataproc batches delete BATCH_ID --region=us-central1
```

---

### Workflow Template Commands

```bash
# Create
gcloud dataproc workflow-templates create TEMPLATE --region=us-central1

# List
gcloud dataproc workflow-templates list --region=us-central1

# Describe
gcloud dataproc workflow-templates describe TEMPLATE --region=us-central1

# Export to YAML
gcloud dataproc workflow-templates export TEMPLATE \
  --destination=template.yaml --region=us-central1

# Import from YAML
gcloud dataproc workflow-templates import TEMPLATE \
  --source=template.yaml --region=us-central1

# Instantiate (run)
gcloud dataproc workflow-templates instantiate TEMPLATE \
  --region=us-central1 \
  --parameters=key=value

# Delete
gcloud dataproc workflow-templates delete TEMPLATE --region=us-central1
```

---

### Python Client Library

```python
from google.cloud import dataproc_v1 as dataproc
import time

PROJECT = "my-project"
REGION  = "us-central1"

# ── Create a cluster ──────────────────────────────────────────────────
cluster_client = dataproc.ClusterControllerClient(
    client_options={"api_endpoint": f"{REGION}-dataproc.googleapis.com:443"}
)

cluster = {
    "project_id": PROJECT,
    "cluster_name": "python-cluster",
    "config": {
        "master_config": {"num_instances": 1, "machine_type_uri": "n2-standard-8"},
        "worker_config": {"num_instances": 4, "machine_type_uri": "n2-highmem-8"},
        "software_config": {"image_version": "2.2"},
        "gce_cluster_config": {"zone_uri": "us-central1-a"},
    },
}

operation = cluster_client.create_cluster(
    request={"project_id": PROJECT, "region": REGION, "cluster": cluster}
)
result = operation.result()
print(f"Cluster created: {result.cluster_name}")

# ── Submit a PySpark job ──────────────────────────────────────────────
job_client = dataproc.JobControllerClient(
    client_options={"api_endpoint": f"{REGION}-dataproc.googleapis.com:443"}
)

job = {
    "placement": {"cluster_name": "python-cluster"},
    "pyspark_job": {
        "main_python_file_uri": "gs://bucket/my_job.py",
        "args": ["--input=gs://bucket/in", "--output=gs://bucket/out"],
        "properties": {"spark.executor.memory": "4g"},
    },
}

response = job_client.submit_job(
    request={"project_id": PROJECT, "region": REGION, "job": job}
)
job_id = response.reference.job_id
print(f"Submitted job: {job_id}")

# ── Poll job status ───────────────────────────────────────────────────
while True:
    job_status = job_client.get_job(
        request={"project_id": PROJECT, "region": REGION, "job_id": job_id}
    )
    state = job_status.status.state.name
    print(f"Job state: {state}")
    if state in ("DONE", "ERROR", "CANCELLED"):
        break
    time.sleep(10)

# ── Delete cluster ─────────────────────────────────────────────────────
operation = cluster_client.delete_cluster(
    request={"project_id": PROJECT, "region": REGION,
             "cluster_name": "python-cluster"}
)
operation.result()
print("Cluster deleted")
```

---

## 12. Pricing Summary

> ⚠️ Prices are approximate as of early 2026 (us-central1). Verify at [cloud.google.com/dataproc/pricing](https://cloud.google.com/dataproc/pricing).

### Cluster-Based Pricing

Dataproc charges a **Dataproc premium** on top of GCE VM costs:

| Resource | GCE Cost | Dataproc Premium | Total (approx) |
|---|---|---|---|
| **vCPU-hour** | ~$0.031/vCPU-hr | +$0.010/vCPU-hr | ~$0.041/vCPU-hr |
| **GB RAM-hour** | ~$0.0042/GB-hr | +$0.001/GB-hr | ~$0.0052/GB-hr |
| **Preemptible vCPU** | ~$0.0065/vCPU-hr | +$0.001/vCPU-hr | ~$0.0075/vCPU-hr |

### Dataproc Serverless Pricing

| Resource | Price |
|---|---|
| **DCU-hour (compute)** | ~$0.069/DCU-hr |
| **Shuffle storage** | ~$0.04/GB-month |

> 💡 1 DCU ≈ 1 vCPU + 4 GB RAM. A typical PySpark batch using 100 DCUs for 1 hour = ~$6.90.

---

### 💰 Cost Optimization Summary

| Strategy | Savings | How |
|---|---|---|
| **Ephemeral clusters** | 70–90% vs. persistent | Delete cluster after each job |
| **Spot secondary workers** | 60–91% on compute | `--secondary-worker-type=spot` |
| **Autoscaling** | 20–50% | Only use what you need |
| **GCS over HDFS** | Eliminate storage VMs | Use `gs://` URIs everywhere |
| **Right-sizing VMs** | 10–30% | Match RAM to actual Spark usage |
| **`--max-idle` deletion** | Eliminate forgotten clusters | Always set idle timeout |
| **Preemptible masters** | N/A (not supported) | Masters must be non-preemptible |
| **Committed Use Discounts** | 20–57% | For truly persistent clusters |

---

### Monthly Cost Estimate

```
Persistent Cluster (4 n2-highmem-8 workers + 1 master, 24/7):
  Master: 8 vCPU × $0.041 × 730h = $240/month
  Workers: 4 × 8 vCPU × $0.041 × 730h = $959/month
  RAM: (1×64 + 4×64) GB × $0.0052 × 730h = $1,214/month
  Total: ~$2,400/month

Ephemeral Cluster (same config, 8h/day, 22 days/month):
  Compute: 2,400 × (8h × 22 / 730h) = ~$578/month
  (+ Spot secondary workers: -60%) = ~$350/month

Dataproc Serverless (100 DCU average, 4h/day, 22 days):
  100 DCU × $0.069 × 4h × 22 days = ~$608/month
```

---

## 13. Quick Reference & Comparison Tables

### Cluster Type Comparison

| Feature | Standard | Single Node | High Availability |
|---|---|---|---|
| **Masters** | 1 | 1 (also worker) | 3 |
| **Workers** | 2+ | 0 | 2+ |
| **HDFS NameNode** | Active only | Single | Active + Standby |
| **Master failure** | Job fails | Job fails | ✅ Survives 1 failure |
| **Zookeeper** | ❌ | ❌ | ✅ |
| **Use case** | Dev / production | Dev / testing | Production critical |
| **Cost** | Medium | Lowest | Highest (+2 masters) |

---

### Dataproc on GCE vs. GKE vs. Serverless

| Feature | Dataproc on GCE | Dataproc on GKE | Dataproc Serverless |
|---|---|---|---|
| **Infrastructure** | GCE VMs | GKE pods | Auto-provisioned |
| **Management** | Managed VMs | K8s cluster required | Fully serverless |
| **HDFS** | ✅ Available | ❌ | ❌ |
| **Hive / Presto** | ✅ | ⚠️ Limited | ❌ |
| **Multi-tenancy** | Per-cluster | ✅ Namespace isolation | Per-job |
| **Startup time** | ~90s | ~2–3 min | ~2–4 min |
| **Custom containers** | Init actions | ✅ Native K8s | ✅ Custom images |
| **Best for** | Most workloads | K8s-native orgs | Infrequent batch |

---

### Supported Job Types

| Job Type | CLI | Language | Best For |
|---|---|---|---|
| PySpark | `pyspark` | Python | Python-native data engineering |
| Spark | `spark` | Scala / Java | High-performance, complex jobs |
| SparkSQL | `spark-sql` | SQL | SQL-first transformations |
| SparkR | `spark-r` | R | R-based statistical analysis |
| Hive | `hive` | HiveQL | SQL on large GCS-backed tables |
| Pig | `pig` | Pig Latin | Legacy ETL (use Spark instead) |
| MapReduce | `hadoop` | Java | Legacy MapReduce (use Spark instead) |
| Presto | `presto` | SQL | Interactive ad-hoc queries |
| Flink | `flink` | Java / Scala | Stateful stream processing |

---

### Key Spark Configuration Reference

| Config | Default | When to Change |
|---|---|---|
| `spark.executor.memory` | 1g | Always tune to worker RAM |
| `spark.executor.memoryOverhead` | 10% of executor | Increase for Python UDFs |
| `spark.executor.cores` | 1 | Set to 4 for most workloads |
| `spark.driver.memory` | 1g | Increase if collect() used |
| `spark.sql.shuffle.partitions` | 200 | Set to 2–3× total worker vCPUs |
| `spark.dynamicAllocation.enabled` | false | Enable for variable workloads |
| `spark.sql.adaptive.enabled` | true (3.x) | Always keep true |
| `spark.sql.adaptive.skewJoin.enabled` | true (3.x) | Always keep true |
| `spark.memory.fraction` | 0.6 | Reduce if storage-heavy |
| `spark.serializer` | JavaSerializer | Use KryoSerializer for speed |
| `spark.kryoserializer.buffer.max` | 64m | Increase for large objects |

---

### Common Anti-Patterns

| ❌ Anti-Pattern | 🔴 Problem | ✅ Fix |
|---|---|---|
| Using HDFS for primary storage | Data lost on cluster delete | Use `gs://` for all data |
| Persistent cluster running 24/7 | Billed when idle | Use ephemeral + `--max-idle` |
| No secondary workers | Paying full price for burst | Add Spot secondary workers |
| `collect()` on large datasets | Driver OOM | Use `write()` to GCS instead |
| Default `shuffle.partitions=200` | Too few for large data | Set to 2–3× total vCPUs |
| Single-core executors | Underutilizes RAM per core | Use 4 cores per executor |
| No init action timeout | Stuck clusters at creation | Set `--initialization-action-timeout` |
| Not enabling AQE | Suboptimal query plans | Set `spark.sql.adaptive.enabled=true` |
| Too many small output files | Slow downstream reads | `coalesce(N)` before write |
| Not setting `--max-idle` | Forgotten clusters burn money | Always set idle timeout |

---

### Dataproc Image Version → Components (Recent)

| Image | Spark | Hadoop | Hive | Python | Flink |
|---|---|---|---|---|---|
| `2.2-debian12` | 3.5.1 | 3.3.6 | 3.1.3 | 3.11 | 1.17 |
| `2.1-debian11` | 3.3.4 | 3.3.4 | 3.1.3 | 3.10 | 1.15 |
| `2.0-debian10` | 3.1.3 | 3.2.3 | 3.1.2 | 3.8 | 1.14 |

---

### Common Init Actions (Public GCS URIs)

| Init Action | URI | Purpose |
|---|---|---|
| **Connectors** | `gs://goog-dataproc-initialization-actions-REGION/connectors/connectors.sh` | BigQuery, GCS connectors |
| **Conda** | `gs://goog-dataproc-initialization-actions-REGION/conda/bootstrap-conda.sh` | Conda package manager |
| **pip** | `gs://goog-dataproc-initialization-actions-REGION/python/pip-install.sh` | pip packages via metadata |
| **Jupyter** | Via `--optional-components=JUPYTER` | Built-in, prefer component flag |
| **Delta Lake** | Custom (copy to your GCS) | Delta Lake JAR + config |

```bash
# Install pip packages via metadata (using pip init action)
gcloud dataproc clusters create my-cluster \
  --initialization-actions=\
    gs://goog-dataproc-initialization-actions-us-central1/python/pip-install.sh \
  --metadata=PIP_PACKAGES="pandas==2.0.3 scikit-learn==1.3.0 pyarrow==14.0.0"
```

---

*Generated for GCP Dataproc | Official Docs: [cloud.google.com/dataproc/docs](https://cloud.google.com/dataproc/docs) | Apache Spark: [spark.apache.org](https://spark.apache.org)*
