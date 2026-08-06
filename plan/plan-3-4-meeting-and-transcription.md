# Plan 3-4 — Meeting Platform & Real-time Transcription  (Meeting Platform)

> Capture a live meeting end-to-end: LiveKit behind a transport adapter (`meeting-service`), Socket.io fan-out + presence + chat (`realtime-gateway`), and the Deepgram transcription pipeline persisting FINALS ONLY to `transcript_chunks` — with the compliance gate GREEN before W9.

---

## 0. Band at a glance

- **Phases / weeks:** **P3 Meeting Platform Integration (W6-8)** then **P4 Real-time & Transcription (W9-12)**. P3 stands up media transport + the non-AV fan-out plane; P4 adds the audio→Deepgram→`transcript_chunks` pipeline behind a compliance gate.
- **Goal:** A participant joins a real LiveKit room from `web-app` (and `mytheus-mobile`), sees other participants and presence, chats in-meeting, and — once P4 lands — sees live captions streaming under the 2 s budget while diarized finals persist to `transcript_chunks` and flow onto the `transcript-chunks` topic. No agents yet (that is Plan 5+); this band delivers the *capture* substrate they consume.
- **Repos & folders touched:**
  - `mytheus-backend` — **`meeting-service`** (LiveKit transport adapter, token mint, webhook, `meeting_rooms`, `nexus-recordings`); **`core-api`** (`chat_messages` persistence + `POST /chat`, `meeting_speakers` mapping + `meeting-events` producer wiring); `libs/shared` (reuse outbox/inbox/envelope/tenancy); `migrations/`.
  - `mytheus-realtime` — **primary, currently docs-only.** New `services/realtime-gateway/` (Node + Socket.io), `services/transcription-service/` (Python FastAPI WS proxy), `proto/`, `libs/`, `docker-compose.yml`.
  - `mytheus-events` — Avro subjects `transcript-chunks-value`, `meeting-room-events-value`, `presence-events-value` (+ their `.dlq`), registered BACKWARD.
  - `mytheus-ui` (`web-app`) — meeting room (LiveKit video grid), live-captions view, transcript view, speaker-mapping UI, in-meeting chat + presence.
  - `mytheus-mobile` — realign `live-session` + `start-session` screens onto real LiveKit media + `realtime-gateway` Socket.io; real meeting clock; mic capture + permissions.
- **Demoable at the end:**
  - **End of P3:** two browsers (+ one phone) join the same `meeting_id` LiveKit room through a `meeting-service`-minted token; video grid renders; presence list updates on join/leave; chat messages fan out across `realtime-gateway` pods and persist to `chat_messages`; a `recording.ready` webhook lands an egress ref in `meeting_rooms` and an object in `nexus-recordings/{org_id}/{meeting_id}/`. Staging deploy fires on merge to `main`.
  - **End of P4:** the same meeting now shows **live captions** (Deepgram interims → finals) under p95 < 2 s; **finals only** land in `transcript_chunks` and on the `transcript-chunks` topic (Avro `TranscriptChunk`); the transcript view replays ordered chunks; the speaker-mapping UI assigns a diarized `speaker_id` → `user_id` in `meeting_speakers`; the **compliance gate** is demonstrably green (Deepgram DPA/SCC on file, at-rest PII redaction node firing when `tenant_settings.pii_redaction_enabled`, `transcript_chunks` and the Deepgram endpoint pinned to `data_residency.region`).

---

## 1. Entry preconditions  (must be GREEN before starting)

Plan 3-4 is **not** a greenfield start. It assumes Plan 0 (foundations) and Plan 1-2 (identity + tenancy + core CRUD + meetings domain) shipped. Concretely, before W6:

1. **Identity is Clerk (RESOLVED DECISION 1).** Clerk JWKS verification lives in `mytheus-backend/libs/shared` (the Supabase-Auth drift is reverted: no `custom_access_token_hook`, no `auth.users → public.users` trigger; `CLERK_*` env restored; Svix webhook user-sync handler implemented in `identity-tenant-service`). `api-gateway` extracts `org_id` **once at the edge** from the Clerk JWT `org` claim and fails closed. **If this is not done, STOP — fix it in the identity band, not here.** Every `meeting-service` token mint and every `realtime-gateway` handshake depends on a trustworthy edge-injected `org_id`.
2. **`organizations`, `users`, `meetings`, `meeting_participants` tables exist** with RLS FORCE on `org_id`, created by Alembic in Plan 1-2. `meetings` carries `livekit_room`, `status`, `scheduled_start/end`, `created_by`; `meeting_participants` PK `(meeting_id, user_id)` carries `role_in_meeting`. Token mint reads `meeting_participants`; if the table is absent, P3 cannot start.
3. **`tenant_settings` and `data_residency` exist** (owned by `identity-tenant-service`, Plan 1-2). `tenant_settings.pii_redaction_enabled` and `data_residency.region ∈ {us, eu, ap}` / `data_residency.object_bucket_region` are readable. P4's compliance gate enforces these; if the rows do not exist, the gate cannot be green.
4. **At-least-once + idempotency is live:** transactional outbox on produce, inbox + `nexus:{org_id}:dedupe:{topic}:{idempotency_key}` (24h TTL) on consume, per-topic DLQ. Avro + Schema Registry, **BACKWARD** compat, subject `{topic}-value`, partition key `org_id` (composite `{org_id}:{meeting_id}` for `transcript-chunks`, [ADR-0017](../adr/0017-transcript-chunks-composite-partition-key.md); see §8). The `EventEnvelopeHeader` (`event_id`, `idempotency_key`, `org_id`, `schema_version`, `producer`, `occurred_at`, `correlation_id`, `causation_id`) is the locked header on every value.
5. **`api-gateway` already proxies REST to `bff`/services and upgrades WSS** straight to the real-time plane (`GW <-->|WSS| RTGW`). The WSS leg bypasses `bff`. Connection-hash / `sid` affinity at the ingress is configurable.
6. **`meeting-events` is produced by `core-api`** (created/updated/ended) from Plan 1-2. This band wires the **consumer** in `meeting-service` (open/close rooms) — if the producer is missing, land it first.
7. **Capacity anchors are the sizing target**, never the `2 orgs × 10 users × 5 meetings` seeder: `A5` = 20,000 peak meetings, `A6` = 100,000 peak participants, `A8` = 5.0 effective occupancy, `A9` = 45 min, `D1` = 216M Deepgram min/mo, `T2` = 360 finals/meeting, `A12` = 8 finals/min.

> WARNING — **The compliance gate is a hard blocker for P4, not a checklist item.** Deepgram is a US-incorporated subprocessor that receives raw C4 conversational audio. No Deepgram stream may open against real tenant audio until §6's gate is green. P3 (media + non-AV fan-out, no ASR) may proceed in parallel with the gate paperwork; P4 may not.

---

## 2. Canonical names introduced in this band

This band introduces **no new** service, table, topic, collection, Redis-key, or AGE-label names. It materializes already-frozen canon. Names used verbatim:

