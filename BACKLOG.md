# Runway — Backlog

Tracked improvements and tech debt items. Add entries here as they come up during development.

---

## Frontend

### Visual Styling Pass
The app currently uses MUI defaults with minimal theming. A dedicated styling pass should establish a cohesive visual identity. Areas to address:

- **Color scheme** — define a custom MUI theme (`createTheme`) with a primary/secondary palette rather than the default blue. Consider a darker, more "power user" feel consistent with the tool's positioning.
- **Expanded month row sections** — section headers (Income, Cards, Loans, Expenses) currently use small caption text + icon. Evaluate weight, spacing, and contrast. Status chips (Confirmed, Locked, Needs Confirmation, Paid, Received, Pending) should have consistent sizing and color mapping across all section types.
- **Table density** — the summary table and expanded sub-tables use `size='small'`. Review padding, row height, and font sizes for readability at scale (30+ months of data).
- **Positive/negative values** — pool balance already colors negative values red. Extend consistent coloring to interest amounts, payment overages, etc.
- **Mobile / narrow viewport** — the current layout assumes a wide screen. Decide whether to address responsiveness now or defer.
- **NavBar** — evaluate the current tab layout; consider a sidebar or a more compact top nav as the page count grows.

Coordinate with the ProjectionPage refactor — some styling decisions are easier to make once the component tree is cleaner.

---

### Projection Summary: Collapsible Sections Within Expanded Month Rows ✓ Done
In the Summary tab, when a month row is expanded, Income, Cards, and Loans are currently shown as a flat block. Each section should have its own independent expand/collapse toggle (arrow icon + section label as a clickable header) so users can drill into just the section they care about. State is tracked per-month per-section (`Map<month, Set<'income'|'cards'|'loans'>>`). All sections default to collapsed when a month is first opened. Applies to both the past/current ledger confirm view and the future projection view.

---

### Variable Fixed Expenses + Fixed Expense Ledger ✓ Done
Full-stack feature to support fixed expenses with variable actual amounts and to track expense payment confirmation in the ledger.

**Backend — schema and model:**
- Add `is_variable BOOLEAN NOT NULL DEFAULT false` to `fixed_expenses` (migration). Update `FixedExpense`, `FixedExpenseRow`, `CreateFixedExpenseRequest`, `UpdateFixedExpenseRequest` — add `isVariable` field throughout.
- New `monthly_fixed_expense_ledger` table (migration): `id` (uuid PK), `fixed_expense_id` (uuid FK), `month` (date), `actual_amount` (NUMERIC(12,2) nullable — only populated for variable expenses), `payment_made` (boolean), `confirmed_at` (timestamptz), `is_immutable` (boolean). Unique constraint on `(fixed_expense_id, month)`. Same immutability rules as card/loan ledger — auto-lock 2 months prior on confirm.
- `MonthlyFixedExpenseLedger` domain + Row models, `MonthlyFixedExpenseLedgerDAO`, `MonthlyFixedExpenseLedgerAccessor` following the existing card/loan ledger pattern.

**Backend — endpoints:**
Extend `LedgerOrchestrator` and `LedgerResource` with:
- `POST /ledger/expenses/confirm` — upsert fixed expense ledger rows for a month (accepts a list of `{fixedExpenseId, actualAmount?, paymentMade}`)
- `POST /ledger/expenses/{id}/payment` — mark a single expense ledger entry as paid/unpaid (always allowed regardless of immutability)
- `DELETE /ledger/expenses/{id}` — delete an expense ledger entry; 404 if missing, 409 if immutable
- Extend `GET /ledger?month=` response (`LedgerMonthResult`) to include `expenses: List<MonthlyFixedExpenseLedger>`

