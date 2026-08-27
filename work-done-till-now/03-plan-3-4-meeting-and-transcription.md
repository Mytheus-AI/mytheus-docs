# 03 Plan 3-4 Meeting Platform and Real-time Transcription

| Field | Value |
|---|---|
| Band | Plan 3-4, phases P3 (Meeting Platform Integration, W6-8) and P4 (Real-time and Transcription, W9-12) |
| Plan file | [plan/plan-3-4-meeting-and-transcription.md](../plan/plan-3-4-meeting-and-transcription.md) |
| Status | **DONE WITH GAPS.** Every section of the plan this band was authorized to build is built and tested against real infrastructure where real infrastructure exists (Postgres, Redis, Redpanda, Schema Registry). `web-app` room UI and captions UI (sections 3.4, 3.8) were deliberately **not built**, by explicit product-owner decision: `mytheus-ui` is out of scope for this band. `mytheus-mobile` section 7 is deferred for the same decision plus missing LiveKit RN credentials. No LiveKit, Deepgram, or S3 credentials exist anywhere in the program; every acceptance check that needs one is marked **BLOCKED**, never claimed green. |
| Complete | Canon amendments plus two ADRs (`mytheus-docs`); partition-key validator scoping and schema fixes (`mytheus-events`); `meeting-service`, `core-api` chat and speaker-mapping, the consent gate (`mytheus-backend`); `realtime-gateway` and `transcription-service` (`mytheus-realtime`) |
| Deliberately not built | `mytheus-ui` sections 3.4 and 3.8 (meeting room video grid, live captions/transcript/speaker-mapping UI) - out of scope by decision. `mytheus-mobile` section 7 (live media, presence, captions) - deferred, same decision plus no LiveKit RN credentials. |
| Blocked, not built | Every acceptance check requiring a live LiveKit account, a live Deepgram account, or an S3-compatible object store: real end-to-end token verification, `recording.ready` landing an object in `nexus-recordings`, any real Deepgram stream, the EU-cell half of the residency check, the reconnect-SLO metric, and the `k6` token-mint p95 smoke. |
| Last verified | 2026-08-05/06, by running each repo's own suite in its Plan 3-4 worktree against the shared dev stack (Postgres, Redis, Redpanda, Schema Registry) |

Clerk credentials arrived mid-band (2026-08-05) and are live: `mytheus-ui` serves real sign-in against `firm-grouse-59.clerk.accounts.dev`, and `mytheus-backend` verifies real RS256 tokens against that JWKS. This does **not** unblock LiveKit, Deepgram, or S3; those remain absent and every task built against them stays on the Protocol-plus-Fake pattern with the live path marked BLOCKED.

---

## 1. What landed, per repo

### 1.1 `mytheus-docs`: two ADRs and the canon they ratify

Both DECISION-REQUIRED callouts in the plan (section 8) are resolved and recorded.

