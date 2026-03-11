# Design: Ledger Confirmation

## Overview

The ledger is the system of record for what actually happened each month. Once a month has passed, the user confirms actuals by entering the pre-payment balance for each card. The engine uses confirmed balances as authoritative starting points, replacing projections for past months.

---

## Core Semantic: What the User Enters

When confirming a month, the user enters the **pre-payment balance** for each card — the balance shown on the account before any payment for that month has been applied. This could be:

- The statement closing balance from the prior cycle
- The current balance checked before making a payment
- Whatever balance the card shows before that month's payment posts

The engine treats this as the authoritative **starting balance** for that month. It applies the month's scheduled payments and interest from that point, computing an ending balance. The ending balance becomes the starting balance for the next month's projection.

This replaces the need to enter payment amount or interest separately — those are derived by the engine.

---

## Data Model

The ledger covers both cards and loans via two parallel tables. Loans have no credit limit or utilization, but otherwise the confirmation semantics are identical — the user enters the pre-payment balance and the engine derives the rest.

### `monthly_card_ledger` (revised)

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `card_id` | `uuid` | No | FK → `cards` |
| `month` | `date` | No | First day of the month (YYYY-MM-01) |
| `pre_payment_balance` | `numeric(12,2)` | No | Balance before any payment for this month. The primary user input. |
| `payment_made` | `boolean` | No | Default `false`. User confirms the payment was actually made this month. |
| `confirmed_at` | `timestamptz` | No | Set when row is created. Not nullable — row creation IS confirmation. |
| `is_immutable` | `boolean` | No | Default `false`. Set to `true` when the month two periods prior is confirmed (auto-close). Locks `pre_payment_balance` only — `payment_made` remains editable on immutable rows. |

**Unique constraint:** `(card_id, month)`

**Removed from earlier schema:** `actual_payment`, `actual_interest`, `actual_ending_balance` — these are not stored; the engine derives them using `pre_payment_balance` as the starting point for that month's run.

---

### `monthly_loan_ledger`

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uuid` | No | PK |
| `loan_id` | `uuid` | No | FK → `loans` |
| `month` | `date` | No | First day of the month (YYYY-MM-01) |
| `pre_payment_balance` | `numeric(12,2)` | No | Balance before any payment for this month. The primary user input. |
| `payment_made` | `boolean` | No | Default `false`. User confirms the payment was actually made this month. |
| `confirmed_at` | `timestamptz` | No | Set when row is created. Row creation IS confirmation. |
| `is_immutable` | `boolean` | No | Default `false`. Auto-set to `true` when the second subsequent month is confirmed. Locks `pre_payment_balance` only — `payment_made` remains editable on immutable rows. |

**Unique constraint:** `(loan_id, month)`

No utilization columns — loans have no credit limit.

---

## Query Patterns

### Determining the read source by month

The same logic applies to both cards and loans, using their respective ledger tables.

Given today's date, for any month M in the projection:

```
if M < current_month:
    if ledger row exists for (entity, M) → use pre_payment_balance as starting balance for M
    else → month is UNCONFIRMED (flag in UI; fall back to entity.balance or prior confirmed ending balance)

