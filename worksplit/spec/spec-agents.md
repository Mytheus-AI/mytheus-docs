# BUILD SPEC — AI Agent Tier (`agent-runtime-service` + `rag-service`)

Source of **intent** only: `mytheus-docs/08-ai-agent-architecture.md`, `09-rag-and-inference-serving.md`,
`_canon/01-service-catalog.md`, `_canon/03-event-taxonomy.md`, ADR-0004 / 0005 / 0008 / 0013,
`plan/plan-5-6`, `plan/plan-7-9`, `plan/plan-10-13`, diagrams 03 / 05 / 07 / 08.
Nothing here is a status claim. Where I checked the repo it is to size the work, and it is called out as such.

## 0. Repo reality check (sizing input only)

| Path | State observed |
|---|---|
| `mytheus-intelligence/services/agent_runtime_service/` | `__init__.py` + `main.py` only — `create_app("agent-runtime-service")`, `/healthz`. No agents, no pipeline. |
| `mytheus-intelligence/services/rag_service/`, `knowledge_graph_service/` | same shape: `__init__.py` + `main.py` scaffold. |
| `mytheus-intelligence/libs/shared/` | has `app, auth, avro_codec, clients, consumer, db, events, inbox, outbox, relay, qdrant, splitter, tenancy, object_storage`. **No** `state.py`, `base_agent.py`, `prompts.py`, `router.py`. |
| `mytheus-intelligence/proto/` | `rag.proto` only. No `embedding/reranker/search/knowledge_graph` protos. |
| `mytheus-events/schemas/` | `agent-results`, `agent-requests`, `agent-responses`, `agent-stream`, `usage-events` **v1.avsc exist**, with the field sets quoted in §1/§3 below. |
| `mytheus-backend/services/` | `core_api`, `billing_metering_service`, `identity_tenant_service`, … directories exist. |

So the Avro contracts the agent tier emits against are already authored; the runtime that fills them is not.

---

## 1. The six agent types

All six are LangGraph graphs on the shared `BaseAgent` (§2), hosted by `agent-runtime-service`, which
**owns no Postgres tables**. Every durable effect leaves as an event; `core-api` commits.

Common output envelope — `agent-results` / `AgentResult` (`mytheus-events/schemas/agent-results/v1.avsc`):

```
header (EventEnvelopeHeader), meeting_id, window_id, user_id,
agent  : enum {PersonalListenerAgent, DecisionAgent, ActionAgent, QuestionAgent, ContextGapAgent, ProactiveAgent}
kind   : enum {note, decision_candidate, action_candidate, suggestion, alert}
relevance_score : float
confidence      : float
content         : string
payload_json    : string   ("free-form per kind")
source_chunk_ids: array<uuid>
```

Partition key `org_id`; retention 3 days; idempotency key
`agentresult:{meeting_id}:{window_id}:{user_id}:{agent}`.

> **Contract gap to resolve (see Task 30):** the frozen `AgentResult` has no typed field for the citation
> contract (`meeting_id`/`chunk_id`/`speaker` or `document_id`/`element_id`) that §5's hallucination control
> requires of ContextGapAgent fills, and no `created_by_agent` / `surfaced` flags. Intent puts those in
> `payload_json` and has `core-api` derive `surfaced` from the confidence band. That mapping is unwritten —
> it must be pinned per `kind` before two services agree on it.

### 1.1 `PersonalListenerAgent` (P7)

- **Spawn cardinality:** one **per participant per meeting** (all others are per-meeting and fan across the roster).
- **Trigger:** every 30-second `TranscriptWindow` (`A17`) built from `transcript-chunks` for meetings the user is in.
- **Input:** `AgentState` with `chunk` = the window, `context` = `get_user_context(user_id)`
  (role / skills / team / recent meetings / topics; Redis `nexus:{org_id}:user_ctx:{user_id}`, 5-min TTL).
- **Graph:** `pregate → (conditional) extract → END`.
  - `pregate`: Haiku, scores `relevance_score` 0–1 for *this* user. Gate-pass anchor 0.35 — 65% of windows are logged and dropped with **no extraction call**.
  - `extract`: Sonnet, produces note text + `confidence`.
- **Output:** `kind = note`. `content`, `relevance_score`, `confidence`, `source_chunk_ids` = the window's chunk ids.
- **Downstream:** `core-api` writes `worklogs` (`content`, `relevance_score`, `confidence`, `created_by_agent = true`,
  `source_chunk_ids`, `surfaced` from the confidence band). Surfaces in the My Notes sidebar, live.
- **Note:** `relevance_score` is the *pre-gate* (does this matter to this user); `confidence` is the *surfacing* gate (§4). Two different scores, both on `AgentState`.

### 1.2 `DecisionAgent` (P8)

- **Trigger:** the same 30s window, **after** the relevance pre-gate. Per meeting.
- **Behaviour:** Sonnet extraction of a decision: the decision text, its `rationale`, its stakeholders, and a `confidence`. Never routes Opus; never routes Haiku (under-routing degrades precision).
- **Output:** `kind = decision_candidate`; `content` = decision text; `payload_json` carries `rationale` + stakeholders (+ optional `supersedes`).
- **Commit path:** candidate only. `core-api` persists `decisions` row with `status='proposed'` and emits **nothing**. The `decisions` topic event and the AGE `PRODUCED` edge exist only after `POST /api/v1/decisions/{id}/confirm`.

### 1.3 `ActionAgent` (P8)

