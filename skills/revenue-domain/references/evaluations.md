# Evaluations

Test scenarios for the revenue-domain skill. Each scenario tests whether the skill produces correct, domain-specific guidance for modeling revenue recognition in dbt on Fabric.

## Scenarios

### Scenario 1: Multi-element SaaS contract modeling

**Prompt**: "I have a SaaS contract that includes a software license, implementation services, and 12 months of post-contract support. The implementation significantly customizes the software. Model this in dbt using the medallion architecture on Fabric."

**Expected behavior**: Claude should:
- Identify TWO performance obligations, not three: (1) software license + implementation combined (because implementation significantly customizes the software, making them not separately identifiable), and (2) post-contract support (distinct -- customer has the software regardless of support renewal)
- Place obligation classification logic in an `int_performance_obligations` model in silver, not in staging or gold
- Allocate the transaction price to the two obligations using relative standalone selling prices in `int_price_allocation`
- Recognize the combined license+implementation obligation over time using cost-to-cost or milestone method (not point-in-time, since implementation is over time)
- Recognize support obligation over time using straight-line (time elapsed) method over 12 months
- Use `fct_revenue_recognition` with grain of `obligation_id` + `period_id`
- Use incremental materialization with `merge` strategy and composite `unique_key`

**Pass criteria**:
1. Correctly identifies two obligations (not three) with explicit reasoning that implementation is not separately identifiable from the license
2. Uses different recognition methods for each obligation (cost-to-cost or milestones for combined, straight-line for support)

### Scenario 2: Deferred revenue balance calculation

**Prompt**: "Write a dbt model for deferred revenue on Fabric. We have a `fct_revenue_recognition` table and a billing table. I need to see the deferred revenue balance per obligation per month, and identify contract assets vs contract liabilities."

**Expected behavior**: Claude should:
- Build `fct_deferred_revenue` with grain of `obligation_id` + `period_id`
- Calculate `deferred_revenue_balance` as `cumulative_billed - cumulative_recognized` (NOT `total_contract_value - recognized`)
- Explain that negative balance means contract asset (recognized > billed), positive means contract liability (billed > recognized)
- Use incremental materialization with `merge` on `['obligation_id', 'period_id']`
- Include a lookback window in the `is_incremental()` block using `DATEADD()` (T-SQL, not `INTERVAL`)
- Reference `dim_period` for period boundaries rather than extracting months from dates

**Pass criteria**:
1. Formula uses `cumulative_billed - cumulative_recognized` (not total contract value minus recognized)
2. Correctly defines contract asset (negative deferred) vs contract liability (positive deferred)

### Scenario 3: Variable consideration with constraint

**Prompt**: "A consulting contract has a $500K base fee plus a $100K performance bonus if the project is delivered by March 31. Historical data shows 70% of similar projects meet the deadline. How should I model the transaction price in dbt?"

**Expected behavior**: Claude should:
- Use the "most likely amount" method (binary outcome: bonus achieved or not), not the expected value method
- Since 70% > 50%, the most likely outcome is receiving the bonus, so include $100K
- Apply the constraint test: is a significant reversal probable? At 70% achievement rate, this is a judgment call -- Claude should flag this as requiring finance input
- Model this in `int_transaction_price` (silver), not in gold
- Transaction price = $600K if constraint is met, or $500K if finance determines reversal risk is too high
- Store the constraint decision and method as auditable fields (`variable_consideration_method`, `reversal_risk_category`)

**Pass criteria**:
1. Selects "most likely amount" method (not expected value) with reasoning about binary outcome
2. Flags the constraint as a judgment requiring finance input rather than automatically including or excluding the $100K

### Scenario 4: Contract modification -- prospective treatment

**Prompt**: "Six months into a 12-month SaaS subscription ($120K total, $10K/month recognized), the customer upgrades to a higher tier adding $60K for the remaining 6 months. The upgrade is a distinct service at standalone selling price. How do I handle this in my dbt revenue models?"

**Expected behavior**: Claude should:
- Identify this as a modification that qualifies as a separate contract (new distinct goods/services at SSP)
- NOT restate the first 6 months of recognition ($60K already recognized stays)
- Create a new contract/obligation for the upgrade: $60K allocated over 6 remaining months ($10K/month)
- The original obligation continues at $10K/month for the remaining 6 months
- Total revenue for the full period: $60K (original, months 1-6) + $60K (original, months 7-12) + $60K (upgrade, months 7-12) = $180K
- Model using `int_contract_modifications` to detect the modification type and route to the correct treatment

**Pass criteria**:
1. Treats the modification as a separate contract (does not restate prior periods or do a cumulative catch-up)
2. Correctly calculates $10K/month original + $10K/month upgrade = $20K/month for months 7-12

### Scenario 5: Revenue reconciliation tests

**Prompt**: "What dbt tests should I write to validate my revenue recognition models? I want to catch allocation errors, over-recognition, and deferred revenue mismatches."

**Expected behavior**: Claude should recommend specific, named tests (not vague "test your data" advice):
- **Allocation balance test**: `SUM(allocated_amount)` per contract equals `transaction_price` in `int_price_allocation`
- **Over-recognition guard**: `cumulative_pct <= 1.0` on `fct_revenue_recognition` using `accepted_range` or `expression_is_true`
- **Recognition <= allocation**: cumulative `recognized_amount` per obligation never exceeds `allocated_amount`
- **Deferred revenue reconciliation**: for fully-billed obligations, `deferred_revenue + cumulative_recognized = cumulative_billed`
- **No recognition after satisfaction**: no rows in `fct_revenue_recognition` for periods after `cumulative_pct = 1.0`
- **Period completeness**: no gaps in the recognition schedule (every period between obligation start and end has a row)
- Tests should use `dbt_utils.expression_is_true` or custom generic tests, not ad-hoc queries

**Pass criteria**:
1. Includes at least 4 specific test definitions with exact expressions or test types
2. Includes the allocation balance test (sum of allocations = transaction price)
