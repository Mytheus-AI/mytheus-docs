# Verification: `mytheus-intelligence` + `mytheus-realtime`

Read-only pass. Nothing modified. All paths absolute-rooted at `/Users/sushan/Projects/Mytheus`.
`.worktrees/` copies and the nested `mytheus-intelligence/mytheus-intelligence/` duplicate were
excluded from every grep; line cites are against the primary checkout.

---

## Part 1 — Verdicts on the 13 claims

### 1. No Dockerfile in either repo; neither appears in `compose.prod.yml` — **CONFIRMED**

`find` for `Dockerfile*` across the whole tree returns exactly three, none in these repos:
`mytheus-ui/Dockerfile`, `mytheus-backend/Dockerfile`, and the backend's worktree copy.

`mytheus-platform/deploy/compose.prod.yml` service keys (lines 78–364): `postgres`, `redis`,
`redpanda`, `temporal`, `migrate`, `web`, `relay`, `workflow`, `api-gateway`, `bff`,
`identity-tenant-service`, `meeting-service`, `integration-service`,
`billing-metering-service`, `admin-ops-service`, `notification-service`, `ui`. Every one of
those runs the single `mytheus-backend` image (`compose.prod.yml:61`) or the UI image
(`:365`). A grep of `compose.prod.yml`, `caddy/`, and `deploy.sh` for
`intelligence|realtime|rag|transcription|reranker|embedding|qdrant` returns **one** hit, and it
is the word "destroy" inside a comment (`compose.prod.yml:38`). There is no Qdrant, no MinIO,
no `realtime-gateway`, no `transcription-service` in production.

**Additional finding not in the brief:** neither repo has CI at all.
`mytheus-backend/.github/workflows/ci.yml` and `mytheus-events/.github/workflows/ci.yml` exist;
`mytheus-intelligence/.github` and `mytheus-realtime/.github` **do not exist**. So there is no
image build, no published artifact, and no test gate on either repo. `compose.prod.yml:59-61`
explicitly relies on CI publishing the backend image; there is no equivalent to point at.

### 2. `rag_service`, `agent_runtime_service`, `knowledge_graph_service` are 13-line stubs; `proto/rag.proto` is a placeholder — **CONFIRMED**

- `mytheus-intelligence/services/rag_service/main.py` — 13 lines. Body is
  `app = create_app("rag-service")` (`:13`). Docstring `:3` says "Scaffold only: boots and
  answers `/healthz`."
- `mytheus-intelligence/services/agent_runtime_service/main.py` — 13 lines, same shape (`:13`).
- `mytheus-intelligence/services/knowledge_graph_service/main.py` — 13 lines, same shape (`:13`).
- These three directories contain **only** `__init__.py` and `main.py`. No `agents/`, no
  retrieval code, no Cypher.
- `mytheus-intelligence/proto/rag.proto` — 15 lines. `:11` `TODO(RAG band): define RagService
  (Query), plus Embed/Rerank/GraphTraverse/FtsSearch messages.` The only message is
  `RagQueryPlaceholder { string query = 1; }` (`:13-15`).

Also: the Apache AGE graph does not exist anywhere. A tree-wide grep for
`apache.?age|nexus_knowledge|cypher|ag_catalog` across `mytheus-intelligence`,
`mytheus-backend/migrations` and `mytheus-platform` hits **only the two docstring lines in the
knowledge_graph_service stub**. No extension, no migration, no graph.

### 3. `search_service` is FTS-only with zero callers; `reranker_service` loads a real cross-encoder with zero callers — **CONFIRMED**

`search_service` is real and good code, and purely lexical: `services/search_service/routes.py:99-110`
is a `websearch_to_tsquery('english', :q)` / `ts_rank` query against
`intelligence.transcript_chunk_search`, with `set_rls` before the SELECT (`:113`) **and** an
explicit `WHERE org_id = :org_id` (`:103`) as an independent second layer. No vector arm, no
fusion.

`reranker_service` is a real BGE-reranker-base cross-encoder (`services/reranker_service/rerank.py:59`,
`model.py`, `RerankResponse.model = "BAAI/bge-reranker-base"` at `rerank.py:56`).

