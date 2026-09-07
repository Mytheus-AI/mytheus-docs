# Agent tier — reality audit (read-only, 2026-09-07)

Root: `/Users/sushan/Projects/Mytheus`. Every claim below is from source read this session.

---

## 1. The three "AI tier" services — quoted in full

### `mytheus-intelligence/services/agent_runtime_service/`

Two files: `__init__.py` (0 bytes) and `main.py` (13 lines). That is the entire service.

`mytheus-intelligence/services/agent_runtime_service/main.py:1-13` — complete file:

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

### `mytheus-intelligence/services/rag_service/`

Two files. `services/rag_service/main.py:1-13` — complete file:

```python
"""rag-service: the frozen five-stage RAGService (expand, retrieve, rerank, assemble, cite).

Scaffold only: boots and answers `/healthz`. RRF fusion across the Qdrant/AGE/FTS arms over the
internal gRPC mesh (org_id in metadata) lands in the RAG band.

Run: uvicorn services.rag_service.main:app --port 8101
"""
...
app = create_app("rag-service")
```

### `mytheus-intelligence/services/knowledge_graph_service/`

Two files. `services/knowledge_graph_service/main.py:1-13` — complete file:

```python
"""knowledge-graph-service: the ONLY writer of the AGE `nexus_knowledge` graph.

Scaffold only: boots and answers `/healthz`. Cypher parameterized by `$org_id`, driven by confirmed
`decisions`/`actions` from core-api, lands in the knowledge-graph band. No silent graph writes.

Run: uvicorn services.knowledge_graph_service.main:app --port 8102
"""
...
app = create_app("knowledge-graph-service")
```

`create_app` (`libs/shared/app.py:15-22`) mounts exactly one route: `GET /healthz` returning
`{"status": "ok", "service": name}`. No routers, no lifespan, no consumers.

**Verdict: all three are 13-line health-probe stubs. There is no agent tier.**

---

## 2. BaseAgent / lifecycle / prompt loader / prompt templates / LangGraph graph

Grep across `mytheus-intelligence`, `mytheus-backend`, `mytheus-realtime`, `mytheus-events`
(`*.py`, `*.ts`, `*.tsx`; excluding `.venv`, `node_modules`, `.worktrees`, `__pycache__`) for
`BaseAgent|StateGraph|prompt|prompt_template|PROMPTS_DIR`:

**One hit, and it is a docstring:**
`mytheus-intelligence/services/agent_runtime_service/main.py:3` — "The `BaseAgent` lifecycle…"

- No `BaseAgent` class — ABSENT.
- No agent lifecycle — ABSENT.
- No prompt loader — ABSENT.
- No prompt template directory (`find -type d -iname "*prompt*"` → zero results anywhere in the
  workspace) — ABSENT.
- No LangGraph graph definition, no `StateGraph`, no node/edge/workflow-node code — ABSENT.
- No LLM call site of any kind — ABSENT.

Note: `mytheus-backend/services/workflow_service/workflows/action_followup.py` exists but is a
*business* workflow ("driven by confirmed actions"), not an agent graph.

---

## 3. `litellm` / `langgraph`: declared, resolved, installed, never imported

Confirmed. `grep -rn "langgraph\|litellm" services libs tests migrations` in
`mytheus-intelligence` returns **zero** matches. The only references are dependency declarations.

Declared — `mytheus-intelligence/pyproject.toml:9-10`:
```toml
    "langgraph>=0.2",
    "litellm>=1.40",
```

Resolved in `uv.lock` (and physically present in `.venv/lib/python3.12/site-packages/`):

| package | locked version | lock line |
|---|---|---|
| `langgraph` | **1.2.7** | `uv.lock:1908-1909` |
| `langgraph-checkpoint` | 4.1.1 | `uv.lock:1925-1926` |
| `langgraph-prebuilt` | 1.1.0 | `uv.lock:1938` |
| `langgraph-sdk` | 0.4.2 | `uv.lock:1951` |
| `litellm` | **1.90.2** | `uv.lock:2067-2068` |

The floors (`>=0.2`, `>=1.40`) are Plan-0-era and now three majors behind what actually resolved —
`langgraph` 0.2 → 1.2.7 is a **major API break** (`langgraph` 1.x). Any plan written against
"langgraph 0.2" idioms will not compile against the locked tree.

---

## 4. What DOES exist that the agent tier can build on

This is the genuinely good news: the *plumbing* is real, tested, and tenant-safe. The agent tier is
missing its brain, not its body.

### 4.1 `libs/shared/consumer.py` — `BaseConsumer` (320 lines) — **the single most reusable asset**

`mytheus-intelligence/libs/shared/consumer.py:11-13` names `agent-runtime-service` as an intended
subclass by name:

