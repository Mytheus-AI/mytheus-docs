# AUDIT — AI / RAG / Agents tier (`mytheus-intelligence`)

READ-ONLY audit. No file in any repo was modified. All claims are from source, not from `.md`
status documents. Root: `/Users/sushan/Projects/Mytheus`.

---

## PER SERVICE — ACTUAL (quoted) / INTENDED / GAP

### 1. `search_service` — REAL, but lexical-only and orphaned

**ACTUAL.** Four source files (`main.py`, `routes.py`, `consumer.py`, `worker.py`).

`services/search_service/main.py:21-31` boots FastAPI, builds an asyncpg engine, mounts one
router:

```python
app = create_app(SERVICE_NAME)

_engine = create_async_engine(
    os.environ.get(
        "DATABASE_URL", "postgresql+asyncpg://mytheus_ai_app:mytheus_ai_app@localhost:5433/mytheus"
    ),
    pool_pre_ping=True,
)
_session_factory = async_sessionmaker(_engine, expire_on_commit=False)

app.include_router(create_search_router(session_factory=_session_factory))
```

The router (`services/search_service/routes.py:86-130`) exposes exactly one route,
`GET /search`. Its entire retrieval is one SQL statement (`routes.py:99-110`):

```sql
SELECT chunk_id, meeting_id, speaker_id, start_ms, end_ms, text,
       ts_rank(tsv, websearch_to_tsquery('english', :q)) AS rank
FROM intelligence.transcript_chunk_search
WHERE org_id = :org_id
  AND tsv @@ websearch_to_tsquery('english', :q)
-- then, conditionally:  AND meeting_id = :meeting_id
-- then:                 ORDER BY rank DESC LIMIT :limit
```

- **Qdrant: NOT queried.** `routes.py` imports no Qdrant client — the only non-stdlib imports
  are FastAPI, pydantic, SQLAlchemy, `libs.shared.auth`, and `libs.shared.tenancy`
  (`routes.py:46-56`). There is no vector arm, no RRF fusion, no hybrid retrieval of any kind.
  It is pure PostgreSQL full-text search.
- **Postgres full-text: YES**, via `websearch_to_tsquery('english', :q)` against a
  `GENERATED ALWAYS AS (to_tsvector(...)) STORED` column with a GIN index, created by
  `migrations/versions/d415f023220e_0003_search_service_transcript_index.py`.
