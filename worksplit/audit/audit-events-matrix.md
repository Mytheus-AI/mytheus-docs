# Event Backbone Integration Audit — `mytheus-events` + cross-repo reality

Read-only audit. Root: `/Users/sushan/Projects/Mytheus`. All claims cited to `path:LINE` in real
schema/producer/consumer code, not to `.md` status files. Worktrees (`.worktrees/`) and vendored
`node_modules`/`.venv` excluded throughout.

---

## TOPIC INVENTORY

21 topics, all Avro, all at v1 only, one file each at `mytheus-events/schemas/<topic>/v1.avsc`.

- Shared header: `mytheus-events/schemas/_envelope/EventEnvelopeHeader.avsc:1` — 8 frozen fields,
  namespace `io.nexus.events`: `event_id`(uuid), `idempotency_key`, `org_id`(uuid),
  `schema_version`(int, default 1), `producer`, `occurred_at`(timestamp-micros),
  `correlation_id`(nullable), `causation_id`(nullable). Every topic value opens with it.
- DLQ wrapper: `mytheus-events/contracts/DlqEnvelope.avsc:1` — `original_topic`,
  `original_partition`, `original_offset`, `org_id`, `idempotency_key`, `consumer_group`,
  `failure_class`(enum DESERIALIZE|SCHEMA_INCOMPAT|BUSINESS_INVALID|MAX_RETRIES), `error`,
  `attempts`, `first_seen_at`, `raw_value_b64`.
- Canonical topic list: `mytheus-events/codegen/_schemas.py:32-38` (`TOPICS`).
- Subjects: 21 `subjects/{topic}-value.json`, each `{subject, schema, compatibility}`.
- Compatibility: every subject `BACKWARD`, default `BACKWARD` —
  `mytheus-events/subjects/compatibility.json:1-26`.
- Partition key: `org_id` on 19 topics; composite `{org_id}:{meeting_id}` on `transcript-chunks`
  and `agent-stream` — `mytheus-events/codegen/_schemas.py:44` (`COMPOSITE_PARTITION_KEY_TOPICS`),
  ADR-0017.
- No key schemas anywhere — the partition key is the raw string.

### Payload shapes (each is `header` + the fields listed)

| Topic | Record name | Payload fields |
|---|---|---|
| transcript-chunks | TranscriptChunk | meeting_id(uuid), chunk_id(uuid), speaker_id, start_ms, end_ms, text, is_final, confidence, redacted |
| document-chunks | DocumentChunk | document_id(uuid), element_id, element_type(enum Title/NarrativeText/Table/ListItem), text, position |
| agent-results | AgentResult | meeting_id, window_id, user_id, agent(enum), kind(enum), relevance_score, confidence, content, payload_json, source_chunk_ids(array) |
| agent-stream | AgentStreamDelta | meeting_id, user_id, window_id, seq, delta, done |
| decisions | Decision | decision_id, meeting_id, content, rationale, confidence, status(enum), confirmed_by |
| actions | Action | action_id, meeting_id, content, assignee_id?, due_date?, confidence, status(enum) |
| notifications | Notification | source_event_id, channel(enum), user_id, priority, title, body, link? |
| agent-requests | AgentRequest | requester_agent_id, responder_target, query, deadline, correlation_id |
| agent-responses | AgentResponse | correlation_id, responder_agent_id, result, success, latency_ms |
| tenant-events | TenantEvent | event_type(enum), org_slug, plan_code |
| meeting-events | MeetingEvent | meeting_id, event_type(enum created/updated/ended), livekit_room? |
| meeting-room-events | MeetingRoomEvent | meeting_id, room, event_type(enum), participant_id? |
| integration-events | IntegrationEvent | provider(enum), event_type, external_id, payload_json |
| usage-events | UsageEvent | user_id?, event_type(enum), quantity(double), source_id |
| billing-events | BillingEvent | event_type(enum), subscription_id?, detail_json |
| graph-events | GraphEvent | event_type(enum), edge_label, from_id, to_id |
| embedding-events | EmbeddingEvent | collection, point_count, meeting_id? |
| presence-events | PresenceEvent | meeting_id, user_id, event_type(enum) |
| notification-delivery-events | NotificationDeliveryEvent | notification_id, channel, status(enum), detail? |
| user-sync-events | UserSyncEvent | clerk_event_id, clerk_user_id, event_type(enum), email? |
| admin-events | AdminEvent | event_type(enum), actor_id, target_type, target_id, detail_json |