- **Trigger:** same window, per meeting.
- **Behaviour:** Sonnet extraction of an action item: `assignee_id`, `due_date`, dependencies, `confidence`.
- **Output:** `kind = action_candidate`; `payload_json` carries assignee/due/dependencies. Surfaces as a new-task toast.
- **Commit path:** identical rule — on confirm, `core-api` emits `actions` and `knowledge-graph-service` writes `ASSIGNED_TO`.

### 1.4 `QuestionAgent` (P9)

- **Trigger:** 30s window **plus a context-gap heuristic**. Per meeting.
- **Behaviour:** Sonnet proposal of role-specific questions the participant should ask. May ground itself with a `rag-service` lookup. Assigns `priority ∈ {1,2,3}`.
- **Output:** `kind = suggestion`; `payload_json` carries `priority`.
- **Downstream:** `core-api` writes `agent_suggestions` (`priority`, `status ∈ {open, asked, dismissed, saved}`).
  UI affordances Ask / Dismiss / Save-for-later. **Save-for-later persists into the next meeting** — the
  `(org_id, user_id, status='saved')` index seeds the next session's sidebar.

### 1.5 `ContextGapAgent` (P10)

- **Trigger:** per 30s window; a cheap, deterministic gap detector fires when the conversation references context this participant lacks.
- **Behaviour:** the A2A fallback chain of §3. Synthesis of the retrieved fill is Sonnet (`L4` ≈ 12 calls/meeting); ContextGap's *fallback* synthesis is one of the two Opus users (`L5`).
- **Output:** `kind = note` (context fill) on `agent-results`, carrying the citation for the fill; UI card reads "Context added from {X}'s agent". Every round-trip — success, fall-through, denial — is logged to `agent_collaborations` (`requester`, `responder`, `query`, `result`, `success`, `latency_ms`), owned by `core-api`.
- **Silent-on-empty:** when the chain exhausts, it emits nothing. It never fabricates a fill.
- **Budget interaction:** `budget_state == 'throttled'` ⇒ skip peers entirely, RAG-only. That is the same code path as the R6 descope.

### 1.6 `ProactiveAgent` (P11)

- **Trigger:** **confirmed** `decisions` / `actions` (not candidates) plus AGE `nexus_knowledge` queries. Gated behind `plans.features.proactive_agent`.
- **Four detectors:**
  1. **Conflict** — for a newly confirmed decision, walk `FOLLOWS_FROM*1..3` in `nexus_knowledge` (`$org_id`-bound) for contradicting priors, complemented by a semantic search over the `decisions__{org_id}` Qdrant collection. If `knowledge-graph-service` is slow/down, the Qdrant path alone is the hot-degradation fallback (reduced recall). This is the Opus user.
  2. **Workload** — sum open `action_items` per assignee over the graph + `core-api` read model; flag at **80% / 95% of capacity**.
  3. **Missing stakeholder** — compare the meeting's `ATTENDED` set against the topic's historically-relevant `Person` set, traversing `departments` / `teams`.
  4. **Dependency** — an action that `FOLLOWS_FROM` an unfinished prior action.
- **Output:** `kind = alert` on `agent-results`, and `notifications`. Conflict is priority-1 (red banner).
- **Rules:** never auto-commits a decision or action. In `throttled` budget state the agent is **suspended entirely**.

---

## 2. BaseAgent lifecycle + versioned prompt loader

### 2.1 `AgentState` (`libs/shared/state.py`)

PDF-original fields: `meeting_id`, `user_id`, `chunk`, `context`, `retrieved_facts`.
SaaS additions: `org_id` (tenant key, injected by authz — fail-closed), `agent_id` (stable identity for A2A authz + audit), `relevance_score`, `confidence`, `prompt_version`, `token_budget_remaining`, `correlation_id`.

`chunk` is a `TranscriptWindow` — a 30-second speaker-merged window (`A17`), **not** a raw `transcript_chunks` row. This is the single load-bearing unit-boundary change.

### 2.2 `BaseAgent` (`libs/shared/base_agent.py`)

Three methods, shared by all six:

- `spawn(state)` — asserts `state.org_id is not None` (fail-closed), then `state.prompt_version = loader.resolve(prompt_key, org_id)`.
- `process(state)` — returns `None` immediately if `token_budget_remaining <= 0` (hard quota, deny-by-default); otherwise `await self._graph.ainvoke(state)` on the subclass-defined LangGraph.
- `terminate(state)` — flush partial worklogs, drop subscriptions.

State machine: `spawn(meeting_id, user_id, org_id) → Spawned → Processing` (one pass per ready window, looping) `→ Terminated` on meeting end or leave.

Construction takes `(loader: PromptLoader, router: LiteLLMRouter)`; the subclass supplies `_build_graph()`.

### 2.3 `PromptLoader` (`libs/shared/prompts.py`)

- Templates are git artifacts keyed `{prompt_key}@{version}` (e.g. `decision_agent@v7`, `rag/query_expansion@v3`).
- `resolve(prompt_key, org_id)` returns the **tenant pin** if one exists, else the global default. Both reviewable in PRs.
- The tenant pin is the canary mechanism: pin 1% of tenants to `@vN+1`, and roll back by removing the pin — no redeploy.
- Rollout gate: prompt `@vN+1` merges only after the golden-set eval harness beats `@vN`; then a 48-hour ~1% canary; then the global default advances. Regression at either gate holds or rolls back.

### 2.4 Eval harness gates (CI, GitHub Actions, per prompt change)

| Agent | Metric | Gate |
|---|---|---|
| PersonalListenerAgent | relevance precision/recall vs role labels | no regression vs current default |
| DecisionAgent / ActionAgent | extraction F1 + confidence calibration (Brier) | F1 ≥ baseline, calibration not worse |
| QuestionAgent | question usefulness (Ask-rate proxy) | ≥ baseline |
| ContextGapAgent | fill correctness + citation validity | citations resolve **100%** |
| ProactiveAgent | conflict precision (false-positive rate) | FP rate ≤ baseline |

