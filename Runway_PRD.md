---
title: Runway -- Product Requirements Document (PRD)
---

# 1. Problem Statement

Power users managing complex debt strategies (multiple credit lines,
equity compensation, variable income, utilization thresholds, hybrid
payoff strategies) are forced to use fragile spreadsheets.\
\
Spreadsheets:\
- Become overwhelming as rows grow\
- Hide logic in cells\
- Are hard to scenario-test cleanly\
- Don't model utilization psychology or thresholds well\
- Are painful to maintain long-term\
\
The user needs:\
- Structured modeling of debt payoff strategy\
- Threshold-based psychological milestones\
- Equity compensation modeling\
- Raise simulation\
- Liquidity buffers and emergency planning\
- Visualization that reduces anxiety

# 2. Target User

Primary Persona:\
High-income, high-leverage professional:\
- Multiple revolving credit lines\
- Executing aggressive payoff strategy\
- Equity compensation (RSUs, ESPP)\
- Forecasting raises/promotions\
- Wants to eliminate revolving debt within 12--24 months\
- Thinks in utilization brackets and psychological milestones\
\
Secondary Persona:\
- Finance-optimized professionals\
- FIRE-adjacent planners

# 3. Core Philosophy

This is NOT a budgeting app.\
\
It is:\
- A Debt Simulation Engine\
- A Stress Reduction Tool\
- A Strategic Planning Dashboard

# 4. Core Features

4.1 Debt Portfolio Manager\
User Inputs:\
- Creditor name\
- Balance\
- APR\
- Credit limit\
- Minimum payment\
- Category (User, Spouse, Emergency-only)\
- Risk tag\
- Active/Closed status\
\
System Calculates:\
- Utilization per card\
- Total utilization\
- Interest accrual\
- Time-to-payoff\
\
4.2 Strategy Engine\
Strategy Types:\
- Avalanche\
- Snowball\
- Hybrid Avalanche\
- Target Threshold Mode\
- Psychological Milestone Mode\
\
Engine auto-allocates payments per paycheck based on rules.\
\
4.3 Income Modeling\
User Inputs:\
- Salary\
- Net per paycheck\
- Pay frequency\
- Raise scenarios\
\
System Shows:\
- Net difference per paycheck\
- Timeline compression\
- Debt-free date under each scenario\
\
4.4 Equity Compensation Modeling\
RSUs:\
- Grant size\
- Vest schedule\
- Sell-to-cover %\
- Expected share price\
\
ESPP:\
- Offering price\
- Discount %\
- Sale timing\
\
System Simulates:\
- Lump sum injections\
- Immediate vs delayed sale\
- Card payoff impact\
\
4.5 Liquidity & Emergency Buffer\
User Defines:\
- Current savings\
- Minimum safety threshold\
- Rollover bucket\
- Emergency credit line\
\
System Calculates:\
- Derailment risk\
- Months of survival\
- Safe vs aggressive mode toggle\
\
4.6 Timeline Projection\
Monthly or paycheck-level timeline showing:\
- Balance per card\
- Total balance\
- Utilization %\
- Milestones achieved\
- Interest paid cumulative\
- Equity injections\
- Raise effects\
\
4.7 Visualizations\
- Total debt burn-down chart\
- Utilization over time\
- Psychological milestone tracker\
- Interest saved comparison\
- Aggression mode indicator

# 5. Calculation Engine Requirements

\- Accurate monthly interest accrual\
- Per-paycheck payments supported\
- Card-level amortization modeling\
- Conservative assumption toggles

# 6. Anxiety Reduction Features

\- Collapsible card views\
- Focus-only view\
- Highlight next payoff target\
- Utilization threshold badges (\<90%, \<75%, \<50%, \<30%)

# 7. Non-Functional Requirements

\- Fully local modeling (no bank integrations initially)\
- Deterministic calculations\
- No black-box math\
- Exportable to CSV\
- Versioned scenarios

# 8. Scenario Engine

User can clone and simulate:\
- Stock flat vs increased\
- Raise vs no raise\
- Emergency expense\
- Closing credit lines\
\
Compare:\
- Debt-free date\
- Interest paid\
- Milestone timing

# 9. Suggested Architecture

Backend:\
- Kotlin (Dropwizard)\
- Dagger (dependency injection)\
- Postgres\
- Dockerized\
\
Frontend:\
- React\
- MUI or Tailwind\
- TanStack Table\
- Recharts\
\
Future:\
- Mobile wrapper (Capacitor or React Native)

# 10. Phase Roadmap

Phase 1 -- Core Engine\
- Debt input\
- Payment strategy\
- Timeline simulation\
- Basic burn-down chart\
\
Phase 2 -- Equity Modeling\
\
Phase 3 -- Raise Simulation\
\
Phase 4 -- Scenario Cloning\
\
Phase 5 -- UX Polish

# 11. Success Metrics

\- Clear debt-free date\
- Reduced spreadsheet overwhelm\
- Confidence in strategy\
- Automatic interest modeling\
- Scenario simulation without manual recalculation

# 12. Future Expansion

\- SaaS for tech professionals with equity comp\
- Debt strategy operating system\
- Premium equity modeling tier
