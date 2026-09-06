# READ-ONLY AUDIT — mytheus-intelligence: embedding + ingestion

Root: `/Users/sushan/Projects/Mytheus`. All paths below are relative to `mytheus-intelligence/`
unless prefixed with another repo name. Source read directly; no `.md` status/progress claim was
taken at face value. The stale duplicate tree at
`mytheus-intelligence/mytheus-intelligence/.worktrees/plan-5-6/` was excluded from every grep —
it is a nested copy of the same repo and inflates all LOC/file counts.

---

## IMPLEMENTED (file:line)

### embedding-service — all real, no stubs

- `services/embedding_service/model.py:41` — `embed_batch()` runs a real
  `sentence-transformers` load of `BAAI/bge-small-en-v1.5`, 384-dim, `normalize_embeddings=True`,
  `show_progress_bar=False`, model cached via `@lru_cache(maxsize=1)` at `:30`. This is the real
  published BGE-small-en weight set, not a random-vector stand-in. Returns
  `(vectors, wall_clock_seconds)` so callers can measure the SLO.
- `services/embedding_service/model.py:36` — device selection: `"cuda"` only when
  `EMBEDDING_GPU=1` (`gpu_requested()`, `:26`), otherwise `"cpu"` fp32. `use_fp16` / the CUDA
  image bake-in from plan 3.7 are absent by admission (GPU-only concerns).
- `services/embedding_service/consumer.py:49-121` — `EmbeddingConsumer(BaseConsumer)`.
  - `:50` group `embedding-service.embed-on-ingest`
  - `:51` topics `["transcript-chunks", "document-chunks"]`
  - `:61-89` two branches building `logical`, `stable_key`, `idem`, and the Qdrant payload
  - `:91` 600/60 split
  - `:104` real embed
  - `:105-109` `PointStruct` construction + `upsert()` into the per-tenant collection
  - `:111-121` `embedding-events` ack through the transactional outbox
  - `:38-46` deterministic point ids: `uuid5(_POINT_NAMESPACE, f"{stable_key}:{index}")`. This is
    genuine idempotency — a redelivered source event recomputes the same ids and overwrites in
    place rather than duplicating. The namespace UUID is fixed and documented as never-change.
  - `:92-102` empty-text path still emits an ack with `point_count: 0` so lag does not grow on a
    legitimately empty message.
- `services/embedding_service/serve.py:70-116` — two REAL Ray Serve deployments:
  `IngestEmbedder` (`num_replicas=1`, `max_ongoing_requests=64`, `@serve.batch(max_batch_size=32,
  batch_wait_timeout_s=0.05)` at `:77`) and `QueryEmbedder` (`max_ongoing_requests=16`,
  `@serve.batch(max_batch_size=3, batch_wait_timeout_s=0.01)` at `:98`). Split deliberately so a
  query burst cannot starve ingest. `:37-46` keeps the module import-safe when Ray is not
  installed, so pytest collection never needs a Ray cluster. `_actor_options()` at `:66-68`
  requests `num_gpus=1` only when `EMBEDDING_GPU=1` AND Ray actually reports a GPU.
- `services/embedding_service/query_embed.py:59-91` — real `POST /embed`. 1-3 query variants
  enforced at `:68`, Redis cache key `nexus:{org_id}:qembed:{sha256(query)[:32]}` (`:51-56`),
  TTL 600s (`:38`), per-query cache hit/miss reported back in `cached[]`. `org_id` comes from
  `Depends(verified_org_id)` (`:66`) — the previous client-supplied `X-Org-Id` header is gone.
- `services/embedding_service/main.py:21-24` — FastAPI process: `create_app("embedding-service")`
  (`/healthz`) with the query-embed app mounted at `/internal`. The consumer and relay run as
  separate processes, deliberately off this API tier.
- `services/embedding_service/worker.py:23-42` — real consumer entrypoint. Real Postgres
  (`mytheus_ai_app`), real Redis, real `AvroCodec` against the Schema Registry, real Qdrant
  `write_client()`, then `run_forever(brokers=...)`.
- `services/embedding_service/relay_main.py:50-71` — real outbox relay entrypoint draining
  `intelligence.outbox` to Redpanda as `mytheus_ai_relay`. One relay per repo, covering
  ingestion-service's `document-chunks` rows too.
- `services/embedding_service/splitter.py` — thin re-export of `libs/shared/splitter.py`.

### The shared consumer machinery — real, and load-bearing

