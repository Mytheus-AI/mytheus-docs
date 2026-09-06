# Audit: `mytheus-backend/services/meeting_service/` + the whole meeting lifecycle

READ-ONLY audit. No file in any repo was modified. All claims are from source, not `.md` status docs.
Root: `/Users/sushan/Projects/Mytheus`. Paths below are relative to that root unless absolute.

Files audited in the target service (9 files, 1389 LOC):

| file | LOC |
|---|---|
| `mytheus-backend/services/meeting_service/__init__.py` | 0 |
| `mytheus-backend/services/meeting_service/main.py` | 17 |
| `mytheus-backend/services/meeting_service/schemas.py` | 20 |
| `mytheus-backend/services/meeting_service/storage.py` | 54 |
| `mytheus-backend/services/meeting_service/rooms.py` | 172 |
| `mytheus-backend/services/meeting_service/routes.py` | 216 |
| `mytheus-backend/services/meeting_service/transport.py` | 281 |
| `mytheus-backend/services/meeting_service/consumer.py` | 305 |
| `mytheus-backend/services/meeting_service/webhook.py` | 324 |

Note: `mytheus-backend/.worktrees/plan-3-4/` and `mytheus-backend/mytheus-backend/.worktrees/plan-5-6/`
contain full copies of these files. All findings below cite the primary tree only.

---

## IMPLEMENTED (file:line)

### meeting-service proper

- **Token mint** `POST /api/meetings/{meeting_id}/token` —
  `mytheus-backend/services/meeting_service/routes.py:153-216`.
  Real end to end: RLS-scoped existence check (`routes.py:107-118`) → participant lookup joined to
  the tenant's consent flag in one round trip (`routes.py:121-150`) → consent gate
  (`routes.py:186-187`) → `meeting_rooms` upsert (`routes.py:189`) → LiveKit token mint
  (`routes.py:205-212`). `org_id`/`user_id` come only from `RequestContext`; the route accepts no
  body at all (`schemas.py:17-20` has only a response model).
- **404-before-403 ordering** `routes.py:173-184` — existence (tenant-scoped, so a cross-tenant id
  is invisible) is checked before participation, so the status code cannot be used to distinguish
  "another tenant's meeting" from "my tenant, not a participant" for ids in other tenants.
- **Deny-by-default** `routes.py:176-184` — `ctx.user_id is None` (verified Clerk user with no local
  `users` mirror row) and "no participant row" both produce the same 403 `NOT_A_PARTICIPANT`.
- **Consent gate is inert by default** `routes.py:141` — `LEFT JOIN tenant_settings` with
  `COALESCE(ts.consent_required, false)`, so a tenant with no settings row is "not required" rather
  than a 500.
- **Statement timeout on the mint path** `routes.py:80,170` — `SET LOCAL statement_timeout = '2000ms'`,
  transaction-local, reverts at commit (matches `libs.shared.tenancy.set_rls`'s `set_config(..., true)`).
- **Deadlock fix is real, not just documented** `routes.py:164-203` — the route depends on
  `tenant_context` (not `tenant_session`) and opens/commits/closes its own short session **before**
  `background_tasks.add_task` is reached. Starlette runs background tasks before a yield-dependency's
  post-yield cleanup, so the previous shape self-deadlocked on repeated mints. The fix in the code
  matches the docstring's account of it (`routes.py:32-52`).
- **Room upsert + deterministic naming** `rooms.py:51-96` — `room_name_for()` is the single source of
  `meeting-{meeting_id}`; `ON CONFLICT (org_id, meeting_id) DO UPDATE SET updated_at = now()`
  deliberately does not touch `livekit_room` or `livekit_metadata_synced_at`; returns
  `RoomUpsertResult(room, needs_livekit_sync)` where `needs_livekit_sync` is
  `livekit_metadata_synced_at IS NULL`.
- **Out-of-band LiveKit metadata sync** `rooms.py:114-150` — calls `transport.ensure_room`, and only
  on success marks the row synced in its own short transaction (`rooms.py:99-111`). Never raises: a
  LiveKit failure is caught and logged (`rooms.py:137-146`), the row stays `NULL`, and the next mint
  for that meeting retries. The retry mechanism is the durable column itself — no queue, no sweeper.
- **`mark_room_ended`** `rooms.py:153-172` — `WHERE ... AND status <> 'ended'`, returns
  `bool(rowcount)`. Tolerant of a missing row and of redelivery; both return `False` rather than raising.
- **LiveKit webhook** `POST /livekit/webhook` — `webhook.py:256-324`. The security ordering is real:
  1. verify signature on the raw body first (`webhook.py:263-267`), any failure → 401;
  2. resolve tenant from the verified event's room metadata (`webhook.py:271-272` →
     `_resolve_org_id` at `webhook.py:107-137`), before `app.current_org` is ever set;
  3. Redis dedupe fast path (`webhook.py:291-297`);
  4. durable `inbox` re-check + apply + outbox enqueue in ONE transaction (`webhook.py:300-323`).
- **Webhook fails closed six ways** `webhook.py:114-136` — metadata absent / not valid JSON / not a
  dict / no `org_id` key / `org_id` not a UUID / UUID names an unknown-or-inactive org. All 400, all
  write nothing, never default to a tenant.