Callers: a tree-wide grep for `search-service|search_service|/rerank|reranker-service|reranker_service`
across all 8 repos returns hits **only** in (a) that repo's own tests, (b) its own migrations
and docstrings, (c) `mytheus-ui/src/lib/marketing/features.ts:241` — a marketing copy string.
Zero production callers. `search_service/routes.py:16-28` says so itself: the response is shaped
`passage` to feed `/rerank` "with zero adapter", and the re-binding of a reranked passage back
to provenance is explicitly left as "a `rag-service` concern for a later task."

### 4. No LLM call anywhere in the 8 repos; `litellm` and `langgraph` installed and never imported — **CONFIRMED**

A tree-wide grep (excluding `.venv`, `node_modules`, worktrees, lockfiles) for
`anthropic|openai|OpenAI|Anthropic|chat\.completions|messages\.create|gpt-4|claude-3|claude-sonnet`
across `*.py` and `*.ts` returns **zero hits**.

`mytheus-intelligence/pyproject.toml:9` `"langgraph>=0.2"`, `:10` `"litellm>=1.40"`. Neither is
imported by any line of source in any repo. The only live LangChain usage is
`libs/shared/splitter.py:18` — `RecursiveCharacterTextSplitter`, a character splitter, not
inference.

`mytheus-intelligence/.env.example:46-48` reserves `OPENAI_API_KEY=` / `ANTHROPIC_API_KEY=`, both
empty, under the comment "LLM routing (LiteLLM)". Nothing reads either variable.

### 5. Audio path hops 1–4 absent — **CONFIRMED, all four**

- **Hop 1, mic capture:** grep for `getUserMedia|MediaRecorder|AudioContext|createLocalAudioTrack|LocalAudioTrack|enableMicrophone|setMicrophoneEnabled`
  across `mytheus-ui/src` and `mytheus-mobile` returns **zero hits**. `mytheus-ui` has
  `src/lib/realtime/useLiveKitRoom.ts` and `src/components/meeting/LiveVideoGrid.tsx`, but no
  audio track is ever published.
- **Hop 2, `subscribe_room_audio`:** `mytheus-backend/services/meeting_service/transport.py:139-141`
  — `raise NotImplementedError("subscribe_room_audio lands with the transcription tap")`. No
  caller outside the fake at `:231`.
- **Hop 3, `/internal/audio`:** `mytheus-realtime/services/transcription-service/src/transcription_service/main.py:119-124`
  — `await websocket.accept()` then `await websocket.close()`. No frame is ever read. Its only
  caller anywhere is `tests/test_healthz.py:17`.
- **Hop 4, `start_egress`:** `mytheus-backend/services/meeting_service/transport.py:135-137` —
  `raise NotImplementedError("start_egress lands with the nexus-recordings bucket (Task 4)")`.
  No caller outside the fake at `:223`.

**Additional:** `transcription-service` loads `LIVEKIT_URL` / `LIVEKIT_API_KEY` /
`LIVEKIT_API_SECRET` (`config.py:40-42`) and **never uses any of them**. The `/internal/audio`
websocket performs no handshake validation of any kind — no signature, no token, no origin
check. As written it is an unauthenticated websocket; harmless today only because it closes
immediately, but it is the exact spot where auth has to land before it carries audio.

### 6. `FakeProvider` persists canned finals through the same path as a real provider with no provenance marker — **CONFIRMED**

`providers/factory.py:20-23`: no `DEEPGRAM_API_KEY` → `FakeProvider()`. Since the compliance
gate is a source constant set to `False` (`compliance.py:39`), `FakeProvider` is the **only**
provider that can ever produce an utterance in any environment today.

`providers/fake.py:26-69` `default_script()` returns two canned interims and one canned final —
`"hello there, how are you"`, `confidence=0.97`. `sessions.py:118-126` pumps every utterance from
either provider into the same `process_utterance`. `persistence.py:53-80` builds a
`TranscriptChunk` and writes it through the outbox.

`Utterance` (`providers/base.py:33-40`) has no provider field. `TranscriptChunk`
(`mytheus-events/schemas/transcript-chunks/v1.avsc`) has fields `header, meeting_id, chunk_id,
speaker_id, start_ms, end_ms, text, is_final, confidence, redacted` — **no provider, no
synthetic flag**. So a fake final is byte-indistinguishable from a real one in
`transcript_chunks`, on the `transcript-chunks` topic, in the FTS index built from it, and in any
vector embedded from it. There is no post-hoc query that can separate them.