- `libs/shared/consumer.py:127-201` — `BaseConsumer.process_raw`:
  - `:136-149` deserialize failure → DLQ (`failure_class="DESERIALIZE"`), never raises
  - `:152-173` missing/unparseable `org_id` or `idempotency_key` → DLQ
    (`failure_class="BUSINESS_INVALID"`). Never guessed, never defaulted (risk checkpoint R6)
  - `:186-188` Redis dedupe short-circuit
  - `:190-201` durable inbox check, then `apply()` and `record_consumed()` **in one transaction**
    scoped by `set_rls(session, org_id)` — the effect and the fact commit together
  - transient exceptions from `apply` propagate uncaught so the offset is not committed and the
    broker redelivers
- `libs/shared/consumer.py:203-258` — DLQ envelope as plain JSON (not Avro, deliberately: the
  message is here because it already failed to conform). DLQ topic is scoped per consumer group:
  `f"{topic}.{self.group}.dlq"` (`:258`) so a topic-name-driven redrive tool cannot replay one
  consumer's poison into another's `apply`.
- `libs/shared/consumer.py:260-319` — `run_forever`: subscribe, poll, `process_raw`, THEN
  `consumer.commit(message=msg, asynchronous=False)`. `enable.auto.commit: False`,
  `auto.offset.reset: earliest`. ONE persistent event loop created at `:292` — see the sustained
  load section for why that line matters.
- `libs/shared/relay.py:65-` — `relay_once`: `FOR UPDATE SKIP LOCKED` (`:46`) so replicas do not
  double-publish, Avro-serialize against `{topic}-value`, produce keyed by `org_id`, mark
  published. `MAX_ATTEMPTS = 5` then dead-letter.
- `libs/shared/events.py:62-64` — `EventEnvelope.to_value()` composes `{header, **payload}`; the
  outbox stores the composed value and the relay serializes it against the registered subject.
- `libs/shared/splitter.py:20-31` — `RecursiveCharacterTextSplitter`, `CHUNK_SIZE=600`,
  `CHUNK_OVERLAP=60`. Never returns `[]` for non-empty input.

### ingestion-service — all real

- `services/ingestion_service/parser.py` — real `unstructured.io`. Supported and exercised:
  TXT, MD, HTML, EML, DOCX, PPTX via unstructured's own partitioners. PDF is deliberately NOT
  routed through `partition_pdf` (that import chain pulls torch/torchvision/onnxruntime/opencv,
  ~2GB, verified live) — instead `pdfminer.six` `extract_text` + unstructured's own
  `partition_text` classifier. Stated limits: no OCR (image-only PDF yields zero elements), no
  PDF-internal table detection. Legacy `.doc`/`.ppt`/`.xls` raise `UnsupportedDocumentType`.
  Element types are folded into the frozen Avro enum `{Title, NarrativeText, Table, ListItem}`;
  everything else becomes `NarrativeText` — never dropped, never invented.
- `services/ingestion_service/pipeline.py:92-150` — `ingest_document()`:
  - `:104` parse
  - `:108-119` one `intelligence.document_elements` row per RAW element, `element_id =
    f"el_{position:04d}"`, `raw` = unstructured's own `to_dict()`
  - `:121` PII redaction gate
  - `:122-146` per split piece: one `intelligence.document_chunks` row (`element_id =
    f"el_{position:04d}_{j}"`) plus one `emit(...)` outbox enqueue with idempotency key
    `docchunk:{document_id}:{element_id}`
  - `:64-76` both inserts are `ON CONFLICT (org_id, document_id, element_id) DO NOTHING`
  - `:86-89` `Table`/`ListItem` keep their typed element boundary UNSPLIT; `Title`/`NarrativeText`
    get the 600/60 split
  - Runs entirely inside the caller's already-open, already-RLS-scoped transaction. Does not touch
    object storage and does not own the job status lifecycle — both callers do.
- `services/ingestion_service/pipeline.py:156-205` — `create_job` / `complete_job` / `fail_job`
  against `intelligence.ingestion_jobs`.
- `services/ingestion_service/routes.py:78-168` — real `POST /ingest`:
  - `:80` `org_id` from `Depends(verified_org_id)`
  - `:84-85` exactly one of `file` or `url`
  - `:93-96` real `httpx` fetch for the URL form
  - `:98-99` 25 MB cap (`MAX_DOCUMENT_BYTES`, `:46`)
  - `:101-107` extension allowlist via `safe_extension`
  - `:109-117` server-generated `document_id`, object key `document_key(org_id, document_id, ext)`,
    real S3/MinIO `put_object`
  - `:119-155` job row, then parse+chunk+emit+complete in a second transaction
  - `:156-160` on any failure the job row is marked `failed` in its own transaction, then 422
- `services/ingestion_service/routes.py:170-205` — `GET /ingest/jobs/{job_id}`, RLS-scoped, 404
  for both "not found" and "not yours" with no distinction leaked.
