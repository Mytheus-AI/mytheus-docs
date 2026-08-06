# 00 — Project Understanding (Mytheus / NEXUS)

> **Status:** Anchor explainer for the `mytheus-docs/plan/` build-plan folder.
> **Purpose:** The single technical "what is this project" reference. Everything downstream (the band plans, the per-repo plans, the ADRs) assumes the reader has internalized this file.
> **Canonical names are used verbatim.** Where a service, topic, schema, or table is named here, that spelling is authoritative.

This document is the polished, corrected synthesis of a 15-agent ground-truth survey of all seven repositories plus the architecture canon. It states (1) what the product is, (2) what scale it is built for, (3) how the system decomposes, and (4) exactly where each repo stands today. Read it before any plan file.

---

## 1. What Mytheus / NEXUS is

**Mytheus** (architecture codename **NEXUS**) is a production, multi-tenant SaaS for **intelligent meeting orchestration**. Every meeting participant is given a **personal AI agent** that joins the meeting on their behalf. Unlike a single shared note-taker, each agent filters everything through *that user's* role and skills — resolved via `get_user_context(user_id)` — and then:

- writes role-relevant notes to `worklogs`,
- surfaces real-time question suggestions as priority cards,
- detects decisions, action items, conflicts, and workload issues,
- fills context gaps by querying *other participants' agents*, and
- drives follow-ups through PM integrations (Calendar / Slack / Jira / Linear).

### The two differentiators

1. **Personalized per-participant intelligence.** There is not one meeting bot; there is one agent *per participant*, each producing role-relevant output. The `PersonalListenerAgent` is spawned once per participant per meeting and writes notes shaped by the user's domain and skills, not a single generic transcript summary.

2. **Agent-to-agent (A2A) collaboration across meetings.** The `ContextGapAgent` can query other users' agents over the `agent-requests` / `agent-responses` topics using correlation IDs, deadlines, and a deny-by-default authorization model. It follows a strict fall-through chain — **peer-agents → related-meeting agents → RAG → silent** — so a gap is filled from the most authoritative live source available, and the system stays quiet rather than fabricating.

### Lineage

The original *"NEXUS Execution Plan v2.1"* PDF was a 4-person, 13-phase / 36-week MVP with **no tenancy**. The current architecture set keeps that 13-phase backbone verbatim but **re-targets it as a production multi-tenant SaaS**. The forcing function for the entire architecture is that **the agent / inference tier scales and bills completely differently from the CRUD tier** — that single fact drives the service decomposition, the cost pipeline, and the scale-out order.

---

## 2. Capacity target + the three scaling families

NEXUS is sized to explicit capacity anchors:

| Anchor | Value | What it drives |
| --- | --- | --- |
| **A1** | 5,000,000 registered users | Identity / control-plane storage |
| **A4** | 3,000 paying tenants | Tenant isolation model, billing |
| **A5** | 20,000 peak concurrent **meetings** | Transcription / embedding / LLM load — **one diarized stream per meeting** |
| **A6** | 100,000 peak concurrent **participants** | Realtime fan-out (WSS / Socket.io) |
| **A8** | 5.0 effective occupancy | Reconciles A5 ↔ A6 (≈ 5 participants/meeting) |

Services are grouped so that each scaling family has **its own autoscaler signal and its own cost center**:

1. **CRUD / control-plane family** — scales on request volume and tenant count. Stateless, cheap, horizontally trivial.
2. **Real-time family** — `realtime-gateway` scales on **A6** (concurrent participants / fan-out); `transcription-service` scales on **A5** (concurrent meetings / diarized streams).
3. **AI / inference family** — scales on **A5** and on per-window agent invocations; GPU-bound and by far the most expensive. This is the tier whose economics dominate the whole design (see §8).

An **async / orchestration** tier (Temporal workflows) rides alongside these but is not a separate scaling anchor — it scales on workflow throughput.

---

## 3. The 20 canonical services + repo ownership

