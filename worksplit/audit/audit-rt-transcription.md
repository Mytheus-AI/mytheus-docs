# mytheus-realtime audit — transcription-service, realtime-gateway, libs

READ-ONLY audit. No files modified. All claims cite `path:LINE` read from source, not from `.md` status docs.

Root: `/Users/sushan/Projects/Mytheus`

Note on scope: the brief said `services/realtime-gateway/` "looks nearly empty: 1 py file". That premise is wrong — realtime-gateway is a **TypeScript** service with 639 lines in `src/server.ts` plus three more src modules and 15 test files. There are no Python files in it at all. Details in the REALTIME-GATEWAY section.

---

## LIVE TRANSCRIPTION REALITY CHECK — is real ASR implemented?

**No. No audio is ever transcribed. And `devseed.py` is NOT the only thing that produces `transcript_chunks` — there is a second, worse path: a canned fake script with no dev guard.**

Four facts, each verified in source.

### 1. The audio ingress is a stub that accepts and immediately closes

`mytheus-realtime/services/transcription-service/src/transcription_service/main.py:119-124`:

```python
@app.websocket("/internal/audio")
async def internal_audio(websocket: WebSocket) -> None:
    # LiveKit server-side audio-hook target. Scaffold: accept then close cleanly so the transport
    # contract exists. The Deepgram WSS proxy + transcript_chunks persistence land later.
    await websocket.accept()
    await websocket.close()
```

It never reads a frame, never references `SessionTracker`, never reaches a provider. The module docstring at `main.py:10-13` explains that the Kafka consumer deliberately shares this process *precisely so that* `/internal/audio` can push audio into the same in-memory `TranscriptionStream` object — that push was never written.

And **nothing connects to it.** A grep for `internal/audio` across `mytheus-backend`, `mytheus-realtime`, `mytheus-platform` returns only: the endpoint definition (`main.py:119`), two comments referencing it (`main.py:11`, `meeting_room_consumer.py:37,235`), and one test (`tests/test_healthz.py:17`). No client anywhere.

### 2. The real ASR engine is Deepgram, written for real, and hard-blocked

Engine: **Deepgram** streaming WSS. Not whisper, not faster-whisper, not assemblyai, not a local model. `providers/deepgram.py:1-2` names it the "LOCKED DEFAULT per the plan."

Dependency: **the `deepgram-sdk` package is deliberately NOT installed.** `pyproject.toml:5-15` declares `websockets>=12` and the code speaks Deepgram's WSS protocol directly. The reasoning is at `deepgram.py:11-16`: pulling a vendor SDK in for a path that cannot run is a dependency they chose not to take, and swapping it in later is a one-file change since nothing else imports Deepgram.

Configured: `DEEPGRAM_API_KEY` is read at `config.py:39` and is **empty in `.env.example:23`**, with a comment saying to leave it unset.

Called: `DeepgramProvider.open_stream` (`deepgram.py:112-125`) is real code — regional endpoint selection, `diarize=true`, `interim_results=true`, `Authorization: Token` header, `websockets.connect`. But **line 117 is `assert_compliance_gate_green()`, called before anything touches the network or the key.** That function (`compliance.py:46-58`) raises unless `COMPLIANCE_GATE_GREEN`, and `compliance.py:39` is:

```python
COMPLIANCE_GATE_GREEN: Final[bool] = False
```

This is a **source constant, deliberately not an environment variable** (`compliance.py:16-25`): condition 1 is a signed Deepgram DPA/SCC, a legal artifact, and they argue an env var would let anyone with deploy access override a fact about whether a contract was signed. Flipping it requires a code change under review. There is also no Deepgram account.

Response parsing at `deepgram.py:40-96` is written correctly against the documented shape, including a carefully reasoned `chunk_id` derivation (`uuid5` over `deepgram:{request_id}:{speaker}:{start_ms}:{end_ms}` — `:82-86`) after two rounds of fixes for (a) `request_id` not being a UUID and (b) two diarized speakers colliding on the same `start_ms`. None of it has ever executed. `deepgram.py:8-9` states plainly: "No test in this build claims coverage of a real Deepgram WSS session; that is BLOCKED, not green-against-a-fake."

