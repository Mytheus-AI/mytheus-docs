# Mytheus / NEXUS — INTENDED SCOPE extracted from `mytheus-docs`

Read-only throughout; nothing was modified. Ignored `.worktrees/` and the nested stale
`mytheus-docs/mytheus-docs/` copy — both are worktree snapshots of older bands.

All completion claims in these docs are treated as unreliable and are quarantined in the
"UNVERIFIED CLAIMS" section near the end. Everything before that section is specification only.

---

# THE PLAN LADDER

## Plan 0 — Scaffold, Tooling & Event Contracts
`plan/plan-0-scaffold-and-contracts.md`

**Goal:** every repo compiles/lints/tests green; frozen Avro contracts exist with a BACKWARD CI
gate; backend gets its first real DDL, idempotency, outbox relay, gRPC and Clerk auth. Nothing
feature-shaped — this is the foundation Plan 1-2 builds on (`plan/plan-0-scaffold-and-contracts.md:22`).

1. `mytheus-events` repo skeleton: `schemas/ subjects/ codegen/ contracts/ fixtures/ tests/
   clients/{python,typescript}` + Makefile (`:57–75`).
2. Author `EventEnvelopeHeader` + `DlqEnvelope`, frozen field order, verbatim from
   `07-event-architecture.md` §3.1 (`:116`).
3. Author the **7 frozen** Avro value schemas: `TranscriptChunk`, `AgentResult`, `Decision`,
   `Action`, `Notification`, `AgentRequest`, `AgentResponse` (`:163`).
4. Author the 14 net-new schemas: `DocumentChunk`, `AgentStreamDelta` + the 12 SaaS payloads
   (`TenantEvent`, `MeetingEvent`, `MeetingRoomEvent`, `IntegrationEvent`, `UsageEvent`,
   `BillingEvent`, `GraphEvent`, `EmbeddingEvent`, `PresenceEvent`, `NotificationDeliveryEvent`,
   `UserSyncEvent`, `AdminEvent`) (`:200`).
5. 21 `{topic}-value` subjects with BACKWARD compatibility config (`:223`); Makefile targets
   `validate / check-compatibility / register / codegen / test` (`:235`); golden fixtures that
   round-trip through generated **Python and TypeScript** clients for all 21 topics (`:270`).
6. `mytheus-backend`: SQLAlchemy 2.0 async model layer + `Base`/metadata (`:305`); wire
   `target_metadata = Base.metadata` in Alembic `env.py` (`:378`).
7. First migrations: `outbox`, `inbox`, `organizations`, `users` (full) + P2 stub DDL for
   `teams`, `team_members`, `departments`, `meetings`, `meeting_participants`, `tenant_settings`,
   `data_residency` — all with RLS FORCE (`:395`, `:47`).
8. Mount the `Idempotency-Key` middleware on **all 10 services** (`:431`).
9. Build the transactional-outbox relay/publisher process (`:457`).
10. Generate the gRPC server + stubs from `proto/identity.proto` (`:483`).
11. **Auth reconciliation (Phase 0C):** restore Clerk RS256 JWKS verification in
    `libs/shared/auth.py` (`:504`); restore the Svix `POST /clerk/webhook` skeleton in
    `identity-tenant-service` (`:546`); re-add `CLERK_*` env and delete the Supabase
    `custom_access_token_hook` + `auth.users` trigger (`:578`); fix READMEs/CLAUDE.md (`:595`).
12. `mytheus-intelligence`: workspace `pyproject.toml` + **7 service skeletons** + shared lib
    (`:604–635`).
13. `mytheus-ui`: Next.js 14 App Router scaffold + Clerk provider placeholder; Tailwind +
    shadcn/ui, Socket.io client, LiveKit React client per `_canon/06-tech-decisions.md` §2.8
    (`:656`).
14. `mytheus-realtime`: `realtime-gateway` (Node + Socket.io + Redis adapter) and
    `transcription-service` (Python FastAPI WS proxy) skeletons that boot `/healthz` (`:677`).
15. `mytheus-platform`: reusable GitHub Actions CI (lint / test / gitleaks / dep-scan /
    avro-compat / docker-build) + a modular-monolith deploy descriptor + `.env` templates.
    **NO Kubernetes, NO Terraform** — those are Plan 14 (`:694–770`).
16. Conventions: uniform `make ci` across repos, pre-commit, canon-location decision, and a
    `CONTRIBUTING.md` carrying the org_id authz-review rule (`:778–790`).
17. `mytheus-mobile`: builds; Clerk SDK + `httpClient` + env base URL added; screens stay on
    mock; a `REALIGN.md` backlog is written (`:844`).

> Substrate note at the top of the file (`:3–7`): wherever this band says Railway/Render, read
> **single-node VPS under Docker Compose** (ADR-0016). Railway and Render are struck entirely —
> not the fast path, not a fallback.

**DoD highlights (`:885`):** a breaking Avro change must *fail* `check-compatibility`; a `POST`
with a repeated `Idempotency-Key` replays from Redis; the relay drains an `outbox` row to a local
Redpanda topic and a second pass publishes nothing; a Clerk-signed RS256 JWT verifies via JWKS.

---

## Plan 1-2 — Foundation and Tenancy (P1 W1–W2, P2 W3–W5)
`plan/plan-1-2-foundation-and-tenancy.md`

**Goal:** the tenant-isolated CRUD spine — Clerk auth at the edge, `org_id` extracted once at
`api-gateway`, RLS `FORCE` on every tenant table from migration #1, `audit_logs` append-only live
at P2, and a `web-app` dashboard rendering an org-scoped view through `bff`. Plus the P1 design
pack (`:12`). This band builds **only** the CRUD/control-plane spine — no agents, no
transcription, no Qdrant/AGE, no LiveKit media, no billing (`:19`).

1. **P1 design pack** in `mytheus-docs/design/`, `adr/`, `threat-model/`: C4 L1/L2, full ERD,
   Avro draft subjects, `/api/v1` OpenAPI draft, sign-in + dashboard wireframes, signed-off
   `design/tech-lock.md`, STRIDE threat model, data-residency region decision, subprocessor DPAs
   (Deepgram/OpenAI/Anthropic/LiveKit), RTBF deletion contract (`:76–104`).
2. Revert the Supabase-Auth drift back to Clerk in `libs/shared` + `.env.example` — first in P2
   because every downstream step depends on it (`:107`).
3. Migration baseline: outbox/inbox, `organizations`, and the canonical RLS policy macro applied
   to every table without exception (`:210`).
4. `identity-tenant-service`: tenancy tables + the `rbac_grants` RBAC grant store (`:292`).
5. `identity-tenant-service`: Clerk Svix webhook user-sync — inbox dedupe + reconciliation +
   org-lifecycle outbox, replacing the Plan-0 `pass` handler and the reverted Supabase trigger
   (`:383`).
6. `core-api`: `meetings` + `meeting_participants` CRUD, `meeting-events` outbox wiring, and the
   org-hierarchy tables it owns (`:428`).
7. `admin-ops-service`: `audit_logs` append-only, tamper-evident — **stood up at P2, not P13**,
   so RLS/authz events are recorded from the moment RLS goes live (`:502`).
8. `api-gateway`: edge JWT pre-validation, `org_id` extraction, routing, service-tier authz
   re-verification, cross-tenant → `404`. The single chokepoint where `org_id` is resolved once
   (`:556`).
9. `bff`: screen-shaped read aggregation for the dashboard; the only caller of `/bff/*` (`:611`).
10. `web-app` shell (Next.js 14): Clerk auth, nav, org-scoped user dashboard (`:642`).
11. CI security: gitleaks secret scanning, SCA dependency scanning, authz-check-reviewed-in-every-PR,
    RLS-isolation testcontainers — the shift-left replacement for the PDF's Phase-13 audit (`:674`).
12. `mytheus-mobile`: Clerk auth + `org_id`-aware `httpClient` + mock-free home/people/library
    (`:750`).

**Acceptance that matters (`:829`):** two tenants see disjoint dashboards; `GET
/api/v1/meetings/{orgA_meeting_id}` as Org B returns **404**; a forced query with no
`app.current_org` returns **zero rows** (RLS fail-closed, proven in CI); the client sends no
`org_id` on the wire.

---

## Plan 3-4 — Meeting Platform + Real-time & Transcription (P3 W6–8, P4 W9–12)
`plan/plan-3-4-meeting-and-transcription.md`

**Goal:** a participant joins a real LiveKit room from `web-app` (and mobile), sees presence and
chats; then P4 adds live captions under the 2 s budget with diarized finals persisted and flowing
onto the bus. No agents — this band delivers the *capture* substrate they consume (`:10`).

1. `meeting-service` — LiveKit transport adapter: `POST /api/meetings/:id/token`
   (participant-verified, p95 < 250 ms), `POST /livekit/webhook` (HMAC-verified, inbox-deduped),
   the `meeting_rooms` table (RLS FORCE), and the `nexus-recordings` bucket (`:62`).
2. `realtime-gateway` (Node + Socket.io) — scaffold the repo: tenant-scoped rooms keyed by
   `meeting_id`, Redis pub/sub cross-pod fan-out, presence, chat relay, captions transport
   (`:187`).
3. `core-api` — `chat_messages` persistence + `POST /chat`, `meeting_speakers` mapping,
   `meeting-events` producer wiring (`:269`).
4. `web-app` — LiveKit room UI (video grid), in-meeting chat + presence (`:319`).
5. **COMPLIANCE GATE — must be GREEN before any Deepgram stream opens (W9)**: Deepgram DPA +
   SCCs on file, the at-rest PII redaction pass live, residency pinning enforced on
   `transcript_chunks` and on the Deepgram endpoint (`:339`).
6. `transcription-service` — Deepgram WS proxy behind a provider adapter: diarization, interim
   vs final split, **finals-only** persistence, at-rest redaction node gated by
   `tenant_settings.pii_redaction_enabled`, residency pinning, the `transcript-chunks` producer
   (outbox + Avro + `chunk:{chunk_id}` dedupe), and the `meeting-room-events` consumer (`:354`).
7. `realtime-gateway` — live-captions wiring (`:434`).
8. `web-app` — live captions view, transcript view, speaker-mapping UI (`:444`).
9. `mytheus-mobile` — realign `live-session`/`start-session` onto real LiveKit media + the
   `realtime-gateway` Socket.io plane; real server-anchored meeting clock; mic capture and
   permissions (`:506`).
10. Avro subjects registered BACKWARD: `transcript-chunks-value`, `meeting-room-events-value`,
    `presence-events-value` + their `.dlq`.
