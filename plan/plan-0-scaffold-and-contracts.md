# Plan 0 — Scaffold, Tooling & Event Contracts  (Scaffold)

> Make every Mytheus (NEXUS) repo buildable, testable and wired together — frozen Avro event contracts, real Alembic/SQLAlchemy DDL, mounted idempotency + outbox relay + gRPC, the Supabase→Clerk auth revert, and CI/CD onto Railway/Render — **before** any feature band starts.

---

## 0. Band at a glance

| Field | Value |
|---|---|
| **Band id / name** | Plan 0 — Scaffold, Tooling & Event Contracts |
| **Theme** | Scaffold |
| **Phase window** | Pre-P1 repo bootstrap (the foundation P1 and weeks W1–W2 build on) |
| **Goal** | Every repo compiles, lints and tests green; `mytheus-events` ships 7 frozen Avro schemas + 12 SaaS schemas + envelope + DLQ with Python/TS codegen and a BACKWARD-compat CI gate; `mytheus-backend` has its first Alembic migrations (outbox, inbox, organizations, users + P2 core-table stubs), a real SQLAlchemy 2.0 async model layer wired into `target_metadata`, the Idempotency-Key middleware mounted on every service, a transactional-outbox relay process, and a generated gRPC server; the **Supabase Auth drift is reverted to Clerk**; the AI repo (`mytheus-intelligence`), the UI (`mytheus-ui`), and the realtime tier (`mytheus-realtime`) are scaffolded and buildable; `mytheus-platform` ships minimal GitHub Actions CI + a Railway/Render modular-monolith deploy target. |
| **Repos & folders touched** | `mytheus-events` (new: `schemas/`, `subjects/`, `codegen/`, `contracts/`, `tests/`, `Makefile`); `mytheus-backend` (`migrations/`, `libs/shared/`, `services/*`, `proto/`, `.env.example`, `README.md`, `CLAUDE.md`); `mytheus-intelligence` (new code tree); `mytheus-ui` (new Next.js app); `mytheus-realtime` (new Node + Python skeletons); `mytheus-platform` (`.github/workflows/`, `railway.json`/`render.yaml`, `env/`); `mytheus-mobile` (Clerk SDK + `httpClient`, env base URL); `mytheus-docs/_canon/` (canon-location decision, no name changes) |
| **Demoable at end** | `make ci` green in every repo; a checked-in golden `TranscriptChunk` event round-trips through generated Python **and** TypeScript clients; `alembic upgrade head` creates `outbox`, `inbox`, `organizations`, `users` (+ stub core tables) with RLS forced; a `POST` with an `Idempotency-Key` replays from Redis; the outbox relay drains an `outbox` row to a local Redpanda topic; a Clerk-signed JWT verifies via JWKS through `libs/shared/auth.py`; `mytheus-ui` `next build` succeeds; `mytheus-intelligence` `ruff`+`mypy` clean; a Railway/Render deploy of the modular monolith answers `/healthz`. |

---

## 1. Entry preconditions  (what must be GREEN before starting)

This is the first band; the "prior bands" are the **current repo state**, which the assignment fixes. Before starting, confirm:

1. **Canon is frozen and authoritative.** `mytheus-docs/_canon/{01-service-catalog,02-data-model,03-event-taxonomy,05-capacity-math,06-tech-decisions}.md` exist and are the single source of names/numbers. This band introduces **no new canonical names**; it only realizes existing canon. (Per `mytheus-docs/CLAUDE.md`: canon-first, one-directional.)
2. **Resolved project decisions apply verbatim** (see the program brief): IDENTITY = **Clerk** (revert the Supabase drift); MOBILE = **realign** (no real wiring this band); MVP DEPLOY = **modular monolith on Railway/Render**, single region, pooled + RLS, **no Kubernetes**; CANON AUTHORITY = Service Catalog wins on service names (the 20 kebab-case names, never the coarse data-model labels `agent-service`/`graph-service`/`budget-service`).
3. **Local toolchain present:** `uv` (or Poetry), Python 3.11+, Node 20+, `npm`/`pnpm`, Docker + Docker Compose, `rpk` (Redpanda CLI), `make`, `git`, GitHub CLI `gh`.
4. **`mytheus-backend` baseline exists** (from CURRENT STATE): real `libs/shared/` (JWT verify currently Supabase, tenancy ContextVar + `SET LOCAL app.current_org`, outbox enqueue, inbox dedupe, EventEnvelope, Idempotency middleware **not mounted**, RFC7807 errors, OTel), `docker-compose.yml` (Postgres/Redis/Redpanda/Temporal), `Makefile`, `pyproject.toml`; 10 service `main.py` stubs that boot FastAPI with **no routes**; `proto/identity.proto`; **no models, no migrations, no tables, zero tests, no relay, no gRPC server**.
5. **`mytheus-events`, `mytheus-ui`, `mytheus-realtime`, `mytheus-intelligence`, `mytheus-platform` are documentation-only** (CLAUDE.md/README.md/system.md/.gitignore) — no code.
6. **DECISION-REQUIRED items in §8 are resolved or explicitly deferred** before scaffolding the affected repo. The three that gate this band: AI repo boundary (one repo vs split GPU), canon physical location, and confirmation of Railway/Render as the CI deploy target.

---

## 2. Canonical names introduced in this band

**No new canonical names are introduced.** Every name used here is reproduced verbatim from canon. The names this band first *realizes in code*:

- **Services (20, kebab-case, Service Catalog):** `api-gateway`, `web-app`, `bff`, `identity-tenant-service`, `core-api`, `meeting-service`, `realtime-gateway`, `transcription-service`, `ingestion-service`, `embedding-service`, `reranker-service`, `agent-runtime-service`, `rag-service`, `knowledge-graph-service`, `search-service`, `integration-service`, `workflow-service`, `billing-metering-service`, `notification-service`, `admin-ops-service`.
- **21 Redpanda topics:** `transcript-chunks`, `document-chunks`, `agent-results`, `decisions`, `actions`, `notifications`, `agent-requests`, `agent-responses`, `agent-stream`, `tenant-events`, `meeting-events`, `meeting-room-events`, `integration-events`, `usage-events`, `billing-events`, `graph-events`, `embedding-events`, `presence-events`, `notification-delivery-events`, `user-sync-events`, `admin-events` — each with a paired `<topic>.dlq`.
- **7 frozen Avro schemas:** `TranscriptChunk`, `AgentResult`, `Decision`, `Action`, `Notification`, `AgentRequest`, `AgentResponse`. **Plus** the shared `EventEnvelopeHeader`, `DlqEnvelope`, and the net-new (not-frozen) `DocumentChunk`, `AgentStreamDelta`, and the 12 SaaS payload schemas `TenantEvent`, `MeetingEvent`, `MeetingRoomEvent`, `IntegrationEvent`, `UsageEvent`, `BillingEvent`, `GraphEvent`, `EmbeddingEvent`, `PresenceEvent`, `NotificationDeliveryEvent`, `UserSyncEvent`, `AdminEvent`.
- **Tables first given DDL this band:** `outbox`, `inbox`, `organizations`, `users` (full); `teams`, `team_members`, `departments`, `meetings`, `meeting_participants`, `tenant_settings`, `data_residency` (stub DDL for P2). Namespace: Avro `io.nexus.events`; Redis dedupe `nexus:{org_id}:dedupe:{topic}:{idempotency_key}`; RLS GUC `app.current_org`.

---

## 3. Work breakdown (ORDERED, by phase then component)

> **Build order rationale.** The chain is: (3.1) `mytheus-events` is the contract every other repo imports, so it is first. (3.2) `mytheus-backend` migrations create the tables the shared libs already write to (outbox/inbox/users **have no DDL today**, so the libs cannot run), then the model layer wires `target_metadata`, then idempotency/relay/gRPC make the bus and edge real. (3.3) The Clerk reconciliation is interleaved into `mytheus-backend` because the shared `auth.py` is the seam the gateway and every service depend on — it must be correct before any protected route exists. (3.4–3.6) The remaining repos scaffold against the now-frozen contracts. (3.7) `mytheus-platform` CI wraps all of them. The canon-location decision (§8) gates how `mytheus-events`/`mytheus-intelligence`/`mytheus-realtime` enforce names in CI, so resolve it before 3.1.

---

### Phase 0A — `mytheus-events`: the frozen event-contract repo

**Repo / folder:** `mytheus-events/` (currently docs-only). Target layout (from its own `CLAUDE.md`):

```
mytheus-events/
├── schemas/        # schemas/<topic>/vN.avsc — EventEnvelopeHeader + topic payload
├── subjects/       # {topic}-value subject defs + BACKWARD compatibility config
├── codegen/        # Python (pydantic/dataclasses) + TypeScript generators
├── contracts/      # outbox/inbox/DLQ specs + shared EventEnvelopeHeader / DlqEnvelope
├── fixtures/       # golden sample events per topic
├── tests/          # schema-compat, contract, golden round-trip
├── clients/        # generated output: clients/python/, clients/typescript/
├── Makefile        # validate / check-compatibility / register / codegen / test
├── pyproject.toml  # tooling deps (fastavro, jsonschema, pytest)
└── package.json    # TS codegen + client build
```

#### Step 0A.1 — Initialize the repo skeleton

```bash
cd mytheus-events
mkdir -p schemas subjects codegen contracts fixtures tests clients/python clients/typescript docs
git checkout -b plan0/events-scaffold
```

Create `mytheus-events/pyproject.toml`:

```toml
[project]
name = "mytheus-events"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["fastavro>=1.9", "confluent-kafka[avro,schemaregistry]>=2.4", "pydantic>=2"]

[dependency-groups]
dev = ["pytest>=8", "ruff>=0.4", "datamodel-code-generator>=0.25"]

[tool.ruff]
line-length = 100
```

Create `mytheus-events/package.json` (TypeScript codegen):

```json
{
  "name": "@mytheus/events",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "codegen": "node codegen/ts_from_avsc.mjs",
    "build": "tsc -p clients/typescript/tsconfig.json",
    "test": "node --test tests/ts"
  },
  "devDependencies": { "typescript": "^5.5", "avsc": "^5.7" }
}
```

#### Step 0A.2 — Author the shared `EventEnvelopeHeader` and `DlqEnvelope`

Create `mytheus-events/schemas/_envelope/EventEnvelopeHeader.avsc` — **frozen field order, append-only with defaults (BACKWARD)**, verbatim from `07-event-architecture.md` §3.1:

```json
{
  "type": "record",
  "name": "EventEnvelopeHeader",
  "namespace": "io.nexus.events",
  "doc": "Shared header prepended to every NEXUS event value. Frozen field order; new fields append-only with defaults (BACKWARD).",
  "fields": [
    { "name": "event_id",       "type": { "type": "string", "logicalType": "uuid" } },
    { "name": "idempotency_key", "type": "string" },
    { "name": "org_id",          "type": { "type": "string", "logicalType": "uuid" } },
    { "name": "schema_version",  "type": "int", "default": 1 },
    { "name": "producer",        "type": "string" },
    { "name": "occurred_at",     "type": { "type": "long", "logicalType": "timestamp-micros" } },
    { "name": "correlation_id",  "type": ["null", "string"], "default": null },
    { "name": "causation_id",    "type": ["null", "string"], "default": null }
  ]
}
```

Create `mytheus-events/contracts/DlqEnvelope.avsc` (verbatim from `07-event-architecture.md` §7):

```json
{
  "type": "record", "name": "DlqEnvelope", "namespace": "io.nexus.events",
  "fields": [
    { "name": "original_topic",     "type": "string" },
    { "name": "original_partition", "type": "int" },
    { "name": "original_offset",    "type": "long" },
    { "name": "org_id",             "type": "string" },
    { "name": "idempotency_key",    "type": "string" },
    { "name": "consumer_group",     "type": "string" },
    { "name": "failure_class",      "type": { "type": "enum", "name": "FailureClass",
              "symbols": ["DESERIALIZE", "SCHEMA_INCOMPAT", "BUSINESS_INVALID", "MAX_RETRIES"] } },
    { "name": "error",              "type": "string" },
    { "name": "attempts",           "type": "int" },
    { "name": "first_seen_at",      "type": { "type": "long", "logicalType": "timestamp-micros" } },
    { "name": "raw_value_b64",      "type": "string" }
  ]
}
```

> **Acceptance:** `python -c "from fastavro.schema import load_schema; load_schema('schemas/_envelope/EventEnvelopeHeader.avsc')"` parses without error; the 8 header fields appear in the frozen order `event_id, idempotency_key, org_id, schema_version, producer, occurred_at, correlation_id, causation_id`.

#### Step 0A.3 — Author the 7 FROZEN Avro value schemas

Each topic value composes `io.nexus.events.EventEnvelopeHeader` as field `header`, then a topic-specific payload, matching the Postgres rows in `_canon/02-data-model.md`. Author `schemas/<topic>/v1.avsc` for the seven frozen schemas. Reference `transcript-chunks` (verbatim from `07-event-architecture.md` §3.1):

`mytheus-events/schemas/transcript-chunks/v1.avsc`:

```json
{
  "type": "record", "name": "TranscriptChunk", "namespace": "io.nexus.events",
  "doc": "transcript-chunks value. Subject: transcript-chunks-value. Compat: BACKWARD.",
  "fields": [
    { "name": "header", "type": "io.nexus.events.EventEnvelopeHeader" },
    { "name": "meeting_id",  "type": { "type": "string", "logicalType": "uuid" } },
    { "name": "chunk_id",    "type": { "type": "string", "logicalType": "uuid" } },
    { "name": "speaker_id",  "type": "string" },
    { "name": "start_ms",    "type": "int" },
    { "name": "end_ms",      "type": "int" },
    { "name": "text",        "type": "string" },
    { "name": "is_final",    "type": "boolean" },
    { "name": "confidence",  "type": "float" }
  ]
}
```

Author the remaining six frozen schemas with payloads drawn from `_canon/02-data-model.md` §2.1 and the agent contracts:

- `schemas/agent-results/v1.avsc` → `AgentResult`: `header`, `meeting_id`, `window_id` (string), `user_id` (uuid), `agent` (enum of the six agent class names), `kind` (enum `note`/`decision_candidate`/`action_candidate`/`suggestion`/`alert`), `relevance_score` (float), `confidence` (float), `content` (string), `payload_json` (string, free-form per kind), `source_chunk_ids` (array of uuid-string).
- `schemas/decisions/v1.avsc` → `Decision`: `header`, `decision_id` (uuid), `meeting_id` (uuid), `content`, `rationale`, `confidence` (float), `status` (enum `confirmed`), `confirmed_by` (uuid).
- `schemas/actions/v1.avsc` → `Action`: `header`, `action_id` (uuid), `meeting_id` (uuid), `content`, `assignee_id` (`["null","string"]` default null), `due_date` (`["null",{"type":"int","logicalType":"date"}]` default null), `confidence` (float), `status` (enum `confirmed`).
- `schemas/notifications/v1.avsc` → `Notification`: `header`, `source_event_id` (string), `channel` (enum `in_app`/`email`/`slack`), `user_id` (uuid), `priority` (int), `title`, `body`, `link` (`["null","string"]` default null).
- `schemas/agent-requests/v1.avsc` → `AgentRequest`: `header`, `requester_agent_id` (string), `responder_target` (string), `query` (string), `deadline` (`{type:long,logicalType:timestamp-micros}`), `correlation_id` (string). *(field names verbatim from canon)*
- `schemas/agent-responses/v1.avsc` → `AgentResponse`: `header`, `correlation_id`, `responder_agent_id`, `result` (string), `success` (boolean), `latency_ms` (int).

> **No-silent-commit guard reflected in schemas:** `Decision.status` and `Action.status` enums contain **only** `confirmed`. The producer is `core-api` on confirmation (per `_canon/03-event-taxonomy.md`); the schema makes a non-confirmed value unrepresentable.

> **Acceptance:** all 7 `.avsc` parse; `Decision`/`Action` status enum = `["confirmed"]`; every value record's first field is `header` typed `io.nexus.events.EventEnvelopeHeader`.

#### Step 0A.4 — Author the net-new schemas (`DocumentChunk`, `AgentStreamDelta`) + 12 SaaS schemas

These are **net-new, not frozen canon** (flagged in `07-event-architecture.md` §2.1 and Glossary §6). Author one `v1.avsc` per topic under `schemas/<topic>/v1.avsc`, each opening with `header`:

| Topic | Schema | Payload (beyond `header`) |
|---|---|---|
| `document-chunks` | `DocumentChunk` | `document_id` (uuid), `element_id` (string), `element_type` (enum Title/NarrativeText/Table/ListItem), `text`, `position` (int) |
| `agent-stream` | `AgentStreamDelta` | `meeting_id`, `user_id`, `window_id`, `seq` (int), `delta` (string), `done` (boolean) |
| `tenant-events` | `TenantEvent` | `event_type` (enum provisioned/suspended/deleted), `org_slug`, `plan_code` |
| `meeting-events` | `MeetingEvent` | `meeting_id`, `event_type` (enum created/updated/ended), `livekit_room` |
| `meeting-room-events` | `MeetingRoomEvent` | `meeting_id`, `room`, `event_type` (enum joined/left/recording_ready), `participant_id` (`["null","string"]`) |
| `integration-events` | `IntegrationEvent` | `provider` (enum google_calendar/slack/jira/linear), `event_type`, `external_id`, `payload_json` |
| `usage-events` | `UsageEvent` | `user_id` (`["null","string"]`), `event_type` (enum meeting_minute/agent_token/transcription_second/storage_gb), `quantity` (double), `source_id` (string) |
| `billing-events` | `BillingEvent` | `event_type` (enum quota_exceeded/dunning/plan_changed), `subscription_id` (`["null","string"]`), `detail_json` |
| `graph-events` | `GraphEvent` | `event_type` (enum edge_created/lineage_updated), `edge_label`, `from_id`, `to_id` |
| `embedding-events` | `EmbeddingEvent` | `collection` (string), `point_count` (int), `meeting_id` (`["null","string"]`) |
| `presence-events` | `PresenceEvent` | `meeting_id`, `user_id`, `event_type` (enum join/leave) |
| `notification-delivery-events` | `NotificationDeliveryEvent` | `notification_id`, `channel`, `status` (enum sent/failed/bounced), `detail` (`["null","string"]`) |
| `user-sync-events` | `UserSyncEvent` | `clerk_event_id` (string), `clerk_user_id`, `event_type` (enum user.created/user.updated/user.deleted), `email` (`["null","string"]`) |
| `admin-events` | `AdminEvent` | `event_type` (enum flag_changed/tenant_suspended/erasure_requested), `actor_id`, `target_type`, `target_id`, `detail_json` |

