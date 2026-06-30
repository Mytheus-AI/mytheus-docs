# Plan 5-6 — Org Context & AI Infrastructure  (AI Substrate)

> This band builds the substrate the six agents stand on: role/skills + `get_user_context()` org context, the DEDICATED GPU inference tier (embedding + reranker, off the API tier), per-tenant Qdrant, document ingestion + hybrid-search FTS arm, and the full Redpanda/Avro/outbox-inbox-DLQ event backbone whose first at-least-once consumer is the embedding consumer.

---

## 0. Band at a glance

| Field | Value |
|---|---|
| **Phases** | **P5 Profiles & Org Context (W13-14)** + **P6 AI Infrastructure (W15-17)** |
| **Theme** | AI Substrate |
| **Goal** | Make the org legible to agents (`role`/`skills`/teams/departments → `get_user_context`), stand up the GPU inference tier and per-tenant Qdrant, ingest documents, and turn the event backbone GA with the embedding consumer as the first real at-least-once consumer. |
| **Repos touched** | `mytheus-backend` (`identity-tenant-service`, `libs/shared`, Alembic), `mytheus-intelligence` (`embedding-service`, `reranker-service`, `ingestion-service`, `search-service`, `libs/shared`, `proto/`), `mytheus-events` (Avro schemas, Schema Registry registration, topic/DLQ provisioning, `nexus_events` lib), `mytheus-ui` (`web-app` org-chart + people pages, `bff` aggregation), `mytheus-mobile` (person profile screen) |
| **Folders touched** | `mytheus-backend/services/identity-tenant-service/`, `mytheus-backend/libs/shared/`, `mytheus-backend/migrations/`, `mytheus-intelligence/services/{embedding,reranker,ingestion,search}-service/`, `mytheus-intelligence/libs/shared/`, `mytheus-intelligence/proto/`, `mytheus-events/schemas/`, `mytheus-events/provisioning/`, `mytheus-events/src/nexus_events/`, `mytheus-ui/apps/web-app/`, `mytheus-mobile/app/` |
| **Demoable at end of P5** | Open the web-app org chart for a seeded org; click a person; see their `role`, `skills`, team, department, and recent meetings rendered from `get_user_context()` (Redis-cached, 5-min TTL). Same data renders on the mobile person profile. |
| **Demoable at end of P6** | Upload a PDF via `POST /ingest`; watch it land in `nexus-documents`, parse into `document_chunks`, emit `document-chunks`, and the embedding consumer embed it (batch 32, sub-100 ms) into `document_chunks__{org_id}` in Qdrant; run a `/search` FTS query and a Qdrant vector search and see both return org-scoped hits; observe `embedding-events` acks and a clean DLQ. |

---

## 1. Entry preconditions  (must be GREEN before starting)

| # | Precondition | Why it gates this band |
|---|---|---|
| 1 | **Plan 1-2 identity foundation is live and reverted to Clerk** — `libs/shared` performs **Clerk JWKS** JWT verification (NOT Supabase Auth), the **Svix webhook** handler in `identity-tenant-service` syncs Clerk users into `public.users`, `CLERK_*` env is restored, and the Supabase `custom_access_token_hook` + `auth.users→public.users` trigger are removed. `organizations` and `users` tables exist with `org_id` everywhere. | `get_user_context()` reads `users` populated by Clerk sync; the `org_id` claim Clerk injects into the JWT (session claims / JWT template) is extracted once at the `api-gateway` edge and is the input to every store in this band. |
| 2 | **Plan 3-4 meeting + realtime tier is live** — `meetings`, `meeting_participants`, `meeting_speakers`, `transcript_chunks`, `chat_messages` tables exist; `transcription-service` produces the `transcript-chunks` topic; `realtime-gateway` is up. | `get_user_context` joins recent meetings; the embedding/search consumers in P6 consume `transcript-chunks`; the seeder needs `meetings`. |
| 3 | **`libs/shared` (mytheus-backend) building blocks exist**: tenancy `ContextVar` + `SET LOCAL app.current_org`, transactional-outbox **enqueue**, inbox Redis-dedupe, `EventEnvelope`, RFC7807 errors, OTel. (The **relay/publisher**, the **gRPC server**, **all** Alembic migrations, and any real routes are still MISSING and are built here.) | This band adds the relay, the `NexusProducer`, the DDL migrations, and the first real consumers on top of these. |
| 4 | **Infra reachable from dev + CI**: Postgres (Supabase, AGE-enabled), Redis (Upstash), Redpanda + Schema Registry, Qdrant Cloud, an S3-compatible object store, and at least one GPU runner (or CPU fallback for dev). `docker-compose` (Postgres/Redis/Redpanda/Temporal) boots. | P6 stands up Qdrant collections, registers Avro subjects, and runs the GPU model servers. |
| 5 | **Partition-key decision (DECISION-REQUIRED §8.1) is at least scheduled for ratification this band.** | The embedding + agent consumer groups and their ordering guarantees depend on the `transcript-chunks` / `agent-stream` partition key; wiring consumers before ratification risks a re-cut. |

Gate to start P6: P5's `get_user_context()` endpoint and the seeder are green (the agent tier reads context), AND the event-backbone foundation (§3.6) is registered.

---

## 2. Canonical names introduced in this band

Use these **verbatim**. None are invented here beyond the two net-new Avro schema names already sanctioned by canon (`DocumentChunk`, `EmbeddingEvent`).

- **Services** (catalog names, never the data-model coarse variants): `identity-tenant-service`, `embedding-service`, `reranker-service`, `ingestion-service`, `search-service`. (`agent-runtime-service`, `rag-service`, `knowledge-graph-service` are referenced as downstream consumers but are NOT built in this band.)
- **Tables** (canon `_canon/02-data-model.md`): `organizations`, `users` (columns `role`, `skills text[]`, `bio`, `avatar_url`), `teams`, `team_members` (PK `(team_id, user_id)`), `departments` (self-referential `parent_id`), `meetings`, `meeting_participants`, `transcript_chunks`, `chat_messages`, `document_chunks` (columns `document_id`, `element_id`, `element_type`, `text`, `position`), plus `document_elements` (owned by `ingestion-service`) and the `outbox` / `inbox` infra tables. `tenant_settings` (for `agent_confidence_overrides`, `retention_days_*`, `pii_redaction_enabled`) is referenced; create the row-shell if Plan 1-2 did not.
- **Qdrant collections** (logical → physical): `transcript_chunks` → `transcript_chunks__{org_id}`, `document_chunks` → `document_chunks__{org_id}`. (`decisions__{org_id}` is provisioned-but-unused until decisions land.) 384-dim BGE-small-en, cosine, int8 scalar quantization, mandatory `org_id` payload filter.
- **AGE graph** `nexus_knowledge` node label `Topic` (`name`, `org_id`) is *seeded* by the role→topics taxonomy in this band; edges are not written yet.
- **Topics** (21 + 21 DLQs): all of `transcript-chunks`, `document-chunks`, `agent-results`, `agent-stream`, `decisions`, `actions`, `agent-requests`, `agent-responses`, `notifications`, `tenant-events`, `meeting-events`, `meeting-room-events`, `integration-events`, `usage-events`, `billing-events`, `graph-events`, **`embedding-events`**, `presence-events`, `notification-delivery-events`, `user-sync-events`, `admin-events`, each paired with `<topic>.dlq`. **Created/registered** in this band; the ones produced/consumed *live* here are `transcript-chunks`, `document-chunks`, `embedding-events` (and the `user-sync-events`/`tenant-events` already used by identity).
- **Avro schemas**: the 7 FROZEN (`TranscriptChunk`, `AgentResult`, `Decision`, `Action`, `Notification`, `AgentRequest`, `AgentResponse`) registered; net-new `DocumentChunk` (document-chunk envelope) and `EmbeddingEvent` registered. Every value opens with `EventEnvelopeHeader` (`event_id`, `idempotency_key`, `org_id`, `schema_version`, `producer`, `occurred_at`, `correlation_id`, `causation_id`).
- **Redis keys**: `nexus:{org_id}:user_ctx:{user_id}` (5-min TTL), `nexus:{org_id}:qembed:{query_hash}` (~10-min TTL), `nexus:{org_id}:dedupe:{topic}:{idempotency_key}` (24h TTL).
- **Object buckets**: `nexus-documents` (`{org_id}/{document_id}/`), `nexus-attachments` (`{org_id}/{meeting_id}/`).
- **Internal surfaces**: `GET /context/{user_id}` (identity), `GET /taxonomy/role-topics` (identity), `POST /embed` (embedding, internal gRPC), `POST /rerank` (reranker, internal gRPC), `POST /ingest` + `GET /ingest/jobs/:id` (ingestion), `GET /search` (search).

