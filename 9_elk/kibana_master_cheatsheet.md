# Kibana Master Reference Cheatsheet
> Senior Engineer Reference — Kibana 7.x / 8.x | Elastic Stack | Last updated: 2024

---

## Quick Reference Card — Top 25 Operations

```bash
# --- Kibana Health & Status ---
GET  /api/status                                        # Kibana health
GET  /api/task_manager/_health                          # Background task health

# --- Data Views ---
GET  /api/data_views                                    # List data views
POST /api/data_views/data_view                          # Create data view
DELETE /api/data_views/data_view/{id}                   # Delete data view

# --- Saved Objects ---
POST /api/saved_objects/_export                         # Export objects (NDJSON)
POST /api/saved_objects/_import                         # Import objects
GET  /api/saved_objects/_find?type=dashboard            # Find dashboards

# --- Alerting ---
POST /api/alerting/rule                                 # Create rule
GET  /api/alerting/rules/_find                          # List rules
POST /api/alerting/rule/{id}/_enable                    # Enable rule
POST /api/alerting/rule/{id}/_disable                   # Disable rule

# --- Reporting ---
POST /api/reporting/generate/printablePdf               # Generate PDF report
POST /api/reporting/generate/csv_searchsource           # Generate CSV
GET  /api/reporting/jobs/download/{job_id}              # Download report

# --- Spaces ---
POST /api/spaces/space                                  # Create space
GET  /api/spaces/space                                  # List spaces
POST /api/spaces/_copy_saved_objects                    # Copy objects between spaces

# --- Cases ---
POST /api/cases                                         # Create case
GET  /api/cases                                         # List cases

# --- Required Header for all mutating API calls ---
# kbn-xsrf: true

# --- Common Kibana URLs ---
/app/discover                                           # Discover
/app/dashboards                                         # Dashboards
/app/lens                                               # Lens editor
/app/management/kibana/dataViews                        # Data views
/app/management/data/index_management                   # Index management
/app/management/data/index_lifecycle_management         # ILM policies
/app/dev_tools                                          # Dev Tools Console
/app/monitoring                                         # Stack Monitoring
/app/fleet                                              # Fleet
/app/ml                                                 # Machine Learning
/app/apm                                                # APM
/app/security                                           # Elastic Security
```

---

<!-- section: architecture -->
## 1. Kibana Overview & Architecture

### Role in the Elastic Stack

Kibana is the visualization and management layer of the Elastic Stack. It serves as:
- **Analytics UI** — Discover, Lens, Dashboards, Canvas, Maps
- **Observability** — APM, Logs, Metrics, Uptime, Synthetics, SLOs
- **Security** — SIEM, CSPM, Endpoint Security, detection rules
- **ML platform** — Anomaly detection, DFA, NLP inference
- **Operations** — Index management, ILM, Snapshot/Restore, Stack Monitoring
- **Agent management** — Fleet, Elastic Agent, integrations

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Browser / Client                           │
│        (React SPA — Kibana UI loaded from Kibana server via HTTP)       │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │ HTTPS :5601
┌────────────────────────────────▼────────────────────────────────────────┐
│                           Kibana Server (Node.js)                       │
│                                                                         │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐                 │
│  │  HTTP Router │  │  Plugin System│  │  Task Manager│                 │
│  │  (Hapi.js)   │  │  (core + xpack│  │  (alerting,  │                 │
│  └──────────────┘  │   plugins)    │  │   reporting, │                 │
│                    └───────────────┘  │   fleet...)  │                 │
│  ┌──────────────────────────────────┐ └──────────────┘                 │
│  │  Saved Objects Service           │                                   │
│  │  (reads/writes .kibana* index)   │                                   │
│  └──────────────────────────────────┘                                   │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │ HTTPS :9200
┌────────────────────────────────▼────────────────────────────────────────┐
│                          Elasticsearch Cluster                          │
│                                                                         │
│  .kibana_*          — saved objects (dashboards, data views, etc.)      │
│  .kibana_task_manager_* — task manager state                            │
│  .kibana_security_*  — security saved objects                           │
│  logs-*, metrics-*, traces-* — your data indices                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Kibana Spaces

Each **Space** is an isolated workspace with its own:
- Saved objects (dashboards, data views, rules, etc.)
- Feature visibility controls
- RBAC assignments

```
Kibana instance
├── space: default          ← URL: /app/discover
├── space: team-ops         ← URL: /s/team-ops/app/discover
├── space: security-team    ← URL: /s/security-team/app/security
└── space: finance-readonly ← URL: /s/finance-readonly/app/dashboards
```

---

<!-- section: installation -->
## 2. Installation & Configuration

### Install Methods

```bash
# DEB
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.12.0-amd64.deb
sudo dpkg -i kibana-8.12.0-amd64.deb
sudo systemctl enable --now kibana

# RPM
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.12.0-x86_64.rpm
sudo rpm -ivh kibana-8.12.0-x86_64.rpm
sudo systemctl enable --now kibana

# Docker
docker run --name kibana --net elastic -p 5601:5601 \
  -e ELASTICSEARCH_HOSTS=https://es01:9200 \
  -e ELASTICSEARCH_SERVICEACCOUNTTOKEN=<token> \
  docker.elastic.co/kibana/kibana:8.12.0
```

### Docker Compose (Elasticsearch + Kibana)

```yaml
version: "3.8"
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    environment:
      - node.name=es01
      - cluster.name=local-dev
      - discovery.type=single-node
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false
      - xpack.license.self_generated.type=trial
    ports: ["9200:9200"]
    volumes:
      - esdata:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=${KIBANA_SYSTEM_PASSWORD}
      - xpack.security.encryptionKey=${KIBANA_ENCRYPTION_KEY}
      - xpack.encryptedSavedObjects.encryptionKey=${SAVED_OBJECTS_KEY}
      - xpack.reporting.encryptionKey=${REPORTING_KEY}
    ports: ["5601:5601"]
    depends_on: [elasticsearch]

volumes:
  esdata:
```

### kibana.yml Full Reference

```yaml
# ─── Server ───────────────────────────────────────────────────────────
server.host: "0.0.0.0"
server.port: 5601
server.name: "my-kibana"
server.basePath: "/kibana"             # If behind reverse proxy with sub-path
server.publicBaseUrl: "https://kibana.example.com/kibana"
server.maxPayload: 1048576             # Max request payload bytes

# ─── Elasticsearch Connection ─────────────────────────────────────────
elasticsearch.hosts: ["https://es-node-1:9200", "https://es-node-2:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "changeme"
# OR use service account token (8.x preferred)
elasticsearch.serviceAccountToken: "AAEAAWVsYXN0aWMva2liYW5hL..."

# ─── Elasticsearch SSL ────────────────────────────────────────────────
elasticsearch.ssl.verificationMode: "certificate"   # full | certificate | none
elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/ca.crt"]
elasticsearch.ssl.certificate: "/etc/kibana/kibana.crt"
elasticsearch.ssl.key: "/etc/kibana/kibana.key"

# ─── Kibana Server TLS ────────────────────────────────────────────────
server.ssl.enabled: true
server.ssl.certificate: "/etc/kibana/server.crt"
server.ssl.key: "/etc/kibana/server.key"
server.ssl.certificateAuthorities: ["/etc/kibana/ca.crt"]

# ─── Encryption Keys (must be 32+ character random strings) ──────────
xpack.security.encryptionKey: "a-32-char-random-string-here!!"
xpack.encryptedSavedObjects.encryptionKey: "another-32-char-random-string!!"
xpack.reporting.encryptionKey: "yet-another-32-char-random-str!!"

# ─── Security & Authentication ────────────────────────────────────────
xpack.security.enabled: true
xpack.security.session.idleTimeout: "1h"
xpack.security.session.lifespan: "24h"
xpack.security.audit.enabled: true
xpack.security.audit.appender.type: rolling-file
xpack.security.audit.appender.fileName: /var/log/kibana/kibana_audit.log

# ─── Authentication Providers ─────────────────────────────────────────
xpack.security.authc.providers:
  basic.basic1:
    order: 0
  saml.saml1:
    order: 1
    realm: "saml-realm-name"
    maxRedirectURLSize: "2kb"
  oidc.oidc1:
    order: 2
    realm: "oidc-realm-name"
  anonymous.anonymous1:
    order: 3
    credentials:
      username: "anonymous_service_account"
      password: "changeme"

# ─── Logging ─────────────────────────────────────────────────────────
logging:
  root:
    level: info
  appenders:
    file:
      type: rolling-file
      fileName: /var/log/kibana/kibana.log
      policy:
        type: size-limit
        size: 100mb
      strategy:
        type: numeric
        max: 7
      layout:
        type: json
  loggers:
    - name: plugins.securitySolution
      level: warn

# ─── Fleet ───────────────────────────────────────────────────────────
xpack.fleet.enabled: true
xpack.fleet.packages:
  - name: fleet_server
    version: latest
xpack.fleet.outputs:
  - name: default
    type: elasticsearch
    hosts: ["https://es-node-1:9200"]
    is_default: true

# ─── APM ─────────────────────────────────────────────────────────────
xpack.apm.serviceMapEnabled: true

# ─── Reporting ───────────────────────────────────────────────────────
xpack.reporting.queue.timeout: 120000
xpack.reporting.kibanaServer.hostname: "kibana.example.com"
xpack.reporting.kibanaServer.port: 5601
xpack.reporting.kibanaServer.protocol: "https"
xpack.reporting.capture.browser.chromium.disableSandbox: false
xpack.reporting.capture.maxAttempts: 3

# ─── Misc ─────────────────────────────────────────────────────────────
telemetry.enabled: false
xpack.ml.enabled: true
savedObjects.maxImportPayloadBytes: 26214400   # 25 MB
```

