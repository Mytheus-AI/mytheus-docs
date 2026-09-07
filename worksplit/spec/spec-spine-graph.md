# Spec: the confirmation spine, and the knowledge graph given AGE does not exist

Read-only investigation. Root: `/Users/sushan/Projects/Mytheus`. Nothing was modified.

---

# PART A — the confirmation spine (no silent commits)

## A0. What the rule actually is, stated from source

The rule is not a convention. It is written into three independent places, and they agree:

- `mytheus-backend/services/core_api/main.py:6-9` — "Confirmation rule: decisions/actions events are ONLY emitted by this service's transactional outbox, ONLY on POST /decisions/:id/confirm or /actions/:id/confirm. Neither route exists yet, and neither topic is wired, which is deliberate: there is no confirmation flow to produce them and a speculative producer here would put unconfirmed edges into the knowledge graph."
- `mytheus-docs/_canon/03-event-taxonomy.md:14-15` — producer of `decisions` and `actions` is "`core-api` (outbox, on confirm)"; idempotency keys are `decision:{decision_id}:confirmed` and `action:{action_id}:confirmed`.
- The Avro schemas enforce it in the wire type. `mytheus-events/schemas/decisions/v1.avsc` declares `status` as `{"type":"enum","name":"DecisionStatus","symbols":["confirmed"]}` with the doc string "status enum is intentionally {confirmed} so a non-confirmed value is unrepresentable (no-silent-commits)". Same for `actions/v1.avsc` (`ActionStatus`, symbols `["confirmed"]`).

So an unconfirmed decision is **not expressible** on the bus. That is the strongest form of this rule available and it is already in place. What is missing is everything on either side of it.

One apparent contradiction is already reconciled in the docs and should not be re-litigated: `mytheus-docs/_canon/01-service-catalog.md:116` lists `agent-runtime-service` among producers of `decisions`/`actions`. `mytheus-docs/08-ai-agent-architecture.md:67` resolves this explicitly — the catalog's producer column means "originates the candidate", not "publishes the confirmed event"; `core-api` is the authoritative producer on confirm. No spec change needed.

The canonical flow (`mytheus-docs/08-ai-agent-architecture.md:55-65`):

```
agent-runtime-service (DecisionAgent/ActionAgent)
  → agent-results (kind = decision_candidate | action_candidate)
  → core-api consumer writes the CANDIDATE row (status='candidate')
  → realtime-gateway → web-app sidebar card
  → user confirms → POST /decisions/:id/confirm
  → core-api: UPDATE row to confirmed + outbox row, ONE transaction
  → decisions/actions topic
  → knowledge-graph-service creates PRODUCED / ASSIGNED_TO edge
```

`mytheus-events/schemas/agent-results/v1.avsc` already carries the candidate: `kind` is an enum including `decision_candidate`, `action_candidate`, `suggestion`; plus `confidence`, `content`, `payload_json`, `source_chunk_ids`. The candidate ingress contract exists.

## A1. Data model — confirmed absent, then specified

### Confirmation of absence

Migrations present (`mytheus-backend/migrations/versions/`):

| Revision | Content |
|---|---|
| `82d44163c933` | 0001 core tables: outbox, inbox, organizations, users + P2 stubs |
| `a1c4e7f20b18` | 0002 least-privilege roles, tenant root |
| `b7d9f3a55c21` | 0003 RBAC grants and canonical columns |
| `c3e8b1d47f92` | 0004 audit_logs append-only |
| `9b2c4d1e6f30` | 0005 retire schema-wide default grants |
| `866f7eedb0e7` | consent columns on meeting_rooms |
| `3d13f46b960f` | meeting_rooms |
| `f3a7c9e0b21d` | meeting_rooms livekit metadata sync |
| `5ed978467cad` | chat_messages |
| `25e7cede5672` | meeting_speakers |
| `56f72ac73fa9` | org hierarchy indexes |

`grep -rn "decisions\|action_items\|agent_suggestions" mytheus-backend/migrations/versions/*.py` returns **zero hits**. None of the three tables exists in any migration. `worklogs` and `agent_collaborations` are likewise absent. This matches the roadmap, which puts `decisions`/`action_items` at P8, `worklogs` at P7, `agent_suggestions` at P9 (`mytheus-docs/22-roadmap-and-phasing.md:108-110`).

### Migration conventions this repo actually uses

Drawn from `25e7cede5672_feat_core_api_add_meeting_speakers.py` (the closest precedent — a non-partitioned, tenant-scoped domain table) and `5ed978467cad_feat_core_api_add_chat_messages.py`:

1. `id uuid` bare primary key with `server_default=gen_random_uuid()`; `org_id` is a plain indexed column, **not** part of a composite PK. Stated as house style at `25e7cede5672:32-34` ("matching `meeting_rooms`'s house style of a bare `id` primary key with `org_id` as a plain indexed column, not a composite PK").
2. `org_id uuid NOT NULL REFERENCES organizations(id) ON DELETE RESTRICT` — the `RESTRICT` is defect D6, cited at `mytheus-backend/libs/shared/infra_models.py:31-33`.
3. RLS, three statements, using the exact predicate constant:
   `_RLS_PREDICATE = "org_id = NULLIF(current_setting('app.current_org', true), '')::uuid"`
   then `ENABLE ROW LEVEL SECURITY`, `FORCE ROW LEVEL SECURITY`, and `CREATE POLICY tenant_isolation ... USING (...) WITH CHECK (...)`. See `25e7cede5672:59-68`.
4. `created_at`/`updated_at` as `TIMESTAMPTZ NOT NULL DEFAULT now()`; soft delete via `deleted_at timestamptz` where canon says soft delete.
5. Value domains as `CHECK` constraints, not Postgres enums (`ck_meeting_speakers_assignment_valid`, `25e7cede5672:118-120`).
6. Named constraints via `op.f(...)`.
7. Grants are **explicit**, never schema-wide — migration `9b2c4d1e6f30` retired schema-wide default grants, and `a1c4e7f20b18` establishes the four roles (`mytheus_app`, `mytheus_relay`, `mytheus_admin`, plus tenant root). A new table needs its own explicit `GRANT` to `mytheus_app`.
8. Where an invariant must not depend on future code being careful, it is enforced by a database trigger (the `meeting_speakers` manual-authoritative trigger, `25e7cede5672:16-27`). **This precedent is directly applicable here** — see the confirmation-monotonicity trigger below.