### 7. `transcription-service/db.py` connects as the `postgres` superuser, bypassing its own correct RLS calls — **CONFIRMED**

`db.py:13-22` states it outright: "Local dev/test connects as the `postgres` superuser... The
`SET LOCAL app.current_org` calls below are still always issued, so the code is correct for a
properly scoped role in production **even though the local superuser connection bypasses RLS**."

`config.py:32-34` default: `postgresql+asyncpg://postgres:postgres@localhost:5432/mytheus`.
`.env.example:9`: `postgresql+asyncpg://postgres:postgres@localhost:5433/mytheus`. Both are the
superuser. There is no `mytheus_rt_app` analogue to `mytheus_app`/`mytheus_ai_app`. Consequence:
`tests/test_rls_isolation.py` and `test_tenancy_and_residency.py` exercise `set_current_org`
against a role for which RLS is a no-op — the isolation they assert is not actually being
enforced by the database in that run.

Note the port discrepancy too: code default `5432`, `.env.example` `5433`. The rest of the
program is on `5433`.

### 8. `/healthz` stays green if the consumer dies — **CONFIRMED, and self-disclosed**

`main.py:15-19`: "KNOWN GAP... If the consumer task dies, its death is logged loudly
(`_log_consumer_exit` below) but `/healthz` still answers ok, so a liveness probe would not
restart the pod."

`main.py:114-116` returns `{"status": "ok", ...}` unconditionally — it never inspects
`consumer_task`. `_log_consumer_exit` (`:48-62`) logs and returns. Because
`meeting_room_consumer.run_consumer` is what opens and closes every transcription session
(`main.py:78-86`), its death means the pod transcribes nothing while reporting healthy.

### 9. `embedding-events` produced by one thing, consumed by nothing — **CONFIRMED**

Produced at `mytheus-intelligence/services/embedding_service/consumer.py:98`
(`topic="embedding-events"`, through `intelligence.outbox`), relayed to Redpanda by
`libs/shared/relay.py`. The topic is created (`mytheus-events/scripts/create_topics.sh:71`), has
a registered schema (`mytheus-events/schemas/embedding-events/v1.avsc`) and a generated client
(`mytheus-events/clients/python/mytheus_events/embedding_events.py`).

No `subscribe`/consumer for it exists in any repo. Every other reference is a test asserting it
was produced (`tests/test_embedding_consumer.py:122-131`, `tests/test_outbox_relay.py`). It is a
write-only ack.

### 10. Intelligence writes `{topic}.{group}.dlq`, which `create_topics.sh` never creates; payload is JSON while `DlqEnvelope.avsc` says Avro — **CONFIRMED, both halves**

`libs/shared/consumer.py:258` — `return f"{topic}.{self.group}.dlq"`, used at `:247`.

`mytheus-events/scripts/create_topics.sh:50` — `mk_dlq() { create "$1.dlq" ... }`, invoked as
`mk_dlq transcript-chunks`, `mk_dlq document-chunks`, etc. (`:53-75`). It creates
`transcript-chunks.dlq`. The intelligence consumers write to
`transcript-chunks.embedding-service.embed-on-ingest.dlq` and
`transcript-chunks.search-service.transcript-index.dlq`. Neither is in the script; the script's
own closing line says "42 expected in total" — the 21 topics and their 21 `.dlq` pairs, with no
room for group-scoped ones. On a cluster with auto-create disabled, every dead-letter is dropped
at produce time.

`consumer.py:215-218` writes `json.dumps(envelope).encode()` and argues the case: "Deliberately
not Avro-encoded... no `.dlq` subject is registered in the Schema Registry either, confirming
this is the intended shape." `mytheus-events/contracts/DlqEnvelope.avsc` is an Avro record whose
doc says "Verbatim from 07-event-architecture.md 7. One paired `<topic>.dlq` (14-day retention)
exists per topic." Both the encoding and the topic-naming rule are contradicted. Note also that
`contracts/` holds **only** `DlqEnvelope.avsc` — the 21 real schemas live in `schemas/*/v1.avsc`,
so `DlqEnvelope` is the one contract with no generated client and no producer conforming to it.

The `consumer.py:220-232` reasoning for group-scoping is genuinely sound (a topic-name-driven
redrive tool would replay one group's poison into another's `apply`). The defect is that the
decision was made in one repo and never propagated to `mytheus-events`.

