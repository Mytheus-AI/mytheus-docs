# Verification: mytheus-backend feature gaps

Read-only audit. Root: `/Users/sushan/Projects/Mytheus/mytheus-backend`.
All line cites are against current working-tree source (worktrees under `.worktrees/` and
`mytheus-backend/.worktrees/` were excluded — they are stale copies).

Baseline facts established while verifying:

- **core-api mounts exactly 5 routers**: `services/core_api/main.py:16-20` →
  `meetings`, `org_hierarchy`, `chat`, `speakers`, `transcript`. Nothing else.
- **Tables that any migration actually creates**: `organizations`, `users`, `departments`,
  `teams`, `team_members`, `meetings`, `meeting_participants`, `data_residency`,
  `tenant_settings`, `outbox`, `inbox` (`migrations/versions/82d44163c933_...py:41-152`),
  `rbac_grants` (`b7d9f3a55c21_...py:51`), `audit_logs` + partitions
  (`c3e8b1d47f92_...py:62,86,92`), `chat_messages` (`5ed978467cad_...py:74,90`),
  `meeting_speakers` (`25e7cede5672_...py:70`), `meeting_rooms` (`3d13f46b960f_...py:51`).
  `transcript_chunks` is read but owned by `mytheus-realtime` — correctly not created here.

---

## Part 1 — verdicts on the 11 claimed gaps

### 1. Gateway routes `/decisions`, `/actions`, `/worklogs`, `/suggestions` 404 in core-api — **PENDING (confirmed)**

`services/api_gateway/edge.py:151-155` maps all four prefixes to `core-api`.
`services/core_api/main.py:16-20` mounts no router serving any of them, and
`services/core_api/main.py:22` says so outright: `# TODO: /decisions /actions /worklogs /suggestions land with their own bands.`
Confirmed 404 (routed, then unhandled).

Nuance the audit missed: **this is the small half of the problem.** See Part 2 item A —
eleven *other* gateway prefixes route to services that mount zero routers.

### 2. `POST /decisions/:id/confirm` / `/actions/:id/confirm` absent; no `decisions` / `action_items` table — **PENDING (confirmed)**

