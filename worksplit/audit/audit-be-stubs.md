# Backend stub-service audit — five apparently-thin services

Read-only audit. No file in any repo was modified. Every claim below is cited to source
(`path/file.py:LINE`); `.md` files are used only as evidence of INTENT, never of STATUS.

Root: `/Users/sushan/Projects/Mytheus`

**Headline:** all five services are pure scaffolds. Two of them are load-bearing — something
real elsewhere in the codebase depends on them and is therefore broken. All five are deployed
to production.

---

## admin-ops-service — 68 LOC

Path: `mytheus-backend/services/admin_ops_service/`

### INTENDED
Sources: `mytheus-docs/_canon/01-service-catalog.md:124`, `mytheus-backend/CLAUDE.md:40`,
`mytheus-backend/system.md:84,150`, `mytheus-backend/README.md:26`.

- Two consoles: **tenant admin** (`/admin/tenant/*` — org-scoped self-serve: members, plan,
  integrations, data export/erasure requests) and **internal ops** (`/admin/ops/*` —
  cross-tenant support, feature flags, tenant suspend/migrate, consumer-lag and cost
  dashboards, audit-log access). RBAC-gated and fully audited.
- Owns feature flags, the ops audit log, `audit_logs` (append-only), impersonation grants,
  and the `nexus-audit-archive` bucket.
- Produces `admin-events` (flag changed, tenant suspended, erasure requested); consumes
  `tenant-events`, `billing-events`, `graph-events`.

### ACTUAL
`services/admin_ops_service/main.py` is 11 lines. The entire application is:

```python
# main.py:9
app = create_app("admin-ops-service")

# main.py:11
# TODO: add routers for /admin/tenant/* and /admin/ops/*
```

Zero routes beyond whatever `create_app` mounts (`/healthz`).

The one genuinely-built asset is `models.py:25-57`, the `AuditLog` SQLAlchemy model — and it
is real work, not a stub. It is `RANGE`-partitioned on `occurred_at`, which is why
`occurred_at` is part of the primary key (`models.py:48-52`), and it carries a
`actor_type IN ('user','agent','system')` check constraint (`models.py:55`).

The matching migration is also real:
`migrations/versions/c3e8b1d47f92_0004_audit_logs_append_only.py`.

- `:62` creates the table; `:83-92` creates monthly partitions plus `audit_logs_default`.
- `:128-129` `ENABLE` then `FORCE ROW LEVEL SECURITY`; `:131` creates the `tenant_isolation`
  policy.
- `:103-108` documents and fixes a genuine subtlety: a partition named directly is governed
  by its *own* ACL, so `DELETE FROM audit_logs_2026_07` succeeded for `mytheus_app` even
  though `DELETE FROM audit_logs` was refused. The per-partition revoke at `:138` closes it.
- `migrations/env.py:22` imports the model so autogenerate sees it.

So the **table** is production-grade. The **service** around it does not exist.

### DEPENDENTS
- `services/api_gateway/edge.py:176-177` routes `/admin/tenant` and `/admin/ops` here.
- Asserted by tests as correct behavior: `tests/test_gateway_edge.py:270`
  (`("/admin/ops/flags", "admin-ops-service")`) and `tests/test_gateway_proxy.py:176`.
- `mytheus-ui/system.md:31` names it as the backend for both admin consoles.
- `scripts/emit_openapi.py:24` states in source that the hand-written OpenAPI set covers
  "`admin-ops-service`, which has no routes at all" — the repo's own tooling knows.

### DEPLOYED?
Yes — `mytheus-platform/deploy/compose.prod.yml:342`.

### GAP
No routes. More seriously: **`audit_logs` has no writer.** Nothing anywhere in any repo
constructs an `AuditLog` or inserts into that table. `models.py:3-5` argues the table was
stood up in P2 rather than P13 specifically so that "RLS and authz events are recorded from
the moment RLS goes live... An audit table that arrives after the thing it audits has a hole
in it exactly where the first incident is." The table arrived; the recording did not. The
hole is present anyway.

---

## billing-metering-service — 11 LOC

Path: `mytheus-backend/services/billing_metering_service/`

### INTENDED
Sources: `mytheus-docs/_canon/01-service-catalog.md:122`,
`mytheus-docs/adr/0011-billing-and-metering.md`, `mytheus-backend/CLAUDE.md:39`,
`mytheus-backend/system.md:76,149,236-240`.