### 3. The default provider in every environment is a fake that writes real rows

`providers/factory.py:20-23`:

```python
def select_provider(settings: Settings) -> TranscriptionProvider:
    if not settings.deepgram_api_key:
        return FakeProvider()
    return DeepgramProvider(api_key=settings.deepgram_api_key)
```

`DEEPGRAM_API_KEY` is unset in every environment, so `select_provider` returns `FakeProvider` everywhere, including `main.py:74` at app startup.

`FakeProvider.open_stream` (`providers/fake.py:92-98`) returns `default_script()` — three canned utterances: two interims and one **final** with text `"hello there, how are you"` (`fake.py:60-68`).

`SessionTracker.on_participant_joined` (`sessions.py:75-88`) opens that stream on a `joined` event and spawns `_pump`, which drains every utterance into `persistence.process_utterance` (`sessions.py:117-126`) — the identical path a real provider would take.

`persistence.process_utterance` (`persistence.py:121-190`) writes the final to `transcript_chunks` and enqueues an `outbox` row, which `mytheus-backend`'s relay publishes to the `transcript-chunks` topic as Avro.

**Consequence: if this service ever ran against a live `meeting-room-events` feed, every meeting join would persist a fabricated transcript row reading "hello there, how are you" into the real table and publish it onto the real bus. Nothing marks it as synthetic.** Downstream consumers (embedding-service, search-service, agent-runtime-service) could not distinguish it from a real meeting.

### 4. The asymmetry with devseed is the tell

`devseed.py` — the *intentional* seed path — refuses to run without an explicit environment guard (`devseed.py:222-229`, `_ENV_GUARD = "TRANSCRIPTION_DEV_SEED"`), and the docstring at `devseed.py:32-39` explains exactly why: "The risk is not corruption but plausible-looking fabricated transcripts landing in a real tenant's history, where nothing downstream could distinguish them from a real meeting."

**That reasoning applies verbatim to `FakeProvider`, which has no guard at all.** The authors clearly understood the hazard; they guarded the tool a developer runs by hand and left the default runtime path ungated. This is a real defect, not a documentation gap.

### Answer to the crux question, stated explicitly

- Is real audio ever transcribed? **No — at no point, in no environment. The audio path is absent at every hop (see AUDIO PATH TRACE).**
- Is `devseed.py` the only thing that produces `transcript_chunks`? **It is the only *intentional* producer today, and the only one that can run without a broker. But it is not the only *possible* producer: `FakeProvider` driven by a `meeting-room-events` `joined` message writes the same rows through the same code, ungated.** In the deployed world (where `meeting-service` ships but `transcription-service` does not), neither runs — so today the table is empty unless someone runs the seeder.

---

## AUDIO PATH TRACE — mic → ? → transcript row

