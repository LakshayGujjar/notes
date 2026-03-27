# GCP Observability Suite — Comprehensive Cheatsheet
## Cloud Monitoring · Cloud Logging · Cloud Trace · Cloud Profiler · Error Reporting

> **Audience:** Senior GCP engineers and SREs. No hand-holding — straight to operational depth.

---

## Table of Contents

1. [Overview & Observability Philosophy](#1-overview--observability-philosophy)
2. [Cloud Monitoring — Concepts & Architecture](#2-cloud-monitoring--concepts--architecture)
3. [Cloud Monitoring — Custom Metrics](#3-cloud-monitoring--custom-metrics)
4. [Cloud Monitoring — Google Managed Service for Prometheus](#4-cloud-monitoring--google-managed-service-for-prometheus-gmp)
5. [Cloud Monitoring — Alerting](#5-cloud-monitoring--alerting)
6. [Cloud Monitoring — Dashboards & MQL](#6-cloud-monitoring--dashboards--mql)
7. [Cloud Monitoring — Terraform](#7-cloud-monitoring--terraform)
8. [Cloud Logging — Concepts & Architecture](#8-cloud-logging--concepts--architecture)
9. [Cloud Logging — Writing Logs](#9-cloud-logging--writing-logs)
10. [Cloud Logging — Log Routing & Sinks](#10-cloud-logging--log-routing--sinks)
11. [Cloud Logging — Querying & Log Analytics](#11-cloud-logging--querying--log-analytics)
12. [Cloud Logging — Terraform](#12-cloud-logging--terraform)
13. [Cloud Trace — Concepts & Architecture](#13-cloud-trace--concepts--architecture)
14. [Cloud Trace — Instrumentation](#14-cloud-trace--instrumentation)
15. [Cloud Trace — gcloud & Analysis](#15-cloud-trace--gcloud--analysis)
16. [Cloud Profiler — Concepts & Architecture](#16-cloud-profiler--concepts--architecture)
17. [Cloud Profiler — Agent Setup](#17-cloud-profiler--agent-setup)
18. [Cloud Profiler — Analysis & Best Practices](#18-cloud-profiler--analysis--best-practices)
19. [Error Reporting — Concepts & Architecture](#19-error-reporting--concepts--architecture)
20. [Error Reporting — Instrumentation](#20-error-reporting--instrumentation)
21. [OpenTelemetry Collector — GCP Setup](#21-opentelemetry-collector--gcp-setup)
22. [Observability for GKE — Complete Setup](#22-observability-for-gke--complete-setup)
23. [Key IAM Roles for Observability](#23-key-iam-roles-for-observability)
24. [Limits & Quotas](#24-limits--quotas)
25. [Costs & Cost Optimization](#25-costs--cost-optimization)
26. [Common Gotchas & Troubleshooting](#26-common-gotchas--troubleshooting)
27. [Quick Reference — gcloud Commands](#27-quick-reference--gcloud-commands)
28. [Decision Tree](#28-decision-tree)

---

## 1. Overview & Observability Philosophy

### Signal Flow Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    GCP Observability Signal Flow                             │
└─────────────────────────────────────────────────────────────────────────────┘

  Your Application / Services
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Service A          Service B          Service C                     │
  │  (Cloud Run)        (GKE Pod)          (GCE VM)                      │
  └────┬────────────────────┬──────────────────┬───────────────────────┘
       │                    │                  │
       │  OTel SDK / Agent  │                  │
       ▼                    ▼                  ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │              OpenTelemetry Collector (DaemonSet / Sidecar)           │
  │   receivers: [otlp, prometheus]                                       │
  │   processors: [batch, memory_limiter, resourcedetection]             │
  │   exporters: [googlecloud, googlemanagedprometheus]                  │
  └───────┬──────────────┬───────────────┬───────────────────────────────┘
          │              │               │
    Metrics           Traces           Logs
          │              │               │
          ▼              ▼               ▼
  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────────────┐
  │   Cloud      │ │   Cloud      │ │   Cloud      │ │    Cloud       │
  │  Monitoring  │ │   Trace      │ │   Logging    │ │   Profiler     │
  │  (+ GMP)     │ │              │ │              │ │   (agent-side) │
  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └───────┬────────┘
         │                │                │                  │
         └────────────────┴────────────────┴──────────────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │    Error Reporting    │
                         │  (consumes logs +     │
                         │   monitoring signals) │
                         └──────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │         Insights & Actions     │
                    │  Dashboards · Alerts · SLOs   │
                    │  Incidents · Runbooks          │
                    └───────────────────────────────┘
```

### The Four Pillars + Error Aggregation

| Signal | GCP Product | OpenTelemetry Equivalent | When to Use |
|---|---|---|---|
| **Metrics** | Cloud Monitoring | OTel Metrics | Quantitative trends, alerting, SLOs |
| **Logs** | Cloud Logging | OTel Logs | Detailed event context, audit trails |
| **Distributed Traces** | Cloud Trace | OTel Traces | End-to-end latency, cross-service debugging |
| **Continuous Profiles** | Cloud Profiler | OTel Profiling (emerging) | CPU/memory hotspots, performance regressions |
| **Error Aggregation** | Error Reporting | — | Exception grouping, triage, alerting |

**When each signal applies:**

- **Metrics**: "Is my system behaving normally?" (latency, error rate, saturation) — always on, aggregated
- **Logs**: "What exactly happened at T?" — detailed context, but high volume; query on demand
- **Traces**: "Why is this request slow?" — cross-service causality; sampled
- **Profiles**: "What code is burning CPU/memory?" — production performance; continuous, low overhead
- **Error Reporting**: "What are the top errors right now, and which team owns them?" — synthesis layer

### Product Relationships & Data Sharing

```
Cloud Logging ──log-based metrics──▶ Cloud Monitoring (alert on log events)
Cloud Trace   ──trace+span IDs──────▶ Cloud Logging (correlate logs to traces)
Cloud Logging ──ERROR severity──────▶ Error Reporting (auto-group exceptions)
Cloud Monitoring ──error count──────▶ Error Reporting (alert on new groups)
Cloud Profiler ──(independent)──────▶ Cloud Console (flame graph UI)
GMP (Prometheus) ──PromQL metrics───▶ Cloud Monitoring backend (unified)
```

### GMP vs. Cloud Monitoring Custom Metrics

| Use Case | Recommendation |
|---|---|
| High-cardinality metrics (>10K time series) | Google Managed Service for Prometheus (GMP) |
| Low-cardinality business metrics | Cloud Monitoring custom metrics |
| Existing Prometheus stack migration | GMP with remote_write |
| New greenfield service | OTel SDK → OTel Collector → both |
| Alerting on GCP infra metrics | Cloud Monitoring (built-in metrics are free) |

---

## PART 1 — CLOUD MONITORING

## 2. Cloud Monitoring — Concepts & Architecture

### Metrics Scope (formerly Workspace)

```
Metrics Scope Project (central)
  ├── Hosts its own metrics
  └── Monitored Projects (can view metrics from these projects)
        ├── project-prod-api
        ├── project-prod-data
        └── project-staging

All projects' metrics visible in one unified Monitoring UI
```

- A **Metrics Scope** is created implicitly when you first use Cloud Monitoring in a project
- Add monitored projects: Console → Monitoring → Settings → Add GCP Projects
- Metrics from monitored projects are **read-only** in the scoping project — no data is copied

### Metric Types

| Kind | Description | Example | Write Frequency |
|---|---|---|---|
| **GAUGE** | Snapshot at a point in time | CPU utilization (0–1) | Every interval |
| **DELTA** | Change since last sample | HTTP request count | Every interval |
| **CUMULATIVE** | Monotonically increasing since process start | Total bytes sent | Every interval |

> **⚠️ Gotcha:** When alerting on DELTA metrics, you must align with `rate()` or `delta()` first. Alerting on raw cumulative values gives meaningless thresholds.

### Metric Descriptor Structure

```python
# Metric descriptor fields
{
  "type":        "custom.googleapis.com/order_service/request_latency_ms",
  "kind":        "GAUGE",          # GAUGE | DELTA | CUMULATIVE
  "valueType":   "DOUBLE",         # BOOL | INT64 | DOUBLE | STRING | DISTRIBUTION
  "unit":        "ms",             # UCUM unit notation
  "description": "HTTP request latency in milliseconds",
  "labels": [
    {"key": "endpoint", "valueType": "STRING", "description": "API endpoint path"},
    {"key": "method",   "valueType": "STRING", "description": "HTTP method"},
    {"key": "status",   "valueType": "INT64",  "description": "HTTP status code"},
  ],
  "monitoredResourceTypes": ["generic_task", "gce_instance", "k8s_container"]
}
```

### Metric Namespaces

| Namespace | Meaning | Cost |
|---|---|---|
| `compute.googleapis.com/` | GCE built-in metrics | Free |
| `run.googleapis.com/` | Cloud Run built-in | Free |
| `container.googleapis.com/` | GKE built-in | Free |
| `custom.googleapis.com/` | User-written custom metrics | Billed |
| `external.googleapis.com/` | Metrics from external systems (Datadog agent, etc.) | Billed |
| `workload.googleapis.com/` | OTel Collector exported (via GCP exporter) | Billed |
| `prometheus.googleapis.com/` | GMP-collected Prometheus metrics | Billed (per sample) |
| `logging.googleapis.com/user/` | Log-based metrics | Billed |

### Monitored Resource Types

| Resource Type | What It Represents | Key Labels |
|---|---|---|
| `gce_instance` | GCE VM | `project_id`, `instance_id`, `zone` |
| `k8s_container` | GKE container | `project_id`, `cluster_name`, `namespace_name`, `pod_name`, `container_name` |
| `k8s_pod` | GKE pod | `project_id`, `cluster_name`, `namespace_name`, `pod_name` |
| `k8s_node` | GKE node | `project_id`, `cluster_name`, `node_name` |
| `cloud_run_revision` | Cloud Run revision | `project_id`, `service_name`, `revision_name`, `location` |
| `gae_app` | App Engine | `project_id`, `module_id`, `version_id` |
| `generic_task` | Custom application task | `project_id`, `location`, `namespace`, `job`, `task_id` |
| `global` | Project-level (no specific resource) | `project_id` |

### Retention Periods

| Resolution | Retention |
|---|---|
| Raw (≤ 1 min alignment) | 6 weeks |
| 10-minute alignment | 6 months |
| 1-hour alignment | 1 year |

> **💡 Best Practice:** For dashboards showing > 6-week trends, either use log-based metrics with BigQuery export, or use GMP (24-month retention). Cloud Monitoring built-in retention is not suitable for YoY comparisons.

### Time Series Data Model

```
Time series key = (metric_type, resource_type, resource_labels, metric_labels)

Example:
  metric_type:     "compute.googleapis.com/instance/cpu/utilization"
  resource_type:   "gce_instance"
  resource_labels: {project_id: "my-project", zone: "us-central1-a", instance_id: "12345"}
  metric_labels:   {}

→ [(t1, 0.42), (t2, 0.55), (t3, 0.61), ...]

High cardinality warning: each unique combination of label values = a new time series
Labels with unbounded values (user IDs, request IDs) create cardinality explosion
```

---

## 3. Cloud Monitoring — Custom Metrics

### Custom vs. Log-Based Metrics

| Dimension | Custom Metrics | Log-Based Metrics |
|---|---|---|
| Data source | Pushed by application | Derived from log entries |
| Latency | Near real-time (~1 min) | Slightly delayed (log ingestion + processing) |
| Cost | Per time series + samples | Per log ingestion (separate from metric cost) |
| Best for | Business KPIs, app internals | Error counts, specific log event rates |
| Histogram support | Yes (DISTRIBUTION type) | Yes (distribution log-based metric) |
| Requires code change | Yes | No (if logs already exist) |

### Writing Custom Metrics — Cloud Monitoring API (Python)

```python
from google.cloud import monitoring_v3
import time

PROJECT_ID = "my-project"
client = monitoring_v3.MetricServiceClient()
project_name = f"projects/{PROJECT_ID}"

def write_gauge_metric(value: float, endpoint: str, method: str) -> None:
    """Write a GAUGE custom metric data point."""
    series = monitoring_v3.TimeSeries()
    series.metric.type = "custom.googleapis.com/order_service/request_latency_ms"
    series.metric.labels["endpoint"] = endpoint
    series.metric.labels["method"] = method
    series.metric.labels["status"] = "200"

    series.resource.type = "generic_task"
    series.resource.labels["project_id"] = PROJECT_ID
    series.resource.labels["location"] = "us-central1"
    series.resource.labels["namespace"] = "production"
    series.resource.labels["job"] = "order-service"
    series.resource.labels["task_id"] = "instance-1"  # Use instance hostname

    point = monitoring_v3.Point()
    point.value.double_value = value
    now = time.time()
    point.interval.end_time.seconds = int(now)
    point.interval.end_time.nanos = int((now - int(now)) * 10**9)
    series.points = [point]

    client.create_time_series(
        request={"name": project_name, "time_series": [series]}
    )

# Write multiple time series in one call (max 200 per request)
def write_batch_metrics(data_points: list) -> None:
    time_series_list = []
    now = time.time()

    for item in data_points:
        series = monitoring_v3.TimeSeries()
        series.metric.type = "custom.googleapis.com/order_service/request_count"
        series.metric.labels["endpoint"] = item["endpoint"]
        series.metric.labels["status"] = str(item["status"])
        series.resource.type = "generic_task"
        series.resource.labels.update({
            "project_id": PROJECT_ID, "location": "us-central1",
            "namespace": "production", "job": "order-service", "task_id": "instance-1"
        })
        point = monitoring_v3.Point()
        point.value.int64_value = item["count"]
        point.interval.end_time.seconds = int(now)
        series.points = [point]
        time_series_list.append(series)

    client.create_time_series(
        request={"name": project_name, "time_series": time_series_list}
    )
```

### Creating a Custom Metric Descriptor Explicitly

```python
from google.cloud import monitoring_v3
from google.api import label_pb2, metric_pb2

def create_metric_descriptor(project_id: str) -> None:
    client = monitoring_v3.MetricServiceClient()
    project_name = f"projects/{project_id}"

    descriptor = monitoring_v3.MetricDescriptor()
    descriptor.type = "custom.googleapis.com/order_service/checkout_duration_ms"
    descriptor.metric_kind = monitoring_v3.MetricDescriptor.MetricKind.GAUGE
    descriptor.value_type = monitoring_v3.MetricDescriptor.ValueType.DISTRIBUTION
    descriptor.unit = "ms"
    descriptor.description = "Checkout flow end-to-end duration in ms"
    descriptor.display_name = "Checkout Duration"

    label = label_pb2.LabelDescriptor()
    label.key = "payment_method"
    label.value_type = label_pb2.LabelDescriptor.ValueType.STRING
    label.description = "Payment method used (card, paypal, etc.)"
    descriptor.labels.append(label)

    descriptor.monitored_resource_types.append("generic_task")

    descriptor = client.create_metric_descriptor(
        name=project_name, metric_descriptor=descriptor
    )
    print(f"Created: {descriptor.name}")
```

### OpenTelemetry SDK → Cloud Monitoring (Python)

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.cloud_monitoring import CloudMonitoringMetricsExporter
from opentelemetry.sdk.resources import Resource

# Initialize once at application startup
resource = Resource.create({
    "service.name": "order-service",
    "service.version": "v2.3.1",
    "deployment.environment": "production",
})

exporter = CloudMonitoringMetricsExporter(project_id=PROJECT_ID)
reader = PeriodicExportingMetricReader(exporter, export_interval_millis=60_000)
provider = MeterProvider(metric_readers=[reader], resource=resource)
metrics.set_meter_provider(provider)

meter = metrics.get_meter("order-service", version="v2.3.1")

# Counter (maps to DELTA in Cloud Monitoring)
request_counter = meter.create_counter(
    name="requests_total",
    description="Total HTTP requests served",
    unit="1",
)

# Histogram (maps to DISTRIBUTION in Cloud Monitoring)
latency_histogram = meter.create_histogram(
    name="request_duration_ms",
    description="HTTP request duration",
    unit="ms",
)

# UpDownCounter (maps to GAUGE)
active_connections = meter.create_up_down_counter(
    name="active_connections",
    description="Current active HTTP connections",
    unit="1",
)

# Usage in request handler
def handle_request(method: str, endpoint: str, status: int, duration_ms: float):
    attrs = {"method": method, "endpoint": endpoint, "status": str(status)}
    request_counter.add(1, attrs)
    latency_histogram.record(duration_ms, attrs)
```

### OpenTelemetry SDK → Cloud Monitoring (Go)

```go
package main

import (
    "context"
    "log"
    "time"

    mexporter "github.com/GoogleCloudPlatform/opentelemetry-operations-go/exporter/metric"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/metric"
    sdkmetric "go.opentelemetry.io/otel/sdk/metric"
    "go.opentelemetry.io/otel/sdk/resource"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

func initMetrics(projectID string) (*sdkmetric.MeterProvider, error) {
    exporter, err := mexporter.New(mexporter.WithProjectID(projectID))
    if err != nil {
        return nil, err
    }
    res, _ := resource.New(context.Background(),
        resource.WithAttributes(
            semconv.ServiceName("order-service"),
            semconv.ServiceVersion("v2.3.1"),
            attribute.String("deployment.environment", "production"),
        ),
    )
    mp := sdkmetric.NewMeterProvider(
        sdkmetric.WithReader(
            sdkmetric.NewPeriodicReader(exporter, sdkmetric.WithInterval(60*time.Second)),
        ),
        sdkmetric.WithResource(res),
    )
    otel.SetMeterProvider(mp)
    return mp, nil
}

func main() {
    mp, err := initMetrics("my-project")
    if err != nil {
        log.Fatal(err)
    }
    defer mp.Shutdown(context.Background())

    meter := otel.Meter("order-service")
    requestCount, _ := meter.Int64Counter("requests_total",
        metric.WithDescription("Total HTTP requests"),
        metric.WithUnit("1"),
    )
    requestDuration, _ := meter.Float64Histogram("request_duration_ms",
        metric.WithDescription("HTTP request duration"),
        metric.WithUnit("ms"),
    )

    // In handler:
    attrs := metric.WithAttributes(
        attribute.String("method", "GET"),
        attribute.String("endpoint", "/api/orders"),
        attribute.Int("status", 200),
    )
    requestCount.Add(context.Background(), 1, attrs)
    requestDuration.Record(context.Background(), 142.7, attrs)
}
```

> **💡 Best Practice:** Use `generic_task` as the monitored resource type for custom applications. It has the most flexible label set. Avoid `global` — it doesn't support per-instance breakdowns, making it useless for multi-instance services.

> **⚠️ Gotcha:** Custom metric labels are permanent once a time series is written. You cannot rename or delete labels without deleting the entire metric descriptor and losing all historical data.

---

## 4. Cloud Monitoring — Google Managed Service for Prometheus (GMP)

### Architecture

```
GKE Cluster
  ├── Application Pods (expose /metrics endpoint)
  │     └── PodMonitoring CR selects pods → scrapes /metrics
  │
  ├── gmp-system namespace
  │     ├── collector (DaemonSet) — scrapes PodMonitorings
  │     └── rule-evaluator — evaluates recording/alerting rules
  │
  └── Data flows to: Cloud Monitoring backend (prometheus.googleapis.com/)
        └── Queryable via PromQL through Cloud Monitoring API or Grafana
```

### GMP vs. Self-Managed Prometheus

| Dimension | GMP | Self-Managed Prometheus |
|---|---|---|
| Retention | 24 months | Configurable (storage cost) |
| Scaling | Automatic (global) | Manual (federation, Thanos, Cortex) |
| High Availability | Built-in, multi-region | Complex setup (2x Prometheus + dedup) |
| Cost | Per sample ingested ($0.09/M after free tier) | Infrastructure + ops cost |
| Remote write support | Yes (`remoteWrite` in PodMonitoring) | Yes |
| PromQL | Full support | Full support |
| Alertmanager | Not included (use Cloud Monitoring alerts) | Included |
| Long-term storage | Cloud Monitoring handles | External (Thanos/Cortex) |
| Cardinality limits | High (Cloud Monitoring backend) | Limited by RAM |

### Enable GMP on GKE

```bash
# Enable Managed Service for Prometheus on an existing cluster
gcloud container clusters update CLUSTER_NAME \
  --enable-managed-prometheus \
  --region=REGION \
  --project=PROJECT_ID

# Or at cluster creation
gcloud container clusters create CLUSTER_NAME \
  --enable-managed-prometheus \
  --workload-pool=PROJECT_ID.svc.id.goog \
  --region=REGION
```

### PodMonitoring CRD

```yaml
# pod-monitoring.yaml — scrapes pods matching the selector in a namespace
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: order-service-monitoring
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
      tier: backend
  endpoints:
  - port: metrics          # Named port in the pod spec
    interval: 30s
    path: /metrics
    timeout: 10s
    # Optional: scrape only specific metric families
    # metricRelabeling:
    # - sourceLabels: [__name__]
    #   regex: "http_requests_.*|process_cpu_.*"
    #   action: keep
  targetLabels:
    metadata:
    - pod
    - container
    - node
    - namespace
```

```yaml
# cluster-pod-monitoring.yaml — cluster-wide scraping (all namespaces)
apiVersion: monitoring.googleapis.com/v1
kind: ClusterPodMonitoring
metadata:
  name: cluster-wide-scraping
spec:
  selector:
    matchLabels:
      prometheus.io/scrape: "true"
  endpoints:
  - port: metrics
    interval: 30s
  targetLabels:
    metadata:
    - pod
    - namespace
    - node
```

### Rules CRD (Recording & Alerting Rules)

```yaml
# rules.yaml — recording rules and alerting rules for GMP
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: order-service-rules
  namespace: production
spec:
  groups:
  - name: order_service_recording
    interval: 1m
    rules:
    # Recording rule: pre-aggregate request rate
    - record: job:http_requests_total:rate5m
      expr: sum(rate(http_requests_total[5m])) by (job, endpoint, status_code)

  - name: order_service_alerts
    rules:
    # Alerting rule: high error rate
    - alert: HighErrorRate
      expr: |
        sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (job)
        /
        sum(rate(http_requests_total[5m])) by (job)
        > 0.01
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High error rate on {{ $labels.job }}"
        description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.job }}"
```

### Querying GMP with PromQL

```bash
# Query GMP metrics via gcloud (PromQL)
gcloud monitoring query --project=PROJECT_ID \
  --query='sum(rate(prometheus_googleapis_com_http_requests_total[5m])) by (job)'

# Or via Grafana with the Cloud Monitoring data source configured
# Grafana datasource type: Google Cloud Monitoring
# Query type: Prometheus Query Language (PromQL)
```

> **💡 Best Practice:** Use GMP for any metric where the number of time series might exceed 10,000, or where 24-month retention is needed (capacity planning, YoY comparison). Use Cloud Monitoring custom metrics for low-cardinality business KPIs where you want simple alerting and dashboard integration.

---

## 5. Cloud Monitoring — Alerting

### Alert Policy Components

```
Alert Policy
  ├── Conditions (1 or more)
  │     ├── Condition type (threshold, absence, log-based, uptime, forecast, PromQL)
  │     ├── Metric filter (which metric + resource labels)
  │     ├── Aggregation (aligner + reducer + group_by)
  │     ├── Threshold + comparison
  │     └── Duration (how long condition must hold before firing)
  │
  ├── Combiner: OR (any condition) | AND (all conditions)
  │
  ├── Notification Channels (where to send)
  │
  ├── Documentation (runbook markdown, shown in incident)
  │
  └── Alert Strategy
        ├── auto_close: (auto-close incident after N seconds if no data)
        └── notification_rate_limit: (minimum time between repeat notifications)
```

### Condition Types

| Type | Use Case | Key Field |
|---|---|---|
| **Metric threshold** | CPU > 80%, latency > 1s | `threshold_value`, `comparison` |
| **Metric absence** | No data received in N minutes | `duration` |
| **Log-based metric** | Error count from logs > 0 | Log-based metric filter |
| **Uptime check failure** | HTTP endpoint unhealthy | `uptime_check_id` |
| **Forecasted threshold** | "Will breach in 1 hour" (ML) | `forecast_horizon` |
| **PromQL condition** | GMP/Prometheus expression | `query` (PromQL string) |

### MQL vs. PromQL for Alerting

| Scenario | Use MQL | Use PromQL |
|---|---|---|
| Cloud Monitoring built-in metrics | ✅ | ❌ (not available in PromQL conditions) |
| GMP/Prometheus metrics | ❌ | ✅ |
| Complex ratio alerts (error rate) | ✅ | ✅ |
| Cross-resource aggregations | ✅ | Limited |
| Existing Prometheus alerting rules migration | ❌ | ✅ |

### Notification Channels

| Channel | Use Case |
|---|---|
| Email | Low-urgency, non-paging |
| PagerDuty | High-severity on-call paging |
| Slack | Team awareness, non-paging |
| Cloud Pub/Sub | Custom webhook, SIEM integration |
| Webhook (HTTPS) | Custom integrations |
| OpsGenie | On-call rotation management |
| Cloud Mobile App | Individual mobile notifications |

```bash
# ── Notification Channels ─────────────────────────────────────────────────────

# Create Slack notification channel
gcloud alpha monitoring channels create \
  --display-name="SRE Slack — #alerts-production" \
  --type=slack \
  --channel-labels=channel_name="#alerts-production" \
  --channel-labels=auth_token=SLACK_BOT_TOKEN \
  --project=PROJECT_ID

# Create PagerDuty channel
gcloud alpha monitoring channels create \
  --display-name="PagerDuty Production" \
  --type=pagerduty \
  --channel-labels=service_key=PAGERDUTY_INTEGRATION_KEY \
  --project=PROJECT_ID

# Create Email channel
gcloud alpha monitoring channels create \
  --display-name="SRE Team Email" \
  --type=email \
  --channel-labels=email_address=sre-team@example.com \
  --project=PROJECT_ID

# List channels and get IDs
gcloud alpha monitoring channels list --project=PROJECT_ID \
  --format="table(name,displayName,type)"

# ── Alert Policies ────────────────────────────────────────────────────────────

# CPU > 80% for 5 min on any GCE instance
gcloud alpha monitoring policies create \
  --display-name="High CPU — GCE Production" \
  --condition-display-name="CPU utilization > 80%" \
  --condition-filter='resource.type="gce_instance" AND metric.type="compute.googleapis.com/instance/cpu/utilization"' \
  --condition-threshold-value=0.8 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-aggregations-per-series-aligner=ALIGN_MEAN \
  --condition-aggregations-alignment-period=300s \
  --condition-duration=300s \
  --notification-channels=CHANNEL_ID \
  --documentation-content="## High CPU\n**Runbook:** https://wiki/cpu-alert\n**Owner:** SRE\n**Severity:** Warning" \
  --combiner=OR \
  --project=PROJECT_ID

# Cloud Run error rate > 1% for 2 min
gcloud alpha monitoring policies create \
  --display-name="High Error Rate — Cloud Run" \
  --condition-display-name="5xx rate > 1%" \
  --condition-filter='resource.type="cloud_run_revision" AND metric.type="run.googleapis.com/request_count" AND metric.label.response_code_class="5xx"' \
  --condition-threshold-value=0.01 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-aggregations-per-series-aligner=ALIGN_RATE \
  --condition-aggregations-alignment-period=120s \
  --condition-duration=120s \
  --notification-channels=CHANNEL_ID \
  --project=PROJECT_ID

# Metric absence alert (no data for 5 min)
gcloud alpha monitoring policies create \
  --display-name="Custom Metric Missing — order-service" \
  --condition-display-name="No data received" \
  --condition-filter='resource.type="generic_task" AND metric.type="custom.googleapis.com/order_service/requests_total"' \
  --condition-absent \
  --condition-duration=300s \
  --notification-channels=CHANNEL_ID \
  --project=PROJECT_ID
```

### SLO Creation and Burn Rate Alerts

```bash
# Step 1: Create a custom service
gcloud alpha monitoring services create \
  --display-name="Order Service" \
  --project=PROJECT_ID
# Note the returned service ID

# Step 2: Create a request-based SLO (99.9% availability, 30-day rolling)
gcloud alpha monitoring slos create \
  --service=SERVICE_ID \
  --display-name="Order Service Availability — 99.9%" \
  --request-based \
  --good-total-ratio-threshold=0.999 \
  --rolling-period-days=30 \
  --project=PROJECT_ID

# Step 3: Create burn rate alert (fast-burn: 1h window at 14x, slow-burn: 6h at 6x)
# (Do this via Terraform or the Console — gcloud SLO burn rate alerting is limited)
```

**Burn rate alert logic:**

```
Fast-burn:  If error budget is being consumed at 14x normal rate over 1 hour
            → 2% of 30-day budget burned in 1 hour → PAGE NOW
Slow-burn:  If error budget is being consumed at 6x normal rate over 6 hours
            → 5% of 30-day budget burned in 6 hours → TICKET

Both conditions must hold simultaneously for the alert to fire
```

### Uptime Checks

```bash
# HTTP uptime check (multi-region)
gcloud monitoring uptime create \
  --display-name="Order Service — /healthz" \
  --resource-type=uptime-url \
  --resource-labels=host=api.example.com,project_id=PROJECT_ID \
  --http-check-path=/healthz \
  --http-check-port=443 \
  --use-ssl \
  --check-interval=60s \
  --timeout=10s \
  --regions=USA,EUROPE,ASIA_PACIFIC \
  --project=PROJECT_ID
```

---

## 6. Cloud Monitoring — Dashboards & MQL

### MQL Syntax Reference

MQL pipelines are a series of `|` operations:

```
fetch <resource_type>
| metric '<metric_type>'
| filter <label_filter>
| align <aligner>(<window>)
| every <period>
| group_by [<label_list>], <aggregation>
| [ratio_to <denominator_stream>]
| [top <n>]
| [window <duration>]
```

### MQL Examples

```
# ── Request rate by response code class (Cloud Run) ───────────────────────────
fetch cloud_run_revision
| metric 'run.googleapis.com/request_count'
| filter resource.service_name == 'order-service'
| align rate(1m)
| every 1m
| group_by [metric.response_code_class],
    [requests_per_sec: aggregate(value.request_count)]

# ── p50/p95/p99 latency (Cloud Run) ─────────────────────────────────────────
fetch cloud_run_revision
| metric 'run.googleapis.com/request_latencies'
| filter resource.service_name == 'order-service'
| align delta(1m)
| every 1m
| group_by [],
    [p50: percentile(value.request_latencies, 50),
     p95: percentile(value.request_latencies, 95),
     p99: percentile(value.request_latencies, 99)]

# ── Error rate ratio ──────────────────────────────────────────────────────────
fetch cloud_run_revision
| metric 'run.googleapis.com/request_count'
| filter resource.service_name == 'order-service'
| align rate(1m)
| every 1m
| group_by [metric.response_code_class], [total: aggregate(value.request_count)]
| {t_5xx: filter metric.response_code_class == '5xx' | value val(0)
   t_total: ident}
| ratio_to t_total

# ── GKE container memory utilization ─────────────────────────────────────────
fetch k8s_container
| metric 'kubernetes.io/container/memory/used_bytes'
| filter
    resource.cluster_name == 'prod-cluster'
    && resource.namespace_name == 'production'
| align mean(1m)
| every 1m
| group_by [resource.pod_name, resource.container_name],
    [memory_used: mean(value.used_bytes)]

# ── Custom metric: average checkout duration ──────────────────────────────────
fetch generic_task
| metric 'custom.googleapis.com/order_service/checkout_duration_ms'
| filter metric.payment_method == 'card'
| align mean(5m)
| every 5m
| group_by [], [avg_duration: mean(value.checkout_duration_ms)]

# ── GCE CPU top 5 instances ───────────────────────────────────────────────────
fetch gce_instance
| metric 'compute.googleapis.com/instance/cpu/utilization'
| align mean(1m)
| every 1m
| group_by [resource.instance_id], [cpu: mean(value.utilization)]
| top 5
```

### SLO Error Budget Dashboard

```
# Remaining error budget (window-based SLO)
fetch consumed_api
| metric 'monitoring.googleapis.com/service_runtime/api/request_count'
| {good: filter metric.response_code < 500
   total: ident}
| ratio_to total
| window 30d
| value val(0) > 0.999  # SLO target
```

---

## 7. Cloud Monitoring — Terraform

```hcl
# ── Notification Channels ─────────────────────────────────────────────────────

resource "google_monitoring_notification_channel" "slack" {
  display_name = "SRE Slack — #alerts-production"
  type         = "slack"
  project      = var.project_id

  labels = {
    channel_name = "#alerts-production"
    auth_token   = var.slack_bot_token  # Use secret reference in production
  }
  sensitive_labels {
    auth_token = var.slack_bot_token
  }
}

resource "google_monitoring_notification_channel" "pagerduty" {
  display_name = "PagerDuty — Production"
  type         = "pagerduty"
  project      = var.project_id

  sensitive_labels {
    service_key = var.pagerduty_service_key
  }
}

# ── Alert Policies ────────────────────────────────────────────────────────────

resource "google_monitoring_alert_policy" "high_cpu" {
  display_name = "High CPU — GCE Production"
  combiner     = "OR"
  project      = var.project_id
  enabled      = true

  conditions {
    display_name = "CPU utilization > 80% for 5 min"

    condition_threshold {
      filter          = <<-EOT
        resource.type = "gce_instance"
        AND metric.type = "compute.googleapis.com/instance/cpu/utilization"
      EOT
      comparison      = "COMPARISON_GT"
      threshold_value = 0.8
      duration        = "300s"

      aggregations {
        alignment_period     = "300s"
        per_series_aligner   = "ALIGN_MEAN"
        cross_series_reducer = "REDUCE_MEAN"
        group_by_fields      = ["resource.label.instance_id", "resource.label.zone"]
      }

      trigger {
        count = 1
      }
    }
  }

  notification_channels = [
    google_monitoring_notification_channel.slack.name,
    google_monitoring_notification_channel.pagerduty.name,
  ]

  documentation {
    content   = <<-EOT
      ## High CPU Alert
      **Severity:** Warning
      **Owner:** SRE Team
      **Runbook:** https://wiki.example.com/runbooks/high-cpu
      **Dashboard:** https://console.cloud.google.com/monitoring/dashboards/custom/DASHBOARD_ID
    EOT
    mime_type = "text/markdown"
  }

  alert_strategy {
    auto_close = "1800s"
    notification_rate_limit {
      period = "300s"
    }
  }
}

resource "google_monitoring_alert_policy" "cloud_run_error_rate" {
  display_name = "High Error Rate — Cloud Run order-service"
  combiner     = "OR"
  project      = var.project_id

  conditions {
    display_name = "5xx error rate > 1% for 2 min"

    condition_threshold {
      filter     = <<-EOT
        resource.type = "cloud_run_revision"
        AND resource.label.service_name = "order-service"
        AND metric.type = "run.googleapis.com/request_count"
        AND metric.label.response_code_class = "5xx"
      EOT
      comparison = "COMPARISON_GT"
      threshold_value = 0.01
      duration   = "120s"

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_RATE"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.pagerduty.name]
}

# ── SLO ───────────────────────────────────────────────────────────────────────

resource "google_monitoring_custom_service" "order_service" {
  service_id   = "order-service"
  display_name = "Order Service"
  project      = var.project_id
}

resource "google_monitoring_slo" "order_availability" {
  service             = google_monitoring_custom_service.order_service.service_id
  project             = var.project_id
  slo_id              = "order-availability-slo"
  display_name        = "Order Service Availability (99.9%)"
  goal                = 0.999
  rolling_period_days = 30

  request_based_sli {
    good_total_ratio {
      good_service_filter  = <<-EOT
        resource.type = "cloud_run_revision"
        AND resource.label.service_name = "order-service"
        AND metric.type = "run.googleapis.com/request_count"
        AND metric.label.response_code_class = "2xx"
      EOT
      total_service_filter = <<-EOT
        resource.type = "cloud_run_revision"
        AND resource.label.service_name = "order-service"
        AND metric.type = "run.googleapis.com/request_count"
      EOT
    }
  }
}

# Burn rate alert for the SLO
resource "google_monitoring_alert_policy" "slo_burn_rate" {
  display_name = "SLO Burn Rate — Order Service Availability"
  combiner     = "AND"
  project      = var.project_id

  conditions {
    display_name = "Fast burn: 14x for 1h"

    condition_threshold {
      filter          = "select_slo_burn_rate(\"${google_monitoring_slo.order_availability.id}\", 3600s)"
      comparison      = "COMPARISON_GT"
      threshold_value = 14.4
      duration        = "0s"

      aggregations {
        alignment_period   = "300s"
        per_series_aligner = "ALIGN_MEAN"
      }
    }
  }

  conditions {
    display_name = "Slow burn: 6x for 6h"

    condition_threshold {
      filter          = "select_slo_burn_rate(\"${google_monitoring_slo.order_availability.id}\", 21600s)"
      comparison      = "COMPARISON_GT"
      threshold_value = 6.0
      duration        = "0s"

      aggregations {
        alignment_period   = "300s"
        per_series_aligner = "ALIGN_MEAN"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.pagerduty.name]
}

# ── Uptime Check ──────────────────────────────────────────────────────────────

resource "google_monitoring_uptime_check_config" "api_health" {
  display_name = "Order Service API — /healthz"
  project      = var.project_id
  timeout      = "10s"
  period       = "60s"

  http_check {
    path           = "/healthz"
    port           = 443
    use_ssl        = true
    validate_ssl   = true
    request_method = "GET"
    accepted_response_status_codes {
      status_value = 200
    }
    headers = {
      "X-Health-Check" = "gcp-uptime"
    }
  }

  monitored_resource {
    type = "uptime_url"
    labels = {
      project_id = var.project_id
      host       = "api.example.com"
    }
  }

  selected_regions = ["USA", "EUROPE", "ASIA_PACIFIC"]
}

# Alert for uptime check failure
resource "google_monitoring_alert_policy" "uptime_failure" {
  display_name = "Uptime Check Failed — api.example.com"
  combiner     = "OR"
  project      = var.project_id

  conditions {
    display_name = "Uptime check failed from 2+ regions"

    condition_threshold {
      filter          = "metric.type=\"monitoring.googleapis.com/uptime_check/check_passed\" AND resource.type=\"uptime_url\" AND metric.label.check_id=\"${google_monitoring_uptime_check_config.api_health.uptime_check_id}\""
      comparison      = "COMPARISON_GT"
      threshold_value = 1
      duration        = "60s"

      aggregations {
        alignment_period     = "60s"
        per_series_aligner   = "ALIGN_NEXT_OLDER"
        cross_series_reducer = "REDUCE_COUNT_FALSE"
        group_by_fields      = ["resource.label.host"]
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.pagerduty.name]
}
```

---

## PART 2 — CLOUD LOGGING

## 8. Cloud Logging — Concepts & Architecture

### Log Entry Structure

```json
{
  "logName":      "projects/my-project/logs/order-service",
  "resource": {
    "type":   "cloud_run_revision",
    "labels": {"service_name": "order-service", "revision_name": "order-service-00010-abc"}
  },
  "timestamp":    "2024-01-15T10:30:00.123456789Z",
  "receiveTimestamp": "2024-01-15T10:30:00.234567890Z",
  "severity":     "ERROR",
  "insertId":     "abc123uniqueid",
  "jsonPayload": {
    "message":    "Payment gateway timeout",
    "order_id":   "ord-12346",
    "gateway":    "stripe",
    "timeout_ms": 5000
  },
  "textPayload":  null,
  "protoPayload": null,
  "labels": {
    "environment": "production",
    "version":     "v2.3.1"
  },
  "trace":    "projects/my-project/traces/abc123def456abc123def456abc123de",
  "spanId":   "abc123def456abc1",
  "traceSampled": true,
  "httpRequest": {
    "requestMethod": "POST",
    "requestUrl":    "/api/v1/orders/ord-12346/pay",
    "status":        503,
    "latency":       "5.001s",
    "userAgent":     "Mozilla/5.0...",
    "remoteIp":      "203.0.113.42"
  }
}
```

### Severity Levels

| Severity | Numeric | When to Use |
|---|---|---|
| `DEFAULT` | 0 | No severity specified |
| `DEBUG` | 100 | Verbose, development-only info |
| `INFO` | 200 | Normal operational events |
| `NOTICE` | 300 | Normal but significant event |
| `WARNING` | 400 | Potential issues, recoverable errors |
| `ERROR` | 500 | Errors that must be investigated |
| `CRITICAL` | 600 | Severe errors; service degraded |
| `ALERT` | 700 | Requires immediate action |
| `EMERGENCY` | 800 | System unusable |

> **💡 Best Practice:** Use `ERROR` only for actionable errors that a human should investigate. Reserve `CRITICAL` and above for on-call-worthy events. Overusing high severity levels creates alert fatigue.

### Log Types

| Log Type | Where | Retention | Configurable? |
|---|---|---|---|
| **Admin Activity** (audit) | `_Required` bucket | 400 days | No |
| **System Event** (audit) | `_Required` bucket | 400 days | No |
| **Data Access** (audit) | `_Default` bucket | 30 days | Must enable per-service |
| **Policy Denied** (audit) | `_Default` bucket | 30 days | Yes |
| **Platform Logs** | `_Default` bucket | 30 days | Yes |
| **User-written logs** | `_Default` bucket | 30 days | Yes |
| **Multi-cloud logs** | Custom bucket | Configurable | Yes |

> **⚠️ Gotcha:** **Data Access audit logs are OFF by default.** They can generate extremely high log volume (and cost). Enable them selectively per service in IAM → Audit Logs → Configure Data Access Audit Logs.

### Log Bucket & Routing Architecture

```
Log Entry arrives
      │
      ▼
┌──────────────────────────────────────────────────────────────────┐
│                         Log Router                               │
│                                                                  │
│  _Required sink (immutable)                                      │
│  filter: Admin Activity + System Event audit logs                │
│  destination: _Required bucket (400-day retention, no delete)    │
│                                                                  │
│  _Default sink (editable)                                        │
│  filter: everything NOT in _Required                             │
│  destination: _Default bucket (30-day retention)                 │
│                                                                  │
│  Custom sinks (user-defined)                                     │
│  filter: user-defined                                            │
│  destination: any of:                                            │
│    ├── Cloud Logging bucket (custom retention)                   │
│    ├── BigQuery dataset (SQL analysis)                           │
│    ├── Cloud Storage bucket (archival)                           │
│    └── Pub/Sub topic (real-time streaming → SIEM)                │
└──────────────────────────────────────────────────────────────────┘

Log Views (access control within a bucket):
  security-audit-bucket
    ├── _AllLogs view (full access)
    ├── security-team-view (filter: resource.type="gce_instance")
    └── devops-view (filter: resource.labels.service_name="order-service")
```

---

## 9. Cloud Logging — Writing Logs

### Python — google-cloud-logging

```python
import google.cloud.logging
import logging
import json

PROJECT_ID = "my-project"

# ── Method 1: stdlib logging integration ──────────────────────────────────────
client = google.cloud.logging.Client(project=PROJECT_ID)
client.setup_logging()  # Attaches GCP handler to root logger

logger = logging.getLogger("order-service")
logger.setLevel(logging.INFO)

# Structured log via extra json_fields
logger.info(
    "Order processed successfully",
    extra={
        "json_fields": {
            "order_id": "ord-12345",
            "customer_id": "cust-67890",
            "amount_usd": 142.50,
            "latency_ms": 87,
            "payment_method": "card",
        }
    },
)

logger.error(
    "Payment gateway timeout",
    extra={
        "json_fields": {
            "order_id": "ord-12346",
            "gateway": "stripe",
            "timeout_ms": 5000,
            "retry_count": 3,
        }
    },
)

# ── Method 2: Direct cloud logger (more control) ──────────────────────────────
cloud_logger = client.logger("order-service")

cloud_logger.log_struct(
    {
        "message": "Inventory check completed",
        "sku": "WIDGET-001",
        "quantity_available": 42,
        "latency_ms": 12,
    },
    severity="INFO",
    labels={"environment": "production", "version": "v2.3.1"},
    http_request={
        "requestMethod": "POST",
        "requestUrl": "/api/v1/inventory/check",
        "status": 200,
        "latency": "0.012s",
        "userAgent": "order-service/v2.3.1",
    },
    resource=google.cloud.logging.Resource(
        type="cloud_run_revision",
        labels={
            "project_id": PROJECT_ID,
            "service_name": "order-service",
            "revision_name": "order-service-00010-abc",
            "location": "us-central1",
        },
    ),
)

# Flush is async by default; call flush() before shutdown
cloud_logger.client.close()
```

### Go — cloud.google.com/go/logging

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"

    "cloud.google.com/go/logging"
    mrpb "google.golang.org/genproto/googleapis/api/monitoredres"
)

const PROJECT_ID = "my-project"

func initLogger(ctx context.Context) (*logging.Client, *logging.Logger) {
    client, err := logging.NewClient(ctx, PROJECT_ID)
    if err != nil {
        log.Fatalf("Failed to create logging client: %v", err)
    }

    logger := client.Logger("order-service",
        logging.CommonLabels(map[string]string{
            "environment": "production",
            "version":     "v2.3.1",
        }),
        logging.CommonResource(&mrpb.MonitoredResource{
            Type: "cloud_run_revision",
            Labels: map[string]string{
                "project_id":    PROJECT_ID,
                "service_name":  "order-service",
                "revision_name": "order-service-00010-abc",
                "location":      "us-central1",
            },
        }),
    )
    return client, logger
}

func logOrderProcessed(logger *logging.Logger, r *http.Request, orderID string, latencyMs int) {
    logger.Log(logging.Entry{
        Severity: logging.Info,
        Payload: map[string]interface{}{
            "message":    fmt.Sprintf("Order %s processed", orderID),
            "order_id":   orderID,
            "latency_ms": latencyMs,
        },
        HTTPRequest: &logging.HTTPRequest{Request: r, Status: 200, Latency: 87},
        Labels:      map[string]string{"component": "payment"},
    })
}

func logError(logger *logging.Logger, err error, orderID string) {
    logger.Log(logging.Entry{
        Severity: logging.Error,
        Payload: map[string]interface{}{
            "message":  err.Error(),
            "order_id": orderID,
        },
    })
    logger.Flush() // Flush errors synchronously
}
```

### OpenTelemetry Log Bridge → Cloud Logging

```python
# OTel log bridge sends OTel log records to Cloud Logging
from opentelemetry._logs import set_logger_provider
from opentelemetry.sdk._logs import LoggerProvider
from opentelemetry.sdk._logs.export import BatchLogRecordProcessor
from opentelemetry.exporter.cloud_logging import CloudLoggingExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry import trace
import logging
import logging.handlers

# Initialize OTel log provider with Cloud Logging exporter
resource = Resource.create({"service.name": "order-service", "service.version": "v2.3.1"})
exporter = CloudLoggingExporter(project_id=PROJECT_ID)
provider = LoggerProvider(resource=resource)
provider.add_log_record_processor(BatchLogRecordProcessor(exporter))
set_logger_provider(provider)

# Attach OTel handler to stdlib logger (bridge pattern)
from opentelemetry.sdk._logs._internal import LoggingHandler
otel_handler = LoggingHandler(level=logging.DEBUG, logger_provider=provider)
logging.getLogger().addHandler(otel_handler)

# Now stdlib logging flows through OTel → Cloud Logging with trace correlation
logger = logging.getLogger("order-service")
logger.info("Order processed", extra={"order_id": "ord-12345"})
```

### Log-Trace Correlation (Injecting Trace Context)

```python
# Inject trace context into structured log entries for clickable correlation
import google.cloud.logging
from opentelemetry import trace as otel_trace

def get_trace_context(project_id: str):
    """Extract current OTel trace context for Cloud Logging correlation."""
    span = otel_trace.get_current_span()
    ctx = span.get_span_context()
    if ctx.is_valid:
        trace_id = format(ctx.trace_id, '032x')
        span_id  = format(ctx.span_id, '016x')
        return (
            f"projects/{project_id}/traces/{trace_id}",  # Full path required
            span_id,
            ctx.trace_flags.sampled,
        )
    return None, None, False

# Usage
cloud_logger = google.cloud.logging.Client(project=PROJECT_ID).logger("order-service")
trace_str, span_id, sampled = get_trace_context(PROJECT_ID)

cloud_logger.log_struct(
    {
        "message":    "Processing payment",
        "order_id":   "ord-12345",
        "amount_usd": 142.50,
    },
    severity="INFO",
    trace=trace_str,          # "projects/PROJECT_ID/traces/TRACE_ID"
    span_id=span_id,          # 16-char hex span ID
    trace_sampled=sampled,    # True = show in Trace viewer
)
```

> **⚠️ Gotcha:** The `trace` field in Cloud Logging must be the **full resource path** `projects/PROJECT_ID/traces/TRACE_ID`, not just the hex trace ID. Using just the hex ID will break the log-to-trace correlation link in the Cloud Console.

> **💡 Best Practice:** Structure all application logs as JSON with consistent field names. Use `message` as the human-readable summary, and include `order_id`, `user_id`, `latency_ms`, `component` as searchable fields. Never log PII or secrets — scrub before writing.

---

## 10. Cloud Logging — Log Routing & Sinks

```bash
# ── Log Sinks ─────────────────────────────────────────────────────────────────

# Sink 1: Audit logs → BigQuery (long-term SQL analysis)
gcloud logging sinks create audit-logs-bq \
  bigquery.googleapis.com/projects/PROJECT_ID/datasets/audit_logs \
  --log-filter='protoPayload.@type="type.googleapis.com/google.cloud.audit.AuditLog"' \
  --use-partitioned-tables \
  --project=PROJECT_ID

# Sink 2: All logs → Cloud Storage (archival, low cost)
gcloud logging sinks create all-logs-gcs \
  storage.googleapis.com/MY_LOG_ARCHIVE_BUCKET \
  --log-filter='' \
  --project=PROJECT_ID

# Sink 3: Security events → Pub/Sub (real-time SIEM streaming)
gcloud logging sinks create security-logs-pubsub \
  pubsub.googleapis.com/projects/PROJECT_ID/topics/security-events \
  --log-filter='
    severity>=WARNING
    OR protoPayload.@type="type.googleapis.com/google.cloud.audit.AuditLog"
    OR (protoPayload.serviceName="iam.googleapis.com" AND protoPayload.methodName=~"SetIamPolicy|Create|Delete")
  ' \
  --project=PROJECT_ID

# Sink 4: App errors → dedicated bucket (separate retention + access)
gcloud logging sinks create app-errors-bucket \
  logging.googleapis.com/projects/PROJECT_ID/locations/global/buckets/app-errors-bucket \
  --log-filter='severity>=ERROR AND resource.type="cloud_run_revision"' \
  --project=PROJECT_ID

# Grant sink service account access to destination
SINK_SA=$(gcloud logging sinks describe audit-logs-bq \
  --format="value(writerIdentity)" --project=PROJECT_ID)
echo "Sink SA: $SINK_SA"

# Grant BigQuery access
bq add-iam-policy-binding \
  --member="$SINK_SA" \
  --role="roles/bigquery.dataEditor" \
  PROJECT_ID:audit_logs

# Grant GCS access
gcloud storage buckets add-iam-policy-binding gs://MY_LOG_ARCHIVE_BUCKET \
  --member="$SINK_SA" \
  --role="roles/storage.objectCreator"

# Grant Pub/Sub access
gcloud pubsub topics add-iam-policy-binding security-events \
  --member="$SINK_SA" \
  --role="roles/pubsub.publisher" \
  --project=PROJECT_ID

# ── Exclusions (cost reduction) ───────────────────────────────────────────────

# Exclude noisy health check logs from _Default sink
gcloud logging sinks update _Default \
  --add-exclusion=name=exclude-health-checks,filter='httpRequest.requestUrl=~"/healthz|/readyz|/livez|/metrics"' \
  --project=PROJECT_ID

# Exclude DEBUG logs from non-prod services in _Default
gcloud logging sinks update _Default \
  --add-exclusion=name=exclude-debug,filter='severity=DEBUG' \
  --project=PROJECT_ID

# ── Log Buckets ────────────────────────────────────────────────────────────────

# Create a custom bucket with Log Analytics and indexing
gcloud logging buckets create security-audit-bucket \
  --location=global \
  --retention-days=365 \
  --description="Security audit logs — 1 year retention" \
  --project=PROJECT_ID

# Enable Log Analytics (allows SQL queries via BigQuery interface)
gcloud logging buckets update security-audit-bucket \
  --location=global \
  --enable-analytics \
  --project=PROJECT_ID

# Create a log view (IAM-gated sub-view of a bucket)
gcloud logging views create security-team-view \
  --bucket=security-audit-bucket \
  --location=global \
  --log-filter='resource.type="gce_instance" OR resource.type="k8s_container"' \
  --description="Security team: infrastructure logs only" \
  --project=PROJECT_ID

# Grant access to view (not the full bucket)
gcloud logging views add-iam-policy-binding security-team-view \
  --bucket=security-audit-bucket \
  --location=global \
  --member="group:security-team@example.com" \
  --role="roles/logging.viewAccessor" \
  --project=PROJECT_ID
```

---

## 11. Cloud Logging — Querying & Log Analytics

### Cloud Logging Query Language

```bash
# ── Core Syntax ───────────────────────────────────────────────────────────────
# Exact match
resource.type="cloud_run_revision"
resource.labels.service_name="order-service"
severity=ERROR

# Severity range
severity>=WARNING        # WARNING, ERROR, CRITICAL, ALERT, EMERGENCY

# Regex match
protoPayload.methodName=~"SetIamPolicy|CreateServiceAccount"

# Negation
-resource.type="cloud_run_revision"
NOT jsonPayload.component="healthcheck"

# JSON payload field access
jsonPayload.order_id="ord-12345"
jsonPayload.latency_ms>1000
jsonPayload.payment_method=~"card|paypal"

# Existence check
jsonPayload.error_code:*         # Field exists
-jsonPayload.user_id:*           # Field does not exist

# Time range (ISO 8601)
timestamp>="2024-01-15T00:00:00Z"
timestamp<"2024-01-16T00:00:00Z"

# ── Production Queries ────────────────────────────────────────────────────────

# All errors in Cloud Run order-service (last hour)
resource.type="cloud_run_revision"
resource.labels.service_name="order-service"
severity>=ERROR

# Specific order across all services
jsonPayload.order_id="ord-12345"

# IAM changes (who changed what permissions)
protoPayload.@type="type.googleapis.com/google.cloud.audit.AuditLog"
protoPayload.serviceName="iam.googleapis.com"
protoPayload.methodName=~"SetIamPolicy|CreateServiceAccount|DeleteServiceAccount|CreateKey"

# SSH key modifications on GCE
protoPayload.methodName="v1.compute.instances.setMetadata"
protoPayload.request.items.key="ssh-keys"

# Requests slower than 1 second
resource.type="cloud_run_revision"
resource.labels.service_name="order-service"
jsonPayload.latency_ms>1000
severity=INFO

# All logs for a specific trace (click-through from Trace viewer)
trace="projects/PROJECT_ID/traces/abc123def456abc123def456abc123de"

# HTTP 5xx errors with details
httpRequest.status>=500
resource.type="cloud_run_revision"

# VPC SC violations (see also VPC SC cheatsheet)
protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
protoPayload.status.code!=0

# Container OOMKilled events
resource.type="k8s_node"
jsonPayload.MESSAGE=~"OOMKilling|oom_kill_process"

# GKE pod crash loop
resource.type="k8s_pod"
jsonPayload.reason="BackOff"
severity>=WARNING
```

### Log Analytics SQL Queries

Log Analytics must be enabled on the bucket. Query via Cloud Console → Logging → Log Analytics.

```sql
-- Top 10 slowest API endpoints (last 24 hours)
SELECT
  JSON_VALUE(json_payload, '$.endpoint') AS endpoint,
  COUNT(*) AS request_count,
  ROUND(AVG(CAST(JSON_VALUE(json_payload, '$.latency_ms') AS FLOAT64)), 2) AS avg_latency_ms,
  APPROX_QUANTILES(
    CAST(JSON_VALUE(json_payload, '$.latency_ms') AS FLOAT64), 100
  )[OFFSET(99)] AS p99_latency_ms,
  APPROX_QUANTILES(
    CAST(JSON_VALUE(json_payload, '$.latency_ms') AS FLOAT64), 100
  )[OFFSET(95)] AS p95_latency_ms
FROM `PROJECT_ID.global._Default._AllLogs`
WHERE
  timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
  AND log_name LIKE '%order-service%'
  AND json_payload IS NOT NULL
  AND JSON_VALUE(json_payload, '$.endpoint') IS NOT NULL
GROUP BY endpoint
HAVING request_count > 10
ORDER BY p99_latency_ms DESC
LIMIT 10;

-- Error rate by service (last 1 hour)
SELECT
  resource.labels.service_name,
  COUNTIF(severity = 'ERROR') AS error_count,
  COUNTIF(severity = 'CRITICAL') AS critical_count,
  COUNT(*) AS total_count,
  ROUND(COUNTIF(severity >= 'ERROR') / COUNT(*) * 100, 4) AS error_rate_pct
FROM `PROJECT_ID.global._Default._AllLogs`
WHERE
  timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
  AND resource.type = 'cloud_run_revision'
GROUP BY resource.labels.service_name
HAVING total_count > 0
ORDER BY error_rate_pct DESC;

-- IAM changes in the last 7 days
SELECT
  timestamp,
  proto_payload.audit_log.authentication_info.principal_email AS actor,
  proto_payload.audit_log.method_name AS action,
  proto_payload.audit_log.resource_name AS target_resource
FROM `PROJECT_ID.global._Required._AllLogs`
WHERE
  timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND log_name LIKE '%cloudaudit%'
  AND proto_payload.audit_log.service_name = 'iam.googleapis.com'
  AND proto_payload.audit_log.method_name LIKE '%SetIamPolicy%'
ORDER BY timestamp DESC;

-- Top error messages with frequency
SELECT
  JSON_VALUE(json_payload, '$.message') AS error_message,
  COUNT(*) AS occurrences,
  MIN(timestamp) AS first_seen,
  MAX(timestamp) AS last_seen,
  COUNT(DISTINCT JSON_VALUE(json_payload, '$.order_id')) AS affected_orders
FROM `PROJECT_ID.global._Default._AllLogs`
WHERE
  timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
  AND severity = 'ERROR'
  AND resource.type = 'cloud_run_revision'
GROUP BY error_message
ORDER BY occurrences DESC
LIMIT 20;
```

### Log-Based Metrics

```bash
# Counter metric: count ERROR logs by endpoint
gcloud logging metrics create order-errors-by-endpoint \
  --description="Count of order processing errors by endpoint" \
  --log-filter='
    resource.type="cloud_run_revision"
    AND resource.labels.service_name="order-service"
    AND severity=ERROR
  ' \
  --metric-labels=endpoint=EXTRACT(jsonPayload.endpoint) \
  --project=PROJECT_ID

# Distribution metric: latency histogram from structured logs
gcloud logging metrics create request-latency-histogram \
  --description="Request latency distribution in ms" \
  --log-filter='
    resource.type="cloud_run_revision"
    AND jsonPayload.latency_ms:*
    AND severity=INFO
  ' \
  --value-extractor='EXTRACT(jsonPayload.latency_ms)' \
  --bucket-options-exponential-buckets-num-finite-buckets=20 \
  --bucket-options-exponential-buckets-growth-factor=2.0 \
  --bucket-options-exponential-buckets-scale=1.0 \
  --metric-labels=endpoint=EXTRACT(jsonPayload.endpoint) \
  --project=PROJECT_ID
```

---

## 12. Cloud Logging — Terraform

```hcl
# ── Log Buckets ────────────────────────────────────────────────────────────────

resource "google_logging_project_bucket_config" "security_audit" {
  project          = var.project_id
  location         = "global"
  retention_days   = 365
  bucket_id        = "security-audit-bucket"
  description      = "Security audit logs — 1 year retention"
  enable_analytics = true  # Enables SQL queries via Log Analytics

  # Custom index for fast field-level queries
  index_configs {
    field_path = "jsonPayload.order_id"
    type       = "INDEX_TYPE_STRING"
  }
  index_configs {
    field_path = "jsonPayload.customer_id"
    type       = "INDEX_TYPE_STRING"
  }
}

resource "google_logging_project_bucket_config" "app_errors" {
  project        = var.project_id
  location       = "global"
  retention_days = 90
  bucket_id      = "app-errors-bucket"
  description    = "Application error logs — 90 days"
}

# ── Log Sinks ─────────────────────────────────────────────────────────────────

# Sink to BigQuery
resource "google_logging_project_sink" "audit_bq" {
  name    = "audit-logs-bq"
  project = var.project_id
  destination = "bigquery.googleapis.com/projects/${var.project_id}/datasets/${google_bigquery_dataset.audit_logs.dataset_id}"
  filter  = "protoPayload.@type=\"type.googleapis.com/google.cloud.audit.AuditLog\""

  unique_writer_identity = true

  bigquery_options {
    use_partitioned_tables = true
  }
}

# Sink to GCS (archival)
resource "google_logging_project_sink" "archive_gcs" {
  name        = "all-logs-gcs-archive"
  project     = var.project_id
  destination = "storage.googleapis.com/${google_storage_bucket.log_archive.name}"
  filter      = ""  # All logs

  unique_writer_identity = true
}

# Sink to Pub/Sub (SIEM streaming)
resource "google_logging_project_sink" "security_pubsub" {
  name    = "security-logs-pubsub"
  project = var.project_id
  destination = "pubsub.googleapis.com/projects/${var.project_id}/topics/${google_pubsub_topic.security_events.name}"
  filter  = "severity>=WARNING OR protoPayload.@type=\"type.googleapis.com/google.cloud.audit.AuditLog\""

  unique_writer_identity = true
}

# Grant sink SAs the required destination permissions
resource "google_bigquery_dataset_iam_member" "audit_sink_writer" {
  dataset_id = google_bigquery_dataset.audit_logs.dataset_id
  role       = "roles/bigquery.dataEditor"
  member     = google_logging_project_sink.audit_bq.writer_identity
  project    = var.project_id
}

resource "google_storage_bucket_iam_member" "archive_sink_writer" {
  bucket = google_storage_bucket.log_archive.name
  role   = "roles/storage.objectCreator"
  member = google_logging_project_sink.archive_gcs.writer_identity
}

resource "google_pubsub_topic_iam_member" "security_sink_publisher" {
  topic   = google_pubsub_topic.security_events.name
  role    = "roles/pubsub.publisher"
  member  = google_logging_project_sink.security_pubsub.writer_identity
  project = var.project_id
}

# ── Log Exclusions ────────────────────────────────────────────────────────────

resource "google_logging_project_exclusion" "health_checks" {
  name        = "exclude-health-checks"
  project     = var.project_id
  description = "Exclude /healthz, /readyz, /livez, /metrics noise from default sink"
  filter      = "httpRequest.requestUrl=~\"/healthz|/readyz|/livez|/metrics\""
}

resource "google_logging_project_exclusion" "debug_logs" {
  name        = "exclude-debug-logs"
  project     = var.project_id
  description = "Exclude DEBUG severity logs (too verbose for production)"
  filter      = "severity=DEBUG"
}

# ── Log-Based Metrics ─────────────────────────────────────────────────────────

resource "google_logging_metric" "order_errors" {
  name        = "order-errors"
  project     = var.project_id
  description = "Count of order processing errors by endpoint"
  filter      = "resource.type=\"cloud_run_revision\" AND resource.labels.service_name=\"order-service\" AND severity=ERROR"

  metric_descriptor {
    metric_kind = "DELTA"
    value_type  = "INT64"
    unit        = "1"
    labels {
      key         = "endpoint"
      value_type  = "STRING"
      description = "API endpoint that produced the error"
    }
  }

  label_extractors = {
    "endpoint" = "EXTRACT(jsonPayload.endpoint)"
  }
}

resource "google_logging_metric" "request_latency_dist" {
  name        = "request-latency-histogram"
  project     = var.project_id
  description = "HTTP request latency distribution (ms)"
  filter      = "resource.type=\"cloud_run_revision\" AND jsonPayload.latency_ms:*"
  value_extractor = "EXTRACT(jsonPayload.latency_ms)"

  metric_descriptor {
    metric_kind = "DELTA"
    value_type  = "DISTRIBUTION"
    unit        = "ms"
    labels {
      key        = "endpoint"
      value_type = "STRING"
    }
  }

  label_extractors = {
    "endpoint" = "EXTRACT(jsonPayload.endpoint)"
  }

  bucket_options {
    exponential_buckets {
      num_finite_buckets = 20
      growth_factor      = 2.0
      scale              = 1.0
    }
  }
}
```

---

## PART 3 — CLOUD TRACE

## 13. Cloud Trace — Concepts & Architecture

### Distributed Trace Flow

```
Client Request
      │
      │  POST /api/v1/orders  (traceparent: 00-abc123-def456-01)
      ▼
┌──────────────────┐
│  order-service   │  Span: "POST /api/v1/orders"
│  (Cloud Run)     │  trace_id: abc123  span_id: aaaa
│                  │─── DB query ─────────────────────────────┐
│                  │    Span: "postgresql.query"              │
│                  │    parent: aaaa  span_id: bbbb           │
│                  │◀─────────────────────────────────────────┘
│                  │
│                  │─── HTTP → payment-service ───────────────┐
│                  │    traceparent: 00-abc123-cccc-01        │
│                  │                                          ▼
│                  │                              ┌───────────────────┐
│                  │                              │ payment-service   │
│                  │                              │ Span: "POST /pay" │
│                  │                              │ parent: cccc      │
│                  │                              │ span_id: dddd     │
│                  │◀─────────────────────────────┘                   │
└──────────────────┘                                                   │
                                                                       │
All spans with trace_id=abc123 assembled into one trace in Cloud Trace ┘
```

### Core Concepts

| Concept | Description |
|---|---|
| **Trace** | Complete end-to-end representation of a single request through all services |
| **Span** | Single operation within a trace (one service call, one DB query, one HTTP request) |
| **Parent span** | Span that caused a child span to be created |
| **Span attributes** | Key-value metadata on a span (`db.system=postgresql`, `http.method=POST`) |
| **Span events** | Time-stamped annotations within a span (`"cache-miss"`, `"retry-2"`) |
| **Span status** | `OK`, `ERROR`, `UNSET` — indicates overall span success/failure |
| **Trace context** | Propagated headers carrying trace ID and span ID across service boundaries |

### Trace Context Propagation Headers

| Standard | Header | Format |
|---|---|---|
| **W3C TraceContext** (recommended) | `traceparent` | `00-{traceId}-{spanId}-{flags}` |
| **B3 (Zipkin)** | `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-Sampled` | Separate headers |
| **Google Cloud** | `X-Cloud-Trace-Context` | `{TRACE_ID}/{SPAN_ID};o={FLAGS}` |

```
W3C traceparent example:
  00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
  ^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^ ^^
  ver  128-bit trace ID (hex)           64-bit span ID   flags (01=sampled)
```

### Sampling Strategies

| Strategy | Description | Cloud Trace Config |
|---|---|---|
| **Always sample** | 100% — high cost, complete data | `AlwaysOn` |
| **Never sample** | 0% — no traces collected | `AlwaysOff` |
| **Ratio-based** | N% of traces sampled | `TraceIdRatioBased(0.1)` = 10% |
| **Rate-based** | Max N traces/second | Use custom sampler |
| **Parent-based** | Honor upstream sampling decision | `ParentBased(root=TraceIdRatioBased(0.1))` |

> **💡 Best Practice:** Use 10% sampling in production as the baseline. Increase to 100% for debugging or canary releases. Use `ParentBased` to ensure all spans in a trace are consistently sampled — don't let individual services make independent sampling decisions.

### Cloud Trace Limits

| Limit | Value |
|---|---|
| Trace retention | 30 days |
| Max attributes per span | 128 |
| Max events per span | 128 |
| Max links per span | 128 |
| Trace ID format | 128-bit (32 hex chars) |
| Span ID format | 64-bit (16 hex chars) |
| Max span name length | 128 characters |
| Free tier | 2.5M spans/month |

---

## 14. Cloud Trace — Instrumentation

### Python — OTel with Cloud Trace Exporter

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.trace.sampling import ParentBasedTraceIdRatio
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.propagate import set_global_textmap
from opentelemetry.propagators.cloud_trace_propagator import CloudTraceFormatPropagator
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

PROJECT_ID = "my-project"

def init_tracing(project_id: str, sample_rate: float = 0.1) -> None:
    """Initialize OTel tracing with Cloud Trace exporter."""
    resource = Resource.create({
        "service.name":           "order-service",
        "service.version":        "v2.3.1",
        "deployment.environment": "production",
        "cloud.provider":         "gcp",
        "cloud.region":           "us-central1",
    })

    exporter = CloudTraceSpanExporter(project_id=project_id)
    sampler  = ParentBasedTraceIdRatio(sample_rate)

    provider = TracerProvider(sampler=sampler, resource=resource)
    provider.add_span_processor(BatchSpanProcessor(
        exporter,
        max_queue_size=2048,
        max_export_batch_size=512,
        export_timeout_millis=30_000,
    ))
    trace.set_tracer_provider(provider)

    # Support both W3C and Cloud Trace header formats
    set_global_textmap(CloudTraceFormatPropagator())

    # Auto-instrument popular libraries
    FlaskInstrumentor().instrument()
    RequestsInstrumentor().instrument()
    SQLAlchemyInstrumentor().instrument()


# Initialize at startup
init_tracing(PROJECT_ID, sample_rate=0.1)
tracer = trace.get_tracer("order-service", tracer_version="v2.3.1")


def process_order(order_id: str, user_id: str) -> dict:
    """Process an order with full distributed tracing."""
    with tracer.start_as_current_span(
        "process-order",
        attributes={
            "order.id":    order_id,
            "user.id":     user_id,
            "service.component": "order-processor",
        }
    ) as root_span:

        # Child span: database fetch
        with tracer.start_as_current_span("db.orders.select") as db_span:
            db_span.set_attribute("db.system", "postgresql")
            db_span.set_attribute("db.name", "orders_db")
            db_span.set_attribute("db.statement", "SELECT * FROM orders WHERE id = $1")
            db_span.set_attribute("db.operation", "SELECT")
            order = fetch_order_from_db(order_id)
            db_span.set_attribute("db.rows_returned", 1)

        root_span.add_event("order-fetched", {"order.status": order["status"]})

        # Child span: inventory check (external HTTP call — auto-instrumented)
        with tracer.start_as_current_span("inventory.check") as inv_span:
            inv_span.set_attribute("peer.service", "inventory-service")
            inv_span.set_attribute("http.method", "GET")
            inv_span.set_attribute("http.url", f"https://inventory-svc/check/{order['sku']}")
            available = check_inventory(order["sku"])

        if not available:
            root_span.set_attribute("order.cancelled_reason", "out_of_stock")
            root_span.set_status(trace.StatusCode.ERROR, "Out of stock")
            raise ValueError(f"SKU {order['sku']} is out of stock")

        # Child span: payment processing
        with tracer.start_as_current_span("payment.charge") as pay_span:
            pay_span.set_attribute("peer.service", "payment-gateway")
            pay_span.set_attribute("payment.method", order["payment_method"])
            pay_span.set_attribute("payment.amount_usd", order["amount"])
            try:
                receipt = charge_payment(order)
                pay_span.set_attribute("payment.receipt_id", receipt["id"])
                pay_span.set_status(trace.StatusCode.OK)
            except PaymentError as e:
                pay_span.record_exception(e)  # Adds exception event with stack trace
                pay_span.set_status(trace.StatusCode.ERROR, str(e))
                root_span.set_status(trace.StatusCode.ERROR, "Payment failed")
                raise

        root_span.add_event("order-complete", {
            "order.id":      order_id,
            "payment.id":    receipt["id"],
        })
        root_span.set_status(trace.StatusCode.OK)
        return {"order_id": order_id, "status": "processed"}
```

### Go — OTel with Cloud Trace Exporter

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"

    texporter "github.com/GoogleCloudPlatform/opentelemetry-operations-go/exporter/trace"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/propagation"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    "go.opentelemetry.io/otel/sdk/resource"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

func initTracing(ctx context.Context, projectID string) (*sdktrace.TracerProvider, error) {
    exporter, err := texporter.New(texporter.WithProjectID(projectID))
    if err != nil {
        return nil, fmt.Errorf("creating trace exporter: %w", err)
    }

    res, _ := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName("order-service"),
            semconv.ServiceVersion("v2.3.1"),
            attribute.String("deployment.environment", "production"),
        ),
    )

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithSampler(
            sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1)),
        ),
        sdktrace.WithResource(res),
    )

    otel.SetTracerProvider(tp)
    // Support W3C traceparent + Google Cloud trace format
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    return tp, nil
}

func processOrder(ctx context.Context, orderID, userID string) error {
    tracer := otel.Tracer("order-service")

    ctx, rootSpan := tracer.Start(ctx, "process-order",
        sdktrace.WithAttributes(
            attribute.String("order.id", orderID),
            attribute.String("user.id", userID),
        ),
    )
    defer rootSpan.End()

    // DB span
    ctx, dbSpan := tracer.Start(ctx, "db.orders.select",
        sdktrace.WithAttributes(
            semconv.DBSystemPostgreSQL,
            semconv.DBName("orders_db"),
            semconv.DBStatement("SELECT * FROM orders WHERE id = $1"),
        ),
    )
    order, err := fetchOrderFromDB(ctx, orderID)
    dbSpan.End()

    if err != nil {
        rootSpan.RecordError(err)
        rootSpan.SetStatus(codes.Error, err.Error())
        return err
    }

    // Payment span
    ctx, paySpan := tracer.Start(ctx, "payment.charge",
        sdktrace.WithAttributes(
            attribute.String("peer.service", "payment-gateway"),
            attribute.Float64("payment.amount_usd", order.Amount),
        ),
    )
    if err := chargePayment(ctx, order); err != nil {
        paySpan.RecordError(err)
        paySpan.SetStatus(codes.Error, err.Error())
        paySpan.End()
        rootSpan.SetStatus(codes.Error, "payment failed")
        return err
    }
    paySpan.SetStatus(codes.Ok, "")
    paySpan.End()

    rootSpan.SetStatus(codes.Ok, "")
    rootSpan.AddEvent("order-complete", sdktrace.WithAttributes(
        attribute.String("order.id", orderID),
    ))
    return nil
}

// Propagate trace context in outbound HTTP calls
func makeTracedHTTPRequest(ctx context.Context, url string) (*http.Response, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    // Inject trace context headers (traceparent, etc.)
    otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))
    return http.DefaultClient.Do(req)
}

// Extract trace context from incoming HTTP requests
func traceMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := otel.GetTextMapPropagator().Extract(r.Context(), propagation.HeaderCarrier(r.Header))
        tracer := otel.Tracer("order-service")
        ctx, span := tracer.Start(ctx, r.URL.Path)
        defer span.End()
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### Log-Trace Correlation (Full Pattern)

```python
# Complete pattern: inject trace context into every log entry
import structlog
import google.cloud.logging
from opentelemetry import trace as otel_trace
from functools import wraps

PROJECT_ID = "my-project"
_gcp_logger = google.cloud.logging.Client(project=PROJECT_ID).logger("order-service")


def get_trace_fields(project_id: str) -> dict:
    """Return Cloud Logging trace/span fields from current OTel context."""
    span = otel_trace.get_current_span()
    ctx  = span.get_span_context()
    if ctx.is_valid:
        return {
            "logging.googleapis.com/trace":   f"projects/{project_id}/traces/{ctx.trace_id:032x}",
            "logging.googleapis.com/spanId":  f"{ctx.span_id:016x}",
            "logging.googleapis.com/traceSampled": ctx.trace_flags.sampled,
        }
    return {}


def structured_log(message: str, severity: str = "INFO", **kwargs):
    """Write a structured log entry with automatic trace correlation."""
    payload = {"message": message, **kwargs, **get_trace_fields(PROJECT_ID)}
    _gcp_logger.log_struct(payload, severity=severity)


# Usage
def process_payment(order_id: str, amount: float):
    with tracer.start_as_current_span("process-payment") as span:
        span.set_attribute("order.id", order_id)
        structured_log(
            "Processing payment",
            order_id=order_id,
            amount_usd=amount,
            component="payment",
        )
        # ... payment logic
        structured_log(
            "Payment completed",
            order_id=order_id,
            severity="INFO",
        )
```

---

## 15. Cloud Trace — gcloud & Analysis

```bash
# List recent traces (last 20)
gcloud trace list --project=PROJECT_ID --limit=20

# Get a specific trace with all spans
gcloud trace describe TRACE_ID --project=PROJECT_ID

# List slow traces (latency > 1000ms) in a time range
gcloud trace list \
  --project=PROJECT_ID \
  --filter="latency>1000ms" \
  --start-time="2024-01-15T00:00:00Z" \
  --end-time="2024-01-15T23:59:59Z" \
  --limit=50 \
  --format="table(traceId,displayName,endTime,durationMs)"
```

**Cloud Trace Console Analysis:**

- **Latency distribution**: histogram of all trace latencies; filter by service/method
- **Trace detail view**: waterfall diagram of all spans in a trace; click to see attributes and events
- **Scatter plot**: latency over time; identify regression timestamps
- **Auto-insights**: Cloud Trace automatically surfaces new latency regressions
- **Log correlation**: click any span → view correlated log entries with matching trace ID

> **💡 Best Practice:** Set span names to fixed strings like `"db.orders.select"`, not dynamic like `"SELECT order ord-12345"`. Dynamic names create high-cardinality trace groupings, destroying the latency distribution view and making analysis impossible.

---

## PART 4 — CLOUD PROFILER

## 16. Cloud Profiler — Concepts & Architecture

### Continuous vs. Point-in-Time Profiling

| Aspect | Continuous Profiling (Cloud Profiler) | Point-in-time (pprof/py-spy) |
|---|---|---|
| Overhead | ~1–2% CPU (statistical sampling) | 0–5% depending on method |
| Duration | Always on in production | Short manual sessions |
| Data retention | 30 days | Until you delete the file |
| Comparison | Previous deploys vs. current | Manual file comparison |
| Use case | Production regression detection | Deep one-off investigation |
| Setup | Agent initialization at startup | Run external tool against process |

### Profile Types

| Profile Type | Languages | Measures |
|---|---|---|
| **CPU Time** | Go, Java, Node.js, Python | Time CPU is actively executing (user + kernel) |
| **Wall Time** | Go, Python | Elapsed clock time (includes I/O wait, sleep) |
| **Heap (allocated)** | Go, Java, Node.js | Live objects in heap memory |
| **Heap (space)** | Java | JVM heap space by region |
| **Contention** | Go, Java | Goroutine/thread blocking on mutexes/locks |
| **Threads** | Java | Active thread count and state |
| **Goroutines** | Go | Live goroutine count and stack |

### Flame Graph Interpretation

```
Flame Graph — X-axis: time proportion, Y-axis: call stack depth

     ┌──────────────────────────────────────────────────────┐  100%
     │              handleHTTPRequest                        │
     ├──────────────────────────┬───────────────────────────┤
     │   processOrder (42%)     │   serializeResponse (35%) │
     ├──────────────┬───────────┤                           │
     │ fetchFromDB  │ callPaymt │  json.Marshal (35%) ← WIDE│
     │ (28%)        │ (14%)     │  frames = hot path        │
     ├──────┬───────┤           │                           │
     │ sql  │ conn  │           │                           │
     │ exec │ pool  │           │                           │
     └──────────────────────────────────────────────────────┘
                                         time →

Wide frames = high CPU cost → optimization targets
Deep stacks = complex call chains (may indicate recursion issues)
```

---

## 17. Cloud Profiler — Agent Setup

### Python

```python
# requirements.txt: google-cloud-profiler>=4.0.0

import googlecloudprofiler
import logging

def start_profiler(project_id: str, service: str, version: str) -> None:
    """Start Cloud Profiler agent. Call once at application startup."""
    try:
        googlecloudprofiler.start(
            service=service,
            service_version=version,
            project_id=project_id,
            # verbose=3,           # Uncomment for debug logging
            # disable_cpu_profiling=False,
            # disable_wall_profiling=False,
            # disable_heap_profiling=False,  # Python heap profiling (3.9+)
        )
        logging.info(f"Cloud Profiler started for {service}@{version}")
    except (ValueError, NotImplementedError) as e:
        # NotImplementedError: not on GCP environment
        logging.warning(f"Cloud Profiler not started: {e}")

# In your main / app factory:
start_profiler(
    project_id="my-project",
    service="order-service",
    version="v2.3.1",  # Matches your container image tag
)
```

### Go

```go
package main

import (
    "log"
    "cloud.google.com/go/profiler"
)

func startProfiler(projectID, service, version string) {
    cfg := profiler.Config{
        Service:        service,
        ServiceVersion: version,
        ProjectID:      projectID,
        // Enable contention (mutex) profiling
        MutexProfiling: true,
        // Force GC before heap snapshot (more accurate but higher overhead)
        // AllocForceGC: true,
        // Enable allocation profiling (tracks all allocations, not just live heap)
        // AllocForceGC: false,
    }
    if err := profiler.Start(cfg); err != nil {
        log.Printf("Warning: Cloud Profiler not started: %v", err)
        // Don't fatal — profiler failure should not crash the service
    }
}

func main() {
    startProfiler("my-project", "order-service", "v2.3.1")
    // ... rest of main
}
```

### Java

```xml
<!-- pom.xml dependency -->
<dependency>
  <groupId>com.google.cloud</groupId>
  <artifactId>google-cloud-profiler</artifactId>
  <version>2.0.0</version>
</dependency>
```

```java
import com.google.devtools.cloudprofiler.Profiler;

public class Application {
    public static void main(String[] args) throws Exception {
        // Start profiler before application initialization
        try {
            Profiler.start(Profiler.newBuilder()
                .setService("order-service")
                .setServiceVersion("v2.3.1")
                // .setProjectId("my-project")  // Auto-detected from metadata server
                .build());
        } catch (Exception e) {
            // Log warning but don't fail startup
            System.err.println("Cloud Profiler not started: " + e.getMessage());
        }

        // ... spring boot or other framework startup
        SpringApplication.run(Application.class, args);
    }
}
```

```bash
# Alternative: Java agent (no code change required)
java -agentpath:/opt/cprof/profiler_java_agent.so=\
  -cprof_service=order-service,\
  -cprof_service_version=v2.3.1,\
  -cprof_project_id=my-project \
  -jar order-service.jar
```

### Node.js

```javascript
// npm install @google-cloud/profiler

// MUST be the very first line of your application entry point
const profiler = require('@google-cloud/profiler');

profiler.start({
  serviceContext: {
    service: 'order-service',
    version: 'v2.3.1',
  },
  projectId: 'my-project',
  // disableHeap: false,   // Heap profiling enabled by default
  // disableCpu: false,    // CPU profiling enabled by default
}).catch((err) => {
  console.warn(`Cloud Profiler not started: ${err.message}`);
});

// Rest of your application
const express = require('express');
// ...
```

### IAM and Environment Requirements

| Environment | IAM Required | Additional Setup |
|---|---|---|
| GKE (Workload Identity) | `roles/cloudprofiler.agent` on SA | No extra config |
| GCE | VM service account needs `roles/cloudprofiler.agent` | Or `cloud-platform` OAuth scope |
| Cloud Run | Default SA or custom SA needs `roles/cloudprofiler.agent` | No extra config |
| Local / non-GCP | Application Default Credentials | `gcloud auth application-default login` |

```bash
# Grant profiler.agent role to a service account
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:order-service-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudprofiler.agent"
```

---

## 18. Cloud Profiler — Analysis & Best Practices

### Reading Flame Graphs

- **Wide frames at the top of the stack**: these functions consume the most time — primary optimization targets
- **Deep stacks**: long chains of single-line frames often indicate sequential blocking calls or unnecessary abstraction
- **GC frames (wide)**: excessive garbage collection → look for allocation-heavy code above it
- **Lock/mutex frames (wide)**: contention — reduce critical section size or use lock-free data structures
- **Unexpected framework frames (wide)**: e.g., JSON marshal consuming 35% → use streaming serializers

### Common Performance Anti-Patterns

| Flame Graph Pattern | Root Cause | Fix |
|---|---|---|
| `json.Marshal` / `json.Unmarshal` takes 30%+ | Large objects serialized on every request | Use `jsoniter`, streaming encoder, or cache serialized form |
| `runtime.gc` / `runtime.mallocgc` wide frames | Excessive memory allocation / leak | Profile heap; reduce allocations; use sync.Pool |
| `sync.Mutex.Lock` wide frames | High lock contention | Reduce critical section; use sharded locks or channels |
| `http.Client.Do` wide frames | No connection pooling or DNS re-resolution | Reuse `http.Client` with `Transport`; increase `MaxIdleConns` |
| `log.Printf` / `zap.Sugar` wide frames | Excessive logging in hot path | Reduce log verbosity; use structured loggers; batch logs |
| `reflect.*` wide frames | Heavy use of reflection (common in ORMs) | Use code-generated marshalers; avoid interface{} in hot paths |

### Comparing Profiles (Regression Detection)

In Cloud Console → Profiler:
1. Select your service and profile type
2. Click **Compare** 
3. Choose two time ranges (e.g., before and after a deploy)
4. Frames highlighted in **red** = regressed (more time), **green** = improved

> **💡 Best Practice:** Tag `service_version` precisely to your container image tag or commit SHA. This makes before/after comparison across deploys exact and reproducible.

---

## PART 5 — ERROR REPORTING

## 19. Error Reporting — Concepts & Architecture

### How Error Reporting Works

```
Error occurs in application
      │
      ├─── Option A: Cloud Logging auto-detection
      │    severity=ERROR + valid stack trace in log entry
      │    → Error Reporting parses the stack trace
      │    → Normalizes the stack (removes line numbers, variable parts)
      │    → Creates/updates error group (fingerprint = canonical stack)
      │
      ├─── Option B: Direct Error Reporting API
      │    client.report_exception() / client.Report()
      │    → Stack trace sent directly to Error Reporting API
      │
      └─── Option C: Error Reporting API key for client-side (browser)
           → JS client sends stack traces directly

Error Group in Error Reporting:
  ├── Group ID (fingerprint hash of canonical stack)
  ├── Service name + version
  ├── First seen / Last seen timestamps
  ├── Occurrence count (last hour, day, week)
  ├── Affected users count
  ├── Representative stack trace
  ├── Status: OPEN | ACKNOWLEDGED | RESOLVED | MUTED
  └── Cloud Monitoring link (errorgroup metric)
```

### Error States Lifecycle

```
OPEN ──(team acknowledges)──▶ ACKNOWLEDGED ──(fix deployed)──▶ RESOLVED
  ▲                                                                   │
  │                                                                   │
  └──────────────── auto-reopen if error recurs after resolution ─────┘

MUTED: suppressed from UI (not fixed, just silenced)
       Use for known/acceptable errors you can't fix immediately
```

### Supported Languages (Auto-grouping)

Go, Java, Node.js, Python, Ruby, PHP, .NET — stack traces from these languages are automatically parsed and grouped by Cloud Logging → Error Reporting integration.

> **⚠️ Gotcha:** Error Reporting only auto-creates groups from **structured** stack traces. A plain string log entry with `severity=ERROR` but no parseable stack trace will NOT create an error group. For Python, use `logger.exception()` or `logger.error("msg", exc_info=True)` to include the stack.

---

## 20. Error Reporting — Instrumentation

### Python

```python
from google.cloud import error_reporting
import logging
import traceback

PROJECT_ID = "my-project"

# Initialize once at startup
er_client = error_reporting.Client(
    project=PROJECT_ID,
    service="order-service",
    version="v2.3.1",
)

# ── Option 1: Direct exception reporting ─────────────────────────────────────

def process_payment(order_id: str, amount: float, request_info: dict) -> dict:
    try:
        result = payment_gateway.charge(amount)
        return result
    except PaymentGatewayError as e:
        er_client.report_exception(
            http_context=error_reporting.HTTPContext(
                method=request_info.get("method", "POST"),
                url=f"/api/v1/orders/{order_id}/pay",
                user_agent=request_info.get("user_agent", ""),
                referrer=request_info.get("referrer", ""),
                response_status_code=503,
            )
        )
        raise  # Re-raise after reporting

# ── Option 2: Flask middleware (auto-catch all unhandled exceptions) ──────────

from flask import Flask
from google.cloud.error_reporting import make_report_exception_middleware

app = Flask(__name__)
app.wsgi_app = make_report_exception_middleware(app.wsgi_app, er_client)

# ── Option 3: Via Cloud Logging (recommended — zero-overhead path) ────────────
# If using google-cloud-logging, ERROR logs with stack traces auto-flow to Error Reporting
# Just use logger.exception() for caught exceptions:

logger = logging.getLogger("order-service")

def process_order(order_id: str):
    try:
        do_work(order_id)
    except ValueError as e:
        logger.exception(  # Logs ERROR with full stack trace → Error Reporting picks it up
            f"Invalid order data for {order_id}",
            extra={"json_fields": {"order_id": order_id, "component": "order-validator"}}
        )
        raise
```

### Go

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"

    "cloud.google.com/go/errorreporting"
)

var errorClient *errorreporting.Client

func initErrorReporting(ctx context.Context, projectID string) {
    var err error
    errorClient, err = errorreporting.NewClient(ctx, projectID,
        errorreporting.Config{
            ServiceName:    "order-service",
            ServiceVersion: "v2.3.1",
            OnError: func(err error) {
                log.Printf("Error Reporting failed: %v", err)
            },
        },
    )
    if err != nil {
        log.Printf("Warning: Error Reporting not initialized: %v", err)
    }
}

// Report with HTTP context
func processOrder(w http.ResponseWriter, r *http.Request) {
    orderID := r.URL.Query().Get("order_id")

    if err := doOrderWork(r.Context(), orderID); err != nil {
        // Report with full HTTP context
        errorClient.Report(errorreporting.Entry{
            Error: fmt.Errorf("order processing failed: %w", err),
            Req:   r,        // Includes method, URL, user agent
            User:  getUserID(r),
        })
        http.Error(w, "Internal Server Error", http.StatusInternalServerError)
        return
    }
}

// Middleware to catch panics and report them
func panicRecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                err := fmt.Errorf("panic: %v", rec)
                errorClient.Report(errorreporting.Entry{
                    Error: err,
                    Req:   r,
                })
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

func main() {
    ctx := context.Background()
    initErrorReporting(ctx, "my-project")
    defer errorClient.Close()

    mux := http.NewServeMux()
    mux.HandleFunc("/api/v1/orders", processOrder)
    http.ListenAndServe(":8080", panicRecoveryMiddleware(mux))
}
```

### Java — Cloud Logging Logback Appender (Auto Error Reporting)

```xml
<!-- logback.xml — errors logged via Cloud Logging appender automatically -->
<!-- appear in Error Reporting with full stack traces                       -->
<configuration>
  <appender name="CLOUD" class="com.google.cloud.logging.logback.LoggingAppender">
    <log>order-service</log>
    <enhancer>com.google.cloud.logging.logback.LoggingEventEnhancer</enhancer>
    <flushLevel>WARNING</flushLevel>
    <resourceType>cloud_run_revision</resourceType>
  </appender>

  <!-- Console for local development -->
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss} %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="CLOUD"/>
    <appender-ref ref="CONSOLE"/>
  </root>
</configuration>
```

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public void processOrder(String orderId) {
        try {
            doWork(orderId);
        } catch (PaymentException e) {
            // This ERROR log with stack trace → Cloud Logging → Error Reporting
            log.error("Payment failed for order {}", orderId, e);
            throw e;
        }
    }
}
```

### Error Reporting Alert in Cloud Monitoring

```bash
# Alert when a new error group is created (new exception type seen)
gcloud alpha monitoring policies create \
  --display-name="New Error Group — order-service" \
  --condition-filter='
    metric.type="clouderrorreporting.googleapis.com/errorgroup/count"
    AND resource.type="clouderrorreporting.googleapis.com/errorGroup"
    AND metric.label.service_name="order-service"
  ' \
  --condition-threshold-value=1 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=0s \
  --condition-aggregations-per-series-aligner=ALIGN_COUNT \
  --condition-aggregations-alignment-period=60s \
  --notification-channels=CHANNEL_ID \
  --documentation-content="New error group detected in order-service. Triage in Error Reporting: https://console.cloud.google.com/errors" \
  --project=PROJECT_ID
```

---

## PART 6 — UNIFIED OBSERVABILITY

## 21. OpenTelemetry Collector — GCP Setup

### Why Use the OTel Collector

```
Without Collector:               With Collector:
  Service A ──▶ GCP Trace          Service A ──▶ ┌────────────────┐
  Service B ──▶ GCP Monitoring     Service B ──▶ │  OTel          │──▶ Cloud Trace
  Service C ──▶ Cloud Logging      Service C ──▶ │  Collector     │──▶ Cloud Monitoring
                                                  │                │──▶ Cloud Logging
  Problem: Config in every app     Service D ──▶ │  (DaemonSet)   │──▶ GMP
                                                  └────────────────┘──▶ Datadog (multi-backend)

Benefits: Centralized config, batching, retry, tail-sampling, multi-backend fanout
```

### OTel Collector GCP Configuration (Kubernetes DaemonSet)

```yaml
# otel-collector-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: observability
data:
  config.yaml: |
    receivers:
      # Receive OTel OTLP signals from applications
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

      # Scrape Prometheus /metrics endpoints from pods
      prometheus:
        config:
          scrape_configs:
          - job_name: kubernetes-pods
            kubernetes_sd_configs:
            - role: pod
            relabel_configs:
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
              action: keep
              regex: "true"
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
              action: replace
              target_label: __address__
              regex: (.+)
              replacement: $1

      # Collect host metrics (node CPU, memory, disk, network)
      hostmetrics:
        collection_interval: 30s
        scrapers:
          cpu: {}
          memory: {}
          disk: {}
          network: {}

      # Collect kubelet stats
      kubeletstats:
        collection_interval: 30s
        auth_type: serviceAccount
        endpoint: ${K8S_NODE_IP}:10250
        insecure_skip_verify: true

    processors:
      # Prevent OOM: limit memory usage
      memory_limiter:
        limit_mib: 512
        spike_limit_mib: 128
        check_interval: 5s

      # Batch for efficiency (reduce API calls)
      batch:
        send_batch_size: 1000
        send_batch_max_size: 1500
        timeout: 10s

      # Auto-detect GCP resource attributes (project_id, zone, cluster_name)
      resourcedetection:
        detectors: [gcp, k8snode, env]
        timeout: 10s
        override: false

      # Add/modify resource attributes
      resource:
        attributes:
        - key: service.name
          from_attribute: k8s.container.name
          action: insert
        - key: deployment.environment
          value: production
          action: upsert

      # Filter out noisy metrics (reduce cost)
      filter/drop_debug_metrics:
        metrics:
          exclude:
            match_type: regexp
            metric_names:
            - ".*_debug_.*"
            - "go_gc_.*"   # Drop internal Go GC metrics if too noisy

      # Tail-based sampling (keep all error traces, sample 10% of success)
      tail_sampling:
        decision_wait: 30s
        num_traces: 50000
        expected_new_traces_per_sec: 1000
        policies:
        - name: errors-policy
          type: status_code
          status_code: {status_codes: [ERROR]}
        - name: slow-traces-policy
          type: latency
          latency: {threshold_ms: 1000}
        - name: probabilistic-policy
          type: probabilistic
          probabilistic: {sampling_percentage: 10}

    exporters:
      # Export to Cloud Monitoring + Cloud Trace + Cloud Logging
      googlecloud:
        project: PROJECT_ID
        log:
          default_log_name: "opentelemetry.io/collector-exported-log"
        metric:
          prefix: "workload.googleapis.com"
        trace: {}

      # Export metrics to Google Managed Prometheus (GMP)
      googlemanagedprometheus:
        project: PROJECT_ID

      # Debug exporter (disable in production)
      # debug:
      #   verbosity: detailed

    extensions:
      health_check:
        endpoint: 0.0.0.0:13133
      pprof:
        endpoint: 0.0.0.0:1777
      zpages:
        endpoint: 0.0.0.0:55679

    service:
      extensions: [health_check, pprof, zpages]

      pipelines:
        traces:
          receivers:  [otlp]
          processors: [memory_limiter, resourcedetection, resource, tail_sampling, batch]
          exporters:  [googlecloud]

        metrics/otel:
          receivers:  [otlp, hostmetrics, kubeletstats]
          processors: [memory_limiter, resourcedetection, resource, filter/drop_debug_metrics, batch]
          exporters:  [googlecloud]

        metrics/prometheus:
          receivers:  [prometheus]
          processors: [memory_limiter, resourcedetection, batch]
          exporters:  [googlemanagedprometheus]

        logs:
          receivers:  [otlp]
          processors: [memory_limiter, resourcedetection, resource, batch]
          exporters:  [googlecloud]
```

```yaml
# otel-collector-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-collector
  namespace: observability
spec:
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      serviceAccountName: otel-collector
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:0.96.0
        args: ["--config=/conf/config.yaml"]
        env:
        - name: K8S_NODE_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        - name: K8S_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        resources:
          limits:
            cpu: "500m"
            memory: "768Mi"
          requests:
            cpu: "100m"
            memory: "256Mi"
        ports:
        - containerPort: 4317  # OTLP gRPC
        - containerPort: 4318  # OTLP HTTP
        - containerPort: 13133 # Health check
        livenessProbe:
          httpGet:
            path: /
            port: 13133
          initialDelaySeconds: 10
          periodSeconds: 30
        volumeMounts:
        - name: config
          mountPath: /conf
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: otel-collector-config
      - name: varlog
        hostPath:
          path: /var/log
      tolerations:
      - operator: Exists  # Run on all nodes including tainted ones
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: observability
spec:
  selector:
    app: otel-collector
  ports:
  - name: otlp-grpc
    port: 4317
    targetPort: 4317
  - name: otlp-http
    port: 4318
    targetPort: 4318
```

### OTel SDK Configuration: App → Collector

```python
# Point your OTel SDK at the collector instead of directly at GCP
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

# Use the DaemonSet collector on the same node (hostPort)
COLLECTOR_ENDPOINT = "http://$(NODE_IP):4317"  # Inject NODE_IP via downward API

trace_exporter = OTLPSpanExporter(endpoint=COLLECTOR_ENDPOINT, insecure=True)
metric_exporter = OTLPMetricExporter(endpoint=COLLECTOR_ENDPOINT, insecure=True)
```

### OTel SDK Auto vs. Manual Instrumentation

| Approach | Effort | Coverage | Customization |
|---|---|---|---|
| **Auto-instrumentation** (zero-code) | Minimal | HTTP, DB, gRPC, messaging | Low — generic span names/attrs |
| **Auto + custom spans** | Medium | All auto + business spans | High — add business context |
| **Fully manual** | High | Precise control | Complete |

> **💡 Best Practice:** Always combine auto-instrumentation (for free infrastructure spans: HTTP in/out, DB queries, gRPC) with manual spans for business operations (`"process-order"`, `"payment.charge"`). The combination gives you full stack coverage with relevant business context.

---

## 22. Observability for GKE — Complete Setup

### GKE Built-in Metrics (Free)

```
kubernetes.io/container/cpu/core_usage_time
kubernetes.io/container/memory/used_bytes
kubernetes.io/container/restart_count
kubernetes.io/node/cpu/total_cores
kubernetes.io/node/memory/total_bytes
kubernetes.io/pod/volume/used_bytes
kubernetes.io/pod/network/received_bytes_count
kubernetes.io/pod/network/sent_bytes_count
container.googleapis.com/container/cpu/usage_time
container.googleapis.com/container/memory/used_bytes
```

### GKE Observability Checklist

```
Cluster Setup:
  [✓] Enable Managed Service for Prometheus: --enable-managed-prometheus
  [✓] Enable Cloud Logging for system + workloads: --logging=SYSTEM,WORKLOAD
  [✓] Enable Cloud Monitoring: --monitoring=SYSTEM
  [✓] Enable Workload Identity: --workload-pool=PROJECT_ID.svc.id.goog
  [✓] Grant collector SA: roles/monitoring.metricWriter, roles/cloudtrace.agent,
      roles/logging.logWriter, roles/cloudprofiler.agent

Application Instrumentation:
  [✓] OTel SDK initialized with service.name + service.version resource attributes
  [✓] Trace context propagated in all outbound HTTP/gRPC calls
  [✓] Trace IDs injected into all structured log entries
  [✓] Profiler agent started at container init (before first request)
  [✓] Error Reporting configured (or Cloud Logging auto-detection enabled)
  [✓] /metrics endpoint exposed for Prometheus scraping

PodMonitoring CRs:
  [✓] PodMonitoring created for each service with a /metrics endpoint
  [✓] interval: 30s, timeout: 10s, path: /metrics

Dashboards (create these):
  [✓] Cluster overview: node CPU%, node mem%, pod restart count, OOMKilled events
  [✓] Service health: request rate, error rate (%), p50/p95/p99 latency per service
  [✓] SLO dashboard: error budget remaining, burn rate over 1h/6h/24h/30d
  [✓] Resource saturation: CPU throttling, OOM events, PVC usage

Alerts (create these):
  [✓] Pod OOMKilled: k8s_container restart_count increasing + reason=OOMKilled
  [✓] High pod restart rate: restart_count > 5 in 15min
  [✓] Node NotReady: node condition not ready for > 2 min
  [✓] API server latency: kube_apiserver_request_duration_seconds p99 > 1s
  [✓] Deployment replica mismatch: desired != ready replicas for > 5min
  [✓] PVC usage > 80%
  [✓] Custom service SLO burn rate (fast + slow burn)
```

### Essential GKE Alert Filters

```bash
# OOMKilled containers
resource.type="k8s_container"
metric.type="kubernetes.io/container/restart_count"
# + condition: restart_count increasing

# Node NotReady
resource.type="k8s_node"
metric.type="kubernetes.io/node/status_condition"
metric.labels.condition="Ready"
metric.labels.status="false"

# High CPU throttling (> 25% of CPU time throttled)
resource.type="k8s_container"
metric.type="kubernetes.io/container/cpu/request_utilization"
# threshold > 0.25

# Workload not available (Deployment desired != ready)
# Use kubernetes.io/pod/phase metric and alert on ratio
```

---

## 23. Key IAM Roles for Observability

| Role | Resource Level | Description |
|---|---|---|
| `roles/monitoring.viewer` | Project | Read metrics, dashboards, alert policies, uptime checks |
| `roles/monitoring.editor` | Project | Create/edit dashboards, alert policies, uptime checks |
| `roles/monitoring.admin` | Project | Full monitoring admin including notification channels |
| `roles/monitoring.metricWriter` | Project | Write custom/workload metrics (for agents, OTel Collector) |
| `roles/monitoring.metricReader` | Project | Read time series data via API |
| `roles/logging.viewer` | Project / Bucket | Read log entries in `_Default` and `_Required` |
| `roles/logging.privateLogViewer` | Project | Read all logs including Data Access audit logs |
| `roles/logging.logWriter` | Project | Write log entries (for application service accounts) |
| `roles/logging.bucketWriter` | Log Bucket | Write to a specific log bucket only |
| `roles/logging.viewAccessor` | Log View | Access a specific log view within a bucket |
| `roles/logging.admin` | Project | Full logging admin (sinks, buckets, metrics) |
| `roles/cloudtrace.agent` | Project | Write trace spans (for OTel agents, services) |
| `roles/cloudtrace.user` | Project | Read trace data in Cloud Console and API |
| `roles/cloudtrace.admin` | Project | Full trace admin |
| `roles/cloudprofiler.agent` | Project | Write profiler data (for Profiler agent) |
| `roles/cloudprofiler.user` | Project | Read profiler flame graphs in Cloud Console |
| `roles/errorreporting.writer` | Project | Write error events via Error Reporting API |
| `roles/errorreporting.viewer` | Project | Read error groups and events |
| `roles/errorreporting.admin` | Project | Full admin (delete, manage error groups) |

### Minimum Required SA Permissions (OTel Collector / Application Agent)

```bash
# Bind all required roles for an application/collector service account
SA="otel-collector-sa@PROJECT_ID.iam.gserviceaccount.com"

for role in \
  "roles/monitoring.metricWriter" \
  "roles/cloudtrace.agent" \
  "roles/logging.logWriter" \
  "roles/cloudprofiler.agent" \
  "roles/errorreporting.writer"; do
  gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:$SA" \
    --role="$role"
done
```

---

## 24. Limits & Quotas

| Resource | Limit | Notes |
|---|---|---|
| **Custom metrics per project** | 500 | Quota increase requestable via Support |
| Time series per custom metric descriptor | 200,000 | Total active time series |
| Metric data points per minute (write) | 100,000 | Per project |
| Alert policies per project | 500 | |
| Notification channels per project | 4,000 | |
| Uptime checks per project | 100 | |
| Dashboard widgets per dashboard | 40 | |
| **Log entries per second (write)** | 60,000 | Per project; burst to 120,000 |
| Log entry max size | 256 KB | Larger entries truncated |
| Log bucket max retention | 3,650 days (10 years) | |
| `_Required` bucket retention | 400 days | Non-configurable, non-deletable |
| `_Default` bucket retention | 30 days | Configurable |
| Log sink destinations per project | 200 | |
| Log-based metrics per project | 500 | |
| Log view filters per bucket | 30 | |
| **Trace spans per minute** | 500,000 | Per project |
| Trace data retention | 30 days | Non-configurable |
| Max attributes per span | 128 | |
| Max events per span | 128 | |
| Max links per span | 128 | |
| Free tier: Cloud Trace | 2.5M spans/month | |
| **Profiler data retention** | 30 days | Non-configurable |
| Profiler profiles per service/version | Unlimited | Oldest auto-purged at 30 days |
| **Error Reporting retention** | 30 days | |
| Error groups per project | No hard limit | |
| GMP samples ingested (free) | First 150M samples/month | |

---

## 25. Costs & Cost Optimization

### What's Free

| Product | Free Tier |
|---|---|
| Cloud Monitoring | All GCP built-in metrics (compute, GKE, Cloud Run, Cloud SQL, etc.) |
| Cloud Monitoring | Cloud Audit Logs (Admin Activity, System Event) |
| Cloud Logging | First 50 GiB/month log ingestion |
| Cloud Logging | `_Required` bucket storage (Admin Activity, System Event) |
| Cloud Trace | First 2.5M spans/month |
| Cloud Profiler | Free (no charges) |
| Error Reporting | Free (no charges) |
| GMP | First 150M samples/month |

### What's Billable

| Product | Billing Unit | Approximate Price |
|---|---|---|
| Cloud Monitoring custom metrics | Per MiB of time series data | ~$0.18/MiB |
| Cloud Logging ingestion | Per GiB over 50 GiB/month | ~$0.50/GiB |
| Cloud Logging storage | Per GiB over default retention | ~$0.01/GiB/month |
| Cloud Trace ingestion | Per million spans over 2.5M | ~$0.20/M spans |
| GMP samples | Per million samples over 150M | ~$0.09/M samples |

### Cost Optimization Strategies

**Cloud Logging (highest impact):**
```bash
# 1. Exclude health check / metrics polling logs (often 30-50% of volume)
gcloud logging sinks update _Default \
  --add-exclusion=name=healthchecks,filter='httpRequest.requestUrl=~"/healthz|/readyz|/livez|/metrics"' \
  --project=PROJECT_ID

# 2. Exclude DEBUG logs in production
gcloud logging sinks update _Default \
  --add-exclusion=name=debug-logs,filter='severity=DEBUG' \
  --project=PROJECT_ID

# 3. Reduce Cloud SQL query logs (extremely high volume)
gcloud logging sinks update _Default \
  --add-exclusion=name=sql-queries,filter='resource.type="cloudsql_database" AND textPayload=~"^Statement:"' \
  --project=PROJECT_ID

# 4. Use GCS for archival instead of long Cloud Logging retention
#    GCS: ~$0.02/GiB/month vs Cloud Logging: ~$0.01/GiB/month
#    But GCS allows years vs 3650 day max in Logging
```

**Cloud Monitoring:**
```bash
# 5. Reduce custom metric cardinality (biggest cost driver)
#    BAD:  labels["user_id"] → millions of time series
#    GOOD: labels["endpoint"] → dozens of time series

# 6. Use GMP for high-cardinality metrics (cheaper per time series)
#    GMP billed per sample; Cloud Monitoring billed per MiB of time series

# 7. Use log-based metrics for counts/distributions instead of custom metrics
#    when the data is already in logs — avoids double-ingestion cost
```

**Cloud Trace:**
```bash
# 8. Downsample traces in production (10% is usually sufficient)
sampler = TraceIdRatioBased(0.1)  # 10% sampling

# 9. Use tail-based sampling in OTel Collector to keep 100% of error traces
#    and 10% of success traces — best of both worlds
```

**Cost Monitoring:**
```bash
# Create a budget alert for observability spend
gcloud billing budgets create \
  --billing-account=BILLING_ACCOUNT_ID \
  --display-name="Observability Budget" \
  --budget-amount=500USD \
  --threshold-rule=percent=0.5 \
  --threshold-rule=percent=0.9 \
  --threshold-rule=percent=1.0 \
  --filter-services=services/95FF-2EF5-5EA1   # Cloud Monitoring
```

---

## 26. Common Gotchas & Troubleshooting

| Issue | Root Cause | Diagnosis | Fix |
|---|---|---|---|
| **Missing traces in Cloud Trace** | Exporter not initialized before first request, or sampler = 0 | Check exporter init order; print sampler rate | Initialize `TracerProvider` before any HTTP server starts; verify `sample_rate > 0` |
| **Log-to-trace link broken** | `trace` field uses short hex ID instead of full path | Check log entry `trace` field value | Use `projects/PROJECT_ID/traces/TRACE_ID` format (full path, not just the hex string) |
| **Custom metric "not found" immediately after write** | Metric descriptor auto-creates on first write but takes 2–3 min to appear in UI | Wait 3 min, then check Metrics Explorer | Normal behavior — wait; create descriptor explicitly if you need it pre-created |
| **Log-based metric not incrementing** | Filter doesn't match actual log entries | Test filter in Logs Explorer → should return matching entries | Debug filter in Logs Explorer first, then paste into metric |
| **Profiler data not appearing** | Missing IAM role, wrong project ID, or environment not detected | Check stderr for `failed to create profiler` errors | Verify `roles/cloudprofiler.agent`; add `verbose=3` to see init errors |
| **Error Reporting groups not created** | Log entry has `severity=ERROR` but no parseable stack trace | Check raw log entry — does it have a multi-line stack? | Use `logger.exception()` / `log.Error(err)` to include stack; don't log just the error message |
| **GMP PodMonitoring not scraping** | Selector labels don't match pod labels, or port name wrong | Check `kubectl get podmonitoring -n NAMESPACE -o yaml`; check `gmp-system` collector pod logs | Fix `matchLabels` to match actual pod labels; verify port name matches container spec |
| **Alert not firing** | Metric filter wrong, alignment period longer than metric interval, or notification channel broken | Test filter in Metrics Explorer; check alert incident history | Validate metric filter in Metrics Explorer first; ensure alignment_period matches metric frequency |
| **Missing GKE container logs** | Node missing `logging-write` scope, or Log Router exclusion filters them out | Check node OAuth scopes; check _Default sink exclusions | Add `logging.write` scope; remove or adjust exclusions |
| **OTel Collector OOM** | Batch size too large, missing `memory_limiter` processor, or traffic spike | Check collector pod memory usage; look for `OutOfMemory` events | Add `memory_limiter` before `batch`; reduce `send_batch_max_size`; increase pod memory limit |
| **Data Access audit logs missing** | They are **OFF by default** | Check IAM → Audit Logs — which services have Data Access enabled? | Enable per-service in Console: IAM → Audit Logs → check DATA_READ, DATA_WRITE |
| **Trace spans with wrong service name** | `service.name` resource attribute not set, or set after `TracerProvider` init | Print `resource.Attributes()` at startup | Set `service.name` in Resource before creating `TracerProvider` |
| **Cloud Run traces missing parent span** | Cloud Run doesn't forward `X-Cloud-Trace-Context` from load balancer by default | Check if `traceparent` header is present in Cloud Run logs | Use W3C `traceparent` header; Cloud Run passes it through. Check propagator config. |
| **High log ingestion cost** | Health check / debug logs, Cloud SQL query logs, verbose framework logs | Run Log Explorer query: group by `resource.type` and `logName`; sort by count | Add targeted exclusions to `_Default` sink for high-volume noisy log sources |

---

## 27. Quick Reference — gcloud Commands

```bash
# ── Cloud Monitoring ──────────────────────────────────────────────────────────

# List dashboards
gcloud monitoring dashboards list --project=PROJECT_ID \
  --format="table(name,displayName)"

# Describe a dashboard (JSON)
gcloud monitoring dashboards describe DASHBOARD_ID --project=PROJECT_ID

# List alert policies
gcloud alpha monitoring policies list --project=PROJECT_ID \
  --format="table(name,displayName,enabled)"

# Describe an alert policy
gcloud alpha monitoring policies describe POLICY_ID --project=PROJECT_ID

# Enable/disable an alert policy
gcloud alpha monitoring policies update POLICY_ID --enabled --project=PROJECT_ID
gcloud alpha monitoring policies update POLICY_ID --no-enabled --project=PROJECT_ID

# List notification channels
gcloud alpha monitoring channels list --project=PROJECT_ID

# List uptime checks
gcloud monitoring uptime list-configs --project=PROJECT_ID

# List custom metric descriptors
gcloud beta monitoring metrics-descriptors list \
  --filter="metric.type=starts_with('custom.googleapis.com/')" \
  --project=PROJECT_ID

# Delete a custom metric descriptor (and all its data)
gcloud beta monitoring metrics-descriptors delete \
  "custom.googleapis.com/order_service/request_latency_ms" \
  --project=PROJECT_ID

# List GMP PodMonitorings
kubectl get podmonitoring --all-namespaces

# ── Cloud Logging ─────────────────────────────────────────────────────────────

# Read recent ERROR logs (last 50)
gcloud logging read 'severity>=ERROR' \
  --limit=50 \
  --project=PROJECT_ID \
  --format=json

# Read Cloud Run service logs (last 1 hour)
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="order-service"' \
  --freshness=1h \
  --project=PROJECT_ID \
  --format=json

# Live tail logs (streaming)
gcloud logging tail 'severity>=WARNING' --project=PROJECT_ID

# List all log sinks
gcloud logging sinks list --project=PROJECT_ID

# Describe a sink
gcloud logging sinks describe SINK_NAME --project=PROJECT_ID

# List log buckets
gcloud logging buckets list --project=PROJECT_ID --location=global

# List log metrics
gcloud logging metrics list --project=PROJECT_ID

# Describe a log metric
gcloud logging metrics describe METRIC_NAME --project=PROJECT_ID

# ── Cloud Trace ───────────────────────────────────────────────────────────────

# List recent traces
gcloud trace list --project=PROJECT_ID --limit=20

# Describe a specific trace (all spans)
gcloud trace describe TRACE_ID --project=PROJECT_ID

# List slow traces (> 1s) in a time range
gcloud trace list \
  --project=PROJECT_ID \
  --filter="latency>1000ms" \
  --start-time="2024-01-15T00:00:00Z" \
  --end-time="2024-01-15T23:59:59Z" \
  --format="table(traceId,displayName,endTime)"

# ── Error Reporting ───────────────────────────────────────────────────────────

# List recent error events
gcloud beta error-reporting events list \
  --project=PROJECT_ID \
  --format="table(eventTime,message,serviceContext.service)"

# List error groups
gcloud beta error-reporting groups list --project=PROJECT_ID

# Delete all error events (clear error history)
gcloud beta error-reporting events delete --project=PROJECT_ID

# ── Profiler ─────────────────────────────────────────────────────────────────

# List profiler services
gcloud beta profiler list-profiles \
  --project=PROJECT_ID \
  --deployment-id=order-service

# View profiles for a service (opens browser)
# Navigate to: https://console.cloud.google.com/profiler;project=PROJECT_ID
```

---

## 28. Decision Tree

### Decision 1: Which Observability Signal For My Problem?

```
PROBLEM: "Something is wrong with my system"
│
├─► Is it a performance issue (SLOW)?
│     │
│     ├─► Is it happening across all requests?
│     │     → Start with METRICS: request latency percentiles, saturation
│     │       Then TRACES: find the slow span (which service/DB call)
│     │       Then PROFILER: find the slow code path in that service
│     │
│     └─► Is it intermittent / specific requests?
│           → Start with TRACES: filter by high-latency traces
│             Then LOGS: view correlated logs for those trace IDs
│             Then PROFILER: compare profiles before/after deploy
│
├─► Is it an ERROR (something failing)?
│     │
│     ├─► Is it a new exception type?
│     │     → ERROR REPORTING: new error group with stack trace + count
│     │
│     ├─► Is it elevated error rate?
│     │     → METRICS: error_rate alert fired → LOGS: find failing requests
│     │
│     └─► Is it a specific user/order/request?
│           → LOGS: search by order_id, user_id, trace_id
│             TRACES: find the specific request trace
│
├─► Is it a MEMORY problem (OOM, leak)?
│     → PROFILER (Heap profile): find allocation hotspots
│       METRICS: container memory usage, OOMKilled restarts
│       LOGS: OOMKilling events in k8s_node logs
│
└─► Is it INCORRECT behavior (wrong results, data bug)?
      → LOGS: detailed event context (structured fields)
        TRACES: find which service returned wrong data
        (Metrics and Profiler won't help here)
```

### Decision 2: Custom Metrics vs. Log-Based Metrics vs. GMP

```
I need to track a metric for alerting/dashboarding
│
├─► Is the data already in my logs (structured JSON)?
│     YES → Use LOG-BASED METRIC (no code change, no extra cost beyond log ingestion)
│            Best for: error counts, specific event rates, log-derived latency
│
├─► Is this a high-cardinality metric (>1000 unique label value combinations)?
│     YES → Use GMP (Google Managed Prometheus)
│            - 24-month retention
│            - Lower per-sample cost than Cloud Monitoring for high cardinality
│            - Use if you already have Prometheus /metrics endpoint
│
├─► Is this a business KPI or app-internal metric NOT in logs?
│   AND low cardinality (< 1000 time series)?
│     YES → Use CLOUD MONITORING CUSTOM METRIC
│            - Simple alerting, native dashboard widgets
│            - Good for: order count, checkout rate, queue depth
│
└─► Are you migrating from self-managed Prometheus?
      YES → Use GMP with PodMonitoring CRD + existing /metrics endpoint
             No application code change required
```

### Decision 3: Log Retention Strategy

```
How long do I need to retain these logs?
│
├─► < 30 days → Default `_Default` bucket (no extra cost)
│
├─► 30 days – 10 years (compliance, audit) → Custom Log Bucket
│     - Up to 3650 days retention in Cloud Logging
│     - Enable Log Analytics for SQL queries
│     - Cost: ~$0.01/GiB/month (Cloud Logging storage)
│
├─► > 10 years OR need minimal cost for cold logs → GCS Archival
│     - Create a log sink to Cloud Storage (GCS)
│     - Cost: ~$0.004/GiB/month (Nearline/Coldline)
│     - Trade-off: no query interface; use BigQuery with external table
│
└─► Need long-term SQL analysis + low query latency → BigQuery Export
      - Create a log sink to BigQuery (partitioned tables)
      - Cost: ~$0.02/GiB storage + query cost
      - Best for: security analysis, compliance reporting, dashboards over months
      - Compatible with Data Studio, Looker, Grafana BigQuery plugin
```

### Decision 4: Log Analytics vs. BigQuery Direct

```
I need to query logs with SQL
│
├─► Query data < 30 days AND log bucket has analytics enabled?
│     → USE LOG ANALYTICS (Cloud Console → Logging → Log Analytics)
│       - Free for buckets with Log Analytics enabled
│       - No export lag; data is immediately queryable
│       - Same SQL dialect as BigQuery
│
├─► Need to query > 30 days of data?
│     → USE BIGQUERY (with log sink + partitioned tables)
│       - Create a sink to BigQuery
│       - Full BigQuery SQL, joins, window functions
│
└─► Need to join logs with other business data (orders DB, user DB)?
      → USE BIGQUERY always
        - Log Analytics doesn't support cross-dataset joins
        - BigQuery can join log tables with application dataset tables
```

### Decision 5: OTel SDK Auto vs. Manual Instrumentation

```
How should I instrument my service?
│
├─► New service, greenfield?
│     → Start with AUTO-INSTRUMENTATION for HTTP/DB/gRPC (zero effort)
│       + Add MANUAL spans for business operations (process-order, charge-payment)
│       + Inject trace context into all log entries
│       This gives 90% coverage for 20% of the effort
│
├─► Existing service with no instrumentation?
│     → Add auto-instrumentation library first (biggest bang for buck)
│       Monitor what you get → add manual spans where gaps exist
│
├─► Legacy service you can't change?
│     → Use OTel Collector with receivers (Prometheus, Zipkin, Jaeger)
│       + service mesh (Istio/Linkerd) for L7 metrics + traces
│
└─► Custom protocol / non-HTTP?
      → MANUAL instrumentation only; auto-instrumentation won't help
```

### Decision 6: Cloud Profiler vs. One-off Profiling

```
I have a performance problem and need to profile
│
├─► Is this a production regression after a recent deploy?
│     → CLOUD PROFILER (already running)
│       Compare: select service → "Compare" → before/after deploy time range
│
├─► Is this a one-time investigation of a known bottleneck?
│     → One-off profiling: pprof (Go), py-spy (Python), async-profiler (Java)
│       More detailed, no sampling approximation, 0% ongoing overhead
│
├─► Is the service already running with Cloud Profiler?
│     YES → USE IT; it's zero-setup and has historical context
│
├─► Has Cloud Profiler not been set up yet?
│     → Add it! ~5 lines of code; ~1-2% CPU overhead; free of charge
│       After adding: profiling data appears within ~5 minutes
│
└─► Do I need goroutine / thread count profiling?
      → Cloud Profiler (Goroutines for Go, Threads for Java)
        One-off: `curl localhost:6060/debug/pprof/goroutine?debug=2` (Go)
```

### Decision 7: Alert Condition Type

```
I need to create an alert. Which condition type?
│
├─► Alert on a metric crossing a threshold?
│     (CPU > 80%, error rate > 1%, latency p99 > 2s)
│     → METRIC THRESHOLD condition
│
├─► Alert when a metric STOPS reporting?
│     (no data = service is down / not sending metrics)
│     → METRIC ABSENCE condition (duration = how long to wait before alerting)
│
├─► Alert on a Prometheus/GMP metric with PromQL?
│     → PROMETHEUS QUERY LANGUAGE condition
│
├─► Alert when a specific log pattern occurs?
│     (error message appears, new exception type)
│     → LOG-BASED METRIC → METRIC THRESHOLD condition
│       OR Error Reporting alert (for exceptions specifically)
│
├─► Alert when an HTTP/TCP endpoint is down?
│     (public API health check, certificate expiry)
│     → UPTIME CHECK condition (multi-region probing)
│
├─► Alert when an SLO is being burned too fast?
│     (error budget is 50% burned in 1 day)
│     → SLO BURN RATE condition
│       Fast burn: 14x rate, 1h window (pages immediately)
│       Slow burn: 6x rate, 6h window (tickets for gradual degradation)
│       *** This is the recommended alerting strategy for SRE teams ***
│
└─► Alert on forecasted future breach (predictive)?
      → FORECASTED THRESHOLD condition (ML-based)
        Good for capacity planning (disk full in 4 hours)
```

### Summary: Observability Stack Decision Matrix

| Scenario | Recommended Stack |
|---|---|
| Single GKE service, greenfield | GMP + OTel Collector + Cloud Trace + Cloud Profiler + Error Reporting |
| Legacy app, no code changes allowed | GMP (PodMonitoring) + service mesh tracing + Cloud Logging (stdout) |
| Serverless (Cloud Run / Cloud Functions) | Built-in metrics (free) + Cloud Logging + OTel SDK → Cloud Trace |
| Multi-cloud / hybrid | OTel Collector with multi-backend exporters + Cloud Monitoring + Cloud Logging |
| High-cardinality metrics (> 10K series) | GMP (Prometheus) — do NOT use Cloud Monitoring custom metrics |
| Long-term compliance log retention | Custom log bucket (Log Analytics enabled) + BigQuery sink |
| Real-time security event streaming | Log sink → Pub/Sub → SIEM (Chronicle / Splunk / Elastic) |
| Performance regression after deploy | Cloud Profiler (compare before/after) + Cloud Trace (p99 latency) |
| Exception triage | Error Reporting (grouping + stack trace) + Cloud Logging (context) |
| SRE / SLO-based operations | Cloud Monitoring SLOs + burn rate alerts + Error Budget dashboard |