> **Acceptance:** there are exactly **21** topic schema folders under `schemas/` plus `_envelope/`; every `<topic>/v1.avsc` value record's `producer`/idempotency conventions match `mytheus-events/system.md`; `usage-events` payload carries `source_id` so the key `usage:{org_id}:{event_type}:{occurred_at}:{source_id}` is constructible.

#### Step 0A.5 — Subjects + BACKWARD compatibility config

Create one subject descriptor per topic in `subjects/<topic>-value.json`:

```json
{ "subject": "transcript-chunks-value", "schema": "schemas/transcript-chunks/v1.avsc", "compatibility": "BACKWARD" }
```

Generate all 21 with a helper, then create `subjects/compatibility.json` pinning every subject to `BACKWARD` (the registry-wide default). **One subject per topic, named `{topic}-value`** — no key schemas (the partition key is the raw `org_id` UUID string).

> **Acceptance:** 21 subject files exist, each `compatibility: "BACKWARD"`, subject name exactly `{topic}-value`.

#### Step 0A.6 — The Makefile (validate / check-compatibility / register / codegen / test)

`mytheus-events/Makefile`:

```makefile
.PHONY: validate check-compatibility register codegen test ci
SR_URL ?= http://localhost:8081

validate:               ## parse + lint every .avsc
	uv run python codegen/validate.py schemas

check-compatibility:    ## BACKWARD gate against registered {topic}-value subjects
	uv run python codegen/check_compat.py --registry $(SR_URL) --subjects subjects

register:               ## register new subject versions (monotonic)
	uv run python codegen/register.py --registry $(SR_URL) --subjects subjects

codegen:                ## generate Python + TypeScript clients
	uv run python codegen/py_from_avsc.py schemas clients/python
	npm run codegen

test:                   ## schema-compat + contract + golden round-trip
	uv run pytest tests -q
	npm test

ci: validate check-compatibility codegen test
```

- `codegen/validate.py`: walks `schemas/`, calls `fastavro.schema.load_schema` on each, asserts the first field is `header` and that header field order matches the frozen list; non-zero exit on any failure.
- `codegen/check_compat.py`: for each subject, POSTs the new schema to the Schema Registry `/compatibility/subjects/{subject}/versions/latest` endpoint under BACKWARD and fails on `is_compatible == false`. (If no prior version is registered, it passes — first registration.)
- `codegen/py_from_avsc.py`: emits `pydantic`/`dataclass` models into `clients/python/mytheus_events/` (one module per topic + `envelope.py`).
- `codegen/ts_from_avsc.mjs`: emits `.ts` types + `avsc` serializers into `clients/typescript/src/`.

> **Acceptance:** `make validate` exits 0; with a local Redpanda Schema Registry up (`docker compose up -d` from `mytheus-backend` provides one), `make register` then `make check-compatibility` exits 0; `make codegen` produces `clients/python/mytheus_events/transcript_chunks.py` and `clients/typescript/src/transcriptChunks.ts`.

#### Step 0A.7 — Golden fixtures + BACKWARD-compatibility contract tests

Create one golden sample per topic in `fixtures/<topic>.json` (a JSON value matching the schema, with a real `header`). Add `tests/test_golden_roundtrip.py`:

```python
import glob, json, fastavro, io
from fastavro.schema import load_schema

def _topic_of(path): return path.split("/")[1]

def test_every_topic_has_golden_and_roundtrips():
    schemas = sorted(glob.glob("schemas/*/v*.avsc"))
    assert len({_topic_of(s) for s in schemas}) == 21
    for s in schemas:
        topic = _topic_of(s)
        sch = load_schema(s)
        rec = json.load(open(f"fixtures/{topic}.json"))
        buf = io.BytesIO(); fastavro.schemaless_writer(buf, sch, rec)
        buf.seek(0); back = fastavro.schemaless_reader(buf, sch)
        assert back["header"]["org_id"] == rec["header"]["org_id"]
        assert "idempotency_key" in back["header"]
```

Add `tests/test_contract.py` asserting: every topic has a `{topic}-value` subject; every value record begins with `header`; `org_id` is mandatory (non-nullable) on every header; `Decision`/`Action` status enum == `["confirmed"]`. Add `tests/test_backward_compat.py` that takes each `v1.avsc`, constructs a synthetic `v2` adding one defaulted field, and asserts `fastavro` can read v1 bytes with the v2 reader schema (proves the additive-only discipline mechanically).

> **Acceptance:** `make test` green; the golden round-trip exercises all 21 topics through the generated Python client; a deliberately breaking change (remove `confidence` from `TranscriptChunk`) makes `make check-compatibility` fail — verify once, then revert.

---

### Phase 0B — `mytheus-backend`: migrations, model layer, idempotency, outbox relay, gRPC

> **Why before features:** the shared libs (`outbox.enqueue`, `inbox.record_consumed`, `auth`) already issue `INSERT INTO outbox/inbox` and read `public.users`, but **no DDL exists**, so nothing runs. This phase makes the libs' writes land in real tables and wires `target_metadata` so Alembic autogenerate works for every later band.

**Repo / folder:** `mytheus-backend/` (baseline exists).

#### Step 0B.1 — Author the SQLAlchemy 2.0 async model layer + `Base`/metadata

Create `mytheus-backend/libs/shared/db.py` — the single declarative base and the shared column mixins:

```python
from __future__ import annotations
import uuid
from datetime import datetime
from sqlalchemy import MetaData, text
from sqlalchemy.orm import DeclarativeBase, mapped_column, Mapped
from sqlalchemy.dialects.postgresql import UUID, TIMESTAMP

# Stable naming convention so Alembic autogenerate produces deterministic constraint names.
NAMING = {
    "ix": "ix_%(column_0_label)s", "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING)

class TenantMixin:
    org_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False, index=True)

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(TIMESTAMP(timezone=True), server_default=text("now()"))
    updated_at: Mapped[datetime] = mapped_column(TIMESTAMP(timezone=True), server_default=text("now()"))
    deleted_at: Mapped[datetime | None] = mapped_column(TIMESTAMP(timezone=True), nullable=True)
```

Add per-service `models.py` modules importing `Base`. This band authors the **full** models for the tables it migrates and **stub** models for the P2 core tables:

- `services/identity_tenant_service/models.py` — `Organization` (PK `id` = `org_id`, `slug`, `plan_id` nullable, timestamps), `User` (`id`, `org_id`, `clerk_user_id` **unique**, `email`, `role`, `skills` `ARRAY(Text)`, `bio`, `avatar_url`), and **stubs** `Team`, `TeamMember` (composite PK), `Department` (self-ref `parent_id`), `TenantSettings` (PK `org_id`), `DataResidency` (PK `org_id`).
- `services/core_api/models.py` — **stub** `Meeting`, `MeetingParticipant` (composite PK) for P2 (so foreign keys and dashboards have a target). Keep columns minimal but canon-named (`_canon/02-data-model.md` §2.1).
- `libs/shared/infra_models.py` — `Outbox`, `Inbox` (the infrastructure tables the libs write to).

`Outbox` model (matches `07-event-architecture.md` §6.1 and the shape `libs/shared/outbox.py` already inserts):

```python
class Outbox(Base):
    __tablename__ = "outbox"
    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    org_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    topic: Mapped[str] = mapped_column(Text, nullable=False)
    partition_key: Mapped[str] = mapped_column(Text, nullable=False)   # the org_id string
    idempotency_key: Mapped[str] = mapped_column(Text, nullable=False)
    schema_subject: Mapped[str] = mapped_column(Text, nullable=False)  # {topic}-value
    payload: Mapped[dict] = mapped_column(JSONB, nullable=False)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMP(timezone=True), server_default=text("now()"))
    published_at: Mapped[datetime | None] = mapped_column(TIMESTAMP(timezone=True), nullable=True)
    attempts: Mapped[int] = mapped_column(Integer, server_default=text("0"))
    __table_args__ = (UniqueConstraint("topic", "idempotency_key", name="outbox_idem_uq"),
                      Index("outbox_unpublished_idx", "created_at",
                            postgresql_where=text("published_at IS NULL")),)
```

`Inbox` model (matches `07-event-architecture.md` §6.2 — composite PK `(consumer_group, org_id, idempotency_key)`):

```python
class Inbox(Base):
    __tablename__ = "inbox"
    consumer_group: Mapped[str] = mapped_column(Text, primary_key=True)
    org_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True)
    idempotency_key: Mapped[str] = mapped_column(Text, primary_key=True)
    processed_at: Mapped[datetime] = mapped_column(TIMESTAMP(timezone=True), server_default=text("now()"))
```

> **Reconcile the lib drift:** `libs/shared/outbox.py` currently inserts `(id, org_id, topic, idempotency_key, payload, created_at)` with `id` a UUID, missing `partition_key`/`schema_subject`. Update `libs/shared/outbox.py` to insert the full canonical column set (`partition_key=str(org_id)`, `schema_subject=f"{topic}-value"`) and let `id` be `BIGSERIAL`. Update `libs/shared/inbox.py` `record_consumed` to write `(consumer_group, org_id, idempotency_key, processed_at)` matching the composite PK (today it writes `(org_id, topic, idempotency_key, consumed_at)` — wrong column names/PK).

> **Acceptance:** `python -c "from libs.shared.db import Base; import libs.shared.infra_models, services.identity_tenant_service.models, services.core_api.models; print(len(Base.metadata.tables))"` prints the table count without import errors; `outbox`/`inbox`/`organizations`/`users` are present in `Base.metadata.tables`.

#### Step 0B.2 — Wire `target_metadata` in Alembic `env.py`

