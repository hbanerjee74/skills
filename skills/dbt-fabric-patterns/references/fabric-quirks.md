# Fabric-Specific Quirks

T-SQL patterns, Delta table behaviors, Liquid Clustering, authentication, and tsql-utils macro mapping. The Fabric-specific knowledge that diverges from Snowflake/Postgres defaults.

## Contents
- [Authentication](#authentication)
- [Delta Table Behaviors](#delta-table-behaviors)
- [Liquid Clustering](#liquid-clustering)
- [SQL Analytics Endpoint](#sql-analytics-endpoint)
- [tsql-utils Macro Mapping](#tsql-utils-macro-mapping)
- [T-SQL Syntax Divergences](#t-sql-syntax-divergences)
- [Lakehouse Architecture](#lakehouse-architecture)

## Authentication

**ServicePrincipal only.** SQL authentication is not supported on Fabric.

Profile configuration:
```yaml
# profiles.yml
my_project:
  target: dev
  outputs:
    dev:
      type: fabric
      driver: "ODBC Driver 18 for SQL Server"
      server: "<workspace>.datawarehouse.fabric.microsoft.com"
      database: "<lakehouse-name>"
      schema: "dbo"
      authentication: ServicePrincipal
      tenant_id: "{{ env_var('AZURE_TENANT_ID') }}"
      client_id: "{{ env_var('AZURE_CLIENT_ID') }}"
      client_secret: "{{ env_var('AZURE_CLIENT_SECRET') }}"
```

For CI/CD: use OIDC federation instead of client secrets. Separate credentials for `refs/heads/main` (deploy) vs `pull_request` (CI).

## Delta Table Behaviors

All tables on Fabric are Delta format. Implications:

- **Schema evolution**: Adding columns to an incremental model requires `--full-refresh` unless `merge_update_columns` is configured to list specific columns.
- **Time travel**: Delta tables support `DESCRIBE HISTORY` for audit trails. Not directly accessible from dbt, but useful for debugging via SQL analytics endpoint.
- **VACUUM**: Fabric handles Delta table maintenance (OPTIMIZE, VACUUM) automatically. Don't run manual VACUUM operations.
- **Small file problem**: Frequent small writes create many small Parquet files. Fabric auto-compacts, but heavy incremental models may benefit from periodic `--full-refresh` to consolidate files.

## Liquid Clustering

Fabric uses Liquid Clustering instead of Hive-style partitioning on silver/gold tables.

### When to use

- Tables >100M rows with common filter patterns
- Queries that frequently filter on the same 1-3 columns
- Gold/mart tables serving BI tools with predictable query patterns

### Configuration

Liquid Clustering is set via Fabric, not dbt. Use `ALTER TABLE` in a post-hook or configure it via the Fabric UI:

```sql
-- In a dbt post-hook or Fabric notebook
ALTER TABLE {{ this }} SET CLUSTER BY (customer_id, order_date)
```

### Clustering column selection

- Choose columns used in `WHERE` clauses, `JOIN` conditions, or `GROUP BY`
- Max 3-4 columns — more columns reduce clustering effectiveness
- High-cardinality columns (like `customer_id`) cluster better than low-cardinality (like `status`)
- Order matters — put the most frequently filtered column first

## SQL Analytics Endpoint

The SQL analytics endpoint is **read-only**. Critical implications:

- **dbt writes** go through the warehouse endpoint (not the analytics endpoint)
- **BI tools** connect to the analytics endpoint for reads
- **No INSERT/UPDATE/DELETE** via the analytics endpoint — this includes ad-hoc fixes
- **Schema changes** (ALTER TABLE) are not supported via the analytics endpoint
- **Views created by dbt** are accessible via both endpoints

If a query fails with a write operation error, check which endpoint you're connected to.

## tsql-utils Macro Mapping

Common macros and their tsql-utils equivalents:

| dbt-utils macro | tsql-utils equivalent | Notes |
|---|---|---|
| `generate_surrogate_key()` | `generate_surrogate_key()` | Same name, same behavior. **Don't use** the deprecated `surrogate_key()` |
| `star()` | `star()` | Same — selects all columns except specified ones |
| `pivot()` | `pivot()` | Same signature |
| `unpivot()` | `unpivot()` | Same signature |
| `date_spine()` | `date_spine()` | Same — generates a date series |
| `datediff()` | Use T-SQL `DATEDIFF()` directly | tsql-utils doesn't wrap this — use native T-SQL |
| `dateadd()` | Use T-SQL `DATEADD()` directly | Same — use native T-SQL |
| `last_day()` | `last_day()` | Available in tsql-utils |
| `safe_cast()` | `TRY_CAST()` directly | Use T-SQL native `TRY_CAST` instead |

### Key differences from dbt-utils

- **String functions**: T-SQL uses `LEN()` not `LENGTH()`, `CHARINDEX()` not `POSITION()`
- **Date functions**: T-SQL uses `GETDATE()` not `CURRENT_TIMESTAMP`, `DATEPART()` not `EXTRACT()`
- **Null handling**: T-SQL uses `ISNULL()` not `COALESCE()` for two-argument null replacement (though `COALESCE` works for multi-argument)
- **Boolean**: T-SQL has no native boolean type. Use `BIT` (0/1) or `VARCHAR` ('true'/'false')

## T-SQL Syntax Divergences

Patterns that trip up LLMs trained primarily on Postgres/Snowflake SQL:

| Pattern | Postgres/Snowflake | T-SQL (Fabric) |
|---|---|---|
| String concatenation | `\|\|` | `+` or `CONCAT()` |
| LIMIT | `LIMIT 10` | `TOP 10` (in SELECT) or `OFFSET/FETCH` |
| Type casting | `::type` | `CAST(x AS type)` or `TRY_CAST(x AS type)` |
| Regex | `~`, `REGEXP` | `LIKE` with wildcards, or `PATINDEX()` |
| Array operations | `ARRAY_AGG`, `UNNEST` | `STRING_AGG()` for aggregation; no native arrays |
| CTE materialization | Optimizer decides | CTEs may be re-evaluated — use temp tables for expensive CTEs referenced multiple times |
| Window function qualify | `QUALIFY` | Not supported — wrap in subquery with `WHERE rn = 1` |
| Interval arithmetic | `INTERVAL '3 days'` | `DATEADD(day, 3, date_column)` |
| Null-safe equality | `IS NOT DISTINCT FROM` | `ISNULL(a, sentinel) = ISNULL(b, sentinel)` or full `CASE` expression |

### QUALIFY workaround

T-SQL doesn't support `QUALIFY`. Use a subquery pattern instead:

```sql
-- Postgres/Snowflake (won't work on Fabric):
-- SELECT * FROM orders QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) = 1

-- T-SQL equivalent:
SELECT * FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
  FROM orders
) ranked
WHERE rn = 1
```

## Lakehouse Architecture

### /Tables vs /Files

| Path | Format | Discoverable | Use for |
|---|---|---|---|
| `/Tables` | Delta (managed) | Auto-discovered by SQL analytics | dbt models, queryable data |
| `/Files` | Any (raw) | Not auto-discovered | Raw files, dlt landing zone |

- **dbt targets `/Tables`** by default — models are created as Delta tables
- **dlt lands in `/Files`** (or `/Tables` with Delta format) — configure dlt's destination accordingly
- **Shortcuts** enable cross-lakehouse references without data movement. Use separate lakehouses per medallion layer, linked by Shortcuts.

### Medallion architecture on Fabric

```
Bronze Lakehouse          Silver Lakehouse         Gold Lakehouse
(dlt landing zone)        (dbt staging + int)      (dbt marts)
├── Tables/               ├── Tables/              ├── Tables/
│   └── raw_*             │   ├── stg_*            │   ├── fct_*
└── Files/                │   └── int_*            │   └── dim_*
    └── raw uploads       └── Shortcuts/           └── Shortcuts/
                              └── → Bronze.Tables      └── → Silver.Tables
```

Each lakehouse has its own SQL analytics endpoint. Shortcuts make cross-layer data accessible without copying.