> "tasks 8-10 (`ingestion-service`, `search-service`, `agent-runtime-service`) subclass it rather
> than re-implement dedupe/DLQ/offset handling."

Interface a subclass implements (`consumer.py:11-15`, `:70-73`, `:95-96`):
```python
class MyConsumer(BaseConsumer):
    group: ClassVar[str]          # e.g. "agent-runtime-service.agents-on-transcript"
    topics: ClassVar[list[str]]   # e.g. ["transcript-chunks"]
    producer_name: ClassVar[str]  # kebab-case, stamps EventEnvelope.producer
    async def apply(self, evt: ConsumedEvent, session: AsyncSession) -> None: ...
```
Construction (`consumer.py:77-93`): `session_factory`, `redis`, `codec: AvroCodec`,
`dlq_producer: Producer`.

`ConsumedEvent` (`consumer.py:59-67`): `topic, org_id: UUID, idempotency_key, header, body,
partition, offset` — `body` is the decoded Avro value with `header` stripped.

Guarantees already enforced before `apply` runs (`consumer.py:17-30`, implementation `:127-173`):
deserialize-failure → DLQ `DESERIALIZE`; missing/unparseable `org_id` → DLQ `BUSINESS_INVALID`
(never guessed, never defaulted); Redis + durable-inbox dedupe; `apply` + inbox `record_consumed`
in **one** transaction scoped by `SET app.current_org`; transient exceptions propagate so the
offset is not committed and the broker redelivers.

Output path: `emit_outbox(session, evt, topic=, idem=, payload=)` (`consumer.py:98-125`) — "the one
sanctioned way a subclass's `apply` acknowledges its work", same transaction as the inbox insert,
auto-populating `correlation_id`/`causation_id` from the inbound header.

**An agent consumer of `transcript-chunks` producing `agent-results` is a ~50-line subclass of this
class.** That is the shape the work should take.

### 4.2 Outbox / inbox / relay — EXISTS, real, migrated

- `libs/shared/outbox.py` (60 lines) — `enqueue(session, envelope)`.
- `libs/shared/inbox.py` (80 lines) — `is_duplicate`, `already_recorded`, `record_consumed`.
- `libs/shared/relay.py` (137 lines) — `relay_once`: polls `intelligence.outbox`
  `FOR UPDATE SKIP LOCKED` (`relay.py:41-47`), Avro-serializes against `{topic}-value` via the real
  Schema Registry, produces keyed by `org_id` (acks=all, idempotent), marks published.
  `MAX_ATTEMPTS = 5` (`relay.py:39`).
- Entrypoint: `services/embedding_service/relay_main.py` (76 lines) — the pattern to copy.
- Migration `migrations/versions/9c2b7e4a1d03_0001_intelligence_schema_outbox_inbox_roles.py`
  creates the `intelligence` schema, outbox/inbox, RLS `tenant_isolation`, and a single permissive
  `outbox_relay_drain` policy scoped to the `mytheus_ai_relay` role (`relay.py:17-22`).

**Reusable verbatim.** An agent-tier outbox row lands on `agent-results` with zero new
infrastructure.

### 4.3 `libs/shared/events.py` — `EventEnvelope` — EXISTS

`events.py:29-64`. Fields: `topic, payload, org_id, producer, idempotency_key, event_id,
schema_version, occurred_at, correlation_id, causation_id`. `schema_subject` → `f"{topic}-value"`;
`partition_key` → `org_id`; `to_value()` → `{"header": …, **payload}`.

**Caveat for the agent tier** (`events.py:11-14`): ADR-0017 makes `transcript-chunks` and
`agent-stream` the *only* topics not partitioned by plain `org_id` — they use a composite
`{org_id}:{meeting_id}` key (confirmed on the producer side at
`mytheus-realtime/services/transcription-service/src/transcription_service/persistence.py:47`).
`EventEnvelope.partition_key` returns plain `org_id` and **does not** support the composite form.
Producing `agent-stream` through the existing outbox/relay path therefore needs a partition-key
override — a real, small, currently-unbuilt gap.

### 4.4 `libs/shared/qdrant.py` — tenant-isolated vector access — EXISTS (183 lines)

ADR-0006 chokepoint, resolved. Interface:
- `write_client()` (`qdrant.py:75-88`) — requires `QDRANT_API_KEY`; **embedding-service only**.
- `read_client()` (`qdrant.py:91-105`) — requires `QDRANT_READ_ONLY_API_KEY`; explicitly named as
  what "`rag-service`, `search-service`, tooling" use. Refuses to fall back to the write key, and
  `_distinct_or_raise` (`qdrant.py:61-72`) fails startup if both keys are the same value.