---

## 3. ContextGapAgent's A2A protocol

### 3.1 Envelopes (already authored in `mytheus-events`)

`AgentRequest` — `header, requester_agent_id, responder_target, query, deadline (timestamp-micros), correlation_id`.
`AgentResponse` — `header, correlation_id, responder_agent_id, result, success, latency_ms`.
Topics `agent-requests` / `agent-responses`: partition key `org_id`, retention 1 day, at-least-once with inbox + Redis dedupe, request key `areq:{correlation_id}`.

Note: `org_id` rides in the envelope header, not as a top-level field — and per ADR-0013 the header `org_id` is **untrusted input** used for routing/partitioning only.

### 3.2 The fallback chain (verbatim, in order)

**peer-agents → related-meeting agents → RAG → silent.**

Each hop carries a deadline; on miss the agent falls through to the next. The end-to-end constraint is
**p95 < 1.5 s** for the collaboration round-trip (Capacity §11.1). The `now + 1.2s` per-hop figure that
appears in the diagrams is explicitly **illustrative, not canon** — per-hop deadlines are tuned
operationally inside the p95 < 1.5 s envelope. If everything misses, the agent is **silent**; there is no
fabricated fill and no cross-tenant source is ever consulted.

`budget_state == 'throttled'` short-circuits straight to the RAG hop.

### 3.3 Authorization — deny-by-default, PEP/PDP split (ADR-0013)

- **PDP = `identity-tenant-service`.** Owns `agent_authz_scopes`; answers `Authorize(org_id, grantee_role, query_class, target_scope) → allow|deny`.
- **PEP = `agent-runtime-service`.** Two checks: **pre-publish** (before a requester may publish) and **pre-process** (before a responder may process).

**Identity is derived, never declared.** `org_id` and role come from the bound principal of
`requester_agent_id`, minted at `POST /agents/spawn` and held in process. If the envelope `org_id`
disagrees with the derived one, the request is denied and audited as a spoof attempt.

Three rules, fail-closed, in order:

| # | Rule | On violation |
|---|---|---|
| 1 | **Same tenant only** — derived `org_id` must equal the responder's bound `org_id`. | → `agent-requests.dlq`, **never served**, audited as a tenant-boundary violation. |
| 2 | **Role scope** — a grant must exist for `(grantee_role, query_class, target_scope)`. Deny-by-default: no grant ⇒ deny. | DENY returned to requester; `agent_collaborations.success = false`, `result = 'authz_denied'`. |
| 3 | **Audit** — every grant *and* every denial appended to `audit_logs` (`actor_type='agent'`, append-only, tamper-evident) and reflected in `agent_collaborations.success`. | n/a — this rule is the record, not a gate. |

`query_class` is derived from the query by a **deterministic, no-LLM classifier** into a fixed set
(`meeting_context` | `decision_lineage` | `action_status`); `responder_target` resolves to a
`target_scope` (`self` | `same_team` | `department` | `org`) relative to the requester. This keeps the PDP a
pure lookup.

**Grant store** (`identity-tenant-service`, RLS FORCE):

```sql
CREATE TABLE agent_authz_scopes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organizations(id),
  grantee_role TEXT NOT NULL,
  query_class  TEXT NOT NULL,
  target_scope TEXT NOT NULL,
  effect       TEXT NOT NULL DEFAULT 'allow' CHECK (effect IN ('allow','deny')),
  created_by   UUID NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at   TIMESTAMPTZ,
  UNIQUE (org_id, grantee_role, query_class, target_scope)
);
```

Semantics: `allow` only when a matching non-deleted `effect='allow'` row exists **and** no matching
`effect='deny'` row exists (explicit deny wins). No rows ⇒ A2A is dormant for that tenant — the safe
default posture for a fresh tenant.

**Decision cache:** key `nexus:{org_id}:a2a:grant:{grantee_role}:{query_class}:{target_scope}`, value
`allow`/`deny`, **TTL 60 s**, populated on first miss. Invalidated on grant create/revoke by deleting
`nexus:{org_id}:a2a:grant:*`; the 60 s TTL is the backstop. `tenant-events` is explicitly **not** the
invalidation channel (that topic is lifecycle-scoped). Accepted staleness: a revoked grant may be honoured
for ≤ 60 s — bounded, same-org only, fully audited. A cross-tenant allow is never cacheable because the key
is itself `org_id`-namespaced.

**Audit persistence:** the agent tier owns no Postgres. `agent_collaborations` is written by `core-api`,
idempotent on `collab:{correlation_id}`, and read back via `GET /api/v1/meetings/:id/collaborations`
(RLS-scoped, participants + admin).

---

## 4. Confidence thresholds and the no-silent-commits rule

### 4.1 Confidence (surfacing gate) — exact numbers

| Confidence | Action |
|---|---|
| `> 0.7` | **Surface** to the user (sidebar card / toast) |
| `0.4 – 0.7` | **Silent-log** to `worklogs` (`surfaced = false`); not rendered |
| `< 0.4` | **Drop** (zero rows) |

Defaults are `0.7 / 0.4`, overridable per tenant via `tenant_settings.agent_confidence_overrides`.
This is orthogonal to the relevance pre-gate (0.35 gate-pass anchor) and orthogonal to the cost pipeline.

### 4.2 No silent commits — exactly which writes are gated

Gated on explicit user confirmation:

- The **`decisions` topic event** and the `decisions` row transition `proposed → confirmed`.
- The **`actions` topic event** and the `action_items` row transition `proposed → confirmed`.
- The AGE edges `(:Decision)-[:PRODUCED]->(:Meeting)` and `(:Action)-[:ASSIGNED_TO]->(:Person)`, and the `FOLLOWS_FROM` lineage edge from `supersedes`.
- The `decisions__{org_id}` Qdrant embedding (embedded **on confirm**).

Not gated (agent output that is durable without confirmation): `worklogs` rows (including silent-logged
ones), `agent_suggestions` rows, `agent_collaborations` rows, `notifications`/alerts, `usage_events`.

Producer rule: `agent-runtime-service` emits **candidates only** on `agent-results`. `core-api` is the
**sole** producer of the `decisions` / `actions` topic events, via its transactional outbox, from the
confirm endpoints. The Service Catalog lists `agent-runtime-service` among those topics' producers; the
documented reconciliation is that the catalog column means "originates the candidate", not "publishes the
confirmed event". `knowledge-graph-service` writes edges only from those confirmed events. Edit rewrites
the row pre-confirm; Dismiss sets `status='dismissed'` and emits nothing.

---

## 5. The RAG pipeline (`rag-service`)

Frozen shape, no shortcut path that skips citation tracking:
**`expand → retrieve → rerank → assemble → cite`**, behind `POST /api/rag/query`.

| Stage | Detail | Warm budget |
|---|---|---|
| 0. Cache | `nexus:{org_id}:rag:{user_id}:{query_hash}`, **10-min TTL**. Per-user (expansion is role-tuned) and tenant-namespaced. | ~2 ms |
| 1. **Expand** | LiteLLM **Haiku**, template `rag/query_expansion@vN`, role-tuned via `get_user_context`. Returns original + up to 2 variants = **2–3 total**. **Skipped** when the tenant is `throttled`/`blocked` — RAG degrades to single-query recall rather than failing. | ~120 ms |
| 2. **Retrieve** | Three arms **in parallel** per variant: **Qdrant vector** (`embedding-service POST /embed` → `transcript_chunks__{org_id}` / `document_chunks__{org_id}`, with a **mandatory `org_id` payload filter even though the collection is per-tenant**); **Postgres FTS** via `search-service` over `transcript_chunks` + `chat_messages` under RLS; **AGE graph** via `knowledge-graph-service` `RELATES_TO` + `FOLLOWS_FROM`, every Cypher parameterized by `$org_id`. | ~80 ms (bounded by slowest arm) |
| 3a. **Fuse** | Reciprocal Rank Fusion, **k = 60**: `RRF(d) = Σ_lists 1/(k + rank_list(d))` across all arms and all variants → **top-50**. | ~5 ms |
| 3b. **Rerank** | `reranker-service`, **BGE-reranker-base cross-encoder**. Receives `(query, passages[50], top_k=10, org_id)` — **the user's actual query, not the variants**. Returns the **top-10** scored `(passage, rerank_score)`. | ~120 ms |
| 4. **Assemble** | Token-budget packing: hard cap **3,000 tokens** of passages by default, configurable per calling agent/surface, **tightened automatically as the tenant nears its `agent_budgets` hard quota**. Plus dedupe of near-duplicates and per-source diversity capping. | ~10 ms |
| 5. **Cite** | Transcript passage → `meeting_id`, `chunk_id`, `speaker` (+`start_ms`). Document passage → `document_id`, `element_id` (+`element_type`). **A passage that cannot be bound to a citation is dropped before return — uncited text never reaches the answer.** | ~5 ms |

Then: emit `usage-events` (`agent_token`, Haiku tier, for the expansion tokens) and write the answer cache.

SLOs: **warm p95 < 500 ms**, **cold p95 < 1.5 s**. The cross-encoder is the cold long pole, which is why
`reranker-service` holds a **warm replica floor ≥ 1 per active region** and never scales to zero during
business hours. `rag-service` holds **no model in-process** — it is an orchestrator over the internal gRPC
mesh, per ADR-0008 (inference off the API tier; `embedding-service` and `reranker-service` are separate
GPU/Ray Serve deployments with their own KEDA autoscalers and cost center).

Embedding contract: **embed-on-ingest, query-embed only** — bulk embedding happens once when a chunk lands
on the bus (batch 32, sub-100 ms/batch); retrieval embeds only the short query, cached at
`nexus:{org_id}:qembed:{query_hash}` (~10 min). An answer-cache miss can still be a query-embed hit.

Consumers: the context-panel UI; `ContextGapAgent` (third rung of the fallback chain);
`ProactiveAgent` (via the `decisions__{org_id}` collection for semantic conflict detection).

---

## 6. Cost control per ADR-0004

Five stages, **executed in this order** — the ADR is explicit that the lock-string list order
("relevance pre-gating → windowed batching → …") is a feature list, not a sequence, and must not be used to
"correct" the execution order back to gate-first:

1. **Windowed batching (`A17` = 30 s).** Deepgram finals merged by speaker turn into 30-second windows.
   `T2` = 360 raw `transcript_chunks`/meeting → **90 windows/meeting**. A 4× reduction at the unit boundary before any other lever.
2. **Relevance pre-gating (Haiku triage).** One cheap Haiku call **per window per participant**, scored against `get_user_context`. **~35% pass** (gate-pass 0.35). The failing 65% are logged and dropped; no extraction call is ever made for them.
3. **Tiered routing (LiteLLM).** Never hardcode a model; route through the tier helper.
   `TIER = {"triage": haiku, "extract": sonnet, "high_stakes": opus}`.
