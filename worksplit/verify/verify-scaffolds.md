# Verification: the five scaffold services in `mytheus-backend/services/`

Read-only pass, 2026-09-07. Root `/Users/sushan/Projects/Mytheus`. Nothing modified.

Total application code across all five: **188 lines including blanks and docstrings**, of which
57 are one SQLAlchemy model and 42 are a Temporal worker. Every `main.py` is 11 lines.

```
admin_ops_service/main.py            11
admin_ops_service/models.py          57
billing_metering_service/main.py     11
integration_service/main.py          11
notification_service/main.py         11
notification_service/consumer.py     45
workflow_service/worker.py           42
workflow_service/workflows/*.py      21 + 14 + 11 + 22
```

---

## 1. What exists now, per service

### `billing_metering_service` — 11 lines, zero routes

`mytheus-backend/services/billing_metering_service/main.py:1-11`, entire file:

```python
"""billing-metering-service — Stripe billing, usage metering, quota enforcement (R4).

Owns: plans, subscriptions, usage_events, usage_aggregates, quotas, invoices, agent_budgets.
Budget states: ok -> throttled -> blocked. Quota enforcement never sheds durable facts.
"""

from libs.shared.app_factory import create_app

app = create_app("billing-metering-service")

# TODO: add routers for /billing /usage /plans /stripe/webhook
```

No models module. No `plans`, `subscriptions`, `usage_events`, `quotas`, `agent_budgets`
tables in any migration (`mytheus-backend/migrations/versions/` — 11 files, none billing).

### `integration_service` — 11 lines, zero routes

`mytheus-backend/services/integration_service/main.py:1-11` is the same shape; the only
distinguishing line is `main.py:11`: `# TODO: add routers for /oauth/{provider} and /webhooks/{provider}`.
No models, no token storage, no envelope encryption, no provider client.

### `notification_service` — 11-line app + a 45-line consumer that is never run

`notification_service/main.py:11`: `# TODO: add router for /notifications (read/dismiss REST surface)`.
`consumer.py` is analysed in §5 below.

### `admin_ops_service` — 11-line app + a real 57-line model

`admin_ops_service/main.py:11`: `# TODO: add routers for /admin/tenant/* and /admin/ops/*`.
`admin_ops_service/models.py:25-57` is the `AuditLog` model — partitioned by `occurred_at`
(`models.py:56`), `actor_type` check constraint (`models.py:55`), FK to `organizations`
(`models.py:33`). It is a real, migrated, RLS-policied, append-only-by-GRANT table
(`migrations/versions/c3e8b1d47f92_0004_audit_logs_append_only.py`).

### `workflow_service` — a real worker with four empty workflows

`worker.py:24-38` connects to Temporal and registers four workflow classes. All four bodies
are `pass` or a bare sleep loop:

- `workflows/meeting_lifecycle.py:9-11` — `# TODO`, `pass`
- `workflows/dunning.py:12-14` — `# TODO`, `pass`
- `workflows/rtbf.py:14-22` — six-step deletion saga entirely in comments, `pass`
- `workflows/action_followup.py:15-21` — real `workflow.sleep` timers at day 1/3/7/10 with
  `# TODO: call notification activity` where each reminder would go. It sleeps ten days and
  does nothing.

Nothing anywhere calls `start_workflow` or `execute_workflow` — grep over
`mytheus-backend/services` and `libs` returns only the four `from temporalio import workflow`
imports and the worker's own `Client`/`Worker` imports. No workflow is ever started.

---

## 2. Gateway routes that 404 today

`services/api_gateway/edge.py:146-180` `ROUTE_MAP`. Nine prefixes resolve to a scaffold:

| prefix | `edge.py` line | upstream | result today |
|---|---|---|---|
| `/billing` | 171 | billing-metering-service | 404 from the service |
| `/usage` | 172 | billing-metering-service | 404 |
| `/plans` | 173 | billing-metering-service | 404 |
| `/stripe/webhook` | 174 | billing-metering-service | 404 — **and it is in `PUBLIC_PREFIXES` (`edge.py:52`)**, so Stripe reaches an unauthenticated 404 and will retry, then disable the endpoint |
| `/notifications` | 175 | notification-service | 404 |
| `/admin/tenant` | 176 | admin-ops-service | 404 |
| `/admin/ops` | 177 | admin-ops-service | 404 |
| `/oauth` | 178 | integration-service | 404 — also public (`edge.py:54` `"/oauth/"`), so an OAuth callback dead-ends |
| `/webhooks` | 179 | integration-service | 404 |