11. `transcript_chunks` migrated with monthly range partitioning, a local `(meeting_id,
    start_ms)` index, and RLS FORCE; the partition-key decision ratified (§8 #1 → ADR-0017).

**Tests named as gates (`:553`):** interims-never-persisted, redaction on/off, residency pinning,
reconnect SLO, cross-pod fan-out, token-mint deny-by-default, RLS isolation.

---

## Plan 5-6 — Org Context and AI Infrastructure (P5 W13–14, P6 W15–17)
`plan/plan-5-6-org-context-and-ai-infra.md`

**Goal:** make the org legible to agents (`role`/`skills`/teams/departments → `get_user_context`),
stand up the GPU inference tier and per-tenant Qdrant, ingest documents, and turn the event
backbone GA with the embedding consumer as the first real at-least-once consumer (`:12`).

1. `identity-tenant-service` — `users.role` / `skills[]`, `teams`, `team_members`, `departments`
   migrations with RLS FORCE (`:59`).
2. The version-controlled **role→topics taxonomy**: `GET /taxonomy/role-topics?version=v1`, with
   `Topic` nodes seeded per org (`:130`).
3. `get_user_context()` LIVE with a Redis cache at `nexus:{org_id}:user_ctx:{user_id}`, TTL
   ≤ 300 s, eviction on role/team change; exposed over **REST and gRPC** (`:163`).
4. The seeder — 2 orgs × 10 users × 5 meetings, idempotent behind `make seed` (`:216`).
5. `web-app` + `bff` — `/org` org-chart UI and `/people/{id}` person profile (`:246`).
6. **Event backbone GA** — all 21 topics + 21 DLQs created, 9 Avro subjects registered BACKWARD,
   the CI compatibility gate live, outbox relay + inbox/dedupe + DLQ routing working (`:277`).
7. `embedding-service` — BGE-small-en 384-dim on Ray Serve (GPU); the first at-least-once
   consumer; **only writer of Qdrant**; embed-on-ingest at batch 32 sub-100 ms into per-tenant
   `{collection}__{org_id}`; `embedding-events` acks; consumer lag < 5 s (`:372`).
8. `reranker-service` — BGE-reranker-base cross-encoder, `POST /rerank` top-50 → top-10, warm
   p95 < 500 ms, with a warm replica floor (`:466`).
9. `ingestion-service` — unstructured.io → typed elements → `document_chunks` +
   `document-chunks` topic + `nexus-documents` bucket; LangChain `RecursiveCharacterTextSplitter`
   600/60; PII redaction gated by `tenant_settings` (`:496`).
10. `search-service` — Postgres FTS (tsvector) arm of hybrid retrieval; `GET /search` returns
    org-scoped hits under RLS (`:544`).
11. `mytheus-mobile` — person-profile screen (`:610`).

**P6 demo gate (`:653`):** upload a PDF and watch ingest → bucket → `document_chunks` →
`document-chunks` → embedding consumer → `document_chunks__{org_id}`; force a poison message and
show it land in the DLQ while the partition keeps moving; embedding-lag and DLQ-rate panels green.

---

## Plan 7-9 — Agents and Monetization (P7 W18–20, P8 W21–23, P9 W24–26)
`plan/plan-7-9-agents-and-monetization.md`

**Goal:** turn live `transcript-chunks` into personalized, cost-bounded agent value. Billing is
built **first** because the R4 cost bomb detonates the moment agents run unmetered (`:12`).

1. `billing-metering-service` — **BUILD FIRST.** `plans`, `subscriptions`, `invoices`,
   `usage_events`, `usage_aggregates`, `agent_budgets`, `quotas` migrated with RLS FORCE; the
   `usage-events` consumer with idempotent dedupe; the Redis budget mirror; the
   `ok → throttled → blocked` state machine; 80% / 95% alerts; an alert-only → enforce flag
   (`:67`).
2. `agent-runtime-service` — `BaseAgent` + `AgentState` + versioned `PromptLoader` + the
   cost-controlled pipeline: 30-second windowing → Haiku relevance pre-gate → LiteLLM tiered
   routing (Haiku / Sonnet / Opus) → semantic + Anthropic prompt caching → inline `check_budget`;
   `usage-events` emission; the `agent-stream` bridge; **writes zero Postgres** (`:245`).
3. `PersonalListenerAgent` — spawned per participant per meeting; relevance pre-gate against
   `get_user_context`; worklogs written through `core-api`; LLM-egress PII redaction (`:356`).
4. `web-app` — worklog "My Notes" panel, agent sidebar shell, and the Usage / budget console
   section (`:429`).
5. `DecisionAgent` + `ActionAgent` — **candidates only, no silent commits** (`:444`).
6. `core-api` — the **SOLE confirm-only producer** of `decisions` / `actions`; the `decisions`,
   `action_items`, `agent_suggestions` tables; the confirm endpoints (`:466`).
7. `knowledge-graph-service` — AGE `nexus_knowledge`, per-label `org_id` index, `$org_id` bound
   on every Cypher, edge creation on confirmation only, four named templates, `graph-events`.
   **The only writer of `nexus_knowledge`** (`:546`).
8. `web-app` — decision/action confirm cards, My Tasks, decision-lineage view (`:609`).
9. `rag-service` — `expand → retrieve → rerank → assemble → cite` over the internal gRPC mesh;
   hybrid RRF (k=60) over Qdrant + Postgres FTS + AGE; citations with uncited passages dropped;
   a 10-minute answer cache; expansion-skip when the tenant is throttled; holds no model
   in-process (`:623`).
10. `QuestionAgent` — real-time suggestion cards, `agent_suggestions` priority 1–3, Ask /
    Dismiss / Save-for-later, Save persisting into the next meeting (`:676`).
11. `mytheus-mobile` — agent sidebar, My Notes, My Tasks, decision lineage, Ask (RAG) screen;
    remove the recap/insights mock screens (`:753`).
12. Eval-harness gates that must pass for every shipped prompt: extraction F1, relevance
    precision/recall, question usefulness, citation validity (`:817`).

---

## Plan 10-13 — Collaboration, Integrations, GA (P10 W27–29, P11 W30–31, P12 W32–33, P13 W34–36)
`plan/plan-10-13-collab-integrations-ga.md`

**Goal:** turn the working single-meeting product into the full differentiator and put it in
front of paying tenants. This is the last 10 weeks of the frozen 36-week backbone and ends at the
W36 MVP go-live boundary; everything after is Plan 14 and explicitly not here (`:16`).

1. `workflow-service` — `MeetingLifecycleWorkflow` real activities replacing the stubs (open →
   transcribe-watch → post-process → close) (`:80`).
2. `identity-tenant-service` — the A2A **PDP** and the `agent_authz_scopes` deny-by-default grant
   store (`:154`).
3. `agent-runtime-service` — `ContextGapAgent`, the A2A **PEP**, and the fallback chain
   peer-agents → related-meeting agents → RAG → silent, inside a p95 < 1.5 s budget (`:210`).
4. `core-api` — `agent_collaborations` persistence + the collaboration-log read API (`:278`).
5. `web-app` — context-fill cards + the `agent_collaborations` log UI (`:298`).
6. `agent-runtime-service` — `ProactiveAgent`: conflict / workload / missing-stakeholder /
   dependency detectors (`:316`).
7. `realtime-gateway` — the Redis pub/sub backbone, `agent-stream` Avro delta fan-out, and the
   per-user priority queue `nexus:{org_id}:queue:agent:{user_id}` (`:349`).
8. `notification-service` — in-app / email / Slack delivery plus the priority queue (`:382`).
9. `web-app` — proactive alert UI + sidebar polish (`:401`).
10. `integration-service` — the OAuth engine, `integration_connections`, `oauth_tokens`
    (envelope-encrypted), and the webhook receiver (`:419`).
11. `integration-service` connectors — Google Calendar + Slack + **one** of Jira/Linear, the R8
    guardrail (`:470`).
12. `workflow-service` — `ActionFollowUpWorkflow`: reminders on day 1/3/7 and escalation to the
    manager via Slack DM at day 10 (`:496`).
13. `admin-ops-service` — both consoles plus `feature_flags`, `ops_audit_log`, `support_grants`
    (`:535`).
14. `web-app` — integration settings page + the tenant-admin and internal-ops consoles (`:558`).
15. `billing-metering-service` — the full Stripe surface: subscriptions, invoices, plans, seats,
    dunning, tax (`:575`).
16. Playwright E2E (web + mobile) and k6/Locust load suites (`:596`).
17. ~30% LLM cost optimization (C4) measured against the live pipeline (`:611`).
18. Final security attestation (`:626`).
19. Postgres backups + PITR + read replica; `audit_logs` partitioning and archival to
    `nexus-audit-archive` (`:640`).
20. Cloudflare + custom domain + the live production deploy on the single-node VPS (`:654`).
21. `mytheus-mobile` — proactive alerts, priority-queue surfacing, Expo push (the absent push
    layer), responsive parity, inclusion in the E2E/GA release pipeline (`:722`).

**Headline demo (P12, `:762`):** an admin OAuth-connects Jira; a confirmed `action_items` row
becomes a real Jira issue; moving it to Done in Jira flips `action_items.status` (loop-suppressed);
a day-1 Slack DM fires; a Calendar event auto-creates a meeting and ingests its attachment.

---

## Plan 14 — Scale-Out (post-W36, demand-driven, NOT a calendar band)
`plan/plan-14-scale-out.md`

**Goal:** peel the monolith into the canonical services on Kubernetes in the frozen Wave 1-4
order; stand up Terraform IaC + GitOps; add cell-based isolation for Enterprise; evolve
single-region → multi-region active-passive → active-active (in-zone only); execute the capacity
escalation paths. Each sub-phase fires only when a real load signal trips its trigger (`:16`).

1. **P14.1 substrate** — `mytheus-platform` scaffold (`terraform/`, `helm/nexus/`, `keda/`,
   `argo/`, `chaos/`, `runbooks/`, `loadtest/`); Terraform layers with state split per
   `(account, region, layer)`; platform operators ESO/KEDA/Argo in the `platform` namespace; a
   Helm umbrella chart; and a **new `nexus-deploy` GitOps repo** holding rendered values per
   `(env, region)` (`:71–211`).
2. **P14.2 Wave 1** (different runtime from birth) — `embedding-service` + `reranker-service`
   (GPU / Ray Serve / vLLM, KEDA on queue depth + GPU util, 2-GPU replica floor);
   `agent-runtime-service` (token-scaled); `transcription-service` + `realtime-gateway`
   (connection-scaled, connection-drain rollouts); `workflow-service` (Temporal worker
   versioning). Plus the standing rule that **all** database migrations are expand/contract
   (`:239–355`).
3. **P14.3 Wave 2** (CRUD core) — `core-api` then `identity-tenant-service` extracted to
   `prod-crud`; HPA on req/s + p95; the PgBouncer ceiling must hold (`:361`).
4. **P14.4 Wave 3** (SaaS business) — `billing-metering-service`, `integration-service`,
   `admin-ops-service`; integrations moved off the critical path (`:402`).
5. **P14.5 Wave 4** (retrieval stores) — `knowledge-graph-service` and `search-service` split out
   of `rag-service`; the AGE arm must hot-drop; `$org_id` templates only (`:424`).
6. **P14.6 cells + multi-region** — cell-based isolation for Enterprise on tainted node pools
   with RLS still FORCE inside the cell; multi-region active-passive → active-active candidate,
   **in-zone only**, with Cloudflare failover and per-plan SLO tiers (`:444–465`).
7. **P14.7 capacity escalations**, each independently trigger-gated (`:489–527`):
   a. Postgres: cells → Citus shard-by-`org_id`.
   b. Graph: AGE → Neptune / Neo4j Aura exit, with the relational MV floor.
   c. Redpanda BYOC / dedicated multi-AZ.
   d. int8 Qdrant quantization (4.42B vectors / ~17 TB).
   e. GPU autoscaling.
   f. The C4 ~30% LLM cost reduction realized at volume.

**Band DoD (`:628`):** the monolith fully decomposed into the canonical services on Kubernetes
across the residency zones in use; per-plan availability tiers (Free 99.5% / Pro 99.9% /
Enterprise 99.95%) met and chaos-proven; **no canon name, number, or seam re-cut.**

---

# CANONICAL SERVICE CATALOG

`_canon/01-service-catalog.md:102–124`. This file self-declares as the **single source of truth
for service names** (`:21`). The coarser labels used in `_canon/02-data-model.md` §1.1
(`identity-service`, `agent-service`, `graph-service`, `billing-service`, `budget-service`,
`tenant-service`, `audit-service`) are **not** service names and must never be used
(`22-roadmap-and-phasing.md:34`).

**Edge / client tier**

| Service | Intended responsibility |
|---|---|
| `api-gateway` | Single ingress: TLS termination, WAF, global rate limiting, Clerk JWT pre-validation, `org_id` extraction, routing to `bff`/services. No business logic. |
| `web-app` | Next.js 14 App Router UI — dashboard, meeting room, agent sidebar, transcript, RAG panel, admin/tenant consoles, integration settings. Calls `bff` only. |
| `bff` | Backend-for-frontend: aggregates and denormalizes reads for screens, fans out to services, shapes payloads. Thin; owns no domain. |

**CRUD / control plane (scales with users)**

| Service | Intended responsibility |
|---|---|
| `identity-tenant-service` | Identity and tenancy control plane: Clerk webhook user-sync, org/tenant provisioning and lifecycle, RBAC, SSO/SAML/SCIM, A2A authorization grants (the PDP, deny-by-default), `get_user_context(user_id)` source-of-record, and sole writer of the `pii_tokens` vault. |
| `core-api` | Domain CRUD heart: meetings, teams, team_members, departments, participants, chat persistence, worklogs, decisions/actions read models, profiles/org-chart. The **sole confirm-only producer** of `decisions`/`actions`. |
| `meeting-service` | LiveKit transport adapter: mints room tokens with participant verification, receives LiveKit webhooks, bridges the server-side audio hook to `transcription-service`. Owns `meeting_rooms`. |
| `integration-service` | External integrations: Google Calendar OAuth, Slack bot OAuth, Jira (default) / Linear (alt) bidirectional action-item sync and status webhooks. Guardrail R8. |
| `billing-metering-service` | SaaS monetization: Stripe billing/subscriptions/seats/dunning/tax, usage metering, plans/tiers, per-tenant token budgets and hard quotas, billing read models. |
| `notification-service` | Unified delivery fabric — in-app (via `realtime-gateway`), email, Slack. Templating, per-user priority queue, dedupe, delivery receipts. The only consumer authorized to fan `notifications` to external channels. |
| `admin-ops-service` | Two consoles: tenant admin (org-scoped self-serve) and internal ops (cross-tenant support, feature flags, tenant suspend/migrate, lag and cost dashboards, audit-log access). |

**Real-time tier (scales with concurrent meetings)**

| Service | Intended responsibility |
|---|---|
| `realtime-gateway` | Socket.io/WSS fan-out for non-AV events: chat, presence, live captions, agent sidebar output, streamed agent tokens, priority alert queue. Redis pub/sub backbone; rooms keyed by `meeting_id`. |
| `transcription-service` | Deepgram WebSocket proxy: keeps the API key server-side, streams audio from the LiveKit hook, speaker diarization, interim-vs-final separation, persists **only finals** to `transcript_chunks`, emits chunks to the bus. |

**AI / inference tier (scales with tokens and vectors)**

| Service | Intended responsibility |
|---|---|
| `ingestion-service` | Document ingestion via unstructured.io: PDF/DOCX/PPTX/HTML/MD/EML → typed elements, LangChain splitting (600/60) + SemanticChunker, PII redaction pass, emits `document-chunks`. |
| `embedding-service` | Dedicated GPU inference for BGE-small-en 384-dim. Embed-on-ingest (batch 32, sub-100 ms) and query-embed only. Upserts to per-tenant Qdrant collections. Off the API tier. |
| `reranker-service` | Dedicated GPU inference for BGE-reranker-base cross-encoder. Reranks top-50 → top-10 for `rag-service`. Off the API tier. |
| `ner-service` | Dedicated inference for the lightweight NER model behind the PII redaction pass. `POST /ner`. Explicitly **not** inside `transcription-service`, which carries the p95 < 2 s caption SLO (ADR-0020; `:115`). |
| `agent-runtime-service` | Hosts the six LangGraph agents on the shared `BaseAgent` lifecycle + `AgentState` + versioned prompt loader; runs the cost-controlled LLM pipeline; is the A2A PEP; multi-agent collaboration over the bus. Writes no Postgres. |
| `rag-service` | The `RAGService` pipeline: expand → retrieve → rerank → assemble → cite. Hybrid retrieval (Qdrant vector + Postgres FTS + AGE relations, RRF fusion), citation tracking, 10-min Redis answer cache. No model in-process. |
| `knowledge-graph-service` | Owns the Apache AGE graph `nexus_knowledge`: node labels and edges, pre-built Cypher templates, org-partitioned, edges created on confirmation, soft delete. Sole writer. |
| `search-service` | Keyword/lexical retrieval: Postgres FTS over `transcript_chunks` and `chat_messages`; in-page transcript search; the FTS arm of hybrid retrieval. |

**Async / orchestration**

| Service | Intended responsibility |
|---|---|
| `workflow-service` | Temporal workers: `MeetingLifecycleWorkflow`, `ActionFollowUpWorkflow` (day 1/3/7 reminders, day-10 manager escalation), tenant-provisioning and right-to-be-forgotten deletion workflows, billing dunning sagas. Durable orchestration only. |

That is 21 entries; the plans still say "20 services" because ADR-0020 added `ner-service` after
that phrasing was frozen.

**Scaling families and autoscaler signals (`:132–137`):** CRUD/control plane on HTTP req/s + p95;
Real-time on concurrent WebSocket connections / live audio streams; AI-inference on queue depth,
GPU utilization and token throughput; Async on Temporal task-queue depth.

**Synchronous API ownership, no overlap (`:195–214`):** `/clerk/webhook`, `/orgs`, `/rbac/*`,
`/scim/v2/*`, `/context/{user_id}` → `identity-tenant-service`; `/meetings`, `/teams`,
`/departments`, `/worklogs`, `/decisions`, `/actions`, `/chat` → `core-api`;
`/api/meetings/:id/token`, `/livekit/webhook` → `meeting-service`; Socket.io rooms →
`realtime-gateway`; `/ingest`, `/ingest/jobs/:id` → `ingestion-service`; `/embed`, `/rerank`
(internal) → embedding/reranker; `/agents/spawn`, `/agents/{id}/terminate` (internal) →
`agent-runtime-service`; `/api/rag/query` → `rag-service`; `/graph/query/{template}`,
`/graph/cypher` (internal) → `knowledge-graph-service`; `/search` → `search-service`;
`/oauth/{provider}`, `/webhooks/{provider}` → `integration-service`; Temporal queues +
`/workflows/*` → `workflow-service`; `/billing/*`, `/stripe/webhook`, `/usage` →
`billing-metering-service`; `/notifications`, `/notifications/:id` → `notification-service`;
`/admin/tenant/*`, `/admin/ops/*` → `admin-ops-service`; `/bff/*` → `bff`; all external ingress →
`api-gateway`.

**Modular-monolith → services split order (`:218–225`):** Wave 1 always-separate (embedding,
reranker, agent-runtime, transcription, realtime-gateway, workflow) → Wave 2 `core-api` then
`identity-tenant-service` → Wave 3 billing-metering, integration, admin-ops → Wave 4
knowledge-graph, search (start as libraries inside `rag-service`).

---

# CANONICAL EVENT TAXONOMY

`_canon/03-event-taxonomy.md`. **21 topics** (9 PDF-original + 12 SaaS-added). Every topic has a
paired `<topic>.dlq` at 14-day retention. Delivery is **at-least-once, idempotent in effect**:
every consumer dedupes via the durable `inbox` table plus the Redis cache
`nexus:{org_id}:dedupe:{topic}:{idempotency_key}` (24h TTL) (`:7`).

| # | Topic | Producer → Consumers (as SPECIFIED) | Avro value | Partition key | Retention | Idempotency key |
|---|---|---|---|---|---|---|
| 1 | `transcript-chunks` | `transcription-service` → `embedding-service`, `agent-runtime-service`, `search-service` | `TranscriptChunk` | `{org_id}:{meeting_id}` | 7d delete | `chunk:{chunk_id}` |
| 2 | `document-chunks` | `ingestion-service` → `embedding-service`, `agent-runtime-service` | `DocumentChunk` † | `org_id` | 7d delete | `docchunk:{document_id}:{element_id}` |
| 3 | `agent-results` | `agent-runtime-service` → `core-api`, `realtime-gateway`, `notification-service` | `AgentResult` | `org_id` | 3d delete | `agentresult:{meeting_id}:{window_id}:{user_id}:{agent}` |
| 4 | `decisions` | `core-api` (outbox, **on confirm only**) → `knowledge-graph-service`, `notification-service` | `Decision` | `org_id` | 30d delete | `decision:{decision_id}:confirmed` (+ `MERGE`) |
| 5 | `actions` | `core-api` (outbox, **on confirm only**) → `knowledge-graph-service`, `workflow-service`, `integration-service` | `Action` | `org_id` | 30d delete | `action:{action_id}:confirmed` (+ `MERGE`) |
| 6 | `notifications` | multiple → `notification-service`, `realtime-gateway` | `Notification` | `org_id` | 3d delete | `notif:{source_event_id}:{channel}:{user_id}` |
| 7 | `agent-requests` | `agent-runtime-service` → `agent-runtime-service` (peer agents) | `AgentRequest` | `org_id` | 1d delete | `areq:{correlation_id}` |
| 8 | `agent-responses` | `agent-runtime-service` → `agent-runtime-service` (requester) | `AgentResponse` | `org_id` | 1d delete | inbox + Redis dedupe |
| 9 | `agent-stream` | `agent-runtime-service` (LiteLLM stream bridge) → `realtime-gateway` → WebSocket | `AgentStreamDelta` † | `{org_id}:{meeting_id}` | 1h delete | throwaway; shed first under backpressure |
| 10 | `tenant-events` | `identity-tenant-service` → `billing-metering-service`, `admin-ops-service`, `workflow-service` | `TenantEvent` | `org_id` | retain (compact) | inbox + Redis dedupe |
| 11 | `meeting-events` | `core-api` → `meeting-service`, `workflow-service`, `integration-service` | `MeetingEvent` | `org_id` | 7d delete | inbox + Redis dedupe |
| 12 | `meeting-room-events` | `meeting-service` → `transcription-service`, `core-api` | `MeetingRoomEvent` | `org_id` | 3d delete | inbox + Redis dedupe |
| 13 | `integration-events` | `integration-service` → `ingestion-service`, `core-api` | `IntegrationEvent` | `org_id` | 7d delete | inbox + Redis dedupe |
| 14 | `usage-events` | `agent-runtime-service`, `rag-service` → `billing-metering-service` | `UsageEvent` | `org_id` | 7d delete | `usage:{org_id}:{event_type}:{occurred_at}:{source_id}` |
| 15 | `billing-events` | `billing-metering-service` → `workflow-service`, `notification-service`, `admin-ops-service` | `BillingEvent` | `org_id` | 30d delete | inbox + Redis dedupe |
| 16 | `graph-events` | `knowledge-graph-service` → `admin-ops-service` | `GraphEvent` | `org_id` | 7d delete | inbox + Redis dedupe |
| 17 | `embedding-events` | `embedding-service` → (lag / metrics only) | `EmbeddingEvent` | `org_id` | 1d delete | inbox + Redis dedupe |
| 18 | `presence-events` | `realtime-gateway` → `core-api` (analytics) | `PresenceEvent` | `org_id` | 1h delete | inbox + Redis dedupe |
| 19 | `notification-delivery-events` | `notification-service` → `admin-ops-service`, `core-api` | `NotificationDeliveryEvent` | `org_id` | 30d delete | inbox + Redis dedupe |
| 20 | `user-sync-events` | `identity-tenant-service` (Clerk inbox) → `identity-tenant-service` | `UserSyncEvent` | `org_id` | 7d delete | inbox + Redis dedupe |
| 21 | `admin-events` | `admin-ops-service` → `workflow-service`, `admin-ops-service` (audit-log write) | `AdminEvent` | `org_id` | retain (compact) | inbox + Redis dedupe |

† `DocumentChunk` and `AgentStreamDelta` are net-new names introduced by `07-event-architecture.md`
and are **not** frozen canon. Canon freezes only **seven** schema names: `TranscriptChunk`,
`AgentResult`, `Decision`, `Action`, `Notification`, `AgentRequest`, `AgentResponse`. The twelve
`*Event` names for the SaaS topics are likewise net-new (`:33`).

**Partition-key rule (ADR-0017, `:7`):** 19 of 21 topics partition on `org_id` alone.
`transcript-chunks` and `agent-stream` use the composite `{org_id}:{meeting_id}` — it preserves
within-meeting speech ordering while letting a hot enterprise tenant spread across partitions.
`org_id` remains present and non-nullable in the envelope on every topic and is the leading
component of the composite.

**Frozen `EventEnvelopeHeader`** (namespace `io.nexus.events`, `:37–48`) — field order frozen by
convention:

| Field | Type | Role |
|---|---|---|
| `event_id` | string (uuid) | Unique per emitted event; distinct from `idempotency_key`. |
| `idempotency_key` | string | Business dedupe key; stable across producer retries of the same fact. |
| `org_id` | string (uuid) | Tenant root; RLS/cell scoping anchor; the partition key (or its leading component). |
| `schema_version` | int (default 1) | Producer's payload schema version, for observability. |
| `producer` | string | Owning service name, kebab-case, from the Service Catalog. |
| `occurred_at` | long (timestamp-micros) | When the fact happened (UTC), not enqueue time. |
| `correlation_id` | ["null","string"] default null | Ties a causal chain across topics; the span join key for async tracing. |
| `causation_id` | ["null","string"] default null | `event_id` of the direct cause; null for root events. |

**Registry contract (`:50–52`):** subject naming `{topic}-value`, one subject per topic, no key
schemas (the partition key is the raw `org_id` UUID string). Compatibility is **BACKWARD** —
allowed: add a field with a default, remove a field that had a default, widen via union with null
+ default, append enum symbols with a default; forbidden: add a required field without a default,
remove a non-defaulted field, rename, narrow a type. Enforcement: CI `check-compatibility`
against the registered subject gates every producer image build.

**Delivery mechanics (`_canon/01-service-catalog.md:185–187`, `07-event-architecture.md:289–435`):**
producers write through a transactional outbox in the same DB transaction as their state change;
a relay publishes to Redpanda with `acks=all` and idempotence enabled; consumers dedupe on
`idempotency_key` via an inbox table plus the Redis cache; poison messages land in a per-topic
DLQ. The `decisions`/`actions` edges into `knowledge-graph-service` are created **only on
confirmation** — no silent commits.

---

# CANONICAL DATA MODEL

`_canon/02-data-model.md`

**PostgreSQL — PDF-original tables, names carried verbatim (§2.1, `:74–89`):**
`organizations` (the tenant root; `id` = `org_id`; deliberately **not** `tenant_isolation`-policied),
`users` (role, `skills text[]`, bio, avatar_url), `teams`, `team_members`, `departments`
(self-referential org tree), `meetings`, `meeting_participants` (+ `consented_at`,
`consent_version` from ADR-0018), `transcript_chunks` (index on `(meeting_id, start_ms)`, finals
only, monthly range partition), `meeting_speakers` (diarized `speaker_id` → `user_id`,
manual/suggested), `worklogs`, `decisions`, `action_items`, `agent_suggestions` (priority 1–3,
status Ask/Dismiss/Save-for-later), `agent_collaborations`, `chat_messages`, `document_chunks`.

**PostgreSQL — SaaS-scale additions (§2.2, `:101–113`):**
`plans` (global catalog, non-tenant), `subscriptions`, `usage_events` (append-only, day
range-partitioned), `invoices` (immutable once finalized, 7-year retention), `audit_logs`
(append-only, no UPDATE/DELETE, monthly partition), `integration_connections`, `oauth_tokens`
(envelope-encrypted under a per-tenant KMS data key), `tenant_settings` (+ `consent_required`),
`data_residency` (`us`/`eu`/`ap`, immutable after `locked_at`), `rbac_grants` (`tier` is the only
thing authorization reads — strictly distinct from `users.role`), `agent_budgets`, `pii_tokens`
(the reversible per-tenant vault; keyed HMAC not a plain digest; **not yet migrated**, ADR-0020
blocks on key custody), `meeting_rooms`.

Additional tables named by the roadmap's per-phase store table (`22-roadmap-and-phasing.md:100–116`)
and by the plans: `usage_aggregates`, `quotas`, `feature_flags`, `ops_audit_log`,
`support_grants`, `agent_authz_scopes`, plus the infrastructure tables `outbox` and `inbox`.

**Qdrant vector collections (§3, `:265–267`)** — three logical collections, all 384-dim
BGE-small-en cosine, each realized as a per-tenant physical collection `{name}__{org_id}` with a
**mandatory `org_id` payload filter** on every query (ADR-0006 defense in depth):
`transcript_chunks`, `document_chunks`, `decisions` (the last enables semantic conflict detection
for `ProactiveAgent`).

**Apache AGE graph `nexus_knowledge` (§4, `:296–310`)** — org-partitioned.
Node labels: `Person`, `Meeting`, `Decision`, `Action`, `Topic`.
Edge labels: `ATTENDED` (Person→Meeting), `PRODUCED` (Decision→Meeting, also Action→Meeting),
`ASSIGNED_TO` (Action→Person), `RELATES_TO` (Decision/Action/Meeting→Topic), `FOLLOWS_FROM`
(Decision→Decision, lineage/supersession).

**Redis key namespaces (§5, `:336–343`)** — all prefixed `nexus:{org_id}:`:
`user_ctx:{user_id}` (5 min), `rag:{user_id}:{query_hash}` (10 min), `presence:{meeting_id}`
(meeting lifetime), `pubsub:meeting:{meeting_id}` (channel), `queue:agent:{user_id}` (session),
`dedupe:{topic}:{idempotency_key}` (24h), `budget:{period}` (period-end),
`ratelimit:{user_id}` (sliding window).

**Object-storage buckets (§6, `:357–362`)** — all keyed `{org_id}/…`, on Cloudflare R2 per
ADR-0023: `nexus-recordings`, `nexus-documents`, `nexus-attachments`, `nexus-invoices` (7-year,
object lock), `nexus-audit-archive` (write-once), `nexus-exports` (auto-expire 7 days).

---

# ADR DECISIONS — 23 one-liners

1. **0001 Event-driven microservices** — adopt event-driven microservices deployed *first as a
   modular monolith*, split along pre-drawn seams as load arrives.
2. **0002 Multi-tenancy model** — pooled multi-tenancy with PostgreSQL Row-Level Security for the
   long tail; cell-based isolation for large/enterprise tenants. Cells are a strategy *within* the
   style, not a competing architecture.
3. **0003 Meeting SDK** — lock **LiveKit**, the open-source self-hostable SFU, as the meeting
   transport, always reached through a transport adapter that keeps the lock reversible.
4. **0004 LLM cost-control pipeline** — five fixed stages executed **window-then-gate**: 30-second
   windowed batching → Haiku relevance pre-gating (~35% pass) → tiered routing Haiku/Sonnet/Opus
   via LiteLLM → semantic + Anthropic prompt caching → per-tenant token budgets with hard quotas.
   Agents run per window, never per raw `transcript_chunks` row.
5. **0005 Graph store** — run `nexus_knowledge` on **Apache AGE** (Cypher over PostgreSQL)
   co-located in the managed primary, org-partitioned on `org_id`, holding two pre-engineered
   exits: Neptune / Neo4j Aura, or a relational materialized-view fallback.
6. **0006 Vector store tenancy** — one physical Qdrant collection per tenant
   (`{logical}__{org_id}`) **and** a mandatory `org_id` payload filter on every query; isolation
   at both the collection and the payload boundary.
7. **0007 Event delivery / idempotency** — at-least-once with idempotency keys, exactly-once in
   effect: transactional outbox in the state-change transaction → Redpanda → Avro deserialize →
   Redis dedupe → inbox primary key.
8. **0008 Dedicated inference serving** — `embedding-service` and `reranker-service` as two
   separate stateless GPU-backed Ray Serve deployments with independent autoscalers, completely
   off the API tier; `rag-service` stays a CPU orchestrator with no model in-process; vLLM is
   reserved for larger LLM paths, not the BGE models.
9. **0009 Data residency** — zone-pinned data residency with in-zone multi-region topology
   (`us`/`eu`/`ap`); no cross-zone replication.
10. **0010 Auth** — keep **Clerk** as identity provider and source of truth, extend it with
    enterprise SSO/SAML/SCIM, and replace the bare webhook with a three-part hardened user-sync.
11. **0011 Billing and metering** — Stripe is the system of record for monetization;
    `billing-metering-service` owns the metering pipeline (`usage-events` → dedupe on a canonical
    idempotency key → day-partitioned `usage_events` → nightly `usage_aggregates` → Stripe metered
    prices) and deny-by-default, fail-closed quota enforcement that degrades discretionary LLM
    work before it blocks and never sheds durable facts.
12. **0012 Realtime transport** — extract non-AV fan-out into a dedicated `realtime-gateway` tier:
    Socket.io over WSS, rooms keyed by `meeting_id`, a managed multi-AZ Redis pub/sub backbone,
    sticky connections with stateless pods, a per-user priority queue, autoscaled on concurrent
    WebSocket connections independently of transcription and CRUD.
13. **0013 Agent-to-agent authorization** — a PEP/PDP split on every A2A request, fail-closed:
    `identity-tenant-service` is the deny-by-default PDP owning the grant store,
    `agent-runtime-service` is the PEP checking before publish and before process; out-of-scope →
    DENY and audited; foreign `org_id` → DLQ, never served.
14. **0014 Compute platform** — Kubernetes (EKS or GKE) provisioned and governed by Terraform IaC;
    PaaS only for the early-stage build; multi-region active-passive evolving to active-active.
15. **0015 Observability** — OpenTelemetry as the single vendor-neutral instrumentation contract
    exporting OTLP to a Collector DaemonSet, fanning out to Prometheus + Grafana (with
    Thanos/Mimir for 13-month global query), Tempo for traces, Loki for logs, Sentry for
    exceptions, PagerDuty for alerting. `org_id` on every signal; `user_id`, `meeting_id`,
    `chunk_id`, `request_id`, `trace_id` are **never** Prometheus labels. SLOs are imported from
    the capacity math, never minted here.
16. **0016 MVP substrate** — the 36-week MVP backend runs on a **single self-hosted VPS under
    Docker Compose**: one multi-stage image running three roles (`web` uvicorn monolith, `relay`
    outbox drain, `workflow` Temporal worker); self-hosted postgres, redis, redpanda (+ Schema
    Registry), temporal; a one-shot `migrate` service running `alembic upgrade head`; Caddy as
    reverse proxy with automatic HTTPS. Kubernetes, Terraform, cells and multi-region stay out of
    scope until Plan 14. Railway and Render are struck entirely. The per-store split is preserved
    so any single store can move to a managed provider independently.
17. **0017 Composite partition key** — `transcript-chunks` and `agent-stream` partition on
    `{org_id}:{meeting_id}`; the other 19 topics keep `org_id` alone; `org_id` stays non-nullable
    in the envelope everywhere.
18. **0018 Consent entity behind a flag** — add `tenant_settings.consent_required boolean NOT NULL
    DEFAULT false`, `meeting_participants.consented_at`, `meeting_participants.consent_version`;
    ratify the 428 Precondition Required response from `POST /api/meetings/:id/token` when consent
    is required and absent. **The migration and the meeting-service check are explicitly out of
    scope for the ADR.**
19. **0019 AI tier verifies JWT** — every REST route in `mytheus-intelligence` verifies the Clerk
    session token in its own process via a single `verified_org_id` dependency (`/ingest`,
    `/ingest/jobs/:id`, `/search`, `/embed`, `/rerank`); the `X-Org-Id` header binding is
    **removed, not deprecated**; the claim carries the already-resolved internal `org_id` UUID
    read from `CLERK_JWT_INTERNAL_ORG_CLAIM` (default claim name `internal_org_id`); RS256 against
    cached JWKS honoring rotation, `nbf` required, JWKS outage → 503 not 500.
20. **0020 PII token vault and NER placement** — `identity-tenant-service` owns and is the only
    writer of `pii_tokens` (per-tenant reversible vault, envelope-encrypted ciphertext, **keyed**
    HMAC so the same raw value mints the same token across all three redaction boundaries;
    resolving a token is a PII disclosure and writes `audit_logs`); NER runs as a new
    `ner-service` on the inference tier, never on the API tier and specifically not inside
    `transcription-service`. **Accepted for the data model and call graph; blocked for
    implementation on a key-custody precondition no part of the program provides.**
21. **0021 Explicit grants** — no schema-wide default privilege grants on TABLES in `public`;
    every migration creating a tenant table issues its own GRANT beside its RLS policy; a
    partition carries no direct grant to any role (access is through the parent); no role holding
    `rolbypassrls` holds any privilege on a tenant table. All four are standing CI assertions.
22. **0022 OpenAPI emitted from code** — each service's OpenAPI document is emitted from its
    FastAPI app by `scripts/emit_openapi.py` (services discovered by walking `services/*/main.py`,
    not a hand-maintained list), committed under `design/openapi/`, and gated three ways:
    `emit_openapi --check` (route changed without regenerating), `spectral lint` (house rules —
    `org_id` in a path, offset pagination, bare `application/json` errors), and `oasdiff breaking`
    (deleted path, newly-required request field). None subsumes another.
23. **0023 Object storage** — **Cloudflare R2** via its S3-compatible API. Chosen for zero egress
    fees (the decisive reason on a read-heavy, user-triggered recordings workload), no new vendor
    or DPA counterparty, and S3 compatibility so the SDK, `{org_id}/` key prefixes, lifecycle
    rules and RTBF prefix-delete all work unchanged. Amazon S3 explicitly rejected.

---

# INTENDED FRONTEND SCOPE

## From `20-frontend-architecture.md`

**Rendering architecture (`:62–73`).** Next.js 14 App Router, **React Server Components first**.
RSC by default for dashboard cards, transcript view, decision history, My Tasks, org chart,
admin/tenant consoles, integration settings and billing. `"use client"` only for the meeting
room, agent sidebar, in-meeting chat, live captions, RAG context panel, and any form with
optimistic state. **Server Actions** for every write path — confirm/edit/dismiss a decision,
assign/complete an action, dismiss/ask/save a suggestion, save integration settings, profile
edits — each calling the canonical endpoint through `bff`/`api-gateway` then `revalidateTag()`.
Route handlers under `app/api/*` are thin only: Clerk session helpers, the LiveKit token proxy,
health.

**Route tree (`:75–120`).**
`(marketing)/` — landing, `pricing`.
`(auth)/` — `sign-in/[[...rest]]`, `sign-up/[[...rest]]`.
`(app)/` — `layout.tsx` establishing `<OrgProvider>` + `<AppShell>`; `dashboard`; `meetings/page.tsx`
(list), `meetings/[meetingId]/page.tsx` (detail, pre/post), `meetings/[meetingId]/room` (client),
`meetings/[meetingId]/transcript` (RSC); `decisions`; `tasks`; `org`; `search`;
`settings/{profile,integrations,billing}`.
`(admin)/` — `tenant/page.tsx`, `ops/page.tsx`, RBAC-gated.
`api/` — `livekit-token/route.ts`, `health/route.ts`.
The `(app)/layout.tsx` is the single place org/tenant context is established; children inherit a
typed `OrgContext` so no screen re-derives tenancy.

**Data-fetching contract (`:129–137`).** RSC reads fetch from `bff` screen-shaped endpoints or a
read endpoint the catalog assigns to `core-api`/`rag-service`/`search-service`, server-side with
the forwarded JWT. Cache tags keyed by resource (`meeting:{id}`, `tasks:{userId}`); Server Actions
`revalidateTag` after a write. The client duplicates no server cache.

**Design system (`:141–217`).** Tailwind CSS + shadcn/ui (copy-in, not a dependency black box)
over a design-token layer expressed as CSS custom properties that Tailwind consumes, so dark
mode, density and per-tenant theming flip by swapping a token set. Frozen PDF-specified
primitives: `Button`, `Input`, `Modal`, `Card`, `EmptyState`, `Avatar`, `Badge`. Localization and
accessibility are explicit sections.

**The meeting room (`:218–248`).** Layout §4.1; token minting and join §4.2. Video scoped to
**1–9 tiles, 6 nominal**.

**The agent sidebar — "the hardest UI problem" — frozen nine-section order (`:254–288`):**

| # | Section | Source agent | Backing entity | Actions |
|---|---|---|---|---|
| 1 | My Notes (live worklog) | `PersonalListenerAgent` | `worklogs` via `agent-results` | read-only stream, pin |
| 2 | Suggestion cards | `QuestionAgent` | `agent_suggestions` (priority 1–3) | **Ask / Dismiss / Save** |
| 3 | Decision detected | `DecisionAgent` | `decisions` | **Confirm / Edit / Dismiss** |
| 4 | New task toast | `ActionAgent` | `action_items` | Confirm / Edit assignee+date / Dismiss |
| 5 | Context-fill card | `ContextGapAgent` | `agent_collaborations` | open source meeting |
| 6 | Conflict banner (red) | `ProactiveAgent` | `decisions` via AGE `FOLLOWS_FROM` | view conflicting decision / dismiss |
| 7 | Workload warning | `ProactiveAgent` | `action_items` sum per assignee | view tasks / dismiss (fires 80% / 95%) |
| 8 | Stakeholder suggestion | `ProactiveAgent` | org-graph traversal | add stakeholder / dismiss |
| 9 | Dependency warning | `ProactiveAgent` | `action_items` dependencies | view blocking item / dismiss |
| — | Save-for-later queue | `QuestionAgent` | `agent_suggestions.status` | persists into next meeting |

The sidebar **renders**, it never decides: confidence gating (>0.7 surface) happens in
`agent-runtime-service`, so anything reaching the sidebar is already above threshold. The
no-silent-commit rule is load-bearing: the card surfaces, the user clicks Confirm, and only then
does `core-api` emit and `knowledge-graph-service` create the edge. **Edit must open a Modal
pre-filled with the extracted `content` / `rationale` / `assignee` / `due_date`.**

**Streaming token updates (`:289–314`).** The client subscribes over Socket.io to two channels:
`agent-stream` token deltas (disposable, progressive) and `agent-results` finals (the durable
card). Three client rules: deltas are throwaway — a missing delta is never data loss; drafts are
correlation-keyed and swapped for the final on `agent-results`; under a slow connection the
gateway drops `agent-stream` first, so the client simply gets the final card slightly later with
no progressive animation. Captions and confirmed outputs are never dropped.

**Priority color language, frozen (`:315–325`).** priority `1` → `--priority-high` (red, act now);
`2` → `--priority-medium` (yellow); `3` → `--priority-low` (blue). The red conflict banner uses
`--conflict` (the same red) and always renders above all other cards regardless of arrival order —
it is the one section that pre-empts. The per-user priority queue is enforced server-side; the
client renders the received order and additionally pins the conflict banner to the top.

**Animation discipline — a behavioral contract, not a suggestion (`:327–337`).** New cards fade +
slide in over ~150 ms; never bounce, flash or shake. Streaming text appends without layout thrash
(reserved min-height per card). At most **one** attention-grabbing motion at a time — the conflict
banner's single subtle pulse on first appearance, then still. Toasts auto-dismiss after a fixed
dwell and queue rather than stack. `prefers-reduced-motion` degrades all entrances to instant,
opacity-only. Enforced through shared Framer Motion variants `cardEnter`, `bannerPulseOnce`,
`toastDwell` so no feature component invents its own motion.

**Save-for-later (`:339–347`).** Save sets `agent_suggestions.status = 'Save-for-later'`; because
that status is durable in Postgres the queue is not client-local — it is re-hydrated on the next
meeting-room mount into a dedicated "Saved" sub-section, so a question parked in Monday's standup
reappears in Wednesday's planning.

**Collapse / expand (`:349–353`).** Collapses to a thin rail showing only unread counts and the
conflict indicator; collapse state persists per user (localStorage + server preference); live
subscriptions stay open while collapsed, badge counts update, full cards render on expand.

**Real-time client (`:357–437`).** The browser holds exactly **two** long-lived connections and
knows exactly **two** origins: the `api-gateway` HTTPS origin and the `realtime-gateway` WSS
origin (plus LiveKit for media). State is layered: RSC cache / TanStack Query / Zustand / Clerk.
Performance is budgeted to the imported SLOs; the whole agent UX collapses into a mobile
bottom-sheet without a second implementation.

**Core screens inventory (`:439–457`) — 13 screens:**

| Screen | Route | Mode | Primary source | Surfaces |
|---|---|---|---|---|
| Dashboard | `/dashboard` | RSC | `bff` aggregate | upcoming meetings, open `action_items`, recent `worklogs` |
| Meeting list | `/meetings` | RSC | `core-api` `/meetings` | `meetings` status + schedule |
| Meeting room | `/meetings/[id]/room` | client | `meeting-service` token, `realtime-gateway`, LiveKit | video, captions, chat, **agent sidebar** |
| Transcript view | `/meetings/[id]/transcript` | RSC | `search-service` FTS over `transcript_chunks` | colored speaker labels, click-to-jump, in-page search |
| Decision history & lineage | `/decisions` | RSC | `core-api` + `knowledge-graph-service` templates | `decisions`, `FOLLOWS_FROM` lineage |
| My Tasks | `/tasks` | RSC | `core-api` `/actions` | `action_items` assigned to me |
| Org chart | `/org` | RSC | `core-api` `/teams`, `/departments` | departments/teams hierarchy |
| Search | `/search` | client | `rag-service` `POST /api/rag/query` + `search-service` | RAG passages + citations |
| Profile / settings | `/settings/profile` | RSC + Server Action | `identity-tenant-service` | role/skills/bio/avatar |
| Integrations | `/settings/integrations` | RSC + Server Action | `integration-service` | `integration_connections` (R8) |
| Billing | `/settings/billing` | RSC | `billing-metering-service` | plan, seats, usage, invoices |
| Tenant admin console | `/tenant` | RSC (RBAC) | `admin-ops-service` `/admin/tenant/*` | members, plan, integrations, export/erasure |
| Internal ops console | `/ops` | RSC (RBAC, flagged) | `admin-ops-service` `/admin/ops/*` | cross-tenant support, flags, lag/cost dashboards |

**Frozen citation contract (`:459–468`).** A transcript passage renders `meeting_id` + `chunk_id`
+ `speaker` and clicking jumps to that timestamp in the transcript view; a document passage
renders `document_id` + `element_id` and clicking opens the source element. The client treats
citations as opaque trusted provenance — it renders and links, it never re-ranks or re-fuses.

**Trust model (`:470–479`).** The browser is **not** a tenancy boundary. `<ClerkProvider>` wraps
the root layout and middleware protects `(app)` and `(admin)`. The client never injects or trusts
`org_id`; the gateway extracts it from the verified JWT server-side. RBAC gating on `(admin)` is
enforced server-side — the client hides admin affordances only as a UX courtesy. The client never
filters by tenant itself; it receives only its tenant's data.

## `plan/frontend-gaps-by-band.md` (analysis only, written 2026-08-14)

**The finding in one line (`:14–17`):** *"The screens exist and the plumbing exists; almost
nothing connects them."* Of the 14 pages in the `(app)` route group, **13 read from
`src/lib/mock/data.ts` and one calls the real API client** — and that one still imports mock data
for part of its content.

**Why this inverts the usual reading (`:19–22`):** the frontend is not behind because pages are
missing. Shells for bands that have not started yet already exist. It is behind because **the
wiring, the realtime transport and the confirm affordances are absent.**

**Method (`:24–36`):** page inventory from `git ls-tree -r origin/main` after `git fetch` —
**40 pages**: `(marketing)` 22, `(app)` 14, `(auth)` 2, `(admin)` 2. Mock-vs-real determined by
grepping imports of `lib/mock` against `lib/api`. Transport capability determined by grepping
`src/` for `livekit`, `WebSocket`, `EventSource`, `socket.io` and by reading `package.json`.
Dependencies on `origin/main` are exactly `@clerk/nextjs`, `framer-motion`, `lenis`, `next`,
`react`, `react-dom` — **no LiveKit client, no WebSocket client, no data-fetching library.**

**The band map (`:40–52`):**

| Band | Plan item | State | Gap |
|---|---|---|---|
| Plan 0 | 0E.1 Next.js scaffold + Clerk provider | Satisfied late | None material |
| Plan 1-2 | 3.10 web-app shell: Clerk auth, nav, dashboard | Built, one path verified | Both acceptance checks outstanding |
| Plan 3-4 | 3.4 LiveKit room UI, chat, presence | Shell only | Whole item; no LiveKit dependency |
| Plan 3-4 | 3.8 Live captions, transcript, speaker mapping | Shell only | Whole item; no realtime transport |
| Plan 5-6 | 3.5 Org-chart UI and person profile | Backend ready, UI on mock | Wiring only |
| Plan 7-9 | 3.4 Worklog My Notes panel, agent sidebar shell | Not started | Whole item |
| Plan 7-9 | 3.8 Decision/action confirm cards, My Tasks, lineage | Shells on mock | **The Confirm affordance** |
| Plan 10-13 | 3.5 Context-fill cards, collaboration log | Not started | Whole item |
| Plan 10-13 | 3.9 Proactive alert UI, sidebar polish | Not started | Whole item |
| Plan 10-13 | 3.14 Integration settings, both consoles | Shells on mock | Wiring |
| Plan 14 | No web-app item | N/A | None |

**Per-band detail worth carrying:**
- **Plan 1-2 (`:63–77`)** — the strongest part of the frontend. `src/middleware.ts` uses
  `clerkMiddleware`; `src/lib/api/client.ts` is server-only **by construction** (it imports
  `auth()` from `@clerk/nextjs/server`, so importing it from a client component is a build error
  rather than a token leak), it reads `API_GATEWAY_URL` and refuses to default it, and it never
  sends `org_id`. `GET /bff/dashboard` has been exercised through the real chain of Cloudflare →
  Caddy → `api-gateway` → `bff`. Two gaps: the section's own acceptance checks (two tenants seeing
  disjoint data; a network capture showing no `org_id` on the wire) are unmet, and
  `(app)/dashboard/page.tsx` imports both the real client and `mockActions`.