4. **Semantic + Anthropic prompt caching.** Anthropic prefix-cache the stable prefixes (system prompt, role→topics taxonomy, versioned template, prior-window context) via `cache_control`; a semantic cache catches near-duplicate windows **before** any model call. Applied *after* tiering. Reversible: disabling falls back to uncached pricing with identical output.
5. **Per-tenant token budgets with hard quotas.** Checked inline **before every dispatch**, as one Redis read against the hot mirror `nexus:{org_id}:budget:{period}` — never a Postgres round-trip on the hot path (read-through to Postgres on mirror miss, fail-closed).

Per-meeting tier breakdown (frozen — cite the symbol, never re-derive):

| Symbol | Stage (tier) | Calls/meeting | Derivation |
|---|---|---:|---|
| `L1` | Triage / pre-gate (**Haiku**) | 540 | windows(90) × roster(6) |
| `L2` | Extraction — Listener/Decision/Action (**Sonnet**) | 567 | 90 × 6 × 0.35 × 3 |
| `L3` | QuestionAgent (**Sonnet**) | 81 | 90 × 6 × 0.15 |
| `L4` | ContextGap synthesis (**Sonnet**) | 12 | collaboration round-trips |
| `L5` | High-stakes (**Opus**) | 6 | ProactiveAgent conflict / ContextGap fallback |
| `L6` | **Total** | **~1,206** | |

Tier mix by volume: **Haiku 45% / Sonnet 55% / Opus < 1%**. Controlled `L7` ≈ 5.789B calls/month,
`L9` ≈ 8,930 calls/sec peak, `L10` = **86%** reduction vs the naive per-chunk `N2` = 41.472B — before any
caching savings.

`agent_budgets` state machine — the enum has **exactly three members** (`ok`, `throttled`, `blocked`); the
80%/95% crossings are **not states**, they are self-transitions on `ok` that emit `billing-events` and set
`alert_80_sent` / `alert_95_sent`:

| Consumption | `state` | Action |
|---|---|---|
| < 80% of `token_quota` | `ok` | normal |
| ≥ 80% | `ok` + `alert_80_sent` | warn tenant admin |
| ≥ 95% | `ok` + `alert_95_sent` | warn again, suggest upgrade |
| ≥ 100% of `token_quota` | `throttled` | Haiku-only; extraction degrades to silent-log; `ProactiveAgent` suspended; RAG expansion skipped |
| ≥ `hard_cap` | `blocked` | agent processing stops — deny-by-default; `notifications` "budget exhausted" card. **Captions/transcription/CRUD continue** — durable facts are never shed. |

Per-plan shape: **Free** has `token_quota = hard_cap`, so there is **no throttle band** — `ok → blocked`
directly. **Pro / Enterprise** have `token_quota < hard_cap`, giving the soft Haiku-only band. Period
rollover (`YYYY-MM`) returns any state to `ok`.

Metering: every metered call emits exactly one `usage-events` record, `event_type = agent_token`,
`model_tier` set, idempotency key `usage:{org_id}:agent_token:{occurred_at}:{call_id}`, through the
transactional outbox. This is the runtime's only durable side effect. **`billing-metering-service` must be
live (alert-only at minimum) before the first agent LLM call** — the plan is explicit that this is a hard
precondition, not a follow-on, because LLM is `C3` = 94% of variable cost.

---

# TASK LIST

Sizes: **S** ≤ 1 day · **M** 2–4 days · **L** ≥ 5 days (days given). Dependencies are task numbers.
"Blocked" flags an external decision or absent infrastructure.

## A. Shared runtime foundation (`mytheus-intelligence`)

**1. `AgentState` dataclass** — S
`libs/shared/state.py` (+ `TranscriptWindow` type). Five PDF fields + `org_id`, `agent_id`, `relevance_score`, `confidence`, `prompt_version`, `token_budget_remaining`, `correlation_id`.
Deps: none. Blocked: no.

**2. `PromptLoader` + the versioned prompt template store** — M
`libs/shared/prompts.py` + `prompts/{prompt_key}/vN.md` in git. `resolve(prompt_key, org_id)` with tenant-pin-over-global-default. Pin storage (config vs table) is an open call — pick one and record it.
Deps: 1. Blocked: no.

**3. `LiteLLMRouter` with tier discipline** — M
`libs/shared/router.py`. `TIER = {triage: haiku, extract: sonnet, high_stakes: opus}`; `complete(tier, template, vars, org_id, max_tokens)`; per-user rate-limit counters `nexus:{org_id}:ratelimit:{user_id}`; emits usage on every call. Includes a lint/test that forbids a hardcoded model string anywhere in the agents.
Deps: 1, 2. **Blocked on external decision:** which provider/model IDs and which account/spend ceiling back `claude-haiku` / `claude-sonnet` / `claude-opus` (`ANTHROPIC_API_KEY`, `LITELLM_CONFIG`).

**4. `BaseAgent` lifecycle** — M
`libs/shared/base_agent.py`. `spawn` (fail-closed on `org_id`, resolves `prompt_version`), `process` (returns `None` on exhausted budget, else `graph.ainvoke`), `terminate` (flush partials, drop subscriptions). Subclass hook `_build_graph()`.
Deps: 1, 2, 3. Blocked: no.

**5. Windowing batcher (`A17` = 30 s)** — M
`agent_runtime_service/windowing.py`. Consume `transcript-chunks` via the existing `libs/shared/consumer.py` + inbox dedupe; merge finals by speaker turn into 30 s `TranscriptWindow`s; stable `window_id`. Test: 360 chunks → ~90 windows.
Deps: 1. Blocked: no.