- **Services:** `meeting-service`, `realtime-gateway`, `transcription-service`, `core-api`, `api-gateway`, `bff`, `web-app`, `identity-tenant-service`.
- **Tables:** `meeting_rooms` (owner `meeting-service`), `transcript_chunks` (owner `transcription-service`), `meeting_speakers` (owner `core-api`), `chat_messages` (owner `core-api`), `meetings` / `meeting_participants` (owner `core-api`), `tenant_settings` / `data_residency` (owner `identity-tenant-service`, read here).
- **Topics:** `transcript-chunks`, `meeting-room-events`, `presence-events`, `meeting-events` (each paired with `<topic>.dlq`).
- **Frozen Avro schema (canon):** `TranscriptChunk`. **Net-new (not frozen canon) schema names** introduced by [07](../07-event-architecture.md): `MeetingRoomEvent`, `PresenceEvent` — usable but flagged.
- **Object bucket:** `nexus-recordings` (key prefix `{org_id}/{meeting_id}/`).
- **Redis keys (all `nexus:{org_id}:…`):** `nexus:{org_id}:pubsub:meeting:{meeting_id}` (Socket.io fan-out channel), `nexus:{org_id}:presence:{meeting_id}` (presence set), `nexus:{org_id}:dedupe:{topic}:{idempotency_key}` (inbox cache), `nexus:{org_id}:dedupe:livekit:{room}:{event}:{participant_sid}` (webhook dedupe).
- **Adapter / interface names:** `MeetingTransport` (Protocol), `LiveKitTransport` (concrete), `TranscriptionProvider` (Protocol), `DeepgramProvider` (concrete).
- **Capacity symbols (imported, never re-derived):** `A5`, `A6`, `A8`, `A9`, `A12`, `D1`, `T2`.

---

## 3. Work breakdown (ordered, grouped by phase then component)

Ordering is a precondition chain. **P3 before P4** because transcription needs a live room emitting `meeting-room-events` and a server-side audio hook to tap. Within P3: **`meeting-service` token mint + webhooks before `realtime-gateway`** (the gateway re-mints tokens on reconnect and joins rooms that only exist once mint works), and **`realtime-gateway` core fan-out before `web-app` room UI** (the UI consumes the socket contract). Within P4: **the compliance gate before any Deepgram stream**, then **`transcription-service` finals-only persistence before captions wiring**, then **`web-app` transcript + speaker-mapping**.

---

### PHASE P3 — Meeting Platform Integration (W6-8)

#### 3.1 `meeting-service` — LiveKit transport adapter, token mint, webhooks, `meeting_rooms`, `nexus-recordings`

**Repo / folder:** `mytheus-backend/services/meeting-service/` (today: a stub `main.py` that boots FastAPI with no routes).

**Steps:**

1. **Add LiveKit + object-storage deps** to the workspace (uv/Poetry):
   ```bash
   cd mytheus-backend
   uv add livekit-api boto3
   # livekit-api: server SDK (AccessToken minting, webhook receiver, room service)
   ```
2. **Add the migration for `meeting_rooms`** (data-model edit lands first per the repo rule; the table is already canon, so this is a materialization, not a new name):
   ```bash
   uv run alembic revision -m "feat(meeting-service): add meeting_rooms"
   ```
   Migration body (RLS FORCE, canonical policy):
   ```sql
   CREATE TABLE meeting_rooms (
     id              uuid        NOT NULL DEFAULT gen_random_uuid(),
     org_id          uuid        NOT NULL REFERENCES organizations(id),
     meeting_id      uuid        NOT NULL REFERENCES meetings(id),
     livekit_room    text        NOT NULL,              -- the LiveKit room name
     status          text        NOT NULL DEFAULT 'pending', -- pending|active|ended
     egress_id       text,                              -- LiveKit egress id (recording)
     recording_ref   text,                              -- nexus-recordings object key
     created_at      timestamptz NOT NULL DEFAULT now(),
     updated_at      timestamptz NOT NULL DEFAULT now(),
     deleted_at      timestamptz,
     PRIMARY KEY (id),
     UNIQUE (org_id, meeting_id),
     UNIQUE (org_id, livekit_room)
   );
   ALTER TABLE meeting_rooms ENABLE ROW LEVEL SECURITY;
   ALTER TABLE meeting_rooms FORCE ROW LEVEL SECURITY;
   CREATE POLICY tenant_isolation ON meeting_rooms
     USING      (org_id = current_setting('app.current_org')::uuid)
     WITH CHECK (org_id = current_setting('app.current_org')::uuid);
   ```
3. **Write the transport adapter — the ONLY place the LiveKit SDK is imported** (`services/meeting_service/transport.py`). Matches [ADR-0003 §Decision.2](../adr/0003-meeting-sdk-livekit.md) verbatim:
   ```python
   from typing import Protocol
   from uuid import UUID
   from livekit import api  # the ONLY import of the LiveKit SDK in the codebase

   class MeetingTransport(Protocol):
       async def mint_join_token(self, *, org_id: UUID, meeting_id: UUID, user_id: UUID,
                                 room: str, role_in_meeting: str, ttl_seconds: int = 600) -> str: ...
       async def start_egress(self, *, org_id: UUID, meeting_id: UUID, room: str) -> str: ...
       async def subscribe_room_audio(self, *, org_id: UUID, room: str): ...   # handle for transcription tap
       def verify_webhook(self, *, body: bytes, auth_header: str): ...

   class LiveKitTransport:
       """Concrete LiveKit binding. Swappable per Tech Decisions 2.7."""
       def __init__(self, url: str, key: str, secret: str):
           self._url, self._key, self._secret = url, key, secret

       async def mint_join_token(self, *, org_id, meeting_id, user_id, room, role_in_meeting, ttl_seconds=600) -> str:
           grant = api.VideoGrants(room=room, room_join=True, can_publish=True, can_subscribe=True)
           tok = (api.AccessToken(self._key, self._secret)
                  .with_identity(str(user_id))               # one identity == user_id
                  .with_grants(grant)
                  .with_ttl(ttl_seconds)                     # 600 s default; reconnect re-mints
                  .with_metadata(f'{{"org_id":"{org_id}","role":"{role_in_meeting}"}}'))
           return tok.to_jwt()

       def verify_webhook(self, *, body, auth_header):
           return api.WebhookReceiver(self._key, self._secret).receive(body.decode(), auth_header)
   ```
   No other service (`core-api`, `transcription-service`, the agent tier) imports the LiveKit SDK — they call this interface or consume `meeting-room-events`.
4. **Token mint route — `POST /api/meetings/:id/token`** (`services/meeting_service/routes.py`). Server-mediated, participant-verified, deny-by-default, p95 < 250 ms ([10 §2.3](../10-realtime-and-transcription.md)):
   ```python
   @router.post("/api/meetings/{meeting_id}/token")
   async def mint_token(meeting_id: UUID, ctx: TenantCtx = Depends(tenant_ctx)):
       # ctx.org_id comes ONLY from the Clerk-verified JWT via api-gateway; never from the body.
       async with tenant_session(ctx.org_id) as db:        # SET LOCAL app.current_org
           row = await db.scalar(select(MeetingParticipant).where(
               MeetingParticipant.meeting_id == meeting_id,
               MeetingParticipant.user_id == ctx.user_id))   # RLS scopes org_id
           if row is None:
               raise Problem(status=403, code="NOT_A_PARTICIPANT")   # deny-by-default
           room = await ensure_meeting_room(db, ctx.org_id, meeting_id)  # upsert meeting_rooms
       token = await transport.mint_join_token(
           org_id=ctx.org_id, meeting_id=meeting_id, user_id=ctx.user_id,
           room=room.livekit_room, role_in_meeting=row.role_in_meeting, ttl_seconds=600)
       return {"token": token, "room": room.livekit_room, "livekit_url": settings.LIVEKIT_URL}
   ```
   Bind a tight `statement_timeout` (2 s) on this path per [05 §3.5](../05-data-architecture.md) so the token-mint SLO holds.