- **Plan 3-4 (`:79–91`)** — both items are shell-only and the blocker is **below the component
  layer**: zero grep matches for `livekit`, `WebSocket`, `EventSource`, `socket.io`. §3.8 is the
  more important of the two: doc 10 makes live in-call captions the flagship real-time surface,
  the imported UX analysis names it the largest single miss, and it is the surface every agent
  output attaches to — so it gates the Plan 7-9 sidebar work as well.
- **Plan 5-6 (`:93–100`)** — the cheapest remaining win and the only band item whose backend is
  finished and measured. `GET /bff/org/chart` resolves a department → team → member tree in
  **exactly three SQL statements regardless of tree shape, at 7.85 ms** against real seeded data,
  and `GET /bff/people/{user_id}` exists beside it. `(app)/org/page.tsx` exists on mock data.
  Nothing needs designing or deciding.
- **Plan 7-9 (`:102–113`)** — the shells run ahead of the plan (`decisions/page.tsx`,
  `tasks/page.tsx` exist on mock; the agent sidebar does not exist in `mytheus-ui` at all, though
  a designed one exists in the `mytheus-web` draft and is the intended harvest source). §3.8
  **carries the most product-critical frontend gap in the program**: the backend enforces no
  silent commits — only `core-api`'s outbox emits `decisions` and `actions`, and only on
  confirmation — so a UI that renders detected actions as passive text with no Confirm affordance
  cannot satisfy that contract. **The invariant is currently enforced on one side only.**