**Backend — engine integration:**
- Add `MonthlyExpenseDetail` to engine output: `fixedExpenseId`, `expenseName`, `month`, `projectedAmount`, `isVariable`. Add `monthlyExpenseDetails: List<MonthlyExpenseDetail>` to `ProjectionResult`. Emit one entry per expense per month the expense applies to.
- Add `EngineConfirmedExpense(fixedExpenseId, month, actualAmount)` to `ProjectionTypes`. Add `confirmedExpenses: List<EngineConfirmedExpense>` to `ProjectionInput`.
- In `computeMonthlyExpenses()`, if a confirmed entry exists for a given expense + month, substitute `actualAmount` for the expense's stored `amount`. Same substitution logic as confirmed income.
- `ProjectionOrchestrator` builds `confirmedExpenses` from `MonthlyFixedExpenseLedgerAccessor` entries up to the current month, same as `confirmedIncome`.

**Frontend:**
- Add `isVariable` to FE fixed expense types. Add a "Variable amount" checkbox to `AddFixedExpenseDialog` and `EditFixedExpenseDialog`.
- Add `MonthlyFixedExpenseLedger`, `ConfirmExpenseEntry` to `ledgerTypes.ts`. Add `LedgerMonthResult.expenses`. Add API functions to `ledgerApi.ts`: `confirmExpenses`, `setExpensePaymentMade`, `deleteExpenseLedger`.
- Add `MonthlyExpenseDetail` to `projectionTypes.ts`. Add `monthlyExpenseDetails` to `ProjectionResult`.
- In the Summary tab's expanded month rows, add an **Expenses** section below Loans. For past/current months: list each applicable expense with its projected amount; variable expenses show an amount input (pre-filled with projected, editable until confirmed); fixed expenses show the projected amount as read-only. Both show a payment status chip and a "Mark Paid" / "Confirm" button. For future months: show a read-only list of applicable expenses and their projected amounts.

---

### Miscellaneous Spend — Entry and Projection Visibility
The engine already supports a `MISC` lump sum target: it subtracts the amount from that month's disposable and tracks it as `totalMiscSpend` in `MonthlySummary`. However, the current UX has two gaps:

1. **Entry method** — MISC spend can only be entered as a one-off lump sum via the Lump Sums page. There is no way to model recurring miscellaneous spend (e.g. "I typically spend $200/month on irregular purchases"). Evaluate whether this belongs as a field on the scenario/plan settings (a flat monthly buffer deducted before extra allocation), or as a recurring lump sum pattern, or something else.
2. **Projection visibility** — `totalMiscSpend` is computed and returned in `MonthlySummary` but never rendered in `ProjectionPage`. It should appear somewhere in the monthly summary row or expanded section so users can see what the engine is accounting for.

Design question to resolve before implementation: should recurring misc spend live on the scenario (a standing monthly buffer) or be expressed as a recurring lump sum? The former is simpler; the latter is more flexible.

---

### Projection Thresholds: Surface Month-1 Crossings Separately
Month-1 crossings are currently excluded entirely from the "Next Target" view because the engine records all thresholds a card is already below at projection start in month 1 (e.g. a card at 65% util logs 90% and 75% in month 1 before any real progress is made). However, a card can also legitimately cross a threshold *due to* that month's payment — and that crossing should not be lost. The correct behavior: for month-1, apply the same "next threshold only" filter as all other months (i.e. show only the single next threshold crossed that month, not all of them), rather than excluding the month entirely. The already-satisfied thresholds (those the card was below before any payment) should be surfaced separately — likely as an "Already Achieved" indicator — so users can see which milestones are already behind them vs. upcoming. Coordinate with the zero-balance exclusion item below.

### Projection Views: Exclude Zero-Balance Cards
Cards that have been fully paid off (ending balance = $0) appear in multiple projection views but are noise once cleared. Apply a consistent "hide paid-off" filter across: (1) the Summary tab's expanded card sub-rows, (2) the Thresholds tab's next-target list (a paid-off card has no next threshold to work toward), and (3) the Card Detail tab's card selector. Coordinate with the existing "Hide Paid-Off Cards Toggle" backlog item — these may be the same toggle applied globally rather than per-view.

### Projection: "Paid Off" Column Lifecycle (Targeting → Yes)

