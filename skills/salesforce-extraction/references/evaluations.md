# Evaluations

Test scenarios for the salesforce-extraction skill. Each scenario tests whether the skill produces correct, Salesforce-specific guidance that Claude would otherwise get wrong.

## Scenarios

### Scenario 1: CDC timestamp selection

**Prompt**: "I'm building an incremental dlt pipeline to extract Salesforce Opportunities into our Fabric warehouse. Which timestamp field should I use as the incremental cursor for change data capture?"

**Expected behavior**: Claude should:
- Recommend `SystemModstamp` as the CDC cursor, not `LastModifiedDate`
- Explain that `LastModifiedDate` misses system-initiated changes (workflow field updates, Process Builder, triggers, batch Apex)
- Note that `SystemModstamp` is indexed while `LastModifiedDate` is not, making it faster for filtered queries
- Mention that `SystemModstamp >= LastModifiedDate` always holds
- Confirm that the dlt Salesforce verified source defaults to `SystemModstamp`

**Pass criteria**:
1. Explicitly recommends `SystemModstamp` over `LastModifiedDate` with reasoning about system-initiated changes
2. Does NOT recommend `LastModifiedDate` as a viable alternative for CDC

### Scenario 2: CPQ Amount field

**Prompt**: "We have Salesforce CPQ installed. I'm writing a dbt mart model for revenue reporting that reads from our Salesforce extraction. Should I use Opportunity.Amount for the deal value?"

**Expected behavior**: Claude should:
- Warn that `Opportunity.Amount` is not the authoritative value in CPQ orgs
- Recommend using `SBQQ__NetTotal__c` from the primary Quote (`SBQQ__Primary__c = true`)
- Explain the sync mechanism: Quote Lines roll up to Quote, which syncs to Opportunity only when the primary Quote triggers sync
- Recommend extracting `SBQQ__Quote__c` and `SBQQ__QuoteLine__c` objects alongside Opportunity
- Suggest a reconciliation model that compares `Opportunity.Amount` vs `SBQQ__NetTotal__c`

**Pass criteria**:
1. States that `Opportunity.Amount` is unreliable in CPQ orgs and identifies `SBQQ__NetTotal__c` as the authoritative field
2. Mentions joining on `SBQQ__Opportunity2__c` with `SBQQ__Primary__c = true`

### Scenario 3: Soft-delete visibility

**Prompt**: "Our Salesforce pipeline shows 10,000 Opportunities in the warehouse, but the Salesforce admin says there should be about 10,200. We're using dlt with the Salesforce verified source. What could explain the discrepancy?"

**Expected behavior**: Claude should:
- Identify soft deletes as the most likely cause — standard `query()` silently excludes `IsDeleted = true` records
- Recommend using `queryAll()` / the REST `queryAll` endpoint to capture soft-deleted records
- Suggest checking whether the dlt source is using `query()` or `queryAll()` by looking for `is_deleted` column data
- Provide a reconciliation query to detect missing records
- Mention that merged records (duplicate Account/Contact merges) also disappear from standard queries

**Pass criteria**:
1. Identifies `query()` vs `queryAll()` as the root cause, not API limits or pagination issues
2. Mentions checking for the `is_deleted` column in the warehouse to verify soft-delete capture

### Scenario 4: Record Type mixing

**Prompt**: "Our revenue dashboard shows total pipeline that's way higher than what sales leadership expects. The numbers seem about 2x what they should be. We're extracting Salesforce Opportunities via dlt into Fabric and aggregating in dbt."

**Expected behavior**: Claude should:
- Ask about or investigate Record Types — the most common cause of pipeline double-counting
- Explain that without `RecordTypeId` filtering, the query returns all Opportunity types (new business, renewals, internal, partner)
- Recommend extracting `RecordTypeId` and `RecordType.DeveloperName`
- Suggest filtering in dbt staging models or adding Record Type as a dimension in the mart
- Provide the diagnostic query: `SELECT DISTINCT RecordType.DeveloperName FROM Opportunity`

**Pass criteria**:
1. Identifies Record Type mixing as a probable cause of the 2x discrepancy
2. Recommends extracting `RecordTypeId` and filtering or partitioning by Record Type in dbt

### Scenario 5: Managed package field discovery

**Prompt**: "We're setting up a new Salesforce extraction pipeline. The org has been running for 5 years with various apps installed. How do I figure out which custom objects and fields I need to extract beyond the standard Salesforce objects?"

**Expected behavior**: Claude should:
- Recommend querying `EntityDefinition` grouped by `NamespacePrefix` to discover installed managed packages
- List common high-impact packages and their namespaces (SBQQ__ for CPQ, clari__ for Clari, gong__ for Gong)
- Warn about Field-Level Security silently omitting fields — recommend a dedicated integration user with broad read access
- Suggest checking whether managed packages override standard field behavior (e.g., CPQ overriding Amount)
- Mention that managed package fields use namespace prefixes in API names, which affects dbt source/staging model naming

**Pass criteria**:
1. Provides the `EntityDefinition` query grouped by `NamespacePrefix` (or equivalent discovery approach)
2. Warns about at least one specific managed package (CPQ/SBQQ__) overriding standard field behavior