---

## 3. Work breakdown  (ORDERED — phase → service/component)

### PHASE P5 — Profiles & Org Context (W13-14)

The precondition chain in P5: schema (3.1) → taxonomy (3.2) → `get_user_context` (3.3, depends on both) → seeder (3.4, exercises 3.1-3.3) → UI (3.5, renders 3.3). Build in this order.

---

#### 3.1 `identity-tenant-service` — role/skills, teams, departments

**Repo / folder:** `mytheus-backend/services/identity-tenant-service/`, migrations under `mytheus-backend/migrations/versions/`.

**Steps**

1. Add SQLAlchemy 2.0 async models. `users` already exists from Plan 1-2 — extend it; `teams`/`team_members`/`departments` are new.

   `mytheus-backend/services/identity-tenant-service/models/org.py`:
   ```python
   from sqlalchemy import String, ForeignKey, UniqueConstraint
   from sqlalchemy.dialects.postgresql import ARRAY, UUID as PgUUID
   from sqlalchemy.orm import Mapped, mapped_column, relationship
   from libs.shared.db import Base, TenantMixin  # provides id, org_id, created_at, updated_at, deleted_at

   class Department(Base, TenantMixin):
       __tablename__ = "departments"
       name: Mapped[str] = mapped_column(String, nullable=False)
       parent_id: Mapped[PgUUID | None] = mapped_column(ForeignKey("departments.id"), nullable=True)

   class Team(Base, TenantMixin):
       __tablename__ = "teams"
       name: Mapped[str] = mapped_column(String, nullable=False)
       department_id: Mapped[PgUUID | None] = mapped_column(ForeignKey("departments.id"), nullable=True)

   class TeamMember(Base, TenantMixin):   # org_id denormalized so RLS is a column compare, not a subquery
       __tablename__ = "team_members"
       __table_args__ = (UniqueConstraint("team_id", "user_id", name="pk_team_members"),)
       team_id: Mapped[PgUUID] = mapped_column(ForeignKey("teams.id"), primary_key=True)
       user_id: Mapped[PgUUID] = mapped_column(ForeignKey("users.id"), primary_key=True)
       role: Mapped[str | None] = mapped_column(String, nullable=True)
   ```

   Extend `users` (in `models/user.py`) with `role: Mapped[str]`, `skills: Mapped[list[str]] = mapped_column(ARRAY(String), default=list)`, `bio: Mapped[str | None]`, `avatar_url: Mapped[str | None]` — only if Plan 1-2 did not already add them.

2. Generate + write the Alembic migration. RLS is `FORCE` on every new table, `org_id` join-denormalized on `team_members`:
   ```bash
   cd mytheus-backend
   poetry run alembic revision -m "p5_org_context_role_skills_teams_departments"
   ```
   Migration body (outline) — `add_column` guards for `users`, `create_table` for the three new tables, then per-table RLS:
   ```sql
   ALTER TABLE users  ADD COLUMN IF NOT EXISTS role TEXT NOT NULL DEFAULT 'member';
   ALTER TABLE users  ADD COLUMN IF NOT EXISTS skills TEXT[] NOT NULL DEFAULT '{}';
   ALTER TABLE users  ADD COLUMN IF NOT EXISTS bio TEXT;
   ALTER TABLE users  ADD COLUMN IF NOT EXISTS avatar_url TEXT;
   -- departments, teams, team_members created with id uuid default gen_random_uuid(), org_id uuid not null
   ALTER TABLE departments  ENABLE ROW LEVEL SECURITY; ALTER TABLE departments  FORCE ROW LEVEL SECURITY;
   CREATE POLICY tenant_isolation ON departments
     USING (org_id = current_setting('app.current_org')::uuid)
     WITH CHECK (org_id = current_setting('app.current_org')::uuid);
   -- repeat for teams, team_members
   CREATE INDEX departments_parent_idx ON departments(parent_id);
   CREATE INDEX team_members_user_idx  ON team_members(user_id);
   ```
   ```bash
   poetry run alembic upgrade head
   ```

3. Add CRUD routes (the `identity-tenant-service` stub currently exposes none). Mount under the canon-owned prefixes (`/orgs`, plus org-chart reads). All routes set `app.current_org` transaction-locally via the tenancy middleware before any query.
   `routes/org.py`: `GET /orgs/{org_id}/departments`, `GET /orgs/{org_id}/teams`, `GET /orgs/{org_id}/teams/{team_id}/members`, `PATCH /users/{user_id}` (role/skills/bio).

**Canonical names:** `users`, `teams`, `team_members`, `departments`, `app.current_org`, `org_id`.

**Acceptance checks:**
- `alembic upgrade head` then `alembic downgrade -1` round-trips clean.
- `SELECT relrowsecurity, relforcerowsecurity FROM pg_class WHERE relname IN ('teams','team_members','departments')` returns `t,t` for all three.
- A query under `SET LOCAL app.current_org = '<orgA>'` cannot read orgB rows (returns 0 rows), proving RLS `FORCE`.
- `PATCH /users/{id}` updating `skills` round-trips an array.

---

#### 3.2 `identity-tenant-service` — the role→topics taxonomy (version-controlled)

**Repo / folder:** `mytheus-backend/services/identity-tenant-service/taxonomy/`.

**Steps**

1. Author the taxonomy as a **git-versioned** artifact (reviewable in PR, pinned by version). `taxonomy/role_topics.v1.yaml`:
   ```yaml
   version: "v1"
   roles:
     engineer:        [architecture, api-design, infra, incidents, code-review, performance]
     designer:        [ux, design-system, prototypes, research, accessibility]
     product_manager: [roadmap, pricing, prioritization, metrics, stakeholders]
     sales:           [pipeline, pricing, contracts, renewals, objections]
     finance:         [budget, forecast, billing, runway]
     member:          [general]
   ```