- Stripe as the system of record for subscriptions, seats, dunning, tax and invoices.
- A `usage-events → aggregation → quota enforcement` metering pipeline: bills metered
  overage and enforces per-tenant hard quotas. ADR-0011 frames this as the defense against
  the **R4 LLM cost spiral** — LLM is stated there as 94% of variable cost, so an unmetered
  tenant is a direct margin leak.
- Owns `plans` (global, no `org_id`), `subscriptions`, `usage_events`, `usage_aggregates`,
  `quotas`, `invoices`, `agent_budgets`, and the `nexus-invoices` R2 bucket.
- Serves `/billing/*`, `/usage`, `/plans`, `POST /stripe/webhook`; produces `billing-events`
  (quota-exceeded, dunning, plan-changed) and `notifications`; consumes `usage-events`,
  `tenant-events`.
- Budget state machine per the stub's own docstring (`main.py:4`): `ok -> throttled ->
  blocked`.

### ACTUAL
The complete file:

```python
# main.py:9
app = create_app("billing-metering-service")

# main.py:11
# TODO: add routers for /billing /usage /plans /stripe/webhook
```

Nothing else. No models module at all (unlike admin-ops).

### DEPENDENTS
- Gateway routes four prefixes here: `services/api_gateway/edge.py:171-174` —
  `/billing`, `/usage`, `/plans`, `/stripe/webhook`.
- `/stripe/webhook` is on the **unauthenticated** public allowlist:
  `services/api_gateway/edge.py:52` and `libs/shared/public_routes.py:31`
  (`PUBLIC_CONTAINS = ("/webhook", "/oauth", "/clerk/", "/stripe/", "/livekit/", "/callback")`).
  Mirrored in `mytheus-platform/scripts/check_authz.py:25`.
- `services/identity_tenant_service/provisioning.py:91` documents that
  `organizations.plan_id` is "a UUID FK to `billing-metering-service`'s `plans` table" — a
  table that **does not exist in any migration**. Provisioning therefore carries a plan id
  pointing at nothing.
- Test-asserted routing: `tests/test_gateway_edge.py:270`, `tests/test_gateway_proxy.py:175`.
- `Makefile:81` has a dev target that runs it on port 8005.

### DEPLOYED?
Yes — `mytheus-platform/deploy/compose.prod.yml:329`.

### GAP
Total. See CROSS-CUTTING §1 for the full negative-result grep.

---

## integration-service — 11 LOC

Path: `mytheus-backend/services/integration_service/`

### INTENDED
Sources: `mytheus-docs/_canon/01-service-catalog.md:120`, `mytheus-backend/CLAUDE.md:38`,
`mytheus-backend/system.md:68,148,204`, `mytheus-backend/README.md:24`.

- Google Calendar OAuth: auto-create meetings, match participants by email, route
  attachments to `ingestion-service`.
- Slack bot OAuth: Block Kit prep briefs and summaries.
- Jira (default) / Linear (alt) OAuth: bidirectional action-item sync plus status webhooks.
- **Guardrail R8**: Calendar + Slack + exactly one PM tool per workspace.
- Owns `integration_connections`, `oauth_tokens` (envelope-encrypted at rest, never returned
  by API), external↔NEXUS id maps (`integration_links`), webhook cursors.
- Produces `integration-events` (event→meeting, attachment-to-ingest, external status →
  action update) and `actions` status sync. Serves `/oauth/{provider}`,
  `/webhooks/{provider}`.

### ACTUAL
The complete file:

```python
# main.py:9
app = create_app("integration-service")