These are real 404s from a live upstream, not gateway 404s — `proxy.py:100-137` resolves
`http://<service>:8000` and forwards; only an unmapped path yields the gateway's own
`problem_response(404, ...)` at `proxy.py:105`. The compose comment at
`compose.prod.yml:318-321` states this was the deliberate intent.

Two of the nine are asserted by tests, so removing them from the map breaks the suite:
`tests/test_gateway_edge.py:270-271` and `tests/test_gateway_proxy.py:175-176`
(`/billing/subscriptions`, `/admin/ops/flags`, `/stripe/webhook`).

UI dependents are thin: `mytheus-ui` links to a `/billing` page in three nav components
(`src/components/app/Sidebar.tsx:40`, `Topbar.tsx:15`, `SettingsTabs.tsx:13`) and
`src/lib/api/org-context.ts:10` documents that `org.plan` is hardcoded `"starter"` because
`subscriptions` does not exist. `settings/integrations/page.tsx:1-40` is entirely
`mockIntegrations` with a comment saying the toggles are "visual only until the
integration-service wiring lands".

---

## 3. Compose presence — all five are deployed

`mytheus-platform/deploy/compose.prod.yml`:

- `workflow:` line 226 — `python -m services.workflow_service.worker`, on the app DB role
- `integration-service:` line 322
- `billing-metering-service:` line 329
- `admin-ops-service:` line 342
- `notification-service:` line 349 — **`uvicorn services.notification_service.main:app` only**

Lines 318-321 label the last four honestly: "These four currently expose no routes beyond
/healthz — they are scaffolds whose bands have not landed."

---

## 4. Verification of the five specific claims

### (a) `notification_service/consumer.py:30-32` commits after a no-op — **CONFIRMED, with a caveat that changes its severity**

`consumer.py:21-34`:

```python
async def run_consumer() -> None:
    consumer = AIOKafkaConsumer(
        *_TOPICS,
        bootstrap_servers=_BROKERS,
        group_id=_GROUP,
        enable_auto_commit=False,
    )
    await consumer.start()
    try:
        async for msg in consumer:
            await _handle(msg)
            await consumer.commit()
```

and `consumer.py:37-41`:

```python
async def _handle(msg: object) -> None:
    # TODO(P1): ...
    _ = msg  # placeholder until the consumer is implemented
```

So: `enable_auto_commit=False` is set correctly, and then the loop manually commits after a
function that discards the message. Group `notification-service` (line 18) on topics
`notifications` and `agent-results` (line 17). If this ran against a populated topic it would
walk the offset to the head and every notification in the retention window would be
unrecoverable through this group.

**Caveat — it has no runner and there is no backlog to destroy.**

1. **No compose entry, no runner.** `compose.prod.yml:349-354` runs only the uvicorn app for
   `notification-service`. `grep -rn "consumer" mytheus-platform/` returns no entrypoint;
   the only hit is that uvicorn command line. Nothing invokes `run_consumer` outside the
   `if __name__ == "__main__"` at `consumer.py:44-45`.
2. **Nothing produces to `notifications` or `agent-results` today.** Grepping both topic names
   across `mytheus-backend`, `mytheus-intelligence`, `mytheus-realtime` returns only this
   consumer's own constants and a docstring at
   `mytheus-intelligence/services/agent_runtime_service/main.py:4` saying the
   `agent-results` flow "land[s] in the agent band" (i.e. not yet).

So the destruction is **latent, not active**. It is a loaded gun with the safety on: the day
someone adds a compose entry for the worker — the natural, obvious next step, and one a
deploy-topology task would do without reading the body of `_handle` — it becomes real and
silent. That is the worst possible failure shape for this code, and it costs one line to
defuse.

### (b) `integration-service` is the sole producer of `integration-events`, and the ingestion consumer can never fire — **CONFIRMED**

Sole designated producer, in three places:
- `mytheus-events/CLAUDE.md:67` — `| integration-events | integration-service | ingestion-service, core-api | org_id |`
- `mytheus-events/README.md:42` — same row
- `mytheus-events/schemas/integration-events/v1.avsc:5` — `"Producer integration-service"`

Producer code: **none anywhere.** Grepping `integration-events` across all eight repos returns
only the schema/registry files, the docs, and the intelligence-side consumer + its tests. No
`.py` in `mytheus-backend` mentions the topic at all.

Consumer side is genuinely complete:
- `mytheus-intelligence/services/ingestion_service/consumer.py:75` —
  `topics = ["integration-events"]`, group `ingestion-service.attachment-to-ingest` (line 74)
- `consumer.py:82-120` — full `apply()`: R8 provider/event_type filter (line 87), payload
  contract validation including required `meeting_id` (line 98), cross-tenant `object_key`
  prefix check that fails closed without fetching (`_resolve_key`, lines 64-70, and the
  handler at 102-118), then object fetch