### 11. `CLERK_JWT_INTERNAL_ORG_CLAIM` unprovisioned makes every intelligence REST route un-callable — **CONFIRMED, and worse than stated**

`libs/shared/auth.py:88` reads `CLERK_JWT_INTERNAL_ORG_CLAIM`, defaulting to the claim name
`internal_org_id`. `:38-46` discloses that the mapping which would populate that claim "is a
Clerk-dashboard / `identity-tenant-service` change outside this repo's code and is NOT made by
this commit."

`org_id_from` (`:131-140`) raises 401 "Token carries no organization" when the claim is absent.
Every REST route in the repo depends on `verified_org_id`: `search_service/routes.py:91`,
`reranker_service/rerank.py`, `embedding_service/query_embed.py`, `ingestion_service/routes.py`.
So today every one of them 401s for every real token.

**Worse:** `mytheus-intelligence/.env.example` (52 lines, read in full) contains **no `CLERK_*`
variable at all** — not `CLERK_JWT_INTERNAL_ORG_CLAIM`, not `CLERK_JWKS_URL`, not
`CLERK_ISSUER`. `auth.py:80` does `os.environ["CLERK_JWKS_URL"]` and `:84` does
`os.environ["CLERK_ISSUER"]` — bare subscript, no default. A process started from `.env.example`
raises `KeyError` inside the dependency, which FastAPI surfaces as a **500**, not the 401 the
module carefully engineers. So the failure mode is not "un-callable"; it is "un-callable and
reported as a server fault."

`mytheus-realtime`'s gateway has the same shape but is a shade better: `.env.example` declares
`CLERK_JWKS_URL=` (empty, with a note) and `server.ts:619` reads it — but `CLERK_ISSUER` is
absent there too, and `libs/auth.ts:35` silently derives the issuer from the JWKS URL's origin,
which is an acceptable default only as long as they always match.

### 12. `.env.example` Redis 6379 contradicts every code default of 6380 — **CONFIRMED for `mytheus-intelligence`; INVERTED in `mytheus-realtime`**

`mytheus-intelligence/.env.example:9` — `REDIS_URL=redis://localhost:6379/0`. Every code default
is 6380: `services/search_service/worker.py:39`, `services/ingestion_service/worker.py:40`,
`services/ingestion_service/main.py:31`, `services/embedding_service/worker.py:36`,
`services/embedding_service/main.py:23`, `tests/conftest.py:40`. `docker-compose.yml:9` documents
6380 as the shared backend-stack port. So the `.env.example` is the single wrong file — copying
it points every service at a Redis that isn't there.

`mytheus-realtime` has the **same bug reversed**, which the brief did not capture:
- `transcription-service/.env.example:2` = `6380` (right) but `config.py:35` defaults to `6379` (wrong).
- `realtime-gateway/.env.example:4` = `6380` (right) but `server.ts:593` defaults to `6379` (wrong).

So across the two repos the example file and the code default disagree in **all four** places,
just in opposite directions. Anything that runs without an explicit `REDIS_URL` — a container
without an env file, a test harness, a bare `uvicorn` — silently reaches for 6379.

Same class of drift on Postgres: `transcription-service/config.py:33` defaults to `5432`,
`.env.example:9` says `5433`.

### 13. The Qdrant isolation test is skipped unless `MYTHEUS_REQUIRE_QDRANT=1` — **CONFIRMED**

`tests/conftest.py:152`, `:161`, `:171` — `_require_or_skip(qdrant_reachable(), "Qdrant",
"MYTHEUS_REQUIRE_QDRANT")`. All three Qdrant fixtures are gated. Every test in
`tests/test_qdrant_api_key_isolation.py` and `tests/test_qdrant_tenancy.py` depends on one of
them, so absent that variable the entire "a read-only credential cannot write" control —
described in that file's own docstring as closing "status-overview gap 19" — silently does not
run. With no CI in this repo (finding 1), nothing anywhere sets that variable.

---

## Part 2 — What ELSE is pending

Beyond the 13, from reading the source:

**A. Nothing in either repo is deployable, and nothing tests it.** No Dockerfile, no CI, no
compose entry, no image registry path. This is not one gap per repo; it is the gap that makes
every other item unverifiable in an environment anyone else can reach.

**B. `mytheus-realtime` has no `Makefile`-driven or scripted way to run both services together.**
The gateway needs `npm run build && npm start` (`package.json`), transcription needs
`uvicorn transcription_service.main:app --port 8091` (`main.py:21`). No compose file exists in
the repo at all.

