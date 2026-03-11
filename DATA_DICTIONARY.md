---
title: Runway -- Data Dictionary
---

# Data Dictionary

Documents all persisted entities and computed models for the Runway debt simulation engine.

---

## Persisted Entities

### `scenarios`

A named container for a full set of plan inputs. MVP ships with one active scenario. Toggling a scenario to active is a status change only — no data migration required, as each scenario's projection rows are already persisted in `scenario_projections`.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `name` | `text` | No | Display name, e.g. "Base Plan", "Aggressive Payoff Draft" |
| `status` | `text` | No | `ACTIVE`, `DRAFT`, or `ARCHIVED`. Only one scenario should be `ACTIVE` at a time. |
| `initial_pool_balance` | `numeric(10,2)` | No | Starting cash balance for the liquidity pool. Default `0.00`. See `Implementation_Appendix.md` §A2. |
| `created_at` | `timestamptz` | No | |
| `updated_at` | `timestamptz` | No | |

**Cloning behavior (P1):** duplicates `plan_settings`, `lump_sums`, `card_payment_schedules`, and `scenario_projections` rows under a new `scenario_id`. Cards are global and are not copied.

---

### `cards`

Global pool of credit lines. All cards are shared across scenarios. P1/P2 may add per-scenario card inclusion/exclusion.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `name` | `text` | No | Display name |
| `balance` | `numeric(12,2)` | No | Current balance at time of last save; used as engine starting point for cards without a confirmed ledger entry |
| `apr` | `numeric(6,4)` | No | Decimal form, e.g. `0.2799` |
| `credit_limit` | `numeric(12,2)` | No | Used for utilization calculation |
| `payment_due_day` | `integer` | No | Day of month (1–31); used by engine for paycheck split logic |
| `active` | `boolean` | No | Soft delete flag. Default `true`. Inactive cards are excluded from engine inputs and UI. |
| `created_at` | `timestamptz` | No | |
| `updated_at` | `timestamptz` | No | |

---

### `loans`

Global pool of installment debt (auto loans, student loans, personal loans, mortgages). Not scenario-scoped. Soft-deleted via the `active` flag.

The engine treats loan payments as committed costs that reduce disposable income each month. Interest is not calculated — it is already amortized into `monthly_payment`. The engine tracks balance to zero and records payoff month.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `name` | `text` | No | e.g. "Car Loan", "Student Loan" |
| `balance` | `numeric(12,2)` | No | Current balance at time of last save |
| `monthly_payment` | `numeric(10,2)` | No | Contractual fixed monthly payment |
| `end_date` | `date` | No | Month of final scheduled payment. Engine stops deducting `monthly_payment` after this month. |
| `apr` | `numeric(6,4)` | Yes | Informational only — not used by engine. Interest is already amortized into the payment. |
| `active` | `boolean` | No | Soft delete. Default `true`. Inactive loans are excluded from engine inputs and UI. |
| `created_at` | `timestamptz` | No | |
| `updated_at` | `timestamptz` | No | |

**Engine resolution rule:** each month, if `month <= loan.endDate` AND `balance > 0`, deduct `monthly_payment` (capped at balance) from balance. When balance hits zero or end date is reached, the loan is done and the freed `monthly_payment` flows into the extra pool available for card payoff.

---

### `card_payment_schedules`

Defines the payment policy for a card within a scenario. Scenario-specific so different scenarios can model different payment strategies per card.

MVP: one active row per `(scenario_id, card_id)`. P1: multiple rows with non-overlapping date ranges to support mid-plan payment mode changes.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `scenario_id` | `uuid` | No | FK → `scenarios` |
| `card_id` | `uuid` | No | FK → `cards` |
| `effective_month_start` | `date` | No | First month this schedule applies (inclusive). MVP: set to plan start date as sentinel. |
| `effective_month_end` | `date` | Yes | Last month this schedule applies (inclusive). `NULL` = currently active. |
| `base_payment_mode` | `text` | No | `MINIMUM` or `FIXED` |
| `base_payment_amount` | `numeric(10,2)` | No | Manual entry; required for both modes |

**Engine resolution rule:** for a given card and month, use the row where `effective_month_start <= month` AND (`effective_month_end IS NULL` OR `effective_month_end >= month`).

**Constraint:** only one row per `(scenario_id, card_id)` may have `effective_month_end = NULL` at a time. Enforced via partial unique index.

---

### ~~`plan_settings`~~ — DEPRECATED

> **Replaced by `allocation_plans` + `allocations`.** See `Implementation_Appendix.md` §A1. This table has been dropped.

---

### `allocation_plans`

Time-ranged allocation strategies for a scenario. The engine uses the plan whose date range covers the month being projected. One open-ended (null `effective_month_end`) plan is active at any time.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `scenario_id` | `uuid` | No | FK → `scenarios` ON DELETE CASCADE |
| `effective_month_start` | `date` | No | First month this plan applies (inclusive, stored as YYYY-MM-01) |
| `effective_month_end` | `date` | Yes | Last month this plan applies (inclusive). `NULL` = currently active. |