| # | Hop | Status | Evidence |
|---|---|---|---|
| 1 | Browser mic → LiveKit SFU | **ABSENT** (degrades cleanly) | `mytheus-ui/src/lib/realtime/useLiveKitRoom.ts:69` reads `NEXT_PUBLIC_LIVEKIT_URL`; `:70-72` sets `status: "unconfigured"` when unset; `:85` returns from the effect before constructing a `Room`. The var is commented out in `mytheus-ui/.env.example:32`. No LiveKit account exists anywhere in the program (`useLiveKitRoom.ts:13-19`). |
| 2 | LiveKit → server-side audio tap | **ABSENT** | `mytheus-backend/services/meeting_service/transport.py:139-141` — `LiveKitTransport.subscribe_room_audio` body is `raise NotImplementedError("subscribe_room_audio lands with the transcription tap")`. `FakeTransport`'s version (`:231-233`) records the call and returns `None`. **Grep across all repos finds no caller** — only the Protocol declaration (`:56`) and the two implementations. Same for `start_egress` (`:135-137`, `NotImplementedError`). |
| 3 | audio tap → `/internal/audio` | **ABSENT** | No caller exists (see §1 above). Nothing constructs a WS client to `TRANSCRIPTION_WS_BIND` (8091). No file input, no egress-to-storage-then-read path, no alternative ingress of any kind. |
| 4 | `/internal/audio` → `SessionTracker` | **STUB / BROKEN** | `main.py:119-124` accepts and closes. It holds no reference to the `tracker` created at `main.py:75`. The wiring the docstring describes does not exist. |
| 5 | LiveKit webhook → `meeting-room-events` | **WIRED but dormant** | `mytheus-backend/services/meeting_service/webhook.py:184-198` (`recording_ready`) and `:234-248` (`joined`/`left`) enqueue outbox rows in the same transaction as the state change. Real code. Cannot fire without a LiveKit account to send webhooks. |
| 6 | `meeting-room-events` → `SessionTracker` | **WIRED** | `meeting_room_consumer.py:216-256` — real confluent-kafka `Consumer`, `AvroDeserializer` against `meeting-room-events-value`, poll in `run_in_executor` (`:236`), manual commit *after* the effect (`:256`), Redis `SET NX` dedupe with 24h TTL (`:125-130`), poison→DLQ routing (`:79-93`, `:170-176`). Genuinely and carefully implemented. |
| 7 | `SessionTracker` → provider stream | **WIRED to the FAKE** | `sessions.py:75-77` calls `provider.open_stream`; `factory.py:20-23` binds that to `FakeProvider`. Real provider blocked at `deepgram.py:117`. |
| 8 | utterance → `transcript_chunks` + outbox | **WIRED** | `persistence.py:84-118` — one `enqueue` whose `is_new` boolean gates the `transcript_chunks` INSERT, both in one transaction (`outbox.py:46-63`, `ON CONFLICT (topic, idempotency_key) DO NOTHING`). Interims never reach it: `persistence.py:133-166` returns before touching `session_factory` at all. |
| 9 | final/interim → live caption | **WIRED** | `captions.py:32-58` publishes `{"type":"caption", ...}` to `nexus:{org_id}:pubsub:meeting:{meeting_id}`. Best-effort, swallows failures (`:53-62`). |
| 10 | caption → browser | **WIRED in code, unreachable in practice** | `realtime-gateway/src/server.ts:262-291` bridges the channel to `socket.emit("caption", ...)`. Gateway is not deployed and the UI has no URL for it (see WEBSOCKET CONTRACT). |

**Summary: hops 1-4 are ABSENT/STUB. Hops 5-9 are WIRED and well built. The pipeline from an *event* to a *row* is real; the pipeline from *audio* to an *event with real content* does not exist at any hop.**

---

## WEBSOCKET CONTRACT — UI expects X, backend serves Y

**The protocol contract matches on both sides, field for field. This is not the disconnect. The disconnect is that no URL is configured and no gateway is deployed to serve one.**

### What the UI expects

`mytheus-ui/src/lib/realtime/useMeetingSocket.ts:60`:
```ts
const url = process.env.NEXT_PUBLIC_REALTIME_WS_URL;
```
`:75-79`:
```ts
const socket = io(`${url.replace(/\/+$/, "")}/v1`, {
  auth: { token },
  transports: ["websocket"],
});
```

- **Protocol:** Socket.io v4 (`socket.io-client`), WebSocket transport only
- **Namespace:** `/v1`
- **Engine.io path:** default `/socket.io` (not overridden)
- **Auth:** Clerk JWT in `handshake.auth.token` — deliberately not a header (`:71-73`: "not a header, which the WebSocket upgrade cannot carry portably anyway")
- **Client→server:** `join` with `{ meetingId }` plus an ack callback expecting `{ ok, presence?, error? }` where `error` is `"FORBIDDEN" | "INVALID_PAYLOAD"` (`:53-57`, `:99-112`)
- **Server→client:** `caption` with `{ meetingId, payload }` (`:120-127`), `presence` with `string[]` (`:129-131`)
- **Caption payload shape** (`captions.ts:29-37`): `{ chunkId, speakerId, startMs, endMs, text, isFinal, confidence? }`
- **Re-join on every reconnect** (`:115-117`), and **re-hydrate durable finals from REST** on each connect via `/api/meeting-transcript?meeting_id=...` (`:81-95`), because the gateway replays nothing

