# Modeling Patterns

Snapshot strategies, incremental model patterns, ref() chains, and materialization decisions for dbt on Fabric. The patterns Context7 covers thinly or not at all.

## Contents
- [Snapshots](#snapshots)
- [Incremental Models](#incremental-models)
- [ref() Dependency Chains](#ref-dependency-chains)
- [Materialization Decisions](#materialization-decisions)

## Snapshots

dbt snapshots track slowly-changing dimensions (SCD Type 2). Key decisions:

### Timestamp vs Check strategy

| Strategy | When to use | Trade-offs |
|---|---|---|
| `timestamp` | Source has a reliable `updated_at` column | Faster — only compares one column. Misses changes if `updated_at` is unreliable |
| `check` | No reliable timestamp, or `updated_at` only reflects metadata changes | Compares all listed columns every run. Slower but catches all changes |

**Default to `timestamp`** when the source has a trustworthy `updated_at`. Use `check` as a fallback when:
- The source's `updated_at` only reflects metadata changes (not data changes)
- The source has no timestamp at all
- You need to track changes to specific columns only

### Snapshot source tables, not staging

Snapshot the raw source table, not the staging view:
- Staging views have no history — they reflect current state only
- Snapshots need to compare current vs previous state, which requires reading from the source
- Configure snapshots in a dedicated `snapshots/` directory, not in `models/`

### Fabric snapshot gotchas

- Snapshots create `dbt_valid_from` and `dbt_valid_to` columns. On Fabric, these are `datetime2` — watch for timezone assumptions.
- `dbt snapshot --select snapshot_name` runs a specific snapshot. Don't include snapshots in `dbt build` unless you want them on every run.
- Schedule snapshots on a cadence matching the source's update frequency — daily snapshots on an hourly-updated source miss intermediate states.

## Incremental Models

### Composite unique_key

When `unique_key` is a list (composite key), **all columns must be present**:

```yaml
config:
  materialized: incremental
  unique_key: ['order_id', 'line_item_id']
  incremental_strategy: merge
```

Gotchas:
- Omitting a column from the composite key causes silent duplicates — the merge matches on fewer columns than intended.
- On Fabric, composite keys work with `merge` strategy. The adapter generates a `MERGE ... ON target.col1 = source.col1 AND target.col2 = source.col2` statement.
- If your "unique key" is actually a surrogate key, use `generate_surrogate_key()` (from tsql-utils) and set a single `unique_key` instead.

### Late-arriving data

Data that arrives after the incremental cutoff timestamp is missed on subsequent runs. Patterns to handle it:

- **Lookback window**: Subtract a buffer from the cutoff. 3 days is typical for most batch sources.

```sql
{% if is_incremental() %}
  where updated_at > (select dateadd(day, -3, max(updated_at)) from {{ this }})
{% endif %}
```

- **`_dlt_load_id` bridge**: When dlt loads bronze data, it stamps each row with `_dlt_load_id`. Use this instead of business timestamps for the incremental predicate — it reflects when the row was loaded, not when the event happened.

- **Deduplication**: Late-arriving data plus lookback windows means duplicate rows. Deduplicate with `qualify`:

```sql
qualify row_number() over (
  partition by order_id
  order by updated_at desc
) = 1
```

### is_incremental() first-run behavior

`is_incremental()` returns `false` on:
- First run (table doesn't exist yet)
- After `--full-refresh`

This means the `{% if is_incremental() %}` block is skipped — the model processes ALL source data. Design your query to handle both cases:
- Without the filter: full historical load
- With the filter: incremental slice only

### Merge predicates on Fabric

The `merge` strategy generates T-SQL `MERGE` statements. Performance tips:
- Keep `unique_key` columns indexed or clustered (use Liquid Clustering on gold tables)
- Avoid wide composite keys — each additional column slows the merge join
- For high-volume tables (>10M rows), add `incremental_predicates` to pre-filter the target:

```yaml
config:
  materialized: incremental
  unique_key: 'id'
  incremental_strategy: merge
  incremental_predicates:
    - "target.load_date >= dateadd(day, -7, getdate())"
```

## ref() Dependency Chains

### Layer rules

```
source() → stg_ → int_ → fct_/dim_
```

| Model type | Can reference | Cannot reference |
|---|---|---|
| `stg_` (staging) | `source()` only | Other models |
| `int_` (intermediate) | `stg_` and other `int_` | `source()`, marts |
| `fct_`/`dim_` (marts) | `int_`, `stg_`, other marts | `source()` |

### Cross-project refs

For multi-project setups (separate dbt projects per domain):
```sql
{{ ref('finance_project', 'fct_invoices') }}
```
Requires the referenced project to be listed in `dependencies` in `dbt_project.yml`.

### Breaking circular refs

Circular references are a hard error in dbt. Common causes and fixes:
- **Two marts referencing each other** — extract shared logic into an `int_` model that both reference
- **Metric model referencing its own output** — use `{{ this }}` for self-references in incremental models only
- **Cross-domain dependencies** — use cross-project refs or agree on a shared `int_` layer

## Materialization Decisions

### Decision framework

Start with the default and promote when needed:

1. **Start with `view`** — zero storage cost, always fresh
2. **Promote to `table`** when the view is too slow (>30s query time) or referenced by 3+ downstream models
3. **Promote to `incremental`** when the table is too slow to rebuild (>10 minutes or >1M source rows)
4. **Never use `ephemeral` on Fabric** — unsupported by the dbt-fabric adapter

### Incremental vs table trade-offs

| Factor | `table` | `incremental` |
|---|---|---|
| Simplicity | Simple — full rebuild every run | Complex — must handle first run, late data, schema changes |
| Freshness | Always consistent | Depends on lookback window and schedule |
| Cost | Rebuilds everything | Processes only new/changed rows |
| Schema changes | Automatic | Requires `--full-refresh` |

**Rule of thumb**: Don't optimize to incremental until rebuild time actually hurts. Premature incremental models add complexity without measurable benefit.