2. Loader + endpoint. `taxonomy/loader.py` loads the pinned version (env `ROLE_TOPICS_TAXONOMY_VERSION=v1`), validates with Pydantic v2, and exposes `topics_for(role: str) -> list[str]`. Route `GET /taxonomy/role-topics?version=v1` returns the map (cacheable; immutable per version).
3. Seed `Topic` nodes into `nexus_knowledge` per org for each distinct taxonomy topic so the graph arm can later traverse `RELATES_TO` (edges still not written this band):
   ```cypher
   MERGE (t:Topic {name: $name, org_id: $org_id})
   ```
   Run as an idempotent provisioning step in the seeder (3.4) and at org provisioning.

**Canonical names:** `Topic` (AGE node), `nexus_knowledge`, `org_id`.

**Acceptance checks:**
- `GET /taxonomy/role-topics?version=v1` returns the full map; an unknown version 404s (no silent fallback).
- `topics_for("engineer")` is deterministic and matches the YAML.
- For a seeded org, `MATCH (t:Topic {org_id:$org}) RETURN count(t)` equals the number of distinct topics.

---

#### 3.3 `identity-tenant-service` — `get_user_context()` LIVE with Redis cache

**Repo / folder:** `mytheus-backend/services/identity-tenant-service/context/`.

`get_user_context()` is the source-of-record for role/skills/team/recent-meetings/topics that `PersonalListenerAgent` and `rag-service` query expansion read heavily (later bands). It is cached at `nexus:{org_id}:user_ctx:{user_id}` with a **5-min TTL**; the cache is an optimization, not a dependency (miss → recompute from Postgres).

**Steps**

1. Implement the resolver. `context/service.py`:
   ```python
   from libs.shared.cache import redis_key, get_redis     # key builder raises if org_id absent
   from libs.shared.tenancy import current_org
   import json, hashlib

   CTX_TTL = 300  # 5 minutes (canon)

   async def get_user_context(session, org_id, user_id) -> dict:
       key = redis_key(org_id, "user_ctx", str(user_id))   # nexus:{org_id}:user_ctx:{user_id}
       r = get_redis()
       cached = await r.get(key)
       if cached:
           return json.loads(cached)
       ctx = await _compute(session, org_id, user_id)       # SET LOCAL app.current_org first
       await r.set(key, json.dumps(ctx), ex=CTX_TTL)
       return ctx

   async def _compute(session, org_id, user_id) -> dict:
       u = await load_user(session, user_id)                # role, skills, bio
       teams = await load_teams(session, user_id)           # via team_members
       dept = await load_department(session, user_id)
       recent = await load_recent_meetings(session, user_id, limit=10)  # meeting_participants ⨝ meetings
       topics = topics_for(u.role)                          # from 3.2 taxonomy
       return {
           "user_id": str(user_id), "org_id": str(org_id),
           "role": u.role, "skills": u.skills, "team": teams, "department": dept,
           "recent_meetings": recent, "topics": topics,
       }
   ```
2. Expose `GET /context/{user_id}` (canon §6). It resolves `org_id` from the request context (Clerk JWT, never the path).
3. Cache invalidation: on `PATCH /users/{id}` (role/skills) and on `team_members` changes, `DEL nexus:{org_id}:user_ctx:{user_id}`.
4. Expose the same resolver over the **gRPC** mesh for the AI tier (`mytheus-intelligence` calls `get_user_context` over gRPC, `org_id` in metadata). Add `proto/identity.proto` `GetUserContext(GetUserContextRequest) returns (UserContext)` and implement the gRPC server (this is the first gRPC server in `identity-tenant-service`; the stub currently has none).

**Canonical names:** `get_user_context`, `nexus:{org_id}:user_ctx:{user_id}`, `meeting_participants`, `team_members`, `departments`.

**Acceptance checks:**
- First call to `GET /context/{user_id}` is a cache miss + Postgres compute; second within 5 min returns identical JSON from Redis (assert via a Redis `MONITOR` or a hit counter).
- TTL on the key is ≤ 300 s (`TTL nexus:{org}:user_ctx:{user}`).
- `PATCH /users/{id}` changing `role` evicts the key; the next context reflects the new role + new `topics`.
- Cross-org isolation: a JWT for orgB cannot fetch orgA's user context (404/empty, never a leak).
- gRPC `GetUserContext` with `org_id` in metadata returns the same payload as REST.

---

#### 3.4 `identity-tenant-service` — the seeder (2 orgs × 10 users × 5 meetings)

**Repo / folder:** `mytheus-backend/services/identity-tenant-service/seed/`, Make target in `mytheus-backend/Makefile`.

**Steps**

1. Write `seed/seed.py`: idempotent (re-runnable), creates **2 orgs × 10 users × 5 meetings** with realistic `role`/`skills`, departments, teams, `team_members`, `meeting_participants`, and a handful of `transcript_chunks` per meeting so downstream P6 embedding has input. Each org gets its `Topic` nodes (3.2) and its Qdrant collections are provisioned lazily in P6.
   ```python
   ORGS = ["acme", "globex"]
   ROLES = ["engineer","designer","product_manager","sales","finance",
            "engineer","designer","product_manager","member","member"]
   # for org in ORGS: create org → 10 users (Clerk-synced shells) → depts/teams →
   #   5 meetings each → participants → seed transcript_chunks (is_final=True)
   ```
2. Add the Make target:
   ```makefile
   seed:  ## Seed 2 orgs x 10 users x 5 meetings
   	poetry run python -m identity_tenant_service.seed.seed
   ```
3. Guard: the seeder runs the elevated migration role, NOT the request-path role, and asserts `app.current_org` is set per org it writes.

**Canonical names:** `organizations`, `users`, `teams`, `team_members`, `departments`, `meetings`, `meeting_participants`, `transcript_chunks`, `Topic`.

**Acceptance checks:**
- `make seed` is idempotent: a second run creates no duplicates (asserts counts: 2 orgs, 20 users, 10 meetings).
- `get_user_context` for any seeded user returns a populated `role`, `skills`, `team`, `recent_meetings`, `topics`.
- Each org's `Topic` node count matches the taxonomy.

---

#### 3.5 `web-app` + `bff` — org-chart UI and person profile

**Repo / folder:** `mytheus-ui/apps/web-app/` (Next.js 14 App Router), `mytheus-backend/services/bff/`.

**Steps**

1. `bff` aggregation endpoints (screen-shaped, read-through `nexus:{org_id}:…` Redis, short TTL): `GET /bff/org/chart` (departments → teams → members tree) and `GET /bff/people/{user_id}` (wraps `get_user_context` + profile). `bff` fans out to `identity-tenant-service`; it owns no domain data.
2. `web-app` routes:
   - `app/(dashboard)/org/page.tsx` — RSC that streams the org chart from `GET /bff/org/chart`.
   - `app/(dashboard)/people/[id]/page.tsx` — person profile rendering role, skills (as chips), team, department, recent meetings, taxonomy topics.
   ```bash
   cd mytheus-ui/apps/web-app
   # routes are added under app/; data via server components calling bff with the Clerk session
   ```
