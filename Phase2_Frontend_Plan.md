---
title: Runway -- Phase 2 Frontend Implementation Plan
---

# Phase 2 Frontend: Entity Management UI

## Context

Backend Phase 2 is complete. The three global input entities — Card, FixedExpense, and IncomeSource — have `GET` (list all) and `POST` (create) endpoints. This plan covers the frontend UI for those endpoints.

---

## Goals

- Add client-side routing with a persistent navbar
- Build per-entity pages: list (DataGrid) + create (modal form)
- Establish feature-first frontend architecture for all future pages

---

## Architecture Principle: Feature-First

The frontend is organized by feature/page, not by layer. Types and API calls live alongside the page component they belong to. There is no global `types/` or `api/` folder.

Shared, cross-cutting concerns (`NavBar.tsx`, `App.tsx` routing) live at the `src/` root. This "bleed" is accepted.

---

## Directory Structure

```
src/
├── cards/
│   ├── CardsPage.tsx          DataGrid list + Add Card dialog
│   ├── cardsApi.ts            getCards(), createCard()
│   └── cardTypes.ts           Card, CreateCardRequest interfaces
├── fixedExpenses/
│   ├── FixedExpensesPage.tsx
│   ├── fixedExpensesApi.ts
│   └── fixedExpenseTypes.ts
├── income/
│   ├── IncomePage.tsx
│   ├── incomeApi.ts
│   └── incomeTypes.ts
├── home/
│   └── HomePage.tsx           Health check UI (extracted from App.tsx)
├── NavBar.tsx                 MUI AppBar with nav links
└── App.tsx                    BrowserRouter + NavBar + Routes
```

---

## New Packages

- `react-router-dom` — client-side routing
- `@mui/x-data-grid` — entity list tables (installed now, used in incremental UI phase onward)

---

## Routes

| Path | Page |
|------|------|
| `/` | HomePage (health check) |
| `/cards` | CardsPage |
| `/fixed-expenses` | FixedExpensesPage |
| `/income` | IncomePage |

---

## NavBar

MUI `AppBar` + `Toolbar`. Brand name "Runway" links to `/`. Nav links use react-router-dom with active-route highlighting.

---

## Entity Pages — Shared Pattern

Each page:
1. Fetches list on mount via its API module
2. Renders an MUI X DataGrid (read-only at this phase)
3. Has a "+ Add" button that opens an MUI Dialog with a create form
4. On successful POST: closes dialog, re-fetches list

No inline editing, no delete — CRUD extensions come in Phase 3.

---

## Data Types

All monetary and rate fields (`balance`, `apr`, `amount`) are typed as `string` on the frontend to preserve BigDecimal precision from the JSON response.

**APR handling (Card):** The backend stores APR as a decimal (e.g. `0.2499` = 24.99%). The UI inputs APR as a percentage and divides by 100 before sending. The DataGrid displays it multiplied back to percentage.

---

## CardsPage

**DataGrid columns:** Name, Balance ($), APR (%), Credit Limit ($), Payment Due Day, Active

**Create form fields:** Name, Balance, APR (as %), Credit Limit, Payment Due Day (1–31), Active

---

## FixedExpensesPage

**DataGrid columns:** Name, Amount ($), Billing Interval (every N months), Anchor Month (month name), Active

**Create form fields:** Name, Amount, Billing Interval (select 1–12), Anchor Month (select, shown as month names), Active

---

## IncomePage

**DataGrid columns:** Name, Amount ($), Frequency, Pay Day 1, Pay Day 2, Anchor Date, Active

**Create form fields:** Name, Amount, Frequency (select: MONTHLY / SEMIMONTHLY / BIWEEKLY), conditional fields:
- MONTHLY: no extra fields
- SEMIMONTHLY: Pay Day 1 + Pay Day 2 (1–31)
- BIWEEKLY: Pay Day 1 (optional) + Pay Anchor Date

---

## Out of Scope (Phase 2)

- Edit / Delete (Phase 3)
- Projection endpoint integration (Phase 3+)
- DataGrid inline editing (incremental UI phase)
- Multi-scenario support (P1+)
