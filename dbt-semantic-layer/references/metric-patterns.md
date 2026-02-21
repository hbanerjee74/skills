# MetricFlow Metric Patterns

Metric type definitions, decision framework, filter syntax, and advanced patterns for dbt MetricFlow on Fabric. Covers what LLMs get wrong: when to use each metric type, how filters interact with metric types, and Fabric-compatible expression patterns.

## Contents
- [Metric Type Decision Framework](#metric-type-decision-framework)
- [Simple Metrics](#simple-metrics)
- [Ratio Metrics](#ratio-metrics)
- [Derived Metrics](#derived-metrics)
- [Cumulative Metrics](#cumulative-metrics)
- [Conversion Metrics](#conversion-metrics)
- [Filters](#filters)
- [Advanced Patterns](#advanced-patterns)

## Metric Type Decision Framework

Choose the metric type based on what you are computing, not how complex it feels:

| Question | Answer | Metric type |
|---|---|---|
| Is it a single measure, optionally filtered? | Yes | **Simple** |
| Is it one measure divided by another? | Yes | **Ratio** |
| Is it a formula combining 2+ metrics? | Yes | **Derived** |
| Does it accumulate over a time window (running total, MTD, YTD)? | Yes | **Cumulative** |
| Does it track a base event followed by a conversion event? | Yes | **Conversion** |

**Common mistakes**:
- Using **derived** when **ratio** suffices. If the formula is just `A / B`, use ratio -- it handles null denominators and zero-division correctly. Derived does not.
- Using **simple** with a complex `expr` when **derived** is clearer. If you need `revenue - cost`, define `revenue` and `cost` as simple metrics, then create a derived metric. Don't put the subtraction in a measure `expr`.
- Using **cumulative** for a simple sum. Cumulative is for running totals. If you want total revenue for a given period, that's a simple metric -- MetricFlow handles the time filtering.

## Simple Metrics

A simple metric wraps a single measure with optional filters. This is the most common type.

### Basic simple metric

```yaml
metrics:
  - name: revenue
    description: "Total order revenue"
    type: simple
    label: "Revenue"
    type_params:
      measure: order_total
```

### Simple metric with filter

```yaml
metrics:
  - name: online_revenue
    description: "Revenue from online orders only"
    type: simple
    label: "Online Revenue"
    type_params:
      measure: order_total
    filter: |
      {{ Dimension('order__channel') }} = 'online'
```

### When to use

- Any time you expose a measure as a queryable metric
- When you need to apply a static filter to a measure (e.g., "revenue from US only")
- As building blocks for ratio and derived metrics

## Ratio Metrics

A ratio metric divides one measure (numerator) by another (denominator). MetricFlow handles null denominators by returning null, not dividing by zero.

### Basic ratio

```yaml
metrics:
  - name: avg_order_value
    description: "Average value per order"
    type: ratio
    label: "Average Order Value"
    type_params:
      numerator:
        name: order_total
      denominator:
        name: order_count
```

### Ratio with filtered components

```yaml
metrics:
  - name: online_conversion_rate
    description: "Percentage of online visits that result in an order"
    type: ratio
    label: "Online Conversion Rate"
    type_params:
      numerator:
        name: order_count
        filter: |
          {{ Dimension('order__channel') }} = 'online'
      denominator:
        name: visit_count
        filter: |
          {{ Dimension('visit__channel') }} = 'online'
```

### When to use

- Averages: revenue / count
- Rates: conversions / visits
- Percentages: category_revenue / total_revenue
- Any `A / B` calculation where A and B are measures

**Prefer ratio over derived for division**. Ratio metrics get special handling for zero denominators. A derived metric with `expr: A / B` will error on division by zero unless you add explicit null-handling.

## Derived Metrics

A derived metric applies a formula to other metrics. Use when the calculation involves more than simple division.

### Basic derived

```yaml
metrics:
  - name: gross_profit
    description: "Revenue minus cost of goods sold"
    type: derived
    label: "Gross Profit"
    type_params:
      expr: revenue - cogs
      metrics:
        - name: revenue
        - name: cost_of_goods_sold
          alias: cogs
```

### Derived with filtered inputs

```yaml
metrics:
  - name: food_gross_profit
    description: "Gross profit from food orders only"
    type: derived
    label: "Food Gross Profit"
    type_params:
      expr: food_revenue - food_cost
      metrics:
        - name: revenue
          alias: food_revenue
          filter: |
            {{ Dimension('order__is_food_order') }} = 1
        - name: cost_of_goods_sold
          alias: food_cost
          filter: |
            {{ Dimension('order__is_food_order') }} = 1
```

Note: On Fabric, filter on `= 1` (BIT), not `= True` (boolean).

### Period-over-period with offset_window

```yaml
metrics:
  - name: revenue_growth_mom
    description: "Month-over-month revenue growth percentage"
    type: derived
    label: "Revenue Growth % (M/M)"
    type_params:
      expr: "(current_revenue - prior_revenue) * 100.0 / prior_revenue"
      metrics:
        - name: revenue
          alias: current_revenue
        - name: revenue
          offset_window: 1 month
          alias: prior_revenue
```

`offset_window` shifts the metric back in time. This is how you build period-over-period comparisons without complex SQL.

### When to use

- Arithmetic on metrics: profit = revenue - cost
- Period-over-period comparisons using `offset_window`
- Complex formulas: `(A - B) / C * 100`
- Combining metrics from different semantic models

**Do not use for simple division** -- use ratio instead.

## Cumulative Metrics

Cumulative metrics accumulate a measure over time. Three variants:

### All-time cumulative (running total)

```yaml
metrics:
  - name: cumulative_revenue
    description: "All-time cumulative revenue"
    type: cumulative
    label: "Cumulative Revenue (All Time)"
    type_params:
      measure: order_total
```

No `window` or `grain_to_date` -- accumulates from the beginning of data.

### Trailing window (rolling)

```yaml
metrics:
  - name: revenue_trailing_30d
    description: "Revenue over the trailing 30 days"
    type: cumulative
    label: "Revenue (Trailing 30 Days)"
    type_params:
      measure: order_total
      cumulative_type_params:
        window: 30 days
```

### Grain-to-date (MTD, QTD, YTD)

```yaml
metrics:
  - name: revenue_mtd
    description: "Month-to-date revenue"
    type: cumulative
    label: "Revenue (MTD)"
    type_params:
      measure: order_total
      cumulative_type_params:
        grain_to_date: month
        period_agg: first     # or 'last' -- controls how partial periods aggregate
```

`grain_to_date` resets the accumulator at the start of each calendar period (month, quarter, year).

### When to use

- Running totals (all-time customer count)
- Rolling windows (trailing 90-day revenue)
- Period-to-date calculations (MTD, YTD)
- Any metric where "the value on Jan 15 includes all data from Jan 1-15"

**Common mistake**: Using cumulative for a simple sum. If you want "total revenue in January," that's a simple metric queried with a January time filter. Cumulative is for when the Jan 15 value includes Jan 1-14 data.

### Null handling with time spine

When there are gaps in the data (days with no orders), cumulative metrics may show nulls. Use `fill_nulls_with` and `join_to_timespine`:

```yaml
metrics:
  - name: revenue_mtd
    type: cumulative
    type_params:
      measure:
        name: order_total
        fill_nulls_with: 0
        join_to_timespine: true
      cumulative_type_params:
        grain_to_date: month
```

## Conversion Metrics

Conversion metrics track the rate at which a base event leads to a conversion event within a time window. Both events must share a common entity.

### Basic conversion

```yaml
metrics:
  - name: signup_to_purchase
    description: "Rate of signups that result in a purchase within 7 days"
    type: conversion
    label: "Signup-to-Purchase Conversion"
    type_params:
      conversion_type_params:
        base_measure: signups
        conversion_measure: purchases
        entity: user_id
        window: 7 days
        calculation: conversion_rate   # Returns a rate (0-1)
```

### With constant properties

`constant_properties` filter the conversion event by specific attribute values:

```yaml
metrics:
  - name: signup_to_pro_purchase
    description: "Rate of signups that purchase a Pro plan within 30 days"
    type: conversion
    label: "Signup-to-Pro Conversion"
    type_params:
      conversion_type_params:
        base_measure: signups
        conversion_measure: purchases
        entity: user_id
        window: 30 days
        calculation: conversion_rate
        constant_properties:
          - base_property: Dimension('signup__signup_channel')
            conversion_property: Dimension('purchase__purchase_channel')
```

### When to use

- Funnel analysis: visit -> signup -> purchase
- Feature adoption: signup -> first use within N days
- Any "did event A lead to event B within a time window" question

**Critical requirement**: The `entity` must exist in **both** the semantic model containing the base measure and the semantic model containing the conversion measure. If the entity names don't match, MetricFlow cannot correlate the events.

**Fabric consideration**: Conversion metrics generate complex SQL with window functions and self-joins. Test the generated SQL on Fabric to verify T-SQL compatibility before relying on conversion metrics in production.

## Filters

### Filter syntax

Filters use Jinja templates referencing dimensions, time dimensions, entities, or metrics:

```yaml
# Dimension filter
filter: |
  {{ Dimension('order__channel') }} = 'online'

# Time dimension filter
filter: |
  {{ TimeDimension('order__order_date', 'month') }} >= '2024-01-01'

# Entity filter
filter: |
  {{ Entity('customer') }} IS NOT NULL

# Metric-as-filter (subquery)
filter: |
  {{ Metric('lifetime_order_count', group_by=['customer']) }} > 5
```

### Fabric-safe filter expressions

When writing filter expressions, use T-SQL syntax:

```yaml
# CORRECT for Fabric
filter: |
  {{ Dimension('customer__signup_source') }} IN ('organic', 'referral')

# CORRECT -- T-SQL string concatenation
filter: |
  {{ Dimension('order__status') }} + '_' + {{ Dimension('order__channel') }} = 'completed_online'

# WRONG on Fabric -- Postgres concat
filter: |
  {{ Dimension('order__status') }} || '_' || {{ Dimension('order__channel') }} = 'completed_online'
```

### Where filters apply

| Metric type | Filter on metric | Filter on component measures |
|---|---|---|
| Simple | Yes -- filters the measure | N/A (one measure) |
| Ratio | Yes -- filters the final result | Yes -- filter numerator/denominator independently |
| Derived | Yes -- filters the final result | Yes -- filter individual input metrics via `metrics[].filter` |
| Cumulative | Yes -- filters before accumulation | N/A |
| Conversion | Limited | Use `constant_properties` for conversion-side filters |

## Advanced Patterns

### Metrics referencing metrics in filters

You can filter a metric based on the value of another metric. This creates a subquery:

```yaml
metrics:
  - name: high_value_customer_revenue
    description: "Revenue from customers with lifetime orders > 10"
    type: simple
    type_params:
      measure: order_total
    filter: |
      {{ Metric('lifetime_order_count', group_by=['customer']) }} > 10
```

MetricFlow generates a subquery that computes `lifetime_order_count` per customer, then filters the outer query. This is powerful but can be slow on large datasets -- monitor Fabric query performance.

### Shared dimensions across metrics in saved queries

When combining multiple metrics in a saved query, you can only group by dimensions that are shared across all referenced metrics. If `revenue` comes from `fct_orders` (which has `channel`) and `signups` comes from `fct_signups` (which does not have `channel`), you cannot group by `channel` in a saved query that includes both.

**Solution**: Ensure all metrics in a saved query share the dimensions you want to group by. This often means adding entity relationships so both semantic models connect to the same dimension model.

### Versioning metrics

When metric definitions change (e.g., "revenue" now excludes refunds), use metric versioning to avoid breaking downstream consumers:

1. Create a new metric (`revenue_v2`) with the updated definition
2. Mark the old metric with `deprecation_date` in meta
3. Migrate consumers to the new metric
4. Remove the old metric after the deprecation date

MetricFlow does not have built-in versioning -- manage it through naming conventions and documentation.