3. Wire the Clerk session token into the `bff` call (server-side fetch with the forwarded `Authorization`); `api-gateway` extracts `org_id` once at the edge.

**Canonical names:** `bff`, `web-app`, `get_user_context`.

**Acceptance checks:**
- `/org` renders the seeded department→team→member tree for the signed-in org only.
- `/people/{id}` renders `get_user_context` fields; skills appear as chips; recent meetings link to meeting pages.
- A user from orgB signing in sees only globex's chart (tenant isolation through the edge `org_id`).

---

### PHASE P6 — AI Infrastructure (W15-17)

P6 precondition chain: event backbone GA (3.6) → embedding-service + Qdrant + the first at-least-once consumer (3.7) → reranker (3.8) → ingestion produces `document-chunks` into the same consumer (3.9) → search FTS arm (3.10). Build 3.6 first — nothing else can be a real consumer until topics/schemas/outbox/inbox/DLQs exist.

---

#### 3.6 Event backbone GA — Redpanda + Avro + Schema Registry + outbox/inbox/DLQ

**Repo / folder:** `mytheus-events/schemas/`, `mytheus-events/provisioning/`, `mytheus-events/src/nexus_events/`; consumed by `mytheus-backend/libs/shared/` and `mytheus-intelligence/libs/shared/`.

`mytheus-events` is currently DOCUMENTATION ONLY — scaffold it. It owns the frozen Avro schemas, the Schema Registry registration + CI compatibility gate, the topic/DLQ provisioning scripts, and the shared `nexus_events` producer/consumer library (the **only** sanctioned way to touch the bus).

**Steps**

1. Scaffold the package:
   ```bash
   cd mytheus-events
   poetry init -n && poetry add confluent-kafka fastavro pydantic
   mkdir -p schemas provisioning src/nexus_events
   ```
2. Author the Avro schemas, each opening with `EventEnvelopeHeader`. Write `schemas/_envelope.avsc` (the frozen header: `event_id`, `idempotency_key`, `org_id`, `schema_version`, `producer`, `occurred_at`, `correlation_id`, `causation_id`), then the 7 FROZEN value schemas plus net-new `DocumentChunk` and `EmbeddingEvent`:
   ```jsonc
   // schemas/document_chunk.avsc  — subject document-chunks-value, BACKWARD
   { "type":"record","name":"DocumentChunk","namespace":"io.nexus.events",
     "fields":[
       {"name":"header","type":"io.nexus.events.EventEnvelopeHeader"},
       {"name":"document_id","type":{"type":"string","logicalType":"uuid"}},
       {"name":"element_id","type":"string"},
       {"name":"element_type","type":{"type":"enum","name":"ElementType",
          "symbols":["Title","NarrativeText","Table","ListItem"]}},
       {"name":"text","type":"string"},
       {"name":"position","type":"int"},
       {"name":"meeting_id","type":["null",{"type":"string","logicalType":"uuid"}],"default":null}
     ]}
   ```
   ```jsonc
   // schemas/embedding_event.avsc — subject embedding-events-value (upsert acks for lag)
   { "type":"record","name":"EmbeddingEvent","namespace":"io.nexus.events",
     "fields":[
       {"name":"header","type":"io.nexus.events.EventEnvelopeHeader"},
       {"name":"collection","type":"string"},
       {"name":"point_count","type":"int"},
       {"name":"source_topic","type":"string"},
       {"name":"batch_latency_ms","type":"int"}
     ]}
   ```
3. Register subjects with **BACKWARD** compatibility and gate compatibility in CI:
   ```bash
   poetry run python provisioning/register_schemas.py --registry "$SCHEMA_REGISTRY_URL" --compat BACKWARD
   # CI: check-compatibility against every {topic}-value before any producer image builds; incompatible PR fails
   ```
4. Provision **all 21 topics + 21 DLQs** with the canon partition counts and retention (`07-event-architecture.md §4.2`). `provisioning/topics.sh`:
   ```bash
   # high-volume meeting
   rpk topic create transcript-chunks  -p 180 -c retention.ms=604800000     # 7d
   rpk topic create agent-results      -p 120 -c retention.ms=259200000     # 3d
   rpk topic create agent-stream       -p 120 -c retention.ms=3600000       # 1h
   rpk topic create document-chunks    -p 24  -c retention.ms=604800000     # 7d
   rpk topic create embedding-events   -p 12  -c retention.ms=86400000      # 1d
   rpk topic create decisions          -p 24  -c retention.ms=2592000000    # 30d
   rpk topic create actions            -p 24  -c retention.ms=2592000000    # 30d
   # ... usage-events(48,7d), notifications(24,3d), agent-requests(12,1d), agent-responses(12,1d),
   #     meeting-events(24,7d), meeting-room-events(24,3d), integration-events(12,7d),
   #     billing-events(12,30d), graph-events(6,7d), presence-events(12,1h),
   #     notification-delivery-events(6,30d), user-sync-events(6,7d)
   rpk topic create tenant-events -p 6 -c cleanup.policy=compact            # latest state per org
   rpk topic create admin-events  -p 6 -c cleanup.policy=compact
   # one DLQ per topic, 14d retention, single partition
   for t in transcript-chunks document-chunks agent-results agent-stream embedding-events \
            decisions actions agent-requests agent-responses notifications tenant-events \
            meeting-events meeting-room-events integration-events usage-events billing-events \
            graph-events presence-events notification-delivery-events user-sync-events admin-events; do
     rpk topic create "$t.dlq" -p 1 -c retention.ms=1209600000
   done
   ```
5. Implement the producer/relay/consumer library in `src/nexus_events/`:
   - `producer.py` — `NexusProducer` (confluent-kafka-python + `AvroSerializer`, `acks=all`, `enable.idempotence=True`, `compression.type=zstd`), keyed by `org_id`; **only the relay calls `produce()`**.
   - `relay.py` — the outbox relay loop: `FOR UPDATE SKIP LOCKED` over `outbox WHERE published_at IS NULL`, produce, `mark_published`. Re-publish on crash (at-least-once).
   - `consumer.py` — `BaseConsumer` (`AvroDeserializer`, manual offset commit **after** effect, `isolation.level=read_committed`), the §6.2 dedupe gate (`nexus:{org_id}:dedupe:{topic}:{key}` NX EX 86400 → inbox PK → apply, all in one txn), and DLQ routing (5-attempt exp backoff + jitter → `DlqEnvelope`).
6. Land the `outbox` / `inbox` DDL as Alembic migrations in each owning repo (`mytheus-backend`, and `mytheus-intelligence` for `ingestion-service`/`search-service`):
   ```sql
   CREATE TABLE outbox (id BIGSERIAL PRIMARY KEY, org_id UUID NOT NULL, topic TEXT NOT NULL,
     partition_key TEXT NOT NULL, idempotency_key TEXT NOT NULL, schema_subject TEXT NOT NULL,
     payload JSONB NOT NULL, created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
     published_at TIMESTAMPTZ NULL, attempts INT NOT NULL DEFAULT 0);
   CREATE INDEX outbox_unpublished_idx ON outbox (created_at) WHERE published_at IS NULL;
   CREATE UNIQUE INDEX outbox_idem_uq ON outbox (topic, idempotency_key);
   CREATE TABLE inbox (org_id UUID NOT NULL, consumer_group TEXT NOT NULL, idempotency_key TEXT NOT NULL,
     processed_at TIMESTAMPTZ NOT NULL DEFAULT now(), PRIMARY KEY (consumer_group, org_id, idempotency_key));
   ```