No route (see #1). No migration creates either table (baseline list above). This is documented
as intentional-for-now in `services/core_api/main.py:6-9`: the confirmation rule is stated, and
the docstring explains a speculative producer would put unconfirmed edges into the knowledge
graph. So: real pending work, with a deliberate reason for not having been faked yet — not a
defect, but not deferred by any ADR either.

### 3. No `GET /chat` history route — **PENDING (confirmed)**

`services/core_api/routes/chat.py:88` is the only route in the module (`POST /api/v1/chat`).
Grep for `chat` in `services/bff/*.py` returns nothing, so the BFF does not read it either.
`chat_messages` is therefore a **write-only table**: rows are inserted and never read by any
code in this repo. Confirmed as stated.

### 4. `DELETE /meetings/{id}` absent; `deleted_at` read everywhere, written nowhere — **PENDING (confirmed, and broader than stated)**

`services/core_api/routes/meetings.py` has POST/GET/GET-one/GET-participant-check/PATCH
(lines 153, 243, 291, 302, 386) — no DELETE. `org_hierarchy.py:273` has
`DELETE /teams/{id}/members/{user_id}`, but that is a hard `DELETE FROM team_members`, not a
soft-delete, and there is **no** delete for teams, departments, meetings, or chat messages.

`deleted_at IS NULL` is read in ~25 places (`libs/shared/authz.py:69`,
`libs/shared/org_resolver.py:81,98`, `services/core_api/routes/meetings.py:137,253`,
`routes/chat.py:58`, `routes/speakers.py:39`, `routes/transcript.py:87`,
`routes/org_hierarchy.py:54,101,164`, `services/bff/org_chart.py:65,68,73`,
`services/bff/dashboard.py:167`, `services/meeting_service/routes.py:113`, …).

The **only** writers anywhere are on `users`, from the Clerk sync path:
`services/identity_tenant_service/routes/auth.py:182,198` and
`services/identity_tenant_service/reconciliation.py:82,96`.
`meetings`, `teams`, `departments`, `organizations`, `chat_messages` have **no** `deleted_at`
writer in production code — `organizations.deleted_at` is set only in
`tests/test_livekit_webhook.py:370` and `tests/test_org_resolver.py:28`.

### 5. No consent-write endpoint; `consent_required=true` → permanent 428 — **PENDING (confirmed; ADR names it as a required follow-up)**

The 428 gate is live: `services/meeting_service/routes.py:138-139,186-187`.
The only write of `consented_at` in the entire repo is
`tests/test_meeting_token_route.py:355` (a raw SQL `UPDATE` in a test fixture).
`services/identity_tenant_service/routes/tenant_settings.py:35-37` deliberately excludes
`consent_required` from the settings PATCH, citing ADR-0018 and saying it "wants its own route
and its own audit action".

**Not deferred-by-ADR — the opposite.** `mytheus-docs/adr/0018-consent-entity-behind-flag.md`
"Follow-ups this ADR creates" item 2 explicitly requires "the `meeting-service` token-mint 428
check **and the consent-acknowledgment write path**". Half of that follow-up shipped; the write
path did not. The audit's severity framing is correct: with `consent_required=true` there is no
in-product way to ever clear the 428.

Corollary also missing: no route to *set* `consent_required` at all, so today the trap is only
reachable by direct SQL. Both routes are the same task.

### 6. No start/stop-recording route — **PENDING (confirmed)**

`services/meeting_service/transport.py:137` — `LiveKitTransport.start_egress` raises
`NotImplementedError("start_egress lands with the nexus-recordings bucket (Task 4)")`.
It has **no caller** anywhere outside `FakeTransport` (`transport.py:223`) and tests.
There is no stop/`stop_egress` method at all — the interface (`transport.py:54`) declares only
`start_egress`.

Consequence chain worth recording: `services/meeting_service/webhook.py:140,315` handles
`egress_ended` and is the only writer of `meeting_rooms.recording_ref` / `egress_id`
(`webhook.py:179`). Since egress is never started, that webhook branch can never fire, so
`recording_ref` and `egress_id` are **columns with no reachable writer**, and `storage.py`'s
key convention has no live producer.

### 7. Nothing writes `meeting_speakers` with `assignment='suggested'` — **PENDING (confirmed; "always empty" is overstated)**

The only production write is the upsert at `services/core_api/routes/speakers.py:102-110`,
whose `assignment` is the literal `'manual'` on both the INSERT and the DO UPDATE branch
(deliberate, per the R7 note at `speakers.py:8-15`). No other writer exists in this repo, and
none in `mytheus-realtime` either — `mytheus-realtime/CLAUDE.md:40` states the diarized
`speaker_id` is emitted as-is and "the mapping is resolved downstream via `meeting_speakers`,
never guessed at write time", i.e. it explicitly declines to write it.

Correction to the claim: the list is **not** always empty — `GET .../speakers`
(`speakers.py:47`) returns rows created by manual PATCHes. What is genuinely missing is the
*suggestion* half: the speaker-mapping UI has no candidate list to present, because no
diarization-derived `suggested` row is ever produced. `confidence` is likewise a
column with no writer.

### 8. `meetings.status` never reconciles with `meeting_rooms.status`; nothing sets `'live'` — **PENDING (confirmed, with one correction)**

`services/meeting_service/webhook.py:233-234` sets `meeting_rooms.status = 'active'` on
participant-joined; `services/meeting_service/consumer.py` sets the row `pending` on
`meeting.created` and ended on `meeting.ended`. Nothing in either service touches
`meetings.status`, and `services/meeting_service/` writing it would be a cross-owner write
anyway.

**Correction:** `'live'` *can* be set — `PATCH /api/v1/meetings/{id}`
(`services/core_api/routes/meetings.py:386-410`) writes whatever `MeetingUpdate` carries, and
`tests/test_meetings_api.py:374` exercises exactly that. So the accurate statement is: *nothing
sets it automatically*; it depends on a client remembering to PATCH. `services/bff/dashboard.py:167`
filters on `status IN ('scheduled','live')`, so a real meeting in progress shows as `scheduled`
on the dashboard unless a client patched it.

### 9. `core_api/main.py:3` claims 5 tables no migration creates — **PENDING (confirmed, exactly 5)**

`services/core_api/main.py:3-4` claims ownership of: meetings ✓, participants ✓, speakers ✓,
worklogs ✗, decisions ✗, action_items ✗, agent_suggestions ✗, agent_collaborations ✗,
chat_messages ✓, teams ✓, team_members ✓, departments ✓.
Five non-existent: **`worklogs`, `decisions`, `action_items`, `agent_suggestions`,
`agent_collaborations`**. Count and identities confirmed.

### 10. "The presence consumer is a no-op with no runner" — **NOT A GAP as stated / MISATTRIBUTED**

Two errors in this claim:

- **"no runner" is false.** `services/core_api/consumer.py:118` defines `run_consumer()` — a
  complete confluent-kafka poll loop with Avro deserialization, DLQ routing and manual commit —
  and `consumer.py:181-182` gives it a `__main__` entrypoint.
  `services/meeting_service/main.py:15-17` documents the convention
  (`python -m services.<svc>.consumer` as its own worker process).
- **"no-op" is a mischaracterization.** `handle_presence_event`
  (`services/core_api/consumer.py:71-116`) validates the header, re-asserts `org_id`, raises
  `PoisonMessage`, dedupes via Redis, calls `set_rls`, and records to the inbox
  (`consumer.py:115`). It deliberately does not write a domain table, and
  `consumer.py:104-108` states why: a durable analytics sink needs a new table, which needs
  `_canon/02-data-model.md` + an ADR first. That is a stated, reasoned deferral, not an
  unfinished stub. It is also the best-tested consumer here
  (`tests/test_presence_events_consumer.py`).

**The claim describes the wrong file.** The genuine no-op consumer is
`services/notification_service/consumer.py:37-41`: `_handle(msg)` is `_ = msg  # placeholder
until the consumer is implemented`, with `# TODO(P1)` for the entire body — no org_id
re-assertion, no `set_rls`, no dedupe, no delivery. It *does* have a runner
(`consumer.py:21`, `:45`) and commits every message (`consumer.py:31`), meaning if it were
ever started it would silently consume and discard the `notifications` and `agent-results`
topics. See Part 2 item C.

**Real residual gap in both, which the claim gestures at:** no process manager starts any
consumer. `docker-compose.yml` defines only `postgres`, `redis`, `redpanda`, `temporal`,
`temporal-ui` — no service, no consumer, no relay. No k8s/compose manifest in
`mytheus-platform` references them either. See Part 2 item B.

### 11. `routes/transcript.py` has zero tests — **PENDING (confirmed)**

`grep -rl transcript tests/` returns only `tests/test_livekit_import_boundary.py` (an unrelated
import-boundary check). `tests/test_openapi_contract.py` has no transcript assertion either.
The module (`services/core_api/routes/transcript.py`, 180 lines) has untested cursor
encode/decode (`:61,:65` — base64+JSON, with a `binascii` failure path), a `from_ms` vs
`cursor` precedence rule (`:96-118`), an `is_final` predicate, and a `has_more`/`next_cursor`
keyset boundary. It is also the only core-api route reading a table this repo does not own,
so a schema drift in `mytheus-realtime` would surface here first — and silently.

---

## Part 2 — what the list misses

### A. Eleven gateway prefixes route to services that mount **zero** routers — **PENDING, larger than item #1**

`services/api_gateway/edge.py:146-180` routes these; the target services are empty shells:

| Gateway prefix | edge.py line | Target | Evidence |
|---|---|---|---|
| `/billing`, `/usage`, `/plans`, `/stripe/webhook` | 172-175 | billing-metering-service | `services/billing_metering_service/main.py:11` — `# TODO: add routers for /billing /usage /plans /stripe/webhook` (whole file is `create_app` + TODO) |
| `/notifications` | 176 | notification-service | `services/notification_service/main.py:11` — `# TODO: add router for /notifications` |
| `/admin/tenant`, `/admin/ops` | 177-178 | admin-ops-service | `services/admin_ops_service/main.py:11` — `# TODO: add routers for /admin/tenant/* and /admin/ops/*` |
| `/oauth`, `/webhooks` | 179-180 | integration-service | `services/integration_service/main.py:11` — `# TODO: add routers for /oauth/{provider} and /webhooks/{provider}` |
| `/orgs`, `/rbac` | 168-169 | identity-tenant-service | `services/identity_tenant_service/main.py:26` — `# TODO: add routers for /orgs /rbac` |

`/oauth/` is additionally in `PUBLIC_PREFIXES` (`edge.py:53`) — an unauthenticated gateway
prefix pointing at nothing. `/stripe/webhook` likewise (`edge.py:51`).

`/api/v1/orgs` is referenced by the UI at
`mytheus-ui/src/app/(auth)/sign-up/page.tsx:7`, which notes it is "deliberately deferred".
Note the surface mismatch: UI calls `/api/v1/orgs`, gateway maps `/orgs`.

`/scim/v2` (`edge.py:170`) is the one honest deferral: `routes/scim.py:33-54` returns
`NotImplementedProblem` (RFC 7807 501) rather than 404 — that is the pattern the other nine
should follow if they stay unbuilt.

- Title: **Return 501 `NotImplementedProblem` for routed-but-unbuilt gateway prefixes** (stopgap)
- Files: the 4 empty `main.py`s + `services/identity_tenant_service/main.py`, mirroring `routes/scim.py`
- Size: **S**. No dependencies. Turns 9 silent 404s into typed, discoverable 501s.

### B. No consumer / relay / service process is started anywhere — **PENDING**

`docker-compose.yml` defines only infrastructure (postgres:3, redis:25, redpanda:36,
temporal:57, temporal-ui:76). There is no service container, no
`python -m services.*.consumer`, and no `services/relay/main.py` runner. Consequence: the
transactional outbox (`libs/shared/outbox.py` → `services/relay/main.py:80 run()`) is never
drained, so **no event this repo enqueues is ever published**, and by extension
`services/meeting_service/consumer.py` never receives `meeting.created`, so `meeting_rooms`
rows are only ever created by the token-mint path. This makes items #7 and #8 unfixable in
practice until it lands.

- Title: **Compose/deploy definitions for API services, relay, and the three Kafka consumers**
- Files: `docker-compose.yml`, `Dockerfile`, `Makefile`, `mytheus-platform/*`
- Size: **M**. Blocks: any end-to-end verification of #7, #8, C.

### C. `notification-service` consumer handler is a true no-op that silently commits — **PENDING**

`services/notification_service/consumer.py:37-41`. Missing: header `org_id` re-assertion,
`set_rls`, `is_duplicate`/`record_consumed`, and channel routing. It calls
`consumer.commit()` unconditionally (`:31`), so once started it would burn through
`notifications` and `agent-results` offsets and drop every message with no DLQ. Worse than
not running it. (This is the consumer claim #10 should have named.)

- Title: **Implement `notification_service` consumer handler (RLS + inbox dedupe + channel routing)**
- Files: `services/notification_service/consumer.py`, `libs/shared/inbox.py`, new tests
- Size: **M**. Depends on: B (runner), and on a `/notifications` read surface (A) to be useful.

### D. BFF is missing 3 of the 6 screen endpoints the UI documents — **PENDING**

`services/bff/main.py:11-13` mounts `dashboard`, `org_chart`, `people` →
`GET /bff/dashboard` (`dashboard.py:152`), `GET /bff/org/chart` (`org_chart.py:169`),
`GET /bff/people/{user_id}` (`people.py:67`).

`mytheus-ui/docs/PAGES.md:41-43` requires, and marks the pages "done" against:
`GET /bff/meetings`, `GET /bff/meeting/:id`, `GET /bff/meeting-room/:id`.
`mytheus-ui/CLAUDE.md:23` and `mytheus-ui/system.md:87` name `/bff/meeting-room/:id`
specifically ("the meeting, participants, the user's worklog, open suggestions, and presence
in one composite payload").

- Title: **Add `bff` meeting screen endpoints: `/bff/meetings`, `/bff/meeting/:id`, `/bff/meeting-room/:id`**
- Files: new `services/bff/meetings.py`, `services/bff/main.py`, `tests/test_bff_meetings.py`
- Size: **M** (**L** for `/bff/meeting-room/:id` — its payload needs worklogs + suggestions,
  which is blocked on #1/#2)
- Depends on: #1 (worklogs/suggestions) for the full room payload; the list and detail
  endpoints can ship without it.

### E. No participant add/remove route — **PENDING (not in the original list)**

`meeting_participants` rows are inserted **only** inside `POST /api/v1/meetings`
(`services/core_api/routes/meetings.py:207-215`, from `body.participant_ids`). There is no
`POST`/`DELETE /meetings/{id}/participants/...`. So a meeting's attendee list is immutable
after creation: nobody can be invited to, or removed from, an existing meeting. Since the
token-mint 428/403 gate keys on participancy (`services/meeting_service/routes.py:121-152`),
a late invitee can never join.

- Title: **Add participant add/remove routes on `core-api`**
- Files: `services/core_api/routes/meetings.py`, `services/core_api/schemas.py`, tests
- Size: **S**. No dependency.

### F. `data_residency` — table with neither reader nor writer — **PENDING**

Created at `migrations/versions/82d44163c933_...py:107`, modelled at
`services/identity_tenant_service/models.py:207`, named in that service's docstring
(`main.py:4`). Zero SQL touches it anywhere in `services/` or `libs/`. Rows are never
created, so the residency columns (`db_cell`, `qdrant_region`, `object_bucket_region`) are
inert. Either provisioning must seed a row, or the table is premature.

- Title: **Seed/maintain `data_residency` at org provisioning, or record why it stays empty**
- Files: `services/identity_tenant_service/provisioning.py`, tests
- Size: **S**. Related to G.

### G. Org provisioning does not create a `tenant_settings` row — **PENDING (self-documented)**

`services/identity_tenant_service/routes/tenant_settings.py:29-33` states it plainly:
`provision_organization()` creates an `organizations` row and a `tenant-events` outbox row but
**not** a `tenant_settings` row, "That provisioning omission is a separate gap and is recorded
as one." The settings PATCH works around it with an upsert; every other reader does not — e.g.
`services/meeting_service/routes.py:139` relies on `COALESCE(ts.consent_required, false)`.

- Title: **Create `tenant_settings` (and `data_residency`) rows in `provision_organization()`**
- Files: `services/identity_tenant_service/provisioning.py`, tests
- Size: **S**. Pairs with F.

### H. Temporal workflows are all empty shells with no registered activities — **PENDING**

`services/workflow_service/worker.py:35` — `activities=[],  # TODO: register activity functions`.
Every workflow body is a TODO: `workflows/dunning.py:13`, `workflows/rtbf.py:15`,
`workflows/meeting_lifecycle.py:10`, `workflows/action_followup.py:18,21`. `rtbf.py` matters
most — it is the GDPR right-to-be-forgotten path, and it deletes nothing.

- Title: **Implement Temporal activities (RTBF first)**
- Files: `services/workflow_service/**`
- Size: **L**. `action_followup` depends on #2 (`action_items`).

### I. Per-tenant rate limiting is a stub — **PENDING (acknowledged)**

`services/api_gateway/main.py:9` — "The per-tenant rate limit is deliberately still a stub. The
full posture belongs with the WAF…". Accurate as a stated deferral; recorded so it is not
mistaken for shipped.

### J. `MeetingTransport.subscribe_room_audio` unimplemented — **PENDING**

`services/meeting_service/transport.py:141` — `NotImplementedError("subscribe_room_audio lands
with the transcription tap")`. No caller. This is the audio path into
`mytheus-realtime`'s transcription-service, so `transcript_chunks` is never populated by
anything in this system today — which in turn means `routes/transcript.py` (item #11) can only
ever return an empty page.

- Title: **Wire the LiveKit → transcription audio tap**
- Files: `services/meeting_service/transport.py`, cross-repo with `mytheus-realtime`
- Size: **L**. Depends on: B.

---

## Consolidated PENDING task list

| # | Task | Files | Size | Depends on |
|---|---|---|---|---|
| 1 | Return 501 `NotImplementedProblem` for the 9 routed-but-unbuilt gateway prefixes | 4 stub `main.py`s + identity `main.py` (pattern: `routes/scim.py`) | S | — |
| 2 | Consent-acknowledgment write path + `consent_required` toggle route (ADR-0018 follow-up 2) | new `services/core_api/routes/consent.py` or meeting-service; `tenant_settings.py`; migration none needed | M | — |
| 3 | Participant add/remove routes | `core_api/routes/meetings.py`, `schemas.py` | S | — |
| 4 | Soft-delete routes (`DELETE /meetings/{id}`, teams, departments) writing `deleted_at` | `core_api/routes/meetings.py`, `org_hierarchy.py` | S | — |
| 5 | `GET /api/v1/chat` history (keyset, RLS-scoped) | `core_api/routes/chat.py`, `schemas.py` | S | — |
| 6 | Tests for `routes/transcript.py` (cursor, `from_ms`, precedence, `is_final`, page boundary) | `tests/test_transcript_api.py` | S | — |
| 7 | Fix `core_api/main.py:3` docstring to claim only tables that exist | `services/core_api/main.py` | S | — |
| 8 | Seed `tenant_settings` + `data_residency` at org provisioning | `identity_tenant_service/provisioning.py` | S | — |
| 9 | Compose/deploy defs for API services, relay, and the 3 consumers | `docker-compose.yml`, `Makefile`, `mytheus-platform` | M | — |
| 10 | Implement `notification_service` consumer handler (RLS + dedupe + routing) | `notification_service/consumer.py` | M | 9 |
| 11 | `bff` meeting screens: `/bff/meetings`, `/bff/meeting/:id` | new `services/bff/meetings.py` | M | — |
| 12 | `meetings.status` ↔ `meeting_rooms.status` reconciliation (auto `'live'`/`'ended'`) | `meeting_service/webhook.py` → event → `core_api` consumer (core-api must remain the writer) | M | 9 |
| 13 | Diarization → `meeting_speakers` `assignment='suggested'` producer | new core-api consumer; cross-repo with `mytheus-realtime` | M | 9, 16 |
| 14 | `worklogs` + `decisions` + `action_items` + `agent_suggestions` + `agent_collaborations`: migrations, models, CRUD routes | migration, `core_api/models.py`, `schemas.py`, new route modules | L | — |
| 15 | `POST /decisions/:id/confirm` + `/actions/:id/confirm` with outbox emission | new route modules, `libs/shared/outbox.py` | M | 14 |
| 16 | Recording: start/stop egress route + `LiveKitTransport.start_egress` | `meeting_service/transport.py`, `routes.py` | M | 9 |
| 17 | `/bff/meeting-room/:id` composite payload | `services/bff/meetings.py` | L | 11, 14 |
| 18 | LiveKit → transcription audio tap (`subscribe_room_audio`) | `meeting_service/transport.py` + `mytheus-realtime` | L | 9 |
| 19 | Temporal activities, RTBF first | `services/workflow_service/**` | L | 14 (for `action_followup`) |
| 20 | Per-tenant rate limiting (currently a stub) | `api_gateway/main.py` + WAF | M | — |

Items 1, 3–8 are all **S** with no dependencies — that is the whole quick-win band.
Item 9 unblocks 10, 12, 13, 16, 18 and is the highest-leverage single piece of work.