### 8.x Enrollment Token Flow

```bash
# On Elasticsearch node — generate enrollment token for Kibana
./bin/elasticsearch-create-enrollment-token -s kibana

# Paste token in Kibana's interactive setup screen at http://localhost:5601
# Kibana auto-configures elasticsearch.hosts, TLS, and service account token

# Verify enrollment
GET /api/status
```

### Nginx Reverse Proxy

```nginx
server {
    listen 443 ssl;
    server_name kibana.example.com;

    ssl_certificate     /etc/nginx/ssl/kibana.crt;
    ssl_certificate_key /etc/nginx/ssl/kibana.key;

    location /kibana/ {
        proxy_pass          http://localhost:5601/kibana/;
        proxy_http_version  1.1;
        proxy_set_header    Upgrade $http_upgrade;
        proxy_set_header    Connection 'upgrade';
        proxy_set_header    Host $host;
        proxy_set_header    X-Real-IP $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto $scheme;
        proxy_read_timeout  300s;
        proxy_send_timeout  300s;
    }
}
```

---

<!-- section: data-views -->
## 3. Data Views (Index Patterns)

A **Data View** tells Kibana which Elasticsearch indices to query, which field is the time field, and how to format/compute fields.

### Create via API

```json
POST /api/data_views/data_view
{
  "data_view": {
    "title": "logs-nginx-*",
    "name": "Nginx Logs",
    "timeFieldName": "@timestamp",
    "namespaces": ["default"]
  }
}

// With runtime field
POST /api/data_views/data_view
{
  "data_view": {
    "title": "orders-*",
    "timeFieldName": "@timestamp",
    "runtimeFieldMap": {
      "revenue": {
        "type": "double",
        "script": { "source": "emit(doc['price'].value * doc['quantity'].value)" }
      }
    }
  }
}

// List all data views
GET /api/data_views

// Get specific data view
GET /api/data_views/data_view/{id}

// Update data view (add field format)
POST /api/data_views/data_view/{id}/fields
{
  "fields": {
    "bytes": {
      "format": {
        "id": "bytes",
        "params": { "pattern": "0,0.[000]b" }
      }
    },
    "url": {
      "format": {
        "id": "url",
        "params": { "urlTemplate": "https://example.com/path/{{value}}", "labelTemplate": "{{value}}" }
      }
    }
  }
}

// Delete data view
DELETE /api/data_views/data_view/{id}
```

### Field Formatters

| Formatter | Use case |
|-----------|----------|
| `bytes` | File sizes, bandwidth (1024 → "1 KB") |
| `url` | Clickable links from field values |
| `color` | Color-code values by range/value |
| `date` | Custom date format (moment.js patterns) |
| `duration` | Milliseconds → human duration |
| `percentage` | Decimal → percentage (0.95 → 95%) |
| `string` | Transform/truncate string values |
| `truncate` | Limit display length |
| `ip` | IP address rendering |

### Runtime Fields in Data Views

```json
POST /api/data_views/data_view/{id}/runtime_fields
{
  "name": "day_of_week",
  "runtimeField": {
    "type": "keyword",
    "script": {
      "source": "emit(doc['@timestamp'].value.dayOfWeekEnum.getDisplayName(TextStyle.FULL, Locale.ROOT))"
    }
  }
}
```

> **Note:** Runtime fields compute at query time. Use indexed fields for high-cardinality, frequently-queried values. Use runtime fields for prototyping or rarely-needed derived values.

---

<!-- section: discover -->
## 4. Discover

### Layout Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  [Data view selector] [KQL query bar]          [Time picker]    │
│──────────────────────────────────────────────────────────────── │
│  [Active filters]                                               │
│  ┌───────────┐  ┌─────────────────────────────────────────────┐ │
│  │ Fields    │  │  Histogram (document count over time)       │ │
│  │ sidebar   │  ├─────────────────────────────────────────────┤ │
│  │           │  │  Document table                             │ │
│  │ Available │  │  ┌─────┬────────┬──────────────────────┐   │ │
│  │ fields    │  │  │  ▶  │@timest.│ message              │   │ │
│  │           │  │  ├─────┼────────┼──────────────────────┤   │ │
│  │ Selected  │  │  │  ▶  │ ...    │ ...                  │   │ │
│  │ fields    │  │  └─────┴────────┴──────────────────────┘   │ │
│  └───────────┘  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Key Operations

```
Adding columns:        Click field name in sidebar → "Add as column"
Remove column:         Click column header → Remove column
Sort:                  Click column header to toggle asc/desc
Expand document:       Click ▶ arrow to see JSON/table view
Filter from value:     Click field value in expanded doc → "+" or "-" filter
Save search:           Top-right → Save → name your search
Share search:          Top-right → Share → copy URL or embed
Export CSV:            Top-right → Share → CSV Reports
Switch to ES|QL:       Toggle near query bar (8.11+)
```

### Saving and Reusing Searches

```json
// Saved searches can be embedded in dashboards as panels
// They appear in the "Add from library" panel picker

// Save via API (creates a search saved object)
POST /api/saved_objects/search
{
  "attributes": {
    "title": "Production Errors Last 24h",
    "description": "Error logs from all production services",
    "kibanaSavedObjectMeta": {
      "searchSourceJSON": "{\"index\":\"{data-view-id}\",\"query\":{\"query\":\"level: error\",\"language\":\"kuery\"},\"filter\":[]}"
    },
    "columns": ["@timestamp", "service.name", "message", "http.response.status_code"],
    "sort": [["@timestamp", "desc"]]
  }
}
```

---

<!-- section: kql -->
## 5. KQL — Kibana Query Language

KQL is a simplified query syntax used in Kibana's search bar. It is converted to Elasticsearch Query DSL automatically.

### Syntax Reference

```
# Field:value (case-insensitive for text fields)
http.response.status_code: 404
service.name: "payment-service"

# Wildcard (* matches any sequence of characters)
host.name: prod-*
message: *connection refused*
url.path: /api/v*/orders

# Field exists check
host.name: *
NOT error.message: *

# Range operators
http.response.bytes > 1000
response_time >= 100 and response_time <= 500
@timestamp > "2024-01-01" and @timestamp < "2024-02-01"

# Boolean operators (AND, OR, NOT — case-insensitive)
status: "active" AND type: "order"
level: "error" OR level: "critical"
NOT http.request.method: "OPTIONS"

# Grouping with parentheses
(status: "active" OR status: "pending") AND NOT deleted: true
(level: "error" OR level: "fatal") AND service.name: payment-*

# Nested fields
user.address.city: "Mumbai"
kubernetes.pod.labels.app: "frontend"

# Phrase match (exact word order)
message: "connection refused by server"

# Numbers
http.response.status_code: 200
process.pid: 1234

# Special character escaping
message: "error\: timeout"     # escape colon
filename: "logs\(2024\).txt"   # escape parentheses
```

### KQL vs Query DSL Equivalents