- **Tenant-scoped: YES, and correctly, with two independent layers.**
  - `org_id` comes only from `Depends(verified_org_id)` (`routes.py:91`). That dependency
    (`libs/shared/auth.py:143-149`) resolves from a Clerk RS256 JWT verified against cached
    JWKS, with `nbf` required and JWKS outage mapped to 503. The previously-trusted
    client-supplied `X-Org-Id` header was removed; `routes.py:7-14` documents that fix
    explicitly ("anyone with network reachability could read another tenant's transcript
    index by supplying its id in that header").
  - Transaction-local RLS is applied **before** the SELECT: `await set_rls(session, org_id)`
    (`routes.py:113`), so the candidate set is constrained at planning time rather than after
    `ORDER BY rank LIMIT` clips it — plus a redundant explicit `WHERE org_id = :org_id`
    (`routes.py:102`). `routes.py:30-43` reasons about why both layers exist.
- **Index build is real, not mocked.** `services/search_service/consumer.py` defines:

  ```python
  class TranscriptChunkSearchConsumer(BaseConsumer):
      group = "search-service.transcript-index"
      topics = ["transcript-chunks"]
      producer_name = "search-service"
  ```

  with `apply()` inserting into `intelligence.transcript_chunk_search` under
  `ON CONFLICT (org_id, chunk_id) DO NOTHING`, skipping `is_final=false` and `redacted=true`
  messages. `services/search_service/worker.py` is a real entrypoint wiring it to Redpanda,
  Redis, Schema Registry and Postgres (`python -m services.search_service.worker`).
- **Ownership discipline is genuine.** `consumer.py`'s docstring records a refusal to
  `ALTER TABLE public.transcript_chunks` (owned by `transcription-service` in
  `mytheus-realtime`; this repo's roles hold zero grants on it), building its own read model
  instead. `chat_messages` FTS is declared BLOCKED — no `chat-messages` Kafka topic exists in
  the registered catalog and `core_api` writes those rows with no outbox path.

**INTENDED** (`mytheus-docs/09-rag-and-inference-serving.md:99-127`, §3.2, and
`_canon/01-service-catalog.md` as cited there):

- Be the **lexical (FTS) arm** of a locked three-arm hybrid retrieval — Qdrant vector +
  Postgres FTS + AGE topic relations — with all three issued in parallel and fused by
  Reciprocal Rank Fusion (k=60) into a top-50 candidate set.
- Own tsvector materializations over **both** `transcript_chunks` **and** `chat_messages`,
  "the FTS arm of hybrid retrieval for `rag-service`", isolated by RLS on `org_id`.
- Be reached by `rag-service` over the internal gRPC mesh, `org_id` in gRPC metadata.

**GAP.**
- The lexical arm exists and is well built; the two arms it must be fused with (vector, graph)
  have no caller, and the component that fuses them (`rag-service`) is a 13-line stub.
- `chat_messages` half of the FTS surface is absent (blocked, and honestly reported in source).
- The gRPC transport does not exist: `proto/rag.proto` contains only
  `message RagQueryPlaceholder { string query = 1; }` plus a
  `// TODO(RAG band): define RagService (Query), plus Embed/Rerank/GraphTraverse/FtsSearch`.
- **Nothing in any of the 8 repos calls `GET /search`.** Grep for `search-service`,
  `SEARCH_SERVICE`, `:8103` across all repos returns only this service's own files, its tests,
  docstrings in sibling modules, and one UI marketing comment
  (`mytheus-ui/src/lib/marketing/features.ts:241`).

---

### 2. `reranker_service` — REAL MODEL, genuinely loaded, ZERO callers

**ACTUAL.** Four source files (`main.py`, `model.py`, `rerank.py`, `serve.py`), 284 LOC.

Model identity and loading — `services/reranker_service/model.py:40-53`:

```python
MODEL_NAME = "BAAI/bge-reranker-base"

def gpu_requested() -> bool:
    return os.environ.get("RERANKER_GPU", "0") == "1"

@lru_cache(maxsize=1)
def _model() -> CrossEncoder:
    device = "cuda" if gpu_requested() else "cpu"
    model: CrossEncoder = CrossEncoder(MODEL_NAME, device=device)
    return model
```

Scoring is a real forward pass, **not** a passthrough — `model.py:68-83`:

```python
model = _model()
pairs = [[query, p] for p in passages]
t0 = time.perf_counter()
scores = model.predict(pairs)
elapsed = time.perf_counter() - t0
ranked = sorted(
    zip(passages, (float(s) for s in scores), strict=True),
    key=lambda pair: pair[1],
    reverse=True,
)
k = max(1, min(top_k, len(ranked)))
return ranked[:k], elapsed
```

- **Which model:** `BAAI/bge-reranker-base`, the real published cross-encoder weights, loaded
  through `sentence_transformers.CrossEncoder` (which wraps the same
  `AutoModelForSequenceClassification` architecture `FlagReranker` uses).
- **Actually loaded/served:** yes. `model.py:3-12` documents the substitution honestly: the
  plan's `FlagReranker` + `use_fp16=True` GPU path is blocked (no CUDA in this environment), so
  it loads fp32 on CPU. That is a device downgrade, not a fake scorer.
- **HTTP surface:** `rerank.py:58-84` creates `POST /rerank` (`RerankRequest` capped at
  `_MAX_PASSAGES = 50`, `rerank.py:39`), mounted at `/internal` by `main.py:16`
  (`app.mount("/internal", create_rerank_app())`). `org_id` comes from `verified_org_id`
  (`rerank.py:64`) and is used for audit logging only — it gates no storage decision, because
  the service owns no data (`main.py:4-5`: "no Postgres table, no Qdrant creds, no consumer,
  no relay").
- **Ray Serve deployment is real:** `serve.py:73-101` defines `RerankerDeployment` with
  `autoscaling_config={"min_replicas": 1, "max_replicas": 4, "target_ongoing_requests": 8}`
  (the warm-replica floor) and `num_gpus` requested only when `RERANKER_GPU=1` **and** Ray
  reports a visible GPU (`serve.py:69-71`). `serve.py` stays import-safe without Ray.
- **Deliberate no-batching:** there is no `@serve.batch` anywhere; `model.py:14-29` and
  `serve.py:21-27` document this as a cross-tenant isolation decision — fanning concurrent
  `/rerank` callers into one `model.predict()` list could score two tenants' passages together.

**INTENDED** (ADR-0008; `09-rag-and-inference-serving.md` §5 and lines 127-151):

- A dedicated, GPU-backed (Ray Serve / vLLM), independently autoscaling cross-encoder service
  completely off the API tier, with its own cost center.
- Called by `rag-service` over internal gRPC on the warm path: top-50 RRF candidates in,
  top-10 out.
- Warm p95 < 500 ms, cold p95 < 1.5 s, held up by a warm-pool replica floor per active region.

**GAP.**
- **Orphaned.** Grepping every repo for `rerank` (excluding the service's own directory and its
  test) yields only prose: `search_service/routes.py:16-27` (which shapes its response field
  `passage` to feed `/rerank` "with zero adapter" but never calls it), `rag_service/main.py:1`,
  `libs/shared/auth.py:5,11`, `ingestion_service/routes.py:6`, `test_healthz.py:18`,
  `test_search_routes.py:156,181`, plus two unrelated string matches in
  `mytheus-realtime/.../devseed.py:88` and `mytheus-backend/tests/test_seed.py:4`.
  **No code anywhere issues a request to `/rerank`.**
- GPU path and both numeric latency SLOs are unmet (CPU-only, acknowledged in source).
- gRPC transport absent (same `RagQueryPlaceholder` gap).

---

### 3. `rag_service` — STUB. Full contents, verbatim (13 lines)

`services/rag_service/main.py`:

```python
"""rag-service: the frozen five-stage RAGService (expand, retrieve, rerank, assemble, cite).

Scaffold only: boots and answers `/healthz`. RRF fusion across the Qdrant/AGE/FTS arms over the
internal gRPC mesh (org_id in metadata) lands in the RAG band.

Run: uvicorn services.rag_service.main:app --port 8101
"""

from __future__ import annotations

from libs.shared.app import create_app

app = create_app("rag-service")
```

`services/rag_service/__init__.py` is 0 bytes. That is the entire service.

**INTENDED** (`mytheus-docs/09-rag-and-inference-serving.md:19-74`, §2 and §3):

- Own `POST /api/rag/query` and run the frozen five-stage pipeline
  `expand → retrieve → rerank → assemble → cite`, fronted by a Redis answer cache keyed
  `nexus:{org_id}:rag:{user_id}:{query_hash}` with a 10-minute TTL.
- Fan out three retrieval arms in parallel per query variant (Qdrant vector via
  `embedding-service POST /embed`; `search-service GET /search` for FTS;
  `knowledge-graph-service` for AGE `RELATES_TO` / `FOLLOWS_FROM`), fuse by RRF k=60 to top-50,
  then call `reranker-service POST /rerank` for top-10.
- Assemble under a token budget with dedupe/diversity, bind every passage to its provenance
  (`meeting_id`/`chunk_id`/`speaker` or `document_id`/`element_id`) with no shortcut path that
  skips citation, and emit `usage-events` for LLM token metering. Warm p95 < 500 ms.

**GAP.** 100% of the above is absent — no route, no cache, no fan-out, no fusion, no assembly,
no citation binding, no metering. Stage 1 (`expand`) is itself an LLM call via LiteLLM; as the
LLM section below establishes, there is no LLM client anywhere in the repo, so even the first
stage has no substrate to build on.

---

### 4. `agent_runtime_service` — STUB. Full contents, verbatim (13 lines)

`services/agent_runtime_service/main.py`:

```python
"""agent-runtime-service: the six LangGraph agents + cost-controlled LLM pipeline.

Scaffold only: boots and answers `/healthz`. The `BaseAgent` lifecycle, tiered routing, per-tenant
budgets, and the `transcript-chunks -> agent-results/agent-stream` flow land in the agent band.

Run: uvicorn services.agent_runtime_service.main:app --port 8100
"""

from __future__ import annotations

from libs.shared.app import create_app

app = create_app("agent-runtime-service")
```

`services/agent_runtime_service/__init__.py` is 0 bytes.

**INTENDED** (`mytheus-docs/08-ai-agent-architecture.md:19-70` and `152-181`; ADR-0004;
ADR-0013; `plan/plan-7-9-agents-and-monetization.md` phases P7–P9):

- Host six LangGraph agents on a shared `BaseAgent`/`AgentState` contract —
  `PersonalListenerAgent` (P7, one per participant per meeting, relevance-scored worklogs),
  `DecisionAgent` and `ActionAgent` (P8, confidence-scored candidates only, never silent
  commits), `QuestionAgent` (P9), `ContextGapAgent` (P10, peer-agent fallback chain over
  `agent-requests`/`agent-responses`), `ProactiveAgent` (P11, conflict/workload/stakeholder
  detectors) — consuming 30-second `transcript-chunks` windows and emitting to `agent-results`.
- Run the five-lever cost-control pipeline (ADR-0004): Haiku relevance pre-gate (~35% pass,
  65% dropped and logged) → 30s windowed batching (90 windows/meeting vs 360 raw chunks) →
  LiteLLM tiered Haiku/Sonnet/Opus routing → semantic + Anthropic prompt caching → per-tenant
  `agent_budgets` with Redis hot mirror `nexus:{org_id}:budget:{period}`, alerts at 80%/95%,
  deny-by-default at `hard_cap`, token usage emitted on `usage-events`.
- Enforce deny-by-default, same-tenant-only, role/scope-checked, fully-audited agent-to-agent
  authorization derived from bound identity and never from the message payload (ADR-0013),
  with foreign-tenant requests routed to the per-topic DLQ.

**GAP.** Total. No agent class, no LangGraph graph, no `BaseAgent`, no `AgentState`, no
windowing, no pre-gate, no LiteLLM router, no `agent_budgets` table, no `agent-results`
producer, no `agent_collaborations` logging, no authorization check. The service is the
docstring.

---

### 5. `knowledge_graph_service` — STUB. Full contents, verbatim (13 lines)

`services/knowledge_graph_service/main.py`:

```python
"""knowledge-graph-service: the ONLY writer of the AGE `nexus_knowledge` graph.

Scaffold only: boots and answers `/healthz`. Cypher parameterized by `$org_id`, driven by confirmed
`decisions`/`actions` from core-api, lands in the knowledge-graph band. No silent graph writes.

Run: uvicorn services.knowledge_graph_service.main:app --port 8102
"""

from __future__ import annotations

from libs.shared.app import create_app

app = create_app("knowledge-graph-service")
```

`services/knowledge_graph_service/__init__.py` is 0 bytes.

**INTENDED** (ADR-0005; `09-rag-and-inference-serving.md:99-127`;
`08-ai-agent-architecture.md` §2.2):

- Be the **sole writer** of the org-partitioned Apache AGE `nexus_knowledge` graph (Cypher over
  PostgreSQL), every template parameterized by `$org_id`, with locked exits to Neptune/Neo4j
  Aura or a relational materialized-view fallback (ADR-0005 retires risk R1).
- Create `PRODUCED` / `ASSIGNED_TO` edges **only on user confirmation** of a decision or action,
  driven by `core-api`'s transactional-outbox `decisions`/`actions` events — the "no silent
  commits" rule.
- Serve `POST /graph/query/{template}` as the graph arm of hybrid retrieval (`RELATES_TO`
  traversal plus `FOLLOWS_FROM` decision lineage), and back `ProactiveAgent` conflict detection
  and missing-stakeholder org-graph traversal.

**GAP.** Total — and deeper than the service itself: the graph store it is meant to write does
not exist anywhere in the program (see KNOWLEDGE GRAPH below). Its own upstream does not exist
either: there are no `decisions`/`actions` tables and no confirm routes in `core-api`.

---

## LLM INFERENCE IN THE CODEBASE — the full grep result and verdict

### Method

Searched **all 8 repos** under `/Users/sushan/Projects/Mytheus` (`mytheus-backend`,
`mytheus-docs`, `mytheus-events`, `mytheus-intelligence`, `mytheus-mobile`, `mytheus-platform`,
`mytheus-realtime`, `mytheus-ui`), case-insensitive, over `*.py *.ts *.tsx *.js *.go *.java
*.kt *.rs *.toml *.txt *.json *.yaml *.yml *.dart`, for:

```
anthropic | openai | claude | gpt- | litellm | vllm | ollama | langchain |
llama_index | langgraph | completion( | chat.completions
```

Excluded: `.venv/`, `node_modules/`, `.worktrees/` (worktree copies of the same intelligence
repo), `package-lock.json`, `build/`, `dist/`. Also separately grepped for `create extension`,
`ag_catalog`, `neo4j`, and `cypher`.

### Complete result — 6 hits. Nothing else in the entire monorepo set.

| # | file:line | matched text | live code / fixture / comment |
|---|---|---|---|
| 1 | `mytheus-intelligence/pyproject.toml:9` | `"langgraph>=0.2",` | **declared dependency — ZERO imports anywhere in any repo** |
| 2 | `mytheus-intelligence/pyproject.toml:10` | `"litellm>=1.40",` | **declared dependency — ZERO imports anywhere in any repo** |
| 3 | `mytheus-intelligence/pyproject.toml:19` | `"langchain-text-splitters>=0.3",` | live dependency, but a text splitter — no inference |
| 4 | `mytheus-intelligence/pyproject.toml:47` | `# Placeholder: real GPU serving (BGE-small-en / BGE-reranker-base on Ray Serve/vLLM)` | **comment** |
| 5 | `mytheus-intelligence/libs/shared/splitter.py:18` | `from langchain_text_splitters import RecursiveCharacterTextSplitter` | **live code** — 600/60 character chunking only. No model, no LLM. |
| 6 | `mytheus-intelligence/services/agent_runtime_service/main.py:1` | `"""agent-runtime-service: the six LangGraph agents + cost-controlled LLM pipeline.` | **docstring inside a 13-line stub** |

Non-code, worth recording — `mytheus-intelligence/.env.example:46-48`:

```
# LLM routing (LiteLLM). Provider keys land per-deployment; never commit real values.
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
```

Both values empty. No other repo declares an LLM provider key in any `.env`, compose file, or
secret template. `mytheus-platform/deploy/compose.prod.yml` contains no LLM-related environment
variable.

### What was NOT found — explicitly

- **Zero** occurrences of `anthropic` or `openai` as a Python/TS **identifier or import** in any
  `.py`, `.ts`, `.tsx`, `.js`, `.go`, `.dart` file across all 8 repos.
- **Zero** occurrences of `claude-<digit>` or `gpt-<digit>` (no model id string exists anywhere).
- **Zero** `chat.completions`, **zero** `litellm.completion(`, **zero** `client.messages.create`.
- **Zero** `llama_index`, **zero** `ollama`, **zero** `vllm` outside the single comment at
  `pyproject.toml:47`.
- **Zero** `import langgraph` / `from langgraph`.
- No HTTP client anywhere pointed at `api.anthropic.com`, `api.openai.com`, or any inference
  gateway host.

### VERDICT

**No LLM call happens anywhere in this codebase.** Not in live code, not in a test fixture, not
behind a feature flag, not in a disabled branch. Two LLM libraries (`litellm`, `langgraph`) are
installed as declared dependencies and are **never imported by a single line of source**. The
third `lang*` package is a character-based text splitter.

The only real ML inference in the entire monorepo is two locally-loaded BGE encoder models via
`sentence_transformers`:

- `services/embedding_service/model.py` — BGE-small-en, 384-dim bi-encoder (embeddings)
- `services/reranker_service/model.py:52` — `CrossEncoder("BAAI/bge-reranker-base")` (reranking)

Both are **retrieval** components. Neither generates text. Consequently, across the whole
product: nothing summarizes, nothing extracts a decision or an action, nothing answers a
question, nothing scores relevance with a model, nothing produces a citation, and no token is
ever metered — because no token is ever produced. For a product branded on "six agents" and
"answers with citations", **the generative layer is entirely absent.**

---

## SEARCH END-TO-END TRACE — UI search box → ?

| Hop | Status | Evidence |
|---|---|---|
| 1. `mytheus-ui/src/app/(app)/search/page.tsx` — the search box | **BROKEN (mock)** | `page.tsx:5` `import { mockSearchIndex, type SearchResult } from "@/lib/mock/data";`. `page.tsx:26-33`: `const results = useMemo(() => { const q = query.trim().toLowerCase(); if (!q) return mockSearchIndex; return mockSearchIndex.filter((item) => item.title.toLowerCase().includes(q) \|\| item.snippet.toLowerCase().includes(q)); }, [query]);` — **no `fetch`, no API client import, no server action, no route handler.** It is a client-side substring filter over a hardcoded array. |
| 2. → BFF (`mytheus-backend/services/bff`) | **ABSENT** | grep for `search`/`rerank`/`rag`/`knowledge_graph`/`agent_runtime` across `services/bff` and `services/api_gateway`: **zero hits**. No search route exists in the BFF. |
| 3. → api-gateway upstream to the intelligence tier | **ABSENT** | same grep, zero hits; no upstream host, no env var, no proxy rule for search-service anywhere. |
| 4. → `search-service GET /search` | **WIRED but unreachable** | route is real (`services/search_service/routes.py:86-130`) and correct, but has no caller in any of the 8 repos. |
| 5. → `intelligence.transcript_chunk_search` (Postgres FTS) | **WIRED** | consumer (`search_service/consumer.py`), worker (`worker.py`), and migration `d415f023220e` are all real and functional. |
| 6. → `reranker-service POST /rerank` | **ABSENT** | no code path calls it (see reranker section). |
| 7. → `rag-service` fusion / citation | **ABSENT** | 13-line stub. |

**The chain does not complete — it breaks at hop 1.** There is no network request at all.

Branch check (per the standing note that `mytheus-ui` has an active rebuild branch): I inspected
`origin/local-mytheus-rebuild` without fetching. Its `src/app/(app)/search/page.tsx` is the same
file — `mockSearchIndex` import at line 5, same `.filter()` at lines 28-29. So this is not a
stale-branch artifact on `main`. **UNVERIFIED:** I did not run `git fetch`, so the rebuild branch
is judged as of the last local fetch; anything pushed since is not reflected here.

---

## KNOWLEDGE GRAPH — exists at all?

**No. It does not exist in any form, in any repo.**

- **No AGE extension.** Grep for `create extension` across all repos returns exactly three
  hits, all `pgcrypto`:
  - `mytheus-intelligence/migrations/versions/9c2b7e4a1d03_0001_intelligence_schema_outbox_inbox_roles.py:95`
  - `mytheus-realtime/services/transcription-service/migrations/versions/be66698dbefb_create_transcript_chunks.py:86`
  - `mytheus-backend/migrations/versions/82d44163c933_0001_core_tables_outbox_inbox_.py:39`
- **Zero** occurrences of `ag_catalog`, `LOAD 'age'`, or the `apache/age` Docker image in any
  code, SQL, migration, or compose file. **Zero** Neo4j. No Cypher statement exists outside
  `mytheus-docs`.
- **No graph table, no node/edge storage, no migration** creating `nexus_knowledge`. The name
  appears only in prose: `knowledge_graph_service/main.py:1,3`; `rag_service/main.py:3`;
  `mytheus-backend/services/identity_tenant_service/seed/seed.py:43,456` (which explicitly
  declares Topic-node seeding **out of scope**, owned by knowledge-graph-service);
  `mytheus-backend/tests/test_role_topics_taxonomy.py:11-13` (same, and notes mytheus-backend
  "reads `nexus_knowledge` but never writes it"); `mytheus-backend/tests/test_seed.py:15`;
  and `mytheus-backend/services/workflow_service/workflows/rtbf.py:4,18` (an RTBF workflow with
  a `# 3. AGE tag + remove nodes/edges` step against a store that doesn't exist).

**Backend for the UI `/decisions` page: also absent.**

- `mytheus-ui/src/app/(app)/decisions/page.tsx:6` — `import { mockDecisions } from "@/lib/mock/data";`
  and `page.tsx:23` sorts that array. Static mock, no fetch.
- There is **no `decisions` and no `action_items` table** in any migration in any repo — grep
  for `CREATE TABLE .*(decision|action_item)` across all `.py` and `.sql`: zero hits.
- `mytheus-backend/services/core_api/main.py` states it plainly:
  - line 3: `Owns: meetings, participants, speakers, worklogs, decisions, action_items, agent_suggestions,`
  - lines 6-7: `Confirmation rule: decisions/actions events are ONLY emitted by this service's transactional outbox, ONLY on POST /decisions/:id/confirm or /actions/:id/confirm. Neither route exists yet,`
  - line 22: `# TODO: /decisions /actions /worklogs /suggestions land with their own bands.`

So the knowledge-graph marketing page and the `/decisions` app page both describe a tier with
no store, no writer, no reader, no producer, and no data. The confirmation flow that ADR-0005
and `08-ai-agent-architecture.md` §2.2 make the *entire* basis for graph writes is unbuilt at
both ends: the agent that would propose a candidate is a stub, and the route that would confirm
one does not exist.

---

## DEPLOYED?

**None of the five. Nothing from `mytheus-intelligence` is deployable at all.**

1. **`mytheus-platform/deploy/compose.prod.yml`** — the complete service list:
   `postgres` (:78), `redis` (:105), `redpanda` (:117), `temporal` (:137), `migrate` (:161),
   `web` (:177, `services.core_api.main:app`), `relay` (:210), `workflow` (:226),
   `api-gateway` (:252), `bff` (:270), `identity-tenant-service` (:277),
   `meeting-service` (:293), `integration-service` (:322), `billing-metering-service` (:329),
   `admin-ops-service` (:342), `notification-service` (:349), `ui` (:364).
   **Absent: search-service, reranker-service, rag-service, agent-runtime-service,
   knowledge-graph-service — and also embedding-service, ingestion-service, and Qdrant.**
   Grep of that file for `intelligence|embedding|ingestion|search|rerank|rag|knowledge|qdrant`
   matches only one unrelated word inside a comment at line 38.
2. **`mytheus-intelligence/docker-compose.yml`** — defines exactly two services: `qdrant`
   (:19, `qdrant/qdrant:v1.12.4`) and `minio` (:52). These are dev **dependencies**. No
   application service from this repo is defined in it. Its header comment explains Postgres/
   Redis/Redpanda/Schema-Registry are deliberately left to the mytheus-backend stack.
3. **`find mytheus-intelligence -name "Dockerfile*"` → no results.** The intelligence repo has
   **no container image at all**, so no orchestrator could run any of these five services even
   if a compose entry were added.
4. `mytheus-platform/docker-compose.yml` defines no services matching any of the five.

---

## VERDICT — how much of the "AI" tier is real

Of the five services audited, one and a half are real: `search-service` is a well-engineered,
properly tenant-isolated PostgreSQL full-text search over its own event-sourced read model with
JWT-derived `org_id` and dual RLS/explicit-filter defense, and `reranker-service` genuinely
loads and runs the real BGE-reranker-base cross-encoder — but neither has a single caller
anywhere in the codebase, and the three services meant to orchestrate them (`rag-service`,
`agent-runtime-service`, `knowledge-graph-service`) are byte-identical 13-line `create_app()`
scaffolds totalling 39 lines against roughly two hundred pages of specification.

The product is AI-branded, yet **no LLM inference occurs anywhere in the monorepo**: `litellm`
and `langgraph` are installed and never imported by a single line, the provider key slots in
`.env.example` are empty, and the only models that execute are two local BGE encoders used for
retrieval — so nothing generates text, which means no agents, no extracted decisions, no
answers, and no citations exist as running behaviour rather than as prose.

The knowledge graph that ADR-0005 and three separate service contracts depend on has no Apache
AGE extension, no migration, no table and no writer, and its upstream is equally absent
(`core-api` has no `decisions`/`action_items` tables and its confirm routes are a TODO), while
the UI pages for both search and decisions are client-side filters over hardcoded mock arrays
with nothing behind them on `main` or on the rebuild branch.

And none of it ships: `mytheus-intelligence` has no Dockerfile and appears in no deploy compose
file, so the entire AI tier is code that has never run anywhere outside a test suite.