### Specified tables

Canon field lists are `mytheus-docs/_canon/02-data-model.md:84-86`. Everything below is those fields plus the platform-wide columns the conventions require.

#### `decisions` — size **M**

Canon (`02-data-model.md:84`): `id`, `org_id`, `meeting_id`, `content`, `rationale`, `confidence`, `status`, `confirmed_by`. "RLS by `org_id`. On confirmation → AGE `Decision`→`Meeting` `PRODUCED` edge. Soft delete."

```
id            uuid PK        DEFAULT gen_random_uuid()
org_id        uuid NOT NULL  REFERENCES organizations(id) ON DELETE RESTRICT
meeting_id    uuid NOT NULL  REFERENCES meetings(id)
content       text NOT NULL
rationale     text NOT NULL DEFAULT ''      -- Avro field is non-null string
confidence    real NOT NULL                 -- Avro "float"
status        text NOT NULL DEFAULT 'candidate'
                CHECK (status IN ('candidate','confirmed','dismissed','superseded'))
confirmed_by  uuid NULL REFERENCES users(id)
confirmed_at  timestamptz NULL
supersedes_id uuid NULL REFERENCES decisions(id)   -- source of the FOLLOWS_FROM edge
source_agent_result_id uuid NULL                    -- provenance of the candidate
source_chunk_ids uuid[] NOT NULL DEFAULT '{}'       -- citation trail, mirrors worklogs
created_at    timestamptz NOT NULL DEFAULT now()
updated_at    timestamptz NOT NULL DEFAULT now()
deleted_at    timestamptz NULL
```

Two columns are net-new relative to canon's field list and need calling out rather than sneaking in:

- **`supersedes_id`.** The `FOLLOWS_FROM` edge (`05-data-architecture.md:499`, "Decision lineage / supersession") has no relational source anywhere in canon. ADR-0005's own rollback story depends on the graph being rebuildable from relational truth ("edges are re-derivable from confirmed `decisions`/`action_items` rows … the graph is a projection, never the only copy of the data"). Without a lineage column on the row, `FOLLOWS_FROM` is *not* re-derivable and that safety net is false. This column is what makes ADR-0005's rebuild promise true. It is also the entire basis of contradiction detection (Part B).
- **`confirmed_at`.** `confirmed_by` alone cannot distinguish "confirmed" from "edited by an admin"; the outbox emission and the audit row both want the moment.

`status` domain: canon does not enumerate it. `'candidate'` and `'confirmed'` are forced by the flow; `'dismissed'` is forced by the UI card contract ("confirm/edit/dismiss card", `08-ai-agent-architecture.md:45`); `'superseded'` pairs with `supersedes_id`. Flag as a decision for the team lead — this is the one place I am naming values canon leaves open.

Indexes:
- `(org_id, meeting_id, created_at DESC)` — the meeting-detail decisions panel and the BFF dashboard arm.
- `(org_id, status) WHERE deleted_at IS NULL` — the pending-candidates queue, which is the whole product surface.
- `(org_id, confirmed_by, confirmed_at DESC) WHERE status='confirmed'` — the "decisions by a person" Cypher template's relational twin (`05-data-architecture.md:526`).
- `(supersedes_id) WHERE supersedes_id IS NOT NULL` — lineage walk.

RLS: `tenant_isolation`, exactly as `meeting_speakers`.

**Confirmation-monotonicity trigger** — the `meeting_speakers` precedent applied to the integrity rule that actually matters here. A `BEFORE UPDATE` trigger raising when `OLD.status='confirmed' AND NEW.status='candidate'`, and when `OLD.status='confirmed' AND NEW.confirmed_by IS DISTINCT FROM NEW.confirmed_by`'s prior value. Rationale, in the same voice as `25e7cede5672`: a confirmed decision has already produced a graph edge and a bus event; silently walking it back to candidate would leave the graph asserting a fact the database no longer holds, and no future code path — including one written by someone who has never read this ADR — should be able to do that. Un-confirming is a *soft delete plus a new candidate*, not an UPDATE. **This trigger is the structural expression of "no silent commits" and is the single highest-value line in Part A.**

#### `action_items` — size **M**

Canon (`02-data-model.md:85`): `id`, `org_id`, `meeting_id`, `content`, `assignee_id`, `due_date`, `confidence`, `status`.

```
id            uuid PK DEFAULT gen_random_uuid()
org_id        uuid NOT NULL REFERENCES organizations(id) ON DELETE RESTRICT
meeting_id    uuid NOT NULL REFERENCES meetings(id)
content       text NOT NULL
assignee_id   uuid NULL REFERENCES users(id)       -- Avro: ["null","string"], default null
due_date      date NULL                             -- Avro: ["null", date], default null
confidence    real NOT NULL
status        text NOT NULL DEFAULT 'candidate'
                CHECK (status IN ('candidate','confirmed','open','in_progress','done','dismissed'))
confirmed_by  uuid NULL REFERENCES users(id)
confirmed_at  timestamptz NULL
blocked_on_decision_id uuid NULL REFERENCES decisions(id)
source_agent_result_id uuid NULL
source_chunk_ids uuid[] NOT NULL DEFAULT '{}'
created_at / updated_at / deleted_at   as above
```

`blocked_on_decision_id` is the relational source for the "actions blocked on a decision" template (`05-data-architecture.md:526`), which otherwise has no backing column anywhere — same argument as `supersedes_id`.

The `status` domain here carries a genuine ambiguity worth surfacing: canon gives one `status` column, but the lifecycle has **two** axes — confirmation (candidate → confirmed) and work progress (open → in_progress → done). Folding both into one column is what I have specified, because it matches canon's single field and because `integration-service` syncs external ticket status back onto `actions` (`_canon/01-service-catalog.md:120`, "external status → action update") which is a *progress* update on an already-confirmed row. The alternative — a separate `confirmation_status` — is cleaner but reopens a canon-frozen field list. Recommend the single column; flag it.