Services are **kebab-case** and are owned by repo according to tier. **The Service Catalog is authoritative for service names.** A persistent naming gotcha: the Data Model canon §1.1 uses coarser labels (identity-service, agent-service, graph-service, billing-service, budget-service, audit-service) — these map onto the catalog names, but **the Service Catalog always wins for service names; the Data Model is authoritative only for entity / table names**.

### `mytheus-backend` — CRUD / control-plane + async / orchestration (modular monolith of ten services)

| Service | Owns / does |
| --- | --- |
| `api-gateway` | Single ingress: TLS, WAF, **Clerk** JWT pre-validation, `org_id` extraction, routing. No business logic. |
| `bff` | Backend-for-frontend; screen-shaped read aggregation. Owns no domain data. |
| `identity-tenant-service` | Owns `organizations`, `users`, teams/departments, RBAC grants, SSO/SAML/SCIM, `tenant_settings`, `data_residency`. **Sole producer of `tenant-events`.** Hosts `get_user_context`. |
| `core-api` | Relational heart: `meetings`, participants/speakers, `worklogs`, `decisions`, `action_items`, `agent_suggestions`, `agent_collaborations`, `chat_messages`. **Sole producer of confirmed `decisions` / `actions` events via the transactional outbox.** |
| `meeting-service` | LiveKit transport adapter, room-token minting, `meeting_rooms`. |
| `integration-service` | OAuth integrations (Calendar/Slack/Jira/Linear), `integration_connections`, `oauth_tokens`. |
| `billing-metering-service` | Stripe billing, usage metering → aggregation → quota enforcement, `agent_budgets`. |
| `admin-ops-service` | Tenant-admin + internal-ops consoles, `audit_logs`, feature flags. |
| `notification-service` | Sole external fan-out (in-app / email / Slack). |
| `workflow-service` | Temporal workers: MeetingLifecycleWorkflow, ActionFollowUpWorkflow, tenant-provisioning, RTBF, dunning. |

### `mytheus-realtime` — real-time tier (two services)

| Service | Owns / does |
| --- | --- |
| `realtime-gateway` | Socket.io / WSS fan-out for non-AV signals. Scales on **A6**. |
| `transcription-service` | Deepgram WS proxy; persists **finals only** to `transcript_chunks`. Scales on **A5**. |

### `mytheus-intelligence` — AI / inference tier (seven services)

| Service | Owns / does |
| --- | --- |
| `ingestion-service` | Document/transcript ingestion + chunking. |
| `embedding-service` | BGE-small-en, 384-dim, **GPU**. |
| `reranker-service` | BGE-reranker-base, **GPU**. |
| `agent-runtime-service` | Hosts the six LangGraph agents (see §4). **Owns NO Postgres tables** — its outputs are written by `core-api`. |
| `rag-service` | Retrieval-augmented generation over Qdrant + AGE + FTS. |
| `knowledge-graph-service` | Apache AGE graph operations. |
| `search-service` | Postgres FTS. |

> **Repo status (corrected):** `mytheus-intelligence` **exists** as a **documentation-only** repo (CLAUDE.md / README.md / system.md / .gitignore) with excellent specs for all 7 services and the 6 agents — but **zero code**. It is *not* missing from disk. See §10.

### Remaining repos

- **`mytheus-ui`** owns `web-app` (Next.js 14). *(Counts toward the 20 canonical services.)*
- **`mytheus-events`** owns the shared Avro contracts (the event-schema repo).
- **`mytheus-platform`** owns IaC / deploy.
- **`mytheus-mobile`** is a first-class native client (being realigned to spec — see §11).

> The "20 canonical services" are the ten in `mytheus-backend`, the two in `mytheus-realtime`, the seven in `mytheus-intelligence`, plus `web-app` in `mytheus-ui`. `mytheus-events`, `mytheus-platform`, and `mytheus-mobile` are contract / infra / client repos rather than backend services.

---

## 4. The six LangGraph agents

All six run on a shared `BaseAgent` lifecycle, one `AgentState` dataclass, and a versioned prompt loader, all inside `agent-runtime-service`.

