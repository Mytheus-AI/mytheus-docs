# BUILD SPEC — Billing, Metering and Quota

Scope: `billing-metering-service` and everything that feeds or reads it.
Docs cited below are the source of **intent** only. Every status claim in §1 was read from code.

---

## 1. Reality, confirmed

All four claims in the brief are true. Details, with line citations.

### 1.1 The service is a docstring and a TODO

`mytheus-backend/services/billing_metering_service/main.py` is **11 lines**:

```
main.py:1   """billing-metering-service — Stripe billing, usage metering, quota enforcement (R4).
main.py:3   Owns: plans, subscriptions, usage_events, usage_aggregates, quotas, invoices, agent_budgets.
main.py:4   Budget states: ok -> throttled -> blocked. Quota enforcement never sheds durable facts.
main.py:7   from libs.shared.app_factory import create_app
main.py:9   app = create_app("billing-metering-service")
main.py:11  # TODO: add routers for /billing /usage /plans /stripe/webhook
```

`billing_metering_service/__init__.py` is empty. There are no `routes/`, `models.py`,
`consumers/`, or tests. The package contains exactly two `.py` files.

### 1.2 No billing table exists in any migration

`mytheus-backend/migrations/versions/` holds 9 revisions. The tables they create:

- `82d44163c933_0001_core_tables_outbox_inbox_.py:41-146` — `departments`, `inbox`, `meetings`,
  `organizations`, `outbox`, `teams`, `data_residency`, `tenant_settings`, `users`,
  `meeting_participants`, `team_members`
- `b7d9f3a55c21_0003_rbac_grants_and_canonical_columns.py:51` — `rbac_grants`
- `c3e8b1d47f92_0004_audit_logs_append_only.py:62,86,92` — `audit_logs` + partitions
- `3d13f46b960f`, `25e7cede5672`, `866f7eedb0e7`, `f3a7c9e0b21d` — `meeting_rooms`,
  `meeting_speakers`, consent columns, LiveKit metadata
- `5ed978467cad_feat_core_api_add_chat_messages.py:74,90` — `chat_messages` + partitions
- `a1c4e7f20b18_0002`, `9b2c4d1e6f30_0005` — roles/grants only

None of `plans`, `subscriptions`, `usage_events`, `usage_aggregates`, `quotas`, `invoices`,
`agent_budgets` appears in any revision. Confirmed.

### 1.3 `organizations.plan_id` is an unconstrained orphan

Created at `82d44163c933_0001_core_tables_outbox_inbox_.py:71` inside the `organizations`
table as a bare nullable `sa.UUID()` — **no FK, no default, no writer, no reader**. The
docs that describe it as an FK are wrong about code, not about intent:

- `mytheus-docs/_canon/02-data-model.md:74` lists `plan_id` among `organizations` key fields.
- `mytheus-docs/04-service-catalog.md:169` — same.
- `mytheus-docs/plan/plan-1-2-foundation-and-tenancy.md:241` is the one honest line:
  `plan_id uuid, -- FK added when plans lands (later band)`.

Two prior audits in this repo already recorded it: `worksplit/audit/audit-be-data.md:122`
("no FK, no writer, no reader") and `worksplit/verify/verify-scaffolds.md:245`.

### 1.4 The gateway routes four prefixes into nothing

`mytheus-backend/services/api_gateway/edge.py`:

```
edge.py:171   "/billing":        "billing-metering-service",
edge.py:172   "/usage":          "billing-metering-service",
edge.py:173   "/plans":          "billing-metering-service",
edge.py:174   "/stripe/webhook": "billing-metering-service",
```

`/stripe/webhook` is on the **unauthenticated** public allowlist at `edge.py:52`, inside
`PUBLIC_PREFIXES` (`edge.py:48-56`), justified there as "signature-verified inbound webhooks…
authenticate by HMAC over the body". `is_public()` at `edge.py:85` short-circuits token
verification for it. **That HMAC verification does not exist** — there is no handler at all.
Today the path is simply open and unhandled; the security property the allowlist comment
asserts is a promise, not a control. **This is the single highest-priority item in this spec
(Task 20) and it is not a P13 item.**

Precision on "404": `route_for()` (`edge.py:183`) resolves the prefix, then `proxy.py:68-77`
builds `http://billing-metering-service:8000/...` from `UPSTREAM_BILLING_METERING_SERVICE` or
the compose service name. **No billing container exists** — `mytheus-backend/docker-compose.yml`
defines only `postgres`, `redis`, `redpanda`, `temporal`, `temporal-ui` (lines 3-76). So the
call fails at connect (proxy error), and would 404 at FastAPI even if the process ran, since
`create_app` registers no billing routers. Either way: nothing serves it.

### 1.5 What already exists and should be reused (not rebuilt)

This is why several tasks below are S rather than M:

- `libs/shared/inbox.py`, `outbox.py`, `idempotency.py`, `events.py`, `cache.py` — the
  at-least-once machinery is built.
- `libs/shared/tenancy.py:33-35` — `set_config('app.current_org', :org_id, true)`,
  transaction-local, the canonical RLS activation.
- `libs/shared/authz.py:128` — `tenant_session`, the per-request bound session.
- `services/core_api/consumer.py` — the repo's consumer recipe, deliberately split into a
  testable `handle_*` core plus a thin `run_consumer()` harness. Its header (lines 1-36) is
  the standing instruction to re-assert `org_id` from the envelope header before any write.
  **The `usage-events` consumer must follow this shape.**
- `mytheus-backend/pyproject.toml:20` — `stripe>=10` is **already a dependency** (resolved to
  15.3.0 in `uv.lock:2669`). No dependency work is needed for the Stripe band.