5. **Webhook route — `POST /livekit/webhook`** (`services/meeting_service/webhook.py`). HMAC-verified through the adapter, deduped on the at-least-once inbox key `{room}:{event}:{participant_sid}` so a redelivered `recording.ready` cannot double-write:
   ```python
   @router.post("/livekit/webhook")
   async def livekit_webhook(request: Request):
       body = await request.body()
       evt = transport.verify_webhook(body=body, auth_header=request.headers["Authorization"])
       org_id = org_id_from_room_metadata(evt)            # resolve tenant from room metadata
       dedupe_key = f"{evt.room.name}:{evt.event}:{getattr(evt.participant, 'sid', '-')}"
       if await inbox_seen(org_id, "livekit", dedupe_key): # nexus:{org_id}:dedupe:livekit:{key}
           return Response(status_code=200)
       async with tenant_session(org_id) as db, outbox(db) as out:
           if evt.event == "room_started":
               await mark_meeting_active(db, org_id, evt); enqueue_room_event(out, "room.created", evt)
           elif evt.event == "participant_joined":
               enqueue_room_event(out, "participant.joined", evt)   # first joiner → transcription opens stream
           elif evt.event == "egress_ended":   # recording.ready
               ref = f"{org_id}/{meeting_id_of(evt)}/{evt.egress_info.egress_id}.mp4"
               await persist_recording_ref(db, org_id, evt, ref)    # meeting_rooms.recording_ref
               enqueue_room_event(out, "recording.ready", evt)
       return Response(status_code=200)
   ```
   The webhook handler verifies → dedupes → normalizes → returns `2xx` fast; the `meeting-room-events` emit rides the outbox in the same transaction as the `meeting_rooms` write.
