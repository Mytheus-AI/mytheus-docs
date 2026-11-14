# 25. Plans, payments and the console fixes

Not a plan band ([ADR-0051](../adr/0051-plans-payments-and-hard-token-stops.md)). Status: **DONE WITH GAPS** (built and verified locally on 2026-11-14; committed to `main` in backend, UI, platform and docs; deployment and a live Razorpay test payment are the gaps).

One session's list from the owner: suspension, the ops cost and audit views, the Briefings voice card, the Workspace overview, paid plans with Razorpay, staff emails, the Decisions toolbar, Home and Ask Myte tidying.

## What landed, per repo

### mytheus-backend

- Suspension: `libs/shared/org_resolver.py` still answers a suspended tenant with 401 `AUTH_REQUIRED`, but adds `reason: tenant_suspended`; `api_gateway/edge.py` passes a problem's extra fields through. Unknown and soft-deleted orgs stay byte-identical.
- Ops cost and console views read the larger of `agent_budgets.tokens_consumed` (live) and the nightly `usage_aggregates`, so usage no longer reads 0 until the rollup runs.
- `admin_ops_service/pipeline.py`: dead-letter counts (every `{topic}.dlq`, high minus low watermark) and consumer lag (high watermark minus committed offset per group and topic) from Redpanda with the Kafka admin API, in a worker thread with short timeouts; `null` only when Redpanda does not answer. Wired into `GET /admin/ops/costs` and `GET /admin/ops/health`.
- The audit search's 403 carries `reason: no_support_grant`.
- Migration `a7c3e9f1b2d4`: the catalog of ADR-0051 D1, subscriptions moved to the newest plan versions, this month's budgets rescaled with `hard_cap = token_quota`, `agent_budgets.alert_75_sent`, `alert_90_sent`, `credit_tokens`, and the `payment_orders` and `payg_settlements` tables (RLS `tenant_isolation`).
- `billing_metering_service/budgets.py`: hard stop, alerts at 75/90/95, `enforce` by default, `set_subscription_plan`, `sync_budget_to_plan` (rescales the month, re-arms alerts no longer crossed), `add_credit_tokens`, and the lazy lapse of a bought month (settling Pay as you go first).
- `billing_metering_service/payments.py`: `POST /billing/checkout`, `POST /billing/checkout/verify` (signature check, then apply in one transaction, then the Redis mirror and the staff email after commit), `GET /billing/status` (member: plan and whether AI is paused; admins also usage).
- `billing_metering_service/payg.py`: window pricing, settlement, Razorpay refunds of the remainder; the nightly job (`jobs.py`) lapses plans and retries refunds. `GET /usage` gains `paid_until`, `payg`, `balance_due`, `last_settlement`.
- `POST /admin/ops/tenants/plan` (staff): any plan, Unlimited included, no payment.
- `libs/shared/staff_mail.py`: one designed email per kind (new workspace, plan change, payment, credits, bug report, feedback) to `STAFF_ALERT_EMAILS` or each staff account's Clerk email, through the existing Resend/SMTP sender; best effort, after commit. Called from both provisioning paths, checkout, the staff plan switch and `POST /admin/tenant/feedback`.

### mytheus-ui

- `/suspended`: `api()` redirects there on `tenant_suspended` (`apiSafe` lets the redirect through); the page signs the member out and explains. Public in the middleware.
- Ops: Cost and lag and Health show dead letters and lag; the audit viewer explains a missing support grant; the workspace panel switches plans; plan names read as product names; no-cap plans show no quota.
- Settings > Briefings draws the Myte from the profile id like the rest of the app.
- Workspace overview: an AI this month card fills the right column; the plan card has an Upgrade button.
- `/workspace/billing/plans`: Free, Pro, Ultra and Pay as you go as cards, the current one glowing in `--signal-line` / `--signal-glow`; Razorpay Checkout in Mytheus colours (`components/billing/CheckoutButton.tsx`); extra credits (`CreditsCard`); a balance-due panel. Usage and billing: Upgrade, the 75% and AI-paused notes, a Payments card with the Pay as you go window and last settlement.
- `BudgetBanner` in the app shell: AI paused for everyone at 100%; admins see the 75/90/95 steps, dismissible per step.
- Marketing pricing rewritten to the four plans.
- Decisions: the meeting filter sits in the search row. Home: the starter chips and Recent decisions removed. Ask Myte: two suggestions, the composer on one line with scope as icon buttons with tooltips (`components/ask-myte/Tip.tsx`), follow-ups as an Ask next list, answer actions as one row of icon buttons.
- Wire types regenerated (`npm run generate:api`).

### mytheus-platform

- `env/prod.env.example`: `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_CURRENCY`, `RAZORPAY_INR_PER_USD`, `BUDGET_ENFORCEMENT_MODE`, `STAFF_ALERT_EMAILS`.

### mytheus-docs

- `_canon/02-data-model.md`: `agent_budgets` columns, `payment_orders`, `payg_settlements`. ADR-0051.

## Deviations, and why

