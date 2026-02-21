# Snapshot Configuration Patterns

Detailed patterns for configuring dbt snapshots on Microsoft Fabric. Covers both YAML (preferred, dbt 1.9+) and Jinja block syntax, strategy-specific configurations, and Fabric-specific tuning.

## Contents
- [YAML vs Jinja Syntax](#yaml-vs-jinja-syntax)
- [Timestamp Strategy Patterns](#timestamp-strategy-patterns)
- [Check Strategy Patterns](#check-strategy-patterns)
- [Schema and Target Configuration](#schema-and-target-configuration)
- [Custom Meta Columns](#custom-meta-columns)
- [Hard Delete Handling](#hard-delete-handling)
- [Scheduling Patterns](#scheduling-patterns)
- [Performance on Fabric](#performance-on-fabric)
- [Snapshot Testing](#snapshot-testing)
- [Migration and Recovery](#migration-and-recovery)

## YAML vs Jinja Syntax

dbt 1.9+ supports YAML-based snapshot configuration. Both syntaxes are supported on Fabric.

### YAML (preferred for new snapshots)

```yaml
# snapshots/snap_customers.yml
snapshots:
  - name: snap_customers
    relation: source('crm', 'customers')
    description: "SCD2 history for customer dimension"
    config:
      schema: snapshots
      unique_key: customer_id
      strategy: timestamp
      updated_at: modified_date
```

The query lives in a separate `.sql` file only if you need transformations (like datetime2 truncation). For simple pass-through snapshots, the YAML `relation` is sufficient.

### Jinja block (legacy, still needed for inline SQL)

```sql
-- snapshots/snap_customers.sql
{% snapshot snap_customers %}
    {{ config(
        target_schema='snapshots',
        unique_key='customer_id',
        strategy='timestamp',
        updated_at='modified_date_truncated'
    ) }}

    select
        *,
        cast(modified_date as datetime2(0)) as modified_date_truncated
    from {{ source('crm', 'customers') }}
{% endsnapshot %}
```

Use Jinja block syntax when you need to:
- Cast or transform columns (datetime2 precision fix)
- Filter rows (`WHERE is_test = 0`)
- Add computed columns (surrogate keys for composite unique_key)
- Join a secondary source for enrichment (rare — prefer downstream joins)

### When to use which

| Scenario | Syntax | Reason |
|---|---|---|
| Simple pass-through | YAML | Cleaner, less error-prone |
| datetime2 truncation needed | Jinja | Requires inline SQL cast |
| Composite key with NULLs | Jinja | Requires ISNULL/CONCAT in SQL |
| Column filtering (`select specific_cols`) | Jinja | YAML snapshots all columns |
| New project, dbt 1.9+ | YAML | Modern convention |

## Timestamp Strategy Patterns

### Basic timestamp snapshot

```yaml
snapshots:
  - name: snap_orders
    relation: source('erp', 'sales_orders')
    config:
      schema: snapshots
      unique_key: order_id
      strategy: timestamp
      updated_at: last_modified_utc
```

Requirements for `updated_at`:
- Must be NOT NULL for all rows (NULLs are silently skipped)
- Must change on every meaningful data modification
- Must increase monotonically (backdated corrections cause missed changes)
- Must be a datetime-compatible type (`datetime2`, `date`, `datetimeoffset`)

### Timestamp with datetime2 precision fix

```sql
{% snapshot snap_orders %}
    {{ config(
        target_schema='snapshots',
        unique_key='order_id',
        strategy='timestamp',
        updated_at='updated_at_sec'
    ) }}

    select
        *,
        cast(last_modified_utc as datetime2(0)) as updated_at_sec
    from {{ source('erp', 'sales_orders') }}
{% endsnapshot %}
```

Precision guide for the cast:
- `datetime2(0)` — second granularity (most common, safest default)
- `datetime2(3)` — millisecond (use if source genuinely updates sub-second)
- `datetime2(7)` — 100-nanosecond (Fabric default, causes most drift issues)

**Rule**: Match the cast precision to the source system's actual update granularity, not the column's declared precision.

### Timestamp with NULL handling

```sql
{% snapshot snap_products %}
    {{ config(
        target_schema='snapshots',
        unique_key='product_id',
        strategy='timestamp',
        updated_at='safe_updated_at'
    ) }}

    select
        *,
        cast(
            isnull(updated_at, created_at) as datetime2(0)
        ) as safe_updated_at
    from {{ source('pim', 'products') }}
{% endsnapshot %}
```

Fallback chain: `updated_at` -> `created_at` -> literal epoch. Never let `updated_at` be NULL with timestamp strategy.

## Check Strategy Patterns

### Explicit column list (recommended)

```yaml
snapshots:
  - name: snap_customers
    relation: source('crm', 'customers')
    config:
      schema: snapshots
      unique_key: customer_id
      strategy: check
      check_cols:
        - status
        - address_line_1
        - address_line_2
        - city
        - state
        - postal_code
        - email
      updated_at: modified_date  # Optional but recommended for ordering
```

Note: `updated_at` is optional with `check` strategy but recommended — it provides ordering for `dbt_valid_from` instead of falling back to the snapshot run time.

### Check with all columns (use sparingly)

```yaml
snapshots:
  - name: snap_reference_data
    relation: source('mdm', 'currency_codes')
    config:
      schema: snapshots
      unique_key: currency_code
      strategy: check
      check_cols: all
```

Only use `check_cols: all` on:
- Small reference/lookup tables (<10K rows, <15 columns)
- Tables with no LOB/text columns
- Tables with no float/decimal columns (floating-point comparison causes phantom changes)

### Check strategy column selection guide

Include in `check_cols`:
- Business-meaningful fields (status, address, name, email)
- Fields that drive downstream logic or reporting

Exclude from `check_cols`:
- Audit columns (`created_at`, `updated_at`, `modified_by`) — these change on every edit regardless
- System columns (`_dlt_load_id`, `_dlt_id`, row hashes)
- Large text/LOB columns unless changes there are business-relevant
- Floating-point columns unless you've verified precision stability

## Schema and Target Configuration

### Dedicated snapshot schema

Always place snapshots in a dedicated schema, separate from models:

```yaml
# dbt_project.yml
snapshots:
  my_project:
    +schema: snapshots
```

On Fabric, this creates a `snapshots` schema in the target database. Benefits:
- Clear separation between transformation models and historical tracking
- Easier to manage permissions (snapshots are append-heavy, models are rebuild)
- Simpler `dbt build --exclude` patterns

### Snapshot directory structure

```
your_dbt_project/
  snapshots/
    snap_customers.yml        # YAML config (dbt 1.9+)
    snap_customers.sql        # Only if inline SQL needed
    snap_orders.yml
    snap_orders.sql
    snap_reference_data.yml
```

Naming convention: `snap_<source>_<entity>` or `snap_<entity>`. Be consistent within the project.

## Custom Meta Columns

Rename dbt's snapshot metadata columns to match your warehouse conventions:

```yaml
snapshots:
  - name: snap_customers
    relation: source('crm', 'customers')
    config:
      unique_key: customer_id
      strategy: timestamp
      updated_at: modified_date
      snapshot_meta_column_names:
        dbt_valid_from: valid_from
        dbt_valid_to: valid_to
        dbt_scd_id: scd_id
        dbt_updated_at: source_updated_at
```

On Fabric, custom meta column names must be valid T-SQL identifiers. Avoid reserved words (`from`, `to`, `date`, `key`). Prefix with `valid_` or `scd_` to be safe.

**Important**: Once a snapshot table exists with certain meta column names, changing them requires either:
1. Renaming the columns with ALTER TABLE (manual, outside dbt)
2. Dropping and recreating the snapshot (`--full-refresh` on snapshot)

Option 2 loses all history. Plan meta column names before first run.

## Hard Delete Handling

### Strategy comparison on Fabric

```yaml
# Option 1: Ignore (default) — deleted rows stay current forever
config:
  hard_deletes: ignore

# Option 2: Invalidate — sets dbt_valid_to on deleted rows
config:
  hard_deletes: invalidate

# Option 3: New record — inserts a "deleted" row with dbt_is_deleted flag
config:
  hard_deletes: new_record
```

### Decision guide

| Source behavior | Recommended `hard_deletes` | Reason |
|---|---|---|
| Never deletes rows | `ignore` | No action needed |
| Soft-deletes via `is_deleted` flag | `ignore` | Track via `check_cols` instead |
| Hard-deletes rows permanently | `invalidate` | Preserves history, closes the record |
| Need to know exactly when deletion happened | `new_record` | Adds explicit deletion record |

### Fabric-specific: new_record column type

When using `hard_deletes: new_record`, dbt adds a `dbt_is_deleted` column with string values (`'True'`/`'False'`). On Fabric, this is a `VARCHAR` column, not `BIT`. Downstream queries must compare strings:

```sql
-- Correct on Fabric
where dbt_is_deleted = 'False'

-- Wrong — BIT comparison
where dbt_is_deleted = 0
```

## Scheduling Patterns

### Match snapshot frequency to source update frequency

| Source update pattern | Snapshot frequency | Config |
|---|---|---|
| Real-time / streaming | Hourly or more | Consider if snapshot is the right tool |
| Hourly batch | Every 1-2 hours | Catches most changes |
| Daily batch | Daily, after source load completes | Most common pattern |
| Weekly/monthly reference data | Daily (cheap) or matching cadence | Daily is fine for small tables |

### Orchestrating snapshot runs

```bash
# Run snapshots independently of models
dbt snapshot

# Run specific snapshots
dbt snapshot --select snap_customers snap_orders

# Run all snapshots then all models (sequential)
dbt snapshot && dbt build --exclude resource_type:snapshot
```

On Fabric, snapshot runs take a MERGE lock on the target table. If models downstream of snapshots are running concurrently, they may read stale data or encounter lock contention. Run snapshots before downstream models, not in parallel.

### CI/CD considerations

- **Do NOT run snapshots in CI/PR environments** — CI schemas are ephemeral and snapshot history would be lost
- Snapshots should only run against persistent environments (dev, staging, production)
- In `dbt build` CI pipelines, exclude snapshots: `dbt build --exclude resource_type:snapshot`

## Performance on Fabric

### Snapshot query tuning

The snapshot MERGE statement on Fabric follows this pattern:
1. Execute the snapshot query (your SELECT)
2. Compare results against the existing snapshot table
3. INSERT new rows, UPDATE `dbt_valid_to` on changed rows

Bottlenecks and fixes:

| Bottleneck | Symptom | Fix |
|---|---|---|
| Large source query | Snapshot takes >5 min | Add WHERE clause to filter inactive/archived rows |
| Wide comparison (check strategy) | Slow even on small tables | Reduce `check_cols` to only business-relevant columns |
| Large snapshot table | MERGE slows over time | Add `incremental_predicates` (not officially supported for snapshots — use a pre-hook to partition old data) |
| datetime2 precision drift | Unbounded table growth | Cast `updated_at` to lower precision |
| Composite key MERGE | Slow ON clause | Use surrogate key instead of composite |

### Monitoring snapshot growth

Add a post-hook to log snapshot table size:

```yaml
snapshots:
  - name: snap_customers
    relation: source('crm', 'customers')
    config:
      unique_key: customer_id
      strategy: timestamp
      updated_at: modified_date
      post_hook:
        - "{{ log('snap_customers row count: ' ~ run_query('select count(*) from ' ~ this).columns[0].values()[0], info=true) }}"
```

If the snapshot table grows faster than expected, investigate:
1. datetime2 precision drift (phantom changes)
2. Source `updated_at` changing without actual data changes (audit column problem)
3. `check_cols` including volatile columns (audit timestamps, computed fields)

## Snapshot Testing

### Generic tests for snapshot tables

```yaml
# snapshots/snap_customers.yml (or in schema.yml)
snapshots:
  - name: snap_customers
    columns:
      - name: customer_id
        tests:
          - not_null
      - name: dbt_valid_from
        tests:
          - not_null
      - name: dbt_scd_id
        tests:
          - unique
```

### Custom test: no phantom duplicates

Detect if the snapshot is creating duplicate history records (symptom of datetime2 drift):

```sql
-- tests/assert_no_phantom_snapshot_duplicates.sql
-- This test fails if there are "change" records where no business data actually changed
with changes as (
    select
        customer_id,
        status,
        email,
        address_line_1,
        dbt_valid_from,
        lag(status) over (partition by customer_id order by dbt_valid_from) as prev_status,
        lag(email) over (partition by customer_id order by dbt_valid_from) as prev_email,
        lag(address_line_1) over (partition by customer_id order by dbt_valid_from) as prev_address
    from {{ ref('snap_customers') }}
)
select *
from changes
where status = prev_status
  and email = prev_email
  and address_line_1 = prev_address
```

If this test returns rows, you have phantom changes — likely datetime2 precision drift.

## Migration and Recovery

### Rebuilding a snapshot (losing history)

```bash
dbt snapshot --select snap_customers --full-refresh
```

**This destroys all history.** The snapshot table is dropped and recreated with only the current source state. Only use when:
- History is corrupted beyond repair (e.g., phantom duplicates from datetime2 drift)
- Schema changes make the existing table incompatible
- You're intentionally resetting

### Adding columns to an existing snapshot

Snapshots auto-add new columns from the source query. However, on Fabric:
- Existing rows will have NULL for the new column
- The new column's type is inferred from the source — verify it matches expectations
- Delta tables handle schema evolution, but MERGE statements may fail if the column order diverges significantly between source and target

### Migrating from Jinja to YAML syntax

1. Create the `.yml` file with identical configuration
2. Remove the `.sql` file (or keep it if you need inline SQL)
3. Run `dbt snapshot --select snap_name` to verify
4. The snapshot table is not affected — it continues from where it left off
