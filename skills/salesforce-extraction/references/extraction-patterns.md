# Extraction Patterns

dlt Salesforce source setup, incremental loading, soft-delete handling, reconciliation queries, and staging model conventions for Salesforce into dbt on Microsoft Fabric.

## Contents
- [Pipeline Setup](#pipeline-setup)
- [Incremental Loading Patterns](#incremental-loading-patterns)
- [Soft-Delete Handling](#soft-delete-handling)
- [Reconciliation Queries](#reconciliation-queries)
- [dbt Staging Conventions](#dbt-staging-conventions)
- [Object Dependency Graph](#object-dependency-graph)
- [Common Extraction Failures](#common-extraction-failures)

## Pipeline Setup

### Minimal pipeline

```python
import dlt
from dlt.sources.salesforce import salesforce_source

def main():
    pipeline = dlt.pipeline(
        pipeline_name="salesforce_to_fabric",
        destination="fabric",          # or your configured destination
        dataset_name="raw_salesforce",
    )

    source = salesforce_source()

    # Select objects explicitly — never extract "everything"
    load_data = source.with_resources(
        "account",
        "opportunity",
        "opportunity_line_item",
        "contact",
        "lead",
        "user",
        "campaign",
        "campaign_member",
        "task",
        "event",
    )

    info = pipeline.run(load_data)
    print(info)

if __name__ == "__main__":
    main()
```

### CPQ-extended pipeline

```python
# Add CPQ objects for orgs with Salesforce CPQ installed
load_data = source.with_resources(
    # Core objects
    "account",
    "opportunity",
    "opportunity_line_item",
    "contact",
    "user",
    # CPQ objects — use lowercase with namespace
    "sbqq__quote__c",
    "sbqq__quote_line__c",
    # Reference objects for resolution in dbt
    "record_type",
    "pricebook2",
    "pricebook_entry",
    "product2",
)
```

### Authentication patterns

**Development / sandbox** (username + password + security token):
```toml
# .dlt/secrets.toml
[sources.salesforce.credentials]
user_name = "admin@company.com.sandbox"
password = "password"
security_token = "token_from_salesforce_settings"
```

**Production** (Connected App with JWT bearer — no password rotation):
```toml
[sources.salesforce.credentials]
user_name = "integration-user@company.com"
consumer_key = "connected_app_consumer_key"
privatekey_file = "/path/to/server.key"
```

**Sandbox routing**: Salesforce sandboxes use `test.salesforce.com` as the login endpoint. Set `domain = "test"` in your dlt config or pass it via `simple-salesforce` configuration.

## Incremental Loading Patterns

### Default behavior

The dlt Salesforce source uses `SystemModstamp` as the incremental cursor with `merge` write disposition for transactional objects. On each run:

1. dlt queries Salesforce for records where `SystemModstamp > last_stored_value`
2. Results are merged into the destination (upsert by Salesforce ID)
3. The latest `SystemModstamp` from the batch is stored in dlt state for the next run

### Verifying the cursor field

After a pipeline run, check that dlt is using the correct cursor:

```python
import dlt

pipeline = dlt.pipeline(pipeline_name="salesforce_to_fabric")
state = pipeline.state

# Navigate to the incremental state for a specific resource
sf_state = state.get("sources", {}).get("salesforce", {})
print(sf_state)
# Look for last_value entries referencing SystemModstamp
```

### Full refresh schedule

Even with incremental loading, schedule periodic full refreshes to catch:
- Formula field value changes (formula definition changed in Salesforce)
- Records that fell outside the CDC window (e.g., long-running batch jobs that update SystemModstamp in bulk)
- Schema changes (new fields added to objects)

```python
import os

# Weekly full refresh via environment variable
if os.environ.get("FULL_REFRESH") == "true":
    info = pipeline.run(load_data, write_disposition="replace")
else:
    info = pipeline.run(load_data)  # incremental (default)
```

### Handling large initial loads

For initial extraction of large objects (>500k records), consider:

1. **Date windowing**: Split the initial load into date windows to avoid API timeout
```python
# Not natively supported by dlt Salesforce source — customize if needed
# Use dlt's incremental with an explicit initial_value to start from a specific date
```

2. **Bulk API activation**: For objects with >50k records, the Bulk API is more efficient. The dlt Salesforce source may not automatically switch to Bulk API — check your version and consider using `simple-salesforce`'s bulk methods for the initial load.

## Soft-Delete Handling

### The visibility problem

| API method | `IsDeleted = false` | `IsDeleted = true` | Merged records |
|---|---|---|---|
| `query()` / REST `query` | Visible | **Invisible** | **Invisible** |
| `queryAll()` / REST `queryAll` | Visible | Visible | Visible |

### dlt and soft deletes

Check whether your dlt Salesforce source version uses `queryAll()`:

1. **Extract a known-deleted record**: Delete a test record in Salesforce, run the pipeline, check if it appears with `is_deleted = true`
2. **Check the source code**: Look for `queryAll` or `query_all` in the dlt Salesforce source implementation
3. **If not supported**: Customize the source to use `queryAll`, or add a reconciliation step

### Reconciliation pattern

If your pipeline uses `query()` (not `queryAll`), add a reconciliation step to detect missing deletes:

```sql
-- dbt model: int_salesforce__deleted_opportunities
-- Detect records that disappeared from the source
WITH current_extract AS (
    SELECT id, system_modstamp
    FROM {{ source('salesforce', 'opportunity') }}
),
previous_extract AS (
    -- Use dbt snapshots or a separate tracking table
    SELECT id
    FROM {{ ref('snapshot_opportunity') }}
    WHERE dbt_valid_to IS NULL
)
SELECT
    p.id AS missing_id,
    'Possibly deleted or merged' AS reason
FROM previous_extract p
LEFT JOIN current_extract c ON p.id = c.id
WHERE c.id IS NULL
```

### Merge-deleted records

When Salesforce merges two records (e.g., duplicate Accounts), the losing record gets `IsDeleted = true` and gains a `MasterRecordId` pointing to the surviving record. Extract `MasterRecordId` to trace merges:

```sql
-- dbt staging model addition
SELECT
    id,
    master_record_id,
    is_deleted,
    CASE
        WHEN is_deleted AND master_record_id IS NOT NULL THEN 'merged'
        WHEN is_deleted AND master_record_id IS NULL THEN 'deleted'
        ELSE 'active'
    END AS record_status
FROM {{ source('salesforce', 'account') }}
```

## Reconciliation Queries

Run these in Salesforce (via Developer Console or Workbench) to validate pipeline completeness:

### Record count validation

```sql
-- Run in Salesforce: total record count including soft deletes
SELECT COUNT(Id) FROM Opportunity USING SCOPE Everything
-- Compare with warehouse count
```

### Detect CPQ presence

```sql
SELECT COUNT(*) FROM SBQQ__Quote__c
-- If > 0, CPQ is installed. Extract Quote and QuoteLine objects.
```

### Detect multi-currency

```sql
SELECT Id FROM CurrencyType LIMIT 1
-- If returns a row, multi-currency is enabled. Extract CurrencyIsoCode.
```

### Detect Record Types

```sql
SELECT SobjectType, DeveloperName, Name
FROM RecordType
WHERE SobjectType IN ('Opportunity', 'Account', 'Lead', 'Case')
  AND IsActive = true
ORDER BY SobjectType, DeveloperName
```

### Detect managed packages

```sql
SELECT NamespacePrefix, COUNT(*)
FROM EntityDefinition
WHERE NamespacePrefix != null
GROUP BY NamespacePrefix
ORDER BY COUNT(*) DESC
```

### Amount reconciliation for CPQ orgs

```sql
-- Run in Salesforce: compare Opp Amount vs CPQ NetTotal
SELECT
    o.Id,
    o.Amount,
    q.SBQQ__NetTotal__c,
    o.Amount - q.SBQQ__NetTotal__c AS variance
FROM Opportunity o
JOIN SBQQ__Quote__c q ON q.SBQQ__Opportunity2__c = o.Id
WHERE q.SBQQ__Primary__c = true
  AND o.Amount != q.SBQQ__NetTotal__c
  AND o.IsClosed = false
ORDER BY ABS(o.Amount - q.SBQQ__NetTotal__c) DESC
LIMIT 20
```

## dbt Staging Conventions

### Naming

```
stg_salesforce__opportunity.sql
stg_salesforce__account.sql
stg_salesforce__sbqq_quote.sql          -- CPQ: drop the __c suffix, keep prefix
stg_salesforce__sbqq_quote_line.sql
stg_salesforce__record_type.sql
```

### Standard staging template

```sql
-- stg_salesforce__opportunity.sql
WITH source AS (
    SELECT * FROM {{ source('salesforce', 'opportunity') }}
),

renamed AS (
    SELECT
        -- IDs
        id AS opportunity_id,
        account_id,
        owner_id,
        record_type_id,

        -- Core fields
        name AS opportunity_name,
        stage_name,
        forecast_category,
        type AS opportunity_type,
        amount,
        close_date,
        is_closed,
        is_won,

        -- CDC and audit
        created_date,
        last_modified_date,
        system_modstamp,
        is_deleted,
        CASE
            WHEN system_modstamp > last_modified_date
            THEN CAST(1 AS BIT)
            ELSE CAST(0 AS BIT)
        END AS is_system_modified,

        -- Soft delete classification
        CASE
            WHEN is_deleted = 1 AND master_record_id IS NOT NULL THEN 'merged'
            WHEN is_deleted = 1 THEN 'deleted'
            ELSE 'active'
        END AS record_status,

        -- dlt metadata
        _dlt_load_id,
        _dlt_id

    FROM source
)

SELECT * FROM renamed
```

### CPQ staging template

```sql
-- stg_salesforce__sbqq_quote.sql
WITH source AS (
    SELECT * FROM {{ source('salesforce', 'sbqq__quote__c') }}
),

renamed AS (
    SELECT
        id AS quote_id,
        sbqq__opportunity2__c AS opportunity_id,
        sbqq__primary__c AS is_primary_quote,
        sbqq__net_total__c AS net_total,
        sbqq__list_total__c AS list_total,
        sbqq__customer_discount__c AS customer_discount_pct,
        sbqq__status__c AS quote_status,
        sbqq__start_date__c AS start_date,
        sbqq__end_date__c AS end_date,
        sbqq__subscription_term__c AS subscription_term_months,
        created_date,
        system_modstamp,
        is_deleted,
        _dlt_load_id

    FROM source
)

SELECT * FROM renamed
```

## Object Dependency Graph

Extraction order matters for referential integrity in the warehouse. Extract in dependency order:

```
Level 0 (no dependencies):
  User, UserRole, RecordType, Product2, Pricebook2, CurrencyType

Level 1 (depends on Level 0):
  Account, Lead, Campaign, PricebookEntry

Level 2 (depends on Level 1):
  Contact, Opportunity, CampaignMember

Level 3 (depends on Level 2):
  OpportunityLineItem, OpportunityContactRole, Task, Event
  SBQQ__Quote__c (depends on Opportunity)

Level 4 (depends on Level 3):
  SBQQ__QuoteLine__c (depends on Quote)
```

dlt handles this ordering internally when you load all resources in a single `pipeline.run()` call. If you split across multiple runs, respect this order to avoid foreign key violations in dbt tests.

## Common Extraction Failures

| Symptom | Likely cause | Fix |
|---|---|---|
| Missing columns in warehouse | FLS restricts integration user | Grant field-level Read access via Permission Set |
| Record count lower than Salesforce | `query()` excluding soft deletes | Switch to `queryAll()` or verify dlt source behavior |
| Amount values don't match Salesforce reports | CPQ overrides, multi-currency, or Record Type mixing | Extract CPQ objects, CurrencyIsoCode, RecordTypeId |
| Stale formula field values | Formula depends on cross-object fields or `TODAY()` | Schedule periodic full refresh |
| `INVALID_FIELD` error on CPQ fields | CPQ not installed in target sandbox | Check namespace: `SELECT COUNT(*) FROM SBQQ__Quote__c` |
| `REQUEST_LIMIT_EXCEEDED` | Too many API calls in 24-hour window | Switch to Bulk API for large objects, reduce polling frequency |
| Duplicate records in warehouse | Missing `unique_key` in dbt incremental, or ID collision | Use Salesforce `Id` as `unique_key` in dbt incremental config |