| KQL | Elasticsearch DSL |
|-----|-------------------|
| `status: "active"` | `{ "term": { "status": "active" } }` |
| `status: active OR pending` | `{ "terms": { "status": ["active","pending"] } }` |
| `title: "quick fox"` | `{ "match": { "title": "quick fox" } }` |
| `title: "quick fox"` (phrase) | `{ "match_phrase": { "title": "quick fox" } }` |
| `bytes > 1000` | `{ "range": { "bytes": { "gt": 1000 } } }` |
| `host: prod-*` | `{ "wildcard": { "host": "prod-*" } }` |
| `host: *` | `{ "exists": { "field": "host" } }` |
| `NOT status: deleted` | `{ "bool": { "must_not": [{"term":{"status":"deleted"}}] } }` |
| `a AND b` | `{ "bool": { "filter": [a, b] } }` |
| `a OR b` | `{ "bool": { "should": [a, b], "minimum_should_match": 1 } }` |

> **Note:** KQL always runs in filter context (no scoring). For relevance scoring, use Lucene syntax mode or build queries in Dev Tools.

### KQL Limitations

- No function_score, no boosting, no fuzzy with custom parameters
- No scripted queries
- No cross-field matching (use `multi_match` via Dev Tools)
- Wildcard at the start (`*foo`) is slow — avoid in high-traffic dashboards
- No aggregation control — use Lens/TSVB for that

---

<!-- section: lens -->
## 6. Lens

Lens is Kibana's primary drag-and-drop visualization builder. It automatically generates optimal chart types and ES queries.

### Chart Types Available in Lens

| Chart Type | Best For |
|------------|----------|
| Bar (vertical/horizontal, stacked/grouped) | Comparing categories over time or across dimensions |
| Line | Trends over time, multiple series |
| Area (stacked/percentage) | Part-to-whole over time |
| Pie / Donut | Proportions across a small set of categories |
| Treemap | Hierarchical proportions |
| Heatmap | Correlation / intensity across two dimensions |
| Gauge | Single KPI value with thresholds |
| Goal | Progress toward a target |
| Metric | Single number with trend indicator |
| Table | Multi-column tabular data |
| Mosaic | Categorical correlation |
| Waffle | Part-to-whole with visual grid |
| Tag cloud | Term frequency visualization |

### Lens Formula Functions

```
# Basic aggregations
count()                          # document count
count(kql='level: "error"')      # conditional count
sum(bytes)                       # sum of field
average(response_time)           # mean
min(price)
max(price)
median(response_time)
percentile(response_time, percentile=95)   # p95
percentile_rank(response_time, value=500)  # % below 500ms
last_value(status)               # last value in time bucket
unique_count(user_id)            # cardinality (approximate)
std_deviation(response_time)

# Time-based calculations
differences(count())             # delta from previous bucket
moving_average(sum(sales), window=7)         # 7-bucket rolling avg
cumulative_sum(sum(revenue))                 # running total
counter_rate(max(requests_total))            # rate for monotonic counters

# Combining metrics in formulas
sum(bytes) / count()                                    # avg bytes
(count(kql='level:"error"') / count()) * 100            # error rate %
sum(revenue) / sum(cost) - 1                           # margin %
clamp(average(score), 0, 100)                          # clamp to range
```

### Time Shift (Period Comparison)

```
# In a formula or metric dimension, add time shift:
count() [time shift: 1 week ago]
sum(sales) [time shift: 1 month ago]

# Combine in formula for % change:
(count() - count(shift='1w')) / count(shift='1w') * 100
```

### Inspecting the Underlying ES Query

In any Lens panel → click the **Inspect** (ℹ️) icon → **Request** tab → see the full Elasticsearch query DSL that Lens generated.

### Lens vs TSVB vs Vega — When to Use Which

| Capability | Lens | TSVB | Vega |
|------------|------|------|------|
| Drag-and-drop | ✅ | ❌ | ❌ |
| Formulas / custom math | ✅ | Partial (bucket script) | ✅ |
| Time series focus | Good | **Best** | ✅ |
| Multiple y-axes | ✅ | ✅ | ✅ |
| Annotations | ✅ | ✅ | ✅ |
| Markdown panel | ❌ | ✅ | ❌ |
| Custom layouts | ❌ | ❌ | ✅ |
| Arbitrary chart types | ❌ | ❌ | ✅ |
| Dashboard controls filter | ✅ | Partial | Partial |
| ES|QL data layer | ✅ (8.11+) | ❌ | ✅ |
| Best for | **Most charts** | Time series + metrics panels | Custom/complex charts |

---

<!-- section: visualizations -->
## 7. Visualizations — TSVB & Vega

### TSVB (Time Series Visual Builder)

TSVB is optimized for time-series data with panel types suited to operational dashboards.

**Panel Types:**

| Panel | Use |
|-------|-----|
| Time Series | Multi-series line/bar over time with annotations |
| Metric | Single large number with trend sparkline |
| Top N | Ranked horizontal bars |
| Gauge | Dial gauge for single metric |
| Table | Tabular breakdown with metrics per row |
| Markdown | Free-text panel with Mustache template variables |

```
# TSVB Aggregation pipeline example (Top N by error rate)
Panel type: Top N
Series:
  Agg: Count
  Filter: level: "error"
Group by: service.name
Order by: Count DESC
```

**TSVB Markdown with Mustache:**
```markdown
## Service Status
Error rate: **{{last_value}}%**
{{#if (gt last_value 5)}}🔴 HIGH{{else}}🟢 OK{{/if}}
```

**Annotations in TSVB:**
- Add an annotation layer pointing to a separate index (e.g., deployments index)
- Use `event_time_field` to mark deployment times on your time-series charts

### Vega & Vega-Lite

Use Vega for charts that Lens/TSVB cannot produce: connected scatter plots, network graphs, custom histograms, multi-layered charts with custom interaction.

**Minimal Vega-Lite with Elasticsearch datasource:**

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {
    "url": {
      "%context%": true,
      "%timefield%": "@timestamp",
      "index": "logs-*",
      "body": {
        "aggs": {
          "time_buckets": {
            "date_histogram": {
              "field": "@timestamp",
              "fixed_interval": "1h",
              "extended_bounds": {
                "min": { "%timefilter%": "min" },
                "max": { "%timefilter%": "max" }
              }
            },
            "aggs": {
              "error_count": {
                "filter": { "term": { "level": "error" } }
              }
            }
          }
        },
        "size": 0
      }
    },
    "format": {
      "property": "aggregations.time_buckets.buckets"
    }
  },
  "transform": [
    { "calculate": "datum.key_as_string", "as": "time" },
    { "calculate": "datum.error_count.doc_count", "as": "errors" }
  ],
  "mark": "line",
  "encoding": {
    "x": { "field": "time", "type": "temporal", "title": "Time" },
    "y": { "field": "errors", "type": "quantitative", "title": "Error Count" },
    "tooltip": [
      { "field": "time", "type": "temporal" },
      { "field": "errors", "type": "quantitative" }
    ]
  }
}
```

**Key Vega Kibana directives:**

| Directive | Meaning |
|-----------|---------|
| `%context%: true` | Apply Kibana time filter + KQL filter to this query |
| `%timefield%: "@timestamp"` | Field used for time filter context |
| `%timefilter%: "min"/"max"` | Inject current time range bounds |
| `%dashboard_context-must_clause%` | Inject current dashboard filter as must clause |

---

<!-- section: dashboards -->
## 8. Dashboards

### Creating a Dashboard

1. **Dashboards** → **Create dashboard**
2. **Add panel** → Choose: Lens, Maps, TSVB, Vega, Saved search, Aggregation-based, Markdown, Image, Controls
3. Resize/reorder panels by drag handles
4. Set time range and auto-refresh
5. **Save** → give name, assign to spaces/tags

### Dashboard Controls

```
# Options List Control — filter by keyword field values
Field: service.name
Allow multiple selections: true
Run now (on value change): true

# Range Slider Control — filter by numeric range
Field: http.response.bytes
Step: 1000

# Time Slider Control — animate through time ranges
```

### Drill-downs

**Dashboard drill-down** (navigate to another dashboard with filters):
- Panel → Edit → Add drill-down → Dashboard drill-down
- Configure: target dashboard, pass filters/time range

**URL drill-down** (navigate to external URL):
- Template: `https://app.example.com/orders/{{context.panel.filters[0].query.term.order_id}}`

### Sharing Dashboards

