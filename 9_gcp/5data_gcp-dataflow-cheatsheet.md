# 🔄 GCP Dataflow — Comprehensive Cheatsheet

> **Audience:** Data engineers, platform engineers, and architects building batch and streaming pipelines on GCP.
> **Last updated:** March 2026 | Covers Apache Beam SDK, Dataflow Prime, Flex Templates, and all core features.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Core Beam Programming Model](#2-core-beam-programming-model)
3. [Reading & Writing Data (I/O Connectors)](#3-reading--writing-data-io-connectors)
4. [Windowing & Triggers (Streaming)](#4-windowing--triggers-streaming)
5. [State & Timers (Stateful Processing)](#5-state--timers-stateful-processing)
6. [Pipeline Patterns & Best Practices](#6-pipeline-patterns--best-practices)
7. [Dataflow Templates & Deployment](#7-dataflow-templates--deployment)
8. [Autoscaling, Performance & Optimization](#8-autoscaling-performance--optimization)
9. [Monitoring, Observability & Debugging](#9-monitoring-observability--debugging)
10. [Access Control, Security & Networking](#10-access-control-security--networking)
11. [gcloud CLI & REST API Quick Reference](#11-gcloud-cli--rest-api-quick-reference)
12. [Pricing Summary](#12-pricing-summary)
13. [Quick Reference & Comparison Tables](#13-quick-reference--comparison-tables)

---

## 1. Overview & Key Concepts

### What is Dataflow?

**Google Cloud Dataflow** is a fully managed, serverless stream and batch data processing service built on the **Apache Beam** programming model. It eliminates infrastructure management — you write a Beam pipeline, Dataflow runs it.

**Core value proposition:**

| Pillar | Description |
|---|---|
| **Fully managed** | No cluster provisioning, patching, or capacity planning |
| **Unified model** | Same SDK and pipeline code for batch and streaming |
| **Autoscaling** | Workers scale up/down automatically based on backlog |
| **Optimized** | Dataflow Shuffle and Streaming Engine reduce cost and latency |
| **Portable** | Run the same Beam pipeline on DirectRunner, Spark, Flink, or Dataflow |

---

### Apache Beam ↔ Dataflow Relationship

```
┌──────────────────────────────────────────────────────────┐
│                  YOUR PIPELINE CODE                       │
│           (Apache Beam SDK: Python / Java / Go)           │
└───────────────────────────┬──────────────────────────────┘
                            │ compile + submit
            ┌───────────────▼───────────────────┐
            │         BEAM RUNNER                │
            │  ┌─────────────┐  ┌─────────────┐ │
            │  │ DirectRunner│  │  DataflowRun │ │
            │  │  (local dev)│  │  (GCP prod)  │ │
            │  └─────────────┘  └──────┬───────┘ │
            └─────────────────────────┼──────────┘
                                      │
            ┌─────────────────────────▼──────────┐
            │          DATAFLOW SERVICE           │
            │  Job Graph Optimizer → GCE Workers  │
            │  (Dataflow Shuffle / Streaming Eng) │
            └────────────────────────────────────┘
```

---

### Core Abstractions

| Abstraction | Description |
|---|---|
| **Pipeline** | The top-level container for your entire data processing job |
| **PCollection** | Immutable, distributed dataset flowing through the pipeline |
| **PTransform** | A data processing operation applied to a PCollection |
| **DoFn** | User-defined function applied per element inside a `ParDo` |
| **Coder** | Serialization/deserialization spec for elements in a PCollection |
| **Schema** | Typed, named-field description of PCollection elements (like a row type) |
| **Runner** | Execution backend (DirectRunner, DataflowRunner, SparkRunner, etc.) |

---

### Batch vs. Streaming

| Dimension | Batch | Streaming |
|---|---|---|
| **PCollection type** | Bounded (finite data) | Unbounded (infinite data) |
| **Data source** | GCS, BigQuery, JDBC | Pub/Sub, Kafka, Kinesis |
| **Windowing** | Optional | Required for grouping |
| **Latency** | Minutes to hours | Seconds to minutes |
| **Watermarks** | Not applicable | Critical for correctness |
| **Cost model** | Pay per job run | Pay continuously |
| **Use case** | Nightly ETL, backfill, ML training | Real-time analytics, alerting, CDC |

---

### Dataflow Execution Model

```
Pipeline Code (Beam SDK)
       │
       ▼
1. GRAPH CONSTRUCTION   — DAG of PTransforms built in memory
       │
       ▼
2. OPTIMIZATION         — Fusion of consecutive transforms, combiner lifting
       │
       ▼
3. SUBMISSION           — Serialized job graph sent to Dataflow service
       │
       ▼
4. WORKER PROVISIONING  — GCE VMs launched in your project
       │
       ▼
5. EXECUTION            — Workers pull work items, process, shuffle data
       │
       ▼
6. AUTOSCALING          — Workers added/removed based on throughput backlog
```

**Key optimizations:**
- **Dataflow Shuffle (batch):** Offloads shuffle to managed service → faster, cheaper than worker-to-worker
- **Streaming Engine:** Offloads windowing state to managed backend → lower worker memory, faster scaling

---

### Dataflow vs. Alternatives

| Feature | Dataflow | Dataproc (Spark) | BigQuery |
|---|---|---|---|
| **Model** | Beam (unified batch+stream) | Spark / Hadoop | SQL only |
| **Management** | Fully serverless | Semi-managed cluster | Fully serverless |
| **Streaming** | ✅ Native, sub-minute | ⚠️ Structured Streaming | ⚠️ Streaming inserts only |
| **Custom logic** | ✅ Arbitrary code (DoFn) | ✅ Arbitrary code | ❌ SQL + UDFs only |
| **State mgmt** | ✅ ValueState, BagState, etc. | ✅ Spark RDD lineage | ❌ |
| **Best for** | Complex streaming ETL | Spark workloads, ML | SQL analytics |

---

### Dataflow Editions

| Edition | Billing | Autoscaling | Best For |
|---|---|---|---|
| **Classic Dataflow** | Per vCPU-hr + GB RAM-hr + Disk | Horizontal only | Predictable workloads |
| **Dataflow Prime** | Resource-based (RSU) | Horizontal + **Vertical** | Variable, bursty workloads |

> 💡 **Dataflow Prime** uses flexible Resource Scheduling Units (RSUs) and can vertically scale worker memory/CPU per pipeline stage — ideal for mixed-complexity pipelines.

---

### SDK Language Maturity

| Language | Maturity | Notes |
|---|---|---|
| **Java** | ✅ Most complete | All features, all I/O connectors, best performance |
| **Python** | ✅ Production-ready | Slightly fewer connectors; uses cross-language transforms for some I/O |
| **Go** | ⚠️ Beta | Growing support; some connectors missing |

---

## 2. Core Beam Programming Model

### Pipeline Creation

```python
# Python — Pipeline with DataflowRunner options
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions, GoogleCloudOptions

options = PipelineOptions(
    runner='DataflowRunner',
    project='my-project',
    region='us-central1',
    temp_location='gs://my-bucket/tmp',
    staging_location='gs://my-bucket/staging',
    job_name='my-pipeline',
    streaming=False,                        # True for streaming
    num_workers=4,
    max_num_workers=20,
    machine_type='n1-standard-4',
    experiments=['use_runner_v2'],
)

with beam.Pipeline(options=options) as p:
    result = (
        p
        | 'ReadLines'  >> beam.io.ReadFromText('gs://bucket/input.txt')
        | 'Transform'  >> beam.Map(str.upper)
        | 'WriteLines' >> beam.io.WriteToText('gs://bucket/output')
    )
```

```java
// Java — Pipeline with DataflowPipelineOptions
import org.apache.beam.sdk.Pipeline;
import org.apache.beam.runners.dataflow.options.DataflowPipelineOptions;
import org.apache.beam.sdk.options.PipelineOptionsFactory;

DataflowPipelineOptions options = PipelineOptionsFactory
    .fromArgs(args)
    .withValidation()
    .as(DataflowPipelineOptions.class);

options.setRunner(DataflowRunner.class);
options.setProject("my-project");
options.setRegion("us-central1");
options.setTempLocation("gs://my-bucket/tmp");
options.setJobName("my-java-pipeline");
options.setNumWorkers(4);
options.setMaxNumWorkers(20);

Pipeline p = Pipeline.create(options);
// ... add transforms ...
p.run().waitUntilFinish();
```

---

### PCollection: Bounded vs. Unbounded

```python
# Bounded PCollection (batch) — from GCS file
lines = p | beam.io.ReadFromText('gs://bucket/data.txt')

# Unbounded PCollection (streaming) — from Pub/Sub
messages = p | beam.io.ReadFromPubSub(
    subscription='projects/proj/subscriptions/my-sub',
    with_attributes=True,
    timestamp_attribute='event_timestamp'
)
```

> 💡 A **bounded** PCollection has a known, finite size. An **unbounded** PCollection is an infinite stream. Windowing is required to group unbounded data for aggregations.

---

### Built-in Transforms

```python
# ParDo — apply a DoFn to each element
output = input_pc | beam.ParDo(MyDoFn())

# Map — shorthand for simple 1:1 transforms (wraps ParDo)
upper = lines | beam.Map(str.upper)

# FlatMap — 1:N (one element produces multiple outputs)
words = lines | beam.FlatMap(str.split)

# Filter — keep elements matching predicate
long_words = words | beam.Filter(lambda w: len(w) > 5)

# GroupByKey — group (key, value) pairs by key
grouped = kvs | beam.GroupByKey()

# CoGroupByKey — join two PCollections by key
result = ({'orders': orders_kv, 'items': items_kv}
          | beam.CoGroupByKey())

# Combine.perKey — aggregate values per key
totals = sales_kv | beam.CombinePerKey(sum)

# CombineGlobally — single aggregate over entire PCollection
total = amounts | beam.CombineGlobally(sum)

# Flatten — merge multiple PCollections of the same type
merged = (pc1, pc2, pc3) | beam.Flatten()

# Partition — split one PCollection into N
partitions = data | beam.Partition(
    lambda elem, n: elem['region'] % n, 3  # into 3 partitions
)
```

---

### DoFn Lifecycle

```python
class MyDoFn(beam.DoFn):

    def setup(self):
        """Called once per worker — initialize connections, load models."""
        self.client = SomeAPIClient()

    def start_bundle(self):
        """Called once per bundle (batch of elements) — reset counters."""
        self.buffer = []

    def process(self, element, timestamp=beam.DoFn.TimestampParam,
                window=beam.DoFn.WindowParam):
        """Called once per element — main processing logic."""
        self.buffer.append(element)
        yield element.upper()          # yield = emit to output PCollection

    def finish_bundle(self):
        """Called after bundle completes — flush buffers."""
        # Must yield WindowedValues when emitting from finish_bundle
        for item in self.buffer:
            yield beam.utils.windowed_value.WindowedValue(
                item, timestamp=0, windows=[beam.transforms.window.GlobalWindow()])

    def teardown(self):
        """Called once when worker shuts down — close connections."""
        self.client.close()
```

```java
// Java DoFn lifecycle
public class MyDoFn extends DoFn<String, String> {

    @Setup
    public void setup() { /* init connections */ }

    @StartBundle
    public void startBundle() { /* reset per-bundle state */ }

    @ProcessElement
    public void processElement(@Element String element,
                               OutputReceiver<String> out,
                               @Timestamp Instant timestamp) {
        out.output(element.toUpperCase());
    }

    @FinishBundle
    public void finishBundle(FinishBundleContext c) { /* flush */ }

    @Teardown
    public void teardown() { /* close resources */ }
}
```

---

### Side Inputs

Side inputs allow a DoFn to access additional data (e.g., a lookup table) alongside the main input.

```python
# Create a side input from a PCollection
lookup_table = (
    p
    | 'ReadLookup' >> beam.io.ReadFromText('gs://bucket/lookup.csv')
    | 'BuildMap'   >> beam.Map(lambda line: line.split(','))
    | 'ToDict'     >> beam.combiners.ToDict()
)

# AsSingleton — side input is a single value
result = data | beam.Map(
    lambda elem, lookup: lookup.get(elem['key'], 'UNKNOWN'),
    lookup=beam.pvalue.AsSingleton(lookup_table)
)

# AsDict — side input is a dict (key → value)
result = data | beam.Map(
    lambda elem, lookup: lookup.get(elem['key']),
    lookup=beam.pvalue.AsDict(lookup_kv_pc)
)

# AsIter — side input is an iterable
result = data | beam.FlatMap(
    lambda elem, blocklist: [elem] if elem not in blocklist else [],
    blocklist=beam.pvalue.AsIter(blocklist_pc)
)
```

```java
// Java side input (AsSingleton)
PCollectionView<Map<String, String>> lookupView =
    lookupPc.apply(View.asMap());

PCollection<String> result = data.apply(
    ParDo.of(new DoFn<String, String>() {
        @ProcessElement
        public void process(@Element String elem,
                            OutputReceiver<String> out,
                            ProcessContext c) {
            Map<String, String> lookup = c.sideInput(lookupView);
            out.output(lookup.getOrDefault(elem, "UNKNOWN"));
        }
    }).withSideInputs(lookupView)
);
```

> ⚠️ Side inputs are **broadcast to all workers** — keep them small (< few GB). For large lookups, consider Bigtable or Spanner enrichment instead.

---

### Tagged Outputs (Multiple Outputs)

```python
# Python — tagged outputs for routing / dead-letter queue
MAIN_TAG   = 'valid'
DLQ_TAG    = 'invalid'

class ParseAndValidate(beam.DoFn):
    def process(self, element):
        try:
            record = json.loads(element)
            if 'user_id' not in record:
                raise ValueError("Missing user_id")
            yield record                              # → main output
        except Exception as e:
            yield beam.pvalue.TaggedOutput(DLQ_TAG,  # → dead-letter
                {'raw': element, 'error': str(e)})

results = lines | beam.ParDo(ParseAndValidate()).with_outputs(
    DLQ_TAG, main=MAIN_TAG
)
valid_records   = results[MAIN_TAG]
invalid_records = results[DLQ_TAG]

# Write DLQ to a separate sink
invalid_records | beam.io.WriteToPubSub('projects/p/topics/dlq')
```

```java
// Java — tagged outputs
TupleTag<TableRow> validTag   = new TupleTag<>() {};
TupleTag<String>   invalidTag = new TupleTag<>() {};

PCollectionTuple results = lines.apply(
    ParDo.of(new DoFn<String, TableRow>() {
        @ProcessElement
        public void process(@Element String elem,
                            MultiOutputReceiver out) {
            try {
                TableRow row = parseRow(elem);
                out.get(validTag).output(row);
            } catch (Exception e) {
                out.get(invalidTag).output(elem + "|" + e.getMessage());
            }
        }
    }).withOutputTags(validTag, TupleTagList.of(invalidTag))
);

PCollection<TableRow> valid   = results.get(validTag);
PCollection<String>   invalid = results.get(invalidTag);
```

---

### Composite Transforms

```python
# Wrap multiple transforms into a reusable PTransform
class NormalizeAndFilter(beam.PTransform):
    def __init__(self, min_length=3):
        self.min_length = min_length

    def expand(self, pcoll):
        return (
            pcoll
            | 'Lowercase'   >> beam.Map(str.lower)
            | 'Strip'       >> beam.Map(str.strip)
            | 'FilterShort' >> beam.Filter(lambda w: len(w) >= self.min_length)
        )

# Use it like any built-in transform
clean_words = raw_lines | 'Normalize' >> NormalizeAndFilter(min_length=4)
```

---

## 3. Reading & Writing Data (I/O Connectors)

### I/O Connector Reference

| Connector | Direction | Bounded/Unbounded | Python | Java |
|---|---|---|---|---|
| BigQuery | Source + Sink | Both | ✅ | ✅ |
| Pub/Sub | Source + Sink | Unbounded | ✅ | ✅ |
| Cloud Storage (Text) | Source + Sink | Bounded | ✅ | ✅ |
| Cloud Storage (Avro) | Source + Sink | Bounded | ✅ | ✅ |
| Cloud Storage (Parquet) | Source + Sink | Bounded | ✅ | ✅ |
| Bigtable | Source + Sink | Bounded | ✅ | ✅ |
| Spanner | Source + Sink | Bounded | ✅ | ✅ |
| Firestore/Datastore | Source + Sink | Bounded | ✅ | ✅ |
| Kafka | Source + Sink | Unbounded | ✅ (XLANG) | ✅ |
| JDBC | Source + Sink | Bounded | ✅ (XLANG) | ✅ |

---

### BigQuery I/O

```python
# ── READ from BigQuery ────────────────────────────────────────────────
# Option 1: SQL query (export-based)
rows = p | beam.io.ReadFromBigQuery(
    query='SELECT user_id, amount FROM `proj.ds.sales` WHERE dt = "2026-03-16"',
    use_standard_sql=True,
    project='my-project'
)

# Option 2: Direct table read (faster for full scans)
rows = p | beam.io.ReadFromBigQuery(
    table='my-project:dataset.sales',
    method=beam.io.ReadFromBigQuery.Method.DIRECT_READ  # uses Storage Read API
)

# ── WRITE to BigQuery ─────────────────────────────────────────────────
# Streaming inserts (low latency)
rows | beam.io.WriteToBigQuery(
    table='my-project:dataset.output',
    schema='user_id:STRING,amount:FLOAT,ts:TIMESTAMP',
    write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND,
    create_disposition=beam.io.BigQueryDisposition.CREATE_IF_NEEDED,
    method=beam.io.WriteToBigQuery.Method.STREAMING_INSERTS
)

# Storage Write API (recommended — exactly-once, lower cost)
rows | beam.io.WriteToBigQuery(
    table='my-project:dataset.output',
    schema=SCHEMA,
    method=beam.io.WriteToBigQuery.Method.STORAGE_WRITE_API
)

# FILE_LOADS (batch, cheapest)
rows | beam.io.WriteToBigQuery(
    table='my-project:dataset.output',
    schema=SCHEMA,
    method=beam.io.WriteToBigQuery.Method.FILE_LOADS,
    triggering_frequency=300   # flush every 5 minutes (streaming mode)
)

# Dynamic destinations — route rows to different tables
def get_table(row):
    return f"my-project:dataset.events_{row['region']}"

rows | beam.io.WriteToBigQuery(
    table=get_table,
    schema=lambda table, schema: SCHEMA,   # or per-table schema
    method=beam.io.WriteToBigQuery.Method.STORAGE_WRITE_API
)
```

```java
// Java BigQuery write with Storage Write API
rows.apply(BigQueryIO.<TableRow>write()
    .to("my-project:dataset.output")
    .withSchema(schema)
    .withWriteDisposition(BigQueryIO.Write.WriteDisposition.WRITE_APPEND)
    .withCreateDisposition(BigQueryIO.Write.CreateDisposition.CREATE_IF_NEEDED)
    .withMethod(BigQueryIO.Write.Method.STORAGE_WRITE_API)
    .withNumStorageWriteApiStreams(4)
    .withStorageWriteApiTriggeringFrequencySec(30)
);
```

---

### Pub/Sub I/O

```python
# ── READ from Pub/Sub ─────────────────────────────────────────────────
# From subscription (recommended — preserves ack tracking)
messages = p | beam.io.ReadFromPubSub(
    subscription='projects/my-proj/subscriptions/my-sub',
    with_attributes=True,             # Returns PubsubMessage objects
    timestamp_attribute='event_time'  # Use message attribute as event timestamp
)

# From topic (Dataflow creates a temporary subscription)
messages = p | beam.io.ReadFromPubSub(
    topic='projects/my-proj/topics/my-topic'
)

# ── WRITE to Pub/Sub ──────────────────────────────────────────────────
results | beam.io.WriteToPubSub(
    topic='projects/my-proj/topics/output-topic',
    with_attributes=False   # True if elements are PubsubMessage objects
)

# Write with attributes
import apache_beam.io.gcp.pubsub as pubsub

tagged = results | beam.Map(
    lambda r: pubsub.PubsubMessage(
        data=json.dumps(r).encode(),
        attributes={'source': 'dataflow', 'env': 'prod'}
    )
)
tagged | beam.io.WriteToPubSub(topic='projects/p/topics/out',
                               with_attributes=True)
```

---

### Cloud Storage I/O

```python
# Text files
lines = p | beam.io.ReadFromText(
    'gs://bucket/input/*.txt',
    skip_header_lines=1
)
output | beam.io.WriteToText(
    'gs://bucket/output/result',
    file_name_suffix='.txt',
    num_shards=10
)

# Avro
records = p | beam.io.ReadFromAvro('gs://bucket/data/*.avro')
output  | beam.io.WriteToAvro(
    'gs://bucket/output/data',
    schema=AVRO_SCHEMA,
    file_name_suffix='.avro'
)

# Parquet
import apache_beam.io.parquetio as parquetio

records = p | parquetio.ReadFromParquet(
    'gs://bucket/data/*.parquet',
    columns=['user_id', 'amount']   # column projection
)
output | parquetio.WriteToParquet(
    'gs://bucket/output/',
    schema=PYARROW_SCHEMA,
    file_name_suffix='.parquet',
    codec='snappy'
)
```

---

### Kafka I/O (Cross-language)

```python
# Python — Kafka via cross-language transform (requires Java expansion service)
from apache_beam.io.kafka import ReadFromKafka, WriteToKafka

messages = p | ReadFromKafka(
    consumer_config={
        'bootstrap.servers': 'kafka-broker:9092',
        'group.id': 'dataflow-consumer-group',
        'auto.offset.reset': 'latest',
    },
    topics=['my-input-topic'],
    with_metadata=True,
    expansion_service='localhost:8088'  # or auto-started
)

output | WriteToKafka(
    producer_config={'bootstrap.servers': 'kafka-broker:9092'},
    topic='my-output-topic',
    key_serializer='org.apache.kafka.common.serialization.StringSerializer',
    value_serializer='org.apache.kafka.common.serialization.StringSerializer',
    expansion_service='localhost:8088'
)
```

---

### Custom I/O: BoundedSource

```python
# Implement a custom bounded source
class MyBoundedSource(beam.io.BoundedSource):

    def __init__(self, url):
        self.url = url

    def estimate_size(self):
        return 1024 * 1024  # 1 MB estimate

    def get_range_tracker(self, start_pos, stop_pos):
        return beam.io.iobase.OffsetRangeTracker(
            start_pos or 0,
            stop_pos or beam.io.iobase.OffsetRangeTracker.OFFSET_INFINITY
        )

    def read(self, range_tracker):
        for i, record in enumerate(fetch_records(self.url)):
            if not range_tracker.try_claim(i):
                return
            yield record

    def split(self, desired_bundle_size, start_pos=None, stop_pos=None):
        yield beam.io.iobase.SourceBundle(
            weight=1.0, source=self, start_position=None, stop_position=None
        )

# Use the custom source
records = p | beam.io.Read(MyBoundedSource('https://api.example.com/data'))
```

---

## 4. Windowing & Triggers (Streaming)

### Window Types

```python
from apache_beam import window

# ── Fixed Windows — non-overlapping, same-duration buckets ───────────
windowed = data | beam.WindowInto(window.FixedWindows(60))  # 60-second windows

# ── Sliding Windows — overlapping windows for rolling aggregations ────
windowed = data | beam.WindowInto(
    window.SlidingWindows(size=300, period=60)  # 5-min window, slide every 1 min
)

# ── Session Windows — gap-based, per-key activity windows ────────────
windowed = data | beam.WindowInto(
    window.Sessions(gap_size=1800)  # new session after 30-min inactivity
)

# ── GlobalWindow — entire stream as one window (default for batch) ────
windowed = data | beam.WindowInto(window.GlobalWindows())
```

| Window Type | Shape | Key Characteristic | Use Case |
|---|---|---|---|
| **Fixed** | Non-overlapping | Same duration per window | Hourly/daily aggregations |
| **Sliding** | Overlapping | Window every N seconds | Rolling averages, moving totals |
| **Session** | Variable length | Gap-based, per-key | User session analytics |
| **Global** | One window | Entire stream | Batch-style on streams |

---

### Watermarks: Event Time vs. Processing Time

```
Event Timeline:
  Event generated at: 10:00:00  ← event time (embedded in message)
  Event arrives at:   10:00:05  ← processing time (when Dataflow sees it)
  Lag = 5 seconds

Watermark = estimate of "how far behind are we in event time?"
  - Watermark at T means: "all events with event_time < T have (likely) arrived"
  - Dataflow estimates watermark from Pub/Sub message timestamps
  - Late data = elements arriving AFTER the watermark has passed their window

         Event Time ──────────────────────────────────►
              [  Window 1  ][  Window 2  ][  Window 3  ]
                                   ▲
                               Watermark
                         (events before here are "on time")
```

---

### Triggers

```python
from apache_beam.transforms.trigger import (
    AfterWatermark, AfterProcessingTime, AfterCount,
    Repeatedly, AfterAny, AccumulationMode
)

# Fire when watermark passes window end (standard), plus early/late firings
windowed = data | beam.WindowInto(
    window.FixedWindows(60),
    trigger=AfterWatermark(
        early=Repeatedly(AfterCount(100)),          # Fire every 100 elements early
        late=Repeatedly(AfterProcessingTime(10))    # Fire every 10s for late data
    ),
    accumulation_mode=AccumulationMode.ACCUMULATING,
    allowed_lateness=window.Duration(seconds=300)   # Accept data up to 5min late
)

# Fire every N elements (count-based)
windowed = data | beam.WindowInto(
    window.GlobalWindows(),
    trigger=Repeatedly(AfterCount(1000)),
    accumulation_mode=AccumulationMode.DISCARDING
)

# Fire every N seconds of processing time
windowed = data | beam.WindowInto(
    window.GlobalWindows(),
    trigger=Repeatedly(AfterProcessingTime(30)),
    accumulation_mode=AccumulationMode.ACCUMULATING
)
```

---

### Accumulation Modes

| Mode | Description | Use Case |
|---|---|---|
| `ACCUMULATING` | Each pane includes ALL data in window (cumulative) | Running totals, dashboards |
| `DISCARDING` | Each pane includes ONLY new data since last firing | Incremental processing, exactly-once sinks |

---

### Allowed Lateness & Late Data

```python
# Accept data up to 5 minutes late after the window closes
windowed = data | beam.WindowInto(
    window.FixedWindows(60),
    trigger=AfterWatermark(
        late=Repeatedly(AfterProcessingTime(30))
    ),
    accumulation_mode=AccumulationMode.ACCUMULATING,
    allowed_lateness=window.Duration(seconds=300)   # 5 min grace period
)

# In your DoFn, detect whether a pane is ON_TIME, EARLY, or LATE
class HandlePane(beam.DoFn):
    def process(self, element, pane_info=beam.DoFn.PaneInfoParam):
        if pane_info.timing == beam.transforms.trigger.PaneInfoTiming.LATE:
            yield beam.pvalue.TaggedOutput('late', element)
        else:
            yield element
```

---

### Timestamp Assignment in DoFns

```python
class AssignEventTimestamp(beam.DoFn):
    def process(self, element):
        # Parse event timestamp from the message and assign it
        event_ts = datetime.fromisoformat(element['event_time'])
        unix_ts = event_ts.timestamp()
        yield beam.window.TimestampedValue(element, unix_ts)

timestamped = raw | beam.ParDo(AssignEventTimestamp())
```

---

### Windowed Pipeline Example: Session Detection

```python
# Detect user sessions from a stream of events
(
    p
    | 'ReadPubSub'    >> beam.io.ReadFromPubSub(subscription=SUB,
                                                 timestamp_attribute='event_ts')
    | 'ParseJSON'     >> beam.Map(json.loads)
    | 'KeyByUser'     >> beam.Map(lambda r: (r['user_id'], r))
    | 'SessionWindow' >> beam.WindowInto(
                            window.Sessions(gap_size=1800),           # 30-min gap
                            trigger=AfterWatermark(),
                            accumulation_mode=AccumulationMode.DISCARDING,
                            allowed_lateness=window.Duration(seconds=60)
                        )
    | 'GroupByUser'   >> beam.GroupByKey()
    | 'BuildSession'  >> beam.Map(lambda kv: {
                            'user_id': kv[0],
                            'event_count': len(list(kv[1])),
                        })
    | 'WriteBQ'       >> beam.io.WriteToBigQuery('proj:ds.sessions', schema=SCHEMA)
)
```

---

## 5. State & Timers (Stateful Processing)

### When to Use Stateful Processing

| Scenario | Use | Why |
|---|---|---|
| Count events per key | `CombinePerKey` | Simpler, more scalable |
| Deduplicate per key | **ValueState** | Need per-key memory across elements |
| Buffer N elements then flush | **BagState** + timer | Micro-batching pattern |
| Track seen keys (bloom filter) | **SetState** | Membership testing |
| Per-key sliding counter | **CombiningState** | Incremental merge |

---

### State Types

```python
import apache_beam as beam
from apache_beam.transforms.userstate import (
    ReadModifyWriteStateSpec, BagStateSpec, SetStateSpec,
    CombiningValueStateSpec, TimerSpec
)
from apache_beam.coders import VarIntCoder, StrUtf8Coder
from apache_beam.transforms.timeutil import TimeDomain

class StatefulDoFn(beam.DoFn):

    # ValueState — single mutable value per key
    COUNT_STATE = ReadModifyWriteStateSpec('count', VarIntCoder())

    # BagState — append-only collection per key
    BUFFER_STATE = BagStateSpec('buffer', beam.coders.FastPrimitivesCoder())

    # SetState — unique values per key
    SEEN_STATE = SetStateSpec('seen', StrUtf8Coder())

    # CombiningState — automatically merges with a CombineFn
    SUM_STATE = CombiningValueStateSpec('sum', VarIntCoder(), sum)

    # Processing-time timer
    FLUSH_TIMER = TimerSpec('flush', TimeDomain.REAL_TIME)

    # Event-time timer
    EXPIRY_TIMER = TimerSpec('expiry', TimeDomain.WATERMARK)

    def process(self, element, count=beam.DoFn.StateParam(COUNT_STATE),
                buffer=beam.DoFn.StateParam(BUFFER_STATE),
                seen=beam.DoFn.StateParam(SEEN_STATE),
                flush_timer=beam.DoFn.TimerParam(FLUSH_TIMER)):

        key, value = element

        # ValueState: read-modify-write
        current_count = count.read() or 0
        count.write(current_count + 1)

        # BagState: append element
        buffer.add(value)

        # SetState: track unique IDs (deduplication)
        if value['id'] in seen.read():
            return     # duplicate — skip
        seen.add(value['id'])

        # Set flush timer 10 seconds from now
        flush_timer.set(beam.utils.timestamp.Timestamp.now() + 10)

    @beam.DoFn.on_timer(FLUSH_TIMER)
    def flush(self, buffer=beam.DoFn.StateParam(BUFFER_STATE)):
        """Fires when flush timer expires — emit buffered data."""
        items = list(buffer.read())
        buffer.clear()
        for item in items:
            yield item
```

---

### Deduplication with ValueState

```python
class DeduplicateDoFn(beam.DoFn):
    SEEN = ReadModifyWriteStateSpec('seen', StrUtf8Coder())

    def process(self, element,
                seen_state=beam.DoFn.StateParam(SEEN)):
        key, record = element
        msg_id = record.get('message_id')

        if seen_state.read() == msg_id:
            return    # Already processed — drop duplicate

        seen_state.write(msg_id)
        yield record

deduped = (
    keyed_records
    | beam.WindowInto(window.FixedWindows(3600))   # 1-hour dedup window
    | beam.ParDo(DeduplicateDoFn())
)
```

---

### Flush-on-Timer Pattern (Micro-batching)

```python
class MicroBatchDoFn(beam.DoFn):
    BUFFER = BagStateSpec('buffer', beam.coders.FastPrimitivesCoder())
    TIMER  = TimerSpec('flush_timer', TimeDomain.REAL_TIME)
    BATCH_INTERVAL_SECS = 30

    def process(self, element,
                buffer=beam.DoFn.StateParam(BUFFER),
                timer=beam.DoFn.TimerParam(TIMER)):
        key, record = element
        buffer.add(record)
        # (Re-)set the timer — only fires if not reset within the interval
        timer.set(
            beam.utils.timestamp.Timestamp.now() + self.BATCH_INTERVAL_SECS
        )

    @beam.DoFn.on_timer(TIMER)
    def flush(self, key=beam.DoFn.KeyParam,
              buffer=beam.DoFn.StateParam(BUFFER)):
        records = list(buffer.read())
        buffer.clear()
        if records:
            # Emit as a batch for downstream processing
            yield (key, records)
```

---

## 6. Pipeline Patterns & Best Practices

### Dead-Letter Queue (DLQ) Pattern

```python
# Robust pipeline with DLQ routing for all errors
VALID_TAG = 'valid'
DLQ_TAG   = 'dlq'

class SafeParser(beam.DoFn):
    def process(self, raw_msg):
        try:
            record = json.loads(raw_msg)
            validate_schema(record)       # Raises on invalid
            yield record
        except Exception as e:
            yield beam.pvalue.TaggedOutput(DLQ_TAG, {
                'raw': raw_msg,
                'error': str(e),
                'ts': datetime.utcnow().isoformat()
            })

results = (
    messages
    | beam.ParDo(SafeParser()).with_outputs(DLQ_TAG, main=VALID_TAG)
)

# Route valid records to BigQuery
results[VALID_TAG]  | beam.io.WriteToBigQuery('proj:ds.events', schema=SCHEMA)

# Route DLQ records to a separate topic/table for reprocessing
results[DLQ_TAG]    | beam.io.WriteToBigQuery('proj:ds.events_dlq',
                                               schema=DLQ_SCHEMA)
```

---

### Fan-Out and Fan-In

```python
# Fan-out: one PCollection → multiple processing branches
parsed = raw | 'Parse' >> beam.Map(json.loads)

# Branch 1: write to BigQuery
parsed | 'ToBQ'    >> beam.io.WriteToBigQuery(...)

# Branch 2: filter and write to Pub/Sub
(parsed
 | 'FilterHigh' >> beam.Filter(lambda r: r['amount'] > 1000)
 | 'ToPubSub'   >> beam.io.WriteToPubSub(topic=ALERT_TOPIC))

# Branch 3: aggregate and write to GCS
(parsed
 | 'KeyByRegion' >> beam.Map(lambda r: (r['region'], r['amount']))
 | 'SumByRegion' >> beam.CombinePerKey(sum)
 | 'ToGCS'       >> beam.io.WriteToText('gs://bucket/region_totals'))

# Fan-in: merge multiple PCollections
all_events = (pc1, pc2, pc3) | 'Merge' >> beam.Flatten()
```

---

### Enrichment Join (Side Input Pattern)

```python
# Streaming enrichment: join event stream with a slowly-changing lookup
# Lookup loaded from BigQuery (re-read periodically via batch pipeline)

lookup_view = (
    p
    | 'LoadUserData' >> beam.io.ReadFromBigQuery(
        query='SELECT user_id, tier FROM `proj.ds.users`',
        use_standard_sql=True
    )
    | 'ToDict' >> beam.combiners.ToDict(
        key=lambda r: r['user_id'],
        value=lambda r: r['tier']
    )
    | beam.pvalue.AsDict()   # Wrap as side input view
)

enriched = (
    events
    | beam.Map(
        lambda event, users: {**event, 'tier': users.get(event['user_id'], 'free')},
        users=lookup_view
    )
)
```

---

### Reshuffle: Preventing Fusion

```python
# Dataflow fuses consecutive transforms onto the same worker for efficiency.
# This can cause issues when you WANT parallelism to increase.
# Use Reshuffle to break fusion and distribute work.

(
    data
    | 'HeavyTransform1' >> beam.ParDo(HeavyDoFn1())
    | 'Reshuffle'       >> beam.Reshuffle()   # ← forces new parallelism boundary
    | 'HeavyTransform2' >> beam.ParDo(HeavyDoFn2())
)
```

> 💡 Use `Reshuffle` after a step that dramatically changes element count (e.g., a FlatMap that explodes rows) so downstream steps can scale independently.

---

### Hot Key Mitigation

```python
# ❌ Hot key problem: one key dominates all traffic
counts = (data
          | beam.Map(lambda x: ('global_key', x))  # All goes to one worker!
          | beam.CombinePerKey(sum))

# ✅ Fix: use withHotKeyFanout
counts = (data
          | beam.Map(lambda x: ('global_key', x))
          | beam.CombinePerKey(sum).with_hot_key_fanout(100))  # Fan out to 100 partitions

# ✅ Alternative: add a random shard suffix, aggregate in two stages
import random
TWO_PHASE_SHARDS = 100

partial = (
    data
    | 'AddShard'    >> beam.Map(lambda x: (f"key_{random.randint(0, TWO_PHASE_SHARDS)}", x))
    | 'PartialSum'  >> beam.CombinePerKey(sum)
    | 'StripShard'  >> beam.Map(lambda kv: ('global_key', kv[1]))
    | 'FinalSum'    >> beam.CombinePerKey(sum)
)
```

---

### CombineFn (Custom Aggregation)

```python
class MeanCombineFn(beam.CombineFn):
    """Compute mean with a (sum, count) accumulator."""

    def create_accumulator(self):
        return (0.0, 0)     # (sum, count)

    def add_input(self, accumulator, element):
        s, c = accumulator
        return (s + element, c + 1)

    def merge_accumulators(self, accumulators):
        sums, counts = zip(*accumulators)
        return (sum(sums), sum(counts))

    def extract_output(self, accumulator):
        s, c = accumulator
        return s / c if c > 0 else 0.0

mean_per_key = data | beam.CombinePerKey(MeanCombineFn())
global_mean  = data | beam.CombineGlobally(MeanCombineFn())
```

---

## 7. Dataflow Templates & Deployment

### Classic Templates

Classic templates are compiled pipeline JARs/wheel files staged to GCS. Runtime parameters are passed as `--parameters`.

```bash
# ── Build and stage a Classic Template (Python) ───────────────────────
python my_pipeline.py \
  --runner=DataflowRunner \
  --project=my-project \
  --region=us-central1 \
  --temp_location=gs://bucket/tmp \
  --template_location=gs://bucket/templates/my-template \
  --staging_location=gs://bucket/staging \
  # Don't execute — just stage the template graph

# ── Run a staged Classic Template ─────────────────────────────────────
gcloud dataflow jobs run my-job \
  --gcs-location=gs://bucket/templates/my-template \
  --region=us-central1 \
  --parameters inputFile=gs://bucket/data/*.txt,outputTable=proj:ds.table
```

> ⚠️ Classic templates **freeze dependencies at build time**. You cannot change SDK version or add connectors at runtime. Use Flex Templates for more flexibility.

---

### Flex Templates

Flex Templates package the pipeline as a **Docker container**, giving full control over dependencies and runtime behavior.

```dockerfile
# Dockerfile for a Python Flex Template
FROM apache/beam_python3.11_sdk:2.55.0

WORKDIR /pipeline
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY my_pipeline.py .

ENTRYPOINT ["python", "my_pipeline.py"]
```

```bash
# ── Build and push container ──────────────────────────────────────────
docker build -t gcr.io/my-project/my-pipeline:latest .
docker push gcr.io/my-project/my-pipeline:latest

# ── Create Flex Template spec (metadata JSON) ─────────────────────────
cat > metadata.json << 'EOF'
{
  "name": "My Pipeline",
  "description": "Reads from Pub/Sub and writes to BigQuery",
  "parameters": [
    {"name": "input_subscription", "label": "Pub/Sub Subscription", "paramType": "TEXT"},
    {"name": "output_table",       "label": "BigQuery Output Table",  "paramType": "TEXT"}
  ]
}
EOF

# ── Stage the Flex Template ────────────────────────────────────────────
gcloud dataflow flex-template build \
  gs://bucket/templates/my-flex-template.json \
  --image=gcr.io/my-project/my-pipeline:latest \
  --sdk-language=PYTHON \
  --metadata-file=metadata.json

# ── Run the Flex Template ──────────────────────────────────────────────
gcloud dataflow flex-template run my-flex-job \
  --template-file-gcs-location=gs://bucket/templates/my-flex-template.json \
  --region=us-central1 \
  --parameters input_subscription=projects/p/subscriptions/sub \
  --parameters output_table=proj:ds.output \
  --parameters max_num_workers=20
```

---

### gcloud dataflow jobs run (Key Flags)

```bash
gcloud dataflow jobs run JOB_NAME \
  --gcs-location=gs://bucket/templates/template \
  --region=us-central1 \
  --zone=us-central1-a \                       # Pin to specific zone
  --num-workers=5 \                            # Initial worker count
  --max-workers=50 \                           # Max workers (autoscaling)
  --worker-machine-type=n1-standard-8 \        # Worker VM type
  --worker-region=us-central1 \               # Worker region
  --subnetwork=regions/us-central1/subnetworks/my-subnet \
  --no-use-public-ips \                        # Private IP workers
  --service-account-email=sa@proj.iam.gserviceaccount.com \
  --dataflow-kms-key=projects/p/locations/us/keyRings/kr/cryptoKeys/k \
  --enable-streaming-engine \                  # Enable Streaming Engine
  --experiments=enable_recommendations \
  --parameters key1=value1,key2=value2
```

---

### Running Locally (DirectRunner)

```python
# Local testing with DirectRunner — no GCP resources needed
options = PipelineOptions(
    runner='DirectRunner',
    direct_num_workers=4,               # Parallel local workers
    direct_running_mode='multi_threading'
)

with beam.Pipeline(options=options) as p:
    (p
     | beam.io.ReadFromText('local_input.txt')
     | beam.Map(str.upper)
     | beam.io.WriteToText('/tmp/output'))
```

> 💡 Always test with **DirectRunner** locally before submitting to Dataflow. It's instant, free, and catches most logical bugs.

---

### CI/CD with Cloud Build

```yaml
# cloudbuild.yaml — build, test, and deploy Flex Template
steps:
  # Step 1: Run unit tests
  - name: 'python:3.11'
    entrypoint: 'pip'
    args: ['install', '-r', 'requirements-test.txt']
  - name: 'python:3.11'
    entrypoint: 'python'
    args: ['-m', 'pytest', 'tests/', '-v']

  # Step 2: Build and push Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/my-pipeline:$COMMIT_SHA', '.']
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/my-pipeline:$COMMIT_SHA']

  # Step 3: Stage Flex Template
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'gcloud'
    args:
      - dataflow
      - flex-template
      - build
      - 'gs://$_BUCKET/templates/my-pipeline-$COMMIT_SHA.json'
      - '--image=gcr.io/$PROJECT_ID/my-pipeline:$COMMIT_SHA'
      - '--sdk-language=PYTHON'
      - '--metadata-file=metadata.json'

substitutions:
  _BUCKET: my-templates-bucket
```

---

### Google-Provided Templates (Key Ones)

| Template | Description |
|---|---|
| **Pub/Sub to BigQuery** | Stream messages from Pub/Sub → BigQuery with DLQ |
| **GCS to BigQuery** | Batch load files from GCS → BigQuery |
| **Datastream to BigQuery** | CDC from MySQL/PostgreSQL/Oracle → BigQuery |
| **BigQuery to GCS** | Export BigQuery table → GCS (Avro/CSV/Parquet) |
| **Pub/Sub to GCS** | Archive Pub/Sub messages → GCS |
| **GCS to Pub/Sub** | Replay files from GCS → Pub/Sub |
| **Spanner to BigQuery** | Bulk export Spanner → BigQuery |
| **Text to BigQuery** | Load text/CSV from GCS → BigQuery with transforms |

```bash
# List all available Google-provided templates
gcloud dataflow flex-template list --region=us-central1
```

---

## 8. Autoscaling, Performance & Optimization

### Horizontal Autoscaling

```python
# Configure autoscaling in pipeline options
options = PipelineOptions(
    num_workers=2,           # Starting workers
    max_num_workers=100,     # Upper bound
    # Dataflow will scale between 2 and 100 based on throughput
    autoscaling_algorithm='THROUGHPUT_BASED'  # or NONE to disable
)
```

Dataflow measures **worker backlog** (unprocessed bytes / elements) and adds workers when backlog grows, removes workers when it shrinks.

---

### Dataflow Shuffle (Batch)

| Mode | Shuffle Location | Memory Use | Speed |
|---|---|---|---|
| **Standard** | Worker-to-worker | High | Baseline |
| **Dataflow Shuffle** | Managed service | Low | 5x faster, 70% less memory |

```bash
# Enable Dataflow Shuffle (batch jobs — enabled by default in newer SDK versions)
--experiments=shuffle_mode=service
```

---

### Streaming Engine

```bash
# Enable Streaming Engine (streaming jobs)
--enable_streaming_engine

# Or in Python
options.view_as(GoogleCloudOptions).enable_streaming_engine = True
```

**Benefits:**
- State stored in managed backend (not worker memory) → workers use less RAM
- Faster worker scaling (no state migration needed)
- Better pipeline update support

> ⚠️ Streaming Engine incurs a small per-vCPU-hour premium (~10–20%). It is almost always worth it for production streaming pipelines.

---

### Fusion and When to Break It

```
Without Reshuffle (fused):
  ReadPubSub → ParseJSON → FilterValid → WriteBQ
       ↑ All on the SAME worker process, no parallelism boundary

With Reshuffle (broken fusion):
  ReadPubSub → ParseJSON → [RESHUFFLE] → FilterValid → WriteBQ
                                ↑ Forces redistribution; downstream can scale
```

```python
# Break fusion after an expensive transform
(
    data
    | 'Expand'    >> beam.FlatMap(expensive_expand)   # 1 → 100 elements
    | 'Reshuffle' >> beam.Reshuffle()                 # Redistribute 100x more elements
    | 'Process'   >> beam.ParDo(process_each)         # Now 100x parallelism
)
```

---

### Batching in DoFns (Throughput)

```python
# ❌ Slow: one API call per element
class CallAPIDoFn(beam.DoFn):
    def process(self, element):
        result = api_client.call(element)   # 1 call per element
        yield result

# ✅ Fast: batch API calls using finish_bundle
class BatchedAPIDoFn(beam.DoFn):
    def start_bundle(self):
        self.buffer = []

    def process(self, element):
        self.buffer.append(element)
        if len(self.buffer) >= 100:
            yield from self._flush()

    def finish_bundle(self):
        yield from self._flush()

    def _flush(self):
        if not self.buffer:
            return
        results = api_client.batch_call(self.buffer)  # 1 call per 100 elements
        self.buffer = []
        for r in results:
            yield beam.utils.windowed_value.WindowedValue(
                r, 0, [beam.transforms.window.GlobalWindow()])
```

---

### Anti-Patterns

| ❌ Anti-Pattern | 🔴 Problem | ✅ Fix |
|---|---|---|
| Calling external API per element | Extremely slow, quota exhaustion | Batch in `finish_bundle` (100+ per call) |
| Using Python `print()` in DoFn | Not captured in Cloud Logging | Use `logging.info()` |
| Large side inputs (GB+) | Workers OOM; slow broadcast | Use Bigtable or Spanner for enrichment |
| Storing state in class variables | Not thread-safe; lost across bundles | Use Beam state API (`ValueState`, etc.) |
| Ignoring hot keys | One worker bottleneck | Use `withHotKeyFanout` or two-phase aggregation |
| No DLQ for parsing | Silent data loss on malformed records | Add tagged output for errors |
| `GroupByKey` without Combine | Excessive shuffle | Lift combining upstream with `CombinePerKey` |
| Missing `Reshuffle` after FlatMap | Downstream stages under-parallelized | Add `Reshuffle` after element explosion |
| Not using Streaming Engine | High worker memory for state | Always enable for streaming pipelines |
| Hardcoded credentials in DoFn | Security risk | Use Workload Identity / Secret Manager |

---

## 9. Monitoring, Observability & Debugging

### Dataflow Monitoring UI

**Console path:** `GCP Console → Dataflow → Jobs → [select job]`

Key panels:
- **Job Graph** — visual DAG of all transforms and stages; click a stage to see metrics
- **Stage Metrics** — elements added, elements processed, wall time, CPU time
- **Autoscaling History** — timeline of worker count changes
- **Worker Logs** — per-worker log output; filter by severity
- **Job Metrics** — system lag (streaming), data freshness, bytes processed

---

### Key Metrics

| Metric | Type | Description | Alert On |
|---|---|---|---|
| `system_lag` | Streaming | Delay between oldest unprocessed event and now (seconds) | > threshold for your SLA |
| `data_freshness` | Streaming | Age of most recent output data | Growing unboundedly |
| `elements_added` | Both | Elements entering each stage | Sudden drop to 0 |
| `elapsed_time` | Batch | Wall-clock time since job start | > SLA threshold |
| `current_num_vcpus` | Both | Number of active vCPUs | Pinned at max_workers |
| `failed_element_count` | Both | Elements sent to error output | > 0 (if unexpected) |

---

### Cloud Monitoring Alerts

```bash
# Create an alert for high system lag (streaming pipeline)
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="Dataflow High System Lag" \
  --condition-filter='
    metric.type="dataflow.googleapis.com/job/system_lag"
    resource.type="dataflow_job"
    resource.labels.job_name="my-streaming-job"' \
  --condition-threshold-value=60 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=300s \
  --project=my-project
```

---

### Structured Logging from DoFns

```python
import logging
import apache_beam as beam

# Configure structured logging (captured by Cloud Logging)
logger = logging.getLogger(__name__)

class MyDoFn(beam.DoFn):
    def process(self, element):
        logger.info('Processing element', extra={
            'json_fields': {
                'user_id': element.get('user_id'),
                'amount':  element.get('amount'),
                'step':    'my-do-fn',
            }
        })
        try:
            result = transform(element)
            yield result
        except ValueError as e:
            logger.error('Transformation failed', extra={
                'json_fields': {'error': str(e), 'element': str(element)}
            })
```

```bash
# Query Dataflow worker logs in Cloud Logging
gcloud logging read \
  'resource.type="dataflow_step"
   resource.labels.job_name="my-pipeline"
   severity>=WARNING' \
  --project=my-project \
  --limit=100 \
  --format="table(timestamp,severity,textPayload)"
```

---

### Debugging Strategies

| Strategy | When | How |
|---|---|---|
| **DirectRunner** | Before submission | Test logic locally with small datasets |
| **Drain** | Stop streaming gracefully | Finish in-flight work, commit state; job completes cleanly |
| **Cancel** | Stop immediately | Discards all in-flight work; use only for stuck jobs |
| **Snapshot** | Pause streaming | Save current state to GCS; resume later from snapshot |
| **Pipeline Update** | Deploy new code | Update streaming job in-place (same job_name, compatible graph) |

```bash
# Drain a streaming job (graceful stop)
gcloud dataflow jobs drain JOB_ID --region=us-central1

# Cancel a job immediately
gcloud dataflow jobs cancel JOB_ID --region=us-central1

# Update a streaming job in place (compatible graph changes only)
python my_pipeline.py \
  --runner=DataflowRunner \
  --update \
  --job_name=my-existing-streaming-job \
  [... other options ...]
```

---

## 10. Access Control, Security & Networking

### IAM Roles

| Role | Who Needs It | Grants |
|---|---|---|
| `roles/dataflow.admin` | Ops/DevOps | Create, update, cancel jobs; view all jobs |
| `roles/dataflow.developer` | Data engineers | Submit jobs, view jobs and metrics |
| `roles/dataflow.viewer` | Analysts, auditors | Read-only access to jobs and metrics |
| `roles/dataflow.worker` | Worker service account | Required on the **worker SA** (not humans) |

---

### Service Accounts: Controller vs. Worker

```
JOB SUBMISSION                         WORKERS
     │                                    │
     ▼                                    ▼
Controller SA                         Worker SA
(submits the job)                (runs on GCE workers)
     │                                    │
Needs:                               Needs:
  dataflow.developer                   dataflow.worker
  iam.serviceAccountUser             bigquery.dataEditor  (if writing BQ)
  (on worker SA)                     pubsub.subscriber    (if reading PS)
                                     storage.objectAdmin  (tmp/staging)
```

```bash
# Grant worker SA the dataflow.worker role
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:worker-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/dataflow.worker"

# Grant worker SA access to BigQuery (for BQ writes)
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:worker-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataEditor"

# Grant worker SA access to GCS temp bucket
gsutil iam ch \
  serviceAccount:worker-sa@my-project.iam.gserviceaccount.com:roles/storage.objectAdmin \
  gs://my-temp-bucket
```

---

### VPC Networking & Private IP

```bash
# Run workers in a custom VPC subnet (private IPs only)
python my_pipeline.py \
  --runner=DataflowRunner \
  --subnetwork=regions/us-central1/subnetworks/my-private-subnet \
  --no_use_public_ips \         # Workers get no external IPs
  --region=us-central1

# Required: enable Private Google Access on the subnet
gcloud compute networks subnets update my-private-subnet \
  --region=us-central1 \
  --enable-private-ip-google-access
```

**Firewall rules required between Dataflow workers:**

```bash
# Workers need to communicate with each other (for shuffle)
gcloud compute firewall-rules create allow-dataflow-workers \
  --network=my-vpc \
  --allow=tcp:12345-12346 \          # Dataflow worker ports
  --source-tags=dataflow \
  --target-tags=dataflow \
  --description="Allow Dataflow worker-to-worker communication"
```

---

### CMEK with Cloud KMS

```bash
# Encrypt pipeline state, shuffle data, and temp files with CMEK
python my_pipeline.py \
  --runner=DataflowRunner \
  --dataflow_kms_key=projects/my-project/locations/us-central1/keyRings/my-ring/cryptoKeys/my-key \
  --temp_location=gs://cmek-protected-bucket/tmp \
  [... other options ...]

# Grant Dataflow service account access to the KMS key
gcloud kms keys add-iam-policy-binding my-key \
  --location=us-central1 \
  --keyring=my-ring \
  --member="serviceAccount:service-PROJECT_NUMBER@dataflow-service-producer-prod.iam.gserviceaccount.com" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"
```

---

## 11. gcloud CLI & REST API Quick Reference

### Essential gcloud dataflow Commands

```bash
# ── JOBS ──────────────────────────────────────────────────────────────
# List all jobs
gcloud dataflow jobs list \
  --region=us-central1 \
  --status=active \
  --format="table(id,name,currentState,startTime)"

# Describe a job (full metadata)
gcloud dataflow jobs describe JOB_ID \
  --region=us-central1 \
  --format=json

# Cancel a job (immediate, discards in-flight work)
gcloud dataflow jobs cancel JOB_ID --region=us-central1

# Drain a job (graceful stop for streaming)
gcloud dataflow jobs drain JOB_ID --region=us-central1

# Show job execution steps
gcloud dataflow jobs export-steps JOB_ID \
  --region=us-central1

# Run a classic template
gcloud dataflow jobs run JOB_NAME \
  --gcs-location=gs://bucket/template \
  --region=us-central1 \
  --parameters key1=val1,key2=val2

# Run a flex template
gcloud dataflow flex-template run JOB_NAME \
  --template-file-gcs-location=gs://bucket/template.json \
  --region=us-central1 \
  --parameters key1=val1

# ── SNAPSHOTS ─────────────────────────────────────────────────────────
# Create a snapshot (pause streaming job, save state)
gcloud dataflow snapshots create \
  --job-id=JOB_ID \
  --region=us-central1 \
  --snapshot-location=gs://bucket/snapshots \
  --ttl=72h

# List snapshots
gcloud dataflow snapshots list --region=us-central1

# ── FORMATTING ────────────────────────────────────────────────────────
--format=json           # Full JSON output
--format=yaml           # YAML output
--format="table(field1,field2)"   # Custom table
--filter="currentState=JOB_STATE_RUNNING"
```

---

### Key Pipeline Options → gcloud Flags

| Python Pipeline Option | gcloud Flag |
|---|---|
| `--project` | `--project` |
| `--region` | `--region` |
| `--num_workers` | `--num-workers` |
| `--max_num_workers` | `--max-workers` |
| `--machine_type` | `--worker-machine-type` |
| `--subnetwork` | `--subnetwork` |
| `--no_use_public_ips` | `--disable-public-ips` |
| `--service_account_email` | `--service-account-email` |
| `--dataflow_kms_key` | `--dataflow-kms-key` |
| `--enable_streaming_engine` | `--enable-streaming-engine` |

---

### Python Client Library

```python
# ── Submit a pipeline programmatically ───────────────────────────────
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

options = PipelineOptions(flags=[], **{
    'runner': 'DataflowRunner',
    'project': 'my-project',
    'region': 'us-central1',
    'temp_location': 'gs://bucket/tmp',
    'job_name': 'my-job',
})

pipeline = beam.Pipeline(options=options)
# ... build pipeline ...
result = pipeline.run()
print(f"Job ID: {result.job_id()}")
# result.wait_until_finish()  # Block until complete

# ── Check job status via REST (googleapiclient) ────────────────────────
from googleapiclient.discovery import build
from google.oauth2 import service_account

service = build('dataflow', 'v1b3')
response = service.projects().locations().jobs().get(
    projectId='my-project',
    location='us-central1',
    jobId='JOB_ID'
).execute()

print(f"Job state: {response['currentState']}")
```

---

### REST API Quick Reference

```bash
# List jobs
curl -X GET \
  "https://dataflow.googleapis.com/v1b3/projects/MY_PROJECT/locations/us-central1/jobs" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)"

# Get job details
curl -X GET \
  "https://dataflow.googleapis.com/v1b3/projects/MY_PROJECT/locations/us-central1/jobs/JOB_ID" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)"

# Update job (drain or cancel)
curl -X PUT \
  "https://dataflow.googleapis.com/v1b3/projects/MY_PROJECT/locations/us-central1/jobs/JOB_ID" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"requestedState": "JOB_STATE_DRAINING"}'
```

---

## 12. Pricing Summary

> ⚠️ Prices are approximate as of early 2026 (us-central1). Verify at [cloud.google.com/dataflow/pricing](https://cloud.google.com/dataflow/pricing).

### Classic Dataflow Billing

| Resource | Price | Unit |
|---|---|---|
| **vCPU** | ~$0.056 | per vCPU-hour |
| **RAM** | ~$0.003375 | per GB RAM-hour |
| **Persistent Disk (Standard)** | ~$0.000054 | per GB-hour |
| **Persistent Disk (SSD)** | ~$0.000298 | per GB-hour |
| **Streaming Engine** | ~$0.0105 | per vCPU-hour (additional) |
| **Dataflow Shuffle** | ~$0.011 | per GB processed |

### Dataflow Prime Billing

| Resource | Price | Unit |
|---|---|---|
| **RSU (Resource Scheduling Unit)** | ~$0.069 | per RSU-hour |

> 💡 Dataflow Prime RSUs abstract away vCPU + RAM + Disk into a single unit, with vertical autoscaling per stage.

---

### Cost Optimization Tips

```bash
# 1. Use Spot/preemptible workers for fault-tolerant batch jobs (60-91% savings)
python pipeline.py \
  --use_public_ips=false \
  --experiments=use_runner_v2 \
  --worker_accelerator_type=preemptible  # or --use_preemptible_workers

# 2. Enable Streaming Engine for streaming (reduces worker count)
--enable_streaming_engine

# 3. Enable Dataflow Shuffle for batch (reduces shuffle time and worker memory)
--experiments=shuffle_mode=service

# 4. Right-size worker machine type
--machine_type=n1-highmem-4    # For memory-intensive (stateful, large side inputs)
--machine_type=n1-highcpu-8    # For CPU-intensive (parsing, transformations)
--machine_type=c2-standard-4   # For compute-optimized (ML inference)
```

---

### Monthly Cost Estimate (Streaming Pipeline)

```
Scenario: Streaming pipeline, 24/7, 8 workers × n1-standard-4
  (4 vCPU, 15 GB RAM, 50 GB disk each)

vCPU cost:    8 × 4 × $0.056 × 730h  = $1,310/month
RAM cost:     8 × 15 × $0.00338 × 730 = $296/month
Disk cost:    8 × 50 × $0.000054 × 730 = $16/month
Stream Eng:   8 × 4 × $0.0105 × 730   = $245/month
─────────────────────────────────────────────────────
TOTAL:                                 ~$1,867/month

With Spot workers (60% discount on vCPU+RAM+Disk):
  Compute savings:  ~$973/month
  Total with Spot: ~$1,139/month
```

> ⚠️ Streaming jobs with Spot workers will be preempted occasionally. Dataflow automatically restores state from Streaming Engine — design for preemption tolerance.

---

## 13. Quick Reference & Comparison Tables

### Beam Transform Cheatsheet

| Transform | Input | Output | Use Case |
|---|---|---|---|
| `ParDo` | PCollection<A> | PCollection<B> | Any per-element transformation |
| `Map` | PCollection<A> | PCollection<B> | Simple 1:1 mapping |
| `FlatMap` | PCollection<A> | PCollection<B> | 1:N mapping (one to many) |
| `Filter` | PCollection<A> | PCollection<A> | Keep elements matching predicate |
| `GroupByKey` | PCollection<KV<K,V>> | PCollection<KV<K,Iter<V>>> | Group values by key |
| `CoGroupByKey` | Multiple PCollection<KV<K,V>> | PCollection<KV<K,CoGbkResult>> | Join multiple KV collections |
| `CombinePerKey` | PCollection<KV<K,V>> | PCollection<KV<K,Out>> | Aggregate per key |
| `CombineGlobally` | PCollection<V> | PCollection<Out> | Single global aggregate |
| `Flatten` | Multiple PCollections<A> | PCollection<A> | Merge PCollections |
| `Partition` | PCollection<A> | N × PCollection<A> | Split into N sub-collections |
| `Reshuffle` | PCollection<A> | PCollection<A> | Break fusion, redistribute |
| `WindowInto` | PCollection<A> | PCollection<A> | Assign windowing strategy |
| `Distinct` | PCollection<A> | PCollection<A> | Remove duplicates |
| `Sample` | PCollection<A> | PCollection<A> | Random sample N elements |

---

### Window Type Comparison

| Type | Overlap | Size | Key Characteristic | Best For |
|---|---|---|---|---|
| `FixedWindows` | None | Fixed | Clean, non-overlapping buckets | Hourly/daily batches |
| `SlidingWindows` | Yes | Fixed | Every window covers last N seconds | Rolling averages |
| `Sessions` | None | Variable | Gap-based, per-key | User session analytics |
| `GlobalWindow` | N/A | Infinite | One window for all data | Batch pipelines |

---

### State Type Comparison

| State Type | Read | Write | Use Case |
|---|---|---|---|
| `ValueState` | Single value | Overwrite | Counter, last-seen value, dedup |
| `BagState` | Iterable | Append-only | Buffer for micro-batching |
| `SetState` | Set membership | Add | Bloom-filter style dedup |
| `MapState` | By key | Put/remove | Per-key lookup within element |
| `CombiningState` | Merged value | Merge with CombineFn | Running totals, sliding sums |

---

### Classic Template vs. Flex Template

| Feature | Classic Template | Flex Template |
|---|---|---|
| **Packaging** | Compiled pipeline graph (JSON) | Docker container |
| **Dependencies** | Frozen at build time | Install at runtime |
| **Custom connectors** | ❌ Very limited | ✅ Any library |
| **Build time** | Fast (no container) | Slower (Docker build) |
| **Flexibility** | Low | High |
| **Startup latency** | ~1–2 min | ~3–5 min (container pull) |
| **Best for** | Simple, stable pipelines | Complex deps, custom I/O |
| **YAML pipelines** | N/A | ✅ Supported |

---

### Dataflow vs. Dataproc vs. BigQuery

| Dimension | Dataflow | Dataproc (Spark) | BigQuery |
|---|---|---|---|
| **Abstraction** | Beam transforms | Spark RDDs / DataFrames | SQL |
| **Streaming** | ✅ Native, sub-minute | ⚠️ Structured Streaming | ⚠️ Streaming inserts |
| **Custom code** | ✅ Arbitrary DoFn logic | ✅ Arbitrary Spark code | ❌ SQL + UDFs only |
| **State management** | ✅ ValueState, BagState | ✅ Spark stateful ops | ❌ None |
| **Management** | Fully serverless | Semi-managed | Fully serverless |
| **Startup time** | 2–5 min | 3–10 min (cluster) | Near-instant |
| **Cost model** | Per vCPU/GB-hr | Per cluster-hr | Per TB scanned |
| **Best for** | Complex streaming ETL, CDC | Spark workloads, ML | SQL analytics, ad-hoc |

---

### Common Pipeline Anti-Patterns

| ❌ Anti-Pattern | Impact | ✅ Fix |
|---|---|---|
| API call per element in `process()` | 1000x slower than batching | Batch in `finish_bundle()` |
| Large mutable class variable | Data corruption across bundles | Use Beam state API |
| Python `print()` for logging | Logs lost or unstructured | `logging.getLogger().info()` |
| No error handling in `process()` | Silent data loss | Use tagged outputs for DLQ |
| Side input > 1 GB | Worker OOM | Use Bigtable/Spanner enrichment |
| Unkeyed data before `GroupByKey` | Full shuffle of all data | Add `beam.Map(lambda x: (key, x))` |
| No `Reshuffle` after FlatMap | Under-parallelized downstream | Add `beam.Reshuffle()` |
| Streaming job without Streaming Engine | High memory pressure | `--enable_streaming_engine` |
| Not setting `max_num_workers` | Runaway cost | Always set a safe upper bound |
| Using Cancel on streaming job | Lost in-flight data | Use Drain for graceful stop |

---

*Generated for GCP Dataflow | Official Docs: [cloud.google.com/dataflow/docs](https://cloud.google.com/dataflow/docs) | Apache Beam: [beam.apache.org](https://beam.apache.org)*
