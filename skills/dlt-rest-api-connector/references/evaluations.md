# Evaluations

Test scenarios for the dlt-rest-api-connector skill. Each scenario tests whether the skill produces correct, Fabric-specific guidance for REST API ingestion with dlt.

## Scenarios

### Scenario 1: REST API config with nested response and pagination

**Prompt**: "Build a dlt pipeline to load data from a CRM API at https://api.crm.example.com/v2/. The contacts endpoint returns `{ "meta": { "total": 5000 }, "data": { "contacts": [...] } }` and uses page-number pagination with a `page` query parameter. I need to land this in our ADLS Gen2 bronze lakehouse."

**Expected behavior**: Claude should produce a `RESTAPIConfig` with:
- `base_url` ending in `/` (`"https://api.crm.example.com/v2/"`)
- `data_selector: "data.contacts"` (dot-path to the nested array, not just `"data"`)
- `paginator` with `type: "page_number"`, `page_param: "page"`, and `total_path: "meta.total"`
- `params` inside `endpoint` (not at the resource level)
- Filesystem destination with `abfss://` URL and Delta table format
- `max_table_nesting=0` on the source to preserve JSON structure in bronze

**Pass criteria**:
1. `data_selector` correctly navigates the nested response to `"data.contacts"`
2. `params` is inside `endpoint`, not at the resource level

### Scenario 2: Pagination type selection

**Prompt**: "I'm integrating with three APIs: (1) GitHub API that returns a Link header with rel=next, (2) a vendor API where the response body has `{ 'next_cursor': 'abc123' }` at the top level, and (3) an internal API that only supports `?offset=0&limit=100`. What pagination config should I use for each in my dlt REST API source?"

**Expected behavior**: Claude should recommend:
- GitHub: `header_link` paginator (no extra config needed, auto-parses `Link` header)
- Vendor API: `cursor` paginator with `cursor_path: "next_cursor"` and `cursor_param` set to whatever query parameter the API expects
- Internal API: `offset` paginator with `limit: 100`, noting the risk of data skew during concurrent writes
- A general recommendation to try `auto` first and fall back to explicit config

**Pass criteria**:
1. Correctly identifies all three paginator types (`header_link`, `cursor`, `offset`)
2. Warns about offset pagination's data consistency risk

### Scenario 3: Schema contracts for production deployment

**Prompt**: "We're moving our dlt REST API pipeline from dev to production. The API sometimes adds new fields to its response without warning. Last week they added a `loyalty_tier` field and our downstream dbt models broke because the column wasn't in our staging model. How should I configure schema contracts?"

**Expected behavior**: Claude should recommend:
- `freeze` or `discard_value` for production (not `evolve`)
- Explain the three-level contract: `tables`, `columns`, `data_type`
- Suggest `{"tables": "evolve", "columns": "freeze", "data_type": "freeze"}` as a good default — allows new endpoints but freezes existing table schemas
- Explain that `freeze` will fail the pipeline on schema drift (catching it before it reaches dbt)
- Mention `discard_value` as an alternative that logs the new field but doesn't load it
- Note that the `evolve` setting they're currently using (default) silently adds columns

**Pass criteria**:
1. Recommends `freeze` or `discard_value` for columns (not `evolve`)
2. Explains the three-level contract mechanism (`tables`, `columns`, `data_type`)

### Scenario 4: OneLake destination URL construction

**Prompt**: "I need to configure my dlt pipeline to write directly to a Fabric lakehouse via OneLake. My workspace is called 'Analytics Prod' and the lakehouse is 'Bronze Lakehouse'. How do I set up the destination?"

**Expected behavior**: Claude should:
- Warn that display names ("Analytics Prod", "Bronze Lakehouse") cannot be used in OneLake URLs
- Explain that workspace and lakehouse GUIDs are required
- Show how to find GUIDs (Fabric portal URL, PowerShell, REST API)
- Provide the correct URL format: `abfss://<workspace-guid>@onelake.dfs.fabric.microsoft.com/<lakehouse-guid>/Tables/`
- Note that OneLake requires Entra ID auth (service principal or managed identity), not storage account keys
- Include `/Tables/` in the path so data is auto-discovered by the SQL analytics endpoint

**Pass criteria**:
1. Explicitly states that display names cannot be used and GUIDs are required
2. URL uses `onelake.dfs.fabric.microsoft.com` with GUID placeholders (not display names)

### Scenario 5: End-to-end pipeline with _dlt_load_id bridge

**Prompt**: "I'm building a dlt pipeline to ingest order data from a REST API into our Fabric bronze lakehouse. The downstream dbt team needs to build incremental models on top of this data. How should I structure the pipeline so the dbt models can incrementally process only new loads?"

**Expected behavior**: Claude should:
- Configure `max_table_nesting=0` to preserve JSON in bronze
- Use Delta format targeting `/Tables/` for Fabric discoverability
- Explain `_dlt_load_id` as the bridge column between dlt and dbt
- Show a dbt incremental model using `_dlt_load_id` (not business timestamps) for the incremental predicate
- Explain why `_dlt_load_id` is better than `updated_at` for the incremental boundary (no late-arriving data problem, no lookback window needed)
- Show the dbt `sources.yml` with `loaded_at_field: _dlt_load_id` for freshness checks

**Pass criteria**:
1. Uses `_dlt_load_id` (not business timestamps) for the dbt incremental predicate
2. Includes `max_table_nesting=0` and Delta format in the pipeline configuration