- A suspended workspace's own members are told it is suspended (ADR-0010 D2 kept one response for every case). The caller holds a Clerk-verified token for that organization, so it already knows the workspace exists; telling it nothing let every read fail into an application error.
- The staff plan switch calls billing's module in-process rather than over a service hop, to avoid a new service secret on the server (ADR-0051 Consequences).
- Prices stay in US dollars and are charged in INR by default: the owner's prices are in dollars, and INR works on every Razorpay account without enabling international payments.
- Pay as you go price per token ($100 per million, the same rate as Ultra) and the credit pack ($10 per 100,000) were not given by the owner; chosen so Pay as you go is never cheaper per token than Ultra's allowance.

## Open gaps

- Deploy, then a Razorpay test-mode payment end to end (keys go in `prod.env`, auto-capture on in the Razorpay dashboard).
- No Razorpay webhook: a payment captured but not verified (browser closed mid-way) is not applied automatically.
- ~~The budget mirror in Redis keeps the old allowance for a workspace until its next charge or plan change after the migration.~~ Closed 14 November 2026: `GET /billing/status` republishes it when the row is blocked (see below).
- Mobile shows none of the plan screens.

## Verification

- Backend: `uv run pytest tests/test_org_resolver.py tests/test_gateway_edge.py tests/test_admin_ops_api.py tests/test_admin_ops_pipeline.py tests/test_billing_metering.py tests/test_billing_checkout_unit.py tests/test_staff_mail.py tests/test_openapi_contract.py`: all pass; the database-backed tests skip without Docker locally and run in CI. `ruff`, `black`, `mypy` clean on every touched package; `python -m scripts.emit_openapi` and `scripts.check_authz` run.
- UI: `npx tsc --noEmit` clean; `node --test tests/ops.test.ts tests/ops-console.test.ts` 13 pass; eslint clean on touched files.

## 14 November 2026: what "AI is paused" actually stops, and owner UI fixes

The owner saw the AI-paused banner while Ask Myte, its popup and New meeting all still worked. Two causes: the app never read the paused state anywhere but the banner, and the agent tier's budget gate reads the Redis mirror, which could still carry a pre-migration allowance (the open gap above).

### mytheus-backend

- `services/core_api/routes/meetings.py`: `create_meeting` reads this month's `agent_budgets.state` under the caller's RLS (read only) and answers 402 `QUOTA_EXCEEDED` when it is `blocked` and `BUDGET_ENFORCEMENT_MODE=enforce`. Covers the app, the API and Capture (the extension creates core-api meetings). `alert_only` never refuses, matching the agent gate.
- `services/billing_metering_service/payments.py`: `GET /billing/status` republishes the mirror (`write_mirror`) when the row is `blocked`; a Redis failure is logged and the status still answers. Every member's page load reads this, so the gate catches up within one page view.
- Tests: `tests/test_meetings_api.py` (refused while blocked under `enforce`, allowed under `alert_only`; database-backed, run in CI), `tests/test_billing_checkout_unit.py` (republish when blocked, not otherwise, Redis down still answers).

### mytheus-ui

- `src/lib/ai-status.tsx`: `AiStatusProvider` (set from `billing.blocked` in `(app)/layout.tsx`), `useAiPaused()`, and `AiPausedNote` (what is off, plus Buy credits or upgrade for admins, "your admin can turn it back on" for members).
- Paused: Ask Myte page and Ctrl+J sheet swap the composer for the note, starters and Talk are disabled, nothing is sent (an `/ask?q=` hand-off included); the Home ask box is disabled; the sidebar New meeting and the meetings-hub New meeting are disabled (a `?new=1` link no longer opens the form); Capture shows the note; `createMeeting` maps a 402 to a plain paused message.
- `BudgetBanner`: a card in the page flow (not sticky, no 1200px inner cap), says Mytes, Ask Myte, Talk and new meetings are off, and publishes its height as `--ai-banner-h`; the Ask Myte frame subtracts it, so the page is no longer cut off at the bottom under the banner.
- Owner fixes, one commit each: Ask Myte history says New chat / Your chats and its rail header lines up with the chat header (h-14, divided); Your Myte drops its ask box and Open Ask Myte; Save Myte sits under the Voice card (`AgentPanel` `beforeSave`); Topics you watch suggestion chips and add row tidied; Settings > Briefings drops the Voice card (the voice is set on Your Myte only) and puts speed beside language; Settings > Profile drops the workspace logo (Workspace console only); the Extra AI credits card no longer glows.

### Deviations, and why

- ADR-0051 D2 said captions and transcription continue while paused; the owner asked that no new meeting start, since meetings are where the Mytes work. Amended in the ADR; meetings already running keep captions.
- core-api reads `agent_budgets` directly (read only, under RLS) rather than calling billing over HTTP, the same in-process read admin-ops already makes; billing stays the one writer.

### Verification

- Backend: `uv run pytest tests/test_billing_checkout_unit.py tests/test_openapi_contract.py` pass; `python -m scripts.emit_openapi --check` matches; ruff, black, mypy clean on touched modules. `tests/test_meetings_api.py` needs Docker (CI).
- UI: `npx tsc --noEmit` clean, eslint clean on touched files; `npm test` 720 of 722 pass, the 2 failures (`tests/skeletons.test.ts`) fail identically without these changes (Windows path separators).
