# ADR-0051: Four plans, Razorpay payments and a hard stop at the AI allowance

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md): `agent_budgets` (new alert and credit columns), `payment_orders` and `payg_settlements`, added in the same change.
- [ADR-0029](0029-plan-7-budget-scope-tier-mapping-and-metering-ownership.md) D1: per-tenant pools and the Pro throttle band. This ADR supersedes the throttle band and the default enforcement mode.
- [ADR-0046](0046-superadmin-ops-console.md): the superadmin console, where staff now switch a workspace's plan.

## Status

**Accepted** (2026-11-14).

## Context

The product sold Free, Pro and Enterprise with no payment path ("payments are not connected"), a generous Free allowance (200,000 tokens), a Pro throttle band, and enforcement in `alert_only`, so nothing ever stopped. The owner asked for real plans, real payments through Razorpay, a hard stop when a workspace runs out, extra credits, a deposit-based Pay as you go plan, earlier admin warnings, a staff-only Unlimited plan for testing, and an email to the superadmins whenever a workspace is created, changes plan or sends a bug report.

## Decision

### D1. The catalog

| code | name | price (per workspace) | AI tokens |
| --- | --- | --- | --- |
| `free` | Free | $0 | 10,000 a month |
| `pro` | Pro | $25 a month | 200,000 a month |
| `ultra` | Ultra | $200 a month | 2,000,000 a month |
| `payg` | Pay as you go | $500 deposit per 30 days | no cap, $100 per 1,000,000 |
| `unlimited` | Unlimited | staff only, free | no cap |

Plans are new versions of the catalog (never edited in place); every subscription moves to the newest version of its plan.

### D2. A hard stop

`hard_cap = token_quota` on every plan and `BUDGET_ENFORCEMENT_MODE` defaults to `enforce`. At the allowance AI is paused for the whole workspace (captions and transcription continue). Extra credits ($10 per 100,000 tokens) add to `credit_tokens` for the current month and survive a plan change. No-cap plans use a cap of 10^15 so the agent tier needs no special case.

### D3. Payments

Razorpay Checkout. The server prices every purchase from the catalog, creates the order and records it in `payment_orders`; it applies the purchase only after verifying `HMAC_SHA256(order_id|payment_id, key_secret)`. A bought plan lasts 30 days and lapses back to Free unless bought again (noticed lazily and by the nightly job). Prices are in US dollars; charges default to INR at a configured rate (`RAZORPAY_INR_PER_USD`) because every Razorpay account accepts INR; `RAZORPAY_CURRENCY=USD` charges dollars where international payments are enabled.

### D4. Pay as you go

A $500 deposit opens a 30-day window with no cap. At its end the usage is priced and written to `payg_settlements`: under the deposit, the remainder is refunded through Razorpay; over it, the balance is due and must be paid before another plan purchase.

### D5. Staff controls and alerts

Staff can set any plan (Unlimited included) from the ops console with no payment and no lapse. Admins are alerted in-app at 75%, 90% and 95%, and see a banner; every member sees an AI-paused banner at 100%. The superadmins are emailed for a new workspace, a plan change, a payment, credits, and every bug report or feedback.

## Consequences

- Workspaces already over 10,000 tokens on Free are blocked when the migration runs, by intent.
- The staff plan switch runs billing's own plan code from `admin-ops-service` in-process (same codebase, same database, billing's module): a recorded exception to a service hop, chosen so no new service secret is needed.
- A failed verification after Razorpay captured a payment is not applied automatically (there is no webhook yet); the member is told to contact support with the payment id.