| Agent | Phase | Role |
| --- | --- | --- |
| **PersonalListenerAgent** | P7 | One spawned per participant per meeting; writes `worklogs`. |
| **DecisionAgent** | P8 | Detects decision candidates. |
| **ActionAgent** | P8 | Detects action-item candidates. |
| **QuestionAgent** | P9 | Real-time suggestion cards (`agent_suggestions`, priority 1–3). |
| **ContextGapAgent** | P10 | Cross-meeting A2A context fill; deny-by-default authz. |
| **ProactiveAgent** | P11 | Conflict / workload / stakeholder / dependency alerts. |

### Confidence thresholds (frozen)

- **> 0.7** → surface to the user.
- **0.4 – 0.7** → silent-log to `worklogs`.
- **< 0.4** → drop.

Overridable per tenant via `tenant_settings.agent_confidence_overrides`.

### The no-silent-commits rule

**Agents emit candidates only.** They never write a durable decision or action. The user confirms a candidate via `core-api`, and **only then does `core-api` (not the agent) produce the `decisions` / `actions` event**, which is what creates the corresponding edge in the AGE knowledge graph. This invariant is what keeps the graph and the durable record human-ratified.

---

## 5. Data stores

| Store | Technology | Role & key properties |
| --- | --- | --- |
| **Relational (system of record)** | **PostgreSQL via Supabase** (SQLAlchemy 2.0 async, asyncpg, Pydantic v2, Alembic) | RLS `FORCE` on `org_id`. High-volume tables (`transcript_chunks` ≈ 1.728 B/mo, `chat_messages`, `meetings`, `worklogs`, `agent_suggestions`, `usage_events` daily, `audit_logs`) are declarative **range partitions**. PgBouncer transaction pooling forces `SET LOCAL app.current_org` inside the transaction. |
| **Graph** | **Apache AGE** (`nexus_knowledge`), co-located in Postgres | Org-partitioned. Nodes: Person / Meeting / Decision / Action / Topic. Edges: ATTENDED / PRODUCED / ASSIGNED_TO / RELATES_TO / FOLLOWS_FROM. **Edges written on confirmation only.** Derived / rebuildable from confirmed rows. R1 exit path: Neptune / Neo4j Aura or relational MV fallback. |
| **Vector** | **Qdrant Cloud** | Per-tenant collections `transcript_chunks__{org_id}`, `document_chunks__{org_id}`, `decisions__{org_id}`. BGE-small-en 384-dim cosine, int8 quantized. **Mandatory `org_id` payload filter** (defense-in-depth). Derived / rebuildable. |
| **Cache / coordination** | **Redis (Upstash)** | Namespaced `nexus:{org_id}:...` for user_ctx (5 m TTL), rag cache (10 m), dedupe (24 h), budget mirror, pub/sub, presence, queues. Ephemeral. |
| **Object** | Object storage | `nexus-recordings`, `nexus-documents`, `nexus-attachments`, `nexus-invoices` (7-yr WORM), `nexus-audit-archive`, `nexus-exports`. Region-pinned via `data_residency`. |

Only the relational store and the object store are systems of record; AGE, Qdrant, and Redis are **derived / rebuildable** from confirmed rows.

---

## 6. The Redpanda event backbone

The bus is **Redpanda** (Kafka API) with **Avro + Schema Registry**, subject naming `{topic}-value`, **BACKWARD** compatibility.

### Five invariants

1. **One bus.** Every event flows through Redpanda.
2. **Partition key = `org_id`, on 19 of 21 topics.** `transcript-chunks` and `agent-stream` are the exception: composite `{org_id}:{meeting_id}` ([ADR-0017](../adr/0017-transcript-chunks-composite-partition-key.md)), with `org_id` still mandatory and still the leading component.
3. **At-least-once + idempotency** via transactional **outbox** (producer side) and **inbox + Redis dedupe** (consumer side).
4. **Poison messages quarantine to the DLQ.** Each topic has a paired `<topic>.dlq` (14-day retention).
5. **No silent commits** (mirrors §4 — durable facts are produced only on confirmation).

### The 21 topics (9 PDF + 12 SaaS)