Edit `mytheus-backend/migrations/env.py`: replace the `target_metadata = None` placeholder. Import every service's models so autogenerate sees all tables, then set `target_metadata = Base.metadata`:

```python
from libs.shared.db import Base
import libs.shared.infra_models            # noqa: F401  (outbox, inbox)
import services.identity_tenant_service.models   # noqa: F401
import services.core_api.models                  # noqa: F401
# (later bands append their service model imports here)
target_metadata = Base.metadata
```

Keep the existing sync-URL derivation (`DATABASE_URL.replace("+asyncpg","")`) for Alembic.

> **Acceptance:** `uv run alembic check` (or `alembic revision --autogenerate --sql`) no longer errors on `target_metadata is None`; autogenerate diffs against an empty DB list all model tables.

#### Step 0B.3 — Generate the FIRST migrations (outbox, inbox, organizations, users + P2 stubs)

Generate two ordered migrations so infra and identity are separable:

```bash
cd mytheus-backend
uv run alembic revision -m "0001 infra: outbox + inbox tables"
uv run alembic revision -m "0002 identity: organizations + users (+ P2 stubs) with RLS"
```

**Migration 0001 (`outbox`, `inbox`)** — `outbox` and `inbox` are RLS-scoped tenant tables per `07-event-architecture.md` §6.1 (they carry `org_id`). In `upgrade()`:

1. Create `outbox` (BIGSERIAL PK, columns per the model, `outbox_idem_uq`, partial index `outbox_unpublished_idx`).
2. Create `inbox` (composite PK).
3. Enable + FORCE RLS and the canonical policy on **both** (the policy from `_canon/02-data-model.md` §7 / `05-data-architecture.md` §3.1):

```python
def _rls(table):
    op.execute(f"ALTER TABLE {table} ENABLE ROW LEVEL SECURITY;")
    op.execute(f"ALTER TABLE {table} FORCE ROW LEVEL SECURITY;")
    op.execute(f"""CREATE POLICY tenant_isolation ON {table}
        USING (org_id = current_setting('app.current_org')::uuid)
        WITH CHECK (org_id = current_setting('app.current_org')::uuid);""")
```

**Migration 0002 (`organizations`, `users` + stubs)**:

1. `organizations` — tenant root, `id UUID PK DEFAULT gen_random_uuid()`, `name`, `slug` unique, `plan_id UUID NULL`, timestamps, `deleted_at`. **`organizations` is the tenant root, RLS keyed differently:** its policy is `USING (id = current_setting('app.current_org')::uuid)` (the row's own `id` is the `org_id`).
2. `users` — `id UUID PK`, `org_id UUID NOT NULL REFERENCES organizations(id)`, `clerk_user_id TEXT UNIQUE NOT NULL`, `email`, `role`, `skills TEXT[]`, `bio`, `avatar_url`, timestamps + `deleted_at`. RLS forced with the standard `org_id` policy.
3. **P2 stub tables** (`teams`, `team_members`, `departments`, `meetings`, `meeting_participants`, `tenant_settings`, `data_residency`) — create with canon columns and the standard RLS policy so the shared-lib writes, FKs, and the Clerk webhook upserts have real targets. Mark them as "stub: extended in P2" in the migration docstring.
4. Ensure `pgcrypto` for `gen_random_uuid()`: `op.execute("CREATE EXTENSION IF NOT EXISTS pgcrypto;")` at the top of 0001.

> **`SET LOCAL` / pooled-connection rule:** the app sets the org per transaction via `libs/shared/tenancy.set_rls(session, org_id)` which already issues `SELECT set_config('app.current_org', :org_id, true)` (transaction-local). RLS policies above bind to that GUC. (Per `mytheus-backend/system.md` §5: transaction-local is mandatory under PgBouncer.)

> **Acceptance:** against the docker-compose Postgres, `uv run alembic upgrade head` creates all tables; `\d+ outbox` shows `outbox_idem_uq` and the partial index; `SELECT relrowsecurity, relforcerowsecurity FROM pg_class WHERE relname IN ('outbox','inbox','organizations','users');` returns true/true for each; an integration test inserting two rows under different `app.current_org` values confirms tenant A cannot read tenant B (RLS fail-closed when the GUC is unset returns zero rows).

#### Step 0B.4 — Mount the Idempotency-Key middleware on every service

`libs/shared/idempotency.py` exists but is **NOT mounted**. Convert it to a Starlette `BaseHTTPMiddleware` (it currently is a bare `call_next` function) and mount it on all 10 services via a shared app factory. Create `libs/shared/app_factory.py`:

```python
from fastapi import FastAPI
from redis.asyncio import from_url
from libs.shared.errors import add_exception_handlers
from libs.shared.idempotency import IdempotencyMiddleware
from libs.shared.telemetry import instrument
import os

def create_app(service_name: str) -> FastAPI:
    app = FastAPI(title=service_name)
    redis = from_url(os.environ["REDIS_URL"], decode_responses=False)
    app.add_middleware(IdempotencyMiddleware, redis=redis)   # mutating POST/PUT/PATCH replay
    add_exception_handlers(app)
    instrument(app)                                          # OTel
    app.add_api_route("/healthz", lambda: {"status": "ok", "service": service_name})
    return app
```

Refactor `libs/shared/idempotency.py` into `class IdempotencyMiddleware(BaseHTTPMiddleware)` preserving the existing logic (skip non-mutating methods; require `Idempotency-Key`; cache `nexus:idem:{key}` for 24h; replay with `X-Idempotent-Replayed: true`). Update each `services/<svc>/main.py` stub to `app = create_app("<svc-name>")` instead of bare `FastAPI()`.

> **Acceptance:** every service's `/healthz` returns 200; a `POST` (to any later route, or a temporary echo route) with a fixed `Idempotency-Key` returns the cached body with header `X-Idempotent-Replayed: true` on the second call; a `POST` without the header is not cached.

#### Step 0B.5 — Build the transactional-outbox relay/publisher process

Create `libs/shared/relay.py` — the single sanctioned producer path (per `07-event-architecture.md` §8.1, `acks=all`, idempotent, Avro against `{topic}-value`). It is the **only** caller of `produce()`; services write the outbox, the relay drains it:

```python
# Outbox relay: poll unpublished → Avro-serialize against {topic}-value → produce(key=org_id) → mark published.
# FOR UPDATE SKIP LOCKED so multiple relay replicas do not double-publish.
async def relay_once(session, producer) -> int:
    rows = (await session.execute(text(
        "SELECT id, topic, partition_key, schema_subject, payload "
        "FROM outbox WHERE published_at IS NULL ORDER BY created_at "
        "FOR UPDATE SKIP LOCKED LIMIT 500"))).mappings().all()
    for r in rows:
        producer.produce(topic=r["topic"], key=r["partition_key"].encode(),
                         value=avro_serialize(r["schema_subject"], r["payload"]),
                         on_delivery=_ack(session, r["id"]))
    producer.flush()
    return len(rows)
```

Provide `services/relay/main.py` (a standalone loop process run as `python -m services.relay.main`) and a `Makefile` target `run-relay`. The relay uses `confluent-kafka` `AvroSerializer` against the Schema Registry, caching serializers per subject. Wire it to the `mytheus-events` generated subjects (the `{topic}-value` names).

> **Note for MVP deploy:** on Railway/Render the relay runs as a second process in the same deployable (a worker dyno/service), not a separate cluster. K8s extraction is post-W36 (Plan 14).

> **Acceptance:** an integration test that (a) inside one transaction writes `organizations`+`users` and an `outbox` row, commits, then (b) runs `relay_once` against the docker-compose Redpanda, asserts the message appears on the topic (consume it back), and asserts `outbox.published_at` is set. A second `relay_once` publishes nothing (idempotent drain). Crash-before-mark re-publishes (the consumer dedupes) — assert with a forced exception between produce and ack.

#### Step 0B.6 — Generate the gRPC server + stubs from the .proto

`proto/identity.proto` exists (the `GetUserContext` RPC) but there is **no generated server**. Run the existing `make proto` target, then add a server skeleton:

```bash
cd mytheus-backend
make proto    # python -m grpc_tools.protoc -I proto --python_out=. --grpc_python_out=. proto/identity.proto
```

Create `services/identity_tenant_service/grpc_server.py` implementing `IdentityServiceServicer.GetUserContext`: read `org-id` from gRPC metadata (NOT the message body), set `app.current_org`, read the `users` row + RBAC tier, and return a `UserContext`. For this band the body can return a stubbed-but-typed `UserContext` (fields wired, data from the `users` table where present, RBAC tier `"member"` default) so the contract compiles and the agent tier can bind against it in later bands. Add `make run-identity-grpc` (`python -m services.identity_tenant_service.grpc_server`).

> **Acceptance:** `make proto` regenerates `identity_pb2.py` + `identity_pb2_grpc.py`; the gRPC server boots and a test client calling `GetUserContext(user_id=...)` with metadata `("org-id", "<uuid>")` gets a `UserContext` back; calling with **no** `org-id` metadata fails closed (gRPC `UNAUTHENTICATED`).

---

### Phase 0C — `mytheus-backend` AUTH RECONCILIATION: revert Supabase drift → Clerk

> **Why interleaved here:** `libs/shared/auth.py` is the seam every protected route and the gateway depend on. The CURRENT STATE has drifted to Supabase Auth (HS256 `SUPABASE_JWT_SECRET`, a `custom_access_token_hook`, an `auth.users → public.users` trigger, `CLERK_*` removed). The resolved decision is **IDENTITY = Clerk**. This must be correct before any feature band adds a route.

**Repo / folder:** `mytheus-backend/libs/shared/`, `services/identity_tenant_service/`, `migrations/sql/`, `.env.example`, `README.md`, `CLAUDE.md`.

#### Step 0C.1 — Restore Clerk JWKS verification in `libs/shared/auth.py`

Replace the Supabase HS256 verifier with Clerk **RS256 JWKS** verification (cached, rotation-honored), per `11-security-architecture.md` §2.1. The gateway pre-validates; the service tier re-verifies (the gateway is a DDoS shield, not a trust delegation). Clerk injects `org_id` into the JWT via the session-claims / JWT template; the verifier extracts it:

```python
"""Clerk JWT verification (JWKS) + org_id extraction. Reverts the Supabase Auth drift.
Clerk issues RS256 session JWTs; org_id is injected via Clerk session claims (JWT template).
api-gateway pre-validates against cached JWKS; the service tier RE-VERIFIES here."""
from __future__ import annotations
import os, time
from uuid import UUID
import jwt
from jwt import PyJWKClient
from fastapi import HTTPException, Security, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer

_bearer = HTTPBearer(auto_error=False)
_JWKS_URL = os.environ["CLERK_JWKS_URL"]          # https://<subdomain>.clerk.accounts.dev/.well-known/jwks.json
_ISSUER   = os.environ["CLERK_ISSUER"]            # https://<subdomain>.clerk.accounts.dev
_jwks = PyJWKClient(_JWKS_URL, cache_keys=True)   # caches + honors key rotation

def verify_token(credentials: HTTPAuthorizationCredentials = Security(_bearer)) -> dict:
    if not credentials:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Missing bearer token")
    try:
        signing_key = _jwks.get_signing_key_from_jwt(credentials.credentials).key
        payload = jwt.decode(credentials.credentials, signing_key, algorithms=["RS256"],
                             issuer=_ISSUER, options={"require": ["exp", "iat", "iss"]})
    except jwt.ExpiredSignatureError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Token expired")
    except jwt.InvalidTokenError as exc:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, str(exc))
    if "org_id" not in payload:                   # Clerk session claim, fail closed
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Token missing org_id claim")
    return payload

def user_id_from(claims: dict) -> str: return claims["sub"]          # Clerk user id (maps to users.clerk_user_id)
def org_id_from(claims: dict) -> UUID: return UUID(claims["org_id"])
```

> **Mapping note:** `users.clerk_user_id` is the link (per `_canon/02-data-model.md` §2.1 `users.clerk_user_id`). The Supabase `auth.py` treated `sub` as `public.users.id`; revert that — `sub` is the Clerk user id, resolved to `users.id` via the `clerk_user_id` unique column.

#### Step 0C.2 — Restore the Svix webhook handler skeleton in `identity-tenant-service`

Rewrite `services/identity_tenant_service/routes/auth.py` from the Supabase DB-webhook to the **Clerk `POST /clerk/webhook`** handler verifying the **Svix** signature (per `11-security-architecture.md` §4), feeding the inbox-deduped user-sync:

```python
"""Clerk webhook user-sync (reverts the Supabase DB-webhook drift).
Verifies the Svix signature, dedupes on the Clerk event_id via the inbox, applies to users
in one transaction, and writes an outbox row → user-sync-events (inbox topic, per canon)."""
from fastapi import APIRouter, Request, HTTPException, Header, status
from svix.webhooks import Webhook, WebhookVerificationError
import os
router = APIRouter(tags=["auth"])
_WH_SECRET = os.environ["CLERK_WEBHOOK_SECRET"]

@router.post("/clerk/webhook")
async def clerk_webhook(request: Request,
                        svix_id: str = Header(alias="svix-id"),
                        svix_timestamp: str = Header(alias="svix-timestamp"),
                        svix_signature: str = Header(alias="svix-signature")):
    body = await request.body()
    try:
        evt = Webhook(_WH_SECRET).verify(body, {"svix-id": svix_id,
              "svix-timestamp": svix_timestamp, "svix-signature": svix_signature})
    except WebhookVerificationError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid Svix signature")
    # TODO(P1): inbox-dedupe on evt["data"]["id"] via nexus:{org_id}:dedupe:user-sync-events:{event_id};
    #           upsert users (clerk_user_id, email, role); write outbox → tenant-events on org change.
    return {"ok": True}
```

Add `svix` to `pyproject.toml` dependencies. Keep the SCIM stub (`routes/scim.py`) raising `NotImplementedError` for now (full SCIM is a later band), but ensure it is Clerk-fronted, not Supabase.

#### Step 0C.3 — Re-add `CLERK_*` env, remove the Supabase hook + trigger

1. **`.env.example`** — remove the `SUPABASE_*` auth block (`SUPABASE_JWT_SECRET`, `SUPABASE_WEBHOOK_SECRET`, `SUPABASE_SERVICE_ROLE_KEY`); keep `DATABASE_URL` (Supabase remains the **managed Postgres host**, per `_canon/06-tech-decisions.md` §2.3 — Supabase Auth is what is reverted, not Supabase Postgres). Re-add:

```bash
# === Clerk (identity) ===
CLERK_SECRET_KEY=sk_test_...
CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_JWKS_URL=https://<subdomain>.clerk.accounts.dev/.well-known/jwks.json
CLERK_ISSUER=https://<subdomain>.clerk.accounts.dev
CLERK_WEBHOOK_SECRET=whsec_...            # Svix signing secret for /clerk/webhook
```

2. **Delete** `migrations/sql/custom_claims_hook.sql` and `migrations/sql/user_sync_trigger.sql` (the Supabase `custom_access_token_hook` and the `auth.users → public.users` trigger). `git rm` both. The Clerk path injects `org_id` into the JWT at Clerk (session claims), so no DB hook is needed; user-sync is the Svix webhook + hourly reconciliation, not a Postgres trigger.

3. If any migration referenced the trigger/hook, ensure 0002 does **not** create `auth.users` artifacts.

#### Step 0C.4 — Update `README.md` / `CLAUDE.md` that still mention the wrong provider

- `mytheus-backend/README.md` and any prose mentioning Supabase Auth / `custom_access_token_hook` / `SUPABASE_JWT_SECRET` → replace with Clerk JWKS verification + Svix webhook user-sync. (`mytheus-backend/CLAUDE.md` already lists `CLERK_SECRET_KEY`/`CLERK_JWKS_URL`/`CLERK_WEBHOOK_SECRET` in its env table — keep it; ensure no Supabase-auth references remain.)
- `mytheus-backend/PREVIOUS.md` — append a comprehensive entry (per the repo's "MUST FOLLOW RULE") describing the Supabase→Clerk revert, why (resolved decision IDENTITY = Clerk), and exactly what changed (auth.py, webhook handler, deleted SQL, env).

> **Acceptance:** `grep -ri "supabase_jwt\|custom_access_token_hook\|handle_auth_user_change\|/supabase/webhook" mytheus-backend` returns nothing; `migrations/sql/` no longer contains the two files; `CLERK_JWKS_URL`/`CLERK_ISSUER`/`CLERK_WEBHOOK_SECRET` are in `.env.example`; a unit test signs a JWT with a test RSA key, serves it via a mock JWKS, and `verify_token` accepts it and extracts `org_id`; an expired/forged token raises 401; the Svix handler rejects a bad signature with 401.

---

### Phase 0D — `mytheus-intelligence`: scaffold the AI/inference repo

**Repo / folder:** `mytheus-intelligence/` (docs-only). Target (from its `CLAUDE.md`): `services/<seven>/`, `libs/shared/`, `proto/`. **Resolve §8 DECISION-REQUIRED (AI repo boundary) first** — the default below is ONE repo for all 7 services with the GPU services isolated as their own packages so a future split is mechanical.

#### Step 0D.1 — Repo skeleton + workspace `pyproject.toml`

```bash
cd mytheus-intelligence
git checkout -b plan0/intelligence-scaffold
mkdir -p services/{agent-runtime-service,rag-service,embedding-service,reranker-service,knowledge-graph-service,search-service,ingestion-service}/src
mkdir -p libs/shared proto tests
```

Root `pyproject.toml` (workspace; deps: FastAPI, LangGraph, LiteLLM, `confluent-kafka`, `qdrant-client`, `httpx`, `grpcio`, plus the published `mytheus-events` Python client and `ray[serve]` as a placeholder extra):

```toml
[project]
name = "mytheus-intelligence"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
  "fastapi>=0.115", "uvicorn[standard]>=0.30", "pydantic>=2",
  "langgraph>=0.2", "litellm>=1.40", "confluent-kafka[avro,schemaregistry]>=2.4",
  "qdrant-client>=1.9", "grpcio>=1.63", "httpx>=0.27", "redis[hiredis]>=5",
]
[project.optional-dependencies]
gpu = ["ray[serve]>=2.20"]   # placeholder; real GPU serving lands in the inference band
[dependency-groups]
dev = ["pytest>=8", "pytest-asyncio>=0.23", "ruff>=0.4", "black>=24", "mypy>=1.10"]
```

#### Step 0D.2 — Seven service skeletons + shared lib import

Each of the seven services gets `src/<svc_module>/main.py` that boots and exposes `/healthz`. For the four FastAPI services (`agent-runtime-service`, `rag-service`, `knowledge-graph-service`, `search-service`, `ingestion-service`) use a FastAPI app; for the two GPU services (`embedding-service`, `reranker-service`) add a **Ray Serve placeholder** deployment that returns a "model-not-loaded (dev stub)" response so the contract exists without a GPU:

```python
# services/embedding-service/src/embedding_service/serve.py — Ray Serve placeholder
from ray import serve
@serve.deployment(num_replicas=1)
class EmbeddingDeployment:
    async def __call__(self, request): return {"status": "stub", "model": "BGE-small-en", "dim": 384}
# Real BGE-small-en 384-dim on Ray Serve/vLLM lands in the inference band; this only proves buildability.
```

`libs/shared/` gets a thin `__init__.py` plus a `clients.py` that imports the generated `mytheus_events` package (so the AI tier consumes contracts, never hand-rolls serializers — per `mytheus-intelligence/CLAUDE.md`). Add `proto/` with a placeholder `rag.proto` (the internal gRPC mesh defined in later bands) and a `.gitkeep`.

> **Acceptance:** `ruff check .` and `mypy .` clean; each of the 5 FastAPI services boots `uvicorn <svc>.main:app` and answers `/healthz`; `python -c "import ray; from ray import serve"` imports (with the `gpu` extra) or the GPU services are skipped behind an import guard when Ray is absent; `pytest -q` (empty/smoke) passes.

---

### Phase 0E — `mytheus-ui` (Next.js 14) + `mytheus-realtime` (Node/Python) scaffolds

#### Step 0E.1 — `mytheus-ui`: Next.js 14 App Router + Clerk provider placeholder

**Repo / folder:** `mytheus-ui/` (docs-only). Per `_canon/06-tech-decisions.md` §2.8: Next.js 14 App Router, TypeScript, Tailwind + shadcn/ui, Socket.io client, LiveKit React client.

```bash
cd mytheus-ui
git checkout -b plan0/ui-scaffold
npx create-next-app@14 . --ts --app --tailwind --eslint --src-dir --import-alias "@/*" --no-git
npm i @clerk/nextjs socket.io-client @livekit/components-react livekit-client
```

Wire the Clerk provider placeholder and the API client:

- `src/app/layout.tsx` — wrap children in `<ClerkProvider>` (from `@clerk/nextjs`), guarded so a missing key in dev does not crash the build (read `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`).
- `src/lib/api.ts` — a typed fetch client reading `NEXT_PUBLIC_API_BASE_URL` (the `api-gateway`/`bff` base), attaching the Clerk session token via `auth()`/`getToken()`, defaulting paths to `/bff/*` (the only screen-shaped surface, per `mytheus-ui/system.md`).
- `src/middleware.ts` — `clerkMiddleware()` placeholder (no protected routes yet).
- `.env.example` — `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY`, `NEXT_PUBLIC_API_BASE_URL=http://localhost:8080`, `NEXT_PUBLIC_REALTIME_URL=ws://localhost:8090`.
- `src/app/page.tsx` — a placeholder dashboard shell (RSC) that renders without live data.

> **Acceptance:** `npm run build` (`next build`) succeeds; `npm run lint` clean; `tsc --noEmit` clean; the app boots `npm run dev` and renders the shell; no real backend call is required to build.

#### Step 0E.2 — `mytheus-realtime`: `realtime-gateway` (Node + Socket.io) + `transcription-service` (Python) skeletons

**Repo / folder:** `mytheus-realtime/` (docs-only). Two services that scale on different signals (per `mytheus-realtime/system.md`): `realtime-gateway` (Node + Socket.io + Redis adapter, scales on A6 connections) and `transcription-service` (FastAPI WS proxy, scales on A5 streams).

```bash
cd mytheus-realtime
git checkout -b plan0/realtime-scaffold
mkdir -p services/realtime-gateway/src services/transcription-service/src
```

- `services/realtime-gateway/` — `package.json` with `socket.io`, `@socket.io/redis-adapter`, `ioredis`, `typescript`; `src/server.ts` boots a Socket.io server, wires the Redis adapter to `nexus:{org_id}:pubsub:meeting:{meeting_id}` (channel name verbatim from `_canon/02-data-model.md` §5), exposes `/healthz`. No real consume yet (Redpanda consume lands in the realtime band).
- `services/transcription-service/` — minimal `pyproject.toml` (FastAPI, `websockets`, `confluent-kafka`); `src/main.py` boots a FastAPI app with `/healthz` and a placeholder WS route `/internal/audio` (the LiveKit server-side audio-hook target). No Deepgram wiring yet.

> **Acceptance:** `npm run build` in `realtime-gateway` compiles TS; the Socket.io server boots and answers `/healthz`; `transcription-service` boots `uvicorn` and answers `/healthz`; both have an `.env.example` with `REDIS_URL`, `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`.

---

### Phase 0F — `mytheus-platform`: minimal CI + Railway/Render deploy + .env management

**Repo / folder:** `mytheus-platform/` (docs-only). **This band ships ONLY the MVP CI + Railway/Render target — NO Kubernetes, NO Terraform** (those are post-W36 Plan 14). Confirm §8 DECISION-REQUIRED (Railway/Render = resolved yes).

#### Step 0F.1 — Reusable GitHub Actions CI workflow

```bash
cd mytheus-platform
git checkout -b plan0/ci-scaffold
mkdir -p .github/workflows env railway
```

Create `.github/workflows/ci.yml` — the standing gates from `16-deployment-and-infrastructure.md` §5.2 and `mytheus-platform/system.md` §9 (shift-left from day one):

```yaml
name: ci
on: { pull_request: {}, push: { branches: [main] } }
jobs:
  python:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync
      - run: uv run ruff check . && uv run black --check .
      - run: uv run mypy .
      - run: uv run pytest -q
  avro-compat:                      # the BACKWARD gate (mytheus-events repo only)
    if: github.repository == 'mytheus/mytheus-events'
    runs-on: ubuntu-latest
    services:
      schema-registry: { image: docker.redpanda.com/redpandadata/redpanda:latest, ports: ["8081:8081"] }
    steps:
      - uses: actions/checkout@v4
      - run: make validate && make register && make check-compatibility
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: gitleaks/gitleaks-action@v2          # secret scan, day one
      - run: pip install pip-audit && pip-audit || true   # dependency scan
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t mytheus/$GITHUB_REPOSITORY:$GITHUB_SHA .
```

Add a per-repo "authz check" lint stub: `scripts/check_authz.py` (asserts every new FastAPI route declares an auth dependency / `required_tier`), wired as a CI step in `mytheus-backend`'s copy of the workflow — the **authz-reviewed-in-every-PR** shift-left rule (`11-security-architecture.md` §3.2). For this band it can be a presence check (route → has `Depends(verify_token)` or is explicitly public) that fails the PR otherwise.

#### Step 0F.2 — Railway/Render modular-monolith deploy target

Create `railway/railway.json` (or `render.yaml`) describing the modular monolith: the FastAPI services run as one web process plus the **outbox relay** as a worker process and the **workflow-service** Temporal worker as a second worker. Single region, pooled Supabase Postgres + RLS. Example `render.yaml`:

```yaml
services:
  - type: web
    name: mytheus-backend
    env: docker
    healthCheckPath: /healthz
    envVars: [{ key: DATABASE_URL, sync: false }, { key: REDIS_URL, sync: false },
              { key: REDPANDA_BROKERS, sync: false }, { key: SCHEMA_REGISTRY_URL, sync: false },
              { key: CLERK_JWKS_URL, sync: false }, { key: CLERK_ISSUER, sync: false },
              { key: CLERK_WEBHOOK_SECRET, sync: false }]
  - type: worker
    name: mytheus-outbox-relay
    env: docker
    dockerCommand: python -m services.relay.main
  - type: worker
    name: mytheus-workflow-worker
    env: docker
    dockerCommand: python -m services.workflow_service.worker
```

A multi-stage `Dockerfile` per the `16-deployment-and-infrastructure.md` §3.1 backend recipe (builder stage resolves wheels; slim `python:3.11-slim` runtime; non-root; `tini` PID 1) lives in `mytheus-backend` and is referenced here.

#### Step 0F.3 — .env management

Create `mytheus-platform/env/` with `.env.example` templates per environment (`dev`, `staging`, `prod`) listing required keys with placeholder values and a `README.md` stating secrets flow through Render/Railway secret config (and ESO+KMS only at the K8s/post-W36 stage). **No secret values committed** — the `gitleaks` gate enforces it.

> **Acceptance:** `ci.yml` is valid (`actionlint` clean); a PR to `mytheus-events` runs the `avro-compat` job and goes green; a Render/Railway deploy from the `mytheus-backend` Dockerfile boots the web process and answers `/healthz`; the relay and workflow worker processes start; no Kubernetes/Terraform artifacts are present in this band.

---

### Phase 0G — Monorepo / multi-repo conventions

**Repo / folder:** all repos + `mytheus-docs/_canon/`.

#### Step 0G.1 — Shared Makefile targets (uniform across repos)

Standardize these targets in every repo's `Makefile`: `install`, `lint`, `typecheck`, `test`, `ci` (= `lint typecheck test`), plus repo-specific (`migrate`, `proto`, `run-relay` in backend; `validate check-compatibility register codegen` in events). `mytheus-backend/Makefile` already has most; add `run-relay` and `ci`.

#### Step 0G.2 — pre-commit

Add `.pre-commit-config.yaml` to each code repo: `ruff` + `black` (Python), `eslint` + `tsc` (TS), `gitleaks` (secret scan), and for `mytheus-events` a hook running `make validate`. Document `pre-commit install` in each README.

#### Step 0G.3 — Canon-authority location decision + CONTRIBUTING

Resolve §8 DECISION-REQUIRED (canon physical location) and **record it** so `mytheus-events`/`mytheus-realtime`/`mytheus-intelligence` can enforce names in CI. Add `CONTRIBUTING.md` to each code repo stating: (1) names/topics/tables come from `mytheus-docs/_canon/` — never invent; (2) **every PR review includes an `org_id` authz check** (shift-left security day one — `11-security-architecture.md` §3.2, `16-deployment-and-infrastructure.md` §5.2); (3) conventional commits, trunk-based, **no `Co-Authored-By` footer** (per each repo's CLAUDE.md); (4) a rename goes Glossary + ADR first.

> **Acceptance:** `make ci` exists and is uniform across repos; `pre-commit run --all-files` passes in each repo; each repo has a `CONTRIBUTING.md` naming the `org_id` authz-review rule; the canon-location decision is recorded (see §8) and the chosen mechanism (submodule/vendored/shared-repo) is in place for the three contract-binding repos.

---

## 4. Data stores stood up in this band

Per the roadmap data-store stand-up schedule (`22-roadmap-and-phasing.md` §3.1 — the Pre-P1/P1 foundation), this band stands up the **relational core only**, in its single-region pooled + RLS form:

- **PostgreSQL via Supabase (managed)** — the system of record (`_canon/06-tech-decisions.md` §2.3). This band creates the first tables (`outbox`, `inbox`, `organizations`, `users` + P2 stubs) with **RLS FORCE on `org_id`** and the canonical `tenant_isolation` policy, and configures **PgBouncer transaction pooling** with `SET LOCAL app.current_org` (`05-data-architecture.md` §3.1/§3.5). `DATABASE_URL` is the pooled connection string (Supabase pooler, port 6543).
- **Redis (Upstash) namespaced `nexus:{org_id}:…`** — stood up for the Idempotency-Key cache (`nexus:idem:{key}`) and the inbox dedupe cache (`nexus:{org_id}:dedupe:{topic}:{idempotency_key}`, 24h TTL). Local dev uses the docker-compose Redis.
- **Redpanda + Schema Registry** — stood up locally (docker-compose) so the relay can produce and the `check-compatibility` gate can run; **topics are created** (the 21 + their `.dlq`) via `rpk` in §5. Production BYOC cluster is a later/post-W36 concern.

**Not stood up this band** (later bands per the roadmap): Qdrant Cloud per-tenant collections, the Apache AGE `nexus_knowledge` graph, object-storage buckets, Temporal Cloud (the worker is scaffolded but workflows have no activities). The high-volume range-partitioned tables (`transcript_chunks`, `chat_messages`, etc.) are **not** created here — only the `organizations`/`users` root + P2 stubs.

---

## 5. Events / topics introduced

This band introduces the **event contract itself** (the frozen schemas + subjects) and the **topic provisioning**, not the producers/consumers (those land per-service in later bands). Specifically:

- **All 21 `{topic}-value` Avro subjects** authored in `mytheus-events` (Phase 0A), registered under **BACKWARD** compatibility, partitioned by `org_id`. Frozen schemas: `TranscriptChunk`, `AgentResult`, `Decision`, `Action`, `Notification`, `AgentRequest`, `AgentResponse`. Plus `EventEnvelopeHeader`, `DlqEnvelope`, and the net-new `DocumentChunk`, `AgentStreamDelta`, and 12 SaaS payloads.
- **21 topics + 21 `<topic>.dlq` topics** created locally for dev with the canon partition counts/retention as a documented `rpk` script (`mytheus-events/scripts/create_topics.sh`), e.g.:

```bash
rpk topic create transcript-chunks -p 180 -r 1 -c retention.ms=604800000   # 7d
rpk topic create transcript-chunks.dlq -p 1 -r 1 -c retention.ms=1209600000 # 14d
rpk topic create decisions -p 24 -r 1 -c retention.ms=2592000000           # 30d
rpk topic create tenant-events -p 6 -r 1 -c cleanup.policy=compact          # compacted
# … all 21 + .dlq, counts/retention from 07-event-architecture.md §4.2
```

- **First real producer wired:** `core-api`'s outbox → the relay (Phase 0B.5). The relay is the only sanctioned producer; for this band it drains whatever any test/service writes to `outbox` (the first end-to-end use is `core-api`/`identity-tenant-service` writing `tenant-events`/`meeting-events` in later bands).
- **DLQs:** the `DlqEnvelope` schema is authored; the base-consumer DLQ routing is specified in `mytheus-events/contracts/` but consumers are built per-service in later bands.

Idempotency-key formulas are reproduced verbatim in `mytheus-events/system.md` and the golden fixtures (e.g. `chunk:{chunk_id}`, `decision:{decision_id}:confirmed`, `usage:{org_id}:{event_type}:{occurred_at}:{source_id}`).

---

## 6. SaaS overlay / preconditions landing here

This band lands the **security shift-left day-one** overlay and the **event-delivery reliability substrate**; billing/budgets and dedicated GPU inference are later bands.

- **Security shift-left (from day one, replacing the PDF Phase-13 audit — `11-security-architecture.md` §9, `_canon/06-tech-decisions.md` §2.11):** `gitleaks` secret scanning in CI + pre-commit; dependency scanning (`pip-audit`); the **authz check in every PR** (`scripts/check_authz.py` + CONTRIBUTING rule); the **Clerk JWKS** verification restored (the auth primitive every later control builds on); **RLS FORCE** on the first migration so tenant isolation is load-bearing from the first table, not retrofitted; fail-closed semantics (unset `app.current_org` → zero rows; missing `org_id` claim → 401; missing `org-id` gRPC metadata → UNAUTHENTICATED).
- **At-least-once delivery substrate (`_canon/06-tech-decisions.md` §2.4):** the transactional **outbox** (relay, `outbox_idem_uq` unique index — the duplicate-decision Guard 2 from `07-event-architecture.md` §9), the **inbox** + Redis dedupe (Guards 1+3), and the `DlqEnvelope` (poison-message quarantine). The four-guard duplicate-decision stack is structurally enabled this band (the `MERGE` Guard 4 lands with `knowledge-graph-service` later).
- **Compliance-by-construction precondition:** `audit_logs`, RTBF, per-tenant KMS are later bands, but the **`org_id`-on-every-row** convention (TenantMixin) and the append-only discipline are set now so those bands extend rather than retrofit.

Billing/metering, per-tenant `agent_budgets`, and the LiteLLM cost pipeline are **out of scope** for this band (they require `usage-events` producers and the agent runtime, which scaffold here but implement later).

---

## 7. Mobile (mytheus-mobile) work in this band

Mobile work in this band is **minimal and non-wiring** (MOBILE = REALIGN; real wiring lands in later bands). The current `mytheus-mobile` is a polished Expo/React Native "Meeting Pod" prototype on 100% mock data (`src/data/mock.ts`), off-spec, with no auth/backend/realtime.

**This band:**

1. **Confirm the Expo project builds.** `cd mytheus-mobile && npm install && npx expo install --check && npm run typecheck` (it already has `expo ~52`, `expo-router ~4`, RN 0.76). Fix any broken install so `tsc --noEmit` is clean and `npx expo start` boots.
2. **Add the Clerk SDK + env-driven API base URL placeholder.** `npx expo install @clerk/clerk-expo expo-secure-store`. Add `app.config.ts`/`.env` reading `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY` and `EXPO_PUBLIC_API_BASE_URL` (placeholder, e.g. `http://localhost:8080`). Wrap `app/_layout.tsx` in `<ClerkProvider tokenCache={…SecureStore}>` guarded so a missing key does not crash dev.
3. **Add an `httpClient` module** at `src/lib/httpClient.ts` — a typed fetch wrapper reading `EXPO_PUBLIC_API_BASE_URL`, attaching the Clerk token via `getToken()`, targeting `/bff/*` (the screen-shaped surface). **It is not yet called by any screen.**
4. **Keep screens on mock data.** `src/data/mock.ts` stays; no screen is rewired this band.
5. **Document the realignment backlog** in `mytheus-mobile/REALIGN.md`: remove `mock.ts`; replace the single-device "Meeting Pod" model with the canonical per-participant client; wire `httpClient` → `bff` (REST); add Socket.io client → `realtime-gateway` (WSS) for captions/agent output/presence; add LiveKit React-Native client for media; keep the good UI components and design tokens (`src/theme`, `src/components`). Map each backlog item to the future band that owns it.

> **Acceptance:** `npm run typecheck` clean; `npx expo start` boots; `@clerk/clerk-expo` and `src/lib/httpClient.ts` exist and compile; `EXPO_PUBLIC_*` placeholders are in `.env.example`; screens still render from `mock.ts`; `REALIGN.md` lists the backlog with band mapping.

---

## 8. DECISION-REQUIRED callouts

> ⚠️ **DECISION-REQUIRED 1 — `mytheus-intelligence` repo boundary.** ONE repo for all 7 AI services, or split the GPU inference services (`embedding-service`, `reranker-service` on Ray Serve/vLLM) from the Python services (`agent-runtime-service`, `rag-service`, `knowledge-graph-service`, `search-service`, `ingestion-service`)?
> - **Options:** (A) one repo, GPU services isolated as their own packages (default — split later is mechanical, simplest for MVP modular monolith); (B) two repos from day one (`mytheus-intelligence` + `mytheus-inference`), clean GPU/CI separation but more cross-repo contract overhead now.
> - **Who decides:** Head of AI / tech lead. **Blocks:** Phase 0D scaffolding (can't `mkdir` the tree until decided) and the `mytheus-platform` CI matrix (one vs two AI repos). **Recommendation:** (A) for MVP; the GPU services already live behind a Ray Serve boundary and split into Wave 1 extraction cleanly.

> ⚠️ **DECISION-REQUIRED 2 — Canon physical location for CI enforcement.** Where do `_canon/` authority files live so `mytheus-events` / `mytheus-realtime` / `mytheus-intelligence` can enforce names + schemas in CI?
> - **Options:** (A) **git submodule** of `mytheus-docs/_canon/` in each contract-binding repo (single source, but submodule friction); (B) **vendored copies** synced by a bot/CI check (no submodule, but drift risk — needs a "canon-sync" CI gate); (C) a **published `@mytheus/canon` package** (npm + PyPI) generated from `mytheus-docs` (cleanest consumption, most tooling to build).
> - **Who decides:** Platform lead + canon owners. **Blocks:** §0G.3 (and therefore the name-enforcement CI step in Phases 0A/0D/0E). **Recommendation:** (A) submodule for MVP (zero build tooling), revisit (C) post-W36. **This must be resolved before Phase 0A.**

> ⚠️ **DECISION-REQUIRED 3 — MVP deploy target = Railway/Render (confirm).** Resolved decision says **yes**: CI deploys the **modular monolith** to Railway/Render, single region, pooled + RLS — **not Kubernetes**. Confirm no team member is reaching for K8s/Terraform in this band.
> - **Options:** (A) Railway/Render modular monolith (resolved — the relay + workflow worker run as worker processes in the same deployable); (B) anything K8s — **rejected for MVP** (post-W36 Plan 14 only).
> - **Who decides:** Eng lead (ratify the resolved decision). **Blocks:** Phase 0F (the deploy descriptor + CI deploy step). **Recommendation:** (A); keep §3.1 container seams real but co-located so Wave 1–4 extraction stays mechanical later.

---

## 9. Risk checkpoints

- **R1 (AGE-on-managed-Postgres awkwardness) — not yet active.** This band creates no AGE graph; the relational core is plain Postgres. Checkpoint: ensure the migration layer does **not** assume AGE so the later `knowledge-graph-service` band can add the `nexus_knowledge` extension without reworking `Base.metadata`.
- **R6 (cross-meeting agent-to-agent context leakage) — substrate only.** No agents run this band, but the `AgentRequest`/`AgentResponse` schemas (Phase 0A.3) carry `org_id` in the header and `correlation_id`/`deadline`, and the topics are `org_id`-partitioned — the bus-level isolation that R6's deny-by-default authz (later band) builds on. Checkpoint: confirm both schemas reject a missing `org_id` (header is non-nullable).
- **R8 (integration scope creep) — guardrail recorded, not built.** `integration_connections` is a P2 stub here; no provider OAuth is wired. Checkpoint: the stub model's `provider` enum is exactly `{google_calendar, slack, jira, linear}` (Calendar + Slack + one of Jira/Linear) so the guardrail is encoded before the integration band.
- **Drift-reintroduction risk (this band's own R):** the Supabase→Clerk revert can be silently undone by a stray `SUPABASE_JWT_SECRET` reference. Checkpoint: the `grep` acceptance in §0C.4 runs in CI as a guard.

---

## 10. Definition of Done & end-of-phase demo

**Definition of Done (Plan 0):**

- [ ] `mytheus-events`: `make ci` green; 7 frozen + 14 net-new schemas authored; 21 `{topic}-value` subjects under BACKWARD; Python **and** TypeScript clients generated; golden fixtures round-trip for all 21 topics; a breaking change fails `check-compatibility`.
- [ ] `mytheus-backend`: `alembic upgrade head` creates `outbox`, `inbox`, `organizations`, `users` + P2 stubs with RLS FORCE; `target_metadata = Base.metadata` wired; Idempotency middleware mounted on all 10 services; outbox relay drains to Redpanda; gRPC `GetUserContext` server generated and boots; `pytest` green (RLS isolation, outbox/inbox idempotency, idempotency-replay, JWT verify tests).
- [ ] **Auth reconciliation:** Clerk JWKS verification restored; Svix `/clerk/webhook` skeleton restored; `CLERK_*` in `.env.example`; Supabase `custom_access_token_hook` + `auth.users` trigger deleted; READMEs/CLAUDE.md/PREVIOUS.md updated; `grep` guard clean.
- [ ] `mytheus-intelligence`: 7 service skeletons + shared lib + Ray Serve placeholder build; `ruff`/`mypy` clean.
- [ ] `mytheus-ui`: `next build` + lint + `tsc` clean; Clerk provider + API client placeholders wired.
- [ ] `mytheus-realtime`: `realtime-gateway` (Node/Socket.io) + `transcription-service` (Python) skeletons boot `/healthz`.
- [ ] `mytheus-platform`: CI workflow (lint/test/gitleaks/dep-scan/avro-compat/docker-build) valid; Railway/Render modular-monolith deploy descriptor present; `.env` templates; **no K8s/Terraform**.
- [ ] `mytheus-mobile`: builds; Clerk SDK + `httpClient` + env base URL added; screens still on mock; `REALIGN.md` backlog written.
- [ ] Conventions: uniform `make ci`, pre-commit, `CONTRIBUTING.md` (org_id authz-review rule), canon-location decision recorded.

**End-of-phase demo (PDF cadence — a demo at the end of every phase):** a single scripted walkthrough proving the foundation end-to-end:

1. `git clone` each repo, `make ci` → all green (record the CI run URLs).
2. In `mytheus-events`: edit `TranscriptChunk` to add a defaulted field → `make check-compatibility` passes (BACKWARD-safe); remove `confidence` → it **fails** (gate works); revert.
3. In `mytheus-backend`: `docker compose up -d && alembic upgrade head`; run the relay; a test writes `organizations`+`users`+an `outbox` row in one transaction, the relay publishes it, `rpk topic consume` shows the Avro message keyed by `org_id`; a second relay pass publishes nothing.
4. Sign a Clerk-style RS256 JWT against a mock JWKS → `verify_token` accepts and extracts `org_id`; an expired token → 401; a `POST` with a repeated `Idempotency-Key` replays from Redis.
5. `next build` for `mytheus-ui`; `/healthz` from `realtime-gateway` and `transcription-service`; `ruff`/`mypy` clean for `mytheus-intelligence`; `npx expo start` for mobile.
6. Deploy `mytheus-backend` to Railway/Render; `curl https://<deploy>/healthz` → 200; the relay + workflow worker processes are running.

---

## 11. Source docs (authoring references used, by path)

- `/Users/sushan/Projects/Mytheus/mytheus-docs/07-event-architecture.md` — envelope, subjects, BACKWARD gate, outbox/inbox/dedupe, DLQ + `DlqEnvelope`, four-guard duplicate-decision stack, partition counts/retention.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/_canon/03-event-taxonomy.md` — the 21 topics, 7 frozen schemas, header fields, idempotency-key formulas, producers/consumers.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/03-container-architecture.md` — the 20 services, communication styles (REST/gRPC/WSS/async), edge path, store ownership, scaling families, Railway/Render modular monolith.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/16-deployment-and-infrastructure.md` — CI/CD pipeline + gates, image build recipe, Railway/Render→K8s migration (K8s post-W36), shift-left scans, expand/contract migrations.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/05-data-architecture.md` — RLS policy SQL, `SET LOCAL app.current_org`, PgBouncer transaction pooling, partitioning (later bands), Redis namespaces.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/_canon/02-data-model.md` — table/column names (`organizations`, `users`, outbox/inbox shapes via §7), RLS, Redis namespaces, the informal-label→catalog mapping.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/11-security-architecture.md` — Clerk JWKS at edge + middleware, Svix webhook, hardened user-sync, shift-left/authz-in-every-PR, fail-closed.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/24-glossary-and-naming.md` — canonical name index (services/topics/schemas/entities), the net-new-not-frozen schema list, label mapping §10.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/_canon/06-tech-decisions.md` — locked stack (FastAPI, SQLAlchemy 2.0 async, Alembic, Pydantic v2, Redpanda+Avro+Registry BACKWARD, outbox/inbox/DLQ, Clerk, Railway/Render→K8s, Next.js 14, GitHub Actions, gitleaks).
- `/Users/sushan/Projects/Mytheus/mytheus-events/system.md` + `CLAUDE.md` — repo layout, Makefile targets, idempotency keys, codegen/consumption, golden samples.
- `/Users/sushan/Projects/Mytheus/mytheus-backend/system.md` + `CLAUDE.md` — service breakdown, ownership, edge path, event contract, recipes (migrations/outbox/gRPC), env vars, `PREVIOUS.md` rule.
- `/Users/sushan/Projects/Mytheus/mytheus-platform/system.md` + `CLAUDE.md` — CI/CD, Railway/Render early stage, scans, the four families (K8s deferred).
- `/Users/sushan/Projects/Mytheus/mytheus-intelligence/system.md` + `CLAUDE.md` — 7 services, Ray Serve GPU split, shared-lib import, statelessness.
- Live repo inspection: `mytheus-backend/libs/shared/{auth,tenancy,outbox,inbox,idempotency,events,errors}.py`, `migrations/{env.py,sql/*}`, `services/identity_tenant_service/routes/auth.py`, `proto/identity.proto`, `.env.example`, `Makefile`, `pyproject.toml`; `mytheus-mobile/package.json` + tree; `mytheus-ui/system.md`; `mytheus-realtime/system.md`.
