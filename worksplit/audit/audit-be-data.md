# mytheus-backend — Data + Shared-Infrastructure Layer Audit

Read-only audit. Root: `/Users/sushan/Projects/Mytheus/mytheus-backend`.
All claims below are from source and migration files, not from `.md` status docs. Citations are `path/file:LINE`.

---

## SCHEMA TABLE

Full schema as actually created by `migrations/versions/`: **16 base tables + 14 partitions**, all in schema `public`, one shared Alembic history (`migrations/env.py:1`).

| table | tenant column | RLS policy | written by | read by | ORPHAN? |
|---|---|---|---|---|---|
| `organizations` | `id` **IS** org_id (no `org_id` column) | **NONE** — policy dropped, RLS DISABLED at `a1c4e7f20b18:110-112` | `services/identity_tenant_service/provisioning.py` | `libs/shared/org_resolver.py`, `services/meeting_service/webhook.py`, `provisioning.py` | no |
| `users` | `org_id` (FK → organizations, RESTRICT) | `tenant_isolation` (`82d44163c933:159`) | `identity_tenant_service/reconciliation.py`, `routes/users.py`, `routes/auth.py`, `seed/seed.py` | `libs/shared/authz.py:68`, `grpc_server.py`, `bff/people.py`, `bff/org_chart.py`, `core_api/routes/meetings.py`, `core_api/routes/org_hierarchy.py`, `context/service.py` | no |
| `teams` | `org_id` | yes | `core_api/routes/org_hierarchy.py` | `bff/org_chart.py`, `context/service.py`, `seed/seed.py` | no |
| `team_members` | `org_id` | yes | `core_api/routes/org_hierarchy.py`, `seed/seed.py` | `bff/org_chart.py`, `context/service.py` | no |
| `departments` | `org_id` | yes | `core_api/routes/org_hierarchy.py` | `bff/org_chart.py`, `context/service.py`, `seed/seed.py` | no |
| `tenant_settings` | `org_id` (PK + FK) | yes | `identity_tenant_service/routes/tenant_settings.py` — **only `pii_redaction_enabled`** | that route; `meeting_service/routes.py` (reads `consent_required`) | **PARTIAL ORPHAN** — `sso_config`, `retention_days_transcripts`, `retention_days_worklogs`, `default_locale`, `agent_confidence_overrides` written by nothing |
| `data_residency` | `org_id` (PK + FK) | yes | **NOTHING** | **NOTHING** | **ORPHAN** — ADR-0009 multi-region placement record, entirely unbuilt. ORM class exists (`identity_tenant_service/models.py:202`) with zero call sites |
| `meetings` | `org_id` | yes | `core_api/routes/meetings.py` | `meeting_service/routes.py`, `bff/dashboard.py`, `core_api/routes/{transcript,chat,speakers}.py`, `context/service.py`, `seed/seed.py` | no |
| `meeting_participants` | `org_id` | yes | `core_api/routes/meetings.py` | `core_api/routes/chat.py`, `meeting_service/routes.py`, `context/service.py` | no |
| `meeting_rooms` | `org_id` (FK, RESTRICT) | yes (`3d13f46b960f:38`) | `meeting_service/rooms.py`, `meeting_service/webhook.py` | `meeting_service/webhook.py` | no |
| `meeting_speakers` | `org_id` (FK, RESTRICT) | yes (`25e7cede5672:97`) | `core_api/routes/speakers.py` | `core_api/routes/speakers.py` | no — but the `assignment='suggested'` writer that the trigger at `25e7cede5672:169` exists to guard **does not exist yet** |
| `chat_messages` (RANGE-partitioned on `created_at`) | `org_id` (FK, RESTRICT) | yes on parent (`5ed978467cad:112`) | `core_api/routes/chat.py` | **NOTHING** — no read route exists | **WRITE-ONLY** |
| `chat_messages_2026_08` | via parent | inherited from parent | — | — | only partition that exists; see MIGRATION CHAIN HEALTH defect 1 |
| `rbac_grants` | `org_id` (FK, RESTRICT) | yes (`b7d9f3a55c21:81`) | **only `identity_tenant_service/seed/seed.py:196`** | `libs/shared/authz.py:68`, `identity_tenant_service/grpc_server.py:34` | **WRITE-ORPHAN** — the authorization store the whole `require_tier` chain reads has **no API write path**; a real tenant cannot grant or revoke a tier |
| `audit_logs` (RANGE-partitioned on `occurred_at`) | `org_id` (FK, RESTRICT) | yes on parent (`c3e8b1d47f92:130`), append-only by GRANT | `libs/shared/audit.py` | **NOTHING** — no read/export/query route | **WRITE-ONLY** |
| `audit_logs_2026_07` … `audit_logs_2027_06` (12) | via parent | inherited | — | — | each has `ix_*_org_occurred` + `ix_*_org_target` |
| `audit_logs_default` | via parent | inherited | — | — | catch-all so a late INSERT degrades to a slow query, not a refused write |
| `outbox` | `org_id` (FK, RESTRICT) | `tenant_isolation` **+ permissive `outbox_relay_drain`** for `mytheus_relay` (`a1c4e7f20b18:196`) | `libs/shared/outbox.py`, `libs/shared/relay.py` | `libs/shared/relay.py` | no |
| `inbox` | `org_id` (FK, RESTRICT) | yes | `libs/shared/inbox.py` | `identity_tenant_service/routes/auth.py`, `meeting_service/webhook.py` | no |

