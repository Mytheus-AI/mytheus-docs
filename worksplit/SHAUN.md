# Shaun — API Surface, Event Backbone, Safe UI Fixes

**18 tasks.** Start with D1 — it is a live cross-tenant hole and it is small.

Audit: https://claude.ai/code/artifact/da281c5f-d990-40b4-9487-d9f857833b08
Evidence: [`audit/`](audit/) · [`verify/verify-backend.md`](verify/verify-backend.md) · [`verify/verify-ui.md`](verify/verify-ui.md)

---

## ⏱️ Ordering

**Write now, verify after Sushan's S1 + S2.** Every request 500s then 403s until those land. D1 and
D2 are pure backend with tests — start there.

**Most UI work is frozen.** The scope of `mytheus-ui` is unresolved (see README). Only D16 and D17
are assigned, because they are correct under either outcome. Do not wire `/meetings`, `/org`, the
dashboard arms or the generated types until Sushan decides.

---

## D1 — Cross-tenant speaker assignment (HIGH, start here)

`core_api/routes/speakers.py:97-133` accepts a caller-supplied `user_id` and relies solely on a
foreign key for tenancy. Its comment claims the FK blocks another tenant's `user_id`.

**Two sibling routes document that as false and each run an explicit probe:**
`routes/meetings.py:186-192` and `routes/org_hierarchy.py:172` both run
`SELECT 1 FROM users` precisely because the FK does not enforce tenancy.

`create_meeting` got the fix. `assign_speaker` never did. No test covers it. Flagged independently
by two auditors.

- [ ] Add the RLS-scoped `SELECT 1 FROM users` probe before the write
- [ ] Correct the misleading comment
- [ ] Test: tenant A assigning a tenant B `user_id` returns a clean 404 — match the 404-before-403
      ordering used at `routes/users.py:80-91`
- [ ] Grep for a fourth route trusting an FK for tenancy

---

## D2 — Transcript read path

`core_api/routes/transcript.py`:

- `:123` missing `deleted_at IS NULL` — serves soft-deleted chunks
- `:134` no `created_at` bound, so monthly partition pruning is forfeited and every query scans all
  partitions
- **Zero tests**, despite being the one endpoint the live UI depends on

- [ ] Add the filter and the bound
- [ ] Tests: pagination, `from_ms` seek, tenant scoping, soft-delete exclusion

---

## D3 — Eleven gateway prefixes route to nothing

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

## D4 — `integration-service`: unblock a finished consumer

The most valuable of the five scaffolds. `mytheus-intelligence/services/ingestion_service/consumer.py:75`
is a **complete, tested** consumer of `integration-events` — S3 fetch, RLS-scoped pipeline, inbox
dedupe, DLQ, a 260-line test suite — and `integration-service` is its **sole designated producer**
with zero producer code. Calendar-attachment ingestion is dead in production and CI cannot see it,
because the tests inject synthetic messages.

Scope this as **minimum viable**: enough to make the built consumer fire. Not the full OAuth
connector band.

- [ ] Minimum producer emitting `integration-events` with `(google_calendar, attachment_to_ingest)`
- [ ] Match the Avro contract exactly — `{provider, event_type, external_id, payload_json}`
- [ ] Decide with Sushan whether the OAuth half is in scope now or stays a scope conversation

---

## D5 — Audit log has no read side

⚠️ **Correction to the first-pass claim.** `libs/shared/audit.py` **does** have 8 live writers —
`authz.py:176`, core_api meetings and org_hierarchy, and four in identity. The trail is being
written.

What is missing is the **read/export side**: `admin_ops_service` has zero routes, so nothing can
retrieve audit rows. The table itself is well built — partitioned through 2027-06 plus a DEFAULT,
FORCE RLS, UPDATE/DELETE revoked per partition.

- [ ] Read/export route, `require_tier("admin")`
- [ ] Audit which mutations still lack a write call and add them
- [ ] Sushan's S5 will call your helper for grant changes — coordinate

---

## D6 — Nothing writes `assignment='suggested'`

⚠️ **Correction:** the speakers list is **not** always empty — a manual PATCH upserts rows
(`speakers.py:102`). The missing half is specifically **`suggested` + `confidence`**.

`speakers.py` is the only writer of `meeting_speakers` and always writes `'manual'`.
`transcription-service` explicitly never writes it (`speakers.py:5-6`). So the diarized advisory
list that `SpeakerMapping.tsx` is built to render — greyed, with confidence, manual wins — never
appears.