- `physical_collection(logical, org_id) -> f"{logical}__{org_id.hex}"` (`qdrant.py:108-112`) — the
  only sanctioned name resolver.
- `search_with_tenant_filter(client, logical, org_id, query_vector, *, limit=50)`
  (`qdrant.py:155-174`) — collection boundary **and** payload `org_id` filter, both layers.
- Logical names frozen (`qdrant.py:43-45`): `transcript_chunks`, `document_chunks`, `decisions`.
- `EMBEDDING_DIM = 384` (BGE-small-en) (`qdrant.py:40`).

**Directly reusable** as the agent tier's retrieval arm — this is what a RAG stage would call.

### 4.5 `embedding-service` — EXISTS, real model

- `services/embedding_service/model.py` — real `BAAI/bge-small-en-v1.5` via
  `sentence-transformers`, `embed_batch()`.
- `services/embedding_service/serve.py` — two Ray Serve deployments: `IngestEmbedder` (batch 32,
  50ms wait, backs the Kafka consumer) and `QueryEmbedder` (backs `POST /embed`), deliberately
  separated so a query burst can't starve ingest (`serve.py:3-7`).
- `services/embedding_service/query_embed.py` (91 lines) — real `POST /embed`, 1-3 query variants,
  Redis-cached at `nexus:{org_id}:qembed:{query_hash}` ~10min TTL. `query_embed.py:5-6` states it
  is built as "the contract… consumed later by `rag-service`'s `expand` -> `retrieve` fan-out".
  Falls back to direct `embed_batch` when Ray is absent, so it works in dev.
- `services/embedding_service/consumer.py:50-52` — `group="embedding-service.embed-on-ingest"`,
  `topics=["transcript-chunks", "document-chunks"]`. **Already consuming the transcript stream.**

**Reusable:** the agent tier gets query embedding over REST today, gRPC later, no changes needed.

### 4.6 `search-service` — EXISTS (FTS arm)

- `services/search_service/consumer.py:77-83` — `group="search-service.transcript-index"`,
  `topics=["transcript-chunks"]`. Builds and owns its **own** FTS index materialized only from the
  topic (`consumer.py:9,19`) — deliberately not `ALTER TABLE` on transcription-service's table.
- `services/search_service/routes.py` — `GET /search` (132 lines), `_MAX_LIMIT = 50`
  (`routes.py:58`). Response shape **engineered to feed `POST /rerank` with zero adapter**
  (`routes.py:16-21`): each hit's text field is literally named `passage` to match
  `RerankedPassage.passage`; hits also carry `chunk_id, meeting_id, speaker_id, start_ms, end_ms`
  for the `cite` stage's provenance re-binding.
- Tenant isolation belt-and-braces: `set_rls` **before** the SELECT plus an explicit
  `WHERE org_id = :org_id` (`routes.py:30-43`), so the candidate set is constrained at plan time,
  not after `ORDER BY rank LIMIT`.

**Reusable as-is** as the RAG pipeline's keyword arm.

### 4.7 `reranker-service` — EXISTS

- `services/reranker_service/model.py` + `rerank.py` (84 lines) — real `BAAI/bge-reranker-base`
  cross-encoder, `POST /rerank`, request `{query, passages}` → scored passages.
- `services/reranker_service/serve.py` — Ray Serve, `min_replicas=1` warm floor
  (`serve.py:13-19`). **Deliberately no `@serve.batch`** (`serve.py:21-27`): batching concurrent
  `/rerank` callers would score two tenants' passages in one `model.predict()`. Each `__call__`
  scores exactly one request.

**Reusable as-is** as the rerank stage.

### 4.8 `ingestion-service` — EXISTS

`routes.py` (207), `pipeline.py` (205), `parser.py` (150, pdfminer/docx/pptx/md), `pii.py` (70),
`consumer.py` (154, `group="ingestion-service.attachment-to-ingest"`, `topics=["integration-events"]`),
plus `libs/shared/object_storage.py` (152, real S3/MinIO). Relevant to the agent tier only as a
document source.

### 4.9 `libs/shared/auth.py` — EXISTS (149 lines)

`verified_org_id` FastAPI dependency: Clerk RS256 against cached JWKS, `nbf` required, JWKS outage
→ 503 not 500. Reads `CLERK_JWT_INTERNAL_ORG_CLAIM`, which is expected to carry the
**already-resolved internal `org_id` UUID**, not Clerk's opaque `org_...` id (`auth.py:38-44`) —
and that Clerk-dashboard mapping is explicitly **not yet made** (`auth.py:43-44`). A real
outstanding dependency for any new authenticated agent REST surface.

### 4.10 Avro schemas — EXISTS, all four agent topics registered