**Canonical names:** all 21 topics + `<topic>.dlq`, `EventEnvelopeHeader`, `DocumentChunk`, `EmbeddingEvent`, `outbox`, `inbox`, `nexus:{org_id}:dedupe:{topic}:{idempotency_key}`.

**Acceptance checks:**
- `rpk topic list` shows all 21 topics + 21 DLQs with the partition counts/retention above; `tenant-events` and `admin-events` are `cleanup.policy=compact`.
- Schema Registry lists all subjects as `{topic}-value`, compatibility `BACKWARD`; the CI gate fails a PR that adds a required field with no default.
- The relay publishes an enqueued `outbox` row exactly once to the bus; a forced crash between `produce` and `mark_published` re-publishes and the consumer dedupes it (one applied effect).
- DLQ: a deliberately poisoned message lands in `<topic>.dlq` as a `DlqEnvelope` with `raw_value_b64`; the partition keeps moving.

---

#### 3.7 `embedding-service` — BGE-small-en on Ray Serve (GPU), the first at-least-once consumer, only writer of Qdrant

**Repo / folder:** `mytheus-intelligence/services/embedding-service/`, `mytheus-intelligence/libs/shared/`, `mytheus-intelligence/proto/`. (`mytheus-intelligence` is DOCUMENTATION ONLY — scaffold the Python packages per its CLAUDE.md layout.)

**Steps**

1. Scaffold:
   ```bash
   cd mytheus-intelligence
   mkdir -p services/embedding-service/embedding_service libs/shared proto
   python -m venv .venv && source .venv/bin/activate
   pip install -e "services/embedding-service[dev]"   # ray[serve], FlagEmbedding/sentence-transformers, qdrant-client, confluent-kafka, grpcio
   ```
2. The Ray Serve deployment hosts **BGE-small-en, 384-dim** with dynamic micro-batching (Ray Serve, NOT vLLM — vLLM is reserved for larger LLM paths). Two separate Ray Serve deployments share the model but separate the lanes so a query-embed burst never starves ingest. `embedding_service/serve.py`:
   ```python
   import ray
   from ray import serve
   from FlagEmbedding import FlagModel

   @serve.deployment(num_replicas=2, ray_actor_options={"num_gpus": 1},
                     max_ongoing_requests=64)
   class BGEEmbedder:
       def __init__(self):
           self._m = FlagModel("BAAI/bge-small-en-v1.5", use_fp16=True)  # baked into the CUDA image
       @serve.batch(max_batch_size=32, batch_wait_timeout_s=0.05)        # batch 32, sub-100ms
       async def embed(self, texts: list[str]) -> list[list[float]]:
           return self._m.encode(texts).tolist()                         # 384-dim
   ```
   Bake weights into the CUDA image at build time (pod start is a *load*, not a *fetch*). CPU fallback for dev (`EMBEDDING_GPU=0`).
3. The Qdrant resolution chokepoint + per-tenant collection provisioning (defense-in-depth: collection boundary AND `org_id` payload filter). `libs/shared/qdrant.py`:
   ```python
   from qdrant_client import QdrantClient
   from qdrant_client.models import (Distance, VectorParams, ScalarQuantization,
       ScalarQuantizationConfig, ScalarType, Filter, FieldCondition, MatchValue, PointStruct)

   def physical(logical: str, org_id) -> str:           # NEVER client-supplied
       return f"{logical}__{org_id.hex}"

   def ensure_collection(client: QdrantClient, name: str):
       if not client.collection_exists(name):
           client.create_collection(name,
               vectors_config=VectorParams(size=384, distance=Distance.COSINE),
               quantization_config=ScalarQuantization(
                   scalar=ScalarQuantizationConfig(type=ScalarType.INT8, always_ram=True)),
               on_disk_payload=True)

   def upsert(client, logical, org_id, points: list[PointStruct]):
       name = physical(logical, org_id)                  # collection boundary
       ensure_collection(client, name)
       client.upsert(name, points)                        # each payload carries org_id (backstop)
   ```
   (See DECISION-REQUIRED §8.2 on per-tenant collections vs a shared collection.)
4. The **embed-on-ingest consumer** — the canonical `BaseConsumer` subclass and the band's flagship at-least-once consumer. Subscribes to `transcript-chunks` AND `document-chunks`, splits (LangChain `RecursiveCharacterTextSplitter` 600/60), embeds at batch 32, upserts to the per-tenant collection, and emits an `embedding-events` ack. `embedding_service/consumer.py`:
   ```python
   class EmbeddingConsumer(BaseConsumer):
       group = "embedding-service.embed-on-ingest"
       topics = ["transcript-chunks", "document-chunks"]

       async def apply(self, evt):                        # dedupe + inbox handled by BaseConsumer
           org_id = evt.header.org_id
           pieces = split_600_60(evt.text)
           vectors = await embedder.embed.remote(pieces)
           if evt.topic == "transcript-chunks":
               logical, payload = "transcript_chunks", {
                   "org_id": str(org_id), "meeting_id": str(evt.meeting_id),
                   "chunk_id": str(evt.chunk_id), "speaker": evt.speaker_id,
                   "start_ms": evt.start_ms, "is_final": evt.is_final}
           else:
               logical, payload = "document_chunks", {
                   "org_id": str(org_id), "document_id": str(evt.document_id),
                   "element_id": evt.element_id, "element_type": evt.element_type}
           points = [PointStruct(id=pid(evt, i), vector=v, payload=payload)
                     for i, v in enumerate(vectors)]
           upsert(qdrant, logical, org_id, points)
           await self.emit_outbox("embedding-events",
               idem=f"emb:{logical}:{payload.get('chunk_id', payload.get('element_id'))}",
               payload={"collection": physical(logical, org_id), "point_count": len(points),
                        "source_topic": evt.topic})
   ```
   Idempotency keys: `chunk:{chunk_id}` (transcripts), `docchunk:{document_id}:{element_id}` (documents) — stable across redelivery; the Qdrant point id is derived deterministically so the upsert is idempotent on redelivery.
5. The **query-embed** path (`POST /embed`, internal gRPC only): 1–3 query variants, cached at `nexus:{org_id}:qembed:{query_hash}` (~10-min TTL). Separate Ray Serve deployment from the ingest lane. (Consumed later by `rag-service`; built here as the contract.)
6. KEDA/autoscaler signal recorded as config: scale on embed queue depth + GPU utilization (deployment to Kubernetes is POST-W36; in the MVP modular monolith this runs as a worker process, still off the API tier).

**Canonical names:** `embedding-service`, `transcript_chunks__{org_id}`, `document_chunks__{org_id}`, `embedding-events`, `POST /embed`, `nexus:{org_id}:qembed:{query_hash}`, BGE-small-en 384-dim.