**C. Cross-repo contract ownership is unowned.** Three independent drifts found — the DLQ
naming/encoding (item 10), the Redis port (item 12), the missing `provider` field on
`TranscriptChunk` (item 6). Each was a locally-reasonable decision in one repo that nothing
propagated to `mytheus-events`. There is no test in any repo that asserts a producer's topic
names are a subset of what `create_topics.sh` creates.

**D. `presence-events` is degraded by an unresolved id contract.**
`realtime-gateway/src/presenceEventsProducer.ts:206` skips the produce entirely when
`org_id`/`user_id` are not yet resolved UUIDs. So `presence-events` is silently partial.
Relatedly, `server.ts:59-63`: the `chat` socket event still carries the raw Clerk `sub` while
`presence` carries the internal `users.id` UUID — two different identity shapes on the same
socket, flagged in the source as a follow-up.

**E. `transcription-service` has no least-privilege DB role, and creating one is cross-repo.**
`db.py:16-21` explains why: `mytheus-backend`'s roles are created with no password by design, and
granting a new cross-repo role SELECT on tables this service does not own is an ops/bootstrap
concern. Real, but it means item 7 cannot be fixed inside `mytheus-realtime` alone.

**F. `embedding-service` GPU path is unexercised.** `EMBEDDING_GPU` / `RERANKER_GPU`
(`.env.example:40-41`) default to 0 with the note "BLOCKED in this dev environment... no CUDA
device here". The Ray Serve `gpu` extra and `services/embedding_service/keda-scaledobject.yaml`
have never run.

**G. Region-local placement is unimplemented and says so.** `residency.py:9-16`: endpoint
*selection* is real and tested; physical multi-region Postgres/Redpanda cells do not exist in any
repo. `data_residency.db_cell` is `'pool-us-1'` for every tenant.

**H. `ap` region silently falls back to the US Deepgram endpoint.** `residency.py:21-29` —
documented, not silently assumed, but it means an `ap` tenant's audio crosses to a US endpoint.
Needs an ADR before any `ap` tenant is onboarded.

**I. `search_service`'s FTS index covers transcripts only.** `CLAUDE.md:38` specifies
"`tsvector` over `transcript_chunks` **+ chat_messages**". `routes.py:102` queries only
`intelligence.transcript_chunk_search`. The chat arm is absent.

**J. No `usage-events` emission from the AI tier.** `CLAUDE.md:33-34` makes `usage-events` an
output of both `agent-runtime-service` and `rag-service`. Neither exists, so the AI tier's
metering — and therefore any per-tenant budget enforcement — has no source.

**K. `agent-requests` / `agent-responses` are created topics with no participant.** Same
`create_topics.sh:60-61`, same situation as `embedding-events`: schema, client, topic, no code.

---

## Part 3 — SIZING THE KEYSTONE: `rag-service`

### What exists to join

| Arm | Status | Reachable how |
|---|---|---|
| FTS | **real** — `search_service/routes.py`, RLS + explicit org filter | `GET /search`, response already shaped `passage` to feed `/rerank` with no adapter (`routes.py:16-28`) |
| Vector | **half** — `embedding_service` writes Qdrant, `libs/shared/qdrant.py:92` gives a read client; `query_embed.py` exists | needs a query-embed call + a Qdrant search; the search side is untested (item 13) |
| Graph | **absent** — stub only, no AGE extension anywhere | not reachable; MVP must be 2-arm |
| Rerank | **real** — BGE-reranker-base cross-encoder | `POST /rerank` |
| LLM | **absent everywhere in 8 repos** | nothing |

### Honest MVP scope

A two-arm (vector + FTS), non-streaming `POST /api/rag/query` that expands nothing, fuses with
RRF k=60, reranks, binds citations, drops uncited passages, and synthesizes one answer.

