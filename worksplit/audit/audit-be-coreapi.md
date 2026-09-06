# core_api audit (READ-ONLY)

Root: `/Users/sushan/Projects/Mytheus`
Target: `mytheus-backend/services/core_api/` — 11 files, 1708 LOC.
Method: source only. No `.md` status/progress claim was trusted; every claim below cites `path/file.py:LINE`.
Nothing was modified.

---

## IMPLEMENTED (file:line)

**11 HTTP routes total, all under prefix `/api/v1`, all real (no stubs).**

| Route | File:line | Auth dependency | Data source (R/W) |
|---|---|---|---|
| `POST /meetings` | `services/core_api/routes/meetings.py:153` | `require_tier("member")` | W `meetings`, `meeting_participants`, `outbox`(topic `meeting-events`), `audit_logs`; R `users` |
| `GET /meetings` | `meetings.py:243` | `require_tier("member")` | R `meetings` + `meeting_participants` |
| `GET /meetings/{meeting_id}` | `meetings.py:291` | `require_tier("member")` | R `meetings` + `meeting_participants` |
| `GET /meetings/{meeting_id}/participants/{clerk_user_id}` | `meetings.py:302` | **`tenant_context` only — no tier gate** | R `meetings`, `meeting_participants JOIN users` |
| `PATCH /meetings/{meeting_id}` | `meetings.py:386` | `require_tier("member")` | W `meetings`, `outbox`, `audit_logs` |
| `GET /meetings/{meeting_id}/transcript` | `routes/transcript.py:95` | `require_tier("member")` | R `transcript_chunks` (foreign-owned table) |
| `GET /meetings/{meeting_id}/speakers` | `routes/speakers.py:47` | `require_tier("member")` | R `meeting_speakers` |
| `PATCH /meetings/{meeting_id}/speakers/{speaker_id}` | `speakers.py:80` | `require_tier("member")` | W `meeting_speakers` (upsert) |
| `POST /chat` | `routes/chat.py:88` | `require_tier("member")` + participant check | W `chat_messages`; R `meetings`, `meeting_participants` |
| `GET /teams` | `routes/org_hierarchy.py:37` | `require_tier("member")` | R `teams` |
| `POST /teams` | `org_hierarchy.py:64` | **`require_tier("admin")`** | W `teams` |
| `GET /departments` | `org_hierarchy.py:91` | `require_tier("member")` | R `departments` |
| `POST /departments` | `org_hierarchy.py:111` | **`require_tier("admin")`** | W `departments` |
| `GET /teams/{team_id}/members` | `org_hierarchy.py:196` | `require_tier("member")` | R `team_members` |
| `PUT /teams/{team_id}/members/{user_id}` | `org_hierarchy.py:220` | **`require_tier("admin")`** | W `team_members`, `audit_logs`; evicts identity cache |
| `DELETE /teams/{team_id}/members/{user_id}` | `org_hierarchy.py:273` | **`require_tier("admin")`** | W (delete) `team_members`, `audit_logs`; evicts identity cache |

(16 rows above because `org_hierarchy` contributes 7 routes; total distinct route decorators = 16, across 5 route modules. The "11 files" figure in the brief is file count, not route count.)

Plus `GET /healthz` injected by `libs/shared/app_factory.py:34`.

Router assembly: `services/core_api/main.py:15-20` — `create_app("core-api")` then five `include_router` calls (meetings, org_hierarchy, chat, speakers, transcript).

### What is genuinely well-built here