### Key columns per table (as created)

- `organizations`: `id` uuid PK, `name`, `slug` UNIQUE, `plan_id` uuid (**never written or read — orphan column**), `clerk_org_id` text UNIQUE (`a1c4e7f20b18:129`), `status` text NOT NULL DEFAULT `'active'` CHECK in (`active`,`suspended`), `created_at`/`updated_at`/`deleted_at`.
- `users`: `id`, `org_id`, `clerk_user_id`, `email`, `role` (domain role, **never authorization**), `skills` text[] NOT NULL DEFAULT `'{}'`, `bio`, `avatar_url`, timestamps. UNIQUE `(org_id, clerk_user_id)` — the global unique on `clerk_user_id` was dropped at `b7d9f3a55c21:86` because Clerk allows one person in several orgs.
- `rbac_grants`: `id`, `org_id`, `user_id`, `tier` CHECK in (`owner`,`admin`,`member`), `granted_by`, `created_at`. UNIQUE `(org_id, user_id)`.
- `tenant_settings`: `org_id` PK, `agent_confidence_overrides` jsonb NOT NULL, `default_locale`, `retention_days_transcripts` (365), `retention_days_worklogs` (365), `pii_redaction_enabled` bool NOT NULL DEFAULT true, `sso_config` jsonb, `consent_required` bool NOT NULL DEFAULT false (`866f7eedb0e7:52`), timestamps. The generic `settings` jsonb bag was dropped at `b7d9f3a55c21:135`.
- `data_residency`: `org_id` PK, `region` CHECK in (`us`,`eu`,`ap`), `db_cell` DEFAULT `'pool-us-1'`, `qdrant_region`, `object_bucket_region`, `locked_at`.
- `meetings`: `id`, `title`, `status` CHECK in (`scheduled`,`live`,`ended`,`cancelled`), `org_id`, `scheduled_start`, `scheduled_end` (CHECK ordered), `livekit_room`, `created_by`, timestamps.
- `meeting_participants`: PK `(meeting_id, user_id)`, `org_id`, `role_in_meeting`, `consented_at`, `consent_version`.
- `meeting_rooms`: `id` PK, `org_id`, `meeting_id`, `livekit_room`, `status` CHECK in (`pending`,`active`,`ended`), `egress_id`, `recording_ref`, `livekit_metadata_synced_at` (`f3a7c9e0b21d:38`), timestamps. UNIQUE `(org_id, meeting_id)` and `(org_id, livekit_room)`.
- `meeting_speakers`: `id` PK, `org_id`, `meeting_id`, `speaker_id` (diarized label, not a user id), `user_id` nullable, `confidence` float, `assignment` CHECK in (`manual`,`suggested`), timestamps. UNIQUE `(org_id, meeting_id, speaker_id)`. Trigger `trg_meeting_speakers_manual_authoritative` refuses any UPDATE moving a row out of `manual`.
- `chat_messages`: PK `(id, created_at)`, `org_id`, `meeting_id`, `user_id`, `content`, `deleted_at`. Partition indexes: `(org_id, meeting_id, created_at)` + GIN on `to_tsvector('english', content)`.
- `audit_logs`: PK `(id, occurred_at)`, `org_id`, `actor_id`, `actor_type` CHECK in (`user`,`agent`,`system`), `action`, `target_type`, `target_id`, `metadata` jsonb, `ip` inet, `occurred_at`.
- `outbox`: `id` BIGSERIAL PK, `org_id`, `topic`, `partition_key`, `idempotency_key`, `schema_subject`, `payload` jsonb, `created_at`, `published_at`, `attempts`. UNIQUE `(topic, idempotency_key)`; partial index `outbox_unpublished_idx` on `created_at WHERE published_at IS NULL`.
- `inbox`: PK `(consumer_group, org_id, idempotency_key)`, `processed_at`.

### Cross-repo note

`transcript_chunks` lives in this same `public` schema but is created by **`mytheus-realtime`** migration `be66698dbefb` (`migrations/_grants.py:20-24`). One physical database, two Alembic histories. Whether they share an `alembic_version` table is **UNVERIFIED** from this repo.

### ORM/model drift

No SQLAlchemy model exists for `meeting_rooms`, `meeting_speakers`, or `chat_messages` — stated explicitly at `866f7eedb0e7:31-37`. Consequence: `alembic revision --autogenerate` against this schema proposes **dropping** those three tables and every `audit_logs_*` partition. Autogenerate output cannot be trusted here without hand editing; both `866f7eedb0e7` and `3d13f46b960f` say so in their docstrings.