- **Webhook applies `participant_joined`** `webhook.py:202-256` — `FOR UPDATE` row lock on
  `meeting_rooms`, `status = 'active'` only when currently `'pending'` (`webhook.py:230-235`), then
  one `meeting-room-events` outbox row.
- **Webhook applies `egress_ended`** `webhook.py:140-201` — computes the recording key via
  `DEFAULT_STORAGE.recording_key`, writes `recording_ref` + `egress_id`, enqueues
  `meeting-room-events{recording_ready}`.
- **`meeting-events` Kafka consumer** `consumer.py:143-197` (`handle_meeting_event`) and
  `consumer.py:212-238` (`process_decoded_message`):
  - `created` → `ensure_meeting_room`, returns `pending_room_sync` (`consumer.py:184-187`);
  - `ended` → `mark_room_ended` (`consumer.py:188-189`);
  - `updated` → recognized no-op (`consumer.py:190-193`);
  - malformed header / non-UUID `meeting_id` / unrecognized `event_type` → `PoisonMessage` →
    `{TOPIC}.dlq` and commit past (`consumer.py:110-130, 229-233`);
  - the LiveKit call is a **detached** `asyncio.create_task` fired only after the transaction commits
    (`consumer.py:199-208, 235-238`), with a module-level strong-ref set so the task cannot be GC'd
    mid-execution (`consumer.py:95`, discarded via `add_done_callback`).
  - `value.get("livekit_room")` is deliberately never read (`consumer.py:181-183`) — the service
    computes the room name itself, sidestepping the Avro `["null","string"]` drift entirely.
- **Recording key convention** `storage.py:43-44` — `{org_id}/{meeting_id}/{egress_id}`, bucket
  constant `nexus-recordings` (`storage.py:29`). One place, so the webhook and a future
  `start_egress` caller cannot disagree.
- **App wiring** `main.py:11-13` — both routers mounted; the consumer runs as its own process
  (`python -m services.meeting_service.consumer`, `consumer.py:302-303`).

### core-api meeting CRUD (`mytheus-backend/services/core_api/routes/meetings.py`)

- **Create** `meetings.py:153-239` — row + participants + `meeting-events{created}` outbox + audit in
  ONE transaction (`tenant_session` yields inside `session.begin()`).
- **Participant tenancy probe** `meetings.py:193-201` — an explicit `SELECT 1 FROM users WHERE id = :id`
  before each participant insert, because the FK does **not** enforce tenancy (Postgres runs RI checks
  as the referenced table's owner with RLS bypassed). The comment records that this was a real bug that
  persisted a cross-tenant participant row and returned 201; the probe is the fix.
- **List** `meetings.py:243-282` — keyset pagination on `(created_at, id)`, base64 opaque cursor
  (`meetings.py:64-82`), `limit+1` to detect the next page, participants loaded in one query for the
  whole page (`meetings.py:102-126`).
- **Get** `meetings.py:291-299`.
- **Participant check** `GET /meetings/{id}/participants/{clerk_user_id}` — `meetings.py:302-384`.
  Takes the caller's *Clerk-native* id (realtime-gateway has no DB and cannot produce our internal
  UUID). Explicitly does **not** read the `X-Org-Id` header the gateway sends; `org_id` comes only
  from the re-verified bearer token (`meetings.py:325-346`). No `require_tier` gate, matching the
  token-mint precedent, because a genuine participant is not guaranteed an `rbac_grants` row.
- **Patch** `meetings.py:386-449` — 404 before any write; derives `event_type = "ended"` when
  `status` becomes `"ended"` (`meetings.py:407`), which was a real gap (every update previously emitted
  `"updated"` unconditionally); idempotency key carries `updated_at` so the outbox unique index does not
  swallow every update after the first (`meetings.py:424`).

### Speakers (`mytheus-backend/services/core_api/routes/speakers.py`)

- **List** `GET /meetings/{id}/speakers` — `speakers.py:47-77`, ordered by `speaker_id` (stable while
  the user assigns). This *is* the backend behind `mytheus-ui/src/components/meeting/SpeakerMapping.tsx`.
- **Assign** `PATCH /meetings/{id}/speakers/{speaker_id}` — `speakers.py:80-135`. Upsert with
  `assignment = 'manual'` as a SQL literal on both the INSERT and the DO UPDATE side; never
  caller-derived. Second, independent guard: the DB trigger
  `meeting_speakers_manual_is_authoritative` from migration
  `mytheus-backend/migrations/versions/25e7cede5672_feat_core_api_add_meeting_speakers.py:56-151`
  refuses any UPDATE that downgrades a manual row.

### Transcript read (`mytheus-backend/services/core_api/routes/transcript.py`)

- **`GET /meetings/{id}/transcript`** `transcript.py:95-150` — finals only, spoken order, keyset
  cursor, `from_ms` server-side seek. `_assert_meeting_visible` (`transcript.py:75-92`) runs *before*
  reading `transcript_chunks`, because that table has no FK to `meetings` (declarative partitioning
  precludes one) and an unchecked read would return an empty page for another tenant's meeting —
  leaking that the id is well-formed rather than not theirs.

### UI side

- **Token forwarding, no secret in the browser** `mytheus-ui/src/app/api/livekit-token/route.ts:37-64`
  — forwards the caller's Clerk session to api-gateway and passes the minted token back; passes 403 /
  404 / 428 through with their distinct codes intact (`route.ts:56-62`).
- **Room connect + re-mint on reconnect** `mytheus-ui/src/lib/realtime/useLiveKitRoom.ts:52-120`.