6. **Consume `meeting-events`** (produced by `core-api`) to open/close rooms: on `meeting.created` pre-create the `meeting_rooms` row + LiveKit room name; on `meeting.ended` mark `status='ended'` and trigger egress stop. Re-assert `org_id` from the header; dedupe via inbox.
7. **`nexus-recordings` bucket.** Create the region-pinned bucket (or per-region buckets) with a `{org_id}/{meeting_id}/` key convention and lifecycle (cold after 30 d; purge on RTBF per [05 §7](../05-data-architecture.md)). Wire `boto3` with `data_residency.object_bucket_region` so the recording lands in the tenant region. LiveKit egress writes directly to the bucket; `recording.ready` only persists the *reference*.
8. **Staging deploy on merge.** Add the `meeting-service` module to the CI matrix; on merge to `main`, build the image and deploy to the Railway/Render staging deployable (modular monolith — `meeting-service` is a module, not yet its own pod). Smoke: hit `POST /api/meetings/{id}/token` against a seeded participant.
9. **Env vars** (`mytheus-backend/.env`): `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `LIVEKIT_WEBHOOK_KEY` (HMAC), `NEXUS_RECORDINGS_BUCKET`, `AWS_*`/S3 creds.

**Canonical names:** `meeting-service`, `meeting_rooms`, `nexus-recordings`, `meeting-room-events`, `meeting-events`, `MeetingTransport`, `LiveKitTransport`, `POST /api/meetings/:id/token`, `POST /livekit/webhook`.

**Acceptance checks:**
- A seeded participant gets `{token, room, livekit_url}`; a non-participant gets `403 NOT_A_PARTICIPANT`. Token-mint p95 < 250 ms under a k6 smoke.
- A signed `egress_ended` webhook persists `meeting_rooms.recording_ref` and lands an object at `nexus-recordings/{org_id}/{meeting_id}/…`; a **redelivered** identical webhook is a no-op (inbox dedupe).
- `org_id` is never read from the request body anywhere in this service (grep test).
- RLS isolation test: tenant A cannot read tenant B's `meeting_rooms`.

---

#### 3.2 `realtime-gateway` — tenant-scoped rooms, Redis pub/sub fan-out, presence, chat relay, captions transport (Node + Socket.io)

**Repo / folder:** `mytheus-realtime/services/realtime-gateway/` (repo is docs-only today — scaffold it).

**Steps:**

1. **Scaffold the repo two-stack layout** ([mytheus-realtime CLAUDE.md](../../mytheus-realtime/CLAUDE.md)):
   ```bash
   cd mytheus-realtime
   mkdir -p services/realtime-gateway/src services/transcription-service/app proto libs
   ```
2. **Init the Node service:**
   ```bash
   cd services/realtime-gateway
   npm init -y
   npm i socket.io @socket.io/redis-adapter ioredis kafkajs @kafkajs/confluent-schema-registry jose pino
   npm i -D typescript tsx @types/node eslint prettier vitest
   ```
   - `socket.io` + `@socket.io/redis-adapter` + `ioredis` — fan-out backbone.
   - `kafkajs` + `@kafkajs/confluent-schema-registry` — produce `presence-events`, consume `agent-stream`/`agent-results`/`notifications` (the agent topics arrive in Plan 5+; wire the consumers now as no-op-tolerant).
   - `jose` — verify the Clerk JWT on the WS handshake (re-verify even though `api-gateway` pre-validated).
3. **Server skeleton** (`src/server.ts`) — Socket.io rooms keyed by `meeting_id`, Redis adapter for cross-pod fan-out, authz on join:
   ```ts
   import { Server } from "socket.io";
   import { createAdapter } from "@socket.io/redis-adapter";
   import Redis from "ioredis";
   import { verifyClerkJwt } from "../../../libs/auth";        // shared
   import { pubsubChannel, presenceKey } from "../../../libs/keys"; // nexus:{org_id}:...

   const pub = new Redis(process.env.REDIS_URL!);
   const sub = pub.duplicate();
   const io = new Server({ path: "/socket.io" });
   io.adapter(createAdapter(pub, sub));                          // cross-pod fan-out

   // Handshake: org_id ONLY from the Clerk JWT, never from the client payload. Fail closed.
   io.use(async (socket, next) => {
     try {
       const { orgId, userId } = await verifyClerkJwt(socket.handshake.auth.token);
       socket.data.orgId = orgId; socket.data.userId = userId; next();
     } catch { next(new Error("UNAUTHENTICATED")); }            // fail closed
   });

   io.on("connection", (socket) => {
     const { orgId, userId } = socket.data;
     socket.on("join", async ({ meetingId }) => {
       if (!(await isParticipant(orgId, meetingId, userId))) return; // deny-by-default
       socket.join(roomName(orgId, meetingId));                 // room scoped by org_id+meeting_id
       await pub.sadd(presenceKey(orgId, meetingId), userId);   // nexus:{org_id}:presence:{meeting_id}
       io.to(roomName(orgId, meetingId)).emit("presence", await presenceList(orgId, meetingId));
       await producePresenceEvent(orgId, meetingId, userId, "join"); // presence-events (Avro)
     });
     socket.on("chat", async (msg) => relayAndPersistChat(orgId, userId, msg)); // see step 6
     socket.on("disconnect", async () => { /* srem presence set; emit; presence-events leave */ });
   });
   io.listen(Number(process.env.WS_BIND_PORT) || 8080);
   ```
4. **Redis pub/sub fan-out** is the canonical `nexus:{org_id}:pubsub:meeting:{meeting_id}` channel (built by `libs/keys`, never hand-rolled). A bus event for meeting `M` lands on one pod, is published on that channel, and every pod holding a socket in room `M` emits to its local sockets ([10 §4.2](../10-realtime-and-transcription.md), [ADR-0012 §3](../adr/0012-realtime-transport-websocket-tier.md)). The Socket.io Redis adapter handles this once `io.adapter(createAdapter(...))` is set; the explicit channel name is used for any non-Socket.io publishers (e.g. the transcription caption bridge in P4).
5. **Presence** — live set in `nexus:{org_id}:presence:{meeting_id}` (Redis), and an analytics event on the `presence-events` topic (`PresenceEvent`, 12 partitions, 1h retention, key `org_id`) consumed by `core-api`. Presence is **ephemeral**: it lives in Redis, never a durable table.
6. **Chat relay + persistence boundary (READ THIS — canon ownership).** `chat_messages` is owned by **`core-api`**, NOT `realtime-gateway` ([03 §5](../03-container-architecture.md), [04 §3.5](../04-service-catalog.md)). `realtime-gateway` therefore **relays** chat over the socket for low latency, but the **durable write is a `core-api` call**:
   ```ts
   async function relayAndPersistChat(orgId, userId, msg) {
     io.to(roomName(orgId, msg.meetingId)).emit("chat", { ...msg, userId }); // fan out now
     await coreApi.post(`/api/v1/chat`, {                                    // durable write owned by core-api
       meeting_id: msg.meetingId, content: msg.content
     }, { idempotencyKey: msg.clientMsgId, bearer: socket-jwt });            // Idempotency-Key
   }
   ```
   The gateway never writes `chat_messages` directly. (`core-api` exposes `POST /chat` — see §3.3.) This keeps the one-writer-per-store invariant intact while still meeting the "chat persistence wired in this band" deliverable.
7. **Captions transport (stub in P3, live in P4).** Add a `caption` emit path now (subscribe to the `nexus:{org_id}:pubsub:meeting:{meeting_id}` channel for `caption` messages and emit to the room). It carries nothing until `transcription-service` publishes captions in P4.
8. **Sticky-connection / stateless-pod posture.** Configure the `api-gateway`/ingress for connection-hash (or `sid` cookie) affinity on `/socket.io`. The pod holds only ephemeral connection state; all room membership/presence is in Redis, so pod loss loses only live sockets (which reconnect with no data loss).
9. **Env vars** (`services/realtime-gateway`): `REDIS_URL`, `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`, `WS_BIND_HOST`, `WS_BIND_PORT`, `CLERK_JWKS_URL`, `CORE_API_URL`, `OTEL_EXPORTER_OTLP_ENDPOINT`.

**Canonical names:** `realtime-gateway`, `presence-events`, `nexus:{org_id}:pubsub:meeting:{meeting_id}`, `nexus:{org_id}:presence:{meeting_id}`, `chat_messages` (written via `core-api`), Socket.io `/v1` namespace, rooms keyed by `meeting_id`.

**Acceptance checks:**
- Two sockets for the same `meeting_id` landing on **different** gateway processes both receive a chat/presence event (cross-pod Redis fan-out test — the canonical hard test).
- A handshake with no/invalid Clerk JWT is rejected (fail closed); a socket cannot join a `meeting_id` room outside its JWT-resolved `org_id`.
- Presence set grows on join, shrinks on disconnect; a `presence-events` record is produced per transition, partitioned by `org_id`.
- A chat message appears in `chat_messages` (via `core-api`) exactly once even if the client retries with the same `Idempotency-Key`.

---

#### 3.3 `core-api` — `chat_messages` persistence + `POST /chat`, `meeting_speakers` mapping, `meeting-events` wiring

**Repo / folder:** `mytheus-backend/services/core-api/`.

**Steps:**

1. **Migration: `chat_messages`** (canon table, monthly range partition by `created_at` per [05 §3.2](../05-data-architecture.md)):
   ```sql
   CREATE TABLE chat_messages (
     id          uuid NOT NULL DEFAULT gen_random_uuid(),
     org_id      uuid NOT NULL REFERENCES organizations(id),
     meeting_id  uuid NOT NULL REFERENCES meetings(id),
     user_id     uuid NOT NULL REFERENCES users(id),
     content     text NOT NULL,
     created_at  timestamptz NOT NULL DEFAULT now(),
     deleted_at  timestamptz,
     PRIMARY KEY (id, created_at)
   ) PARTITION BY RANGE (created_at);
   -- + RLS FORCE + tenant_isolation policy; GIN FTS on content; index (org_id, meeting_id, created_at)
   ```
2. **`POST /chat`** (consumed by `realtime-gateway`'s relay and by `web-app`): accepts `Idempotency-Key`, writes one `chat_messages` row transaction-locally under `app.current_org`. Returns the canonical persisted resource.
3. **Migration: `meeting_speakers`** (canon owner = `core-api`; `transcription-service` only *emits* the diarized label, `core-api` persists the confirmed mapping per [04 §3.5](../04-service-catalog.md), R7):
   ```sql
   CREATE TABLE meeting_speakers (
     id          uuid NOT NULL DEFAULT gen_random_uuid(),
     org_id      uuid NOT NULL REFERENCES organizations(id),
     meeting_id  uuid NOT NULL REFERENCES meetings(id),
     speaker_id  text NOT NULL,                 -- diarized label from Deepgram
     user_id     uuid REFERENCES users(id),     -- nullable until mapped
     confidence  real,
     assignment  text NOT NULL DEFAULT 'suggested', -- manual|suggested (manual is source of truth)
     created_at  timestamptz NOT NULL DEFAULT now(),
     updated_at  timestamptz NOT NULL DEFAULT now(),
     PRIMARY KEY (id),
     UNIQUE (org_id, meeting_id, speaker_id)
   );
   -- + RLS FORCE + tenant_isolation policy
   ```
   Add `PATCH /meetings/:id/speakers/:speaker_id` to set `user_id` + `assignment='manual'` (the speaker-mapping UI target). Manual assignment is authoritative; the suggestion is confidence-scored advisory only.
4. **`meeting-events` producer** is assumed from Plan 1-2; verify it emits created/updated/ended via the outbox so `meeting-service` (rooms) and (later) `workflow-service` consume it.
5. **Consume `presence-events`** for analytics (read model only; no domain write).

**Canonical names:** `core-api`, `chat_messages`, `meeting_speakers`, `meeting-events`, `presence-events`, `/chat`, `/meetings/:id/speakers/:speaker_id`.

**Acceptance checks:**
- `POST /chat` persists once per `Idempotency-Key`; RLS isolation holds.
- `PATCH …/speakers/…` flips `assignment` to `manual` and pins `user_id`; a suggested mapping never overrides a manual one.

---

#### 3.4 `web-app` — LiveKit room UI (video grid), in-meeting chat + presence

**Repo / folder:** `mytheus-ui/` (Next.js 14 App Router).

**Steps:**

1. **Deps:** `npm i livekit-client @livekit/components-react socket.io-client`.
2. **Meeting room route** `(app)/meetings/[meetingId]/room/page.tsx`: on mount, `POST /api/v1/meetings/:id/token` (through `api-gateway` → `meeting-service`), receive `{token, room, livekit_url}`, connect the LiveKit room, render the participant video grid with `@livekit/components-react`. Re-mint the token on reconnect.
3. **Socket.io provider** wraps the room subtree only: connect to the `realtime-gateway` WSS origin (through `api-gateway`) on the `/v1` namespace, `emit("join", { meetingId })`, render presence from the `presence` event and chat from the `chat` event. Never open a second socket.
4. **Chat composer** posts over the socket (`emit("chat", …)` with a client-generated `clientMsgId` reused as `Idempotency-Key`); the message renders optimistically and settles when echoed.
5. **Captions panel placeholder** (filled in P4).

**Canonical names:** `web-app`, `realtime-gateway`, `meeting-service`, Socket.io `/v1`, rooms keyed by `meeting_id`.

**Acceptance checks:** two browsers join the same room, see each other's video, presence list, and chat in real time; refresh re-mints the token and re-hydrates without duplicate chat.

---

### PHASE P4 — Real-time & Transcription (W9-12)

#### 3.5 COMPLIANCE GATE — must be GREEN before any Deepgram stream opens (W9)

**Owner:** Security/Compliance + `transcription-service` lead. **Blocks:** all of §3.6-§3.8.

This gate operationalizes [12 §4-§6](../12-compliance-and-privacy.md). Three conditions, each with a verifiable artifact:

1. **Deepgram DPA + SCCs signed and on file.** Deepgram receives raw C4 conversational audio; a Data Processing Agreement plus Standard Contractual Clauses (Module 2/3) for any `eu` tenant whose audio touches a US endpoint, with the **zero-retention / no-model-training** vendor flag set. Artifact: signed DPA/SCC PDF in the compliance vault; the subprocessor register lists Deepgram with region + transfer mechanism.
2. **At-rest PII redaction node live in the transcription ingestion path.** Per [12 §5.1](../12-compliance-and-privacy.md), Deepgram receives audio (nothing to redact pre-ASR); redaction runs on the **finals, before persisting `transcript_chunks`**, gated by `tenant_settings.pii_redaction_enabled`. Layered detector: deterministic regex/validators (email, phone, IBAN, card via Luhn, government IDs) + a lightweight NER pass (run off the API tier), replacing spans with typed reversible placeholders (`‹EMAIL›`, `‹PHONE›`) keyed to a per-tenant token vault. Artifact: a test proving a final containing `jane@acme.com` lands in `transcript_chunks` as `‹EMAIL›` when the tenant has the flag on, and raw (in-region) when off.
   > Note: the canon "PII redaction pass" responsibility is listed on `ingestion-service` (document path). In THIS band there is no `ingestion-service` and no LLM tier yet; the live redaction here is the **transcript-at-rest** stage in `transcription-service`. The always-on LLM-egress redaction lands when the agent/RAG tier arrives (Plan 5+) — call it out as deferred, do not silently skip it.
3. **`data_residency` pin enforced.** `transcription-service` reads `data_residency.region` to (a) select the Deepgram **regional endpoint** (US/EU) and (b) ensure `transcript_chunks` and the `transcript-chunks` topic land in the tenant region (region-local Postgres + Redpanda). Artifact: a test that an `eu` tenant's stream opens against the EU Deepgram endpoint and its chunks persist to the EU cell.

**Acceptance checks (gate is green iff all pass):** signed DPA/SCC present; redaction test passes both on/off; residency test passes for `us` and `eu`; an audit-log entry records the redaction-toggle and any cross-border decision. **Until green, the Deepgram WSS client is feature-flagged off and `transcription-service` refuses to open a stream against real audio.**

---

#### 3.6 `transcription-service` — Deepgram WS proxy, interim-vs-final, FINALS-ONLY persistence, `transcript-chunks` topic

**Repo / folder:** `mytheus-realtime/services/transcription-service/` (Python FastAPI WS proxy).

**Steps:**

1. **Init the Python service:**
   ```bash
   cd mytheus-realtime/services/transcription-service
   python -m venv .venv && source .venv/bin/activate
   pip install fastapi "uvicorn[standard]" websockets deepgram-sdk sqlalchemy[asyncio] asyncpg \
               aiokafka fastavro confluent-kafka pydantic
   ```
2. **Internal WS endpoint for the LiveKit server-side audio hook** (`app/audio_in.py`): LiveKit's server-side track subscription delivers **mixed room audio** (PCM) over an internal WS bound to `TRANSCRIPTION_WS_BIND`. Validate the hook handshake with `LIVEKIT_API_KEY`/`SECRET`. The audio **never leaves the backend trust boundary** to reach Deepgram.
3. **Provider adapter** (`app/providers.py`) — same pattern as the transport adapter, so Deepgram stays swappable and a DPA-constrained tenant can be pinned to self-hosted Whisper later without touching callers ([10 §3.3](../10-realtime-and-transcription.md)):
   ```python
   class TranscriptionProvider(Protocol):
       async def open_stream(self, *, org_id, meeting_id, region, diarize=True): ...
   class DeepgramProvider:
       """LOCKED DEFAULT. Streaming WSS, diarization, interim+final. Key server-side."""
   # class WhisperProvider:  # optional cost tier (batch/Free-plan), self-hosted faster-whisper — NOT this band
   ```
4. **One diarized Deepgram session per meeting.** Open a single WSS session per `meeting_id` with `diarize=True`, interim+final, against the region endpoint from `data_residency.region`. The `DEEPGRAM_API_KEY` stays server-side (never sent to a client). This is the locked 6× cost lever — `D1` = 216M min/mo, not per-participant.
5. **Interim vs final split** (the hard rule):
   - **Interims** → live captions only. Emit to `realtime-gateway` (publish a `caption` message on `nexus:{org_id}:pubsub:meeting:{meeting_id}`). **Never persisted, never on the bus.**
   - **Finals (`is_final=true`)** → merge by speaker turn → run the at-rest PII redaction node (§3.5) → write **one** `transcript_chunks` row → produce **one** `TranscriptChunk` to the `transcript-chunks` topic → also emit as a caption.
6. **Migration: `transcript_chunks`** (canon owner = `transcription-service`; monthly range partition on `created_at`, local index `(meeting_id, start_ms)` per [05 §3.2](../05-data-architecture.md)):
   ```sql
   CREATE TABLE transcript_chunks (
     id          uuid        NOT NULL DEFAULT gen_random_uuid(),
     org_id      uuid        NOT NULL,
     meeting_id  uuid        NOT NULL,
     speaker_id  text        NOT NULL,             -- diarized label, NOT user_id
     start_ms    integer     NOT NULL,
     end_ms      integer     NOT NULL,
     text        text        NOT NULL,             -- redacted if pii_redaction_enabled
     is_final    boolean     NOT NULL DEFAULT true, -- ONLY finals persisted
     confidence  real        NOT NULL,
     created_at  timestamptz NOT NULL DEFAULT now(),
     deleted_at  timestamptz,
     PRIMARY KEY (id, created_at)
   ) PARTITION BY RANGE (created_at);
   -- monthly partition + local index (meeting_id, start_ms); RLS FORCE + tenant_isolation; pg_partman premake
   ```
   Use `pg_partman` (premake 3 months) and the nightly `workflow-service` job for partition maintenance (that job is wired in a later band; create the partitions for the current month now).
7. **Produce `transcript-chunks`** via the outbox + Avro `TranscriptChunk`, enveloped, idempotency key `chunk:{chunk_id}`, partition key the composite `{org_id}:{meeting_id}` ([ADR-0017](../adr/0017-transcript-chunks-composite-partition-key.md); see §8). The Avro value (`proto/transcript_chunk.avsc`):
   ```json
   {
     "type": "record", "name": "TranscriptChunk", "namespace": "io.nexus.events",
     "fields": [
       {"name": "header", "type": "io.nexus.events.EventEnvelopeHeader"},
       {"name": "chunk_id", "type": "string"},
       {"name": "meeting_id", "type": "string"},
       {"name": "speaker_id", "type": "string"},
       {"name": "start_ms", "type": "int"},
       {"name": "end_ms", "type": "int"},
       {"name": "text", "type": "string"},
       {"name": "is_final", "type": "boolean", "default": true},
       {"name": "confidence", "type": "float"},
       {"name": "redacted", "type": "boolean", "default": false}
     ]
   }
   ```
8. **Consume `meeting-room-events`** (from `meeting-service`): open the Deepgram stream on the first `participant.joined`, close it on room end / last leave. Ensure idempotency so a redelivered event never opens a duplicate stream.
9. **Diarization mapping is downstream.** Emit the diarized `speaker_id` as-is; the `speaker_id → user_id` mapping is resolved by `core-api` in `meeting_speakers` (§3.3), never guessed at write time.
10. **Do NOT produce `usage-events`.** Transcription minutes are metered by `billing-metering-service` in the control plane — see DECISION-REQUIRED §8 for the unresolved producer of the `transcription_second` / `meeting_minute` dimensions.
11. **Reconnect target:** Deepgram WS proxy reconnect success **> 99.9% within 2 s**; on reconnect, interims are discarded and finals resume.
12. **Env vars:** `DEEPGRAM_API_KEY`, `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `REDIS_URL`, `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`, `TRANSCRIPTION_WS_BIND`, `DATABASE_URL`, `OTEL_EXPORTER_OTLP_ENDPOINT`.

