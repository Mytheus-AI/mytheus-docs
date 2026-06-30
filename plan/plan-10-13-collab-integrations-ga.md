# Plan 10-13 — Multi-Agent, Proactive Intelligence, Integrations & Production GA  (MVP Completion & Go-Live)

> Complete the differentiated product and ship it: cross-meeting agent collaboration (`ContextGapAgent` over `agent-requests`/`agent-responses` with deny-by-default A2A authz), proactive intelligence (`ProactiveAgent` over confirmed `decisions`/`actions` + the AGE `nexus_knowledge` graph), external integrations (Google Calendar + Slack + ONE of Jira/Linear, with the live Jira connection as the end goal), the `ActionFollowUpWorkflow`, both `admin-ops-service` consoles, and the full P13 hardening + live production deploy on Railway/Render.

---

## 0. Band at a glance

- **Phases / weeks:** **P10 Workflows & Multi-Agent (W27-29)** → **P11 Proactive Intelligence & Real-time Polish (W30-31)** → **P12 External Integrations (W32-33)** → **P13 Testing / DevOps / Hardening / Production (W34-36)**. This band is the last 10 weeks of the frozen 36-week PDF backbone ([22-roadmap-and-phasing.md §2](../22-roadmap-and-phasing.md)) and ends at the W36 MVP go-live boundary; everything after (cells, multi-region, K8s, Citus, BYOC Redpanda) is **Plan 14 scale-out**, explicitly NOT here.
- **Goal:** turn the working single-meeting product (Plans 0-9) into the full NEXUS differentiator and put it in front of paying tenants. By end of band: a `ContextGapAgent` fills context gaps by asking *other participants'* agents (peer → related-meeting → RAG → silent) inside a p95 < 1.5s budget, all deny-by-default-authorized and audited; a `ProactiveAgent` surfaces conflict/workload/missing-stakeholder/dependency alerts; the `realtime-gateway` Redis pub/sub backbone fans streamed `agent-stream` Avro deltas to a per-user priority queue; a confirmed `action_items` row becomes a **live Jira ticket** and stays in two-way sync; the `ActionFollowUpWorkflow` chases overdue actions (day 1/3/7 + day-10 escalation); both `admin-ops-service` consoles are live; and the whole system is load-tested, security-attested, cost-optimized ~30% (C4), backed up (PITR + read replica), fronted by Cloudflare on a custom domain, and **deployed to production on Railway/Render**.
- **Repos & folders touched:**
  - `mytheus-intelligence` — **`agent-runtime-service`** (`ContextGapAgent`, `ProactiveAgent`, A2A PEP, fallback chain), **`rag-service`** (RAG-only context-fill arm, conflict-detection complement query). Currently docs-only with excellent specs.
  - `mytheus-backend` — **`workflow-service`** (`MeetingLifecycleWorkflow` real activities, `ActionFollowUpWorkflow`, dunning saga), **`identity-tenant-service`** (A2A PDP + `agent_authz_scopes`), **`core-api`** (`agent_collaborations` persistence + collaboration-log read API, conflict surfacing), **`integration-service`** (OAuth engine, `integration_connections`, `oauth_tokens`, Calendar/Slack/Jira connectors, webhooks), **`notification-service`** (in-app/email/Slack delivery, priority queue), **`billing-metering-service`** (full Stripe surface: subscriptions/invoices/plans/seats/dunning/tax), **`admin-ops-service`** (`feature_flags`, `ops_audit_log`, `support_grants`, both consoles).
  - `mytheus-realtime` — **`realtime-gateway`** (Redis pub/sub backbone, `agent-stream` delta fan-out, per-user priority queue surfacing).
  - `mytheus-events` — Avro subjects for `agent-requests`, `agent-responses`, `agent-stream`, `integration-events`, `notifications`, `billing-events`, `admin-events`, `graph-events` (+ their `.dlq`), registered BACKWARD.
  - `mytheus-ui` (`web-app`) — collaboration / context-fill cards, `agent_collaborations` log UI, proactive alert UI + sidebar polish, integration settings page, admin-tenant + internal-ops consoles, billing UI.
  - `mytheus-mobile` — proactive alerts + priority-queue surfacing, integration-driven follow-ups, Expo push (the absent push layer), mobile-responsive parity, inclusion in the Playwright/E2E + GA release pipeline.
  - `mytheus-platform` — Cloudflare + custom domain, Postgres PITR/read-replica config, `audit_logs` partition/archival policy, Railway/Render production env + GitHub Actions promote gate.
- **Demoable at the end of each phase** (PDF demo cadence — a demo at the end of EVERY phase):
  - **End of P10:** in a live meeting, participant A's `ContextGapAgent` detects a gap ("the Q3 pricing decision"), asks participant B's agent over `agent-requests`; B's agent answers over `agent-responses` inside the deadline; A sees a "Context added from B's agent" card; a cross-tenant probe is DLQ'd and audited; the `MeetingLifecycleWorkflow` runs real activities (open → transcribe-watch → post-process → close) end-to-end; the `agent_collaborations` log UI shows every round-trip with success/latency.
  - **End of P11:** the same meeting surfaces a **red conflict banner** ("this decision contradicts the W29 architecture decision") computed over the AGE `FOLLOWS_FROM` lineage + `decisions__{org_id}` Qdrant complement; a workload alert ("Priya already owns 6 open actions"); streamed `agent-stream` token deltas render progressively in the sidebar via the Redis pub/sub backbone, with urgent alerts pre-empting notes in `nexus:{org_id}:queue:agent:{user_id}`.
  - **End of P12 (the headline):** an admin connects **Jira** via OAuth on the integration settings page; a confirmed NEXUS `action_items` row creates a **real Jira issue**; moving that issue to "Done" in Jira flips `action_items.status` to `done` (loop-suppressed); the `ActionFollowUpWorkflow` fires a day-1 reminder Slack DM through `notification-service`; a Google Calendar event auto-creates a `meetings` row and ingests its attachment; both `admin-ops-service` consoles operate (members/plan/integrations; cost/lag dashboards, tenant suspend, audit-log access, feature flags).
  - **End of P13 (GA):** Playwright E2E (web + mobile) and k6/Locust load suites are green against the §5 SLOs; the security attestation is signed; LLM cost is measured ~30% down (C4 ≈ $6.99/MAU); Stripe subscriptions/invoices/dunning/tax are live; Postgres has PITR + a read replica; `audit_logs` partitions archive to `nexus-audit-archive`; the app is live behind Cloudflare on the custom domain, deployed on Railway/Render. **First paying tenant onboarded.**

---

## 1. Entry preconditions  (must be GREEN before starting)

Plan 10-13 is the *capstone* band. It assumes Plans 0-9 shipped a working multi-tenant meeting product with the agent runtime, RAG, the knowledge graph, and billing-metering live. Concretely, before W27:

1. **Identity is Clerk (RESOLVED DECISION 1), reverted from the Supabase-Auth drift.** Clerk JWKS verification lives in `mytheus-backend/libs/shared`; the Svix webhook user-sync handler is implemented in `identity-tenant-service`; `CLERK_*` env is restored; the `custom_access_token_hook` and the `auth.users → public.users` trigger are removed. `api-gateway` extracts `org_id` **once at the edge** from the Clerk JWT `org` claim and fails closed. **If this is not true, STOP — fix it in the identity band, not here.** The A2A PDP, the integration OAuth RBAC gate, both admin consoles, and every push registration depend on a trustworthy edge-injected `org_id` and the `owner`/`admin`/`member` RBAC grants.
2. **`agent-runtime-service` is live (P7) running the per-window pipeline.** The six-agent LangGraph runtime processes per **30-second window** (A17), not per raw chunk; `PersonalListenerAgent`, `DecisionAgent`, `ActionAgent`, `QuestionAgent` already emit confidence-scored candidates (>0.7 surface, 0.4-0.7 silent-log to `worklogs`, <0.4 drop). This band adds the 5th and 6th agents (`ContextGapAgent`, `ProactiveAgent`). If the runtime/window machinery is absent, P10 cannot start.
3. **Per-tenant token budgets + metering are live (P7).** `usage-events → usage_aggregates → agent_budgets` with `nexus:{org_id}:budget:{period}` hot mirror, 80/95% alerts, and `ok`/`throttled`/`blocked` enforcement (R4). The `throttled` state already degrades `ContextGapAgent` to RAG-only — this band wires that path. If budgets are not enforced, the A2A and proactive LLM load detonates the R4 cost bomb.
4. **Confirmed `decisions`/`actions` exist with the no-silent-commit rule + AGE graph (P8).** `core-api` is the SOLE producer of the `decisions`/`actions` topics (on user confirmation, via transactional outbox); `knowledge-graph-service` consumes them and `MERGE`-writes the `PRODUCED`/`ASSIGNED_TO`/`ATTENDED`/`RELATES_TO`/`FOLLOWS_FROM` edges into `nexus_knowledge` (edges on confirmation only). `ProactiveAgent` reads this graph; the `ActionFollowUpWorkflow` is started by the `actions` topic. If the graph is not populating from confirmed rows, P11 has nothing to reason over.
5. **RAG is live (P9).** `rag-service` answers `POST /api/rag/query` (org-scoped, three arms: Qdrant `transcript_chunks__{org_id}` + Postgres FTS + AGE `RELATES_TO`, RRF k=60, reranker warm floor ≥ 1/region). This is the terminal hop of the `ContextGapAgent` fallback chain and the conflict-detection complement. If RAG is down, the fallback chain has no floor.
6. **At-least-once + idempotency is live (Plan 0).** Transactional outbox on produce, inbox + `nexus:{org_id}:dedupe:{topic}:{idempotency_key}` (24h TTL) on consume, per-topic `<topic>.dlq`. Avro + Schema Registry, **BACKWARD** compat, subject `{topic}-value`, partition key `org_id`. The `EventEnvelopeHeader` (`event_id`, `idempotency_key`, `org_id`, `schema_version`, `producer`, `occurred_at`, `correlation_id`, `causation_id`) opens every value. The A2A protocol, integration sync, and follow-up workflows all ride this contract ([ADR-0007](../adr/0007-event-delivery-idempotency.md)).
7. **`realtime-gateway` is live (P3-P4) with Socket.io fan-out.** The WSS leg bypasses `bff` and rides `api-gateway` straight to the gateway; rooms are keyed by `meeting_id`. This band adds the **Redis pub/sub backbone** (`nexus:{org_id}:pubsub:meeting:{meeting_id}`), `agent-stream` delta fan-out, and the per-user priority queue. If basic fan-out is absent, P11 polish has nothing to polish.
8. **`notification-service` exists as an event-driven worker (stub or P-earlier).** It is the **sole consumer authorized to fan `notifications` to external channels**. This band lights up its three delivery paths (in-app via `realtime-gateway`, email, Slack) and the priority queue. The `ActionFollowUpWorkflow`, dunning, prep briefs, and summaries all deliver through it — they never send directly.
9. **Capacity anchors are the sizing target**, never the seeder: `A4` = 3,000 tenants, `A5` = 20,000 peak meetings, `A6` = 100,000 peak participants, `E7` = 12 A2A round-trips/meeting, `L4` = 12 `ContextGapAgent` synthesis calls/meeting, `R4` = 4.8M meetings/month, `C2` ≈ $9.73/MAU → `C4` ≈ $6.99 post-optimization. SLOs imported verbatim from [Capacity §11](../_canon/05-capacity-math.md): A2A round-trip p95 < 1.5s, agent e2e p50 < 2s / p95 < 4s, RAG warm p95 < 500ms.