### What the gateway serves

`mytheus-realtime/services/realtime-gateway/src/server.ts`:

- `:198` — `const io = new Server(httpServer, { path: "/socket.io" });` ✅ matches
- `:213` — `const nsp = io.of("/v1");` ✅ matches
- `:216-217` — `nsp.use(async (socket, next) => { const token = socket.handshake.auth?.token ... })` ✅ matches
- `:337` — `socket.on("join", async (payload: JoinPayload, ack?: (res: JoinAck) => void) => ...)` ✅ matches
- `:291` — `socket.emit("caption", outbound);` ✅ matches
- `:393`, `:487`, `:521` — `nsp.to(roomName(...)).emit("presence", members)` ✅ matches
- `:411` — `socket.on("chat", ...)` (UI does not use this hook for chat; not a mismatch, just unused here)
- `:189` — plain HTTP `/healthz`
- `:591` — `port: Number(process.env.PORT ?? 8090)`

Caption payload produced by `transcription-service` (`captions.py:43-51`) is `{chunkId, speakerId, startMs, endMs, text, isFinal, confidence}` — identical to the UI's `CaptionPayload` interface. The gateway forwards it verbatim (`server.ts:262-291`, and the header comment at `:64-82` documents the field list the UI's `captions.ts:8-12` says it copied "verbatim rather than inferred").

Redis channel name matches exactly on both sides: `captions.py:29` returns `f"nexus:{org_id}:pubsub:meeting:{meeting_id}"`; `libs/keys.ts:36-38` returns the same template. `libs/keys.ts:30-34` warns that a mismatch here fails silently with no error to catch — they got it right.

### The actual gap

- **Exact expected URL:** `wss://${NEXT_PUBLIC_REALTIME_WS_URL}/socket.io/?EIO=4&transport=websocket` on namespace `/v1`.
- **Exact served URL: none.** `NEXT_PUBLIC_REALTIME_WS_URL` is commented out in `mytheus-ui/.env.example:31`, so `useMeetingSocket.ts:61` sets `status: "disabled"` and `:67` returns before creating a socket. And there is nothing to point it at even if set: `realtime-gateway` appears in no compose file and **has no Dockerfile** (see DEPLOYED?). In a hypothetical local run it would be `ws://localhost:8090/v1`.

The one endpoint in this chain that *is* served: `/api/meeting-transcript` exists at `mytheus-ui/src/app/api/meeting-transcript/`, fronting `core-api`'s `GET /api/v1/meetings/{id}/transcript`.

---

## REALTIME-GATEWAY — what's there

**Not 1 file, not Python, not empty.** It is a TypeScript Socket.io service:

| File | Lines |
|---|---|
| `src/server.ts` | 639 |
| `src/presenceEventsProducer.ts` | 229 |
| `src/coreApiClient.ts` | 165 |
| `src/presence.ts` | 139 |
| 15 test files under `test/` | ~2,100 total |

Tests include `cross-pod-fanout`, `cross-pod-captions`, `cross-tenant-room-collision`, `presence-pod-death`, `caption-latency`, `caption-unsubscribe-on-disconnect`, `stop-drains-concurrent-disconnect`, `auth`, `keys`, `gateway`, `captions`, `presence`, `presence-events-kafka`, `presence-events-producer`, `coreApiClient`.

It is the ADR-0012 WebSocket fan-out tier, and it is substantially built:

- **Transport:** HTTP server + Socket.io namespace `/v1`, `@socket.io/redis-adapter` for cross-pod broadcast (`server.ts:198-213`)
- **Auth:** Clerk JWT verified via JWKS in `libs/auth.ts` (jose `createRemoteJWKSet`, fail-closed on every failure mode — `libs/auth.ts:1-12`). `org_id`/`user_id` come only from a verified token, never from the client payload.
- **Authz:** deny-by-default room join via a `core-api` participant check — `coreApiClient.ts:116-117`, `GET /api/v1/meetings/{id}/participants/{userId}`
- **Presence:** Redis-only (never a durable table), heartbeat/stale/sweep cycle (`presence.ts`), mirrored to the `presence-events` Kafka topic (`presenceEventsProducer.ts`), plus a reconciliation sweep for pod death (`server.ts:510-521`)
- **Chat:** relayed over the socket, durably written by `core-api` at `POST /api/v1/chat` (`coreApiClient.ts:151`)
- **Captions:** subscribes the explicit Redis channel **per-socket**, not by Socket.io room membership. The reasoning at `server.ts:239-252` is that the channel is fed by a non-Socket.io publisher, so every subscribed pod already receives the raw message; layering `nsp.to(room)` on top would re-broadcast via the adapter's internal channels and deliver the caption once per pod per client. `:274-279` treats the channel name (not the message body) as the authenticated fact, so a mislabeled message cannot land in the wrong meeting's UI.
- **Tenancy:** `roomName(orgId, meetingId)` is org-scoped (`libs/keys.ts:78-80`). The 30-line comment above it (`:48-77`) documents a real shipped cross-tenant leak — captions once, presence/chat again — and why org-scoping the room is what makes it structurally impossible rather than dependent on `meeting_id` uniqueness.
- **Shutdown:** `stop()` waits for in-flight disconnect handlers before tearing down Redis clients (`server.ts:556-580`)

**This is the most complete component in the audit scope.** Its only blocker is packaging: no Dockerfile, no compose entry, no deployed URL.

`libs/` (shared, 3 files): `auth.ts` (56) Clerk JWT verification; `keys.ts` (91) the canonical Redis key / room builders; `envelope.ts` (116) the event envelope. Small, well-documented, and the single source of truth for key shapes across both services.

---

## STORAGE DESTINATION vs BACKEND READ PATH — same store or split?

**Same Postgres, same table, same column set, same ordering index. No split.** I checked this precisely because the brief flagged it as a likely disconnect; it is not one.

**Writer** — `persistence.py:96-115`:
```sql
INSERT INTO transcript_chunks
(id, org_id, meeting_id, speaker_id, start_ms, end_ms, text, is_final, confidence, redacted)
VALUES (...)
```

**Owner / migration** — `mytheus-realtime/services/transcription-service/migrations/versions/be66698dbefb_create_transcript_chunks.py:88-104`. Same columns, `PARTITION BY RANGE (created_at)` monthly, `PRIMARY KEY (id, created_at)`. RLS `ENABLE` + `FORCE` + `tenant_isolation` policy on `org_id = NULLIF(current_setting('app.current_org', true), '')::uuid` (`:118-123`). Index `ix_transcript_chunks_meeting_start (meeting_id, start_ms)` on the parent (`:116`). Partitions are premade by hand (current + 2 months) because pg_partman is not installed (`:18-28`), with a DEFAULT catch-all partition (`:138`). Each partition is explicitly REVOKE'd (`_partition_privileges.lock_down_partition`) after they found `mytheus_app` could `DELETE FROM transcript_chunks_2026_08` cross-tenant despite the parent's RLS (`:36-43`).

**Reader** — `mytheus-backend/services/core_api/routes/transcript.py:58`:
```python
_COLUMNS = "id, meeting_id, speaker_id, start_ms, end_ms, text, is_final, confidence, redacted"
```
`GET /api/v1/meetings/{meeting_id}/transcript` (`:95`), keyset-paginated, ordered `(start_ms, id)` — the same tuple the writer's index is built on (`:29-32`). Explicitly read-only (`:12-15`: "`transcript_chunks` is owned and written only by `transcription-service` in `mytheus-realtime`... Nothing in this module writes, and no migration in this repo touches that table").

**Same physical database:** `.env.example` sets `DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5433/mytheus`, and the migration's own table comment (`:106-113`) states the table is "Owned and migrated by transcription-service (mytheus-realtime), NOT mytheus-backend, despite sharing this Postgres instance."

**Second destination — the bus:** finals also go to the shared `outbox` table (`outbox.py`), topic `transcript-chunks`, subject `transcript-chunks-value`, partition key `{org_id}:{meeting_id}` per ADR-0017 (`persistence.py:46-50`), idempotency key `chunk:{chunk_id}` (`:42-43`). `mytheus-backend`'s relay drains it. `outbox.py:1-19` justifies this as the one deliberate cross-service write: `outbox` is cross-cutting infrastructure every producer writes to, not a domain table.

