# Shaun - API Surface, Event Backbone, Safe UI Fixes

**18 tasks.** Start with D1 - it is a live cross-tenant hole and it is small.

Audit: https://claude.ai/code/artifact/da281c5f-d990-40b4-9487-d9f857833b08
Evidence: [`audit/`](audit/) · [`verify/verify-backend.md`](verify/verify-backend.md) · [`verify/verify-ui.md`](verify/verify-ui.md)

---

## Ordering

**Write now, verify after Sushan's S1 + S2.** Every request 500s then 403s until those land. D1 and
D2 are pure backend with tests - start there.

**Most UI work is frozen.** The scope of `mytheus-ui` is unresolved (see README). Only D16 and D17
are assigned, because they are correct under either outcome. Do not wire `/meetings`, `/org`, the
dashboard arms or the generated types until Sushan decides.

---

## D1 - Cross-tenant speaker assignment (HIGH, start here)

`core_api/routes/speakers.py:97-133` accepts a caller-supplied `user_id` and relies solely on a
foreign key for tenancy. Its comment claims the FK blocks another tenant's `user_id`.

**Two sibling routes document that as false and each run an explicit probe:**
`routes/meetings.py:186-192` and `routes/org_hierarchy.py:172` both run
`SELECT 1 FROM users` precisely because the FK does not enforce tenancy.

`create_meeting` got the fix. `assign_speaker` never did. No test covers it. Flagged independently
by two auditors.

- [ ] Add the RLS-scoped `SELECT 1 FROM users` probe before the write
- [ ] Correct the misleading comment
- [ ] Test: tenant A assigning a tenant B `user_id` returns a clean 404 - match the 404-before-403
    ordering used at `routes/users.py:80-91`
- [ ] Grep for a fourth route trusting an FK for tenancy

---

## D2 - Transcript read path

`core_api/routes/transcript.py`:

- `:123` missing `deleted_at IS NULL` - serves soft-deleted chunks
- `:134` no `created_at` bound, so monthly partition pruning is forfeited and every query scans all
partitions
- **Zero tests**, despite being the one endpoint the live UI depends on

- [ ] Add the filter and the bound
- [ ] Tests: pagination, `from_ms` seek, tenant scoping, soft-delete exclusion

---

## D3 - Eleven gateway prefixes route to nothing

Bigger than first reported. `edge.py:146-180` routes all of these to services that 404:
`/decisions`, `/actions`, `/worklogs`, `/suggestions`, `/billing`, `/usage`, `/plans`,
`/notifications`, `/admin/tenant`, `/admin/ops`, `/oauth`, `/webhooks`, plus `/orgs` and `/rbac`.

Worse: **the tests assert this routing as correct** (`test_gateway_edge.py:270`,
`test_gateway_proxy.py:175-176`), so the suite is green on a route map to nowhere. And nothing
checks that a ROUTE_MAP prefix has a route behind it.

- [ ] Decide per prefix: remove until real, or implement. Removal is the honest default.
- [ ] **CI check that every ROUTE_MAP prefix resolves to a mounted route.** This is the durable fix.
- [ ] Update the tests so they stop asserting dead routing
- [ ] Coordinate with Samar's P17 (`/stripe/webhook`) and Sushan's S5 (`/orgs`, `/rbac` become real)

---

## D4 - `integration-service`: unblock a finished consumer

The most valuable of the five scaffolds. `mytheus-intelligence/services/ingestion_service/consumer.py:75`
is a **complete, tested** consumer of `integration-events` - S3 fetch, RLS-scoped pipeline, inbox
dedupe, DLQ, a 260-line test suite - and `integration-service` is its **sole designated producer**
with zero producer code. Calendar-attachment ingestion is dead in production and CI cannot see it,
because the tests inject synthetic messages.

Scope this as **minimum viable**: enough to make the built consumer fire. Not the full OAuth
connector band.

- [ ] Minimum producer emitting `integration-events` with `(google_calendar, attachment_to_ingest)`
- [ ] Match the Avro contract exactly - `{provider, event_type, external_id, payload_json}`
- [ ] Decide with Sushan whether the OAuth half is in scope now or stays a scope conversation

---

## D5 - Audit log has no read side
**Correction to the first-pass claim.** `libs/shared/audit.py` **does** have 8 live writers -
`authz.py:176`, core_api meetings and org_hierarchy, and four in identity. The trail is being
written.