Indexes:
- `(org_id, assignee_id, status) WHERE deleted_at IS NULL` — "My Tasks" (P8 deliverable, `22-roadmap-and-phasing.md:88`) and the ProactiveAgent workload detector, which sums open actions per assignee (`08-ai-agent-architecture.md:49`).
- `(org_id, meeting_id, created_at DESC)`.
- `(org_id, due_date) WHERE status IN ('confirmed','open','in_progress') AND deleted_at IS NULL` — `ActionFollowUpWorkflow`'s day-1/3/7 reminder scan (`_canon/01-service-catalog.md:121`).
- `(blocked_on_decision_id) WHERE blocked_on_decision_id IS NOT NULL`.

Same RLS policy; same confirmation-monotonicity trigger.

#### `agent_suggestions` — size **S**

Canon (`02-data-model.md:86`): `id`, `org_id`, `user_id`, `meeting_id`, `question`, `priority` (1–3), `status` (Ask/Dismiss/Save-for-later). "QuestionAgent output. Save-for-later persists into next meeting."

```
id          uuid PK DEFAULT gen_random_uuid()
org_id      uuid NOT NULL REFERENCES organizations(id) ON DELETE RESTRICT
user_id     uuid NOT NULL REFERENCES users(id)
meeting_id  uuid NOT NULL REFERENCES meetings(id)
question    text NOT NULL
priority    smallint NOT NULL CHECK (priority BETWEEN 1 AND 3)
status      text NOT NULL DEFAULT 'pending'
              CHECK (status IN ('pending','ask','dismiss','save_for_later'))
carried_from_suggestion_id uuid NULL REFERENCES agent_suggestions(id)
source_agent_result_id uuid NULL
created_at / updated_at / deleted_at  as above
```

Note this table is **not** on the confirmation spine — it has no outbox emission and no graph edge. It is P9, and it is in this spec only because the gateway's `/api/v1/suggestions` prefix needs a home. `carried_from_suggestion_id` implements "Save-for-later persists into next meeting" — canon states the behaviour but names no mechanism; without a link column, "the same question, carried" is indistinguishable from "the agent asked it again".

Indexes: `(org_id, user_id, meeting_id, priority)`; `(org_id, user_id, status) WHERE status='save_for_later' AND deleted_at IS NULL` (the carry-forward scan).

#### `worklogs` — size **S**, and a scope note

`/api/v1/worklogs` is one of the four 404ing prefixes, so it is in scope for "fill the prefixes", but it is a **P7** table (`22-roadmap-and-phasing.md:108`) and it is **not** on the confirmation spine — PersonalListenerAgent notes are written directly, with no confirm step and no graph edge. Canon fields (`02-data-model.md:83`): `id`, `org_id`, `meeting_id`, `user_id`, `content`, `relevance_score`, `created_by_agent`, `source_chunk_ids uuid[]`. Same RLS/soft-delete conventions. Listed for completeness; it should be sequenced with P7, not bundled into the P8 confirmation work.

#### Grants — size **S**, easy to forget

`9b2c4d1e6f30` retired schema-wide default grants, so each new table needs explicit `GRANT SELECT, INSERT, UPDATE ON <table> TO mytheus_app`. `mytheus_relay` must get **nothing** — `mytheus-backend/migrations/_grants.py:37-39` states it holds a grant on exactly one table, `outbox`, and that reaching anywhere else is precisely what `outbox_relay_drain` was written to prevent. Do not widen it.

### Data-model sizing summary

| Piece | Size | Note |
|---|---|---|
| `decisions` table + RLS + indexes + monotonicity trigger | **M** | The trigger and `supersedes_id` are the substantive parts |
| `action_items` table + RLS + indexes + trigger | **M** | Status-axis ambiguity to resolve first |
| `agent_suggestions` | **S** | P9; not on the spine |
| `worklogs` | **S** | P7; not on the spine |
| Explicit grants for all of the above | **S** | Easy to omit; `_grants.py:37-39` is the guardrail |

---

## A2. Routes — size **L** in aggregate

Current state: `mytheus-backend/services/core_api/main.py:14-20` mounts exactly five routers — `meetings`, `org_hierarchy`, `chat`, `speakers`, `transcript`. Line 22 is the standing TODO: `# TODO: /decisions /actions /worklogs /suggestions land with their own bands.` `mytheus-backend/services/core_api/routes/` contains no `decisions.py` or `actions.py`.

Every route below follows the three invariants stated at `mytheus-backend/services/core_api/routes/meetings.py:3-14` — `org_id` never a path or query parameter (it comes from `RequestContext`, from the verified token); row write + outbox row + audit row commit in ONE transaction because they are one fact; a row in another tenant is a **404, never a 403**.

### The two confirm routes — the spine itself

**`POST /api/v1/decisions/:id/confirm`** and **`POST /api/v1/actions/:id/confirm`**. Each is one transaction doing four things:

1. RLS-scoped `SELECT ... FOR UPDATE` on the row; missing → 404 (`not_found()` from `libs.shared.authz`).
2. Authorization. `require_tier("member")` is the floor, but tier is not sufficient here — the confirmer must be a participant of the meeting. `routes/chat.py:59-84` (`_require_participant`) is the exact precedent and its reasoning transfers verbatim: without it, any member-tier org user could confirm a decision in any meeting in their org. Reuse `ErrorCode.NOT_A_PARTICIPANT` (403).
3. State change: `status='confirmed'`, `confirmed_by=ctx.user_id`, `confirmed_at=now()`. Re-confirming an already-confirmed row is a **no-op returning 200**, not an error — the outbox `ON CONFLICT (topic, idempotency_key) DO NOTHING` already makes a double-confirm a no-op at the event layer (`libs/shared/outbox.py:30-32`, "Guard 2 of the duplicate-decision stack"), and the route should agree with it rather than 409.
4. `await enqueue(session, envelope)` + `audit(...)`.

The envelope, built from `libs.shared.events.EventEnvelope`:

```python
EventEnvelope(
    topic="decisions",
    org_id=str(ctx.org_id),
    producer="core-api",
    idempotency_key=f"decision:{decision_id}:confirmed",   # taxonomy line 14, verbatim
    payload={
        "decision_id": str(row.id),
        "meeting_id": str(row.meeting_id),
        "content": row.content,
        "rationale": row.rationale,
        "confidence": row.confidence,
        "status": "confirmed",      # the enum's only symbol
        "confirmed_by": str(ctx.user_id),
    },
)
```

`schema_subject` (`decisions-value`) and `partition_key` (org_id) are derived properties on the envelope; `to_value()` composes `{header, **payload}`. The payload keys above are exactly the non-header fields of `mytheus-events/schemas/decisions/v1.avsc` — no more, no less, since the relay serializes against the registered subject and a stray key fails there. Actions is the same shape against `actions/v1.avsc` with `idempotency_key=f"action:{action_id}:confirmed"`.

Two new audit constants are needed in `libs/shared/audit.py` (which currently ends at `ACTION_TEAM_MEMBER_REMOVED`, line 93): `ACTION_DECISION_CONFIRMED = "decision.confirmed"` and `ACTION_ACTION_CONFIRMED = "action.confirmed"`.

Size: **M** for the pair. Small in lines, large in consequence — this is the code the whole architecture's integrity claim rests on, and it needs the atomicity test (row + outbox in one transaction, rollback leaves neither) plus the double-confirm test.

### The candidate-ingress consumer — the piece most likely to be missed

**Nothing currently writes a candidate row.** `mytheus-backend/services/core_api/consumer.py` consumes exactly one topic, `presence-events`, and its docstring is explicit that it writes no domain table. `agent-results` — the topic that carries `decision_candidate` and `action_candidate` — has `core-api` as a listed consumer (`_canon/03-event-taxonomy.md:12`) and **no consumer exists**.

Without this, the confirm routes have nothing to confirm. A build plan that specifies only the tables and the confirm routes ships a spine with no input.

The consumer follows the recipe `consumer.py` documents at lines 30-36: re-assert `org_id` from the envelope header with a strict `uuid.UUID(...)` parse before any write, dedupe via `libs.shared.inbox.is_duplicate` / `record_consumed` plus the Redis key, DLQ on parse failure. Dedupe key from taxonomy: `agentresult:{meeting_id}:{window_id}:{user_id}:{agent}`. Route on `kind`: `decision_candidate` → insert `decisions` with `status='candidate'`; `action_candidate` → `action_items`; `suggestion` → `agent_suggestions`; `note` → `worklogs`; `alert` → no row (rides `notifications`).

Size: **M**.

### List and read routes the UI needs

Follow `routes/meetings.py` throughout: keyset cursor over `(created_at, id)`, base64-opaque (`meetings.py:64-80`, and note the reason `id` is in the key — `created_at` alone is not unique and would make a row unreachable by paging), `DEFAULT_LIMIT = 50`, `MAX_LIMIT = 200`, `{data, page}` envelope per `mytheus-docs/design/openapi/core-api.yaml:89-94`, `ErrorCode.INVALID_CURSOR` on a malformed cursor.

| Route | Purpose | Size |
|---|---|---|
| `GET /api/v1/decisions?meeting_id=&status=` | Meeting decisions panel; candidate queue | S |
| `GET /api/v1/decisions/:id` | Detail | S |
| `PATCH /api/v1/decisions/:id` | The *edit* in confirm/edit/dismiss. Editable only while `status='candidate'` | S |
| `POST /api/v1/decisions/:id/dismiss` | The *dismiss* arm | S |
| `GET /api/v1/decisions/:id/lineage` | Decision-lineage UI (P8 deliverable). **Reads the graph via `knowledge-graph-service`** — the one route on this list whose backing store is the Part B question | M |
| `GET /api/v1/actions?assignee_id=&status=&due_before=` | My Tasks | S |
| `GET /api/v1/actions/:id`, `PATCH`, `POST /:id/dismiss` | Mirror of decisions | S |
| `GET /api/v1/suggestions?meeting_id=` + `PATCH /:id` (Ask/Dismiss/Save-for-later) | QuestionAgent cards | S |
| `GET /api/v1/worklogs?meeting_id=` | Worklog panel | S |

Plus **OpenAPI + generated clients**: ADR-0022 emits OpenAPI from code, and `mytheus-docs/design/openapi/core-api.yaml` currently contains no `decisions`/`actions`/`suggestions`/`worklogs` paths (grep returns nothing). Spectral lints the specs (`design/openapi/.spectral.yaml`), and the BFF arm rename below is a tracked breaking change. Size **M**.

**BFF consequence, size S but visible:** `mytheus-backend/services/bff/dashboard.py:90-107` defines `UnbuiltArm` for `action_items`, `worklogs`, `decisions`, with a docstring saying the class is replaced by a typed arm when a read model lands, and that "that rename is a visible breaking change rather than a silent shape swap". Landing these routes triggers exactly that rename. It is pre-authorized by that docstring; it still needs an `oasdiff` breaking-change entry.

### Route sizing summary

| Piece | Size |
|---|---|
| Two confirm routes + audit constants + atomicity/idempotency tests | **M** |
| `agent-results` candidate consumer | **M** |
| List/read/patch/dismiss across four resources | **L** (many small routes; the volume is the cost) |
| OpenAPI emission + generated clients + Spectral | **M** |
| BFF typed-arm replacement | **S** |
| **Aggregate** | **L** |

---

## A3. Outbox emission on confirm, and the consumer that acts on it

### Emission

The mechanism exists and needs no new infrastructure. `libs/shared/outbox.py:19-25` states the pattern:

```python
async with session.begin():
    await session.execute(...)        # state change
    await enqueue(session, envelope)  # atomic intent-to-emit
```

`tenant_session` yields inside `session.begin()` (`routes/meetings.py:8-11`), so the handler structurally cannot split the row write from the outbox row. `enqueue` inserts with `ON CONFLICT (topic, idempotency_key) DO NOTHING` against the `outbox_idem_uq` unique constraint (`infra_models.py:46`) and returns the existing id on a duplicate — a double-confirm produces one event. Working precedents to copy: `services/identity_tenant_service/provisioning.py:74-132` (state change + outbox row in one transaction, with RLS set before the outbox write because the outbox is itself a tenant-scoped table — line 24-25 makes the point explicitly, and it applies here too) and `services/meeting_service/webhook.py:22`.