- `services/ingestion_service/worker.py` — a real entrypoint wiring real Postgres, Redis,
  MinIO, Schema Registry and Redpanda
- `mytheus-intelligence/tests/test_ingestion_consumer.py` — 260 lines against a real Schema
  Registry and real Postgres, including `test_missing_org_id_on_integration_events_goes_to_dlq_never_guessed`
  (line 232) and the wrong-provider R8 case (line 177)

This is the single clearest hole in the system: a fully implemented, integration-tested,
deployable consumer subscribed to a topic that has no producer and never will until
`integration_service` grows past 11 lines. The work is already paid for on the receiving end.

### (c) `workflow_service/worker.py:24-38` is a working worker with `activities=[]` — **CONFIRMED**

`worker.py:35`: `activities=[],  # TODO: register activity functions`. The worker connects
(`worker.py:25`), registers four workflows (29-34), prints and runs (37-38). It is in compose
at line 226 and, given a reachable Temporal (`temporal:` at compose line 137), it starts,
polls `mytheus-backend`, and idles forever. Nothing starts a workflow, so the empty activity
list is not currently even reachable — it would only bite once a caller exists.

### (d) `admin_ops_service` owns `audit_logs`, has zero routes, and nothing writes audit rows via that path — **HALF FALSE**

Zero routes: **confirmed** (`main.py:11`).
Owns the model: **confirmed** (`models.py:25-57`), and `migrations/env.py:22` imports it
explicitly with the comment `(audit_logs)` so alembic sees the table.

**"Nothing anywhere writes audit rows" is wrong.** `libs/shared/audit.py` is the shared writer
("The one way every service writes to `audit_logs`", `audit.py:1`; the INSERT at `audit.py:32`;
fail-closed-by-design, `audit.py:6-12`). It has **eight live call sites**:

- `libs/shared/authz.py:176` — every authz denial (`ACTION_AUTHZ_DENY`)
- `services/identity_tenant_service/reconciliation.py:106`
- `services/identity_tenant_service/routes/users.py:109`
- `services/identity_tenant_service/routes/auth.py:149`
- `services/identity_tenant_service/routes/tenant_settings.py:154`
- `services/core_api/routes/org_hierarchy.py:258` and `:302`
- `services/core_api/routes/meetings.py:228` and `:437`

plus `tests/test_audit.py` (175 lines) covering append-only enforcement, RLS, and partition
routing.

So the audit **write** path is live and exercised. What `admin_ops_service` is missing is the
**read** side: there is no way for a tenant admin or an internal operator to see any of it.
The trail is being written into a room with no door. That is a materially different (and much
smaller) hole than "nothing writes audit rows".

### (e) `organizations.plan_id` is an FK to a non-existent `plans` table — **FALSE as stated; true in substance**

`services/identity_tenant_service/models.py:53`:

```python
plan_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), nullable=True)
```

No `ForeignKey(...)`. The migration agrees —
`migrations/versions/82d44163c933_0001_core_tables_outbox_inbox_.py:75`:
`sa.Column('plan_id', sa.UUID(), nullable=True),`. No FK constraint is emitted.

It is therefore **not an FK**; it is an unconstrained nullable uuid with no writer and no
reader (grep for `plan_id` across `mytheus-backend/*.py` returns exactly these two lines).
`mytheus-docs/plan/plan-1-2-foundation-and-tenancy.md:241` says so on purpose:
`plan_id       uuid,                   -- FK added when plans lands (later band)`.

The documents describe it as an FK (`_canon/02-data-model.md:74`,
`19-api-architecture.md:148`, `04-service-catalog.md:169`), and
`docs/plan/plan-7-9-agents-and-monetization.md:104` writes
`plan_id uuid NOT NULL REFERENCES plans(id)` for `subscriptions`. So the doc/code divergence
is real, but the deployed database has no dangling constraint and nothing breaks. The
practical consequence is only that `org.plan` cannot be answered —
`mytheus-ui/src/lib/api/org-context.ts:10` already hardcodes `"starter"` and says why.
This is a **documentation defect, not a schema defect**.

---

## 5. Cross-repo grep: does ANY of this exist anywhere?

Searched `mytheus-backend`, `mytheus-intelligence`, `mytheus-realtime`, `mytheus-ui`,
`mytheus-mobile`, `mytheus-events`, `mytheus-platform`, `mytheus-docs` for `*.py`, `*.ts`,
`*.tsx`, `*.swift`, `*.kt`.

