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