What is missing is the **read/export side**: `admin_ops_service` has zero routes, so nothing can
retrieve audit rows. The table itself is well built - partitioned through 2027-06 plus a DEFAULT,
FORCE RLS, UPDATE/DELETE revoked per partition.

- [ ] Read/export route, `require_tier("admin")`
- [ ] Audit which mutations still lack a write call and add them
- [ ] Sushan's S5 will call your helper for grant changes - coordinate

---

## D6 - Nothing writes `assignment='suggested'`
**Correction:** the speakers list is **not** always empty - a manual PATCH upserts rows
(`speakers.py:102`). The missing half is specifically **`suggested` + `confidence`**.

`speakers.py` is the only writer of `meeting_speakers` and always writes `'manual'`.
`transcription-service` explicitly never writes it (`speakers.py:5-6`). So the diarized advisory
list that `SpeakerMapping.tsx` is built to render - greyed, with confidence, manual wins - never
appears.

- [ ] Write `suggested` rows from the diarized labels the transcription path already produces
- [ ] Preserve the `meeting_speakers_manual_is_authoritative` trigger semantics (`25e7cede5672:56`)

---

## D7 - `meetings.status` never reconciles with `meeting_rooms.status`
**Correction:** `'live'` **can** be set via PATCH (`meetings.py:386`, covered by
`tests/test_meetings_api.py:374`). What is missing is **automatic** reconciliation.

`participant_joined` sets `meeting_rooms.status='active'` (`webhook.py:230-235`) and never touches
`meetings.status`. So a meeting with people in the room still reads `'scheduled'`.

- [ ] Reconcile from the webhook path
- [ ] Decide which of the two state machines is authoritative and write it down - they are separate
    by construction (`866f7eedb0e7:65` vs `b7d9f3a55c21:204`)

---

## D8 - Chat history is unrecoverable

`POST /chat` is the only `chat_messages` route. There is **no `GET /chat`** anywhere - not in
core_api, not in bff. A client that misses the socket relay cannot recover history, unlike
transcript, which has a re-hydrate endpoint for exactly this reason (`transcript.py:6-10`).
Depends on Samar's P2 - chat inserts currently fail outright.

- [ ] `GET /chat` with cursor pagination, mirroring the transcript read path

---

## D9 - Soft delete is unreachable

`deleted_at` is read in every existence check (`routes.py:113`, `meetings.py:132`, `speakers.py:39`,
`transcript.py:87`) and **written nowhere**. There is no `DELETE /meetings/{id}` and no cancel
route, so a meeting can never be removed through the API.

- [ ] `DELETE` / cancel route writing `deleted_at`, with the audit write and the outbox event
- [ ] Confirm no read path forgets the filter (D2 fixes one)

---

## D10 - The attendee list is immutable after creation

No participant add/remove route exists. Participants can only be set at meeting creation.

- [ ] Add / remove participant routes
- [ ] Reuse the `SELECT 1 FROM users` RLS probe from `meetings.py:193-201` - same tenancy trap as D1

---

## D11 - LiveKit room naming and the fake-sync trap

`meeting_service/rooms.py:51-96` names rooms `meeting-{meeting_id}` with no tenant segment.

**Low severity by itself** - webhook tenant resolution reads org from room *metadata* and fails
closed six ways, and meeting IDs are UUIDs. The reason to fix it is that it is the one identifier in
the meeting path that is not tenant-qualified.

**The latent trap matters more:** rooms "synced" via `FakeTransport` are marked
`livekit_metadata_synced_at` **permanently**, so at real-LiveKit cutover they never re-sync and
webhook tenant resolution breaks for every pre-existing room.

- [ ] Add an org segment to the room name
- [ ] Fix the fake-sync marking and migrate existing rows - one migration, both changes
- [ ] Keep `ON CONFLICT (org_id, meeting_id)` semantics

---

## D12 - Nothing reads any DLQ, and the DLQ contract is inconsistent

- **No DLQ reader exists anywhere.** Failed messages accumulate unseen.
- `mytheus-intelligence` writes `{topic}.{group}.dlq`; `create_topics.sh` only creates `{topic}.dlq`
- DLQ payloads are **JSON** while `contracts/DlqEnvelope.avsc` specifies **Avro**