**6. Context Resolver (`get_user_context` client)** — S
`agent_runtime_service/context.py`. gRPC to `identity-tenant-service`, Redis-cached `nexus:{org_id}:user_ctx:{user_id}` 5-min TTL, populates `AgentState.context`.
Deps: 1. **Blocked on infrastructure:** `get_user_context` must be callable over gRPC from `identity-tenant-service` (a Plan 5-6 precondition), and `users.role` / `users.skills` must be populated.

**7. Relevance pre-gate** — S
`pipeline/pregate.py`. One Haiku call per window per participant → `relevance_score`; drop-and-log below the per-tenant threshold. Metric on the realised pass rate against the 0.35 anchor.
Deps: 3, 5, 6. Blocked: no.

**8. Prompt cache layer** — M
`pipeline/cache.py`. Anthropic `cache_control` on stable prefixes + a semantic near-duplicate cache ahead of the model call. Applied after tiering; a flag disables it with identical output.
Deps: 3. Blocked: no.

**9. Budget guard (`check_budget`)** — M
`pipeline/budget.py`. One Redis read of `nexus:{org_id}:budget:{period}`; read-through to Postgres on miss, fail-closed. Returns `ok|throttle|block`. `throttle` forces `tier="triage"` and degrades extraction to silent-log; `block` denies and emits the `notifications` card.
Deps: 1. **Blocked on infrastructure:** `agent_budgets` + the Redis mirror are owned by `billing-metering-service` (Task 26). **Blocked on external decision:** DECISION-REQUIRED #1 — per-seat vs per-tenant `included_agent_tokens` — you cannot set a quota you cannot compute.

**10. `usage-events` emission through the outbox** — S
Every metered call emits exactly one `UsageEvent` (`agent_token`, `model_tier`, `source_id = call_id`), key `usage:{org_id}:agent_token:{occurred_at}:{call_id}`, via `libs/shared/outbox.py`. Retry must not double-count.
Deps: 3. Blocked: no (schema `usage-events/v1.avsc` exists).

**11. Outbox producer for `agent-results`** — S
Avro-serialized `AgentResult` with the idempotency key `agentresult:{meeting_id}:{window_id}:{user_id}:{agent}`; confidence-band gating applied at emit (§4.1).
Deps: 1, 4. Blocked: on Task 30 (the `payload_json` field map).

**12. Stream bridge → `agent-stream`** — M
Bridge LiteLLM streaming deltas to `AgentStreamDelta` (`window_id`, `seq`, `delta`, `done`), composite partition key `{org_id}:{meeting_id}`, plus the per-user priority queue `nexus:{org_id}:queue:agent:{user_id}` so alerts pre-empt notes. Shed first under backpressure.
Deps: 3, 11. **Blocked on infrastructure:** `realtime-gateway` fan-out for `agent-stream`.

**13. PII redaction before LLM egress** — M
Redact the window before it leaves for Anthropic/OpenAI via LiteLLM, gated on `tenant_settings.pii_redaction_enabled`. Plan 3-4 explicitly deferred this node to the agent band; it is always-on in intent.
Deps: 3. **Blocked on infrastructure:** the PII token vault / NER placement of ADR-0020, and readable `tenant_settings`.

**14. Agent spawner + `POST /agents/spawn` · `/agents/{id}/terminate`** — M
Consume `meeting-events` / `meeting-room-events`; spawn one `PersonalListenerAgent` **per participant per meeting** and the per-meeting agents once each; mint the `agent_id` binding (`org_id` + RBAC role) that Task 22's PEP derives identity from; inject `org_id` and the budget mirror into `AgentState`.
Deps: 4, 6, 9. Blocked: no.

## B. The six agents

**15. `PersonalListenerAgent`** — M
`agents/personal_listener_agent.py`. Graph `pregate → extract`, conditional edge on the gate. Emits `kind=note`.
Deps: 4, 7, 11. Blocked: no.

**16. `DecisionAgent`** — M
Sonnet extraction of decision + `rationale` + stakeholders + `confidence`; `kind=decision_candidate`; per-meeting registration; Opus never selected.
Deps: 4, 7, 11. Blocked: no.

**17. `ActionAgent`** — M
Sonnet extraction of `assignee_id`, `due_date`, dependencies, `confidence`; `kind=action_candidate`.
Deps: 4, 7, 11. Blocked: no.

**18. `QuestionAgent`** — M
Context-gap heuristic + Sonnet question proposal, `priority` 1–3, optional `rag-service` grounding; `kind=suggestion`.
Deps: 4, 7, 11, 24. Blocked: no.

**19. `ContextGapAgent` + the fallback chain** — L (5–7 days)
`agents/context_gap_agent.py`. Deterministic gap detector; hops peer → related-meeting → RAG → silent with per-hop deadlines inside the p95 < 1.5 s envelope; `throttled` ⇒ RAG-only; emits the context-fill candidate on `agent-results` with its citation; meters synthesis tokens.
Deps: 4, 11, 21, 22, 24. **Blocked:** see Task 22 (the PDP must exist first). Also the R6 descope target — see §Bands.

**20. `ProactiveAgent` (four detectors)** — L (6–8 days)
`agents/proactive_agent.py`. Conflict (AGE `FOLLOWS_FROM*1..3` + `decisions__{org_id}` Qdrant complement, with the Qdrant-only hot-degradation path), workload (80%/95% of capacity), missing-stakeholder, dependency. Opus for high-stakes synthesis; suspended entirely in `throttled`; gated on `plans.features.proactive_agent`. Emits `kind=alert` + `notifications`; never auto-commits.
Deps: 4, 11, 27, 28. **Blocked on infrastructure:** `knowledge-graph-service` writing AGE edges (Task 28) and `decisions__{org_id}` being embedded on confirm; **blocked on external decision:** DECISION-REQUIRED #3 / R1 — whether AGE can be enabled on the managed Postgres at all.