- **Plan 10-13 (`:115–121`)** — not started, as expected. The context-fill card and the proactive
  conflict banner both exist as finished designs in the `mytheus-web` draft, so these are
  harvest-and-wire rather than design-from-nothing. §3.14's surfaces already have shells:
  `(app)/settings/integrations`, `(admin)/admin`, `(admin)/ops`.

**Cross-cutting blockers, to be treated as prerequisites rather than parts of any one item
(`:123–142`):**
1. **No realtime transport.** No LiveKit client, no WebSocket client. Blocks both Plan 3-4 items
   completely.
2. **No test tooling.** CI runs typecheck, lint and an image build; there are no test files and no
   Vitest or Playwright. (Illustrative regression: the marketing routes once returned 404 to
   signed-out visitors because `src/middleware.ts` allowlisted `/legal(.*)`, a path this app does
   not have, while the real pages are `/privacy` and `/terms`. Fixed; nothing prevents its return.)
3. **Clerk JWT template unmapped.** `public_metadata.internal_org_id` must be mapped into the
   token claim. Not UI code, but it blocks the AI tier the sidebar work depends on.
4. **DNS.** Apex and `www` still point at a parked page; the origin already serves the app.
5. **A moving target.** `origin/local-mytheus-rebuild` is the working branch; any gap list against
   `main` ages quickly and one landing commit remains unmerged.

