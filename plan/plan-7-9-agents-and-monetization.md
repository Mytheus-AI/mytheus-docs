# Plan 7-9 — Agent Intelligence & Monetization  (Agents + Billing)

> Ship the core agent value on the cost-controlled LLM pipeline with billing/metering + per-tenant token budgets LIVE FIRST: `PersonalListenerAgent` (P7), then `DecisionAgent`/`ActionAgent` + the AGE `nexus_knowledge` graph (P8), then RAG-powered `QuestionAgent` (P9) — agents propose, `core-api` commits, and no tenant can run an unbounded LLM bill.

---

## 0. Band at a glance

- **Phases / weeks:** **P7 Personal Listener Agent (W18-20)** → **P8 Decisions / Actions + Knowledge Graph (W21-23)** → **P9 RAG & Question Agent (W24-26)**. P7 lights up the agent tier and the cost/billing substrate that gates it; P8 adds confirm-only decisions/actions and the knowledge graph; P9 adds request-time retrieval and live question suggestions.
- **Goal:** turn live `transcript-chunks` into personalized, cost-bounded agent value. P7 stands up `agent-runtime-service` (the five-lever cost-controlled pipeline) and `billing-metering-service` (usage metering + `agent_budgets` hard quotas) **together**, because the R4 cost bomb detonates the moment agents run unmetered. P8 ships `DecisionAgent`/`ActionAgent` as candidate emitters with `core-api` as the SOLE confirm-only producer of the `decisions`/`actions` events, plus `knowledge-graph-service` writing AGE edges on confirmation only. P9 ships `rag-service` (`expand → retrieve → rerank → assemble → cite`) and `QuestionAgent` (priority 1-3 suggestion cards).
- **Repos & folders touched:**
  - `mytheus-intelligence` — **PRIMARY.** New `services/agent-runtime-service/` (six-agent runtime, `BaseAgent`, `AgentState`, `PromptLoader`, tiered LiteLLM router, budget gate), `services/rag-service/` (gRPC orchestrator), `services/knowledge-graph-service/` (AGE writer + Cypher templates), `libs/shared/` (router, prompt loader, envelopes, tenancy), `proto/` (gRPC mesh). Consumes `embedding-service` / `reranker-service` / `search-service` stood up in Plan 5-6.
  - `mytheus-backend` — **`billing-metering-service`** (`usage_events`, `usage_aggregates`, `agent_budgets`, `quotas`, `plans`, `subscriptions`, `invoices`, Stripe + Redis budget mirror); **`core-api`** (consume `agent-results` → write `worklogs` / candidate `agent_suggestions`; confirm endpoints → SOLE producer of `decisions`/`actions`; `decisions`/`action_items` tables); `libs/shared`; `migrations/`.
  - `mytheus-events` — Avro subjects `agent-results-value`, `agent-stream-value`, `agent-requests-value`, `agent-responses-value`, `decisions-value`, `actions-value`, `usage-events-value`, `billing-events-value`, `graph-events-value` (+ each `.dlq`), registered BACKWARD.
  - `mytheus-ui` (`web-app`) — agent sidebar shell, My Notes (worklogs), decision/action confirm cards, My Tasks, decision lineage view, Ask (RAG) panel, Usage / budget console section.
  - `mytheus-mobile` — agent sidebar, My Notes, My Tasks, decisions lineage, Ask (RAG) screen wired to `bff` + `agent-results` over `realtime-gateway`; remove recap/insights mock screens.
- **Demoable at the end:**
  - **End of P7 (W20):** in a live meeting, each participant's `PersonalListenerAgent` produces role-relevant **worklogs** in the My Notes sidebar (`>0.7` surfaced, `0.4-0.7` silent-logged); every LLM call is metered to `usage-events`; the tenant-admin Usage panel shows token burn against `agent_budgets`; an 80%/95% alert fires; a Free tenant that exhausts `included_agent_tokens` is `blocked` while transcription/captions continue.
  - **End of P8 (W23):** `DecisionAgent`/`ActionAgent` surface confirm/edit/dismiss cards; on confirm, `core-api` writes the row and is the SOLE producer of `decisions`/`actions`; `knowledge-graph-service` creates `PRODUCED`/`ASSIGNED_TO`/`RELATES_TO`/`FOLLOWS_FROM` edges in `nexus_knowledge`; the decision-lineage UI walks `FOLLOWS_FROM`; My Tasks lists `action_items`. R1 go/no-go on AGE is recorded.
  - **End of P9 (W26):** `POST /api/rag/query` returns cited passages (hybrid RRF over Qdrant + FTS + AGE, reranked top-50→top-10, 10-min answer cache); `QuestionAgent` drops priority 1-3 suggestion cards into `agent_suggestions` with Ask/Dismiss/Save-for-later; the Ask panel and mobile Ask screen render cited answers.

---

## 1. Entry preconditions  (must be GREEN before starting)

Plan 7-9 sits on top of Plans 0, 1-2, 3-4, and **5-6**. Before W18:

1. **Identity is Clerk (RESOLVED DECISION 1).** Clerk JWKS verification lives in `mytheus-backend/libs/shared`; the Supabase-Auth drift is reverted (no `custom_access_token_hook`, no `auth.users → public.users` trigger; `CLERK_*` env restored; Svix webhook user-sync in `identity-tenant-service`). `api-gateway` extracts `org_id` **once at the edge** from the Clerk JWT `org` claim and fails closed. Every agent spawn, every budget check, and every `POST /api/rag/query` depends on a trustworthy edge-injected `org_id`. **If not done, STOP — fix it in the identity band.**
2. **Capture substrate is live (Plan 3-4).** `transcript-chunks` flows from `transcription-service` (FINALS ONLY, Avro `TranscriptChunk`, partition key per the ratified DECISION-REQUIRED #1 of Plan 3-4); `transcript_chunks`, `meeting_speakers`, `chat_messages`, `meetings`, `meeting_participants` exist with RLS FORCE; `realtime-gateway` fan-out + `agent-stream`/`agent-results`/`notifications` consumers are wired (no-op-tolerant) so agent output can reach the UI.
3. **Inference tier is live (Plan 5-6).** `embedding-service` (BGE-small-en, 384-dim, GPU Ray Serve, **only writer of Qdrant**) embeds-on-ingest from `transcript-chunks`/`document-chunks` into per-tenant collections `transcript_chunks__{org_id}` / `document_chunks__{org_id}`; `reranker-service` (BGE-reranker-base cross-encoder, warm-replica floor ≥ 1/region) answers `POST /rerank`; `search-service` owns the Postgres FTS (`tsvector` GIN on `transcript_chunks` + `chat_messages`) under RLS; `ingestion-service` owns `document_chunks` / `document_elements`. **RAG in P9 cannot fuse three arms without these three servers — if any is missing, land it in Plan 5-6 first.**
4. **At-least-once + idempotency is live.** Transactional outbox on produce, inbox + `nexus:{org_id}:dedupe:{topic}:{idempotency_key}` (24h TTL) on consume, per-topic DLQ. Avro + Schema Registry, **BACKWARD**, subject `{topic}-value`, partition key `org_id`. The `EventEnvelopeHeader` (`event_id`, `idempotency_key`, `org_id`, `schema_version`, `producer`, `occurred_at`, `correlation_id`, `causation_id`) opens every value.
5. **`tenant_settings` / `data_residency` exist** (owned by `identity-tenant-service`). `tenant_settings.agent_confidence_overrides` and `tenant_settings.pii_redaction_enabled` are readable; the per-tenant confidence override and LLM-egress redaction node both key off them.
6. **`get_user_context(user_id)` is callable over gRPC** from `identity-tenant-service` (role/skills/team/recent meetings/topics), Redis-cached at `nexus:{org_id}:user_ctx:{user_id}` (5-min TTL). PersonalListener relevance pre-gating and RAG expansion both depend on it. Role + skills were captured at onboarding (Plan 1-2) into `users.role` / `users.skills text[]`.
7. **The role→topics taxonomy is version-controlled** (YAML/JSON in `libs/shared`), reviewable in PRs — it seeds `RELATES_TO` and the relevance pre-gate.
8. **Capacity anchors are the sizing target**, never the seeder: `A17`=30s window, `T2`=360 finals/meeting → **90 windows/meeting**, `A7`=6 roster, gate-pass **0.35**, `L1`=540 Haiku triage, `L2`=567 Sonnet extraction, `L6`≈1,206 calls/meeting, `L7`≈5.789B/mo, `L10`=86% reduction, `C1`≈$4.06/meeting, `C2`≈$9.73/MAU, `C3`=94% LLM, `C4`≈$6.99.

> WARNING — **Billing is a HARD P7 precondition, not a P8 follow-on.** The agent tier is a cost center where LLM is 94% of variable cost (`C3`). The moment `PersonalListenerAgent` runs unmetered, a single tenant can defeat the gating and run an unbounded bill (R4). `billing-metering-service` (usage metering + `agent_budgets` mirror) MUST be live, in alert-only mode at minimum, **before** the first agent dispatches an LLM call. Build §3.1 (billing) before §3.2 (runtime).

---

## 2. Canonical names introduced in this band

This band introduces **no new** service, table, topic, collection, Redis-key, AGE-label, or agent-class names — all are frozen canon being materialized. Names used verbatim:

- **Services:** `agent-runtime-service`, `rag-service`, `knowledge-graph-service` (mytheus-intelligence); `billing-metering-service`, `core-api` (mytheus-backend); `embedding-service`, `reranker-service`, `search-service`, `ingestion-service` (called, stood up Plan 5-6); `identity-tenant-service`, `notification-service`, `realtime-gateway`, `bff`, `api-gateway`, `web-app`.
- **Agents:** `PersonalListenerAgent` (P7), `DecisionAgent`, `ActionAgent` (P8), `QuestionAgent` (P9). `ContextGapAgent` (P10) and `ProactiveAgent` (P11) are OUT of this band — the runtime is built to host them, but they do not ship here.
- **Tables:** `worklogs`, `decisions`, `action_items`, `agent_suggestions` (owner `core-api`); `usage_events`, `usage_aggregates`, `agent_budgets`, `quotas`, `plans`, `subscriptions`, `invoices` (owner `billing-metering-service`); `tenant_settings` (read, owner `identity-tenant-service`).
- **Topics:** `agent-results`, `agent-stream`, `agent-requests`, `agent-responses`, `decisions`, `actions`, `usage-events`, `billing-events`, `graph-events`, `notifications` (each paired with `<topic>.dlq`).
- **Frozen Avro schemas (canon):** `AgentResult`, `Decision`, `Action`, `AgentRequest`, `AgentResponse`, `Notification`. **Net-new (not frozen canon) schema names** from [07](../07-event-architecture.md): `UsageEvent`, `BillingEvent`, `GraphEvent`, the `agent-stream` token-delta envelope — usable but flagged.
- **AGE graph:** `nexus_knowledge`. Nodes `Person`, `Meeting`, `Decision`, `Action`, `Topic`. Edges `ATTENDED`, `PRODUCED`, `ASSIGNED_TO`, `RELATES_TO`, `FOLLOWS_FROM`.
- **Qdrant:** `decisions__{org_id}` (embedded on confirm, P8); `transcript_chunks__{org_id}`, `document_chunks__{org_id}` (read by RAG, P9).
- **Redis keys (all `nexus:{org_id}:…`):** `nexus:{org_id}:budget:{period}` (hot `agent_budgets` mirror), `nexus:{org_id}:rag:{user_id}:{query_hash}` (RAG answer cache, 10-min TTL), `nexus:{org_id}:qembed:{query_hash}` (query-embed cache, Plan 5-6), `nexus:{org_id}:user_ctx:{user_id}` (5-min), `nexus:{org_id}:dedupe:{topic}:{idempotency_key}` (inbox), `nexus:{org_id}:queue:agent:{user_id}` (per-user priority queue), `nexus:{org_id}:ratelimit:{user_id}` (LiteLLM per-user counters).
- **Utilities / classes:** `BaseAgent`, `AgentState`, `PromptLoader`, `LiteLLMRouter`, `RAGService`, `get_user_context(user_id)`, `check_budget(org_id, est_tokens)`.
- **Idempotency key (frozen):** `usage:{org_id}:{event_type}:{occurred_at}:{source_id}` (usage-events).
- **Capacity symbols (imported, never re-derived):** `A17`, `T2`, `A7`, `L1`-`L10`, `N1`-`N3`, `C1`-`C4`, `D1`, `EM4`, `V5`.

---

## 3. Work breakdown (ordered, grouped by phase then component)

Ordering is a precondition chain. **Within P7: billing before runtime before the agent** — the budget gate must exist before any LLM call, and the agent reads the budget mirror inline. **Within P8: candidate agents before the confirm path before the graph** — `core-api` confirm endpoints are the SOLE producer of `decisions`/`actions`, and `knowledge-graph-service` writes edges only on those confirmed events. **Within P9: `rag-service` before `QuestionAgent`** where QuestionAgent leans on retrieval, and the answer cache before the consumers that benefit from it.

---

### PHASE P7 — Personal Listener Agent (W18-20)

#### 3.1 `billing-metering-service` — usage metering, `agent_budgets`, Redis mirror, R4 enforcement  (BUILD FIRST)

**Repo / folder:** `mytheus-backend/services/billing-metering-service/` (today: a stub `main.py` that boots FastAPI with no routes).

**Why first:** the agent tier cannot dispatch an LLM call until there is a budget to check. This component lands the metering pipeline and the `agent_budgets` state machine in **alert-only** mode (per [ADR-0011 §Migration](../adr/0011-billing-and-metering.md) and [ADR-0004 §Migration](../adr/0004-llm-cost-control-pipeline.md)), then flips on deny-by-default once real per-tenant burn curves are observed.

**Steps:**

1. **Add deps** to the workspace:
   ```bash
   cd mytheus-backend
   uv add stripe confluent-kafka fastavro redis
   ```
2. **Migrations — the billing/metering core** (data-model edit is materialization of frozen canon, not a new name). Generate:
   ```bash
   uv run alembic revision -m "feat(billing-metering-service): plans, subscriptions, invoices, usage_events, usage_aggregates, agent_budgets, quotas"
   ```
   Migration body (RLS FORCE on tenant-scoped tables; `plans` is the one global non-tenant catalog):
   ```sql
   -- plans: GLOBAL non-tenant catalog, versioned, never hard-deleted (invoices must reproduce).
   CREATE TABLE plans (
     id                       uuid        NOT NULL DEFAULT gen_random_uuid(),
     code                     text        NOT NULL,          -- free | pro | enterprise
     name                     text        NOT NULL,
     seat_price_cents         integer     NOT NULL DEFAULT 0,
     included_meeting_minutes bigint      NOT NULL,          -- per-seat OR per-tenant: see DECISION-REQUIRED #1
     included_agent_tokens    bigint      NOT NULL,          -- per-seat OR per-tenant: see DECISION-REQUIRED #1
     features                 jsonb       NOT NULL DEFAULT '{}'::jsonb,
     version                  integer     NOT NULL DEFAULT 1,
     created_at               timestamptz NOT NULL DEFAULT now(),
     PRIMARY KEY (id),
     UNIQUE (code, version)
   );  -- NO org_id, NO RLS: global catalog.

   CREATE TABLE subscriptions (
     id                     uuid        NOT NULL DEFAULT gen_random_uuid(),
     org_id                 uuid        NOT NULL REFERENCES organizations(id),
     plan_id                uuid        NOT NULL REFERENCES plans(id),
     stripe_subscription_id text,
     status                 text        NOT NULL DEFAULT 'trialing', -- trialing|active|past_due|canceled
     seats                  integer     NOT NULL DEFAULT 1,
     trial_ends_at          timestamptz,
     current_period_start   timestamptz,
     current_period_end     timestamptz,
     created_at             timestamptz NOT NULL DEFAULT now(),
     updated_at             timestamptz NOT NULL DEFAULT now(),
     PRIMARY KEY (id),
     UNIQUE (org_id)                                          -- exactly one active sub per tenant
   );

   CREATE TABLE invoices (
     id                uuid        NOT NULL DEFAULT gen_random_uuid(),
     org_id            uuid        NOT NULL REFERENCES organizations(id),
     subscription_id   uuid        NOT NULL REFERENCES subscriptions(id),
     stripe_invoice_id text,
     status            text        NOT NULL,                  -- open|paid|void
     amount_due_cents  integer     NOT NULL,
     tax_cents         integer     NOT NULL DEFAULT 0,
     pdf_url           text,                                  -- → nexus-invoices bucket
     created_at        timestamptz NOT NULL DEFAULT now(),
     PRIMARY KEY (id)
   );  -- IMMUTABLE once finalized; 7-yr retention; excluded from RTBF purge.

   -- usage_events: append-only metering feed, DAY-partitioned by occurred_at.
   CREATE TABLE usage_events (
     id              uuid        NOT NULL DEFAULT gen_random_uuid(),
     org_id          uuid        NOT NULL,
     event_type      text        NOT NULL,   -- agent_token | meeting_minute | transcription_second | storage_gb
     quantity        numeric     NOT NULL,
     model_tier      text,                    -- haiku|sonnet|opus for agent_token; null otherwise
     idempotency_key text        NOT NULL,    -- usage:{org_id}:{event_type}:{occurred_at}:{source_id}
     occurred_at     timestamptz NOT NULL,
     PRIMARY KEY (id, occurred_at)
   ) PARTITION BY RANGE (occurred_at);
   -- daily partitions via pg_partman; UNIQUE (org_id, idempotency_key) per partition (dedupe backstop);
   -- index (org_id, occurred_at); RLS FORCE + tenant_isolation.

   CREATE TABLE usage_aggregates (
     id             uuid        NOT NULL DEFAULT gen_random_uuid(),
     org_id         uuid        NOT NULL REFERENCES organizations(id),
     dimension      text        NOT NULL,     -- = usage_events.event_type
     billing_period text        NOT NULL,     -- YYYY-MM
     quantity       numeric     NOT NULL,
     tier_breakdown jsonb       NOT NULL DEFAULT '{}'::jsonb, -- {haiku, sonnet, opus} for agent_token
     updated_at     timestamptz NOT NULL DEFAULT now(),
     PRIMARY KEY (id),
     UNIQUE (org_id, dimension, billing_period)               -- upsert key (rollup is idempotent)
   );

   CREATE TABLE agent_budgets (
     id              uuid        NOT NULL DEFAULT gen_random_uuid(),
     org_id          uuid        NOT NULL REFERENCES organizations(id),
     period          text        NOT NULL,     -- YYYY-MM
     token_quota     bigint      NOT NULL,
     tokens_consumed bigint      NOT NULL DEFAULT 0,
     hard_cap        bigint      NOT NULL,     -- Free: token_quota = hard_cap (no throttle band)
     alert_80_sent   boolean     NOT NULL DEFAULT false,
     alert_95_sent   boolean     NOT NULL DEFAULT false,
     state           text        NOT NULL DEFAULT 'ok',       -- ok | throttled | blocked
     created_at      timestamptz NOT NULL DEFAULT now(),
     updated_at      timestamptz NOT NULL DEFAULT now(),
     PRIMARY KEY (id),
     UNIQUE (org_id, period)                                  -- one row per tenant per month
   );

   CREATE TABLE quotas (
     id             uuid        NOT NULL DEFAULT gen_random_uuid(),
     org_id         uuid        NOT NULL REFERENCES organizations(id),
     dimension      text        NOT NULL,     -- meeting_minute | transcription_second | storage_gb
     period         text        NOT NULL,     -- YYYY-MM
     quota          numeric     NOT NULL,
     consumed       numeric     NOT NULL DEFAULT 0,
     state          text        NOT NULL DEFAULT 'ok',
     PRIMARY KEY (id),
     UNIQUE (org_id, dimension, period)
   );
   ```
   Apply RLS FORCE + the canonical `tenant_isolation` policy (`USING (org_id = current_setting('app.current_org')::uuid)`) on every table except `plans`. `usage_events` uses `pg_partman` daily premake; raw rows retained 90 days then summarized.
3. **Seed the `plans` catalog** (Free/Pro/Enterprise) per [SaaS Business §3.1](../18-saas-business-architecture.md). The exact `included_*` semantics are **NOT canon** — see DECISION-REQUIRED #1; seed against the resolved decision, never guess:
   ```sql
   INSERT INTO plans (code, name, seat_price_cents, included_meeting_minutes, included_agent_tokens, features) VALUES
     ('free',       'Free',       0,    300,  200000,  '{"context_gap_agent":false,"proactive_agent":false,"max_concurrent_meetings":20,"retention_days_max":90,"support_tier":"community"}'),
     ('pro',        'Pro',        2500, 3000, 4000000, '{"context_gap_agent":true,"proactive_agent":false,"integrations":["google_calendar","slack"]}'),
     ('enterprise', 'Enterprise', 0,    0,    0,       '{"sso_saml":true,"scim_provisioning":true,"cell_isolation":true,"proactive_agent":true}');
   ```
4. **The `usage-events` consumer** (`services/billing_metering_service/consumers/usage.py`). Per [ADR-0011 §3.1](../adr/0011-billing-and-metering.md), **only `agent_token` traverses the topic** (its sole producers `agent-runtime-service` + `rag-service`); the other three dimensions are derived inside this service:
   ```python
   async def on_usage_event(msg: UsageEvent) -> None:
       org_id = msg.header.org_id
       if await inbox_seen(org_id, "usage-events", msg.header.idempotency_key):
           return                                              # at-least-once dedupe (24h Redis + UNIQUE)
       async with tenant_session(org_id) as db:
           await db.execute(insert(UsageEventRow).values(      # append-only, UNIQUE(org_id, idempotency_key)
               org_id=org_id, event_type=msg.event_type, quantity=msg.quantity,
               model_tier=msg.model_tier, idempotency_key=msg.header.idempotency_key,
               occurred_at=msg.occurred_at).on_conflict_do_nothing())
           await bump_budget_mirror(db, org_id, msg)           # increment Redis + check thresholds
   ```
5. **Derive `meeting_minute` / `transcription_second` / `storage_gb` inside the service** — NOT via the topic (DECISION-REQUIRED #2 governs ownership). `meeting_minute`/`transcription_second` from `meetings` durations + `transcript_chunks` start/end timestamps; `storage_gb` from a nightly storage-sweep job (`workflow-service` Temporal schedule). Write them directly as `usage_events` rows.
6. **Budget mirror + threshold logic** (`bump_budget_mirror`) — the hot path that the agent tier reads. Implements the [ADR-0004 §Stage 5](../adr/0004-llm-cost-control-pipeline.md) / [Cost & FinOps §7](../17-cost-and-finops.md) state machine. `ok` (incl. 80/95 warn bands) → `throttled` at 100% of `token_quota` → `blocked` at `hard_cap`:
   ```python
   async def bump_budget_mirror(db, org_id, msg) -> None:
       if msg.event_type != "agent_token":
           return
       b = await db.scalar(select(AgentBudget).where(
           AgentBudget.org_id == org_id, AgentBudget.period == period_of(msg.occurred_at)).with_for_update())
       b.tokens_consumed += int(msg.quantity)
       key = f"nexus:{org_id}:budget:{b.period}"
       await redis.hset(key, mapping={"tokens_consumed": b.tokens_consumed,
                                      "token_quota": b.token_quota, "hard_cap": b.hard_cap})
       await redis.expireat(key, period_end_epoch(b.period))
       if b.tokens_consumed >= b.hard_cap:
           b.state = "blocked";  await emit("billing-events", quota_exceeded(org_id))
       elif b.tokens_consumed >= b.token_quota:                 # Pro/Enterprise soft band; Free has quota==cap
           b.state = "throttled"; await emit("billing-events", quota_throttled(org_id))
       elif b.tokens_consumed >= 0.95 * b.token_quota and not b.alert_95_sent:
           b.alert_95_sent = True; await emit("billing-events", budget_alert(org_id, 95))  # → notification-service
       elif b.tokens_consumed >= 0.80 * b.token_quota and not b.alert_80_sent:
           b.alert_80_sent = True; await emit("billing-events", budget_alert(org_id, 80))
       await redis.hset(key, "state", b.state)
   ```
7. **Nightly rollup** (`workflow-service` Temporal schedule, idempotent upsert keyed `(org_id, dimension, billing_period)`): collapse `usage_events` → `usage_aggregates`; report `agent_token` tier dollars + metered overage to Stripe usage records. `/usage` and dashboards read **aggregates only**, never the raw firehose.
8. **`/usage` API + tenant-admin Usage section** (RBAC `admin`): read `usage_aggregates` + `agent_budgets`; render consumption vs quota by dimension with the Haiku/Sonnet/Opus split. Stripe webhook + dunning are landed in their own band; this band lands metering + budgets.
9. **Provision an `agent_budgets` row per tenant per current `YYYY-MM`** on first use (lazy create from `subscriptions.plan_id`), with `token_quota`/`hard_cap` derived from `plans.included_agent_tokens × seats` (or per-tenant — DECISION-REQUIRED #1).
10. **Ship in alert-only mode first.** Set the enforcement feature flag (`admin-ops-service`) so `alert_80_sent`/`alert_95_sent` fire but `throttled`/`blocked` do NOT degrade agents until real burn curves validate the quota math. Flip per-cohort, Enterprise last.
11. **Env vars** (`mytheus-backend/.env`): `STRIPE_API_KEY`, `STRIPE_WEBHOOK_SECRET`, `REDIS_URL`, `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`, `DATABASE_URL`, `BUDGET_ENFORCEMENT_MODE` (`alert_only`|`enforce`).

**Canonical names:** `billing-metering-service`, `usage_events`, `usage_aggregates`, `agent_budgets`, `quotas`, `plans`, `subscriptions`, `invoices`, `usage-events`, `billing-events`, `UsageEvent`, `nexus:{org_id}:budget:{period}`, `usage:{org_id}:{event_type}:{occurred_at}:{source_id}`.

**Acceptance checks:**
- An `agent_token` `usage-events` message increments `usage_events` exactly once even on redelivery (UNIQUE `(org_id, idempotency_key)` + Redis dedupe); the Redis budget mirror reflects the new total within one consume.
- Crossing 80% sets `alert_80_sent` (once) and emits `billing-events`; crossing 95% sets `alert_95_sent`; `state` stays `ok` through both — they are flags, not states.
- With enforcement on: a Pro tenant at 100% of `token_quota` flips to `throttled`; a Free tenant (`token_quota == hard_cap`) flips straight `ok → blocked`; durable facts (`transcript-chunks`, future `decisions`/`actions`) are never shed.
- Deny-by-default: if the Redis mirror is unreachable, the budget read falls through to Postgres (never fails open).
- RLS isolation: tenant A cannot read tenant B's `usage_events`/`agent_budgets`.

---

#### 3.2 `agent-runtime-service` — `BaseAgent` + `AgentState` + `PromptLoader` + the cost-controlled pipeline

**Repo / folder:** `mytheus-intelligence/services/agent-runtime-service/` (repo is docs-only today — scaffold it), with shared pieces in `libs/shared/` and gRPC in `proto/`.

**Why second:** the runtime hosts all six agents but ships only `PersonalListenerAgent` here. It writes **nothing to Postgres** — it emits `agent-results` and lets `core-api` commit. The pipeline is the R4 fix; build it before any agent so every agent inherits the gate.

**Steps:**

1. **Scaffold the service + shared layout** per [mytheus-intelligence CLAUDE.md](../../mytheus-intelligence/CLAUDE.md):
   ```bash
   cd mytheus-intelligence
   mkdir -p services/agent-runtime-service/agent_runtime_service/agents libs/shared proto
   cd services/agent-runtime-service
   python -m venv .venv && source .venv/bin/activate
   pip install -e ".[dev]"   # or:
   pip install langgraph litellm "fastapi" "uvicorn[standard]" aiokafka confluent-kafka fastavro \
               redis grpcio grpcio-tools pydantic anthropic openai sqlalchemy
   ```
2. **`AgentState` dataclass** (`libs/shared/state.py`) — exactly the PDF fields plus the tenant/cost additions ([08 §3.1](../08-ai-agent-architecture.md)):
   ```python
   from dataclasses import dataclass, field
   from uuid import UUID

   @dataclass
   class AgentState:
       meeting_id: UUID
       user_id: UUID
       chunk: "TranscriptWindow"            # a 30s window (A17), NOT a raw transcript_chunks row
       context: dict                        # get_user_context() result
       retrieved_facts: list[dict] = field(default_factory=list)
       org_id: UUID = None                  # tenant key, injected by authz; spawn asserts present
       agent_id: str = ""                   # stable identity for A2A authz + audit
       relevance_score: float | None = None # PersonalListener pre-gate (0-1)
       confidence: float | None = None      # surfacing gate (0-1)
       prompt_version: str = ""             # resolved template version (staged rollout)
       token_budget_remaining: int = 0      # mirror of agent_budgets for fast quota check
       correlation_id: str | None = None
   ```
3. **`BaseAgent` lifecycle** (`libs/shared/base_agent.py`) — `spawn` / `process` / `terminate`, fail-closed on `org_id` and on exhausted budget ([08 §3.2](../08-ai-agent-architecture.md)):
   ```python
   class BaseAgent:
       name: str
       prompt_key: str                      # e.g. "personal_listener"

       def __init__(self, loader: "PromptLoader", router: "LiteLLMRouter"):
           self._loader, self._router = loader, router
           self._graph = self._build_graph()           # subclass-defined LangGraph

       async def spawn(self, state: AgentState) -> None:
           assert state.org_id is not None, "org_id required (fail-closed)"
           state.prompt_version = self._loader.resolve(self.prompt_key, state.org_id)

       async def process(self, state: AgentState) -> "AgentResult | None":
           if state.token_budget_remaining <= 0:
               return None                              # hard quota: deny-by-default
           return await self._graph.ainvoke(state)

       async def terminate(self, state: AgentState) -> None:
           ...                                          # flush partial worklogs, drop subscriptions
   ```
4. **`PromptLoader`** (`libs/shared/prompts.py`) — `{prompt_key}@{version}`, tenant pin overrides global default, both reviewable in git (the canary mechanism, [08 §3.3 / §6.2](../08-ai-agent-architecture.md)):
   ```python
   class PromptLoader:
       def resolve(self, prompt_key: str, org_id: UUID) -> str:
           return self._tenant_pins.get((org_id, prompt_key)) or self._global_default[prompt_key]
   ```
5. **Stage 1 — windowed batching (`A17`=30s)** (`agent_runtime_service/windowing.py`). Consume `transcript-chunks`; merge Deepgram finals by speaker turn into 30s `TranscriptWindow`s (90 windows/meeting vs 360 raw `transcript_chunks`). Agents run **per window, not per chunk** — this is the 4× unit-boundary reduction before any other lever, and the order is **window-then-gate** ([ADR-0004 §Order reconciliation](../adr/0004-llm-cost-control-pipeline.md)).
6. **Stage 2 — relevance pre-gate (Haiku triage)** (`pipeline/pregate.py`). One cheap Haiku call per window per participant scores relevance 0-1 via `get_user_context`; **~35% pass** (gate-pass 0.35). The 65% that fail are logged and dropped — **no extraction call is ever made**. This is the *pre-gate* (does this matter to this user), distinct from *confidence*.
7. **Stage 3 — tiered routing (LiteLLM)** (`libs/shared/router.py`). The router picks the cheapest model that can do the job: **Haiku** triages/scores; **Sonnet** extracts (Listener/Decision/Action/Question); **Opus** is reserved for confirmed high-stakes reasoning (ProactiveAgent/ContextGap, P10-11 — not this band). Tier mix by volume: Haiku 45% / Sonnet 55% / Opus <1%. Never hardcode a model — route through the tier helper:
   ```python
   class LiteLLMRouter:
       TIER = {"triage": "claude-haiku", "extract": "claude-sonnet", "high_stakes": "claude-opus"}
       async def complete(self, *, tier, template, vars, org_id, max_tokens) -> "Completion":
           model = self.TIER[tier]
           out = await litellm.acompletion(model=model, messages=render(template, vars),
                                            max_tokens=max_tokens,
                                            extra_headers={"anthropic-beta": "prompt-caching"})  # Stage 4
           await emit_usage(org_id, tokens=out.usage, tier=tier_name(model))   # usage-events: agent_token
           return out
   ```
8. **Stage 4 — semantic + Anthropic prompt caching** (`pipeline/cache.py`). Anthropic-prompt-cache the stable prefixes (system prompt, role→topics taxonomy, versioned template, prior-window context) via `cache_control`; route semantically near-duplicate windows through a semantic cache **before** any model call. Applied *after* tiering. Reversible — disabling falls back to uncached pricing, identical output.
9. **Stage 5 — budget gate, inline before every dispatch** (`pipeline/budget.py`). Read the hot mirror `nexus:{org_id}:budget:{period}` (one Redis read, never a Postgres round-trip on the hot path), deny-by-default ([ADR-0004 §Stage 5](../adr/0004-llm-cost-control-pipeline.md), [mytheus-intelligence system.md §4](../../mytheus-intelligence/system.md)):
   ```python
   async def check_budget(org_id: UUID, est_tokens: int) -> Literal["ok", "throttle", "block"]:
       b = await redis.hgetall(f"nexus:{org_id}:budget:{period()}")
       if not b:                                            # mirror miss → read through to Postgres (fail-closed)
           b = await load_budget_from_pg(org_id, period())
       consumed, cap, quota = int(b["tokens_consumed"]), int(b["hard_cap"]), int(b["token_quota"])
       if consumed + est_tokens >= cap:
           await emit("billing-events", quota_exceeded(org_id)); return "block"   # deny-by-default
       if consumed >= quota:
           return "throttle"                                # Haiku-only: suppress Sonnet/Opus
       return "ok"
   ```
   `throttle` forces `tier="triage"` and degrades extraction to silent-log; `block` emits a `notifications` "budget exhausted" sidebar card and stops — captions/transcription continue (durable facts never shed).
10. **Emit `usage-events` for every metered call** — Avro `UsageEvent`, `event_type=agent_token`, `model_tier` set, idempotency `usage:{org_id}:agent_token:{occurred_at}:{call_id}`, partition key `org_id`, via the transactional outbox. This is the only durable side effect the runtime has, and it is what `billing-metering-service` (§3.1) consumes.
11. **Wire `agent-stream`** — bridge LiteLLM streaming token deltas onto `agent-stream` (token-delta envelope) so the UI sees output land under the R5 budget (p50 < 2s) via `realtime-gateway`. Per-user priority queue `nexus:{org_id}:queue:agent:{user_id}` so urgent output pre-empts notes.
12. **The runtime spawner** (`agent_runtime_service/spawner.py`): on a meeting starting (consume `meeting-events`/`meeting-room-events`), spawn **one `PersonalListenerAgent` per participant per meeting**; the per-meeting agents (Decision/Action/Question) are registered but only Listener ships in P7. `spawn` injects `org_id` and the budget mirror into `AgentState`.
13. **Env vars:** `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `LITELLM_CONFIG` (tier map + budget hooks), `REDIS_URL`, `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`, `IDENTITY_GRPC_URL` (`get_user_context`), `OTEL_EXPORTER_OTLP_ENDPOINT`, `SENTRY_DSN`.

**Canonical names:** `agent-runtime-service`, `BaseAgent`, `AgentState`, `PromptLoader`, `LiteLLMRouter`, `agent-results`, `agent-stream`, `usage-events`, `get_user_context`, `check_budget`, `nexus:{org_id}:budget:{period}`.

**Acceptance checks:**
- Agents run per **window**: 360 `transcript_chunks` produce ~90 windows; an extraction call is made only for the ~35% that pass the Haiku pre-gate (count test against gate-pass 0.35).
- The runtime writes **zero** Postgres rows (grep + integration test); all durable output is `agent-results`.
- Every LLM call emits exactly one `usage-events` `agent_token` record with the right `model_tier`; a retried call dedupes (no double-count).
- With the budget mirror in `block`, no model is called and a `notifications` card is emitted; in `throttle`, only Haiku is called.
- Routing never selects Opus for an extraction window (tier-discipline test).

---

#### 3.3 `PersonalListenerAgent` — per participant per meeting, relevance pre-gate, worklogs via `core-api`

**Repo / folder:** `mytheus-intelligence/services/agent-runtime-service/agent_runtime_service/agents/personal_listener_agent.py`; the worklog write path lands in `mytheus-backend/services/core-api/`.

**Steps:**

1. **Implement the agent** subclassing `BaseAgent` ([08 §2.1](../08-ai-agent-architecture.md), [mytheus-intelligence system.md §2](../../mytheus-intelligence/system.md)):
   ```python
   class PersonalListenerAgent(BaseAgent):
       name = "PersonalListenerAgent"
       prompt_key = "personal_listener"

       def _build_graph(self):
           g = StateGraph(AgentState)
           g.add_node("pregate", self._relevance_pregate)   # Haiku: score relevance 0-1 to this user
           g.add_node("extract", self._extract_worklog)      # Sonnet: produce note content + confidence
           g.add_conditional_edges("pregate", self._passes_gate, {True: "extract", False: END})
           g.set_entry_point("pregate"); g.add_edge("extract", END)
           return g.compile()

       async def _relevance_pregate(self, s: AgentState) -> AgentState:
           if await check_budget(s.org_id, est_tokens=300) == "block":
               return s
           s.relevance_score = await self._router_score(s)   # Haiku triage
           return s

       async def _extract_worklog(self, s: AgentState) -> AgentResult:
           note = await self._router.complete(tier="extract", template=f"{self.prompt_key}@{s.prompt_version}",
                                              vars={"window": s.chunk, "ctx": s.context},
                                              org_id=s.org_id, max_tokens=400)
           s.confidence = note.confidence
           return AgentResult(kind="worklog", content=note.text, relevance_score=s.relevance_score,
                              confidence=s.confidence, source_chunk_ids=s.chunk.chunk_ids,
                              created_by_agent=True, user_id=s.user_id, meeting_id=s.meeting_id)
   ```
2. **Confidence gating on emit** (uniform policy, overridable per tenant via `tenant_settings.agent_confidence_overrides`): `>0.7` → emit `agent-results` (My Notes surfaces it); `0.4-0.7` → emit `agent-results` flagged silent-log (persisted to `worklogs`, not surfaced); `<0.4` → drop. The relevance pre-gate decides *whether to extract*; confidence decides *whether to surface*.
3. **`core-api` consumes `agent-results`** and writes `worklogs` (the AI tier never writes Postgres — [mytheus-intelligence system.md §13](../../mytheus-intelligence/system.md)). Migration:
   ```bash
   cd mytheus-backend && uv run alembic revision -m "feat(core-api): add worklogs"
   ```
   ```sql
   CREATE TABLE worklogs (
     id               uuid        NOT NULL DEFAULT gen_random_uuid(),
     org_id           uuid        NOT NULL,
     meeting_id       uuid        NOT NULL REFERENCES meetings(id),
     user_id          uuid        NOT NULL REFERENCES users(id),
     content          text        NOT NULL,
     relevance_score  real,
     confidence       real,
     created_by_agent boolean     NOT NULL DEFAULT true,
     source_chunk_ids uuid[]      NOT NULL DEFAULT '{}',
     surfaced         boolean     NOT NULL DEFAULT false,   -- false for 0.4-0.7 silent-log
     created_at       timestamptz NOT NULL DEFAULT now(),
     deleted_at       timestamptz,
     PRIMARY KEY (id, created_at)
   ) PARTITION BY RANGE (created_at);
   -- monthly partition; index (org_id, user_id, meeting_id); RLS FORCE + tenant_isolation; pg_partman.
   ```
   The `core-api` consumer dedupes on the `agent-results` `idempotency_key`, sets `surfaced` from the confidence band, and exposes `GET /api/v1/meetings/:id/worklogs?user_id=me` for the My Notes panel.
4. **PII redaction before LLM egress** (always-on, per [08 §6.4](../08-ai-agent-architecture.md)): the window is redacted before it leaves for Anthropic/OpenAI via LiteLLM, gated by `tenant_settings.pii_redaction_enabled`. This is the LLM-egress redaction node that Plan 3-4 explicitly deferred to this band — wire it now in `agent-runtime-service` (do not skip).
5. **Register in the spawner** as per-participant. Add to the golden-set eval harness (relevance precision/recall vs role labels) before merge — a prompt change must beat the current default.

**Canonical names:** `PersonalListenerAgent`, `worklogs`, `agent-results`, `get_user_context`, `tenant_settings.agent_confidence_overrides`.

**Acceptance checks:**
- One `PersonalListenerAgent` is spawned per participant per meeting (spawn-count test against the roster).
- A window below the per-tenant relevance threshold makes **no** Sonnet extraction call (pre-gate test).
- A `>0.7` note appears in My Notes; a `0.4-0.7` note lands in `worklogs` with `surfaced=false` and does NOT appear in the sidebar; a `<0.4` note is dropped (zero rows).
- A window containing `jane@acme.com` is redacted before egress when `pii_redaction_enabled=true`.
- `worklogs` is written only by `core-api`; `agent-runtime-service` writes no Postgres.

---

#### 3.4 `web-app` — worklog My Notes panel + agent sidebar shell (P7)

**Repo / folder:** `mytheus-ui/`.

**Steps:**

1. **Agent sidebar shell** in the meeting room (`(app)/meetings/[meetingId]/room`): a collapsible right-rail with calm animation discipline, sections for My Notes, suggestion cards, decision/action cards (filled P8/P9). Pairs with the locked design ([Tech Decisions §2.8](../_canon/06-tech-decisions.md)).
2. **My Notes (worklogs):** subscribe to `agent-results` worklog deltas over the `realtime-gateway` socket (`/v1` namespace) for live append; hydrate history from `GET /api/v1/meetings/:id/worklogs?user_id=me` (through `bff`). Show only `surfaced=true` notes; silent-logged notes are not rendered.
3. **Usage / budget console section** (tenant-admin, RBAC `admin`): read `/usage` (aggregates + `agent_budgets`); show token burn vs quota with the Haiku/Sonnet/Opus split and the 80/95% alert state; a `blocked` Free tenant sees an upgrade prompt.
4. **Streaming token deltas:** render `agent-stream` token deltas into the live note for the calm "typing" effect; finals settle from `agent-results`.

**Acceptance checks:** a participant sees their own role-relevant notes stream in live; another participant's notes never appear in their sidebar; the Usage panel shows real burn and the alert state.

---

### PHASE P8 — Decisions / Actions + Knowledge Graph (W21-23)

#### 3.5 `DecisionAgent` + `ActionAgent` — candidates only (no silent commits)

**Repo / folder:** `mytheus-intelligence/services/agent-runtime-service/agent_runtime_service/agents/{decision_agent,action_agent}.py`.

**Steps:**

1. **`DecisionAgent`** ([08 §2.1](../08-ai-agent-architecture.md)): on a 30s window (post pre-gate), Sonnet-extract decision candidates — `rationale`, stakeholders, `confidence`. **Never silently commits**: emit a *decision candidate* on `agent-results` (kind=`decision_candidate`) that surfaces a confirm/edit/dismiss card. The `decisions` topic event and the AGE `PRODUCED` edge happen ONLY on user confirmation (§3.6).
2. **`ActionAgent`**: same window, extract action candidates — `assignee_id`, `due_date`, dependencies, `confidence`. Emit `action_candidate` on `agent-results` (new-task toast). On confirmation, the `actions` event and the AGE `ASSIGNED_TO` edge are created.
3. **Both run through the full pipeline** — pre-gate, tiered routing (Sonnet extraction), caching, budget gate — and inherit confidence gating. A candidate is surfaced (`>0.7`), silent-logged (`0.4-0.7`), or dropped (`<0.4`); a surfaced candidate is still *unconfirmed* until the user acts.
4. **Register both as per-meeting agents** (fan reasoning across the roster, not per-participant). Add to the eval harness (extraction F1 + confidence calibration via Brier; F1 ≥ baseline, calibration not worse) before merge.
5. **Tiered-routing reconciliation:** extraction defaults to Sonnet, never Haiku (under-routing degrades decision/action precision). Opus is not used by these agents.

**Canonical names:** `DecisionAgent`, `ActionAgent`, `agent-results`, `AgentResult`.

**Acceptance checks:**
- A decision/action candidate appears as a card; **no** `decisions`/`actions` topic event and **no** AGE edge exist until the user confirms (the load-bearing "no silent commits" test).
- Extraction always routes Sonnet; a candidate below 0.4 is dropped.

---

#### 3.6 `core-api` — the SOLE confirm-only producer of `decisions` / `actions`

**Repo / folder:** `mytheus-backend/services/core-api/`.

**Why here:** the [Service Catalog](../_canon/01-service-catalog.md) lists `agent-runtime-service` among producers of `decisions`/`actions`, but per [08 §2.2](../08-ai-agent-architecture.md) the authoritative producer **on confirmation** is `core-api` via its transactional outbox. The agent originates the *candidate*; `core-api` publishes the confirmed event. This is the one place the truth is committed.

**Steps:**

1. **Migrations — `decisions` and `action_items`** ([05 §2](../05-data-architecture.md)):
   ```sql
   CREATE TABLE decisions (
     id            uuid        NOT NULL DEFAULT gen_random_uuid(),
     org_id        uuid        NOT NULL,
     meeting_id    uuid        NOT NULL REFERENCES meetings(id),
     content       text        NOT NULL,
     rationale     text,
     confidence    real        NOT NULL,
     status        text        NOT NULL DEFAULT 'proposed',   -- proposed|confirmed|superseded|dismissed
     confirmed_by  uuid REFERENCES users(id),
     supersedes    uuid REFERENCES decisions(id),             -- FOLLOWS_FROM lineage source
     created_at    timestamptz NOT NULL DEFAULT now(),
     deleted_at    timestamptz,
     PRIMARY KEY (id)
   );  -- index (org_id, meeting_id), (org_id, status); RLS FORCE + tenant_isolation.

   CREATE TABLE action_items (
     id           uuid        NOT NULL DEFAULT gen_random_uuid(),
     org_id       uuid        NOT NULL,
     meeting_id   uuid        NOT NULL REFERENCES meetings(id),
     content      text        NOT NULL,
     assignee_id  uuid REFERENCES users(id),
     due_date     date,
     status       text        NOT NULL DEFAULT 'proposed',    -- proposed|confirmed|done|dismissed
     confidence   real        NOT NULL,
     confirmed_by uuid REFERENCES users(id),
     created_at   timestamptz NOT NULL DEFAULT now(),
     deleted_at   timestamptz,
     PRIMARY KEY (id)
   );  -- index (org_id, assignee_id, status), (org_id, due_date); RLS FORCE + tenant_isolation.
   ```
2. **`agent_suggestions` migration** (also used by QuestionAgent in P9; create now):
   ```sql
   CREATE TABLE agent_suggestions (
     id          uuid        NOT NULL DEFAULT gen_random_uuid(),
     org_id      uuid        NOT NULL,
     user_id     uuid        NOT NULL REFERENCES users(id),
     meeting_id  uuid        NOT NULL REFERENCES meetings(id),
     content     text        NOT NULL,
     priority    integer     NOT NULL,                        -- 1-3
     status      text        NOT NULL DEFAULT 'open',         -- open|asked|dismissed|saved
     created_at  timestamptz NOT NULL DEFAULT now(),
     deleted_at  timestamptz,
     PRIMARY KEY (id, created_at)
   ) PARTITION BY RANGE (created_at);   -- monthly partition; index (org_id, user_id, status); RLS FORCE.
   ```
3. **Persist candidates as `proposed`** when `core-api` consumes `agent-results` decision/action candidates (so an unconfirmed claim has a row to confirm/edit/dismiss against, but `status='proposed'` and **no topic event**).
4. **Confirm endpoints — the SOLE producers** of the topic events:
   ```python
   @router.post("/api/v1/decisions/{decision_id}/confirm")
   async def confirm_decision(decision_id: UUID, body: ConfirmBody, ctx: TenantCtx = Depends(tenant_ctx)):
       async with tenant_session(ctx.org_id) as db, outbox(db) as out:
           d = await db.scalar(select(Decision).where(Decision.id == decision_id))   # RLS-scoped
           d.status = "confirmed"; d.confirmed_by = ctx.user_id
           if body.supersedes: d.supersedes = body.supersedes                        # FOLLOWS_FROM source
           enqueue_event(out, "decisions", DecisionConfirmed(d))    # → knowledge-graph-service + Qdrant
       return d
   ```
   The same pattern for `POST /api/v1/actions/{id}/confirm` → `actions` event. Edit re-writes the row pre-confirm; dismiss sets `status='dismissed'` and emits nothing.
5. **The `decisions` / `actions` events drive two consumers:** `knowledge-graph-service` (edges, §3.7) and `embedding-service` (the `decisions__{org_id}` Qdrant collection, embedded on confirm — read later by ProactiveAgent in P11 and the RAG graph/vector arm).
6. **My Tasks read model:** `GET /api/v1/actions?assignee_id=me&status=confirmed` over the `(org_id, assignee_id, status)` index.

**Canonical names:** `core-api`, `decisions`, `action_items`, `agent_suggestions`, `decisions` (topic), `actions` (topic), `Decision`, `Action`.

**Acceptance checks:**
- A `decisions`/`actions` topic event is produced **only** by a `core-api` confirm endpoint, never by `agent-runtime-service` (producer grep + integration test).
- Confirm writes the row and emits exactly once (outbox); a dismissed candidate emits nothing and leaves no confirmed row.
- `supersedes` is captured so the graph can build `FOLLOWS_FROM`.

---

#### 3.7 `knowledge-graph-service` — AGE `nexus_knowledge`, edges on confirmation only, `$org_id` on every Cypher

**Repo / folder:** `mytheus-intelligence/services/knowledge-graph-service/`. **The only writer of `nexus_knowledge`.**

**Steps:**

1. **Scaffold + AGE-enabled Postgres.** AGE is co-located in the system-of-record Postgres ([ADR-0005](../adr/0005-graph-store-age-with-migration.md)). Ensure `shared_preload_libraries=age` (or self-host — see DECISION-REQUIRED #3 / R1). Migration to create the graph + indexes:
   ```sql
   CREATE EXTENSION IF NOT EXISTS age;
   LOAD 'age'; SET search_path = ag_catalog, "$user", public;
   SELECT create_graph('nexus_knowledge');
   -- index the org_id agtype property on every label so the partition predicate is index-backed:
   CREATE INDEX person_org_idx  ON nexus_knowledge."Person"
     USING btree (agtype_access_operator(properties, '"org_id"'));
   CREATE INDEX meeting_org_idx ON nexus_knowledge."Meeting"
     USING btree (agtype_access_operator(properties, '"org_id"'));
   -- repeat for Decision, Action, Topic and the edge tables.
   ```
2. **Consume confirmed `decisions` / `actions`** (from `core-api`) and write edges — **on confirmation only**, never speculatively ([ADR-0005 D1](../adr/0005-graph-store-age-with-migration.md), [mytheus-intelligence CLAUDE.md invariants](../../mytheus-intelligence/CLAUDE.md)):
   ```python
   async def on_decision_confirmed(evt: DecisionConfirmed):
       org_id = evt.header.org_id
       if await inbox_seen(org_id, "decisions", evt.header.idempotency_key):
           return                                              # idempotent consume
       await cypher(org_id, """
         MATCH (m:Meeting {meeting_id: $meeting_id, org_id: $org_id})
         MERGE (d:Decision {decision_id: $decision_id, org_id: $org_id})
           SET d.status = $status, d.confidence = $confidence
         MERGE (d)-[:PRODUCED]->(m)
       """, decision_id=evt.decision_id, meeting_id=evt.meeting_id,
            status=evt.status, confidence=evt.confidence, org_id=org_id)
       if evt.supersedes:                                       # decision lineage
           await cypher(org_id, """
             MATCH (d:Decision {decision_id: $new, org_id: $org_id})
             MATCH (p:Decision {decision_id: $old, org_id: $org_id})
             MERGE (d)-[:FOLLOWS_FROM]->(p)
           """, new=evt.decision_id, old=evt.supersedes, org_id=org_id)
       await emit("graph-events", graph_edge_written(org_id, "PRODUCED", evt.decision_id))
   ```
   `on_action_confirmed` writes `(:Action)-[:PRODUCED]->(:Meeting)` and `(:Action)-[:ASSIGNED_TO]->(:Person)`; topic-extraction links `RELATES_TO` to `Topic` from the role→topics taxonomy. `ATTENDED` (`Person`→`Meeting`) is written when a participant joins.
3. **`$org_id` on EVERY Cypher template** — defense-in-depth on top of tenant-partitioned topics. The four named templates (`POST /graph/query/{template}`, internal gRPC) are bounded-depth and single-org ([ADR-0005 D2/D3](../adr/0005-graph-store-age-with-migration.md)): *related decisions for a topic*, *decisions by a person*, *actions blocked on a decision*, *decision lineage*:
   ```cypher
   -- decision lineage (FOLLOWS_FROM*1..10), org-scoped — backs the lineage UI + RAG graph arm.
   MATCH (d:Decision {org_id: $org_id})-[:FOLLOWS_FROM*1..10]->(prior:Decision {org_id: $org_id})
   WHERE d.decision_id = $decision_id AND prior.deleted_at IS NULL
   RETURN prior
   ```
   `knowledge-graph-service` rejects any ad-hoc `/graph/cypher` that does not bind `$org_id` (fail-closed).
4. **Soft delete** via a `deleted_at` property; templates filter it. Cross-store RTBF deletes graph nodes/edges by `org_id` (the shared Temporal saga step).
5. **Query budget:** scale on Cypher req/s with read replicas; graph queries backing live agent features fit inside the agent latency budget (p50 < 2s). Templates are single-org so a noisy tenant cannot blow the budget for everyone.
6. **gRPC surface** in `proto/knowledge_graph.proto`; `org_id` rides in gRPC metadata on every hop. `rag-service` (graph arm, P9) and the `core-api` read path (lineage UI) are the read consumers.
7. **Env vars:** `POSTGRES_DSN` (AGE `nexus_knowledge`), `REDIS_URL`, `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`, `KG_GRPC_BIND`, `OTEL_EXPORTER_OTLP_ENDPOINT`.

**Canonical names:** `knowledge-graph-service`, `nexus_knowledge`, `Person`/`Meeting`/`Decision`/`Action`/`Topic`, `ATTENDED`/`PRODUCED`/`ASSIGNED_TO`/`RELATES_TO`/`FOLLOWS_FROM`, `graph-events`, `POST /graph/query/{template}`.

**Acceptance checks:**
- An edge exists **only** after a `core-api` confirm event — an unconfirmed candidate writes nothing to `nexus_knowledge`.
- Every template binds `$org_id`; a traversal cannot return another tenant's node (cross-tenant isolation test); `/graph/cypher` without `$org_id` is rejected.
- A `FOLLOWS_FROM` chain is created when a confirmed decision supersedes another; the lineage template walks it bounded-depth.
- A redelivered `decisions` event is a no-op (inbox dedupe); the graph is fully rebuildable from confirmed rows (edge-on-confirmation invariant).

---

#### 3.8 `web-app` — decision/action confirm cards, My Tasks, decision lineage (P8)

**Repo / folder:** `mytheus-ui/`.

**Steps:**

1. **Confirm/edit/dismiss cards** in the agent sidebar: render decision/action candidates from `agent-results` over the socket; Confirm calls `POST /api/v1/{decisions,actions}/:id/confirm` (through `bff`), Edit re-writes pre-confirm, Dismiss closes the card. Only Confirm produces the topic event/edge.
2. **My Tasks** `(app)/tasks`: `GET /api/v1/actions?assignee_id=me&status=confirmed`; group by due date; mark done.
3. **Decision lineage view** `(app)/meetings/[meetingId]/decisions/[id]`: call the `decision lineage` template (via `bff` → `knowledge-graph-service`) and render the `FOLLOWS_FROM` chain (superseded → current).

**Acceptance checks:** confirming a card creates the row + edge + topic event; dismissing creates nothing durable; the lineage view shows the supersession chain.

---

### PHASE P9 — RAG & Question Agent (W24-26)

#### 3.9 `rag-service` — `expand → retrieve → rerank → assemble → cite` over gRPC, RRF, citations, answer cache

**Repo / folder:** `mytheus-intelligence/services/rag-service/`. The **sync orchestrator** of hybrid retrieval — holds no model in-process; reaches `embedding-service` / `reranker-service` / `search-service` / `knowledge-graph-service` over the internal gRPC mesh ([09](../09-rag-and-inference-serving.md), [mytheus-intelligence system.md §5](../../mytheus-intelligence/system.md)).

**Steps:**

1. **Scaffold + gRPC stubs.** Define `proto/rag.proto`, `proto/embedding.proto`, `proto/reranker.proto`, `proto/knowledge_graph.proto`, `proto/search.proto`; regenerate. Every request carries `org_id` in gRPC metadata.
   ```bash
   cd mytheus-intelligence/services/rag-service
   pip install grpcio grpcio-tools redis litellm fastapi "uvicorn[standard]"
   python -m grpc_tools.protoc -I../../proto --python_out=. --grpc_python_out=. ../../proto/*.proto
   ```
2. **`POST /api/rag/query`** (owned by `rag-service`) — `user_id` from JWT, `org_id` injected. Answer cache check first:
   ```python
   async def query(q: str, ctx: TenantCtx) -> RagAnswer:
       key = f"nexus:{ctx.org_id}:rag:{ctx.user_id}:{sha256(q)}"
       if hit := await redis.get(key):                          # 10-min TTL answer cache
           return RagAnswer.parse_raw(hit)
       variants = await self.expand(q, ctx)                     # Stage 1
       cands    = await self.retrieve(variants, ctx)            # Stage 2 (3 arms, parallel) + 3a RRF
       top10    = await self.rerank(q, cands, ctx)              # Stage 3b cross-encoder
       packed   = self.assemble(top10, ctx)                     # Stage 4 token-budget packing
       answer   = self.cite(packed)                             # Stage 5 (uncited dropped)
       await emit_usage(ctx.org_id, tokens=expand_tokens, tier="haiku")   # usage-events: agent_token
       await redis.set(key, answer.json(), ex=600)
       return answer
   ```
3. **Stage 1 — expand** (LiteLLM Haiku): 2-3 query variants via `get_user_context` role-tuning. **Skip when the tenant is `throttled`/`blocked`** — RAG degrades to single-query recall rather than failing ([09 §3.1](../09-rag-and-inference-serving.md)).
4. **Stage 2 — hybrid retrieve, three arms IN PARALLEL** per variant:
   - **Vector:** `embedding-service POST /embed` (cached `nexus:{org_id}:qembed:{query_hash}`) → Qdrant search over `transcript_chunks__{org_id}` / `document_chunks__{org_id}` with the **mandatory `org_id` payload filter** (defense-in-depth on top of the per-tenant collection).
   - **FTS:** `search-service /search` (Postgres `tsvector` over `transcript_chunks` + `chat_messages`, RLS by `org_id`).
   - **Graph:** `knowledge-graph-service` `RELATES_TO` traversal + `FOLLOWS_FROM` lineage, every Cypher parameterized by `$org_id`.
5. **Stage 3a — RRF fusion** (`k=60`) → top-50: `RRF(d) = Σ 1/(k + rank_list(d))`. **Stage 3b — rerank**: `reranker-service` BGE-reranker-base cross-encoder scores the top-50 `(query, passage)` pairs → **top-10** (rerank against the user's actual query, not variants).
6. **Stage 4 — assemble** with a token budget (default **3,000 tokens**, configurable per surface): pack in reranked order until the cap, dedupe near-duplicates, cap per-source contribution (diversity). The cap **tightens automatically** as a tenant nears its `agent_budgets` hard quota — every assembled token is metered.
7. **Stage 5 — cite**: bind every passage to provenance — transcript → `meeting_id`/`chunk_id`/`speaker`; document → `document_id`/`element_id`. **A passage that cannot bind a citation is dropped before return** — uncited text never reaches the answer. Emit `usage-events` for the metered tokens.
8. **Answer cache** `nexus:{org_id}:rag:{user_id}:{query_hash}`, 10-min TTL, per-user (role-tuned expansion/graph traversal) and tenant-namespaced (clean RTBF/plan-change flush).
9. **Consumers wired:** the context-panel UI (direct retrieval), and the pre-built hook for `ContextGapAgent`'s RAG fallback (P10 — not this band). `QuestionAgent` (§3.10) uses retrieval to ground questions.
10. **SLOs:** warm p95 < 500 ms, cold p95 < 1.5 s; the cross-encoder is the cold-path long pole, so `reranker-service` keeps a warm-replica floor (Plan 5-6). Keep arms parallel; `rag-service` adds no in-process model.
11. **Env vars:** `EMBEDDING_GRPC_URL`, `RERANKER_GRPC_URL`, `SEARCH_GRPC_URL`, `KG_GRPC_URL`, `REDIS_URL`, `ANTHROPIC_API_KEY`, `OTEL_EXPORTER_OTLP_ENDPOINT`.

**Canonical names:** `rag-service`, `RAGService`, `POST /api/rag/query`, `embedding-service`, `reranker-service`, `search-service`, `knowledge-graph-service`, `nexus:{org_id}:rag:{user_id}:{query_hash}`, `nexus:{org_id}:qembed:{query_hash}`, `transcript_chunks__{org_id}`, `document_chunks__{org_id}`.

**Acceptance checks:**
- Every retrieval flows all five stages; there is no path that skips citation tracking; an uncited passage is dropped before return.
- The three arms run in parallel (trace test: stage bounded by slowest arm, not the sum); RRF produces top-50, rerank top-10.
- A warm repeat query returns from cache in ~one Redis GET (cache-hit test); a cache miss with the same words but different `user_id` may still hit the query-embed cache.
- A throttled tenant's query skips expansion and still returns cited answers (degrade-not-fail test).
- The `org_id` payload filter is applied on Qdrant even though the collection is per-tenant (belt-and-suspenders test); no cross-tenant passage is ever returned.

---

#### 3.10 `QuestionAgent` — real-time suggestion cards, `agent_suggestions` priority 1-3

**Repo / folder:** `mytheus-intelligence/services/agent-runtime-service/agent_runtime_service/agents/question_agent.py`.

**Steps:**

1. **Implement** ([08 §2.1](../08-ai-agent-architecture.md)): on a 30s window + a context-gap heuristic, Sonnet-propose role-specific questions with `priority` 1-3 and `status` in {Ask, Dismiss, Save-for-later}. May ground a question with a quick `rag-service` lookup. Emit `agent_suggestions` via `agent-results`.
2. **`core-api` persists** the suggestion (`agent_suggestions`, migrated in §3.6) with `priority`/`status`; Save-for-later persists into the **next** meeting (the `(org_id, user_id, status='saved')` index seeds the next session's sidebar).
3. **Confidence + budget** as for all agents; runs per-meeting, Sonnet tier. Add to the eval harness (question usefulness / Ask-rate proxy ≥ baseline).
4. **Suggestion card UI** (`web-app` agent sidebar): render priority 1-3 cards with Ask / Dismiss / Save; Ask flips `status='asked'`, Save persists into the next meeting's open queue.

**Canonical names:** `QuestionAgent`, `agent_suggestions`, `agent-results`.

**Acceptance checks:**
- A suggestion card appears with `priority ∈ {1,2,3}`; Save-for-later persists and re-appears in the user's next meeting.
- The agent runs through the budget gate and confidence gate; a `<0.4` suggestion is dropped.

---

## 4. Data stores stood up in this band

Per the roadmap data-store schedule ([22-roadmap-and-phasing.md §3.1](../22-roadmap-and-phasing.md)) and [05 Data Architecture](../05-data-architecture.md):

| Store | What lands here | Owner | Phase |
|---|---|---|---|
| PostgreSQL `plans` | global Free/Pro/Enterprise catalog (no `org_id`, versioned) | `billing-metering-service` | P7 |
| PostgreSQL `subscriptions`, `invoices` | per-tenant subscription mirror + immutable invoices | `billing-metering-service` | P7 |
| PostgreSQL `usage_events` | append-only metering feed, **daily** range partition by `occurred_at`, UNIQUE `(org_id, idempotency_key)`, `pg_partman` | `billing-metering-service` | P7 |
| PostgreSQL `usage_aggregates` | nightly rollup, upsert `(org_id, dimension, billing_period)` | `billing-metering-service` | P7 |
| PostgreSQL `agent_budgets`, `quotas` | per-tenant per-`YYYY-MM` token quota + state machine | `billing-metering-service` | P7 |
| PostgreSQL `worklogs` | PersonalListener notes, monthly partition, `surfaced` flag | `core-api` | P7 |
| Redis `nexus:{org_id}:budget:{period}` | hot mirror of `agent_budgets.tokens_consumed`, TTL period-end | `billing-metering-service` | P7 |
| PostgreSQL `decisions`, `action_items` | confirmed (and proposed) decisions/actions | `core-api` | P8 |
| PostgreSQL `agent_suggestions` | QuestionAgent output, monthly partition | `core-api` | P8/P9 |
| AGE `nexus_knowledge` | org-partitioned graph; edges on confirmation only; per-label `org_id` index | `knowledge-graph-service` | P8 |
| Qdrant `decisions__{org_id}` | confirmed-decision vectors (embedded on confirm), int8 quantized | `embedding-service` (writer) | P8 |
| Redis `nexus:{org_id}:rag:{user_id}:{query_hash}` | RAG answer cache, 10-min TTL | `rag-service` | P9 |

Read but not stood up here (Plan 5-6): Qdrant `transcript_chunks__{org_id}` / `document_chunks__{org_id}`, `nexus:{org_id}:qembed:{query_hash}`, `nexus:{org_id}:user_ctx:{user_id}`, the FTS GIN indexes. Stripe webhook/dunning, `tenant-events`, the storage-sweep job, and the cold transcript archive are wired in their own bands.

---

## 5. Events / topics introduced

All Avro + Schema Registry, subject `{topic}-value`, BACKWARD, partition key `org_id`, every topic paired with a `<topic>.dlq` (14-day retention). Every value opens with `EventEnvelopeHeader`.

| Topic | Producer | Consumers (this band) | Avro schema | Retention | Idempotency key |
|---|---|---|---|---|---|
| `agent-results` | `agent-runtime-service` | `core-api` (worklogs / candidates / suggestions) | `AgentResult` (FROZEN) | 7 days | per result |
| `agent-stream` | `agent-runtime-service` (LiteLLM bridge) | `realtime-gateway` → UI | token-delta envelope † | 1 hour | n/a (ephemeral deltas) |
| `usage-events` | `agent-runtime-service`, `rag-service` (**`agent_token` only**) | `billing-metering-service` | `UsageEvent` † | 7 days | `usage:{org_id}:{event_type}:{occurred_at}:{source_id}` |
| `billing-events` | `billing-metering-service` | `notification-service`, `workflow-service` | `BillingEvent` † | 30 days | per state change |
| `decisions` | **`core-api` (confirm only)** | `knowledge-graph-service`, `embedding-service` | `Decision` (FROZEN) | 7 days | per confirmation |
| `actions` | **`core-api` (confirm only)** | `knowledge-graph-service`, `workflow-service` (follow-up, later) | `Action` (FROZEN) | 7 days | per confirmation |
| `graph-events` | `knowledge-graph-service` | `admin-ops-service` (ops dashboards) | `GraphEvent` † | 7 days | per edge write |
| `agent-requests` / `agent-responses` | `agent-runtime-service` (`ContextGapAgent`, P10) | — (wired no-op-tolerant; live P10) | `AgentRequest` / `AgentResponse` (FROZEN) | 1 hour | `correlation_id` |
| `notifications` | `agent-runtime-service`, `billing-metering-service` | `notification-service` → `realtime-gateway` | `Notification` (FROZEN) | 1 hour | per notification |

† `UsageEvent`, `BillingEvent`, `GraphEvent`, and the `agent-stream` token-delta envelope are net-new schema **names** from [07](../07-event-architecture.md) — usable but **not frozen canon** (only `TranscriptChunk`, `AgentResult`, `Decision`, `Action`, `Notification`, `AgentRequest`, `AgentResponse` are frozen). `AgentResult`, `Decision`, `Action`, `Notification`, `AgentRequest`, `AgentResponse` ARE frozen.

**Producer notes:** only `agent_token` traverses `usage-events` (sole producers `agent-runtime-service` + `rag-service`); `meeting_minute`/`transcription_second`/`storage_gb` are derived inside `billing-metering-service` and written directly as `usage_events` rows ([ADR-0011 §3.1](../adr/0011-billing-and-metering.md)) — see DECISION-REQUIRED #2. `decisions`/`actions` are produced ONLY by `core-api` on confirmation, never by `agent-runtime-service` ([08 §2.2](../08-ai-agent-architecture.md)).

---

## 6. SaaS overlay / preconditions landing here

The dominant overlay this band is **billing + budgets + the R4 enforcement arm**, plus the **dedicated-inference cost center** going live for RAG, plus the **LLM-egress PII redaction node** deferred from Plan 3-4.

1. **Billing + metering + per-tenant token budgets (P7, BUILD FIRST).** `usage-events → usage_aggregates → quotas/agent_budgets`; the Redis hot mirror; the `ok → throttled → blocked` state machine; deny-by-default at `hard_cap`; the 100% quota-breach enforcement SLO. Ships alert-only first, then per-cohort enforcement. This is the billing-safety backstop that lets NEXUS sell to A4=3,000 tenants without a runaway bill. Durable facts are **never** shed for quota — only discretionary LLM work degrades.
2. **Dedicated GPU inference cost center (P9).** `rag-service` orchestrates over `embedding-service` / `reranker-service` (off the API tier, Plan 5-6) — the inference tier scales and bills independently of CRUD, which is what makes `agent_budgets` and the `C2` model meaningful.
3. **LLM-egress PII redaction (P7).** Windows are redacted before they leave for Anthropic/OpenAI via LiteLLM, gated by `tenant_settings.pii_redaction_enabled` — the always-on egress node Plan 3-4 flagged as deferred to the agent/RAG band. Wire it now; do not skip.
4. **Security shift-left.** Agent-to-agent authz (deny-by-default, org+role scope, audited) is built into the runtime contract (`AgentState.agent_id`, `org_id` fail-closed on spawn) even though `ContextGapAgent` ships in P10 — the boundary is established here so P10 inherits it. An authz check is reviewed in every PR.

Compliance gate (subprocessor DPA/SCC) was green from Plan 3-4 (Deepgram); Anthropic/OpenAI subprocessor DPAs + SCCs must be on file before the first LLM egress in P7 — verify the subprocessor register lists them with the zero-retention/no-training flags.

---

## 7. Mobile (mytheus-mobile) work in this band

`mytheus-mobile` today is an off-spec single-device "Meeting Pod" prototype on hardcoded `src/data/mock.ts`, with mock recap/insights screens. This band makes the phone a **first-class agent client**: real worklogs, tasks, decisions lineage, and RAG — wired to `bff` (REST) and `agent-results` over `realtime-gateway` (Socket.io). Keep the good UI components and design tokens; replace the mock.

**Threads into:** P7 (My Notes), P8 (My Tasks + decisions lineage), P9 (Ask / RAG).

**Steps:**

1. **Remove the recap/insights mock screens.** Delete their dependence on `src/data/mock.ts` (the live-session mock was already removed in Plan 3-4; this band removes the agent-output mocks). Replace with real agent output.
2. **Agent sidebar / My Notes (P7):** in the live-session screen, render `worklogs` — hydrate from `GET /api/v1/meetings/:id/worklogs?user_id=me` (through `api-gateway` → `bff`), append live from `agent-results` worklog deltas over the `realtime-gateway` `/v1` socket. Show only `surfaced=true`; stream `agent-stream` token deltas for the calm typing effect. `org_id` is server-derived (never client-set).
3. **Decision/action confirm cards (P8):** render decision/action candidates from `agent-results`; Confirm calls `POST /api/v1/{decisions,actions}/:id/confirm`, Dismiss closes the card. Only Confirm commits.
4. **My Tasks (P8):** a tab reading `GET /api/v1/actions?assignee_id=me&status=confirmed`; group by due date; mark done.
5. **Decisions lineage (P8):** a decision detail screen calling the `decision lineage` template (via `bff` → `knowledge-graph-service`); render the `FOLLOWS_FROM` chain.
6. **Ask (RAG) screen (P9):** a search/ask surface posting `POST /api/rag/query` (through `bff`), rendering cited passages as source cards; a transcript citation jumps to the timestamp, a document citation opens the element. Honor the warm < 500 ms budget; show a calm loading state on cold.
7. **Usage awareness:** when the tenant is `throttled`/`blocked`, surface the same upgrade/degradation messaging the web console shows (read from `bff`); the phone never bypasses the budget.
8. **Env:** `EXPO_PUBLIC_API_GATEWAY_URL`, `EXPO_PUBLIC_REALTIME_WS_URL`, `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY`, `EXPO_PUBLIC_SENTRY_DSN`, `EXPO_PUBLIC_ENV`. No LLM/inference keys ship.

**Constraints (do not break):** contract-compatibility with `mytheus-ui` (same REST endpoints, same Socket.io `/v1` event names/payloads, same RAG response contract); `org_id` never client-set; confidence/confirmation/citation rules identical to web (only `surfaced` worklogs render; only Confirm commits; uncited RAG passages never shown).

**Acceptance checks:** the phone shows real My Notes streaming live, confirm cards that commit decisions/actions, My Tasks, a `FOLLOWS_FROM` lineage view, and a cited RAG answer from the Ask screen — with **zero** values from `src/data/mock.ts` on these screens, and the budget state honored.

---

## 8. DECISION-REQUIRED callouts

> ⚠️ **DECISION-REQUIRED #1 — PLAN LIMITS per-seat vs per-tenant (BLOCKS P7 metering go-live).**
> `plans.included_meeting_minutes` and `plans.included_agent_tokens` are frozen as `plans` columns in [Data Model §2.2](../_canon/02-data-model.md), but the canon does **NOT** freeze **per-seat vs per-tenant** semantics. [SaaS Business §3.1](../18-saas-business-architecture.md) interprets them **per seat / mo** (to align with the per-seat margin model); [Cost & FinOps §5.1](../17-cost-and-finops.md) notes a per-tenant pooling read is equally valid. This directly changes the `agent_budgets` quota math: `token_quota`/`hard_cap` are either `included_agent_tokens × subscriptions.seats` (per-seat) or `included_agent_tokens` flat (per-tenant).
> **Options:** (a) **Per-seat** — `token_quota = included_agent_tokens × seats`; aligns to the Pro $25/seat margin model; a tenant adding seats raises its budget. (b) **Per-tenant pooled** — flat `included_agent_tokens` per tenant regardless of seats; simpler, but seat growth does not raise the budget and the margin model must be re-derived. (c) Per-seat for Free/Pro, contracted-pool for Enterprise.
> **Who decides:** Pricing/GTM lead + Canon owners (must amend [Data Model §2.2](../_canon/02-data-model.md) to freeze the semantics, with an ADR). **What it blocks:** the `agent_budgets` provisioning math (§3.1 step 9) and therefore metering go-live — you cannot set a quota you cannot compute. **Resolve before P7 enforcement flips on (W18).**

> ⚠️ **DECISION-REQUIRED #2 — `usage_events` derived-dimension producer ownership (BLOCKS P7 aggregation of three of four dimensions).**
> [ADR-0011 §3.1](../adr/0011-billing-and-metering.md) reconciles topic vs table: **only `agent_token` traverses the `usage-events` topic** (sole producers `agent-runtime-service` + `rag-service`); `meeting_minute`, `transcription_second`, `storage_gb` are **derived inside `billing-metering-service`** and written directly as `usage_events` rows. But the Service Catalog does not assign named producers for those three derivations, and Plan 3-4 left this open (its DECISION-REQUIRED #3). The derivation source must be pinned: `meeting_minute`/`transcription_second` from `meetings` + `transcript_chunks` durations, `storage_gb` from a nightly sweep — but **which component owns each** and whether any needs a Service-Catalog producer edge is unresolved.
> **Options:** (a) **`billing-metering-service` derives all three internally** (ADR-0011's reading) — no new topic producer; requires the derivation jobs (§3.1 step 5) + the `workflow-service` storage-sweep schedule. (b) **`meeting-service`/`transcription-service` emit onto the topic** — requires amending the Service Catalog §3/§5 producer map first (canon edit + ADR). (c) Hybrid: derive minutes/seconds in `billing-metering-service`, add a dedicated `storage_gb` sweep producer (catalog-amended).
> **Who decides:** Billing lead + Canon owners. **What it blocks:** P7 aggregation/quota for `meeting_minute`/`transcription_second`/`storage_gb`; if unresolved, only `agent_token` meters and minutes/storage bill nothing. **Resolve before P7 metering aggregates them (W19).**

> ⚠️ **DECISION-REQUIRED #3 — R1 DESCOPE CHECKPOINT: AGE on managed Supabase (BLOCKS the P8 knowledge-graph build path if AGE is gated).**
> AGE on managed Supabase is genuinely awkward ([ADR-0005 Context](../adr/0005-graph-store-age-with-migration.md), Risk R1): the extension lags Postgres majors and managed providers gate `shared_preload_libraries`. If `CREATE EXTENSION age` / `shared_preload_libraries=age` cannot be enabled on the managed instance, §3.7 cannot run as written.
> **Options:** (a) **Stay on AGE** if the managed instance permits the extension — the default; no app change. (b) **Self-host Postgres + AGE** for the graph — removes the managed-extension gate, adds Postgres ops. (c) **Neo4j Aura / Amazon Neptune** — lift the same 5 labels / 5 edges / 4 templates behind the unchanged `POST /graph/query/{template}` API. (d) **Relational materialized-view fallback** — edge tables + recursive CTEs answering the four templates; lowest novelty, predictable SQL, sacrifices ad-hoc Cypher. The schema is store-agnostic by design so any exit is API-shielded and per-tenant.
> **Who decides:** Data/Platform lead (assign a named owner) + Canon owners — a **go/no-go at the START of P8 (W21)**. **What it blocks:** the `knowledge-graph-service` writer (§3.7) and therefore the lineage UI and the RAG graph arm. **Resolve at the P8 entry gate; record the decision and owner in the Risk Register.**

---

## 9. Risk checkpoints

- **R4 (LLM cost spiral) — the headline risk of this band.** Checkpoint: `billing-metering-service` is live (alert-only at minimum) **before** the first agent LLM call (§3.1 precedes §3.2); the budget mirror gates every call inline (`check_budget` is a Redis read, not a Postgres round-trip); the pipeline runs window-then-gate at gate-pass 0.35 (~1,206 calls/meeting, not the naive ~8,640); deny-by-default at `hard_cap` is provable; the 100% quota-breach enforcement SLO holds. The `usage-events` metering is exactly-once-in-effect (idempotency key + inbox dedupe) so the hard cap is a margin guarantee, not an approximation.
- **R1 (AGE on managed Supabase) at P8.** Checkpoint: the §8 #3 go/no-go is recorded with a named owner before §3.7 starts; the graph schema (5 labels, 5 edges, 4 templates) is expressed store-agnostically so the Neptune/Aura or relational-MV exit is a driver swap behind `POST /graph/query/{template}`; edges are re-derivable from confirmed `decisions`/`action_items` rows (edge-on-confirmation), so the graph can be rebuilt from relational truth on any migration.
- **R5 (agent / RAG latency budget).** Checkpoint: agent end-to-end p50 < 2s / p95 < 4s (streamed `agent-stream` + per-user priority queue); RAG warm p95 < 500 ms (arms parallel, reranker warm floor, `rag-service` holds no in-process model); graph templates bounded-depth single-org so a noisy tenant cannot blow the budget.
- **R6 (multi-agent collaboration, pre-authorized descope).** Not active this band (`ContextGapAgent` is P10), but the deny-by-default agent-to-agent authz boundary is established in the runtime contract here so P10 inherits it; the throttled-tenant RAG-only degradation path is already the same code path the budget state machine uses.
- **No-silent-commits invariant.** Checkpoint: `core-api` is the sole producer of `decisions`/`actions` on confirmation; `knowledge-graph-service` writes edges only on those events; an unconfirmed candidate never reaches the topic or the graph (the load-bearing P8 test).

---

## 10. Definition of Done & end-of-phase demo  (a demo at the END of EVERY phase)

**End of P7 (W20) — Personal Listener + Billing demo + DoD:**
- **Demo:** in a live meeting, each participant's `PersonalListenerAgent` streams role-relevant worklogs into My Notes (`>0.7` surfaced, `0.4-0.7` silent-logged, `<0.4` dropped); every LLM call is metered to `usage-events`; the Usage panel shows token burn vs `agent_budgets` with the Haiku/Sonnet/Opus split; an 80%/95% alert fires; a Free tenant at `hard_cap` is `blocked` while captions/transcription continue.
- **DoD:** `billing-metering-service` shipped (plans/subscriptions/invoices/usage_events/usage_aggregates/agent_budgets/quotas migrated with RLS FORCE; `usage-events` consumer with idempotent dedupe; Redis budget mirror; `ok→throttled→blocked` state machine; alert-only→enforce flag); `agent-runtime-service` shipped (windowing, Haiku pre-gate, LiteLLM tiered routing, prompt caching, inline `check_budget`, `usage-events` emit, `agent-stream` bridge, writes zero Postgres); `PersonalListenerAgent` per-participant with confidence gating + LLM-egress PII redaction; `worklogs` migrated in `core-api`; My Notes + agent sidebar shell + Usage console shipped; mobile My Notes wired; pre-gate count test, zero-Postgres test, budget deny-by-default test, RLS isolation test all green; DECISION-REQUIRED #1 + #2 resolved.

**End of P8 (W23) — Decisions/Actions + Knowledge Graph demo + DoD:**
- **Demo:** `DecisionAgent`/`ActionAgent` surface confirm/edit/dismiss cards; on Confirm, `core-api` writes the row and is the SOLE producer of the `decisions`/`actions` event; `knowledge-graph-service` creates `PRODUCED`/`ASSIGNED_TO`/`FOLLOWS_FROM` edges in `nexus_knowledge`; the lineage UI walks `FOLLOWS_FROM`; My Tasks lists `action_items`; a dismissed candidate leaves nothing durable.
- **DoD:** `DecisionAgent`/`ActionAgent` shipped as candidate emitters (no silent commits); `decisions`/`action_items`/`agent_suggestions` migrated in `core-api`; confirm endpoints are the sole topic producers; `knowledge-graph-service` shipped (AGE `nexus_knowledge`, per-label `org_id` index, `$org_id` on every template, edge-on-confirmation consumer, four named templates, graph-events); `decisions__{org_id}` embedded on confirm; confirm cards + My Tasks + lineage UI shipped; mobile confirm cards + My Tasks + lineage wired; R1 go/no-go recorded with owner (DECISION-REQUIRED #3); no-silent-commits test, cross-tenant graph isolation test, edge-on-confirmation test all green.

**End of P9 (W26) — RAG & Question Agent demo + DoD:**
- **Demo:** `POST /api/rag/query` returns cited passages (hybrid RRF over Qdrant + FTS + AGE, reranked top-50→top-10, warm < 500 ms, 10-min answer cache); a transcript citation jumps to the timestamp; `QuestionAgent` drops priority 1-3 suggestion cards into `agent_suggestions` with Ask/Dismiss/Save-for-later; Save persists into the next meeting; the Ask panel and mobile Ask screen render cited answers; a throttled tenant still gets single-query cited answers.
- **DoD:** `rag-service` shipped (five-stage `RAGService` over gRPC, `org_id` in metadata + mandatory Qdrant payload filter, RRF k=60, citations with uncited-dropped, answer cache, expansion-skip on throttle, usage-events emit); `QuestionAgent` shipped (priority 1-3, Save-for-later persistence, budget + confidence gates); Ask panel + suggestion cards shipped; mobile Ask screen wired; five-stage/citation test, parallel-arms trace test, cache-hit test, degrade-not-fail test, cross-tenant retrieval isolation test all green; eval harness gates (extraction F1, relevance P/R, question usefulness, citation validity) pass for every shipped prompt.

---

## 11. Source docs

Authoring references used (by path):
- [08-ai-agent-architecture.md](../08-ai-agent-architecture.md) — the six agents, `BaseAgent`/`AgentState`/`PromptLoader`, the five-lever cost-controlled pipeline, confidence thresholds, no-silent-commits, A2A authz contract.
- [09-rag-and-inference-serving.md](../09-rag-and-inference-serving.md) — `RAGService` five stages, hybrid retrieval + RRF, citations, answer cache, embedding/reranker off the API tier, per-stage latency budget.
- [17-cost-and-finops.md](../17-cost-and-finops.md) — COGS/meeting, the `agent_budgets` state machine, the LLM tier-mix cost line, `usage-events` attribution, per-tenant hard quotas.
- [18-saas-business-architecture.md](../18-saas-business-architecture.md) — plans/seats, the metering pipeline, `agent_budgets`, quota enforcement semantics, tenant-admin Usage console.
- [_canon/05-capacity-math.md](../_canon/05-capacity-math.md) — `A17`, `L1`-`L10`, `N1`-`N3`, gate-pass 0.35, tier mix, `C1`-`C4`, the SLO targets imported by symbol.
- [_canon/06-tech-decisions.md](../_canon/06-tech-decisions.md) — §2.5 LLM cost pipeline + per-tenant quotas, §2.6 agents + A2A authz, §2.9 RAG, §2.10 graph schema, §2.12 SaaS business platform.
- [05-data-architecture.md](../05-data-architecture.md) — `worklogs`/`decisions`/`action_items`/`agent_suggestions`/`usage_events`/`agent_budgets` physical schema, AGE DDL + org-partitioning, Redis namespaces, partitioning.
- [adr/0004-llm-cost-control-pipeline.md](../adr/0004-llm-cost-control-pipeline.md) — the window-then-gate ordered pipeline, the `agent_budgets` state machine, build sequencing (windowing + budgets in P7, gating/tiering in P8-9), rollback.
- [adr/0005-graph-store-age-with-migration.md](../adr/0005-graph-store-age-with-migration.md) — AGE now + Neptune/Aura + relational-MV exits, org-partitioning, edge-on-confirmation, the R1 trigger conditions and migration mechanics.
- [adr/0011-billing-and-metering.md](../adr/0011-billing-and-metering.md) — Stripe split, the `usage-events → aggregation → enforcement` pipeline, the idempotency key, the topic-vs-table producer reconciliation, alert-only-first rollout.
- [mytheus-intelligence/system.md](../../mytheus-intelligence/system.md) — the implementation-facing AI-tier design (six agents, pipeline, RAG, gRPC mesh, statelessness, data ownership, events in/out, SLOs).
- [mytheus-intelligence/CLAUDE.md](../../mytheus-intelligence/CLAUDE.md) — service map, invariants (never write Postgres, graph-on-confirmation, deny-by-default, inline budgets), recipes for adding an agent / retrieval arm / Cypher template.