| Work | Days |
|---|---|
| FastAPI app, route, request/response models, inbound auth, token pass-through to callees | 2–3 |
| Two retrieval clients (search-service HTTP; query-embed + Qdrant read), `org_id` threaded on every hop | 2–3 |
| RRF fusion + **provenance re-binding through rerank** — `/rerank` returns bare scored text with no id (`rerank.py:49-52`), so a fused hit must be matched back by passage text; `search_service/routes.py:26-28` names this unsolved and hands it to rag-service | 2 |
| LLM synthesis via litellm — first LLM call in the program: model choice, prompt template + loader, timeouts, retries, token accounting | 3–4 |
| Citation binding + the "every passage binds to a citation or it is dropped" invariant + tests | 2–3 |
| Redis answer cache under `nexus:{org_id}:` | 1 |
| Adversarial tenant-isolation tests at the bar this repo already holds (see `tests/test_search_routes.py` — two orgs sharing vocabulary, one dominating the LIMIT) | 3–4 |
| Dockerfile + compose entry + CI (shared with the repo-wide item) | 1–2 |

**≈ 16–22 engineer-days — roughly 3.5 to 4.5 weeks for one engineer**, for something explicitly
2-arm, non-streaming, with no budget enforcement and no gRPC.

Excluded from that number and stated as such: the graph arm (no AGE), streaming, tiered routing,
per-tenant budget enforcement, and the gRPC mesh. Keeping REST is correct and already sanctioned
three times over in this repo's own docstrings (`rerank.py:17-24`, `search_service/routes.py:1-5`);
`proto/rag.proto` should stay a placeholder.

### Hard dependencies — what must land before rag-service can be built or run

1. **Item 11, the Clerk org claim.** Non-negotiable and first. Every callee
   (`/search`, `/embed`, `/rerank`) requires a verified Clerk RS256 token carrying a resolved
   internal `org_id` UUID. rag-service is a server with no user token of its own; it must either
   forward the caller's bearer or a service-to-service auth scheme must be designed. Neither
   exists. Until the claim is provisioned by `identity-tenant-service` + the Clerk dashboard,
   rag-service would be born as un-callable as the four routes already are. **Also fix the
   missing `CLERK_JWKS_URL`/`CLERK_ISSUER` in `.env.example`, or it 500s rather than 401s.**
2. **A provider API key and a decision on which model.** `.env.example:46-48` are empty slots.
   This is the first spend in the program; it wants a budget answer before the first call, not
   after.
3. **Vectors that exist to retrieve.** The document path is unblocked
   (`ingestion` → `document-chunks` → `embedding` → Qdrant) and can carry an MVP on its own. The
   transcript path is blocked behind audio hops 1–4 (item 5) — so an MVP built on transcripts
   would have nothing to retrieve.
4. **Item 13's Qdrant gate.** The vector arm's isolation control is currently unrun. Building a
   second Qdrant reader on top of an unverified control is how a cross-tenant leak arrives.
5. **A Dockerfile and a compose entry** for rag-service *and* for search-service, reranker-service
   and embedding-service — rag-service is useless without its three callees deployed alongside it.

---

## Part 4 — Deferred by ADR / external dependency vs actionable now

### DEFERRED — not engineering, do not schedule as work

| Item | Nature |
|---|---|
| Deepgram compliance gate (`compliance.py:1-32`, `COMPLIANCE_GATE_GREEN = False` at `:39`) | **Legal.** A signed DPA/SCC + zero-retention flag + subprocessor register. `compliance.py:6-8`: "It does not exist yet and cannot be produced by writing code — no PR against this repo can make condition 1 true." Correctly built as a source constant so no ops toggle can override it. |
| Region-local Postgres/Redpanda cells (`residency.py:9-16`) | **Infrastructure that does not exist in any repo.** One Postgres, one broker, everywhere. |
| GPU serving for embedding/reranker (`.env.example:37-41`) | **No CUDA device in any environment.** Ray Serve path unexercised by necessity. |
| gRPC mesh / `proto/rag.proto` | **Design decision, repeatedly ratified.** REST-now/gRPC-later is the established convention. Not a gap. |

### EXTERNAL PROCUREMENT — blocked, but not legal, and self-serve

| Item | Nature |
|---|---|
| LiveKit account (`transcription-service/.env.example:14-18`; `transport.py:135-141`) | No account exists, so egress and the audio hook cannot be exercised. This is **procurement, not a legal gate** — worth separating from Deepgram in planning, because the audio path (hops 2–4) unblocks the moment an account exists, whereas Deepgram needs a countersigned contract. Hop 1 (browser mic capture) needs neither and is actionable today. |

### ACTIONABLE NOW — every one of these can be done with what is on disk

Ordered by ratio of consequence to effort.

