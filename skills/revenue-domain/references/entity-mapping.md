# Entity Mapping

Complete entity-to-model mapping for the revenue recognition domain. Covers every entity from source system to gold layer, including grain definitions, key strategies, SCD types, and relationship cardinalities.

## Contents
- [Entity Catalog](#entity-catalog)
- [Dimension Models](#dimension-models)
- [Fact Models](#fact-models)
- [Intermediate Models](#intermediate-models)
- [Relationship Map](#relationship-map)
- [Source System Field Mapping](#source-system-field-mapping)

## Entity Catalog

Every entity in the revenue domain, classified by model type and medallion layer.

| Entity | Model | Layer | Grain | Natural Key | Surrogate Key | SCD Type |
|---|---|---|---|---|---|---|
| Customer | `dim_customer` | Gold | One customer | `customer_id` | `customer_sk` | SCD2 |
| Product | `dim_product` | Gold | One sellable item | `product_id` | `product_sk` | SCD2 |
| Contract | `dim_contract` | Gold | One contract | `contract_id` | `contract_sk` | SCD2 |
| Accounting Period | `dim_period` | Gold | One fiscal period | `period_id` | -- (natural key) | SCD0 |
| Contract Line | `fct_contract_line` | Gold | One line per contract | `contract_id` + `line_number` | `contract_line_sk` | None (append) |
| Performance Obligation | `fct_performance_obligation` | Gold | One obligation per contract | `contract_id` + `obligation_id` | `obligation_sk` | None |
| Price Allocation | `fct_price_allocation` | Silver/Gold | One allocation per obligation | `contract_id` + `obligation_id` | -- | None |
| Revenue Recognition | `fct_revenue_recognition` | Gold | One event per obligation per period | `obligation_id` + `period_id` | -- | Incremental |
| Invoice Line | `fct_invoice_line` | Gold | One line per invoice | `invoice_id` + `line_number` | `invoice_line_sk` | None (append) |
| Deferred Revenue | `fct_deferred_revenue` | Gold | One balance per obligation per period | `obligation_id` + `period_id` | -- | Incremental |

## Dimension Models

### dim_customer

**Grain**: One row per customer. SCD2 -- tracks changes to customer attributes over time via dbt snapshot.

| Column | Type | Source | Notes |
|---|---|---|---|
| `customer_sk` | `int` (surrogate) | Generated | `generate_surrogate_key(['customer_id'])` |
| `customer_id` | `varchar` | CRM `accounts.id` | Natural key |
| `customer_name` | `varchar` | CRM `accounts.name` | |
| `customer_segment` | `varchar` | CRM `accounts.segment` | Enterprise / Mid-Market / SMB |
| `billing_country` | `varchar` | CRM `accounts.billing_country` | Needed for jurisdiction-specific recognition rules |
| `currency_code` | `varchar` | CRM `accounts.currency` | Contract currency; drives FX translation |
| `is_active` | `bit` | Derived | `contract_status NOT IN ('terminated', 'expired')` |
| `dbt_valid_from` | `datetime2` | Snapshot | SCD2 effective date |
| `dbt_valid_to` | `datetime2` | Snapshot | SCD2 expiration date |

**Why SCD2**: Customer segment and billing country affect revenue reporting breakdowns. Historical accuracy matters for period-over-period analysis.

### dim_product

**Grain**: One row per product or SKU. SCD2 -- price list changes and product reclassification need history.

| Column | Type | Source | Notes |
|---|---|---|---|
| `product_sk` | `int` (surrogate) | Generated | |
| `product_id` | `varchar` | Catalog `products.id` | Natural key |
| `product_name` | `varchar` | Catalog `products.name` | |
| `product_category` | `varchar` | Catalog `products.category` | License / Service / Support / Hardware |
| `recognition_pattern` | `varchar` | Catalog `products.rev_pattern` | `point_in_time` or `over_time` |
| `default_ssp` | `decimal(18,2)` | Price list | Standalone selling price for allocation |
| `ssp_determination_method` | `varchar` | Derived | `observable` / `adjusted_market` / `expected_cost_plus_margin` |
| `is_standalone_capable` | `bit` | Catalog | Can customer benefit from this alone? |

**Critical field**: `recognition_pattern` drives whether the obligation recognizes at a point or over time. This must come from the product master, not be inferred at query time.

### dim_contract

**Grain**: One row per contract. SCD2 -- contract amendments create new versions.

| Column | Type | Source | Notes |
|---|---|---|---|
| `contract_sk` | `int` (surrogate) | Generated | |
| `contract_id` | `varchar` | ERP `contracts.id` | Natural key |
| `customer_id` | `varchar` | ERP `contracts.customer_id` | FK to `dim_customer` |
| `contract_start_date` | `datetime2` | ERP | Use `datetime2`, not `date` -- same-day amendments |
| `contract_end_date` | `datetime2` | ERP | |
| `contract_term_months` | `int` | Derived | `DATEDIFF(month, start_date, end_date)` |
| `contract_status` | `varchar` | ERP | `draft` / `active` / `amended` / `terminated` / `expired` |
| `contract_currency` | `varchar` | ERP | |
| `amendment_of` | `varchar` | ERP | Points to original `contract_id` if this is an amendment |
| `total_contract_value` | `decimal(18,2)` | ERP | Sum of all line items (list price, before allocation) |

**Amendment handling**: When a contract is amended, the ERP creates a new contract record. Link via `amendment_of`. The original contract's status changes to `amended`. Both the original and amendment must have separate performance obligations and allocations.

### dim_period

**Grain**: One row per accounting period. SCD0 -- periods don't change.

| Column | Type | Source | Notes |
|---|---|---|---|
| `period_id` | `varchar` | Generated | Format: `YYYY-MM` (e.g., `2025-01`) |
| `period_start_date` | `date` | Derived | First day of period |
| `period_end_date` | `date` | Derived | Last day of period |
| `fiscal_year` | `int` | Derived | May differ from calendar year |
| `fiscal_quarter` | `int` | Derived | Q1-Q4 per fiscal calendar |
| `is_closed` | `bit` | ERP GL calendar | Whether period is closed for posting |

**Fiscal calendar alignment**: The period dimension must align with the entity's fiscal calendar, not calendar months. A company with Feb 1 fiscal year start has Q1 = Feb-Apr. This affects all period-based recognition calculations.

## Fact Models

### fct_revenue_recognition

**Grain**: One recognition event per performance obligation per accounting period. This is the central fact table.

| Column | Type | Source | Notes |
|---|---|---|---|
| `obligation_id` | `varchar` | `int_price_allocation` | PK part 1 |
| `period_id` | `varchar` | `int_recognition_schedule` | PK part 2 |
| `contract_id` | `varchar` | `int_price_allocation` | FK to `dim_contract` |
| `customer_id` | `varchar` | `dim_contract` | Denormalized for query performance |
| `product_id` | `varchar` | `int_performance_obligations` | FK to `dim_product` |
| `recognition_date` | `date` | Schedule | Date within the period when revenue is recognized |
| `recognized_amount` | `decimal(18,2)` | Calculated | `allocated_amount * recognition_pct` |
| `total_obligation_amount` | `decimal(18,2)` | `int_price_allocation` | For percentage calculations |
| `recognition_method` | `varchar` | `dim_product` | `point_in_time` / `over_time` |
| `recognition_pct` | `decimal(8,6)` | Schedule | Percentage of obligation recognized this period |
| `cumulative_pct` | `decimal(8,6)` | Schedule | Running total of recognition percentage |
| `currency_code` | `varchar` | `dim_contract` | |

**Materialization**: Incremental with `merge` on `['obligation_id', 'period_id']`. Late-period adjustments overwrite prior recognition amounts for the same obligation+period.

### fct_invoice_line

**Grain**: One line item per invoice. Tracks billing events separately from recognition.

| Column | Type | Source | Notes |
|---|---|---|---|
| `invoice_id` | `varchar` | ERP AR | PK part 1 |
| `line_number` | `int` | ERP AR | PK part 2 |
| `contract_id` | `varchar` | ERP AR | FK to `dim_contract` |
| `obligation_id` | `varchar` | Derived | Mapped from contract line to obligation |
| `invoice_date` | `date` | ERP AR | When billed |
| `due_date` | `date` | ERP AR | Payment due |
| `billed_amount` | `decimal(18,2)` | ERP AR | Amount invoiced |
| `currency_code` | `varchar` | ERP AR | |

**Key distinction**: `billed_amount` (invoiced) is NOT `recognized_amount`. A 12-month SaaS subscription billed annually has one `fct_invoice_line` row for the full amount but twelve `fct_revenue_recognition` rows (one per month).

### fct_deferred_revenue

**Grain**: One balance snapshot per obligation per period.

| Column | Type | Source | Notes |
|---|---|---|---|
| `obligation_id` | `varchar` | `int_price_allocation` | PK part 1 |
| `period_id` | `varchar` | `dim_period` | PK part 2 |
| `cumulative_billed` | `decimal(18,2)` | `fct_invoice_line` | Total billed through this period |
| `cumulative_recognized` | `decimal(18,2)` | `fct_revenue_recognition` | Total recognized through this period |
| `deferred_revenue_balance` | `decimal(18,2)` | Calculated | `cumulative_billed - cumulative_recognized` |
| `total_obligation_amount` | `decimal(18,2)` | `int_price_allocation` | |

**Negative balance**: When `deferred_revenue_balance < 0`, the entity has a contract asset (recognized more than billed). This is valid for over-time obligations where service delivery leads billing.

## Intermediate Models

These live in silver. They implement business logic and are consumed by gold models.

| Model | Purpose | Consumes | Produces |
|---|---|---|---|
| `int_performance_obligations` | Classify line items into distinct obligations | `stg_erp__contract_lines`, `dim_product` | `obligation_id`, `obligation_type` |
| `int_standalone_selling_prices` | Determine SSP per product using appropriate method | `dim_product`, `stg_erp__price_lists` | `standalone_selling_price`, `ssp_method` |
| `int_transaction_price` | Calculate total consideration including variable amounts | `stg_erp__contracts`, `stg_erp__variable_outcomes` | `transaction_price` per contract |
| `int_price_allocation` | Allocate transaction price to obligations by relative SSP | `int_performance_obligations`, `int_transaction_price`, `int_standalone_selling_prices` | `allocated_amount` per obligation |
| `int_recognition_schedule` | Generate period-by-period recognition percentages | `int_price_allocation`, `dim_product`, `dim_period` | `recognition_pct`, `cumulative_pct` per obligation per period |
| `int_billing_schedule` | Map invoices to obligations and accumulate billed amounts | `fct_invoice_line`, `int_performance_obligations` | `cumulative_billed` per obligation per period |
| `int_contract_position` | Calculate contract asset/liability position | `fct_revenue_recognition`, `int_billing_schedule` | `net_position`, `position_type` |

## Relationship Map

```
dim_customer ──1:M──▶ dim_contract ──1:M──▶ fct_contract_line
                           │                        │
                           │                   (classified into)
                           │                        ▼
                           ├──1:M──▶ int_performance_obligations
                           │                        │
                           │                   (allocated via)
                           │                        ▼
                           ├──1:1──▶ int_transaction_price ──▶ int_price_allocation
                           │                                          │
                           │                                     (scheduled)
                           │                                          ▼
                           │                                 int_recognition_schedule
                           │                                          │
                           │                                     (materialized)
                           │                                          ▼
                           ├──────────────────────────▶ fct_revenue_recognition
                           │                                          │
                           └──1:M──▶ fct_invoice_line          (balanced against)
                                          │                           │
                                     (accumulated)                    ▼
                                          └──────────▶ fct_deferred_revenue

dim_product ──referenced by──▶ fct_contract_line, int_performance_obligations
dim_period  ──referenced by──▶ fct_revenue_recognition, fct_deferred_revenue
```

**Cardinality rules**:
- One customer has many contracts (1:M)
- One contract has many line items (1:M)
- One contract has many performance obligations (1:M), but NOT 1:1 with line items -- combined obligations collapse multiple lines
- One obligation has one price allocation (1:1)
- One obligation has many recognition events over time (1:M, one per period)
- One invoice line maps to one obligation (M:1 -- multiple invoice lines can fund one obligation)

## Source System Field Mapping

### ERP Contract Module

| ERP Field | Maps To | Transform |
|---|---|---|
| `contracts.contract_number` | `dim_contract.contract_id` | Cast to varchar, trim |
| `contracts.bill_to_account` | `dim_contract.customer_id` | Join to CRM account mapping |
| `contracts.effective_date` | `dim_contract.contract_start_date` | Cast to `datetime2` |
| `contracts.expiration_date` | `dim_contract.contract_end_date` | Cast to `datetime2` |
| `contracts.total_value` | `dim_contract.total_contract_value` | `decimal(18,2)` |
| `contract_lines.line_num` | `fct_contract_line.line_number` | Cast to int |
| `contract_lines.item_id` | `fct_contract_line.product_id` | Join to product catalog |
| `contract_lines.unit_price` | `fct_contract_line.list_price` | `decimal(18,2)` |
| `contract_lines.quantity` | `fct_contract_line.quantity` | `decimal(18,4)` |

### ERP Accounts Receivable

| ERP Field | Maps To | Transform |
|---|---|---|
| `invoices.invoice_num` | `fct_invoice_line.invoice_id` | Cast to varchar |
| `invoices.invoice_date` | `fct_invoice_line.invoice_date` | Cast to date |
| `invoice_lines.line_amount` | `fct_invoice_line.billed_amount` | `decimal(18,2)` |
| `invoice_lines.contract_ref` | `fct_invoice_line.contract_id` | Join to contract mapping |

### CRM

| CRM Field | Maps To | Transform |
|---|---|---|
| `accounts.account_id` | `dim_customer.customer_id` | Cast to varchar |
| `accounts.account_name` | `dim_customer.customer_name` | Trim, title case |
| `accounts.segment` | `dim_customer.customer_segment` | Map to standard segments |
| `accounts.country` | `dim_customer.billing_country` | ISO 3166-1 alpha-2 |
