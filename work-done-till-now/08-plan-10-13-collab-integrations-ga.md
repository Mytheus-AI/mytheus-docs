# Plan 10-13: Multi-Agent, Proactive Intelligence, Integrations and Production GA

**Band:** [plan/plan-10-13-collab-integrations-ga.md](../plan/plan-10-13-collab-integrations-ga.md)
**Status:** IN PROGRESS. P10 built, verified per repo, security-reviewed, merged and deployed to production on 2026-11-10 (section 1); the live two-person demo is still to do. P11, P12 and P13 not started.

---

## 1. Phase P10: Workflows and Multi-Agent Collaboration (2026-11-10)

**Status:** P10 DONE WITH GAPS: every component of plan sections 3.1 to 3.5 is built, tested in its repo with that repo's CI mirrored locally, reviewed for security, fixed after review, merged and deployed to production (1.7). Not yet demonstrated in a live meeting (1.6).

Build decisions: [ADR-0035](../adr/0035-plan-10-context-gap-agent-and-a2a-build.md) (D1 to D11), on top of [ADR-0013](../adr/0013-agent-to-agent-authorization.md). Canon first: `_canon/02-data-model.md` (`agent_collaborations` full column list, `agent_authz_scopes`, `nexus:{org_id}:a2a:grant:{grantee_role}:{query_class}:{target_scope}`, `nexus:{org_id}:a2a:grant:version`) and `_canon/03-event-taxonomy.md` (`agent-results` `kind` `context_fill`).

### 1.1 What landed, per repo