**Suggested order, by evidence gained per unit of work rather than band number (`:144–159`):**
1. Plan 5-6 §3.5 — pure wiring against a measured endpoint; proves the client pattern on a second
   screen and removes one mock import.
2. Plan 1-2 §3.10 acceptance checks — two tenants, disjoint data, and a capture showing no
   `org_id`; the isolation claim demonstrated at the client, currently proven only in DB tests.
3. Plan 3-4 §3.8 — add a realtime transport and land captions. Highest product value; unblocks the
   sidebar surfaces later.
4. Plan 3-4 §3.4 — LiveKit room, once the transport decision is made.
5. Plan 7-9 §3.8 — the Confirm affordance, closing the no-silent-commits contract on the side that
   currently cannot honour it.

Items 1 and 2 need no new dependency and no new decision.

## `plan/frontend-meeting-ux-gap-analysis.md`

Imported from `mytheus-web` at commit `82b7d96`, preserved because that repo is a two-commit
prototype no reading path leads to and this is the only systematic comparison anyone has made
between what the docs specify for the live meeting experience and what a UI actually surfaces
(`:3–6`). Explicitly **an execution INPUT, not canon and not a record of work done**: it
introduces no name, number or decision, and where the draft diverges from the docs, **the docs
win until an ADR says otherwise** (`:8–11`).