```bash
# URL types:
# Snapshot URL — current state encoded in URL (no saved object needed)
/app/dashboards#/view/DASHBOARD_ID?_g=(filters:!(),time:(from:now-1h,to:now))&_a=(...)

# Short URL (requires Kibana URL service)
POST /api/short_url
{
  "locatorId": "DASHBOARD_APP_LOCATOR",
  "params": { "dashboardId": "abc-123", "timeRange": { "from": "now-7d", "to": "now" } }
}

# Embed iframe
<iframe src="https://kibana.example.com/app/dashboards#/view/DASHBOARD_ID?embed=true&_g=..." />

# Generate PDF report of dashboard
POST /api/reporting/generate/printablePdf
{
  "jobParams": "(layout:(id:preserve_layout),objectType:dashboard,relativeUrls:!('/app/dashboards#/view/DASHBOARD_ID?_g=...'))"
}
```

---

<!-- section: esql -->
## 10. ES|QL — Elasticsearch Query Language 🔖 8.11+

ES|QL is a pipe-based query language executed inside Elasticsearch's compute engine (columnar processing).

### Basic Syntax

```sql
FROM index-pattern
| WHERE condition
| EVAL new_field = expression
| STATS metric = agg_function BY group_field
| SORT field ASC|DESC
| LIMIT n
| KEEP field1, field2
| DROP field3
| RENAME old_name AS new_name
```

### Real ES|QL Examples

```sql
-- 1. Error rate by service over last 1 hour
FROM logs-*
| WHERE @timestamp > NOW() - 1 HOUR
| STATS total = COUNT(*), errors = COUNT(*) WHERE level == "error" BY service.name
| EVAL error_rate = ROUND(errors * 100.0 / total, 2)
| SORT error_rate DESC
| LIMIT 20

-- 2. Top 10 slowest endpoints (p95 latency)
FROM traces-apm-*
| WHERE @timestamp > NOW() - 24 HOURS AND transaction.type == "request"
| STATS p95 = PERCENTILE(transaction.duration.us, 95), req_count = COUNT(*) BY transaction.name
| WHERE req_count > 100
| SORT p95 DESC
| LIMIT 10

-- 3. Parse unstructured log with GROK
FROM logs-app-*
| WHERE @timestamp > NOW() - 1 HOUR
| GROK message "%{IP:client_ip} - - \[%{HTTPDATE:timestamp}\] \"%{WORD:method} %{URIPATHPARAM:path} HTTP/%{NUMBER:http_version}\" %{NUMBER:status_code:int} %{NUMBER:bytes:int}"
| WHERE status_code >= 400
| STATS count = COUNT(*) BY status_code, path
| SORT count DESC
| LIMIT 20

-- 4. Count distinct users per hour
FROM events-*
| WHERE @timestamp > NOW() - 7 DAYS
| EVAL hour_bucket = DATE_TRUNC(1 HOURS, @timestamp)
| STATS unique_users = COUNT_DISTINCT(user.id) BY hour_bucket
| SORT hour_bucket ASC

-- 5. Multi-value field expansion
FROM products-*
| MV_EXPAND tags
| STATS count = COUNT(*) BY tags
| SORT count DESC
| LIMIT 10

-- 6. EVAL with CASE expression
FROM orders-*
| EVAL order_size = CASE(
    amount < 100, "small",
    amount < 1000, "medium",
    "large"
  )
| STATS count = COUNT(*), revenue = SUM(amount) BY order_size

-- 7. Enrich with lookup data
FROM logs-network-*
| WHERE @timestamp > NOW() - 1 HOUR
| ENRICH ip-geo-policy ON source.ip WITH country = geo.country_name, city = geo.city_name
| STATS connections = COUNT(*) BY country
| SORT connections DESC

-- 8. Rate calculation with EVAL
FROM metrics-*
| WHERE @timestamp > NOW() - 1 HOUR AND metricset.name == "cpu"
| STATS max_cpu = MAX(system.cpu.total.pct) BY host.name, @timestamp
| EVAL cpu_pct = ROUND(max_cpu * 100, 1)
| WHERE cpu_pct > 80
| SORT cpu_pct DESC

-- 9. String manipulation
FROM logs-app-*
| WHERE @timestamp > NOW() - 24 HOURS
| EVAL service = SPLIT(host.name, "-") | MV_EXPAND service
| EVAL env = CASE(service == "prod", "production", service == "stg", "staging", "dev")
| STATS count = COUNT(*) BY env

-- 10. DISSECT for structured log parsing
FROM logs-syslog-*
| DISSECT message "%{timestamp} %{hostname} %{process}[%{pid}]: %{log_message}"
| KEEP timestamp, hostname, process, log_message
| WHERE process == "sshd"
| LIMIT 100
```

### ES|QL REST API

```bash
POST /_query
{
  "query": "FROM logs-* | WHERE @timestamp > NOW() - 1 HOUR | STATS count = COUNT(*) BY service.name | SORT count DESC | LIMIT 10",
  "format": "json"
}

# With parameters
POST /_query
{
  "query": "FROM logs-* | WHERE service.name == ?service AND @timestamp > NOW() - ?hours HOUR | LIMIT 100",
  "params": ["payment-service", 24]
}
```

### ES|QL Source Commands

| Command | Description |
|---------|-------------|
| `FROM index-pattern` | Query from indices |
| `ROW field = value, ...` | Create synthetic row (useful for testing) |
| `SHOW INFO` | Show ES cluster info |
| `SHOW FUNCTIONS` | List all ES|QL functions |

---

<!-- section: alerting -->
## 11. Alerting & Rules

### Alerting Architecture

```
┌──────────────────────────────────────────────────┐
│              Kibana Task Manager                 │
│  Runs rule checks on schedule                    │
│                                                  │
│  Rule (check every 5m, lookback 15m)             │
│    ↓ query Elasticsearch                         │
│    ↓ evaluate condition                          │
│    ↓ if threshold met → create Alert             │
│    ↓ Alert → run Actions via Connector           │
│    └──► Slack / PagerDuty / Email / Webhook      │
└──────────────────────────────────────────────────┘
```

### Create a Rule via API

```json
POST /api/alerting/rule
{
  "name": "High Error Rate - Payment Service",
  "rule_type_id": ".es-query",
  "consumer": "alerts",
  "schedule": { "interval": "5m" },
  "params": {
    "index": ["logs-*"],
    "timeField": "@timestamp",
    "timeWindowSize": 15,
    "timeWindowUnit": "m",
    "esQuery": "{\"query\":{\"bool\":{\"must\":[{\"match\":{\"level\":\"error\"}},{\"match\":{\"service.name\":\"payment\"}}]}}}",
    "thresholdComparator": ">",
    "threshold": [100],
    "aggType": "count"
  },
  "actions": [
    {
      "id": "slack-connector-id",
      "group": "threshold met",
      "params": {
        "message": "🚨 High error rate on payment service: {{context.value}} errors in last 15m"
      }
    }
  ],
  "notify_when": "onThrottleInterval",
  "throttle": "30m"
}
```

### Create a Connector

```json
POST /api/actions/connector
{
  "name": "Slack - Ops Alerts",
  "connector_type_id": ".slack",
  "config": {},
  "secrets": {
    "webhookUrl": "https://hooks.slack.com/services/T000/B000/xxxx"
  }
}

// PagerDuty connector
POST /api/actions/connector
{
  "name": "PagerDuty - Critical",
  "connector_type_id": ".pagerduty",
  "config": {
    "apiUrl": "https://events.pagerduty.com"
  },
  "secrets": {
    "routingKey": "your-pagerduty-routing-key"
  }
}

// Generic webhook connector
POST /api/actions/connector
{
  "name": "Custom Webhook",
  "connector_type_id": ".webhook",
  "config": {
    "url": "https://api.example.com/alerts",
    "method": "POST",
    "headers": { "Content-Type": "application/json", "Authorization": "Bearer token" }
  },
  "secrets": {}
}
```

### Rule Management

```bash
GET  /api/alerting/rules/_find?per_page=50&sort_field=name&sort_order=asc
GET  /api/alerting/rule/{id}
PUT  /api/alerting/rule/{id}                    # Full update
POST /api/alerting/rule/{id}/_enable
POST /api/alerting/rule/{id}/_disable
POST /api/alerting/rule/{id}/_mute_all          # Mute all alerts from this rule
POST /api/alerting/rule/{id}/_unmute_all
DELETE /api/alerting/rule/{id}
GET  /api/alerting/rule/{id}/state              # Current alert states
GET  /api/actions/connectors                    # List connectors
GET  /api/actions/connector_types               # List available connector types
```

### Alert Notification Variables

```
# Available in action message templates (Mustache)
{{alertName}}          — rule name
{{alertId}}            — rule ID
{{spaceId}}            — Kibana space
{{tags}}               — rule tags
{{date}}               — alert creation timestamp
{{context.message}}    — context set by rule type
{{context.value}}      — metric value that triggered
{{context.conditions}} — condition description
{{params.index}}       — rule params
{{rule.id}}
{{rule.name}}
{{rule.spaceId}}
```

