# REST API Source Patterns

Detailed RESTAPIConfig structure, pagination types, auth patterns, resource dependencies, and incremental loading. The patterns Context7 covers thinly or gets wrong in practice.

## Contents
- [RESTAPIConfig Deep Dive](#restapiconfig-deep-dive)
- [Pagination Types](#pagination-types)
- [Authentication Patterns](#authentication-patterns)
- [Data Selector and Response Extraction](#data-selector-and-response-extraction)
- [Incremental Loading from REST APIs](#incremental-loading-from-rest-apis)
- [Resource Dependencies](#resource-dependencies)
- [Error Handling and Retries](#error-handling-and-retries)

## RESTAPIConfig Deep Dive

### Full config anatomy

```python
from dlt.sources.rest_api import rest_api_source, RESTAPIConfig
import dlt

config: RESTAPIConfig = {
    "client": {
        "base_url": "https://api.example.com/v2/",
        "auth": { ... },           # see Authentication Patterns
        "paginator": { ... },      # see Pagination Types
        "headers": {               # static headers applied to all requests
            "Accept": "application/json",
            "X-Api-Version": "2024-01",
        },
    },
    "resource_defaults": {
        "primary_key": "id",
        "write_disposition": "merge",
        "endpoint": {
            "params": {
                "per_page": 100,   # default params for all endpoints
            },
        },
    },
    "resources": [
        # ... resource definitions
    ],
}
```

### Resource definition — all fields

```python
{
    "name": "invoices",                # table name in destination
    "table_name": "billing_invoices",  # override destination table name (optional)
    "primary_key": "id",               # single column or list for composite
    "write_disposition": "merge",
    "selected": True,                  # set False to skip this resource
    "endpoint": {
        "path": "billing/invoices",
        "method": "GET",               # GET is default; POST for search endpoints
        "params": {
            "status": "finalized",
            "expand[]": "line_items",   # API-specific query params
        },
        "json": { ... },               # POST body (when method is POST)
        "data_selector": "data",       # JSON path to the array of records
        "paginator": { ... },          # override client-level paginator
        "response_actions": [          # custom response handling
            {"status_code": 404, "action": "ignore"},
        ],
        "incremental": {               # incremental loading config
            "start_param": "since",
            "end_param": "until",
            "cursor_path": "updated_at",
            "initial_value": "2024-01-01T00:00:00Z",
        },
    },
}
```

### String shorthand vs explicit config

```python
"resources": [
    # String shorthand — all three are equivalent:
    "users",
    # Means: name="users", endpoint.path="users", table_name="users"

    # Explicit equivalent:
    {
        "name": "users",
        "endpoint": {
            "path": "users",
        },
    },
]
```

Use string shorthand only when:
- The endpoint path matches the desired table name
- No params, pagination overrides, or data selectors needed
- Default `write_disposition` from `resource_defaults` is correct

### Common config mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| `params` at resource level (not inside `endpoint`) | Params silently ignored, unfiltered results | Move `params` inside `endpoint: { params: {...} }` |
| Missing `data_selector` on nested response | dlt tries to load the wrapper object, not the array | Set `data_selector` to the dot-path of the array |
| `base_url` without trailing slash | Path concatenation breaks (`/v2items` instead of `/v2/items`) | Always end `base_url` with `/` |
| `primary_key` on append-only resources | Unnecessary overhead, potential key violations on duplicates | Remove `primary_key` when `write_disposition` is `append` |

## Pagination Types

### json_link — next URL in response body

The most common pattern. The API returns a URL to the next page somewhere in the response JSON.

```python
"paginator": {
    "type": "json_link",
    "next_url_path": "paging.next",  # dot-path to the next page URL
}
```

**API response shape this handles**:
```json
{
  "data": [...],
  "paging": {
    "next": "https://api.example.com/v2/users?cursor=abc123"
  }
}
```

**When `next_url_path` is null or missing**, pagination stops. This is how dlt knows it's on the last page.

### header_link — Link header (RFC 5988)

GitHub API, many standards-compliant APIs. The `Link` header contains `rel="next"`.

```python
"paginator": {
    "type": "header_link",
}
```

No additional config needed — dlt parses the standard `Link` header format:
```
Link: <https://api.example.com/users?page=2>; rel="next"
```

### page_number — sequential page numbers

```python
"paginator": {
    "type": "page_number",
    "page_param": "page",         # query parameter name (default: "page")
    "total_path": "meta.total",   # dot-path to total record count in response
    "maximum_page": 100,          # hard limit — use when total_path unavailable
}
```

**Stopping condition**: dlt stops when `page * page_size >= total` (from `total_path`) or when `page >= maximum_page`. You need at least one — without either, dlt paginates forever.

**Gotcha**: some APIs use 0-based pages, others 1-based. Check the API docs. dlt defaults to page 1.

### cursor — opaque cursor tokens

```python
"paginator": {
    "type": "cursor",
    "cursor_path": "meta.next_cursor",  # dot-path to cursor in response
    "cursor_param": "cursor",            # query parameter to send cursor in
}
```

**API response shape**:
```json
{
  "data": [...],
  "meta": {
    "next_cursor": "eyJpZCI6MTAwfQ=="
  }
}
```

When `next_cursor` is null, pagination stops. Cursor-based pagination is the most reliable for large datasets — no risk of skipping or duplicating records during concurrent writes.

### offset — offset/limit numeric pagination

```python
"paginator": {
    "type": "offset",
    "limit": 100,
    "offset_param": "offset",   # query param for offset (default: "offset")
    "limit_param": "limit",     # query param for limit (default: "limit")
}
```

**Risk**: offset pagination can miss or duplicate records if the underlying data changes between pages. Prefer cursor-based when available.

### Pagination decision process

1. **Start with `"paginator": "auto"`** — dlt auto-detects from response shape
2. If auto returns only one page of data, inspect the API response manually
3. Look for: a `next` URL in the body (`json_link`), `Link` header (`header_link`), cursor tokens (`cursor`), or total count (`page_number`)
4. If the API uses offset/limit and nothing else, use `offset` as last resort

## Authentication Patterns

### Bearer token

```python
"auth": {
    "type": "bearer",
    "token": dlt.secrets["api_token"],
}
```

### API key in header

```python
"auth": {
    "type": "api_key",
    "name": "X-API-Key",        # header name
    "api_key": dlt.secrets["api_key"],
    "location": "header",       # "header" (default) or "query"
}
```

### API key in query parameter

```python
"auth": {
    "type": "api_key",
    "name": "api_key",
    "api_key": dlt.secrets["api_key"],
    "location": "query",
}
```

### OAuth2 client credentials

```python
"auth": {
    "type": "oauth2_client_credentials",
    "access_token_url": "https://auth.example.com/oauth/token",
    "client_id": dlt.secrets["client_id"],
    "client_secret": dlt.secrets["client_secret"],
    "scopes": ["read:data"],     # optional
}
```

### Secrets management

Never hardcode tokens. Use dlt's resolution order:
1. **Environment variables**: `SOURCES__MY_SOURCE__API_TOKEN` (double underscore for nesting)
2. **`secrets.toml`** (`.dlt/secrets.toml` — gitignored):
```toml
[sources.my_source]
api_token = "sk-..."
```
3. **`dlt.secrets` dictionary**: accessed in code as `dlt.secrets["api_token"]`

For Azure deployments, use Key Vault references via environment variables set from the vault.

## Data Selector and Response Extraction

### Problem: APIs wrap arrays in metadata

Most APIs don't return a bare array. They wrap it:
```json
{
  "status": "ok",
  "meta": { "total": 500, "page": 1 },
  "data": {
    "items": [
      { "id": 1, "name": "First" },
      { "id": 2, "name": "Second" }
    ]
  }
}
```

Without `data_selector`, dlt tries to load the entire response as one record. You need:
```python
"data_selector": "data.items"
```

### Common data_selector patterns

| API response structure | data_selector value |
|---|---|
| `{ "data": [...] }` | `"data"` |
| `{ "results": [...] }` | `"results"` |
| `{ "data": { "items": [...] } }` | `"data.items"` |
| `{ "response": { "records": [...] } }` | `"response.records"` |
| `[...]` (bare array) | `"$"` or omit entirely |

### Debugging data_selector

If your pipeline loads but produces unexpected table structure (single row with a JSON blob instead of multiple rows), the `data_selector` is wrong. Steps:
1. Make a raw API call (curl or browser) and inspect the JSON
2. Find the array of records you want to load
3. Trace the dot-path from the root to that array

## Incremental Loading from REST APIs

### Cursor-based incremental

For APIs that support filtering by timestamp:

```python
{
    "name": "events",
    "endpoint": {
        "path": "events",
        "incremental": {
            "start_param": "since",           # query param for start timestamp
            "cursor_path": "created_at",      # field in each record to track
            "initial_value": "2024-01-01T00:00:00Z",
        },
    },
    "write_disposition": "append",
}
```

On first run, dlt sends `?since=2024-01-01T00:00:00Z`. On subsequent runs, it sends `?since=<max created_at from previous run>`. The cursor value is persisted in dlt's pipeline state.

### Incremental with end parameter

Some APIs require both start and end:

```python
"incremental": {
    "start_param": "updated_after",
    "end_param": "updated_before",
    "cursor_path": "updated_at",
    "initial_value": "2024-01-01T00:00:00Z",
}
```

### When incremental doesn't work

Not all APIs support timestamp-based filtering. Alternatives:
- **Full replace**: `write_disposition: "replace"` — reload everything each run. Acceptable for small reference tables (<10K records).
- **Client-side dedup**: Use `append` with `merge` in dbt downstream. Load everything, deduplicate in the silver layer.

## Resource Dependencies

### Parent-child resolution

When a child endpoint needs an ID from a parent:

```python
"resources": [
    {
        "name": "customers",
        "endpoint": { "path": "customers" },
    },
    {
        "name": "orders",
        "endpoint": {
            "path": "customers/{customer_id}/orders",
            "params": {
                "customer_id": {
                    "type": "resolve",
                    "resource": "customers",
                    "field": "id",
                },
            },
        },
    },
]
```

dlt first loads all customers, then for each customer, fetches their orders by substituting `{customer_id}` in the path. This runs sequentially — one API call per parent record.

### Performance implications

- Parent-child resolution generates N+1 API calls (1 for parent, N for children)
- For parents with thousands of records, this is slow. Consider:
  - Does the API have a bulk endpoint that accepts multiple IDs?
  - Can you use a different endpoint that returns all children without a parent filter?
- dlt parallelizes child requests within a resource, but the parent must complete first

## Error Handling and Retries

### Response actions

```python
"endpoint": {
    "response_actions": [
        {"status_code": 404, "action": "ignore"},    # skip missing resources
        {"status_code": 429, "action": "retry"},      # rate limit — retry with backoff
    ],
}
```

### Rate limiting

dlt handles `429 Too Many Requests` automatically with exponential backoff. If the API uses non-standard rate limiting:
- `Retry-After` header is respected automatically
- Custom rate limit headers can be handled via `response_actions`

### Timeout configuration

For slow APIs, configure at the client level:
```python
"client": {
    "base_url": "...",
    "request_timeout": 60,  # seconds
}
```