The relay (`libs/shared/relay.py`, `services/relay/`) drains unpublished rows `FOR UPDATE SKIP LOCKED`, Avro-serializes against `{topic}-value`, publishes keyed by `org_id`, marks published. It is topic-agnostic — it needs **no change** for two new topics beyond the subjects being registered, which they already are (`mytheus-events/subjects/decisions-value.json`, `actions-value.json`, both `BACKWARD` compatibility).

So the outbox work is genuinely small: **S**. The `enqueue` call and the correct envelope. Everything underneath is built and tested.

### Which consumer acts on it

Per `_canon/03-event-taxonomy.md:14-15`:

- `decisions` → `knowledge-graph-service`, `notification-service`
- `actions` → `knowledge-graph-service`, `workflow-service`, `integration-service`

The **primary** consumer, and the one the integrity rule is about, is `knowledge-graph-service`. Its own docstring says so: `mytheus-intelligence/services/knowledge_graph_service/main.py:1-4` — "the ONLY writer of the AGE `nexus_knowledge` graph … Cypher parameterized by `$org_id`, driven by confirmed `decisions`/`actions` from core-api … No silent graph writes." It creates `PRODUCED` (Decision→Meeting) on a decision event, and `PRODUCED` + `ASSIGNED_TO` (Action→Person) on an action event (`05-data-architecture.md:493-499`).

That service is a **scaffold**. Its entire implementation is:

```python
from libs.shared.app import create_app
app = create_app("knowledge-graph-service")
```

No consumer, no routes, no store. Same for `agent_runtime_service/main.py` — scaffold, "Scaffold only: boots and answers `/healthz`."

Taxonomy also specifies at-least-once with `MERGE` on both topics — the graph writer must be idempotent by construction, which `MERGE` gives on a property graph and which the relational fallback gives via `INSERT ... ON CONFLICT DO NOTHING` on an edge table.

**This is where Part A hands off to Part B.** The confirm route can be built, tested, and shipped, and it will correctly emit — but the consumer that closes the loop cannot be written until the graph store question is answered. Consumer size: **M** relational / **L** on AGE (the extra weight is standing up the store, not the consumer logic).

---

## A4. The 404ing gateway prefixes

`mytheus-backend/services/api_gateway/edge.py:146-155` — `ROUTE_MAP` already contains all four:

```python
"/api/v1/worklogs":    "core-api",
"/api/v1/decisions":   "core-api",
"/api/v1/actions":     "core-api",
"/api/v1/suggestions": "core-api",
```

The gateway is **not** the defect. It resolves these prefixes correctly and proxies to `core-api`; `core-api` has no matching router mounted (`main.py:14-20`), so FastAPI returns 404 and the gateway relays it. The routes specified in A2 fill these prefixes with **no gateway change at all** — the map entries are already correct and were written in advance of the routes.

One thing worth preserving: the comment at `edge.py:156-160` documents a real past defect (meeting-service missing from the map entirely, making LiveKit token minting unreachable) and the deliberate near-collision between `/api/v1/meetings` and `/api/meetings`, kept apart by longest-prefix matching. Nothing in this work touches that, but a new prefix must not be added that prefixes an existing one.

Gateway work: **none**. Size **S** only for the integration test asserting each prefix now returns 200 rather than 404.

---

## A5. Part A build order and total

| # | Piece | Size | Blocks |
|---|---|---|---|
| 1 | Resolve two open questions: `decisions.status` domain; `action_items` one-status-axis vs two | — | 2 |
| 2 | Migration: `decisions` + `action_items` + RLS + indexes + monotonicity triggers + grants | M | 3,4 |
| 3 | `POST /decisions/:id/confirm`, `POST /actions/:id/confirm` + outbox + audit | M | 6 |
| 4 | `agent-results` candidate consumer in core-api | M | 5 |
| 5 | List/read/patch/dismiss routes; `agent_suggestions` + `worklogs` migrations and routes | L | 7 |
| 6 | `knowledge-graph-service` consumer of `decisions`/`actions` | M–L | Part B |
| 7 | OpenAPI + clients + BFF typed arms + gateway-prefix integration test | M | — |

Aggregate: **L**. Items 2 and 3 together are the minimum that makes the integrity rule *operable*; item 4 is the minimum that makes it *reachable*; item 6 is the minimum that makes it *complete*, and it is gated on Part B.

---

# PART B — the knowledge graph, given AGE does not exist

## B1. Confirming the audit — and one correction to it

**Confirmed, and it is stronger than "two docstring lines".**

Repo-wide search for AGE, `ag_catalog`, Cypher, Neo4j across `*.py`, `*.ts`, `*.tsx`, `*.sql`, `*.yml`, `*.yaml`, `*.toml`, `*.txt`, `*.json`, `Dockerfile*` returns eight files. Every one is prose:

| File:line | What it is |
|---|---|
| `mytheus-intelligence/services/knowledge_graph_service/main.py:1,3` | Docstring: "the ONLY writer of the AGE `nexus_knowledge` graph" |
| `mytheus-intelligence/services/rag_service/main.py:3` | Docstring: "RRF fusion across the Qdrant/AGE/FTS arms" |
| `mytheus-backend/tests/test_seed.py:15` | Comment about `Topic` node counts |
| `mytheus-backend/tests/test_role_topics_taxonomy.py:11-13` | Comment: Topic-node seeding into AGE is "explicitly OUT OF SCOPE"; this repo "reads `nexus_knowledge` but never writes it" |
| `mytheus-backend/services/identity_tenant_service/seed/seed.py:43,456` | Comment: Topic nodes owned by knowledge-graph-service |
| `mytheus-backend/libs/shared/idempotency.py`, `services/workflow_service/workflows/rtbf.py` | Substring matches on unrelated words |
| `mytheus-ui/src/app/(marketing)/privacy/page.tsx` | Substring match |

No `ag_catalog`. No `CREATE EXTENSION age`. No Cypher statement anywhere. No Neo4j driver in any dependency file. No graph schema in any migration. `knowledge_graph_service` is two lines of `create_app`.