Currently the "Paid Off" column in the Summary sub-rows, Card Detail, and Loan Detail tabs always shows `—`. The engine's `paidOff: true` flag (when projected ending balance hits zero) should eventually drive a "Targeting" state, and confirmed ledger data should drive the final "Yes" state. Full lifecycle:

| State | Condition | Display |
|---|---|---|
| Not targeting | Projected ending balance > $0 | `—` |
| Targeting | `paidOff === true` from projection (projected balance hits $0) | `Targeting` (amber) |
| Confirmed paid | Ledger row exists for that month with `paymentMade: true` AND `prePaymentBalance > 0` AND `paymentAmount >= prePaymentBalance` (or balance confirmed at $0) | `Yes` (green) |

Requires the FE to cross-reference ledger data with projection rows per card per month. Defer until ledger confirmation UI is live and ledger data is reliably available in `ProjectionPage`.

---

### Projection Summary: Batch Ledger Confirmation Per Month ✓ Done
Currently each "Mark Paid" button in an expanded month row fired an immediate API call, and each income "Confirm" button also fired immediately. This triggers a projection snapshot invalidation per action — so confirming 3 cards and 2 income paychecks in one month dirtied the snapshot 5 times and would cause 5 recomputes if the user fetched the projection between clicks.

**Change:** Convert all per-item confirmation actions within an expanded month to local state only. "Mark Paid" on cards and loans becomes a toggle — it updates UI state (pending) but sends nothing to the server. Income and expense "Confirm" buttons do the same. The existing "Confirm All" button at the top of the expanded section becomes the single outbound action: it collects all pending changes (balance entries, payment-made marks, income actuals, expense actuals) and fires all API calls before re-fetching the projection once.

Concretely:
- Add `pendingPaid: { cards: Set<cardId>, loans: Set<loanId> }` state per month
- "Mark Paid" toggles into `pendingPaid` rather than calling `setCardPaymentMade` / `setLoanPaymentMade`
- "Confirm All" sends: `confirmMonth` (balance + payment amount entries), then `setCardPaymentMade` / `setLoanPaymentMade` for all `pendingPaid` entries, then income/expense confirmations — all sequentially or in parallel before the single projection re-fetch
- Visual state: pending-paid rows show a distinct indicator (e.g. dashed border or "Pending" label) to make clear they haven't been sent yet

---

### Projection Summary: Hide Paid-Off Cards Toggle
Add a toggle (checkbox or switch) above the Summary tab's expandable table to hide cards where `isPaidOff === true` from the expanded card sub-rows. When enabled, paid-off cards are filtered out entirely rather than shown at reduced opacity. Useful for focusing on active debt once cards start clearing.

### Paycheck View (Alternate Projection Mode)
Add a second top-level projection view mode alongside the current month view, selectable as a persistent user setting. The **paycheck view** expands per pay period instead of per month — each row is a paycheck date, and within it the user sees income received, bills assigned to that pay period, and the running balance after payments.

**Core concept:** bills (cards, loans, fixed expenses) are assigned to a specific pay period within the month. A biweekly user with paychecks on the 1st and 15th would see two rows per month; each card's payment due date determines which paycheck it falls under. The user should also be able to manually override which pay period a bill is assigned to (e.g. "pay this card from paycheck 2 even though it's due on the 5th").

**Open design questions (to resolve before implementation):**
- **Where does pay-period assignment live?** Options: (a) a new `pay_period` field on `card_payment_schedules` and `fixed_expenses`, (b) a separate assignment table per scenario, (c) engine-derived from `paymentDueDay` vs. paycheck dates with no manual override. Option (b) is most flexible and keeps entity data clean; option (c) is simplest but loses manual control.
- **Does this belong in the engine or the frontend?** The engine already emits `MonthlyIncomeDetail` with paycheck dates. The paycheck view could be purely a frontend reorganization of existing engine output (group cards/expenses by which paycheck date precedes their due date) — no engine changes needed for the basic case. Manual overrides would require engine or orchestrator awareness. Lean toward frontend-only first, add override support as a follow-on.
- **Scenario setting vs. global setting?** The view mode (month vs. paycheck) is likely a UI preference, not scenario-specific — store in `localStorage` rather than the DB for now. Pay-period assignments for bills, if added, are scenario-specific and belong in the scenario data model.
- **Ledger confirm flow in paycheck view:** confirming income and payments in the paycheck view should mirror the month view — same batch-confirm approach, scoped to a pay period instead of a month.