**Headline (`:23–32`):** the draft nails the agent sidebar (Needs-you cards, feed, conflict →
peek) and the post-call artifacts (transcript, decisions, tasks, worklog, recordings, ask). The
real holes are **in the live meeting itself** — live captions in-call (the single biggest miss),
no confirm surface for action items, proactive alerts beyond conflict, an in-meeting RAG panel,
an inert Edit modal, no meetings list or meeting-detail hub, and non-progressive agent output.

**Tier 1 — in-meeting surfaces ("this IS the meeting software"), with docs sources (`:38–48`):**

| # | Gap | Status | Docs source |
|---|---|---|---|
| 1 | **Live captions** in-call (interim→final, click-to-jump, speaker-colored) | MISSING | `10-realtime §3.2/§4.1`; `20-frontend §7` |
| 2 | **ActionAgent card** — Confirm/Edit/Dismiss for detected actions (passive feed text only today) | MISSING | `08-ai-agent §2.1`; `20-frontend §5.1` |
| 3 | **Workload warning** (assignee at 80%/95% capacity) | MISSING | `08-ai-agent §2.1`; `20-frontend §5.1` |
| 4 | **Missing-stakeholder** suggestion + **dependency warning** | MISSING | `08-ai-agent §2.1` |
| 5 | **In-meeting RAG context panel** — source cards with the frozen citation contract | MISSING | `20-frontend §7.1` |
| 6 | **Edit modal** for decisions/actions, pre-filled | MISSING — the button is inert | `20-frontend §5.1`; `08-ai-agent §2.2` |
| 7 | **Progressive/streamed agent output** (tokens stream, then the draft swaps for the confirmed card) | PARTIAL — pops in whole | `10-realtime §5`; `20-frontend §5.2` |
| 8 | **Priority strip / pre-emption** — frozen red(1)/yellow(2)/blue(3); urgent alerts pin above the feed | PARTIAL — kind-colored, no priority queue | `20-frontend §5.3` |
| 9 | **Recording/transcription consent gate** at join | MISSING — **proposed, not canon** | `12-compliance §3.3`; `plan-3-4 §8` |