**One correctness note, handled properly:** `transcript_chunks` carries no FK to `meetings` (declarative partitioning precludes it — migration `:11-16`). `transcript.py:75-92` therefore does an RLS-scoped `meetings` visibility check *before* reading chunks, so another tenant's meeting id is a 404 rather than an empty page that would leak "well-formed but not yours."

Verdict: **one store, one schema, one writer, one reader. Correct.**

---

## IMPLEMENTED / PARTIAL / STUB / MISSING

### IMPLEMENTED
- `realtime-gateway` end to end — auth, presence, chat, caption bridge, cross-pod fan-out, org-scoped rooms, graceful drain (`server.ts`, 639 lines, 15 test files)
- `meeting_room_consumer` — Kafka consumer, Avro deserialization, Redis dedupe, DLQ routing, commit-after-effect, non-blocking poll (`meeting_room_consumer.py`)
- `SessionTracker` — one stream per `meeting_id` (never per participant, the locked 6× cost lever — `sessions.py:1-6`), idempotent on redelivered `joined`/`left`
- `persistence` — the interim/final split, with the outbox `is_new` boolean as the single atomic dedupe gate for both writes
- `transcript_chunks` migration — partitioning, RLS, per-partition privilege lockdown
- `captions` publish; `redaction` (90 lines + 122-line test); `residency` endpoint selection; `audit` session-decision logging
- `core-api` transcript read endpoint — keyset pagination, `from_ms` server-side seek
- UI caption reducer `captions.ts` — pure, correct interim-replace-in-place / final-append, idempotent on `chunkId`, ordered by `startMs` with a `chunkId` tiebreak
- `libs/` — `auth.ts`, `keys.ts`, `envelope.ts`

### PARTIAL
- `DeepgramProvider` — protocol-correct and fully written, `chunk_id` derivation carefully fixed twice, but unreachable and **zero coverage of a live session** (stated at `deepgram.py:8-9`)
- `useLiveKitRoom` / `useMeetingSocket` — complete, reviewed, typechecked; both permanently in `unconfigured` / `disabled`
- Residency — endpoint *selection* is real and tested (`residency.py:31-33`); region-local Postgres/Redpanda placement explicitly BLOCKED, one cell exists (`residency.py:9-19`)
- Compliance gate — conditions 2 (redaction) and 3 (residency pin) implemented in code; condition 1 (signed DPA/SCC) is a legal artifact no PR can satisfy (`compliance.py:4-15`)

### STUB
- `main.py:119-124` — `/internal/audio`, accept-then-close
- `transport.py:139-141` — `LiveKitTransport.subscribe_room_audio`, `NotImplementedError`
- `transport.py:135-137` — `LiveKitTransport.start_egress`, `NotImplementedError`

### MISSING
- **Any audio capture, egress, or track subscription — anywhere in any repo**
- **Any Dockerfile in `mytheus-realtime`** (`find -iname "Dockerfile*"` → empty)
- Consumer liveness in `/healthz` — disclosed at `main.py:15-19`: if the consumer task dies its death is logged loudly but `/healthz` still returns ok, so a liveness probe would not restart the pod
- **A guard preventing `FakeProvider` from writing to a real database** (the defect from §4 above)
- `meeting_speakers` diarization-label→user mapping (referenced as future work in `captions.ts:90-93`)

---

## CONFIG/ENV REQUIRED

**transcription-service** (`config.py:30-44`, `.env.example`):

| Var | Default | State |
|---|---|---|
| `DATABASE_URL` | `postgresql+asyncpg://postgres:postgres@localhost:5432/mytheus` (code) / `:5433` (`.env.example`) | set |
| `REDIS_URL` | `redis://localhost:6379` (code) / `:6380` (`.env.example`) | set |
| `REDPANDA_BROKERS` | `localhost:9092` | set |
| `SCHEMA_REGISTRY_URL` | `http://localhost:8081` | set |
| `TRANSCRIPTION_WS_BIND` | `0.0.0.0:8091` | set, nothing connects |
| `DEEPGRAM_API_KEY` | `None` | **unset → FakeProvider** |
| `LIVEKIT_URL` / `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` | `None` | **all unset** |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `None` | unset |
| `TRANSCRIPTION_DEV_SEED` | — | must be `1` for the seeder (`devseed.py:222`) |

