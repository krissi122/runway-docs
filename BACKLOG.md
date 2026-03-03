# Runway — Backlog

Tracked improvements and tech debt items. Add entries here as they come up during development.

---

## Frontend

### Income Table: Meaningful Anchor Date Display
The `payAnchorDate` column is currently hidden from the income table because a raw date is not useful at a glance. For biweekly income, derive and display the next upcoming payday (or the next two) rather than the raw anchor. For semimonthly and monthly, the pay day columns already cover this. Revisit once the projection engine is wired up and we know what cadence data is available in the response.

### Migrate Dialogs to Reusable Components
Each entity (Card, FixedExpense, IncomeSource) now has separate Add and Edit dialogs, which duplicates the field definitions. Once all entities are built out, evaluate shared form field components (e.g. `CardFormFields`) that both Add and Edit dialogs compose — keeping dialog shells separate but eliminating field duplication. Also consider a shared `FormDialog` wrapper for the common `Dialog > DialogTitle > DialogContent > DialogActions / Cancel / Save` shell.

### Extract Reusable Validation Patterns
Each dialog re-implements its own inline `useMemo` validation. Common checks (non-empty string, positive number, day-of-month 1–31, month 1–12) are duplicated across files. Extract a small set of validator utilities or a `useFormField` hook so validation logic is consistent and testable in isolation.

---

## Backend

### End-of-Month Date Clamping for Fixed Expenses
When a fixed expense has a `dueDayOfMonth` of 29, 30, or 31 and the billing cycle falls in February, the projection engine must treat the due date as the last day of February rather than skipping or erroring. Apply the same clamping logic to any month shorter than the stored day (e.g. April 31 → April 30). The canonical rule: `min(dueDayOfMonth, lastDayOfMonth)`.

### DB Integration Test Improvements
Expand integration test coverage for database interactions. Ensure tests run against a real Postgres instance (e.g. via Testcontainers) rather than mocks, cover edge cases in JDBI queries, and validate Liquibase migrations apply cleanly from a blank schema.

---

## Infrastructure

### Auth0 and User Profiles
Integrate Auth0 for authentication. Add user identity to the data model so all entities (cards, income sources, fixed expenses, plan settings) are scoped per user. Frontend acquires a JWT via Auth0 SDK and passes it as a Bearer token; backend validates the token and resolves the user on each request. User profile record stores Auth0 subject ID and any app-level preferences.