**Tier 2 — lifecycle / structural surfaces (`:52–57`):**

| # | Gap | Status | Docs source |
|---|---|---|---|
| 10 | `/meetings` list + `/meetings/[id]` detail hub (pre/post) | MISSING — only home "Up next" | `20-frontend §2.2/§7` |
| 11 | Per-participant post-meeting summary artifact | PARTIAL — transcript "Overview" only | `diagrams/06 §3.4`; `21-integration §4.3` |
| 12 | Server-anchored meeting clock | PARTIAL — hardcoded count-up | `plan-3-4 §7` |
| 13 | Live presence — real join/leave roster + in-call count | PARTIAL — static | `10-realtime §4.1`; `plan-3-4 §3.2` |
| 14 | Collapse-to-rail + mobile bottom-sheet, conflict auto-expands | PARTIAL — full hide/show only | `20-frontend §5.4/§5.6` |

**Tier 3 — config/settings the docs imply but the draft lacks (`:61–65`):**

| # | Gap | Status | Docs source |
|---|---|---|---|
| 15 | PII redaction toggle | MISSING | `plan-3-4 §3.5`; `12-compliance §5` |
| 16 | Data-residency region selector (us/eu/ap) | MISSING | `plan-3-4 §3.5-3.6`; `12-compliance §6` |
| 17 | Per-tenant agent confidence-threshold overrides | MISSING | `08-ai-agent §6.1` |
| 18 | Per-tenant token-budget alerts (80%/95%/cap) surfaced in-app | MISSING | `08-ai-agent §4.1/§4.4` |

**Area-by-area verdicts (`:71–135`).** A. Pre-join/device setup is **strong** — device check,
in-app role-filtered prep brief, countdown and lobby status BUILT; consent gate MISSING;
participant-verified short-TTL token mint is backend. B. In-call media mostly built as mock —
controls, speaker stage, thumbnails, pin, mic state, speaking wave BUILT; video grid PARTIAL
(docs scope 1–9 tiles / 6 nominal, draft shows 1 + 3 thumbs + "12+" with an inert layout switch).
C. Roster BUILT, live presence PARTIAL. D. Live transcription is **the big hole** — captions
MISSING, but the transcript replay page and the **speaker-mapping UI** are BUILT. E. Of the six
agents: Decision BUILT (Edit inert), Question BUILT, ContextGap BUILT, PersonalListener PARTIAL
(notes blended into the feed, not a distinct live My Notes section), **Action MISSING**, Proactive
PARTIAL (conflict only). F. Chat BUILT; Save-for-later surface present but cross-meeting
persistence is mock; in-meeting RAG panel MISSING. G. Post-call artifacts **strong** — decisions
with confidence and lineage, tasks, worklog, recordings library, Ask with citations, transcript
replay all BUILT. H. Lifecycle — prep brief and integrations surfaced; `/meetings` list + detail
MISSING; server-anchored clock PARTIAL. I. Toasts and "Listening" status BUILT; conflict is a card
not a top-pinned pre-emptive banner; no frozen priority strip. J. Role concept surfaced;
per-tenant overrides/theming/locale MISSING.

**DIVERGENCES — the draft has these, the docs do NOT spec them; a decision is needed on each
(`:139–150`):** host-admit lobby ("X wants to join" → Admit/Deny) versus docs specifying only a
pre-join device check and **explicitly no waiting-room/host-admit flow**; raise-hand; emoji
reactions on chat; a manual Record toggle as a per-user call control (docs record via LiveKit
egress tied to lifecycle/consent, not a free toggle); large-meeting affordances ("16 in call",
"12+", change-layout) against a documented 1–9 tile scope.

**UX-shaping SLOs restated (`:155`):** token-mint p95 < 250 ms, captions p95 < 2 s, agent output
p50 < 2 s.

## Supporting specification the frontend must honour

**The six agents and their surfacing (`08-ai-agent-architecture.md:33–40`).**
`PersonalListenerAgent` (P7, one per participant per meeting) → My Notes; `DecisionAgent` (P8) →
decision-detected card; `ActionAgent` (P8) → new-task toast; `QuestionAgent` (P9) → suggestion
card priority 1–3; `ContextGapAgent` (P10) → context-fill card; `ProactiveAgent` (P11) → conflict
banner, workload warning, stakeholder/dependency card.

**Confidence thresholds, PDF-verbatim (`08-ai-agent-architecture.md:296–306`):** `> 0.7` surface
to the user; `0.4 – 0.7` silent log to `worklogs`, no surfacing; `< 0.4` drop. Overridable per
tenant via `tenant_settings.agent_confidence_overrides`.

**Public REST surface the UI consumes (`19-api-architecture.md:133–232`)** — all `/api/v1`:
`GET /users/me`, `GET|PUT /users/:id/profile`, `GET /orgs/:id`, `GET /orgs/:id/teams`,
`POST /teams`, `GET|PATCH /teams/:id`, `GET|POST /departments`, `GET /context/:user_id`,
`GET|POST /orgs`, `/rbac/*`, `/scim/v2/*`; `POST|GET /meetings`, `GET|PATCH /meetings/:id`,
`POST /meetings/:id/token` (meeting-service, p95 < 250 ms), `GET /meetings/:id/transcript`,
`GET|POST /meetings/:id/chat`; `GET|POST /worklogs`, `PATCH /worklogs/:id`, `GET /decisions`,
`POST /decisions/:id/confirm`, `GET /actions`, `POST /actions/:id/confirm`, `GET /suggestions`,
`PATCH /suggestions/:id`, `GET /collaborations`; `POST /rag/query`, `POST /ingest/document`,
`GET /ingest/jobs/:id`, `GET /search`; `GET /billing/subscription`, `POST /billing/subscription`,
`POST /billing/checkout-session`, `GET /billing/invoices`, `GET /plans`, `GET /usage`;
`GET|POST /admin/tenant/*`, `GET|POST /admin/ops/*`, `POST /integrations/oauth/:provider`;
`GET /notifications`, `PATCH /notifications/:id`.
**Deliberately not public (`:227`):** `POST /embed`, `POST /rerank`, `/agents/spawn`,
`/agents/:id/terminate`, `/graph/query/{template}`, `/graph/cypher`, Temporal queues and
`/workflows/*`.

---

# WHAT THE DOCS CLAIM IS DONE — **UNVERIFIED CLAIMS**

Everything in this section is an assertion from `work-done-till-now/00-status-overview.md` dated
2026-08-29. It is recorded here only so it can be diffed against code reality. **None of it is
evidence.**

**Claimed band status (`:25–32`):**

| Band | Claimed status | Claimed detail |
|---|---|---|
| Plan 0 | **DONE WITH GAPS** | backend, events, intelligence, realtime, platform complete; UI and mobile incomplete, both deliberately |
| Plan 1-2 | **DONE WITH GAPS — every section built; no end-to-end run yet** | P1 landed 2026-07-27 (canon amendments + a seventeen-file design pack); five P2 sessions on 07-29/07-30 closed all seven blocking defects and built §3.2, 3.2b, 3.3–3.9, 3.11 in backend, §7 in mobile, §3.10 in UI; three least-privilege DB roles, tenant-root carve-out, `rbac_grants`, `audit_logs`, deny-by-default authz gate, meetings CRUD, Clerk webhook, `bff` dashboard, CI in two repos |
| Plan 3-4 | **DONE WITH GAPS** | `meeting-service`, `core-api` chat/speakers/consent-gate, `realtime-gateway`, `transcription-service` built and tested against real infra; since 2026-08-27 the `meeting-room-events` I/O loop is wired and UI §3.4 (LiveKit client + video grid) and §3.8 step 1 (live captions) are built on explicit go. **Still unbuilt: §3.8 steps 2–3 (transcript view, speaker-mapping UI), §3.4 steps 3–4 (Socket.io presence and chat), and mobile** |
| Plan 5-6 | **IN PROGRESS** | nine of ten tasks done against real live infra (Postgres, Redis, Redpanda, Schema Registry, Qdrant, MinIO); **only GPU packaging is faked/BLOCKED**; `search-service` (task 10) and the whole-band final review unfinished |
| Plan 7-9 | **NOT STARTED** | — |
| Plan 10-13 | **NOT STARTED** | — |
| Plan 14 | **NOT STARTED** (post-GA, demand-driven) | — |