- **Keyset pagination, never OFFSET**, on both paginated collections: `meetings.py:64-83` over `(created_at, id)`, `transcript.py:61-72` over `(start_ms, id)`. Both encode base64-opaque cursors and both include the id tiebreak so a non-unique leading column cannot skip or repeat a row. Malformed cursor → `400 INVALID_CURSOR`, not 500 (`meetings.py:82`, `transcript.py:70`).
- **Tenant scoping is RLS-only, deliberately.** No route writes a hand-rolled `org_id = :org` predicate; the comments at `meetings.py:130-131` and `chat.py:53-55` explain that adding one invites a route that forgets it and works anyway on a privileged connection. `libs/shared/authz.py:126-145` (`tenant_session`) sets `app.current_org` **transaction-locally**, which is the correct choice under connection pooling.
- **404-never-403 on cross-tenant**, centralised in `authz.py:not_found()` with the reasoning written out: a 403 confirms existence and turns any UUID into a probe.
- **Atomicity**: `tenant_session` yields inside `session.begin()` (`authz.py:143`), so row write + outbox row + audit row commit as one fact. `meetings.py:8-10` states this as an invariant and the code holds it.
- **`org_id` is never accepted from a caller.** `schemas.py:8-9` states it; verified — it appears in no request model and in no path/query parameter of any route.
- **`extra="forbid"`** on every request model (`schemas.py:24,45,143,157,173,187,214`) so a typo'd field is a 422 rather than a silent no-op.
- **Errors** are RFC 7807 with stable codes throughout (`libs/shared/errors.ProblemDetail`), wired once in `app_factory.py:25`.

---

## TRANSCRIPT ENDPOINTS — detailed answer

### How chunks are read

`transcript.py:134-146`, raw SQL:

```sql
SELECT id, meeting_id, speaker_id, start_ms, end_ms, text, is_final, confidence, redacted
FROM transcript_chunks
WHERE meeting_id = :meeting_id AND is_final = true [AND (start_ms,id) > (:cur_start,:cur_id) | AND start_ms >= :from_ms]
ORDER BY start_ms ASC, id ASC
LIMIT :limit
```

`limit` default 100, max 500 (`transcript.py:53-56`), fetched as `limit + 1` to detect a next page without a COUNT.

### Tenant scoping — yes, and better than the minimum

Two layers:
1. RLS on `transcript_chunks` (`ENABLE` + `FORCE` + policy `tenant_isolation`, created in `mytheus-realtime/services/transcription-service/migrations/versions/be66698dbefb_create_transcript_chunks.py:118-123`, predicate `org_id = NULLIF(current_setting('app.current_org', true), '')::uuid`, fail-closed to zero rows when unset).
2. An explicit `_assert_meeting_visible` pre-check (`transcript.py:75-92`) that 404s before touching the chunks table. The reasoning at `transcript.py:80-83` is correct and non-obvious: `transcript_chunks` carries **no FK to `meetings`** (declarative partitioning precludes it, and the owning migration says so at `be66698dbefb:11-16`), so without this pre-check another tenant's meeting id would return an empty 200 page — which leaks "well-formed but empty" vs "not yours".

### Pagination — yes

Keyset over `(start_ms, id)`, the same tuple the ORDER BY uses and the same leading column as the table's index `ix_transcript_chunks_meeting_start (meeting_id, start_ms)` (`be66698dbefb:116`, created on the partitioned parent so it propagates to every partition). `from_ms` provides a server-side timestamp seek; `cursor` wins when both are supplied (`transcript.py:114-115`) so an in-progress walk is not silently restarted.

### ADR-0017 composite partition key — **the question conflates two different things**

`mytheus-docs/adr/0017-transcript-chunks-composite-partition-key.md` ratifies `{org_id}:{meeting_id}` as the **Redpanda topic partition key** for exactly two topics: `transcript-chunks` and `agent-stream`. Its stated motivation is Kafka partition saturation for a single hot tenant (360 finals/meeting on `transcript-chunks`), and it explicitly leaves the other 19 topics on `org_id`-only.

core_api **neither produces nor consumes** either of those topics. Its only producer is `meeting-events` (`meetings.py:214,420`), whose key is `org_id` via `libs/shared/events.py:39` — which is exactly what ADR-0017 preserves for non-exempt topics. So there is nothing for core_api to honour or violate on the ADR-0017 axis. **Not a defect; the ADR simply does not reach this code.**

The **table** partition key is a different axis entirely: `PARTITION BY RANGE (created_at)`, monthly, hand-premade three months ahead plus a `DEFAULT` catch-all (`be66698dbefb:88-138`), because pg_partman is not installed on the shared image.

### Real defect A — no partition pruning