`transcript-chunks`, `document-chunks`, `agent-results`, `agent-stream` (highest-count; **shed first** under load), `decisions`, `actions` (both: `core-api`, on confirm only), `agent-requests`, `agent-responses` (A2A), `notifications`, plus the SaaS control-plane topics: `tenant-events` (sole producer `identity-tenant-service`, **log-compacted**), `meeting-events`, `meeting-room-events`, `integration-events`, `usage-events` (producers `agent-runtime-service` + `rag-service` only — derived dimensions are written directly as rows), `billing-events`, `graph-events`, `embedding-events`, `presence-events`, `notification-delivery-events`, `user-sync-events`, `admin-events`.

### The seven frozen Avro schemas

Only these are **frozen canon**: **TranscriptChunk, AgentResult, Decision, Action, Notification, AgentRequest, AgentResponse.** The remaining schemas are net-new and **not** frozen.

### EventEnvelopeHeader

Every value opens with the shared `EventEnvelopeHeader`:

```
event_id, idempotency_key, org_id, schema_version,
producer, occurred_at, correlation_id, causation_id
```

---

## 7. Multi-tenancy via `org_id` + RLS

**`organizations.id` IS `org_id`** — there is no second tenant identifier. `org_id` is resolved **once at the edge** from the verified JWT and propagated; it is **never re-derived** from path / body / header, and is **never a path or query parameter**.

**Six independent defense-in-depth backstops:**

1. **Postgres RLS** — `USING (org_id = current_setting('app.current_org')::uuid)`, transaction-local.
2. **Per-tenant Qdrant collections** + mandatory `org_id` payload filter.
3. **`$org_id`-parameterized AGE Cypher.**
4. **Tenant-keyed Redpanda partitions.**
5. **`nexus:{org_id}:` Redis prefixes.**
6. **Per-tenant object prefixes.**

**Hybrid isolation model:** pooled + RLS for the Free / Pro long tail; cell-based isolation for Enterprise (RLS stays *on* inside a cell).

**Everything fails closed:** an unset `org_id` matches zero rows; a cross-tenant fetch returns **404, never 403**.

**RBAC vs domain role:** RBAC tiers (owner / admin / member) live in a grant store and are distinct from `users.role`, which is a **domain / skills attribute** used for agent relevance — not an authorization level.

---

## 8. The cost-controlled LLM pipeline + R4

LLM is **≈ 94 % of variable cost**. A naive per-chunk design (**N2 ≈ 41.5 B calls/mo ≈ $224 M/mo** all-Sonnet) is the **R4 cost bomb** — deliberately **NOT built**.

The locked pipeline runs agents per **30-second window (A17)**, not per raw chunk, in execution order **window-then-gate**:

1. **Windowed batching** — 360 chunks → ≈ 90 windows.
2. **Relevance pre-gating** — Haiku, ≈ 35 % pass rate.
3. **Tiered routing via LiteLLM** — Haiku triage → Sonnet extract → Opus high-stakes.
4. **Caching** — semantic + Anthropic prompt caching.
5. **Per-tenant token budgets** — hard quotas in `agent_budgets`, **deny-by-default**.

**Result:** **L7 ≈ 5.79 B calls/mo** — an **86 % reduction (L10)** before caching — plus a ≈ 30 % Phase-13 optimization target (**C4 → ≈ $6.99 / MAU**).

The other locked lever: **one diarized Deepgram stream per meeting** (a 6× cost lever). `meeting_minute` is **billed once per meeting, not per participant**.

**Budget states:** `agent_budgets.state` is `ok → throttled → blocked` (80 % / 95 % are **alert flags, not states**). Quota enforcement **never sheds durable facts** (transcripts, decisions, actions, usage-events) — **only discretionary LLM work degrades**.

---

## 9. The roadmap P1–P13 (W1–W36) + post-W36 scale-out

The PDF's 36-week, 13-phase backbone is kept verbatim, with five SaaS overlay streams pulled earlier as preconditions (security shift-left P1→P13; compliance live before P4; dedicated GPU inference at P6; billing & metering by P7; multi-region post-W36).