# main.py:11
# TODO: add routers for /oauth/{provider} and /webhooks/{provider}
```

### DEPENDENTS — HARD BREAK
`mytheus-intelligence/services/ingestion_service/consumer.py:75` subscribes:

```python
topics = ["integration-events"]
```

and filters at `:56` for `_CALENDAR_ATTACHMENT = ("google_calendar", "attachment_to_ingest")`.
Its own docstring (`:10-14`) explains it expects that topic to also carry Slack and
Jira/Linear traffic for other consumers.

`mytheus-backend/system.md:204` names `integration-service` as the **sole producer** of
`integration-events`. It produces nothing. A finished, tested consumer is bound to a topic
that will never receive a production message.

This will not be caught by CI: the consumer's test suite manufactures its own messages —
`mytheus-intelligence/tests/test_ingestion_consumer.py:112` calls
`consumer._codec.serialize_for("integration-events", value)` and feeds it in directly
(`:116`, `:180`, `:218`). The tests pass; the pipeline is dead end to end.

Also:
- Gateway routes `/oauth` and `/webhooks` here (`services/api_gateway/edge.py:178-179`),
  both on the public/unauthenticated allowlist (`edge.py:54`).
- `mytheus-ui/system.md:30` names it for the Settings→Integrations screen.
- `mytheus-backend/CLAUDE.md:142` instructs that workflow activities "drive CRUD over REST
  against `core-api`/`integration-service`" — an instruction that cannot currently be
  followed.

### DEPLOYED?
Yes — `mytheus-platform/deploy/compose.prod.yml:322`.

### GAP
Total. No OAuth flow, no token storage, no envelope encryption, no provider client, no
webhook handler, no `integration_connections`/`oauth_tokens` migration.

---

## notification-service — 56 LOC

Path: `mytheus-backend/services/notification_service/`

### INTENDED
Sources: `mytheus-docs/_canon/01-service-catalog.md:123`, `mytheus-backend/CLAUDE.md:41`,
`mytheus-backend/system.md:92,151,207-208`.

- Unified delivery fabric: in-app (via `realtime-gateway`), email, and Slack DM/channel.
  Templating, per-user priority queue (urgent alerts pre-empt notes), dedupe, delivery
  receipts.
- **Sole consumer authorized** to fan `notifications` out to external channels.
- Owns `notification_log`, delivery receipts, user channel prefs, and the Redis priority
  queue `nexus:{org_id}:queue:agent:{user_id}`.
- Consumes `notifications` and `agent-results`; produces `notification-delivery-events`.
- Serves `GET /notifications`, `PATCH /notifications/:id` (the durable read/dismiss side).

### ACTUAL
`main.py` is 11 lines — `main.py:9` `app = create_app("notification-service")` and
`main.py:11` `# TODO: add router for /notifications (read/dismiss REST surface)`. Zero routes.

`consumer.py` (45 lines) is the only substance, and it is half-real. The transport layer
genuinely works — `consumer.py:21-34` constructs an `AIOKafkaConsumer` over
`_TOPICS = ["notifications", "agent-results"]` (`:17`) with `enable_auto_commit=False`
(`:26`) and a commit-after-handle loop:

```python
# consumer.py:30-32
async for msg in consumer:
    await _handle(msg)
    await consumer.commit()
```

But the handler is a placeholder:

```python
# consumer.py:37-41
async def _handle(msg: object) -> None:
    # TODO(P1): re-assert org_id from msg.headers -> SET app.current_org ->
    #   is_duplicate/record_consumed (inbox + Redis dedupe) ->
    #   route json.loads(msg.value) to email / Slack / in-app based on payload.channel.
    _ = msg  # placeholder until the consumer is implemented
```

**This is worse than an absent service.** The loop consumes a message, does nothing with it,
then *commits the offset*. Every notification it sees is permanently discarded rather than
left on the topic to be processed once the handler lands. An empty service that drops data
is a different class of problem from one that merely 404s.

Mitigating factor, and it is the only thing preventing live data loss today: `consumer.py`
has **no compose service anywhere**. `compose.prod.yml:349` deploys only the routeless
uvicorn API (`services.notification_service.main:app`). The consumer is not running in
production. It is a loaded gun, not a fired one.

No `notification_log` table exists in any migration.

### DEPENDENTS — planned hard break (mobile)
The entire mytheus-mobile push tier is specified against this service:
- `mytheus-mobile/system.md:55` — register an Expo push token with `notification-service`.
- `mytheus-mobile/system.md:169,199-200` — delivery path
  `notifications` topic → `notification-service` → Expo Push → APNs/FCM.
- `mytheus-mobile/system.md:172,300` (N7) — flags that Expo device-token registration is a
  *net-new* backend contract this service must expose, "the one place the mobile client
  implies a small backend addition."
- `mytheus-mobile/CLAUDE.md:21-22,122`, `README.md:19,44` repeat it as binding guidance.

