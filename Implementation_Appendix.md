---
title: Runway -- Implementation Appendix
---

# Implementation Appendix

Design decisions made after the original MVP documents were written. These supersede conflicting sections in `Debt_Payoff_Simulator_PRD_MVP.md`, `Debt_Payoff_Simulator_MVP_Implementation_Plan.md`, and `DATA_DICTIONARY.md`.

---

## A1. `plan_settings` Deprecated — Replaced by `allocation_plans` + `allocations`

### Reason

The original `plan_settings` model (focus card, optional secondary card, split %) was too rigid. It couldn't express targeting multiple cards simultaneously, routing surplus to a liquidity pool, or changing allocation strategy mid-plan.

### Replacement Model

**`allocation_plans`** — one record per date range per scenario.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `scenario_id` | `uuid` | No | FK → `scenarios` ON DELETE CASCADE |
| `effective_month_start` | `date` | No | First month this plan applies (inclusive, stored as YYYY-MM-01) |
| `effective_month_end` | `date` | Yes | Last month this plan applies (inclusive). `NULL` = currently active. |

**Constraint:** `effective_month_end IS NULL OR effective_month_end > effective_month_start`

**Unique index:** only one row per `scenario_id` may have `effective_month_end = NULL` at a time (enforced via partial unique index).

---

**`allocations`** — child rows belonging to one `allocation_plan`.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `plan_id` | `uuid` | No | FK → `allocation_plans` ON DELETE CASCADE |
| `target` | `text` | No | `CARD` or `POOL` |
| `card_id` | `uuid` | Yes | FK → `cards`. Required when `target = CARD`, null when `target = POOL`. |
| `amount` | `numeric(10,2)` | No | Monthly dollar amount to allocate to this target |

**Constraints:**
- `target IN ('CARD', 'POOL')`
- `(target = 'CARD' AND card_id IS NOT NULL) OR (target = 'POOL' AND card_id IS NULL)`

---

### Engine Logic (replaces focus/secondary/split calculation)

Per month, the engine resolves the active plan as the row where `effective_month_start <= month AND (effective_month_end IS NULL OR effective_month_end >= month)`.

```
extraPool = monthlyNet - nonDebtExpenses - sum(basePayments)

for each allocation in active plan (in order):
    if target = CARD:
        applied = min(allocation.amount, card.remainingBalance)
        card.remainingBalance -= applied
        overflow = allocation.amount - applied  →  poolBalance += overflow
    if target = POOL:
        poolBalance += allocation.amount

if extraPool > sum(allocations):
    remainder → poolBalance
```

**If `sum(allocations) > extraPool`:** projection is still valid and can be displayed. The ledger entry for that month cannot be finalized until the user resolves the shortfall (by updating the plan or increasing income).

---

### Auto-Close Behavior (on create)

When a new `allocation_plan` is saved for a scenario:

1. Find the existing plan where `effective_month_end IS NULL`.
2. If found and its `effective_month_start < new plan's start`: set its `effective_month_end` to the month immediately before the new plan's `effective_month_start`.
3. Insert the new plan with `effective_month_end = NULL`.

**Example:**
- March plan saved → `{ start: 2026-03-01, end: null }`
- June plan saved → March plan updated to `{ end: 2026-05-01 }`, June plan inserted as `{ start: 2026-06-01, end: null }`

**Mid-timeline insertion** (inserting a plan between two existing bounded plans) is deferred to backlog. Only the "append a future plan" case is supported in MVP.

---

### What Was Removed

- `plan_settings` table (dropped via migration)
- `PlanSettings`, `PlanSettingsRow`, `UpsertPlanSettingsRequest` Kotlin models
- `PlanSettingsDAO`, `PlanSettingsAccessor`, `PlanSettingsResource`
- Frontend: `planSettingsTypes.ts`, `planSettingsApi.ts`, plan settings form section

---

## A2. Liquidity Pool — `initial_pool_balance` on `scenarios`

### Reason

The engine tracks a running liquidity pool balance (cash buffer that accumulates from surplus each month). This pool needs a starting seed value representing cash already on hand at the time the scenario begins.

### Change

Add `initial_pool_balance` column to `scenarios`:

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `initial_pool_balance` | `numeric(10,2)` | No | Starting cash balance for the liquidity pool. Default `0.00`. |

