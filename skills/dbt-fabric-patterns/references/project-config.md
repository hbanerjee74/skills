# Project Configuration

dbt_project.yml patterns, selector syntax, tag strategies, and CI/CD pipeline configuration for dbt on Fabric. Focuses on the patterns that Context7 documents thinly.

## Contents
- [Selectors and Graph Operators](#selectors-and-graph-operators)
- [Tag Strategies](#tag-strategies)
- [dbt_project.yml Patterns](#dbt_projectyml-patterns)
- [CI/CD Pipeline Patterns](#cicd-pipeline-patterns)

## Selectors and Graph Operators

### Graph operator reference

| Operator | Meaning | Example |
|---|---|---|
| `+model` | All ancestors (upstream) | `dbt build --select +fct_orders` |
| `model+` | All descendants (downstream) | `dbt build --select stg_stripe__payments+` |
| `1+model` | Direct parents only | `dbt build --select 1+fct_orders` |
| `model+1` | Direct children only | `dbt build --select fct_orders+1` |
| `@model` | Ancestors + descendants + descendants' ancestors | `dbt build --select @fct_orders` |

### Common selector patterns

```bash
# Run everything downstream of a changed staging model
dbt build --select stg_stripe__payments+

# Run a mart and everything it depends on
dbt build --select +fct_orders

# Slim CI: only modified models and their children
dbt build --select state:modified+ --defer --state ./prod-manifest/

# Run all models in a directory
dbt build --select path:models/marts/finance/

# Intersection: models that are BOTH tagged nightly AND in the finance directory
dbt build --select "tag:nightly,path:models/marts/finance/"

# Union: models tagged nightly OR in the finance directory
dbt build --select "tag:nightly path:models/marts/finance/"

# Exclude specific models
dbt build --select "tag:nightly" --exclude "fct_legacy_orders"
```

### Selector gotchas

- **Space = union, comma = intersection.** `"tag:a tag:b"` runs both; `"tag:a,tag:b"` runs only models with both tags.
- **`@` is expensive.** It traverses the full graph in both directions. Use `+model+` for just ancestors and descendants without the "descendants' ancestors" expansion.
- **`state:modified+` needs a manifest.** Download the production manifest before CI runs. Missing manifest = full build.

## Tag Strategies

### Tag cascading

Tags in `dbt_project.yml` cascade to all children:

```yaml
models:
  my_project:
    +tags: ["all-models"]
    staging:
      +tags: ["staging", "hourly"]
    marts:
      +tags: ["marts"]
      finance:
        +tags: ["finance", "nightly"]
```

A model at `models/marts/finance/fct_invoices.sql` inherits: `["all-models", "marts", "finance", "nightly"]`.

### Recommended tag taxonomy

| Tag | Purpose | Applied to |
|---|---|---|
| `hourly`/`daily`/`weekly` | Schedule cadence | Models by refresh frequency |
| `critical` | Alerting priority | Models where failures page on-call |
| `pii` | Data classification | Models containing personal data |
| Domain tags (`finance`, `sales`) | Ownership/filtering | Models by business domain |

**Avoid**: tags that duplicate directory structure (if `models/marts/finance/` already implies finance, a `finance` tag is redundant unless you need cross-directory selection).

### Model-level tag override

Tags set in model config are additive — they don't replace project-level tags:

```sql
{{ config(tags=["sla-4h"]) }}
```

This model gets `["sla-4h"]` plus any tags inherited from `dbt_project.yml`.

## dbt_project.yml Patterns

### Model-level configs

Set defaults at the directory level, override at the model level:

```yaml
models:
  my_project:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: view
      +schema: intermediate
    marts:
      +materialized: table
      +schema: marts
      +grants:
        select: ['reporter_role']
```

### Meta fields

`meta` is a free-form dictionary for custom metadata. Useful for:
- Documentation: `meta: {owner: "data-team", sla: "4h"}`
- Tooling integration: `meta: {contains_pii: true}` (queryable via `dbt ls -s config.meta.contains_pii:true`)
- Elementary: `meta: {elementary: {timestamp_column: "updated_at"}}` for anomaly detection

### Packages

For Fabric projects, use these packages (not their Postgres/Snowflake equivalents):

```yaml
# packages.yml
packages:
  - package: dbt-labs/tsql_utils
    version: [">=0.10.0", "<1.0.0"]
  - package: calogica/dbt_expectations
    version: [">=0.10.0", "<1.0.0"]
  - package: dbt-labs/dbt_project_evaluator
    version: [">=0.8.0", "<1.0.0"]
  - package: elementary-data/elementary
    version: [">=0.16.0", "<1.0.0"]
```

**Critical**: `tsql-utils` replaces `dbt-utils`. Many macros have the same name but different signatures. Always check `tsql-utils` docs when using utility macros.

## CI/CD Pipeline Patterns

### Slim CI

Only build what changed:

```bash
# Download production manifest (from previous deploy)
# Then run:
dbt build --select state:modified+ --defer --state ./prod-manifest/
```

- `state:modified+` selects changed models and their downstream dependents
- `--defer` falls back to production for unmodified upstream models
- `--state` points to the production manifest directory

### Schema-only dry runs

dbt 1.8+ supports `--empty` flag for schema validation without data:

```bash
dbt run --select state:modified+ --empty
```

Creates tables with correct schemas but zero rows. Catches column type mismatches and ref() errors without full materialization.

### CI schema isolation

Use PR-specific schemas to avoid CI runs colliding:

```yaml
# profiles.yml (CI)
target:
  schema: "ci_pr_{{ env_var('PR_NUMBER') }}"
```

Clean up on PR close with a GitHub Actions workflow that drops the schema.

### Manifest lifecycle

1. **After deploy to production**: Upload manifest to artifact storage
2. **Before CI run**: Download production manifest for `--state`
3. **On PR close**: Clean up CI schema

### Linting

SQLFluff with GitHub-native annotations:

```bash
sqlfluff lint models/ --format github-annotation-native
```

Produces inline PR annotations pointing to exact lines with lint violations.