Mobile has no implementing code yet — the only hits in `.ts`/`.tsx` are UI labels
(`mytheus-mobile/app/profile.tsx:33`, `src/components/icons.tsx:268`). So this is a break
that lands the moment mobile push work starts, not one crashing today.

Also: gateway routes `/notifications` here (`services/api_gateway/edge.py:175`).

### DEPLOYED?
Yes — `mytheus-platform/deploy/compose.prod.yml:349` (API only; consumer not deployed).

### GAP
Total on delivery. Plus the offset-commit data-loss hazard above.

---

## workflow-service — 110 LOC

Path: `mytheus-backend/services/workflow_service/`

### INTENDED
Sources: `mytheus-docs/_canon/01-service-catalog.md:121`, `mytheus-backend/CLAUDE.md:42`,
`mytheus-backend/system.md:100-106,248`.

- Temporal durable orchestration, stateless workers scaling on task-queue depth.
- Four workflows: `MeetingLifecycleWorkflow`, `ActionFollowUpWorkflow` (reminders at day
  1/3/7, manager escalation via Slack DM at day 10), the right-to-be-forgotten deletion
  saga, and the billing dunning saga.
- Owns no durable state; activities drive CRUD over REST against `core-api` /
  `integration-service` rather than cross-writing their tables. Internal `/workflows/*`
  admin surface, not public.

### ACTUAL
This is the most honest of the five — every file says plainly what it is — and it is still
empty of behavior.

`worker.py:24-38` is a **real, working** Temporal worker: it connects
(`worker.py:25`), and registers all four workflow classes (`:29-34`). But:

```python
# worker.py:35
activities=[],  # TODO: register activity functions
```

A worker with zero activities can host workflows that do nothing but sleep.

The four workflow bodies:

| File | Body |
| --- | --- |
| `workflows/meeting_lifecycle.py:9-11` | `pass` (TODO: room open/close, post-meeting processing, cleanup) |
| `workflows/dunning.py:12-14` | `pass` (TODO: staged retry / downgrade / suspend) |
| `workflows/rtbf.py:14-22` | `pass`, under a six-step TODO: Postgres purge, Qdrant delete-by-org filter, AGE node/edge removal, Redis `nexus:{org_id}:*` flush, object-storage prefix delete, audit completion record |
| `workflows/action_followup.py:16-21` | the only executable logic in the service |

`action_followup.py` is a no-op machine:

```python
# action_followup.py:16-21
for days in (1, 3, 7):
    await workflow.sleep(timedelta(days=days))
    # TODO: call notification activity to send reminder

await workflow.sleep(timedelta(days=3))  # day 10 total
# TODO: call Slack DM escalation activity to manager
```

Four `sleep` calls totalling 14 days, sending nothing at any of them.

### DEPENDENTS
- `Makefile:84` and `CLAUDE.md:144,168` — dev/run entry points.
- `libs/shared/db.py:69` references `services/workflow_service/workflows/rtbf.py` as the
  consumer of a privileged DB role that is "deliberately NOT in the web process env." A
  credential is being scoped for a workflow that does nothing.
- `mytheus-backend/PREVIOUS.md:124` proposes wiring the outbox relay into `workflow_service`
  (superseded — the relay is its own service at `compose.prod.yml:210`).

### DEPLOYED?
**Yes, and it is the one of the five that actually runs.**
`mytheus-platform/deploy/compose.prod.yml:226`:

```yaml
workflow:
  <<: *backend-app
  command: ["python", "-m", "services.workflow_service.worker"]
```

A worker process polling a Temporal task queue on which it can perform no work.

### GAP
Worker harness real; all four workflow bodies empty; zero activities registered.

---

# CROSS-CUTTING FINDINGS

## 1. BILLING — is there ANY metering/usage-recording code anywhere?

**No. None, in any of the seven repos.**

Grepped all `.py` across `mytheus-backend`, `mytheus-intelligence`, `mytheus-realtime`,
`mytheus-events`, `mytheus-platform` for
`stripe|usage_event|usage_aggregate|quota|agent_budget|subscription|meter`. Every hit is one
of:

- Route-map strings — `services/api_gateway/edge.py:171-174`.
- Path allowlists — `libs/shared/public_routes.py:31`,
  `mytheus-platform/scripts/check_authz.py:25`, `services/api_gateway/edge.py:52`.