**Canonical names:** `transcription-service`, `transcript_chunks`, `transcript-chunks`, `TranscriptChunk`, `meeting-room-events`, `TranscriptionProvider`, `DeepgramProvider`, `nexus:{org_id}:pubsub:meeting:{meeting_id}`.

**Acceptance checks:**
- Interims drive captions but produce **zero** `transcript_chunks` rows and **zero** bus messages (the load-bearing test).
- Each final writes exactly one `transcript_chunks` row indexed `(meeting_id, start_ms)` and one `transcript-chunks` message; a redelivered final (`chunk:{chunk_id}`) never double-writes.
- The Deepgram key is never present in any client-facing payload or log (grep + log-scrub test).
- With `pii_redaction_enabled=true`, a final containing an email persists as `‹EMAIL›`; with it off, raw text persists in-region.
- An `eu` tenant's stream opens against the EU Deepgram endpoint and its chunks land in the EU cell.

---

#### 3.7 `realtime-gateway` — live captions wiring (P4)

**Repo / folder:** `mytheus-realtime/services/realtime-gateway/`.

**Steps:** subscribe to `caption` messages on `nexus:{org_id}:pubsub:meeting:{meeting_id}` (published by `transcription-service`) and emit `caption` to the room's sockets. Interim captions are mutable (replace-in-place by `caption_id`); final captions are append. Honor backpressure ordering: under client slow-consume, drop low-value deltas first; **never drop captions or finals**. Caption path p95 < 2 s (Deepgram-bound, no LLM).