### Maintenance Windows 🔖 8.8+

```json
POST /api/maintenance_window
{
  "title": "Weekly Deployment Window",
  "duration": 3600000,
  "enabled": true,
  "rrule": {
    "freq": 2,
    "interval": 1,
    "byhour": [22],
    "byminute": [0],
    "bysecond": [0],
    "tzid": "Asia/Kolkata"
  }
}
```

---

<!-- section: ml -->
## 12. Machine Learning (X-Pack ML) 💼 Licensed

### Anomaly Detection Job Types

| Type | Use case |
|------|----------|
| Single metric | One metric, no split (e.g., total request count) |
| Multi-metric | Multiple metrics in one job (CPU + memory + network) |
| Population | Detect outliers relative to a population (e.g., users with unusual login patterns) |
| Advanced | Full control over detectors, custom splits |
| Categorization | Group unstructured log messages into categories, then detect anomalies in counts per category |

### Create Anomaly Detection Job via API

```json
PUT /_ml/anomaly_detectors/response-time-anomaly
{
  "description": "Detect anomalies in API response time",
  "analysis_config": {
    "bucket_span": "15m",
    "detectors": [
      {
        "function": "high_mean",
        "field_name": "http.response.duration",
        "by_field_name": "service.name",
        "detector_description": "High mean response time per service"
      },
      {
        "function": "high_count",
        "over_field_name": "http.response.status_code",
        "detector_description": "High count of status codes"
      }
    ],
    "influencers": ["service.name", "host.name"]
  },
  "data_description": {
    "time_field": "@timestamp",
    "time_format": "epoch_ms"
  },
  "model_plot_config": { "enabled": false },
  "analysis_limits": { "model_memory_limit": "256mb" }
}

// Create datafeed
PUT /_ml/datafeeds/datafeed-response-time-anomaly
{
  "job_id": "response-time-anomaly",
  "indices": ["traces-apm-*"],
  "query": { "bool": { "must": [{ "term": { "transaction.type": "request" } }] } },
  "scroll_size": 1000,
  "frequency": "150s",
  "query_delay": "60s"
}

// Open job + start datafeed
POST /_ml/anomaly_detectors/response-time-anomaly/_open
POST /_ml/datafeeds/datafeed-response-time-anomaly/_start

// Get anomaly results
GET /_ml/anomaly_detectors/response-time-anomaly/results/records
{
  "record_score": 50,
  "sort": "record_score",
  "descending": true,
  "start": "now-7d",
  "end": "now"
}
```

### Anomaly Score Interpretation

| Score Range | Severity |
|-------------|----------|
| 0–25 | Low (informational) |
| 25–50 | Warning |
| 50–75 | Minor |
| 75–100 | Critical |

### Trained Models / NLP

```bash
# Deploy a pre-trained model (Kibana ML UI → Trained Models → Deploy)
# Or via API:
POST /_ml/trained_models/.elser_model_2/deployment/_start
{
  "number_of_allocations": 1,
  "threads_per_allocation": 1,
  "queue_capacity": 1024
}

# Test inference
POST /_ml/trained_models/.elser_model_2/_infer
{
  "docs": [{ "text_field": "What is the best approach for log analytics?" }]
}

# Get model stats
GET /_ml/trained_models/.elser_model_2/_stats
```

---

<!-- section: apm -->
## 13. APM (Application Performance Monitoring)

### APM Data Flow

```
Your Application
  │ (APM agent SDK)
  ▼
APM Server (or Fleet-managed APM integration)
  │
  ├──► traces-apm-* (transactions, spans)
  ├──► logs-apm-*   (APM-correlated logs)
  ├──► metrics-apm-* (service metrics)
  └──► .apm-*       (service map, settings)
  │
  ▼
Kibana APM UI
```

### Key APM Views

```
Services inventory     → list of all detected services, health, latency, throughput, error rate
Service detail         → transactions, errors, metrics, logs, infrastructure for one service
Transaction list       → ranked by latency/throughput
Transaction detail     → waterfall trace with spans, labels, metadata
Error groups           → grouped by error.grouping_key, with rate and sample
Service map            → dependency graph between services + external dependencies
Correlations           → find fields that correlate with high latency or high error rate
Anomaly detection      → ML-based anomaly detection on latency and error rate
SLOs                   → Service Level Objective tracking
```

### APM API Examples

```bash
# List services
GET /api/apm/services?start=2024-01-01T00:00:00Z&end=2024-01-02T00:00:00Z&environment=production

# Service transactions
GET /api/apm/services/payment-service/transactions/groups/main_statistics?
  start=...&end=...&transactionType=request&environment=production

# Transaction samples
GET /api/apm/services/payment-service/transactions/traces/samples?
  transactionId=abc123&traceId=xyz789

# Error groups
GET /api/apm/services/payment-service/errors/groups/main_statistics?start=...&end=...
```

### APM Agent Configuration (Central Config)

```json
// Set agent config centrally via Kibana APM → Settings → Agent Configuration
// Applied dynamically without agent restart
{
  "service": { "name": "payment-service", "environment": "production" },
  "settings": {
    "transaction_sample_rate": "0.1",
    "log_level": "warn",
    "capture_body": "off",
    "sanitize_field_names": "password,secret,token"
  }
}
```

---

<!-- section: observability -->
## 14. Observability — Logs, Metrics, Uptime, Synthetics

### Logs Explorer 🔖 8.9+

Logs Explorer replaces the legacy Logs UI. It is built on Discover with enhanced log-specific features:
- Log level coloring (debug, info, warn, error, critical)
- Smart column display for log-specific fields
- Field breakdown by category
- Log rate analysis (spike detection with ML)
- Context and surrounding logs

### Infrastructure UI

```
Inventory views:
  Hosts       → CPU, memory, disk, network per host
  Kubernetes Pods → pod metrics, grouped by namespace/node
  Docker Containers → container resource usage
  AWS EC2/RDS/S3 → (requires AWS integration)

Key metrics per host:
  system.cpu.total.pct       → CPU utilization
  system.memory.actual.used.pct → Memory utilization
  system.network.in.bytes    → Network ingress
  system.network.out.bytes   → Network egress
  system.diskio.read.bytes   → Disk read
  system.diskio.write.bytes  → Disk write
```

### Synthetics Monitor (Browser Journey)

```typescript
// journey.ts — Playwright-based synthetic monitor
import { journey, step, expect, before } from '@elastic/synthetics';

journey('Checkout Flow', ({ page, params }) => {
  before(async () => {
    await page.goto(params.url || 'https://shop.example.com');
  });

  step('Load homepage', async () => {
    expect(page.url()).toContain('example.com');
    await page.waitForSelector('nav');
  });

  step('Add item to cart', async () => {
    await page.click('[data-test="product-1"]');
    await page.click('[data-test="add-to-cart"]');
    await page.waitForSelector('[data-test="cart-count"]');
  });

  step('Proceed to checkout', async () => {
    await page.click('[data-test="checkout-btn"]');
    expect(page.url()).toContain('/checkout');
  });
});
```

### SLOs (Service Level Objectives) 🔖 8.x+

```json
POST /api/observability/slos
{
  "name": "Payment API Availability",
  "description": "99.9% availability for payment service",
  "indicator": {
    "type": "sli.apm.transactionErrorRate",
    "params": {
      "service": "payment-service",
      "environment": "production",
      "transactionType": "request",
      "transactionName": "POST /api/payment"
    }
  },
  "timeWindow": { "duration": "30d", "type": "rolling" },
  "budgetingMethod": "occurrences",
  "objective": { "target": 0.999 }
}
```

---

<!-- section: security -->
## 15. Elastic Security (SIEM)

### Security Rule Types

| Rule Type | How it works |
|-----------|--------------|
| Custom query | KQL/EQL/ES|QL query with threshold |
| Threshold | Fire when count of matching docs exceeds N |
| Machine Learning | Uses ML anomaly score |
| EQL (Event Query Language) | Sequence and event correlation |
| Indicator match | Match events against threat intel indicators |
| New terms | Alert when a field value appears for the first time |
| ES|QL | ES|QL query with condition (8.13+) |

### EQL Example (Sequence Detection)