Note: the `.env.example` Postgres (`:5433`) and Redis (`:6380`) ports are the **test-container** ports, not the app ports the code defaults to.

Python deps (`pyproject.toml:5-15`): `fastapi`, `uvicorn[standard]`, `websockets>=12`, `confluent-kafka[avro,schemaregistry]>=2.4`, `sqlalchemy[asyncio]`, `asyncpg`, `alembic`, `redis`, `pydantic`. **No `deepgram-sdk`, by explicit decision.**

**realtime-gateway** (`.env.example`): `PORT=8090`, `WS_BIND_HOST=0.0.0.0`, `REDIS_URL=redis://localhost:6380`, `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`, `CLERK_JWKS_URL` (**empty — no live Clerk instance**), `CORE_API_URL=http://localhost:8000`, `OTEL_EXPORTER_OTLP_ENDPOINT`, optional `PRESENCE_HEARTBEAT_INTERVAL_MS` / `PRESENCE_STALE_AFTER_MS` / `PRESENCE_SWEEP_INTERVAL_MS`.

Node deps (`package.json`): `socket.io`, `@socket.io/redis-adapter`, `ioredis`, `jose`, `kafkajs`, `@kafkajs/confluent-schema-registry`, `pino`.

**mytheus-ui**: `NEXT_PUBLIC_REALTIME_WS_URL` and `NEXT_PUBLIC_LIVEKIT_URL` — both **commented out** in `.env.example:31-32`.

---

## DEPLOYED?

**No. Nothing from `mytheus-realtime` is in `mytheus-platform/deploy/compose.prod.yml`.**

Services defined there (grep `^  [a-z].*:`): `postgres`, `redis`, `redpanda`, `temporal`, `migrate`, `web`, `relay`, `workflow`, `api-gateway`, `bff`, `identity-tenant-service`, `meeting-service`, `integration-service`, `billing-metering-service`, `admin-ops-service`, `notification-service`, `ui`. **No `transcription-service`. No `realtime-gateway`.**

Worse: **there is no Dockerfile anywhere in `mytheus-realtime`** — neither service is buildable as an image, so it could not be added to compose without new packaging work.

They exist only as future-tense plans in `mytheus-platform`:
- `system.md:33` — real-time family on a `prod-realtime` node pool with KEDA on active connections
- `system.md:37` — replica floors of 6 each
- `system.md:61` and `CLAUDE.md:38` — "Wave 1"
- `system.md:98` — `api-gateway` routes WebSocket traffic to `realtime-gateway`
- `system.md:128` — Deepgram listed as the transcription vendor with a server-side key
- `README.md:104`, `CLAUDE.md:10,187` — repo inventory

CI does touch it: `mytheus-platform/.github/workflows/ci.yml:15` lists `transcription-service` among the Python repos it lints.

**Operational consequence:** `meeting-service` **is** deployed (`compose.prod.yml:293`), so in production `meeting-room-events` has a producer and **no consumer**. Messages accumulate on the topic with nothing subscribed to `CONSUMER_GROUP = "transcription-service"`.

---

## Tests — what they actually prove

`tests/test_devseed.py` (75 lines) — proves only pure-function properties: `chunk_id` determinism across runs (`:15-25`), no collision across meetings (`:27-30`), interim/final interleaving with a shared `chunk_id` (`:33-42`), speaker round-robin and monotonic timestamps (`:45-54`), that seeded text contains PII shapes so redaction is exercised (`:57-65`), and that `main()` returns 2 without `TRANSCRIPTION_DEV_SEED` (`:68-75`). **It never runs `seed()` and never touches a database.** Its own docstring says the persistence behavior is covered by `test_persistence_finals_only.py`, which is honest.