| # | Task | Files | Size | Depends on |
|---|---|---|---|---|
| 1 | Fix Redis port default drift in all four places | `mytheus-intelligence/.env.example:9`; `transcription-service/config.py:35`; `realtime-gateway/src/server.ts:593`; (+ `config.py:33` Postgres 5432→5433) | **S** | — |
| 2 | Add the missing `CLERK_*` vars to `mytheus-intelligence/.env.example`; make `auth.py:80,84` fail with a clear 503/config error rather than a bare `KeyError` → 500 | `.env.example`, `libs/shared/auth.py` | **S** | — |
| 3 | Add `{topic}.{group}.dlq` topics to `create_topics.sh`, or make the consumers emit to `{topic}.dlq` — decide which, in `mytheus-events`, and write the test that asserts producer topic names ⊆ created topics | `mytheus-events/scripts/create_topics.sh`, `mytheus-intelligence/libs/shared/consumer.py:253-258` | **S** | contract decision |
| 4 | Reconcile `DlqEnvelope.avsc` with the JSON reality — either register the subject and encode Avro, or restate the contract as JSON | `mytheus-events/contracts/DlqEnvelope.avsc`, `consumer.py:203-251` | **S** | #3 |
| 5 | Wire consumer liveness into `/healthz` | `transcription-service/src/transcription_service/main.py:114-116` | **S** | — |
| 6 | Add a `provider` (or `synthetic`) field to `TranscriptChunk` + `Utterance` and stamp it at the single write point, so fake finals are separable forever | `mytheus-events/schemas/transcript-chunks/v1.avsc` (BACKWARD-compat, needs a default), `providers/base.py:33-40`, `persistence.py:53-80` | **S–M** | — |
| 7 | Run the Qdrant isolation suite — set `MYTHEUS_REQUIRE_QDRANT=1` wherever tests run, once tests run somewhere | `tests/conftest.py:152,161,171` | **S** | #9 |
| 8 | Dockerfiles for all 7 intelligence services + both realtime services | new | **M** | — |
| 9 | CI for both repos (test + build + publish image), mirroring `mytheus-backend/.github/workflows/ci.yml` | new | **M** | #8 |
| 10 | Add the AI + realtime services to `compose.prod.yml`, plus Qdrant and MinIO | `mytheus-platform/deploy/compose.prod.yml` | **M** | #8, #9 |
| 11 | Least-privilege DB role for `transcription-service` (`mytheus_rt_app`), created + granted in `mytheus-backend`'s bootstrap, so its RLS calls actually bind | `mytheus-backend` migrations + `bootstrap_roles`, `transcription-service/db.py`, `config.py` | **M** | cross-repo coordination |
| 12 | Browser mic capture + publish a LiveKit audio track (audio hop 1) | `mytheus-ui/src/lib/realtime/useLiveKitRoom.ts`, meeting components | **M** | — (needs no account for local dev) |
| 13 | Provision the resolved-org-id Clerk claim (dashboard template + `identity-tenant-service` writing `public_metadata.internal_org_id` at org creation) | `mytheus-backend/services/identity_tenant_service`, Clerk config | **M** | — |
| 14 | `chat_messages` arm of the FTS index (`CLAUDE.md:38` specifies it; `routes.py:102` omits it) | `search_service/consumer.py`, new migration | **M** | — |
| 15 | Consume `embedding-events`, or delete the ack and its topic | `mytheus-intelligence`, `mytheus-events` | **M** | product decision |
| 16 | **`rag-service` MVP** — see Part 3 | `services/rag_service/` | **L (16–22 d)** | #2, #13, #8–#10, #7, provider key |
| 17 | `knowledge-graph-service` — provision the AGE extension, migration, Cypher write path on confirmed decisions/actions | `services/knowledge_graph_service/`, new migration | **L** | AGE extension on the Supabase image (unverified — check before committing to this) |
| 18 | `agent-runtime-service` — `BaseAgent`, window merging, six agents, tiered routing, budgets, `usage-events` | `services/agent_runtime_service/` | **XL** | #16 (shares the LLM plumbing), #13 |

**One correction to the framing in the brief.** Item 12 is stated as "`.env.example` Redis port
6379 contradicts every code default of 6380." That is exactly right for `mytheus-intelligence`
and exactly backwards for both `mytheus-realtime` services, where the `.env.example` is correct
and the *code* defaults to 6379. Fixing only the example files would leave two of the four
wrong.