```eql
/* Detect process injection: cmd.exe spawned by Office app */
sequence by host.id with maxspan=10s
  [process where process.parent.name in ("WINWORD.EXE", "EXCEL.EXE", "POWERPNT.EXE")]
  [process where process.name == "cmd.exe"]

/* Detect lateral movement via SMB + credential access */
sequence by user.name with maxspan=60s
  [authentication where event.outcome == "success" and source.ip != "127.0.0.1"]
  [network where network.protocol == "smb"]
```

### Cases API

```json
// Create case
POST /api/cases
{
  "title": "Suspicious Lateral Movement - Host prod-db-01",
  "description": "## Summary\nDetected credential access followed by SMB activity",
  "severity": "high",
  "tags": ["lateral-movement", "credential-access"],
  "connector": {
    "id": "servicenow-connector-id",
    "name": "ServiceNow ITSM",
    "type": ".servicenow",
    "fields": { "urgency": "1", "impact": "1", "severity": "1" }
  },
  "settings": { "syncAlerts": true }
}

// Add comment
POST /api/cases/{case_id}/comments
{
  "type": "user",
  "comment": "Confirmed: user john.doe@example.com accessed prod-db-01 from unusual IP 10.0.1.50"
}

// Attach alerts to case
POST /api/cases/{case_id}/comments
{
  "type": "alert",
  "alertId": ["alert-id-1", "alert-id-2"],
  "index": [".siem-signals-default", ".siem-signals-default"],
  "rule": { "id": "rule-id", "name": "Suspicious SMB Activity" },
  "owner": "securitySolution"
}
```

---

<!-- section: fleet -->
## 16. Fleet & Elastic Agent

### Fleet Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         Kibana (Fleet UI)                        │
│  Agent Policies ─── Integrations ─── Output config              │
└───────────────────────────────┬──────────────────────────────────┘
                                │ manages via Fleet Server
┌───────────────────────────────▼──────────────────────────────────┐
│                          Fleet Server                            │
│  (runs as Elastic Agent with fleet-server integration)           │
└──────────┬────────────────────┬────────────────────┬────────────┘
           │ enrolls            │ enrolls            │ enrolls
┌──────────▼──┐          ┌──────▼──────┐      ┌──────▼───────┐
│ Elastic     │          │ Elastic     │      │ Elastic      │
│ Agent       │          │ Agent       │      │ Agent        │
│ (host-1)    │          │ (host-2)    │      │ (k8s DaemonS)│
└──────────┬──┘          └──────┬──────┘      └──────┬───────┘
           │                   │                     │
           └───────────────────┼─────────────────────┘
                               │ sends data
                     ┌─────────▼─────────┐
                     │   Elasticsearch   │
                     └───────────────────┘
```

### Fleet API

```json
// Create agent policy
POST /api/fleet/agent_policies
{
  "name": "Production Linux Hosts",
  "namespace": "production",
  "description": "Policy for all production Linux servers",
  "monitoring_enabled": ["logs", "metrics"],
  "inactivity_timeout": 1209600
}

// Add integration (package policy) to agent policy
POST /api/fleet/package_policies
{
  "name": "system-1",
  "namespace": "production",
  "policy_id": "agent-policy-id",
  "package": { "name": "system", "version": "1.38.0" },
  "inputs": [
    {
      "type": "system/metrics",
      "enabled": true,
      "streams": [
        {
          "enabled": true,
          "data_stream": { "type": "metrics", "dataset": "system.cpu" },
          "vars": { "period": { "value": "10s" } }
        }
      ]
    }
  ]
}

// List enrolled agents
GET /api/fleet/agents?perPage=50&showInactive=false

// Get enrollment token
GET /api/fleet/enrollment_api_keys

// Upgrade agent
POST /api/fleet/agents/{agent_id}/upgrade
{ "version": "8.12.0" }

// Unenroll agent
POST /api/fleet/agents/{agent_id}/unenroll
```

### Elastic Agent Enrollment

```bash
# Download and install Elastic Agent
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.12.0-linux-x86_64.tar.gz
tar xzvf elastic-agent-8.12.0-linux-x86_64.tar.gz
cd elastic-agent-8.12.0-linux-x86_64

# Enroll with Fleet Server (requires enrollment token from Kibana Fleet UI)
sudo ./elastic-agent install \
  --url=https://fleet-server:8220 \
  --enrollment-token=<enrollment-token> \
  --certificate-authorities=/etc/ssl/ca.crt

# Standalone mode (without Fleet)
./elastic-agent run -c /etc/elastic-agent/elastic-agent.yml
```

---

<!-- section: spaces-rbac -->
## 17. Spaces & Access Control

### Spaces API

```json
// Create a space
POST /api/spaces/space
{
  "id": "team-platform",
  "name": "Platform Engineering",
  "description": "Platform and SRE team workspace",
  "color": "#0077CC",
  "initials": "PE",
  "disabledFeatures": ["siem", "ml"]
}

// List spaces
GET /api/spaces/space

// Copy saved objects between spaces
POST /api/spaces/_copy_saved_objects
{
  "spaces": ["team-platform"],
  "objects": [
    { "type": "dashboard", "id": "infrastructure-overview" },
    { "type": "index-pattern", "id": "metrics-data-view" }
  ],
  "includeReferences": true,
  "overwrite": false,
  "createNewCopies": true
}
```

### Kibana Role Management

```json
// Create custom Kibana role
POST /api/security/role/platform-reader
{
  "elasticsearch": {
    "cluster": ["monitor"],
    "indices": [
      {
        "names": ["logs-*", "metrics-*"],
        "privileges": ["read", "view_index_metadata"]
      }
    ]
  },
  "kibana": [
    {
      "spaces": ["team-platform"],
      "base": [],
      "feature": {
        "discover": ["read"],
        "dashboard": ["read"],
        "visualize": ["read"],
        "maps": ["read"],
        "infrastructure": ["read"],
        "logs": ["read"],
        "apm": ["read"],
        "ml": ["read"]
      }
    }
  ]
}

// Assign role to user
POST /api/security/user/jane.doe
{
  "password": "initialPass123!",
  "roles": ["platform-reader"],
  "full_name": "Jane Doe",
  "email": "jane@example.com"
}
```

### Feature Privilege Levels

| Level | Can do |
|-------|--------|
| `read` | View objects (dashboards, visualizations) |
| `all` | View + create + edit + delete objects |
| `minimal_read` | Minimal view (some apps only) |
| `minimal_all` | Minimal admin |

---

<!-- section: saved-objects -->
## 18. Saved Objects

### Export & Import

```json
// Export specific objects
POST /api/saved_objects/_export
{
  "type": ["dashboard", "lens", "index-pattern"],
  "includeReferencesDeep": true,
  "excludeExportDetails": false
}

// Export by ID
POST /api/saved_objects/_export
{
  "objects": [
    { "type": "dashboard", "id": "abc-123" },
    { "type": "lens", "id": "def-456" }
  ],
  "includeReferencesDeep": true
}

// Import (with overwrite)
POST /api/saved_objects/_import?overwrite=true
Content-Type: multipart/form-data
--boundary
Content-Disposition: form-data; name="file"; filename="export.ndjson"
Content-Type: application/ndjson
<ndjson content>

// Import with createNewCopies (regenerate all IDs)
POST /api/saved_objects/_import?createNewCopies=true
```

### Saved Object CRUD

```json
// Create
POST /api/saved_objects/tag
{
  "attributes": { "name": "production", "description": "Production dashboards", "color": "#FF0000" }
}

// Get
GET /api/saved_objects/dashboard/my-dashboard-id

// Update
PUT /api/saved_objects/dashboard/my-dashboard-id
{ "attributes": { "title": "New Title" } }

// Delete
DELETE /api/saved_objects/dashboard/my-dashboard-id

// Find
GET /api/saved_objects/_find?type=dashboard&search=infrastructure&search_fields=title
GET /api/saved_objects/_find?type=lens&has_reference={"type":"index-pattern","id":"abc"}
GET /api/saved_objects/_find?type=dashboard&fields=title,description&per_page=50
```

### Saved Object Types Reference

| Type | Description |
|------|-------------|
| `dashboard` | Kibana dashboard |
| `visualization` | Legacy aggregation-based visualization |
| `lens` | Lens visualization |
| `map` | Elastic Maps visualization |
| `search` | Saved Discover search |
| `index-pattern` | Legacy index pattern (pre-8.x) |
| `data-view` | Data view (8.x+) |
| `canvas-workpad` | Canvas workpad |
| `canvas-element` | Canvas element |
| `alert` | Kibana alerting rule |
| `action` | Connector (action) |
| `tag` | Object tag |
| `config` | Kibana advanced settings |
| `url` | Short URL |
| `cases` | Security/Observability case |
| `slo` | Service Level Objective |
| `ml-job` | ML job |

---

<!-- section: api-reference -->
## 19. Kibana REST API Reference

### Authentication

```bash
# Basic auth
curl -u elastic:password https://kibana:5601/api/status

