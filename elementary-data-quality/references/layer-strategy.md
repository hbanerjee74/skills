# Test Placement by Medallion Layer

Detailed strategy for placing Elementary tests at each layer of the medallion architecture, with complete YAML examples and decision rationale.

## Contents
- [Overview](#overview)
- [Bronze Layer](#bronze-layer)
- [Silver Layer](#silver-layer)
- [Gold Layer](#gold-layer)
- [Cross-Layer Patterns](#cross-layer-patterns)

## Overview

### Guiding principle

Place tests where they catch problems closest to the source, but only where the signal-to-noise ratio is high enough to act on.

| Layer | Primary concern | Test focus | Severity default |
|---|---|---|---|
| Bronze | Did the data arrive? Did the shape change? | Volume, schema | `error` for schema; `warn` for volume |
| Silver | Is the cleaned data within expected ranges? | Column anomalies, freshness | `warn` (promoting to `error`) |
| Gold | Are consumer contracts intact? | Exposure validity, dimension stability | `error` for exposure tests |

### What NOT to test at each layer

- **Bronze**: Don't run `column_anomalies` -- raw data is messy by nature, and most column-level issues are noise.
- **Silver**: Don't run `schema_changes_from_baseline` -- you control the schema via dbt models. Schema tests belong on sources you don't control.
- **Gold**: Don't run `all_columns_anomalies` -- gold tables should have targeted, specific tests. Broad sweeps indicate you haven't defined what matters.

## Bronze Layer

Bronze tables are raw ingestion targets. You don't control the upstream schema or volume. Tests here detect ingestion failures and upstream changes.

### Required tests

#### `schema_changes_from_baseline`

Catches when upstream sources add, remove, or retype columns. This is your early warning system for breaking changes.

```yaml
# models/bronze/schema.yml
version: 2

models:
  - name: raw_stripe_payments
    description: "Raw payments from Stripe API via dlt"
    columns:
      - name: id
        data_type: varchar
      - name: amount
        data_type: bigint
      - name: currency
        data_type: varchar
      - name: status
        data_type: varchar
      - name: created
        data_type: datetime2
      - name: customer_id
        data_type: varchar
      - name: _dlt_load_id
        data_type: varchar
      - name: _dlt_id
        data_type: varchar
    tests:
      - elementary.schema_changes_from_baseline:
          fail_on_added: true
          enforce_types: true
          config:
            severity: error
          tags: ["elementary", "bronze"]
```

**Why `fail_on_added: true`:** New columns from upstream may contain PII or break downstream assumptions. Better to fail and review than silently pass through.

**Why `enforce_types: true`:** A column changing from `bigint` to `varchar` will break downstream `CAST` operations. Catch it here.

#### `volume_anomalies`

Catches empty loads, duplicate loads, and ingestion pipeline failures.

```yaml
  - name: raw_hubspot_contacts
    description: "Raw contacts from HubSpot API via dlt"
    config:
      elementary:
        timestamp_column: "_dlt_load_id"  # use dlt's load timestamp
    tests:
      - elementary.volume_anomalies:
          fail_on_zero: true
          anomaly_direction: both
          time_bucket:
            period: day
            count: 1
          training_period:
            period: day
            count: 21
          config:
            severity: error  # zero rows is always an error
          tags: ["elementary", "bronze"]
```

**`timestamp_column` for dlt tables:** dlt adds `_dlt_load_id` to every table. This is a string identifier, not a timestamp. For time-bucketed anomaly detection, you need a proper datetime column. Options:

1. If the source has a timestamp (e.g., `created_at`), use that
2. If using dlt with `_dlt_load_id`, consider adding a `loaded_at` column in your dlt resource
3. As a last resort, use `volume_anomalies` without `timestamp_column` (compares total counts, less precise)

### Optional tests

#### `schema_changes` (no baseline)

Use when you want to detect drift but can't maintain a baseline (e.g., 50+ bronze tables):

```yaml
  - name: raw_erp_inventory
    tests:
      - elementary.schema_changes:
          config:
            severity: warn
          tags: ["elementary", "bronze"]
```

### Complete bronze layer example

```yaml
version: 2

models:
  - name: raw_stripe_payments
    config:
      elementary:
        timestamp_column: "created"
    columns:
      - name: id
        data_type: varchar
      - name: amount
        data_type: bigint
      - name: currency
        data_type: varchar
      - name: status
        data_type: varchar
      - name: created
        data_type: datetime2
    tests:
      - elementary.schema_changes_from_baseline:
          fail_on_added: true
          enforce_types: true
          config:
            severity: error
          tags: ["elementary", "bronze"]
      - elementary.volume_anomalies:
          fail_on_zero: true
          anomaly_direction: both
          config:
            severity: error
          tags: ["elementary", "bronze"]

  - name: raw_hubspot_contacts
    config:
      elementary:
        timestamp_column: "updated_at"
    columns:
      - name: contact_id
        data_type: bigint
      - name: email
        data_type: nvarchar
      - name: updated_at
        data_type: datetime2
    tests:
      - elementary.schema_changes_from_baseline:
          enforce_types: true
          config:
            severity: error
          tags: ["elementary", "bronze"]
      - elementary.volume_anomalies:
          fail_on_zero: true
          config:
            severity: error
          tags: ["elementary", "bronze"]
```

## Silver Layer

Silver tables are cleaned and typed. You control the schema. Tests here validate data quality after transformation.

### Required tests

#### `column_anomalies` (targeted)

Monitor specific columns with explicit monitor lists:

```yaml
# models/silver/schema.yml
version: 2

models:
  - name: stg_stripe__payments
    config:
      elementary:
        timestamp_column: "created_at"
    columns:
      - name: payment_id
        tests:
          - elementary.column_anomalies:
              column_anomalies:
                - null_count
                - distinct_count
              config:
                severity: warn
              tags: ["elementary", "silver"]
      - name: amount_cents
        tests:
          - elementary.column_anomalies:
              column_anomalies:
                - min
                - max
                - zero_count
                - null_count
              anomaly_sensitivity: 2.5
              config:
                severity: warn
              tags: ["elementary", "silver"]
      - name: currency_code
        tests:
          - elementary.column_anomalies:
              column_anomalies:
                - null_count
                - distinct_count
              config:
                severity: warn
              tags: ["elementary", "silver"]
```

#### `freshness_anomalies`

Ensures the table is being updated on schedule:

```yaml
  - name: stg_stripe__payments
    config:
      elementary:
        timestamp_column: "created_at"
    tests:
      - elementary.freshness_anomalies:
          config:
            severity: warn
          tags: ["elementary", "silver"]
```

### Optional tests

#### `all_columns_anomalies` (narrow tables only)

For silver tables with fewer than 20 columns where you want initial broad coverage:

```yaml
  - name: stg_hubspot__contacts
    config:
      elementary:
        timestamp_column: "updated_at"
    tests:
      - elementary.all_columns_anomalies:
          exclude_prefix: "_"
          anomaly_sensitivity: 3.5
          config:
            severity: warn
          tags: ["elementary", "silver"]
```

**Migration path:** Start with `all_columns_anomalies`, review the Elementary report after 2 weeks, then replace with targeted `column_anomalies` on the columns that matter.

### Complete silver layer example

```yaml
version: 2

models:
  - name: stg_stripe__payments
    description: "Cleaned Stripe payments"
    config:
      elementary:
        timestamp_column: "created_at"
    tests:
      - elementary.freshness_anomalies:
          config:
            severity: warn
          tags: ["elementary", "silver"]
    columns:
      - name: payment_id
        tests:
          - elementary.column_anomalies:
              column_anomalies:
                - null_count
                - distinct_count
              config:
                severity: warn
              tags: ["elementary", "silver"]
      - name: amount_cents
        tests:
          - elementary.column_anomalies:
              column_anomalies:
                - min
                - max
                - zero_count
              config:
                severity: warn
              tags: ["elementary", "silver"]

  - name: stg_hubspot__contacts
    description: "Cleaned HubSpot contacts"
    config:
      elementary:
        timestamp_column: "updated_at"
    tests:
      - elementary.freshness_anomalies:
          config:
            severity: warn
          tags: ["elementary", "silver"]
      - elementary.all_columns_anomalies:
          exclude_prefix: "_"
          anomaly_sensitivity: 3.5
          config:
            severity: warn
          tags: ["elementary", "silver"]
```

## Gold Layer

Gold tables are consumer-facing marts. Tests here protect the contract between your data platform and its consumers (BI tools, APIs, downstream teams).

### Required tests

#### `exposure_schema_validity`

Validates that the model's schema matches what exposures declare:

```yaml
# models/gold/schema.yml
version: 2

models:
  - name: fct_weekly_revenue
    description: "Weekly revenue aggregation for finance dashboard"
    tests:
      - elementary.exposure_schema_validity:
          config:
            severity: error  # broken BI dashboard = immediate action
          tags: ["elementary", "gold"]
```

**Prerequisite:** You must have exposures defined in your dbt project:

```yaml
# models/exposures.yml
exposures:
  - name: finance_dashboard
    type: dashboard
    owner:
      name: Finance Team
      email: finance@company.com
    depends_on:
      - ref('fct_weekly_revenue')
      - ref('fct_monthly_costs')
    description: "Power BI dashboard for weekly financial reporting"
```

#### `dimension_anomalies`

Monitors category distribution stability on dimension tables:

```yaml
  - name: dim_customers
    config:
      elementary:
        timestamp_column: "updated_at"
    tests:
      - elementary.dimension_anomalies:
          dimensions:
            - customer_tier
            - region
            - acquisition_channel
          where_expression: "is_active = 1"
          config:
            severity: warn
          tags: ["elementary", "gold"]
```

#### `volume_anomalies` (on fact tables)

Fact tables should have predictable volume. A sudden drop means missing data in reports:

```yaml
  - name: fct_daily_orders
    config:
      elementary:
        timestamp_column: "order_date"
    tests:
      - elementary.volume_anomalies:
          fail_on_zero: true
          anomaly_direction: drop
          seasonality: day_of_week  # weekday/weekend patterns
          config:
            severity: error
          tags: ["elementary", "gold"]
```

### Complete gold layer example

```yaml
version: 2

models:
  - name: fct_daily_orders
    description: "Daily order aggregation"
    config:
      elementary:
        timestamp_column: "order_date"
    tests:
      - elementary.volume_anomalies:
          fail_on_zero: true
          anomaly_direction: drop
          seasonality: day_of_week
          config:
            severity: error
          tags: ["elementary", "gold"]
      - elementary.exposure_schema_validity:
          config:
            severity: error
          tags: ["elementary", "gold"]

  - name: fct_weekly_revenue
    description: "Weekly revenue for finance dashboard"
    config:
      elementary:
        timestamp_column: "revenue_week"
    tests:
      - elementary.volume_anomalies:
          fail_on_zero: true
          anomaly_direction: drop
          time_bucket:
            period: week
            count: 1
          config:
            severity: error
          tags: ["elementary", "gold"]
      - elementary.exposure_schema_validity:
          config:
            severity: error
          tags: ["elementary", "gold"]

  - name: dim_customers
    description: "Customer dimension table"
    config:
      elementary:
        timestamp_column: "updated_at"
    tests:
      - elementary.dimension_anomalies:
          dimensions:
            - customer_tier
            - region
          where_expression: "is_active = 1"
          config:
            severity: warn
          tags: ["elementary", "gold"]
    columns:
      - name: customer_id
        tests:
          - elementary.column_anomalies:
              column_anomalies:
                - null_count
                - distinct_count
              config:
                severity: error
              tags: ["elementary", "gold"]

  - name: dim_products
    description: "Product dimension table"
    config:
      elementary:
        timestamp_column: "updated_at"
    tests:
      - elementary.dimension_anomalies:
          dimensions:
            - category
            - subcategory
          config:
            severity: warn
          tags: ["elementary", "gold"]
```

## Cross-Layer Patterns

### Tagging strategy

Use consistent tags to run tests selectively:

```bash
# Run all Elementary tests
dbt test --select tag:elementary

# Run only bronze-layer tests (fast, run on every dbt build)
dbt test --select tag:elementary tag:bronze

# Run silver tests (run on daily builds)
dbt test --select tag:elementary tag:silver

# Run gold tests (run before BI refresh)
dbt test --select tag:elementary tag:gold
```

### Severity escalation across layers

| Layer | Default severity | Escalate to `error` when |
|---|---|---|
| Bronze | `error` for schema, `error` for fail_on_zero | Always `error` -- bronze failures break everything downstream |
| Silver | `warn` | After 2 weeks of stable baseline and tuned sensitivity |
| Gold | `error` for exposure tests, `warn` for anomalies | After confirming test is not overly sensitive to legitimate variation |

### Monitoring without `timestamp_column`

Some tables genuinely lack a timestamp (reference tables, static lookups). For these:

- Use `volume_anomalies` without `timestamp_column` -- compares total row count between runs
- Use `schema_changes_from_baseline` -- no timestamp needed
- Do NOT use `column_anomalies` or `freshness_anomalies` -- these need time buckets to be meaningful

```yaml
  - name: ref_country_codes
    tests:
      - elementary.schema_changes_from_baseline:
          config:
            severity: error
          tags: ["elementary", "gold"]
      - elementary.volume_anomalies:
          fail_on_zero: true
          config:
            severity: error
          tags: ["elementary", "gold"]
```