- [ ] Reconcile the topic naming - coordinate with Samar's P3
- [ ] Reconcile the encoding: either honour the Avro contract or change the contract
- [ ] A minimal DLQ inspection path, even a script

---

## D13 - Import the generated Avro clients

`mytheus-events` generates Python and TypeScript clients from the 21 schemas. **Adoption is zero** -
the only reference anywhere is a test asserting the package is *absent*. Producers hand-roll dicts.

No payload drift exists today because both Python tiers encode through the live Schema Registry
with `auto.register.schemas: false`, which fails at encode time. That safeguard is compensating for
the hand-rolling.

**The packaging blocker cited in code as the reason was already fixed** in `pyproject.toml`. No
obstacle remains.

- [ ] Import the Python client in backend and intelligence producers/consumers
- [ ] Import the TS client in `realtime-gateway/src/presenceEventsProducer.ts`
- [ ] Delete the stale "package is absent" test

---

## D14 - Delete the `user-sync-events` topic

ADR-0010 D4 decided there is **no per-user propagation topic**, and `routes/auth.py:19-21` says so
explicitly - the `TOPIC` constant is used only as a Postgres/Redis dedupe namespace. But the topic,
its subject and its `.dlq` are still created and will always be empty.

- [ ] Remove from `codegen/_schemas.py` `TOPICS`, `subjects/`, and `create_topics.sh`
- [ ] Rename the constant so it stops reading as a topic name
- [ ] Update the expected count in `create_topics.sh` and Samar's P3 gate

---

## D15 - BFF imports another service in-process

`bff/people.py:43` imports identity-tenant-service **in-process, across a container boundary**. Its
justification ("one deployable") has expired - compose runs them separately, so **two processes now
write the same Redis key**.

- [ ] Replace with a real call through the proper boundary, or move the logic
- [ ] Check for other in-process cross-service imports

---

## D16 - The mock `notFound()` bug - in three routes, not one *safe under either UI outcome*

`mockMeetings.find(...)` -> `notFound()` runs **before** the real fetch in all three:
`transcript/page.tsx:56`, `room/page.tsx:22`, `[meetingId]/page.tsx:40`.

**Every real backend UUID 404s.** The one page with a genuine backend read gates its own existence
on a mock array.

Also here - **the dashboard renders two clocks**: `dashboard/page.tsx:47` uses the real date while
`StatGrid`/`TodaysMeetings` use `MOCK_NOW = 2026-07-01` (`format.ts:9`). Today it reads
"September 7 · 0 meetings today" above a July stat grid. Visible to anyone who opens the app now.
*`mytheus-ui` is sensitive - `git fetch` first, no build, minimal diff.*

- [ ] Remove the mock existence gate in all three routes; let the real fetch determine 404
- [ ] Fix `MOCK_NOW` so the dashboard shows one consistent date
- [ ] Add a loading state to the transcript page - it has none

---

## D17 - Realtime auth and connection gates *safe under either UI outcome*

- `MeetingRoom.tsx:45-53` fetches the Clerk token **once on mount** and never refreshes. Socket.io
and LiveKit both auto-reconnect and re-handshake with a stale JWT.
- `useLiveKitRoom.ts` **never re-mints on reconnect** despite its own docstring at `:6-11` promising
it.
- `useLiveKitRoom.ts:85` gates connection on `NEXT_PUBLIC_LIVEKIT_URL` - a value the **server always
supplies** in the token response, so the gate blocks on config that is not needed.
- `useLiveKitRoom.ts:46-50` hand-writes a **second** `TokenResponse` duplicate.

Mobile already solves the token problem correctly - `src/api/provider.tsx` pulls a fresh token per
request. Follow that pattern.
Cannot be verified until Samar's P11 makes the env vars work in a container.

- [ ] Refresh the token on reconnect; re-mint on LiveKit reconnect
- [ ] Drop the `NEXT_PUBLIC_LIVEKIT_URL` connection gate; prefer the server-supplied value
- [ ] Collapse both hand-written `TokenResponse` duplicates onto the generated type

---

## D18 - Reconcile the rebuild branch

`origin/local-mytheus-rebuild` is **6 ahead / 11 behind** main, last touched 2026-08-13, and
overlaps `mock/data.ts`, `MeetingRoom.tsx`, `.env.example` and 8 marketing files.