# API key (created in Kibana Security → API Keys)
curl -H "Authorization: ApiKey <base64-id:key>" https://kibana:5601/api/status

# Service account token
curl -H "Authorization: Bearer <service-account-token>" https://kibana:5601/api/status

# Required header for ALL state-mutating requests
-H "kbn-xsrf: true"
```

> **Note:** Every POST/PUT/PATCH/DELETE request to the Kibana API requires the `kbn-xsrf: true` header. Without it, you get `400 Request must contain a kbn-xsrf header`.

### Complete API Examples

```bash
# Status
curl -su elastic:password https://kibana:5601/api/status | jq '.status.overall'

# Create data view
curl -su elastic:password -X POST https://kibana:5601/api/data_views/data_view \
  -H "kbn-xsrf: true" -H "Content-Type: application/json" \
  -d '{"data_view":{"title":"logs-*","timeFieldName":"@timestamp","name":"All Logs"}}'

# Generate PDF report
curl -su elastic:password -X POST https://kibana:5601/api/reporting/generate/printablePdf \
  -H "kbn-xsrf: true" -H "Content-Type: application/json" \
  -d '{"jobParams":"(layout:(id:preserve_layout),objectType:dashboard,relativeUrls:!(\"/app/dashboards#/view/DASHBOARD_ID?_g=(time:(from:now-24h,to:now))\"))"}'

# Download report
curl -su elastic:password https://kibana:5601/api/reporting/jobs/download/{job_id} \
  -o dashboard-report.pdf

# Create short URL
curl -su elastic:password -X POST https://kibana:5601/api/short_url \
  -H "kbn-xsrf: true" -H "Content-Type: application/json" \
  -d '{"locatorId":"DASHBOARD_APP_LOCATOR","params":{"dashboardId":"abc-123","timeRange":{"from":"now-7d","to":"now"}}}'
```

### Spaces API Summary

```bash
GET    /api/spaces/space                        # List spaces
POST   /api/spaces/space                        # Create space
GET    /api/spaces/space/{id}                   # Get space
PUT    /api/spaces/space/{id}                   # Update space
DELETE /api/spaces/space/{id}                   # Delete space
POST   /api/spaces/_copy_saved_objects          # Copy objects
POST   /api/spaces/_resolve_copy_saved_objects_errors  # Resolve copy conflicts
GET    /api/spaces/_get_shareable_references    # Get shareable refs
POST   /api/spaces/_update_objects_spaces       # Move objects
```

---

<!-- section: dev-tools -->
## 21. Dev Tools

### Console Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+Enter` / `Cmd+Enter` | Execute current request |
| `Ctrl+/` | Toggle comment |
| `Ctrl+I` | Auto-indent |
| `Ctrl+Space` | Autocomplete |
| `Ctrl+Up/Down` | Navigate between requests |
| `Ctrl+Alt+L` | Collapse/expand all |

### Search Profiler

```json
// Paste into Search Profiler, select index, click Profile
{
  "query": {
    "bool": {
      "must": [{ "match": { "message": "error" } }],
      "filter": [{ "term": { "level": "error" } }]
    }
  },
  "aggs": {
    "by_service": { "terms": { "field": "service.name" } }
  }
}

// Profile output shows:
// - Per-shard timing for each query clause (create_weight, build_scorer, score)
// - Aggregation timing
// - Collector timing
```

### Grok Debugger Patterns Reference

```
COMMON PATTERNS:
%{IP:client_ip}              → IP address
%{HOSTNAME:hostname}         → Hostname
%{NUMBER:status:int}         → Integer number
%{NUMBER:bytes:float}        → Float number  
%{WORD:method}               → Single word
%{URIPATH:path}              → URI path
%{HTTPDATE:timestamp}        → Apache/Nginx date format
%{GREEDYDATA:message}        → Everything remaining
%{DATA:field}                → Non-greedy match
%{NOTSPACE:field}            → Non-whitespace chars
%{UUID:trace_id}             → UUID format

EXAMPLE — Apache Combined Log:
%{COMBINEDAPACHELOG}
expands to:
%{IPORHOST:clientip} %{HTTPDUSER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] "(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})" %{NUMBER:response} (?:%{NUMBER:bytes}|-)
```

### Painless Lab Examples

```java
// Context: score — boost documents by field value
double boost = doc['featured'].value ? 2.0 : 1.0;
return _score * boost;

// Context: filter — conditional filter
return doc['price'].value < params.max_price && doc['stock'].value > 0;

// Context: ingest processor
if (ctx.containsKey('response_time') && ctx.response_time != null) {
  ctx.response_time_category = ctx.response_time > 1000 ? 'slow' : ctx.response_time > 100 ? 'medium' : 'fast';
}

// Context: update
if (!ctx._source.containsKey('tags')) { ctx._source.tags = new ArrayList(); }
if (!ctx._source.tags.contains(params.tag)) { ctx._source.tags.add(params.tag); }

// Context: sort
return doc['priority'].value * 1000 - doc['created_at'].value.getMillis();
```

---

<!-- section: stack-management -->
## 22. Stack Management

### Advanced Settings Reference

| Setting | Default | Description |
|---------|---------|-------------|
| `dateFormat` | `MMM D, YYYY @ HH:mm:ss.SSS` | Global date display format |
| `dateFormat:tz` | `Browser` | Default timezone |
| `defaultIndex` | _(none)_ | Default data view |
| `discover:sampleSize` | `500` | Max rows shown in Discover |
| `discover:rowHeightOption` | `0` | Row height (0=compact, -1=auto) |
| `theme:darkMode` | `false` | Dark mode |
| `timepicker:timeDefaults` | `{ "from": "now-15m", "to": "now" }` | Default time range |
| `timepicker:quickRanges` | _(see UI)_ | Quick time picker presets |
| `visualization:colorMapping` | _(see UI)_ | Custom color assignments |
| `search:queryLanguage` | `kuery` | Default search language (kuery/lucene) |
| `csv:separator` | `,` | CSV export separator |
| `notifications:banner` | _(none)_ | Custom info banner text |
| `savedObjects:perPage` | `20` | Objects per page in Saved Objects UI |
| `doc_table:highlight` | `true` | Highlight search terms in Discover |
| `metrics:max_buckets` | `2000` | Max histogram buckets in TSVB |

---

<!-- section: performance-ops -->
## 23. Kibana Performance & Operations

### Task Manager Health

```bash
GET /api/task_manager/_health

# Response includes:
# status: OK / degraded / unavailable
# stats.managed_configuration: task polling intervals
# stats.workload: scheduled tasks count by type
# stats.runtime: task execution stats
# stats.capacity_estimation: recommended worker count

# Common task manager issues:
# - "degraded" → overloaded, too many tasks; increase polling interval or add Kibana nodes
# - Tasks stuck → check .kibana_task_manager-* index health in Elasticsearch
```

### Kibana Clustering (Multiple Instances)

```yaml
# All Kibana instances share the same kibana.yml settings
# Sticky sessions NOT required — stateless (session stored in Elasticsearch)
# Load balance: nginx round-robin is fine

# Same encryption keys across ALL instances (critical!)
xpack.security.encryptionKey: "same-key-on-all-kibana-instances!!"
xpack.encryptedSavedObjects.encryptionKey: "same-key-on-all-kibana-instances!!"
xpack.reporting.encryptionKey: "same-key-on-all-kibana-instances!!"
```

> **Note:** If encryption keys differ across Kibana nodes, saved alerts, encrypted saved objects, and reports will fail. Always use the same keys.

### Memory Tuning

```bash
# /etc/default/kibana or systemd override
NODE_OPTIONS="--max-old-space-size=2048"   # 2GB heap

# Or in /usr/share/kibana/config/node.options
--max-old-space-size=2048
```

### Structured JSON Logging

```yaml
logging:
  root:
    level: info
  appenders:
    json-file:
      type: rolling-file
      fileName: /var/log/kibana/kibana.json
      policy:
        type: size-limit
        size: 100mb
      layout:
        type: json
  loggers:
    - name: http.server.response
      level: debug
      appenders: [json-file]
```