if M == current_month:
    if ledger row exists for (entity, M) → use pre_payment_balance as starting balance for M
    else → use most recent confirmed ending balance (prior month's engine output), or entity.balance if none

if M > current_month:
    always projection — use scenario_projections / live engine output
```

### Starting balance resolution for engine (on recompute)

For each card and each loan, to determine the starting balance for the projection:

1. Find the most recent ledger row for the entity where `month <= current_month`
2. Run the engine forward from that month using `pre_payment_balance` as the start — the engine's output for that month gives the ending balance, which is the starting balance for month+1
3. If no ledger row exists → fall back to `entity.balance` (`cards.balance` or `loans.balance`)

### Example: today is March 15

| Month | Ledger row? | Engine starting balance source |
|---|---|---|
| January | Confirmed | Jan `pre_payment_balance` → engine derives Jan ending balance → Feb starting balance |
| February | Confirmed | Feb `pre_payment_balance` → engine derives Feb ending balance → Mar starting balance |
| March | Confirmed | Mar `pre_payment_balance` → engine runs Mar forward |
| March | **Not confirmed** | Feb confirmed ending balance (or `cards.balance`) as Mar starting balance |
| April+ | N/A | Projection from Mar ending balance |

### Example: today is March 15, January is unconfirmed

| Month | Ledger row? | Result |
|---|---|---|
| January | **Missing** | Flag as unconfirmed in UI. Projection uses `cards.balance` as starting point — accuracy degrades from here. |
| February | Confirmed | Feb `pre_payment_balance` used; Jan gap noted but Feb is still the best anchor. |
| March | Not confirmed | Falls back to Feb's engine-derived ending balance. |

The engine always uses the **most recent confirmed month** as the anchor and projects forward from there. Gaps in ledger history don't break the projection — they just reduce accuracy for the months around the gap.

---

## Immutability

A confirmed ledger row becomes immutable automatically when the **second subsequent month** is confirmed. This gives the user a one-month grace window to correct a confirmation before it locks.

**Example:**
- Confirm January → January row created, `is_immutable = false`
- Confirm February → January row set to `is_immutable = true`; February row created, `is_immutable = false`
- Confirm March → February row set to `is_immutable = true`

Immutable rows are read-only. An immutable row can only be unlocked by an explicit admin action (out of scope for now — assume irreversible once locked).

The auto-close trigger fires server-side: whenever a new ledger row is written for month M, the row for month M-2 (if it exists) is set to `is_immutable = true`.

---

## API

### Confirm a month
```
POST /ledger/confirm
```
Body:
```json
{
  "month": "2026-03-01",
  "cards": [
    { "cardId": "uuid", "prePaymentBalance": "1842.57", "paymentMade": true },
    { "cardId": "uuid", "prePaymentBalance": "4200.00", "paymentMade": false }
  ],
  "loans": [
    { "loanId": "uuid", "prePaymentBalance": "8200.00", "paymentMade": true }
  ]
}
```
- Creates one `monthly_card_ledger` row per card entry and one `monthly_loan_ledger` row per loan entry (upsert on their respective unique constraints if not immutable)
- Triggers auto-close of month-2 rows for both cards and loans
- Updates `cards.balance` and `loans.balance` to the engine-derived ending balance for each confirmed entity
- Returns the created/updated rows

### Get ledger entries for a month
```
GET /ledger?month=2026-03-01
```
Returns all card and loan ledger entries for that month.

### Get unconfirmed past months
```
GET /ledger/unconfirmed-months
```
Returns a list of months (before the current month) that have at least one card or loan with no confirmed ledger entry. Used by the UI to drive alerts.

### Mark payment made / unmade
```
PATCH /ledger/cards/{id}/payment
PATCH /ledger/loans/{id}/payment
```
Body: `{ "paymentMade": true }`

Always allowed — `is_immutable` does not block this. The balance is locked but the user can update `payment_made` at any time, including on immutable rows.

### Delete / unconfirm (if not immutable)
```
DELETE /ledger/cards/{id}
DELETE /ledger/loans/{id}
```
Returns 409 if `is_immutable = true`. Otherwise deletes the row, reverting that entity/month to unconfirmed state.

---

## UI Design

### Where confirmation lives

The **Projection Summary tab** is the confirmation entry point. Each month row in the summary table gains a status indicator and a confirm action.

### Month row states

| State | Indicator | Action available |
|---|---|---|
| Future | — | None |
| Current month, unconfirmed | "Pending" chip | "Confirm" button |
| Current month, confirmed | "Confirmed" chip | "Edit" (if not immutable) |
| Past month, confirmed | "Confirmed" chip | "Edit" (if not immutable) / balance locked if immutable, but payment checkbox still editable |
| Past month, **unconfirmed** | "Needs confirmation" warning chip | "Confirm" button |

### Confirm dialog

Opens when the user clicks "Confirm" on a month row. Shows all cards and loans together in a single dialog — no stepping through one at a time.

Contains:

- Month label ("Confirming February 2026")
- Brief instruction: "Enter the balance for each account before that month's payment was made."
- **Cards section**: one input row per active card — card name, balance field (pre-filled with the projected value as a suggestion), "Payment made" checkbox
- **Loans section**: one input row per active loan — loan name, balance field (pre-filled with the projected value as a suggestion), "Payment made" checkbox
- Cards and loans that were already paid off (projected balance = $0) are shown as read-only at $0.00 with the payment checkbox hidden — no input needed
- Save button triggers `POST /ledger/confirm`
- On success: dialog closes, projection reruns from that month forward, summary table refreshes

### Unconfirmed past month alert

If today is past the 1st and there are unconfirmed months before the current month, the Projection page shows a banner at the top:

> "2 past months need confirmation. Your projections may be inaccurate until actuals are entered."

Clicking the banner scrolls to the earliest unconfirmed month row. The row itself is visually distinct (e.g. amber/warning background or icon in the month column).

### Behavior when a past month is unconfirmed

- The unconfirmed month row still displays projection data (not blanked out)
- The row has a warning indicator
- The month is flagged in the summary with a tooltip: "Unconfirmed — showing projected values"
- The engine still runs and produces future projections using `cards.balance` (or the nearest confirmed anchor) as the starting point
- Future projections are still shown — the tool doesn't block on missing confirmations, it just surfaces the accuracy gap

---

## `cards.balance` and `loans.balance` — Kept Current via Confirmation

Both `cards.balance` and `loans.balance` are user-set fields with no automatic update logic. Left unaddressed, they become stale quickly — a card set at $5,000 six months ago still reads $5,000 even if the real balance is $2,800. Any unconfirmed fallback to the stored balance would produce a materially wrong projection.

**Resolution: auto-update on confirmation.**

When the user confirms month M, the engine runs from `pre_payment_balance` and derives an ending balance for that month. As a side effect of the confirmation write, update `cards.balance` / `loans.balance` to that engine-derived ending balance. This keeps the stored balance current without requiring the user to manually edit their cards or loans each month.

**Implications:**
- `cards.balance` and `loans.balance` mean "most recent confirmed ending balance" once the user has confirmed at least one month
- A user who edits these balances manually will have them overwritten on the next confirmation — this is acceptable; confirmation is more authoritative than a manual edit
- For entities that have never been confirmed, the stored balance still reflects the original setup value (stale risk remains, but this is an onboarding concern, not a ledger design concern)

---

## What Changes for `scenario_projections`

Currently projections are fully stateless (computed live on every request). The ledger system doesn't require caching `scenario_projections` — the projection endpoint can continue to run live. The engine simply uses confirmed `pre_payment_balance` values as starting anchors for confirmed months, and falls through to `cards.balance` otherwise.

Caching `scenario_projections` to DB is a separate performance optimization (tracked in BACKLOG.md). The ledger design works correctly with or without that cache.

---

## Pool Balance

The liquidity pool is scenario-specific and entirely synthetic — it is not a real bank account. There is no "actual" pool balance to confirm. The pool balance for past months is always derived by running the engine from the confirmed starting balances. No pool ledger entry is needed or created during confirmation.

---

## Open Questions / Deferred

- **income_ledger usage**: The `income_ledger` table schema exists but there is no confirmation workflow for income. Income actuals don't affect the engine (which derives income from `income_sources` and `projected_income`). Deferred — treat income ledger as a future audit/history feature.
- **Partial month confirmation**: Confirming some cards but not all for a given month. The API supports partial confirmation (one row per card). The engine uses whichever cards are confirmed and falls back to `cards.balance` for the rest. The UI should indicate per-card confirmation status in the dialog, not just per-month.

## Backlog

- **(Maybe) Inline ledger editing in the Projection Summary table**: Rather than opening a dialog, allow the user to confirm a month by editing balance and payment-made fields inline in the expanded card sub-rows of the summary table. Would replace the dialog for power users who prefer a faster confirmation flow. Evaluate after the dialog UX is validated.