The engine initializes `poolBalance = initial_pool_balance` at the start of the projection. Each month, overflow from card allocations and unallocated extra pool surplus are added to `poolBalance`. The pool is never automatically deployed for debt payment — it only grows unless the user explicitly creates a `POOL` allocation target.

The running pool balance appears in projection output as a field on `MonthlySummaryRow`.

---

## A3. `MonthlySummaryRow` — Updated Fields

Add `poolBalance` to the computed projection output:

| Field | Type | Notes |
|---|---|---|
| `poolBalance` | `BigDecimal` | Running liquidity pool balance at end of month |

---

## A4. UI — "Allocations" Page

The former "Plan" nav page is replaced by two concerns:

- **Plan page** (`/plan`): Scenario name + `initial_pool_balance` only. Two fields.
- **Allocations page** (`/allocations`): DataGrid of `allocation_plans`. Each row shows its date range and a summary of its allocations. Add/Edit dialog contains effective month start and a sub-list of allocation targets (Card or Pool, with amount).

Nav links updated: **Cards · Fixed Expenses · Income · Plan · Allocations · Schedules · Lump Sums**

---

## A5. `allocations` — Updated Targets (BASE_POOL)

`target` now supports three values: `CARD`, `POOL`, `BASE_POOL`.

**`BASE_POOL`** — the minimum balance the liquidity pool must hold. The engine tops up the pool toward this target balance before routing any money to `POOL` or `CARD` allocations. Once the pool meets or exceeds the target, `BASE_POOL` allocations are skipped. Only one `BASE_POOL` allocation per plan is meaningful (the engine processes the first one found).

**Engine dispatch order (per month, from extra pool):**
1. `BASE_POOL` — top up pool to target balance (only if pool < target)
2. `POOL` — route fixed amount into pool
3. `CARD` — sorted by `allocation.amount` ascending, applied greedily
4. Unrouted remainder → pool

`FOCUS` lump sums follow the same dispatch order as the active plan.

**Constraints:**
- `target IN ('CARD', 'POOL', 'BASE_POOL', 'LOAN')`
- `card_id IS NOT NULL` when `target = CARD`, null otherwise
- `loan_id IS NOT NULL` when `target = LOAN`, null otherwise

---

## A6. Loans

### Reason

Installment debt (auto loans, student loans, personal loans, mortgages) needs to participate in the projection engine but differs from revolving credit cards:

- No credit limit → no utilization or threshold tracking
- Interest is already amortized into the fixed monthly payment → engine does not calculate interest
- Loans have a contractual end date → engine stops expecting payments after that month
- APR is stored as informational only (nullable)

### Data Model

New global `loans` table (see `DATA_DICTIONARY.md`). Not scenario-scoped — same as `cards`.

### Engine Treatment

Loans are processed as a separate debt category alongside cards:

1. **Base payment step:** for each active loan (`month <= endDate AND balance > 0`), deduct `monthly_payment` (capped at balance) from the loan balance. This payment is a committed cost — it reduces the same disposable income pool that card base payments draw from.
2. **No interest calculation** — not needed; the amortized payment already accounts for it.
3. **Extra payments:** loans can be targeted by allocation plans (`target = LOAN`) and lump sums (`target = SPECIFIC_LOAN`). Extra payments reduce balance further, potentially paying off the loan before `endDate`.
4. **Payoff cascade:** when a loan reaches zero balance or its `endDate` passes, the freed `monthly_payment` flows into the extra pool and becomes available for card payoff in subsequent months.
5. **Stop condition:** the projection ends when all cards AND all loans are paid off (balance = 0).

### Allocation and Lump Sum Extensions

- `allocations.target` gains `LOAN`; new nullable `loan_id` FK → `loans`
- `lump_sums.target` gains `SPECIFIC_LOAN`; new nullable `target_loan_id` FK → `loans`

---

## A7. Backlog Items (from this design session)

- **Mid-timeline plan insertion:** inserting an `allocation_plan` between two existing bounded plans requires rebalancing end dates of the surrounding plans. Deferred.
- **Plan validation UI:** surface a warning when `sum(allocations) > extraPool` so users know the month can't be finalized.
- **Allocation percentages:** `allocations.amount` is currently a fixed dollar amount. A future version should support percentage-of-disposable allocations.
- **Loan APR in projection:** if APR is present, the engine could optionally compute and display interest as a projection-quality metric (informational, not affecting balance calculation).