- An error-code enum with no raiser — `libs/shared/errors.py:52`
  (`QUOTA_EXCEEDED = "QUOTA_EXCEEDED"`), and its contract test
  `tests/test_openapi_contract.py:49`.
- Gateway routing tests — `tests/test_gateway_edge.py:270`,
  `tests/test_gateway_proxy.py:175`.
- The stub's own docstring — `services/billing_metering_service/main.py:1-4`.
- Passing mentions in unrelated comments — `libs/shared/openapi.py:156`,
  `services/workflow_service/workflows/dunning.py:1`.
- Generated event schemas with no producer or consumer —
  `mytheus-events/clients/python/mytheus_events/billing_events.py:12-13`
  (`event_type: Literal['quota_exceeded', 'dunning', 'plan_changed']`),
  `mytheus-events/clients/python/mytheus_events/__init__.py:18` (`UsageEvent`).
- One false positive in an unrelated test — `mytheus-intelligence/tests/test_embedding_consumer.py:183`
  ("trigger subscription/assignment", Kafka partition assignment).

`stripe` appears in `pyproject.toml` and `uv.lock` as an **installed but entirely unused**
dependency.

**No migration creates a single billing table.** The complete set of tables created across
`migrations/versions/`: `data_residency`, `departments`, `inbox`, `meeting_participants`,
`meetings`, `organizations`, `outbox`, `team_members`, `teams`, `tenant_settings`, `users` —
plus `audit_logs` via raw SQL. Absent: `plans`, `subscriptions`, `usage_events`,
`usage_aggregates`, `quotas`, `invoices`, `agent_budgets`.

### What does the UI `/billing` page talk to?

**Nothing.** `mytheus-ui/src/app/(app)/billing/page.tsx:2`:

```tsx
import { mockBilling } from "@/lib/mock/data";
```

`:22-23` destructures `plan, seatsUsed, seatsTotal, pricePerSeat, billingCycle,
nextInvoiceDate, nextInvoiceAmount, invoices` straight out of that fixture and renders them.
There is no `fetch`, no server action, no bff call anywhere in the file. The seat-usage
progress bar at `:26` is computed from mock numbers.

**Doc-vs-source conflict worth escalating:** `mytheus-ui/docs/PAGES.md:52` records
`| /billing | RSC | billing-metering-service via bff | done |`. All three claims are false —
the page is a client-data mock, it talks to no service, and it is not done. This is the
clearest instance in the audit of a status doc asserting something the source contradicts.

## 2. INTEGRATIONS — is there ANY OAuth/connector code anywhere?

**No.**

Grepped all `.py` for `oauth|slack_|notion|google_calendar|jira|linear|integration_connection`.
Every backend hit is an allowlist string or a comment:
`libs/shared/public_routes.py:24,31`, `scripts/check_authz.py:11`,
`services/api_gateway/edge.py:54,178`, `scripts/emit_openapi.py:51`, and the stub's own
docstring (`services/integration_service/main.py:1-3,11`).

The only places provider names appear as **live values** are downstream of a producer that
does not exist: `mytheus-intelligence/services/ingestion_service/consumer.py:10,14,56` and
its tests (`test_ingestion_consumer.py:108,163-165,206,250`), plus the schema
`mytheus-events/clients/python/mytheus_events/integration_events.py:12`
(`provider: Literal['google_calendar', 'slack', 'jira', 'linear']`).

No token store, no envelope encryption, no provider SDK, no webhook signature verification,
no `integration_connections` / `oauth_tokens` migration.

### The UI `/settings/integrations` page

It admits this in its own source comment —
`mytheus-ui/src/app/(app)/settings/integrations/page.tsx:10-13`:

```tsx
/* Integration connections — the deliberately small surface (guardrail R8):
   Google Calendar + Slack, plus ONE project tracker (Jira or Linear) per
   workspace. Connect/disconnect toggles are visual only until the
   integration-service wiring lands. */
```

`:16` seeds state from `mockIntegrations`; `toggle()` at `:22-30` only mutates React state
(including the one-tracker-per-workspace rule at `:26-27`, enforced client-side only).
Nothing persists; nothing is sent anywhere.