## C. A2A protocol

**21. Collaboration client (`agent-requests` / `agent-responses`)** — M
Correlation-ID request/response with in-flight tracking and deadline expiry; publish/consume through outbox/inbox with `idempotency_key = areq:{correlation_id}`; foreign `org_id` → `agent-requests.dlq`.
Deps: 4, 11. Blocked: no (both schemas exist).

**22. A2A PEP (pre-publish + pre-process) + decision cache** — M
Derive `org_id` + role from the bound principal of `requester_agent_id`, never from the payload; envelope-vs-derived mismatch → deny + audit as spoof. Deterministic no-LLM `classify_query` → `query_class`; `resolve_scope` → `target_scope`. Redis decision cache `nexus:{org_id}:a2a:grant:{role}:{class}:{scope}`, TTL 60 s.
Deps: 14, 21, 23. Blocked: no once 23 lands.

**23. A2A PDP + `agent_authz_scopes`** — M — *repo: `mytheus-backend/services/identity_tenant_service/`*
Migration (RLS FORCE + `tenant_isolation`), `Authorize(...)` with explicit-deny-wins and absence-is-deny, cache invalidation on grant create/revoke (`nexus:{org_id}:a2a:grant:*`, **not** via `tenant-events`), audit of grant/revoke to `audit_logs`, and a platform-default grant template an admin can adopt or tighten. A fresh tenant has no grants ⇒ A2A dormant.
Deps: none within this tier. **Blocked on canon:** ADR-0013 notes no canon table name is frozen; `_canon/02-data-model.md` should record the table first.

## D. RAG

**24. `rag-service` — the five-stage pipeline** — L (7–10 days)
`services/rag_service/`. `POST /api/rag/query`; answer cache `nexus:{org_id}:rag:{user_id}:{query_hash}` 10-min TTL; expand (Haiku, 2–3 variants, skipped when throttled/blocked); three arms in parallel; RRF k=60 → top-50; rerank → top-10 against the user's actual query; assemble under a 3,000-token cap with dedupe + source diversity, tightening near the hard quota; cite with uncited-dropped; emit `usage-events`.
Deps: 3, 6, 9, 25. **Blocked on infrastructure:** `embedding-service`, `reranker-service`, `search-service`, `knowledge-graph-service` must all be serving (Plan 5-6 / Task 28); the graph arm additionally depends on the R1 outcome.

**25. gRPC protos + generated stubs** — S
`proto/embedding.proto`, `reranker.proto`, `search.proto`, `knowledge_graph.proto` (only `rag.proto` exists). `org_id` in gRPC metadata on every hop; codegen wired into the Makefile.
Deps: none. Blocked: no.

## E. Backend write paths (`mytheus-backend`)

**26. `billing-metering-service` — `agent_budgets`, mirror, state machine** — L (5–7 days)
Migrations for `plans`/`subscriptions`/`invoices`/`usage_events`/`usage_aggregates`/`agent_budgets`/`quotas` with RLS FORCE (`plans` excepted); the `usage-events` consumer with inbox dedupe; the Redis budget mirror; the three-state machine with the 80/95 self-transitions; an alert-only → enforce flag.
Deps: none within this tier — **must precede Task 9's enforcement and the first LLM call.**
**Blocked on external decision:** DECISION-REQUIRED #1 (per-seat vs per-tenant) and #2 (derived-dimension producer ownership).

**27. `core-api` — agent read models and confirm endpoints** — L (5–7 days)
Consume `agent-results` (dedupe on `idempotency_key`): write `worklogs` with `surfaced` set from the confidence band; persist decision/action candidates as `status='proposed'` with **no topic event**; write `agent_suggestions`. Confirm endpoints `POST /api/v1/{decisions,actions}/{id}/confirm` as the **sole** producers of the `decisions`/`actions` events via the transactional outbox; edit-pre-confirm; dismiss emits nothing. Read APIs for My Notes / My Tasks. Plus `agent_collaborations` persistence (idempotent on `collab:{correlation_id}`) and `GET /api/v1/meetings/:id/collaborations`.
Deps: 11. Blocked: no.

**28. `knowledge-graph-service` — edges on confirmation only** — L (5–7 days) — *repo: `mytheus-intelligence`*
AGE `nexus_knowledge`; per-label `org_id` index; consume confirmed `decisions`/`actions` → `PRODUCED`, `ASSIGNED_TO`, `RELATES_TO`, `FOLLOWS_FROM`; `$org_id` bound on **every** Cypher, ad-hoc `/graph/cypher` without it rejected; four named bounded-depth templates behind `POST /graph/query/{template}`; soft delete; `graph-events`.
Deps: 27. **Blocked on external decision:** DECISION-REQUIRED #3 / R1 go/no-go at the P8 entry gate — AGE on managed Postgres vs self-hosted vs Neo4j Aura/Neptune vs the relational materialized-view fallback.

## F. Quality, contracts, observability

**29. Eval harness + golden set** — L (5 days)
CI job over labelled transcripts, one metric per agent per §2.4, gating every prompt change; wired to the canary/promote/rollback flow (48 h, ~1% of tenants, via tenant pins).
Deps: 2, 15–20 (incrementally, per agent as it lands). **Blocked on infrastructure:** labelled meeting transcripts must exist — none were located in the repos.