`mytheus-events/schemas/`, with fixtures, subjects, golden roundtrip tests, and topic-creation.
`mytheus-events/scripts/create_topics.sh:55-61` creates `agent-results` (retention 120, D3),
`agent-stream` (120, H1), `agent-requests` (12, D1), `agent-responses` (12, D1), each with a `.dlq`.

**`schemas/agent-results/v1.avsc`** (subject `agent-results-value`, BACKWARD) — fields
(`:7-19`): `header`, `meeting_id` (uuid), `window_id` (string), `user_id` (uuid),
`agent` enum `AgentClass` ∈ {`PersonalListenerAgent`, `DecisionAgent`, `ActionAgent`,
`QuestionAgent`, `ContextGapAgent`, `ProactiveAgent`}, `kind` enum `AgentResultKind` ∈
{`note`, `decision_candidate`, `action_candidate`, `suggestion`, `alert`}, `relevance_score`
(float), `confidence` (float), `content` (string), `payload_json` (string, "Free-form per kind"),
`source_chunk_ids` (array of uuid). Doc line 5: "bus is transient transport, read models persist
in core-api."

**`schemas/agent-stream/v1.avsc`** (`:7-13`): `header, meeting_id, user_id, window_id, seq (int),
delta (string), done (boolean)`. Doc: "Net-new (not frozen canon)… throwaway, shed first under
backpressure."

**`schemas/agent-requests/v1.avsc`** (`:7-12`): `header, requester_agent_id, responder_target,
query, deadline (timestamp-micros), correlation_id`. Key `areq:{correlation_id}`. Doc: "deny-by-
default authz is enforced by the consuming agent."

**`schemas/agent-responses/v1.avsc`** (`:7-12`): `header, correlation_id, responder_agent_id,
result, success (boolean), latency_ms (int)`.

**`schemas/transcript-chunks/v1.avsc`** (the input, `:7-16`): `header, meeting_id, chunk_id,
speaker_id, start_ms, end_ms, text, is_final, confidence, redacted (default false)`.

**The contracts are complete and frozen. Nothing in this list needs designing — only implementing
against.** Note `agent-results` carries no `card`/`priority`/`title` field: the 9-section sidebar
taxonomy the UI renders lives in `payload_json`, and that mapping is unspecified anywhere in code.

### 4.11 Transcript production — EXISTS end to end

`mytheus-realtime/services/transcription-service/src/transcription_service/persistence.py:5,31-32`:
finals → redact → one `transcript_chunks` row → one `transcript-chunks` message, `TOPIC =
"transcript-chunks"`, `SCHEMA_SUBJECT = "transcript-chunks-value"`, composite partition key
`{org_id}:{meeting_id}` (`:47`). There is also a dev seeder (`devseed.py`, 245 lines) that writes
outbox rows the relay publishes to `transcript-chunks` — **so the input to the agent tier can be
produced on demand today, without a live meeting.**

---

## 5. Backend tables: `agent_collaborations`, `agent_suggestions`, `decisions`, `action_items`, `agent_budgets`

**None of the five exists in any migration.** `grep -rn "create_table" migrations/versions/*.py`
gives the complete inventory:

`82d44163c933_0001_core_tables_outbox_inbox_.py` — `departments` (:41), `inbox` (:53),
`meetings` (:60), `organizations` (:71), `outbox` (:82), `teams` (:97), `data_residency` (:107),
`tenant_settings` (:113), `users` (:120), `meeting_participants` (:137), `team_members` (:146).
Plus `b7d9f3a55c21_0003` (:51, RBAC), `c3e8b1d47f92_0004` (audit_logs),
`25e7cede5672` (meeting_speakers, :70), `3d13f46b960f` (meeting_rooms, :51),
`5ed978467cad` (chat_messages), `f3a7c9e0b21d`, `866f7eedb0e7`, `56f72ac73fa9`, `a1c4e7f20b18`,
`9b2c4d1e6f30`.

`grep -rn "agent_collaborations|agent_suggestions|agent_budgets"` across `migrations/` → **zero**.
`worklogs`, `decisions`, `action_items` → **zero** create_table.

### The `core_api/main.py:3` claim — exactly which are false

`mytheus-backend/services/core_api/main.py:3-4` (verbatim):
> "Owns: meetings, participants, speakers, worklogs, decisions, action_items, agent_suggestions,
> agent_collaborations, chat_messages, teams, team_members, departments."

Of the twelve named: **7 exist** (`meetings`, `meeting_participants`, `meeting_speakers`,
`chat_messages`, `teams`, `team_members`, `departments`). **5 do not exist in any migration**:

1. `worklogs`
2. `decisions`
3. `action_items`
4. `agent_suggestions`
5. `agent_collaborations`