**Acceptance checks:** a spoken phrase appears as an interim caption then settles to a final within p95 < 2 s; a dropped socket re-hydrates caption history from `core-api` on reconnect (finals are durable).

---

#### 3.8 `web-app` — live captions view, transcript view, speaker-mapping UI

**Repo / folder:** `mytheus-ui/`.

**Steps:**

1. **Live captions view** in the meeting room: render the `caption` socket channel (interim-then-final, no layout shift, append-only final track).
2. **Transcript view** `(app)/meetings/[meetingId]/transcript`: read ordered finals from `core-api` `GET /meetings/:id/transcript` (FTS over `transcription-service`-owned `transcript_chunks`), virtualized (a meeting yields up to ~360 finals — never render un-virtualized), timestamp jump via `(meeting_id, start_ms)`.
3. **Speaker-mapping UI:** list diarized `speaker_id`s from `meeting_speakers`; let the user assign each to a `user_id` (`PATCH …/speakers/:speaker_id`, `assignment='manual'`). Show the confidence-scored suggestion as advisory; manual wins.

**Acceptance checks:** captions stream live; the transcript replays in order and is searchable; assigning a speaker updates `meeting_speakers` and the transcript view relabels that speaker's turns.

---

## 4. Data stores stood up in this band

Per the roadmap data-store schedule (`22-roadmap-and-phasing.md` §3.1) and [05 Data Architecture](../05-data-architecture.md):

| Store | What lands here | Owner | Phase |
|---|---|---|---|
| PostgreSQL `meeting_rooms` | LiveKit room↔meeting map, egress/recording refs; RLS FORCE | `meeting-service` | P3 |
| PostgreSQL `chat_messages` | in-meeting chat, monthly range partition by `created_at`, GIN FTS | `core-api` | P3 |
| PostgreSQL `meeting_speakers` | diarized `speaker_id` ↔ `user_id` mapping (manual authoritative) | `core-api` | P3/P4 |
| PostgreSQL `transcript_chunks` | FINALS-ONLY rows, monthly range partition on `created_at`, local index `(meeting_id, start_ms)`, `pg_partman` premake | `transcription-service` | P4 |
| Redis `nexus:{org_id}:presence:{meeting_id}` | live presence set (ephemeral) | `realtime-gateway` | P3 |
| Redis `nexus:{org_id}:pubsub:meeting:{meeting_id}` | Socket.io cross-pod fan-out channel + caption bridge | `realtime-gateway` | P3/P4 |
| Object `nexus-recordings` | LiveKit recordings, key `{org_id}/{meeting_id}/`, region from `data_residency.object_bucket_region`, cold after 30 d | `meeting-service` | P3 |

Not in this band: Qdrant collections and the AGE `nexus_knowledge` graph (Plan 5+, when embeddings/agents/knowledge-graph land). The cold Parquet transcript archive and `pg_partman` nightly maintenance job are wired with `workflow-service` later; this band only creates the current-month partitions.

---

## 5. Events / topics introduced

All Avro + Schema Registry, subject `{topic}-value`, BACKWARD, every topic paired with a `<topic>.dlq` (14-day retention). Partition key is `org_id`, except `transcript-chunks` below, which is composite-keyed `{org_id}:{meeting_id}` ([ADR-0017](../adr/0017-transcript-chunks-composite-partition-key.md)). Every value opens with `EventEnvelopeHeader`.

| Topic | Producer | Consumers (this band) | Avro schema | Retention | Idempotency key |
|---|---|---|---|---|---|
| `meeting-room-events` | `meeting-service` | `transcription-service` (open/close stream), `core-api` | `MeetingRoomEvent` † | 3 days | `{room}:{event}:{participant_sid}` |
| `presence-events` | `realtime-gateway` | `core-api` (analytics) | `PresenceEvent` † | 1 hour | per join/leave transition |
| `transcript-chunks` | `transcription-service` | (downstream `embedding-service`/`agent-runtime-service`/`search-service` arrive Plan 5+; a smoke consumer here) | `TranscriptChunk` (FROZEN) | 7 days | `chunk:{chunk_id}` |
| `meeting-events` | `core-api` (existing) | `meeting-service` (open/close rooms) | `MeetingEvent` † | 7 days | per meeting transition |