`tests/test_meeting_room_consumer.py` (147 lines) — 5 tests, all against `process_decoded_message` / `handle_meeting_room_event`, never `run_consumer`: redelivered `joined` does not reopen a stream (`:59`), `recording_ready` is acked with no session effect (`:77`), missing header raises `PoisonMessage` (`:99`), poison is dead-lettered and not re-raised (`:110`), a tombstone (`None` value) is dead-lettered rather than killing the loop (`:131`). These prove the **decode→effect seam and the DLQ discipline**. They prove nothing about the live broker harness, which is marked `# pragma: no cover` at `meeting_room_consumer.py:186`.

`tests/test_healthz.py:16-19` — asserts the `/internal/audio` WebSocket handshake completes. Its own comment calls it a scaffold: "the server accepts then closes; a clean handshake is the contract we assert." **This test passing is not evidence of an audio path; it is evidence of the stub.**

Other transcription tests worth noting as genuinely load-bearing: `test_persistence_finals_only.py` (213), `test_rls_isolation.py` (138), `test_deepgram_chunk_id.py` (163), `test_redaction.py` (122), `test_partition_privileges.py` (76), `test_deepgram_key_never_leaks.py` (57), `test_compliance_gate.py` (100).

---

## LiveKit ownership summary

- **Token minting is owned by `meeting-service`** — `mytheus-backend/services/meeting_service/transport.py:112-133` (`api.AccessToken`, `with_ttl` taking a `timedelta`), exposed at `POST /api/meetings/{meeting_id}/token` (`services/meeting_service/routes.py:154`).
- **The UI holds no LiveKit secret.** `mytheus-ui/src/app/api/livekit-token/route.ts:11-14` forwards the caller's Clerk session to `api-gateway` and passes the minted token back. `:37-64` passes through 403/404/428 distinctly.
- **A documented path correction:** `route.ts:16-24` records that plan 3-4 §3.4 and CLAUDE.md both say `POST /api/v1/meetings/:id/token`, but that path routes to `core-api` which does not mint tokens; the shipped mint is `meeting-service` at `/api/meetings/...`. They followed the shipped route and recorded the docs as the bug. Worth relaying — it is a doc defect in two places.
- **`transport.py:31`** is the only LiveKit SDK import in `mytheus-backend`, enforced by `tests/test_livekit_import_boundary.py`.
- **Webhooks** are implemented (`webhook.py`, `verify_webhook` at `transport.py:143-148` using `api.TokenVerifier` + `WebhookReceiver`) and produce `meeting-room-events`.
- **`transcription-service` reads `LIVEKIT_*` config** (`config.py:40-42`) but never uses it — no LiveKit SDK import, no handshake validation on `/internal/audio`.

---

## VERDICT

Real ASR is not implemented: the only Deepgram client is blocked by a source-constant compliance gate that no configuration can flip, and the `/internal/audio` LiveKit hook that would feed it is a four-line accept-then-close stub with zero callers anywhere in the program — the audio path is absent at every hop from microphone to provider, not merely misconfigured. What is built downstream of the missing audio is genuinely good work: the Kafka consumer, the interim/final split with its atomic outbox+row dedupe, the partitioned RLS'd `transcript_chunks` table, and a 639-line `realtime-gateway` with 15 test files are real, and the UI's Socket.io contract matches the gateway field for field while the storage destination matches `core-api`'s read path exactly — the two disconnects the brief suspected are not there. The disconnects that are there are packaging and defaults: neither service has a Dockerfile or a compose entry, so `NEXT_PUBLIC_REALTIME_WS_URL` has no URL it could ever point at, while `meeting-service` ships to prod producing `meeting-room-events` that nothing consumes. The sharpest live risk is `providers/factory.py:20-23`: with `DEEPGRAM_API_KEY` unset — which is every environment — `SessionTracker` opens a `FakeProvider` whose canned "hello there, how are you" script is written to real `transcript_chunks` rows and published to the real `transcript-chunks` topic with no dev guard and no marker distinguishing it from a genuine transcript, an asymmetry `devseed.py`'s own `TRANSCRIPTION_DEV_SEED=1` gate shows the authors understood elsewhere.