### Gateway

- `mytheus-backend/services/api_gateway/edge.py:161-162` — `/api/meetings` → `meeting-service`,
  `/livekit/webhook` → `meeting-service`; `/api/v1/meetings` → `core-api` (`edge.py:148`).
  Longest-prefix match (`edge.py:184-189`) so the deliberate near-collision resolves correctly.

---

## PARTIAL

- **`LiveKitTransport`** `transport.py:63-148` — `ensure_room` (`:74`), `mint_join_token` (`:112`),
  `verify_webhook` (`:143`) are written against the real SDK and look correct (including two SDK
  version-drift fixes: `with_ttl` takes a `timedelta` not raw seconds at `:130`, and
  `WebhookReceiver` takes a `TokenVerifier` not a raw key/secret pair at `:147`). But there is
  **zero test coverage** of any of them, and `select_transport()` (`transport.py:250-264`) returns
  `FakeTransport` whenever `LIVEKIT_API_KEY` is unset — which is every environment that exists.
- **Two disconnected state machines.**
  - `meeting_rooms.status`: `pending → active → ended`. CHECK constraint at
    `mytheus-backend/migrations/versions/866f7eedb0e7_feat_consent_columns_meeting_rooms_.py:65`.
    Default `'pending'` at
    `mytheus-backend/migrations/versions/3d13f46b960f_feat_meeting_service_add_meeting_rooms.py:57`.
    Transitions: `pending → active` on `participant_joined` (`webhook.py:230-235`);
    `* → ended` on `meeting-events{ended}` (`consumer.py:188` → `rooms.py:153-172`).
  - `meetings.status`: `scheduled | live | ended | cancelled`. CHECK at
    `mytheus-backend/migrations/versions/b7d9f3a55c21_0003_rbac_grants_and_canonical_columns.py:204`.
    Transitions: **only** a client `PATCH /meetings/{id}` (`meetings.py:386`).
  - Nothing reconciles them. Nothing ever sets `meetings.status = 'live'` — not the webhook, not the
    consumer, not the token mint. A meeting can be `meeting_rooms.status = 'active'` with people in it
    while `meetings.status` is still `'scheduled'`, indefinitely.
- **Recording** — the *receiving* half exists (`webhook.py:140-201` persists `recording_ref` and
  `egress_id` on `egress_ended`), and the key convention is real (`storage.py:43`). The *starting*
  half does not (see STUB). So the `egress_ended` branch is unreachable in practice.
- **transcription-service session lifecycle**
  `mytheus-realtime/services/transcription-service/src/transcription_service/sessions.py:50-104` —
  genuinely real and idempotent: one stream per `meeting_id` (never per participant — the locked 6x
  cost lever), participant refcount, redelivered `joined` increments without opening a second stream
  (`sessions.py:54-57`), redelivered `left` after close is a safe no-op (`sessions.py:95-97`), and the
  residency/redaction decision is audited in the same transaction that reads it
  (`sessions.py:59-73`). It is the audio *into* the stream that is absent.
- **`FakeTransport`** `transport.py:157-241` — honest about itself: tokens are structurally JWT-shaped
  (`fake.<b64payload>.unsigned`, `transport.py:221`) so callers that only inspect shape cannot
  accidentally treat the fake as real, but the literal `unsigned` segment guarantees real verification
  of a fake token fails. `room_metadata` (`transport.py:176`) lets a test drive the real round trip:
  `ensure_room` → webhook event carrying that metadata → `_resolve_org_id`.

---

## STUB/FAKE

- **`start_egress`** `transport.py:135-137` — `raise NotImplementedError("start_egress lands with the
  nexus-recordings bucket (Task 4)")`. Grep across `mytheus-backend/services/` finds **no caller**:
  the only hits are the Protocol declaration (`transport.py:54`), this raise, and `FakeTransport`'s
  recorder (`transport.py:223-229`).
- **`subscribe_room_audio`** `transport.py:139-141` — `raise NotImplementedError`. No caller anywhere.
- **`/internal/audio`** — the LiveKit server-side audio-hook target.
  `mytheus-realtime/services/transcription-service/src/transcription_service/main.py:119-124`:
  ```python
  @app.websocket("/internal/audio")
  async def internal_audio(websocket: WebSocket) -> None:
      await websocket.accept()
      await websocket.close()
  ```
  Comment on `:121-122` calls itself a scaffold. This is the single most consequential stub in the
  whole flow.
- **`MeetingLifecycleWorkflow`**
  `mytheus-backend/services/workflow_service/workflows/meeting_lifecycle.py:6-11` — the entire class:
  ```python
  @workflow.defn
  class MeetingLifecycleWorkflow:
      @workflow.run
      async def run(self, meeting_id: str, org_id: str) -> None:
          # TODO: coordinate room open/close, post-meeting processing, cleanup
          pass
  ```
- **Compliance gate, closed at the source**
  `mytheus-realtime/services/transcription-service/src/transcription_service/compliance.py:39` —
  `COMPLIANCE_GATE_GREEN: Final[bool] = False`. Deliberately a source constant, not an env var or DB
  row, so no deploy-access holder can flip it (`compliance.py:18-25`). `DeepgramProvider.open_stream`
  calls `assert_compliance_gate_green()` as its first statement, before touching the network or the
  key (`providers/deepgram.py:112`). This is a well-designed hard block, not an accident — but it does
  mean no real transcription can occur.