**Prerequisites:** income source `payDay1`, `payDay2`, and `payAnchorDate` fields must be populated (currently nullable and unused in the engine). These drive the paycheck date computation. The `ProjectionPage` refactor (component extraction) should land first to make adding a second view mode manageable.

### Income Table: Meaningful Anchor Date Display
The `payAnchorDate` column is currently hidden from the income table because a raw date is not useful at a glance. For biweekly income, derive and display the next upcoming payday (or the next two) rather than the raw anchor. For semimonthly and monthly, the pay day columns already cover this. Revisit once the projection engine is wired up and we know what cadence data is available in the response.

### Refactor ProjectionPage into Components
`ProjectionPage.tsx` is ~1,050 lines and owns too much: the summary table, all expanded month row content, the ledger confirm UI, future-month projection views, card/loan/income/threshold tabs, and all associated state. The file is hard to navigate and changes to one section risk breaking others.

**Priority extraction — expanded month row sections:**
The highest-value split is the expanded row content within the Summary tab. Each section inside an expanded month should become its own component with clearly typed props fed from the page:
- `MonthIncomeSection` — income rows (both ledger-confirm and future-projection variants)
- `MonthCardsSection` — card rows with balance/payment inputs, status chips, confirm/mark-paid actions
- `MonthLoansSection` — loan rows, same pattern
- `MonthExpensesSection` — fixed expense rows (once variable expenses are implemented)

Each section component receives the relevant slice of data (cards, ledger entries, edits, callbacks) as props. No shared state should live inside section components — all state stays lifted at the page level or in a context, passed down as props or callbacks. This keeps the data flow explicit and sections independently readable.

**Secondary extractions (can follow):**
- Tab content components: `CardDetailTab`, `LoanDetailTab`, `UtilizationTab`, `ThresholdsTab`
- The expanded month wrapper itself: `ExpandedMonthRow` — owns the Collapse, section layout, and "Confirm All" button; receives sections as children or composes them directly

The page file after refactor should be primarily state, data derivation, and tab/section composition — no inline table JSX.

### Ledger View Design
The current ledger confirmation UI lives inline within the projection Summary tab — past/current months expose balance inputs, mark-paid toggles, and income confirmation directly in expanded month rows. As the ledger grows in importance (variable expenses, paycheck view, backdated entries), this inline approach may not be the right home for it. Design a dedicated ledger view before adding more ledger features. Open questions: should ledger confirmation be a separate page or tab? How should it relate to the projection summary? What is the right level of detail per entry?

---

### Migrate Dialogs to Reusable Components
Each entity (Card, FixedExpense, IncomeSource) now has separate Add and Edit dialogs, which duplicates the field definitions. Once all entities are built out, evaluate shared form field components (e.g. `CardFormFields`) that both Add and Edit dialogs compose — keeping dialog shells separate but eliminating field duplication. Also consider a shared `FormDialog` wrapper for the common `Dialog > DialogTitle > DialogContent > DialogActions / Cancel / Save` shell.

### Extract Reusable Validation Patterns
Each dialog re-implements its own inline `useMemo` validation. Common checks (non-empty string, positive number, day-of-month 1–31, month 1–12) are duplicated across files. Extract a small set of validator utilities or a `useFormField` hook so validation logic is consistent and testable in isolation.

---

## Engine

### Per-Scenario Card and Income Scoping
Currently cards, income sources, and fixed expenses are global entities — all active records are loaded for every projection regardless of scenario. This means every scenario sees the same card set and income configuration simultaneously; there is no way to model different card sets or income assumptions per scenario. This is intentional for the MVP but will become a significant limitation once multi-scenario branching (e.g. "what if I pay off card X first?") is added. When multi-scenario support is designed, consider adding a `scenario_id` FK (or a junction table) to cards and income sources so each scenario can have its own snapshot.

