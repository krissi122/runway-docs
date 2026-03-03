---
title: Runway -- P1+ Feature Backlog
---

# P1+ Feature Backlog

Working document tracking features intentionally deferred from MVP. Ordered loosely by priority. Update as design decisions are made.

---

## P1

### Multi-schedule payment support
**Context:** `card_payment_schedules` is designed with `effective_month_start` and `effective_month_end` columns, but MVP inserts one row per `(scenario_id, card_id)` with a sentinel start date and null end date.

**P1 work:**
- Allow multiple schedule rows per `(scenario_id, card_id)` with non-overlapping date ranges
- UI to add a future payment mode change to a card within a scenario (e.g. "switch Card A to FIXED $500 starting June 2026")
- Engine already resolves payment policy by date range — no engine changes needed, only the constraint relaxation and UI

---

### Scenario cloning
**Context:** Scenarios are fully persisted. Cloning duplicates `plan_settings`, `lump_sums`, `card_payment_schedules`, and `scenario_projections` rows under a new `scenario_id`. Cards are global and not copied.

**P1 work:**
- Backend clone endpoint
- UI to clone a scenario and give it a name
- Cloned scenario starts as `DRAFT`

---

### Scenario comparison view
**Context:** Multiple scenarios can be persisted simultaneously. Each has its own `scenario_projections` rows.

**P1 work:**
- UI to select two scenarios and compare their projection timelines side by side
- Key comparison points: payoff month, total interest paid, debt burn-down curve
- MUI X Charts overlay or dual DataGrid view

---

### Per-scenario card inclusion/exclusion
**Context:** Cards are currently a global pool — all cards participate in every scenario.

**P1/P2 work:**
- Add a `scenario_cards` join table to explicitly include/exclude cards per scenario
- Engine receives only the included cards for a given scenario
- UI to toggle card participation per scenario

---

### Fixed expense scheduling (paycheck view)
**Context:** `fixed_expenses` uses `billing_interval` + `anchor_month` to determine which months an expense hits. The engine already uses this correctly per month. What's deferred is surfacing the specific billing dates in the paycheck view so the user can see which week of the month a bill lands.

**P1 work:**
- UI to show fixed expense hits on the paycheck view calendar
- No schema or engine changes needed

---

### Income source paycheck scheduling
**Context:** `income_sources` has `pay_day_1`, `pay_day_2`, and `pay_anchor_date` columns today but they are nullable and unused in MVP.

**P1 work:**
- UI to populate pay day fields when creating/editing an income source
- Paycheck view uses these to assign income to the correct paycheck within the month
- Engine input unchanged — it still receives a monthly net total per source

---

### Loan management
**Context:** MVP models revolving credit lines only. Installment loans (personal loans, auto loans, student loans) have fundamentally different payoff behavior — fixed monthly payments, no credit limit, no utilization metric, and amortizing principal reduction.

**P1 work:**
- Decide whether loans extend the `cards` table (with a `debt_type` discriminator) or live in a separate `loans` table
- Engine needs a new code path for installment debt: fixed payment applied to principal + interest, no utilization tracking
- UI to add/edit loans separately from revolving cards
- Payoff projection for loans folds into the same timeline and extra pool allocation logic

---

## P2+

### RSU / ESPP vest schedule modeling
Model equity compensation events as future income lump sums tied to vest dates. Apply toward debt per user-defined rules.

---

### Raise / income change scenarios
Allow `plan_settings.net_per_paycheck` to change at a future month within a scenario. Requires time-varying income model similar to `card_payment_schedules`.

---

### APR change modeling
Allow a card's APR to change at a future month within a scenario. Low priority — user confirmed APR changes are out of scope for MVP and P1.

---

### Authentication
Multi-user support. All entities would gain a `user_id` FK. Out of scope until the tool is shared beyond a single user.

---

### Daily interest modeling
Current model applies interest monthly after payments. Daily interest (based on average daily balance) is more accurate but significantly more complex. Deferred indefinitely.
