# Runway — Backlog

Tracked improvements and tech debt items. Add entries here as they come up during development.

---

## Frontend

### Projection Thresholds: Surface Month-1 Crossings Separately
Month-1 crossings are currently excluded entirely from the "Next Target" view because the engine records all thresholds a card is already below at projection start in month 1 (e.g. a card at 65% util logs 90% and 75% in month 1 before any real progress is made). However, a card can also legitimately cross a threshold *due to* that month's payment — and that crossing should not be lost. The correct behavior: for month-1, apply the same "next threshold only" filter as all other months (i.e. show only the single next threshold crossed that month, not all of them), rather than excluding the month entirely. The already-satisfied thresholds (those the card was below before any payment) should be surfaced separately — likely as an "Already Achieved" indicator — so users can see which milestones are already behind them vs. upcoming. Coordinate with the zero-balance exclusion item below.

### Projection Views: Exclude Zero-Balance Cards
Cards that have been fully paid off (ending balance = $0) appear in multiple projection views but are noise once cleared. Apply a consistent "hide paid-off" filter across: (1) the Summary tab's expanded card sub-rows, (2) the Thresholds tab's next-target list (a paid-off card has no next threshold to work toward), and (3) the Card Detail tab's card selector. Coordinate with the existing "Hide Paid-Off Cards Toggle" backlog item — these may be the same toggle applied globally rather than per-view.

### Projection Summary: Hide Paid-Off Cards Toggle
Add a toggle (checkbox or switch) above the Summary tab's expandable table to hide cards where `isPaidOff === true` from the expanded card sub-rows. When enabled, paid-off cards are filtered out entirely rather than shown at reduced opacity. Useful for focusing on active debt once cards start clearing.

### Projection Summary: Paycheck Grouping
In the Summary tab expandable rows, group card details by paycheck rather than showing a flat card list. Each month expands to two paycheck sub-rows (Paycheck 1, Paycheck 2); each paycheck sub-row then expands to show cards whose `paymentDueDay` falls before the next paycheck date. Requires: (1) storing a pay anchor date on income sources so the frontend can derive actual paycheck dates per month, and (2) deciding whether the split logic lives in the frontend or the projection engine adds paycheck-level rows to its output. The UI hierarchy needs different column configs at each level (month summary / paycheck summary / card detail), which MUI X DataGrid Pro detail panels or a custom MUI Table + Collapse solution can support.

### Income Table: Meaningful Anchor Date Display
The `payAnchorDate` column is currently hidden from the income table because a raw date is not useful at a glance. For biweekly income, derive and display the next upcoming payday (or the next two) rather than the raw anchor. For semimonthly and monthly, the pay day columns already cover this. Revisit once the projection engine is wired up and we know what cadence data is available in the response.

### Migrate Dialogs to Reusable Components
Each entity (Card, FixedExpense, IncomeSource) now has separate Add and Edit dialogs, which duplicates the field definitions. Once all entities are built out, evaluate shared form field components (e.g. `CardFormFields`) that both Add and Edit dialogs compose — keeping dialog shells separate but eliminating field duplication. Also consider a shared `FormDialog` wrapper for the common `Dialog > DialogTitle > DialogContent > DialogActions / Cancel / Save` shell.

### Extract Reusable Validation Patterns
Each dialog re-implements its own inline `useMemo` validation. Common checks (non-empty string, positive number, day-of-month 1–31, month 1–12) are duplicated across files. Extract a small set of validator utilities or a `useFormField` hook so validation logic is consistent and testable in isolation.

---

## Engine

### Per-Scenario Card and Income Scoping
Currently cards, income sources, and fixed expenses are global entities — all active records are loaded for every projection regardless of scenario. This means every scenario sees the same card set and income configuration simultaneously; there is no way to model different card sets or income assumptions per scenario. This is intentional for the MVP but will become a significant limitation once multi-scenario branching (e.g. "what if I pay off card X first?") is added. When multi-scenario support is designed, consider adding a `scenario_id` FK (or a junction table) to cards and income sources so each scenario can have its own snapshot.

---

### Ledger-Aware Starting Balances in ProjectionService
`ProjectionService` currently always resolves starting card balances from `cards.balance`. Per the data dictionary hybrid model, the current month should prefer `monthly_card_ledger.actual_ending_balance` for confirmed cards, falling back to `cards.balance` only when no confirmed ledger entry exists. This requires `monthly_card_ledger` to be implemented and queryable by `(card_id, month)`. The engine itself is unaffected — it always receives first-of-month dates and resolved balances; the hybrid resolution lives entirely in the service layer.

---

### Dynamic Minimum Payment Calculation
Currently `MINIMUM` payment mode in `EngineCardSchedule` is treated identically to `FIXED` — the stored `basePaymentAmount` is applied as a static amount every month. The first-class fix: compute the true minimum each month as `max(min_floor, balance × min_pct)`, where `min_floor` (default $25) and `min_pct` (default 1%) are stored on the card payment schedule. The frontend should derive and display the projected minimum for the current month's balance dynamically in the payment schedule UI, so users can see what they'd actually owe before entering a custom amount. Until this is implemented the engine applies the static stored amount — projections using MINIMUM mode will show optimistically earlier payoff than reality (payment doesn't shrink as the balance decreases).

---

## Backend

### End-of-Month Date Clamping for Fixed Expenses
When a fixed expense has a `dueDayOfMonth` of 29, 30, or 31 and the billing cycle falls in February, the projection engine must treat the due date as the last day of February rather than skipping or erroring. Apply the same clamping logic to any month shorter than the stored day (e.g. April 31 → April 30). The canonical rule: `min(dueDayOfMonth, lastDayOfMonth)`.

### Audit and Refactor Test Coverage
Audit tests across all layers and fill gaps with appropriate test types. Resource tests use `ResourceExtension` + MockK (mock the layer below). Orchestrator tests use MockK (mock accessors). DAO tests are integration tests against the real DB. Accessor tests are not needed — they are thin JDBI wrappers covered by DAO tests. Missing coverage to address: `LoanResource`, `ProjectedIncomeResource` (resource tests); `LoanDAO`, `AllocationDAO`, `ProjectedIncomeDAO` (DAO integration tests); `ProjectionOrchestrator` (orchestrator unit test).

### DB Integration Test Improvements
Expand integration test coverage for database interactions. Ensure tests run against a real Postgres instance (e.g. via Testcontainers) rather than mocks, cover edge cases in JDBI queries, and validate Liquibase migrations apply cleanly from a blank schema.

---

## Performance

### Projection Latency Optimization
The `/projection` endpoint is showing noticeable latency. Currently every projection request loads all scenario data from the DB and recomputes from scratch. Expected to improve significantly once projection results are persisted — the engine only needs to rerun when scenario data changes, and cached results can be served directly. When investigating further: profile DB query count per request (N+1 risk in allocation/card loading), and consider whether the engine itself has any hot loops worth optimizing for large projection windows. Address after persistence layer is in place.

---

## Infrastructure

### Auth0 and User Profiles
Integrate Auth0 for authentication. Add user identity to the data model so all entities (cards, income sources, fixed expenses, plan settings) are scoped per user. Frontend acquires a JWT via Auth0 SDK and passes it as a Bearer token; backend validates the token and resolves the user on each request. User profile record stores Auth0 subject ID and any app-level preferences.