### Common Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| "Kibana server is not ready yet" | ES unreachable, or migration in progress | Check ES health; wait for migration to complete |
| Task manager degraded | Too many tasks, insufficient polling workers | Add Kibana nodes; increase `xpack.task_manager.max_workers` |
| Reporting timeout | Complex dashboard, slow ES, Chromium issue | Increase `xpack.reporting.queue.timeout`; check Chromium logs |
| "Invalid encryption key" | Mismatched encryptionKey across Kibana nodes | Sync encryption keys across all Kibana instances |
| Saved object migration failure | Corrupt saved object in `.kibana` | Use Upgrade Assistant; check migration logs; manually fix corrupt objects |
| High memory (Node.js OOM) | Too many background tasks or large reporting jobs | Increase `--max-old-space-size`; reduce concurrent reports |
| Session expiry loops | Session timeout too short | Increase `xpack.security.session.idleTimeout` |

---

<!-- section: authentication -->
## 24. Kibana Authentication & SSO

### SAML Configuration

```yaml
# elasticsearch.yml — configure SAML realm
xpack.security.authc.realms.saml.saml1:
  order: 2
  idp.metadata.path: /etc/elasticsearch/saml/idp-metadata.xml
  idp.entity_id: "https://idp.example.com/sso/saml"
  sp.entity_id: "https://kibana.example.com"
  sp.acs: "https://kibana.example.com/api/security/saml/callback"
  sp.logout: "https://kibana.example.com/logout"
  attributes.principal: "nameid"
  attributes.groups: "groups"
  attributes.mail: "mail"
  attributes.name: "displayName"

# kibana.yml
xpack.security.authc.providers:
  saml.saml1:
    order: 0
    realm: saml1
    description: "Sign in with Corporate SSO"
    icon: "logoSecurity"
  basic.basic1:
    order: 1
    description: "Sign in with username/password"
```

### OIDC Configuration (Okta example)

```yaml
# elasticsearch.yml
xpack.security.authc.realms.oidc.oidc1:
  order: 2
  rp.client_id: "kibana-app"
  rp.response_type: "code"
  rp.redirect_uri: "https://kibana.example.com/api/security/oidc/callback"
  op.issuer: "https://dev-xxx.okta.com"
  op.authorization_endpoint: "https://dev-xxx.okta.com/oauth2/v1/authorize"
  op.token_endpoint: "https://dev-xxx.okta.com/oauth2/v1/token"
  op.userinfo_endpoint: "https://dev-xxx.okta.com/oauth2/v1/userinfo"
  op.jwkset_path: "https://dev-xxx.okta.com/oauth2/v1/keys"
  claims.principal: sub
  claims.groups: groups

# Add client secret to keystore:
./bin/elasticsearch-keystore add xpack.security.authc.realms.oidc.oidc1.rp.client_secret
```

### Role Mapping (SAML/LDAP Groups → Kibana Roles)

```json
PUT /_security/role_mapping/saml-ops-team
{
  "enabled": true,
  "roles": ["kibana_admin", "superuser"],
  "rules": {
    "all": [
      { "field": { "realm.name": "saml1" } },
      { "field": { "groups": "ops-team" } }
    ]
  }
}

PUT /_security/role_mapping/saml-readonly-users
{
  "enabled": true,
  "roles": ["platform-reader"],
  "rules": {
    "all": [
      { "field": { "realm.name": "saml1" } },
      { "field": { "groups": "kibana-readonly" } }
    ]
  }
}
```

---

<!-- section: upgrade -->
## 25. Kibana Upgrade & Migration

### Upgrade Checklist

```
Pre-upgrade:
□ 1. Read release notes and breaking changes for target version
□ 2. Run Upgrade Assistant (Stack Management → Upgrade Assistant)
□ 3. Fix all deprecation warnings flagged by Upgrade Assistant
□ 4. Snapshot .kibana* and .kibana_task_manager* indices in Elasticsearch
□ 5. Note current Kibana version (must upgrade to same minor as Elasticsearch)
□ 6. Test upgrade in staging environment first

Upgrade (single instance):
□ 1. Stop Kibana: systemctl stop kibana
□ 2. Install new Kibana package (DEB/RPM)
□ 3. Review kibana.yml — compare with release notes for config changes
□ 4. Start Kibana: systemctl start kibana
□ 5. Monitor logs for migration progress: journalctl -u kibana -f
□ 6. Verify at /api/status that migration completed

Upgrade (multi-node, rolling):
□ 1. Upgrade Elasticsearch cluster first (rolling)
□ 2. Stop one Kibana node
□ 3. Upgrade the package
□ 4. Start and verify it joins successfully
□ 5. Repeat for remaining nodes

Post-upgrade:
□ 1. Verify all dashboards load correctly
□ 2. Verify all alerting rules are running
□ 3. Verify Fleet agents are checking in
□ 4. Verify APM data is flowing
□ 5. Clear browser cache on all users' machines
```

### Backup .kibana Indices

```json
// Snapshot just Kibana indices before upgrade
PUT /_snapshot/my-repo/kibana-pre-upgrade-snapshot
{
  "indices": ".kibana*,.kibana_task_manager*",
  "include_global_state": false
}

// Restore if upgrade fails
POST /_snapshot/my-repo/kibana-pre-upgrade-snapshot/_restore
{
  "indices": ".kibana*,.kibana_task_manager*",
  "include_global_state": false,
  "rename_pattern": "(.+)",
  "rename_replacement": "restored_$1"
}
```

---

## Appendix A: Lens vs TSVB vs Vega vs Aggregation-Based — Full Comparison

| Feature | Lens | TSVB | Vega/Vega-Lite | Agg-Based (Legacy) |
|---------|------|------|-----------------|---------------------|
| Ease of use | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| Drag-and-drop | ✅ | ❌ | ❌ | Partial |
| Formula/custom math | ✅ Full formulas | Bucket Script only | ✅ Full custom | ❌ |
| Time series optimization | Good | ✅ Best | ✅ | Good |
| Dashboard controls (filters) | ✅ | Partial | Partial (via context) | ✅ |
| ES|QL support | ✅ 8.11+ | ❌ | ✅ | ❌ |
| Annotations | ✅ | ✅ | ✅ | ❌ |
| Reference lines/thresholds | ✅ | ✅ | ✅ | ❌ |
| Multiple y-axes | ✅ | ✅ | ✅ | ❌ |
| Markdown/HTML in panel | ❌ | ✅ | ❌ | ❌ |
| Custom chart types | ❌ | ❌ | ✅ Any | Limited |
| Map visualization | ❌ | ❌ | ✅ | ✅ (coord map) |
| Pixel-perfect control | ❌ | ❌ | ✅ | ❌ |
| External data sources | ❌ | ❌ | ✅ (any URL) | ❌ |
| Learning curve | Low | Medium | High | Low-Medium |
| Migration path | ← Target | → Migrate to Lens | Keep for custom | → Migrate to Lens |
| **Best for** | **Standard charts, KPIs, tables** | **Ops time-series dashboards** | **Custom/complex charts** | **Legacy, migrate away** |

---

## Appendix B: kibana.yml Quick Settings Reference

```yaml
# Critical settings (must configure before production)
server.publicBaseUrl: "https://kibana.example.com"
xpack.security.encryptionKey: "<32+ char random string>"
xpack.encryptedSavedObjects.encryptionKey: "<32+ char random string>"
xpack.reporting.encryptionKey: "<32+ char random string>"

# Performance
xpack.task_manager.max_workers: 10
xpack.task_manager.poll_interval: 3000
server.keepAliveTimeout: 120000
elasticsearch.requestTimeout: 30000
elasticsearch.shardTimeout: 30000

# Logging
logging.root.level: warn
logging.appenders.file.type: rolling-file
logging.appenders.file.fileName: /var/log/kibana/kibana.log

# Session management
xpack.security.session.idleTimeout: "1h"
xpack.security.session.lifespan: "8h"
xpack.security.session.cleanupInterval: "1h"

# Reporting
xpack.reporting.queue.timeout: 120000
xpack.reporting.capture.maxAttempts: 3
xpack.reporting.roles.enabled: false   # 8.x — use feature privileges instead

# Feature flags
xpack.ml.enabled: true
xpack.fleet.enabled: true
xpack.apm.enabled: true
xpack.cloudSecurityPosture.enabled: true
telemetry.enabled: false
```

---

*End of Kibana Master Reference Cheatsheet*
*Covers Kibana 7.x / 8.x — verify version-specific behavior at elastic.co/docs/kibana*