**Acceptance checks:**
- A `transcript-chunks` event produces a 384-dim int8 vector in `transcript_chunks__{org_id}` carrying `org_id`/`meeting_id`/`chunk_id`/`speaker` payload.
- Batch latency for a batch of 32 is **< 100 ms** (warm); consumer lag is **< 5 s** (alert at 30 s).
- Redelivering the same `transcript-chunks` event upserts the *same* point (no duplicate) — proven by stable point id + dedupe.
- A Qdrant search with the wrong `org_id` payload filter returns **0** results even against the right collection (defense-in-depth holds).
- `embedding-events` ack is emitted per processed message; `embedding-service` is the **only** writer touching Qdrant (no other service has Qdrant write creds).

---

#### 3.8 `reranker-service` — BGE-reranker-base cross-encoder (GPU), warm floor

**Repo / folder:** `mytheus-intelligence/services/reranker-service/`.

**Steps**

1. Scaffold + Ray Serve deployment hosting **BGE-reranker-base** (cross-encoder). `reranker_service/serve.py`:
   ```python
   @serve.deployment(num_replicas=1, ray_actor_options={"num_gpus": 1})  # warm floor >= 1/region
   class BGEReranker:
       def __init__(self):
           from FlagEmbedding import FlagReranker
           self._r = FlagReranker("BAAI/bge-reranker-base", use_fp16=True)
       async def rerank(self, query: str, passages: list[str], top_k: int = 10):
           scores = self._r.compute_score([[query, p] for p in passages])  # joint (query,passage)
           ranked = sorted(zip(passages, scores), key=lambda x: x[1], reverse=True)
           return ranked[:top_k]
   ```
2. Expose `POST /rerank` (internal gRPC only): top-50 → top-10. Owns no data. `org_id` passed for audit/rate accounting only.
3. **Warm-replica floor ≥ 1 per active region**, never scale to zero during business hours; autoscale up on rerank req/s + GPU util. SLO split: warm p95 < 500 ms, cold p95 < 1.5 s.

**Canonical names:** `reranker-service`, BGE-reranker-base, `POST /rerank`.

**Acceptance checks:**
- `POST /rerank` over 50 passages returns 10 scored passages, warm p95 **< 500 ms**.
- A scaled-to-zero region cold-starts under **1.5 s**; with the warm floor on, steady-state stays warm.
- The service holds no Postgres table and no Qdrant creds.

---

#### 3.9 `ingestion-service` — unstructured.io → `document_chunks` + `document-chunks` topic + `nexus-documents`

**Repo / folder:** `mytheus-intelligence/services/ingestion-service/`.

**Steps**

1. Scaffold + Alembic migrations for the two owned tables (this is one of the only two relational writers in `mytheus-intelligence`):
   ```sql
   CREATE TABLE document_chunks (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), org_id UUID NOT NULL,
     document_id UUID NOT NULL, element_id TEXT NOT NULL, element_type TEXT NOT NULL,
     text TEXT NOT NULL, position INT NOT NULL, created_at TIMESTAMPTZ NOT NULL DEFAULT now());
   CREATE TABLE document_elements (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), org_id UUID NOT NULL,
     document_id UUID NOT NULL, element_id TEXT NOT NULL, element_type TEXT NOT NULL, raw JSONB NOT NULL);
   -- plus ingestion job state table; RLS FORCE + tenant_isolation policy on all three
   ```
2. The parse pipeline. `ingestion_service/pipeline.py`:
   ```python
   from unstructured.partition.auto import partition
   from langchain_text_splitters import RecursiveCharacterTextSplitter
   from langchain_experimental.text_splitter import SemanticChunker

   async def ingest(org_id, document_id, blob_uri):
       elements = partition(file=fetch(blob_uri))          # PDF/DOCX/PPTX/HTML/MD/EML → typed elements
       splitter = RecursiveCharacterTextSplitter(chunk_size=600, chunk_overlap=60)
       for pos, el in enumerate(elements):
           el_type = map_type(el)                          # Title/NarrativeText/Table/ListItem
           text = redact_pii(el.text) if pii_enabled(org_id) else el.text   # tenant_settings.pii_redaction_enabled
           for j, piece in enumerate(splitter.split_text(text)):
               element_id = f"el_{pos:04d}_{j}"
               await write_document_chunk(org_id, document_id, element_id, el_type, piece, pos)
               await emit_outbox("document-chunks",
                   idem=f"docchunk:{document_id}:{element_id}",
                   payload=DocumentChunk(document_id, element_id, el_type, piece, pos))
   ```
   Narrative docs use `SemanticChunker`; tables/lists keep the typed element boundary.
3. Source blobs land in **`nexus-documents`** at `{org_id}/{document_id}/` (calendar attachments route to `nexus-attachments` at `{org_id}/{meeting_id}/`). Object writes are per-tenant-prefixed.
4. Surfaces: `POST /ingest` (file/url) and `GET /ingest/jobs/:id`. Consume `integration-events` (calendar attachment → ingest). The `document-chunks` events flow to the §3.7 embedding consumer.

**Canonical names:** `ingestion-service`, `document_chunks`, `document_elements`, `document-chunks`, `DocumentChunk`, `nexus-documents`, `nexus-attachments`, `tenant_settings.pii_redaction_enabled`.

**Acceptance checks:**
- `POST /ingest` of a sample PDF writes the blob to `nexus-documents/{org_id}/{document_id}/`, rows into `document_chunks` with `element_type` ∈ {Title,NarrativeText,Table,ListItem}, and emits one `document-chunks` event per chunk.
- `GET /ingest/jobs/:id` reports status; a re-run is idempotent (stable `element_id` → dedupe).
- With `pii_redaction_enabled=true` the emitted chunk text is redacted before the bus.
- The embedded result of an ingested doc appears in `document_chunks__{org_id}` (end-to-end ingestion → embedding).

---

#### 3.10 `search-service` — Postgres FTS (tsvector) arm of hybrid retrieval

**Repo / folder:** `mytheus-intelligence/services/search-service/`.

**Steps**

1. Scaffold; this service owns FTS **indexes**, not source rows (read models over `transcript_chunks` + `chat_messages`). Add a `tsvector` materialization migration:
   ```sql
   ALTER TABLE transcript_chunks ADD COLUMN IF NOT EXISTS tsv tsvector
     GENERATED ALWAYS AS (to_tsvector('english', text)) STORED;
   CREATE INDEX IF NOT EXISTS transcript_chunks_tsv_idx ON transcript_chunks USING gin(tsv);
   ALTER TABLE chat_messages ADD COLUMN IF NOT EXISTS tsv tsvector
     GENERATED ALWAYS AS (to_tsvector('english', content)) STORED;
   CREATE INDEX IF NOT EXISTS chat_messages_tsv_idx ON chat_messages USING gin(tsv);
   ```
2. Consume `transcript-chunks` to keep the index warm (the generated column auto-updates on insert; the consumer is a lag/observability hook, at-least-once).
3. Surfaces: `GET /search` (in-page transcript search) and the internal gRPC FTS arm for `rag-service` (built as the contract; `rag-service` is later). Every query runs **under RLS** (`USING (org_id = current_setting('app.current_org'))`).

**Canonical names:** `search-service`, `transcript_chunks`, `chat_messages`, `GET /search`.