**Engine resolution rule:** for a given month, use the plan where `effective_month_start <= month AND (effective_month_end IS NULL OR effective_month_end >= month)`.

**Constraint:** only one row per `scenario_id` may have `effective_month_end = NULL` at a time (partial unique index).

---

### `allocations`

Child rows of `allocation_plans`. Each row directs a fixed monthly dollar amount to a specific card or the liquidity pool.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `plan_id` | `uuid` | No | FK → `allocation_plans` ON DELETE CASCADE |
| `target` | `text` | No | `CARD`, `POOL`, `BASE_POOL`, or `LOAN` |
| `card_id` | `uuid` | Yes | FK → `cards`. Required when `target = CARD`, null otherwise. |
| `loan_id` | `uuid` | Yes | FK → `loans`. Required when `target = LOAN`, null otherwise. |
| `amount` | `numeric(10,2)` | No | Monthly dollar amount to allocate |

---

### `lump_sums`

One-off payment events applied in a specific month before regular payment allocation. Scenario-specific.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `scenario_id` | `uuid` | No | FK → `scenarios` |
| `apply_date` | `date` | No | Month the lump sum is applied; day component ignored by engine |
| `amount` | `numeric(10,2)` | No | |
| `target` | `text` | No | `FOCUS`, `SPECIFIC_CARD`, or `SPECIFIC_LOAN` |
| `target_card_id` | `uuid` | Yes | FK → `cards`. Required when `target = SPECIFIC_CARD`, null otherwise. |
| `target_loan_id` | `uuid` | Yes | FK → `loans`. Required when `target = SPECIFIC_LOAN`, null otherwise. |
| `created_at` | `timestamptz` | No | |

---

### `fixed_expenses`

Global pool of mandatory, non-discretionary expenses. Not scenario-scoped — the same across all scenarios. Soft-deleted via the `active` flag.

The engine sums amounts applicable to a given month (where `(month - anchor_month) % billing_interval == 0`) to derive `monthlyNonDebtExpenses` for that month. This varies month to month.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `name` | `text` | No | e.g. "Mortgage", "Utilities", "Groceries" |
| `amount` | `numeric(10,2)` | No | |
| `billing_interval` | `integer` | No | Months between occurrences. `1` = monthly, `2` = bimonthly, `3` = quarterly, `6` = semiannual, `12` = annual |
| `anchor_month` | `integer` | No | Month (1–12) the billing cycle starts from. Used with `billing_interval` to determine applicable months. |
| `active` | `boolean` | No | Soft delete. Default `true`. |
| `created_at` | `timestamptz` | No | |
| `updated_at` | `timestamptz` | No | |

---

### `income_sources`

Global pool of recurring income sources (e.g. Primary Salary, Partner Salary). Not scenario-scoped. Soft-deleted via the `active` flag.

The engine sums monthly-equivalent amounts across all active sources to derive `monthlyNet`. Monthly equivalent = `amount * occurrences_per_year / 12`.

`pay_day_1`, `pay_day_2`, and `pay_anchor_date` are nullable in MVP. Populated and used in P1 for paycheck view scheduling.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `name` | `text` | No | e.g. "Primary Salary", "Partner Salary" |
| `amount` | `numeric(10,2)` | No | Net amount per occurrence |
| `frequency` | `text` | No | `MONTHLY`, `SEMIMONTHLY`, or `BIWEEKLY` |
| `pay_day_1` | `integer` | Yes | Day of month for first payment. Used by `MONTHLY` and `SEMIMONTHLY`. MVP: unused. |
| `pay_day_2` | `integer` | Yes | Day of month for second payment. `SEMIMONTHLY` only. MVP: unused. |
| `pay_anchor_date` | `date` | Yes | A known pay date from which biweekly cadence is derived. `BIWEEKLY` only. MVP: unused. |
| `active` | `boolean` | No | Soft delete. Default `true`. |
| `created_at` | `timestamptz` | No | |
| `updated_at` | `timestamptz` | No | |

---

### `income_ledger`

Historical record of income actually received. Not used for forward projection — the engine always derives future income from `income_sources`. `income_source_id` is nullable to allow recording ad-hoc income not tied to a recurring source.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `income_source_id` | `uuid` | Yes | FK → `income_sources`. Null for ad-hoc entries. `SET NULL` on source delete. |
| `received_on` | `date` | No | Specific date income was received |
| `amount` | `numeric(10,2)` | No | Actual amount received |
| `note` | `text` | Yes | Optional context |
| `created_at` | `timestamptz` | No | |

---

### `projected_income`

One-off expected future income events (e.g. freelance payment, year-end bonus). Folded into the engine as additional income in the months they land. `income_source_id` is nullable for standalone projections not tied to a recurring source. No `active` flag — expired entries are filtered at query time (`WHERE expected_on >= today`).

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `income_source_id` | `uuid` | Yes | FK → `income_sources`. Null for standalone entries. `SET NULL` on source delete. |
| `name` | `text` | Yes | Required when `income_source_id` is null |
| `expected_on` | `date` | No | Date the income is expected |
| `amount` | `numeric(10,2)` | No | |
| `created_at` | `timestamptz` | No | |