- `services/ingestion_service/consumer.py:73-154` — `IntegrationEventConsumer`, group
  `ingestion-service.attachment-to-ingest`, topic `integration-events`.
  - `:87-90` R8 guardrail: no-op for anything that is not
    `(provider="google_calendar", event_type="attachment_to_ingest")`
  - `:64-70` `_resolve_key` — the object key from another service's event body is used ONLY if it
    starts with `f"{verified_org_id}/"`. Rejection is fail-closed: never fetch, log, return.
    Blast radius closed is a cross-tenant READ from the shared `nexus-attachments` bucket.
  - `:121` `document_id` is ALWAYS server-generated, never taken from `payload_json`
- `services/ingestion_service/pii.py` — real regex redactor for email / phone / SSN-shaped /
  credit-card-shaped text, honestly labelled a structural baseline rather than NER. Gate reads
  Redis key `nexus:{org_id}:tenant_settings:pii_redaction_enabled`, defaults to `False`.
- `services/ingestion_service/main.py:22-34` and `worker.py:27-46` — real FastAPI app and real
  consumer entrypoint, same wiring shape as embedding-service.
- `libs/shared/object_storage.py:58-147` — `BUCKET_DOCUMENTS = "nexus-documents"`,
  `BUCKET_ATTACHMENTS = "nexus-attachments"`, both `{org_id}/`-prefixed key builders, plus
  `safe_extension`, `ensure_bucket`, `put_object`, `get_object`.

### Qdrant tenancy chokepoint

- `libs/shared/qdrant.py:108-112` — `physical_collection(logical, org_id) ->
  f"{logical}__{org_id.hex}"`. Documented as the ONLY place allowed to compute a physical name.
- `libs/shared/qdrant.py:75-88` — `write_client()`: requires `QDRANT_API_KEY`, raises
  `QdrantAuthNotConfigured` if absent. No unauthenticated fallback.
- `libs/shared/qdrant.py:91-105` — `read_client()`: requires `QDRANT_READ_ONLY_API_KEY`, never
  falls back to the write key.
- `libs/shared/qdrant.py:61-72` — `_distinct_or_raise`: refuses to build any client when the two
  keys are equal, because that configuration passes every isolation test while granting readers
  write access.
- `libs/shared/qdrant.py:115-130` — `ensure_collection`: idempotent create with COSINE distance,
  INT8 scalar quantization `always_ram=True`, `memmap_threshold=50_000`, `on_disk_payload=True`.
- `libs/shared/qdrant.py:133-135` — `_org_filter`: mandatory `org_id` payload backstop.
- `libs/shared/qdrant.py:138-152` — `upsert()`: resolve → ensure → upsert, returns the physical
  name for the ack payload.
- `libs/shared/qdrant.py:155-174` — `search_with_tenant_filter()`: physical collection AND payload
  filter together.
- `libs/shared/qdrant.py:40` — `EMBEDDING_DIM = 384`; `:43-45` frozen logical names
  `transcript_chunks`, `document_chunks`, `decisions`.

### Migrations

- `migrations/versions/9c2b7e4a1d03_0001_intelligence_schema_outbox_inbox_roles.py`
  - `:95-96` `pgcrypto`, `CREATE SCHEMA IF NOT EXISTS intelligence`
  - `:98-128` `intelligence.outbox` + `intelligence.inbox` with partial index on unpublished rows
  - `:60-75` roles `mytheus_ai_app` and `mytheus_ai_relay`, both
    `LOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE`
  - `:82-90` `ENABLE` + `FORCE ROW LEVEL SECURITY` and a `tenant_isolation` policy per table
  - `:141-155` least-privilege grants; the relay role gets `SELECT, UPDATE` on outbox only, plus a
    single permissive `outbox_relay_drain` policy scoped to that role — never a BYPASSRLS role
- `migrations/versions/b3f7a1c9e402_0002_ingestion_document_tables.py:72-133` —
  `intelligence.document_chunks`, `intelligence.document_elements`,
  `intelligence.ingestion_jobs`; all FORCE RLS + `tenant_isolation`; unique
  `(org_id, document_id, element_id)` underpinning the `ON CONFLICT DO NOTHING` writes.
- `migrations/versions/d415f023220e_0003_search_service_transcript_index.py:81-107` —
  `intelligence.transcript_chunk_search` with a `GENERATED ALWAYS AS (to_tsvector(...)) STORED`
  column and a GIN index. `:5-10` explicitly REFUSES the plan's literal
  `ALTER TABLE public.transcript_chunks` because that table belongs to `mytheus-realtime`.

---

## PARTIAL / STUB / MISSING