---

### Ledger-Aware Starting Balances in ProjectionService ✓ Done
`ProjectionOrchestrator` resolves starting balances from the most recent confirmed ledger entry per card and loan, falling back to the stored `balance` when no confirmed entry exists. The effective start month shifts back to the most recently confirmed month so the engine simulates any intervening months rather than treating a stale balance as current.

---

### Dynamic Minimum Payment Calculation
Currently `MINIMUM` payment mode in `EngineCardSchedule` is treated identically to `FIXED` — the stored `basePaymentAmount` is applied as a static amount every month. The first-class fix: compute the true minimum each month as `max(min_floor, balance × min_pct)`, where `min_floor` (default $25) and `min_pct` (default 1%) are stored on the card payment schedule. The frontend should derive and display the projected minimum for the current month's balance dynamically in the payment schedule UI, so users can see what they'd actually owe before entering a custom amount. Until this is implemented the engine applies the static stored amount — projections using MINIMUM mode will show optimistically earlier payoff than reality (payment doesn't shrink as the balance decreases).

---

### Allocation Plan Rethink
The current allocation plan system (`AllocationPlan` + `Allocation` records, managed via a dedicated page) was built as a general-purpose extra-funds routing mechanism. In practice it is complex to configure, hard to understand at a glance, and the UX has been removed from the nav bar pending a rethink. The allocation page and API remain in place but are not surfaced.

Open questions before reimplementing:
- **What is the actual user need?** Most users want to answer "where does my extra money go each month?" — probably a simpler priority-ordered list (focus card first, then secondary, then pool) rather than a full allocation table.
- **Does BASE_POOL / POOL / CARD / LOAN as a target model make sense to expose directly?** Or should this be abstracted into something higher-level (e.g. "pay down debt aggressively" vs. "build a buffer first")?
- **Relationship to the scenario settings page** — plan settings (`PlanSettingsPage`) already has focus card and split concepts. Allocations may just be a more powerful version of the same thing, or they may be redundant.

Design this before reimplementing. The backend model can be kept or replaced depending on the outcome.

---

## Backend

### Early Next-Month Unlock
Allow the next calendar month to become editable in the ledger confirmation UI before the calendar turns over. A user-configurable unlock day (e.g. 25) means that on March 25, April's rows appear in the expanded month view with the full confirm/save UI rather than the read-only future projection view.

**Design:**
- Store `nextMonthUnlockDay INT` on the scenario (or as a global user preference). Default: null (disabled — current behavior).
- Backend: `ProjectionOrchestrator` or a helper derives `effectiveCurrentMonth`: if `currentDay >= unlockDay`, treat next month as editable (i.e. `effectiveCurrentMonth = currentMonth + 1` for the purpose of determining which months show confirm UI). Return `effectiveCurrentMonth` in the projection response so the frontend doesn't need to re-derive it.
- Frontend: `isPastOrCurrent(month)` currently compares against a hardcoded `currentMonthStr`. Replace with a comparison against `effectiveCurrentMonth` from the projection result. Next month's row shows the ledger confirm UI (income inputs, card balance fields, mark paid toggles, Save Payments button) rather than the read-only future view.
- The unlock day setting should be editable on the Plan/Scenario settings page.

---

### Backdated Entry Confirmation
When a user adds a projected income entry (or other ledger-eligible entity) backdated to a date within the current month, the confirmation UI should surface it for confirmation. Scope: current month and prior months only — future months are never confirmed.

**Root cause to investigate:** `ProjectedIncomeAccessor.getUpcoming()` may filter by `expectedOn >= today`, which would exclude entries backdated to earlier in the current month. If so, the engine never receives them and they don't appear in `monthlyIncomeDetails` for past paychecks, making them unconfirmable. Fix: widen the query window to `expectedOn >= startOfCurrentMonth` (or the projection's `effectiveStartMonth`) so intra-month backdated entries are included.

