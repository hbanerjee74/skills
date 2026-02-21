# Evaluations

Test scenarios for the dbt-fabric-patterns skill. Each scenario tests whether the skill produces correct, Fabric-specific guidance.

## Scenarios

### Scenario 1: Incremental model with composite key

**Prompt**: "Write a dbt incremental model for order line items on Fabric. The table has order_id and line_item_id as the composite key, and I need to handle late-arriving data."

**Expected behavior**: Claude should produce a model with:
- `unique_key: ['order_id', 'line_item_id']` (list, not single column)
- `incremental_strategy: merge`
- A lookback window in the `is_incremental()` block (e.g., `dateadd(day, -3, ...)`)
- Deduplication via subquery with `ROW_NUMBER()` (not `QUALIFY` — unsupported on Fabric)
- T-SQL syntax throughout (no Postgres/Snowflake idioms)

**Pass criteria**:
1. Composite `unique_key` is a list with both columns
2. Uses `DATEADD()` (T-SQL) not `INTERVAL` or `date_add()` (Spark/Postgres)

### Scenario 2: Snapshot strategy decision

**Prompt**: "I have a customers table in our source system. The updated_at column only changes when the customer's profile metadata is edited, not when their address or status changes. Should I use timestamp or check strategy for my dbt snapshot?"

**Expected behavior**: Claude should recommend `check` strategy because:
- `updated_at` is unreliable for detecting all changes (only reflects metadata edits)
- Should list the specific columns to check (`address`, `status`, etc.)
- Should mention that `check` is slower but catches all changes
- Should configure the snapshot pointing at the source table, not a staging model

**Pass criteria**:
1. Recommends `check` strategy with explicit reasoning about unreliable `updated_at`
2. Snapshot targets the source table, not `stg_` model

### Scenario 3: CI/CD pipeline setup

**Prompt**: "Set up a GitHub Actions CI pipeline for our dbt-fabric project. We want to lint SQL, run slim CI on PRs, and do a full deploy on merge to main."

**Expected behavior**: Claude should produce a pipeline with:
- SQLFluff with `--format github-annotation-native` for inline PR annotations
- `dbt build --select state:modified+ --defer --state ./prod-manifest/` for slim CI
- `--empty` flag option for schema-only validation
- OIDC federation for Azure auth (not client secrets in CI)
- PR-specific schema (`ci_pr_{{ PR_NUMBER }}`) with cleanup on PR close
- Manifest upload after production deploy

**Pass criteria**:
1. Uses `dbt build` (not separate `dbt run` + `dbt test`)
2. Includes manifest lifecycle (upload after deploy, download before CI)

### Scenario 4: Materialization decision

**Prompt**: "I have an intermediate model that joins 3 staging tables and is referenced by 5 mart models. It takes about 45 seconds to query. What materialization should I use on Fabric?"

**Expected behavior**: Claude should recommend `table` (not `view` or `ephemeral`) because:
- Referenced by 5 downstream models (>3 threshold)
- 45s query time is significant when multiplied by 5 downstream consumers
- `ephemeral` is unsupported on Fabric
- `incremental` is overkill for an intermediate model unless source volume is very large

**Pass criteria**:
1. Recommends `table` with reasoning about downstream fan-out
2. Explicitly states `ephemeral` is unsupported on Fabric