So "5 tables no migration creates" is **correct**, and those are the five.

A sixth, in a different service: `services/billing_metering_service/main.py:3` claims
"Owns: plans, subscriptions, usage_events, usage_aggregates, quotas, invoices, **agent_budgets**"
— `agent_budgets` likewise has no migration (and neither do the other six in that list). There is
no per-tenant LLM budget table, so the "cost-controlled pipeline" in the agent-runtime docstring
has no persistence to enforce against.

Related things that DO exist:
- `tenant_settings.agent_confidence_overrides` JSONB (migration `82d44163c933:116`,
  backfilled/NOT NULL'd at `b7d9f3a55c21:147-151`; model at
  `services/identity_tenant_service/models.py:184`). The per-tenant confidence-threshold knob is
  already storable.
- `audit_logs.actor_type IN ('user','agent','system')` (migration `c3e8b1d47f92:75`;
  `libs/shared/audit.py:25-29`) — the audit trail can already distinguish agent action from user
  action, deliberately, from day one.
- `services/bff/dashboard.py:97,115-123,204-215` — `action_items`, `worklogs`, `decisions` are
  modeled as `UnbuiltArm` and returned shaped-but-empty; the dashboard "never fails the screen"
  (`dashboard.py:10`). So the UI wiring for these three already degrades gracefully.

---

## 6. `POST /decisions/:id/confirm` and `POST /actions/:id/confirm`

**ABSENT — and deliberately, documented so.**

`mytheus-backend/services/core_api/main.py:6-9` (verbatim):
> "Confirmation rule: decisions/actions events are ONLY emitted by this service's transactional
> outbox, ONLY on POST /decisions/:id/confirm or /actions/:id/confirm. **Neither route exists yet,
> and neither topic is wired**, which is deliberate: there is no confirmation flow to produce them
> and a speculative producer here would put unconfirmed edges into the knowledge graph."

`core_api/main.py:12-22` mounts exactly five routers — `chat, meetings, org_hierarchy, speakers,
transcript` — followed by `# TODO: /decisions /actions /worklogs /suggestions land with their own
bands.` `ls services/core_api/routes/` confirms: `chat.py, meetings.py, org_hierarchy.py,
speakers.py, transcript.py`. Nothing else.

The gateway, however, **already routes them**: `services/api_gateway/edge.py:152-156` maps
`/api/v1/decisions`, `/api/v1/actions`, `/api/v1/suggestions`, `/api/v1/worklogs` → `core-api`.
So the edge points at four prefixes core-api does not serve.

Supporting machinery that exists for when the routes are built:
- `libs/shared/idempotency.py:30-31` already reasons about `POST /decisions/{id}/confirm` colliding
  with `POST /meetings` on a reused idempotency key.
- `libs/shared/outbox.py:14` — "the unique index (topic, idempotency_key) makes a **double-confirm
  a no-op**." The at-most-once confirm semantics are already guaranteed by the outbox schema.
- `services/workflow_service/workflows/action_followup.py:1` — "ActionFollowUpWorkflow — driven by
  confirmed actions" — a consumer waiting for a producer that doesn't exist.

**The confirmation gate — the spine of the no-silent-commits rule — is entirely unbuilt, but every
invariant around it is already written down and half-enforced by existing schema.**

---

## 7. Who consumes `agent-results`

**One subscriber, no-op.** `mytheus-backend/services/notification_service/consumer.py` (45 lines,
complete):

- `:17` — `_TOPICS = ["notifications", "agent-results"]`, `:18` `_GROUP = "notification-service"`.
- `:21-34` — a real `AIOKafkaConsumer` loop with `enable_auto_commit=False` and explicit commit.
- `:37-41` — the handler, verbatim:
```python
async def _handle(msg: object) -> None:
    # TODO(P1): re-assert org_id from msg.headers -> SET app.current_org ->
    #   is_duplicate/record_consumed (inbox + Redis dedupe) ->
    #   route json.loads(msg.value) to email / Slack / in-app based on payload.channel.
    _ = msg  # placeholder until the consumer is implemented
```

Note it plans to route on `payload.channel` — a field **`agent-results/v1.avsc` does not have**.
That handler as sketched cannot consume `agent-results` as specified; the routing key for an
agent result would have to be `kind`/`agent`, not `channel`.

`notification_service/` contains only `__init__.py`, `consumer.py`, `main.py` — **no `routes.py`**,
despite `main.py:3` claiming "REST API is the durable read/dismiss side." So there is no durable
notification read model either.

**No other consumer of `agent-results` exists in any repo.**

### What would need to consume it for output to reach a user

Two independent paths, neither built:

1. **Durable read model** — `core-api` needs an `agent_suggestions`/`agent_collaborations` table
   (§5) and a consumer writing agent results into it via the inbox pattern, so the sidebar can
   re-hydrate on reload. `agent-results/v1.avsc:5` says exactly this: "bus is transient transport,
   **read models persist in core-api**." Neither table nor consumer exists.
2. **Live push** — `realtime-gateway` needs an `agent-results`/`agent-stream` consumer that
   publishes onto the per-meeting Redis channel (§8). Absent.

---

## 8. `realtime-gateway` — no agent path at all

`mytheus-realtime/services/realtime-gateway/src/`: `server.ts` (639), `presence.ts` (139),
`presenceEventsProducer.ts` (229), `coreApiClient.ts` (165).

**Kafka direction is producer-only.** `grep -n "kafka|Kafka|Consumer" src/*.ts` — every hit is in
`presenceEventsProducer.ts` (`:50-51` imports `SchemaRegistry` + `Kafka, Partitioners`; `:132-138`
builds a producer). There is **no Kafka consumer anywhere in the gateway.** It cannot receive
`agent-results` or `agent-stream` today by any route.

**Socket events it emits** (`server.ts:55-95` header comment): `presence`, `chat`, `caption`. That
is the whole vocabulary. No `agent`, no `agent-card`, no `agent-delta`.

**The caption pattern is the template to copy.** `transcription-service` publishes onto the
explicit Redis channel `nexus:{org_id}:pubsub:meeting:{meeting_id}`; a dedicated third Redis
connection `captionSubClient` (`server.ts:207`) subscribes; `server.ts:262-292` fans out to
`localChannelSockets` keyed by the **full** channel string (which embeds `org_id`, not just
`meeting_id`) and derives `meetingId` from the *channel*, never the message body (`:273-280`), so a
body/channel mismatch can never mislabel into the wrong meeting. Subscribe/unsubscribe lifecycle at
`:300-329`, joined at `:372`, unsubscribed on disconnect at `:493`.

**An agent card path is structurally the same 60 lines** — a second channel, a second sub-client
(or a `type` discriminant on the same channel; `:272` already does `if (msg.type !== "caption")
return;`), and a `socket.emit("agent", …)`.

The gateway's own comments name the missing feature repeatedly and consistently:
- `server.ts:78-82` — "never drop captions or confirmed agent output… the thing to shed under
  backpressure is `agent-stream` token deltas, **a different, not-yet-built feature**".
- `server.ts:283-287` — same, "a separate, not-yet-built feature in this repo".

So backpressure policy for `agent-stream` is already *decided* (shed deltas first, never shed
confirmed output) and merely unimplemented.

### UI side

`mytheus-ui/src/lib/realtime/` — three files: `captions.ts`, `useLiveKitRoom.ts`,
`useMeetingSocket.ts`. `useMeetingSocket.ts` listens for exactly `connect`, `connect_error`,
`caption` (`:121`), `presence` (`:130`); emits `join` (`:101`); rehydrates finals from a REST
endpoint (`:87-90`). **No agent listener.**

`mytheus-ui/src/components/sidebar/AgentSidebar.tsx` (349 lines) is fully built visually and runs
on **mock data**. `:5-9` imports `mockAgentSidebar, AgentSidebarCard, SidebarPriority` from
`@/lib/mock/data`. Its header comment (`:11-18`) is the product spec:

> "The agent sidebar — the product's centerpiece, rendered in the frozen 9-section order (doc 20
> §7): My Notes · Suggestion · Decision · New-task toast · Context-fill · Conflict banner (red,
> ALWAYS pinned to the top, regardless of arrival order) · Workload warning · Stakeholder
> suggestion · Dependency warning · Save-for-later. **Static mock cards — no socket stream;** the
> only live behavior is the UX contract: Confirm visibly commits (and is the ONLY thing that
> commits), Dismiss removes, bookmarks collect into Save-for-later."

Card kinds it `pick()`s (`:50-58`): `note, suggestion, decision, task, context, conflict, workload,
stakeholder, dependency`. Compare to `AgentResultKind` in the Avro schema: `note,
decision_candidate, action_candidate, suggestion, alert`. **These two taxonomies do not line up** —
five wire kinds vs nine UI sections. The mapping (presumably via `payload_json` + `agent` +
`relevance_score`) is defined nowhere. Also `src/components/app/dashboard/AgentActivityStrip.tsx`
(62 lines), same mock posture.

(Per standing instruction, `mytheus-ui` was read only — no changes. Note its
`origin/local-mytheus-rebuild` branch exists and was not fetched or inspected; the above is `main`.)

---

## 9. `proto/rag.proto` — quoted in full