**Frontend side:** the `isPastOrCurrent` check already marks the current month as editable. As long as the engine emits the backdated income entry in `monthlyIncomeDetails`, the confirm UI will pick it up automatically. No FE changes expected unless the month row isn't rendered at all (e.g. all cards paid off and no other entries exist for that month).

---

### Projection Snapshots (Persisted Engine Output) ✓ Done
Cache the full `ProjectionResult` per scenario so the engine does not recompute on every request. Serve the cached snapshot immediately on startup; recompute lazily on first request after any invalidating write.

**Schema** — drop `scenario_projections` (currently unused), create `scenario_snapshots`:
```sql
CREATE TABLE scenario_snapshots (
    scenario_id          UUID PRIMARY KEY REFERENCES scenarios(id) ON DELETE CASCADE,
    result               JSONB NOT NULL,
    snapshot_version     INT  NOT NULL,   -- increment when ProjectionResult shape changes; mismatch forces recompute
    computed_at          TIMESTAMPTZ NOT NULL,
    valid_for_month      DATE NOT NULL,   -- start month the engine ran for; stale if != currentMonth
    dirty                BOOLEAN NOT NULL DEFAULT true
);
```

**Invalidation rules:**
A snapshot must be recomputed when `dirty = true` OR `valid_for_month != currentMonth` OR `snapshot_version != CURRENT_VERSION`. All write paths set `dirty = true` via `ScenarioSnapshotAccessor`:
- Global entity writes (cards, loans, income sources, fixed expenses, ledger confirmations) → `markAllDirty()`. These entities are not scenario-scoped — every scenario uses them, so every snapshot is stale. When per-scenario card scoping (P1 backlog) lands, this can be narrowed to only dirty scenarios that include the changed entity.
- Scenario-scoped writes (schedules, allocation plans, lump sums, scenario update) → `markDirty(scenarioId)`

**Cache-check flow in `ProjectionOrchestrator.project()`:**
1. Load snapshot for scenario
2. If snapshot exists, `!dirty`, `valid_for_month == currentMonth`, and `snapshot_version == CURRENT_VERSION`: deserialize and return
3. Otherwise: run full engine computation, upsert snapshot with `dirty = false`, return result

**Force-refresh:** `POST /scenarios/{id}/projection?force=true` bypasses cache check, recomputes unconditionally. Wires to the existing "Re-run" button in the frontend.

**Implementation steps:**
1. New migration: drop `scenario_projections`, create `scenario_snapshots`
2. `ScenarioSnapshotDAO` — `upsert`, `getByScenarioId`, `markDirty(scenarioId)`, `markAllDirty()`
3. `ScenarioSnapshotAccessor` — wraps DAO; owns Jackson `ObjectMapper` serialization/deserialization of `ProjectionResult`; define `CURRENT_VERSION` constant here
4. Update `ProjectionOrchestrator.project()` — cache-check at top, upsert at bottom; accept `force: Boolean = false`
5. Update `ScenarioResource` — pass `?force` query param through to orchestrator; inject `ScenarioSnapshotAccessor`
6. Inject `ScenarioSnapshotAccessor` into all write resources and call `markDirty` / `markAllDirty` after successful writes

---

### Timezone Handling
Date values flow between the frontend, backend, and database in a mix of local and UTC contexts, causing subtle off-by-one issues. Known manifestations:

- **Frontend date inputs** — dates entered by the user (e.g. paycheck anchor dates, lump sum dates) are treated as local time by JavaScript. When serialized to ISO strings or passed as query params they may shift by a day depending on the user's UTC offset.
- **DB storage** — `DATE` columns (e.g. `month`, `paycheck_date`, `apply_date`) store dates without timezone; `TIMESTAMPTZ` columns (e.g. `confirmed_at`) are stored in UTC. Round-trip behavior between Kotlin's `LocalDate` / `Instant` and Postgres is generally safe via JDBI, but the frontend → backend boundary is not.
- **`currentMonthStr` on the frontend** — computed from `new Date()` using local time. In negative UTC offsets (e.g. US timezones), the first day of the month at midnight local time is still the last day of the prior month in UTC, which can cause the current month to be misidentified.