**Its merge would clobber `.env.example`** - that file did not exist at the merge-base, and main
added the whole Clerk + gateway contract to it.

The branch also carries the `mytheus-web` scope claim (commit `0455cc8`) that is currently
unresolved - see the README. **Reconcile the branch mechanically; do not resolve the scope question
yourself.**

- [ ] Rebase onto main, resolving the `.env.example` conflict in main's favour
- [ ] Inventory exactly what the 6 commits change so Sushan can decide the scope question with facts
- [ ] Do not merge until the scope question is settled

---

## Frozen pending Sushan's scope decision - do not start

Wiring `/meetings` -> `GET /api/v1/meetings` · `/org` -> `GET /bff/org/chart` · the three hardcoded
`[]` dashboard arms · importing `generated/identity-tenant-service.ts` and
`generated/meeting-service.ts` (zero importers today) · UI test infrastructure (there is **none** -
no `error.tsx` anywhere, only `(app)/loading.tsx` and root `not-found.tsx` exist repo-wide, while
`transcript.ts:81-85` deliberately rethrows `ApiError` with nothing to catch it).

All of this may belong in `mytheus-web`, which has never been audited.

## Do NOT build (no backend exists, or deferred)

`/tasks`, `/decisions`, `/search`, `/billing`, `/settings/integrations` - no backend for any; no
`decisions` or `action_items` table in any migration. 10 of 11 marketing feature pages promise
features with nothing behind them. `/students/verify:48-51` claims a verification link was sent
while making **no network call** - raise it, it is a truthfulness issue, not wiring. Temporal
workflows including `RTBFWorkflow` are comment-only stubs registered as real workflows - note it,
do not build it.

Also note: `_arm`'s broad `except Exception` in the BFF masks authz errors as "degraded". Worth
tightening before the empty arms get real loaders - which is frozen work, so just record it.

---

## Your checklist

- [ ] **D1** Cross-tenant speaker assignment *security, first*
- [ ] **D2** Transcript filter, partition bound, tests
- [ ] **D3** Eleven dead gateway prefixes + a CI check
- [ ] **D4** `integration-service` minimum producer
- [ ] **D5** Audit log read/export side
- [ ] **D6** `suggested` speaker writer
- [ ] **D7** Meeting status reconciliation
- [ ] **D8** `GET /chat` history *(needs Samar P2)*
- [ ] **D9** Delete / cancel meeting
- [ ] **D10** Participant add / remove
- [ ] **D11** Room org segment + fake-sync cutover
- [ ] **D12** DLQ naming, encoding, and a reader
- [ ] **D13** Import the generated Avro clients
- [ ] **D14** Delete `user-sync-events`
- [ ] **D15** BFF in-process import
- [ ] **D16** Mock 404 in three routes + dashboard clocks
- [ ] **D17** Realtime token refresh + connection gates *(needs Samar P11)*
- [ ] **D18** Reconcile the rebuild branch

---

# PART 3 - BUILD PROGRAM: the confirmation spine, the knowledge graph, retrieval, integrations, delivery

Roughly 45 tasks, about 84 engineer-days. Full specs:
[`spec/spec-spine-graph.md`](spec/spec-spine-graph.md), [`spec/spec-agents.md`](spec/spec-agents.md),
[`spec/spec-agent-reality.md`](spec/spec-agent-reality.md),
[`spec/spec-integ-notif.md`](spec/spec-integ-notif.md).

Why this grouping: it is the data spine and everything that reads from or writes to it. Your Part 1
lane is already API surface and data correctness, and the spine is the same discipline.

## E1 - The confirmation spine, size L, about 20 days

"No silent commits" is the architecture's central integrity rule and it currently has no data
model. `decisions`, `action_items` and `agent_suggestions` exist in none of the 11 revisions.

- [ ] Migration for `decisions`, `action_items` and `agent_suggestions` per
      `_canon/02-data-model.md:84-86`, following the `meeting_speakers` conventions
      (`25e7cede5672`): bare uuid PK, `org_id` FK `ON DELETE RESTRICT`, `tenant_isolation` RLS,
      CHECK domains