`mytheus-intelligence/proto/rag.proto:1-15`, complete file:

```protobuf
// Placeholder for the internal gRPC mesh of the AI/inference tier.
//
// rag-service fans out to embedding / reranker / knowledge-graph / search over this mesh; org_id
// rides in gRPC metadata on every hop (never in the message body). The real service and message
// definitions land in the RAG band; this file only reserves the package + path.

syntax = "proto3";

package mytheus.intelligence.rag.v1;

// TODO(RAG band): define RagService (Query), plus Embed/Rerank/GraphTraverse/FtsSearch messages.
// org_id travels in gRPC metadata, not in these messages.
message RagQueryPlaceholder {
  string query = 1;
}
```

It is the only file in `proto/`. No generated stubs, no `grpc_tools` invocation, no server. Note the
mitigation already in place: every service that *would* be a gRPC arm exposes a real REST contract
today and its docstring says the gRPC move is a transport change only — `query_embed.py:5-10`
("Routing this over gRPC instead of REST is a transport change only; the cache key, TTL, and
batching stay identical"), `search_service/routes.py:1-5`, `reranker_service/rerank.py`.
**The mesh being absent does not block the agent tier — REST works today.**

---

## 10. KEY DELIVERABLE — shortest path from "a transcript chunk exists" to "an agent card appears in a user's sidebar"

Every component on the path, in order:

| # | Component | Where | Status |
|---|---|---|---|
| 1 | `transcript_chunks` row + `transcript-chunks` Avro message, key `{org_id}:{meeting_id}` | `mytheus-realtime/.../transcription_service/persistence.py:5,31-32,47` | **EXISTS** |
| 1b | Dev seeder to produce chunks without a live meeting | `.../transcription_service/devseed.py` | **EXISTS** |
| 2 | Topic `transcript-chunks` + registered schema | `mytheus-events/schemas/transcript-chunks/v1.avsc`, `scripts/create_topics.sh` | **EXISTS** |
| 3 | Consumer base class (dedupe, DLQ, RLS, one-txn inbox+effect) | `libs/shared/consumer.py:11` `BaseConsumer` | **EXISTS** |
| 4 | **`AgentConsumer(BaseConsumer)` on `transcript-chunks`** | would be `services/agent_runtime_service/consumer.py` | **ABSENT** |
| 5 | **Windowing — group chunks into a `window_id`** (the Avro schema requires the field; nothing computes one) | nowhere | **ABSENT** |
| 6 | Query embedding (`POST /embed`, Redis-cached) | `services/embedding_service/query_embed.py` | **EXISTS** |
| 7 | Vector retrieval, tenant-isolated | `libs/shared/qdrant.py:155` `search_with_tenant_filter` | **EXISTS** |
| 8 | FTS retrieval arm (`GET /search`, `passage`-shaped for rerank) | `services/search_service/routes.py` | **EXISTS** |
| 9 | Rerank (`POST /rerank`, real cross-encoder) | `services/reranker_service/rerank.py` | **EXISTS** |
| 10 | **RAG orchestration (expand→retrieve→RRF→rerank→assemble→cite)** | `services/rag_service/main.py` | **STUB** (13-line healthz) |
| 11 | Graph arm (AGE traversal) | `services/knowledge_graph_service/main.py` | **STUB** (13-line healthz) |
| 12 | gRPC mesh between arms | `proto/rag.proto` | **STUB** (placeholder message) — *not blocking; REST works* |
| 13 | **`BaseAgent` lifecycle + LangGraph graph + the six agent classes** | nowhere | **ABSENT** |
| 14 | **Prompt templates + loader** | nowhere (no `*prompt*` dir in the workspace) | **ABSENT** |
| 15 | **LLM call path (litellm), tiered routing** | `litellm` 1.90.2 installed, zero imports | **ABSENT** |
| 16 | **Per-tenant budget enforcement** (`agent_budgets`) | claimed at `billing_metering_service/main.py:3`, no migration | **ABSENT** |
| 16b | Per-tenant confidence thresholds | `tenant_settings.agent_confidence_overrides` JSONB | **EXISTS** (unread by anything) |
| 17 | **Produce `agent-results` via outbox** | outbox/relay **EXIST**; agent producer **ABSENT** | **ABSENT** (infra EXISTS) |
| 17b | **Produce `agent-stream` deltas** — needs composite `{org_id}:{meeting_id}` partition key that `EventEnvelope.partition_key` does not support (`events.py:11-14,47-48`) | — | **ABSENT + small infra gap** |
| 18 | Topic `agent-results` / `agent-stream` + schemas + DLQs | `mytheus-events` `create_topics.sh:55-56`, `schemas/agent-*/v1.avsc` | **EXISTS** |
| 19 | Relay draining `intelligence.outbox` → Redpanda | `libs/shared/relay.py`, `embedding_service/relay_main.py` | **EXISTS** |
| 20 | **Durable read model: `agent_suggestions` / `agent_collaborations` tables** | no migration | **ABSENT** |
| 21 | **`core-api` consumer writing `agent-results` → read model** | nowhere | **ABSENT** |
| 22 | `agent-results` subscriber | `notification_service/consumer.py:17` subscribes; `:37-41` is a `TODO` no-op that routes on a nonexistent `payload.channel` field | **STUB** |
| 23 | **`realtime-gateway` Kafka consumer for `agent-results`/`agent-stream`** — gateway is producer-only today | `realtime-gateway/src/*` | **ABSENT** |
| 24 | Redis per-meeting pub/sub fan-out with org-scoped channel + cross-pod dedupe | `server.ts:207,262-292,300-329` (caption path) | **EXISTS** (reusable pattern, wrong payload type) |
| 25 | **Socket event `agent` emitted to the browser** | `server.ts:55-95` emits only `presence`/`chat`/`caption` | **ABSENT** |
| 26 | **UI socket listener for agent cards** | `useMeetingSocket.ts:121,130` — `caption` + `presence` only | **ABSENT** |
| 27 | **`AgentResultKind` (5 wire kinds) → 9-section sidebar taxonomy mapping** | defined nowhere | **ABSENT** |
| 28 | Sidebar rendering, 9 sections, priority strips, Confirm/Dismiss/Save UX | `AgentSidebar.tsx` (349 lines) | **EXISTS on mock data** (`:5-9,11-18`) |
| 29 | **`POST /decisions/:id/confirm` / `POST /actions/:id/confirm`** — the no-silent-commits gate | `core_api/main.py:7` "Neither route exists yet"; gateway already routes the prefixes (`edge.py:152-156`) | **ABSENT** |
| 30 | `decisions` / `action_items` tables behind the confirm gate | no migration | **ABSENT** |
| 31 | Double-confirm idempotency | `outbox.py:14` unique `(topic, idempotency_key)` | **EXISTS** |
| 32 | `knowledge-graph-service` writing confirmed edges only | `main.py` stub | **STUB** |

### The honest summary of the path

**Tally: 14 EXISTS, 4 STUB, ~17 ABSENT.**

Everything *around* the agent is built and built well — event contracts, at-least-once delivery
with tenant-safe DLQ, outbox/relay, per-tenant Qdrant isolation, two real inference models, an FTS
arm shaped to feed the reranker with zero adapter, an org-scoped realtime fan-out pattern, and a
finished sidebar UI. What is missing is the **entire middle**: nothing turns a transcript chunk
into an agent result, and nothing carries an agent result to a browser.

**The shortest walkable path is four cuts, in this order:**

1. **`AgentConsumer(BaseConsumer)` in `agent_runtime_service`** — subclass, `topics =
   ["transcript-chunks"]`, window chunks, call one agent (start with `PersonalListenerAgent`),
   `emit_outbox(topic="agent-results", …)`. The existing relay publishes it. This is the smallest
   change that puts a real `agent-results` message on the bus, and needs a LangGraph graph +
   prompt loader + litellm call underneath it (items 13-15).
2. **Read model + consumer in `core-api`** — migration for `agent_suggestions`, plus a consumer
   writing `agent-results` into it. Without this, output is transient and the sidebar cannot
   rehydrate; the schema's own doc line demands it.
3. **`realtime-gateway` Kafka consumer → existing Redis channel → new `agent` socket event**, and
   the matching `useMeetingSocket` listener + swapping `AgentSidebar` off `mockAgentSidebar`.
   Structurally a clone of the caption path.
4. **`POST /decisions/:id/confirm`** — the gate. Nothing may write the knowledge graph without it,
   and `core_api/main.py:6-9` records that as a deliberate, load-bearing constraint.

### The four things not on anyone's list that will bite

- **`window_id` has no definition.** `agent-results/v1.avsc:9` requires it; nothing in any repo
  computes or documents a windowing strategy over `transcript-chunks`.
- **Kind taxonomy mismatch.** 5 `AgentResultKind` symbols vs 9 frozen UI sections
  (`AgentSidebar.tsx:11-18`). Somebody must specify the mapping, and it will live in
  `payload_json`, which is typed `string`.
- **`agent-stream` cannot use the existing outbox path unchanged** — ADR-0017 composite partition
  key, and `EventEnvelope.partition_key` (`events.py:47-48`) returns plain `org_id`.
- **`langgraph>=0.2` in `pyproject.toml` resolved to 1.2.7** — a major-version gap. Plan against
  the locked version, not the floor.