**Acceptance checks:**
- `GET /search?q=...` returns org-scoped FTS hits ranked by `ts_rank`; an orgB JWT cannot see orgA transcripts (RLS).
- A new `transcript_chunks` row is searchable after insert (generated `tsv` + GIN index).
- The FTS arm returns a ranked list shaped for RRF fusion (rank-ordered passage ids).

---

## 4. Data stores stood up in this band

Citing the roadmap data-store provisioning table (`22-roadmap-and-phasing.md §3.1`):

| Store | What lands in this band | Tenancy mechanism |
|---|---|---|
| **PostgreSQL** (Supabase) | `teams`, `team_members`, `departments` (P5); `document_chunks`, `document_elements`, ingestion job state, `outbox`/`inbox` (P6); `transcript_chunks.tsv` + `chat_messages.tsv` FTS indexes (P6); `users` role/skills columns | RLS `FORCE` on `org_id`; `app.current_org` set transaction-locally; PgBouncer transaction pooling |
| **Qdrant Cloud** | Per-tenant collections `transcript_chunks__{org_id}` + `document_chunks__{org_id}` (and `decisions__{org_id}` provisioned-but-idle), 384-dim cosine, int8 scalar quantization, `on_disk_payload` | Per-tenant collection boundary + mandatory `org_id` payload filter (ADR-0006) |
| **Apache AGE** `nexus_knowledge` | `Topic` nodes seeded from the role→topics taxonomy (no edges yet) | `org_id` property on every node; `$org_id` on every Cypher template |
| **Redis** (Upstash) | `nexus:{org_id}:user_ctx:{user_id}` (5-min), `nexus:{org_id}:qembed:{query_hash}` (~10-min), `nexus:{org_id}:dedupe:{topic}:{idempotency_key}` (24h) | Key builder rejects unprefixed keys |
| **Object storage** | `nexus-documents` (`{org_id}/{document_id}/`), `nexus-attachments` (`{org_id}/{meeting_id}/`) | Per-tenant key prefix |
| **Redpanda + Schema Registry** | All 21 topics + 21 DLQs created; 7 frozen + `DocumentChunk` + `EmbeddingEvent` Avro subjects registered (BACKWARD) | Partitioned by `org_id` |

---

## 5. Events / topics introduced

| Topic | Avro subject | Producer(s) live this band | Consumer(s) live this band | DLQ |
|---|---|---|---|---|
| `transcript-chunks` | `transcript-chunks-value` (`TranscriptChunk`) | `transcription-service` (Plan 3-4) | **`embedding-service`** (first at-least-once consumer), `search-service` | `transcript-chunks.dlq` |
| `document-chunks` | `document-chunks-value` (`DocumentChunk`, net-new) | **`ingestion-service`** | **`embedding-service`** | `document-chunks.dlq` |
| `embedding-events` | `embedding-events-value` (`EmbeddingEvent`, net-new) | **`embedding-service`** (upsert acks) | lag/metrics (admin-ops dashboards / Prometheus) | `embedding-events.dlq` |
| `integration-events` | `integration-events-value` | `integration-service` (later) | **`ingestion-service`** (calendar attachment → ingest) | `integration-events.dlq` |
| `user-sync-events` / `tenant-events` | `*-value` | `identity-tenant-service` (Clerk inbox / lifecycle) | `identity-tenant-service` / downstream | paired DLQs |

All **21** topics + **21** DLQs are *created and registered* in §3.6 even though only the above are *exercised* live this band — so later bands wire consumers onto a backbone that already exists. Delivery is at-least-once + idempotent: transactional `outbox` (same txn as the state change) → relay → `NexusProducer`; consumers dedupe via `inbox` PK + `nexus:{org_id}:dedupe:*` (24h) → DLQ on poison (5-attempt exp backoff + jitter). Partition key = `org_id` (subject to §8.1 ratification for the composite meeting sub-key).

---

## 6. SaaS overlay / preconditions landing here

- **Tenant isolation defense-in-depth (the headline of this band).** Six layers fire: Postgres RLS `FORCE` + `app.current_org`; Qdrant per-tenant collection + `org_id` payload filter; AGE `$org_id`; Redpanda tenant-keyed partitions + consumer rehydrates `org_id` before any write; Redis `nexus:{org_id}:*`; object prefixes `{org_id}/`. The embedding consumer is the first place event-sourced writes hit Qdrant — it MUST re-assert `org_id` from the envelope header, never trust the partition key alone.
- **Dedicated inference (ADR-0008) is the substrate, not optional.** `embedding-service` + `reranker-service` are off the API tier with their own cost center from the first commit — even co-located in the MVP modular monolith they run as separate worker/Ray processes, never inside the request path. This is the precondition for R4 (cost) and R5 (latency) in later agent bands.
- **PII redaction shift-left.** Ingestion redacts before the chunk reaches the bus, gated by `tenant_settings.pii_redaction_enabled` — compliance lands at the ingest boundary, before any vector exists.
- **Per-tenant capacity attribution.** Qdrant collection-level metrics (point count, segment RAM) are tagged with `org_id` at emission (one-way hex name; never reverse the collection name) — the input to later usage metering and noisy-neighbor detection.
- **No K8s/Terraform/cells here.** MVP is the modular monolith on Railway/Render, single region, pooled + RLS. KEDA/GPU node pools/cells are POST-W36 (Plan 14); recorded as config intent only.

---

## 7. Mobile (mytheus-mobile) work in this band

Realign one screen onto the new org-context substrate; **no agent UI yet**.

**Repo / folder:** `mytheus-mobile/app/` (Expo/React Native).

**Steps**