**Recommended fix direction:**
- Frontend: always derive `currentMonthStr` and format user-entered date values in UTC (`new Date().toISOString().slice(0,7) + '-01'` or equivalent), not local time.
- Backend: validate that all `LocalDate` parameters deserialized from query strings and JSON bodies are treated as calendar dates with no timezone conversion. Ensure no `Instant`-to-`LocalDate` conversions happen without an explicit zone.
- Audit any `new Date(isoString)` calls that feed into date comparisons — if the ISO string has no time component (e.g. `"2026-03-01"`), append `T12:00:00` before parsing to avoid UTC midnight rollback.

### Loan Payment Due Day
Add `paymentDueDay` to loans (migration + model + DAO), thread it through `EngineLoan` → `MonthlyLoanDetail` → projection API response → frontend `MonthlyLoanDetail` type. Use it to sort loan rows by due day in the projection page (same pattern as fixed expenses). Bump `ScenarioSnapshotAccessor.CURRENT_VERSION` when the shape change lands.

---

### End-of-Month Date Clamping for Fixed Expenses
When a fixed expense has a `dueDayOfMonth` of 29, 30, or 31 and the billing cycle falls in February, the projection engine must treat the due date as the last day of February rather than skipping or erroring. Apply the same clamping logic to any month shorter than the stored day (e.g. April 31 → April 30). The canonical rule: `min(dueDayOfMonth, lastDayOfMonth)`.

### Audit and Refactor Test Coverage
Audit tests across all layers and fill gaps with appropriate test types. Resource tests use `ResourceExtension` + MockK (mock the layer below). Orchestrator tests use MockK (mock accessors). DAO tests are integration tests against the real DB. Accessor tests are not needed — they are thin JDBI wrappers covered by DAO tests. Missing coverage to address: `LoanResource`, `ProjectedIncomeResource` (resource tests); `LoanDAO`, `AllocationDAO`, `ProjectedIncomeDAO` (DAO integration tests); `ProjectionOrchestrator` (orchestrator unit test).

### DB Integration Test Improvements
Expand integration test coverage for database interactions. Ensure tests run against a real Postgres instance (e.g. via Testcontainers) rather than mocks, cover edge cases in JDBI queries, and validate Liquibase migrations apply cleanly from a blank schema.

---

## Performance

### Projection Latency Optimization
The `/projection` endpoint was showing noticeable latency from full recompute on every request. Projection snapshots (now implemented) address the common case — cached results are served directly when nothing has changed. Remaining work if latency is still a concern: profile DB query count per request (N+1 risk in allocation/card loading), and consider whether the engine itself has any hot loops worth optimizing for large projection windows.

### Frontend Rendering Performance
The projection page is noticeably slow, particularly as the number of months, cards, and loans grows. Root causes to investigate:

- **Re-render scope** — `ProjectionPage` holds all state at the top level; any state change (edit field keystroke, section toggle, month expand) re-renders the entire tree. Identify hot paths with React DevTools Profiler.
- **Large list rendering** — the summary DataGrid renders all months at once (up to 360 rows). Evaluate whether MUI X DataGrid's built-in virtualization is active and effective, or whether a windowed list approach is needed for the expanded row content.
- **Memoization** — derived data (filtered card details, grouped income by month, ledger lookups) is recomputed on every render. Candidate selectors for `useMemo`; candidate callbacks for `useCallback`.
- **Expanded row content** — each expanded month renders multiple full tables. Once `ProjectionPage` is refactored into section components, `React.memo` on those components becomes straightforward.

Coordinate with the ProjectionPage refactor — component extraction is a prerequisite for effective memoization.

---

## Infrastructure

### Auth0 and User Profiles
Integrate Auth0 for authentication. Add user identity to the data model so all entities (cards, income sources, fixed expenses, plan settings) are scoped per user. Frontend acquires a JWT via Auth0 SDK and passes it as a Bearer token; backend validates the token and resolves the user on each request. User profile record stores Auth0 subject ID and any app-level preferences.


Ability to assign cards to a paycheck period instead of by due date