> WARNING — **The MVP go-live target is Railway/Render single-region, pooled + PostgreSQL RLS (RESOLVED DECISION 3).** This band ships ZERO Kubernetes, ZERO Terraform cells, ZERO multi-region, ZERO Citus, ZERO BYOC Redpanda. Every K8s/HPA/KEDA/cell/Terraform construct quoted from [13](../13-scalability-and-performance.md)/[16](../16-deployment-and-infrastructure.md) is the **post-W36 scale-out** target (Plan 14) and is cited here only to dimension the modular-monolith equivalent. Do not build the K8s substrate in P13.

---

## 2. Canonical names introduced in this band

This band introduces **no new** service, table, topic, agent, collection, Redis-key, AGE-label, or bucket names beyond canon. It materializes already-frozen names plus the two ADR-0013 grant-store / query-taxonomy names that the ADR itself authors. Names used verbatim:

- **Services:** `agent-runtime-service`, `rag-service`, `knowledge-graph-service`, `workflow-service`, `identity-tenant-service`, `core-api`, `integration-service`, `notification-service`, `billing-metering-service`, `admin-ops-service`, `realtime-gateway`, `bff`, `api-gateway`, `web-app`.
- **Agents (frozen class names):** `ContextGapAgent`, `ProactiveAgent` (the 5th and 6th of the six; the prior four — `PersonalListenerAgent`, `DecisionAgent`, `ActionAgent`, `QuestionAgent` — shipped in P7-P9).
- **Temporal workflows:** `MeetingLifecycleWorkflow`, `ActionFollowUpWorkflow` (canonical), plus the SaaS dunning saga (a `workflow-service` sibling, no new public name).
- **Tables (canon):** `agent_collaborations`, `decisions`, `action_items`, `worklogs`, `agent_suggestions` (owner `core-api`); `integration_connections`, `oauth_tokens` (owner `integration-service`); `subscriptions`, `invoices`, `plans`, `agent_budgets`, `usage_events`, `usage_aggregates` (owner `billing-metering-service`); `feature_flags`, `ops_audit_log`, `support_grants`, `audit_logs` (owner `admin-ops-service`); `tenant_settings`, `data_residency`, `departments`, `teams`, `team_members` (read here). **`agent_authz_scopes`** — the A2A grant store, authored by [ADR-0013](../adr/0013-agent-to-agent-authorization.md), owned by `identity-tenant-service`.
- **Topics (canon):** `agent-requests`, `agent-responses`, `agent-stream`, `agent-results`, `integration-events`, `notifications`, `notification-delivery-events`, `actions`, `decisions`, `meeting-events`, `billing-events`, `admin-events`, `graph-events`, `usage-events`, `tenant-events` (each paired with `<topic>.dlq`).
- **Frozen Avro schemas (canon, 7):** `AgentRequest`, `AgentResponse`, `Action`, `Notification` (load-bearing here). **Net-new (not frozen) schema names** from [07](../07-event-architecture.md): `AgentStreamDelta`, `IntegrationEvent`, `BillingEvent`, `AdminEvent`, `GraphEvent` — usable but flagged.
- **AGE graph `nexus_knowledge`:** nodes `Person`/`Meeting`/`Decision`/`Action`/`Topic`; edges `ATTENDED`/`PRODUCED`/`ASSIGNED_TO`/`RELATES_TO`/`FOLLOWS_FROM` (edges on confirmation only). `ProactiveAgent` reads `FOLLOWS_FROM` lineage; no new edge label is introduced.
- **Qdrant collections:** `transcript_chunks__{org_id}`, `document_chunks__{org_id}`, `decisions__{org_id}` (the conflict-detection complement). BGE-small-en 384-dim cosine int8.
- **Redis keys (all `nexus:{org_id}:…`):** `nexus:{org_id}:pubsub:meeting:{meeting_id}` (Socket.io fan-out), `nexus:{org_id}:queue:agent:{user_id}` (per-user priority queue), `nexus:{org_id}:a2a:grant:{grantee_role}:{query_class}:{target_scope}` (A2A decision cache, 60s TTL), `nexus:{org_id}:oauth:{state}` (PKCE verifier, 10-min TTL), `nexus:{org_id}:dedupe:{topic}:{idempotency_key}` (inbox, 24h), `nexus:{org_id}:budget:{period}` (budget hot mirror), `nexus:{org_id}:ratelimit:{user_id}` (per-user LLM limit), `nexus:{org_id}:dedupe:stripe:{event_id}` (Stripe webhook dedupe).
- **Object buckets:** `nexus-attachments` (calendar attachments, `{org_id}/{meeting_id}/`), `nexus-invoices` (immutable, 7-yr, object lock), `nexus-audit-archive` (object lock), `nexus-exports` (auto-expiring GDPR exports).
- **Sync surfaces (canon §6):** `/oauth/{provider}`, `/webhooks/{provider}` (integration-service); `/billing/*`, `/stripe/webhook`, `/usage` (billing-metering-service); `/admin/tenant/*`, `/admin/ops/*` (admin-ops-service); `/agents/*`, `/workflows/*` (internal only).
- **Providers (closed enum):** `google_calendar | slack | jira | linear` — R8 hard cap.
- **Capacity symbols (imported, never re-derived):** `A4`, `A5`, `A6`, `E7`, `L4`, `R4`, `C2`, `C3`, `C4`, `L7`, `L10`, `V7`.

---

## 3. Work breakdown (ordered, grouped by phase then component)

Ordering is a precondition chain. **P10 before P11** because `ProactiveAgent` reasons over confirmed `decisions`/`actions` + the graph, and the A2A/collaboration substrate (PEP/PDP, `agent_collaborations`) is the runtime template the proactive alerts reuse. **P11 before P12** because the realtime pub/sub backbone + priority queue + `notification-service` delivery fabric are what integration-driven alerts and follow-ups deliver through. **P12 before P13** because the full integration + billing surface is what the GA hardening attests, load-tests, and cost-optimizes. Within each phase the backend producer lands before its UI consumer.

---

### PHASE P10 — Workflows & Multi-Agent Collaboration (W27-29)

#### 3.1 `workflow-service` — `MeetingLifecycleWorkflow` real activities (replace the stubs)

**Repo / folder:** `mytheus-backend/services/workflow-service/` (today: a Temporal worker that registers 4 workflows with ZERO activities — `system.md §10`).

**Why first in P10:** the `MeetingLifecycleWorkflow` is the durable spine that schedules post-meeting processing (summaries, the prep-brief timer in P12) and coordinates room open/close. It is consumed by `meeting-events` and must run real activities before any of the agent-collaboration or integration timers hang off it.

**Steps:**

1. **Add Temporal + gRPC client deps** (the worker already exists; add the activity-side deps):
   ```bash
   cd mytheus-backend
   uv add temporalio grpcio
   ```
2. **Define the workflow + activities** under `services/workflow_service/workflows/meeting_lifecycle.py`. Activities drive CRUD over REST against `core-api` (never cross-write its tables, per the one-writer rule):
   ```python
   from datetime import timedelta
   from temporalio import workflow, activity
   from temporalio.common import RetryPolicy

   @activity.defn(name="open_meeting_room")
   async def open_meeting_room(meeting_id: str, org_id: str) -> str:
       # POST core-api / meeting-service to ensure the room is provisioned.
       # org_id travels in the request header; RLS scopes the write.
       return await core_api.ensure_room(meeting_id=meeting_id, org_id=org_id)

   @activity.defn(name="run_post_processing")
   async def run_post_processing(meeting_id: str, org_id: str) -> None:
       # Trigger summary build + final worklog flush via core-api read models.
       await core_api.finalize_meeting(meeting_id=meeting_id, org_id=org_id)

   @activity.defn(name="close_meeting_room")
   async def close_meeting_room(meeting_id: str, org_id: str) -> None:
       await core_api.close_room(meeting_id=meeting_id, org_id=org_id)

   @workflow.defn(name="MeetingLifecycleWorkflow")
   class MeetingLifecycleWorkflow:
       @workflow.run
       async def run(self, meeting_id: str, org_id: str) -> None:
           rp = RetryPolicy(maximum_attempts=5, initial_interval=timedelta(seconds=1))
           await workflow.execute_activity(
               open_meeting_room, args=[meeting_id, org_id],
               start_to_close_timeout=timedelta(seconds=30), retry_policy=rp)
           # Wait for the meeting-ended signal (delivered from the meeting-events consumer).
           await workflow.wait_condition(lambda: self._ended)
           await workflow.execute_activity(
               run_post_processing, args=[meeting_id, org_id],
               start_to_close_timeout=timedelta(minutes=10), retry_policy=rp)
           await workflow.execute_activity(
               close_meeting_room, args=[meeting_id, org_id],
               start_to_close_timeout=timedelta(seconds=30), retry_policy=rp)

       _ended: bool = False
       @workflow.signal
       def meeting_ended(self) -> None:
           self._ended = True
   ```
3. **Wire the `meeting-events` consumer to start/signal the workflow** (the consumer lives in `workflow-service`, started off the bus). Use a deterministic workflow id `meetinglifecycle-{meeting_id}` so a redelivered `meeting-events {created}` does not start a duplicate:
   ```python
   await client.start_workflow(
       MeetingLifecycleWorkflow.run, args=[meeting_id, org_id],
       id=f"meetinglifecycle-{meeting_id}",
       task_queue="nexus-meeting-lifecycle",
       id_reuse_policy=WorkflowIDReusePolicy.REJECT_DUPLICATE)
   # on meeting-events {ended}: client.get_workflow_handle(id).signal(meeting_ended)
   ```
4. **Register the workflow + activities on the worker** (`python -m services.workflow_service.worker`).

**Canonical names:** `MeetingLifecycleWorkflow`, `meeting-events`, `core-api`, `workflow-service`.

**Acceptance checks:**
- Starting a meeting fires `MeetingLifecycleWorkflow` exactly once (Temporal Web shows one execution `meetinglifecycle-{meeting_id}`); a redelivered `meeting-events {created}` does not start a second.
- On `meeting-events {ended}`, the signal advances the workflow; `run_post_processing` then `close_meeting_room` complete; activities retry on transient failure and the workflow survives a worker restart mid-flight (kill the worker, restart, it resumes).
- No activity cross-writes a `core-api` table; all mutations go over REST.

