# Semantic Model Patterns

Entity relationships, join paths, dimension and measure design, and semantic model architecture for dbt on Fabric. Covers what Context7 and official docs handle thinly: multi-hop join behavior, entity type consequences, and Fabric-specific expression patterns.

## Contents
- [Entity Relationships](#entity-relationships)
- [Join Logic](#join-logic)
- [Dimensions](#dimensions)
- [Measures](#measures)
- [Semantic Model Architecture](#semantic-model-architecture)

## Entity Relationships

### Entity types and their consequences

Entity types control how MetricFlow joins models. Choosing wrong types silently breaks queries.

**`primary`** -- one row per entity value, no nulls. MetricFlow treats this as the "owning" model for that entity.

```yaml
# fct_orders: one row per order
entities:
  - name: order_id
    type: primary
```

**`foreign`** -- many rows per entity value. MetricFlow knows this side of the join can fan out.

```yaml
# fct_orders: many orders per customer
entities:
  - name: customer_id
    type: foreign
```

**`unique`** -- at most one row per entity value, but nulls are allowed. Use for columns that are unique but nullable (e.g., an optional `email` column that serves as a lookup key).

```yaml
# dim_customers: email is unique but some customers don't have one
entities:
  - name: email
    type: unique
```

**`natural`** -- no cardinality guarantees. MetricFlow allows joins but cannot optimize them. Avoid unless you have a legitimate shared concept without enforced uniqueness (rare).

### Entity naming across models

MetricFlow joins models by matching entity **names**, not column names. The `expr` field maps the entity name to the actual column:

```yaml
# In fct_orders:
entities:
  - name: customer          # Entity name -- must match across models
    type: foreign
    expr: customer_id        # Actual column in fct_orders

# In dim_customers:
entities:
  - name: customer          # Same entity name -- MetricFlow auto-joins
    type: primary
    expr: customer_id        # Actual column in dim_customers
```

**Common mistake**: Using different entity names (`customer_id` in one model, `customer` in another) and expecting MetricFlow to join them. Entity names must match exactly. Use `expr` to map to the underlying column.

### Multiple entities per model

A semantic model can have multiple foreign entities:

```yaml
# fct_order_items: grain is one row per order line item
entities:
  - name: order_item_id
    type: primary
  - name: order
    type: foreign
    expr: order_id
  - name: product
    type: foreign
    expr: product_id
  - name: customer
    type: foreign
    expr: customer_id
```

This enables MetricFlow to join `fct_order_items` to `dim_orders`, `dim_products`, and `dim_customers` in a single query.

## Join Logic

### How MetricFlow resolves joins

MetricFlow uses a graph of entities to determine join paths. Rules:

1. **Left joins for fact-to-dimension**: When querying a measure from `fct_orders` with a dimension from `dim_customers`, MetricFlow left-joins `dim_customers` onto `fct_orders`. All fact rows are preserved.
2. **Full outer joins for multi-fact queries**: When querying measures from multiple fact tables, MetricFlow full-outer-joins them to preserve all rows from both.
3. **No fan-out joins**: MetricFlow rejects joins that would multiply rows (joining a primary entity to a non-unique entity). This prevents accidental double-counting.

### Multi-hop joins

MetricFlow supports up to **two hops** (three tables). Example: `fct_orders` -> `dim_customers` -> `dim_countries`.

```yaml
semantic_models:
  - name: orders
    model: ref('fct_orders')
    defaults:
      agg_time_dimension: order_date
    entities:
      - name: order_id
        type: primary
      - name: customer
        type: foreign
        expr: customer_id
    measures:
      - name: revenue
        agg: sum
        expr: amount
    dimensions:
      - name: order_date
        type: time
        type_params:
          time_granularity: day

  - name: customers
    model: ref('dim_customers')
    entities:
      - name: customer
        type: primary
        expr: customer_id
      - name: country
        type: foreign
        expr: country_id
    dimensions:
      - name: customer_segment
        type: categorical

  - name: countries
    model: ref('dim_countries')
    entities:
      - name: country
        type: primary
        expr: country_id
    dimensions:
      - name: country_name
        type: categorical
      - name: region
        type: categorical
```

Now you can query `revenue` grouped by `country_name` -- MetricFlow traverses `orders -> customers -> countries` automatically.

**Two-hop limit**: If your schema requires orders -> customers -> addresses -> cities, MetricFlow cannot traverse all four. Solutions:
- Denormalize `city_name` into `dim_customers` (push it one hop closer)
- Create a `dim_customer_locations` model that pre-joins addresses and cities

### Ambiguous join paths

If two paths exist between the same entities, MetricFlow raises an error. Example: `fct_orders` has both `billing_customer_id` and `shipping_customer_id`, both pointing to `dim_customers`.

Solution: Use distinct entity names:

```yaml
# fct_orders
entities:
  - name: billing_customer
    type: foreign
    expr: billing_customer_id
  - name: shipping_customer
    type: foreign
    expr: shipping_customer_id

# dim_customers -- define the same model twice with different entity names
# Option 1: Two semantic models pointing to the same dbt model
- name: billing_customers
  model: ref('dim_customers')
  entities:
    - name: billing_customer
      type: primary
      expr: customer_id

- name: shipping_customers
  model: ref('dim_customers')
  entities:
    - name: shipping_customer
      type: primary
      expr: customer_id
```

This pattern is verbose but necessary. MetricFlow does not support role-playing dimensions natively -- you must create separate semantic models.

## Dimensions

### Time dimensions

Every semantic model that has measures must have at least one time dimension. The `defaults.agg_time_dimension` specifies which one MetricFlow uses for time-series queries.

```yaml
dimensions:
  - name: order_date
    type: time
    type_params:
      time_granularity: day    # Minimum granularity for queries
```

**Granularity options**: `day`, `week`, `month`, `quarter`, `year`. Setting `time_granularity: month` prevents users from querying at daily grain -- MetricFlow returns an error.

**Multiple time dimensions**: A model can have several time dimensions (e.g., `order_date`, `ship_date`). Only one is the default `agg_time_dimension`. Metrics can override this per-measure.

**Fabric gotcha**: If your time column is stored as `datetime2`, the `expr` should truncate it:

```yaml
dimensions:
  - name: order_date
    type: time
    expr: "CAST(order_datetime AS DATE)"   # T-SQL cast
    type_params:
      time_granularity: day
```

Do not use `date_trunc('day', ...)` -- that is Snowflake/Postgres syntax. On Fabric, use `CAST(... AS DATE)` or `CONVERT(DATE, ...)`.

### Categorical dimensions

Categorical dimensions are discrete values used for grouping and filtering:

```yaml
dimensions:
  - name: order_status
    type: categorical
  - name: is_high_value
    type: categorical
    expr: "CASE WHEN amount > 1000 THEN 'Yes' ELSE 'No' END"
```

**Fabric gotcha**: Do not use boolean expressions in `expr` for categorical dimensions. T-SQL has no boolean type. Use string values (`'Yes'`/`'No'`) or integers (`1`/`0`), not `TRUE`/`FALSE`.

### Dimension naming conventions

MetricFlow references dimensions using the `entity__dimension` syntax:

```
Dimension('customer__customer_segment')
TimeDimension('order__order_date', 'month')
```

The entity prefix is the entity name from the semantic model where the dimension lives. This means dimension names should be descriptive without the entity prefix (use `customer_segment`, not `segment`).

## Measures

### Aggregation types on Fabric

Available aggregations and their T-SQL compatibility:

```yaml
measures:
  # Basic aggregations -- all work on Fabric
  - name: total_revenue
    agg: sum
    expr: amount

  - name: avg_order_value
    agg: average
    expr: amount

  - name: unique_customers
    agg: count_distinct
    expr: customer_id

  - name: max_order_value
    agg: max
    expr: amount

  # Count pattern -- sum of literal 1
  - name: order_count
    agg: sum
    expr: "1"

  # Conditional aggregation -- works on Fabric
  - name: refunded_amount
    agg: sum
    expr: "CASE WHEN order_status = 'refunded' THEN amount ELSE 0 END"

  # Boolean sum -- column must be BIT (0/1) on Fabric
  - name: express_orders
    agg: sum_boolean
    expr: is_express    # Must be BIT column, not boolean
```

### Measures that DON'T work on Fabric

```yaml
# WILL FAIL on Fabric -- PERCENTILE_CONT not supported
- name: p99_order_value
  agg: percentile
  expr: amount
  agg_params:
    percentile: 0.99
    use_discrete_percentile: false

# WILL FAIL on Fabric -- median is percentile(0.5)
- name: median_order_value
  agg: median
  expr: amount
```

**Workaround**: Compute percentiles in the mart model using T-SQL `APPROX_PERCENTILE_CONT` (available on Fabric) or a subquery pattern, then expose the result as a `max` or `sum` measure (since it's already pre-aggregated to one value per group).

### `create_metric: true` shorthand

Adding `create_metric: true` to a measure auto-generates a simple metric with the same name:

```yaml
measures:
  - name: revenue
    agg: sum
    expr: amount
    create_metric: true   # Auto-creates a 'revenue' simple metric
```

This is equivalent to:

```yaml
metrics:
  - name: revenue
    type: simple
    type_params:
      measure: revenue
```

**Rule**: Use `create_metric: true` for straightforward measures that need no filters, offsets, or custom labels. Define a separate metric block for anything more complex. Never use both for the same measure -- it creates a duplicate definition error.

## Semantic Model Architecture

### Which dbt models get semantic models

Not every dbt model needs a semantic model. Decision framework:

| Model type | Gets a semantic model? | Reason |
|---|---|---|
| `stg_*` (staging) | **No** | No business logic, no measures. Staging is a pass-through. |
| `int_*` (intermediate) | **Rarely** | Only if the intermediate model is the final grain for a specific metric. Usually not. |
| `fct_*` (facts) | **Yes** | Facts contain measures (revenue, counts, amounts). |
| `dim_*` (dimensions) | **Yes** | Dimensions provide grouping and filtering context. |
| Pre-joined wide marts | **No** | If you have the semantic layer, you don't need wide marts. |

### File organization

Place semantic model YAML in the same directory as the dbt model it references:

```
models/
  marts/
    finance/
      fct_orders.sql
      fct_orders.yml           # Tests and docs
      sem_fct_orders.yml       # Semantic model (separate file)
      dim_customers.sql
      dim_customers.yml
      sem_dim_customers.yml
    marketing/
      fct_campaigns.sql
      sem_fct_campaigns.yml

  metrics/
    revenue_metrics.yml        # Metrics can live in a dedicated directory
    marketing_metrics.yml
```

**Why separate files**: Semantic model YAML can be long. Mixing it with column tests and docs makes files unwieldy. A `sem_` prefix or dedicated `semantic_models/` directory keeps things navigable.

### One semantic model per dbt model

Each semantic model maps to exactly one dbt model via the `model` field. Do not create a semantic model that references multiple dbt models -- MetricFlow does not support this. Cross-model relationships are expressed through entity matching, not multi-model semantic models.

**Exception**: Role-playing dimensions (see [Ambiguous join paths](#ambiguous-join-paths)) require multiple semantic models pointing to the same dbt model with different entity names.