Models that do exist: `services/core_api/models.py` (Meeting, MeetingParticipant), `services/identity_tenant_service/models.py` (Organization, User, RbacGrant, Team, TeamMember, Department, TenantSettings, DataResidency), `services/admin_ops_service/models.py` (AuditLog), `libs/shared/infra_models.py` (Outbox, Inbox).

---

## MIGRATION CHAIN HEALTH

**Linear, single head, no branches, no gaps.** `branch_labels` is `None` on all eleven; `depends_on` is `None` on all eleven.

```
82d44163c933  (down_revision = None)   0001 core tables, outbox/inbox, RLS on 11 tables
  -> a1c4e7f20b18                      0002 three least-privilege roles, tenant-root carve-out, 7 missing org FKs, relay drain policy
  -> b7d9f3a55c21                      0003 rbac_grants + canonical columns
  -> c3e8b1d47f92                      0004 audit_logs partitioned, append-only by GRANT
  -> 3d13f46b960f                      meeting_rooms
  -> 866f7eedb0e7                      consent columns + meeting_rooms status CHECK
  -> 5ed978467cad                      chat_messages (current-month partition only)
  -> 25e7cede5672                      meeting_speakers + manual-authoritative trigger
  -> f3a7c9e0b21d                      meeting_rooms.livekit_metadata_synced_at
  -> 56f72ac73fa9                      two missing lookup indexes
  -> 9b2c4d1e6f30  (HEAD)              0005 retire schema-wide default grants, revoke image roles
```

Every migration has a written `downgrade()`, including a deliberately defect-restoring one in 0005 ("a downgrade is not the place to keep a fix the upgrade is being rolled back from", `9b2c4d1e6f30:265`).

### Defect 1 — `chat_messages` has no partition covering today (LIVE)

`5ed978467cad:53-55` hardcodes `_PARTITION_YEAR = 2026`, `_PARTITION_MONTH = 8`. Only `chat_messages_2026_08` is created, covering `[2026-08-01, 2026-09-01)`. Today is **2026-09-06**. Every `POST /chat` INSERT now raises `no partition of relation "chat_messages" found for row`.

No `pg_partman`, no cron job, no nightly maintenance, and no later migration creates another partition — grepping `PARTITION OF` across `*.py`/`*.sql`/`*.yml`/`Makefile` returns only `c3e8b1d47f92` and `5ed978467cad`. The migration docstring (`5ed978467cad:23`) calls this "a known, accepted, and explicitly flagged risk, not an oversight". The risk has since materialized. `audit_logs` is unaffected — 12 monthly partitions through 2027-06 plus `audit_logs_default`.

### Defect 2 — `migrations/_grants.py` is dead code

`grant_tenant_table` (`_grants.py:34`) and `grant_append_only_table` (`_grants.py:47`) are imported by **zero** migrations. Grepping the whole repo for `grant_tenant_table` returns only its own definition, its own docstring, `9b2c4d1e6f30`'s prose, and one test docstring.

This is currently harmless: the four tables created between 0002 and 0005 (`rbac_grants`, `meeting_rooms`, `chat_messages`, `meeting_speakers`) obtained their ACLs from the `ALTER DEFAULT PRIVILEGES` installed by `a1c4e7f20b18:227`, and 0005 runs last so a fresh rebuild still grants them correctly. But the mechanism 0005's entire cost/benefit argument rests on has never once been exercised, and the next table-creating migration gets an **empty ACL** unless its author remembers a helper no existing migration demonstrates.

### Defect 3 — the repo-boundary rebuild break (disclosed)

`mytheus-realtime`'s `transcript_chunks` migration `be66698dbefb` contains no GRANT and relied entirely on the default privilege that `9b2c4d1e6f30` retires. Existing databases are fine (a default ACL only affects objects created after it). A rebuild **from an empty cluster** where that migration runs after `9b2c4d1e6f30` produces `permission denied for table transcript_chunks` for `mytheus_app`. That path is disaster recovery and a new developer's first `make dev`. Deliberately left unfixed from this repo on ownership grounds (`9b2c4d1e6f30:77-92`).

---

## TENANCY AT DB LAYER

**Verdict: genuinely enforced. This is the strongest layer in the system, and the claims are not decorative.**

What is actually true in the database:

1. **Three login roles, none elevated.** `mytheus_app` (request path), `mytheus_relay` (outbox drain only), `mytheus_admin` (operator seam), created at `a1c4e7f20b18:88-98` with `LOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE` stated explicitly, plus an `ALTER ROLE` branch so the attributes are re-asserted if the role already exists.
2. **RLS is FORCEd, not merely ENABLEd,** on every `org_id` table. FORCE removes the table owner's exemption, which is what makes a SECURITY DEFINER escape hatch impossible.
3. **One identical fail-closed predicate everywhere:** `org_id = NULLIF(current_setting('app.current_org', true), '')::uuid`. An unset or reset-to-empty GUC yields NULL, the predicate is false, and the query returns zero rows rather than raising. Repeated verbatim in `82d44163c933:31`, `b7d9f3a55c21:29`, `c3e8b1d47f92:48`, `3d13f46b960f:37`, `5ed978467cad:57`, `25e7cede5672:82`.
4. **The GUC is set transaction-locally.** `libs/shared/tenancy.py:33` — `set_config('app.current_org', :org_id, true)`. The third argument being `true` is what stops one tenant's org id leaking into another tenant's reuse of a pooled connection (Supabase pooler / PgBouncer).
5. **`organizations` is the one deliberate carve-out.** Policy dropped and RLS disabled at `a1c4e7f20b18:110-112`, because the edge must resolve `clerk_org_id -> org_id` before any tenant is known, and under the policy that path had no executable form (FORCE RLS also defeats a SECURITY DEFINER resolver). Three compensating controls replace it: column-level `GRANT SELECT (id, clerk_org_id, name, slug, status, deleted_at)` and nothing else (`a1c4e7f20b18:79,149`) so `SELECT *` is refused by the database; a `COMMENT ON TABLE` recording the rule; and exactly two sanctioned query shapes in `libs/shared/org_resolver.py:49,55`, both carrying an equality predicate on the lookup key.
6. **The relay's cross-tenant reach is a policy, not a role attribute.** `mytheus_relay` holds `SELECT, UPDATE` on `outbox` alone plus one permissive `outbox_relay_drain` policy (`a1c4e7f20b18:196`), so the reach is inspectable in `pg_policies` and cannot extend to any other table. A `BYPASSRLS` role was explicitly rejected.
7. **`mytheus_admin` is elevated in GRANTS only.** It remains subject to FORCE RLS, so an RTBF purge must name the tenant it is purging.
8. **Partition-bypass is closed three ways.** A partition is a table with its own ACL, and the parent's RLS only applies on access through the parent. `c3e8b1d47f92:113` and `5ed978467cad:105` revoke each partition explicitly, and `9b2c4d1e6f30` closes the cause: it retires the schema-wide `ALTER DEFAULT PRIVILEGES` for TABLES entirely, then sweeps every existing partition clean of any non-owner grantee via `aclexplode(relacl)` (`9b2c4d1e6f30:141-163`) rather than a hardcoded role list.
9. **Arm 2 of that migration is the material one.** `supabase/postgres` ships its own `ALTER DEFAULT PRIVILEGES ... GRANT ALL ON TABLES TO anon, authenticated, service_role`, and `service_role` holds `rolbypassrls` while `authenticator` is a LOGIN role that is a member of all three. The migration quotes an empirically reproduced result with no `app.current_org` set: `SET ROLE service_role; SELECT count(*) FROM audit_logs -> 352` across every tenant, and `DELETE FROM audit_logs_2026_08 -> DELETE 1`. Step 3 revokes all of it.
10. **CI asserts the properties.** Fourteen tests in `tests/test_roles_and_policies.py`, including `test_no_application_role_is_superuser_or_bypasses_rls:49`, `test_every_org_id_table_has_a_tenant_isolation_policy:98`, `test_no_application_role_can_reach_a_partition_directly:144`, `test_a_new_partition_inherits_no_privileges:315`, `test_no_bypassrls_role_holds_privileges_on_a_public_table:264`, `test_relay_role_is_scoped_to_outbox_alone:591`, `test_admin_role_is_still_subject_to_rls:604`.

### Holes