| Phase | Weeks | Scope |
| --- | --- | --- |
| **P1** Architecture & Design | W1–2 | C4 diagrams, ERD, Avro drafts, repo + CI + Compose, threat model, residency strategy, subprocessor DPAs. |
| **P2** Foundation | W3–5 | `identity-tenant-service`, `core-api`, `bff`, `web-app`, `api-gateway`; **RLS + org_id from the first migration**; `audit_logs` stood up here. |
| **P3** Meeting Platform | W6–8 | `meeting-service`, `realtime-gateway`, LiveKit behind an adapter. |
| **P4** Real-time & Transcription | W9–12 | `transcription-service`; **compliance gate GREEN before W9**. |
| **P5** Profiles & Org Context | W13–14 | Role / skills, teams / departments, `get_user_context`. |
| **P6** AI Infrastructure | W15–17 | embedding / reranker / ingestion / search services, Redpanda / Avro / Schema Registry, **dedicated GPU tier**, per-tenant Qdrant, outbox / inbox / DLQ. |
| **P7** Personal Listener Agent | W18–20 | `agent-runtime-service` + PersonalListenerAgent; **billing metering + token budgets must be live**. |
| **P8** Decisions/Actions + Knowledge Graph | W21–23 | DecisionAgent, ActionAgent, AGE (**R1 checkpoint**). |
| **P9** RAG & Question Agent | W24–26 | `rag-service`, QuestionAgent. |
| **P10** Workflows & Multi-Agent | W27–29 | Temporal MeetingLifecycleWorkflow, ContextGapAgent, A2A authz (**R6 checkpoint**). |
| **P11** Proactive Intelligence & Real-time Polish | W30–31 | ProactiveAgent, Redis pub/sub backbone, streamed output. |
| **P12** External Integrations | W32–33 | Calendar / Slack / Jira-Linear, ActionFollowUpWorkflow, admin-ops consoles (**R8 guardrail**). |
| **P13** Testing/DevOps/Hardening/Production | W34–36 | Playwright / k6, final security attestation, full Stripe surface, PITR / replica, Cloudflare. |

### Deployment posture

**The MVP deploys as a modular monolith on Railway / Render.** Kubernetes is **post-W36 scale-out**, not an MVP requirement. The ten `mytheus-backend` services ship as one deployable monolith; the boundaries are enforced in code (module ownership, no cross-service table access) so that extraction later is mechanical.

### Post-W36 scale-out

Services are extracted in a frozen **Wave 1–4** order:

- **Wave 1** — GPU inference + real-time + workers.
- **Wave 2** — CRUD core.
- **Wave 3** — SaaS business services.
- **Wave 4** — retrieval stores.

…moving to **cells**, multi-region **active-passive → active-active** (in-zone only), with **Citus / Neptune** escalation paths held in reserve.

---

## 10. Current state of each repo (the starting line)

> **Ground truth as of today: across all seven repos there is almost no working application code.** One repo (`mytheus-backend`) has a small real shared library and full local infra; one (`mytheus-mobile`) has a polished but divergent mock-only UI prototype; **the other five are documentation-only**.

### `mytheus-backend` — EARLY-STAGE SCAFFOLD *(the only repo with real backend code)*

- **REAL / WORKING:** `libs/shared/` (8 modules) is genuinely functional — JWT verification, tenancy helper (`org_id` ContextVar + transaction-local `set_config('app.current_org', …, true)`), transactional-outbox enqueue (raw SQL), inbox dedupe (Redis `SETNX` + inbox INSERT), `EventEnvelope` dataclass, Idempotency-Key middleware, RFC 7807 error handler, OpenTelemetry setup. Two complete Supabase SQL files (a `custom_access_token_hook` and an `auth.users → public.users` sync trigger). Full local infra (docker-compose: Postgres / Redis / Redpanda / Temporal; Makefile; .env.example; pyproject.toml). Extensive docs.
- **SCAFFOLD / STUB:** All 10 `services/*/main.py` boot FastAPI + telemetry but expose **no routes** (`# TODO: add routers`). Only routers present are in `identity-tenant-service` (an auth webhook with a real HMAC check but `pass # TODO` handlers; SCIM routes that all `raise NotImplementedError`). The notification consumer connects but `_handle` is commented out. The Temporal worker registers 4 workflows with **zero activities**.
- **MISSING:** **No SQLAlchemy models anywhere** (entire ORM layer absent); `migrations/versions/` empty, `target_metadata = None` — so the `outbox` / `inbox` / `users` / domain tables the shared helpers write to **have no DDL and do not exist**. Zero tests. No outbox relay / publisher. No gRPC server / stubs (only `.proto`). No api-gateway routing logic. Idempotency middleware not mounted anywhere.
- **DRIFT (to be reverted):** Auth code / .env / SQL drifted **Clerk → Supabase Auth**, while README.md and CLAUDE.md still reference Clerk (JWKS, Svix, `CLERK_*`). **Per the resolved decision in §11, identity is Clerk; the Supabase Auth drift will be reverted.** README also references a nonexistent `app.main:app` entry point.