| Item | State | Evidence |
| --- | --- | --- |
| `rag-service` | **STUB** — 14 lines, `/healthz` only | `services/rag_service/main.py` |
| `agent-runtime-service` | **STUB** — same shape | `services/agent_runtime_service/main.py` |
| `knowledge-graph-service` | **STUB** — same shape | `services/knowledge_graph_service/main.py` |
| Internal gRPC mesh | **ABSENT** | `proto/rag.proto` holds only `RagQueryPlaceholder`; every internal contract is "REST now, gRPC later" (`query_embed.py:5-10`, `search_service/routes.py:1-5`) |
| GPU / Ray packaging | **BLOCKED by admission** | `model.py:1-12`, `serve.py:9-20`. CPU fp32 only |
| batch-32 < 100 ms SLO | **UNMET, and unasserted** | `test_embedding_sustained_load.py:20-26` records ~3536 ms per batch-32 on CPU |
| KEDA autoscaling | **CONFIG ONLY, nothing reads it** | `services/embedding_service/keda-scaledobject.yaml:1-6` says so verbatim |
| Containerization | **ABSENT** | `find mytheus-intelligence -iname 'Dockerfile*'` (excluding worktree) returns nothing |
| `CLERK_JWT_INTERNAL_ORG_CLAIM` wiring | **NOT PROVISIONED** | `libs/shared/auth.py:44-50` — the claim is expected to carry the already-resolved internal `org_id` UUID; that Clerk-template / identity-tenant-service mapping is explicitly "NOT made by this commit". Every REST surface in this repo is therefore un-callable by a real token today |
| `integration-service` (producer of `integration-events`) | **ABSENT** | `mytheus-backend/services/integration_service/` is 11 lines total (`main.py` only, `__init__.py` empty); `grep -rn "integration-events" mytheus-backend --include=*.py` → **zero hits** |
| `SemanticChunker` for narrative docs | **Deliberately not implemented** | `pipeline.py:28-32` — would force this service to load its own embedding model |
| `chat_messages` FTS arm | **BLOCKED, reported** | `search_service/consumer.py:34-45` — no `chat-messages` topic exists in the 21-subject catalog and `core_api` writes those rows with no outbox path |
| `.env.example` Redis port | **INCONSISTENT** | `.env.example:9` says `redis://localhost:6379/0`; every code default says `:6380` (`worker.py:36`, `main.py:23`, `ingestion_service/main.py:31`). `6380` is correct per the backend stack |
| Composite partition key | **Latent** | `transcript-chunks` / `agent-stream` use `{org_id}:{meeting_id}` (`mytheus-events/codegen/_schemas.py:43`); this repo's relay keys by `org_id` alone. Harmless today — this repo never produces either topic — but it would break silently if it ever did |

---

## THE INGEST→EMBED→STORE PIPELINE — hop by hop

### Arm A — transcript path (the live one)

| # | Hop | Status | Evidence |
| --- | --- | --- | --- |
| A1 | Deepgram ASR → `transcription-service` | **WIRED** | `mytheus-realtime/services/transcription-service/src/transcription_service/providers/deepgram.py` is a real provider module |
| A2 | Finals only → one `public.transcript_chunks` row + one `public.outbox` row, topic `transcript-chunks`, subject `transcript-chunks-value`, composite key `{org_id}:{meeting_id}` | **WIRED** | `.../persistence.py:5, 31-32, 47`; interims never persisted or produced (`:120-150`) |
| A3 | `mytheus-backend` relay drains `public.outbox` → Redpanda | **WIRED** | relay service exists and is in prod compose (`mytheus-platform/deploy/compose.prod.yml:210`) |
| A4 | `EmbeddingConsumer` consumes `transcript-chunks`; envelope `org_id` re-asserted, Redis+inbox dedupe | **WIRED in code, NOT RUNNING** | `consumer.py:51,61`; `libs/shared/consumer.py:152-201`. No image, no compose entry, no process anywhere in deploy |
| A5 | 600/60 split → real BGE-small-en embed | **WIRED** | `consumer.py:91,104` → `model.py:41` |
| A6 | Upsert into `transcript_chunks__{org_id.hex}`, payload `{org_id, meeting_id, chunk_id, speaker, start_ms, is_final}` | **WIRED** | `consumer.py:67-74,105-109` → `qdrant.py:138-152` |
| A7 | Ack `{collection, point_count, meeting_id}` → `intelligence.outbox` (same txn as inbox insert) | **WIRED** | `consumer.py:111-121`; `libs/shared/consumer.py:190-201` |
| A8 | `relay_main.py` drains `intelligence.outbox` → `embedding-events` topic | **WIRED** | `relay_main.py:50-71`. Subject `embedding-events-value` IS registered and the payload matches the Avro record field-for-field (`mytheus-events/schemas/embedding-events/v1.avsc`: `collection` string, `point_count` int, `meeting_id` nullable string). No serialization mismatch |
| A9 | Anyone consuming `embedding-events` | **ABSENT** | No consumer group anywhere subscribes to it. Ack topic exists for lag tracking only |
| A10 | Anyone reading the vectors back out of Qdrant | **ABSENT** | See the next section |