#### 3.2 `identity-tenant-service` — the A2A PDP and `agent_authz_scopes` grant store

**Repo / folder:** `mytheus-backend/services/identity-tenant-service/`.

**Why before the agent PEP:** the PEP in `agent-runtime-service` cannot authorize a single A2A request until the PDP and its deny-by-default grant store exist. Per [ADR-0013](../adr/0013-agent-to-agent-authorization.md), identity is the **PDP** (policy-decision-point); the agent runtime is the **PEP** (policy-enforcement-point).

**Steps:**

1. **Add the `agent_authz_scopes` table** (data-model materialization; the name is authored by ADR-0013). Edit `_canon/02-data-model.md` to record the ADR-0013 table FIRST if not already present, then:
   ```bash
   uv run alembic revision -m "feat(identity): add agent_authz_scopes (ADR-0013)"
   ```
   Migration body (verbatim from ADR-0013, RLS FORCE):
   ```sql
   CREATE TABLE agent_authz_scopes (
       id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
       org_id          UUID NOT NULL REFERENCES organizations(id),
       grantee_role    TEXT NOT NULL,   -- requester RBAC role (from get_user_context)
       query_class     TEXT NOT NULL,   -- 'meeting_context'|'decision_lineage'|'action_status'
       target_scope    TEXT NOT NULL,   -- 'self'|'same_team'|'department'|'org'
       effect          TEXT NOT NULL DEFAULT 'allow' CHECK (effect IN ('allow','deny')),
       created_by      UUID NOT NULL,
       created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
       deleted_at      TIMESTAMPTZ,
       UNIQUE (org_id, grantee_role, query_class, target_scope)
   );
   ALTER TABLE agent_authz_scopes ENABLE ROW LEVEL SECURITY;
   ALTER TABLE agent_authz_scopes FORCE ROW LEVEL SECURITY;
   CREATE POLICY tenant_isolation ON agent_authz_scopes
     USING (org_id = current_setting('app.current_org')::uuid)
     WITH CHECK (org_id = current_setting('app.current_org')::uuid);
   ```
2. **Implement the PDP as an internal gRPC/REST method** `Authorize(grantee_role, query_class, target_scope, org_id)` → `allow|deny`. Deny-by-default: return `allow` only when a non-deleted `effect='allow'` row matches AND no matching `effect='deny'` row exists (explicit deny wins); absence of any row is deny.
   ```python
   async def authorize(session, org_id, grantee_role, query_class, target_scope) -> str:
       rows = await session.execute(select(AgentAuthzScope).where(
           AgentAuthzScope.org_id == org_id,
           AgentAuthzScope.grantee_role == grantee_role,
           AgentAuthzScope.query_class == query_class,
           AgentAuthzScope.target_scope == target_scope,
           AgentAuthzScope.deleted_at.is_(None)))
       effects = {r.effect for r in rows.scalars()}
       return "deny" if ("deny" in effects or "allow" not in effects) else "allow"
   ```
3. **Ship a platform-default grant template** an admin can adopt/tighten (e.g. `same_team` `meeting_context` for common roles), surfaced in the tenant-admin console (§3.13). A fresh tenant has NO grants → A2A is disabled by default (the safe posture).
4. **Decision-cache invalidation:** on grant create/revoke, delete `nexus:{org_id}:a2a:grant:*` keys; the 60s TTL is the backstop (do NOT use `tenant-events` for this — that topic is lifecycle-scoped).
5. **Audit every grant/revoke** to `audit_logs` (`actor_type='user'`, action `a2a.grant.create|revoke`).

**Canonical names:** `agent_authz_scopes`, `identity-tenant-service`, `get_user_context`, `audit_logs`, `nexus:{org_id}:a2a:grant:*`.

**Acceptance checks:**
- A tenant with no rows: every `authorize()` returns `deny` (A2A dormant).
- An `effect='deny'` row beats a matching `effect='allow'` row.
- Grant revoke clears the cache; a probe within 60s of a missed invalidation still denies via TTL expiry.
- RLS test: tenant A cannot read tenant B's `agent_authz_scopes`.

#### 3.3 `agent-runtime-service` — `ContextGapAgent`, the A2A PEP, and the fallback chain

**Repo / folder:** `mytheus-intelligence/services/agent-runtime-service/` (docs-only today; excellent specs exist for the 6 agents).

**Why after the PDP + lifecycle:** the `ContextGapAgent` issues `agent-requests` and must be authorized by the PEP→PDP path; it runs inside the `MeetingLifecycleWorkflow`-bounded meeting.

**Steps:**

1. **Scaffold the agent-runtime service** (if not already from P7) and add LangGraph + Kafka + gRPC deps:
   ```bash
   cd mytheus-intelligence
   uv add langgraph confluent-kafka grpcio redis
   ```
2. **Implement `ContextGapAgent` as a LangGraph node** under `services/agent_runtime_service/agents/context_gap_agent.py`. It runs per 30-second window like the others; on detecting a context gap it walks the **fallback chain `peer-agents → related-meeting agents → RAG → silent`** (verbatim, [14 §5.2](../14-reliability-and-dr.md)) with a deadline = the A2A round-trip budget (p95 < 1.5s):
   ```python
   class ContextGapAgent:
       async def run_window(self, ctx: WindowCtx) -> CandidateOrNone:
           gap = self._detect_gap(ctx)                       # cheap, deterministic classify
           if not gap:
               return None
           if ctx.budget_state == "throttled":               # R6/R4: throttled → RAG-only
               return await self._rag_fill(ctx, gap)
           # Hop 1: peer agents in THIS meeting
           r = await self._ask_peers(ctx, gap, scope="peer",
                                     deadline_ms=ctx.deadline_ms)
           if r: return r
           # Hop 2: related-meeting agents
           r = await self._ask_peers(ctx, gap, scope="related_meeting",
                                     deadline_ms=ctx.deadline_ms)
           if r: return r
           # Hop 3: RAG (org-scoped, the floor)
           r = await self._rag_fill(ctx, gap)
           if r: return r
           return None                                       # Hop 4: silent — never fabricate
   ```
3. **Implement the PEP — pre-publish and pre-process checks** (ADR-0013). Identity is **derived from `requester_agent_id`'s bound principal, never read from the payload**. The envelope `org_id` is untrusted input cross-checked against the derived value:
   ```python
   async def pep_pre_publish(self, req: AgentRequest) -> None:
       derived_org, role = self._bound_identity(req.requester_agent_id)  # in-process binding
       if req.header.org_id != derived_org:
           await self._audit_deny(req, reason="org_spoof"); raise A2ADenied
       query_class = classify_query(req.query)               # deterministic, no LLM
       target_scope = resolve_scope(req.responder_target, derived_org)
       decision = await self._cached_authorize(derived_org, role, query_class, target_scope)
       if decision != "allow":
           await self._audit_deny(req, reason="authz_denied")
           await self._log_collab(req, success=False, result="authz_denied")
           raise A2ADenied

   async def pep_pre_process(self, req: AgentRequest, responder_org: str) -> None:
       derived_org, _ = self._bound_identity(req.requester_agent_id)
       if derived_org != responder_org:                      # Rule 1: same-tenant only
           await self._to_dlq("agent-requests", req, "FOREIGN_ORG"); raise A2ADenied
   ```
   The cached authorize reads/writes `nexus:{org_id}:a2a:grant:{grantee_role}:{query_class}:{target_scope}` (60s TTL); on miss it calls the PDP (§3.2).
4. **Publish `agent-requests` / consume `agent-responses` through the outbox/inbox bus** with `idempotency_key = areq:{correlation_id}`; deadline carried in the envelope. Foreign `org_id` → `agent-requests.dlq`, never served (defense-in-depth on top of tenant-partitioning).
5. **Emit the context-fill candidate via `agent-results`** (NOT a direct write). Like every agent, `ContextGapAgent` proposes; `core-api` persists the collaboration log (§3.4) and the user sees a "Context added from {X}'s agent" card. Confidence thresholds apply (>0.7 surface).
6. **Meter the synthesis cost** to `usage-events` (`agent_token`, ~`L4` = 12 Sonnet synthesis calls/meeting) so budgets enforce.

**Canonical names:** `ContextGapAgent`, `agent-requests`, `agent-responses`, `agent-results`, `AgentRequest`, `AgentResponse`, `correlation_id`, `agent-runtime-service`, `rag-service`.

**Acceptance checks:**
- A context gap with peers available: peer answers before deadline; card shows "Context added from B's agent"; `agent_collaborations` logs `success=true` + latency.
- Peers silent / deadline miss: falls through to related-meeting, then RAG; if RAG misses, stays **silent** (no card, never fabricated).
- A2A round-trip p95 < 1.5s under `E7` = 12 round-trips/meeting load (measured in LT-style harness).
- **Cross-tenant probe** (forged envelope `org_id`): DLQ'd to `agent-requests.dlq`, never served, audited as a tenant-boundary violation.
- `budget_state == 'throttled'`: skips peers entirely, RAG-only (the R6 descope path exercised as a runtime degradation).

#### 3.4 `core-api` — `agent_collaborations` persistence + collaboration-log read API

**Repo / folder:** `mytheus-backend/services/core-api/`.

**Why:** `agent-runtime-service` owns NO Postgres tables; `agent_collaborations` is owned by `core-api` and persisted via its transactional outbox semantics (ADR-0013 logical-logging note). The collaboration log UI reads from here.

**Steps:**

1. **Confirm the `agent_collaborations` table exists** (canon table; if landed in P8/P9 reuse it, else materialize). Columns per canon/data-model: `id, org_id, meeting_id, requester, responder, query, result, success, latency_ms, created_at`. RLS FORCE.
2. **Add a consumer/endpoint that records each round-trip.** The agent tier emits a collaboration signal (carried on `agent-results` or a narrow internal call); `core-api` writes the `agent_collaborations` row idempotently (dedupe on `idempotency_key = collab:{correlation_id}`).
3. **Expose the read API** `GET /api/v1/meetings/:id/collaborations` (cursor-paginated, RBAC: participants + admin) returning requester/responder/query/result/success/latency.
4. **Surface conflict candidates** path (used in P11): a read endpoint `GET /api/v1/decisions/:id/lineage` that `web-app` calls to render the conflict banner context.

**Canonical names:** `agent_collaborations`, `core-api`, `decisions`.