**30. Pin the `AgentResult.payload_json` field map per `kind`** — S
Write the per-`kind` JSON contract (`rationale`/stakeholders/`supersedes`; `assignee_id`/`due_date`/dependencies; `priority`; alert class; **and the citation object for context fills**) plus how `core-api` derives `surfaced` and `created_by_agent`. Decide whether the citation deserves promotion to a typed Avro field (BACKWARD-compatible append with a default).
Deps: none. **Blocked on canon owners** if it becomes a schema change to a frozen name.

**31. Confidence-gate + tenant-override plumbing** — S
Read `tenant_settings.agent_confidence_overrides`; apply `0.7 / 0.4` defaults when absent; enforce uniformly at emit for all six agents.
Deps: 11. **Blocked on infrastructure:** `tenant_settings` readable (a Plan 7-9 entry precondition).

**32. Agent-tier observability + SLO dashboards** — M
Tenant-tagged Prometheus metrics: realised gate-pass rate, tier mix, cache hit rates, per-stage RAG latency, RRF candidate count, A2A round-trip latency and denial rate, DLQ rate, budget-state distribution. Panels for agent p50 < 2 s / p95 < 4 s, A2A p95 < 1.5 s, RAG warm p95 < 500 ms, quota-breach enforcement 100%.
Deps: 7, 9, 21, 24. Blocked: no.

**33. Cross-tenant isolation test suite** — M
Forged-envelope-`org_id` A2A probe → DLQ + audit, never served; RLS probes on every agent-written table; Qdrant `org_id` payload filter asserted even on a per-tenant collection; every Cypher template proven to bind `$org_id`; no cross-tenant passage returned by RAG.
Deps: 22, 24, 27, 28. Blocked: no.

**34. Zero-Postgres invariant test for `agent-runtime-service`** — S
Static grep + integration assertion that the runtime opens no Postgres connection and writes no row; all durable output is events.
Deps: 11. Blocked: no.

---

## Effort roll-up

34 tasks: **9 S**, **16 M**, **9 L**.
L tasks sum to ≈ 50–62 days; M ≈ 32–64 days; S ≈ 9 days. Serial engineer-days ≈ **90–135**, i.e. roughly
**9 calendar weeks for a 3-person team** — which lines up with the intended P7–P11 span (W18–W31, 14 weeks)
once UI, mobile, and the non-agent work in those bands are added on top.

---

## Roadmap bands — what is MVP and what is deliberately later

The 36-week plan (W1–W36) **is** the MVP band; there is no smaller in-scope band inside it. Within it:

**MVP core — P7–P9 (W18–W26):**
- Tasks 1–18, 24–27, 29–34 — the shared runtime, the cost pipeline, `PersonalListenerAgent`, `DecisionAgent`, `ActionAgent`, `QuestionAgent`, the full RAG pipeline, `billing-metering-service`, the `core-api` confirm path, `knowledge-graph-service` (Task 28).
- Billing/metering (26) is a **hard precondition of P7**, before the first agent LLM call — not a follow-on.

**MVP, later band — P10–P11 (W27–W31):**
- Tasks 19, 21, 22, 23 — `ContextGapAgent` and the whole A2A protocol land in P10.
- Task 20 — `ProactiveAgent` lands in P11.
- These are in the 36 weeks. They are **not** required for the P7–P9 demos, and building the runtime so it *hosts* all six while shipping only four is the documented P7 posture.

**Pre-authorized descopes (do not report these as required work):**
- **R6, at P10.** If A2A collaboration is unstable or misses the p95 < 1.5 s round-trip budget, `ContextGapAgent` descopes to **RAG-only context fill** and peer-agent collaboration ships as v2. That collapses Tasks 19/21/22/23 to "call `rag-service` on gap detected" — the fallback chain simply stops at the RAG rung, which is the same code path the `throttled` budget state already takes. The cost is losing cross-meeting agent context, not losing context fill.
- **R1, at P8.** If AGE cannot be enabled, Task 28 swaps its driver behind the unchanged `POST /graph/query/{template}` API (self-hosted Postgres+AGE, Neo4j Aura/Neptune, or a relational materialized-view fallback). Task 20's conflict detector degrades to the `decisions__{org_id}` Qdrant path.

**Explicitly out of the 36 weeks (scale-out, post-W36):** cells, multi-region, BYOC Redpanda, full GPU
autoscaling. The MVP substrate is a modular monolith on a single-node VPS under Docker Compose (ADR-0016) —
which is worth naming, because ADR-0008's "dedicated GPU inference tier on Ray Serve/vLLM with its own KEDA
autoscaler" is the *target* topology, not the MVP substrate. Tasks 24 and 32 should be sized against
whatever the VPS actually runs, and the C4/L9 capacity anchors are the scale-out sizing target, never the
MVP's.

## Open decisions that block work (consolidated)

| # | Decision | Blocks | Owner named in the docs |
|---|---|---|---|
| DR-1 | `included_agent_tokens` per-seat vs per-tenant | Tasks 9, 26 — the `agent_budgets` quota math, therefore metering go-live | Pricing/GTM lead + canon owners (needs an ADR) |
| DR-2 | Producer ownership of the three derived usage dimensions | Task 26 aggregation for `meeting_minute` / `transcription_second` / `storage_gb` | Billing lead + canon owners |
| DR-3 | R1 — AGE on managed Postgres go/no-go | Tasks 20, 28, and RAG's graph arm in 24 | Data/Platform lead + canon owners, at the P8 entry gate |
| — | LLM provider account, model IDs, spend ceiling | Task 3 and every agent that routes through it | not assigned in the docs |
| — | `AgentResult.payload_json` per-`kind` map + citation typing | Tasks 11, 19, 27 | Task 30 |