- `mytheus-events/schemas/usage-events/v1.avsc` and `billing-events/v1.avsc` exist and are
  registered (`subjects/usage-events-value.json`, `subjects/billing-events-value.json`).

### 1.6 The Avro schemas, as they actually stand

`mytheus-events/schemas/usage-events/v1.avsc`:

```
:9   event_type enum UsageEventType { meeting_minute, agent_token, transcription_second, storage_gb }
:10  quantity   double
:11  source_id  string
:8   user_id    ["null","string"] default null
:7   header     EventEnvelopeHeader
```

`mytheus-events/schemas/billing-events/v1.avsc`:

```
:8   event_type enum BillingEventType { quota_exceeded, dunning, plan_changed }
:9   subscription_id ["null","string"] default null
:10  detail_json string
```

**Two gaps to note before writing code.**

(a) `UsageEvent` has **no `model_tier` field**, but `plan/plan-7-9-agents-and-monetization.md:136`
writes `model_tier=msg.model_tier` in the consumer and `:157` defines
`usage_aggregates.tier_breakdown jsonb` as `{haiku, sonnet, opus}`. The Haiku/Sonnet/Opus split
the `/usage` console is specified to render (`plan-7-9:230`) **cannot be computed from the
topic as it exists**. Resolve by adding `model_tier` as a BACKWARD-compatible optional field
(`["null","string"] default null`) — this is Task 6, and it must land before
`agent-runtime-service` starts producing, because a schema change after production begins
means a backfill you cannot do (7-day topic retention).