- **[ADR-0017](../adr/0017-transcript-chunks-composite-partition-key.md)** - `transcript-chunks` and `agent-stream` are composite-keyed `{org_id}:{meeting_id}`; every other topic, including `agent-results` (a candidate the ADR explicitly excludes), stays `org_id` alone. `_canon/03-event-taxonomy.md` amended accordingly.
- **[ADR-0018](../adr/0018-consent-entity-behind-flag.md)** - `meeting_participants.consented_at timestamptz NULL` / `consent_version text NULL` and `tenant_settings.consent_required boolean NOT NULL DEFAULT false` land in `_canon/02-data-model.md`. Narrowed post-implementation (see section 3) once the shipped gate turned out to check presence-or-absence rather than the ADR's original version-pinned re-consent language.
- **Propagation, first pass:** `plan-3-4` section 8 (both callouts marked RESOLVED with ADR citations, the false BLOCKED banner removed), `plan-5-6` section 8.1 (also corrects a second document's misclassification of `agent-results` as a composite candidate), `06-multi-tenancy.md` (qualified to "19 of 21" topics), and `adr/0001`, `adr/0007`, `adr/0009` (annotated with a forward pointer to ADR-0017, never rewritten - the repo's supersede-don't-rewrite rule).
- **Propagation, second pass, self-initiated:** a wider regex sweep, run to zero unscoped hits, found 11 more stale copies of the same pre-decision language, the most severe being a literal build-step instruction in `plan-3-4` itself ("Produce `transcript-chunks` ... partition key `org_id`") that an implementer could have followed verbatim into the wrong key. Also fixed: `adr/0007`'s "alternatives considered" table, `plan/README.md`, `plan/00-HUMAN-OVERVIEW.md`, `plan/00-PROJECT-UNDERSTANDING.md`, `plan-0`, `plan-7-9`, `plan-10-13`, `design/c4-l1-system-context.md`, `diagrams/07`, `diagrams/09` (a second miss beyond round one).
- `diagrams/05` and `diagrams/09` updated for consistency with the new canon, per the repo's binding rule that diagrams are a derived view of the same truth.
- 3 commits: `717a14f` (canon plus ADRs), `55e5549` (fix round: the reviewer's six findings plus the self-initiated sweep), `800cf4e` (ADR-0018 narrowed to match what shipped).

### 1.2 `mytheus-events`: the partition-key invariant, scoped not deleted

- `codegen/_schemas.py`'s `COMPOSITE_PARTITION_KEY_TOPICS` now contains exactly `{"transcript-chunks", "agent-stream"}`; the validator's org_id-alone rule is scoped to exempt those two rather than removed outright, and a new invariant requires `meeting_id` non-nullable on both (currently already true for both, so it only guards a future regression).
- `schemas/_envelope/EventEnvelopeHeader.avsc`'s doc string, which shipped to every one of the 21 registered subjects as user-facing Schema-Registry metadata, no longer claims `org_id` is unqualified "the Kafka partition key".
- `TranscriptChunk` gained a `redacted: boolean` field, registered BACKWARD against the live registry (subject now shows versions `[1, 2]`).
- `clients/python/mytheus_events/meeting_events.py` regenerated: it had drifted, typing `livekit_room: str` while the schema had gone `["null", "string"]` since an earlier commit. A full audit of all 21 topic schemas for the same drift class found no other offender.
- 3 commits: `4139d5a` (validator scoping, invariant, envelope fix), `0177d1f` (doc-string fix round), `1549e25` (regenerated client, prerequisite for `mytheus-backend`'s `meeting-events` consumer).
- Gates run against the live registry, not just parsed: `make validate` (21 topics), `make check-compatibility` (21 subjects BACKWARD), and the validator's failure mode re-proved twice by deliberately breaking a schema (an unqualified `org_id`, then a nullable `EventEnvelopeHeader.org_id`) and restoring it.

### 1.3 `mytheus-backend`: `meeting-service`, `core-api` additions, the consent gate

`services/meeting_service/` moves from a routeless stub to a real module.

- **`transport.py`** - `MeetingTransport` (Protocol), `LiveKitTransport` (the only file in the codebase importing `livekit-api`, grep-tested in both directions: it passes on the real routes and fails on a deliberately unsecured probe), `FakeTransport` for every test. Adapted to the real installed `livekit-api==1.1.1` signature, which differs from the plan's literal snippet (`with_ttl` takes a `timedelta`, `WebhookReceiver` takes a `TokenVerifier`).
- **`routes.py`** - `POST /api/meetings/{id}/token`. Deny-by-default (`403 NOT_A_PARTICIPANT` before the meeting-existence check, cross-tenant `404 not 403` proven by check ORDER through mutation testing, not just by outcome), the 428 consent gate, and a genuinely-cancelling 2 s `statement_timeout` (proved with `SELECT pg_sleep(2.5)` inside the same transaction shape the route uses, not merely that the `SET` succeeded).
- **`webhook.py`** - `POST /livekit/webhook`. HMAC-verified through the adapter, deduped on the inbox key `{room}:{event}:{participant_sid}` (`nexus:{org_id}:dedupe:livekit:{room}:{event}:{participant_sid}`), resolves `org_id` from LiveKit room metadata via a sanctioned second reader on `organizations` (`org_resolver.resolve_org_by_id`, added because the existing `resolve_org_id` takes a `clerk_org_id`, not an internal id).
- **`rooms.py`** - the single shared room-naming and `ensure_meeting_room` logic used by the token-mint route, the webhook, and the `meeting-events` consumer, so three independent writers of the same string cannot disagree by construction.
- **`meeting_events_consumer.py`** - a live `confluent-kafka` consumer loop (not just unit-tested business logic) that opens/closes `meeting_rooms` on `meeting.created` / `meeting.ended`, re-asserting `org_id` from the header and deduping via inbox.
- **Migrations:** `meeting_rooms` (`ENABLE` + `FORCE` RLS, `tenant_isolation`, `UNIQUE (org_id, meeting_id)`, `UNIQUE (org_id, livekit_room)`, and `ck_meeting_rooms_status_valid` - the CHECK constraint the plan's literal SQL omitted, added once its gap was flagged); the three consent columns via `866f7eedb0e7`; `meeting_rooms.livekit_metadata_synced_at` via `f3a7c9e0b21d` (see section 3 for why).
- **`core-api`:** `POST /chat` (`Idempotency-Key`, RLS-scoped write to `chat_messages`); the participant-check endpoint, resolving tenancy solely from the caller's own re-verified bearer token and returning a dedicated `ParticipantCheck` model carrying the resolved internal `org_id` and `user_id` (added rather than overloading the shared `Participant` model, which is also embedded in `Meeting.participants` where no per-participant org is resolved); `PATCH /meetings/:id/speakers/:speaker_id` with a database-level `BEFORE UPDATE` trigger making manual assignment authoritative over any suggested one.
- **Final suite state:** **200 passed, 1 failed.** The one failure is `tests/test_relay.py::test_relay_real_bus_roundtrip`, a known-flaky live-Redpanda-consumer-poll test unrelated to any file this band touched; it passes in isolation and on reruns. `ruff`, `black`, and `mypy --strict` clean; `check_authz.py` clean.

### 1.4 `mytheus-realtime`: `realtime-gateway` and `transcription-service`, both from a docs-only repo

**`realtime-gateway`** (Node, TypeScript, Socket.io):

- Socket.io server with the Redis adapter for cross-pod fan-out; rooms keyed by `meeting_id` (see section 3 for the deliberate `org_id` drop and its later correction).
- Presence as a Redis ZSET scored by last-heartbeat rather than a plain set: reads filter stale members immediately, a periodic per-pod sweep physically reaps them (proven against a real `kill -9` across two separately spawned OS processes: process A joins then is SIGKILLed with no disconnect handler ever firing, and process B - which never held a local socket in that meeting - does the reaping and re-broadcast), and the read-then-remove sweep step is a single atomic Lua script so two racing sweeps cannot both report the same reap.
- Chat relay to `core-api`'s `POST /chat`; the gateway never writes `chat_messages` directly.
- `presence-events` producer against the real broker, with a `presence-events.dlq` write on produce failure (plain JSON, no registered `.dlq` subject) rather than silent loss.
- Captions bridge: subscribes `transcription-service`'s caption channel on `nexus:{org_id}:pubsub:meeting:{meeting_id}` and delivers by a literal per-channel local-socket set, not Socket.io room membership - closing a real cross-tenant leak found empirically during Task 8 (see section 3).
- **42/42 vitest** across unit, integration (fake JWKS plus fake `core-api`), real-broker, and cross-pod tests. `npm run typecheck`, `make ci` (both stacks) green.

**`transcription-service`** (Python, FastAPI, its own Alembic chain):

- `TranscriptionProvider` (Protocol), `DeepgramProvider` (the compliance gate is a source constant, `COMPLIANCE_GATE_GREEN = False`, with no environment-variable override path, checked inside `open_stream` before any network I/O - a reviewer tried multiple bypass vectors, including direct construction and configuring a key, and could not open a stream), `FakeProvider`.
- Interim-versus-final split with the load-bearing test: 2 interims plus 1 final through `process_utterance` against real Postgres asserts **zero** `transcript_chunks` rows and **zero** outbox rows from the interims, then drains the (empty) outbox against the real broker and, for the final, consumes the produced `TranscriptChunk` back off a real `confluent_kafka` consumer and decodes it, checking the raw Kafka key is `{org_id}:{meeting_id}`.
- Deterministic PII redaction (`redaction.py`): email, phone, IBAN, credit card via Luhn, and a US SSN-shaped government-ID pattern, replacing spans with typed placeholders (`‹EMAIL›`, and so on). Gated by `tenant_settings.pii_redaction_enabled`, tested both on and off against real Postgres. The NER pass and the per-tenant reversible token vault the plan also asks for are **not implemented** - disclosed, not silently dropped (see section 3).
- Residency: `data_residency.region` selects the Deepgram regional WSS endpoint (`us`/`eu`; `ap` falls back to `us`), tested against real Postgres for both `us` and `eu`. The "chunks physically land in the EU cell" half is BLOCKED - no second regional Postgres/Redpanda cell exists anywhere.
- `transcript-chunks` producer: composite key, `chunk:{chunk_id}` idempotency, `chunk_id` minted deterministically as `uuid5(NAMESPACE_URL, "deepgram:{request_id}:{speaker_id}:{start_ms}:{end_ms}")` after two collision-closing rounds (see section 3).
- `meeting-room-events` consumer: the dedupe and session-open/close business logic is real and unit-tested. The live `confluent-kafka` subscribe/poll/commit I/O loop (`run_consumer` in `meeting_room_consumer.py`) is now wired, mirroring `mytheus-backend/services/meeting_service/consumer.py`'s shape: subscribe, poll, `AvroDeserializer` against the registered `meeting-room-events-value` subject, `process_decoded_message` (the testable seam that routes a `PoisonMessage` - missing header/`org_id`/`idempotency_key`/`meeting_id`/`event_type`, or a value that did not decode to a record - to `meeting-room-events.dlq` instead of crash-looping the partition), then `consumer.commit(message=msg)` only after the effect. `main.py`'s FastAPI `lifespan` starts it as a background task on boot and drains it on shutdown, wiring `SessionTracker`, the Redis client, and `select_provider` together for the first time. Three poison-message tests added (`test_missing_header_raises_poison_message`, `test_poison_message_is_dead_lettered_and_not_reraised`, `test_value_that_does_not_decode_to_a_record_is_dead_lettered`), all three deliberately free of the `tenant` fixture because the path they cover is rejected before any Postgres or Redis call. **Not run against real Postgres/Redis/Redpanda in this session** - no `uv`, Docker, or the shared dev stack were reachable from where this change was made; only `python -m py_compile` and a line-length/formatting audit were run. Needs a real `pytest -q` pass against the shared dev stack before this is claimed **MET** rather than **PARTIAL**.

- **Self-review round on the above, before it was called done.** Four defects were found in the first cut of the wiring and fixed; they are recorded because three of them were introduced by the act of moving the loop into the API process, and a future reader who "simplifies" the code back will reintroduce them.
  1. **Blocking `Consumer.poll` on the FastAPI event loop.** `meeting_service/consumer.py` owns its whole process, so a blocking poll there stalls an event loop with nothing else on it. Here the loop shares a process with `/internal/audio`, the LiveKit audio hook (it has to: `SessionTracker` holds each meeting's open `TranscriptionStream` in memory, so a separate consumer process could not reach it). Calling `poll` directly would have frozen the audio hook for up to a second at a time, continuously, against a caption SLO of p95 < 2 s. Now run via `loop.run_in_executor` (librdkafka releases the GIL while waiting, so the loop is genuinely free). `Consumer.commit` was left on the event loop deliberately - it defaults to `asynchronous=True` and only enqueues - and that is commented so it is not "fixed" into a blocking call.
  2. **`consumer.close()` racing a worker thread still inside `poll()`.** The first cut stopped the task by cancelling it, which surfaces at the `run_in_executor` await and runs `finally: consumer.close()` immediately - concurrent use of one `Consumer` from two threads, which confluent-kafka does not support and which is a known segfault hazard, on every pod shutdown. Replaced with an `asyncio.Event` stop flag: the loop finishes its in-flight poll and its in-flight message (a clean drain), then closes. `main.py` allows 10 s for that and only then falls back to cancelling.
  3. **A tombstone would have killed the loop rather than been dead-lettered.** `AvroDeserializer` returns `None` for a null value, and `_validate`'s `.get` on it raises `AttributeError`, not `PoisonMessage` - so it would escape the DLQ path, end the loop, and take this pod's transcription with it. `process_decoded_message` now types its input `Any` and converts a non-record into the ordinary DLQ path.
  4. **A clean shutdown logged a false alarm, and a real crash logged nothing.** The first cut caught `CancelledError` and `Exception` together and called `logger.exception` on both, so every normal shutdown emitted a scary traceback. Separated; and because a background task's death is otherwise silent until shutdown (the backend's equivalent takes its process down and Kubernetes restarts it - this one cannot rely on that), a done-callback now logs the death the moment it happens.

  Two smaller corrections in the same pass: `run_consumer` takes `brokers`/`schema_registry_url` as parameters instead of reading `os.environ` directly, because `config.py` is this service's documented single reader of the environment; and `main.py`'s Redis client is built with `decode_responses=True` to match `tests/conftest.py`, so the tested and the running configuration cannot diverge.
- `audit.py`: one `audit_logs` row per session-open, in the same transaction as the residency and redaction reads, recording both decisions - closes the compliance gate's fourth condition, found missing (and undisclosed) by the first review pass and fixed in the first fix round.
- **41/41 pytest**, all against real Postgres/Redis/Redpanda/Registry on the load-bearing paths, no mocks. `ruff`, `black`, `mypy --strict` clean; `make ci` green for both stacks.

### 1.5 `mytheus-ui`: untouched, by decision

Sections 3.4 (meeting room video grid, in-meeting chat and presence UI) and 3.8 (live captions, transcript view, speaker-mapping UI) were not built. This is a deliberate exclusion recorded in the governing decisions for this band, not an omission: `mytheus-ui` was declared out of scope. Clerk sign-in went live mid-band for reasons unrelated to this band's scope (see the header note); no meeting-surface code was added.

### 1.6 `mytheus-mobile`: untouched, deferred

Section 7 (live-session media, presence, captions, the real meeting clock, mic capture) was not started. Deferred for the same out-of-scope decision as `mytheus-ui`, compounded by the absence of LiveKit RN credentials that section 7's acceptance checks require regardless.

---

## 2. Definition of Done audit

Against section 10 of [plan-3-4-meeting-and-transcription.md](../plan/plan-3-4-meeting-and-transcription.md), reproduced verbatim per line.

### End of P3 (W8) - Meeting Platform demo and DoD

| DoD line | Verdict | Evidence or reason |
|---|---|---|
| `meeting-service` token mint (p95 < 250 ms) shipped | **PARTIAL** | The route is built and its deny-by-default, cross-tenant, and timeout behavior are mutation-tested. The p95 < 250 ms figure itself is **not proven**: `FakeTransport.mint_join_token` does no I/O, so it cannot stand in for real LiveKit latency, and no `k6` smoke was run because there is no LiveKit endpoint to smoke against. Disclosed by the implementer, not silently claimed. |
| `meeting-service` webhook (HMAC-verified, inbox-deduped) shipped | **MET** | `POST /livekit/webhook`, HMAC through the adapter, deduped on `nexus:{org_id}:dedupe:livekit:{room}:{event}:{participant_sid}`. Real end-to-end delivery from a live LiveKit account is BLOCKED (no account), but the handler, its dedupe, and its fail-closed org resolution are tested against real Postgres. |
| `meeting_rooms` migrated with RLS FORCE | **MET** | `ENABLE` + `FORCE` RLS, `tenant_isolation` policy, `ck_meeting_rooms_status_valid`. Confirmed against the live database, not only the migration source. |
| `realtime-gateway` scaffolded with Redis-adapter fan-out, presence, chat relay | **MET** | Section 1.4. Cross-pod fan-out proven with two separately spawned OS processes. |
| `chat_messages` + `meeting_speakers` migrated in `core-api` | **MET** | Section 1.3. `chat_messages` partitions correctly inherit no application-role grants directly (contrast with the `transcript_chunks` defect in section 3). |
| Staging deploy fires on merge | **NOT MET** | Not attempted in this band. No task added `meeting-service` to a CI deploy matrix or built a staging pipeline; the eight tasks executed were schema, adapter, and route work only. |
| Mobile media + presence wired | **NOT MET** | Deferred by decision. Section 1.6. |
| Cross-pod fan-out test, RLS isolation test, token-mint deny-by-default test all green | **MET** | Cross-pod: two spawned node processes (section 1.4). RLS isolation: proven at the database level for `meeting_rooms`, `chat_messages`, `meeting_speakers` (and, after the fix in section 3, `transcript_chunks`). Deny-by-default: mutation-tested check order (section 1.3). |

### End of P4 (W12) - Real-time and Transcription demo and DoD

| DoD line | Verdict | Evidence or reason |
|---|---|---|
| Compliance gate green before any Deepgram stream | **PARTIAL** | The gate **as an engineering control** is real and unbypassable: `COMPLIANCE_GATE_GREEN = False` blocks `DeepgramProvider.open_stream` before any network I/O, verified by a reviewer actively trying to bypass it. The gate's **substantive conditions** are not met: no Deepgram DPA/SCC is signed, no LiveKit Cloud subprocessor registration exists, and the "zero-retention" flag is unset, because there is no live vendor relationship. This is a legal and compliance gap, not an engineering one, and the code correctly refuses to open a stream regardless. |
| `transcription-service` shipped: provider adapter | **MET** | `TranscriptionProvider` / `DeepgramProvider` / `FakeProvider`. Section 1.4. |
| `transcription-service` shipped: interim-vs-final split | **MET** | The load-bearing zero-persistence test, section 1.4. |
| `transcription-service` shipped: finals-only persistence | **MET** | Same test; only `is_final=true` writes a row. |
| `transcription-service` shipped: at-rest redaction node | **PARTIAL** | The deterministic regex/validator layer (email, phone, IBAN-Luhn, government ID) is real and tested on and off. The plan's NER pass and per-tenant reversible token vault are **not implemented**. |
| `transcription-service` shipped: residency pinning | **PARTIAL** | Endpoint selection (`us`/`eu`) is real and tested against real Postgres. Chunks physically landing in a separate EU cell is BLOCKED - no second regional cell exists. |
| `transcription-service` shipped: `transcript-chunks` producer (outbox + Avro + `chunk:{chunk_id}` dedupe) | **MET** | Section 1.4, after two collision-closing rounds; the final key discriminates on speaker and both time boundaries. |
| `transcription-service` shipped: `meeting-room-events` consumer | **PARTIAL, upgraded from prior session** | Dedupe and session-open/close logic is real and unit-tested (unchanged). The live `confluent-kafka` I/O loop is now wired (`run_consumer`, section 1.4) and started from `main.py`'s `lifespan`, but has not yet been run against the real broker/registry in this session - code review and `py_compile` only. |
| Partition-key DECISION ratified (section 8 #1) | **MET** | [ADR-0017](../adr/0017-transcript-chunks-composite-partition-key.md). Section 1.1. |
| `transcript_chunks` migrated (monthly partition, local index, RLS FORCE) | **MET, with a closed critical defect en route** | The parent table is correctly RLS-FORCEd. Every partition initially had RLS disabled and full DML grants to `mytheus_app`/`mytheus_admin` - a live cross-tenant leak on the most sensitive table in the product, found by the shared exceptionless CI guard and closed in the same task. Full account in section 3. |
| Captions wired through `realtime-gateway` | **MET** | Section 1.4, including the cross-tenant leak found and closed in the same task (section 3). |
| `web-app` captions/transcript/speaker-mapping shipped | **NOT MET** | Deliberately out of scope. Section 1.5. |
| Mobile captions + real meeting clock + mic capture shipped | **NOT MET** | Deferred. Section 1.6. |
| Interims-never-persisted test green | **MET** | The load-bearing test, section 1.4. |
| Redaction test green | **MET, for the implemented scope** | On/off tested for the deterministic layer; the plan's NER/vault scope was never built, so there is no test for what does not exist. |
| Residency test green | **PARTIAL** | Endpoint-selection half green; EU-cell-placement half BLOCKED (no second cell). |
| Reconnect-SLO test green | **NOT MET** | Not implemented. Meaningless to build against `FakeProvider` and unexercisable against a permanently gate-closed `DeepgramProvider`. The mechanism (fault-inject the fake, assert interims are discarded and finals resume) and the untestable > 99.9%-within-2s metric were not separated, per a reviewer's own noted precision gap. |

**Verdict tally: 9 MET, 8 PARTIAL, 6 NOT MET**, of 23 individual lines audited (the two multi-clause DoD paragraphs in the plan text are split one row per clause, matching the style of [02-plan-1-2-foundation-and-tenancy.md](02-plan-1-2-foundation-and-tenancy.md)). Every PARTIAL and NOT MET traces to one of three named causes: missing live vendor credentials (LiveKit, Deepgram, S3), the `mytheus-ui`/`mytheus-mobile` out-of-scope decision, or work explicitly traded off within the band's own priority order (staging deploy, NER/vault, the consumer I/O loop, reconnect mechanics).

---

## 3. Deviations from the plan, and why

| Deviation | Rationale |
|---|---|
| **`mytheus-ui` sections 3.4 and 3.8 not built.** | Explicit product-owner decision, recorded before this band started: `mytheus-ui` is out of scope. Not attempted, not a shortfall found during the work. |
| **`mytheus-mobile` section 7 not built.** | Same decision, plus no LiveKit RN credentials exist to satisfy the section's own acceptance checks regardless. |
| **Every LiveKit, Deepgram, and S3 code path is built to a Protocol with a Fake implementation; the live path is BLOCKED, never faked green.** | No credentials exist for any of the three. The governing decision for this band states plainly: any acceptance check needing a live provider is marked BLOCKED. Applies across `LiveKitTransport`, `DeepgramProvider`, and `nexus-recordings`. |
| **ADR-0018's consent gate is `consent_required AND consented_at IS NULL`, not the ADR's original version-pinned "no matching `consented_at` for the current `consent_version`".** | No `tenant_settings.consent_version` or equivalent current-version pointer exists anywhere in canon, so a version-pinned gate would invent an undefined compliance decision rather than implement a recorded one. The implementer and reviewer agreed the shipped behavior is correct; ADR-0018 itself was narrowed afterward (commit `800cf4e`) to describe what shipped, with an explicit consequence recorded: a tenant that bumps `consent_version` does **not** re-prompt users who consented to the older version. This must be closed - a canon addition landing first - before `consent_required` is turned on for any real tenant. |
| **`transcript_chunks` lives in its own Alembic chain under `mytheus-realtime/services/transcription-service/`, not a competing head in `mytheus-backend`'s chain.** | The honest answer to an open migration-ownership question: the table is owned by `transcription-service`, which lives in a different repo than the Postgres migration history it was, until now, entirely absent from. This is also the contributing cause of the partition-grant defect below: the backend's own partition-privilege convention, and the CI guard that checks it, is scoped to that repo's suite and was not inherited across the repo boundary. |
| **`transcript_chunks` partitions initially inherited full DML grants for `mytheus_app` and `mytheus_admin`, with RLS disabled on every partition - found, then closed, in the same task.** | Root cause: `mytheus-backend` migration `a1c4e7f20b18` (Plan 1-2) runs `ALTER DEFAULT PRIVILEGES` schema-wide, in `public`, not scoped to that repo. Any migration from any repo creating a table or partition in the shared Postgres silently inherits those grants unless it explicitly revokes. PostgreSQL compounds it: RLS on a partitioned parent is enforced only when querying through the parent; a partition queried by name is checked against its own row-security state, which was off. Empirically proven both ways: a live query as `mytheus_app` through the parent returned only the caller's own tenant's row; the identical query against a partition by name returned every tenant's rows, writable and deletable. Closed with a `lock_down_partition()` migration helper that `REVOKE ALL`s from all three application roles immediately after every `CREATE TABLE ... PARTITION OF`, plus a new guard scoped to `transcript_chunks` via `pg_inherits` so future partitions are covered without anyone remembering. The schema-wide root cause is not fixed - see the open gaps. |
| **`chunk_id` is minted deterministically via `uuid5`, not randomly, across three iterations.** | Round 1 (`request_id + ":" + start_ms`) was not a valid UUID and every Postgres insert failed. Round 2 (`uuid5(..., f"deepgram:{request_id}:{start_ms}")`) fixed the format but not the collision surface: `request_id` is constant for a whole Deepgram session, so two distinct finals sharing a millisecond timestamp - ordinary behavior when two speakers start talking at once - collided and the second was silently swallowed by the outbox idempotency gate. Round 3 adds `speaker_id` and `end_ms` to the key, so a collision now requires matching speaker and both time boundaries, which is what a genuine redelivery looks like. Determinism was preserved throughout on purpose: a random key would trade a rare silent drop for guaranteed-broken dedupe on genuine redelivery, which is worse. |
| **`realtime-gateway` Socket.io rooms are keyed by `meeting_id` alone, with `org_id` deliberately dropped from the room name.** | `mytheus-realtime/CLAUDE.md` states rooms are keyed by `meeting_id`; the org-embedding in an earlier draft was a repo-local embellishment on canon, not a canon requirement being weakened. Isolation instead rests on the deny-by-default participant check at `join` time (the only place a socket enters a room) plus, after the fix below, literal per-channel delivery for captions. Verified this is not a regression by tracing every write path into a room. |
| **A real cross-tenant leak in the captions path, found and closed within the same task.** | If org A and org B genuinely have meetings sharing a literal `meeting_id`, both orgs' users legitimately pass their own participant checks and both join Socket.io room `meeting:{id}`. The pre-fix relay delivered a caption to that whole room, so an org A caption reached an org B socket. Reproduced against the pre-fix code with a failing test first, then closed by delivering directly to a per-channel (`org_id` plus `meeting_id`) local-socket set, bypassing room membership for this path entirely - structurally closed rather than contingent on `meeting_id` uniqueness holding. |
| **`presence-events` producer originally wrote raw Clerk claims (the opaque `org_...` string, the `sub` claim) into envelope fields typed as internal UUIDs.** | The Avro schema types `header.org_id` and `payload.user_id` as UUID; canon defines `header.org_id` as the internal tenant root, resolved through `organizations.clerk_org_id`. `realtime-gateway` has no database and cannot resolve this locally. Fixed by resolving through `core-api`'s participant-check response, which already returns the resolved internal ids as part of the deny-by-default `join` flow - no new query, no new endpoint. |
| **`X-Org-Id`, sent by the gateway on the participant-check call, was inert - `core-api` never read it - and was removed rather than documented.** | A header that appears to convey tenancy while being silently discarded is a trap for a future reader who might trust it or copy the pattern into a route that does not ignore it. Since nothing on the sending side read anything back from it either, there was nothing worth keeping under a different name. |
| **`meeting_rooms.livekit_metadata_synced_at` (new column, new migration) replaces an earlier design that called LiveKit synchronously inside the token-mint route's open database transaction.** | The synchronous call created two real defects: a LiveKit blip could crash-loop the `meeting-events` consumer indefinitely (an ordinary exception is not dead-lettered and is redelivered on restart before the offset commits), and the call's default 60 s timeout could hold a pooled connection for up to ~120 s per join, undercutting the same route's own 2 s `statement_timeout`. The fix decouples the LiveKit call into a background task that fires only after the tenant-scoped session has already committed, retried automatically on the next mint or webhook rather than requiring new cross-tenant infrastructure (a full outbox-style sweep was considered and rejected as disproportionate to a fix round). A residual gap is disclosed: a room that never receives a token mint or a webhook after its one sync attempt fails stays permanently unsynced, judged acceptable because that state is only reachable by a room nobody ever actually joins. |
| **Redaction ships as a deterministic regex/validator layer only; the plan's NER pass and per-tenant reversible token vault are not implemented.** | A real NER model is a dependency disproportionate to this band; a per-tenant token vault (encryption, storage, access control for reversal) is a feature of its own scale. Disclosed explicitly, not silently dropped, and the deterministic layer covers every case the plan's own acceptance test exercises. |
| **`transcription-service`'s `meeting-room-events` consumer had real dedupe and session logic but no live `confluent-kafka` I/O loop wired - now closed in a follow-up session.** | The original trade: the team lead's stated priority order was "land the compliance gate, finals-only persistence, and the producer correctly over rushing the reconnect logic and captions bridge," and the I/O wiring was the piece traded to keep that order. Closed by mirroring `mytheus-backend/services/meeting_service/consumer.py`'s established subscribe/poll/commit + DLQ shape rather than inventing a new one, and wiring `run_consumer` into `main.py`'s `lifespan` as a background task - the missing piece that had left `SessionTracker` unreachable from any real process. Verification against the real broker/registry is still outstanding (see section 1.4 and open gap 8). |
| **Reconnect success-rate mechanics (> 99.9% within 2 s) not implemented.** | Meaningless to build against `FakeProvider` (there is nothing to reconnect to) and unexercisable against `DeepgramProvider` while the compliance gate stays closed. |
| **Staging deploy on merge (plan step 3.1.8) not attempted.** | No task in this band's eight-task breakdown covered CI-matrix or deploy-pipeline work; all eight were schema, adapter, and route/service work. Recorded as an open gap rather than a silent skip. |

---

## 4. Open gaps

Ordered by what blocks the most.

| # | Gap | Blocks | Picked up by |
|---|---|---|---|
| 1 | **No LiveKit, Deepgram, or S3-compatible object-store credentials exist anywhere in the program.** | Real end-to-end token verification, `recording.ready` landing an object in `nexus-recordings`, any real Deepgram stream, the EU-cell half of the residency check, the reconnect-SLO metric, the `k6` token-mint p95 smoke, and turning the compliance gate substantively green. | Explicit go plus real vendor accounts. |
| 2 | **`web-app` room UI and captions UI (plan sections 3.4, 3.8) not built.** | The full end-to-end browser demo described in the plan's own end-of-P3 and end-of-P4 demo sections. | Deliberately deferred; needs the maintainer's go the same way Plan 1-2's `mytheus-ui` reconciliation did. |
| 3 | **`mytheus-mobile` section 7 not built.** | The phone leg of the same demo. | Deferred, same decision plus missing LiveKit RN credentials. |
| 4 | **The compliance gate's substantive conditions are unmet: no Deepgram DPA/SCC signed, LiveKit Cloud not yet in the subprocessor register, zero-retention flag unset.** | Turning `COMPLIANCE_GATE_GREEN` on for any real tenant. The engineering control itself is correct and unbypassable. | Legal and compliance, not engineering. |
| 5 | **The platform-level partition-grant leak is not durably fixed.** `mytheus-backend` migration `a1c4e7f20b18`'s schema-wide `ALTER DEFAULT PRIVILEGES` means any future table or partition created by any repo against the shared Postgres re-inherits full DML grants unless its own migration explicitly revokes. `transcript_chunks` is now locked down and guarded; the class of bug that produced it is not closed. | Every future cross-repo migration against this database. | `mytheus-backend` or `mytheus-platform`: either retire the schema-wide grant for narrower per-table ones, or add a `ddl_command_end` event trigger that auto-revokes on every new relation in `public` unless allow-listed. Neither belongs in a single service repo's migration. |
| 6 | **The partition-privilege guard test lives only in `mytheus-backend`'s suite**, scoped to that repo's pytest run. A repo creating a table in the shared Postgres has no way to discover it needs this check before shipping. | The next repo to add a partitioned table to this database, the same way `transcription-service` did. | A cross-repo or platform-level check, not a per-repo one. |
| 7 | **NER redaction pass and the per-tenant reversible token vault not implemented.** | The plan's full redaction scope; the deterministic layer covers its own acceptance test but not the plan's stated design. | A dedicated task once the compliance gate's other conditions are being pursued anyway. |
| 8 | **`transcription-service`'s `meeting-room-events` consumer's live I/O loop is wired but unverified against real infra.** `run_consumer`/`process_decoded_message` exist and are started from `main.py`'s `lifespan`; only `py_compile` and the (infra-independent) poison-message unit tests were exercised this session - no `uv run pytest -q` against the shared dev stack (Postgres/Redis/Redpanda/Registry) was run. | Confidence that the loop actually drains `meeting-room-events` end to end against a real broker. | Next `mytheus-realtime` session with access to the shared dev stack: run the full suite, then a live-broker smoke (produce a `joined`/`left` pair, confirm `SessionTracker` opens/closes and the offset commits). |
| 8a | **Nothing routes the LiveKit audio hook to the replica that opened the meeting's Deepgram stream.** Surfaced by wiring the loop, not caused by it. All `transcription-service` replicas share one `meeting-room-events` consumer group, so a meeting's `joined` is delivered to whichever replica owns that `org_id` partition, and that replica opens the stream into its own in-memory `SessionTracker`. The LiveKit audio hook then connects to `/internal/audio` on whatever replica the ingress picks, which need not be the same one. Harmless today (one replica, and `/internal/audio` is still a scaffold that accepts and closes), and invisible until both are real. | Any multi-replica deployment of `transcription-service`, which is the whole point of anchor `A5` (20k concurrent streams). | Needs a decision, canon-first, between routing the audio hook by `meeting_id` affinity at the ingress and opening the stream lazily from the audio-hook connection instead of from the consumer. Not a code fix in this service alone. |
| 8b | **`/healthz` does not reflect consumer liveness.** If the `meeting-room-events` consumer task dies, the death is now logged loudly, but `/healthz` still answers `ok`, so a Kubernetes liveness probe would not restart the pod: it would serve HTTP happily while opening no transcription sessions at all. | Operational detection of a dead consumer. | A small follow-up in `mytheus-realtime`: it changes `/healthz`'s response contract and `tests/test_healthz.py`'s exact-equality assertion, so it was left disclosed rather than folded into this change. |
| 9 | **`usage_events` producer for `meeting_minute` / `transcription_second` still has no assigned owner (plan section 8, DECISION-REQUIRED #3).** | Metering accuracy in Plan 7-9. Does not block this band's capture path. | Billing lead plus canon owners, before Plan 7-9 needs it. |
| 10 | **Staging deploy on merge for `meeting-service`: partly met, partly blocked, and one clause was wrong about what existed.** Re-audited against the shipped pipeline rather than the plan text. (a) "Add `meeting-service` to the CI matrix" is met by construction: there is no matrix, `mytheus-backend` is one repo with one `pytest` suite that runs seven meeting-related test files, and `meeting-service` has its own container in `deploy/compose.prod.yml`. (b) "Deploy on merge" already existed and exceeds the ask: `ci.yml`'s `deploy` job is gated on all six checks, publishes to GHCR, deploys over an SSH forced command and then verifies `/healthz`. (c) "…to the **staging** deployable" is NOT met and is not a `meeting-service` problem at all: no staging environment exists anywhere in the program, already recorded in [05-deployment-and-cicd.md](05-deployment-and-cicd.md) section 6. (d) The "smoke: `POST /api/meetings/{id}/token` against a seeded participant" is NOT met and is blocked on a real Clerk JWT, a seeded org/meeting/participant, and LiveKit credentials, all of which are recorded placeholders. | Only (c) and (d) remain, and both are blocked on infrastructure and credentials rather than on engineering. | (c) needs a host, DNS and a duplicate secret set, plus the owner's go. (d) unblocks when LiveKit credentials and a seeded staging tenant exist. |
| 10a | **Closed en route: `meeting-service` had no healthcheck, so nothing in the pipeline could tell whether it came up.** Found while auditing gap 10, logged by nobody before. Of the six healthchecks in `compose.prod.yml`, none was on `meeting-service` (it inherits `x-backend-app`, which declares none), and `deploy.sh` waited on `mytheus-web-1` alone. A merge that broke `meeting-service` therefore built green, deployed green, passed CI's public `/healthz` probe (which the gateway answers) and went silently dark. An unauthenticated probe cannot substitute: `services/api_gateway/edge.py` raises `401 AUTH_REQUIRED` before proxying, so a 401 proves only that the gateway is alive. Fixed by giving `meeting-service` the same probe `web` has, and by replacing `deploy.sh`'s best-effort wait with a `wait_healthy` helper that hard-fails and dumps the container's last 50 log lines. The old loop ran 30 times and then fell through to "Done." regardless of outcome, so `set -e` could not catch it; `web` is now gated through the same helper. | Nothing further; this was the substance of what gap 10 was meant to protect. | Done. Verified by `bash -n` and a YAML parse only, NOT by an actual deploy. |
| 10b | **The script that performs every production deploy is unversioned.** `ci.yml` SSHes to a forced command, `/usr/local/sbin/mytheus-deploy.sh`, which exists only on the VPS and is in no repository: it cannot be reviewed, diffed or rolled back, and the `wait_healthy` gate added in 10a therefore applies only to `deploy/deploy.sh`, the documented manual fallback. The `compose.prod.yml` healthcheck does apply to every path, because Dokploy, the server script and the manual script all run the same compose file. | Reviewability of the deploy path, and enforcement of any deploy-time gate on the merge path. | `mytheus-platform`: bring the server-side script into the repo and have the forced command invoke the checked-out copy, or move the gate into `ci.yml` where it is already versioned. |
| 11 | **`tests/test_relay.py::test_relay_real_bus_roundtrip` is a known-flaky live-broker test**, observed flaking by three independent agents across this band, always passing in isolation. | Confidence in the "N passed, 1 failed" framing every session in this band has had to restate. | Needs a real wait-condition instead of a fixed poll window. Carried over from Plan 1-2's gap list, now with more evidence. |
| 12 | **The shared dev Postgres is now ahead of `main`.** `mytheus-backend`'s `main` checkout cannot run `alembic upgrade head` or any alembic operation against the shared dev database until this branch merges (`Can't locate revision '25e7cede5672'`). Services still run fine; they only query tables. | Anyone doing alembic work from `main` against the shared dev stack before this branch merges. | Merge this branch, or point `main`'s alembic operations at a separate database meanwhile. |
| 13 | ~~**`meeting_rooms` is still not represented in `_canon/02-data-model.md`.**~~ **CLOSED.** Added to `_canon/02-data-model.md` §2.2, the §1.1 ownership map, and the Glossary (doc 24), from the shipped schema across all three of its migrations. No ADR was written, and deliberately so: `_canon/01-service-catalog.md` already named `meeting_rooms` and assigned it to `meeting-service`, and [ADR-0003](../adr/0003-meeting-sdk-livekit.md) §3 already recorded the decision, so nothing here introduced a name, a number, or a decision. This was canon disagreeing with itself, not canon being changed. En route it also closed a dangling cross-reference: ADR-0003 §3 cited "[Data Model §2.1]" as the home of `meeting_rooms`, a section that did not contain it and could not, since §2.1 is reserved for tables carried verbatim from the PDF; the citation now points at §2.2. Correcting a broken pointer is not rewriting a decision, so the repo's supersede-do-not-rewrite rule is intact. | - | - |
| 14 | ~~**`mytheus-realtime` has no lint step wired into `make lint` for the gateway (Node/TypeScript) stack.**~~ **STALE, and wrong in both directions.** The wiring already existed: commit `d450ab3` put `cd services/realtime-gateway && npm run lint` in the `lint` target, backed by `eslint . && prettier --check .` over a substantive type-aware `eslint.config.mjs` (ESLint 10 + typescript-eslint + `no-console: error`), not a stub. What nobody had noticed is that **it failed**: `npm run lint` exited 1 with "Code style issues found in 30 files", which is every file in the project. Root cause was not style at all: the repo had **no `.gitattributes`**, so a checkout on Windows under the common global `core.autocrlf=true` rewrote every file to CRLF, while `prettier --check` defaults to `endOfLine: "lf"`. The committed blobs are LF, so Linux was unaffected and the gate was green there and red for any Windows developer. Fixed in `mytheus-realtime` `be9e106` with `* text=auto eol=lf`; `eol=lf` rather than bare `text=auto` is load-bearing, since `text=auto` alone still honours `core.autocrlf` for the working tree. `git add --renormalize .` produced an empty content diff, confirming no file was actually mis-formatted. | - | Done, and verified by running it, not by reading it: after the fix `npm run lint` exits 0 ("All matched files use Prettier code style!") and `npm run typecheck` exits 0. |
| 14a | **`mytheus-realtime` has no CI workflow at all.** There is no `.github/` directory in the repo. `make lint`, `make typecheck` and `make test` exist and work, but nothing runs them automatically on a PR or a push, so every gate in this repo fires only when a human types `make ci`. Contrast `mytheus-backend`, which has a six-job `ci.yml` gating its deploy. This is why gap 8's transcription-service work could be pushed to `main` unverified with nothing to catch it. The session that pushed it told the maintainer that CI would be the first real test of that code; that was wrong, and is corrected here, because there is no CI in this repo to be that test. | Every quality gate in `mytheus-realtime`, on both stacks. Also the reason unverified code can reach `main` in this repo unnoticed. | A `mytheus-realtime` CI session: the workflow needs Postgres, Redis, Redpanda and Schema Registry service containers for the transcription-service suite, and Redis plus a broker for the gateway's cross-pod tests. `mytheus-backend/.github/workflows/ci.yml` is the working template. |
| 15 | **The idempotency-cache collision inherited from Plan 1-2 is worse than previously recorded**: `libs/shared/idempotency.py` keys purely on `nexus:{org_id}:idem:{key}`, with no route, method, or path component, so the real blast radius is cross-endpoint collision (any two different mutating endpoints sharing a reused key), not same-endpoint-different-body as Plan 1-2's record stated. Left alone in this band (shared middleware, all nine services, explicitly out of scope for any single task), but the corrected shape is recorded here so whoever fixes it knows the true blast radius. | Whoever eventually closes Plan 1-2 open gap 3. | Next backend session that touches `libs/shared/idempotency.py`. |
| 16 | ~~**`test_roles_and_policies.py::test_relay_role_drains_outbox_across_tenants` runs an unscoped `DELETE FROM outbox` before seeding its own rows.**~~ **CLOSED, and the recorded description understated it in two ways.** First, there were **two** unscoped statements, not one: the `DELETE FROM outbox` that made the count assertion exact, and a `UPDATE outbox SET published_at = now()` at the end of the test proving the relay can mark rows published. The second is the more damaging: a `DELETE` loses rows loudly, whereas stamping every queued row published leaves it in place and simply never sends it. That is total event loss with no error and no log, which is precisely the defect D4 shape this test exists to guard, reintroduced by the test itself. Second, the evidence recorded as showing no harm is in fact the signature of the harm: "outbox held 3 rows, 0 unpublished" after the run is exactly the state the two statements produce, since the `DELETE` leaves only the test's own three rows and the `UPDATE` publishes them. Fixed by naming the three orgs the test creates in both statements. Scoping the count to those orgs makes the assertion exact without requiring an empty table, so the `DELETE` was removed rather than scoped. What the test proves is unchanged: three rows across three distinct tenant roots, visible to `mytheus_relay` with no `app.current_org` set, which is zero rows under the fail-closed policy alone. | - | Done, and verified in both directions rather than by reading: see section 5.2. |
| 16a | **Windows-only false red in `tests/test_meeting_service_org_id_boundary.py`, found while establishing a green baseline for gap 16.** `_pydantic_model_field_names` keyed its result dict with `str(Path)`, which uses the OS separator, while `test_the_scan_actually_finds_the_response_model` asserts membership of the literal `"services/meeting_service/schemas.py::TokenResponse"`. On Windows the key is built with backslashes, so the assertion failed against a model the scan had in fact found and reported correctly. Green on Linux and macOS, red on Windows, on unchanged code. Fixed by building the key with `as_posix()`. Worth reading next to gap 14, which is the same class of defect found independently in `mytheus-realtime` in the same week (a missing `.gitattributes` letting `core.autocrlf` turn every file CRLF and fail `prettier --check`): both are platform-dependent behaviour in gates that had only ever been exercised on Linux and macOS, and both presented as a broken gate rather than as a portability bug. A third instance should be treated as a pattern rather than a coincidence. | Nothing in the product. It made `mytheus-backend`'s suite unusable as a regression baseline on Windows, which is what surfaced it. | Done. Full suite green on Windows afterwards; see section 5.2. |

---

## 5. Verification

Every command below was run and its output observed during this band, in each repo's Plan 3-4 worktree against the shared dev stack (Postgres, Redis, Redpanda, Schema Registry). Nothing here is inherited from a prior band's claim; where a number is carried forward from earlier in this same band's session log, that is stated explicitly.

| Repo | Command | Result |
|---|---|---|
| `mytheus-backend` | `uv run pytest -q` | **200 passed, 1 failed** (`tests/test_relay.py::test_relay_real_bus_roundtrip`, known flake, reproduced standalone across 3 runs, passes in isolation) |
| `mytheus-backend` | `ruff check .` / `black --check .` / `mypy .` | Clean (103 files black-checked, 72 source files mypy-checked as of the last fix round) |
| `mytheus-backend` | `python scripts/check_authz.py` | Passes on every real route |
| `mytheus-realtime` (`realtime-gateway`) | `npm test` | **42/42 vitest**, across unit, integration, real-broker, and cross-pod-process suites |
| `mytheus-realtime` (`realtime-gateway`) | `npm run typecheck`, `make ci` | Clean / green |
| `mytheus-realtime` (`transcription-service`) | `uv run pytest -q` | **41/41**, all real Postgres/Redis/Redpanda/Registry on load-bearing paths |
| `mytheus-realtime` (`transcription-service`) | `ruff` / `black` / `mypy --strict` | Clean |
| `mytheus-events` | `make validate` | 21 topics OK |
| `mytheus-events` | `make check-compatibility` | 21 subjects BACKWARD against the live Schema Registry |
| `mytheus-events` | Deliberate-break-and-restore of the validator (unqualified `org_id`, then a nullable `EventEnvelopeHeader.org_id`) | Both correctly exit non-zero, then pass again once restored |
| Live DB (direct query) | `SELECT` / `UPDATE` / `DELETE` on a `transcript_chunks` partition by name, as `mytheus_app`, before the fix | **Succeeded across the tenant boundary** - the defect described in section 3, reproduced empirically, cleaned up afterward with zero residue |
| Live DB (direct query) | Same probe, after the fix | Zero rows returned; `mytheus-backend`'s own `test_no_application_role_can_reach_a_partition_directly` (10/10 in that file) passes when run from the Plan 3-4 worktree |
| `mytheus-realtime` | `test_partition_privileges.py`, guard-fires check | A hand-created `transcript_chunks_2099_01` partition without the lockdown helper reproduced the original defect exactly (full DML inherited); the guard failed listing all 8 leaked grants; dropped, both tests pass clean afterward |
| `mytheus-realtime` (`realtime-gateway`) | Cross-pod fan-out test | Two separately spawned OS node processes, distinct PIDs, real Redis adapter |
| `mytheus-realtime` (`realtime-gateway`) | Presence sweep test | A real `kill -9` on one of two spawned processes; the surviving process performs the reap and emits the leave event |
| `mytheus-realtime` (`transcription-service`) | Compliance-gate bypass attempt | A reviewer tried environment variables, direct `DeepgramProvider` construction, and configuring a key; none opened a stream |
| `mytheus-docs` | Regex sweep for the pre-decision partition-key language, before and after the fix rounds | Converged to zero unscoped hits across two rounds |
| `mytheus-docs` | `grep` for em-dashes and en-dashes in files touched this band | Found none in new prose; pre-existing instances elsewhere in the corpus (2,153 counted repo-wide) are flagged as a separate, unfixed repo-hygiene item, not this band's regression |

**NOT run, and not claimed:** any command requiring a LiveKit, Deepgram, or S3 account; the `k6` token-mint smoke; a browser or mobile run against a live meeting.

### 5.1 Follow-up session: `meeting-room-events` I/O loop

Run from an environment with no `uv`, no Docker, and no reachable shared dev stack - only `python -m py_compile` could be exercised, not `pytest`.

| Repo | Command | Result |
|---|---|---|
| `mytheus-realtime` (`transcription-service`) | `python -m py_compile src/transcription_service/meeting_room_consumer.py src/transcription_service/main.py tests/test_meeting_room_consumer.py` | OK, no syntax errors |
| `mytheus-realtime` (`transcription-service`) | Line-length audit of the three touched files against `line-length = 100` | No line over 100 |
| `mytheus-realtime` (`transcription-service`) | Script-assisted audit for the `black --check` failure mode that a line-length check misses: a split call whose joined form would fit in 100 characters, which `black` collapses | Found one (a `raise PoisonMessage(...)` at exactly 100 joined) and fixed it; re-scan clean. A stand-in for `black`, not a substitute - `black --check` itself was not run |
| `mytheus-realtime` (`transcription-service`) | `uv run pytest -q` (full suite: the 3 new poison-message tests plus the pre-existing 41) | **NOT run** - no `uv` and no reachable Postgres/Redis in this environment |
| `mytheus-realtime` (`transcription-service`) | `ruff` / `black --check` / `mypy --strict` | **NOT run** - same reason. Imports, import ordering, and annotations were checked by reading, which is not the same as running the gates |

This is disclosed as a gap, not claimed green. The code mirrors an already-proven pattern (`mytheus-backend/services/meeting_service/consumer.py`, itself real-broker tested in section 1.3/5), but "mirrors a tested pattern" is not the same as "tested" - and the self-review in section 1.4 found that the divergence from that pattern is exactly where the defects were, because this consumer shares a process with the FastAPI app and the backend's does not. Open gap 8 stays open until a session with the shared dev stack runs the real suite and the three quality gates.

### 5.2 Follow-up session: gaps 16 and 16a, `mytheus-backend` test hygiene

Run against a real local dev stack (Docker Compose Postgres on 5433, Redis on 6380, Redpanda plus Schema Registry on 9092/8081), on Windows. The database was found stranded at migration `82d44163c933` with none of the `mytheus_*` roles present, so the stack was brought to head first; that is recorded below because it means the run exercised the full migration chain from Plan 0 forward, not an accumulated local state.

| Repo | Command | Result |
|---|---|---|
| `mytheus-backend` | `alembic upgrade head` from `82d44163c933` | All 9 migrations applied clean, ending at `56f72ac73fa9` |
| `mytheus-backend` | `python -m scripts.bootstrap_roles` | All three roles set; the script's own re-assertion that none holds `rolsuper` or `rolbypassrls` passed |
| `mytheus-backend` | `uv run pytest tests/test_roles_and_policies.py -q` | **10 passed.** This is the file Plan 5-6 ran with 10 deselected under its do-not-run instruction, so those 10 assertions have now run for the first time since that instruction was issued |
| `mytheus-backend` | Guard-fires check for gap 16: seed a bystander `outbox` row belonging to an org the test does not create, then run the file with the fix applied | The row survives, still unpublished |
| `mytheus-backend` | Same check with the pre-fix code restored (`git stash`) | **The row is gone.** The pre-fix test reported `10 passed` in the same run that deleted it, which is why this survived three bands: the test was green the entire time it was destroying data |
| `mytheus-backend` | `uv run pytest -q` (full suite) | **290 passed, 0 failed.** One pre-existing failure, `test_the_scan_actually_finds_the_response_model`, was gap 16a and was fixed rather than tolerated, because a red suite cannot serve as a regression baseline |
| `mytheus-backend` | `ruff check .` / `black --check` / `mypy .` | Clean; mypy over 88 source files |

**NOT run, and not claimed:** anything in `mytheus-realtime`, including the gap 8 verification that stack would now support. The dev stack used here is a local Compose project on one machine, not a shared instance, so nothing in this session bears on gap 12.

Note for whoever closes gap 11: `test_relay_real_bus_roundtrip` passed in every run above, against a live broker. That is not evidence the flake is gone. It is a timing-dependent test with a fixed poll window, and a machine fast enough to win the race every time is exactly the condition under which it has always passed in isolation.

---

## 6. What to do next

1. **Get the maintainer's explicit go for `mytheus-ui` and `mytheus-mobile`**, the same way Plan 1-2's section 3.10 reconciliation required one, before building sections 3.4, 3.8, or section 7.
2. **Close the platform-level partition-grant leak durably** (open gap 5): narrow `mytheus-backend`'s schema-wide `ALTER DEFAULT PRIVILEGES`, or add a `ddl_command_end` event trigger. This is the single highest-leverage fix outstanding, because it protects every future table in the shared database, not just this band's.
3. **Verify `transcription-service`'s `meeting-room-events` consumer I/O loop against the real shared dev stack** (open gap 8) - the subscribe/poll/commit wiring and DLQ routing are now written and code-reviewed, but need `uv run pytest -q` and a live-broker smoke run from an environment with Docker/`uv`/the shared dev stack reachable.
4. **Pursue the compliance gate's substantive conditions** (Deepgram DPA/SCC, LiveKit Cloud subprocessor registration, zero-retention flag) once real vendor relationships exist - the engineering gate is ready to be turned on the moment they land.
5. **Promote `test_relay_real_bus_roundtrip` off the deferred-minor list** (open gap 11) - it has now flaked across three separate bands' worth of sessions.

---

## Final whole-branch review

Run after all eight tasks passed their own task-level reviews. Its purpose was the one thing a
task-level review structurally cannot do: look at all four repos at once and find what falls between
tasks. It earned that purpose.

The first attempt, one reviewer over the whole branch, produced no output at all and was treated as
failed rather than re-prompted. It was replaced by two reviewers, each taking half the branch with a
bounded output budget and explicit permission to report "nothing above Minor" rather than manufacture
findings.

Verdicts:

- `mytheus-backend` and `mytheus-events`: MERGE. No Critical, no Important. It cleared the
  least-reviewed code deliberately, confirming the detached `asyncio.Task` in `f4306bf` keeps a
  strong reference with an `add_done_callback` discard, never touches a session before the triggering
  transaction commits, and cannot crash the poll loop; that the token-mint route's hand-managed
  session closes on every exit path including exceptions; and that the deadlock regression test
  exists rather than accepting the claim. It also cleared the two-Alembic-histories risk: the
  `transcription-service` history renames its version table to
  `alembic_version_transcription_service` so it cannot collide with the backend's on the same
  physical database, and `transcript_chunks` holds no foreign key into backend-owned tables.
- `mytheus-docs`: clean and independently mergeable. Every prior "not ratified" or "PROPOSED"
  mention was propagated to cite the ADRs, and ADR-0001, ADR-0007 and ADR-0009 carry pure
  forward-pointer annotations rather than rewrites.
- `mytheus-realtime`: MERGE WITH FIXES, on one Critical.

### The Critical, and why every earlier review missed it

Presence and chat still delivered by Socket.io room membership after `roomName()` had stopped
carrying `org_id`. Three presence broadcasts and the chat relay were affected. Two organizations
whose meetings shared a literal `meeting_id` would place sockets from both in one room, so a member
of one would receive the other's presence member list, carrying internal user UUIDs, and its live
chat content.

This was not an oversight by any single task. Presence and chat were built while the room name still
carried `org_id`, and were correct then. The room key lost `org_id` later, while fixing an unrelated
identity bug, and two reviewers examined that change and cleared it, one tracing every write path and
reasoning that emitting is not joining. That reasoning holds for authorization and fails for delivery
targeting. A later task then found and fixed the leak on its own path without anyone asking what else
the room-key change had made unsafe.

The signal that exposed it was a test asymmetry: captions had a dedicated test for two organizations
sharing one literal `meeting_id`; presence and chat had nothing equivalent.

Fixed in `6679ab4` by restoring `org_id` to `roomName()` itself as a two-argument function, so the
compiler enforces it at all five call sites rather than four separate call sites each needing to
remember. That is safe now in a way it was not when the argument was first dropped, because every
caller threads the same `presenceOrgId`, resolved once at join and reused everywhere including by the
reconciliation sweep. The reasoning is recorded in `libs/keys.ts` so it is not reversed without
someone reading why. A collision test covering presence and chat in both directions was added and
verified to fail against the pre-fix code, not merely to pass against the fix.

### Residual items accepted rather than fixed

- The `stop()` drain was changed from a single snapshot to a loop, removing reliance on an unstated
  Socket.io ordering guarantee. The accompanying test does not discriminate the loop from the
  snapshot: both pass, because `io.close()` registers cleanups synchronously before its own callback
  in every scenario that could be constructed. The loop was kept regardless and the limit recorded
  rather than overclaimed.
- Interim captions are not redacted. Confirmed as correct and now documented at the point the code
  skips it. Note this is not an at-rest-only rule: a final's live caption carries the already-redacted
  text, so only interims are ever unredacted. It rests on the assumption that every tenant treats at
  rest as the boundary. A tenant whose posture puts the live plane in scope would need a second
  `tenant_settings` flag, separate from `pii_redaction_enabled`. None exists.

### A theme worth carrying forward

Three separate test doubles in this band were found to be unfaithful to the services they doubled:
one read a header the real service never read, one minted a fresh organization id per call so two
users intended to share an organization silently did not, and one encoded a contract that the real
producer had already stopped honouring. Each was invisible until something outside the double
depended on the real behaviour. Test doubles in this program need fidelity review in their own right,
not only the code under test.

---

## 7. Related

- [00-status-overview.md](00-status-overview.md) for program-wide state.
- [02-plan-1-2-foundation-and-tenancy.md](02-plan-1-2-foundation-and-tenancy.md) for what this band inherited, including the entry preconditions this band's own recon confirmed GREEN.
- [adr/0017-transcript-chunks-composite-partition-key.md](../adr/0017-transcript-chunks-composite-partition-key.md) and [adr/0018-consent-entity-behind-flag.md](../adr/0018-consent-entity-behind-flag.md) for the two decisions this band ratified.
- [plan/plan-5-6-org-context-and-ai-infra.md](../plan/plan-5-6-org-context-and-ai-infra.md) for the next band, whose entry preconditions this band's recon already checked: identity live (GREEN), this band's tier live (GREEN once merged), `libs/shared` building blocks (GREEN), infra reachable (PARTIAL - Qdrant, an S3-compatible store, and a GPU runner are still missing), and the partition-key decision (already ratified here, see ADR-0017).