- [ ] The highest-value single item in this part: a confirmation-monotonicity trigger that raises
      when a confirmed row returns to candidate. It applies the `meeting_speakers`
      manual-authoritative precedent and makes "no silent commits" structural rather than
      conventional.
- [ ] The piece most likely to be dropped: nothing writes a candidate row. `core_api/consumer.py`
      handles only `presence-events`, while `agent-results` carries `decision_candidate` and
      `action_candidate` in its Avro enum and lists core-api as a consumer that does not exist.
      Without this the confirm routes have nothing to confirm. Size M.
- [ ] `POST /decisions/:id/confirm` and `POST /actions/:id/confirm`, with outbox emission and
      audit. Their absence is deliberate and documented at `core_api/main.py:6-9`, because a
      speculative producer would put unconfirmed edges into the knowledge graph. You are
      implementing the confirmation gate, not removing a guard.
- [ ] The outbox on confirm is genuinely small: `enqueue` plus an envelope with
      `idempotency_key=decision:{id}:confirmed`. The relay is topic-agnostic, both Avro subjects
      are registered, and `outbox.py:14`'s unique `(topic, idempotency_key)` already makes a
      double-confirm a no-op.
- [ ] List, read, patch and dismiss routes, plus the `worklogs` migration and routes
- [ ] The gateway needs no change. `edge.py:146-155` already maps all four prefixes; the 404 is
      core-api having no router mounted (`main.py:22` TODO). This closes four of the eleven dead
      prefixes from your D3.

Three decisions are needed before the migration:

0. `decisions.status` has no enumerated domain in canon at all. Proposed:
   `candidate`, `confirmed`, `dismissed`, `superseded`. Note the Avro `status` enums carry
   `confirmed` as their only symbol, so an unconfirmed decision is unrepresentable on the bus by
   construction. The database domain is wider than the wire domain, deliberately.

1. `action_items.status` has two axes, confirmation (candidate to confirmed) and work progress
   (open, in_progress, done), while canon gives one column. The spec recommends the single column,
   because `integration-service` syncs external ticket status back onto actions, which is a
   progress update on an already-confirmed row.
2. `decisions.supersedes_id` and `action_items.blocked_on_decision_id` are net-new relative to
   canon and are load-bearing. Without them `FOLLOWS_FROM` and "actions blocked on a decision" have
   no relational source, and ADR-0005's "graph is a projection, never the only copy" rollback
   promise is false.

## E2 - The knowledge graph: take the R1 relational fallback, size M

Apache AGE exists nowhere. Beyond the docstrings, the production image
`supabase/postgres:15.8.1.060` ships 1535 extension files including pgvector and no `age.control`
and no `age.so`, so `CREATE EXTENSION age` fails on the image `compose.prod.yml:79` runs today.

Recommendation, pending Sushan's nod: option (b), the R1 relational materialized-view fallback,
taken now at its designated P8 checkpoint.

- ADR-0005 D4's own trigger is met, more decisively than the ADR anticipated
- `22-roadmap-and-phasing.md:191-199` pre-authorises R1 as planned behaviour, not a slip
- The migration machinery this normally costs is free right now. All of ADR-0005's snapshot,
  bulk-load, dual-read and per-tenant cutover apparatus is migration from an installed AGE. Nothing
  is installed, `knowledge-graph-service` abstracts nothing yet, and no consumer exists. This is
  the cheapest moment the decision will ever be made, and it gets more expensive weekly.
- It is strictly better on isolation: RLS at the database, the same mechanism and test suite as
  every other table, rather than AGE's template-parameter arrangement. Edges commit in the same
  transaction as the confirmed row. One backup domain, one PITR timeline, one residency boundary,
  which is ADR-0016's whole thesis.
- The feature does not suffer. Contradiction detection's semantic arm is Qdrant
  `decisions__{org_id}` under every option, and its structural arm is a depth-bounded recursive CTE
  over roughly three decisions per meeting. What is given up is ad-hoc Cypher, which has no caller.

- [ ] `knowledge-graph-service` consumer of `decisions` and `actions`, size M to L
- [ ] The condition on the recommendation: build `POST /graph/query/{template}` for real, because
      that shield is what keeps the Neo4j Aura exit open
- [ ] Amend ADR-0005 rather than superseding it: a status note plus a correction that R1's original
      trigger, managed Supabase gating extensions, no longer binds since ADR-0016 moved to a
      self-hosted VPS with root. The real blocker is the Nix-built image.