**The correction, and it matters for the options below.** The audit's framing inherits ADR-0005's assumption of *managed Supabase*. That is no longer the substrate. ADR-0016 replaced the PaaS with a **self-hosted single-node VPS under Docker Compose**, and `mytheus-platform/deploy/compose.prod.yml:79` runs `image: supabase/postgres:15.8.1.060` — the Supabase *image*, self-hosted, with root on the box. So R1's stated trigger ("managed Supabase gates `shared_preload_libraries`") **does not bind**: we control the box and can set `shared_preload_libraries` freely.

The binding constraint is different, and I verified it directly rather than assuming. Inspecting the actual image:

```
$ docker run --rm --entrypoint sh supabase/postgres:15.8.1.060 -c \
    "ls .../share/postgresql/extension/ | wc -l"
1535
$ ... | grep -i '^age'
(no output)
$ ... | grep -i vector
vector.control  vector--0.8.0.sql  ... (pgvector present)
```

The image ships 1535 extension files including pgvector, pg_graphql, pgmq, TimescaleDB, PostGIS, pgrouting — and **no `age`**. There is no `age.control` and no `age.so`. `CREATE EXTENSION age` fails on the production image today. That is the real blocker: not a managed-provider policy, but the image itself.

(Separately: the local test container `mytheus-pg-test` is running `postgres:16-alpine`, not the Supabase image the compose files declare, and its `pg_available_extensions` is 61 rows with no `age`. That divergence between the test substrate and the declared dev/prod substrate is a real finding but is out of scope here — worth handing to whoever owns the test harness.)

So: ADR-0005 is **Accepted** and specifies a store that has never been installed, on an image that cannot install it, against a risk whose stated trigger has been made obsolete by a later ADR. Zero implementation exists on either side of it.

## B2. What ADR-0005 and the roadmap already authorize

**ADR-0005's own fallbacks (D4, `adr/0005-graph-store-age-with-migration.md`).** Two exits are named, pre-engineered against a schema the ADR insists is store-agnostic ("expressible in property-graph Cypher … and in relational edge tables alike"):

| Exit | ADR-0005's stated trigger | What moves | What stays |
|---|---|---|---|
| Neptune / Neo4j Aura | Cypher req/s outgrows a Postgres-coupled graph, **or** Citus sharding makes AGE-on-coordinator the bottleneck | Labels/edges lift into a managed graph; the service swaps its driver behind the same API | `$org_id` templates, edge-on-confirmation, soft delete, consuming agents |
| Relational materialized-view fallback | "Graph traffic stays light and AGE itself proves operationally awkward … (**R1 triggered early**)" | Edges become plain Postgres edge tables; the four templates answered with recursive CTEs | Same data, same `org_id` scoping, predictable SQL; sacrifices ad-hoc Cypher |

The MV row's trigger is a precise description of the present situation: graph traffic is zero, and AGE is unavailable on the chosen image. ADR-0005 calls this "the low-novelty exit" and, in Alternatives Considered, describes "no graph; recursive CTEs over relational edge tables only" as "**viable for the four templates**", rejected as the default only because it forecloses ad-hoc Cypher exploration "that the product roadmap wants".

**The R1 descope (`22-roadmap-and-phasing.md:191-199`).**

- **Trigger:** "Apache AGE proves operationally awkward on managed Supabase (extension instability, ops friction, query latency)."
- **Fallback:** "Migrate `nexus_knowledge` to **Neo4j Aura** (or self-host Postgres+AGE); relational materialized-view fallback documented as the at-scale exit."
- **Cost of pulling:** "Graph store swap; Cypher templates re-pointed; **`knowledge-graph-service` already abstracts this**."

Note the phase: R1's checkpoint is **P8 (W21–23)** (`22-roadmap-and-phasing.md:88`), the same phase that builds `decisions`/`action_items` and the graph. The decision point is *now*, by design, not late.

**What `knowledge-graph-service` was supposed to abstract.** Precisely one thing: the internal sync API. Per `_canon/01-service-catalog.md:118` and ADR-0005 D3, consumers call only `POST /graph/query/{template}` and `POST /graph/cypher` (internal-only). ADR-0005's migration notes name this "the migration shield": "Consumers (ProactiveAgent, ContextGapAgent, `core-api` read path) call only `POST /graph/query/{template}` with `$org_id` bound; the engine swap happens **behind** that API. No agent or UI code changes on migration."

The four named templates are the whole query surface (`05-data-architecture.md:526`): *related decisions for a topic*, *decisions by a person*, *actions blocked on a decision*, *decision lineage*.

**The consequence nobody has priced.** That abstraction is worth a great deal *if it exists*. It does not — the service is two lines. So the "cost of pulling R1" line in the roadmap ("`knowledge-graph-service` already abstracts this") is, today, **false**. The good news is that this cuts in our favour: there is no AGE-coupled code to unwind, no Cypher to re-point, no dual-read verification to run, no per-tenant cutover. The entire elaborate migration machinery ADR-0005 specifies is machinery for a migration we do not have to perform. **Choosing the store now costs the migration cost of nothing.** That is a fact worth putting in front of whoever makes the call.

## B3. The three options, honestly costed

Baseline for all three: zero graph code exists, so every option pays for the `knowledge-graph-service` implementation (the `decisions`/`actions` consumer, the four templates behind `POST /graph/query/{template}`, RTBF integration). The differences below are *incremental* to that shared cost.

---

### (a) Install and use AGE on the existing Postgres

**What it costs.**

- A custom Postgres image. `supabase/postgres:15.8.1.060` has no `age`; the image is Nix-built (extensions live under `/nix/store/...-postgresql-and-plugins-15.8/`), so this is not a one-line `apt-get install`. It means either building AGE against that Nix derivation or abandoning the Supabase image for a stock Postgres plus AGE plus every extension the codebase already relies on. Note the image ships pgvector, pgmq, pg_cron, pgaudit, pgsodium — dropping it has its own audit cost.
- Version coupling, permanently. AGE tracks Postgres majors slowly. Pinning to PG15 for AGE's sake makes every future Postgres upgrade an AGE-availability question first. ADR-0005 lists this as accepted ("the extension lags Postgres major versions") but accepted it on the assumption the extension was at least *obtainable*.
- `shared_preload_libraries` change plus a restart, on the single-node VPS where Postgres, Redis, Redpanda and Temporal all live (ADR-0016). Available to us — root access is exactly what ADR-0016 bought (`0016:47`, "Root access permits tuning PostgreSQL") — but it is a config change to the one stateful component with no replica.
- Backup and restore rehearsal, redone. `ADR-0016:105` requires a restore drill before any real tenant. A custom image with an extension in `shared_preload_libraries` is a new restore path and invalidates the previous drill.
- Ongoing: AGE is a small project relative to Postgres. Owning it on a self-hosted single-node box, pre-revenue, with no graph traffic, is real operational surface.

