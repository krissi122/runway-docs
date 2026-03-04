---
title: Debt Payoff Simulator -- MVP Implementation Plan
---

> **Note:** Design decisions made after this document was written are captured in `Implementation_Appendix.md`. Where they conflict, the appendix takes precedence.

# 1. Purpose

This document outlines the implementation plan for the Debt Payoff
Simulator MVP.

The goal of this phase is to build:
- Containerized backend and frontend
- Full-stack CRUD for all engine input entities
- Stateless projection engine endpoint
- Incremental UI rendering from projection output
- Clean separation between engine logic and presentation

# 2. Architectural Principles

- Backend owns all financial projection logic.
- ProjectionEngine must be pure (no DB, no framework dependencies).
- Frontend holds draft state and sends full payload to backend for recalculation.
- No duplication of amortization or threshold logic in frontend.
- MVP supports one scenario at a time: the single ACTIVE scenario. Multi-scenario branching is out of scope.
- Incremental build approach: get real data flowing before building projection views.

# 3. Tech Stack

Backend:
- Kotlin
- Dropwizard
- Dagger (dependency injection — introduced in Phase 4)
- JDBI (persistence)
- Liquibase (schema management)
- Postgres

Frontend:
- React + TypeScript (Vite)
- MUI
- MUI X DataGrid
- MUI X Charts

Infrastructure:
- Docker Compose (Postgres, Backend, Frontend)

# 4. Phase 1 -- Foundation (Containers & Routing) ✅

- Docker Compose with postgres, backend, frontend
- GET /health
- POST /projection (stub)
- All Liquibase migrations

# 5. Phase 2 -- Global Entities (Full Stack)

Wire JDBI into Dropwizard (JdbiFactory, register on Environment). Pass
Jdbi to resources directly — defer Dagger until Phase 4.

For each global entity (no FK dependencies, fully independent):

**cards**
- Backend: CardRow, CardDao, CardResource — full CRUD endpoints
- Frontend: Cards management UI (list, create, edit, delete)

**fixed_expenses**
- Backend: FixedExpenseRow, FixedExpenseDao, FixedExpenseResource — full CRUD endpoints
- Frontend: Fixed Expenses management UI (list, create, edit, delete)

**income_sources**
- Backend: IncomeSourceRow, IncomeSourceDao, IncomeSourceResource — full CRUD endpoints
- Frontend: Income Sources management UI (list, create, edit, delete)

# 6. Phase 3 -- Scenario & Scoped Entities (Full Stack)

MVP has exactly one ACTIVE scenario. The scenario record is created once
(seeded or via a first-run flow) and is never switched. Scoped entities
always belong to this single scenario.

**scenarios**
- Backend: ScenarioRow, ScenarioDao, ScenarioResource — full CRUD endpoints
- Frontend: Scenario management UI (view/edit the active scenario)

~~**plan_settings**~~ — **DEPRECATED.** Replaced by `allocation_plans` + `plan_allocations`. See `Implementation_Appendix.md` §A1.

**allocation_plans** + **plan_allocations** (replaces plan_settings)
- Backend: AllocationPlanRow, PlanAllocationRow, AllocationPlanDao — full CRUD under /scenarios/{id}/allocation-plans
- Auto-close behavior: creating a new plan automatically closes the previous open-ended plan
- Frontend: Allocations page — DataGrid of plans with Add/Edit dialog for date range + allocation target list

**card_payment_schedules** (one active row per card per scenario in MVP)
- Backend: CardPaymentScheduleRow, CardPaymentScheduleDao — full CRUD under /scenarios/{id}/card-schedules
- Frontend: Payment schedule management UI per card

**lump_sums**
- Backend: LumpSumRow, LumpSumDao — full CRUD under /scenarios/{id}/lump-sums
- Frontend: Lump Sum management UI (list, create, edit, delete)

# 7. Phase 4 -- Projection Engine + /projection Wiring

With real data flowing, build and wire the engine incrementally.

- Introduce Dagger to manage the dependency graph (Jdbi, DAOs, engine)
- Define clean engine domain types (Card, PlanSettings, LumpSumEvent, etc.)
  — no UUIDs or timestamps; these are separate from the Row models
- Implement ProjectionEngine (pure Kotlin, no DB or framework deps)
- Outputs: MonthlySummary rows, MonthlyCardDetail rows, threshold crossing metadata
- Implement:
  - Monthly debt budget calculation
  - Base payment logic
  - Extra allocation (focus/secondary split)
  - Interest application after payments
  - Payoff rollover behavior
  - Utilization tracking
  - Threshold crossing detection
- Unit test each calculation rule in isolation
- Integration tests: seed data via API → run engine → assert output
- Wire POST /projection:
  - Pull inputs from DB by scenario_id
  - Run engine
  - Persist results to scenario_projections and monthly_card_ledger
  - Return projection output
- Maintain /projection as stateless (accepts full payload for recalculation without persistence)

# 8. Phase 5 -- Frontend Projection Views

- Timeline DataGrid (monthly summary + per-card detail)
- Debt burn-down chart (MUI X Charts)
- Utilization chart
- Paycheck view derived from monthly projection

# 9. Non-Goals for MVP

- No RSU/ESPP modeling
- No raise scenario engine
- No multi-scenario branching (one ACTIVE scenario only)
- No authentication
- No daily interest modeling

# 10. Development Philosophy

- Build incrementally.
- Keep projection engine deterministic and well-tested.
- Avoid premature abstraction.
- Favor clarity over cleverness.
- Optimize for correctness first, polish second.