- [ ] Write `suggested` rows from the diarized labels the transcription path already produces
- [ ] Preserve the `meeting_speakers_manual_is_authoritative` trigger semantics (`25e7cede5672:56`)

---

## D7 — `meetings.status` never reconciles with `meeting_rooms.status`

⚠️ **Correction:** `'live'` **can** be set via PATCH (`meetings.py:386`, covered by
`tests/test_meetings_api.py:374`). What is missing is **automatic** reconciliation.

`participant_joined` sets `meeting_rooms.status='active'` (`webhook.py:230-235`) and never touches
`meetings.status`. So a meeting with people in the room still reads `'scheduled'`.

- [ ] Reconcile from the webhook path
- [ ] Decide which of the two state machines is authoritative and write it down — they are separate
      by construction (`866f7eedb0e7:65` vs `b7d9f3a55c21:204`)

---

## D8 — Chat history is unrecoverable

`POST /chat` is the only `chat_messages` route. There is **no `GET /chat`** anywhere — not in
core_api, not in bff. A client that misses the socket relay cannot recover history, unlike
transcript, which has a re-hydrate endpoint for exactly this reason (`transcript.py:6-10`).

⚠️ Depends on Samar's P2 — chat inserts currently fail outright.

- [ ] `GET /chat` with cursor pagination, mirroring the transcript read path

---

## D9 — Soft delete is unreachable

`deleted_at` is read in every existence check (`routes.py:113`, `meetings.py:132`, `speakers.py:39`,
`transcript.py:87`) and **written nowhere**. There is no `DELETE /meetings/{id}` and no cancel
route, so a meeting can never be removed through the API.

- [ ] `DELETE` / cancel route writing `deleted_at`, with the audit write and the outbox event
- [ ] Confirm no read path forgets the filter (D2 fixes one)

---

## D10 — The attendee list is immutable after creation

No participant add/remove route exists. Participants can only be set at meeting creation.

- [ ] Add / remove participant routes
- [ ] Reuse the `SELECT 1 FROM users` RLS probe from `meetings.py:193-201` — same tenancy trap as D1

---

## D11 — LiveKit room naming and the fake-sync trap

`meeting_service/rooms.py:51-96` names rooms `meeting-{meeting_id}` with no tenant segment.

**Low severity by itself** — webhook tenant resolution reads org from room *metadata* and fails
closed six ways, and meeting IDs are UUIDs. The reason to fix it is that it is the one identifier in
the meeting path that is not tenant-qualified.

**The latent trap matters more:** rooms "synced" via `FakeTransport` are marked
`livekit_metadata_synced_at` **permanently**, so at real-LiveKit cutover they never re-sync and
webhook tenant resolution breaks for every pre-existing room.

- [ ] Add an org segment to the room name
- [ ] Fix the fake-sync marking and migrate existing rows — one migration, both changes
- [ ] Keep `ON CONFLICT (org_id, meeting_id)` semantics

---

## D12 — Nothing reads any DLQ, and the DLQ contract is inconsistent

- **No DLQ reader exists anywhere.** Failed messages accumulate unseen.
- `mytheus-intelligence` writes `{topic}.{group}.dlq`; `create_topics.sh` only creates `{topic}.dlq`
- DLQ payloads are **JSON** while `contracts/DlqEnvelope.avsc` specifies **Avro**

- [ ] Reconcile the topic naming — coordinate with Samar's P3
- [ ] Reconcile the encoding: either honour the Avro contract or change the contract
- [ ] A minimal DLQ inspection path, even a script

---

## D13 — Import the generated Avro clients

`mytheus-events` generates Python and TypeScript clients from the 21 schemas. **Adoption is zero** —
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

## D14 — Delete the `user-sync-events` topic

ADR-0010 D4 decided there is **no per-user propagation topic**, and `routes/auth.py:19-21` says so
explicitly — the `TOPIC` constant is used only as a Postgres/Redis dedupe namespace. But the topic,
its subject and its `.dlq` are still created and will always be empty.

- [ ] Remove from `codegen/_schemas.py` `TOPICS`, `subjects/`, and `create_topics.sh`
- [ ] Rename the constant so it stops reading as a topic name
- [ ] Update the expected count in `create_topics.sh` and Samar's P3 gate

---

## D15 — BFF imports another service in-process

`bff/people.py:43` imports identity-tenant-service **in-process, across a container boundary**. Its
justification ("one deployable") has expired — compose runs them separately, so **two processes now
write the same Redis key**.

- [ ] Replace with a real call through the proper boundary, or move the logic
- [ ] Check for other in-process cross-service imports

