# Evaluations

Test scenarios for the dbt-semantic-layer skill. Each scenario tests whether the skill produces correct, Fabric-specific semantic layer guidance.

## Scenarios

### Scenario 1: Semantic model with wrong entity types

**Prompt**: "Create a dbt semantic model YAML for an `fct_order_items` table on Fabric. Columns: order_item_id (PK), order_id (FK to fct_orders), product_id (FK to dim_products), customer_id (FK to dim_customers), quantity, unit_price, order_date. I need measures for total revenue, item count, and average unit price."

**Expected behavior**: Claude should produce a semantic model with:
- `order_item_id` as `type: primary` (grain of the table)
- `order_id`, `product_id`, `customer_id` as `type: foreign` (not primary or unique)
- Entity names that match the corresponding primary entities in other models (e.g., `order`, `product`, `customer` with `expr` mapping to the column)
- `defaults.agg_time_dimension: order_date`
- Revenue measure using `expr: "quantity * unit_price"` with `agg: sum`
- Item count as `agg: sum` with `expr: "1"` (not `agg: count`)
- Average unit price as `agg: average`
- T-SQL compatible expressions (no Postgres syntax)

**Pass criteria**:
1. All FK columns use `type: foreign`, not `type: primary` or `type: unique`
2. `defaults.agg_time_dimension` is set to a time dimension

### Scenario 2: Metric type selection

**Prompt**: "I need these metrics for my dbt project on Fabric: (1) total revenue, (2) average order value, (3) month-over-month revenue growth percentage, (4) month-to-date revenue, (5) percentage of orders that are high-value (over $500). What metric types should I use for each and write the YAML?"

**Expected behavior**: Claude should recommend and implement:
1. Total revenue: **simple** metric wrapping a sum measure
2. Average order value: **ratio** metric (revenue / order_count), NOT a derived metric with division
3. MoM revenue growth: **derived** metric using `offset_window: 1 month` with the growth formula
4. MTD revenue: **cumulative** metric with `grain_to_date: month`
5. Pct high-value orders: **derived** metric referencing a filtered simple metric (high_value_orders) and total order_count

**Pass criteria**:
1. Average order value uses `type: ratio` (not derived with manual division)
2. MoM growth uses `offset_window` in a derived metric (not a cumulative metric or manual SQL)

### Scenario 3: Semantic layer vs denormalized mart decision

**Prompt**: "We're building a dbt project on Fabric. Our analytics team wants a 'customer 360' view that combines customer profile, their order history metrics (total spend, order count, avg order value), product preferences, and support ticket counts. This data feeds a Power BI dashboard, a Jupyter notebook for the data science team, and an API for the customer success tool. Should I build a wide denormalized mart or use the semantic layer?"

**Expected behavior**: Claude should recommend the **semantic layer** approach because:
- Three consumers (Power BI, notebook, API) -- metric definitions must be consistent
- Multiple measures from different source models (orders, products, support tickets) -- MetricFlow handles the joins
- Different consumers likely need different dimension slices -- MetricFlow composes dynamically
- Should acknowledge the Fabric limitation: no hosted Semantic Layer API, so recommend saved query exports for Power BI and the API, and `dbt sl query` for notebooks
- Should NOT recommend a wide `dim_customer_360` mart as the primary approach

**Pass criteria**:
1. Recommends semantic layer over denormalized mart with reasoning about multi-consumer consistency
2. Mentions Fabric's lack of hosted Semantic Layer API and suggests saved query exports as a workaround

### Scenario 4: Multi-hop join configuration

**Prompt**: "I have fct_orders (order_id, customer_id, amount, order_date), dim_customers (customer_id, region_id, name), and dim_regions (region_id, region_name, country). I want to query total revenue grouped by region_name. Write the semantic model YAML for all three models."

**Expected behavior**: Claude should produce:
- Three semantic models with entity names that enable multi-hop joins: orders -> customers -> regions
- `customer` entity in both `fct_orders` (foreign) and `dim_customers` (primary) with matching names
- `region` entity in both `dim_customers` (foreign) and `dim_regions` (primary) with matching names
- Entity names use `expr` to map to actual column names when they differ
- A note that this is a two-hop join (within MetricFlow's limit)
- T-SQL compatible expressions throughout

**Pass criteria**:
1. Entity names match across models (e.g., `customer` in both fct_orders and dim_customers, not `customer_id` vs `customer`)
2. All three models have correct entity types (primary for the grain column, foreign for references)

### Scenario 5: Fabric-specific measure expression errors

**Prompt**: "Write a dbt semantic model for fct_transactions on Fabric with these measures: total amount, count of successful transactions (where status = 'success'), a is_weekend categorical dimension based on the transaction date, and the 95th percentile transaction amount."

**Expected behavior**: Claude should:
- Define total amount as `agg: sum`
- Define successful transaction count using `expr: "CASE WHEN status = 'success' THEN 1 ELSE 0 END"` with `agg: sum`
- Define is_weekend dimension using T-SQL: `expr: "CASE WHEN DATEPART(weekday, transaction_date) IN (1, 7) THEN 'Yes' ELSE 'No' END"` (not `EXTRACT(dow ...)` or `DAYOFWEEK()`)
- **Warn** that percentile aggregation (`agg: percentile`) is not supported on Fabric and suggest a workaround (compute in the mart SQL, or use `APPROX_PERCENTILE_CONT` in a pre-aggregated model)
- Not use boolean values (`TRUE`/`FALSE`) in any expression

**Pass criteria**:
1. Uses T-SQL `DATEPART(weekday, ...)` for the weekend dimension (not Postgres/Snowflake date functions)
2. Warns that percentile aggregation is unsupported on Fabric and offers an alternative approach