† `MeetingRoomEvent`, `PresenceEvent`, `MeetingEvent` are net-new schema **names** introduced by [07](../07-event-architecture.md) — usable but **not frozen canon** (only the seven names `TranscriptChunk`, `AgentResult`, `Decision`, `Action`, `Notification`, `AgentRequest`, `AgentResponse` are frozen). `TranscriptChunk` IS frozen.

**Producer/consumer notes:** `transcription-service` produces `transcript-chunks` only (not `usage-events` — see §8). The `transcript-chunks` partition key is the composite `{org_id}:{meeting_id}`, ratified by [ADR-0017](../adr/0017-transcript-chunks-composite-partition-key.md) (§8) and canon 03; doc 07/doc 10, which already described this key, now cite the ADR.

---

## 6. SaaS overlay / compliance gate landing here

The dominant overlay this band is the **compliance shift-left gate** (§3.5) — it is the single hardest precondition in the whole MVP because Deepgram is the first subprocessor to receive raw C4 conversational audio. Recap of what must be live before W9:

1. **Subprocessor governance:** Deepgram DPA + SCC + zero-retention flag; subprocessor register updated.
2. **PII redaction at the boundary:** transcript-at-rest redaction in `transcription-service`, gated by `tenant_settings.pii_redaction_enabled`, per-tenant reversible token vault. (LLM-egress always-on redaction deferred to the agent/RAG band — flagged, not skipped.)
3. **Residency pinning:** `data_residency.region` selects the Deepgram endpoint and pins `transcript_chunks` + `transcript-chunks` to the tenant region.
4. **Audit:** redaction-toggle changes and any cross-border decision are written to `audit_logs` (append-only, owned by `admin-ops-service`).

Billing/budgets and dedicated GPU inference are **not** in this band (no LLM tier yet). The one metering touchpoint — who emits the `transcription_second` / `meeting_minute` usage dimensions — is an open ownership question surfaced in §8; resolving it is a precondition for Plan 7-9 metering, not for this band's capture path.

---

## 7. Mobile (mytheus-mobile) work in this band

`mytheus-mobile` today is an off-spec single-device "Meeting Pod" prototype on hardcoded `src/data/mock.ts` with a fake waveform, a static transcript, and a hardcoded 767-second timer. This band **realigns the live-meeting screens onto real backend transports** — keep the good UI components and design tokens, replace the fake data.

**Threads into:** P3 (LiveKit media + Socket.io presence/chat) and P4 (live captions + real meeting clock).

**Steps:**