### `mytheus-realtime` — DOCUMENTATION-ONLY

Four files only (README, system.md, CLAUDE.md, .gitignore). No `services/realtime-gateway/`, no `services/transcription-service/`, no `proto/`, no `libs/`, no package.json, no pyproject.toml. Design is thoroughly specified; implementation has not begun.

### `mytheus-events` — DOCUMENTATION-ONLY

Four files only. No `schemas/*.avsc`, no Schema Registry subjects, no `EventEnvelopeHeader` / `DlqEnvelope` definitions, no codegen, no Makefile (despite docs invoking `make validate` / `check-compatibility` / `register` / `codegen`). Complete written spec of all 21 topics and the 8-field envelope — nothing executable or importable. Also references a `_canon/` authority not present in the repo.

### `mytheus-ui` — DOCUMENTATION-ONLY

Four root files. No package.json, tsconfig, next.config; no `app/`, `components/`, `lib/`, `e2e/`. No code has ever been committed (3 commits, all markdown). Stack / structure fully specified in prose only.

### `mytheus-platform` — DOCUMENTATION-ONLY

Four root files. No `terraform/`, `k8s/`, `helm/`, `.github/workflows/`, `observability/`, or `runbooks/` (all verified MISSING). The only real artifact is a well-tuned .gitignore. High-quality spec, zero implementation.

### `mytheus-intelligence` — DOCUMENTATION-ONLY *(exists; not missing)*

**The repo EXISTS** as a **documentation-only** repo: CLAUDE.md / README.md / system.md / .gitignore, with **excellent specs** for its **7 services** (`ingestion-service`, `embedding-service`, `reranker-service`, `agent-runtime-service`, `rag-service`, `knowledge-graph-service`, `search-service`) and the **6 agents** — but **zero code**. The entire AI / inference tier is specified and unbuilt. *(Earlier survey notes that called this repo "missing / not on disk" were wrong and are superseded by this file.)*

### `mytheus-mobile` — POLISHED MOCK PROTOTYPE *(being realigned)*

The docs describe a multi-tenant LiveKit / Socket.io enterprise client. The actual code today is a different product: a self-contained single-device **"Meeting Pod"** Expo / React Native UI prototype on entirely hardcoded mock data. Real, well-crafted UI exists (Expo Router shell, 5-tab bar, ≈ 20 components, design tokens, reanimated animations, working local interactivity, live-session timer) — but **every piece of content is mock** (`src/data/mock.ts`, 375 lines): the "live" meeting is fake (static transcript, cosmetic waveform, elapsed time hardcoded to 767 s, no audio capture). **Missing:** all backend integration, realtime, auth (no Clerk), push, state libs, secure storage, tests. **Per the resolved decision in §11, the "Meeting Pod" mock will be realigned into a first-class client against the documented web contract.**

### Starting-line implications for the build plan