---

### `scenario_projections`

Persisted projection rows produced by the engine for each scenario. One row per `(scenario_id, card_id, month)`. Future month rows are overwritten each time inputs change and the engine reruns. Past month rows are never overwritten — the ledger is the source of truth for past months.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `scenario_id` | `uuid` | No | FK → `scenarios` |
| `card_id` | `uuid` | No | FK → `cards` |
| `month` | `date` | No | First day of the month (YYYY-MM-01) |
| `projected_payment` | `numeric(10,2)` | No | |
| `projected_interest` | `numeric(10,2)` | No | |
| `projected_ending_balance` | `numeric(12,2)` | No | |
| `projected_utilization_pct` | `numeric(6,4)` | No | `projected_ending_balance / credit_limit` |

**Unique constraint:** `(scenario_id, card_id, month)`

**Monthly totals** (total debt, total interest, total payments, total utilization) are aggregated at query time from card-level rows. No separate summary table.

---

### `monthly_card_ledger`

Ground truth record of what actually happened each month per card. Scenario-independent. Rows are written when the user confirms activity for a card. Records become immutable when the month closes.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `card_id` | `uuid` | No | FK → `cards` |
| `month` | `date` | No | First day of the month (YYYY-MM-01) |
| `actual_payment` | `numeric(10,2)` | Yes | Null until confirmed |
| `actual_interest` | `numeric(10,2)` | Yes | Null until confirmed |
| `actual_ending_balance` | `numeric(12,2)` | Yes | Null until confirmed. Used as engine starting point for confirmed cards when recomputing. |
| `confirmed_at` | `timestamptz` | Yes | Null until confirmed |
| `is_immutable` | `boolean` | No | Set to true when month closes. Immutable records cannot be edited. |

**Unique constraint:** `(card_id, month)`

---

## Read Logic

### On load (no recompute)
- **Past months:** read from `monthly_card_ledger`. Actuals if confirmed, empty if not. Scenario is irrelevant.
- **Future months:** read from `scenario_projections` for the active scenario.

### Current month (hybrid)
The current month is the only month where both tables are in play simultaneously:
- Per card: check `monthly_card_ledger` first. If confirmed, use actuals.
- Fall back to `scenario_projections` for cards not yet confirmed.

### On recompute (inputs changed)
The engine only reruns when a user saves changed inputs. Starting balance resolution per card:
- If the card has a confirmed `actual_ending_balance` in `monthly_card_ledger` for the current month → use that.
- Otherwise → use `cards.balance`.

Recompute overwrites `scenario_projections` rows from the current month forward for the affected scenario. Past rows in `monthly_card_ledger` are never touched.

---

## Computed Models (Not Persisted)

Produced by the `ProjectionEngine` at runtime. Results are written to `scenario_projections` on save. These types define the engine's output contract.

### `MonthlyCardProjectionRow`

One row per card per month in the projection.

| Field | Type | Notes |
|---|---|---|
| `month` | `String` | `YYYY-MM` |
| `cardId` | `UUID` | |
| `startingBalance` | `BigDecimal` | |
| `basePaymentApplied` | `BigDecimal` | |
| `extraPaymentApplied` | `BigDecimal` | |
| `interestApplied` | `BigDecimal` | Interest applied after payments |
| `endingBalance` | `BigDecimal` | |
| `utilizationPct` | `BigDecimal` | `endingBalance / creditLimit` |

### `MonthlyLoanDetail`

One row per loan per month in the projection.

| Field | Type | Notes |
|---|---|---|
| `month` | `String` | `YYYY-MM` |
| `loanId` | `UUID` | |
| `loanName` | `String` | |
| `startingBalance` | `BigDecimal` | Balance before this month's payment |
| `payment` | `BigDecimal` | Amount actually applied (capped at balance; includes any extra payments) |
| `endingBalance` | `BigDecimal` | |
| `paidOff` | `Boolean` | True when ending balance is zero |

No `utilizationPct` — loans have no credit limit.

---

### `MonthlySummaryRow`

One row per month, aggregated across all cards.

| Field | Type | Notes |
|---|---|---|
| `month` | `String` | `YYYY-MM` |
| `totalStartingDebt` | `BigDecimal` | |
| `totalEndingDebt` | `BigDecimal` | |
| `totalInterest` | `BigDecimal` | |
| `totalBasePayments` | `BigDecimal` | |
| `totalExtraPayments` | `BigDecimal` | |
| `totalUtilizationPct` | `BigDecimal` | Aggregate utilization across all cards |
| `poolBalance` | `BigDecimal` | Running liquidity pool balance at end of month. See `Implementation_Appendix.md` §A2. |

**Note:** Utilization threshold crossings (90%, 75%, 50%, 25%) are derived client-side from projection and ledger rows via MUI X. No separate threshold table.