- **`FakeProvider`** `providers/fake.py:84-99` — returns a scripted three-utterance sequence
  (`providers/fake.py:26-70`: two interims then one final, "hello" → "hello there" → "hello there,
  how are you"). This is what runs in every environment (`providers/factory.py:21-23` returns it
  whenever `DEEPGRAM_API_KEY` is unset).
- **`select_transport()` / `select_provider()`** both default to the fake. Documented as
  fail-toward-obviously-fake rather than fail-toward-silent-no-op-against-a-real-provider
  (`transport.py:250-258`, `providers/factory.py:6-9`). Sound reasoning; the consequence is that
  nothing real is ever exercised.

---

## MISSING

- **No consent-write endpoint anywhere in the codebase.** `meeting_participants.consented_at` is
  *read* at `routes.py:186` and *written* only by a test
  (`mytheus-backend/tests/test_meeting_token_route.py:355`) and defined by the migration
  (`866f7eedb0e7:59`). Grep for `consented_at` across all `.py`/`.ts`/`.tsx` finds no route, no server
  action, no UI. **Consequence:** if any tenant sets `tenant_settings.consent_required = true`, every
  meeting in that tenant becomes permanently unjoinable — 428 forever, with no API to clear it. The
  gate is inert today only because the flag defaults false.
- **No start-recording / stop-recording route.** Not in meeting-service, not in core-api, not in the
  UI. The task brief asked specifically about these: they do not exist in any form.
- **No dedicated end-meeting action.** Ending is a generic `PATCH /meetings/{id}` with
  `{"status": "ended"}` (`meetings.py:386`, event type derived at `:407`).
- **No join/leave endpoints.** Join is implicit in the token mint; leave is only observed via the
  LiveKit webhook.
- **No delete/cancel route.** `deleted_at` is read in every existence check
  (`routes.py:113`, `meetings.py:132`, `speakers.py:39`, `transcript.py:87`) and written nowhere.