1. Remove the mock for people data (`src/data/mock.ts`) for the person/profile path; keep the good UI components and design tokens.
2. Add Clerk auth to the mobile app (Clerk Expo SDK) so the session JWT carries `org_id`; wire the authenticated session into the REST client.
3. Build the **person profile** screens reading `get_user_context`-backed data from `bff` (REST):
   - `app/people/[id].tsx` — person profile rendering `role`, `skills` (chips reusing the existing chip component), team, department, recent meetings, taxonomy `topics`, from `GET /bff/people/{user_id}`.
   - Surface light **org context** (the user's department/team) on the profile header.
4. Point the REST client at the same `bff` endpoints the web-app uses (`GET /bff/people/{id}`); `api-gateway` extracts `org_id` at the edge so the mobile client never sends a tenant id.

**Canonical names:** `bff`, `get_user_context`, `nexus:{org_id}:user_ctx:{user_id}`.

**Acceptance checks:**
- The person profile renders live `get_user_context` data for a seeded user (no mock), identical fields to the web-app `/people/{id}`.
- An unauthenticated mobile session cannot fetch any profile (Clerk gate); a session for globex cannot fetch an acme profile.
- Design tokens / components are reused (no visual regression on the kept UI).

---

## 8. DECISION-REQUIRED callouts

> ⚠️ **DECISION-REQUIRED 8.1 — Ratify the `transcript-chunks` / `agent-stream` partition key BEFORE wiring the embedding + agent consumers.**
> The locked key for every topic is `org_id`. `07-event-architecture.md §4.1` proposes (NOT ratified) a composite `"{org_id}:{meeting_id}"` sub-key for the high-volume meeting topics (`transcript-chunks`, `agent-results`, `agent-stream`) to stop one enterprise tenant hashing onto a single partition, preserving ordering only **within a meeting** — the only ordering scope the embedding consumer and agents need.
> **Why it bites now:** §3.7's embedding consumer group and its ordering/replay guarantees are defined by this key; choosing it after the consumer ships forces a consumer-group + partition re-cut.
> **Options:** (a) keep pure `org_id` (canon-locked, simplest, hot-tenant risk); (b) ratify the composite `{org_id}:{meeting_id}` for the three meeting topics (spreads hot tenants, requires updating Tech Decisions §2.4 + Data Model §7 FIRST).
> **Who decides:** Canon owners (Tech Decisions + Data Model) + the events/streaming lead. **Blocks:** §3.7 embedding consumer wiring and all later agent consumers.

> ⚠️ **DECISION-REQUIRED 8.2 — Confirm Qdrant tenancy model = per-tenant collections (ADR-0006) vs a shared collection with `org_id` filter.**
> ADR-0006 locks **per-tenant collections** (`transcript_chunks__{org_id}`) + a mandatory `org_id` payload filter. The counter-force is collection-count overhead at the long tail (~9,000 collections at 3,000 active orgs × 3) vs a single shared collection with an `org_id` payload index (lowest count, but one missed filter leaks verbatim transcript text and RTBF becomes a slow `delete-by-filter`).
> **Why it bites now:** §3.7 `resolve_collection()` + the int8/HNSW config block are written against whichever boundary wins; reversing later is a full backfill.
> **Options:** (a) per-tenant collections for all (ADR-0006 as written); (b) per-tenant collections for enterprise/cell tenants + a shared `org_id`-filtered collection for the Free long tail (hybrid, reduces count, weakens long-tail isolation). 
> **Who decides:** Data architecture owner + security (cross-tenant-leak risk owner) + ADR-0006 owner. **Blocks:** §3.7 Qdrant provisioning + §3.9→§3.7 ingestion-to-vector path.

---

## 9. Risk checkpoints

- **R4 (LLM cost spiral) — substrate guard.** This band does not run agents, but it lands the off-API-tier inference split (ADR-0008) and per-tenant Qdrant that make per-tenant cost attribution possible later. Checkpoint: confirm `embedding-service`/`reranker-service` have an independent process + cost tag, never the request path.
- **R6 (agent-to-agent cross-tenant leak) — isolation rehearsal.** The embedding consumer is the first event-sourced writer; verify it re-asserts `org_id` from the envelope header and applies both the collection boundary and the payload filter. A missing `org_id` in a record must **DLQ**, never guess a tenant.
- **R8 (integration scope) — boundary respected.** `ingestion-service` consumes `integration-events` only for Calendar attachments; do not widen ingestion triggers beyond Calendar + direct `/ingest` uploads.
- **Lag/DLQ SLOs.** Embedding consumer lag **< 5 s** (alert 30 s); DLQ rate **< 0.1%**; duplicate-delivery post-dedupe **~0**. Wire these gauges to Prometheus this band so the backbone GA is observable from day one.

---

## 10. Definition of Done & end-of-phase demo  (a demo at the END of EVERY phase)

**P5 Definition of Done (W14):**
- Migrations for `teams`/`team_members`/`departments` + `users` role/skills applied with RLS `FORCE`; `make seed` is idempotent (2 orgs × 10 users × 5 meetings).
- `GET /context/{user_id}` (REST + gRPC) returns cached role/skills/team/recent-meetings/topics; key TTL ≤ 300 s; eviction on role/team change verified.
- `GET /taxonomy/role-topics?version=v1` serves the version-pinned taxonomy; `Topic` nodes seeded per org.
- web-app `/org` + `/people/{id}` and the mobile person profile render live `get_user_context` data; cross-org isolation proven.
- **P5 demo:** open the seeded org chart, click a person on web and mobile, show identical `get_user_context`-backed profile, change a role and watch the cache evict and topics update.

**P6 Definition of Done (W17):**
- All 21 topics + 21 DLQs created; 9 Avro subjects registered BACKWARD; CI compatibility gate live; outbox relay + inbox/dedupe + DLQ routing working.
- `embedding-service` embeds `transcript-chunks` + `document-chunks` at batch 32 sub-100 ms into per-tenant Qdrant collections (only writer); `embedding-events` acks flow; lag < 5 s.
- `reranker-service` `/rerank` top-50→top-10 warm p95 < 500 ms with a warm floor.
- `ingestion-service` `POST /ingest` → `nexus-documents` → `document_chunks` → `document-chunks` → embedded, end-to-end; PII redaction gated by `tenant_settings`.
- `search-service` `GET /search` returns org-scoped FTS hits under RLS.
- Both §8 DECISION-REQUIRED items have a recorded ratification (or an explicit deferral with the chosen default).
- **P6 demo:** upload a PDF, watch it traverse ingest → bucket → `document_chunks` → `document-chunks` → embedding consumer → `document_chunks__{org_id}`; run an FTS `/search` and a Qdrant vector search side by side returning org-scoped hits; force a poison message and show it land in the DLQ while the partition keeps moving; show the embedding-lag and DLQ-rate Grafana panels green.

---

## 11. Source docs  (authoring references used, by path)

- `/Users/sushan/Projects/Mytheus/mytheus-docs/04-service-catalog.md` — `identity-tenant-service`, `embedding-service`, `reranker-service`, `ingestion-service`, `search-service` ownership, surfaces, producer/consumer map, split waves.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/06-multi-tenancy.md` — RLS `FORCE` + `app.current_org`, per-tenant Qdrant + payload filter, Redis namespacing, context propagation, RTBF.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/08-ai-agent-architecture.md` — `get_user_context` consumers, `AgentState` context fields, taxonomy role/skills usage (substrate for later bands).
- `/Users/sushan/Projects/Mytheus/mytheus-docs/09-rag-and-inference-serving.md` — `embedding-service`/`reranker-service` runtime, embed-on-ingest/query-embed, warm floor, FTS arm, query-embed cache.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/07-event-architecture.md` — envelope header, subject/BACKWARD rules, partition counts/retention, outbox/inbox/dedupe/DLQ, embedding consumer, partition-key proposal (§8.1).
- `/Users/sushan/Projects/Mytheus/mytheus-docs/05-data-architecture.md` — physical store sizing, partitioning, Qdrant quantization/retention.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/_canon/02-data-model.md` — table/column names, Qdrant collection naming, Redis namespaces, object buckets, `Topic` node, cross-cutting data rules.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/adr/0006-vector-store-tenancy.md` — per-tenant collection decision, `resolve_collection()` chokepoint, int8/HNSW config, lifecycle-at-activation, alternatives (§8.2).
- `/Users/sushan/Projects/Mytheus/mytheus-docs/adr/0008-dedicated-inference-serving.md` — off-API-tier inference, Ray Serve vs vLLM, autoscaler signals, warm floor, split-wave-1.
- `/Users/sushan/Projects/Mytheus/mytheus-intelligence/system.md` — AI-tier statelessness, data ownership (embedding-service only writer of Qdrant), events in/out, gRPC `org_id` metadata, SLOs.
