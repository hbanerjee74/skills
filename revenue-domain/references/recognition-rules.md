# Recognition Rules

ASC 606 five-step model mapped to dbt transforms. Covers multi-element arrangements, variable consideration patterns, standalone selling price estimation, over-time measurement methods, and contract modification handling.

## Contents
- [Five-Step Pipeline in dbt](#five-step-pipeline-in-dbt)
- [Multi-Element Arrangements](#multi-element-arrangements)
- [Variable Consideration](#variable-consideration)
- [Standalone Selling Price Estimation](#standalone-selling-price-estimation)
- [Over-Time Recognition Methods](#over-time-recognition-methods)
- [Contract Modifications](#contract-modifications)
- [Period-End Close Patterns](#period-end-close-patterns)
- [Data Quality Rules by Layer](#data-quality-rules-by-layer)

## Five-Step Pipeline in dbt

The ASC 606 five-step model maps directly to a dbt DAG. Each step is a distinct intermediate or mart model.

```
Step 1          Step 2                    Step 3              Step 4               Step 5
─────────       ─────────────────         ──────────────      ──────────────       ──────────────
stg_erp__   →   int_performance_      →   int_transaction  →  int_price_       →   fct_revenue_
contracts       obligations               price               allocation           recognition
stg_erp__       (classify line items      (sum fixed +        (relative SSP        (apply schedule
contract_       into distinct             constrained         method per            per obligation
lines           obligations)              variable)           obligation)           per period)
```

**DAG dependency order**: Steps must run in sequence -- Step 4 depends on outputs from Steps 2 and 3, Step 5 depends on Step 4. Use `ref()` to enforce this; never bypass with direct table references.

## Multi-Element Arrangements

Contracts with multiple deliverables (license + services + support) are the hardest pattern. Claude defaults to treating each line item as a separate obligation -- this is often wrong.

### Identifying Distinct Obligations

Two criteria must BOTH be met for a promise to be a separate performance obligation:

1. **Capable of being distinct**: Customer can benefit from the good/service on its own or with readily available resources
2. **Distinct within the contract**: The promise is separately identifiable from other promises (not highly interrelated or significantly modified by other promises)

```sql
-- int_performance_obligations.sql
-- Decision tree for obligation classification
case
    -- Both criteria met → distinct obligation
    when p.is_standalone_capable = 1
        and cl.is_separately_identifiable = 1
    then cast(cl.contract_id as varchar) + '-' + cast(cl.line_number as varchar)

    -- Capable but not separately identifiable → combine with related items
    -- Group by the "primary" deliverable they modify/customize
    when p.is_standalone_capable = 1
        and cl.is_separately_identifiable = 0
    then cast(cl.contract_id as varchar) + '-' + cast(cl.bundle_group as varchar)

    -- Not capable of being distinct → always combine
    else cast(cl.contract_id as varchar) + '-' + cast(cl.bundle_group as varchar)
end as obligation_id
```

### Common Multi-Element Patterns

| Arrangement | Obligations | Why |
|---|---|---|
| SaaS license + implementation | **One** (usually) | Implementation significantly customizes/modifies the software -- not separately identifiable |
| SaaS license + training | **Two** | Training is generic, customer could hire a third party -- separately identifiable |
| Hardware + 3-year warranty | **One or two** | Standard warranty = one (assurance-type, not a separate obligation). Extended warranty = two (service-type, separate obligation) |
| Software license + PCS (post-contract support) | **Two** | PCS is distinct -- customer has the license regardless of whether they renew support |
| Consulting + deliverable report | **One** | Report is an output of the consulting, not separately valuable |

### Bundle Group Assignment

The `bundle_group` field in source data determines which line items combine into a single obligation. If the ERP doesn't provide this, derive it:

```sql
-- Derive bundle_group from product relationships
with line_items as (
    select
        contract_id,
        line_number,
        product_id,
        product_category,
        -- Primary items anchor the bundle
        case
            when product_category in ('license', 'hardware') then line_number
            -- Services/support attach to the nearest primary item above them
            else max(case when product_category in ('license', 'hardware')
                         then line_number end)
                 over (partition by contract_id
                       order by line_number
                       rows between unbounded preceding and current row)
        end as bundle_group
    from {{ ref('stg_erp__contract_lines') }}
    join {{ ref('dim_product') }} using (product_id)
)
```

## Variable Consideration

Transaction price often includes amounts that are uncertain at contract inception. ASC 606 requires estimation.

### Types of Variable Consideration

| Type | Example | Estimation Method | dbt Pattern |
|---|---|---|---|
| Volume discounts | "10% off if you buy >1000 units" | Expected value (probability-weighted) | Lookup against tier thresholds |
| Performance bonuses | "$50K bonus if delivered by March" | Most likely amount (binary outcome) | Single amount with probability gate |
| Penalties | "$10K/day for late delivery" | Most likely amount | Deduction from base price |
| Right of return | "30-day return policy" | Expected value using historical return rate | `base_amount * (1 - expected_return_rate)` |
| Price concessions | Implicit discount for long-term customers | Expected value | Historical concession analysis |

### Estimation Methods in SQL

**Expected value** -- use when there are multiple possible outcomes:

```sql
-- Variable consideration: expected value method
select
    contract_id,
    sum(outcome_amount * outcome_probability) as expected_variable_amount
from {{ ref('stg_erp__variable_outcomes') }}
group by contract_id
```

**Most likely amount** -- use when outcome is binary (bonus achieved or not):

```sql
-- Variable consideration: most likely amount method
select
    contract_id,
    case when achievement_probability > 0.5
         then bonus_amount
         else 0
    end as most_likely_variable_amount
from {{ ref('stg_erp__performance_bonuses') }}
```

### The Constraint

Variable consideration is included in the transaction price only to the extent that a significant reversal is NOT probable. This is a judgment call encoded as a business rule:

```sql
-- int_transaction_price.sql (constraint application)
select
    contract_id,
    base_amount,
    estimated_variable_amount,
    -- Apply constraint: only include if reversal is not probable
    case
        when reversal_risk_category = 'low' then estimated_variable_amount
        when reversal_risk_category = 'medium'
            then estimated_variable_amount * constraint_factor  -- e.g., 0.5
        when reversal_risk_category = 'high' then 0
    end as constrained_variable_amount,
    base_amount + constrained_variable_amount as transaction_price
from {{ ref('int_variable_consideration') }}
```

**`reversal_risk_category`** is typically set by finance during contract review and stored in the ERP. If not available, derive from historical data using a lookback of similar contracts.

## Standalone Selling Price Estimation

The allocation step requires a standalone selling price (SSP) for each obligation. ASC 606 prescribes a hierarchy of methods.

### SSP Method Hierarchy

```sql
-- int_standalone_selling_prices.sql
select
    product_id,
    case
        -- Level 1: Observable price (product sold standalone)
        when observable_ssp is not null
        then observable_ssp

        -- Level 2: Adjusted market assessment
        when competitor_price is not null
        then competitor_price * market_adjustment_factor

        -- Level 3: Expected cost plus margin
        when estimated_cost is not null
        then estimated_cost * (1 + target_margin_pct)

        -- Residual: only when SSP is highly variable/uncertain
        -- Handled separately in int_price_allocation
        else null  -- triggers residual allocation
    end as standalone_selling_price,
    case
        when observable_ssp is not null then 'observable'
        when competitor_price is not null then 'adjusted_market'
        when estimated_cost is not null then 'expected_cost_plus_margin'
        else 'residual'
    end as ssp_determination_method
from {{ ref('dim_product') }}
left join {{ ref('stg_erp__price_lists') }} using (product_id)
left join {{ ref('stg_erp__cost_estimates') }} using (product_id)
```

### Residual Allocation

When one obligation's SSP is highly variable, allocate it as the residual:

```sql
-- In int_price_allocation.sql, handle residual method
with non_residual as (
    select
        contract_id,
        sum(standalone_selling_price) as total_non_residual_ssp
    from obligations
    where ssp_determination_method != 'residual'
    group by contract_id
),
allocations as (
    select
        o.contract_id,
        o.obligation_id,
        case
            when o.ssp_determination_method = 'residual'
            then tp.transaction_price - nr.total_non_residual_ssp
            else o.standalone_selling_price
                / sum(case when o.ssp_determination_method != 'residual'
                           then o.standalone_selling_price end)
                  over (partition by o.contract_id)
                * (tp.transaction_price - coalesce(
                    (select transaction_price - total_non_residual_ssp
                     from non_residual where contract_id = o.contract_id), 0))
        end as allocated_amount
    from obligations o
    join transaction_prices tp on o.contract_id = tp.contract_id
    left join non_residual nr on o.contract_id = nr.contract_id
)
```

**Constraint**: Residual can only be used for ONE obligation per contract. If two obligations have uncertain SSPs, use the expected cost plus margin method for at least one.

## Over-Time Recognition Methods

When a performance obligation is satisfied over time, measure progress using output or input methods.

### Method Selection Decision Tree

```
Is the obligation satisfied over time?
├── YES: Does an output measure exist?
│   ├── YES: Use output method (units, milestones, surveys)
│   └── NO: Does input faithfully depict transfer of control?
│       ├── YES: Use input method (cost-to-cost, hours)
│       └── NO: Use time-elapsed (straight-line) as default
└── NO: Recognize at point in time (full amount when control transfers)
```

### Output Method: Milestones

```sql
-- int_recognition_schedule.sql (milestone-based)
select
    o.obligation_id,
    p.period_id,
    m.milestone_name,
    m.completion_date,
    m.milestone_weight / sum(m.milestone_weight) over (partition by o.obligation_id)
        as recognition_pct,
    'over_time_milestones' as recognition_method,
    sum(m.milestone_weight) over (
        partition by o.obligation_id
        order by m.completion_date
        rows between unbounded preceding and current row
    ) / sum(m.milestone_weight) over (partition by o.obligation_id)
        as cumulative_pct
from {{ ref('int_performance_obligations') }} o
join {{ ref('stg_erp__milestones') }} m on o.obligation_id = m.obligation_id
join {{ ref('dim_period') }} p
    on m.completion_date between p.period_start_date and p.period_end_date
where m.is_complete = 1
```

### Input Method: Cost-to-Cost

```sql
-- int_recognition_schedule.sql (cost-to-cost)
select
    o.obligation_id,
    p.period_id,
    ce.costs_incurred_to_date,
    ce.estimated_total_cost,
    ce.costs_incurred_to_date / nullif(ce.estimated_total_cost, 0)
        as cumulative_pct,
    -- Period recognition = change in cumulative %
    cumulative_pct - coalesce(
        lag(cumulative_pct) over (partition by o.obligation_id order by p.period_id),
        0
    ) as recognition_pct,
    'over_time_cost_to_cost' as recognition_method
from {{ ref('int_performance_obligations') }} o
join {{ ref('stg_erp__cost_estimates') }} ce on o.obligation_id = ce.obligation_id
join {{ ref('dim_period') }} p on ce.estimate_period = p.period_id
```

**Cost-to-cost gotcha**: If estimated total cost changes (scope change), the cumulative percentage is recalculated prospectively -- do NOT restate prior periods. The catch-up adjustment appears in the current period.

### Straight-Line (Time Elapsed)

```sql
-- int_recognition_schedule.sql (straight-line / SaaS default)
with obligation_periods as (
    select
        o.obligation_id,
        p.period_id,
        p.period_start_date,
        p.period_end_date,
        -- Calculate days of service in this period
        datediff(day,
            case when p.period_start_date > o.service_start_date
                 then p.period_start_date else o.service_start_date end,
            case when p.period_end_date < o.service_end_date
                 then p.period_end_date else o.service_end_date end
        ) + 1 as service_days_in_period,
        datediff(day, o.service_start_date, o.service_end_date) + 1
            as total_service_days
    from {{ ref('int_performance_obligations') }} o
    join {{ ref('dim_period') }} p
        on p.period_end_date >= o.service_start_date
        and p.period_start_date <= o.service_end_date
)
select
    obligation_id,
    period_id,
    cast(service_days_in_period as decimal(18,6))
        / cast(total_service_days as decimal(18,6)) as recognition_pct,
    'over_time_straight_line' as recognition_method,
    sum(cast(service_days_in_period as decimal(18,6))
        / cast(total_service_days as decimal(18,6)))
        over (partition by obligation_id order by period_id) as cumulative_pct
from obligation_periods
where service_days_in_period > 0
```

**Partial period handling**: A subscription starting mid-month recognizes a partial amount in the first period. Use day-count ratio, not `1 / total_months` -- months have different day counts.

## Contract Modifications

When a contract is amended after inception, the accounting treatment depends on the nature of the change.

### Modification Types

| Type | Treatment | dbt Pattern |
|---|---|---|
| **New distinct goods/services at SSP** | Treat as separate contract | New `contract_id` with its own obligations |
| **New distinct goods/services NOT at SSP** | Prospective: terminate old, create new | Close old obligations at modification date; create new obligations with remaining consideration |
| **Not distinct** (change to existing obligation) | Cumulative catch-up | Recalculate `recognition_pct` from inception; recognize adjustment in modification period |

### Modification in dbt

```sql
-- int_contract_modifications.sql
select
    cm.original_contract_id,
    cm.modification_contract_id,
    cm.modification_date,
    cm.modification_type,
    -- For prospective treatment: calculate remaining consideration
    case when cm.modification_type = 'prospective'
    then (select sum(allocated_amount - cumulative_recognized)
          from {{ ref('int_price_allocation') }} pa
          join {{ ref('fct_revenue_recognition') }} rr
              on pa.obligation_id = rr.obligation_id
          where pa.contract_id = cm.original_contract_id)
         + cm.additional_consideration
    end as remaining_consideration,
    -- For cumulative catch-up: recalculate from inception
    case when cm.modification_type = 'cumulative_catch_up'
    then cm.revised_transaction_price
    end as revised_total
from {{ ref('stg_erp__contract_modifications') }} cm
```

**Modification timing**: Process modifications in the period they occur. The `int_recognition_schedule` must detect modifications and adjust future recognition percentages without restating closed periods.

## Period-End Close Patterns

### Revenue Close Checklist (dbt jobs)

Run in this order at period end:

1. **`dbt build --select tag:revenue_bronze`** -- refresh staging models from ERP
2. **`dbt build --select tag:revenue_silver`** -- recalculate allocations, schedules
3. **`dbt build --select tag:revenue_gold`** -- materialize `fct_revenue_recognition`, `fct_deferred_revenue`
4. **`dbt test --select tag:revenue_reconciliation`** -- run reconciliation tests

### Reconciliation Tests

```yaml
# schema.yml — revenue reconciliation tests
models:
  - name: fct_revenue_recognition
    tests:
      - dbt_utils.expression_is_true:
          # Total recognized across all periods = total allocated
          expression: >
            (select sum(recognized_amount) from {{ ref('fct_revenue_recognition') }}
             where contract_id = '{{ var("test_contract_id") }}')
            =
            (select sum(allocated_amount) from {{ ref('int_price_allocation') }}
             where contract_id = '{{ var("test_contract_id") }}'
             and obligation_id in (
               select obligation_id from {{ ref('fct_revenue_recognition') }}
               where cumulative_pct = 1.0
             ))
```

### Key reconciliation assertions:

1. **Allocation = Transaction Price**: `SUM(allocated_amount)` per contract must equal `transaction_price`
2. **Recognition <= Allocation**: Cumulative recognized per obligation must never exceed allocated amount
3. **Deferred + Recognized = Billed** (for fully billed contracts): `deferred_revenue + cumulative_recognized = cumulative_billed`
4. **No recognition after obligation satisfied**: `cumulative_pct` must not exceed 1.0

## Data Quality Rules by Layer

### Bronze (staging)

| Rule | Test Type | Model |
|---|---|---|
| Contract ID not null | `not_null` | `stg_erp__contracts` |
| Contract dates valid | `expression_is_true` (`end >= start`) | `stg_erp__contracts` |
| Line amounts positive | `expression_is_true` (`amount > 0`) | `stg_erp__contract_lines` |
| No duplicate contract lines | `unique` on composite key | `stg_erp__contract_lines` |

### Silver (intermediate)

| Rule | Test Type | Model |
|---|---|---|
| Every line classified into an obligation | `relationships` (line → obligation) | `int_performance_obligations` |
| Allocation sums to transaction price | `expression_is_true` | `int_price_allocation` |
| Recognition percentages between 0 and 1 | `accepted_range` | `int_recognition_schedule` |
| Cumulative pct monotonically increasing | Custom test | `int_recognition_schedule` |

### Gold (marts)

| Rule | Test Type | Model |
|---|---|---|
| No negative recognized amounts | `expression_is_true` | `fct_revenue_recognition` |
| Cumulative pct <= 1.0 | `accepted_range` | `fct_revenue_recognition` |
| Deferred revenue reconciles | Custom test | `fct_deferred_revenue` |
| Period coverage complete (no gaps) | Custom test | `fct_revenue_recognition` |