- **Nothing ever populates `meeting_speakers` with `assignment = 'suggested'`.**
  `speakers.py:97-122` is the only writer of the table in the entire codebase (grep for
  `meeting_speakers` across `services/` returns only `speakers.py`, `schemas.py:116` and the
  migration), and it always writes `'manual'`. `transcription-service` explicitly never writes it
  (`speakers.py:5-6`: "`transcription-service` only ever emits the diarized label, never writes this
  table"). **Consequence:** `GET /meetings/{id}/speakers` returns `[]` until a human PATCHes something;
  the diarized-speaker list that `SpeakerMapping.tsx` is built to render never appears, and the whole
  "advisory suggestion, greyed, with confidence, manual wins" UI design
  (`mytheus-ui/src/components/meeting/SpeakerMapping.tsx:5-14`) is dead code paths in practice.
- **`meeting-events` has exactly one consumer** — meeting-service's own
  (`consumer.py:86-87`, group `meeting-service`). `mytheus-intelligence` consumes `transcript-chunks`
  / `embedding-events` / ingestion topics, never meeting events. `mytheus-realtime`'s
  transcription-service consumes `meeting-room-events` only
  (`meeting_room_consumer.py:63-64`, group `transcription-service`).
- **No liveness wiring for the transcription consumer** — disclosed honestly at
  `mytheus-realtime/.../main.py:15-19`: if the consumer task dies, `_log_consumer_exit`
  (`main.py:48-62`) logs loudly but `/healthz` (`main.py:114-116`) still returns ok, so a liveness
  probe would not restart the pod.
- **No S3 client anywhere.** `storage.py` is pure string computation (`storage.py:8-16` says so
  explicitly and gives a defensible reason: LiveKit egress writes the object itself). But with no
  `start_egress` caller supplying that key as an output path, nothing on either side exists.

---

## THE MEETING END-TO-END FLOW

Traced from "user clicks join" backwards to meeting creation and forwards to "a transcript row
exists". Each hop marked WIRED / BROKEN / ABSENT with evidence.

### Step 1 — User creates a meeting — **WIRED**
`POST /api/v1/meetings` → `meetings.py:154`. Inserts the row, validates each participant is visible
under RLS (`meetings.py:193-201`), inserts `meeting_participants`, enqueues
`meeting-events{event_type: "created"}` to the outbox (`meetings.py:213-227`), writes the audit row
(`meetings.py:229-238`). All in one transaction (`tenant_session` yields inside `session.begin()`).

### Step 2 — Outbox relay publishes `meeting-events` — **WIRED**
`libs/shared/relay.py`, exercised by `mytheus-backend/tests/test_relay.py` including the
dead-letter-after-max-attempts case.

### Step 3 — meeting-service consumer applies `created` — **WIRED**
`consumer.py:184-187` → `rooms.ensure_meeting_room` (`rooms.py:62-96`) upserts the `meeting_rooms`
row with `livekit_room = meeting-{meeting_id}` and returns `needs_livekit_sync = True` on first
insert. `record_consumed` writes the inbox row (`consumer.py:195`). Proven by
`mytheus-backend/tests/test_meeting_events_consumer.py` against a real Postgres.

### Step 4 — `ensure_room` writes `{"org_id": ...}` as LiveKit **room** metadata — **BROKEN**
Path: `consumer.py:199-208` (`_fire_and_forget_sync`) → `rooms.sync_livekit_metadata:135` →
`transport.ensure_room`. The `LiveKitTransport` implementation (`transport.py:74-110`) is correct in
principle — two calls, `CreateRoom` then `UpdateRoomMetadata`, because the former is idempotent but
does not update an existing room's metadata while the latter requires the room to exist; 5s explicit
timeout (`transport.py:37,102`) rather than the SDK's 60s-per-call default.

**But** `select_transport()` (`transport.py:259-264`) returns `FakeTransport` because
`LIVEKIT_API_KEY` is unset in every environment. `FakeTransport.ensure_room` (`transport.py:178-180`)
writes the metadata to an in-process dict, not to LiveKit. The row is then marked
`livekit_metadata_synced_at = now()` (`rooms.py:148-150`) — i.e. **the row is marked synced when
nothing was synced**, and the self-heal-on-next-mint retry will never fire once a fake sync has
"succeeded". This matters for step 7: without real room metadata, `_resolve_org_id` cannot resolve a
tenant for any real webhook.

### Step 5 — User clicks join; token is minted — **WIRED**
`useLiveKitRoom.ts:115` → `mintToken` (`:52-66`) → `POST /api/livekit-token`
(`mytheus-ui/src/app/api/livekit-token/route.ts:37`) → `api()` forwards the Clerk session →
api-gateway (`edge.py:161`) → `mint_token` (`routes.py:158`).

**Token-minting ownership — no split-brain.** This was checked carefully, since it was the sharpest
question in the brief:
- `mytheus-ui/package.json` declares `@livekit/components-react`, `@livekit/components-styles`,
  `livekit-client` — **no `livekit-server-sdk`**. A grep of `mytheus-ui/src` for `AccessToken` and
  `LIVEKIT_API_SECRET` returns nothing.
- `route.ts:11-14` states, and the code confirms, that the handler holds no LiveKit secret and only
  forwards to the backend.
- `route.ts:16-24` records a **deliberate documented-path correction**: Plan 3-4 §3.4 step 2 and
  `mytheus-ui`'s CLAUDE.md both say to call `POST /api/v1/meetings/:id/token`, which routes to
  core-api and does not mint tokens; the shipped route is `/api/meetings/{id}/token` on
  meeting-service (`routes.py:154`), which is what the handler actually calls (`route.ts:51`). The
  docs are the bug, not the code.

**Conclusion: `meeting-service` alone owns token minting.** The UI is a pure pass-through. Identity,
room naming and grants are therefore agreed on by construction rather than by two implementations
happening to match:
- identity = `str(user_id)`, the internal `users.id` (`transport.py:126`);
- room = `meeting-{meeting_id}` from the single helper (`rooms.py:51-53`), returned to the client in
  `TokenResponse.room` and used verbatim;
- grants = `room_join=True, can_publish=True, can_subscribe=True` (`transport.py:122`) — note these
  are **uniform for every role**: `role_in_meeting` is carried only in the token's participant
  metadata (`transport.py:131`), not reflected in the grants, so an `attendee` and a host receive
  identical publish rights.
- TTL = 600s (`routes.py:83`), with the client re-minting on every reconnect
  (`useLiveKitRoom.ts:9-11,115`).

### Step 6 — Browser connects to the SFU — **ABSENT in every environment**
`useLiveKitRoom.ts:120` — `room.connect(minted.livekit_url ?? configuredUrl, minted.token)`. But
`configuredUrl = process.env.NEXT_PUBLIC_LIVEKIT_URL` (`:69`) is unset everywhere, so the hook
short-circuits to `status: "unconfigured"` at `:70-72` and the effect returns immediately at `:85` —
it never even mints. The docstring (`useLiveKitRoom.ts:13-19`) states this explicitly. And the
server-side fallback is empty too: `TokenResponse.livekit_url` is
`os.environ.get("LIVEKIT_URL", "")` (`routes.py:215`), i.e. `""` with no LiveKit configured.

### Step 7 — LiveKit fires `participant_joined` webhook — **ABSENT**
`webhook.py:207` is a real, well-defended route (see IMPLEMENTED), and its logic is genuinely proven
by `mytheus-backend/tests/test_livekit_webhook.py` (12 tests, real Postgres). But no LiveKit
deployment exists to fire it, and per step 4 no real room metadata exists for `_resolve_org_id` to
read even if one did. The tests drive it via `FakeTransport.verify_webhook_result`
(`transport.py:170,235-241`), so the *routing and tenancy* logic is verified while *signature
verification against a real LiveKit HMAC* is not.

### Step 8 — `meeting-room-events{joined}` outbox row → relay → topic — **WIRED** (conditional on 7)
`webhook.py:238-252`, same transaction as the state change, idempotency key
`{room}:{event}:{participant_sid}`.

### Step 9 — transcription-service consumes `meeting-room-events` — **WIRED**
`mytheus-realtime/.../meeting_room_consumer.py:114-142` — Redis dedupe (`:126-132`, 24h TTL, key
`nexus:{org_id}:dedupe:meeting-room-events:{idempotency_key}`), then routes `joined`/`left` to
`SessionTracker` (`:137-140`). `recording_ready` is acknowledged and ignored (`:141-143`). Poison
messages, including a null-value tombstone that decodes to `None`, go to the DLQ rather than killing
the loop (`:168-176`). Offsets commit only after the effect.

### Step 10 — Open a transcription stream — **BROKEN by design**
`sessions.py:75-77` → `provider.open_stream`. `select_provider` (`providers/factory.py:21-23`)
returns `FakeProvider` (no `DEEPGRAM_API_KEY`). Even with a key,
`DeepgramProvider.open_stream` raises `ComplianceGateClosedError` at `providers/deepgram.py:112`
before any network I/O, because `COMPLIANCE_GATE_GREEN = False` (`compliance.py:39`) and condition 1
(signed Deepgram DPA/SCC on file) is a legal artifact no code change can satisfy.

### Step 11 — Audio reaches the stream — **ABSENT. This is the hard break.**
There is no audio path of any kind, and the gap is *architectural*, not merely credential-shaped:

1. `MeetingTransport.subscribe_room_audio` (`transport.py:56-58`) — the declared "handle for
   transcription tap". `LiveKitTransport`'s implementation raises `NotImplementedError`
   (`transport.py:139-141`). Nothing calls it.
2. `/internal/audio`, the LiveKit server-side audio-hook target
   (`mytheus-realtime/.../main.py:119-124`) — accepts the WebSocket and immediately closes it.
3. Most decisively: **the `TranscriptionStream` Protocol has no method that accepts audio.**
   `providers/base.py:43-48` declares exactly two members — `utterances()` and `close()`. There is no
   `send()`, `write()`, or `feed()`. Even if the hook at (2) were implemented and the tap at (1)
   existed, there is no interface through which bytes could be handed to a provider. This has to be
   designed, not just wired.

So no audio ever reaches Deepgram, and `_DeepgramStream.utterances()` (`providers/deepgram.py:40`)
would iterate a websocket that is never fed.

### Step 12 — Utterances → `transcript_chunks` row + `transcript-chunks` outbox — **WIRED, but only from fakes**
`sessions.py:_pump:106-131` drains `stream.utterances()` into
`persistence.process_utterance` (`persistence.py:79-98`): finals only (interims are dropped and can
never reach the outbox or an INSERT, `persistence.py:11`), redacted when
`tenant_settings.pii_redaction_enabled` (`redaction.py`), ONE `transcript_chunks` row and ONE outbox
row in one transaction, dedup-gated on `chunk:{chunk_id}`. This machinery is real. Its only inputs
today are `FakeProvider.default_script()` (`providers/fake.py:26-70`) or the dev seeder
(`devseed.py`, refuses to run without `TRANSCRIPTION_DEV_SEED=1`, `devseed.py:34-41`).

### Step 13 — UI reads the transcript — **WIRED**
`GET /api/v1/meetings/{id}/transcript` (`transcript.py:95`) → `mytheus-ui/src/lib/api/transcript.ts`
(chunks + speakers fetched in parallel, `:73-79`) → `TranscriptView.tsx` / `SpeakerMapping.tsx`.

### Step 14 — Recording lands in `nexus-recordings` — **ABSENT**
No `start_egress` caller (step: STUB), no bucket, no LiveKit egress output-path configuration, so
`webhook._apply_egress_ended` (`webhook.py:140`) is unreachable and `meeting_rooms.recording_ref`
stays NULL forever.

### Net assessment of the flow
**A `transcript_chunks` row can only come into existence today via `devseed.py` or a `FakeProvider`
script.** There is no path from a human speaking to a persisted transcript. Two segments of the ring
are missing: recording never starts, and audio never moves from a room to a transcriber. The second
of those requires an interface that does not exist yet (`TranscriptionStream` cannot accept audio),
so it is not unblocked merely by obtaining a LiveKit account.

---

## DEFECTS FOUND

### D-1 — `assign_speaker` accepts another tenant's `user_id` (cross-tenant write) — HIGH
`mytheus-backend/services/core_api/routes/speakers.py:97-133`.

The route relies **solely** on the FK `fk_meeting_speakers_user_id_users`
(migration `25e7cede5672:108`, `["user_id"] → ["users.id"]`, no org component) to keep the assigned
user inside the tenant, catching `IntegrityError` and translating it to 422 with the message
"user_id does not name a user in this organization" (`speakers.py:123-133`).

But `meetings.py:186-192` documents, from a bug that actually shipped, that this exact mechanism does
not work:

> PostgreSQL runs referential-integrity checks as the referenced table's owner with RLS bypassed, so
> `fk_meeting_participants_user_id_users` resolves another tenant's `users` row even though this
> transaction cannot SELECT it, and `meeting_participants`' WITH CHECK passes because the `org_id`
> written is the caller's own. This comment previously asserted the opposite and was wrong: the route
> returned 201 and persisted a row referencing another tenant's user.

`create_meeting` was fixed by adding an explicit `SELECT 1 FROM users WHERE id = :id` probe under RLS
(`meetings.py:202-203`). `assign_speaker` never received the same fix — the `org_id` it writes is
`ctx.org_id` (the caller's own, `speakers.py:113`), so the WITH CHECK passes and the FK resolves
cross-tenant. Result: a caller can bind an arbitrary other-tenant `users.id` into their own
`meeting_speakers.user_id`.

`mytheus-backend/tests/test_meeting_speakers_api.py` has no test for this: `:205`
(`test_unknown_user_id_is_a_422_not_a_500`) uses a nonexistent id, and `:345` covers a cross-tenant
*meeting*, not a cross-tenant *user in the body*.

**Fix shape:** mirror `meetings.py:202-203` — probe `SELECT 1 FROM users WHERE id = :id` under RLS
before the upsert, and raise the same tenant-confidential error. (Not applied; this is a read-only
audit.)

### D-2 — `meetings.status` and `meeting_rooms.status` never reconcile — MEDIUM
See PARTIAL. Nothing sets `meetings.status = 'live'`. A meeting with people actively in the room
reads as `'scheduled'` to every consumer of `GET /meetings` and `GET /meetings/{id}`.

### D-3 — Consent gate has no off-ramp — MEDIUM (latent)
See MISSING. `consent_required = true` makes a tenant's meetings permanently unjoinable. Latent only
because the flag defaults false and no route sets it either.

### D-4 — Fake sync poisons the real self-heal — LOW (but will bite at cutover)
`rooms.py:148-150` marks `livekit_metadata_synced_at = now()` after `FakeTransport.ensure_room`
"succeeds". The retry-on-next-mint mechanism keys off that column being NULL (`rooms.py:95`). So every
room established while running against the fake is permanently marked synced; when real LiveKit
credentials are configured later, those rooms will never get their real metadata written, and their
webhooks will fail `_resolve_org_id` with "room metadata is absent" (`webhook.py:118`). Recovery
requires a manual `UPDATE meeting_rooms SET livekit_metadata_synced_at = NULL`.

---

## TENANT SCOPING

Strong throughout, with one exception (D-1).

- Every query in `meeting_service` runs under `set_rls(session, ctx.org_id)` (`routes.py:171`,
  `webhook.py:302`, `consumer.py:180`, `rooms.py:149`).
- Existence checks deliberately carry **no** hand-written `org_id` predicate (`routes.py:107-118`,
  `meetings.py:129-140`, `speakers.py:34-44`, `transcript.py:75-92`) so a cross-tenant id is genuinely
  invisible rather than excluded by a predicate a future route could forget.
- `rooms.py` writes carry explicit `org_id` predicates *in addition to* RLS (`rooms.py:80,109,168`) —
  belt and braces, correct.
- `meeting_rooms` has `FORCE ROW LEVEL SECURITY`, verified by
  `mytheus-backend/tests/test_meeting_rooms_rls.py:107` and the canonical policy shape at `:126`;
  `:85` proves it fails closed with no GUC set.
- `meeting_speakers` is force-RLS'd at migration `25e7cede5672:124`.
- The webhook resolves the tenant **before** setting `app.current_org`, via
  `libs.shared.org_resolver.resolve_org_by_id` on the un-RLS'd `organizations` table
  (`webhook.py:107-137`) — the sanctioned second reader. Correct ordering: no tenant is established
  until resolution succeeds.
- `check_participant` explicitly ignores the `X-Org-Id` header realtime-gateway sends and derives
  `org_id` only from the re-verified bearer token (`meetings.py:325-346`), matching CLAUDE.md's "the
  gateway is a DDoS shield, not a trust delegation".
- `test_meeting_service_org_id_boundary.py:46` statically asserts no Pydantic model in
  meeting-service declares an `org_id` field, and `:57` asserts the scan actually finds the response
  model (a meta-test against a vacuously-passing scan — good practice).

---

## TESTS — real, not smoke

All backend tests run against a real Postgres with real migrations and real RLS role separation
(request-path role `mytheus_app` vs bootstrap role), with `verify_token` overridden so no live Clerk
is needed and everything downstream — Clerk-org resolution, `app.current_org`, RLS, the actual route
body — is real.

| file | tests | LOC | nature |
|---|---|---|---|
| `tests/test_meeting_token_route.py` | 12 | 395 | all 6 acceptance cases: mint, 403 non-participant, cross-tenant 404, consent-off mint, 428, consent-on mint; plus the repeated-mint deadlock regression (`:43-44`) |
| `tests/test_livekit_webhook.py` | 12 | 443 | all 6 fail-closed metadata cases, redelivery no-op, bad signature writes nothing, `joined` activates a pending room, `left` does not reactivate, unhandled type is a 200 no-op, and a full `ensure_room` → webhook round trip (`:205`) |
| `tests/test_meeting_events_consumer.py` | 15 | 562 | created/updated/ended, dedupe, DLQ routing via a fake producer |
| `tests/test_meetings_api.py` | 17 | 414 | CRUD, pagination, cross-tenant, incl. `test_another_tenants_user_cannot_be_named_as_a_participant` |
| `tests/test_meeting_speakers_api.py` | 7 | 416 | PATCH create/flip, 422, cross-tenant meeting 404, RLS isolation, and the DB trigger tested against a live DB (`:218`, `:282`) |
| `tests/test_meeting_rooms_rls.py` | 4 | 151 | isolation, fail-closed with no GUC, `relforcerowsecurity`, canonical policy shape |
| `tests/test_participant_check_api.py` | 11 | 290 | incl. the RLS-vs-row-existence distinction on the `users` JOIN |
| `tests/test_meeting_transport.py` | 5 | 97 | `FakeTransport` only |

**Not covered, and honestly disclaimed in-source rather than papered over:**
`LiveKitTransport`'s three real SDK methods (`transport.py:66-68` — "no test here claims coverage of
the real SDK calls below"), `DeepgramProvider` end-to-end (`providers/deepgram.py:8-9` — "that is
BLOCKED, not green-against-a-fake"), any real HMAC signature verification, any real audio, and D-1's
cross-tenant-user case (a genuine coverage gap, not a disclosed one).

`tests/test_livekit_import_boundary.py` structurally enforces that `transport.py:31` is the only
import of the LiveKit SDK in the codebase — ADR-0003's swappability boundary is machine-checked, not
a review convention.

---

## OUTBOUND DEPENDENCIES

- **Postgres** — `meetings`, `meeting_participants`, `meeting_rooms`, `meeting_speakers`,
  `tenant_settings`, `organizations` (via `org_resolver`), `inbox`, `outbox`, `audit_logs`,
  `transcript_chunks` (read-only, from core-api).
- **Redis** — webhook dedupe fast path (`webhook.py:293`); consumer dedupe (`consumer.py:176`);
  transcription-service dedupe + caption pub/sub.
- **Redpanda + Schema Registry** — consumes `meeting-events`, produces `meeting-room-events` (via
  outbox + relay), DLQs to `meeting-events.dlq` / `meeting-room-events.dlq`.
- **LiveKit Cloud** — `livekit-api` SDK, imported at exactly one place (`transport.py:31`).
- **S3 `nexus-recordings`** — named in `storage.py:29`; no client, no call, no bucket.
- **Clerk** — via `libs.shared.authz` / `libs.shared.auth`.

## INBOUND

- `api-gateway` → `/api/meetings/*` and `/livekit/webhook` (`edge.py:161-162`).
- `mytheus-ui` → `/api/livekit-token` (own route handler) → gateway → meeting-service token mint.
- LiveKit Cloud → `POST /livekit/webhook` (unauthenticated by URL; HMAC is the only gate —
  `webhook.py:3-8` says so plainly).
- `realtime-gateway` → core-api `GET /api/v1/meetings/:id/participants/:clerkUserId`
  (`mytheus-realtime/services/realtime-gateway/src/coreApiClient.ts:117`).
- `transcription-service` ← `meeting-room-events` topic.

## CONFIG/ENV REQUIRED

**meeting-service:** `DATABASE_URL`, `REDIS_URL` (defaults `redis://localhost:6379`,
`webhook.py:293`), `REDPANDA_BROKERS` (default `localhost:9092`), `SCHEMA_REGISTRY_URL` (default
`http://localhost:8081`), `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`. `LIVEKIT_API_KEY`
is the selector: unset → `FakeTransport` (`transport.py:259-261`); set → the other two are read with
`os.environ[...]` and will `KeyError` if missing (`transport.py:262-263`).

**mytheus-ui:** `NEXT_PUBLIC_LIVEKIT_URL` — unset in every environment, which makes
`useLiveKitRoom` inert.

**transcription-service:** `DATABASE_URL`, `REDIS_URL`, `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`,
`DEEPGRAM_API_KEY` (irrelevant while the compliance gate is closed), `TRANSCRIPTION_DEV_SEED=1` for
the seeder.

**UNVERIFIED:** whether any deployed environment supplies any LiveKit variable. I found no `.env`,
compose file, or deploy manifest in the audited trees that sets them, and every in-source comment
(`transport.py:10-13`, `useLiveKitRoom.ts:13-15`, `rooms.py`, the plan docs) states no LiveKit account
exists for the program. Treat "LiveKit is unconfigured everywhere" as strongly indicated but not
directly confirmed against a running environment.

---

## VERDICT

The meeting-service code that exists is unusually well-built: the tenant-isolation reasoning
(404-before-403, no hand-written `org_id` predicates, fail-closed webhook tenant resolution), the
transaction-ordering deadlock fix, and the poison-message/DLQ discipline are all real, correctly
implemented, and tested against a live Postgres rather than mocked — and the source docstrings are
consistently honest about what is not exercised. But the lifecycle is a ring with two segments
missing: nothing ever starts a recording (`start_egress` raises, no caller), and nothing ever moves
audio from a room to a transcriber — `subscribe_room_audio` raises, `/internal/audio` accepts and
immediately closes, and the `TranscriptionStream` Protocol has no method that accepts audio at all,
so that gap is architectural rather than credential-shaped and will not close when a LiveKit account
arrives. Three things are worth fixing regardless of LiveKit: `assign_speaker` accepts a cross-tenant
`user_id` because it trusts an FK that `create_meeting` already proved does not enforce tenancy (D-1,
a real cross-tenant write); `meetings.status` is never reconciled with `meeting_rooms.status` (D-2);
and rooms synced against `FakeTransport` are permanently marked synced, which will silently break
webhook tenant resolution at real-LiveKit cutover (D-4).

**On the split-brain question specifically: there is none.** `meeting-service` alone mints LiveKit
tokens (`routes.py:158` → `transport.py:112`). `mytheus-ui/src/app/api/livekit-token/route.ts` is a
pure pass-through holding no secret, `mytheus-ui` has no `livekit-server-sdk` dependency and no
`AccessToken` import, and identity (`users.id`), room name (`meeting-{meeting_id}`) and grants come
from one implementation, so there is nothing for the two sides to disagree about. The only
UI/backend divergence found is documentary: the plan and `mytheus-ui`'s CLAUDE.md both name a token
path (`/api/v1/meetings/:id/token`) that routes to core-api and does not mint; the UI follows the
shipped path (`/api/meetings/:id/token`) and records the docs as the bug at `route.ts:16-24`. The UI
is right.