### `scripts/create_topics.sh` vs the declared list

`mytheus-events/scripts/create_topics.sh:53-74` creates exactly the 21 canonical topics plus one
paired `<topic>.dlq` each = 42, and prints "42 expected in total" (line 76). **No mismatch against
the contract list** — one `mk`/`mk_compact` line per entry in `TOPICS`, same names.

Partition counts and retention are inline in the script (not derived from any file, so they can
drift from canon silently, but they currently match the documented values):

- `transcript-chunks` 180p/7d · `document-chunks` 24p/7d · `agent-results` 120p/3d ·
  `agent-stream` 120p/1h · `decisions` 24p/30d · `actions` 24p/30d · `notifications` 24p/3d ·
  `agent-requests` 12p/1d · `agent-responses` 12p/1d
- `tenant-events` 6p **compacted** · `meeting-events` 24p/7d · `meeting-room-events` 24p/3d ·
  `integration-events` 12p/7d · `usage-events` 48p/7d · `billing-events` 12p/30d ·
  `graph-events` 6p/7d · `embedding-events` 12p/1d · `presence-events` 12p/1h ·
  `notification-delivery-events` 6p/30d · `user-sync-events` 6p/7d · `admin-events` 6p **compacted**
- Every `.dlq` is 1 partition / 14d retention (`create_topics.sh:51`).
- Replication defaults to `-r 1` (single-broker VPS per ADR-0016), overridable via `REPLICAS`.
- The script is genuinely idempotent: `TOPIC_ALREADY_EXISTS` is treated as success, anything else
  aborts (`create_topics.sh:27-45`).

### Idempotency-key formulas — defined for only 8 of 21

`mytheus-events/codegen/_schemas.py:52-61` defines `IDEMPOTENCY_KEYS` for:
`transcript-chunks` (`chunk:{chunk_id}`), `document-chunks`
(`docchunk:{document_id}:{element_id}`), `agent-results`, `decisions`, `actions`, `notifications`,
`agent-requests`, `usage-events`.

The other 13 topics have **no canonical key formula**, so each producer invents one:
- `meeting:{meeting_id}:created` — `mytheus-backend/services/core_api/routes/meetings.py:224`
- `meeting:{meeting_id}:{event_type}:{updated_at}` — `.../routes/meetings.py:434`
- `org:{org_id}:provisioned` — `mytheus-backend/services/identity_tenant_service/provisioning.py:147`
- the LiveKit webhook `dedupe_key` — `mytheus-backend/services/meeting_service/webhook.py:196,248`

These are reasonable but unreviewed by the contract repo, and the outbox unique index
`(topic, idempotency_key)` is the only thing enforcing them.

---

## PRODUCER/CONSUMER MATRIX

"REAL" = an actual produce or subscribe call site in shipping code. Docstrings, TODOs and
scaffold `main.py` files are NOT counted as producers or consumers.