---

## D16 — The mock `notFound()` bug — in three routes, not one ⚠️ *safe under either UI outcome*

`mockMeetings.find(...)` → `notFound()` runs **before** the real fetch in all three:
`transcript/page.tsx:56`, `room/page.tsx:22`, `[meetingId]/page.tsx:40`.

**Every real backend UUID 404s.** The one page with a genuine backend read gates its own existence
on a mock array.

Also here — **the dashboard renders two clocks**: `dashboard/page.tsx:47` uses the real date while
`StatGrid`/`TodaysMeetings` use `MOCK_NOW = 2026-07-01` (`format.ts:9`). Today it reads
"September 7 · 0 meetings today" above a July stat grid. Visible to anyone who opens the app now.

⚠️ *`mytheus-ui` is sensitive — `git fetch` first, no build, minimal diff.*

- [ ] Remove the mock existence gate in all three routes; let the real fetch determine 404
- [ ] Fix `MOCK_NOW` so the dashboard shows one consistent date
- [ ] Add a loading state to the transcript page — it has none

---

## D17 — Realtime auth and connection gates ⚠️ *safe under either UI outcome*

- `MeetingRoom.tsx:45-53` fetches the Clerk token **once on mount** and never refreshes. Socket.io
  and LiveKit both auto-reconnect and re-handshake with a stale JWT.
- `useLiveKitRoom.ts` **never re-mints on reconnect** despite its own docstring at `:6-11` promising
  it.
- `useLiveKitRoom.ts:85` gates connection on `NEXT_PUBLIC_LIVEKIT_URL` — a value the **server always
  supplies** in the token response, so the gate blocks on config that is not needed.
- `useLiveKitRoom.ts:46-50` hand-writes a **second** `TokenResponse` duplicate.

Mobile already solves the token problem correctly — `src/api/provider.tsx` pulls a fresh token per
request. Follow that pattern.

⚠️ Cannot be verified until Samar's P11 makes the env vars work in a container.

- [ ] Refresh the token on reconnect; re-mint on LiveKit reconnect
- [ ] Drop the `NEXT_PUBLIC_LIVEKIT_URL` connection gate; prefer the server-supplied value
- [ ] Collapse both hand-written `TokenResponse` duplicates onto the generated type

---

## D18 — Reconcile the rebuild branch

`origin/local-mytheus-rebuild` is **6 ahead / 11 behind** main, last touched 2026-08-13, and
overlaps `mock/data.ts`, `MeetingRoom.tsx`, `.env.example` and 8 marketing files.

**Its merge would clobber `.env.example`** — that file did not exist at the merge-base, and main
added the whole Clerk + gateway contract to it.

The branch also carries the `mytheus-web` scope claim (commit `0455cc8`) that is currently
unresolved — see the README. **Reconcile the branch mechanically; do not resolve the scope question
yourself.**

- [ ] Rebase onto main, resolving the `.env.example` conflict in main's favour
- [ ] Inventory exactly what the 6 commits change so Sushan can decide the scope question with facts
- [ ] Do not merge until the scope question is settled

---

## ⛔ Frozen pending Sushan's scope decision — do not start

Wiring `/meetings` → `GET /api/v1/meetings` · `/org` → `GET /bff/org/chart` · the three hardcoded
`[]` dashboard arms · importing `generated/identity-tenant-service.ts` and
`generated/meeting-service.ts` (zero importers today) · UI test infrastructure (there is **none** —
no `error.tsx` anywhere, only `(app)/loading.tsx` and root `not-found.tsx` exist repo-wide, while
`transcript.ts:81-85` deliberately rethrows `ApiError` with nothing to catch it).

All of this may belong in `mytheus-web`, which has never been audited.

## Do NOT build (no backend exists, or deferred)

`/tasks`, `/decisions`, `/search`, `/billing`, `/settings/integrations` — no backend for any; no
`decisions` or `action_items` table in any migration. 10 of 11 marketing feature pages promise
features with nothing behind them. `/students/verify:48-51` claims a verification link was sent
while making **no network call** — raise it, it is a truthfulness issue, not wiring. Temporal
workflows including `RTBFWorkflow` are comment-only stubs registered as real workflows — note it,
do not build it.

Also note: `_arm`'s broad `except Exception` in the BFF masks authz errors as "degraded". Worth
tightening before the empty arms get real loaders — which is frozen work, so just record it.

---

## Your checklist

- [ ] **D1** Cross-tenant speaker assignment ⚠️ *security, first*
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
