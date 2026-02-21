# Azure Destination Patterns

ADLS Gen2 filesystem destination, Delta format, OneLake URL construction, secrets management, and the medallion handoff from dlt (bronze) to dbt (silver). The Fabric-specific patterns that Context7 covers thinly or not at all.

## Contents
- [Filesystem Destination to ADLS Gen2](#filesystem-destination-to-adls-gen2)
- [Delta Format Configuration](#delta-format-configuration)
- [OneLake Destination](#onelake-destination)
- [Secrets and Credential Management](#secrets-and-credential-management)
- [Lakehouse Landing Patterns](#lakehouse-landing-patterns)
- [Medallion Handoff to dbt](#medallion-handoff-to-dbt)

## Filesystem Destination to ADLS Gen2

### Basic ADLS Gen2 setup

Install the Azure extra:
```bash
pip install "dlt[az]"
```

Pipeline configuration:
```python
import dlt

pipeline = dlt.pipeline(
    pipeline_name="rest_api_to_adls",
    destination="filesystem",
    dataset_name="raw_api_data",
)
```

### Config files

**`.dlt/config.toml`** (safe to commit):
```toml
[destination.filesystem]
bucket_url = "abfss://bronze@mystorageaccount.dfs.core.windows.net/raw/"
```

**`.dlt/secrets.toml`** (gitignored):
```toml
[destination.filesystem.credentials]
azure_storage_account_name = "mystorageaccount"
azure_storage_account_key = "base64-key-here"
```

### URL schemes

| Scheme | Format | When to use |
|---|---|---|
| `abfss://` | `abfss://<container>@<account>.dfs.core.windows.net/<path>/` | Standard ADLS Gen2 — most common |
| `az://` | `az://<container>/<path>` | Shorthand — requires `azure_account_host` in credentials |

**Always use `abfss://`** for clarity and to avoid needing the separate `azure_account_host` config.

### Authentication methods

**Storage account key** (simplest, for dev):
```toml
[destination.filesystem.credentials]
azure_storage_account_name = "mystorageaccount"
azure_storage_account_key = "base64-key-here"
```

**Service principal** (recommended for prod):
```toml
[destination.filesystem.credentials]
azure_storage_account_name = "mystorageaccount"
azure_client_id = "00000000-0000-0000-0000-000000000000"
azure_client_secret = "client-secret"
azure_tenant_id = "00000000-0000-0000-0000-000000000000"
```

**Managed identity** (Azure-hosted pipelines):
No credentials needed — set only the `bucket_url`. The identity attached to the compute resource authenticates automatically.

## Delta Format Configuration

### Why Delta for Fabric

Fabric lakehouses **only discover Delta tables** in the `/Tables` path. Parquet files land in `/Files` and are not queryable via the SQL analytics endpoint. To make dlt-loaded data queryable:

```toml
[destination.filesystem]
bucket_url = "abfss://bronze@mystorageaccount.dfs.core.windows.net/Tables/"
```

Install the Delta extra:
```bash
pip install "dlt[az,deltalake]"
```

Configure Delta as the table format:
```python
pipeline = dlt.pipeline(
    pipeline_name="rest_api_to_adls",
    destination=dlt.destinations.filesystem(
        bucket_url="abfss://bronze@mystorageaccount.dfs.core.windows.net/Tables/",
        table_format="delta",
    ),
    dataset_name="raw_api_data",
)
```

Or in `config.toml`:
```toml
[destination.filesystem]
bucket_url = "abfss://bronze@mystorageaccount.dfs.core.windows.net/Tables/"
table_format = "delta"
```

### Delta vs Parquet trade-offs for bronze

| Factor | Delta | Parquet |
|---|---|---|
| Fabric discoverability | Auto-discovered in `/Tables` | Must be in `/Files`, not queryable via SQL endpoint |
| Schema evolution | Supported via Delta protocol | Must manage manually |
| Merge write disposition | Supported | Not supported — append only |
| Overhead | Delta log maintenance | None |
| Queryable via SQL analytics | Yes | No |

**Default to Delta** when landing in Fabric lakehouses. Use Parquet only for intermediate/staging files that won't be queried directly.

## OneLake Destination

### OneLake URL construction — GUIDs required

OneLake URLs use workspace and lakehouse **GUIDs**, not display names. This is the single most common mistake.

**Correct**:
```toml
[destination.filesystem]
bucket_url = "abfss://aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee@onelake.dfs.fabric.microsoft.com/ffffffff-gggg-hhhh-iiii-jjjjjjjjjjjj/Tables/"
```

Where:
- `aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee` = workspace GUID
- `ffffffff-gggg-hhhh-iiii-jjjjjjjjjjjj` = lakehouse GUID

**Wrong** (will fail with auth or path errors):
```toml
# DO NOT use display names
bucket_url = "abfss://my-workspace@onelake.dfs.fabric.microsoft.com/my-lakehouse/Tables/"
```

### Finding GUIDs

1. **Fabric portal URL**: Open the lakehouse in the browser. The URL contains both GUIDs:
   ```
   https://app.fabric.microsoft.com/groups/<workspace-guid>/lakehouses/<lakehouse-guid>
   ```
2. **PowerShell**: `Get-FabricWorkspace` and `Get-FabricItem`
3. **REST API**: Fabric Admin API `/v1/workspaces` endpoint

### OneLake authentication

OneLake uses Entra ID (Azure AD) authentication. Storage account keys do NOT work.

**Service principal**:
```toml
[destination.filesystem.credentials]
azure_client_id = "00000000-0000-0000-0000-000000000000"
azure_client_secret = "client-secret"
azure_tenant_id = "00000000-0000-0000-0000-000000000000"
```

The service principal needs the **Contributor** role on the Fabric workspace (not just Storage Blob Data Contributor — that's for raw ADLS).

### OneLake vs ADLS Gen2 — when to use which

| Factor | ADLS Gen2 (direct) | OneLake |
|---|---|---|
| URL host | `<account>.dfs.core.windows.net` | `onelake.dfs.fabric.microsoft.com` |
| Auth | Storage keys, SAS, service principal | Entra ID only (service principal, managed identity) |
| Path structure | Container / path | Workspace GUID / Lakehouse GUID / path |
| Fabric integration | Via Shortcut to the storage account | Native — tables auto-appear in lakehouse |
| Best for | Existing Azure storage, shared across teams | Direct Fabric integration, single-team |

**Recommendation**: Use OneLake when the lakehouse is the primary consumer. Use ADLS Gen2 when multiple consumers (Fabric, Databricks, Synapse) need the same data.

## Secrets and Credential Management

### Resolution order

dlt resolves credentials in this order (first match wins):
1. **Environment variables** — `DESTINATION__FILESYSTEM__CREDENTIALS__AZURE_STORAGE_ACCOUNT_NAME`
2. **`secrets.toml`** — `.dlt/secrets.toml` (gitignored)
3. **In-code** — passed directly to `dlt.destinations.filesystem(...)`

### Environment variable naming convention

Double underscores (`__`) represent nesting levels:
```bash
# Equivalent to [destination.filesystem.credentials] azure_storage_account_name
export DESTINATION__FILESYSTEM__CREDENTIALS__AZURE_STORAGE_ACCOUNT_NAME="mystorageaccount"
export DESTINATION__FILESYSTEM__CREDENTIALS__AZURE_STORAGE_ACCOUNT_KEY="base64-key"

# Equivalent to [destination.filesystem] bucket_url
export DESTINATION__FILESYSTEM__BUCKET_URL="abfss://bronze@mystorageaccount.dfs.core.windows.net/Tables/"
```

### CI/CD pattern

In GitHub Actions, set secrets as environment variables:
```yaml
env:
  DESTINATION__FILESYSTEM__BUCKET_URL: ${{ secrets.ADLS_BUCKET_URL }}
  DESTINATION__FILESYSTEM__CREDENTIALS__AZURE_STORAGE_ACCOUNT_NAME: ${{ secrets.AZURE_STORAGE_ACCOUNT_NAME }}
  DESTINATION__FILESYSTEM__CREDENTIALS__AZURE_STORAGE_ACCOUNT_KEY: ${{ secrets.AZURE_STORAGE_ACCOUNT_KEY }}
```

For Azure-hosted runners with managed identity, only `DESTINATION__FILESYSTEM__BUCKET_URL` is needed.

## Lakehouse Landing Patterns

### Target /Tables for queryable data

```
Bronze Lakehouse
├── Tables/           ← dlt lands Delta tables here (auto-discovered by SQL analytics)
│   ├── raw_customers/
│   │   ├── _delta_log/
│   │   └── *.parquet
│   └── raw_orders/
└── Files/            ← raw files, not auto-discovered
    └── staging/
```

Configure `bucket_url` to include `/Tables/`:
```toml
bucket_url = "abfss://bronze@mystorageaccount.dfs.core.windows.net/Tables/"
```

### Dataset naming

dlt's `dataset_name` becomes a folder under the target path:
```python
pipeline = dlt.pipeline(
    pipeline_name="hubspot_pipeline",
    destination="filesystem",
    dataset_name="hubspot",  # creates /Tables/hubspot/ folder
)
```

Tables land as: `/Tables/hubspot/contacts/`, `/Tables/hubspot/deals/`, etc.

### Multiple sources to one lakehouse

Use different `dataset_name` values to namespace:
```python
# Pipeline 1
pipeline_hubspot = dlt.pipeline(dataset_name="hubspot", ...)
# Pipeline 2
pipeline_stripe = dlt.pipeline(dataset_name="stripe", ...)
```

Result:
```
Tables/
├── hubspot/
│   ├── contacts/
│   └── deals/
└── stripe/
    ├── customers/
    └── invoices/
```

## Medallion Handoff to dbt

### Bronze → Silver boundary

dlt owns bronze (extract + load). dbt owns silver and gold (transform). The boundary is the Delta table in the lakehouse.

**dbt source definition** pointing at dlt-loaded tables:
```yaml
# models/staging/_sources.yml
sources:
  - name: hubspot_raw
    schema: hubspot           # matches dlt dataset_name
    tables:
      - name: contacts
        loaded_at_field: _dlt_load_id
        freshness:
          warn_after: { count: 24, period: hour }
          error_after: { count: 48, period: hour }
      - name: deals
        loaded_at_field: _dlt_load_id
```

### _dlt metadata columns

Every dlt-loaded table includes these system columns:

| Column | Type | Purpose |
|---|---|---|
| `_dlt_load_id` | `VARCHAR` | Unique load batch identifier — use for incremental models |
| `_dlt_id` | `VARCHAR` | Unique row hash — use for deduplication |

### Incremental models using _dlt_load_id

```sql
-- models/staging/stg_hubspot__contacts.sql
with source as (
    select * from {{ source('hubspot_raw', 'contacts') }}
    {% if is_incremental() %}
      where _dlt_load_id > (select max(_dlt_load_id) from {{ this }})
    {% endif %}
)

select
    id as contact_id,
    json_value(properties, '$.email') as email,
    json_value(properties, '$.firstname') as first_name,
    json_value(properties, '$.lastname') as last_name,
    _dlt_load_id
from source
```

Note the use of `json_value()` (T-SQL) to extract fields from JSON columns created by `max_table_nesting=0`. This is the standard pattern: dlt preserves JSON in bronze, dbt unpacks it in silver.