The 12 marketing pages promise capability with zero backing code —
`mytheus-ui/src/lib/marketing/features.ts:293` carries the comment
`// integration-service (R8 guardrail)`, and
`mytheus-ui/src/app/(marketing)/features/integrations/page.tsx` exists as a public-facing
feature page.

## 3. NOTIFICATIONS — any email/push/in-app delivery anywhere?

**None, in any repo.** Grepped `.py` across backend, intelligence and realtime for
`notification_log|smtp|sendgrid|resend|expo|push_token|send_email|deliver`. Every hit is
either a substring false positive (`redelivered`, `undelivered`, `deliverable` in test
names — `tests/test_relay.py:20-34,77-80`, `tests/test_meeting_events_consumer.py:225,270`,
`tests/test_clerk_webhook.py:185`) or unrelated prose in migration comments.

No SMTP client, no email provider SDK, no Expo/APNs/FCM code, no Slack send, no in-app
notification write. The `notifications` topic has exactly one subscriber — a handler that is
a documented placeholder (`services/notification_service/consumer.py:37-41`) — and **no
producer at all**.

## 4. WORKFLOW — what does the 110 LOC actually do?

It starts a Temporal worker and idles.

- ~15 lines: `worker.py` connection and registration boilerplate that genuinely works.
- ~4 lines: `activities=[]` plus imports.
- Four workflow classes whose `run()` bodies are `pass`, `pass`, `pass`, and a loop of four
  `workflow.sleep()` calls followed by nothing.
- The remainder is docstrings and TODO comments — accurate, detailed ones. `rtbf.py:15-21`
  in particular is a genuinely useful six-store deletion checklist for whoever implements it.

**Total executable statements across all four workflows: four `sleep` calls**
(`action_followup.py:17,20`).

## 5. Topic-graph reality check

Producers/consumers actually present in source, versus the taxonomy in
`mytheus-backend/system.md:201-212`:

| Topic | Designated producer | Producer in source? | Consumer in source? |
| --- | --- | --- | --- |
| `integration-events` | integration-service | **No** | **Yes** — `mytheus-intelligence/services/ingestion_service/consumer.py:75` |
| `usage-events` | AI tier | No | No (billing-metering is empty) |
| `notifications` | identity/billing/integration/workflow + agents | No | Placeholder only — `notification_service/consumer.py:17` |
| `billing-events` | billing-metering-service | No | No |
| `admin-events` | admin-ops-service | No | No |
| `agent-results` | agent tier | Not verified in backend | Placeholder only — `notification_service/consumer.py:17` |

Only `integration-events` has the dangerous asymmetry: a real consumer with no producer.
The rest are symmetric voids.

---

# VERDICT

## Honest placeholders

Scaffolds that say what they are and that nothing currently depends on:

**workflow-service.** Every file's docstring matches its contents exactly. The worker starts
and idles harmlessly. `action_followup.py` is the only one that could mislead — it looks
like it does something — but its TODOs are inline at each wake point.

**billing-metering-service.** Nothing calls it; the UI is mocked and the mock is imported
under an obvious name. The only false claim about it lives in `mytheus-ui/docs/PAGES.md:52`,
not in code. Note the one real consequence: `organizations.plan_id` is a documented FK
(`identity_tenant_service/provisioning.py:91`) to a nonexistent table.

**admin-ops-service.** Comments accurate, and it shipped genuinely good work in the
`audit_logs` table and its append-only-by-GRANT migration. One real caveat that should not
be lost: **the table has no writer**, so the security-architecture requirement it was built
early to satisfy is unmet regardless.

## Load-bearing but empty

**1. integration-service — the hard break, and the one to fix or formally defer first.**
`mytheus-intelligence`'s `ingestion_service` is a finished, tested consumer
(`consumer.py:75`, filtering `("google_calendar", "attachment_to_ingest")` at `:56`) bound
to `integration-events`. Its sole sanctioned producer is this empty service. The
calendar-attachment ingestion path is dead end to end, in production, right now — and CI
will never notice, because the consumer's tests inject synthetic messages
(`test_ingestion_consumer.py:112`). This is the specific failure mode where green tests and
a green deploy coexist with a dead pipeline.

