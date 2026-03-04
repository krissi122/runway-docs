---
title: Debt Payoff Simulator -- PRD (MVP)
---

# 1. Goal

Replace the existing Google Sheet with a deterministic monthly
projection engine that:\
\
- Calculates balances and interest over time\
- Applies user-defined base payments (issuer minimum OR fixed \"real
minimum\")\
- Allocates extra dollars to focus/secondary targets\
- Tracks utilization thresholds (\<90%, \<75%, \<50%, \<25%) per card
and overall\
- Generates a derived paycheck checklist view from monthly plan\
\
This is a strategic debt simulation tool, not a full personal finance
platform.

# 2. Non-Goals (Out of Scope for MVP)

\- RSU/ESPP vest schedules or tax modeling\
- Raise/promotion scenario comparisons\
- Scenario cloning or branching\
- Bank integrations\
- Risk scoring or emergency modeling\
- Strategy presets (snowball/avalanche libraries)\
- Daily interest or statement date modeling

# 3. Core Data Model

Card\
- id\
- name\
- balance (current)\
- apr (decimal format, e.g., 0.2799)\
- creditLimit\
- basePaymentMode (MINIMUM \| FIXED)\
- basePaymentAmount (manual entry; required for both modes)\
\
~~PlanSettings~~ — **DEPRECATED.** Replaced by `AllocationPlan` + `Allocation`. See `Implementation_Appendix.md` §A1.\

AllocationPlan (replaces PlanSettings)\
- scenarioId\
- effectiveMonthStart\
- effectiveMonthEnd (null = active)\
- allocations: List\<Allocation\>\

Allocation\
- target (CARD | POOL)\
- cardId (if CARD)\
- amount\
\
LumpSumEvent\
- date\
- amount\
- target (FOCUS \| SPECIFIC_CARD)\
- targetCardId (if specific)\
\
MonthlyCardProjectionRow (computed)\
- month (YYYY-MM)\
- cardId\
- startingBalance\
- basePaymentApplied\
- extraPaymentApplied\
- interestApplied\
- endingBalance\
- utilizationPct\
\
MonthlySummaryRow (computed)\
- month\
- totalStartingDebt\
- totalEndingDebt\
- totalInterest\
- totalBasePayments\
- totalExtraPayments\
- totalUtilizationPct\
- thresholdCrossings (derived)

# 4. Calculation Rules

Monthly Net Income:\
monthlyNet = netPerPaycheck \* payFrequency / 12\
monthlyDebtBudget = max(0, monthlyNet - monthlyNonDebtExpenses)\
\
Base Payments:\
For each card:\
basePayment = min(basePaymentAmount, startingBalance)\
\
Sum base payments across all cards:\
baseTotal = sum(basePayment)\
\
Extra Pool:\
extraPool = max(0, monthlyDebtBudget - baseTotal)\
\
Allocation — **DEPRECATED model replaced.** See `Implementation_Appendix.md` §A1.\
Extra pool is distributed per the active AllocationPlan's target list:\
- CARD targets: apply min(amount, remainingBalance); overflow → pool\
- POOL targets: add amount to pool balance\
- Surplus (extraPool > sum of targets) → pool automatically\
\
Lump Sums:\
Applied in the month they occur before payment allocation.\
\
Interest:\
monthlyRate = apr / 12\
interestApplied = remainingBalanceAfterPayments \* monthlyRate\
\
Interest is applied after payments within the month.\
\
Payoff Behavior:\
When a card reaches zero, its base payment naturally frees future
allocation.

# 5. Utilization Threshold Tracking

Hard-coded thresholds:\
- 90%\
- 75%\
- 50%\
- 25%\
\
For each card and overall utilization:\
Detect first month utilization drops below each threshold.\
Store/display crossing month.

# 6. UI Screens (MVP)

Dashboard:\
- Total debt\
- Projected debt-free month\
- Total projected interest\
- Total utilization + milestone tracker\
- Burn-down chart\
- Utilization chart\
\
Cards:\
Editable table for card inputs and utilization display\
\
Plan:\
Income settings, focus/secondary selectors, split slider\
\
Timeline:\
Monthly projection table with key metrics and milestone events\
\
Paycheck View:\
Derived from monthly plan\
Split monthly payments into two checklists per month\
Rounding rule: paycheck 1 = floor(half), paycheck 2 = remainder

# 7. Acceptance Criteria

1\) Deterministic monthly projection through payoff.\
2) Base payments respect manual entry for MINIMUM and FIXED modes.\
3) Extra allocation respects focus/secondary and split percentage.\
4) Threshold crossings displayed for each card and total utilization.\
5) Paycheck view sums exactly to monthly totals.
