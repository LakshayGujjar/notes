# 📊 GCP Looker — Comprehensive Cheatsheet

> **Audience:** Data analysts, data engineers, BI developers, and architects building governed analytics on GCP.
> **Last updated:** March 2026 | Covers LookML, Explores, Embedding, API, BigQuery integration, and administration.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [LookML Fundamentals](#2-lookml-fundamentals)
3. [Explores & Joins](#3-explores--joins)
4. [Derived Tables](#4-derived-tables)
5. [Dimensions, Measures & Calculations](#5-dimensions-measures--calculations)
6. [Filters, Parameters & Templated SQL](#6-filters-parameters--templated-sql)
7. [Dashboards & Looks](#7-dashboards--looks)
8. [User Attributes & Row-Level Security](#8-user-attributes--row-level-security)
9. [Caching & Performance](#9-caching--performance)
10. [Looker API](#10-looker-api)
11. [Embedding](#11-embedding)
12. [Administration & Governance](#12-administration--governance)
13. [Looker & BigQuery Integration](#13-looker--bigquery-integration)
14. [LookML Best Practices & Patterns](#14-lookml-best-practices--patterns)
15. [Pricing Summary](#15-pricing-summary)
16. [Quick Reference & Comparison Tables](#16-quick-reference--comparison-tables)

---

## 1. Overview & Key Concepts

### What is Looker?

**Looker** is Google Cloud's enterprise business intelligence and data platform built around a **governed semantic layer**. Rather than connecting BI tools directly to raw tables, Looker centralizes all business logic — metric definitions, joins, access rules — in **LookML**, a version-controlled modeling language. Every query is generated from this single source of truth.

**Core value proposition:**

| Pillar | Description |
|---|---|
| **Semantic layer** | Business logic defined once in LookML, reused everywhere |
| **Single source of truth** | `revenue` means the same thing in every dashboard |
| **Governed BI** | Access control, row-level security, and audit trails built-in |
| **Headless BI** | API-first — embed analytics in any application |
| **Version-controlled models** | LookML lives in Git — diff, review, and deploy like code |

---

### Looker vs. Looker Studio

| Dimension | Looker | Looker Studio |
|---|---|---|
| **Target user** | Enterprise BI teams, data engineers | Individuals, small teams |
| **Data modeling** | ✅ LookML semantic layer | ❌ Ad-hoc connections only |
| **Governance** | ✅ Roles, RLS, access grants | ❌ Minimal |
| **Version control** | ✅ Git-based LookML | ❌ |
| **Embedding** | ✅ Signed SSO embed, SDK | ⚠️ Limited iframe |
| **API** | ✅ Full REST API + SDK | ⚠️ Limited |
| **Cost** | Per-seat licensing | Free |
| **Best for** | Enterprise analytics, embedded BI | Quick dashboards, ad-hoc reports |

> 💡 Use **Looker** when you need governed metrics, row-level security, embedding, or a semantic layer shared across teams. Use **Looker Studio** for quick, personal dashboards connected directly to BigQuery or Sheets.

---

### Deployment Models

| Model | Hosting | Data Residency | Notes |
|---|---|---|---|
| **Looker (Google Cloud core)** | Google-managed GCP | GCP regions | GDPR/data residency compliant; newer model |
| **Looker (original)** | Customer VPC or Looker-hosted | Flexible | Legacy model; same LookML experience |

---

### Core Architecture

```
LookML Project (Git repo)
├── manifest.lkml          ← project settings, dependencies
├── my_model.model.lkml    ← connection + explore declarations
├── views/
│   ├── orders.view.lkml   ← table definition + fields
│   ├── users.view.lkml
│   └── order_items.view.lkml
└── dashboards/
    └── sales.dashboard.lookml  ← LookML dashboard definition

Runtime hierarchy:
  Project → Model → Explore → View → Field (Dimension / Measure)
```

| Component | Description |
|---|---|
| **Project** | Git repository containing all LookML files |
| **Model** | Declares database connection + which Explores exist |
| **Explore** | Query entry point — the "FROM" table + all its joins |
| **View** | Maps to a table or derived table; contains fields |
| **Dimension** | A column or expression — appears in SELECT / GROUP BY |
| **Measure** | An aggregation — appears in SELECT as agg function |
| **Look** | Saved query + single visualization |
| **Dashboard** | Collection of Looks / tiles with shared filters |

---

### The Semantic Layer

```
Raw Tables (BigQuery)                LookML Semantic Layer           BI Consumers
  orders (SQL table)   ─────────────► orders.view.lkml     ──────►  Looker Explore
  users  (SQL table)   ─────────────► users.view.lkml       ──────►  Embedded dashboards
  products (SQL table) ─────────────► products.view.lkml    ──────►  Looker API
                                                             ──────►  Google Sheets
                            ↑ Business logic lives here ↑
                  (revenue definition, joins, access control)
```

> 💡 The semantic layer means **`revenue` is defined once** in LookML. Whether accessed via dashboard, API, or embedding, the SQL is always identical — eliminating metric sprawl.

---

### Supported Databases

BigQuery, Cloud Spanner, Snowflake, Redshift, PostgreSQL, MySQL, SQL Server, Databricks, Trino/Presto, Athena, Oracle, and 50+ others via JDBC.

---

### Looker vs. Alternatives

| Tool | Modeling | Governance | Embedding | Git | Best For |
|---|---|---|---|---|---|
| **Looker** | ✅ LookML | ✅ Native | ✅ Signed SSO | ✅ | Enterprise governed BI |
| **Tableau** | ⚠️ Calculated fields | ⚠️ Tableau Server | ⚠️ Limited | ❌ | Visualization-heavy analytics |
| **Power BI** | ⚠️ DAX measures | ⚠️ RLS | ⚠️ Embed requires Premium | ❌ | Microsoft ecosystem |
| **dbt + Metabase** | ✅ dbt models | ⚠️ Basic | ❌ | ✅ (dbt) | Cost-sensitive, OSS stack |

---

## 2. LookML Fundamentals

### Project Structure

```
my_looker_project/
├── manifest.lkml                 # Project settings, constants, imports
├── models/
│   └── ecommerce.model.lkml      # Connection + explore declarations
├── views/
│   ├── orders.view.lkml          # One view per table (recommended)
│   ├── users.view.lkml
│   ├── order_items.view.lkml
│   └── _base_events.view.lkml    # Base view for extends
├── explores/
│   └── order_explore.explore.lkml # Explore-only file (optional separation)
└── dashboards/
    └── sales_overview.dashboard.lookml
```

---

### Model File

```lookml
# ecommerce.model.lkml

connection: "my_bigquery_connection"

# Include all view files
include: "/views/*.view.lkml"
include: "/views/**/*.view.lkml"   # Recursive include

# Declare explores
explore: orders {
  label: "Orders & Customers"
  description: "Analyze order transactions with customer and product data"

  join: users {
    type:         left_outer
    sql_on:       ${orders.user_id} = ${users.id} ;;
    relationship: many_to_one
  }

  join: order_items {
    type:         left_outer
    sql_on:       ${orders.id} = ${order_items.order_id} ;;
    relationship: one_to_many
  }
}
```

---

### View File: sql_table_name vs. derived_table

```lookml
# views/orders.view.lkml

view: orders {
  # Option 1: Point to an existing database table
  sql_table_name: `my_project.my_dataset.orders` ;;

  # Option 2: Define a derived table (covered in Section 4)
  # derived_table: {
  #   sql: SELECT * FROM `my_project.my_dataset.orders` WHERE status = 'complete' ;;
  # }

  # Primary key (always define this!)
  dimension: id {
    primary_key: yes
    type:        number
    sql:         ${TABLE}.order_id ;;
  }

  dimension: status {
    type:        string
    sql:         ${TABLE}.status ;;
    label:       "Order Status"
    description: "Current fulfillment status of the order"
    group_label: "Order Details"
  }

  dimension: amount {
    type:            number
    sql:             ${TABLE}.sale_price ;;
    value_format_name: usd
  }

  # Boolean dimension
  dimension: is_returned {
    type: yesno
    sql:  ${TABLE}.returned_at IS NOT NULL ;;
  }

  # Tier / bucket dimension
  dimension: order_value_tier {
    type: tier
    tiers: [10, 50, 100, 500, 1000]
    style: integer
    sql:  ${amount} ;;
  }

  # Date dimension group — creates raw, date, week, month, quarter, year
  dimension_group: created {
    type:      time
    timeframes: [raw, date, week, month, quarter, year]
    datatype:  timestamp
    sql:       ${TABLE}.created_at ;;
  }

  # Duration between two dimension_groups
  dimension: days_to_ship {
    type:        duration_day
    sql_start:   ${TABLE}.created_at ;;
    sql_end:     ${TABLE}.shipped_at ;;
  }

  # ── MEASURES ──────────────────────────────────────────────────────
  measure: count {
    type:        count
    label:       "Number of Orders"
    drill_fields: [id, status, created_date, amount]
  }

  measure: total_revenue {
    type:            sum
    sql:             ${amount} ;;
    value_format_name: usd
    label:           "Total Revenue"
  }

  measure: average_order_value {
    type:            average
    sql:             ${amount} ;;
    value_format_name: usd
  }

  measure: count_distinct_customers {
    type: count_distinct
    sql:  ${user_id} ;;
  }

  # Filtered measure — like COUNT(CASE WHEN ...)
  measure: count_returned {
    type:    count
    filters: [is_returned: "yes"]
    label:   "Returned Orders"
  }

  # Ratio / calculated measure
  measure: return_rate {
    type:            number
    sql:             ${count_returned} / NULLIF(${count}, 0) ;;
    value_format_name: percent_2
    label:           "Return Rate"
  }

  dimension: user_id {
    type:   number
    sql:    ${TABLE}.user_id ;;
    hidden: yes    # Internal join key — hide from Explore UI
  }
}
```

---

### Field Metadata Parameters

| Parameter | Applies To | Description |
|---|---|---|
| `label` | All fields | Display name in the UI |
| `description` | All fields | Tooltip shown to users |
| `hidden: yes` | All fields | Hides from field picker |
| `tags: ["tag1"]` | All fields | Searchable tags |
| `group_label` | All fields | Groups fields in picker |
| `primary_key: yes` | Dimensions | Marks PK; affects count accuracy |
| `value_format` | Number fields | Excel-style format string `"#,##0.00"` |
| `value_format_name` | Number fields | Named format: `usd`, `percent_2`, `decimal_2` |
| `drill_fields` | Measures | Fields shown when drilling |
| `required_fields` | Measures | Fields that must also be selected |
| `can_filter: no` | Dimensions | Prevent use as dashboard filter |

---

## 3. Explores & Joins

### Basic Explore with Joins

```lookml
# Full explore with all common parameters

explore: orders {
  label:       "Orders"
  description: "Order analysis across customers and products"
  tags:        ["finance", "operations"]
  hidden:      no

  # Always apply a filter (user sees it and can change it)
  always_filter: {
    filters: [orders.created_date: "last 90 days"]
  }

  # Apply filter user cannot remove
  sql_always_where: ${orders.status} != 'test' ;;

  # Row-level security using user attribute
  access_filter: {
    field: orders.region
    user_attribute: allowed_region
  }

  join: users {
    type:         left_outer
    sql_on:       ${orders.user_id} = ${users.id} ;;
    relationship: many_to_one        # Many orders → one user
    fields:       [users.name, users.email, users.country]  # Only expose these fields
  }

  join: order_items {
    type:         left_outer
    sql_on:       ${orders.id} = ${order_items.order_id} ;;
    relationship: one_to_many        # One order → many items
  }

  join: products {
    type:         left_outer
    sql_on:       ${order_items.product_id} = ${products.id} ;;
    relationship: many_to_one
    view_label:   "Product Details"  # Override display name in UI
  }
}
```

---

### Join Relationships & Fanout

| Relationship | Description | Count Behavior | Use Case |
|---|---|---|---|
| `many_to_one` | Many rows on left join to one on right | ✅ Correct counts on base | Join fact → dimension (most common) |
| `one_to_one` | Each row on left joins one on right | ✅ Correct | Join two facts with same grain |
| `one_to_many` | One row on left joins many on right | ⚠️ Fan-out! Use `count_distinct` | Join order → order_items |
| `many_to_many` | Multiple matches both sides | ❌ Severe fanout | Avoid; use bridge table |

> ⚠️ **Fan-out warning:** When joining `one_to_many`, the base table rows are duplicated. Always use `count_distinct` on the base table's PK for accurate counts, not `count`.

---

### Access Filter (Row-Level Security)

```lookml
# Restricts rows based on user's attribute value
# User with allowed_region = "EMEA" sees only EMEA orders

explore: orders {
  access_filter: {
    field:          orders.region          # LookML field to filter on
    user_attribute: allowed_region         # User attribute to compare
  }
}

# Multiple access filters = AND logic
explore: orders {
  access_filter: {
    field:          orders.region
    user_attribute: allowed_region
  }
  access_filter: {
    field:          orders.brand
    user_attribute: allowed_brand
  }
}
```

---

### Required Access Grants

```lookml
# Define an access grant (who gets it? set in Admin → Users)
access_grant: finance_team {
  user_attribute:  department
  allowed_values:  ["finance", "executive"]
}

# Lock an entire explore to only finance team
explore: orders {
  required_access_grants: [finance_team]
}

# Lock a single sensitive field
dimension: cost_price {
  type:                   number
  sql:                    ${TABLE}.cost ;;
  required_access_grants: [finance_team]
}
```

---

### Conditionally Filter

```lookml
# Apply filter ONLY when certain fields are not selected
explore: events {
  conditionally_filter: {
    filters: [events.event_date: "last 7 days"]
    unless:  [events.user_id, events.session_id]  # Don't filter when these are in query
  }
}
```

---

## 4. Derived Tables

### SQL Derived Table

```lookml
view: user_order_summary {
  derived_table: {
    sql:
      SELECT
        user_id,
        COUNT(*)          AS order_count,
        SUM(sale_price)   AS lifetime_value,
        MIN(created_at)   AS first_order_date,
        MAX(created_at)   AS last_order_date
      FROM `my_project.my_dataset.orders`
      WHERE status = 'complete'
      GROUP BY user_id
    ;;
  }

  dimension: user_id {
    primary_key: yes
    type:        number
    sql:         ${TABLE}.user_id ;;
  }

  measure: total_lifetime_value {
    type:            sum
    sql:             ${TABLE}.lifetime_value ;;
    value_format_name: usd
  }
}
```

---

### Native Derived Table (NDT)

```lookml
# Built entirely from LookML — no raw SQL
view: user_order_facts {
  derived_table: {
    explore_source: orders {
      column: user_id        { field: orders.user_id }
      column: order_count    { field: orders.count }
      column: total_revenue  { field: orders.total_revenue }
      column: first_order    { field: orders.created_date }

      filters: [orders.status: "complete"]
      bind_all_filters: no
    }
  }

  dimension: user_id {
    primary_key: yes
    type:        number
    sql:         ${TABLE}.user_id ;;
  }

  measure: avg_revenue_per_user {
    type: average
    sql:  ${TABLE}.total_revenue ;;
  }
}
```

---

### Persistent Derived Table (PDT) with Datagroup

```lookml
# Define when cache/PDTs should refresh
datagroup: daily_refresh {
  # Trigger when BigQuery daily ETL job completes
  sql_trigger: SELECT MAX(updated_at) FROM `proj.ds.etl_log` WHERE job = 'daily' ;;
  max_cache_age: "24 hours"
  label: "Daily ETL Refresh"
  description: "Fires after nightly ETL completes"
}

# PDT materialized to a scratch schema in BigQuery
view: daily_revenue_pdt {
  derived_table: {
    sql:
      SELECT
        DATE(created_at)  AS order_date,
        region,
        SUM(sale_price)   AS revenue,
        COUNT(*)          AS order_count
      FROM `proj.ds.orders`
      GROUP BY 1, 2
    ;;
    datagroup_trigger: daily_refresh    # Rebuild when datagroup fires
    # persist_for: "24 hours"           # Alternative: simple time-based
    partition_keys: ["order_date"]      # BigQuery partition on PDT output
    cluster_keys:   ["region"]
  }

  dimension: order_date {
    primary_key: yes
    type:        date
    sql:         ${TABLE}.order_date ;;
  }

  measure: total_revenue {
    type:            sum
    sql:             ${TABLE}.revenue ;;
    value_format_name: usd
  }
}
```

---

### Incremental PDT

```lookml
view: events_pdt {
  derived_table: {
    sql:
      SELECT
        event_id,
        user_id,
        event_type,
        created_at
      FROM `proj.ds.events`
      WHERE {% incrementcondition %} created_at {% endincrementcondition %}
    ;;
    datagroup_trigger: daily_refresh
    increment_key:    "created_at"        # Partition column to increment on
    increment_offset: 3                   # Re-process last 3 days for late data
  }
}
```

> 💡 **When to use each derived table type:**
> - **SQL DT** — complex SQL not expressible in LookML; one-off aggregations
> - **NDT** — reuse existing Explore logic; maintainable without raw SQL
> - **PDT** — expensive queries; pre-compute for dashboard performance
> - **Incremental PDT** — huge tables; only append new data each run

---

## 5. Dimensions, Measures & Calculations

### Dimension Types Deep Dive

```lookml
view: examples {
  sql_table_name: `proj.ds.examples` ;;

  # String
  dimension: name {
    type: string
    sql:  ${TABLE}.name ;;
  }

  # Number (calculated)
  dimension: profit_margin {
    type: number
    sql:  (${TABLE}.revenue - ${TABLE}.cost) / NULLIF(${TABLE}.revenue, 0) ;;
    value_format_name: percent_2
  }

  # Yes/No boolean
  dimension: is_high_value {
    type: yesno
    sql:  ${TABLE}.order_value >= 1000 ;;
  }

  # Tier (bucketing)
  dimension: age_group {
    type:  tier
    tiers: [18, 25, 35, 45, 55, 65]
    style: integer    # or "relational" for < 18, >= 65 labels
    sql:   ${TABLE}.age ;;
  }

  # Location (latitude/longitude for maps)
  dimension: location {
    type:        location
    sql_latitude:  ${TABLE}.lat ;;
    sql_longitude: ${TABLE}.lng ;;
  }

  # Duration between two timestamp columns
  dimension_group: created {
    type:       time
    timeframes: [raw, date, week, month, year]
    sql:        ${TABLE}.created_at ;;
  }

  dimension_group: shipped {
    type:       time
    timeframes: [raw, date]
    sql:        ${TABLE}.shipped_at ;;
  }

  dimension: days_to_ship {
    type:      duration_day
    sql_start: ${TABLE}.created_at ;;
    sql_end:   ${TABLE}.shipped_at ;;
    label:     "Days to Ship"
  }
}
```

---

### Measure Types & Patterns

```lookml
view: measure_examples {
  sql_table_name: `proj.ds.sales` ;;

  # Basic aggregations
  measure: count          { type: count }
  measure: total_revenue  { type: sum;  sql: ${TABLE}.revenue ;; value_format_name: usd }
  measure: avg_order      { type: average; sql: ${TABLE}.order_value ;; }
  measure: max_order      { type: max; sql: ${TABLE}.order_value ;; }
  measure: min_order      { type: min; sql: ${TABLE}.order_value ;; }
  measure: p95_order      { type: percentile; percentile: 95; sql: ${TABLE}.order_value ;; }

  # Count distinct
  measure: unique_customers {
    type: count_distinct
    sql:  ${TABLE}.user_id ;;
  }

  # Filtered measure (conditional aggregation)
  measure: mobile_revenue {
    type:    sum
    sql:     ${TABLE}.revenue ;;
    filters: [platform: "mobile"]   # Adds CASE WHEN to SQL
    label:   "Mobile Revenue"
  }

  # Measure using other measures (no sql needed)
  measure: mobile_revenue_share {
    type:            number
    sql:             ${mobile_revenue} / NULLIF(${total_revenue}, 0) ;;
    value_format_name: percent_2
  }

  dimension: platform { type: string; sql: ${TABLE}.platform ;; }
}
```

---

### Table Calculations (In-UI, Not LookML)

Table calculations run **after** the query returns — they compute on the result set, not in the database.

| Function | Example | Use Case |
|---|---|---|
| `offset()` | `${orders.total_revenue} - offset(${orders.total_revenue}, 1)` | Period-over-period change |
| `percent_of_total()` | `percent_of_total(${orders.total_revenue})` | Share of total |
| `running_total()` | `running_total(${orders.count})` | Cumulative total |
| `diff_days()` | `diff_days(${orders.created_date}, now())` | Days since event |
| `if()` | `if(${orders.status} = "complete", "✅", "❌")` | Conditional display |

> ⚠️ Table calculations are **not version-controlled** and can't be reused across dashboards. Prefer LookML measures for anything reusable or governed.

---

## 6. Filters, Parameters & Templated SQL

### Filter Fields

```lookml
view: orders {
  # filter field — adds a WHERE clause but doesn't SELECT the column
  filter: date_filter {
    type:        date
    label:       "Order Date Filter"
    description: "Filter orders by date range (doesn't add date to results)"
    sql_start:   ${TABLE}.created_at ;;
  }

  filter: region_filter {
    type: string
    sql:  ${TABLE}.region ;;
  }
}
```

---

### Parameter Fields & Liquid Templating

```lookml
view: dynamic_metrics {
  sql_table_name: `proj.ds.sales` ;;

  # Parameter: user selects a metric type
  parameter: metric_selector {
    type:          unquoted
    default_value: "revenue"
    allowed_values: [
      { label: "Revenue" value: "revenue" }
      { label: "Orders"  value: "orders" }
      { label: "Profit"  value: "profit" }
    ]
  }

  # Dimension that changes based on parameter selection
  dimension: selected_metric_value {
    type: number
    sql:
      {% if metric_selector._parameter_value == 'revenue' %}
        ${TABLE}.revenue
      {% elsif metric_selector._parameter_value == 'orders' %}
        1
      {% else %}
        ${TABLE}.profit
      {% endif %}
    ;;
    label: "Selected Metric"
  }

  # Measure that changes dynamically
  measure: dynamic_total {
    type: sum
    sql:  ${selected_metric_value} ;;
    label: "{% parameter metric_selector %} Total"
  }
}
```

---

### User Attributes in Liquid

```lookml
view: personalized_view {
  sql_table_name: `proj.ds.sales` ;;

  dimension: my_region_data {
    type: string
    sql:
      CASE
        WHEN ${TABLE}.region = '{{ _user_attributes["allowed_region"] }}'
        THEN ${TABLE}.region
        ELSE 'Other'
      END
    ;;
  }
}
```

---

### _in_query Condition

```lookml
view: flexible_dimension {
  # Change SQL behavior based on whether another field is in the query
  dimension: revenue_or_profit {
    type: number
    sql:
      {% if orders.is_returned._in_query %}
        ${TABLE}.revenue - ${TABLE}.return_cost
      {% else %}
        ${TABLE}.revenue
      {% endif %}
    ;;
  }
}
```

---

### Dynamic SQL with Liquid

```lookml
view: filtered_view {
  sql_table_name: `proj.ds.events` ;;

  # Parameter-driven date granularity
  parameter: date_granularity {
    type: unquoted
    allowed_values: [
      { label: "Day"   value: "DATE" }
      { label: "Week"  value: "WEEK" }
      { label: "Month" value: "MONTH" }
    ]
  }

  dimension: period {
    type: string
    sql:  DATE_TRUNC(${TABLE}.event_date, {% parameter date_granularity %}) ;;
    label: "Period (by {% parameter date_granularity %})"
  }
}
```

---

## 7. Dashboards & Looks

### LookML Dashboard File Structure

```lookml
# dashboards/sales_overview.dashboard.lookml

- dashboard: sales_overview
  title: "Sales Overview"
  layout: newspaper
  preferred_viewer: dashboards-next

  # Dashboard-level filters
  filters:
    - name: date_filter
      title: "Date Range"
      type: date_filter
      default_value: "last 30 days"
      allow_multiple_values: false
      required: false

    - name: region_filter
      title: "Region"
      type: field_filter
      explore: orders
      field: orders.region
      default_value: ''

  # Tiles / elements
  elements:
    - title: "Total Revenue"
      name:  total_revenue_kpi
      model: ecommerce
      explore: orders
      type:  single_value
      fields: [orders.total_revenue]
      filters:
        orders.created_date: "{% date_start date_filter %} to {% date_end date_filter %}"
      row: 0
      col: 0
      width: 4
      height: 4
      vis_config:
        type: single_value
        custom_color_enabled: true
        custom_color: "#4CAF50"
        font_size: large
        title_hidden: false

    - title: "Revenue by Region"
      name:  revenue_by_region
      model: ecommerce
      explore: orders
      type:  looker_bar
      fields: [orders.region, orders.total_revenue]
      sorts:  [orders.total_revenue desc]
      limit:  10
      filters:
        orders.created_date: "{% date_start date_filter %} to {% date_end date_filter %}"
        orders.region: "{{ region_filter }}"
      row: 0
      col: 4
      width: 12
      height: 8
      vis_config:
        type: looker_bar
        x_axis_label: "Region"
        y_axis_label: "Revenue"
        show_value_labels: true
        label_density: 25
        legend_position: center

    - title: "Daily Revenue Trend"
      name:  daily_revenue
      model: ecommerce
      explore: orders
      type:  looker_line
      fields: [orders.created_date, orders.total_revenue, orders.count]
      sorts:  [orders.created_date asc]
      filters:
        orders.created_date: "{% date_start date_filter %} to {% date_end date_filter %}"
      row: 8
      col: 0
      width: 16
      height: 8
      vis_config:
        type: looker_line
        y_axes:
          - label: "Revenue"
            series:
              - id: orders.total_revenue
                name: "Revenue"
                axisId: revenue_axis
```

---

### Dashboard Layout Grid

```
Grid: 24 columns × unlimited rows
Each cell unit = ~50px height

col:   0–23  (horizontal position, 0 = leftmost)
row:   0–N   (vertical position, 0 = top)
width: 1–24  (columns wide)
height: 1–N  (row units tall)

Example layouts:
  Full width:   col:0, width:24
  Half width:   col:0, width:12  +  col:12, width:12
  Third width:  col:0, width:8   +  col:8, width:8  +  col:16, width:8
  KPI row:      4× col:0/6/12/18, width:6, height:4
```

---

### Scheduling & Delivery

| Destination | Notes |
|---|---|
| **Email** | PDF, CSV, PNG, Excel attachments |
| **Slack** | Webhook or native Slack app integration |
| **Google Drive** | Sheets or Drive folder |
| **Webhook** | POST payload to any HTTPS endpoint |
| **Amazon S3 / GCS** | File export to cloud storage |
| **SFTP** | Secure file transfer |

```bash
# Schedule a Look via API
curl -X POST "https://mycompany.looker.com:19999/api/4.0/scheduled_plans" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Daily Revenue Report",
    "look_id": 42,
    "crontab": "0 8 * * 1-5",
    "scheduled_plan_destination": [{
      "format": "csv",
      "type": "email",
      "address": "team@company.com"
    }]
  }'
```

---

## 8. User Attributes & Row-Level Security

### Built-in vs. Custom User Attributes

| Attribute | Type | Example Value |
|---|---|---|
| `email` | Built-in | `analyst@company.com` |
| `name` | Built-in | `Jane Smith` |
| `id` | Built-in | `42` |
| `locale` | Built-in | `en_US` |
| `timezone` | Built-in | `America/New_York` |
| `allowed_region` | Custom | `EMEA` |
| `tenant_id` | Custom | `cust_12345` |
| `department` | Custom | `finance` |

---

### Row-Level Security Patterns

```lookml
# ── Pattern 1: Tenant Isolation (SaaS) ────────────────────────────────
explore: orders {
  access_filter: {
    field:          orders.tenant_id
    user_attribute: tenant_id         # Each embedded user gets their own tenant_id
  }
}

# ── Pattern 2: Regional Data Access ───────────────────────────────────
explore: sales {
  access_filter: {
    field:          sales.region
    user_attribute: allowed_region    # "EMEA", "APAC", "AMER", or "" for all
  }
}

# ── Pattern 3: Dynamic Row Filter via Liquid ───────────────────────────
view: restricted_data {
  sql_table_name: `proj.ds.sales` ;;

  dimension: region {
    type: string
    sql:  ${TABLE}.region ;;
  }

  # Use sql_always_where in explore for this:
  # sql_always_where: ${TABLE}.region = '{{ _user_attributes["allowed_region"] }}' ;;
}

# ── Pattern 4: Access Grant for Field-Level Security ──────────────────
access_grant: can_see_pii {
  user_attribute:  pii_access
  allowed_values:  ["yes"]
}

view: users {
  dimension: email {
    type:                   string
    sql:                    ${TABLE}.email ;;
    required_access_grants: [can_see_pii]    # Hidden if user lacks grant
  }

  dimension: hashed_email {
    type: string
    sql:  TO_HEX(MD5(${TABLE}.email)) ;;
    # Shown to everyone as a safe alternative
  }
}
```

---

### Looker API: Create User Attribute

```python
import looker_sdk
from looker_sdk import models40 as models

sdk = looker_sdk.init40()

# Create a custom user attribute
attribute = sdk.create_user_attribute(
    body=models.WriteUserAttribute(
        name="allowed_region",
        label="Allowed Region",
        type="string",
        default_value="AMER",
        user_can_view=False,
        user_can_edit=False,
        hidden_value_domain_whitelist=""
    )
)
print(f"Created attribute ID: {attribute.id}")

# Assign value to a specific user
sdk.set_user_attribute_user_value(
    user_id=42,
    user_attribute_id=attribute.id,
    body=models.WriteUserAttributeWithValue(value="EMEA")
)
```

---

## 9. Caching & Performance

### How Looker Caching Works

```
Query submitted
      │
      ▼
Cache key = SHA256(SQL + database + user_context)
      │
      ├── Cache HIT?  ─────► Return cached result (0 DB cost)
      │
      └── Cache MISS? ─────► Run SQL against database
                                   │
                                   ▼
                            Store result in cache
                            (until datagroup fires or persist_for expires)
```

---

### Datagroup Definition & Assignment

```lookml
# manifest.lkml or model file

# Option 1: SQL trigger (fires when value changes)
datagroup: nightly_etl {
  sql_trigger:  SELECT MAX(job_id) FROM `proj.ds.etl_log` WHERE status='success' ;;
  max_cache_age: "24 hours"
}

# Option 2: Interval-based (fires every N hours)
datagroup: every_6_hours {
  interval_trigger: "6 hours"
  max_cache_age:    "6 hours"
}

# Assign to an explore
explore: orders {
  persist_with: nightly_etl    # All queries in this explore use this cache policy
}

# Assign to a PDT
view: revenue_pdt {
  derived_table: {
    sql: SELECT ... ;;
    datagroup_trigger: nightly_etl
  }
}
```

---

### Aggregate Awareness

```lookml
# Define a pre-aggregated rollup table
view: orders_daily_agg {
  derived_table: {
    sql:
      SELECT
        DATE(created_at) AS date,
        region,
        COUNT(*)         AS order_count,
        SUM(sale_price)  AS revenue
      FROM `proj.ds.orders`
      GROUP BY 1, 2
    ;;
    datagroup_trigger: nightly_etl
  }
  # ... define fields ...
}

# Register as an aggregate_table on the explore
explore: orders {
  aggregate_table: orders_daily_summary {
    query: {
      dimensions: [orders.created_date, orders.region]
      measures:   [orders.count, orders.total_revenue]
    }
    materialization: {
      datagroup_trigger: nightly_etl
    }
  }
}
```

> 🚀 When a user queries `orders` with only `created_date`, `region`, `count`, and `total_revenue`, Looker **automatically rewrites the query** to hit the aggregate table — no user action needed. This can reduce query cost by 100x for high-cardinality tables.

---

### Performance Optimization Checklist

| Optimization | How | Impact |
|---|---|---|
| **Partition pushdown** | Use `always_filter` on partition column | 🚀 Massive BQ cost/speed |
| **Correct relationships** | Use `many_to_one` where appropriate | ✅ Accurate counts |
| **count_distinct vs. count** | Use `count` on base table, `count_distinct` on joined | ✅ Correct after fan-out |
| **Aggregate awareness** | Define `aggregate_table` for common queries | 🚀 Major performance |
| **PDTs** | Pre-compute heavy derived tables | 🚀 Eliminate repeated compute |
| **Cache warming** | Schedule common dashboards nightly | ✅ Fast morning load |
| **BI Engine** | Enable BigQuery BI Engine reservation | 🚀 Sub-second response |
| **Avoid SELECT \*** | Always specify columns in PDT SQL | ✅ Reduce I/O |

---

## 10. Looker API

### Authentication (OAuth2 Client Credentials)

```python
# pip install looker-sdk

import looker_sdk
from looker_sdk import models40 as models

# Configure via environment variables or looker.ini file
# LOOKERSDK_BASE_URL=https://mycompany.looker.com:19999
# LOOKERSDK_CLIENT_ID=your_client_id
# LOOKERSDK_CLIENT_SECRET=your_client_secret

sdk = looker_sdk.init40()   # Automatically authenticates

# Or configure explicitly
import looker_sdk.rtl.api_settings as settings

config = settings.ApiSettings(
    base_url="https://mycompany.looker.com:19999",
    client_id="your_client_id",
    client_secret="your_client_secret",
    verify_ssl="true"
)
sdk = looker_sdk.init40(config=config)
```

---

### Key API Operations

```python
import looker_sdk
from looker_sdk import models40 as models
import json

sdk = looker_sdk.init40()

# ── Run a query directly ──────────────────────────────────────────────
query = sdk.create_query(
    body=models.WriteQuery(
        model="ecommerce",
        view="orders",
        fields=["orders.created_date", "orders.total_revenue", "orders.count"],
        filters={"orders.created_date": "last 30 days"},
        sorts=["orders.created_date desc"],
        limit="100"
    )
)
results = sdk.run_query(query_id=query.id, result_format="json")
data = json.loads(results)
print(f"Got {len(data)} rows")

# ── Get dashboard metadata ────────────────────────────────────────────
dashboard = sdk.dashboard(dashboard_id="42")
print(f"Dashboard: {dashboard.title}, tiles: {len(dashboard.dashboard_elements)}")

# ── Get all Looks ─────────────────────────────────────────────────────
looks = sdk.all_looks(fields="id,title,folder_id,updated_at")
for look in looks:
    print(f"Look {look.id}: {look.title}")

# ── Run a Look and get results ────────────────────────────────────────
result = sdk.run_look(look_id=42, result_format="csv")
print(result[:500])   # First 500 chars of CSV

# ── Create a user ─────────────────────────────────────────────────────
user = sdk.create_user(
    body=models.WriteUser(
        first_name="Jane",
        last_name="Analyst",
        email="jane@company.com",
        is_disabled=False
    )
)
print(f"Created user ID: {user.id}")

# Grant a role to the user
roles = sdk.all_roles(fields="id,name")
viewer_role = next(r for r in roles if r.name == "Viewer")
sdk.set_user_roles(user_id=user.id, body=[viewer_role.id])

# ── Schedule a delivery ────────────────────────────────────────────────
plan = sdk.create_scheduled_plan(
    body=models.WriteScheduledPlan(
        name="Daily Orders Report",
        look_id=42,
        crontab="0 8 * * 1-5",   # 8am Mon–Fri
        enabled=True,
        scheduled_plan_destination=[
            models.ScheduledPlanDestination(
                format="csv_zip",
                type="email",
                address="team@company.com",
                message="Morning report attached"
            )
        ]
    )
)

# ── List folders ──────────────────────────────────────────────────────
folders = sdk.all_folders(fields="id,name,parent_id")
for f in folders:
    print(f"Folder {f.id}: {f.name}")
```

---

### REST API Quick Reference

```bash
BASE="https://mycompany.looker.com:19999/api/4.0"
TOKEN=$(curl -s -X POST "$BASE/login" \
  -d "client_id=CID&client_secret=SECRET" | jq -r .access_token)

# Run an inline query
curl -X POST "$BASE/queries/run/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model":"ecommerce","view":"orders",
       "fields":["orders.count"],"limit":"1"}'

# Get dashboard
curl "$BASE/dashboards/42" -H "Authorization: Bearer $TOKEN"

# Get all users
curl "$BASE/users?fields=id,email,name" -H "Authorization: Bearer $TOKEN"

# Run a Look as CSV
curl "$BASE/looks/42/run/csv" -H "Authorization: Bearer $TOKEN"
```

---

### API Rate Limits & Best Practices

| Limit | Value | Notes |
|---|---|---|
| API calls per minute | ~6,000 (varies by plan) | Use SDK's retry logic |
| Concurrent queries | Plan-dependent | Use query queueing |
| Max result rows | 5,000 (default), 100,000 (max) | Use streaming for larger sets |
| Token expiration | 1 hour | SDK auto-refreshes |

> 💡 For large data exports, use `sdk.run_query()` with `result_format="json_detail"` + `apply_vis=False` for maximum rows, or use the BigQuery subscription directly.

---

## 11. Embedding

### Embedding Types

| Type | Authentication | Use Case | Complexity |
|---|---|---|---|
| **Signed Embed (SSO)** | Signed URL with HMAC | External customers, multi-tenant | High |
| **Private Embed** | Looker login required | Internal authenticated users | Low |
| **Public Embed** | No auth | Public-facing dashboards | Lowest |
| **Looker Components** | SDK + OAuth | Custom React data apps | Highest |

---

### Signed Embed URL Generation (Python)

```python
import hashlib
import hmac
import json
import time
import urllib.parse
import binascii
import os

LOOKER_HOST    = "mycompany.looker.com"
LOOKER_SECRET  = os.environ["LOOKER_EMBED_SECRET"]

def generate_signed_embed_url(
    embed_path: str,          # e.g., "/embed/dashboards/42"
    user: dict,
    session_length: int = 3600
) -> str:

    nonce = binascii.b2a_hex(os.urandom(16)).decode("utf-8")
    current_time = str(int(time.time()))

    # Build the embed spec
    embed_user = {
        "external_user_id": user["id"],           # Unique identifier for this user
        "first_name":       user.get("first_name", ""),
        "last_name":        user.get("last_name", ""),
        "email":            user["email"],
        "permissions":      ["access_data", "see_looks", "see_user_dashboards"],
        "models":           ["ecommerce"],         # Which models they can access
        "group_ids":        user.get("group_ids", []),
        "external_group_id": user.get("group", ""),
        "user_attributes":  user.get("attributes", {}),  # e.g., {"tenant_id": "cust_123"}
        "access_filters":   {}
    }

    params = {
        "nonce":          nonce,
        "time":           current_time,
        "session_length": str(session_length),
        "external_user_id": user["id"],
        "permissions":    json.dumps(embed_user["permissions"]),
        "models":         json.dumps(embed_user["models"]),
        "group_ids":      json.dumps(embed_user.get("group_ids", [])),
        "external_group_id": embed_user["external_group_id"],
        "user_attributes": json.dumps(embed_user["user_attributes"]),
        "access_filters": json.dumps(embed_user["access_filters"]),
        "first_name":     embed_user["first_name"],
        "last_name":      embed_user["last_name"],
        "email":          embed_user["email"],
        "force_logout_login": "true"
    }

    embed_path_with_params = embed_path
    query_string = "&".join(f"{k}={urllib.parse.quote(str(v), safe='')}"
                            for k, v in params.items())

    string_to_sign = "\n".join([
        LOOKER_HOST,
        "/embed" + embed_path,
        "",           # permissions (embedded in path in newer versions)
        nonce,
        current_time,
        str(session_length),
        json.dumps(embed_user["permissions"]),
        json.dumps(embed_user["models"])
    ])

    signature = hmac.new(
        LOOKER_SECRET.encode("utf-8"),
        string_to_sign.encode("utf-8"),
        hashlib.sha1
    ).digest()
    encoded_sig = urllib.parse.quote(
        binascii.b2a_base64(signature).decode("utf-8").strip()
    )

    return (f"https://{LOOKER_HOST}/embed{embed_path}"
            f"?{query_string}&signature={encoded_sig}")

# Usage
url = generate_signed_embed_url(
    embed_path="/dashboards/42",
    user={
        "id":         "customer_123",
        "email":      "customer@example.com",
        "first_name": "Jane",
        "last_name":  "Customer",
        "attributes": {"tenant_id": "cust_123", "allowed_region": "EMEA"}
    }
)
print(url)  # Use this as the iframe src in your web app
```

---

### JavaScript Embed SDK

```javascript
// npm install @looker/embed-sdk

import { LookerEmbedSDK } from '@looker/embed-sdk'

// Initialize once (in your app setup)
LookerEmbedSDK.init('mycompany.looker.com', {
  // For signed embed: provide the URL signing endpoint on your backend
  acquireSession: '/api/looker/session'
})

// Embed a dashboard
LookerEmbedSDK.createDashboardWithId(42)
  .appendTo('#dashboard-container')         // DOM element to embed into
  .withClassName('looker-dashboard')
  .withTheme('dark_minimal')
  .withParams({                             // URL parameters / filter defaults
    'orders.region': 'EMEA',
    'orders.created_date': 'last 30 days'
  })
  .on('dashboard:loaded', (event) => {
    console.log('Dashboard loaded:', event.dashboard.title)
  })
  .on('dashboard:filters:changed', (event) => {
    console.log('Filters changed:', event.dashboard.filters)
  })
  .on('drillmenu:click', (event) => {
    console.log('User drilled into:', event.label)
  })
  .build()
  .connect()
  .then(dashboard => {
    // Update filters programmatically
    dashboard.updateFilters({
      'orders.region': 'AMER'
    })
    dashboard.run()   // Re-run the dashboard with new filters
  })
  .catch(console.error)

// Embed a Look
LookerEmbedSDK.createLookWithId(42)
  .appendTo('#look-container')
  .on('look:ready', (event) => console.log('Look ready'))
  .build()
  .connect()

// Embed an Explore
LookerEmbedSDK.createExploreWithId('ecommerce/orders')
  .appendTo('#explore-container')
  .build()
  .connect()
```

---

### CSP & Allowed Domains

```bash
# Add your application domain to Looker's embed allowlist
# Admin → Embed → Embedded Domain Allowlist

# Example domains to add:
# https://app.mycompany.com
# https://dashboard.mycompany.com
# http://localhost:3000  (for local development)

# Content-Security-Policy on your app server must include:
# frame-src https://mycompany.looker.com;
```

---

## 12. Administration & Governance

### Roles & Permissions Architecture

```
Looker Roles (named sets of permissions + model sets)
    │
    ├── Permission Set (what you can DO)
    │     ├── access_data
    │     ├── see_looks
    │     ├── see_user_dashboards
    │     ├── explore
    │     ├── create_table_calculations
    │     ├── download_with_limit / without_limit
    │     ├── schedule_look_emails
    │     ├── develop             (LookML development)
    │     ├── deploy              (push to production)
    │     └── manage_users        (admin)
    │
    └── Model Set (which MODELS you can access)
          └── [ecommerce, marketing, finance]
```

| Built-in Role | Typical Permissions |
|---|---|
| **Admin** | All permissions across all models |
| **Developer** | develop + deploy + access_data on assigned models |
| **Viewer** | see_looks + see_user_dashboards + access_data |

---

### Git Integration & Development Workflow

```
Production Branch (main)
    │
    ▼
Looker Production Mode
    │  Developer creates branch
    ▼
Feature Branch (feat/new-revenue-metric)
    │  Develop in Looker IDE
    │  Validate LookML
    │  Test in Dev Mode (your branch)
    ▼
Pull Request → Code Review → CI lint/test
    │  Merge to main
    ▼
Deploy to Production (Admin → Projects → Deploy)
```

```bash
# Looker CLI for LookML validation (lookml-tools)
pip install lookml-tools
lkmlint my_project/     # Lint all LookML files

# Or use lookml-test for data tests
# defined in LookML as lookml_test blocks
```

---

### LookML Tests

```lookml
# tests/revenue_tests.lkml

test: total_revenue_is_positive {
  explore_source: orders {
    column: total_revenue {}
  }
  assert: total_revenue_is_positive {
    expression: ${orders.total_revenue} > 0 ;;
  }
}

test: order_count_matches_table {
  explore_source: orders {
    column: count {}
    filters: [orders.created_date: "2026-01-01 to 2026-01-31"]
  }
  assert: january_orders_exist {
    expression: ${orders.count} > 0 ;;
  }
}
```

---

### Looker API: User & Group Management

```python
import looker_sdk
from looker_sdk import models40 as models

sdk = looker_sdk.init40()

# Create a group
group = sdk.create_group(body=models.WriteGroup(name="EMEA Analysts"))

# Add user to group
sdk.add_group_user(
    group_id=group.id,
    body=models.GroupIdForGroupUserInclusion(user_id=42)
)

# Assign role to group
roles = sdk.all_roles(fields="id,name")
viewer_role = next(r for r in roles if r.name == "Viewer")
sdk.set_role_groups(role_id=viewer_role.id,
                    body=[group.id])

# Set user attribute value for a group
sdk.update_user_attribute_group_value(
    group_id=group.id,
    user_attribute_id=7,        # ID of "allowed_region" attribute
    body=models.WriteUserAttributeWithValue(value="EMEA")
)
```

---

### System Activity

Looker's **System Activity** is a built-in Explore that queries Looker's own internal database. Key Explores:

| Explore | What It Shows |
|---|---|
| `history` | All query runs: user, query, runtime, status |
| `user` | User list with last login, roles |
| `look` | All Looks with view count, last accessed |
| `dashboard` | All dashboards with view count |
| `query` | Unique queries with cache hit rate |
| `pdt_builds` | PDT build history and duration |
| `connection_usage` | Database query counts and bytes |

```lookml
# Useful System Activity query (run in Explore → System Activity → History):
# Fields: user.email, query.view, history.query_run_count, history.average_runtime
# Filter: history.created_date = "last 7 days"
# Sort: history.query_run_count desc
# → Shows your most active users and their most common queries
```

---

## 13. Looker & BigQuery Integration

### BigQuery Connection Setup

```bash
# 1. Create a dedicated service account for Looker
gcloud iam service-accounts create looker-bq-sa \
  --display-name="Looker BigQuery Service Account"

# 2. Grant required BigQuery roles
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:looker-bq-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataViewer"

gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:looker-bq-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/bigquery.jobUser"

# 3. For PDTs: grant access to a scratch dataset
bq mk --dataset my-project:looker_scratch
bq update \
  --add_iam_policy=policy.json \        # bigquery.dataEditor on scratch dataset
  my-project:looker_scratch

# 4. Download service account key (or use Workload Identity for GCE-hosted Looker)
gcloud iam service-accounts keys create looker-sa-key.json \
  --iam-account=looker-bq-sa@my-project.iam.gserviceaccount.com
```

---

### BigQuery-Optimized LookML Patterns

```lookml
view: events_bigquery {
  # BigQuery partitioned table
  sql_table_name: `my_project.analytics.events` ;;

  # ALWAYS force partition filter to prevent full table scans
  dimension: event_date {
    primary_key: no
    type:        date
    sql:         ${TABLE}.event_date ;;
  }

  dimension_group: event {
    type:       time
    timeframes: [raw, date, week, month, year]
    sql:        ${TABLE}.event_timestamp ;;
  }

  measure: event_count {
    type: count
  }
}

# Apply mandatory partition filter in explore
explore: events {
  view_name: events_bigquery

  # Force users to always filter on partition column
  always_filter: {
    filters: [events_bigquery.event_date: "last 7 days"]
  }

  # Or use sql_always_where for non-removable filter
  sql_always_where: ${events_bigquery.event_date} >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY) ;;
}
```

---

### BigQuery ML in Looker

```lookml
# Call a BQML model from a LookML derived table
view: churn_predictions {
  derived_table: {
    sql:
      SELECT
        user_id,
        predicted_churned,
        predicted_churned_probs[OFFSET(1)].prob AS churn_probability
      FROM
        ML.PREDICT(
          MODEL `my_project.ml_models.churn_model`,
          (
            SELECT
              user_id,
              days_since_last_order,
              total_orders,
              avg_order_value
            FROM `my_project.features.user_features`
            WHERE prediction_date = CURRENT_DATE()
          )
        )
    ;;
    datagroup_trigger: daily_refresh
  }

  dimension: user_id {
    primary_key: yes
    type:        number
    sql:         ${TABLE}.user_id ;;
  }

  dimension: churn_probability {
    type:            number
    sql:             ${TABLE}.churn_probability ;;
    value_format_name: percent_2
  }

  dimension: churn_risk_tier {
    type: tier
    tiers: [0.3, 0.6, 0.8]
    sql:  ${churn_probability} ;;
    style: relational
  }
}
```

---

### Looker + Dataform Integration

```lookml
# Dataform manages the SQL transformations;
# Looker references the resulting BigQuery tables

view: orders {
  # Point directly to Dataform-managed table
  sql_table_name: `my_project.dataform_prod.orders` ;;

  # Reference Dataform-managed view
  # sql_table_name: `my_project.dataform_prod.orders_with_customer_info` ;;
}

# Dataform datagroup trigger — fire when Dataform workflow completes
datagroup: dataform_completed {
  sql_trigger:
    SELECT MAX(completion_time)
    FROM `my_project.dataform_internal.workflow_invocations`
    WHERE status = 'SUCCEEDED'
  ;;
  max_cache_age: "24 hours"
}
```

---

## 14. LookML Best Practices & Patterns

### Refinements (Non-Destructive Extension)

```lookml
# Original view (don't edit)
# views/orders.view.lkml
view: orders {
  sql_table_name: `proj.ds.orders` ;;
  dimension: id { primary_key: yes; type: number; sql: ${TABLE}.id ;; }
  measure: count { type: count }
}

# Refinement file — add fields without editing original
# views/orders_refinements.view.lkml
view: +orders {           # + prefix = refinement
  dimension: high_value_flag {
    type:  yesno
    sql:   ${TABLE}.sale_price >= 500 ;;
    label: "High Value Order"
  }

  # Override an existing field
  measure: +count {
    label: "Total Orders (refined)"
    drill_fields: [id, status]
  }
}
```

---

### Extends (Inheritance)

```lookml
# Base view with common fields
view: _base_events {
  extension: required     # This view can't be used directly

  dimension: event_id {
    primary_key: yes
    type:        string
    sql:         ${TABLE}.event_id ;;
  }

  dimension_group: occurred {
    type:       time
    timeframes: [raw, date, week, month]
    sql:        ${TABLE}.occurred_at ;;
  }

  measure: event_count {
    type:  count
    label: "Events"
  }
}

# Concrete view extending the base
view: page_views {
  extends: [_base_events]
  sql_table_name: `proj.ds.page_views` ;;

  # Add page-view-specific fields
  dimension: page_url {
    type: string
    sql:  ${TABLE}.url ;;
  }
}

view: button_clicks {
  extends: [_base_events]
  sql_table_name: `proj.ds.button_clicks` ;;

  dimension: button_id {
    type: string
    sql:  ${TABLE}.button_id ;;
  }
}
```

---

### Manifest File

```lookml
# manifest.lkml

project_name: "my_ecommerce_project"

# Environment-specific constants
constant: bq_project {
  value: "my-prod-project"
  export: override_optional
}

constant: dataset {
  value: "analytics"
  export: override_optional
}

# Use constants in views:
# sql_table_name: `@{bq_project}.@{dataset}.orders` ;;

# Import from another Looker project
remote_dependency: shared_views {
  url:    "https://github.com/myorg/looker-shared-views"
  ref:    "v1.2.0"    # Git tag or commit SHA
}

# Localization
localization_settings: {
  default_locale: en
  localization_level: permissive
}
```

---

### CI/CD for LookML (GitHub Actions)

```yaml
# .github/workflows/lookml-ci.yml
name: LookML CI

on:
  pull_request:
    paths: ['**.lkml']

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install lookml-tools
        run: pip install lookml-tools looker-sdk

      - name: Lint LookML
        run: lkmlint .

      - name: Run LookML Tests via API
        env:
          LOOKERSDK_BASE_URL: ${{ secrets.LOOKER_BASE_URL }}
          LOOKERSDK_CLIENT_ID: ${{ secrets.LOOKER_CLIENT_ID }}
          LOOKERSDK_CLIENT_SECRET: ${{ secrets.LOOKER_CLIENT_SECRET }}
        run: |
          python << 'EOF'
          import looker_sdk
          sdk = looker_sdk.init40()
          results = sdk.run_lookml_test(
              project_id="my_ecommerce_project",
              file_id="tests/revenue_tests.lkml"
          )
          failures = [r for r in results if not r.success]
          if failures:
              for f in failures:
                  print(f"FAILED: {f.test_name} — {f.errors}")
              exit(1)
          print(f"All {len(results)} tests passed")
          EOF

      - name: Validate LookML Project
        env:
          LOOKERSDK_BASE_URL: ${{ secrets.LOOKER_BASE_URL }}
          LOOKERSDK_CLIENT_ID: ${{ secrets.LOOKER_CLIENT_ID }}
          LOOKERSDK_CLIENT_SECRET: ${{ secrets.LOOKER_CLIENT_SECRET }}
        run: |
          python << 'EOF'
          import looker_sdk
          sdk = looker_sdk.init40()
          errors = sdk.lookml_model_explore(
              lookml_model_name="ecommerce",
              explore_name="orders"
          )
          print("Project validates OK")
          EOF
```

---

### Common Anti-Patterns

| ❌ Anti-Pattern | 🔴 Problem | ✅ Fix |
|---|---|---|
| Missing `primary_key` on views | Inaccurate counts after joins | Always define `primary_key: yes` |
| `many_to_many` joins | Severe fanout; wrong numbers | Use a bridge/fact table; break into two `many_to_one` |
| Deriving everything in DTs | Slow builds, hard to maintain | Use Dataform/dbt for complex transforms; DT for Looker-specific agg |
| No datagroups | Cache never invalidates (or always misses) | Assign `persist_with` on every explore |
| `sql: ${TABLE}.*` in PDT | Full column scan in BQ | Always specify columns explicitly |
| Sensitive fields in base view, no access_grant | All users see PII | Add `required_access_grants` to sensitive dimensions |
| No `always_filter` on partitioned tables | Full BQ table scan on every query | Force partition filter with `always_filter` |
| `count` after `one_to_many` join | Inflated counts from fan-out | Use `count_distinct` on the base PK |
| Huge explores with all views joined | Slow explore load; confusing UX | Split into focused explores per use case |
| User-defined dashboards for prod metrics | Version drift; no governance | Use LookML dashboards for governed content |

---

## 15. Pricing Summary

> ⚠️ Looker pricing is **negotiated** and not publicly listed per-unit. The below is representative as of early 2026. Contact Google Sales for current pricing.

### Licensing Model

| Seat Type | Who | Capabilities |
|---|---|---|
| **Developer** | Data modelers, LookML engineers | Full IDE access, develop + deploy |
| **Standard (User)** | Analysts who build Explores/dashboards | Explore data, create content |
| **Viewer** | Dashboard consumers | View dashboards and Looks only |

### Platform vs. Embedded

| Model | Use Case | Pricing Basis |
|---|---|---|
| **Looker Platform** | Internal BI for employees | Per named user per month |
| **Looker Embedded** | Embed in customer-facing apps | Per query, per user, or tiered |

### Cost Factors

| Factor | Impact |
|---|---|
| Number of Developer seats | Highest per-seat cost |
| Number of Standard seats | Medium cost |
| Number of Viewer seats | Lowest per-seat cost |
| Embedding volume | Scales with user count or query count |
| GCP hosting (Google Cloud core) | Additional GCP infra costs |
| BigQuery query costs | Driven by PDT rebuilds + user queries |

### 💰 Cost Optimization Tips

| Tip | Impact |
|---|---|
| Use Viewer seats for dashboard consumers | Lowest seat cost |
| Cache aggressively with datagroups | Reduce BigQuery query costs |
| Use PDTs for expensive repeated queries | Shift BQ compute to off-peak |
| Use BI Engine for repeated small queries | Avoid BQ slot consumption |
| Use aggregate awareness | Massively reduce BQ bytes scanned |
| Right-size seat types | Don't give Standard to Viewers |
| Monitor via System Activity | Identify expensive/unused content |

---

## 16. Quick Reference & Comparison Tables

### LookML Dimension Types

| Type | SQL Behavior | Example |
|---|---|---|
| `string` | Column as-is (GROUP BY) | `${TABLE}.name` |
| `number` | Numeric expression (GROUP BY) | `${TABLE}.price * 0.9` |
| `yesno` | Boolean expression → Yes/No | `${TABLE}.returned_at IS NOT NULL` |
| `date` | DATE column | `DATE(${TABLE}.created_at)` |
| `time` | TIMESTAMP/DATETIME column | `${TABLE}.created_at` |
| `tier` | CASE WHEN buckets | Numeric ranges → labels |
| `duration_*` | DATEDIFF between two timestamps | `days_to_ship` |
| `location` | lat/lon pair for map | `(${TABLE}.lat, ${TABLE}.lng)` |
| `zipcode` | ZIP for geographic map | `${TABLE}.zip` |

### LookML Measure Types

| Type | SQL Behavior | Notes |
|---|---|---|
| `count` | `COUNT(*)` | Fast; use on base table only |
| `count_distinct` | `COUNT(DISTINCT field)` | Required after one_to_many joins |
| `sum` | `SUM(field)` | Apply `value_format_name: usd` |
| `average` | `AVG(field)` | Be careful with NULLs |
| `max` / `min` | `MAX(field)` / `MIN(field)` | Useful for latest/earliest values |
| `median` | `PERCENTILE_CONT(0.5)` | May not be supported on all DBs |
| `percentile` | `PERCENTILE_CONT(n/100)` | Set `percentile: 95` |
| `list` | `STRING_AGG(field, ', ')` | Creates concatenated list |
| `number` | Arithmetic on other measures | `${sum_a} / NULLIF(${sum_b}, 0)` |

---

### Join Relationship Fanout Guide

| Relationship | Base Count | Joined Count | When to Use |
|---|---|---|---|
| `many_to_one` | ✅ Accurate | ✅ Accurate | Fact → Dimension table |
| `one_to_one` | ✅ Accurate | ✅ Accurate | Two tables at same grain |
| `one_to_many` | ⚠️ Inflated (use `count_distinct`) | ✅ Accurate | Order → Line items |
| `many_to_many` | ❌ Severely inflated | ❌ Severely inflated | **Avoid — use bridge table** |

---

### Derived Table Strategy Comparison

| Strategy | Materialized? | Build Time | SQL Required | Best For |
|---|---|---|---|---|
| **SQL DT** | ❌ (every query) | On query | ✅ Raw SQL | Simple one-off transformations |
| **NDT** | ❌ (every query) | On query | ❌ LookML only | Reusing Explore logic |
| **PDT** | ✅ (on schedule) | Async build | ✅ Raw SQL | Expensive repeated aggregations |
| **Incremental PDT** | ✅ (append) | Faster rebuild | ✅ Raw SQL | Large append-only tables |

---

### Embedding Type Comparison

| Type | Auth | Provisioning | User Control | Complexity |
|---|---|---|---|---|
| **Signed SSO** | HMAC-signed URL | Auto on-the-fly | ✅ Full | High |
| **Private** | Looker login | Manual or SAML | ✅ Full | Low |
| **Public** | None | N/A | ❌ None | Lowest |
| **Looker Components** | SDK + OAuth | Auto | ✅ Full | Highest |

---

### Datagroup Trigger Comparison

| Trigger Type | When It Fires | Best For |
|---|---|---|
| `sql_trigger` | When SQL query returns a new value | Post-ETL (most precise) |
| `interval_trigger` | Every N hours | Simple time-based refresh |
| `max_cache_age` | Fallback when trigger hasn't fired | Safety net for stale cache |

---

### Looker vs. Looker Studio Decision Guide

| Requirement | Use Looker | Use Looker Studio |
|---|---|---|
| Governed metric definitions | ✅ | ❌ |
| Row-level security | ✅ | ❌ |
| Version-controlled dashboards | ✅ | ❌ |
| Embed in customer applications | ✅ | ⚠️ Limited |
| REST API for programmatic access | ✅ | ⚠️ Limited |
| Free to use | ❌ | ✅ |
| Quick personal dashboard | ❌ (overkill) | ✅ |
| Direct Google Sheets/Drive data | ❌ | ✅ |
| No IT involvement needed | ❌ | ✅ |

---

### Key LookML Parameters Quick Reference

| Parameter | Applies To | Description |
|---|---|---|
| `connection` | Model | Database connection name |
| `include` | Model | Import view/explore files |
| `sql_table_name` | View | Base table reference |
| `derived_table` | View | Define DT/PDT |
| `primary_key: yes` | Dimension | Marks primary key field |
| `type` | Field | Field type (string, number, etc.) |
| `sql` | Field | SQL expression |
| `label` | Field | UI display name |
| `description` | Field | Tooltip text |
| `hidden: yes` | Field | Hide from field picker |
| `value_format_name` | Number field | Format name (usd, percent_2) |
| `group_label` | Field | Field picker group |
| `drill_fields` | Measure | Fields for drill-down |
| `filters` | Measure | Conditional aggregation |
| `relationship` | Join | Cardinality declaration |
| `sql_on` | Join | Join condition |
| `type` | Join | Join type (left_outer, etc.) |
| `access_filter` | Explore | Row-level security |
| `always_filter` | Explore | Default visible filter |
| `sql_always_where` | Explore | Non-removable SQL filter |
| `persist_with` | Explore | Cache policy (datagroup) |
| `aggregate_table` | Explore | Pre-aggregated rollup |
| `datagroup_trigger` | PDT | Rebuild trigger |
| `increment_key` | PDT | Incremental append column |
| `extends` | View/Explore | Inherit from base |
| `extension: required` | View | Abstract/base-only view |
| `required_access_grants` | Field/Explore | Access control grant |

---

*Generated for GCP Looker | Official Docs: [cloud.google.com/looker/docs](https://cloud.google.com/looker/docs) | LookML Reference: [docs.looker.com/reference/lookml-quick-reference](https://docs.looker.com/reference/lookml-quick-reference)*
