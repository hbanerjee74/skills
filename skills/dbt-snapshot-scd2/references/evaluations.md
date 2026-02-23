# Evaluations

Test scenarios for the dbt-snapshot-scd2 skill. Each scenario tests whether the skill produces correct, Fabric-specific snapshot guidance that goes beyond what Context7 and training data provide.

## Scenarios

### Scenario 1: datetime2 precision drift diagnosis

**Prompt**: "My dbt snapshot on Fabric keeps growing every run even though the source data hasn't changed. The snapshot table has 500K rows but the source only has 50K. What's going on and how do I fix it?"

**Expected behavior**: Claude should diagnose datetime2 precision drift as the primary suspect and provide a concrete fix:
- Identify that `datetime2(7)` (Fabric default) causes fractional-second mismatches during MERGE comparison
- Recommend casting `updated_at` to `datetime2(0)` or `datetime2(3)` in the snapshot query
- Provide the Jinja block syntax with the `cast(updated_at as datetime2(0))` pattern
- Mention that `--full-refresh` will be needed to clean up the corrupted history (with the caveat that it destroys history)
- Suggest the phantom duplicate detection test to confirm the diagnosis

**Pass criteria**:
1. Mentions `datetime2` precision or fractional-second mismatch as the root cause
2. Provides a working `cast(... as datetime2(0))` fix in the snapshot SQL

### Scenario 2: Strategy choice with unreliable updated_at

**Prompt**: "I need to build a dbt snapshot for our customer dimension table on Fabric. The table has an `updated_at` column, but it only updates when the customer logs in — not when their address or plan tier changes (those are updated by a batch process that doesn't touch `updated_at`). Which strategy should I use?"

**Expected behavior**: Claude should recommend `check` strategy with explicit column list:
- Explain that `timestamp` would miss address and plan tier changes because `updated_at` doesn't reflect those modifications
- List specific columns to include in `check_cols` (address fields, plan tier, status — business-meaningful columns)
- Explicitly exclude audit columns (`updated_at`, `created_at`, `modified_by`) from `check_cols`
- Configure the snapshot to target the source table, not a staging view
- Optionally recommend passing `updated_at` as the `updated_at` config for ordering even with check strategy

**Pass criteria**:
1. Recommends `check` strategy with reasoning about unreliable `updated_at`
2. Uses `source()` not `ref('stg_...')` in the snapshot relation

### Scenario 3: Composite key with NULL-able columns

**Prompt**: "I'm building a snapshot for a line-items table on Fabric. The unique key is (order_id, line_item_id, variant_id), but variant_id can be NULL for non-variant products. My snapshot seems to be creating duplicate rows for those products. How do I fix this?"

**Expected behavior**: Claude should identify the NULL equality problem in T-SQL MERGE and provide a fix:
- Explain that `NULL = NULL` evaluates to false in T-SQL, causing the MERGE ON clause to never match rows where `variant_id` is NULL
- Provide a solution using a synthetic composite key with ISNULL: `concat(cast(isnull(variant_id, -1) as varchar(20)), '-', ...)` or similar
- Use `isnull()` (T-SQL) not `coalesce()` or `ifnull()` for the two-argument case
- Set `unique_key` to the synthetic column name, not the list of raw columns
- Warn about choosing a sentinel value that won't collide with real data

**Pass criteria**:
1. Identifies NULL = NULL as the root cause of duplicate rows
2. Provides a T-SQL-compatible fix using `ISNULL()` (not Postgres/Snowflake syntax)

### Scenario 4: Snapshot scheduling and hard deletes

**Prompt**: "Our source system hard-deletes customer records when they close their account. We run dbt snapshots daily on Fabric. How do we track when a customer was deleted? Also, should we include snapshots in our regular dbt build?"

**Expected behavior**: Claude should address both the hard-delete tracking and the scheduling concern:
- Recommend `hard_deletes: 'new_record'` or `hard_deletes: 'invalidate'` with clear reasoning for each
- If recommending `new_record`, mention that `dbt_is_deleted` is a `VARCHAR` column on Fabric (values `'True'`/`'False'`), not `BIT`
- Advise against including snapshots in `dbt build` — recommend running them separately with `dbt snapshot` on a matching cadence
- Provide the exclude pattern: `dbt build --exclude resource_type:snapshot`
- Explain that CI/PR environments should never run snapshots (ephemeral schemas lose history)

**Pass criteria**:
1. Recommends a `hard_deletes` configuration with correct Fabric-specific data type details
2. Advises running snapshots separately from `dbt build` with the exclude pattern