The query has **no `created_at` predicate**. Every request therefore touches all monthly partitions plus `transcript_chunks_default`. Postgres will Merge Append across the per-partition local indexes so it still works and still avoids a sort — but pruning is entirely forfeited, and per-request cost scales with **retention depth**, not with meeting size. The docstring's claim at `transcript.py:29-32` that "the keyset page is an index range scan rather than a sort" is true per partition and understates the fan-out. Fixable by bounding `created_at` from the meeting's `scheduled_start`/`created_at`, which the route already has in hand from `_assert_meeting_visible`.

### Real defect B — soft-deleted chunks are served

`transcript_chunks` has `deleted_at timestamptz` (`be66698dbefb:101`). Every other read in core_api filters it: `meetings.py:139`, `meetings.py:253`, `chat.py:58`, `speakers.py:39`, `transcript.py:87` (the meeting pre-check itself), `org_hierarchy.py:55,101,164`. The chunk query at `transcript.py:123` builds `where = ["meeting_id = :meeting_id", "is_final = true"]` and **never adds `deleted_at IS NULL`**. A redaction or RTBF soft-delete on a chunk still returns that chunk from this endpoint. One-line fix; inconsistent with the whole rest of the service.

### Minor / non-defects checked

- `is_final = true` is filtered even though the writer only persists finals — defensive, and the reasoning at `transcript.py:120-122` is sound.
- `confidence` is `real NOT NULL` in the DDL but `float | None` in `schemas.py:129` — harmless widening.
- `redacted` is exposed, which is right: `schemas.py:117-119` explains a reader needs to know odd text is redaction rather than mis-transcription.
- No writes anywhere in the module — the read-only boundary claimed at `transcript.py:12-15` holds.

### Why there is no search here

`transcript.py:17-27` refuses to add a `tsv` column + GIN index to `transcript_chunks` because that table belongs to another repo — the second-writer trap. It cites `search-service`'s identical refusal in `mytheus-intelligence/migrations/versions/d415f023220e_0003_search_service_transcript_index.py`, which builds its own `transcript_chunk_search` read model instead. **This is the correct call and is consistently applied across both repos.** Canon's "read via core-api FTS" line is the thing that is wrong, not the code.

---

## PARTIAL

### 1. `PATCH /speakers` — cross-tenant `user_id` guard is missing, and the comment asserting otherwise is factually wrong

`speakers.py:123-133` catches `IntegrityError` and comments:

> "The FK on user_id -> users(id) is the enforcement: a user id from another tenant, or one that does not exist, is not insertable (same mechanism `routes.meetings`'s participant-insert comment describes for meeting_participants)."

That is the **inverse** of what the cited comment says. Two sibling files spell the actual behaviour out at length:

- `org_hierarchy.py:172-193` (`_visible_user_id`): "THIS CHECK IS LOAD-BEARING FOR TENANT ISOLATION… PostgreSQL performs referential-integrity checks as the referenced table's owner and with row-level security BYPASSED, so the foreign key happily resolves another tenant's `users` row… Verified by deleting this call and re-running `test_a_user_from_another_tenant_cannot_be_added`: the insert returned 200 and wrote a row joining this tenant's team to another tenant's user."
- `meetings.py:186-192`: "This comment previously asserted the opposite and was wrong: the route returned 201 and persisted a row referencing another tenant's user."

Both of those files therefore run an explicit `SELECT 1 FROM users WHERE id = :id` visibility probe before inserting (`meetings.py:193-195`, `org_hierarchy.py:189-191`). **`speakers.py` runs no such probe.** Its upsert at `speakers.py:100-118` passes `body.user_id` straight through; `meeting_speakers`' own WITH CHECK constrains `org_id` (the caller's own, so it passes) and nothing constrains `user_id`.

Expected consequence: `PATCH /meetings/{id}/speakers/{sid}` with another tenant's `user_id` returns 200 and writes a cross-tenant reference into `meeting_speakers`, rather than the 422 the comment predicts. **UNVERIFIED at runtime** — not executed against a live DB — but the code path is structurally identical to the two cases this repo has already found and fixed twice, and there is no test covering it (`test_meeting_speakers_api.py` has 7 tests; none is a cross-tenant user case by name).

### 2. Presence-events consumer is code with no runner

`consumer.py:118 run_consumer()` and `consumer.py:181 if __name__ == "__main__"` exist and are well-written (manual commit, DLQ on poison, strict `uuid.UUID(header["org_id"])` re-assertion). But nothing starts it:

- `Makefile:65-91` has `run-gateway`, `run-core-api`, `run-identity`, `run-bff`, `run-meeting`, `run-billing`, `run-workers`, `run-relay`, `run-identity-grpc`. No core-api consumer target.
- `docker-compose.yml` defines only `postgres`, `redis`, `redpanda`, `temporal`, `temporal-ui` — no application services at all.
- `Dockerfile:42` CMD is the uvicorn web process.
- Only `tests/test_presence_events_consumer.py:24` imports it.

And even if it ran, it is a deliberate no-op: `consumer.py:96-113` logs structurally and writes the inbox dedupe row, writing nothing to any domain table, because a durable analytics sink would need a canon/ADR decision first. So core_api's sole inbound event path currently changes no state and is not running.

### 3. `main.py`'s ownership claim outruns the schema

`main.py:3-4`: "Owns: meetings, participants, speakers, worklogs, decisions, action_items, agent_suggestions, agent_collaborations, chat_messages, teams, team_members, departments."

Grep across `mytheus-backend/migrations/versions/` for `worklogs|action_items|agent_suggestions`: the only hit is `tenant_settings.retention_days_worklogs` (`b7d9f3a55c21_...py:112`). **No `worklogs`, `decisions`, `action_items`, `agent_suggestions` or `agent_collaborations` table exists in any migration.** Five of the twelve claimed tables are aspirational.

### 4. `list_meetings` status filter is unvalidated

`meetings.py:249`: `status_filter: Annotated[str | None, Query(alias="status")]`. Not constrained to the `MeetingStatus` Literal that `schemas.py:20` defines and `MeetingUpdate` uses. No injection risk (bound parameter at `meetings.py:257`), but `?status=Scheduled` returns an empty page instead of a 422.

---

## STUB/FAKE

**None.** Every one of the 16 routes executes real SQL against real tables. No `NotImplementedError`, no `pass`, no hardcoded/mock response payloads, no `TODO` inside a handler body. The only TODO in the service is `main.py:22`, which is an honest scope note, not a stubbed route.

This is unusual for this codebase and worth stating plainly: core_api does not fake anything.

---

## MISSING (routes the UI, gateway, or docs reference but that do not exist here)

| Missing | Referenced at | Effect |
|---|---|---|
| `/api/v1/decisions` | `services/api_gateway/edge.py:152` (ROUTE_MAP → `core-api`) | Gateway routes it successfully; core-api 404s |
| `/api/v1/actions` | `edge.py:153` | same |
| `/api/v1/worklogs` | `edge.py:151` | same |
| `/api/v1/suggestions` | `edge.py:155` | same |
| `POST /decisions/:id/confirm`, `POST /actions/:id/confirm` | `main.py:6-9` names these as the ONLY legitimate producers of the `decisions`/`actions` topics; `libs/shared/idempotency.py:30` uses one as its worked example | Avro schemas exist (`mytheus-events/schemas/decisions`, `.../actions`) with no producer. Deliberate — `main.py:7-9` argues a speculative producer would put unconfirmed edges into the knowledge graph |
| `DELETE /meetings/{id}` | every read filters `deleted_at IS NULL` | Soft delete is unreachable through the API; only a direct DB write or the admin/RTBF seam can set it |
| `GET /chat` (history) | no reader anywhere — not in core_api, not in bff | `POST /chat` is the sole `chat_messages` route. A client that misses the socket relay cannot recover chat history, unlike transcript which explicitly has a re-hydrate endpoint for exactly this reason (`transcript.py:6-10`) |
| Search over transcripts | canon "read via core-api FTS" | Deliberately absent, see above; lives in `search-service` |
| Org-context | — | **Not missing, and not core_api's.** `GET /context/{user_id}` is `services/identity_tenant_service/routes/context.py:32`; org chart is `services/bff/org_chart.py:169` |

`bff/dashboard.py:115-123` already models `action_items`, `worklogs` and `decisions` as `UnbuiltArm` shapes that return empty rather than failing the screen (`dashboard.py:97`), so the UI degrades cleanly — the gateway does not.

---