| # | Topic | PRODUCED BY (repo/file:line) | CONSUMED BY (repo/file:line) | Schema matches both sides? | Class |
|---|---|---|---|---|---|
| 1 | `transcript-chunks` | **REAL** — `mytheus-realtime/services/transcription-service/src/transcription_service/persistence.py:31` (`TOPIC`), value built at `:64` from typed `TranscriptChunk`; written to the **shared `public.outbox`** via `.../transcription_service/outbox.py:37`, drained by `mytheus-backend/services/relay/main.py` | **REAL ×2** — `mytheus-intelligence/services/embedding_service/consumer.py:51` (`topics = ["transcript-chunks","document-chunks"]`, group `embedding-service.embed-on-ingest`); `mytheus-intelligence/services/search_service/consumer.py:78` (group `search-service.transcript-index`) | YES — producer uses a typed mirror of the `.avsc` (`.../events/transcript_chunk.py:1-15`); consumers decode via live Schema Registry (`mytheus-intelligence/libs/shared/avro_codec.py:64`) | CONNECTED |
| 2 | `document-chunks` | **REAL** — `mytheus-intelligence/services/ingestion_service/pipeline.py:136` (`emit("document-chunks", f"docchunk:{document_id}:{chunk_element_id}", {...})`) | **REAL** — `mytheus-intelligence/services/embedding_service/consumer.py:51` | YES — emitted dict `{document_id, element_id, element_type, text, position}` matches `schemas/document-chunks/v1.avsc` exactly | CONNECTED |
| 3 | `meeting-events` | **REAL ×2** — `mytheus-backend/services/core_api/routes/meetings.py:217` (`created`) and `:423` (`updated`/`ended`, type chosen at `:418`) | **REAL** — `mytheus-backend/services/meeting_service/consumer.py:86` (`TOPIC`), `:270` (`consumer.subscribe([TOPIC])`) | YES — `{meeting_id, event_type, livekit_room}` matches | CONNECTED |
| 4 | `meeting-room-events` | **REAL ×2** — `mytheus-backend/services/meeting_service/webhook.py:188` (`recording_ready`), `:242` (`joined`/`left`) | **REAL** — `mytheus-realtime/services/transcription-service/src/transcription_service/meeting_room_consumer.py:61` (`TOPIC`), `:224` (`subscribe`) | YES — `{meeting_id, room, event_type, participant_id}` matches; consumer validates against the registered `meeting-room-events-value` subject (`meeting_room_consumer.py:147`) | CONNECTED |
| 5 | `presence-events` | **REAL** — `mytheus-realtime/services/realtime-gateway/src/presenceEventsProducer.ts:112` (`TOPIC`), subject pinned at `:114`, wired in `.../src/server.ts:178` | **REAL** — `mytheus-backend/services/core_api/consumer.py:55` (`TOPIC`), `:142` (`subscribe`) | YES — `{meeting_id, user_id, event_type}`; note the producer is **direct-to-Kafka, no outbox** (`presenceEventsProducer.ts:6`), deliberate | CONNECTED |
| 6 | `embedding-events` | **REAL ×2** — `mytheus-intelligence/services/embedding_service/consumer.py:98` (empty-text ack) and `:114` (normal ack) | **NONE** anywhere | Producer side matches `{collection, point_count, meeting_id}` | **DEAD LETTER** (arguably by design — README lists consumers as "(lag / metrics)") |
| 7 | `tenant-events` | **REAL** — `mytheus-backend/services/identity_tenant_service/provisioning.py:139` (`{event_type:"provisioned", org_slug, plan_code}`) | **NONE** — declared consumers `billing-metering-service`, `admin-ops-service`, `workflow-service` are all scaffolds | Producer matches schema | **DEAD LETTER — functional bug.** `mytheus-backend/libs/shared/org_resolver.py:32-34` states a suspension only takes effect once a `tenant-events` consumer invalidates the cache. That consumer does not exist, so suspension propagation is silently broken. |
| 8 | `integration-events` | **NONE** — declared producer `integration-service` is an 11-line scaffold: `mytheus-backend/services/integration_service/main.py:1-11` (`# TODO: add routers for /oauth/{provider} and /webhooks/{provider}`) | **REAL, fully built** — `mytheus-intelligence/services/ingestion_service/consumer.py:75` (`topics = ["integration-events"]`, group `ingestion-service.attachment-to-ingest`), entrypoint `.../ingestion_service/worker.py`, S3 fetch at `libs/shared/object_storage.py:148` | Consumer expects `{provider, event_type, external_id, payload_json}` — matches the schema; nothing produces it | **PHANTOM** (most conspicuous — a complete consumer that can never fire) |
| 9 | `notifications` | **NONE** — README says "multiple"; no call site in any repo | **STUB** — `mytheus-backend/services/notification_service/consumer.py:17` subscribes, but `_handle()` at `:37-42` is a `TODO(P1)` no-op that discards the message | n/a | **PHANTOM** + stub consumer |
| 10 | `agent-results` | **NONE** — declared producer `agent-runtime-service` is a scaffold: `mytheus-intelligence/services/agent_runtime_service/main.py:1-13` ("Scaffold only: boots and answers `/healthz`") | **STUB** — same `notification_service/consumer.py:17` no-op | n/a | **PHANTOM** + stub consumer |
| 11 | `agent-stream` | **NONE** | **NONE** — the gateway's live path is a **Redis** pub/sub caption channel (`mytheus-realtime/services/realtime-gateway/src/server.ts:314`), not this Kafka topic; `:80` and `:285` are comments about a "not-yet-" wired lane | n/a | **ORPHAN** |
| 12 | `decisions` | **NONE** — `mytheus-backend/services/core_api/main.py:6-7`: events are emitted ONLY on `POST /decisions/:id/confirm`, and "Neither route exists yet"; `:22` `# TODO: /decisions /actions /worklogs /suggestions land with their own bands` | **NONE** — `knowledge-graph-service` is a scaffold (`mytheus-intelligence/services/knowledge_graph_service/main.py:1-13`) | n/a | **ORPHAN** |
| 13 | `actions` | **NONE** — same as `decisions` | **NONE** — declared consumers knowledge-graph / workflow / integration all scaffolds; `mytheus-backend/services/workflow_service/workflows/action_followup.py:1` is a docstring reference only | n/a | **ORPHAN** |
| 14 | `agent-requests` | **NONE** | **NONE** | n/a | **ORPHAN** |
| 15 | `agent-responses` | **NONE** | **NONE** | n/a | **ORPHAN** |
| 16 | `usage-events` | **NONE** — declared producers `agent-runtime-service` and `rag-service` are both scaffolds (`mytheus-intelligence/services/rag_service/main.py:1-13`) | **NONE** — `billing-metering-service` is a scaffold (`mytheus-backend/services/billing_metering_service/main.py:1-11`) | n/a | **ORPHAN** |
| 17 | `billing-events` | **NONE** — billing-metering-service scaffold | **NONE** — only a Temporal workflow docstring: `mytheus-backend/services/workflow_service/workflows/dunning.py:1` | n/a | **ORPHAN** |
| 18 | `graph-events` | **NONE** — knowledge-graph-service scaffold | **NONE** — admin-ops-service scaffold | n/a | **ORPHAN** |
| 19 | `notification-delivery-events` | **NONE** — only a docstring claim: `mytheus-backend/services/notification_service/consumer.py:4` "Produces: notification-delivery-events" (no call site; `_handle` is a no-op) | **NONE** | n/a | **ORPHAN** |
| 20 | `user-sync-events` | **NONE** — `mytheus-backend/services/identity_tenant_service/routes/auth.py:46` defines `TOPIC = "user-sync-events"`, but the module docstring at `:19-21` states explicitly: "Per-user changes write NO outbox row. There is no per-user propagation topic in canon (ADR-0010 D4)". The constant is used **only** as an inbox/dedupe namespace for the Clerk Svix webhook, never as a Kafka topic. | **NONE** | n/a | **ORPHAN** — a declared Kafka topic whose name is used as a Postgres/Redis dedupe key. The topic and its subject exist and will always be empty. |
| 21 | `admin-events` | **NONE** — admin-ops-service scaffold | **NONE** — only a docstring: `mytheus-backend/services/workflow_service/workflows/rtbf.py:3` "Triggered by admin-events (erasure_requested)" | n/a | **ORPHAN** |