**Billing / metering / quota: NO implementation.** Every hit is a string constant, a route-map
entry, or a doc. Specifically: `QUOTA_EXCEEDED` as an enum member in
`libs/shared/errors.py:52` and echoed into four generated TS clients and
`mytheus-mobile/src/api/client.ts:33`; `/stripe/webhook` in the route map and in
`libs/shared/public_routes.py:31`; and the Avro contract
`mytheus-events/clients/python/mytheus_events/billing_events.py:12`
(`quota_exceeded | dunning | plan_changed`) with its TS twin. No Stripe SDK anywhere — not in
`pyproject.toml`, not in any `package.json`. No `usage_events` writer. No budget check on any
request path. The error code exists; nothing can ever raise it.

**OAuth / connectors: NO implementation.** Every hit is a path-allowlist string
(`public_routes.py:31`, `edge.py:54`, `scripts/check_authz.py:25`,
`mytheus-platform/scripts/check_authz.py:25`), the Avro `IntegrationEvent` provider enum, or
UI mock data (`mytheus-ui/src/lib/mock/data.ts`, `settings/integrations/page.tsx`,
marketing copy in `lib/marketing/features.ts` and `components/marketing/feature-visuals.tsx`).
`mytheus-mobile/app/sign-in.tsx` matched only on Clerk's OAuth, which is unrelated. No Google
API client, no Slack SDK, no Jira/Linear client, no token table, no envelope encryption.

**Notification delivery: NO implementation.** No SendGrid, SES, Resend, Postmark, SMTP, APNs
or FCM client in any repo. Hits are: the `notification_service` scaffold itself, the Avro
`Notification` / `NotificationDeliveryEvent` contracts in `mytheus-events`, an in-app
notifications toggle in `mytheus-ui/src/app/(app)/settings/page.tsx`, a
`mytheus-mobile/app/profile.tsx` preference row, and an icon. Not one line that sends
anything to a human.

The event contracts are the pattern here: `mytheus-events` has fully generated Python and
TypeScript clients and registered Avro subjects for `notifications`,
`notification-delivery-events`, `usage-events`, `billing-events` and `integration-events`
(`codegen/_schemas.py:34-35`), and `scripts/create_topics.sh:67` creates the topics with DLQs.
The pipes are laid and registered. Nothing is connected at either end except the one
intelligence consumer in (b).

---

## 6. Minimum viable implementation and recommendation, per service

"Minimum viable" here means the smallest thing that stops the service being a hole for its
*dependents* — not the band's full spec.

### `integration_service` — **(a) implement. Size M. Highest value of the five.**

MVP: Google Calendar only, one provider, one event. OAuth authorization-code flow at
`GET /oauth/google_calendar` + `GET /oauth/google_calendar/callback`; a
`integration_connections` table holding the refresh token (encrypted — do not skip this, it
is the one part that is painful to retrofit); a poll or push handler that, on a calendar event
with an attachment, writes the blob to `nexus-attachments` under `{org_id}/` and inserts an
outbox row for `integration-events` with `provider=google_calendar`,
`event_type=attachment_to_ingest`, `payload_json` carrying `meeting_id`, `object_key`,
`filename`. Skip Slack, skip Jira/Linear, skip the webhook surface.

Reasoning: this is the only one of the five where a *complete, tested, deployed* dependent is
already waiting (`mytheus-intelligence/services/ingestion_service/`, §4b). The producer side is
the last mile of work that is otherwise finished. It is also the only one where the receiving
contract is fully pinned — the Avro schema, the R8 filter, and the cross-tenant `object_key`
rule are all already written and tested, so the MVP has no design latitude to get wrong. M
rather than S only because OAuth token storage and encryption is genuinely fiddly.

### `notification_service` — **(a) implement, but the urgent part is S and takes one line.**

Two separable pieces:

1. **Immediately: make the consumer safe.** Either delete the `await consumer.commit()` at
   `consumer.py:32`, or raise `NotImplementedError` in `_handle`. Size **S — minutes.** Do
   this regardless of what is decided about the rest. A no-op handler that commits is a trap
   for whoever next adds the worker to compose, and they will not read `_handle` before doing
   it. The current state is safe only by the accident of having no runner and no producer;
   both of those accidents are one commit away from ending.
2. **Then, MVP: in-app only.** A `notifications` table, `GET /notifications` and
   `POST /notifications/:id/dismiss`, and a real `_handle` that does the header/org
   re-assertion, inbox dedupe, and an INSERT for `payload.channel == "in_app"`, ignoring
   email and Slack. Size **M.** No email provider, no Slack fan-out — those are the parts that
   need vendor accounts and deliverability work, and no producer exists to exercise them yet.