(b) `BillingEvent.event_type` has three symbols; ADR-0011 §4 and the state machine need a
distinct **`quota_approaching`** signal for the 80%/95% alerts (ADR-0011:191 "billing-events
{quota approaching}"). `plan-7-9:144-146` calls `budget_alert(org_id, 95)`. Today that must be
squeezed into `quota_exceeded` + `detail_json`, which conflates "warn" with "blocked" for
`notification-service`. Either add an enum-tail symbol with a default (BACKWARD-safe, per
ADR-0011:271) or specify the `detail_json` discriminator explicitly. Pick one in Task 6; do not
leave it to the consumer author to guess.

---

## 2. Data model

Seven tables. The DDL in `plan/plan-7-9-agents-and-monetization.md:86-183` is complete and
canon-conformant; this section is the delta, the rationale, and the indexes it omits.

Tenant column is `org_id uuid NOT NULL` on all of them **except `plans`**.

### 2.1 `plans` — the documented RLS exception

```sql
CREATE TABLE plans (
  id                       uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  code                     text NOT NULL,        -- free | pro | enterprise
  name                     text NOT NULL,
  seat_price_cents         integer NOT NULL DEFAULT 0,
  included_meeting_minutes bigint  NOT NULL,     -- semantics BLOCKED on Decision A
  included_agent_tokens    bigint  NOT NULL,     -- semantics BLOCKED on Decision A
  features                 jsonb   NOT NULL DEFAULT '{}'::jsonb,
  version                  integer NOT NULL DEFAULT 1,
  created_at               timestamptz NOT NULL DEFAULT now(),
  UNIQUE (code, version)
);
-- NO org_id. NO RLS. NO FORCE.
```

**Why `plans` is the exception, stated so a reviewer can check it rather than trust it.**
The repo's RLS rule is that every tenant-scoped table carries `org_id` and a `tenant_isolation`
policy `USING (org_id = current_setting('app.current_org')::uuid)`. A policy needs a column to
key on. `plans` is a **global catalog** — `_canon/02-data-model.md:101` states it plainly:
"No (global catalog) … Global, non-tenant. Versioned; old plan rows retained for invoice
reproducibility." Every tenant reads the same three rows. Adding `org_id` would mean a copy of
the catalog per tenant, which breaks the versioning property invoices depend on: a 7-year-
retained invoice (`_canon/02-data-model.md:104`) must be able to resolve the exact plan row it
was priced from, and that row must not be per-tenant mutable.

`plans` is therefore the **second** deliberately un-policied table in this schema. The first is
`organizations` (`_canon/02-data-model.md:74`, "deliberately NOT `tenant_isolation`-policied…
it carries no `org_id` column"; its policy keys on its own `id` per
`plan/plan-0-scaffold-and-contracts.md:422`). The two are exceptions for opposite reasons:
`organizations` is the tenant root, `plans` is above tenancy entirely.

Protection instead of RLS: `GRANT SELECT` to the app role, **no INSERT/UPDATE/DELETE**. The
catalog is changed by migration only. This matches ADR-0021's explicit-grants posture
(`adr/0021-explicit-grants-no-schema-wide-default-privileges.md`) and the retirement of
schema-wide defaults in migration `9b2c4d1e6f30_0005`. Write the reason into the migration
docstring — an un-policied table is exactly the thing a future reader will "fix" by adding a
policy and break invoice reproducibility.

### 2.2 The six tenant-scoped tables

All six get `ALTER TABLE … ENABLE ROW LEVEL SECURITY` **and** `FORCE ROW LEVEL SECURITY`, plus
the canonical policy:

```sql
CREATE POLICY tenant_isolation ON <t>
  USING (org_id = current_setting('app.current_org')::uuid);
```

`FORCE` matters: without it the table owner bypasses the policy, and the migration role is the
owner. `meeting_rooms` is the in-repo precedent (`_canon/02-data-model.md:113`, "`ENABLE` +
`FORCE`, `tenant_isolation`").

| Table | Key columns | Uniqueness | Notes |
|---|---|---|---|
| `subscriptions` | `org_id`, `plan_id`→`plans(id)`, `stripe_subscription_id`, `status`, `seats`, `trial_ends_at`, `current_period_start/end` | `UNIQUE (org_id)` | One active sub per tenant. `status` CHECK in (`trialing`,`active`,`past_due`,`canceled`). |
| `invoices` | `org_id`, `subscription_id`, `stripe_invoice_id`, `status`, `amount_due_cents`, `tax_cents`, `pdf_url` | `UNIQUE (stripe_invoice_id)` | Immutable once finalized; 7-yr retention; **excluded from RTBF purge** (`plan-7-9:134`). |
| `usage_events` | `org_id`, `event_type`, `quantity numeric`, `model_tier`, `idempotency_key`, `occurred_at` | `PK (id, occurred_at)`; `UNIQUE (org_id, idempotency_key)` per partition | `PARTITION BY RANGE (occurred_at)`, daily. Append-only. 90-day raw retention. |
| `usage_aggregates` | `org_id`, `dimension`, `billing_period` (`YYYY-MM`), `quantity`, `tier_breakdown jsonb` | `UNIQUE (org_id, dimension, billing_period)` | The upsert key that makes the rollup idempotent and re-runnable (ADR-0011:274). |
| `agent_budgets` | `org_id`, `period` (`YYYY-MM`), `token_quota`, `tokens_consumed`, `hard_cap`, `alert_80_sent`, `alert_95_sent`, `state` | `UNIQUE (org_id, period)` | `state` CHECK in (`ok`,`throttled`,`blocked`) — **exactly three**, see §4. |
| `quotas` | `org_id`, `dimension`, `period`, `quota`, `consumed`, `state` | `UNIQUE (org_id, dimension, period)` | Non-token dimensions: `meeting_minute`, `transcription_second`, `storage_gb`. |

**Indexes the plan DDL omits and the query paths need:**

- `usage_events (org_id, occurred_at)` — the rollup's scan key. Per-partition.
- `usage_events (org_id, event_type, occurred_at)` — the rollup groups by dimension; without it
  every nightly run seq-scans a day partition per tenant.
- `usage_aggregates (org_id, billing_period)` — the `/usage` API's read.
- `agent_budgets (org_id, period)` is the unique constraint; no second index needed. The hot
  path is Redis, not this table.
- `invoices (org_id, created_at DESC)` — the `/billing` invoice list.
- `subscriptions (stripe_subscription_id)` — webhook lookup, and it is not the RLS path: a
  webhook arrives with no tenant context and must resolve `org_id` **from** this row. See §5.3.

**Partitioning.** `usage_events` is day-partitioned by `occurred_at`. `plan-7-9:150` specifies
`pg_partman` daily premake. The repo has no pg_partman today — `audit_logs`
(`c3e8b1d47f92_0004:86`) and `chat_messages` (`5ed978467cad:90`) both create partitions with
plain `op.execute(f"CREATE TABLE {name} PARTITION OF …")` in the migration plus a DEFAULT
partition. **Follow the existing pattern rather than introducing pg_partman for one table**
(Task 4); a DEFAULT partition plus a rolling premake job is less new infrastructure and matches
what the operator already knows. Note the trap the existing migrations already solved: a new
partition is a new table and inherits no grants under ADR-0021, so the premake path must
re-`GRANT` — `migrations/_grants.py:6` documents exactly this.

### 2.3 `organizations.plan_id`

Do **not** add the FK. `subscriptions` is the relationship (`_canon/02-data-model.md:102`,
`plan-7-9:105`: `subscriptions.plan_id uuid NOT NULL REFERENCES plans(id)`), and
`subscriptions.org_id` is `UNIQUE`, so `organizations.plan_id` is a second, denormalized,
never-written copy of a fact that now has an owner. Two writable homes for one fact is how they
drift. Drop the column (Task 18), and fix the four docs that call it an FK. `verify-scaffolds.md:429`
already lists this as pending work.

---

## 3. The metering path

```
agent-runtime-service ─┐
rag-service ───────────┴─> usage-events topic (org_id-keyed, 7d) ─> billing-metering-service
                                                                      │ inbox dedupe
   billing-metering-service internal derivation ──────────────────────┤
   (meeting_minute, transcription_second, storage_gb)                 v
                                                              usage_events (day-partitioned)
                                                                      │ nightly rollup
                                                                      v
                                                              usage_aggregates
                                                                  ├──> Stripe metered usage (P13)
                                                                  └──> quotas / agent_budgets
                                                                            │
                                                          Redis nexus:{org_id}:budget:{period}
                                                                            │
                                                              agent-runtime-service pre-dispatch
```

### 3.1 Who emits

**Only `agent_token` traverses the topic.** Its sole sanctioned producers are
`agent-runtime-service` and `rag-service` (`_canon/01-service-catalog.md:122` lists
`usage-events` as a *consumed* topic for billing; ADR-0011:145 is the load-bearing
reconciliation paragraph). The other three dimensions are **derived inside
`billing-metering-service`** and written directly as `usage_events` rows:

- `meeting_minute` — from `meetings` durations. Billed **once per meeting, not per participant**
  (ADR-0011:145; per-participant billing is the 6× inflation `D3` explicitly marked "do not do
  this" in the capacity math).
- `transcription_second` — from `transcript_chunks` start/end timestamps.
- `storage_gb` — nightly sweep of the object buckets.

Adding a topic producer for any of these requires amending `_canon/01-service-catalog.md` §3/§5
**first** (ADR-0011:145, ADR-0011:236). That is a canon edit, not an engineering choice.

### 3.2 Idempotency

Canonical key, frozen: `usage:{org_id}:{event_type}:{occurred_at}:{source_id}` (ADR-0011:158).
Derived from immutable facts only — never `event_id`, enqueue time, or a retry counter
(ADR-0011:161). Three layers, all required:

1. Redis dedupe `nexus:{org_id}:dedupe:usage-events:{idempotency_key}`, 24h TTL.
2. `UNIQUE (org_id, idempotency_key)` on `usage_events` + `ON CONFLICT DO NOTHING` — the durable
   backstop for redelivery beyond 24h.
3. The durable inbox table (`libs/shared/inbox.py`).

This is what makes the hard quota a margin *guarantee* rather than an approximation
(ADR-0004:246). A DLQ replay carries the **original** key, so a replay already applied dedupes
out and can never double-bill (ADR-0011:277).

### 3.3 Aggregation

Nightly Temporal schedule in `workflow-service`. Idempotent upsert on
`(org_id, dimension, billing_period)`, so re-running a period is safe and is the documented
repair path for a metering bug (ADR-0011:274). `/usage` and every dashboard read
**aggregates only**, never the raw firehose (ADR-0011:150).

Accepted consequence: `/usage` display lags up to ~24h. Quota *enforcement* does not, because
it runs off the Redis mirror (ADR-0011:257). Say this in the UI (§6) — a usage number that
silently lags a day is a support ticket generator.

### 3.4 How enforcement reaches a request before it runs

The check is **inline in `agent-runtime-service`, before LiteLLM dispatch** — not at the
gateway. ADR-0004:158-169 gives the exact shape: a single Redis `HGET` on
`nexus:{org_id}:budget:{period}`, returning `ok` / `throttled` / `blocked`. It is a hot-path
in-memory read specifically so it does not add a Postgres round-trip to a p50 < 2s budget
(R5, ADR-0004:193).

Two properties that are easy to get wrong and are load-bearing:

- **Fail-closed, but not fail-broken.** `plan-7-9:243` — "if the Redis mirror is unreachable,
  the budget read falls through to Postgres (never fails open)." Slower, still correct. A
  Redis outage must not become free unlimited LLM spend, and must not become a total outage
  either.
- **Never shed durable facts.** Quota degradation touches discretionary LLM work only.
  `transcript-chunks`, `decisions`, `actions`, and `usage-events` themselves are never dropped
  for quota reasons (ADR-0011:65, :195). A `blocked` tenant still gets transcription and
  captions; it loses agent output.

Non-token dimensions (`quotas`) are enforced at their own call sites — meeting start for
minutes, upload for storage — not on the agent path.

---

## 4. `agent_budgets` — the hard cap under R4

Per ADR-0004 this is Stage 5 of the five-stage cost-control pipeline and the last line of
defence against the LLM cost spiral. The sizing, so the priority is legible: at canon anchors
the naive per-chunk design is `N2` = 41.5B LLM calls/month ≈ **$224M/month**, LLM is **94%** of
variable cost (`C3`), and the pipeline reduces this by **86%** (`L10`) to ≈ $18.29M/month
(ADR-0004:36-44, :175-182). `agent_budgets` is what guarantees that a tenant whose traffic
defeats the gating still cannot run an unbounded bill — "the floor under margin"
(ADR-0011:184).

**One row per `(org_id, period=YYYY-MM)`.** Columns per `_canon/02-data-model.md:111`.

### 4.1 The state machine — three states, and two flags that are not states

```
[*] ──> ok
ok ──> ok        consume < 80%
ok ──> ok        cross 80%  → set alert_80_sent, emit billing-events
ok ──> ok        cross 95%  → set alert_95_sent, emit billing-events
ok ──> throttled reach 100% of token_quota (soft)   [or dunning day-14]
throttled ──> blocked  reach hard_cap
throttled ──> ok  new period (YYYY-MM rollover)
blocked   ──> ok  new period / upgrade
```

ADR-0004:131 is explicit and worth quoting into the code: the enum has **exactly three
members**; "the 80%/95% crossings are **not** states; they are self-transitions on `ok`". The
acceptance check at `plan-7-9:241` tests precisely this — `state` stays `ok` through both
alerts. Model them as booleans; a four- or five-state enum is a canon deviation.

**Behaviour per state** (ADR-0011:189-193):

| state | trigger | agent tier |
|---|---|---|
| `ok` | `< 80%` | all six agents normal |
| `ok` + `alert_80_sent` | crosses 80% | notify owner/admin; **no behaviour change** |
| `throttled` | 100% of `token_quota`, or dunning day-14 | Haiku triage only; suspend Sonnet/Opus extraction and `ProactiveAgent`; `ContextGapAgent` falls back to RAG-only |
| `blocked` | `≥ hard_cap` | agent spawns denied; transcription + captions continue; UI shows upgrade prompt |

The `throttled` path costs no new code: it is the same degradation the cost-control pipeline
already implements (ADR-0011:195).

**Per-plan shape** (ADR-0004:155-156): Free sets `token_quota = hard_cap`, so there is **no
throttle band** — Free transitions `ok → blocked` directly, a deliberate and user-visible
cliff. Pro/Enterprise set `token_quota < hard_cap`, giving a soft Haiku-only band before the
hard stop, and bill metered overage rather than blocking.

### 4.2 The dependency, both directions

**`agent_budgets` → agent tier.** The budget check is inline in `agent-runtime-service` before
every LiteLLM dispatch. No budget table and no Redis mirror means no check, and P7 turns on
token consumption. `22-roadmap-and-phasing.md:152` states the precondition without hedging:
metering and per-tenant budgets "must be live by P7 … or the R4 cost bomb detonates unmetered."
It is one of only two `crit`-marked tasks on the roadmap gantt (`:298-300, :312`).

**Agent tier → `agent_budgets`.** The budget is fed by `agent_token` events that only
`agent-runtime-service` and `rag-service` produce. Until they emit, `tokens_consumed` is always
zero and the state machine is untestable against real burn. This is why ADR-0011:276 and
`plan-7-9:244` both require shipping **alert-only first**: `alert_80_sent`/`alert_95_sent` fire,
`throttled`/`blocked` do not degrade anything, until real per-tenant burn curves validate the
quota math. Then flip per-cohort via the `admin-ops-service` feature flag, **Enterprise last**.
Rollback is flipping the flag — no code change (ADR-0011:276).

Practical consequence for sequencing: build the tables, consumer, mirror and state machine
**before** the agent tier produces, ship enforcement **off**, and treat the flag flip as a
separate dated decision with an owner. Do not let "budgets are live" mean "enforcement is on";
they are two events weeks apart.

### 4.3 Provisioning

Lazy-create one row per tenant per `YYYY-MM` on first use, from `subscriptions.plan_id` →
`plans.included_agent_tokens` (`plan-7-9:230`). **This is why `plans` and `subscriptions` must
exist at P7** even though the roadmap files them under P13 — see §7.2.

The exact quota arithmetic is **blocked on Decision A** (per-seat vs per-tenant). You cannot
set a quota you cannot compute.

---

## 5. Stripe

Stripe is the system of record for subscriptions, seats, tax, invoice PDFs and payment methods.
NEXUS mirrors the minimum needed to gate features and quotas, and **no PAN ever touches NEXUS**
— card management is delegated to the hosted Stripe Customer Portal specifically to stay out of
PCI-DSS cardholder-data scope (ADR-0011:69-81).

| Concern | System of record | NEXUS mirror |
|---|---|---|
| Card / payment method | Stripe Customer Portal | none |
| Tax | Stripe Tax | `invoices.tax_cents`, read-only |
| Subscription state | Stripe | `subscriptions` |
| Invoice + PDF | Stripe | `invoices` (`pdf_url` → `nexus-invoices` bucket) |
| Plan catalog | **NEXUS** `plans` | mapped to Stripe products/prices |
| Metered usage | **NEXUS** `usage_aggregates` | reported *to* Stripe metered prices |
| Quota / budget | **NEXUS** `quotas`, `agent_budgets` | none |

### 5.1 Seats

A seat is a billable user slot (`subscriptions.seats`). Seat math is enforced **at member-invite
time by `identity-tenant-service`**, which calls `billing-metering-service` to validate against
`subscriptions.seats` and the plan cap before creating the membership row (ADR-0011:110). Pro
auto-increments the Stripe subscription quantity with proration on; Free blocks at the cap.
Note the cross-service call direction — identity calls billing, not the reverse.

### 5.2 Dunning

A Temporal saga in `workflow-service` driven by `billing-events {dunning}`: reminders via
`notification-service`, `agent_budgets.state → throttled` at **day 14**, hard suspension at
**day 21**. The saga never sends notifications itself; it uses the same `notification-service`
fabric the agents use (ADR-0011:222).

Reminder cadence has a known cross-doc seam: ADR-0011:222 says day 1/3/7, and ADR-0011:224
resolves it — `18-saas-business-architecture.md` §4.4's diagram (day 0/7/14) is the
**authoritative dunning timeline**. Day-14 and day-21 agree in both. Build to §4.4.

### 5.3 The webhook

`POST /stripe/webhook`, owned by `billing-metering-service`, on the unauthenticated allowlist by
design. Three properties, all mandatory:

1. **Signature-verified** against `STRIPE_WEBHOOK_SECRET` before the body is parsed. This is the
   *only* authentication on the path (`edge.py:49-53`).
2. **Idempotent** — the Stripe event id is the key, deduped via `nexus:{org_id}:dedupe:stripe:{event_id}`.
3. **Tenant resolution runs backwards.** The request arrives with no session and no `org_id`;
   the gateway's `resolve_org_id` (`edge.py:111`) cannot run for it. The handler must resolve
   `org_id` from `subscriptions.stripe_subscription_id` (or the Stripe customer id) *before*
   opening a tenant session — hence the index in §2.2. Until `org_id` is resolved, the handler
   holds no RLS context and must not touch a tenant-scoped table.

| Stripe event | Effect | Emits |
|---|---|---|
| `customer.subscription.created/updated` | sync `status`, `seats`, period dates | `plan-changed` if changed |
| `invoice.paid` | `invoices.status=paid`; **call identity-tenant-service** to flip `organizations.status=active` | `recovered` |
| `invoice.payment_failed` | move to `past_due`; start dunning saga | `dunning` |
| `customer.subscription.deleted` | **call identity-tenant-service** toward `pending_deletion` after grace | `canceled` |

**Hard ownership rule.** `billing-metering-service` **never** writes `organizations.status` and
**never** produces `tenant-events`. `identity-tenant-service` is the sole producer of
`tenant-events` because it owns that column; billing *calls* it (ADR-0011:212, :244). A PR that
has billing write `organizations.status` directly is rejected on sight.

### 5.4 P13 vs now

**Now (P7 band):** the webhook endpoint with signature verification — because the path is
currently open and unhandled (§1.4). A verifying handler that 501s every event type is a
security fix, and it is small.

**P13:** subscription/seat sync, invoice mirror + PDF, metered usage reporting, Customer Portal
sessions, the dunning saga, Stripe Tax, the reconciliation job.

---

## 6. What `/billing` needs to become real

`mytheus-ui/src/app/(app)/billing/page.tsx` is **128 lines** rendering `mockBilling` from
`src/lib/mock/data.ts:184-197` — a hardcoded object: `plan: "pro"`, `seatsUsed: 4`,
`seatsTotal: 10`, `pricePerSeat: 24`, three invoices dated 2026-04 through 2026-06. There is no
`src/lib/api/billing.ts`. Related, and honest about it:
`src/lib/api/org-context.ts:10-12` documents that `org.plan` "needs `subscriptions`, which is
the billing band", and deliberately reports `"starter"` — the least-privileged value — so a
plan-gated feature hides rather than offers something unpaid for. **Preserve that posture**: on
any billing fetch failure, degrade to least-privilege, never to the mock.

There is also **no `/usage` page**; `src/app/(app)/` contains billing, dashboard, decisions,
meetings, org, search, settings, tasks. The tenant-admin Usage console (`plan-7-9:230`) is a new
route, and it is the P7-band UI — `/billing` is largely P13.

To make the page real:

1. `GET /api/v1/billing/subscription` — the tenant's `subscriptions` row: `plan_id`, `seats`,
   `status`, `current_period_*`, `trial_ends_at` (`19-api-architecture.md:198`).
2. `GET /plans` — the catalog, for the plan-change surface.
3. `GET /billing/invoices` — from the `invoices` mirror; `pdf_url` links to Stripe's PDF, which
   NEXUS does not render.
4. `POST /billing/portal-session` — returns a Stripe Customer Portal URL. **"Change plan" and
   any card management redirect there**; do not build a card form, that is the PCI decision.
5. `GET /usage` — `usage_aggregates` + `agent_budgets`: consumption vs quota per dimension,
   with the Haiku/Sonnet/Opus split (blocked on the `model_tier` gap, §1.6a).
6. Budget state must be **visible**: a `throttled` or `blocked` tenant sees why its agents went
   quiet. ADR-0004:203 names the Free hard-stop "a real UX cliff" and requires it be surfaced.
   A silent stop is the worst version of this feature.
7. Label the ~24h aggregation lag on the usage figures (§3.3).
8. RBAC: `admin` tier, read from `rbac_grants` — not `users.role`, which is a job attribute and
   is never consulted for permissions (`_canon/02-data-model.md:110`).

---

## 7. Banding — what is MVP and what is not

Per `22-roadmap-and-phasing.md`.

### 7.1 The line

- **P7 (W18-20), MVP, `crit`-marked:** metering + per-tenant token budgets. `:150-152` —
  "usage metering (`usage-events` → aggregation → quota enforcement) and per-tenant token
  budgets **must be live by P7**". `:108` puts `agent_budgets`, `usage_events` and Redis
  `budget:{period}` in P7. `:237` — "Metering + per-tenant token budgets (P7); Stripe surface
  (P13)". `:312` names it one of the two hard preconditions on the whole plan.
- **P13, deferred:** `:114` — "`subscriptions`, `invoices`, `plans` (full billing surface)".
  `:299-300` — "Stripe seats/dunning/tax (P13)". `:152` — "Full Stripe subscriptions/seats/
  dunning/tax **can lag to P13**".
- **Explicitly Stage 2 / post-W36 scale-out:** multi-region billing, cell isolation for
  Enterprise, BYOC. `:170` puts `billing-metering-service` extraction in **Wave 3**, "on
  SaaS-business pressure". Not required for MVP; do not report as pending work.

### 7.2 One seam, flagged rather than resolved

The roadmap files `plans` and `subscriptions` at **P13** (`:114`). But `agent_budgets`
provisioning at P7 derives `token_quota`/`hard_cap` from `subscriptions.plan_id` →
`plans.included_agent_tokens` (`plan-7-9:230`), and `plan-7-9:88` creates all seven tables in a
**single P7 migration**.

These are consistent if read precisely — P13 owns the *full Stripe billing surface*, not the
tables' first existence — and inconsistent if read literally. The resolution this spec adopts,
because it is the only one under which P7 can compute a quota:

> **`plans` and `subscriptions` land at P7 as a minimal internal catalog and a locally-managed
> subscription row (`stripe_subscription_id` NULL, `status` seeded from onboarding). P13 adds
> Stripe as the system of record, the sync, and `invoices`.**

`invoices` genuinely defers to P13 — nothing at P7 reads it. Flagged so the roadmap owner can
correct §7.1 if they disagree; it changes ~1 day of Task 3, not the shape of the work.

### 7.3 Two external decisions block P7 go-live

Both are already recorded as DECISION-REQUIRED in `plan-7-9:776-787`. Neither is engineering.

**Decision A — plan limits per-seat vs per-tenant** (`plan-7-9:778-782`). Canon freezes
`included_agent_tokens` as a `plans` column but **does not freeze the semantics**.
`18-saas-business-architecture.md` §3.1 reads it per-seat/month; `17-cost-and-finops.md` §5.1
says a per-tenant pooled read is equally valid. It decides directly whether
`token_quota = included_agent_tokens × seats` or flat. Options: (a) per-seat, aligns to the Pro
$25/seat margin model; (b) per-tenant pooled, simpler but seat growth stops raising budget and
the margin model must be re-derived; (c) per-seat for Free/Pro, contracted pool for Enterprise.
**Owner:** Pricing/GTM lead + canon owners (requires a Data Model §2.2 amendment + ADR).
**Blocks:** Task 9 and therefore enforcement go-live. **Deadline: before W18.**

**Decision B — derived-dimension producer ownership** (`plan-7-9:783-787`). ADR-0011 says
billing derives `meeting_minute`/`transcription_second`/`storage_gb` internally, but the Service
Catalog assigns no named producer, and Plan 3-4 left it open too. Options: (a) billing derives
all three, no catalog edit; (b) `meeting-service`/`transcription-service` emit on the topic,
requires a canon edit + ADR first; (c) hybrid with a catalog-amended `storage_gb` sweep
producer. **Owner:** Billing lead + canon owners. **Blocks:** Tasks 12-14 — if unresolved, only
`agent_token` meters and minutes/storage bill nothing. **Deadline: before W19.**

**Decision C — Stripe account, products, prices, tax registration.** Not in the docs because it
is not an engineering artifact. Blocks the entire P13 band (Tasks 21-28). No test-mode key
means no webhook to verify against and no price ids to map `plans.code` onto. It can be started
today at near-zero cost and should be, because it has a lead time nothing else in this spec has.

---

## 8. Task list

Repo root `/Users/sushan/Projects/Mytheus`. Sizes: **S** ≤ 1 day, **M** 2-3 days, **L** 4+
(days given). "Blocked" means blocked on an external decision, not on engineering.

### Band P7 — metering and per-tenant budgets (MVP, `crit`)

**1. Decide plan-limit semantics (Decision A).**
Docs only: `_canon/02-data-model.md`, a new ADR. **S** (decision, not build). Deps: none.
**BLOCKED — external** (Pricing/GTM + canon owners). Gates Task 9.

**2. Decide derived-dimension producer ownership (Decision B).**
Docs only: `_canon/01-service-catalog.md` §3/§5, ADR amendment if option (b)/(c). **S**.
Deps: none. **BLOCKED — external** (Billing lead + canon owners). Gates Tasks 12-14.

**3. Migration: the seven billing/metering tables.**
`mytheus-backend/migrations/versions/<new>_billing_metering_core.py`. All seven tables per §2,
RLS `ENABLE` + `FORCE` + `tenant_isolation` on six, explicit grants per ADR-0021, `plans`
`SELECT`-only with the exception documented **in the migration docstring**. Follow
`c3e8b1d47f92_0004` for partitioned-table shape and `migrations/_grants.py` for the
partition-grant trap. **M**. Deps: none (DDL does not depend on Decision A; only the *seed
values* do).

**4. `usage_events` daily partition management.**
Same migration + a premake job. Initial partitions + DEFAULT partition in the migration; a
rolling premake that re-`GRANT`s each new partition; 90-day drop. Use the existing
`op.execute("CREATE TABLE … PARTITION OF …")` pattern rather than adding pg_partman (§2.2).
**M**. Deps: 3.

**5. SQLAlchemy models + service skeleton.**
`services/billing_metering_service/models.py`, `routes/__init__.py`, wire routers into
`main.py` (replacing the TODO at `main.py:11`). Seven models; `create_app` already exists.
**S**. Deps: 3.

**6. Close the two Avro gaps.**
`mytheus-events/schemas/usage-events/v1.avsc` — add `model_tier ["null","string"] default null`.
`billing-events/v1.avsc` — add a `quota_approaching` enum-tail symbol with default, or specify
the `detail_json` discriminator. Regenerate `clients/typescript/`, update
`subjects/*-value.json`, pass CI `check-compatibility` as BACKWARD. **S**. Deps: none.
**Do this before Task 7 and before `agent-runtime-service` produces** — a schema change after
production starts cannot be backfilled through a 7-day retention window.

**7. `usage-events` consumer with idempotent dedupe.**
`services/billing_metering_service/consumers/usage.py`. Follow the `services/core_api/consumer.py`
split exactly: a testable `handle_usage_event(decoded_msg)` plus a thin `run_consumer()`.
Re-assert `org_id` from the envelope header before any write (the standing repo rule,
`core_api/consumer.py:32-36`). Redis dedupe + `ON CONFLICT DO NOTHING` on
`UNIQUE (org_id, idempotency_key)` + durable inbox. DLQ on parse failure. **M**. Deps: 3, 5, 6.

**8. Redis budget mirror + the `ok→throttled→blocked` state machine.**
`bump_budget_mirror` per `plan-7-9:209-230`. `SELECT … FOR UPDATE` on the `agent_budgets` row;
`HSET` `nexus:{org_id}:budget:{period}`; `EXPIREAT` at period end. **Three states; 80/95 are
booleans, not states** (§4.1). Emits `billing-events`. **M**. Deps: 7.

**9. `agent_budgets` provisioning.**
Lazy-create per `(org_id, YYYY-MM)` from `subscriptions.plan_id` → `plans.included_agent_tokens`.
**S** once Decision A is in; the arithmetic is one line, the decision is the work. Deps: 3, 5.
**BLOCKED on Task 1.**

**10. Seed the `plans` catalog.**
Migration-only INSERT of free/pro/enterprise per `plan-7-9:185-196`. **S**. Deps: 3.
**BLOCKED on Task 1** — `included_*` values depend on the semantics. Do not guess
(`plan-7-9:185`: "seed against the resolved decision, never guess").

**11. Budget check contract for the agent tier.**
The `within_budget(org_id, period, est_tokens)` read (ADR-0004:158-169) plus the documented
Postgres fall-through when Redis is unreachable. Lives in `mytheus-intelligence`, but the
contract, key format and fail-closed semantics are specified and tested here. **S**.
Deps: 8. Two-way dependency with the agent tier — see §4.2.

**12. Derive `meeting_minute` + `transcription_second`.**
`services/billing_metering_service/derivation/`. From `meetings` durations and
`transcript_chunks` timestamps; write directly as `usage_events` rows with canonical
idempotency keys. **Once per meeting, never per participant** (§3.1). **M**. Deps: 3, 5.
**BLOCKED on Task 2.**

**13. `storage_gb` nightly sweep.**
Temporal schedule in `workflow-service` + the sweep in billing. **M**. Deps: 3, 5.
**BLOCKED on Task 2.**

**14. `quotas` enforcement for non-token dimensions.**
Check + update at meeting start (minutes) and upload (storage). **M**. Deps: 3, 12, 13.
**BLOCKED on Task 2.**

**15. Nightly rollup → `usage_aggregates`.**
Temporal schedule in `workflow-service`; idempotent upsert on
`(org_id, dimension, billing_period)`; re-runnable for a past period. **M**. Deps: 3, 7.

**16. `GET /usage` API.**
Reads `usage_aggregates` + `agent_budgets` only, never raw rows. RBAC `admin` via `rbac_grants`.
**S**. Deps: 5, 15.

**17. `billing-events` producer + `notification-service` consumer wiring.**
80%/95% alerts and state transitions → in-app/email/owner Slack DM. **S**. Deps: 6, 8.

**18. Drop `organizations.plan_id`; correct the docs.**
Migration + `_canon/02-data-model.md:74`, `04-service-catalog.md:169`,
`19-api-architecture.md:148`, `24-glossary-and-naming.md:87`. Already tracked at
`verify-scaffolds.md:429`. **S**. Deps: 3.

**19. Deploy the service.**
`mytheus-backend/docker-compose.yml` (currently infra-only, lines 3-76) + `Dockerfile` +
`UPSTREAM_BILLING_METERING_SERVICE`. Without this the gateway map at `edge.py:171-174` still
points at nothing. **S**. Deps: 5.

**20. `POST /stripe/webhook`: signature verification.**
Verify against `STRIPE_WEBHOOK_SECRET` before parsing; reject unsigned. May 501 every event
type initially. `stripe>=10` is already a dependency (`pyproject.toml:20`). **S**. Deps: 5, 19.
**Do this in the P7 band, not P13** — the path is on the unauthenticated allowlist
(`edge.py:52`) with no handler and no verification behind it. Needs a webhook secret from
Decision C, but a test-mode secret is minutes of work.

**21. Tenant-admin Usage console (UI).**
New route `mytheus-ui/src/app/(app)/usage/`, plus `src/lib/api/usage.ts`. Consumption vs quota
per dimension, Haiku/Sonnet/Opus split, **visible budget state**, labelled aggregation lag.
**M**. Deps: 16, and the `model_tier` half depends on Task 6.

**22. Enforcement feature flag + alert-only rollout.**
`BUDGET_ENFORCEMENT_MODE` (`alert_only` | `enforce`) via `admin-ops-service`. Ships
`alert_only`; per-cohort flip, Enterprise last; rollback is the flag. **S**. Deps: 8.

**23. P7 test suite.**
Redelivery increments exactly once; 80/95 set flags once and leave `state = ok`; Pro at 100%
→ `throttled`; Free (`quota == cap`) → `blocked` directly; Redis-down falls through to Postgres
and never fails open; tenant A cannot read tenant B's `usage_events`/`agent_budgets`; durable
facts never shed. The acceptance list is `plan-7-9:238-245`. **M**. Deps: 7, 8, 9, 12, 15.

### Band P13 — the Stripe surface (deferred; not MVP)

**24. Stripe account, products, prices, tax registration.**
No repo files. **S** of engineering, unknown calendar. Deps: 1. **BLOCKED — external
(Decision C).** Longest lead time in the spec; start now.

**25. Stripe webhook handlers.**
The four event types in §5.3; idempotent on Stripe event id; resolve `org_id` from
`stripe_subscription_id` **before** opening a tenant session. **M**. Deps: 20, 24.

**26. Subscription + seat sync, and the seat-check API.**
Two-way sync; the endpoint `identity-tenant-service` calls at member-invite time (ADR-0011:110)
plus its call site. Pro auto-increments Stripe quantity with proration. **M**. Deps: 25.

**27. `invoices` mirror + PDF.**
Mirror on webhook; `pdf_url` → `nexus-invoices` bucket; 7-yr retention; excluded from RTBF
purge. **M**. Deps: 25.

**28. Report metered usage to Stripe.**
From `usage_aggregates` on the nightly schedule — never raw events (ADR-0011:265: Stripe's
metered API is not sized for L7 = 5.79B calls/month). **M**. Deps: 15, 24.

**29. Customer Portal session endpoint.**
`POST /billing/portal-session`. Keeps NEXUS out of PCI scope. **S**. Deps: 24.

**30. Dunning saga.**
Temporal in `workflow-service`: reminders per `18-saas §4.4`, day-14 → `throttled`, day-21 →
call `identity-tenant-service` → `tenant-events {suspended}`; cancel on `invoice.paid`.
**Billing never writes `organizations.status`.** **L — 4 days.** Deps: 25, 17.

**31. Stripe reconciliation job.**
Re-pull subscription/invoice state keyed on Stripe ids, rewrite the mirror; the documented
missed-webhook repair (ADR-0011:275). **M**. Deps: 25, 27.

**32. Make `/billing` real (UI).**
Replace `mockBilling` in `page.tsx` (128 lines) with live fetches; new `src/lib/api/billing.ts`;
plan/seats/invoices/portal per §6; **degrade to least-privilege on error**, matching
`org-context.ts:10-12`. Also update `org.plan` to read the real subscription. **M**.
Deps: 26, 27, 29.

### Totals

32 tasks. **P7 band: 23 tasks** — 11 S, 11 M, 1 decision-only ≈ **26 engineering days**.
**P13 band: 9 tasks** — 2 S, 6 M, 1 L ≈ **20 days**. **Total ≈ 46 engineering days**, of which
**26 are MVP-band**. Excludes calendar time on Decisions A, B and C.

Sequencing note: Tasks 3-8, 15-19 and 23 are unblocked **today** and are ~15 days of the P7
band. Tasks 9, 10, 12, 13 and 14 sit behind Decisions A and B. Task 20 should be pulled to the
front of the queue regardless of band.

### Not in scope — explicitly Stage 2

Multi-region billing, Enterprise cell isolation, BYOC, and the Wave-3 extraction of
`billing-metering-service` to its own deployment unit (`22-roadmap-and-phasing.md:170`, "on
SaaS-business pressure"). These are post-W36 scale-out. They are **not** pending MVP work and
should not be reported as such.