### Matrix score

- **6 / 21** topics have BOTH a real producer and a real consumer (1–5, and 6 if `embedding-events`'
  "no consumer" is accepted as designed).
- **2** dead-letter, **3** phantom, **11** orphan.
- **0** topics show payload-level schema drift.

---

## ORPHAN / DEAD-LETTER / PHANTOM / DRIFT

### ORPHAN (11) — declared, nobody produces or consumes
`agent-stream`, `decisions`, `actions`, `agent-requests`, `agent-responses`, `usage-events`,
`billing-events`, `graph-events`, `notification-delivery-events`, `user-sync-events`,
`admin-events`.

Root cause is uniform: six services are `main.py`-only scaffolds that between them own every one
of these topics —
`mytheus-intelligence/services/agent_runtime_service/main.py`,
`.../knowledge_graph_service/main.py`,
`.../rag_service/main.py`,
`mytheus-backend/services/billing_metering_service/main.py`,
`.../integration_service/main.py`,
`.../admin_ops_service/main.py`.

`user-sync-events` is a distinct case: it is not blocked on a service, it is a topic that canon
(ADR-0010 D4) decided should not carry traffic at all, yet it still occupies a topic, a subject
and a `.dlq`.

### DEAD LETTER (2) — produced, nothing consumes
- `embedding-events` — `mytheus-intelligence/services/embedding_service/consumer.py:98,114`.
  README lists consumers as "(lag / metrics)", so an empty consumer set may be intentional; there
  is however no metrics consumer either, so nothing reads the lag acks today.