**2. notification-service — two distinct problems.**
*(a)* The consumer commits Kafka offsets on messages it discards
(`consumer.py:30-32` commits after a no-op `_handle`). If ever started, it destroys the
notification backlog rather than leaving it to be processed later. Currently inert only
because the consumer has no compose service — a deployment change would arm it.
*(b)* The entire mytheus-mobile push architecture is specified against this service across
five documents, including a net-new contract (`mytheus-mobile/system.md:172`, N7 at `:300`)
that the backend must expose and has not.

## Deployed-but-empty

All five are in `mytheus-platform/deploy/compose.prod.yml` (lines 226, 322, 329, 342, 349).
This is **deliberate and honestly documented in the file itself** —
`compose.prod.yml:317-321`:

> These four currently expose no routes beyond /healthz — they are scaffolds whose bands have
> not landed. Deployed anyway so the topology is complete and the gateway's route map has a
> real upstream to reach: a request to /billing gets a 404 FROM billing-metering-service,
> which is the truth, rather than a 404 from the gateway that looks identical to a routing bug.

That is a defensible engineering choice, clearly reasoned. The consequence to be aware of:
the gateway route map (`services/api_gateway/edge.py:171-179`) advertises **nine prefixes** —
`/billing`, `/usage`, `/plans`, `/stripe/webhook`, `/notifications`, `/admin/tenant`,
`/admin/ops`, `/oauth`, `/webhooks` — every one of which resolves to a 404 in production. And
`tests/test_gateway_edge.py:270` and `tests/test_gateway_proxy.py:175-176` assert that
routing as correct behavior. The test suite is green on a route map to nowhere. Four of those
nine (`/stripe/webhook`, `/oauth`, `/webhooks`, and by prefix any `*/webhook`) are on the
unauthenticated public allowlist.

`mytheus-backend/PREVIOUS.md:45` records that a prior pass **deleted** route stubs that were
`raise NotImplementedError` across all of these services, on the principle that "routes
should be added when features are implemented, not pre-declared." The current
docstring-plus-TODO shape is the result of that cleanup, and is the better of the two.

---

# UNVERIFIED

- Whether Temporal task queues in the deployed environment have workflows actually
  *scheduled* against them. Nothing in `mytheus-backend` calls `client.start_workflow`, which
  suggests not, but I did not exhaustively grep `mytheus-intelligence` / `mytheus-realtime`
  for Temporal clients.
- Whether `mytheus-mobile` has an active branch adding notification code. I read only the
  checked-out working tree and did not `git fetch`. Same caveat applies to `mytheus-ui`,
  whose rebuild branch is known to be active.
- Whether the AI tier (`mytheus-intelligence`) emits `usage-events` under a name I did not
  grep for. I searched the literal topic string and found only the schema registry entry
  (`mytheus-events/codegen/_schemas.py:35`).
- Runtime behavior of any of this. The audit is static only; nothing was executed.

---

# KEY FILES

Absolute paths, in the order worth opening:

- `/Users/sushan/Projects/Mytheus/mytheus-intelligence/services/ingestion_service/consumer.py` — the orphaned consumer (break #1)
- `/Users/sushan/Projects/Mytheus/mytheus-backend/services/notification_service/consumer.py` — the offset-committing no-op (break #2)
- `/Users/sushan/Projects/Mytheus/mytheus-backend/services/api_gateway/edge.py` — the route map to nine 404s
- `/Users/sushan/Projects/Mytheus/mytheus-platform/deploy/compose.prod.yml` — all five deployed; reasoning at :317-321
- `/Users/sushan/Projects/Mytheus/mytheus-backend/services/admin_ops_service/models.py` — the real `audit_logs` model with no writer
- `/Users/sushan/Projects/Mytheus/mytheus-backend/migrations/versions/c3e8b1d47f92_0004_audit_logs_append_only.py` — the append-only-by-GRANT migration
- `/Users/sushan/Projects/Mytheus/mytheus-backend/services/workflow_service/workflows/action_followup.py` — the 14-day sleep that sends nothing
- `/Users/sushan/Projects/Mytheus/mytheus-ui/src/app/(app)/billing/page.tsx` — mock-backed
- `/Users/sushan/Projects/Mytheus/mytheus-ui/docs/PAGES.md` line 52 — the false "done" row
- `/Users/sushan/Projects/Mytheus/mytheus-backend/services/identity_tenant_service/provisioning.py` line 91 — FK to a nonexistent `plans` table