1. **The shared library in `mytheus-backend` is the one reusable asset** — outbox / inbox / tenancy / auth / errors / telemetry are real and should anchor Plan 0/1.
2. **The data layer is the critical gap everywhere** — no ORM models, no migrations, no tables exist; nothing the helpers write to is real yet. This is the first thing the foundation band must produce.
3. **`mytheus-events` must be built before any cross-repo event flow** — sibling repos are meant to consume generated clients that do not exist.
4. **`mytheus-intelligence` exists but is empty** — the entire AI / inference tier (7 services, 6 agents, the cost pipeline) must be implemented from its existing specs.
5. **Doc drift (Clerk vs Supabase) must be reconciled before P2** — and the resolved direction is **Clerk** (§11).
6. **`mytheus-mobile` is off-spec** — and the resolved direction is **realign to the documented client** (§11).

---

## 11. Resolved decisions + open-decisions register

### Resolved decisions (binding on the build plan)

1. **Identity = Clerk.** The `mytheus-backend` code/.env/SQL drift to Supabase Auth **will be reverted**. JWT verification, the `org_id` claim injection, the user-sync mechanism, and SSO/SAML/SCIM are all built against the **Clerk** contract (JWKS, Svix webhooks). The Supabase `custom_access_token_hook` path is dropped in favor of the Clerk equivalent. This must land **before P2**.
2. **`mytheus-mobile` is realigned into a first-class client.** The off-spec single-device **"Meeting Pod"** mock is *not* the product. Mobile is re-aligned to the documented multi-participant LiveKit / agent-sidebar client and built against the same `api-gateway` / `realtime-gateway` contracts as `web-app` (including Clerk auth). The well-crafted UI shell is retained as a starting point where it fits the real contract.
3. **MVP deploys as a modular monolith on Railway / Render.** Kubernetes is **post-W36 scale-out**, not an MVP requirement. `mytheus-platform` provisions Railway / Render for the MVP; the K8s / Helm / cells work is staged for the Wave 1–4 extraction.

### Open-decisions register (assign owners before the relevant band)

| # | Open decision | Blocks |
| --- | --- | --- |
| O1 | **RESOLVED** ([ADR-0017](../adr/0017-transcript-chunks-composite-partition-key.md)). `transcript-chunks` and `agent-stream` are composite-keyed `{org_id}:{meeting_id}`; every other topic keeps `org_id` alone. Canon 03 and docs 07/10 are reconciled to this. | P3–P4 (unblocked) |
| O2 | **RESOLVED, schema only** ([ADR-0018](../adr/0018-consent-entity-behind-flag.md)). `consented_at` / `consent_version` on `meeting_participants` and `consent_required` on `tenant_settings` are amended into canon, defaulted off. The 428 token-mint gate's application code and the migration remain later tasks; whether it must ship before P3/P4 launch is still the legal/compliance call ADR-0018 assigns. | P3–P4 (schema unblocked; gate build still pending) |
| O3 | **No `manager_id` / reporting edge** exists in the data model — ActionFollowUpWorkflow day-10 manager escalation is best-effort over the departments tree + organizer fallback. Decide whether a first-class `manager_id` is added. | P12 |
| O4 | **Plan limits per-seat vs per-tenant** (`included_meeting_minutes` / `included_agent_tokens`) is unresolved, not canon. Affects metering quota math. | P7–P9 |
| O5 | **Derived `usage_events` dimension producers** (`meeting_minute`, `transcription_second`, `storage_gb`) are not named in the Service Catalog (docs point to transcription-service / meeting-service / a storage-sweep job). Resolve ownership before metering. | P7 |
| O6 | **Where canon physically lives:** the `_canon/` authority referenced by `mytheus-events` and `mytheus-realtime` is not present in those repos. Decide on a shared docs repo vs vendored copies so naming / schema authority is CI-enforceable. | P1 |
| O7 | **R1 / R6 / R8 descope checkpoints** are pre-authorized but unscheduled as decisions: AGE on Supabase vs exit to Neo4j Aura (**R1 / P8**); A2A ships vs collapses to RAG-only (**R6 / P10**); which one of Jira / Linear ships first (**R8 / P12**). Assign go/no-go owners. | P8 / P10 / P12 |

---

*This file is the technical anchor for the `mytheus-docs/plan/` folder. When a downstream plan, ADR, or repo CLAUDE.md disagrees with the canonical names, scaling anchors, invariants, or resolved decisions stated here, this file governs until explicitly amended.*