- `tenant-events` — `mytheus-backend/services/identity_tenant_service/provisioning.py:139`.
  **Not intentional.** `mytheus-backend/libs/shared/org_resolver.py:32-34` makes cache
  invalidation on org suspension conditional on a `tenant-events` consumer that does not exist.

### PHANTOM (3) — consumed, nobody produces
- `integration-events` — consumer at `mytheus-intelligence/services/ingestion_service/consumer.py:75`
  is complete (S3 fetch, RLS-scoped pipeline, inbox dedupe, DLQ) and unreachable.
- `notifications` — subscribed at `mytheus-backend/services/notification_service/consumer.py:17`,
  handler is a no-op.
- `agent-results` — same subscription, same no-op.

### SCHEMA DRIFT — none at the payload level; three structural drifts

Every producer's hand-built dict was checked field-for-field against its `.avsc` and matches.
Both Python tiers serialize through the **live Schema Registry** rather than a vendored copy —
`mytheus-intelligence/libs/shared/avro_codec.py:36-47` (`get_latest_version`, `auto.register.schemas: false`)
and `mytheus-backend/services/relay/main.py:45-60` — so a drifted payload fails at encode time
rather than shipping silently. The TS producer pins the subject and fetches the id at connect:
`mytheus-realtime/services/realtime-gateway/src/presenceEventsProducer.ts:114`, `:186`.

The real drifts are structural:

1. **DLQ topic-name split — unprovisioned topics.**
   `mytheus-intelligence/libs/shared/consumer.py:253-258` routes to `{topic}.{group}.dlq`
   (e.g. `transcript-chunks.embedding-service.embed-on-ingest.dlq`), justified at `:222-227` so
   two consumer groups on one source topic don't share a DLQ. But
   `mytheus-events/scripts/create_topics.sh:51` only ever creates `{topic}.dlq`.
   `mytheus-backend` and `mytheus-realtime` both use the plain form —
   `mytheus-backend/services/core_api/consumer.py:169`,
   `mytheus-backend/services/meeting_service/consumer.py:124`,
   `mytheus-realtime/services/realtime-gateway/src/presenceEventsProducer.ts:113`.
   Consequence: intelligence's DLQ topics rely on broker auto-create — exactly what
   `mytheus-platform/deploy/deploy.sh:79-86` refuses to tolerate — and any redrive tool keyed on
   topic name will miss them.

2. **DLQ payload format contradicts `contracts/DlqEnvelope.avsc`.**
   The contract is an Avro record with `raw_value_b64`. Every actual DLQ write is JSON:
   `mytheus-intelligence/libs/shared/consumer.py:246` (`produce(... json ...)`, rationale at
   `:215-218` — no `.dlq` subject is registered), and
   `mytheus-realtime/services/realtime-gateway/src/presenceEventsProducer.ts:166`
   (`JSON.stringify(dlqEnvelope)`). The Avro DLQ contract is unimplemented in all three repos and
   the contract file was never reconciled.

3. **Three outbox tables, one cross-repo write.**
   - `mytheus-backend/libs/shared/outbox.py:34` → `public.outbox`, drained by
     `mytheus-backend/services/relay/main.py`.
   - `mytheus-intelligence/libs/shared/outbox.py:37` → `intelligence.outbox`, drained by
     `mytheus-intelligence/services/embedding_service/relay_main.py` ("One relay per repo" —
     `mytheus-intelligence/services/ingestion_service/worker.py:7-10`).
   - `mytheus-realtime`'s transcription-service writes into **mytheus-backend's** `public.outbox`
     cross-repo: `.../transcription_service/outbox.py:1-19`.
     So `transcript-chunks` only reaches Redpanda if the **backend** relay process is running.
     That coupling is documented only in that docstring.