1. **Remove mock data for the live path.** Delete the live-session/start-session dependence on `src/data/mock.ts`. (Other screens' mock removal is in their own bands; here, only the meeting/live surfaces.)
2. **Clerk auth on the session screens.** Gate `(app)/meetings/[meetingId]/room` behind the Clerk Expo session; `org_id` is server-derived (the client never sets it).
3. **Wire LiveKit media** (`src/realtime/LiveKitProvider.tsx`): call `POST /api/v1/meetings/:id/token` (through `api-gateway` → `meeting-service`), connect the LiveKit RN room with `{token, room, livekit_url}`, render the participant grid, re-mint on reconnect. Add `@livekit/react-native` + `@livekit/react-native-webrtc`.
4. **Wire Socket.io to `realtime-gateway`** (`src/realtime/SocketProvider.tsx`): join the `meeting_id` room on the `/v1` namespace, render **presence** from the presence channel and **live captions** from the `caption` channel (interim-then-final, < 2 s budget), and in-meeting chat. The Redis fan-out backbone is server-side; the client is pod-agnostic.
5. **Replace the fake waveform / static transcript** with the real caption stream and the `GET /api/v1/meetings/:id/transcript` read model; treat captions as ephemeral (re-hydrate finals from REST on reconnect — the live plane carries no durable truth).
6. **Replace the hardcoded 767 s timer with the real meeting clock** derived from the meeting's `scheduled_start` / room `room.created` time (server truth), ticking locally but anchored to server time.
7. **Mic capture + permissions.** Request `RECORD_AUDIO`/`NSMicrophoneUsageDescription` (and camera for video) **contextually at join**, not at launch; configure iOS `audio` background mode and the Android foreground-service flags so meeting audio survives backgrounding (scoped to an active meeting only). Declare these in `app.config.ts`.
8. **Env:** `EXPO_PUBLIC_API_GATEWAY_URL`, `EXPO_PUBLIC_REALTIME_WS_URL`, `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY`, `EXPO_PUBLIC_LIVEKIT_URL`, `EXPO_PUBLIC_SENTRY_DSN`, `EXPO_PUBLIC_ENV`. No secret keys ship (the LiveKit room token is minted per join, never an env var).

**Constraints (do not break):** contract-compatibility with `mytheus-ui` (same endpoints, same Socket.io `/v1` event names/payloads, same token-mint flow); `org_id` never client-set; no LiveKit/Deepgram keys in the bundle; captions/presence carry no durable truth.

**Acceptance checks:** a phone joins the same `meeting_id` room as two browsers; the phone shows real video tiles, real presence, live captions under the 2 s budget, and a meeting clock anchored to server time; mic permission is requested at join; backgrounding the app during the meeting keeps audio alive; no value comes from `src/data/mock.ts` on these screens.

---

## 8. DECISION-REQUIRED callouts

> ✅ **DECISION-REQUIRED #1 — RESOLVED. PARTITION-KEY CONTRADICTION for `transcript-chunks`.**
> **Resolved by [ADR-0017](../adr/0017-transcript-chunks-composite-partition-key.md), option (c).** `transcript-chunks` and `agent-stream` are composite-keyed `{org_id}:{meeting_id}`; every other topic, including `agent-results`, keeps `org_id` alone. Canon [03-event-taxonomy](../_canon/03-event-taxonomy.md) is amended accordingly, and the "not ratified and is not used" sentence is superseded precisely for these two topics. `07-event-architecture.md §4.1` and `10-realtime-and-transcription.md §3.2/§5` are propagated and cite ADR-0017. The `transcription-service` producer's key function and the Avro/Registry subject config are unblocked: build against the composite key for `transcript-chunks`, `org_id` alone for everything else.

> ✅ **DECISION-REQUIRED #2 — RESOLVED. CONSENT entity landed schema-first, defaulted off.**
> **Resolved by [ADR-0018](../adr/0018-consent-entity-behind-flag.md).** `meeting_participants.consented_at timestamptz NULL` / `consent_version text NULL` and `tenant_settings.consent_required boolean NOT NULL DEFAULT false` are amended into canon [Data Model §2.1/§2.2](../_canon/02-data-model.md); `12-compliance-and-privacy.md §3.3` is propagated and no longer describes this as a proposal. **What is still open:** the Alembic migration and the `meeting-service` 428-Precondition-Required application code were explicitly out of scope for ADR-0018 and remain later tasks that depend on this schema landing first. `consent_required` defaults `false`, so no tenant's token-mint behavior changes until a tenant is explicitly flipped on — flipping it is a legal/compliance decision, not an engineering one (ADR-0018). P3/P4 may proceed against the ratified schema; whether the 428 gate itself must ship before recording real audio in a given jurisdiction is still the legal/compliance call ADR-0018 assigns, not an open canon question.

> ⚠️ **DECISION-REQUIRED #3 — `usage_events` derived-dimension producer ownership (BLOCKS Plan 7-9 metering, not this band).**
> The `usage_events.event_type` values `meeting_minute` and `transcription_second` are **not assigned to any service** in the catalog. Docs flag candidates: `transcription-service`, `meeting-service`, or a storage-sweep job. Critically, [mytheus-realtime system.md §3](../../mytheus-realtime/system.md) states `transcription-service` **does NOT produce `usage-events`** (billing meters it), and canon 03 lists `usage-events` producers as only `agent-runtime-service` + `rag-service`.
> **Options:** (a) **`meeting-service` emits `meeting_minute`** from room lifecycle (`room.created`→`room.ended` duration) and **`transcription-service` emits `transcription_second`** from Deepgram stream duration — both via `usage-events`; requires adding them as authorized producers in canon 03. (b) A **`workflow-service` storage-sweep / meeting-lifecycle job** derives both from `meeting_rooms` + `transcript_chunks` durations post-meeting (keeps the real-time tier off the billing bus, consistent with system.md). (c) `billing-metering-service` derives them from `meeting-room-events` + chunk counts it already consumes.
> **Who decides:** Billing lead + Canon owners. **What it blocks:** metering accuracy in Plan 7-9; if unresolved, `meeting_minute`/`transcription_second` have no producer and those plans bill nothing. **Resolve before metering needs them — does not block this band's capture path.**

---

## 9. Risk checkpoints

- **R2 (meeting-SDK lock, RESOLVED by ADR-0003) → residual R23 (LiveKit SFU self-host ops).** This band runs on **LiveKit Cloud** (the sanctioned early-stage / Railway-Render fallback), so R23 is deferred to the scale-out band. Checkpoint: the `MeetingTransport` adapter is the *only* SDK import (grep test) so the eventual self-host swap is a one-interface reimplementation. **LiveKit-Cloud-as-subprocessor** means the Deepgram-style DPA/SCC obligation also applies to LiveKit Cloud here (dropped at self-host) — include LiveKit Cloud in the subprocessor register for this band.
- **R5 / R19 (real-time latency + fan-out).** Checkpoint: caption path p95 < 2 s, token-mint p95 < 250 ms, Deepgram reconnect > 99.9% within 2 s, and the cross-pod Redis fan-out test green. `realtime-gateway` is a **Wave 1 always-separate** extraction — keep it its own process even in the modular monolith so the seam never has to be re-cut.
- **R7 (diarization speaker accuracy).** Checkpoint: `meeting_speakers.assignment='manual'` is authoritative; the suggestion is confidence-scored advisory; the transcription stage never guesses `user_id` at write time.
- **R-compliance (subprocessor egress).** Checkpoint: the §3.5 gate is provably green before any Deepgram stream opens against real audio; the redaction node and residency pin have passing tests.

---

## 10. Definition of Done & end-of-phase demo  (a demo at the END of EVERY phase)

**End of P3 (W8) — Meeting Platform demo + DoD:**
- **Demo:** two browsers + one phone join the same `meeting_id` LiveKit room via `meeting-service`-minted tokens; video grid + presence + cross-pod chat work live; a `recording.ready` webhook persists a `meeting_rooms.recording_ref` and an object in `nexus-recordings/{org_id}/{meeting_id}/`.
- **DoD:** `meeting-service` token mint (p95 < 250 ms) + webhook (HMAC-verified, inbox-deduped) shipped; `meeting_rooms` migrated with RLS FORCE; `realtime-gateway` scaffolded with Redis-adapter fan-out, presence, chat relay; `chat_messages` + `meeting_speakers` migrated in `core-api`; staging deploy fires on merge; mobile media + presence wired; cross-pod fan-out test, RLS isolation test, token-mint deny-by-default test all green.

**End of P4 (W12) — Real-time & Transcription demo + DoD:**
- **Demo:** the same meeting now streams **live captions** under p95 < 2 s; **finals only** persist to `transcript_chunks` and flow on `transcript-chunks` (Avro `TranscriptChunk`); the transcript view replays ordered, searchable chunks; the speaker-mapping UI assigns `speaker_id → user_id`; the compliance gate is shown green (DPA/SCC on file, redaction firing on/off by `pii_redaction_enabled`, `eu` tenant pinned to EU Deepgram endpoint + EU cell).
- **DoD:** compliance gate green **before** any Deepgram stream; `transcription-service` shipped with the provider adapter, interim-vs-final split, finals-only persistence, at-rest redaction node, residency pinning, `transcript-chunks` producer (outbox + Avro + `chunk:{chunk_id}` dedupe), `meeting-room-events` consumer; partition-key DECISION ratified (§8 #1); `transcript_chunks` migrated (monthly partition, local index, RLS FORCE); captions wired through `realtime-gateway`; `web-app` captions/transcript/speaker-mapping shipped; mobile captions + real meeting clock + mic capture shipped; interims-never-persisted test, redaction test, residency test, reconnect-SLO test all green.

---

## 11. Source docs

Authoring references used (by path):
- [10-realtime-and-transcription.md](../10-realtime-and-transcription.md) — LiveKit transport, Deepgram proxy, interim-vs-final, finals-only, `realtime-gateway` fan-out, per-path latency budget.
- [03-container-architecture.md](../03-container-architecture.md) — C4 L2 containers, the three WebSocket flows (§4.3), store ownership (§5), edge path (§6).
- [04-service-catalog.md](../04-service-catalog.md) — `meeting-service` (§3.6), `realtime-gateway` (§4.1), `transcription-service` (§4.2) responsibilities, owned data, events, failure modes.
- [12-compliance-and-privacy.md](../12-compliance-and-privacy.md) — subprocessor DPA/SCC (§4), PII redaction boundaries (§5), residency pin (§6), consent (§3.3).
- [05-data-architecture.md](../05-data-architecture.md) — `transcript_chunks` partitioning/indexing (§3.2-§3.4), Redis namespaces (§6), `nexus-recordings` + cold archive (§7).
- [_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md) — `transcript-chunks` / `meeting-room-events` / `presence-events` schemas, partition keys, retention, idempotency; the `org_id`-sole-key freeze.
- [adr/0003-meeting-sdk-livekit.md](../adr/0003-meeting-sdk-livekit.md) — LiveKit lock, transport adapter, `meeting_rooms`, one diarized stream per meeting, reversibility.
- [adr/0012-realtime-transport-websocket-tier.md](../adr/0012-realtime-transport-websocket-tier.md) — `realtime-gateway` split, Socket.io rooms by `meeting_id`, Redis pub/sub backbone, sticky/stateless, captions/presence/chat phasing.
- [mytheus-realtime/system.md](../../mytheus-realtime/system.md) — the implementation-facing real-time tier design (two services, three flows, event contracts, statelessness).
- [mytheus-mobile/system.md](../../mytheus-mobile/system.md) — mobile client architecture, live transports, background audio, permissions, net-new log.
- [mytheus-backend/system.md](../../mytheus-backend/system.md) — `meeting-service` / `core-api` responsibilities, ownership, edge path, outbox/inbox, confirmation rule.
</content>
</invoke>
