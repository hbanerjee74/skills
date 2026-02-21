# Elementary Test Catalog

Complete reference for every Elementary test type with configuration parameters, usage patterns, and Fabric-specific notes.

## Contents
- [Anomaly Detection Tests](#anomaly-detection-tests)
- [Schema Tests](#schema-tests)
- [Shared Configuration Parameters](#shared-configuration-parameters)
- [Fabric-Specific Configuration Notes](#fabric-specific-configuration-notes)

## Anomaly Detection Tests

### `volume_anomalies`

Monitors row count over time buckets. Detects unexpected increases or decreases in table volume.

**When to use:** Bronze and gold tables where row count is a reliable health signal. Not useful on tables with highly variable volume (e.g., event tables with seasonal spikes).

```yaml
models:
  - name: fct_daily_orders
    config:
      elementary:
        timestamp_column: "order_date"
    tests:
      - elementary.volume_anomalies:
          anomaly_direction: drop
          fail_on_zero: true
          anomaly_sensitivity: 3
          training_period:
            period: day
            count: 21
          detection_period:
            period: day
            count: 2
          time_bucket:
            period: day
            count: 1
          config:
            severity: warn
```

**Parameters specific to this test:**

| Parameter | Default | Description |
|---|---|---|
| `fail_on_zero` | `false` | Fail immediately if table has zero rows (no baseline needed) |
| `anomaly_direction` | `both` | `both`, `spike`, or `drop` |

**Fabric note:** Volume anomalies work reliably on Fabric. No T-SQL compatibility issues.

---

### `column_anomalies`

Monitors specific metrics on a single column over time buckets. Place on individual columns where you know which metrics matter.

**When to use:** Silver and gold columns where you have a clear expectation of data shape. Prefer this over `all_columns_anomalies` for targeted monitoring.

```yaml
columns:
  - name: payment_amount
    tests:
      - elementary.column_anomalies:
          column_anomalies:
            - min
            - max
            - zero_count
            - null_count
          timestamp_column: "processed_at"
          anomaly_sensitivity: 2.5
          time_bucket:
            period: day
            count: 1
          config:
            severity: warn
```

**Available monitors by column type:**

| Monitor | Numeric | String | Boolean | Date |
|---|---|---|---|---|
| `null_count` | Yes | Yes | Yes | Yes |
| `null_percent` | Yes | Yes | Yes | Yes |
| `missing_count` | Yes | Yes | - | Yes |
| `missing_percent` | Yes | Yes | - | Yes |
| `min` | Yes | - | - | - |
| `max` | Yes | - | - | - |
| `average` | Yes | - | - | - |
| `standard_deviation` | Yes | - | - | - |
| `variance` | Yes | - | - | - |
| `sum` | Yes | - | - | - |
| `zero_count` | Yes | - | - | - |
| `zero_percent` | Yes | - | - | - |
| `distinct_count` | Yes | Yes | Yes | - |
| `distinct_percent` | Yes | Yes | Yes | - |
| `min_length` | - | Yes | - | - |
| `max_length` | - | Yes | - | - |
| `average_length` | - | Yes | - | - |

**Fabric note:** All monitors work on Fabric. `min_length`/`max_length` use T-SQL `LEN()` internally, which excludes trailing spaces. This matches Elementary's behavior on other platforms.

---

### `all_columns_anomalies`

Runs default monitors on ALL columns of a model. Broad sweep, high cost.

**When to use:** Silver models with <20 columns where you want an initial coverage sweep. Replace with targeted `column_anomalies` once you know which columns and monitors matter.

```yaml
models:
  - name: stg_hubspot__contacts
    config:
      elementary:
        timestamp_column: "updated_at"
    tests:
      - elementary.all_columns_anomalies:
          exclude_prefix: "_"          # skip dlt metadata columns
          exclude_regexp: ".*_raw$"    # skip unprocessed columns
          anomaly_sensitivity: 3.5     # higher = less sensitive (new deployment)
          config:
            severity: warn
```

**Parameters specific to this test:**

| Parameter | Default | Description |
|---|---|---|
| `exclude_prefix` | none | Exclude columns starting with this string |
| `exclude_regexp` | none | Exclude columns matching this regex |

**Fabric note:** On wide Fabric tables (50+ columns), this test can be slow due to the number of metric queries. Consider `column_anomalies` on specific columns instead.

---

### `freshness_anomalies`

Monitors whether the table is being updated on its expected schedule. Compares the most recent `timestamp_column` value against expected update frequency.

**When to use:** Any table with a reliable timestamp where you expect regular updates. Critical for silver tables fed by scheduled pipelines.

```yaml
models:
  - name: stg_stripe__payments
    config:
      elementary:
        timestamp_column: "created_at"
    tests:
      - elementary.freshness_anomalies:
          anomaly_sensitivity: 3
          config:
            severity: warn
```

**Fabric note:** Uses `DATEDIFF` internally, which is native T-SQL. No compatibility issues.

---

### `dimension_anomalies`

Monitors the distribution of values in a categorical column. Detects when a category appears or disappears, or when its proportion changes significantly.

**When to use:** Gold dimension tables where category distribution should be stable. Examples: `order_status`, `customer_tier`, `product_category`.

```yaml
models:
  - name: dim_customers
    config:
      elementary:
        timestamp_column: "updated_at"
    tests:
      - elementary.dimension_anomalies:
          dimensions:
            - customer_tier
            - region
          where_expression: "is_active = 1"  # T-SQL BIT, not boolean
          config:
            severity: warn
```

**Parameters specific to this test:**

| Parameter | Default | Description |
|---|---|---|
| `dimensions` | required | List of column names to monitor as dimensions |

**Fabric note:** The `where_expression` must use T-SQL syntax. Use `= 1` / `= 0` for BIT columns, not `= true` / `= false`.

---

### `event_freshness_anomalies`

Monitors the time gap between consecutive events. Detects when the pipeline stops producing events or when gaps between events grow abnormally.

**When to use:** Event-driven bronze/silver tables where you expect a continuous stream of records.

```yaml
models:
  - name: stg_events__page_views
    config:
      elementary:
        timestamp_column: "event_timestamp"
    tests:
      - elementary.event_freshness_anomalies:
          event_timestamp_column: "event_timestamp"
          update_timestamp_column: "loaded_at"
          config:
            severity: warn
```

**Fabric note:** Works without issues on Fabric. Both timestamp columns should be `datetime2`.

## Schema Tests

### `schema_changes_from_baseline`

Compares current table schema against a baseline defined in your YAML. Catches column additions, removals, and type changes.

**When to use:** Bronze tables where upstream sources can change. Also useful on gold tables that serve as API contracts.

```yaml
models:
  - name: raw_erp_customers
    columns:
      - name: customer_id
        data_type: bigint
      - name: name
        data_type: nvarchar
      - name: email
        data_type: nvarchar
      - name: created_at
        data_type: datetime2
      - name: status
        data_type: varchar
    tests:
      - elementary.schema_changes_from_baseline:
          fail_on_added: true
          enforce_types: true
          config:
            severity: error
```

**Parameters:**

| Parameter | Default | Description |
|---|---|---|
| `fail_on_added` | `false` | Fail if columns exist in table but not in baseline |
| `enforce_types` | `false` | Fail if baseline columns are missing `data_type` |

**Fabric note:** Use T-SQL data types in the baseline (`datetime2`, `bit`, `nvarchar`, not `timestamp`, `boolean`, `text`).

---

### `schema_changes`

Monitors schema changes between dbt runs. Does NOT require a baseline definition. Detects any column addition, removal, or type change since the last run.

**When to use:** When you want to detect schema drift but don't want to maintain a baseline. Less precise than `schema_changes_from_baseline` but lower maintenance.

```yaml
models:
  - name: stg_salesforce__accounts
    tests:
      - elementary.schema_changes:
          config:
            severity: warn
```

**Fabric note:** No T-SQL compatibility issues. Works by comparing metadata between runs stored in Elementary's internal tables.

---

### `exposure_schema_validity`

Validates that a model's actual schema matches the columns declared in a dbt exposure. Protects BI tool contracts.

**When to use:** Gold models that are referenced by exposures in your dbt project.

```yaml
# In your exposure definition:
exposures:
  - name: weekly_revenue_dashboard
    type: dashboard
    owner:
      name: Analytics Team
    depends_on:
      - ref('fct_weekly_revenue')

# In your model definition:
models:
  - name: fct_weekly_revenue
    tests:
      - elementary.exposure_schema_validity:
          config:
            severity: error
```

**Fabric note:** No compatibility issues. This test compares metadata only, no SQL execution against the table.

## Shared Configuration Parameters

These parameters apply to all anomaly detection tests.

### `timestamp_column`

The column used to bucket data for time-series analysis. **Always set this.** Can be configured at model level (recommended) or test level.

```yaml
# Model level (recommended -- applies to all tests on this model):
models:
  - name: my_model
    config:
      elementary:
        timestamp_column: "created_at"

# Test level (overrides model level):
    tests:
      - elementary.volume_anomalies:
          timestamp_column: "loaded_at"  # different from model default
```

### `training_period`

How much historical data to use for establishing the baseline.

| Scenario | Recommended `training_period` |
|---|---|
| Daily-loaded tables | `period: day, count: 21` (3 weeks) |
| Hourly event tables | `period: day, count: 14` (2 weeks of hourly buckets) |
| Weekly batch loads | `period: day, count: 56` (8 weeks) |
| New tables (<2 weeks of data) | `period: day, count: 7` (minimum viable) |

### `detection_period`

How far back to look for anomalies in each test run. Default: 2 days.

Set this to cover the gap between your dbt runs. If dbt runs daily, `count: 2` is safe (catches yesterday + buffer). If dbt runs hourly, `count: 1` with `period: day` is sufficient.

### `time_bucket`

The granularity for metric aggregation. Default: 1 day.

| Data pattern | Recommended `time_bucket` |
|---|---|
| Daily loads | `period: day, count: 1` |
| Hourly events | `period: hour, count: 1` |
| Weekly reports | `period: week, count: 1` |

### `anomaly_sensitivity`

Number of standard deviations from the mean to trigger an anomaly. Default: 3.

| Value | Sensitivity | Use when |
|---|---|---|
| 2 | High (more alerts) | Critical tables where false positives are acceptable |
| 3 | Medium (default) | Most tables |
| 4 | Low (fewer alerts) | Noisy tables, initial rollout period |

### `seasonality`

Account for weekly patterns (e.g., weekday vs weekend volume).

```yaml
tests:
  - elementary.volume_anomalies:
      seasonality: day_of_week  # only valid value
```

Use this when your data has clear weekday/weekend patterns (e.g., B2B transaction tables with low weekend volume).

### `ignore_small_changes`

Suppress alerts for changes below a percentage threshold:

```yaml
tests:
  - elementary.volume_anomalies:
      ignore_small_changes:
        spike_failure_percent_threshold: 10  # ignore spikes < 10%
        drop_failure_percent_threshold: 5    # but catch drops > 5%
```

### `detection_delay`

Skip the most recent time buckets to account for late-arriving data:

```yaml
tests:
  - elementary.volume_anomalies:
      detection_delay:
        period: hour
        count: 6  # skip last 6 hours
```

Use this on tables where data arrives with a known lag (common on Fabric with scheduled pipeline runs).

## Fabric-Specific Configuration Notes

### T-SQL in `where_expression`

All `where_expression` values must use T-SQL syntax:

```yaml
# CORRECT (T-SQL):
where_expression: "status = 'active' AND created_at >= DATEADD(day, -30, GETDATE())"

# WRONG (Postgres):
where_expression: "status = 'active' AND created_at >= CURRENT_DATE - INTERVAL '30 days'"
```

### BIT columns in filters

Fabric uses `BIT` (0/1), not boolean:

```yaml
# CORRECT:
where_expression: "is_active = 1"

# WRONG:
where_expression: "is_active = true"
```

### `datetime2` precision

Fabric default is `datetime2(6)`. When comparing timestamps in `where_expression`, use explicit `CAST` to avoid precision mismatches:

```yaml
where_expression: "CAST(loaded_at AS date) = CAST(GETDATE() AS date)"
```

### Elementary internal schema

Always isolate Elementary's internal tables:

```yaml
# dbt_project.yml
models:
  elementary:
    +schema: "elementary"
```

This prevents Elementary's metadata tables from mixing with your business models. On Fabric, this creates a separate `elementary` schema in your Lakehouse/Warehouse.