**Acceptance checks:**
- Every authorized A2A round-trip and every denial appears in `agent_collaborations` with correct `success`/`result`.
- Redelivery of the collaboration signal does not create a duplicate row (idempotent on `collab:{correlation_id}`).
- `GET /meetings/:id/collaborations` is RLS-scoped (tenant B cannot read tenant A's log) and returns within CRUD p95 < 300ms.

#### 3.5 `web-app` — context-fill cards + `agent_collaborations` log UI

**Repo / folder:** `mytheus-ui/` (`web-app`).

**Steps:**

1. **Render the context-fill card** in the agent sidebar: when a `ContextGapAgent` candidate settles on `agent-results`, show "Context added from {responder}'s agent" with the cited source (`meeting_id`/`chunk_id` or `decision_id`), Confirm/Dismiss affordances where canon allows.
2. **Add a collaboration-log panel** (meeting detail / admin) reading `GET /api/v1/meetings/:id/collaborations`: a table of requester → responder, query class, success, latency, with denied rows visibly flagged.
3. **Respect the calm-not-flashy discipline**: deltas stream into a draft card via `agent-stream` (P11 wires the transport), reserved min-height to prevent layout thrash.

**Canonical names:** `agent-results`, `agent_collaborations`, `ContextGapAgent`.

**Acceptance checks:**
- Context-fill cards render with a citation and never appear for a silent fall-through.
- The collaboration-log panel shows grants and denials; denials are visually distinct.

---

### PHASE P11 — Proactive Intelligence & Real-time Polish (W30-31)

#### 3.6 `agent-runtime-service` — `ProactiveAgent` (conflict / workload / missing-stakeholder / dependency)

**Repo / folder:** `mytheus-intelligence/services/agent-runtime-service/agents/proactive_agent.py`.

**Why after P10:** `ProactiveAgent` reasons over **confirmed** `decisions`/`actions` and the AGE `nexus_knowledge` graph (edges exist only on confirmation, P8) and reuses the per-window runtime + budget plumbing P10 hardened.

**Steps:**

1. **Implement the four alert classes** as graph + vector queries (Opus only for high-stakes synthesis; routine on Sonnet — the tiered routing already enforced by budgets):
   - **Conflict:** for a newly confirmed decision, query the AGE `FOLLOWS_FROM` lineage for contradicting prior decisions, complemented by a `decisions__{org_id}` Qdrant semantic search (the **R1 hot-degradation complement** — if `knowledge-graph-service` is slow/down, fall back to the Qdrant `decisions__{org_id}` path only, [14 §5.3](../14-reliability-and-dr.md)):
     ```cypher
     // knowledge-graph-service, $org_id scopes the org-partitioned graph
     MATCH (d:Decision {org_id:$org_id})-[:FOLLOWS_FROM*1..3]->(prior:Decision {org_id:$org_id})
     WHERE d.decision_id = $decision_id
     RETURN prior
     ```
   - **Workload:** count open `action_items` `ASSIGNED_TO` a `Person` over the graph + `core-api` read model; alert if the assignee is over a threshold ("Priya already owns 6 open actions").
   - **Missing-stakeholder:** compare meeting `ATTENDED` set against the topic's historically-relevant `Person` set; alert if a key stakeholder is absent.
   - **Dependency:** detect an action that `FOLLOWS_FROM` an unfinished prior action.
2. **Score and route through the confidence gate** (>0.7 surface; the red conflict banner is priority-1). Emit candidates via `agent-results`; **never** auto-commit. Meter tokens to `usage-events`.
3. **Wire the budget degradation:** in `throttled`, suspend `ProactiveAgent` entirely (per [18 §5.4](../18-saas-business-architecture.md) — `throttled` routes all windows to Haiku and suspends `ProactiveAgent`).
4. **Gate behind `plans.features.proactive_agent`** (Enterprise-on by default; the `bff`/authz middleware reads the flag).

**Canonical names:** `ProactiveAgent`, `nexus_knowledge`, `FOLLOWS_FROM`, `ASSIGNED_TO`, `ATTENDED`, `decisions__{org_id}`, `agent-results`, `knowledge-graph-service`.

**Acceptance checks:**
- A confirmed decision that contradicts a `FOLLOWS_FROM` ancestor raises a conflict alert; with `knowledge-graph-service` killed, the alert still fires via the `decisions__{org_id}` Qdrant complement (R1 hot degradation), at reduced recall.
- A workload alert fires when an assignee crosses the open-action threshold; a missing-stakeholder alert fires when a known stakeholder is absent.
- In `throttled` budget state, `ProactiveAgent` emits nothing (suspended); durable facts still flow.
- No alert ever writes `decisions`/`actions` directly.

#### 3.7 `realtime-gateway` — Redis pub/sub backbone, `agent-stream` delta fan-out, per-user priority queue

**Repo / folder:** `mytheus-realtime/services/realtime-gateway/`.

**Why:** at `A6` = 100k concurrent participants across horizontally-scaled pods, in-process fan-out cannot deliver an agent output to a connection on another pod. The Redis pub/sub backbone (`nexus:{org_id}:pubsub:meeting:{meeting_id}`) makes any pod able to deliver to any connection ([13 §6.6](../13-scalability-and-performance.md), R19).

**Steps:**

1. **Add the Redis pub/sub backbone.** On `agent-stream`/`agent-results` consume, publish to `nexus:{org_id}:pubsub:meeting:{meeting_id}`; every pod subscribes for the rooms it holds and fans to its local Socket.io sockets:
   ```js
   // realtime-gateway (Node + Socket.io)
   sub.subscribe(`nexus:${orgId}:pubsub:meeting:${meetingId}`);
   sub.on("message", (_ch, payload) => io.to(meetingId).emit(evt.channel, JSON.parse(payload)));
   // on consuming agent-stream from Redpanda:
   await pub.publish(`nexus:${orgId}:pubsub:meeting:${meetingId}`,
                     JSON.stringify({ channel: "agent-stream", delta }));
   ```
2. **Stream `agent-stream` Avro deltas** (`AgentStreamDelta`, 1-hour retention, throwaway) into correlation-keyed draft cards; the UI renders the final on `agent-results`. A missed delta is never data loss.
3. **Per-user priority queue** `nexus:{org_id}:queue:agent:{user_id}`: urgent alerts (conflict, workload) pre-empt notes. Implement as a Redis sorted set scored by priority (1=red conflict, 2=yellow, 3=blue note); the gateway drains highest-priority-first per user:
   ```js
   await redis.zadd(`nexus:${orgId}:queue:agent:${userId}`, priority, JSON.stringify(item));
   // drain: ZPOPMIN delivers priority-1 (red) before priority-3 (note)
   ```
4. **Backpressure shed order** (reuse the value-tier rule, [07 §10.2](../07-event-architecture.md)): drop `agent-stream` deltas FIRST under load; never drop captions or confirmed output.

**Canonical names:** `realtime-gateway`, `agent-stream`, `agent-results`, `AgentStreamDelta`, `nexus:{org_id}:pubsub:meeting:{meeting_id}`, `nexus:{org_id}:queue:agent:{user_id}`.

**Acceptance checks:**
- An agent output produced on pod 1 reaches a participant connected to pod 2 within the Socket.io fan-out hop < 50ms.
- `agent-stream` deltas render progressively; a dropped delta is invisible to the user (final card is complete).
- A red conflict alert pre-empts queued notes for that user (delivered first).
- Under induced backpressure, `agent-stream` is shed before captions/confirmed output.

#### 3.8 `notification-service` — in-app / email / Slack delivery + priority queue

**Repo / folder:** `mytheus-backend/services/notification-service/`.

**Why now:** P11 lights up the delivery fabric the proactive alerts (and P12 prep briefs, summaries, follow-ups, dunning) all use. `notification-service` is the **sole consumer authorized to fan `notifications` to external channels**.

**Steps:**

1. **Consume `notifications`** (and `agent-results` for priority alerts), dedupe on `idempotency_key = notif:{source_event_id}:{channel}:{user_id}`, enforce per-user channel preferences, deliver via three paths: in-app (publish to the `realtime-gateway` backbone), email (provider SDK), Slack (`chat.postMessage` using a short-lived scoped token fetched from `integration-service`, never persisting the raw token).
2. **Emit `notification-delivery-events`** (delivery receipts) for the ops dashboards.
3. **Priority queue:** mirror the `realtime-gateway` priority ordering so urgent in-app alerts pre-empt routine notifications.

**Canonical names:** `notification-service`, `notifications`, `notification-delivery-events`, `agent-results`, `integration-service`.

**Acceptance checks:**
- A `notifications` event delivers to in-app + email + Slack per the user's channel prefs; redelivery is a no-op (deduped).
- The Slack send uses a short-lived scoped token; the raw `oauth_tokens` value is never persisted by `notification-service`.
- `notification-delivery-events` receipts appear for each delivery.

#### 3.9 `web-app` — proactive alert UI + sidebar polish

**Repo / folder:** `mytheus-ui/` (`web-app`).

**Steps:**

1. **Red conflict banner** pinned to top, pre-empting notes (the frozen red/yellow/blue priority language). Workload/stakeholder/dependency alerts render as priority-2/3 cards.
2. **Sidebar polish:** streamed deltas → draft → settled card with reserved heights; `prefers-reduced-motion` degrades entrances to opacity-only.
3. **Alert actions:** acknowledge/dismiss write through to `core-api`; nothing auto-confirms.

**Acceptance checks:**
- The conflict banner pins above notes and survives streaming without layout thrash.
- Alerts render in priority order matching the per-user queue.

---

### PHASE P12 — External Integrations (W32-33) — *the headline: connect to Jira and go live*

#### 3.10 `integration-service` — OAuth engine, `integration_connections`, `oauth_tokens`, webhook receiver

**Repo / folder:** `mytheus-backend/services/integration-service/` (stub `main.py`, no routes today).

**Why first in P12:** Calendar/Slack/Jira connectors are all built on the same four primitives — connection registry, OAuth engine, encrypted token vault, webhook receiver ([21 §2](../21-integration-architecture.md)). Build the framework, then the three connectors.

**Steps:**

1. **Add the two owned tables** (canon names; materialize). Edit `_canon/02-data-model.md` first if columns differ:
   ```bash
   uv run alembic revision -m "feat(integration): add integration_connections + oauth_tokens"
   ```
   ```sql
   CREATE TABLE integration_connections (
     id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
     org_id             uuid NOT NULL REFERENCES organizations(id),
     provider           text NOT NULL CHECK (provider IN
                          ('google_calendar','slack','jira','linear')),  -- R8 closed enum
     external_account_id text,
     status             text NOT NULL DEFAULT 'active'                  -- active|degraded|disconnected
                          CHECK (status IN ('active','degraded','disconnected')),
     scopes             text[] NOT NULL DEFAULT '{}',
     installed_by       uuid NOT NULL,
     created_at         timestamptz NOT NULL DEFAULT now(),
     updated_at         timestamptz NOT NULL DEFAULT now(),
     UNIQUE (org_id, provider)            -- at most one connection per provider per tenant
   );
   CREATE TABLE oauth_tokens (
     id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
     org_id             uuid NOT NULL REFERENCES organizations(id),
     connection_id      uuid NOT NULL REFERENCES integration_connections(id),
     access_token_enc   bytea NOT NULL,   -- AES-256-GCM; wrapped DEK+nonce packed inside
     refresh_token_enc  bytea,
     expires_at         timestamptz,
     token_version      int  NOT NULL DEFAULT 1
   );
   -- RLS FORCE + canonical tenant_isolation policy on BOTH tables.
   ```
   Plus the two service-private maps (external↔NEXUS id map, webhook cursors), `org_id`-scoped.
2. **Envelope-encrypt tokens** ([21 §2.2](../21-integration-architecture.md), [11 §6.2/§7](../11-security-architecture.md)): per-tenant KMS data key (DEK) wrapped by the tenant KEK; AES-256-GCM; self-describing `[wrapped_dek_len ‖ wrapped_dek ‖ nonce ‖ ciphertext ‖ tag]` packed inside the bytea. Zeroize the plaintext DEK after use. **No API path returns token material**; structured logs redact `*token*`/`*secret*`/`code`/`client_secret`. `token_version` increments on refresh; N failed refreshes → `status='degraded'` + a re-auth `notifications` event.
3. **OAuth flow engine** (auth-code + PKCE, server-mediated). HMAC `state` binds `org_id`,`user_id`,nonce,exp=10m; PKCE verifier in `nexus:{org_id}:oauth:{state}` (10-min TTL). Callback at **`/oauth/{provider}`**; **RBAC-gated to `owner`/`admin`** at `bff` and re-checked in `integration-service`. Validate `{provider}` against the closed enum AND the one-PM-connection rule before minting the authorize redirect.
4. **Webhook receiver** at **`/webhooks/{provider}`**: (1) signature verify (Slack `v0` signing secret + 5-min skew; Jira/Linear shared-secret HMAC; Google channel token + TLS); (2) replay guard / dedupe by provider delivery id; (3) fast-ACK `200` to an inbox row within the provider deadline (Slack 3s, Google ~10s); (4) async worker drains the inbox idempotently and advances the cursor.

**Canonical names:** `integration_connections`, `oauth_tokens`, `/oauth/{provider}`, `/webhooks/{provider}`, `integration-service`, `nexus:{org_id}:oauth:{state}`.

**Acceptance checks:**
- A second PM connection install (Linear while Jira active) is rejected; the existing PM connection must be disconnected first.
- `GET` endpoints expose only `status`/`scopes`/`provider`/`external_account_id`/`installed_by` — never token material; logs redact token fields.
- A replayed webhook (same delivery id) is deduped; ACK returns inside the provider deadline; the cursor advances exactly once.
- RLS test on both tables; a `member` cannot hit `/oauth/{provider}` (403 → not a tenant leak path).

#### 3.11 `integration-service` — Google Calendar + Slack + ONE of Jira/Linear connectors

**Repo / folder:** `mytheus-backend/services/integration-service/connectors/`.

**Why:** these are the provider-specific glue over §3.10. **R8 hard cap: Calendar + Slack + exactly ONE of Jira/Linear.** The DECISION-REQUIRED in §8 must pick Jira vs Linear before this step; the band's stated end goal is the **live Jira** connection.

**Steps:**

1. **Google Calendar connector** ([21 §3](../21-integration-architecture.md)):
   - Register a push channel (`events.watch`) per connected calendar; store channel id + `resourceId` + `syncToken` in the cursor map; renew before ~7-day expiry via a `MeetingLifecycleWorkflow` child timer.
   - On push: `events.list(syncToken)` → delta; resolve attendee emails → `user_id` via `identity-tenant-service` (tenant-scoped on `users.email`); emit `integration-events {kind=calendar_event_upserted}` for `core-api` to upsert `meetings` + `meeting_participants` (idempotent on `(org_id, calendar event_id)`). `integration-service` does NOT write `meetings`.
   - Attachments: download bytes (Drive scope), write to `nexus-attachments/{org_id}/{meeting_id}/`, emit `integration-events {kind=attachment_to_ingest}` for `ingestion-service` (reuses the unstructured.io → `document_chunks` → `document_chunks__{org_id}` path).
2. **Slack connector** ([21 §4](../21-integration-architecture.md)): bot OAuth + workspace/channel/user id maps. `integration-service` **builds** Block Kit (prep briefs, post-meeting summaries) but **delivery goes through `notification-service`** via the `notifications` topic (the sole external fan-out). Prep brief: a `MeetingLifecycleWorkflow` child timer fires at T-15m per matched participant; builds a personalized brief from `get_user_context` + a `rag-service` query; one `notifications {channel=slack_dm}` per user. Post-meeting summary on `meeting-events {ended}`: confirmed decisions/actions only.
3. **Jira connector (the end goal) — bidirectional `action_items` sync** ([21 §5](../21-integration-architecture.md)):
   - **Outbound:** consume `actions` (confirmed, from `core-api` outbox); create a Jira issue `POST /rest/api/3/issue` with field mapping (`content`→`summary`, long-form+back-link→`description` ADF, `assignee_id`→accountId, `due_date`→`duedate`, status per §5.3 map, `id`→label `nexus:{id}` + remote link). Carry `actions.header.idempotency_key`; first look up the id map, fall back to a provider-side idempotent search on the `nexus:{id}` label. Write the external key/URL to the id map.
   - **Inbound:** Jira status webhook → verify → dedupe → id-map lookup → if status differs from `last_synced_status`, map external→NEXUS status and emit `actions {status sync}` for `core-api` to update `action_items.status` (idempotent). **Loop suppression:** drop the webhook when its status already equals `last_synced_status` (an echo of NEXUS's own write).
   - **Status map** (integration-service-defined): NEXUS `open`/`in_progress`/`done`/`cancelled` ↔ Jira To Do/In Progress/Done/Won't-Do; unmapped external states fall to `open` + a low-priority "review status mapping" `notifications`.
   - Assignee resolution: `users.id` → Jira accountId via the install-time account map + email match; if none, create unassigned + a re-map `notifications`.

**Canonical names:** `integration-events`, `IntegrationEvent`, `actions`, `Action`, `meetings`, `meeting_participants`, `nexus-attachments`, `document_chunks__{org_id}`, `notifications`, `get_user_context`, `rag-service`, `ingestion-service`.

**Acceptance checks:**
- **Calendar:** a created/updated event auto-creates/patches a `meetings` row (one per calendar event, idempotent); attendees email-match to `meeting_participants`; an attachment lands in `nexus-attachments` and becomes searchable in `document_chunks__{org_id}` before the meeting.
- **Slack:** a per-participant Block Kit prep brief delivers at T-15m via `notification-service`; the post-meeting summary contains only confirmed decisions/actions.
- **Jira (headline):** confirming a NEXUS action creates a real Jira issue with the correct fields and the `nexus:{id}` label; a redelivered `actions` event does NOT create a duplicate ticket; moving the issue to Done in Jira flips `action_items.status` to `done`; a status change that originated in NEXUS is loop-suppressed (no echo).

#### 3.12 `workflow-service` — `ActionFollowUpWorkflow` (day 1/3/7 + day-10 escalation)

**Repo / folder:** `mytheus-backend/services/workflow-service/workflows/action_followup.py`.

**Why after the Jira connector:** the follow-up chases overdue confirmed actions whose status is now kept in sync with Jira; escalation delivers a Slack DM via `notification-service`.

**Steps:**

1. **Start on `actions {confirmed}`** with workflow id `actionfollowup-{action_item_id}` (dedupes duplicate starts). Each reminder activity re-reads `action_items.status` via `core-api`; on `done`/`cancelled` (from either sync direction) the workflow completes early.
   ```python
   @workflow.defn(name="ActionFollowUpWorkflow")
   class ActionFollowUpWorkflow:
       @workflow.run
       async def run(self, action_id: str, org_id: str) -> None:
           for days in (1, 2, 4, 3):                  # cumulative: day 1, 3, 7, 10
               await workflow.sleep(timedelta(days=days))   # durable timer
               status = await workflow.execute_activity(
                   read_action_status, args=[action_id, org_id],
                   start_to_close_timeout=timedelta(seconds=15))
               if status in ("done", "cancelled"):
                   return
               if days == 3:                            # the day-10 hop
                   await workflow.execute_activity(
                       escalate_to_manager, args=[action_id, org_id], ...)
               else:
                   await workflow.execute_activity(
                       remind_assignee, args=[action_id, org_id], ...)
   ```
2. **Reminders (day 1/3/7):** emit a `notifications` event (assignee DM/in-app) through `notification-service`. Idempotent + deduped downstream so an at-least-once timer never double-pings.
3. **Day-10 escalation (best-effort, see §8 DECISION):** resolve the assignee's manager from the `departments` tree (`departments.parent_id`) + `teams.department_id` + `team_members`; if unresolvable, escalate to `meetings.created_by` (organizer), then a tenant admin. The escalation is a Slack DM via `notification-service`. **The data model has no `manager_id`/reporting edge** — this is documented best-effort.
4. **Cancellation:** a status change signals the workflow to complete; Temporal durable timers guarantee no missed/duplicate reminder across worker restarts.

**Canonical names:** `ActionFollowUpWorkflow`, `actions`, `action_items`, `notifications`, `notification-service`, `departments`, `teams`, `team_members`, `meetings.created_by`, `workflow-service`.

**Acceptance checks:**
- A confirmed action that stays open fires day-1, day-3, day-7 reminder DMs; a day-10 escalation DM to the resolved manager (or organizer/admin fallback).
- Marking the action `done` (via Jira sync or UI) before day 10 completes the workflow with no further reminders.
- Killing + restarting the worker mid-timer does not drop or duplicate a reminder.

#### 3.13 `admin-ops-service` — both consoles + `feature_flags`, `ops_audit_log`, `support_grants`

**Repo / folder:** `mytheus-backend/services/admin-ops-service/`.

**Why now:** P12 is when there is a plan, membership, billing, and an integration surface worth administering ([22 §3.1](../22-roadmap-and-phasing.md): admin-ops stores land at P12).

**Steps:**

1. **Materialize the admin-ops stores** (canon names): `feature_flags`, `ops_audit_log`, `support_grants` (+ the `audit_logs` trail, live since P2). RLS FORCE where tenant-scoped.
   ```bash
   uv run alembic revision -m "feat(admin-ops): add feature_flags, ops_audit_log, support_grants"
   ```
2. **Tenant-admin console** `/admin/tenant/*` (RBAC-gated, every write to `audit_logs`): Members (invite/remove, RBAC tier change, ownership transfer); Billing (plan change, seats, invoice download, Stripe Customer Portal link — `owner`); Usage (consumption vs quota from `/usage`); Integrations (install/revoke Calendar/Slack/Jira-Linear, R8); SSO & SCIM (Enterprise); Settings (`tenant_settings` retention, PII redaction, agent confidence overrides); Data export & erasure (→ `nexus-exports` / RTBF). Also exposes the **A2A grant template** management (§3.2).
3. **Internal-ops console** `/admin/ops/*` (NEXUS staff only, deny-by-default, full audit, **no standing cross-tenant data access**): tenant directory; suspend/reinstate (emits `admin-events {tenant suspended}` → `workflow-service`); tenant migrate (residency workflow — Plan 14 trigger only); **feature flags** (`admin-events {flag changed}`); **cost & lag dashboards** (consumer-lag, per-tenant LLM cost, token-budget burn, DLQ rate — reads `billing-events`/`graph-events`/Prometheus, aggregate queries over control-plane events, NOT tenant rows); **audit-log access** (read-only on `audit_logs`, access itself audited); **support impersonation** (time-boxed, customer-consented, every action tagged `actor_type=system`).
4. **Consume `tenant-events`/`billing-events`/`graph-events`** for the cross-tenant views; **produce `admin-events`** (`flag changed`, `tenant suspended`, `erasure requested`) that `workflow-service` turns into durable orchestrations.

**Canonical names:** `admin-ops-service`, `feature_flags`, `ops_audit_log`, `support_grants`, `audit_logs`, `/admin/tenant/*`, `/admin/ops/*`, `admin-events`, `tenant-events`, `billing-events`, `graph-events`, `nexus-exports`.

**Acceptance checks:**
- Tenant-admin: an admin installs Jira, changes plan, sets a confidence override, requests a GDPR export (lands in `nexus-exports`, auto-expiring) — each appends to `audit_logs`.
- Internal-ops: a staff suspend emits `admin-events {tenant suspended}` → `workflow-service` flips `organizations.status=suspended` (via `identity-tenant-service`, which is the sole `tenant-events` producer); the cost/lag dashboard renders from aggregate event queries, NOT tenant-row selects; impersonation is time-boxed and every impersonated action is tagged `actor_type=system`.
- A feature-flag flip emits `admin-events {flag changed}`.

#### 3.14 `web-app` — integration settings page + both consoles

**Repo / folder:** `mytheus-ui/` (`web-app`).

**Steps:**

1. **Integration settings page** (RBAC owner/admin): per-provider status/`scopes`/`installed_by`/last-sync; Calendar (watched calendars, unmatched-attendee map/invite); Slack (default channel, per-user DM prefs link); PM as a **single radio** (Jira default-selected; the unselected option's Connect disabled — R8 UI enforcement). Never displays token material.
2. **Tenant-admin console** screens over `/admin/tenant/*`; **internal-ops console** over `/admin/ops/*` (staff build).

**Acceptance checks:**
- The PM connector renders as one radio; selecting Jira disables Linear's Connect.
- No token material is ever rendered.

---

### PHASE P13 — Testing / DevOps / Hardening / Production (W34-36)

#### 3.15 `billing-metering-service` — the full Stripe surface (subscriptions / invoices / plans / seats / dunning / tax)

**Repo / folder:** `mytheus-backend/services/billing-metering-service/`.

**Why P13:** metering + per-tenant budgets shipped at P7 (the R4 backstop); the full Stripe **subscriptions/seats/dunning/tax** surface is the P13 monetization completion ([22 §3.1](../22-roadmap-and-phasing.md), [18 §4](../18-saas-business-architecture.md)).

**Steps:**

1. **Materialize the billing tables** if not present: `subscriptions`, `invoices` (immutable once finalized, 7-yr, `nexus-invoices` object lock), `plans` (global, no `org_id` — the deliberate exception). `seats` as Stripe subscription quantity.
2. **Stripe object mapping:** `organizations`→Customer, `subscriptions`→Subscription, seats→quantity, metered overage→metered price items (reported from `usage_aggregates`), `invoices`→Invoice, tax→**Stripe Tax** (VAT/GST/US sales tax; VAT-ID validation for EU B2B reverse-charge).
3. **`POST /stripe/webhook`** — signature-verified, idempotent on the Stripe event id via `nexus:{org_id}:dedupe:stripe:{event_id}`. Translate `customer.subscription.created/updated`→sync `subscriptions`/`seats`; `invoice.finalized`→write `invoices` + PDF to `nexus-invoices`; `invoice.paid`→mark paid + call `identity-tenant-service` to flip `organizations.status=active` if recovering; `invoice.payment_failed`→`past_due` + start dunning. **`billing-metering-service` never produces `tenant-events`** — it calls `identity-tenant-service` which owns `organizations.status`.
4. **Dunning saga** (`workflow-service`, durable timers): day 0 (email + in-app, `past_due`), retry (Stripe smart retries), day 7 (reminder #2 + owner Slack DM), day 14 (reminder #3 + agent-throttle warning → `agent_budgets.state=throttled`), day 21 (`workflow-service` calls `identity-tenant-service.suspend(org_id)` → `tenant-events {suspended}`). Delivery via `notification-service`.

**Canonical names:** `subscriptions`, `invoices`, `plans`, `usage_aggregates`, `agent_budgets`, `billing-events`, `/stripe/webhook`, `/billing/*`, `nexus-invoices`, `nexus:{org_id}:dedupe:stripe:{event_id}`.

**Acceptance checks:**
- A new Pro subscription bills $25/seat; metered overage reports to Stripe from `usage_aggregates`; an invoice finalizes immutably into `nexus-invoices` under object lock.
- Stripe Tax computes VAT for an EU B2B customer with reverse-charge; `invoices.tax_cents` mirrors it.
- A failed payment runs the dunning saga to day-21 suspension; `tenant-events {suspended}` is emitted by `identity-tenant-service`, not billing.
- A redelivered Stripe webhook is a no-op (deduped on event id).

#### 3.16 Playwright E2E (web + mobile) + k6/Locust load

**Repo / folder:** `mytheus-ui/e2e/`, `mytheus-mobile/e2e/`, `mytheus-platform/load/`.

**Steps:**

1. **Playwright E2E** ([16 §5.2](../16-deployment-and-infrastructure.md)) covering the full loop: sign in (Clerk) → join meeting → captions → agent sidebar (context-fill card confirm, conflict banner) → confirm decision → Jira ticket created → follow-up reminder → notification tap. **Include `mytheus-mobile`** via Detox/Playwright-mobile in the same release pipeline.
2. **k6/Locust load** pinned to the §5 SLOs ([13 §7](../13-scalability-and-performance.md)) — at MVP scale, run the scenarios scaled to the Railway/Render single-region capacity, with the anchors as the design target:
   - LT-1 meeting throughput (Locust, agent e2e p50<2s/p95<4s); LT-3 LLM/agent pipeline (tier mix Haiku 45%/Sonnet 55%/Opus <1%, budget enforced 100%, no DLQ growth); LT-4 RAG (k6, warm<500ms/cold<1.5s); LT-9 **noisy-neighbor** (one tenant at 10× share must not regress another tenant's SLO — the SaaS-defining test); **A2A round-trip** (p95 < 1.5s under E7=12 round-trips/meeting).
3. **CI gate:** a run fails the build if a target p95 is breached or error rate > 0.1%; nightly k6 in GitHub Actions gates the staging deploy.

**Acceptance checks:**
- The web + mobile Playwright/Detox suites pass against staging.
- LT-3 holds the cost-shape assertion (tier mix + ~35% gate-pass); LT-9 shows zero SLO regression for non-burst tenants; A2A p95 < 1.5s.

#### 3.17 ~30% LLM cost optimization (C4), measured against the live pipeline

**Repo / folder:** `mytheus-intelligence/services/agent-runtime-service/`, `rag-service/`.

**Steps:**

1. **Measure the live baseline:** LLM cost/tenant/day and cost/MAU from `usage_aggregates` (per [15 §4.1](../15-observability-and-sre.md)); the target is `C2` ≈ $9.73 → `C4` ≈ $6.99 (~30% down, the PDF R4 P13 target measured against the real pipeline, not naive).
2. **Apply the levers** (all already locked, now tuned with live data): tighten the relevance pre-gate toward ~35% pass; widen the window where freshness allows; confirm tiered routing isn't over-escalating to Opus (Opus <1%); lean hardest on **Anthropic prompt caching + semantic cache** (the dominant lever); per-user LLM rate limit via `nexus:{org_id}:ratelimit:{user_id}`.
3. **Lock cost as a CI assertion** (LT-3 cost-shape) and an error budget ([15 §6.3](../15-observability-and-sre.md)): a drift toward more Sonnet/Opus is a cost regression caught in CI.

**Canonical names:** `usage_aggregates`, `agent_budgets`, `C2`, `C3`, `C4`, `L7`, `L10`, `nexus:{org_id}:ratelimit:{user_id}`.

**Acceptance checks:**
- Measured cost/MAU is ~30% below the pre-optimization baseline and within ~$6.99 (C4); the tier mix and gate-pass rate hold; the CI cost-shape assertion is green.

#### 3.18 Final security attestation

**Repo / folder:** `mytheus-platform/security/`, all services.

**Steps:**

1. **Run the attestation over the continuously-secured system** ([11 §9](../11-security-architecture.md)): the P13 "security audit" is a **final attestation**, not the first security work (threat model authored P1, authz reviewed every PR since P2, SCA + `gitleaks` from day one).
2. **Verify the controls:** RLS FORCE on every tenant table (fail-closed: unset `app.current_org` → zero rows); authz middleware deny-by-default; A2A deny-by-default + audit; `oauth_tokens` envelope-encrypted + never returned/logged; `audit_logs` append-only; per-tenant KMS crypto-shred on RTBF; STRIDE matrix controls all map to a canonical mechanism.
3. **Pre-Enterprise-GA pen test / SOC2 readiness** review against the controls.

**Acceptance checks:**
- A scripted cross-tenant query attempt returns zero rows (RLS backstop) and fires the `org_id`-missing alarm ([15 §9.6](../15-observability-and-sre.md)).
- The attestation checklist is signed; no critical finding open.

#### 3.19 Postgres backups + PITR + read replica; `audit_logs` partition/archival

**Repo / folder:** `mytheus-platform/data/`.

**Steps:**

1. **Enable continuous WAL archiving + PITR** and provision an async **read replica** (lag < 5s) on the managed/Supabase Postgres ([14 §4](../14-reliability-and-dr.md)). Route read-heavy paths (`bff` dashboard reads, `search-service` FTS, `rag-service` FTS arm) to the replica; writes + read-your-write (token mint, decision confirm) to the primary.
2. **`audit_logs` monthly range partitioning + cold archive** to `nexus-audit-archive` (object lock); 1y hot. (The table has captured since P2; P13 hardens partitioning/archival, [22 §3.1](../22-roadmap-and-phasing.md).)
3. **RPO/RTO:** in-region PITR RPO ≈ seconds, RTO ≤ 2h (Free single-region tier — the MVP go-live posture). Multi-region cross-replica is Plan 14.

**Acceptance checks:**
- A PITR restore to a pre-incident timestamp succeeds in staging; the read replica serves dashboard reads at < 5s lag.
- A monthly `audit_logs` partition rolls to `nexus-audit-archive` under object lock.

#### 3.20 Cloudflare + custom domain + live production deploy (Railway/Render)

**Repo / folder:** `mytheus-platform/edge/`, `mytheus-platform/deploy/`.

**Steps:**

1. **Cloudflare CDN + WAF + custom domain** ([11 §5](../11-security-architecture.md), [16 §8.1](../16-deployment-and-infrastructure.md)): TLS 1.3 termination, L3/L4 DDoS absorption, OWASP CRS WAF, bot management; static `web-app` assets CDN-cached; the API path proxied to `api-gateway`. Global + per-tenant rate limits at the gateway.
2. **Production deploy on Railway/Render** (RESOLVED DECISION 3): the modular monolith with the 20 catalog modules drawn along their final seams; **single region, pooled + PostgreSQL RLS**. GitHub Actions promotes the digest-pinned image through staging → prod after the staging smoke + load gate ([16 §5](../16-deployment-and-infrastructure.md)). **No K8s, no Terraform cells, no multi-region — that is Plan 14.**
3. **Custom domain DNS + health-checked routing** via Cloudflare; environment secrets (Clerk, Stripe live, LiveKit, Deepgram, LLM, Redpanda) injected as platform env vars (the Railway/Render equivalent of ESO+KMS — KMS still backs the per-tenant `oauth_tokens` DEKs).
4. **Onboard the first paying tenant** through the self-serve funnel (trial-first, role-capture step 1, Calendar connect as the activation accelerant).

**Acceptance checks:**
- The app is reachable at the custom domain behind Cloudflare with a valid TLS 1.3 cert; WAF blocks an OWASP-CRS probe; per-tenant rate limit sheds a burst at the gateway.
- A merge to `main` promotes through staging (smoke + load gate green) to the live Railway/Render prod; a rollback is a digest revert.
- A first real tenant signs up, connects Calendar, runs a meeting with the live agent sidebar, and a confirmed action becomes a live Jira ticket.

---

## 4. Data stores stood up in this band

Cite [22-roadmap-and-phasing.md §3.1](../22-roadmap-and-phasing.md) (stores by phase):

| Phase | Postgres | AGE `nexus_knowledge` | Qdrant | Redis (new keys) | Object buckets |
|---|---|---|---|---|---|
| **P10** | `agent_authz_scopes` (identity, ADR-0013); `agent_collaborations` (core-api, if not earlier) | reads `FOLLOWS_FROM`/`ASSIGNED_TO`/`ATTENDED` (confirmed edges from P8) | reads `decisions__{org_id}` | `nexus:{org_id}:a2a:grant:*` (60s TTL) | — |
| **P11** | — | reads lineage for `ProactiveAgent` | reads `decisions__{org_id}` (conflict complement) | `nexus:{org_id}:queue:agent:{user_id}`; `nexus:{org_id}:pubsub:meeting:{meeting_id}` | — |
| **P12** | `integration_connections`, `oauth_tokens` (integration); `feature_flags`, `ops_audit_log`, `support_grants` (admin-ops) | — | — | `nexus:{org_id}:oauth:{state}` (10-min TTL) | `nexus-attachments`; `nexus-exports` (export jobs) |
| **P13** | `subscriptions`, `invoices`, `plans` (full Stripe surface); `audit_logs` partitioning/archival hardened (table live since P2) | — | — | `nexus:{org_id}:dedupe:stripe:{event_id}` | `nexus-invoices`, `nexus-audit-archive`, `nexus-exports` |

All tenant-scoped tables carry `org_id UUID REFERENCES organizations(id)` with RLS `ENABLE` + `FORCE` and the canonical `tenant_isolation` policy; `plans` is the deliberate global exception (no `org_id`). Postgres PITR + read replica are configured in P13 (§3.19).

---

## 5. Events / topics introduced  (Avro subjects, producers/consumers, DLQs)

All topics are Avro + Schema Registry (subject `{topic}-value`, **BACKWARD**), partitioned by `org_id`, at-least-once with the `EventEnvelopeHeader`, each paired with `<topic>.dlq` (14-day retention). This band lights up (not invents) the following edges:

| Topic | Avro schema | Produced (here) | Consumed (here) | Notes |
|---|---|---|---|---|
| `agent-requests` | `AgentRequest` (frozen) | `agent-runtime-service` (ContextGapAgent) | `agent-runtime-service` (peer agents) | `idempotency_key=areq:{correlation_id}`; foreign `org_id` → `agent-requests.dlq`, never served (ADR-0013) |
| `agent-responses` | `AgentResponse` (frozen) | `agent-runtime-service` (responder) | `agent-runtime-service` (requester) | deadline-bounded; 1-day retention |
| `agent-stream` | `AgentStreamDelta` (net-new) | `agent-runtime-service` (LiteLLM stream bridge) | `realtime-gateway` → WebSocket | 1-hour retention, throwaway; shed first under backpressure |
| `agent-results` | `AgentResult` (frozen) | `agent-runtime-service` | `core-api`, `realtime-gateway`, `notification-service` | carries ContextGap/Proactive candidates |
| `integration-events` | `IntegrationEvent` (net-new) | `integration-service` | `ingestion-service`, `core-api` | `calendar_event_upserted`, `attachment_to_ingest`, `external_status_changed`; 12 partitions, 7-day |
| `actions` | `Action` (frozen) | `core-api` (on confirm), `integration-service` (status sync) | `integration-service`, `workflow-service`, `knowledge-graph-service`, `core-api` | bidirectional Jira sync; loop-suppressed via `last_synced_status` |
| `notifications` | `Notification` (frozen) | `integration-service`, `billing-metering-service`, `workflow-service`, `identity-tenant-service` (+ agent tier) | `notification-service` | prep briefs, summaries, reminders, re-auth/re-map prompts, dunning |
| `notification-delivery-events` | `NotificationDeliveryEvent` (net-new) | `notification-service` | `admin-ops-service`, `core-api` | delivery receipts |
| `billing-events` | `BillingEvent` (net-new) | `billing-metering-service` | `workflow-service`, `notification-service`, `admin-ops-service` | quota-approaching, dunning, plan-changed |
| `admin-events` | `AdminEvent` (net-new) | `admin-ops-service` | `workflow-service`, `admin-ops-service` (audit write) | flag changed, tenant suspended, erasure requested; compacted |
| `graph-events` | `GraphEvent` (net-new) | `knowledge-graph-service` | `admin-ops-service` | edge created/lineage updated; feeds cost/lag dashboards |
| `meeting-events` | `MeetingEvent` (net-new) | `core-api` | `workflow-service`, `integration-service` | starts `MeetingLifecycleWorkflow`; schedules prep-brief/summary |

DLQ replay is an `admin-ops-service` operation re-producing with the **original `idempotency_key`** so dedupe/inbox decides novelty ([07 §7](../07-event-architecture.md)). Per-tenant `org_id` lag tags make a noisy-neighbor incident isolable.

---

## 6. SaaS overlay / preconditions landing here

This band lands the back half of every SaaS overlay stream and closes the W36 MVP:

- **Security shift-left → final attestation (P13, §3.18).** The continuous program (threat model P1, authz-in-every-PR P2+, SCA/secret-scan day one) culminates in a final attestation + pre-Enterprise pen test/SOC2 readiness. A2A deny-by-default authz (P10) is the band's most novel security surface; `oauth_tokens` envelope encryption (P12) is the crown-jewel control.
- **Billing + budgets (P13, §3.15).** Metering + per-tenant `agent_budgets` shipped at P7; the full Stripe subscriptions/seats/dunning/tax + immutable `invoices` complete monetization. The ~30% cost optimization (§3.17) protects the C4 margin against the live pipeline.
- **Compliance (P12-P13).** Subprocessor DPAs/SCCs cover the new egress (Slack/Jira/Google + the existing Deepgram/LLM/LiveKit); `data_residency` pins region; RTBF + `nexus-exports`; `audit_logs` partition/archival to `nexus-audit-archive` under object lock. R8 caps the subprocessor surface.
- **Reliability/DR (P13, §3.19).** PITR + read replica + RPO/RTO for the **Free single-region tier** (the MVP posture). Multi-region active-passive, cells, and chaos engineering are **Plan 14**.
- **Dedicated inference (already P6/P7).** This band runs `ContextGapAgent`/`ProactiveAgent` on that tier off the API tier; it does not re-decide it.

---

## 7. Mobile (mytheus-mobile) work in this band

`mytheus-mobile` threads through every phase of this band — it is a **first-class GA client**, not an afterthought. It adds zero backend surface except the one flagged push-token registration (N7), and stays contract-compatible with `mytheus-ui`.

- **P10 — collaboration surfacing.** Render `ContextGapAgent` context-fill cards in the bottom-sheet agent sidebar exactly as web does (correlation-keyed draft → settled on `agent-results`); show the "Context added from {X}'s agent" card with its citation. No A2A logic client-side — the client renders the result the gateway delivers.
- **P11 — proactive alerts + priority queue surfacing.** Render the red/yellow/blue priority strip and the **red conflict banner** (auto-expands the bottom sheet a notch); the per-user priority queue (`nexus:{org_id}:queue:agent:{user_id}`) is server-side — the client renders the order it receives. Stream `agent-stream` deltas into draft cards via the existing `SocketProvider`; a missed delta is non-fatal (final `agent-results` is complete).
- **P12 — integration-driven follow-ups + push (replace the absent push layer).** Surface integration-driven action follow-ups (the `ActionFollowUpWorkflow` reminders) and Jira-synced `action_items.status` changes as they arrive over the notification fabric. **Wire Expo push (N7):** register the Expo device push token with `notification-service` (modeled as a `notification-service` channel-preference write — the one flagged backend addition); delivery rides `notifications` → `notification-service` → Expo Push → APNs/FCM; tap routing via deep links (`mytheus://tasks`, `mytheus://meetings/{id}/room`, `mytheus://notifications`). Push carries no durable truth — `GET /api/v1/notifications` reconciles on foreground. **Integration install stays web-only**; mobile shows integration state read-only.
- **P13 — mobile-responsive parity + GA pipeline.** Achieve read/interactive parity (join + media, captions < 2s, sidebar confirm/edit/dismiss, dashboard/My-Tasks/decisions/transcript) per `system.md §1.1`; admin/billing-purchase stay web-only. **Include mobile in the Playwright/E2E + GA release pipeline:** Detox specs for sign-in, join-meeting, caption render, agent-card confirm, conflict-banner, notification-tap routing run in the same release gate; EAS Build/Submit + OTA channels (`staging`/`production`) ship alongside the web deploy; the mobile build is part of the GA cut.

**Acceptance checks:** the bottom-sheet sidebar renders context-fill cards + the conflict banner with no layout thrash; a backgrounded device receives an Expo push for a follow-up reminder and deep-links to My Tasks; the Detox smoke is green in the GA pipeline; the client never sets `org_id` and never holds a LiveKit/Slack/Jira credential.

---

## 8. DECISION-REQUIRED callouts

> ⚠️ **DR-1 — R6 DESCOPE CHECKPOINT (P10, go/no-go before W29 demo).** If A2A collaboration is unstable or **misses the p95 < 1.5s round-trip budget** under `E7` = 12 round-trips/meeting load, **descope `ContextGapAgent` to RAG-only context fill and ship A2A as v2** ([23 R6](../23-risk-register.md), [22 §5](../22-roadmap-and-phasing.md), [ADR-0013 rollback](../adr/0013-agent-to-agent-authorization.md)). The descope is a **config flip**, not a redesign: set the per-tenant/platform flag to skip the peer-agents + related-meeting hops; `ContextGapAgent` goes straight to the org-scoped RAG hop; the boundary stays enforced by partitioning + RAG org-scoping even with the protocol off. **Options:** (a) ship full A2A; (b) ship A2A behind a 1%-tenant canary; (c) descope to RAG-only now, A2A as v2. **Owner:** AI/Inference lead + Eng lead (joint go/no-go at the P10 demo). **Blocks:** the P10 "Context added from X's agent" demo and the A2A authz attestation; P11 proactive alerts do NOT block on this (they read the graph, not A2A).

> ⚠️ **DR-2 — R8 GUARDRAIL: Jira vs Linear (decide BEFORE P12 / W32).** Ship Calendar + Slack + **exactly ONE** of Jira/Linear ([23 R8](../23-risk-register.md), [21 §9](../21-integration-architecture.md)). The band's stated end goal is the **live Jira** connection (default). **DECIDE WHICH** before P12 connector work: the closed `integration_connections.provider` enum, the one-PM-connection invariant, and the UI radio enforce the cap, but the connector code is provider-specific. **Options:** (a) Jira (default, the stated end goal — Atlassian REST `POST /rest/api/3/issue`); (b) Linear (GraphQL `issueCreate`). **Owner:** Product + the first design-partner tenant's stack. **Blocks:** §3.11 PM connector, the P12 headline demo, the bidirectional-sync acceptance checks. Shipping both is out of scope by policy (requires a superseding ADR, not a feature PR).

> ⚠️ **DR-3 — `manager_id` / reporting edge does NOT exist (decide BEFORE P12 / §3.12).** The data model encodes **no direct manager/reports-to relationship** — neither `nexus_knowledge` (no reporting edge label) nor any column (`departments` has only `parent_id`; `teams` has `department_id`; `team_members` has a `role` but no manager link). The `ActionFollowUpWorkflow` **day-10 manager escalation is best-effort** over the `departments` tree + organizer/admin fallback ([21 §6](../21-integration-architecture.md)). **Options:** (a) ship best-effort escalation (departments-tree → `meetings.created_by` → tenant admin) — no schema change; (b) add a first-class **`manager_id`** to the org model (a **Data Model + Glossary change** + ADR, propagated canon-first) before P12 so escalation can be authoritative. **Owner:** Data-model owner + Product. **Blocks:** the authoritative-escalation acceptance check in §3.12 (best-effort ships regardless; first-class `manager_id` is the upgrade).

> ⚠️ **DR-4 — Confirm MVP go-live target + per-plan availability tiers (BEFORE P13 / §3.20).** Confirm the MVP go-live is **Railway/Render single-region, pooled + PostgreSQL RLS** (RESOLVED DECISION 3 — Kubernetes/cells/multi-region are **Plan 14**), and **confirm the per-plan availability tiers for launch** (Free 99.5% single-region, Pro 99.9%, Enterprise 99.95% — [14 §2](../14-reliability-and-dr.md), [18 §3.1](../18-saas-business-architecture.md)). At single-region MVP, Pro/Enterprise multi-region SLAs cannot be met until Plan 14; decide whether to (a) launch all tiers at the Free single-region availability and gate the higher SLAs behind Plan 14, or (b) hold Enterprise GA until cells/multi-region land. **Owner:** Eng lead + Product/GTM. **Blocks:** the §3.20 production deploy posture and the public SLA commitments at launch.

---

## 9. Risk checkpoints

- **R6 (P10) — multi-agent collaboration novelty / A2A latency.** Checkpoint at the P10 demo: if A2A misses p95 < 1.5s or is unstable, invoke the pre-authorized descope to **RAG-only context fill, ship A2A as v2** (DR-1). The fallback chain `peer-agents → related-meeting agents → RAG → silent` already degrades per-request; the descope removes the peer hops entirely. Owner: AI/Inference + Eng lead. ([23 §3.6 R6](../23-risk-register.md))
- **R8 (P12) — integration scope creep.** Hard guardrail: **Calendar + Slack + ONE of Jira/Linear** (DR-2), enforced at the closed `provider` enum, the one-PM-connection invariant, the UI radio, and `/oauth/{provider}` validation. Pre-authorized further descope: **Calendar + Slack only** if scope pressure threatens W36. Owner: Product. ([23 §3.8 R8](../23-risk-register.md))
- **Adjacent residuals exercised here:** **R10** (A2A data leak — deny-by-default authz + foreign-`org_id`-DLQ, P10); **R12** (LLM cost residual — the ~30% C4 optimization + budget enforcement, P13); **R17/R18** (event/webhook duplication — idempotency + loop suppression on Jira sync, P12); **R19** (realtime fan-out — Redis pub/sub backbone + priority queue, P11); **R14** (billing/provisioning — full Stripe surface + dunning saga, P13). Each owned by its doc per [23 §5](../23-risk-register.md).

---

## 10. Definition of Done & end-of-phase demo  (PDF cadence: a demo at the end of EVERY phase)

**P10 DoD (W29):** `ContextGapAgent` + A2A PEP/PDP + deny-by-default `agent_authz_scopes` live; `MeetingLifecycleWorkflow` runs real activities; `agent_collaborations` persisted + log UI. **Demo:** a live meeting where A's agent fills a gap from B's agent (peer hop), a cross-tenant probe is DLQ'd + audited, the fallback chain falls through to RAG on a deadline miss, and the collaboration log shows grants + denials. Gates: A2A p95 < 1.5s (or DR-1 descope invoked); RLS tests green; authz-in-PR green.

**P11 DoD (W31):** `ProactiveAgent` (conflict/workload/stakeholder/dependency) live over the graph + `decisions__{org_id}` complement; `realtime-gateway` Redis pub/sub backbone + `agent-stream` deltas + per-user priority queue; `notification-service` three-channel delivery; alert UI + sidebar polish. **Demo:** a confirmed decision raises a red conflict banner (and still fires via the Qdrant complement with `knowledge-graph-service` killed — R1 hot degradation); a workload alert; streamed deltas render progressively cross-pod; urgent alerts pre-empt notes.

**P12 DoD (W33) — the headline:** `integration-service` framework + Calendar + Slack + **live Jira**; `ActionFollowUpWorkflow`; both `admin-ops-service` consoles. **Demo:** an admin OAuth-connects Jira on the settings page; a confirmed action becomes a real Jira issue; a Jira "Done" flips `action_items.status` (loop-suppressed); a day-1 follow-up Slack DM fires; a Calendar event auto-creates a meeting + ingests its attachment; the internal-ops cost/lag dashboard + tenant suspend operate. Gates: R8 cap enforced (DR-2 decided); webhook idempotency + loop suppression proven.

**P13 DoD (W36) — GA:** Playwright (web + mobile) + k6/Locust green against §5 SLOs; final security attestation signed; LLM cost ~30% down (C4); full Stripe surface live; PITR + read replica; `audit_logs` archival; Cloudflare + custom domain; **live production deploy on Railway/Render**; first paying tenant onboarded. **Demo:** the public custom-domain app, a real tenant running a meeting end-to-end (captions → context-fill → conflict alert → confirm → live Jira ticket → follow-up push to mobile), the cost dashboard showing C4, and a staging→prod promote + rollback. Gates: DR-4 confirmed (single-region MVP posture + per-plan SLAs); noisy-neighbor LT-9 shows zero cross-tenant regression; error-budget policy armed.

---

## 11. Source docs  (authoring references used, by path)

- [21-integration-architecture.md](../21-integration-architecture.md) — `integration-service` framework, OAuth + envelope-encrypted `oauth_tokens`, Calendar/Slack/Jira-Linear connectors, `ActionFollowUpWorkflow`, R8 guardrail, status map + loop suppression.
- [18-saas-business-architecture.md](../18-saas-business-architecture.md) — Stripe subscriptions/seats/dunning/tax, usage metering + `agent_budgets`, plans/tiers, both consoles, dunning saga, tenant lifecycle.
- [13-scalability-and-performance.md](../13-scalability-and-performance.md) — SLOs, the load-test scenario catalog (LT-1..LT-9, noisy-neighbor, cost-shape), Redis pub/sub fan-out, caching, the C4 cost target.
- [14-reliability-and-dr.md](../14-reliability-and-dr.md) — PITR/replica RPO/RTO, the R6 hot fallback chain, R1 hot degradation (Qdrant `decisions__{org_id}` complement), per-store DR.
- [11-security-architecture.md](../11-security-architecture.md) — Clerk identity, RBAC tiers, A2A authz cross-ref, envelope encryption, edge/WAF, audit log, shift-left → final attestation.
- [15-observability-and-sre.md](../15-observability-and-sre.md) — cost/tenant/day metering, error budgets, dashboards (cost/lag), runbooks, alerting.
- [16-deployment-and-infrastructure.md](../16-deployment-and-infrastructure.md) — Railway/Render → K8s path (K8s is Plan 14), CI/CD gates, Cloudflare/custom domain, expand/contract migrations.
- [23-risk-register.md](../23-risk-register.md) — R6/R8 checkpoints + pre-authorized descopes, R10/R12/R14/R17/R18/R19 residuals.
- [07-event-architecture.md](../07-event-architecture.md) — topic catalog, envelope, outbox/inbox/dedupe/DLQ, duplicate-decisions guards, backpressure.
- [adr/0013-agent-to-agent-authorization.md](../adr/0013-agent-to-agent-authorization.md) — PEP/PDP split, derived identity, `agent_authz_scopes`, the three rules, 60s decision cache, R6 descope.
- [adr/0007-event-delivery-idempotency.md](../adr/0007-event-delivery-idempotency.md) — at-least-once + idempotency, outbox/inbox, DLQ, BACKWARD compatibility.
- [22-roadmap-and-phasing.md](../22-roadmap-and-phasing.md) — the P10-P13 phase rows, §3.1 stores-by-phase, R6/R8 descope context, the W36 MVP boundary.
- `mytheus-backend/system.md` — the ten bounded contexts, one-writer rule, confirmation rule, event-delivery contract, Temporal workflows.
- `mytheus-mobile/system.md` — the parity matrix, push-token N7 addition, offline strategy, the net-new decisions log.