---

## SHARED-SCHEMA ADOPTION

**Zero adoption. No repo imports the generated clients.** A grep for `mytheus_events` /
`@mytheus/events` across all five repos returns only docstrings explaining why not, plus one test.

### Python client
`mytheus-events/clients/python/mytheus_events/` — 23 modules (one per topic + `envelope.py`).
The only importer anywhere is `mytheus-intelligence/libs/shared/clients.py:21`
(`import mytheus_events` inside a lazy loader), and its only caller is
`mytheus-intelligence/tests/test_healthz.py:33`, which **asserts the package is not installed**
(`pytest.raises(ModuleNotFoundError, match="mytheus-events is not installed")`).

The stated blocker is **stale**. `mytheus-intelligence/libs/shared/avro_codec.py:6-8` says "the
root `mytheus-events/pyproject.toml` has no `[build-system]` table, so `pip install -e` against it
does not currently work". `mytheus-events/pyproject.toml:29-35` now has hatchling with
`[tool.hatch.build.targets.wheel] packages = ["clients/python/mytheus_events"]`. The wheel is
buildable today; nobody depends on it. The client is also pydantic-models-only — no Avro
ser/deser methods — so even installed it would not replace `avro_codec.py`.

### TypeScript client
`mytheus-events/clients/typescript/` — 23 modules, `package.json` names it `@mytheus/events`
v0.1.0 with a `dist/` build. It is never built, never published, and `mytheus-realtime` has no
workspace link into it: `mytheus-realtime/libs/envelope.ts:7-9` spells this out and hand-mirrors
the envelope instead.

### Hand-rolled mirrors in production code
- `mytheus-backend/libs/shared/events.py:41-55` — the 8-field header assembled by hand.
- `mytheus-realtime/services/transcription-service/src/transcription_service/events/envelope.py:5-14`
- `mytheus-realtime/services/transcription-service/src/transcription_service/events/transcript_chunk.py:1-15`
- `mytheus-realtime/libs/envelope.ts:3-9` and `mytheus-realtime/libs/envelope.ts:59` (DLQ mirror).

### Mitigating factor, and the drifts that already happened
Because the Python tiers resolve schemas from the live registry, hand-rolled payload drift is a
loud runtime failure, not silent corruption. It has already bitten twice and been fixed:
- `mytheus-backend/services/core_api/routes/meetings.py:410-417` — the producer emitted
  `event_type: "updated"` unconditionally and never `"ended"`, despite the schema having all three
  symbols since day one.
- `mytheus-backend/tests/test_meeting_events_consumer.py:298` and
  `mytheus-backend/services/meeting_service/consumer.py:15-17` — the **generated client** had
  drifted from `schemas/meeting-events/v1.avsc` and had to be fixed in `mytheus-events` separately.
- `mytheus-realtime/libs/keys.ts:19-20` — putting a raw Clerk id (`org_2abc…`) where a UUID
  `org_id` belonged routed every `presence-events` record to `presence-events.dlq` as poison. A
  shipped bug, caught only because the Avro schema types `org_id` as a UUID.

### Contract-repo quality (the part that is genuinely good)
- `mytheus-events/.github/workflows/ci.yml:26-60` — the BACKWARD gate runs against a live Redpanda
  Schema Registry service container, and critically registers the **base-branch** schemas first
  (`:48-57`) so the compat check has a prior version to compare against instead of passing
  trivially on an empty registry.
- Tests: `tests/test_contract.py`, `tests/test_backward_compat.py`, `tests/test_golden_roundtrip.py`
  plus 21 golden fixtures in `fixtures/`.
- `codegen/` has real generators: `py_from_avsc.py`, `ts_from_avsc.mjs`, `gen_subjects.py`,
  `register.py`, `check_compat.py`, `validate.py`, `gen_fixtures.py`.

---

## BROKER & DEPLOY

**Broker: Redpanda** (not Kafka), `redpandadata/redpanda:v24.1.1`, in the deploy compose at
`mytheus-platform/deploy/compose.prod.yml:117-129`, with the built-in Schema Registry on
`:8081` (`:126`) and Kafka on `:9092`/`:29092` (`:124-125`). Volume `redpandadata` at `:416`.