## E3 - The RAG pipeline, size L, about 20 days

Every arm already exists and is genuinely good. Nothing calls any of them.

- [ ] Five stages: expand, retrieve, rerank, assemble, cite
- [ ] Expand into two or three query variants, skipped when throttled
- [ ] Three parallel arms fused by RRF k=60 to top-50
- [ ] Rerank against the user's actual query rather than the expansion, to top-10 via
      `bge-reranker-base`. `search-service`'s response is already shaped to feed `/rerank` with
      zero adapter (`routes.py:16-21`).
- [ ] A 3,000-token assembly, dropping uncited passages
- [ ] `proto/rag.proto` is a 15-line placeholder but is not blocking: every arm exposes a real REST
      contract and the docstring says gRPC is a transport change only

## E4 - Delivery to the browser, about 8 days

Agent output cannot reach a user today.

- [ ] `realtime-gateway` is Kafka producer-only. Add a consumer for `agent-results` and
      `agent-stream`. The existing caption path is a directly copyable template of about 60 lines
      with org-scoping already correct: an org-scoped Redis channel, per-channel local socket sets,
      and `meetingId` derived from the channel rather than the body (`server.ts:262-292`).
- [ ] A new `agent` socket event plus the listener in `useMeetingSocket.ts`, which today listens for
      `caption` and `presence` only
- [ ] A design decision nobody has made: `AgentSidebar.tsx` is 349 lines of finished UI on
      `mockAgentSidebar`, and its nine frozen sections do not map onto the five `AgentResultKind`
      symbols. That mapping is specified nowhere. Settle it before wiring.
- [ ] The `agent_suggestions` durable read model, which the schema's own documentation demands
- [ ] Note the frozen-schema constraint: `agent-stream` needs ADR-0017's composite
      `{org_id}:{meeting_id}` partition key, which `EventEnvelope.partition_key`
      (`events.py:47-48`) does not support

## E5 - Integrations, 19 tasks (16 in the MVP band), about 36 days

- [ ] Do the calendar-attachment path first. It is the highest-leverage work in the whole program,
      because it lights up already-built and already-tested downstream code with zero changes in
      the intelligence repo.
- [ ] Correction to what I told you earlier: `/webhooks` is routed (`edge.py:179`) but is not on the
      public allowlist. `PUBLIC_PREFIXES` (`edge.py:41-55`) has only `/clerk/webhook`,
      `/stripe/webhook`, `/livekit/webhook` and `/oauth/`, so every provider webhook is
      auth-rejected before reaching the service. A one-line fix, load-bearing, and it goes first.
- [ ] Schema drift, and the registered schema wins. Doc 21 section 7 prints
      `kind/connection_id/meeting_id/external_ref/body_json` while the registered Avro is
      `provider/event_type/external_id/payload_json`. The only real payload contract is
      `{meeting_id, object_key, filename}` with a mandatory `{org_id}/` key prefix
      (`consumer.py:64-100`).
- [ ] Data model: `integration_connections`, `oauth_tokens` envelope-encrypted at rest,
      `integration_links`
- [ ] OAuth per provider, token refresh and revocation, webhook receivers, bidirectional action
      sync
- [ ] The `/settings/integrations` UI, currently mock-backed. Check the mytheus-web scope decision
      first, since this is `(app)` surface.
- [ ] Respect the R8 guardrail: Calendar plus Slack plus exactly one of Jira or Linear. The absence
      of a second tracker is the cap working, not a gap.

External blockers, start on day one: Google verification, a Slack app, and the Jira-versus-Linear
choice. Also blocked outside this band: `action_items` does not exist in core-api until your E1
lands, so sync has nothing to sync.

Encryption blocker: no KMS exists. The spec recommends an env-wrapped per-tenant DEK using the same
envelope layout, so the KMS swap is later and local.

## Your build checklist

- [ ] **E1** Confirmation spine: migration, monotonicity trigger and candidate consumer first
- [ ] **E2** Knowledge graph, pending Sushan's R1 nod, plus the ADR-0005 amendment
- [ ] **E3** RAG pipeline
- [ ] **E4** Delivery to the browser, and settle the sidebar mapping
- [ ] **E5** Integrations, calendar-attachment path first, allowlist fix before that