### Arm B — document path, direct upload

| # | Hop | Status | Evidence |
| --- | --- | --- | --- |
| B1 | `POST /ingest` (file or URL), Clerk-verified `org_id` | **WIRED in code, GATED** | `routes.py:78-96`. Gated on the unprovisioned `CLERK_JWT_INTERNAL_ORG_CLAIM` |
| B2 | 25 MB cap, extension allowlist, server-generated `document_id` | **WIRED** | `routes.py:98-109` |
| B3 | Blob → MinIO/R2 `nexus-documents/{org_id}/{document_id}...` | **WIRED** | `routes.py:110-117`; `object_storage.py:79` |
| B4 | `ingestion_jobs` row `status='processing'` | **WIRED** | `routes.py:119-128` |
| B5 | Real unstructured.io parse → typed elements | **WIRED** | `pipeline.py:104` → `parser.py` |
| B6 | PII gate (Redis `nexus:{org_id}:tenant_settings:pii_redaction_enabled`) | **WIRED — and the producer now exists** | `pii.py:121`; the key is actually written by `mytheus-backend/services/identity_tenant_service/routes/tenant_settings.py:61`, asserted in `mytheus-backend/tests/test_tenant_settings_api.py:151`. The gap the module docstring describes has since been closed cross-repo |
| B7 | `document_elements` (raw) + `document_chunks` (split), both `ON CONFLICT DO NOTHING` | **WIRED** | `pipeline.py:108-134` |
| B8 | One outbox row per chunk, key `docchunk:{document_id}:{element_id}` | **WIRED** | `pipeline.py:135-145` |
| B9 | `complete_job` with `chunk_count` | **WIRED** | `routes.py:153-155` |
| B10 | `relay_main.py` → `document-chunks` topic | **WIRED** | Payload `{document_id, element_id, element_type, text, position}` matches `mytheus-events/schemas/document-chunks/v1.avsc` exactly |
| B11 | `EmbeddingConsumer` document branch → `document_chunks__{org_id.hex}` | **WIRED** | `consumer.py:76-89,105-109` |
| B12 | Anyone reading those vectors | **ABSENT** | See next section |

Note on double-splitting: `ingestion-service` splits 600/60 at B7, and `EmbeddingConsumer`
applies the same 600/60 split AGAIN at A5/B11. This is intentional and harmless (a no-op on
already-short text) and documented at `libs/shared/splitter.py:5-13`.

### Arm C — document path, calendar attachment

| # | Hop | Status | Evidence |
| --- | --- | --- | --- |
| C1 | `integration-service` downloads the attachment and writes it to `nexus-attachments` | **ABSENT** | `mytheus-backend/services/integration_service/` = 11 lines, `main.py` only |
| C2 | `integration-service` emits `integration-events` with `payload_json = {meeting_id, object_key, filename}` | **ABSENT** | `grep -rn "integration-events" mytheus-backend --include=*.py` → zero hits. `consumer.py:26-30` admits it: "`integration-service` does not exist yet in this program (P12, a later plan band)"; the payload contract is this task's own reading of the docs |
| C3 | `IntegrationEventConsumer` filters to `(google_calendar, attachment_to_ingest)` | **WIRED, unreachable** | `consumer.py:87-90` |
| C4 | `_resolve_key` cross-tenant prefix guard | **WIRED** | `consumer.py:64-70,102-118` |
| C5 | MinIO GET → `ingest_document` → same as B5-B11 | **WIRED, unreachable** | `consumer.py:120-154` |

**Summary of the trace: Arm A is WIRED end-to-end in code up to the point where nothing reads
the result. Arm B is WIRED end-to-end in code, gated only on the Clerk claim. Arm C is BROKEN at
its head — the consumer is complete and correct but subscribes to a topic that no producer in
the program writes.** And nothing in any arm is deployed.

---

## WHO CONSUMES THE EMBEDDINGS

**Nobody. Confirmed by grep, not inferred.**

`grep -rn "read_client\|search_with_tenant_filter\|from libs.shared.qdrant" --include="*.py" .`
(worktree excluded) returns exactly these non-definition hits:

- `tests/conftest.py:153` (`write_client`), `tests/conftest.py:162` (`read_client`)
- `tests/test_qdrant_tenancy.py:12,75` (`search_with_tenant_filter`)
- `tests/test_qdrant_api_key_isolation.py:35,40,49,64,71,80,87,101,134-157`
- `tests/test_embedding_consumer.py:28`, `tests/test_embedding_sustained_load.py:41`
  (`physical_collection` only, for verification)
- `services/embedding_service/worker.py:19` (`write_client` — the writer)
- `services/embedding_service/consumer.py:34` (`upsert` — the writer)

**Zero non-test readers.** The intended reader is `rag-service`'s vector arm, and
`services/rag_service/main.py` is a 14-line healthz stub. `LOGICAL_DECISIONS` (`qdrant.py:45`) is
declared and never used at all — no producer, no consumer.

The `embedding-events` ack topic has no consumer group anywhere either.

The FTS half is the same story: `search-service`'s `GET /search` has no caller in
`mytheus-backend`, `mytheus-realtime`, or `mytheus-ui` — the only cross-repo hit for
"search-service" outside comments is a marketing copy string at
`mytheus-ui/src/lib/marketing/features.ts:241`.

**Verdict: embeddings are written and never read outside the test suite.**

---

## TENANCY IN THE VECTOR STORE

ADR-0006 resolves to **collection-per-tenant AND a payload filter — both layers, and both are
really implemented.**

1. **Collection boundary** — `physical_collection()` at `qdrant.py:108-112` produces
   `{logical}__{org_id.hex}`. It is the single sanctioned resolution point; nothing else in the
   repo string-formats `__{org_id}` inline (verified by grep).
2. **Payload backstop** — `_org_filter()` at `qdrant.py:133-135` is applied on every read through
   `search_with_tenant_filter()` (`:155-174`). Every written point carries `org_id` in its payload
   (`consumer.py:68`, `consumer.py:84`).
3. **Provenance of `org_id`** — from the verified event envelope header via `parse_org_id`
   (`libs/shared/consumer.py:152`), never from the event body. Missing or unparseable → DLQ, never
   defaulted (`:155-173`). For the REST surfaces, from `verified_org_id` (Clerk RS256 + JWKS).
4. **Sole-writer as infrastructure, not convention** — Qdrant's two-key model. `write_client()` /
   `read_client()` each hard-require their own key and refuse to fall back; `_distinct_or_raise`
   refuses to build anything when the two keys match. `docker-compose.yml` sets
   `QDRANT__SERVICE__API_KEY` and `QDRANT__SERVICE__READ_ONLY_API_KEY` separately.

### Is `tests/test_qdrant_api_key_isolation.py` testing something real?

**Yes** — with one important caveat.

It asserts the *property* against a live `qdrant/qdrant:v1.12.4`:
- `:49-68` a read-only client's `upsert` raises 403 (collection pre-provisioned with the write
  client so the failure under test is the upsert itself, not a missing collection)
- `:71-85` a read-only client's `create_collection` raises 403 — closing the path where a rogue
  writer provisions empty collections
- `:87-101` a read-only client CAN still search
- `:134-157` constructor-level fail-closed: no key → raise; identical keys → raise

The docstring's permission matrix (none → 401, read-only → 403 on write, write → 200) is recorded
as verified directly against the pinned image, and the tests deliberately assert the property
rather than "some env var is set" — an explicit lesson from the partition-grant finding in
`mytheus-backend` where six assertions were all scoped to roles the program created and none saw
the role the image shipped.

**Caveat:** `tests/conftest.py:84-89` — `_require_or_skip` silently `pytest.skip`s when Qdrant is
unreachable, unless `MYTHEUS_REQUIRE_QDRANT=1` is set (in which case it raises rather than
skipping). In a bare CI with no Qdrant container these tests prove nothing and say nothing. The
escape hatch is deliberate and documented, and `docker-compose.yml`'s own header calls out that
the previous hand-started keyless `docker run` Qdrant was exactly the reason these tests used to
skip silently. But the guarantee is only as good as the environment flag.

### One residual fragility

`org_id` is rendered two different ways in two adjacent places: `.hex` (undashed) for the
collection name (`qdrant.py:112`) and `str(org_id)` (dashed) for the payload value
(`consumer.py:68,84`) and the filter (`qdrant.py:135`). Write and read agree today, so the
backstop genuinely matches. But an edit to either side that "normalizes" the format would make
the payload filter match nothing while the collection boundary silently carries the whole load —
and every isolation test would still pass, because collection-per-tenant alone is sufficient for
them.

---

## CONFIG/ENV REQUIRED

From `.env.example` and the code defaults:

**Vector store**
- `QDRANT_URL` (default `http://localhost:6333`, `qdrant.py:57-58`)
- `QDRANT_API_KEY` — embedding-service ONLY. Required, no fallback.
- `QDRANT_READ_ONLY_API_KEY` — every other consumer. Required, no fallback.
- The two MUST differ or client construction raises.
- Dev dummies: `dev-qdrant-write` / `dev-qdrant-read`.