**What it forecloses.** Little in principle — ADR-0005's exits stay open, and it is the only option that keeps ad-hoc Cypher. In practice it forecloses easy Postgres upgrades, and it commits the team to a custom database image before the first tenant.

**Contradiction detection under it.** The intended design, unchanged: ProactiveAgent asks `knowledge-graph-service` for decisions contradicting past decisions along the `FOLLOWS_FROM` chain (`08-ai-agent-architecture.md:49`), a bounded-depth org-scoped traversal (`FOLLOWS_FROM*1..10`). Inside the p50 < 2s / p95 < 4s agent budget.

---

### (b) The R1 descope — relational materialized-view fallback

**What it costs.**

- Two edge tables and one materialization, on the substrate that already exists. No image change, no extension, no `shared_preload_libraries`, no restart, no new restore drill. Concretely: `decision_edges (org_id, from_decision_id, to_decision_id, kind, created_at, deleted_at)` and `entity_topic_edges (org_id, entity_type, entity_id, topic_name, ...)`, plus `ATTENDED` which `meeting_participants` already is. `PRODUCED` and `ASSIGNED_TO` need no edge table at all — they are `decisions.meeting_id`, `action_items.meeting_id` and `action_items.assignee_id`, columns specified in A1.
- The four templates as recursive CTEs behind `POST /graph/query/{template}`. Lineage is a `WITH RECURSIVE` walk of `supersedes_id` with a depth bound of 10, mirroring `FOLLOWS_FROM*1..10` exactly. This is unremarkable SQL and it is index-backed by the indexes A1 already specifies.
- Every convention the codebase enforces applies unchanged: RLS `tenant_isolation` on the edge tables gives tenant scoping **at the database**, which is strictly stronger than AGE's `$org_id` template parameter — ADR-0005 D2 has to argue at length that the template parameter is the primary guard with an index as backup, because AGE has no RLS. Under (b), `org_id` scoping is the same mechanism as every other table in the system, verified by the existing RLS test suite.
- RTBF gets simpler, not harder: the cross-store deletion saga (`05-data-architecture.md:8.1`) loses a store.
- Cost of *later* moving to a real graph: this is the honest debit. ADR-0005's migration machinery would then have to be built and run. But per B2, it has to be built either way — nothing abstracts anything today — and the schema stays store-agnostic by construction because the edges are literally the ADR's five labels.

**What it forecloses.** Ad-hoc Cypher exploration — ADR-0005's stated reason for rejecting this as the default ("decision-influence analysis, multi-hop stakeholder discovery"). That is a genuine loss, and it is a loss of a *future* capability with no current consumer: `POST /graph/cypher` is internal-only, and nothing calls it. It also forecloses unbounded-depth traversal, though every specified template is depth-bounded anyway.

**Contradiction detection under it.** This is the part that deserves scrutiny rather than assumption, and it holds up. Decompose what ProactiveAgent's conflict detector actually does (`08-ai-agent-architecture.md:49`): a new confirmed decision contradicting a past one. Two mechanisms, and the graph is only half:

1. *Candidate retrieval* — find past decisions plausibly about the same subject. This is **already Qdrant**, not the graph. `05-data-architecture.md:4` gives a `decisions__{org_id}` collection, and ADR-0005's Alternatives section concedes "Qdrant `decisions__{org_id}` already supports semantic conflict detection". Unaffected by this choice.
2. *Lineage/structure* — has this decision already been superseded, what chain does it sit in, what actions are blocked on it. Under (b) this is a recursive CTE over `supersedes_id`, org-scoped by RLS, depth-bounded at 10, hitting the `(supersedes_id)` partial index. For the sizing canon gives — E4 ≈ 3 decisions/meeting, and ADR-0005's own "the graph is a *stock* of confirmed facts, not a high-volume flow … ~8 confirmed facts/meeting" — a per-tenant lineage chain is tens to low hundreds of rows. A depth-10 recursive CTE over that returns in single-digit milliseconds, comfortably inside p50 < 2s.

The honest limit: contradiction detection under (b) is exactly as good as under AGE for the four named templates, and worse the moment someone wants a query nobody has written yet. Since the semantic arm is Qdrant either way, and the structural arm is bounded and small, the practical difference for the *shipped* feature is close to nil.

---

### (c) Neo4j Aura

**What it costs.**

- A sixth stateful store, with its own backup, PITR, residency, and DR runbook. ADR-0005 rejects this as the day-one choice for exactly this reason, and ADR-0016's whole argument is that the MVP runs on one box with fixed, sunk cost.
- A recurring bill, pre-revenue. Aura's free tier will not carry a multi-tenant production graph; the paid tier is real money for a product with no tenants.
- A network hop on the live agent latency path — ADR-0005 names this as a reason not to start here, since two of the three graph consumers run inside live meetings under p50 < 2s.
- Residency. `data_residency` pins tenant data to a region/cell (`_canon/02-data-model.md:109`); a second provider means a second residency story and a second GDPR surface, and `12-compliance-and-privacy.md` obligations extend to it. This is the largest hidden cost.
- Loss of transactional edge writes. ADR-0005's first listed Positive is that "an edge is committed in the same Postgres transaction that confirms its `decisions`/`action_items` source row, so the graph never diverges from the confirmed relational truth — important because edges are the input to conflict detection." Aura cannot be in that transaction. Divergence becomes possible and needs reconciliation — which is why the taxonomy already mandates `MERGE`-based idempotent consumption.
- Loss of RLS. Tenant isolation becomes application-enforced only, in a store with no `org_id` policy engine. That is a step down from every other store in this system.