## OUTBOUND DEPENDENCIES — each edge, and does the far side exist?

| Edge | Where | Far side |
|---|---|---|
| Produces topic `meeting-events` via transactional outbox | `meetings.py:214-227` (created), `meetings.py:420-436` (updated/ended); `libs/shared/outbox.enqueue` | **Exists.** Schema `mytheus-events/schemas/meeting-events/v1.avsc`. Drained by `services/relay/main.py` (`libs/shared/relay.py:86` produces keyed by `partition_key`). Consumed by `services/meeting_service/consumer.py:86` and `services/workflow_service/workflows/meeting_lifecycle.py`. Full loop is real. |
| Partition key for that topic = `org_id` | `libs/shared/events.py:39` | Correct per ADR-0017 (composite key is scoped to `transcript-chunks`/`agent-stream` only) |
| Event-type discrimination | `meetings.py:418` — `"ended" if fields.get("status") == "ended" else "updated"` | Correct; all three symbols exist in the Avro enum. Covered by `test_status_ended_emits_event_type_ended_not_updated` |
| Consumes topic `presence-events` | `consumer.py:55` | Topic + schema exist; producer is `mytheus-realtime`'s `presenceEventsProducer.ts`. **No process runs the consumer** (see PARTIAL 2) |
| Reads `transcript_chunks` (cross-repo table) | `transcript.py:138` | Table exists, owned by `mytheus-realtime/services/transcription-service`. **Open grant gap — see below** |
| Reads `users`, `rbac_grants`, `organizations` | `authz.py:66-70` (`_SELECT_PRINCIPAL`), plus per-route probes | Owned by `identity_tenant_service`. Exist |
| Python import of another service | `org_hierarchy.py:32` → `services.identity_tenant_service.context.service.invalidate_user_context`, called at `org_hierarchy.py:331` | Exists. **In-process call, not HTTP** — couples the two deployables at import time. `org_hierarchy.py:318-330` argues the direction is correct (the table's writer evicts the cache owner's key rather than either writing the other's store) and notes `bff` imports `get_user_context` the same way. Defensible in a modular monolith; a hard coupling the moment these split |
| Writes `audit_logs` | `libs/shared/audit.audit` from `meetings.py:228,437`, `org_hierarchy.py:258,302`, `authz.py:_audit_denial` | Exists, partitioned (`c3e8b1d47f92`) |
| Redis — idempotency middleware | `app_factory.py:23-24` | Exists. Chat relies entirely on it for replay dedupe (`chat.py:15-25` explains why a table-level unique key is not expressible on a partitioned table) |
| OpenTelemetry | `app_factory.py:26` | No-op unless `OTEL_EXPORTER_OTLP_ENDPOINT` set |

### Cross-repo GRANT gap — disclosed in writing, still open

`migrations/versions/9b2c4d1e6f30_0005_retire_schema_wide_default_grants.py:73-95` states it plainly:

> "`mytheus-realtime`'s `transcript_chunks` migration (`be66698dbefb`) creates its table in this same `public` schema and contains no GRANT at all — it relies entirely on the default privilege this migration retires… What breaks is a rebuild from an EMPTY CLUSTER in which that migration runs after this one: the parent table comes out with an empty ACL and `mytheus_app` gets 'permission denied for table transcript_chunks'."

The prescribed one-line fix is `GRANT SELECT, INSERT, UPDATE, DELETE ON transcript_chunks TO mytheus_app, mytheus_admin;` in the realtime repo. I grepped every file under `mytheus-realtime/services/transcription-service/migrations/` for `GRANT`: **zero hits.** The fix has not been applied. So on any clean-cluster rebuild — disaster recovery, or a new developer's first `make dev` — `GET /meetings/:id/transcript` fails with a permission error, while every other core_api route works.

Related and correctly handled: `mytheus-realtime/.../migrations/_partition_privileges.py` REVOKEs on each partition so a partition cannot be named directly to bypass the parent's RLS (a defect that was live and verified before the fix). Querying through the parent, which is what `transcript.py` does, only needs the parent grant — hence the gap above is the live one.

---

## INBOUND — who calls core_api

**`mytheus-ui`** (Next.js, server-side only — the client imports `auth()` from `@clerk/nextjs/server` so a client-component import fails the build):
- `src/lib/api/transcript.ts:44` — `GET /api/v1/meetings/{id}/transcript`, walks the cursor to completion with `limit: 500`, bounded at `MAX_PAGES = 20` (`transcript.ts:34`)
- `src/lib/api/transcript.ts:56` — `GET /api/v1/meetings/{id}/speakers`
- `src/app/(app)/meetings/[meetingId]/transcript/actions.ts:34` — `PATCH /api/v1/meetings/{id}/speakers/{sid}`, then `revalidatePath`
- `src/app/api/meeting-transcript/route.ts` — the caption strip's re-hydrate source after a socket drop
- Types are generated from core-api's OpenAPI: `src/lib/api/generated/core-api`
- Screens fall back to mock data when `API_GATEWAY_URL` is unset (`transcript.ts:60-70` distinguishes "no backend configured" from a real 404 and refuses to paper over the latter)

**`mytheus-realtime` realtime-gateway** (TypeScript, no database of its own):
- `src/coreApiClient.ts:117` — `GET /api/v1/meetings/{id}/participants/{clerkUserId}` on **every** Socket.io join, to decide FORBIDDEN vs allow, and to resolve internal `org_id`/`user_id` before producing `presence-events`
- `src/coreApiClient.ts:151` — `POST /api/v1/chat` with `Idempotency-Key: clientMsgId`
- Test double: `test/helpers/fakeCoreApi.ts:134`

**`mytheus-mobile`**:
- `src/api/hooks/index.ts:30` — `GET /api/v1/meetings?status=&limit=50`
- `src/api/hooks/index.ts:45` — `GET /api/v1/teams`
- `src/api/hooks/index.ts:53` — `GET /api/v1/departments`
- `__tests__/api-client.test.ts` exercises the client against these paths

**`api_gateway`**: `services/api_gateway/edge.py:147-157` proxies 8 prefixes to core-api (`/api/v1/meetings`, `/teams`, `/departments`, `/worklogs`, `/decisions`, `/actions`, `/chat`, `/suggestions`). Four of the eight have no route behind them. Longest-prefix matching (`edge.py:184-191`) keeps `/api/v1/meetings` (core-api) apart from `/api/meetings` (meeting-service). The gateway strips and re-signs internal headers (`edge.py:128-141`), and core_api correctly ignores them — `meetings.py:333-347` documents that `X-Org-Id` "isn't even declared as a parameter" and the only tenancy source is the re-verified bearer token.

**`bff`**: does **not** call core-api over HTTP. It reads core-api-owned tables directly from Postgres — `services/bff/dashboard.py:166` (`FROM meetings`), `services/bff/org_chart.py:68` (`FROM teams`), `org_chart.py:72` (`FROM team_members JOIN users`). Second *reader*, not second writer, so the one-writer rule holds; but the dashboard bypasses core-api's route layer entirely, so any authorization or shaping logic added to those routes will not apply to the dashboard.

---

## CONFIG/ENV REQUIRED

Web process:
- `DATABASE_URL` — **must name `mytheus_app`**, not a superuser. RLS is `FORCE`d, but a `rolbypassrls` credential still defeats every isolation property in this service. `tests/conftest.py:36-46` explains that `DATABASE_URL` names `mytheus_app` "in any correctly configured environment"
- `REDIS_URL` — idempotency middleware (`app_factory.py:23`); `POST /chat` replay-dedupe depends on it entirely
- `CLERK_ISSUER`, `CLERK_JWKS_URL`, `CLERK_JWT_ORG_CLAIM`, `CLERK_SECRET_KEY` — token verification (`libs/shared/auth`)
- `OTEL_EXPORTER_OTLP_ENDPOINT` — optional; telemetry is a no-op without it

Consumer process (not currently launched):
- `REDPANDA_BROKERS` (default `localhost:9092`), `SCHEMA_REGISTRY_URL` (default `http://localhost:8081`), `REDIS_URL` (`consumer.py:128-132`)

Serving:
- `Dockerfile:42` — `CMD ["uvicorn", "services.core_api.main:app", "--host", "0.0.0.0", "--port", "8000"]` (the image's default role)
- `Makefile:69` — dev, port 8002
- **`docker-compose.yml` contains no core-api service** — only `postgres`, `redis`, `redpanda`, `temporal`, `temporal-ui`. There is no compose-level deployment of any application service in this repo.

---

## TESTS covering core_api

Seven test files touch it. They are **real assertions against a live Postgres**, not smoke — and notably, they run as the least-privilege `mytheus_app` role rather than the schema owner, which `tests/conftest.py:14-20` argues is the only way isolation claims mean anything ("proving isolation for a role the application never uses proves nothing. That was exactly the flaw in the retired `app_rls` role").

| File | Tests | Character |
|---|---|---|
| `tests/test_meetings_api.py` | 17 | Strongest. Cross-tenant GET → 404 not 403 (`:191`), cross-tenant PATCH → 404 (`:201`), list isolation (`:211`), another tenant's user as participant (`:221`), cursor stability and completeness (`:320`), malformed cursor → 400 not 500 (`:340`), limit bounds (`:346`), event+audit in one transaction (`:136`), validation failure writes nothing (`:167`), unknown field rejected (`:183`), two updates both reach the outbox (`:366`), `status=ended` emits `ended` not `updated` (`:384`), member cannot create a team and the denial is audited (`:268`), suspended tenant refused (`:305`) |
| `tests/test_team_members_api.py` | 11 | Includes `test_a_user_from_another_tenant_cannot_be_added`, the test that proved the FK does not enforce tenancy |
| `tests/test_participant_check_api.py` | 8 | Includes an isolated proof that the join result depends on `users` RLS, not merely row existence |
| `tests/test_meeting_speakers_api.py` | 7 | Covers the manual-is-authoritative property; **no cross-tenant `user_id` case** |
| `tests/test_chat_api.py` | 6 | Proves the same `Idempotency-Key` posted twice yields exactly one row |
| `tests/test_presence_events_consumer.py` | 4 | Exercises `handle_presence_event` directly against real Postgres + fakeredis |
| `tests/test_openapi_contract.py` | — | Contract shape |

**The hole: zero tests for `routes/transcript.py`.** `grep -rn transcript tests/` returns exactly one hit, in `test_livekit_import_boundary.py:4`, and it is an unrelated mention in a docstring. The endpoint the UI depends on for both the transcript screen and caption re-hydration has no test for the cursor, no cross-tenant test, no `from_ms` test, no `is_final` test, and no test that would have caught the missing `deleted_at` filter.

Caveat: `tests/conftest.py:4` says tests that cannot reach their infrastructure "skip gracefully". A green run does not by itself prove the database tests executed. **Not run during this audit — UNVERIFIED.**

---

## VERDICT

core_api is the most genuinely finished service I have read in this repo: 16 real routes across 5 modules, no stubs or mock responses anywhere, keyset pagination done correctly on both collections, tenant isolation enforced structurally through transaction-local RLS rather than hand-written predicates, and a test suite that proves those isolation properties against the same least-privilege role the application actually runs as. The defects are narrow and specific rather than architectural: `speakers.py:123` repeats a cross-tenant `user_id` mistake that two sibling files document at length and guard against, and its comment asserts the opposite of what this codebase has twice verified to be true; `transcript.py:123` omits the `deleted_at IS NULL` filter that every other read in the service applies; the transcript query has no `created_at` bound so it forfeits partition pruning entirely; `transcript_chunks` has a written-down cross-repo GRANT gap that breaks the transcript endpoint on any clean-cluster rebuild; and that same endpoint — the one both the UI transcript screen and the caption re-hydrate path depend on — has no tests at all.

The larger gap is scope rather than quality. Four gateway prefixes (`/decisions`, `/actions`, `/worklogs`, `/suggestions`) route here and hit nothing; `main.py`'s "Owns:" list names five tables no migration creates; the presence-events consumer is a deliberate no-op with no process configured to run it; and there is no read route for chat history and no delete route for meetings despite every query filtering `deleted_at`. On ADR-0017 specifically: it governs Redpanda topic keys for two topics core_api neither produces nor consumes, so it is not violated here — the table's own monthly `created_at` range partitioning is the axis that actually matters for this endpoint, and that one is not being used.