**Postgres — the SAME physical database as `mytheus-backend`**
- `DATABASE_URL` → `postgresql+asyncpg://mytheus_ai_app:mytheus_ai_app@localhost:5433/mytheus`
- `RELAY_DATABASE_URL` → `mytheus_ai_relay@localhost:5433/mytheus`
- `BOOTSTRAP_DATABASE_URL` → `supabase_admin@localhost:5433/mytheus` (migrations only; must be
  able to `CREATE SCHEMA`, which backend's `postgres` role cannot on this Supabase image)

**Event backbone**
- `REDPANDA_BROKERS=localhost:9092`, `SCHEMA_REGISTRY_URL=http://localhost:8081`

**Redis**
- `REDIS_URL` — `.env.example:9` says `redis://localhost:6379/0`, but every code default is
  `redis://localhost:6380` (`worker.py:36`, `embedding_service/main.py:23`,
  `ingestion_service/main.py:31`). `6380` matches the backend test stack; `.env.example` is wrong.

**Object storage (ingestion-service only)**
- `MINIO_ENDPOINT_URL=http://localhost:9002`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`

**Auth (all REST surfaces)**
- `CLERK_ISSUER`, `CLERK_JWKS_URL`, `CLERK_JWT_INTERNAL_ORG_CLAIM` — the last is expected to carry
  the already-resolved internal `org_id` UUID and is **not provisioned anywhere**.

**Model providers — this is the important answer**
- **Embeddings are LOCAL, not an API.** `sentence-transformers` downloads
  `BAAI/bge-small-en-v1.5` from HuggingFace on first use (`model.py:22,37`). **No API key
  required, no per-call cost.** But: first run needs network access and ~130 MB of weights, and
  since there is no Dockerfile there is no baked-in weight cache — a cold container would download
  on every start.
- `EMBEDDING_GPU=0` / `RERANKER_GPU=0`.
- `OPENAI_API_KEY` and `ANTHROPIC_API_KEY` are declared **blank** in `.env.example` and are
  **used by nothing in this repo today**. No LLM provider call exists anywhere in the embedding or
  ingestion paths — they are placeholders for the unbuilt `agent-runtime-service`.
- `IDENTITY_GRPC_ADDR=localhost:50051` — for a gRPC mesh that does not exist.
- `OTEL_EXPORTER_OTLP_ENDPOINT`, `SENTRY_DSN` — both blank.

### Batching, retry, backpressure

- **Batching, two levels.** Ray Serve `@serve.batch` (32 / 50 ms for ingest, 3 / 10 ms for query)
  at `serve.py:77,98`. And in the consumer, "batch = however many pieces one message's split
  produced" — for a typical single utterance that is **1**, so the consumer path is effectively
  unbatched in practice. Note the consumer calls `embed_batch` directly (`consumer.py:104`), NOT
  through the Ray `IngestEmbedder` deployment — so the 32-item micro-batching does not apply to the
  real ingest path at all today.
- **Retry / backpressure** is the Kafka contract, not application logic: a transient `apply`
  failure propagates, the offset is not committed, the broker redelivers
  (`libs/shared/consumer.py:27-30, 312-317`). Poison messages are classified to DLQ instead. The
  relay retries up to `MAX_ATTEMPTS = 5` then dead-letters (`relay.py:40`).
- **No** explicit retry loop, **no** rate limiter, **no** circuit breaker on Qdrant or on the
  model. The only intended backpressure signal is KEDA consumer-lag scaling, and nothing reads
  that config.

---

## POSTGRES: same database, no schema conflict

Same physical Postgres as `mytheus-backend` (`localhost:5433/mytheus`), different schema. Every
table this repo owns lives in `intelligence.*`:

`intelligence.outbox`, `intelligence.inbox`, `intelligence.document_chunks`,
`intelligence.document_elements`, `intelligence.ingestion_jobs`,
`intelligence.transcript_chunk_search`.

No name collides destructively with `public.*`. In particular `intelligence.document_chunks` and
`intelligence.transcript_chunk_search` are distinct from `public.transcript_chunks` (owned by
`mytheus-realtime`'s `transcription-service`, a RANGE-partitioned table on which
`mytheus_ai_app`/`mytheus_ai_relay` hold **zero** grants — verified live per
`search_service/consumer.py:9-13`). Migration `d415f023220e` explicitly refuses to `ALTER` it.

This does contradict this repo's own CLAUDE.md invariant "the AI tier NEVER writes Postgres
directly" — but that same CLAUDE.md carves out `ingestion-service` and `search-service` as the
two permitted relational writers of their own tables. Stated deviation, not a silent one.

Two separate relays exist and must both run: `mytheus-backend`'s drains `public.outbox`, this
repo's `relay_main.py` drains `intelligence.outbox`. A relay only ever sees the table it is
pointed at.

---

## SUSTAINED-LOAD TEST — what it actually proves

`tests/test_embedding_sustained_load.py`:

- Runs the **real** `run_forever` (not `process_raw` in a loop) against **real** Redpanda with a
  real consumer group and a real backlog of `BACKLOG = 12` schema-valid `TranscriptChunk`
  messages, in a background thread. `DRAIN_TIMEOUT_S = 420.0`. `pytestmark = pytest.mark.slow`.
- Reads lag from the **broker itself** — committed offsets from the group's own metadata,
  high-watermark from the broker (`_group_lag`, `:74-90`) — deliberately the same quantity KEDA's
  kafka trigger would scale on, rather than a number the test computes for itself.

**Asserts:** the backlog fully drains, every message produces its vector in the tenant's Qdrant
collection, offsets advance to the high-watermark, lag returns to zero.

**Deliberately does NOT assert** lag < 5 s. The docstring records why rather than relaxing the
threshold until it passes: batch-32 embedding measures ~3536 ms on CPU against a sub-100 ms GPU
target, so a sustained backlog cannot clear inside 5 s on this hardware and no consumer tuning
changes that. The measured drain rate is printed so the number is on the record.

**Its real value** is documented at `libs/shared/consumer.py:266-279`: it caught a genuine
production-class bug. The obvious `asyncio.run(self.process_raw(...))` per message created and
CLOSED a fresh event loop each time; the async clients (redis.asyncio, Qdrant, SQLAlchemy async
engine) bind their pools to the first loop, so message #2 raised `RuntimeError: Event loop is
closed`, escaped the poll loop, and killed the consumer **having committed exactly one offset**.
That was the real behaviour until 2026-08-29 and was invisible to every other test, because they
all call `process_raw` directly inside one loop and none ever ran a second message through
`run_forever`. It surfaced as a consumer group whose lag went 24 → 23 and then stopped.

So: it proves drain correctness and offset advance, it explicitly does not prove the SLO, and it
is the only test in the repo that exercises the real Kafka loop. That is a genuinely valuable
test, honestly scoped.

---

## DEPLOYED?

**No. Nothing in `mytheus-intelligence` is deployed, and nothing is even deployable.**

- `grep -n "intelligence\|qdrant\|embedding\|ingestion\|transcription\|realtime"
  mytheus-platform/deploy/compose.prod.yml` → **zero hits.**
- `compose.prod.yml` contains only: `postgres`, `redis`, `redpanda`, `temporal`, `migrate`, `web`,
  `relay`, `workflow`, `api-gateway`, `bff`, `identity-tenant-service`, `meeting-service`,
  `integration-service`, `billing-metering-service`, `admin-ops-service`,
  `notification-service`, `ui`. All `mytheus-backend` plus the UI.
- **No Qdrant** in prod compose → even if the worker were deployed it would have no vector store.
- **No MinIO / R2 wiring** in prod compose → ingestion has nowhere to put blobs.
- **No `mytheus-realtime`** either → the `transcript-chunks` producer is not deployed, so the
  source of Arm A is not running in prod regardless.
- **No Dockerfiles** anywhere in `mytheus-intelligence` → there is no image to add to a compose
  file. `Makefile` has only `install / lint / typecheck / test / ci`; there is no build target.
- `docker-compose.yml` in this repo is **dev-only** and covers Qdrant + MinIO only, deliberately
  excluding Postgres/Redis/Redpanda/Registry because those belong to the backend stack.

---

## VERDICT

The embedding and ingestion services are genuinely and carefully built — real BGE-small-en
weights, real unstructured.io parsing with honestly documented format limits, real two-key Qdrant
isolation enforced by the server rather than by convention, real transactional outbox/inbox with
group-scoped DLQ, and a sustained-load test that caught a real event-loop bug no other test could
see. The pipeline is wired end-to-end in code for the transcript arm and the upload arm; the
calendar-attachment arm is complete but subscribes to a topic whose producer
(`mytheus-backend/services/integration_service/`, 11 lines) does not exist. The decisive gap is
the far end and the deployment: every vector written to Qdrant is read by nothing outside
`tests/`, because `rag-service`, `agent-runtime-service`, and `knowledge-graph-service` are all
14-line healthz stubs and the gRPC mesh is a placeholder `.proto` — and with no Dockerfile and no
entry in `compose.prod.yml`, none of this tier has ever run anywhere but a developer laptop.
