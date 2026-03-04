---
title: Runway -- Implementation Appendix
---

# Implementation Appendix

Design decisions made after the original MVP documents were written. These supersede conflicting sections in `Debt_Payoff_Simulator_PRD_MVP.md`, `Debt_Payoff_Simulator_MVP_Implementation_Plan.md`, and `DATA_DICTIONARY.md`.

---

## A1. `plan_settings` Deprecated — Replaced by `allocation_plans` + `plan_allocations`

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

**`plan_allocations`** — child rows belonging to one `allocation_plan`.

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

## A5. Backlog Items (from this design session)

- **Mid-timeline plan insertion:** inserting an `allocation_plan` between two existing bounded plans requires rebalancing end dates of the surrounding plans. Deferred.
- **Plan validation UI:** surface a warning when `sum(allocations) > extraPool` so users know the month can't be finalized.