- **`service_role` still holds `rolbypassrls`.** `ALTER ROLE ... NOBYPASSRLS` requires SUPERUSER, which the migration runner (`postgres`, demoted by the image's own `10000000000000_demote-postgres.sql`) does not have. What 0005 removes is its *reach* — it now holds no privilege on anything in `public`. `supabase_admin` could re-grant at any time; only `test_no_bypassrls_role_holds_privileges_on_a_public_table` would catch it. Self-disclosed at `9b2c4d1e6f30:96-100`.
- **The image's `ALTER DEFAULT PRIVILEGES FOR ROLE supabase_admin` survives** for the same missing-membership reason. Inert only while nothing runs DDL as `supabase_admin`; `psql -U supabase_admin` inside the container is the path of least resistance for an operator, and any table created that way is born granting full DML to a BYPASSRLS role. Self-disclosed at `9b2c4d1e6f30:101-110`.
- **No runtime assertion that the connected role lacks BYPASSRLS.** `libs/shared/db.py:44-51` warns in a docstring that pointing `DATABASE_URL` at `postgres` or any BYPASSRLS role "silently disables tenant isolation everywhere at once" — that was defect D1 — but nothing in `get_engine()` checks. The dev `postgres` role *does* hold BYPASSRLS by image design (`docker/postgres-dev-password.sql:19`). A misconfigured production env var reintroduces D1 with no error and no failing test.
- **`plan_id` on `organizations` is an unconstrained orphan uuid** — no FK, no writer, no reader.
- **`data_residency.db_cell`** implies a future multi-cell topology; there is no code for cell routing, and how RLS composes with a per-cell split is **UNVERIFIED**.
- **`event trigger` auto-revoke was considered and correctly rejected** — `CREATE EVENT TRIGGER` needs SUPERUSER, which the runner lacks (`9b2c4d1e6f30:44-56`). Retiring the default privilege is the stronger fix anyway.

---

## SHARED LIBS — used vs dead

`libs/shared/` — 19 modules, 1794 lines. Importer counts are direct `libs.shared.<mod>` imports across `services/`, `libs/`, `scripts/`, `tests/`.

| module | LOC | what it is | status |
|---|---|---|---|
| `db.py` | 111 | `Base` + deterministic `NAMING` convention; lazy `lru_cache`d `get_engine`(`DATABASE_URL`) / `get_admin_engine`(`ADMIN_DATABASE_URL`) + sessionmakers; `TenantMixin` (declared_attr `org_id` FK RESTRICT, indexed), `TimestampMixin` | **USED** — 31 importers, 10 services |
| `authz.py` | 196 | The 4-step chain: `verify_token` → `resolve_org_id` → `tenant_session` → `require_tier`. Frozen `RequestContext`, `TIER_RANK {member:1, admin:2, owner:3}`, `_SELECT_PRINCIPAL` LEFT JOIN on `rbac_grants`, `_client_ip` that drops an unparseable XFF rather than letting `inet` casting break a fail-closed audit write | **USED** — 16 importers, 15 services |
| `errors.py` | 185 | RFC 7807 `ProblemDetail`, frozen `ErrorCode` enum, `add_exception_handlers` | **USED** — 26 importers, 15 services |
| `app_factory.py` | 38 | `create_app(service_name)`: mounts `IdempotencyMiddleware`, error handlers, OTel, OpenAPI post-processing, `/healthz` | **USED** — 27 importers, all 10 HTTP services |
| `auth.py` | 132 | Clerk JWKS verification, `clerk_org_id_from`, `user_id_from`; JWKS fetched lazily so import never needs Clerk env | **USED** — 23 importers |
| `tenancy.py` | 37 | `set_org_id`/`get_org_id` contextvar + `set_rls` (transaction-local) | **USED** — 14 importers, 10 services |
| `org_resolver.py` | 101 | The only sanctioned `organizations` reader. Two functions by design: `resolve_org_id` (opaque Clerk claim → raises `ProblemDetail`) and `resolve_org_by_id` (internal candidate id → returns `None`, for the LiveKit webhook). Deliberately no cache; exit trigger documented as "build it in the same change that adds the `tenant-events` consumer" | **USED** — 11 importers, 6 services |
| `audit.py` | 93 | The one `audit_logs` writer. Takes the caller's session so the audit row and the change commit together. Frozen `ACTOR_TYPES`. Explicitly refuses a `best_effort=True` escape hatch | **USED** — 8 importers, 6 services |
| `outbox.py` | 58 | `enqueue(session, envelope)` — insert in caller's transaction, `ON CONFLICT (topic, idempotency_key) DO NOTHING`, returns existing id on duplicate | **USED** — 4 importers, 3 services |
| `inbox.py` | 51 | `is_duplicate` (Redis `nexus:{org}:dedupe:{topic}:{key}`, 24h TTL, SET NX) + `record_consumed` (durable inbox insert) | **USED** — 5 importers, 4 services |
| `events.py` | 56 | `EventEnvelope` dataclass: 8-field frozen header + topic payload, `to_value()` composes the Avro value | **USED** — 6 importers, 3 services |
| `relay.py` | 149 | `relay_once` — `FOR UPDATE SKIP LOCKED` drain, per-row failure isolation, `MAX_ATTEMPTS=5`, DLQ to `{topic}.dlq`. `RelayProducer` Protocol keeps it unit-testable | **USED** — 5 importers, 3 services |
| `idempotency.py` | 205 | `Idempotency-Key` middleware. Key is `nexus:{org_id}:idem:{key}` plus a request fingerprint (fixes defect D2 cross-tenant leak and the cross-endpoint collision). Unresolvable tenant → passed through **uncached** | **USED via `app_factory` only** — 0 direct service importers, which is correct by design |
| `cache.py` | 60 | Lazy process-wide Redis + frozen `nexus:{org_id}:` key builder that raises on falsy org_id; 1.0s socket timeout so "Redis is slow" never becomes "the request never completes" | **USED** — 5 importers, 3 services |
| `openapi.py` | 293 | Post-processes the FastAPI spec to add the bearer `securityScheme`, the `Idempotency-Key` parameter, and RFC 7807 error responses — three parts of the contract FastAPI's generator cannot see. Sources `ErrorCode` from the Python enum rather than restating it | **USED** — via `app_factory` + `scripts/emit_openapi.py` |
| `public_routes.py` | 37 | The single definition of "reachable without a bearer token". Two lists by kind: `PUBLIC_PREFIXES` (probes, docs — genuinely unauthenticated) and `PUBLIC_CONTAINS` (webhooks/OAuth — authenticated by signature, *not* by token). Dependency-free on purpose so the AST walker can import it without importing the app | **USED** — exactly its two stated consumers: `libs/shared/openapi.py` and `scripts/check_authz.py` |
| `clerk_admin.py` | 108 | Writes `internal_org_id` into the Clerk organization's `public_metadata`; never raises | **USED** — 2 importers, 1 service |
| `telemetry.py` | 24 | OTel `TracerProvider` + OTLP gRPC span exporter + `FastAPIInstrumentor` | **USED via `app_factory`** — but no-ops without an endpoint (see OBSERVABILITY) |
| `infra_models.py` | 67 | SQLAlchemy ORM for `outbox` and `inbox` | **NO RUNTIME CONSUMER** — imported only by `migrations/env.py:22`, purely so autogenerate sees the tables. All runtime access is raw `text()` SQL in `outbox.py`/`inbox.py` |

**Nothing is outright dead.** `infra_models.py` is the only module with zero runtime importers, and it has a legitimate autogenerate-only role. `openapi.py`, `public_routes.py`, `telemetry.py` and `idempotency.py` all show 0 direct *service* importers but are reached through `app_factory` or a CI script.

### Scripts

- `scripts/bootstrap_roles.py` (91) — sets the three role passwords from env; refuses to default. Used by `make roles`.
- `scripts/emit_openapi.py` (142) — emits specs; `make openapi` / `make openapi-check`.
- `scripts/check_authz.py` (175) — static AST walker failing CI on a route with no auth dependency that is not in `public_routes`.

---

## EVENT INFRA

**Real, and the outbox pattern is properly built — with one structural inconsistency and one missing shared piece.**

What exists:

- **Transactional outbox, shared.** `libs/shared/outbox.py::enqueue` inserts in the caller's open transaction; state change and intent-to-emit commit atomically or not at all.
- **Relay, shared and hardened.** `libs/shared/relay.py::relay_once` — `SELECT ... FOR UPDATE SKIP LOCKED LIMIT :limit` so replicas don't double-publish; Avro-serialize against `{topic}-value`; produce keyed by `org_id`; mark published. Per-row failure isolation (defect D5): a row that will not serialize does not abort the batch, its `attempts` increments, and at `MAX_ATTEMPTS = 5` the **raw** payload goes to `{topic}.dlq` (deliberately not Avro — it is there because Avro failed) and the row is marked published so the queue drains. Broker-side non-delivery is treated as batch-wide, not per-row: `flush()` returning nonzero raises before anything is marked, so the whole batch re-publishes and consumers dedupe.
- **Frozen envelope.** `libs/shared/events.py::EventEnvelope` — header of `event_id`, `idempotency_key`, `org_id`, `schema_version`, `producer`, `occurred_at` (timestamp-micros), `correlation_id`, `causation_id`; `schema_subject` = `{topic}-value`; `partition_key` = `org_id`.
- **ADR-0007 idempotency is real and two-layer.** Redis fast path `nexus:{org_id}:dedupe:{topic}:{idempotency_key}` with `SET NX EX 86400`, and the `inbox` table (PK `(consumer_group, org_id, idempotency_key)`) as the durable authority when the cache has evicted, written in the handler's own transaction. Each consumer group applies a fact exactly once, independently.
- **Concrete producer** in `services/relay/main.py:28-31` — `confluent_kafka` + `SchemaRegistryClient` + `AvroSerializer` (use-latest-version).

What is missing:

- **No shared consumer harness.** There is a `RelayProducer` Protocol but no `Consumer` counterpart. Each consumer re-implements the loop: `services/core_api/consumer.py:119-123`, `services/meeting_service/consumer.py:247-251`, `services/notification_service/consumer.py:14`.
- **Two Kafka client libraries in one codebase.** `confluent-kafka[avro,schemaregistry]>=2.4` (relay, core_api, meeting_service) **and** `aiokafka>=0.11` (notification_service alone, `consumer.py:14`). Both declared at `pyproject.toml:14-15`. The aiokafka consumer therefore has none of the Schema Registry / Avro deserialization path the other two share.
- **No DLQ consumer, no replay tooling.** Rows dead-lettered by `relay._dead_letter` land on `{topic}.dlq` with nothing reading them.
- **No outbox-depth or consumer-lag metric.** The relay's whole design rationale is that a silent drain failure was invisible (defect D4); the only signal today is a `logger.error` line.
- **No topic creation, no schema registration** in this repo. A `mytheus-events` repo exists at the workspace root — out of scope, **UNVERIFIED**.

---

## STORAGE / PII VAULT / OBSERVABILITY

### Object storage / R2 (ADR-0023) — **ADR-ONLY**

No S3, R2, or boto3 client exists anywhere in `services/` or `libs/`. `boto3>=1.43.64` is a declared dependency (`pyproject.toml:31`) imported by nothing.

What does exist is `services/meeting_service/storage.py` (52 lines): a key-convention module and nothing else. `RECORDINGS_BUCKET = "nexus-recordings"`, a `RecordingStorage` Protocol, and `KeyConventionRecordingStorage.recording_key()` returning `f"{org_id}/{meeting_id}/{egress_id}"`. Its own docstring says "no S3 bucket and no LiveKit account exist for this build" and "No S3 API call happens anywhere in this module". The rationale is defensible — LiveKit's egress process writes the object directly, so this service would never upload bytes — but there is no bucket, no lifecycle policy, no presigned-URL reader, and `services/meeting_service/transport.py:137` is `raise NotImplementedError("start_egress lands with the nexus-recordings bucket (Task 4)")`.

`.env.example` contains no R2 account id, endpoint URL, access key, or secret. Note the naming mismatch worth flagging: **ADR-0023 says Cloudflare R2; the code names an S3 bucket and describes AWS credentials handed to LiveKit egress.**

### PII token vault / NER (ADR-0020) — **ADR-ONLY**

No vault table, no token store, no NER model, no tokenizer. No `presidio`, spaCy, or any NLP dependency in `pyproject.toml`.

What exists is a single boolean and a switch for it: `tenant_settings.pii_redaction_enabled` (default **true**) and `services/identity_tenant_service/routes/tenant_settings.py`, which writes Postgres then mirrors to Redis at `nexus:{org_id}:tenant_settings:pii_redaction_enabled`, returning 503 if the mirror fails (because a stale-OFF mirror *under*-redacts, which is the unrecoverable direction). That route's own docstring (lines 4-5) records that the redaction gate "defaults to `False`, honestly, because **nothing in the program ever wrote that key**" — i.e. the switch is now wired, but whether anything downstream reads it is **UNVERIFIED** (the consumer is `ingestion-service`, in another repo). `consent_required` is deliberately not editable through this route.

### Observability — **tracing real-if-configured; logging thin; metrics zero**

- **Tracing: real code, off by default.** `libs/shared/telemetry.py` builds a genuine `TracerProvider` with a `Resource(SERVICE_NAME)`, a `BatchSpanProcessor` over an OTLP gRPC `OTLPSpanExporter`, and `FastAPIInstrumentor.instrument_app`. But line 17-19 returns immediately if `OTEL_EXPORTER_OTLP_ENDPOINT` is unset, and there is **no collector in `docker-compose.yml`** — so it is a no-op in every local and test run. Not a stub; just never switched on.
- **Logging: bare stdlib.** `logging.getLogger(__name__)` only. No structlog, no JSON formatter, no request-id or trace-id correlation, no central configuration. `logging.basicConfig` appears exactly once in the entire codebase, at `services/identity_tenant_service/seed/seed.py:445`. `LOG_LEVEL` is in `.env.example` and read by nothing.
- **Metrics: none at all.** No Prometheus client, no OTel meter, no counters, no histograms. Grep for `prometheus|metrics|Counter(` across `services/` and `libs/` returns zero. The two subsystems whose entire design rationale is "a silent failure would be invisible" — outbox depth and relay dead-lettering — emit log lines only.
- **Sentry: declared, never imported.** `sentry-sdk[fastapi]>=2` at `pyproject.toml:29` and `SENTRY_DSN` in `.env.example`; `sentry_sdk` appears in no source file.
- **Health: `/healthz` on every service** via `app_factory` — returns `{"status":"ok","service":...}` unconditionally, no dependency check.

---

## FULL REQUIRED ENV VAR LIST

### Hard-required — raises `KeyError` at the point of use

| var | site | scope |
|---|---|---|
| `DATABASE_URL` | `libs/shared/db.py:49`, `migrations/env.py:32`, `services/relay/main.py:84`, `services/identity_tenant_service/grpc_server.py:146` | every process. Must name `mytheus_app` on the request path |
| `ADMIN_DATABASE_URL` | `libs/shared/db.py:72` | seed script + RTBF purges only; deliberately **absent** from the web process env |
| `CLERK_JWKS_URL` | `libs/shared/auth.py:49` | every service that verifies a token |
| `CLERK_ISSUER` | `libs/shared/auth.py:53` | same |
| `REDPANDA_BROKERS` | `services/relay/main.py:94` | relay |
| `SCHEMA_REGISTRY_URL` | `services/relay/main.py:94` | relay |
| `LIVEKIT_URL` | `services/meeting_service/transport.py:262` | meeting-service — **only reached if `LIVEKIT_API_KEY` is set** |
| `LIVEKIT_API_SECRET` | `services/meeting_service/transport.py:263` | same |

### Required in practice — the feature 503s, exits, or silently no-ops without it

| var | site | failure mode |
|---|---|---|
| `CLERK_WEBHOOK_SECRET` | `services/identity_tenant_service/routes/auth.py:50` | 503 `DEPENDENCY_UNAVAILABLE` on every Clerk webhook |
| `GATEWAY_SIGNING_KEY` | `services/api_gateway/edge.py:59` | 503 — fails closed and loudly, no default key |
| `CLERK_SECRET_KEY` | `libs/shared/clerk_admin.py:52` | returns `None`; `internal_org_id` is never published to Clerk. Never raises — silent |
| `LIVEKIT_API_KEY` | `services/meeting_service/transport.py:259` | falls back to `FakeTransport()` — a forgotten config yields obviously-fake tokens rather than a deep runtime error |
| `BOOTSTRAP_DATABASE_URL` | `migrations/env.py:32`, `scripts/bootstrap_roles.py:39` | falls back to `DATABASE_URL`, which names `mytheus_app` and holds no DDL privilege — the first `CREATE TABLE` fails |
| `MYTHEUS_APP_PASSWORD` | `scripts/bootstrap_roles.py:45` | exit 2. Without it the role exists with correct grants but cannot authenticate |
| `MYTHEUS_RELAY_PASSWORD` | same | exit 2 |
| `MYTHEUS_ADMIN_PASSWORD` | same | exit 2 |

### Optional, with a code default

| var | site | default | note |
|---|---|---|---|
| `REDIS_URL` | `libs/shared/app_factory.py:23`, `libs/shared/cache.py:39`, `routes/auth.py:115`, `meeting_service/webhook.py:292`, `meeting_service/consumer.py:259`, `core_api/consumer.py:131` | `redis://localhost:6379` | **the dev compose publishes 6380** (`docker-compose.yml`), and `.env.example` says 6380. Both code defaults point at the wrong port |
| `RELAY_DATABASE_URL` | `services/relay/main.py:84` | falls back to `DATABASE_URL` | falling back means the relay runs as `mytheus_app`, which the fail-closed policy gives an empty queue forever — defect D4's exact shape |
| `CLERK_JWT_ORG_CLAIM` | `libs/shared/auth.py:57` | `org_id` | value is Clerk's opaque `org_...`, never a UUID |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `libs/shared/telemetry.py:17` | unset → tracing disabled entirely | |
| `TEMPORAL_ADDRESS` | `services/workflow_service/worker.py:19` | — | |
| `TEMPORAL_NAMESPACE` | `services/workflow_service/worker.py:20` | — | |
| `TEMPORAL_TASK_QUEUE` | `services/workflow_service/worker.py:21` | — | |
| `ROLE_TOPICS_TAXONOMY_VERSION` | `services/identity_tenant_service/taxonomy/loader.py:102` | re-read fresh on every call | |
| `REDPANDA_BROKERS` (consumers) | `core_api/consumer.py:128`, `meeting_service/consumer.py:256`, `notification_service/consumer.py:16` | has a default here, unlike the relay | |
| `SCHEMA_REGISTRY_URL` (consumers) | `core_api/consumer.py:129`, `meeting_service/consumer.py:257` | has a default here | |
| `UPSTREAM_<SERVICE_NAME>` | `services/api_gateway/proxy.py:77` | `http://{service}:8000` | one per routed service, uppercased with `-`→`_` |

### Declared in `.env.example`, read by no code

`CLERK_PUBLISHABLE_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `SENTRY_DSN`, `ENVIRONMENT`, `LOG_LEVEL`.

### Config loading mechanism

There is **no** settings object. `pydantic-settings>=2` is a declared dependency (`pyproject.toml:8`) and `BaseSettings` is subclassed nowhere. Every service reads `os.environ` directly at the point of use, mostly inside a lazily-called `lru_cache`d function so that importing a module never requires the env to be set. That is a deliberate and consistently applied pattern (`db.py`, `cache.py`, `auth.py` all document it), but it means there is no single place listing what a given service needs, and no startup-time validation — a missing var surfaces on the first request that touches it.

---

## VERDICT

The tenancy layer is the real thing and the strongest part of this codebase: three non-BYPASSRLS login roles, `FORCE ROW LEVEL SECURITY` with one identical fail-closed predicate on all fifteen `org_id` tables, a single deliberate tenant-root carve-out with three named compensating controls, and fourteen CI tests that would catch a regression — `9b2c4d1e6f30` in particular closed an empirically reproduced cross-tenant read/write/delete and recorded both surviving residuals honestly instead of claiming a clean bill.

Around that core the schema is well ahead of the code: `data_residency` is a pure orphan, `rbac_grants` — the authorization store the entire `require_tier` chain reads — is writable only by the seed script, `audit_logs` and `chat_messages` are write-only with no read path, and R2 object storage, the PII token vault, metrics and Sentry are ADR text plus an unimported dependency.

Two things need action now: `chat_messages` has no partition covering any date after 2026-09-01, so `POST /chat` has been raising for six days, and `migrations/_grants.py` — the mechanism 0005's entire cost/benefit argument rests on — is imported by zero migrations, so the next table-creating migration in any repo silently ships with an empty ACL.