| Repo | PR | Branch |
|---|---|---|
| `mytheus-events` | [Mytheus-AI/mytheus-events#3](https://github.com/Mytheus-AI/mytheus-events/pull/3) | `feat/p10-context-fill` |
| `mytheus-backend` | [Mytheus-AI/mytheus-backend#12](https://github.com/Mytheus-AI/mytheus-backend/pull/12) | `feat/p10-a2a-collab` |
| `mytheus-intelligence` | [Mytheus-AI/mytheus-intelligence#7](https://github.com/Mytheus-AI/mytheus-intelligence/pull/7) | `feat/p10-context-gap-agent` |
| `mytheus-realtime` | [Mytheus-AI/mytheus-realtime#7](https://github.com/Mytheus-AI/mytheus-realtime/pull/7) | `feat/p10-agent-context` |
| `mytheus-ui` | [Mytheus-AI/mytheus-ui#32](https://github.com/Mytheus-AI/mytheus-ui/pull/32) | `feat/p10-context-fill` |
| `mytheus-platform` | [Mytheus-AI/mytheus-platform#8](https://github.com/Mytheus-AI/mytheus-platform/pull/8) | `feat/p10-agent-runtime-a2a` |
| `mytheus-docs` | this branch | `docs/p10-a2a` |

- **mytheus-events:** `AgentResult.kind` gains `context_fill`, appended last with enum default `note` (BACKWARD-safe); Python and TypeScript clients regenerated.
- **mytheus-backend** (plan 3.1, 3.2, 3.4)
  - `identity-tenant-service`: migration for `agent_authz_scopes` (RLS ENABLE + FORCE, `tenant_isolation` with USING and WITH CHECK); the PDP `authorize()` (no row is a deny, an explicit deny wins); gRPC `IdentityService.Authorize(org_id, grantee_role, query_class, target_scope) -> decision` added to `proto/identity.proto` and served by `identity-grpc` (a database failure returns UNAVAILABLE, which the runtime treats as deny and never caches); REST `GET`/`POST /api/v1/agent-authz-scopes`, `DELETE /api/v1/agent-authz-scopes/{id}`, `POST /api/v1/agent-authz-scopes/template` (owner and admin only, `a2a.grant.create` / `a2a.grant.revoke` audit rows). Every change deletes the tenant's `nexus:{org_id}:a2a:grant:*` decision keys, skipping the version key, then INCRs `nexus:{org_id}:a2a:grant:version`.
  - `core-api`: migration for `agent_collaborations` (columns per canon, no foreign key on `meeting_id` so a security entry with the nil meeting can be stored); the `agent-results` consumer branch for `ContextGapAgent` / `context_fill` (one row per collaboration entry, `ON CONFLICT (id) DO NOTHING`, the requester must equal the record's `user_id`, the fill only on the row with `id == fill.id`, the band re-derived from `agent_confidence_overrides`, one `audit_logs` row per new entry with `actor_type = 'agent'`: `a2a.request.allowed` / `denied` / `blocked`); `GET /api/v1/meetings/{id}/collaborations` (participants and admins; a `related_meeting` responder is named only to the requester and admins, others see "A colleague"; never the fill), `GET /api/v1/meetings/{id}/context-fills` (the caller's own surfaced, undismissed fills), `POST /api/v1/context-fills/{id}/dismiss` (requester only, 404 otherwise); internal `GET /internal/meetings/{id}/related-attendees?user_id=` behind `X-Mytheus-Service` / `X-Mytheus-Service-Token` / `X-Mytheus-Org` with the `agent-runtime-service` entry of `MYTHEUS_CORE_CALLER_TOKENS`; the api-gateway route for `/api/v1/context-fills`; OpenAPI re-emitted.
  - `workflow-service`: real `MeetingLifecycleWorkflow` with three activities, each over REST (none existed, so each got a small internal endpoint): `open_meeting_room` (meeting-service `POST /internal/meetings/{id}/room/open`, which also returns the scheduled start), `run_post_processing` (core-api `GET /internal/meetings/{id}/post-meeting`, read-only: checks the meeting ended and returns counts, because no summary builder exists yet), `close_meeting_room` (meeting-service `POST /internal/meetings/{id}/room/close`). A separate consumer, `python -m services.workflow_service.consumer`, starts the workflow on `meeting-events` `created` (id `meetinglifecycle-{meeting_id}`, REJECT_DUPLICATE) and signal-with-starts it on `ended`. The wait for `ended` is bounded at 7 days from the scheduled start; a run that times out fails with `MeetingNeverEnded`, so a later `ended` restarts it under ALLOW_DUPLICATE_FAILED_ONLY and post-processing still runs once. Activities authenticate as `workflow-service` in `MYTHEUS_CORE_CALLER_TOKENS` and the new `MYTHEUS_MEETING_CALLER_TOKENS`.
- **mytheus-intelligence** (plan 3.3): `ContextGapAgent` (`agents/context_gap_agent.py`) and `a2a/` (signed identity, PEP, decision cache, PDP client, requester and responder bus, related-attendees client); prompts `context_gap_gate@v1` and `context_gap_agent@v1`; the `Authorize` RPC added to the vendored `proto/identity.proto`. Per window: a deterministic detector, one shared Haiku gate, a deterministic `query_class`, at most two requesters (participants whose own retrieval lacks the context), then peer, related-meeting, RAG, silent inside `CONTEXT_GAP_DEADLINE_MS` (peer hop 60 percent of it). Responders answer from their own participant-scoped retrieval with no model call; the requester makes one Sonnet synthesis. Consumer groups: `agent-runtime-a2a` on `agent-requests`, `agent-runtime-resp-{hostname}` on `agent-responses` (latest). Env: `AGENT_IDENTITY_SECRET` (32+ chars, else A2A is off and the agent uses RAG only), `CONTEXT_GAP_A2A`, `CONTEXT_GAP_DEADLINE_MS`, `CONTEXT_GAP_GATE_THRESHOLD`, `CONTEXT_GAP_HAVE_SCORE`, `CONTEXT_GAP_MAX_REQUESTERS`, optional `CORE_URL` + `CORE_CALLER_TOKEN`.
- **mytheus-realtime:** `agentOutput.ts` routes a surfaced `context_fill` as pub/sub `agent-context` tagged with the requester; `server.ts` emits it only to that user's sockets in the meeting.
- **mytheus-ui:** `ContextFillCard` in the agent sidebar ("Context added from {name}'s agent", rationale, citation chips, Dismiss), a live-plus-rehydrate reducer (`src/lib/realtime/contextFills.ts`, `/api/meeting-context-fills`), the real collaboration log at `/meetings/[meetingId]/collaborations` (mock replaced; denials and blocks visibly distinct), and an "Agent collaboration" tab in the workspace admin console at `/tenant/agents` (Off, Same team, Department, Whole org, over the grant endpoints).
- **mytheus-platform:** `enable-tiers.sh` writes `AGENT_IDENTITY_SECRET`, `CORE_URL`, `CORE_CALLER_TOKEN` (and lists `agent-runtime-service` in `MYTHEUS_CORE_CALLER_TOKENS`) and a `workflow-service` token listed in both caller lists; `compose.prod.yml` adds `workflow-consumer` (profile `workers`) and pins `hostname: agent-runtime`.

### 1.2 Definition of Done audit (P10 DoD, plan section 10)

| DoD line | State | Evidence |
|---|---|---|
| `ContextGapAgent` + A2A PEP/PDP + deny-by-default `agent_authz_scopes` live | MET (deployed, not yet exercised live) | Deployed 2026-11-10 (1.7); `agent-runtime` has its A2A settings; `agent_authz_scopes` exists in production |
| `MeetingLifecycleWorkflow` runs real activities | MET (deployed, not yet observed on a real meeting) | Tested in `mytheus-backend`; `workflow` worker and `workflow-consumer` running in production |
| `agent_collaborations` persisted + log UI | MET (deployed, not yet seen with real rows) | Table in production; consumer, routes and page deployed |
| Demo: A's agent fills a gap from B's agent (peer hop) | NOT MET | Needs deploy and a live two-person call |
| Demo: a cross-tenant probe is DLQ'd + audited | PARTIAL | Real-broker test: a forged org lands on `agent-requests.dlq` with an `org_spoof` entry; core-api audits it as `a2a.request.blocked` |
| Demo: falls through to RAG on a deadline miss | PARTIAL | Agent tests cover deadline miss then related-meeting then RAG then silent |
| Demo: collaboration log shows grants + denials | PARTIAL | Route and page tests; not seen live |
| Gate: A2A p95 < 1.5 s | MET (offline) | Latency harness, faked RAG with median 450 ms, 240 round trips: p50 483 ms, p95 867 ms, p99 936 ms |
| Gate: RLS tests green | MET | Isolation tests for both new tables, plus the six CI tenant-isolation assertions |
| Gate: authz-in-PR green | MET | `scripts/check_authz.py` clean; backend CI `authz-check` pass |

### 1.3 Deviations from the plan, and why

1. **The agent identity is an HMAC-signed id, signed per request** (ADR-0035 D1), not an in-process binding: requester and responder can be different processes, and a per-request signature stops a copied id from being replayed.
2. **`grantee_role` is the RBAC tier** (D2): the domain role is free text.
3. **The PDP is a gRPC method** (D3) next to `get_user_context`, which the runtime already calls; a database failure is UNAVAILABLE, so an outage is never cached as a deny or an allow.
4. **A participant only becomes a requester when their own retrieval lacks the context**, at most two per window, with one shared gate call (D4): the literal plan would run the chain for everyone on every window and exceed `L4`.
5. **Responders answer without a model call** (D5): one retrieval per peer keeps the round-trip inside 1.5 s; the requester makes the single synthesis call.
6. **`agent_collaborations` holds the fill** (D7) instead of a new table; the log never exposes it.
7. **`query_class` bounds the returned passages by citation kind** (D9), found by the security review: before the fix a tenant denying `decision_lineage` could still receive decision passages through a query classified as `meeting_context`.
8. **Responder-side security records use hop `peer` and the nil meeting**: the request carries neither the hop nor the meeting.
9. **`run_post_processing` is read-only** (checks the meeting ended, returns counts): no summary builder exists to trigger yet.
10. **Display names in the log are emails**: `users` has no name column.
11. **Explicit deny wins through upsert**: ADR-0013's unique key means an allow and a deny cannot coexist for one grant, so setting `deny` replaces the allow.

### 1.4 Security review

A read-only review of the intelligence and backend PRs found nothing critical: identity is derived only from a verified signature (`hmac.compare_digest`, strict UUID parsing), every PDP, Redis or identity error denies and is never cached, both tables have FORCE and WITH CHECK, routes answer 404 across tenants, and internal tokens compare in constant time. Seven findings were fixed before merge: the `query_class` bound (1.3 item 7); per-request signatures with a 5 s future-deadline limit and 60 s `correlation_id` dedupe; a non-blocking, rate-limited (10/s) DLQ for bad signatures; the grant version key against a stale allow after revoke; the delivery-error buffer; the lifecycle wait counted from the scheduled start with a restart after timeout; the `related_meeting` responder hidden from other participants.

### 1.5 Verification (2026-11-10)

Each repo's CI mirrored locally before the push:

- `mytheus-events`: `make codegen`, `make validate`, `make test` (py and ts) pass. GitHub CI: compatibility, schemas, security pass.
- `mytheus-backend`, on throwaway `postgres:15` and `redis`: `MYTHEUS_REQUIRE_DB=1 uv run pytest -q` 789 passed, 1 skipped (744 on main); the six tenant-isolation assertions pass; `ruff`, `black --check`, `mypy`, `scripts/check_authz.py`, `scripts.emit_openapi --check`, Spectral (0 errors) and `oasdiff` (no breaking changes) clean; `alembic upgrade head`, `downgrade -1` twice, `upgrade head`; `bootstrap_roles` passes. GitHub CI: lint, test, migrations, authz-check, openapi, security, docker pass.
- `mytheus-intelligence`: `uv run pytest -q` 417 passed, 113 skipped with a throwaway Redpanda and Schema Registry (416 / 114 without), including a real-broker round trip and a forged org landing on the DLQ; `ruff`, `black --check`, `mypy` clean; both images build. Live gate eval on Haiku 5.5: gap recall 1.0, false-gap rate 0.111 (one case: a reference explained in the same window), query hit rate 1.0, 16 calls, 9,893 tokens, repeated once with the same result; recorded as the baseline.
- `mytheus-realtime`: `npx vitest run` 66 passed (60 on main) against a throwaway Redis and Redpanda with the new schema registered; `tsc`, eslint, `docker build` clean.
- `mytheus-ui`: `npm test` 57 passed (42 on main); `tsc --noEmit`, `next lint`, a Node 20 `docker build` (`next build`) and `generate:api:check` pass.
- `mytheus-platform`: `bash -n scripts/enable-tiers.sh`; `docker compose -f deploy/compose.prod.yml config -q`.
- Cross-repo: the hop and result vocabularies, the `payload_json` fields, the REST response shapes, the core-api internal headers and the gateway routes were compared between the repos and match.

### 1.6 Open gaps and handover

1. **Merged and deployed** (1.7).
2. **A2A is off for every tenant until an admin grants**: open `/tenant/agents` and choose a scope.
3. **The live demo** (a two-person call, a gap filled from a peer, the log) is not done.
4. **The A2A p95 is measured offline** with a faked retrieval delay; a live measurement under `E7` load belongs to P13 load testing.
5. **`run_post_processing` triggers nothing yet**: the summary builder it should call does not exist.
6. **Mobile** is out of scope for this phase.

### 1.7 Merge and deploy (2026-11-10)

Merged in order with each main build and deploy watched to completion before the next merge, so no repo's deploy pulled another's stale image (the P9 race, `07` section 11.5):

- `mytheus-events` #3, then the production Schema Registry re-registered by the maintainer through an SSH tunnel (`make register`): `agent-results-value` is id 23, the 20 other subjects unchanged, `register OK: 21 subjects under BACKWARD`.
- `mytheus-backend` #12. Its first main build failed one test, `test_the_run_survives_a_worker_restart`: a race in the test, not the workflow. The first worker could stop after the room row committed but before the activity's completion reached Temporal, so the restarted worker re-ran the idempotent open and correctly reported `opened: false`. Fixed in #13 (the test waits until no activity is pending; 11 passed three runs in a row locally), main run 42766147943 green, deploy `healthz OK` at 06:08 UTC.
- `mytheus-intelligence` #7: main run 42766598119 built both images and, for the first time, deployed itself through its own `deploy` job with the new forced-command key (`07` section 12.5 item 5), `healthz OK` at 06:11 UTC.
- `mytheus-realtime` #7: run 42770101305, `healthz OK` at 06:41 UTC.
- `mytheus-ui` #32 (with the generated API types re-run against the merged backend): run 42770303200, quality, build, deploy success.
- `mytheus-platform` #8: deploy run 42770701388, `healthz OK` at 06:46 UTC. Then `scripts/enable-tiers.sh` on the server, dry run first: it wrote `MYTHEUS_WORKFLOW_SERVICE_TOKEN`, `AGENT_IDENTITY_SECRET`, `CORE_URL` and `CORE_CALLER_TOKEN` and brought up all 37 services, `workflow-consumer` among them.
- `mytheus-docs` #8.

Checked on the server afterwards: `agent-runtime` has `AGENT_IDENTITY_SECRET`, `CORE_URL` and `CORE_CALLER_TOKEN` set (`CONTEXT_GAP_A2A` unset, which defaults to on); the `workflow` worker logged `Worker started on task queue: mytheus-backend`; no errors in `agent-runtime`, `workflow`, `workflow-consumer`, `identity-grpc` or `core-api-consumer`; `agent_collaborations` and `agent_authz_scopes` exist in the `mytheus` database and `alembic_version` is `c8f2d3e4a510`, the backend head.

**Found while deploying, outside P10: the Ask page was answering "Not available".** `rag-service` (P9) verifies the asker's Clerk JWT itself and reads `CLERK_JWKS_URL` and `CLERK_ISSUER`, but those were only in `prod.env`, which it does not load, so every `POST /api/rag/query` raised `KeyError: CLERK_JWKS_URL` and the gateway returned 500. Fixed on the server by copying both into `prod.rag.env` and recreating `rag-service`; made permanent in `enable-tiers.sh` (`mytheus-platform` #8, which now also checks `CLERK_ISSUER` is staged). After the fix the gateway and `rag-service` both logged `POST /api/rag/query 200`.

---

## 2. Phase P11: Proactive Intelligence and Real-time Polish (2026-11-10)

**Status:** P11 DONE WITH GAPS: plan sections 3.6 to 3.9 built, tested per repo with each CI mirrored locally, security-reviewed, fixed after review, merged and deployed. The dependency alert cannot fire yet (2.5 item 1) and nothing has been seen in a live meeting.

Build decisions: [ADR-0036](../adr/0036-plan-11-proactive-agent-and-delivery-build.md) (D1 to D11). Canon first: `_canon/02-data-model.md` (`notification_log`, `notification_preferences`) and `_canon/03-event-taxonomy.md` (`agent-runtime-service` consumes `decisions` and `actions`).

### 2.1 What landed, per repo

| Repo | PR | Branch |
|---|---|---|
| `mytheus-backend` | [Mytheus-AI/mytheus-backend#15](https://github.com/Mytheus-AI/mytheus-backend/pull/15) | `feat/p11-notifications` |
| `mytheus-intelligence` | [Mytheus-AI/mytheus-intelligence#8](https://github.com/Mytheus-AI/mytheus-intelligence/pull/8) | `feat/p11-proactive-agent` |
| `mytheus-realtime` | [Mytheus-AI/mytheus-realtime#8](https://github.com/Mytheus-AI/mytheus-realtime/pull/8) | `feat/p11-priority-queue` |
| `mytheus-ui` | [Mytheus-AI/mytheus-ui#34](https://github.com/Mytheus-AI/mytheus-ui/pull/34) | `feat/p11-alerts` |
| `mytheus-platform` | [Mytheus-AI/mytheus-platform#9](https://github.com/Mytheus-AI/mytheus-platform/pull/9) | `feat/p11-proactive-deploy` |

No `mytheus-events` change: `AgentResult.kind` already had `alert`, and `Notification` and `NotificationDeliveryEvent` already existed.

- **mytheus-intelligence:** `ProactiveAgent` (`agents/proactive_agent.py`) with a new `decisions`/`actions` consumer loop (group `agent-runtime-proactive`, latest). Conflict: candidates from `knowledge-graph-service` lineage over gRPC (`libs/shared/kg_client.py`, `KG_GRPC_ADDR`, `KG_CALLER_TOKEN`, 800 ms timeout) and a decision-only `rag-service` search (a caller-forced `source` filter that only narrows), judged by one Sonnet call `proactive_conflict@v1`; `supersedes` is never a conflict; with the graph down only the rag path runs. Workload and dependency: deterministic from the `core-api` action context. Missing stakeholder: per window, a Haiku gate `proactive_stakeholder@v1`, then people who attended the meetings behind the topic's decisions, once per (meeting, person) and only after a surfaced alert. One alert per recipient with deterministic ids; per recipient, detail from another meeting only if they attended it (redacted otherwise); URLs stripped from title, body and rationale. Suspended when the budget is `throttled`/`blocked` or the plan lacks `proactive_agent` (override `PROACTIVE_AGENT_ALL_PLANS`). The KG gRPC server binds `KG_GRPC_BIND` (50104).
- **mytheus-backend:** `notification-service` migration (`notification_log`, `notification_preferences`, RLS FORCE); the real consumer (`python -m services.notification_service.consumer`, group `notification-service`) for surfaced `ProactiveAgent` alerts and `notifications` events: one `notification_log` row per item, delivery per preferences (in-app always; email through SMTP when `SMTP_URL` is set and the user opted in, sent after the commit; Slack recorded as skipped until P12), one `notification-delivery-events` receipt per attempt through the outbox (a skip is `failed` with detail `skipped:...`, because the receipt schema has no skipped status). Routes `GET /notifications`, `PATCH /notifications/{id}` (owner only, 404 otherwise, audited), `GET`/`PUT /notifications/preferences`. `core-api` internal `GET /internal/actions/{id}/context` and `GET /internal/meetings/{id}/participants`; `billing-metering-service` `GET /internal/plan-features` (caller `agent-runtime-service` only).
- **mytheus-realtime:** `alertQueue.ts`: the per-user `nexus:{org_id}:queue:agent:{user_id}` sorted set (priority then age), at most 100 items and nothing older than 24 h, drained only into the alert's own meeting room as `agent-alert` to the recipient's sockets, cross-pod through a content-free nudge on the meeting channel. Backpressure sheds `agent-stream` deltas first.
- **mytheus-ui:** `AlertCard` and the red conflict banner pinned above every card, yellow and blue cards, Mark read and Dismiss through `PATCH /notifications/{id}`, the alerts reducer with rehydrate from `/api/meeting-alerts`, link allowlist, streamed draft cards with reserved heights and reduced motion, and notification preferences on Settings, Profile (email off by default).
- **mytheus-platform:** `notification-consumer` (profile `workers`), `knowledge-graph-grpc` (profile `kg`), `kg` in the default profiles of `enable-tiers.sh`, and `KG_GRPC_ADDR`, `KG_CALLER_TOKEN` (listed in `MYTHEUS_KG_CALLER_TOKENS`) and `PROACTIVE_AGENT_ALL_PLANS=1` for `agent-runtime`.

### 2.2 Definition of Done audit (P11 DoD, plan section 10)

| DoD line | State | Evidence |
|---|---|---|
| `ProactiveAgent` (conflict/workload/stakeholder/dependency) live over the graph + `decisions__{org_id}` complement | PARTIAL | Deployed; conflict, workload and stakeholder tested; dependency cannot fire (2.5 item 1) |
| `realtime-gateway` Redis pub/sub backbone + `agent-stream` deltas + per-user priority queue | MET | 80 tests including a two-instance cross-pod test (p50 1.6 ms, p95 2.8 to 13 ms, assert under 50 ms) |
| `notification-service` three-channel delivery | PARTIAL | In-app delivers; email wired, off until SMTP is configured and a user opts in; Slack waits for P12 |
| Alert UI + sidebar polish | MET (not browser-checked) | 77 UI tests; build passes |
| Demo: a confirmed decision raises a red conflict banner | NOT MET | Needs a live meeting |
| Demo: still fires via the Qdrant complement with `knowledge-graph-service` killed | MET (tested) | Agent test with the graph down |
| Demo: a workload alert | MET (tested) | Agent and core-api route tests |
| Demo: streamed deltas render progressively cross-pod | MET (tested) | Gateway cross-pod and UI draft tests |
| Demo: urgent alerts pre-empt notes | MET (tested) | Priority-order test: a priority-1 alert queued after priority-3 items is delivered first |

### 2.3 Deviations from the plan, and why

1. **Alerts are `notification_log` rows read and dismissed through `notification-service`**, not `core-api` (ADR-0036 D5): canon already gives that service the log and the `/notifications` surface.
2. **`ProactiveAgent` consumes `decisions` and `actions`** (D1): the checks are triggered by confirmations, which the transcript windows do not carry.
3. **Workload and dependency make no model call** (D3); the conflict judge is Sonnet, not Opus.
4. **The plan gate has a deployment override, set in production** (D8): there are no paying tenants yet.
5. **Email is opt-in** (D10) and is sent after the commit, so a crash between commit and send loses that email rather than duplicating it.
6. **Alerts are delivered only in their own meeting** (D11), so the meeting-scoped rehydrate always finds them.
7. **Missing-stakeholder ids are keyed by `{meeting_id}:{person_id}`**, so two absent people do not collide.

### 2.4 Review

A read-only review of the four code PRs found RLS, owner-only routes, caller tokens, header-derived org, tenant-scoped dedupe and fail-closed gates correct, and five issues, all fixed before merge: alert text revealing decisions and attendees from meetings a recipient never attended (now redacted per recipient, D9); a silent stakeholder window blocking later alerts; alerts about another meeting lost after a reload (D11); an uncapped per-user queue; URLs from meeting text reaching alert emails (stripped) and an open relative-link pass-through in the UI (allowlisted). Email was made opt-in at the same time (D10).

### 2.5 Open gaps and handover

1. **The dependency alert cannot fire**: no table records that one action depends on another (P8 never stored the candidate's `depends_on`), so the context route returns `depends_on: []`. Needs a column on `action_items` set at confirm.
2. **Stakeholder alerts stay silent until `RELATES_TO` edges exist** for the topic's decisions.
3. **The rag decision arm is participant-scoped**, so the conflict search sees only decisions from meetings the confirming user attended; the graph lineage covers the rest when `kg` runs.
4. **Email needs `SMTP_URL` and `SMTP_FROM`** on the server, and Slack needs P12.
5. **Live demo** of the conflict banner in a real meeting is not done.

### 2.6 Verification (2026-11-10)

- `mytheus-backend`: on throwaway `postgres:15` and `redis`: pytest 818 passed, 1 skipped (25 new); the six isolation assertions; `ruff`, `black --check`, `mypy`, `check_authz`, `emit_openapi --check`, Spectral 0 errors, `oasdiff` no breaking changes; migrations up, down, up. GitHub CI green; deploy `healthz OK` 07:58 UTC.
- `mytheus-intelligence`: `uv run pytest -q` 462 passed, 117 skipped; Avro round trip and real-Redpanda consumer tests pass against a throwaway broker; `ruff`, `black`, `mypy` clean; both images build. Live conflict-judge eval, Sonnet 5.5, 12 golden pairs: precision 1.0, recall 1.0, false-positive rate 0.0, 9,211 tokens; baseline recorded. Deploy `healthz OK` 08:01 UTC.
- `mytheus-realtime`: `npx vitest run` 80 passed; `tsc`, eslint, prettier, `docker build`. Shed test with the client's TCP paused: all 80 captions, the suggestion and the alert arrived; 68 of 80 deltas were shed. Deploy `healthz OK` 08:02 UTC.
- `mytheus-ui`: `npm test` 77 passed; `tsc`, `next lint`, `next build`; merged with the mock-data PR (#33) and the generated types re-run against the merged backend.
- Cross-repo: the alert fields emitted by the agent and read by `notification-service` and the gateway were compared and match.

---

## 3. Phase P12: External Integrations, follow-ups and the admin consoles (2026-11-10)

**Status:** P12 built, tested per repo with each CI mirrored locally, security-reviewed and fixed after review; in PRs awaiting merge. Live connections need each provider's app registered (3.5 item 1).

Build decisions: [ADR-0037](../adr/0037-plan-12-integrations-follow-ups-and-consoles-build.md) (D1 to D8). Canon first: `_canon/02-data-model.md` (`integration_id_map`, `integration_cursors`, `integration_webhook_inbox`, `feature_flags`, `ops_audit_log`, `support_grants`, `meetings.calendar_event_id`).

### 3.1 What landed, per repo

| Repo | PR | Branch |
|---|---|---|
| `mytheus-backend` | [Mytheus-AI/mytheus-backend#17](https://github.com/Mytheus-AI/mytheus-backend/pull/17) | `feat/p12-integrations` |
| `mytheus-backend` | [Mytheus-AI/mytheus-backend#16](https://github.com/Mytheus-AI/mytheus-backend/pull/16) | `feat/p12-followups-admin-ops` |
| `mytheus-ui` | [Mytheus-AI/mytheus-ui#35](https://github.com/Mytheus-AI/mytheus-ui/pull/35) | `feat/p12-integrations-consoles` |
| `mytheus-platform` | [Mytheus-AI/mytheus-platform#10](https://github.com/Mytheus-AI/mytheus-platform/pull/10) | `feat/p12-integrations-deploy` |

No `mytheus-events` change: `Action.status` stays confirmed-only on purpose (ADR-0037 D2).

- **`integration-service` (#17):** migration `7c3e9a1f5b20` (`integration_connections`, the id map, cursors, webhook inbox; one active connection per provider account across all tenants); OAuth with signed state and PKCE, the authorize redirect returning to the dashboard page `/settings/integrations/callback/{provider}`, which completes it through `POST /oauth/{provider}` under the user's own session (the user and organization must match the state); tokens envelope-encrypted with `libs/shared/crypto`, never returned or logged; `GET /integrations`, connect, disconnect; `/webhooks/{provider}` with per-provider verification, a deduped inbox and fast ACK, drained by `python -m services.integration_service.worker`; Google Calendar (watch, delta sync, tenant-organized events only, the organizer's attachments to `attachment_to_ingest`), Slack (bot install, user map, internal token and message-building routes), Jira (`python -m services.integration_service.consumer` on confirmed `actions`: one issue per action with a `nexus:{id}` label, idempotent; inbound status re-read with the tenant's token, then `core-api` `POST /internal/actions/{id}/external-status`, echo-suppressed). `core-api`: the external-status route, the `calendar_event_upserted` consumer (idempotent on `meetings.calendar_event_id`), `GET /internal/actions/{id}/status`. `notification-service`: real Slack DMs with a short-lived fetched token. api-gateway: every path is decoded and rejected on dot segments, encoded or doubled slashes, or a leading `/internal`, before routing; `/webhooks/` is the only new public prefix.
- **`workflow-service` and `admin-ops-service` (#16):** migration `e2b6c4a8f913` (on `7c3e9a1f5b20`): `feature_flags` (tenant rows read-only to tenants), `ops_audit_log`, `support_grants`. `ActionFollowUpWorkflow` started from confirmed `actions` (`actionfollowup-{id}`), reminders on day 1, 3 and 7 and escalation on day 10 (owner or admin in the assignee's department, then the organizer, then an org admin), each through workflow-service's own outbox as in-app and Slack `notifications`; status re-read each step. `MeetingLifecycleWorkflow` sends a Slack prep brief at T-15 minutes and a summary on end. Tenant-admin `/admin/tenant/*` (members, settings, support grants, data export recorded as pending); internal-ops `/admin/ops/*` for users in `MYTHEUS_STAFF_USER_IDS` (tenants with suspend and reinstate through identity's caller-token route, feature flags, costs from aggregates, audit log only under a live support grant), every call in `ops_audit_log`.
- **`mytheus-ui` (#35):** `/settings/integrations` (Calendar, Slack, the PM radio with Jira and Linear unavailable, Not configured states), the callback page, tenant console screens (members, settings, support access, data export), and the ops console (`/ops`, flags, costs, audit) gated by `GET /admin/ops/whoami`.
- **`mytheus-platform` (#10):** `integration-worker` and `integration-consumer`; `enable-tiers.sh` writes `APP_BASE_URL`, `OAUTH_STATE_SECRET`, `INTEGRATION_SERVICE_URL`, `CORE_API_URL` and one caller secret each for notification-service, integration-service and admin-ops-service.

### 3.2 Definition of Done audit (P12 DoD, plan section 10)

| DoD line | State | Evidence |
|---|---|---|
| `integration-service` framework + Calendar + Slack + live Jira | PARTIAL | Built and tested against fakes; live needs the provider apps (3.5 item 1) |
| `ActionFollowUpWorkflow` | MET (tested) | Temporal time-skipping tests: reminders 1/3/7, escalation 10, early completion, worker restart |
| Both `admin-ops-service` consoles | MET (tested) | API and UI tests; staff gate, audit, support grants |
| Demo: admin OAuth-connects Jira, a confirmed action becomes a real issue, Done flips the action | NOT MET | Needs a registered Atlassian app |
| Demo: day-1 follow-up Slack DM | NOT MET | Needs a registered Slack app |
| Demo: a Calendar event creates a meeting and ingests its attachment | NOT MET | Needs a registered Google app (Drive scope needs Google verification) |
| Demo: ops cost dashboard and tenant suspend | MET (tested) | Ops route tests |
| Gate: R8 cap enforced | MET | Second PM connection 409; Linear unavailable |
| Gate: webhook idempotency and loop suppression | MET (tested) | Replay dedupe and echo suppression tests |

### 3.3 Deviations from the plan, and why

1. **Jira status reaches `action_items` through a `core-api` internal route**, not `actions` (D2); NEXUS to Jira status sync is not built.
2. **The OAuth callback runs through the signed-in dashboard** (D4), after the review showed a public callback let a connect link connect someone else's account into the attacker's workspace.
3. **Staff are an allowlist** (D5) and audit-log reads need a live support grant (D8).
4. **Slack messages are title, text and link** turned into Block Kit by `notification-service`: the `Notification` schema has no blocks field. Prep briefs do not query `rag-service` yet.
5. **Jira issues go to the first project the installing account sees, as Task.**
6. **Reinstate emits no event**: neither frozen schema has a reinstated value. **A data export is recorded as pending**: the RTBF workflow is still a stub.

### 3.4 Review

A read-only review found ten issues, all addressed before merge: gateway dot-segment traversal reaching `/internal/*` (also present on the live gateway for signed-in users; fixed for every path); the OAuth state not bound to the user; one provider account connectable by many tenants; Jira webhook claims, payload status and label mapping trusted; calendar invites from outside the tenant ingested; data export emitting the tenant-erasure event; support grants not enforced on audit reads; two migration heads (re-chained); tenant-writable feature flags. One is mitigated, not removed: `set_organization_status` stays executable by the shared `mytheus_app` role, because no identity-only role exists; a test pins its single grantee and call site.

### 3.5 Open gaps and handover

1. **Register the provider apps** and set their credentials in `prod.env`: Google (`GOOGLE_OAUTH_CLIENT_ID`/`SECRET`), Slack (`SLACK_CLIENT_ID`/`SECRET`, `SLACK_SIGNING_SECRET`, Events URL `https://api.mytheus.cloud/webhooks/slack`), Atlassian (`JIRA_CLIENT_ID`/`SECRET`), each with the redirect URI `https://dashboard.mytheus.cloud/settings/integrations/callback/{provider}`.
2. **Jira webhook JWT claim** for the cloud id is unconfirmed; check against a real delivery.
3. **`MYTHEUS_STAFF_USER_IDS`** must be set by hand for the ops console.
4. **No identity-only database role** (3.4).
5. **NEXUS to Jira status sync**, RTBF-backed data export, Slack prep briefs grounded in RAG, and a re-auth notification on a degraded connection are not built.

### 3.6 Verification (2026-11-10)

- `mytheus-backend` #17: on throwaway `postgres:15` and `redis:7`: pytest 926 passed, 1 skipped (98 integration tests); the six isolation assertions; `ruff`, `black`, `mypy`, `check_authz`, `emit_openapi --check`, migrations up, down, up, `bootstrap_roles`. GitHub CI green.
- `mytheus-backend` #16: pytest 853 passed, 1 skipped locally with #17's migration copied in (the 3 failures were #17's schema against pre-#17 code and a known timing-dependent dashboard test); all other gates clean. GitHub CI fails until #17 merges (its migration's parent is #17's).
- `mytheus-ui` #35: `npm test` 105 passed; `tsc`, `next lint`, `next build` clean. GitHub CI green.
- `mytheus-platform` #10: `bash -n`, `docker compose config -q`. GitHub CI green.