**Provisioning is verified, not performed.** `mytheus-platform/deploy/deploy.sh:60-104`
deliberately does not create topics (to avoid a second copy of the 42 partition/retention
settings); it counts topics and subjects and **fails the deploy** if `< 42` topics or `< 21`
subjects, printing the manual remedy (`BROKERS=… bash scripts/create_topics.sh` +
`make register SR_URL=…`). Note it counts, it does not check names — 42 wrong topics would pass.

### The deploy gap that dwarfs everything else

Every Kafka consumer in this program is a **standalone worker process**, by explicit convention:
`mytheus-backend/services/meeting_service/main.py:15-17` — "The `meeting-events` consumer runs as
its own worker process (`python -m services.meeting_service.consumer`), not inside this FastAPI
app — same convention as `services/core_api/consumer.py`."

`mytheus-platform/deploy/compose.prod.yml` runs: `postgres`, `redis`, `redpanda`, `temporal`,
`migrate`, `web`, `relay`, `workflow`, `api-gateway`, `bff`, `identity-tenant-service`,
`meeting-service`, `integration-service`, `billing-metering-service`, `admin-ops-service`,
`notification-service`, `ui` (service keys at `:78`–`:364`).

**Not one consumer worker is in that file.** No `python -m services.core_api.consumer`, no
`python -m services.meeting_service.consumer`, no notification consumer. And no
`mytheus-intelligence` service (embedding, ingestion, search, agent-runtime, rag, knowledge-graph)
and no `mytheus-realtime` service (realtime-gateway, transcription-service) appears at all.

The only event-plane process deployed is `relay` (`compose.prod.yml:210-221`, running as
`mytheus_relay`, `python -m services.relay.main`). In the deployed stack the relay produces into
Redpanda and **nothing on the other end is running** — including the two consumers
(`core_api/consumer.py`, `meeting_service/consumer.py`) that are fully built inside the very
image that is deployed.

---

## VERDICT

The contract layer is genuinely good — 21 real Avro schemas with a frozen 8-field envelope, one
`{topic}-value` subject each under a BACKWARD gate that CI enforces against a live registry with
the base-branch schemas registered first, and a `create_topics.sh` that matches the declared
inventory exactly with no name or count mismatch; there is no payload-level schema drift anywhere
because both Python tiers encode through the live Schema Registry rather than a vendored copy.
But the backbone connects almost nothing: only 6 of 21 topics have a real producer *and* a real
consumer, 11 are fully orphaned behind six `main.py`-only scaffold services, and 3 are phantoms —
most conspicuously `integration-events`, where a complete S3-fetch ingestion consumer with inbox
dedupe and DLQ routing can never fire because `integration-service` is 11 lines of TODO, while
`tenant-events` is a dead letter that silently breaks org-suspension propagation by
`org_resolver.py`'s own admission. Shared-schema adoption is zero: every repo hand-mirrors the
envelope, the generated Python and TypeScript clients have no importer outside a test that asserts
the package is absent, and the packaging blocker that justified this was fixed in
`pyproject.toml` without anyone adopting the result. The single largest disconnect is operational
rather than contractual — every consumer is a standalone worker process and none appear in
`compose.prod.yml`, so production currently runs a relay producing into Redpanda with nothing
consuming from it, and neither `mytheus-intelligence` nor `mytheus-realtime` is deployed at all.

---

## UNVERIFIED

- Whether the 21 subjects are actually registered in any running Schema Registry — no live broker
  was reachable from this audit; only the registration tooling and CI job were read.
- Whether `mytheus-mobile` references any topic names (outside the grep set for this task).
- Whether `mytheus-intelligence` / `mytheus-realtime` services are deployed by some path outside
  `mytheus-platform/deploy/` (e.g. a separate Dokploy project or the KEDA manifest at
  `mytheus-intelligence/services/embedding_service/keda-scaledobject.yaml`, which implies a
  Kubernetes target that has no counterpart in this repo set).
- Whether `agent-stream`'s intended path is Kafka at all — `realtime-gateway/src/server.ts:314`
  uses Redis pub/sub for captions, and `:80`/`:285` describe the Kafka lane as future work.