**Claimed repo state (`:44–56`):** `mytheus-docs` — canon, "22 ADRs" (there are **23** on disk),
diagrams, eight band plans. `mytheus-backend` — ten bounded contexts, full six-job `ci.yml` gating
a VPS deploy, *the only repo with real CI*. `mytheus-events` — 21 topics registered, BACKWARD gate
green against the live registry. `mytheus-intelligence` — seven services;
embedding/reranker/ingestion/search built; `ner-service` named in canon and unbuilt.
`mytheus-realtime` — `realtime-gateway` 42/42, `transcription-service` incl. the
`meeting-room-events` loop and a dev-only `devseed`; **no CI workflow at all**.
`mytheus-platform` — compose/deploy substrate only; the Terraform/Helm/k8s trees its own
CLAUDE.md describes **do not exist: zero `.tf` files, zero charts**. `mytheus-ui` — canonical
`web-app`, mock-data site plus the real LiveKit client, video grid and live-caption stream since
2026-08-27; **no test infrastructure and no CI**. `mytheus-mobile` — **entirely on mock data**, no
API client, no Clerk, no tests. `mytheus-web` — reference only, not a program repo.

**Claims the status doc itself retracts — useful diff seeds:** gap 32 states that
`work-done-till-now/02` §14.1 records a ten-artifact `mytheus-mobile` API layer (`src/api/client.ts`,
`types.ts`, `present.ts`, `provider.tsx`, `hooks/`, `queryKeys`…) that "does not exist and never
did". Gap 33 states the repo inventory's claim that the org move was "verified against each
clone's `origin` remote" was true of only 2 of 9 clones. Both are the status folder over-reporting.

**Open items the doc lists as still true and blocking (from the Live gap list, `:90–143`):**
no end-to-end run exists (1); `Idempotency-Key` reuse with a different body replays instead of
returning 409, and the key has no route/method/path component (3); the live VPS deploy has never
run — no longer blocked by D1, awaiting explicit go (9); **the compliance gate's substantive
conditions are unmet** — no Deepgram DPA/SCC signed, LiveKit Cloud not in the subprocessor
register, zero-retention flag unset, though the engineering control
(`COMPLIANCE_GATE_GREEN = False`, unbypassable) is correct and blocking (12); NER redaction and
the `pii_tokens` vault ratified but unimplemented (13); **no key custody exists anywhere in the
program** and two tables need it (22); `Topic` node seeding into `nexus_knowledge` has no
legitimate writer and `knowledge-graph-service` is a bare four-line scaffold with zero routes (16);
GPU packaging unbuilt so every GPU SLO is unmeasured — both models measured on CPU only (18);
mobile person-profile screen unbuilt and unscoped (21); **two of nine repos have no CI at all**
(23); `mytheus-platform`'s own CI is red on two pre-existing faults (25); **the script that
performs every production deploy is unversioned** and exists only on the VPS, in no repository
(26); the documented token-mint path is wrong in two reader docs (27); a partition-grant line is
missing from a realtime migration (28); two substrate residuals need a superuser (29); the
migration chain cannot replay into a second database on a cluster where the roles exist (30);
**a foreign key does not confine a reference to its tenant, and code in this program assumed it
did** — PostgreSQL runs RI checks as the referenced table's owner with RLS bypassed (31);
**the production event backbone has never been provisioned** — nothing ever created the 21 topics
or registered the 21 Avro subjects, and the deploy could not tell (34); a vendored script has
drifted between two repos (35); all three `mytheus-intelligence` consumers would have processed
exactly one message and stopped, due to `asyncio.run` per message (36); **there is no
observability stack anywhere in the program**, so the lag and DLQ panels several plans name
cannot exist (37); a schema change reached the repo unregistered and no gate could see it (38);
**the `oasdiff` breaking-change gate exits 0 and could never have failed a build** (39).

---

# PHASE BOUNDARY — MVP SCOPE vs DELIBERATELY DEFERRED

The boundary is **W36** and it is explicit (`22-roadmap-and-phasing.md:222–246`). Stage 1 is the
frozen 13-phase PDF backbone producing a real multi-tenant, compliant, billable MVP validated at
seeder scale. Stage 2 takes that MVP to the capacity anchors. **Nothing in Stage 2 may be reported
as "missing" — it is deferred by design.**

| Dimension | **Stage 1 — MVP (W1–W36), IN SCOPE** | **Stage 2 — Scale-out (post-W36), DEFERRED** |
|---|---|---|
| Target load | Seeder: 2 orgs × 10 users × 5 meetings → first paying tenants | A1 = 5M users, A4 = 3,000 tenants, A5 = 20k peak meetings, A6 = 100k peak participants |
| Compute | **Modular monolith on a single-node VPS** (ADR-0016) | Kubernetes EKS/GKE + Terraform; Wave 1–4 extraction |
| Tenancy | Pooled + PostgreSQL RLS, single region | + cell-based isolation for large/enterprise tenants |
| Regions | **Single region** | Multi-region active-passive → active-active, per-plan tiers |
| Event bus | Redpanda Serverless (trial → small dedicated) | BYOC/dedicated multi-AZ Redpanda; ~17.9k ev/s peak |
| Inference | `embedding-service`/`reranker-service` on modest GPU | Autoscaling GPU/Ray Serve; ~525 embeds/s peak |
| Vectors | Per-tenant Qdrant collections, **no quantization needed** | int8 quantization; 4.42B vectors / ~17 TB live stock |
| LLM cost | Cost-controlled pipeline live (gating/batching/tiering/caching) | + the ~30% reduction (C4) realized at volume |
| Postgres | Single primary + backups; PITR + read replica land at P13 | + monthly range partitions, cold tier after 90d |
| Billing | Metering + per-tenant token budgets at P7; Stripe surface at P13 | Full Stripe seats/dunning/tax at volume |

**Explicitly IN MVP scope** and therefore fair to report as missing if absent: all six agents;
AGE `nexus_knowledge`; the cost-controlled pipeline and per-tenant budgets; the compliance gate;
`audit_logs` from P2 (not P13); the two `admin-ops-service` consoles at P12; PITR + read replica
and `audit_logs` archival at P13; Playwright E2E and k6/Locust at P13; the ~30% cost *measurement*
at P13; Cloudflare + custom domain; a live production deploy; the first paying tenant.

**Explicitly OUT of MVP scope — do not report these as gaps:**
- Kubernetes and Terraform. Plan 0 says it in so many words: *"This band ships ONLY the MVP CI +
  deploy target — NO Kubernetes, NO Terraform (those are post-W36 Plan 14)"*
  (`plan/plan-0-scaffold-and-contracts.md:696`). The absence of `.tf` files and Helm charts in
  `mytheus-platform` is the plan being followed, not a defect.
- The Wave 1–4 service extraction — the monolith is *supposed* to still be a monolith at W36.
- Cell-based isolation; multi-region of any kind; the `nexus-deploy` GitOps repo.
- BYOC / dedicated multi-AZ Redpanda; int8 Qdrant quantization; full GPU autoscaling.
- Citus shard-by-`org_id`; the AGE → Neptune / Neo4j Aura migration.
- Realizing C4 at volume (the P13 measurement is in scope; the at-scale realization is not).

**Deferred by explicit decision rather than by phase:**
- **SSO / SAML / SCIM** — deferred at the tenant-claim decision (`work-done-till-now/00-status-overview.md:17`),
  even though ADR-0010 and `19-api-architecture.md` §5.1 still specify the surface.
- **The `pii_tokens` migration and any application code** — ADR-0020 is "Accepted for the data
  model and the call graph, **Blocked for implementation** on the key-custody precondition", and
  states plainly that ratifying it does not authorize writing the migration.
- **The 428 consent check in `meeting-service`** — ADR-0018 lists the Alembic migration and the
  application code as *"Out of scope for this ADR and this task"*; the flag defaults to `false`
  so no tenant's join behavior changes until a legal decision flips it.
- **`ner-service`** — named in canon by ADR-0020 but assigned to no plan band.
- **`mytheus-mobile`** — realign only in Plans 0 and 1-2; Plan 5-6 §7 was never assigned a task
  (status gap 21 calls for an explicit scoping decision before any future task picks it up).
- **`mytheus-web`** — a reference-only draft. Not a build target, not deployed, not in CI, not in
  canon. Its only value to the program is the harvested gap analysis.

**Pre-authorized descope paths — pulling one is planned behavior, not a failure
(`22-roadmap-and-phasing.md:195–199`):**

| Risk | Phase | Trigger | Fallback shipped instead |
|---|---|---|---|
| **R1** | P8 (W21–23) | Apache AGE proves operationally awkward on managed Postgres (extension instability, ops friction, query latency) | Migrate `nexus_knowledge` to **Neo4j Aura** (or self-host Postgres+AGE); relational materialized-view fallback documented as the at-scale exit. `knowledge-graph-service` already abstracts this. |
| **R6** | P10 (W27–29) | Multi-agent collaboration is unstable or misses the p95 < 1.5 s A2A round-trip budget | Descope `ContextGapAgent` to **RAG-only context fill**; ship A2A as **v2**. The runtime fallback chain already stops at the RAG rung — the descope is "stop at that rung". |
| **R8** | P12 (W32–33) | Integration scope creep threatens the W36 date | Hard guardrail: **Calendar + Slack + exactly one of Jira/Linear**. Cost of pulling: none — it is a cap that prevents slip. The absence of a second issue tracker is the guardrail working. |

**Two more scope facts that shape what counts as a gap:**
- The **compliance gate** is a hard precondition on P4, not a P13 activity: Deepgram DPA + SCCs,
  the PII redaction pass, and the residency pin must all be green **before W9**
  (`22-roadmap-and-phasing.md:130–143`). An unmet gate is a *blocked band*, not a missing feature.
- The **security program is shift-left**: the threat model is authored at P1, an authorization
  check is reviewed in every PR from P2 onward, and dependency + secret scanning are wired into CI
  on day one. The P13 "security audit" survives only as a **final attestation** over a
  continuously secured system, never as the first time anyone looks at authz
  (`22-roadmap-and-phasing.md:124–128`).
