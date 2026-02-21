# Field Semantics

Salesforce field semantic traps that produce silently wrong data in extraction pipelines. Focuses on behaviors that are not obvious from field names or API documentation.

## Contents
- [System Fields](#system-fields)
- [Formula Fields](#formula-fields)
- [Polymorphic Lookups](#polymorphic-lookups)
- [Currency and Multi-Currency](#currency-and-multi-currency)
- [Field-Level Security Filtering](#field-level-security-filtering)
- [CPQ Field Hierarchy](#cpq-field-hierarchy)
- [Picklist Coupling Traps](#picklist-coupling-traps)

## System Fields

### Timestamp fields

Every Salesforce record has four system timestamp fields:

| Field | Updated by user edit | Updated by system process | Indexed | Use for CDC |
|---|---|---|---|---|
| `CreatedDate` | Set once at creation | Never changes | Yes | No |
| `LastModifiedDate` | Yes | No | No | **No** |
| `SystemModstamp` | Yes | Yes | Yes | **Yes** |
| `LastActivityDate` | No — updated when Tasks/Events are logged | Indirectly | No | No |

**Critical detail**: `SystemModstamp` is always >= `LastModifiedDate`. When they differ, a system process (workflow rule, Process Builder, trigger, batch Apex, or Salesforce internal process) updated the record. The delta between these two timestamps is your blind spot if you use `LastModifiedDate` for CDC.

**Extraction rule**: Always extract both `SystemModstamp` and `LastModifiedDate`. In dbt, create a derived column `is_system_modified` as `SystemModstamp > LastModifiedDate`. This flags records that were updated by automation — useful for audit and debugging.

### The `LastActivityDate` trap

`LastActivityDate` on Account and Opportunity reflects the date of the most recent closed Task or upcoming Event. It is **not** a record modification timestamp. Pipelines that use it as a CDC cursor will:
- Miss records with no Activities
- Process the same record repeatedly as Activities are logged
- Never capture the record if only field data changes (no Activity)

### `IsDeleted` and the Recycle Bin

`IsDeleted` is a system field present on all queryable objects. Key behaviors:
- Standard `query()` / REST `query` endpoint: `IsDeleted` records are **invisible** — they are excluded from results entirely, not just filtered
- `queryAll()` / REST `queryAll` endpoint: Returns all records including `IsDeleted = true`
- Records remain in the Recycle Bin for 15 days, then are hard-deleted
- Hard-deleted records are gone permanently — `queryAll()` cannot retrieve them
- Some objects have a `MasterRecordId` field populated on merge-deleted records, pointing to the surviving record

**dbt pattern**: In staging models, add a column `_is_soft_deleted` from the source `is_deleted` field (dlt lowercases it). In mart models, filter `WHERE NOT _is_soft_deleted` for current-state reporting, but keep deleted records in a separate audit model.

## Formula Fields

Formula fields are computed server-side on read. Key extraction behaviors:

- **Included in API responses**: Formula fields are returned by `query()` and `queryAll()` with their computed values at query time
- **Not back-updated in warehouses**: If a formula's definition changes in Salesforce, previously extracted values reflect the old formula. The warehouse does not retroactively update.
- **Cannot be used as CDC cursors**: Formula fields are not writable, have no modification timestamp, and are not indexed
- **Cross-object formulas**: A formula on Opportunity can reference Account fields. If the Account field changes, the Opportunity formula value changes, but the Opportunity's `SystemModstamp` does NOT update.

**What goes wrong**: A formula field `Opportunity.Days_Since_Created__c` is `TODAY() - CreatedDate`. Extracting this daily gives different values each day for the same record, but the record's `SystemModstamp` never changes. An incremental pipeline will extract the record once and never update the formula value.

**Rule**: For volatile formula fields (those referencing `TODAY()`, `NOW()`, or cross-object fields), either:
1. Recalculate them in dbt using the base fields
2. Schedule periodic full refreshes for objects with volatile formulas

## Polymorphic Lookups

Some Salesforce lookup fields can point to multiple object types:

| Field | On Object | Can point to |
|---|---|---|
| `WhoId` | Task, Event | Contact, Lead |
| `WhatId` | Task, Event | Account, Opportunity, Campaign, Case, custom objects |
| `OwnerId` | Most objects | User, Group (Queue) |
| `RelatedToId` | EmailMessage | Multiple objects |

**Extraction impact**: A polymorphic lookup stores a Salesforce ID. The first 3 characters of the ID encode the object type (e.g., `003` = Contact, `00Q` = Lead). The API returns just the ID — you must resolve the object type yourself.

**dbt pattern**: In staging, add a derived column using the ID prefix:
```sql
CASE
  WHEN LEFT(who_id, 3) = '003' THEN 'Contact'
  WHEN LEFT(who_id, 3) = '00Q' THEN 'Lead'
  ELSE 'Unknown'
END AS who_type
```

Or extract the `Task` / `Event` object with the `Who.Type` relationship field if your extraction supports it.

## Currency and Multi-Currency

### Single-currency orgs

Straightforward. All currency fields are in the org's default currency. No conversion needed.

### Multi-currency orgs

When multi-currency is enabled:
- Every record with currency fields gains a `CurrencyIsoCode` field
- Salesforce stores the **original currency amount** in the standard field (e.g., `Amount`)
- A parallel `*Converted` field (e.g., `ConvertedAmount`) holds the value in the corporate currency, converted using Salesforce's exchange rate table at the time of the last save
- `DatedConversionRate` (if Advanced Currency Management is enabled) provides historical rates

**Extraction trap**: If you extract only `Amount` without `CurrencyIsoCode`, you are summing values in different currencies. A pipeline that sums `Opportunity.Amount` across EUR and USD deals produces meaningless numbers.

**Rule**: In multi-currency orgs, always extract `CurrencyIsoCode` alongside every currency field. In dbt, either:
1. Use the extracted `CurrencyIsoCode` to convert in dbt using your own rate table
2. Extract and use `ConvertedAmount` if Salesforce's rates are acceptable
3. Extract the `CurrencyType` object for Salesforce's current exchange rates

**Detection**: `SELECT Id FROM CurrencyType LIMIT 1` — if it returns rows, multi-currency is enabled.

## Field-Level Security Filtering

Salesforce Field-Level Security (FLS) controls which fields a user can see. This affects extraction:

- If the integration user lacks FLS access to a field, the API **silently omits** the field from results — no error, no null, the field simply does not appear in the response
- `SELECT *` equivalent queries will return different columns depending on the user's FLS profile
- Managed package fields often have restrictive FLS defaults

**What goes wrong**: A pipeline extracts Opportunities with an integration user that lacks access to `SBQQ__NetTotal__c`. The field is silently missing from all rows. The pipeline loads successfully, but the CPQ total column is absent from the warehouse. Downstream dbt models that reference it fail — but only after days of loading apparently clean data.

**Rule**: Use a dedicated integration user with a Permission Set that grants Read access to all fields needed by the pipeline. After initial extraction, validate the extracted schema against expected columns:
```sql
-- In dbt, assert expected columns exist
SELECT
  sbqq__net_total__c  -- Will fail at parse time if column missing
FROM {{ source('salesforce', 'sbqq__quote__c') }}
LIMIT 0
```

## CPQ Field Hierarchy

When Salesforce CPQ is installed, the value flow is:

```
Product Catalog (Product2 + PricebookEntry)
  └─ Quote Line (SBQQ__QuoteLine__c)
       ├─ SBQQ__ListPrice__c        ← from PricebookEntry
       ├─ SBQQ__Discount__c         ← sales rep discount
       ├─ SBQQ__NetPrice__c         ← calculated after discounts
       ├─ SBQQ__CustomerPrice__c    ← after customer-specific pricing
       └─ SBQQ__NetTotal__c         ← line net total
            └─ rolls up to Quote (SBQQ__Quote__c)
                 ├─ SBQQ__NetTotal__c    ← sum of line net totals (AUTHORITATIVE)
                 ├─ SBQQ__ListTotal__c   ← sum of list prices
                 └─ syncs to → Opportunity.Amount (WHEN Primary Quote syncs)
```

**Critical detail**: The sync from Quote to Opportunity only happens for the Primary Quote (`SBQQ__Primary__c = true`). If the primary Quote is not set, or if the sync trigger fails or is disabled, `Opportunity.Amount` and `SBQQ__NetTotal__c` diverge with no warning.

**Extraction rule for CPQ orgs**: Always extract both Opportunity and the Quote/QuoteLine objects. In dbt, build a reconciliation model:
```sql
SELECT
  o.id AS opportunity_id,
  o.amount AS opp_amount,
  q.sbqq__net_total__c AS cpq_net_total,
  ABS(o.amount - q.sbqq__net_total__c) AS amount_variance,
  CASE
    WHEN ABS(o.amount - q.sbqq__net_total__c) > 0.01 THEN 'MISMATCH'
    ELSE 'OK'
  END AS sync_status
FROM {{ ref('stg_salesforce__opportunity') }} o
LEFT JOIN {{ ref('stg_salesforce__sbqq_quote') }} q
  ON q.sbqq__opportunity2__c = o.id
  AND q.sbqq__primary__c = true
```

## Picklist Coupling Traps

### `StageName` and `ForecastCategory`

These fields appear linked but are independently editable:

| Scenario | `StageName` | `ForecastCategory` | Problem |
|---|---|---|---|
| Normal | Negotiation | Best Case | Expected mapping, no issue |
| Manual override | Negotiation | Commit | Rep manually promoted forecast — reports conflict |
| Admin remap | Qualification | Closed | Admin changed the default mapping — historical data uses old mapping |

**Rule**: Never assume one implies the other. Extract both. Build a mapping table in dbt from `OpportunityStage` metadata and flag deviations.

### `LeadSource` and `CampaignId`

`LeadSource` is a picklist set at Lead or Contact creation. `CampaignId` links to a Campaign. They are often inconsistent — a Lead with `LeadSource = 'Web'` may be associated with an offline Campaign, or vice versa. Attribution models must handle this explicitly.

### `Type` on Opportunity

The `Type` field (New Business, Existing Business, etc.) is a picklist, not derived from any relationship. It is manually set and frequently wrong or outdated. Do not use it as the sole dimension for new vs. expansion revenue — cross-reference with Account's `CreatedDate` and Opportunity `RecordType`.