**What it forecloses.** Nothing technically — it is the terminal state ADR-0005 points at. It forecloses the single-box, fixed-cost posture ADR-0016 chose deliberately, and it spends the compliance and ops budget now for capability needed at a scale that does not exist.

**Contradiction detection under it.** Best-in-class: full Cypher, mature planner, unbounded traversal. For the four bounded templates against a per-tenant subgraph of a few hundred nodes, indistinguishable in outcome from (b) — and slower end-to-end by the network hop.

---

## B4. Recommendation

**Take (b): the R1 relational materialized-view fallback. Now, at the P8 checkpoint, and record it as the planned path it is.**

The reasoning, in order of weight:

1. **The trigger has fired on its own terms.** ADR-0005 D4 gives the MV exit the trigger "graph traffic stays light and AGE itself proves operationally awkward … (R1 triggered early)". Graph traffic is zero. AGE is not merely awkward — it is absent from the production image, verified by inspection, and installing it means abandoning or rebuilding that image. This is not a judgment call about whether the trigger has been met; it is met, more decisively than the ADR anticipated.

2. **The roadmap pre-authorizes this as planned behaviour.** `22-roadmap-and-phasing.md:191-199` lists R1 as one of three pre-authorized descopes with a verbatim ID, phase, trigger and fallback — "*project-scope* safety valves". P8 is the checkpoint, and P8 is the phase this build plan is for. Pulling R1 at its designated checkpoint is the plan executing correctly. It should be written up as a decision, not as a slip, and it needs no supersession of ADR-0005 — the ADR *contains* this exit. What it needs is a short amendment noting that R1's original trigger (managed-Supabase gating) was made obsolete by ADR-0016 and replaced by a concrete one (no `age` in the chosen image), plus a status note that the graph runs on the D4 relational exit.

3. **The migration this normally costs is free right now.** ADR-0005's elaborate exit machinery — snapshot per `org_id`, bulk load, driver swap, dual-read verification, per-tenant flag cutover, keep-AGE-warm rollback — is all migration *from* an installed AGE. Nothing is installed. There is no Cypher to re-point and no consumer to cut over. The roadmap's "`knowledge-graph-service` already abstracts this" is false today, which means the store decision is being made *before* anything couples to it. This is the cheapest moment this decision will ever be made, and it gets more expensive every week something is built on top.

4. **It is strictly better on the things this system already treats as non-negotiable.** Tenant isolation becomes RLS-enforced at the database, the same mechanism and the same test suite as every other table, rather than AGE's template-parameter-plus-index arrangement that ADR-0005 D2 has to argue for. Edges commit in the same transaction as the confirmed row, preserving ADR-0005's own first-listed Positive. One backup domain, one PITR timeline, one residency boundary, one restore drill — which is ADR-0016's entire thesis. RTBF loses a store instead of gaining one.

5. **The feature does not suffer.** Contradiction detection's semantic arm is Qdrant `decisions__{org_id}` under every option. Its structural arm is a depth-bounded walk over a per-tenant stock of confirmed decisions that canon itself sizes at ~3 decisions and ~5 actions per meeting. A recursive CTE over `supersedes_id` with the partial index from A1 answers it in milliseconds. The four named templates are all expressible; ADR-0005 says so directly ("viable for the four templates").

6. **What we give up is real, and it is a future option, not a present capability.** Ad-hoc Cypher exploration — the ADR's stated reason for preferring AGE — has no consumer today; `POST /graph/cypher` is internal-only and nothing calls it. Buying it now costs a custom database image (a) or a sixth stateful store, a bill, and a second residency surface (c). Neither is worth paying pre-revenue for a capability with no caller.

**One condition attached, and it is the thing that makes the recommendation safe.** The API shield must be built for real this time, because it is what makes (b) reversible: `knowledge-graph-service` exposes `POST /graph/query/{template}` and nothing else, no consumer touches SQL or Cypher directly, and the four templates live behind named functions. That is ADR-0005 D3's contract and it costs the same to honour under (b) as under AGE. If it is honoured, moving to (c) later is the driver swap the ADR describes — and the edges remain re-derivable from `decisions`/`action_items` rows, which is exactly why A1 specifies `supersedes_id` and `blocked_on_decision_id` as real columns. Without those columns, `FOLLOWS_FROM` and "actions blocked on a decision" have no relational source and ADR-0005's "the graph is a projection, never the only copy of the data" is not true. With them, it is.

**Do not pick (a).** It buys ad-hoc Cypher nobody calls, at the price of a custom Nix-built Postgres image, a permanent version coupling, and a redone restore drill on a single-node box, before the first tenant.

**Do not pick (c) yet.** It is the correct terminal state and ADR-0005 already points at it. Its own triggers — Cypher req/s saturating replicas, or Citus sharding — are both far away and both measurable, so the decision to take it can be made on evidence later instead of on speculation now.

**Sizing.** (b) is **M**: two edge tables in one migration following the `meeting_speakers` template, four recursive-CTE templates, the `decisions`/`actions` consumer, and the `/graph/query/{template}` surface. (a) is **L** and front-loads image and ops work before any product value. (c) is **L** plus recurring cost plus a compliance and residency workstream.

---

## Open questions for the team lead

1. `decisions.status` domain — canon does not enumerate it. Proposed `candidate | confirmed | dismissed | superseded`.
2. `action_items.status` — one column carrying both confirmation and work-progress axes (proposed, matches canon's single field and integration-service's status sync back), or two columns (cleaner, reopens a frozen field list).
3. `supersedes_id` and `blocked_on_decision_id` are net-new relative to canon's field lists. They are what make `FOLLOWS_FROM` and "actions blocked on a decision" re-derivable from relational truth. Recommend accepting; needs an explicit nod since canon field lists are frozen.
4. Confirm that the R1 pull is recorded as a planned descope with an ADR-0005 amendment (status note + obsolete-trigger correction), not as an ADR supersession.
5. Out of scope but worth routing: `mytheus-pg-test` runs `postgres:16-alpine` while both compose files declare `supabase/postgres:15.8.1.060`. Tests are not running against the declared substrate.