Reasoning: the UI has a notifications surface and `action_followup.py:18` already has a
`# TODO: call notification activity` where the reminders go, so this is on the critical path
for workflow_service too. But the ordering matters: fix the commit today, build the rest when
a producer exists.

### `admin_ops_service` — **(a) implement, S.**

MVP: one route. `GET /admin/tenant/audit-logs?limit&cursor`, RLS-scoped, ordered by
`occurred_at DESC`. The model is done, the table is migrated and partitioned, the writers are
live and populating it right now, and `libs/shared/authz.py:176` is writing authz denials into
it as we speak.

Reasoning: this is the cheapest real win available. Rows are accumulating today with no way to
read them, and the whole justification for standing `audit_logs` up in P2 rather than P13
(`models.py:3-5`: "An audit table that arrives after the thing it audits has a hole in it
exactly where the first incident is") is defeated if nobody can query it during an incident.
One read route on an existing, tested table. Leave `/admin/ops/*` (the internal-ops console)
for later.

### `workflow_service` — **(c) leave as-is.**

It is not a hole. It is a correctly-constructed worker whose workflows have no bodies and
whose callers do not exist. `activities=[]` is honest: there are no activities to register
because no workflow calls one. Nothing 404s (it exposes no HTTP surface and has no gateway
route), nothing silently loses data, and no dependent is blocked on it.

The one thing worth doing is small and not urgent: `action_followup.py` is the workflow that
will land first, and it needs a notification activity that does not exist yet, so it is
gated behind notification_service anyway. Revisit when that MVP lands.

### `billing_metering_service` — **(b) narrow the gateway map, and (c) otherwise leave.**

Do not implement. There is no billing band, no Stripe account wired, no `plans` table, and the
UI already hardcodes `"starter"` and knows it does. An MVP here is a large piece of work (L)
with no dependent waiting on it.

But **remove `/stripe/webhook` from `ROUTE_MAP` (`edge.py:174`) and from `PUBLIC_PREFIXES`
(`edge.py:52`)**. That specific pair is worse than a hole: it is a public, unauthenticated,
advertised endpoint that returns 404 to a real external sender. If a Stripe account is ever
pointed at `api.mytheus.cloud/stripe/webhook` — during a sales demo, a test-mode experiment,
anything — Stripe retries the 404 on a backoff and then automatically disables the endpoint,
and the person debugging it will reasonably assume the gateway is misrouting. The other three
(`/billing`, `/usage`, `/plans`) are authenticated and only ever hit by our own UI, so their
404 is harmless and self-explanatory; leave them. Note that `tests/test_gateway_edge.py:270`
and `tests/test_gateway_proxy.py:175` assert the billing routes, so a map change touches the
tests.

---

## 7. On the framing: is a deployed empty service worse than no service?

Not in general, and the compose comment at lines 318-321 gives the correct reason — a 404 from
`billing-metering-service` distinguishes "the band has not landed" from "the gateway is
misrouting", and that distinction is worth four idle containers.

It is worse in exactly two cases, both of which are present here, and both of which are narrow
enough to fix without touching the topology:

1. **When the 404 is served to a party that will act on it.** `/stripe/webhook` and `/oauth/`
   are public (`edge.py:52-55`) and are reached by external senders who retry, disable
   endpoints, or show the user a broken redirect. An authenticated 404 our own UI receives is
   information; an unauthenticated 404 Stripe receives is a malfunction.
2. **When the scaffold contains a live-fire mistake waiting for a runner.**
   `notification_service/consumer.py:32` is the case. The container is harmless; the file
   inside it is not, and deploying the service makes adding the worker feel routine.

So: keep the topology, fix those two things, and implement the three services (integration,
admin-ops read, notification in-app) where a dependent is already built and waiting.

## 8. Priority order

1. `notification_service/consumer.py:32` — delete the commit or raise. **S, minutes, do first.**
2. `/stripe/webhook` out of `ROUTE_MAP` and `PUBLIC_PREFIXES`. **S.**
3. `admin_ops_service` — `GET /admin/tenant/audit-logs`. **S.**
4. `integration_service` — Google Calendar OAuth + attachment producer. **M, highest value.**
5. `notification_service` — in-app notifications table + REST + real `_handle`. **M.**
6. `workflow_service`, `billing_metering_service` — leave.
7. Correct the docs that call `organizations.plan_id` an FK
   (`_canon/02-data-model.md:74`, `19-api-architecture.md:148`, `04-service-catalog.md:169`);
   it is an unconstrained orphan column by design, per
   `plan/plan-1-2-foundation-and-tenancy.md:241`.
