# Verification: data layer (mytheus-backend) + mytheus-platform

Read-only. All paths relative to `/Users/sushan/Projects/Mytheus`. Worktrees (`.worktrees/`)
and `.venv` excluded from every search. Date of check: 2026-09-07.

---

## Verdict table

| # | Claim | Verdict |
|---|---|---|
| 1 | `data_residency` orphan | **PENDING** (confirmed) |
| 2 | `tenant_settings` columns with no writer | **PENDING** (confirmed, with corrections) |
| 3 | `audit_logs` / `chat_messages` write-only | **PENDING** (confirmed) |
| 4 | No ORM model for 3 tables → autogenerate DROPs | **PENDING** (confirmed; worse than stated) |
| 5 | `migrations/_grants.py` imported by zero migrations | **NOT A GAP** (forward-looking helper, test-enforced) |
| 6 | No `pg_partman`; audit partitions end 2027-06 | **PENDING** (confirmed, but the two tables fail *differently*) |
| 7 | R2 (ADR-0023) / PII vault (ADR-0020) unbuilt | **PENDING** (confirmed; R2 partly DEFERRED-BY-ADR) |
| 8 | Metrics zero; Sentry unimported; no Prom/Grafana | **PENDING** (confirmed; traces *are* wired — correction) |
| 9 | `confluent-kafka` + `aiokafka` mixed, no shared base | **PENDING** (confirmed) |
| 10 | No backups; no observability; `observability/` absent | **PENDING** (confirmed) |
| 11 | Zero resource limits / logging on 17 containers | **PENDING** (confirmed; 17 exact) |
| 12 | `deploy.sh` only on the VPS, not in VCS | **NOT A GAP** — claim is false |
| 13 | CD concurrency per-repo, three repos race | **PENDING** (confirmed; source admits it) |
| 14 | oasdiff gate has never executed | **PENDING** (confirmed; source admits it) |
| 15 | Env vars required by code, absent from all examples | **PENDING** (confirmed; full list below) |

---

## Data layer

### 1. `data_residency` — orphan. PENDING (confirmed)

ORM model exists and is complete:
- `mytheus-backend/services/identity_tenant_service/models.py:202-222` — `class DataResidency`,
  `__tablename__ = "data_residency"` at :207, columns `region`, `db_cell`, `qdrant_region`,
  `object_bucket_region`, `locked_at`, plus `CheckConstraint("region IN ('us','eu','ap')")` at :222.

Created and evolved by migrations:
- `migrations/versions/82d44163c933_0001_core_tables_outbox_inbox_.py:107-111` (create)
- `migrations/versions/b7d9f3a55c21_0003_rbac_grants_and_canonical_columns.py:157-177` (4 columns + check)

Every other reference in the repo is a **docstring or a migration**. Zero application reads,
zero application writes:
```
grep -rn "DataResidency|data_residency" services libs tests --include=*.py
→ models.py:5 (docstring), models.py:202/207 (definition),
  main.py:4 (docstring), tests/test_consent_columns_migration.py:84 (docstring)
```
Notably `services/identity_tenant_service/provisioning.py` (171 lines — the org-creation path)
does **not** insert a `data_residency` row, contradicting the model's own docstring at :204-206
("populated for every tenant"). So the table is not merely unread — the invariant it documents
is already violated for every tenant created so far.

**Task:** `provisioning: insert the data_residency row on org creation, and read it at storage/vector placement`
- Files: `services/identity_tenant_service/provisioning.py`, `services/identity_tenant_service/models.py`, new migration (backfill existing orgs), `tests/`
- Size: **M**
- Depends on: nothing to write it; the *read* side depends on #7 (R2) and on Qdrant region selection in `mytheus-intelligence`.

### 2. `tenant_settings` — one writable column of nine. PENDING (confirmed, corrected)

The table has **9** columns, not 8 (`mytheus-backend/services/identity_tenant_service/models.py:165-200`):
`org_id`, `default_locale`, `retention_days_transcripts`, `retention_days_worklogs`,
`agent_confidence_overrides`, `pii_redaction_enabled`, `sso_config`, `consent_required`, plus
`TimestampMixin`.

- `pii_redaction_enabled` — **read and written**. `routes/tenant_settings.py:102-123` (GET),
  `:126-181` (PATCH, upsert at :140-145), mirrored to Redis at :179.
- `consent_required` — **read, never written by any route.** Read at
  `services/meeting_service/routes.py:139` (`COALESCE(ts.consent_required, false)`) and enforced
  at `:186-187`. `routes/tenant_settings.py:35` states it is *deliberately* not editable. Only
  tests write it (`tests/test_meeting_token_route.py:331,349,367`). So the consent gate that
  ADR-0018 describes is unreachable in production — there is no operator path to turn it on.
  This is a real gap the original claim missed.
- `sso_config`, `retention_days_transcripts`, `retention_days_worklogs`, `default_locale`,
  `agent_confidence_overrides` — **no reader and no writer anywhere.** Confirmed by grep across
  `services/` and `libs/`: the only hits are the model definition itself
  (`models.py:177,178,181,184,192`) and `routes/scim.py:21` which explicitly notes
  "`tenant_settings.sso_config` exists unused".

**Tasks:**
- `expose consent_required on the tenant-settings admin surface` — files: `routes/tenant_settings.py`, `design/openapi/*.yaml`, tests. Size **S**. Depends on: nothing.
- `retention job that reads retention_days_transcripts/worklogs` — files: `services/workflow_service/workflows/`, new consumer. Size **L**. Depends on: `retention` workflow (see #New-4) and RTBF stub.
- `default_locale + agent_confidence_overrides readers` — files: notification/agent paths. Size **M**. Depends on: agent runtime in `mytheus-intelligence` (out of this scope).
- `sso_config` — DEFERRED-BY-ADR in practice: `routes/scim.py:21` returns 501 for all SCIM ops by design.

### 3. `audit_logs` and `chat_messages` are write-only. PENDING (confirmed)

- `audit_logs`: single writer `libs/shared/audit.py:32` (`INSERT INTO audit_logs ...`).
  The owning service is a shell: `services/admin_ops_service/main.py` is 11 lines and ends with
  `# TODO: add routers for /admin/tenant/* and /admin/ops/*`. There is **no** route file in
  `services/admin_ops_service/` at all (`ls` → `__init__.py, main.py, models.py`).
  Only tests SELECT from it (`tests/test_audit.py`, `test_meetings_api.py:159,282`, etc.).
- `chat_messages`: single writer `services/core_api/routes/chat.py:106`
  (`INSERT INTO chat_messages ...`). The only decorator in that file is
  `@router.post("/chat", ...)` at `:88`. No GET. The migration itself anticipates the missing
  route: `migrations/versions/5ed978467cad...py:94` — *"a future `GET /meetings/:id/chat`"*.

Full route inventory across the repo (every `@router` decorator) contains no audit and no chat
read endpoint.

**Tasks:**
- `GET /admin/ops/audit-logs (cursor-paginated, tenant-scoped)` — files: new `services/admin_ops_service/routes/audit.py`, `main.py`, `design/openapi/admin-ops.yaml`, tests. Size **M**. Depends on: nothing (grant already `SELECT, INSERT` per `c3e8b1d47f92...py:139,144`).
- `GET /meetings/{id}/chat` — files: `services/core_api/routes/chat.py`, `schemas.py`, `design/openapi/core-api.yaml`, tests. Size **S**. Depends on: **#6** — the index it would use lives on the 2026-08 partition only.

### 4. Missing ORM models → `alembic --autogenerate` proposes DROPs. PENDING — confirmed, and the risk is larger than stated

`migrations/env.py:20-25` imports exactly four model modules:
`libs.shared.infra_models`, `services.admin_ops_service.models`, `services.core_api.models`,
`services.identity_tenant_service.models`. `target_metadata = Base.metadata` at `:26`.

Complete set of `__tablename__` in the repo:
```
identity_tenant_service/models.py:39,59,100,132,143,155,173,207
  → organizations, users, rbac_grants, teams, team_members, departments,
    tenant_settings, data_residency
admin_ops_service/models.py:26   → audit_logs
core_api/models.py:28,58         → meetings, meeting_participants
```
No model for `meeting_rooms`, `meeting_speakers`, `chat_messages` — confirmed. All three were
created by raw `op.execute` / `op.create_table` migrations
(`3d13f46b960f`, `25e7cede5672`, `5ed978467cad`).

**Risk is higher than the claim states, for three reasons:**

1. `migrations/env.py:51,54` calls `context.configure(...)` with **no `include_object` filter**
   and no `include_schemas`. Autogenerate will therefore also propose dropping every
   **partition** (`audit_logs_2026_07` … `audit_logs_2026_06`, `audit_logs_default`,
   `chat_messages_2026_08`) and — because sibling repos share this one database — the
   `transcript_chunks` table owned by `mytheus-realtime`
   (referenced at `migrations/_grants.py:22-26`).
2. `Makefile:20-22` makes this the *documented* path:
   ```
   migrate:
   	$(ALEMBIC) revision --autogenerate -m "$(msg)"
   	$(ALEMBIC) upgrade head
   ```
   It autogenerates **and immediately applies**. There is no review step between generation and
   `upgrade head`.
3. No guard test exists. `grep -rln "autogenerate|compare_metadata" tests/ scripts/` → only the
   `Makefile`.

So the failure mode is: a developer runs `make migrate msg=...` to add one column and silently
drops three tables, a dozen partitions, and another repo's table. This is the single highest-risk
finding in the data layer.

**Tasks:**
- `add include_object filter to migrations/env.py (skip partitions + tables not in metadata)` — files: `migrations/env.py`. Size **S**. Depends on: nothing. **Do this first.**
- `add ORM models for meeting_rooms, meeting_speakers, chat_messages` — files: `services/meeting_service/models.py` (new), `services/core_api/models.py`, `migrations/env.py`. Size **M**. Depends on: nothing.
- `split make migrate into make revision + make upgrade` — files: `Makefile`. Size **S**.
- `test that autogenerate produces an empty diff against head` — files: `tests/test_migrations_autogenerate.py` (new). Size **M**. Depends on: the two above.

### 5. `migrations/_grants.py` imported by zero migrations. NOT A GAP

Confirmed factually — `grep -rn "grant_tenant_table" migrations/versions/*.py` returns nothing;
the only mentions of `_grants` are docstrings (`9b2c4d1e6f30...py:62,260`).

But it is **correct** that no existing migration imports it. Migration order is:
```
82d44163c933 → a1c4e7f20b18 → b7d9f3a55c21 → c3e8b1d47f92 → 3d13f46b960f →
866f7eedb0e7 → 5ed978467cad → 25e7cede5672 → f3a7c9e0b21d → 56f72ac73fa9 → 9b2c4d1e6f30
```
`9b2c4d1e6f30` (which retires `ALTER DEFAULT PRIVILEGES`) is **last**. Every table created
before it inherited its grant from the default privilege, which `ALTER DEFAULT PRIVILEGES ...
REVOKE` does not retroactively remove. `_grants.py` is a helper for the *next* migration, and
`tests/test_roles_and_policies.py::test_every_org_id_table_grants_read_to_the_request_role`
(referenced at `migrations/_grants.py:12-14`) fails the build if a future migration forgets it.

Not pending. The only residual item is documentation, and `CONTRIBUTING.md` should be checked to
confirm it points at the helper — worth one line, not a task.

### 6. No `pg_partman`. PENDING — confirmed, but the two tables fail differently

`grep -rni "partman"` across the repo → three hits, all comments deferring it:
`c3e8b1d47f92...py:10`, `5ed978467cad...py:21`, `5ed978467cad...py:126`.
Nothing in `mytheus-platform` either (no cron, no maintenance container in
`deploy/compose.prod.yml`).

**`audit_logs`** — `c3e8b1d47f92_0004_audit_logs_append_only.py:44-46`:
`_FIRST_PARTITION_YEAR = 2026`, `_FIRST_PARTITION_MONTH = 7`, `_PARTITION_COUNT = 12`
→ partitions `2026_07` through `2027_06`. **But line 92 creates a DEFAULT partition:**
```
op.execute("CREATE TABLE audit_logs_default PARTITION OF audit_logs DEFAULT;")
```
So audit_logs will **not** raise after 2027-06. It degrades silently: rows pile into the default
partition, and once they are there Postgres refuses to create the correct monthly partition
without first moving them out. That is a *different and quieter* failure than the claim states —
arguably worse, because nothing alerts.

**`chat_messages`** — `5ed978467cad_feat_core_api_add_chat_messages.py:54-55`
(`_PARTITION_YEAR = 2026`, `_PARTITION_MONTH = 8`) creates **exactly one** partition,
`chat_messages_2026_08` (`:88-92`), and **no DEFAULT partition**. The migration's own
`COMMENT ON TABLE` at `:122-129` says so: *"An INSERT past this partition's upper bound will
raise until that job (or a follow-up migration) adds the next one."*
No later migration adds one (`grep "chat_messages_20"` → only that file).

**Today is 2026-09-07.** `POST /chat` has been raising for every request since 2026-09-01.
This is a live defect, not a forward-looking risk.

**Tasks:**
- `hotfix migration: create chat_messages partitions 2026_09..2027_08 + REVOKE on each` — files: new migration. Size **S**. **Urgent — currently broken in prod.** Depends on: nothing.
- `pg_partman extension + nightly maintenance for audit_logs and chat_messages` — files: new migration (`CREATE EXTENSION pg_partman`, `part_config` rows), `mytheus-platform/deploy/compose.prod.yml` (maintenance sidecar or cron), postgres image must carry the extension. Size **L**. Depends on: the postgres image (currently a supabase/postgres image per `9b2c4d1e6f30...py:100-101`) shipping `pg_partman`; verify before planning.
- `drain audit_logs_default before it accumulates` — files: new migration/runbook. Size **S**. Depends on: pg_partman task.

### 7. R2 / object storage (ADR-0023) and PII vault (ADR-0020). PENDING

**Object storage:** `boto3>=1.43.64` is declared at `mytheus-backend/pyproject.toml:32` and
**imported nowhere** (`grep -rn "import boto3|boto3\." services libs` → zero hits).
`services/meeting_service/storage.py` is 54 lines that compute a key string and make no I/O
call; its docstring at `:3-8` explicitly says *"no S3 bucket and no LiveKit account exist for
this build, and this task does not create one"*, and `:19-20` defers the lifecycle policy to
`mytheus-platform`. That much is **DEFERRED-BY-ADR / by task brief**, honestly documented.
What is genuinely pending is the platform side (bucket, credentials, lifecycle, RTBF purge),
which does not exist in `mytheus-platform` at all.

**PII vault:** worse — `grep -rni "token_vault|pii_vault|vault"` across
`mytheus-backend/{services,libs}` and `mytheus-intelligence/{libs,services}` returns
**zero hits**. ADR-0020 has no code anywhere, not even a placeholder. The only PII machinery is
the `pii_redaction_enabled` flag (#2) and, per `routes/tenant_settings.py:11`, a redaction module
at `services/ingestion_service/pii.py` in a sibling repo that reads the flag from a Redis mirror
key `nexus:{org_id}:tenant_settings:pii_redaction_enabled`.

**Tasks:**
- `provision R2 bucket + lifecycle + credentials, wire meeting-service upload/presign` — files: `mytheus-platform/deploy/`, `mytheus-backend/services/meeting_service/storage.py`, env examples. Size **L**. Depends on: #15 (env plumbing), #1 (`data_residency.object_bucket_region`).
- `PII token vault: schema, service, NER placement per ADR-0020` — files: new. Size **L**. Depends on: ADR-0020 being re-confirmed as in-scope for this phase; currently nothing at all exists.

### 8. Metrics / Sentry / Prometheus. PENDING — with one correction

- **Sentry:** `sentry-sdk[fastapi]>=2` at `pyproject.toml:31`. `grep -rn "sentry_sdk" services libs`
  → **zero**. Confirmed. `SENTRY_DSN` *is* present in `mytheus-platform/env/{dev,staging,prod}.env.example`,
  so the DSN is provisioned into an application that never reads it.
- **Metrics:** confirmed zero. No `prometheus_client`, no `opentelemetry.metrics`, no
  `MeterProvider` anywhere.
- **Correction to the claim:** *tracing* is wired and real.
  `libs/shared/telemetry.py:1-17` builds a `TracerProvider` + `BatchSpanProcessor` +
  `OTLPSpanExporter`, instrumented via `FastAPIInstrumentor`, and
  `libs/shared/app_factory.py:26` calls `setup_telemetry(app, service_name=...)` for every
  service. It is a no-op unless `OTEL_EXPORTER_OTLP_ENDPOINT` is set (`telemetry.py:17`), and
  no collector exists to receive it (#10) — so the *effect* is zero, but the code is not.

**Tasks:**
- `initialize sentry_sdk in app_factory when SENTRY_DSN is set` — files: `libs/shared/app_factory.py`, `pyproject.toml`. Size **S**. Depends on: nothing.
- `add OTel MeterProvider + the four family metrics (req/s, p95, consumer lag, queue depth)` — files: `libs/shared/telemetry.py`, `app_factory.py`, consumers. Size **M**. Depends on: #10 (a collector to export to).
- `stand up OTel Collector + Prometheus + Grafana` — see #10.

### 9. Kafka client split. PENDING (confirmed)

Both are declared in `mytheus-backend/pyproject.toml:16-17` (`aiokafka>=0.11`,
`confluent-kafka[avro,schemaregistry]>=2.4`).

- `aiokafka`: exactly one call site — `services/notification_service/consumer.py:14`
  (`from aiokafka import AIOKafkaConsumer`), 45 lines.
- `confluent-kafka`: `services/relay/main.py:28-31`, `services/core_api/consumer.py:119-123`,
  `services/meeting_service/consumer.py:247-251`.

The two `confluent-kafka` consumers in this repo (`core_api/consumer.py` 182 lines,
`meeting_service/consumer.py` 305 lines) duplicate the same subscribe/poll/deserialize/commit
harness with no shared base.

Cross-repo: `mytheus-intelligence` **does** have one —
`mytheus-intelligence/libs/shared/consumer.py:44,261-302` is a real shared consumer base with
at-least-once commit ordering. `mytheus-realtime/services/transcription-service/src/
transcription_service/meeting_room_consumer.py:187-201` writes its own again. `mytheus-events`
has no Python at all. So there are **three independent implementations** of the same loop across
three repos, plus one aiokafka outlier.

**Tasks:**
- `promote mytheus-intelligence/libs/shared/consumer.py to a shared package, adopt in backend + realtime` — files: new shared dist or vendored copy, `services/core_api/consumer.py`, `services/meeting_service/consumer.py`, `mytheus-realtime/.../meeting_room_consumer.py`. Size **L**. Depends on: a decision on how the four repos share Python code (there is no shared package today) — this is a blocking prerequisite, not a detail.
- `port notification_service off aiokafka onto the same base, drop the aiokafka dep` — files: `services/notification_service/consumer.py`, `pyproject.toml`. Size **S**. Depends on: the above.

---

## Platform

### 10. No backups, no observability, `observability/` absent. PENDING (confirmed)

Full file listing of `mytheus-platform` (excluding `.git`) is **25 files**:
```
.pre-commit-config.yaml  Makefile  README.md  .gitignore  CONTRIBUTING.md
system.md  PREVIOUS.md  docker-compose.yml  CLAUDE.md
deploy/{README.md, compose.prod.yml, deploy.sh}
deploy/postgres-init/README.md  deploy/caddy/mytheus.caddy
env/{dev,prod,staging}.env.example  env/prod.db.{app,relay,migrate,temporal}.env.example  env/README.md
scripts/check_authz.py
.github/workflows/{ci.yml, deploy.yml}
```
- **`observability/` does not exist.** `README.md:58` and `CLAUDE.md:74` both document it in the
  directory tree; `CLAUDE.md:130` instructs contributors to "Add Prometheus rules / Grafana
  dashboards under `observability/`". The directory is fiction.
- **No backups of any volume.** `deploy/compose.prod.yml:414-416` declares `pgdata`, `redisdata`,
  `redpandadata` and nothing backs any of them up. `grep -rni "backup|pg_dump|restic|borg"` over
  the repo returns only aspirational prose: `system.md:137` (PITR + replica + MirrorMaker2, as a
  future design), `CLAUDE.md:141` (same, as a future step), and `deploy/README.md:133` which
  mentions a **Hostinger VPS snapshot** as a rollback target — a whole-VM snapshot, not a
  database backup, and not automated from this repo.
- No log aggregation. Containers have no `logging:` driver config at all (#11), so `docker logs`
  with the default json-file driver and **no rotation** is the entire logging story.

**Tasks:**
- `nightly pg_dump of pgdata to R2 with retention + a restore drill` — files: `deploy/compose.prod.yml` (backup sidecar), `deploy/deploy.sh`, new `deploy/backup/`. Size **M**. Depends on: #7 (R2 bucket) for the destination — or use local disk + VPS snapshot as an interim.
- `create observability/ with OTel Collector + Prometheus + Grafana + Loki compose` — files: new `observability/`, `deploy/compose.prod.yml`. Size **L**. Depends on: #11 (resource limits — these four containers on the same VPS as 17 others will need budgets).
- `fix README.md:58 / CLAUDE.md:74 to stop documenting a directory that does not exist` — files: `README.md`, `CLAUDE.md`. Size **S**. Depends on: nothing (or fold into the above).

### 11. Zero resource limits and zero logging config. PENDING (confirmed)

`deploy/compose.prod.yml` declares exactly **17 services**:
`postgres` (:78), `redis` (:105), `redpanda` (:117), `temporal` (:137), `migrate` (:161),
`web` (:177), `relay` (:210), `workflow` (:226), `api-gateway` (:252), `bff` (:270),
`identity-tenant-service` (:277), `meeting-service` (:293), `integration-service` (:322),
`billing-metering-service` (:329), `admin-ops-service` (:342), `notification-service` (:349),
`ui` (:364).

```
grep -n "deploy:|resources:|limits:|mem_limit|cpus|logging:|max-size" deploy/compose.prod.yml
→ zero matches
```
The only hits for that pattern are `restart:` lines (:66, 103, 115, 135, 155, 175, 401).
Confirmed: no memory limit, no CPU limit, no log rotation, on any of the 17. On a single VPS
also hosting the `atithivishram-prod` stack (`compose.prod.yml:367-376, 405-413`), one leaking
container takes down both products, and json-file logs grow until the disk fills.

**Tasks:**
- `add mem_limit/cpus and logging max-size+max-file to all 17 services` — files: `deploy/compose.prod.yml` (extend the existing `*backend` YAML anchor at :61-76 so most services inherit it). Size **M**. Depends on: measuring actual RSS per container first — do not guess limits on a live box.

### 12. `deploy.sh` not in version control. NOT A GAP — the claim is false

`mytheus-platform/deploy/deploy.sh` exists, is 110 lines, and is tracked in git with real history:
```
$ git log --oneline -3 -- deploy/deploy.sh
24cd382 feat(deploy): fail the deploy when the event backbone is not provisioned
b878ab2 fix(deploy): gate the deploy on meeting-service actually coming up
88b13d3 refactor(deploy): pull the backend image from GHCR, drop the sibling build
```
`mytheus-platform` is a git repo (`git rev-parse --is-inside-work-tree` → `true`).
This may have been true at some earlier point; it is not true now. Nothing to do.

### 13. CD concurrency is per-repo — three repos can race. PENDING (confirmed)

Three workflows declare the *same* concurrency group name in *three different repositories*:
- `mytheus-platform/.github/workflows/deploy.yml:34-36` — `group: deploy-vps`
- `mytheus-backend/.github/workflows/ci.yml:304-306` — `group: deploy-vps`
- `mytheus-ui/.github/workflows/ci.yml:76-78` — `group: deploy-vps`

GitHub scopes `concurrency` per repository, so these three groups are unrelated and all three can
run `ssh → docker compose up` against the same VPS simultaneously. The source **already documents
this**, at `mytheus-backend/.github/workflows/ci.yml:301-303`:
> *"Shares a concurrency group name with mytheus-platform's deploy workflow only by convention;
> GitHub scopes concurrency per repository, so the real serialisation is that the deploy script
> itself is a single `docker compose up` against one project."*

That stated mitigation is not a lock. Two concurrent `docker compose up -d` runs against one
project name interleave; and `mytheus-backend`'s deploy also runs the `migrate` service
(`compose.prod.yml:161-176`), so a race can put two Alembic runs against one database.

(Note: `mytheus-events/.github/workflows/ci.yml` also matched a `deploy|ssh` grep but declares no
concurrency group — worth checking whether it deploys. It has no `deploy-vps` group.)

**Tasks:**
- `serialize deploys with a real cross-repo lock (repository_dispatch into mytheus-platform, or an flock in deploy.sh)` — files: all three workflow files, `mytheus-platform/deploy/deploy.sh`. Size **M**. Depends on: deciding whether platform becomes the single deploy entrypoint (recommended) — that is a design call to make first.

### 14. The oasdiff gate has never executed. PENDING (confirmed — by the source itself)

`mytheus-backend/.github/workflows/ci.yml:186-219`, job `openapi`, step `oasdiff breaking-change gate`.
It is guarded at `:200`:
```yaml
if: github.event_name == 'pull_request'
```
and the workflow triggers are `pull_request: {}` and `push: branches: [main]` (`:11-14`).
The comment at `:196-198` states it outright:
> *"This step has never run in Actions (it is pull_request-only and no PR has carried a spec
> change), which is why the exit code had never been observed."*

The `--fail-on ERR` correctness reasoning at `:194-199` is sound and was verified locally per the
comment, but the gate has zero production evidence. The two sibling steps in the same job
(`emit_openapi --check` at `:181`, `spectral lint` at `:189-192`) run on every push and *have*
executed.

**Tasks:**
- `land one PR that intentionally breaks a spec, confirm the gate goes red, then revert` — files: `design/openapi/*.yaml` (throwaway). Size **S**. Depends on: nothing. This is the cheapest high-value item on the whole list.

### 15. Env vars required by code, absent from every example file. PENDING — full list

Env vars referenced anywhere in `mytheus-backend/{services,libs}` (`os.environ` / `os.getenv`),
20 total:
```
ADMIN_DATABASE_URL   CLERK_ISSUER      CLERK_JWKS_URL    CLERK_JWT_ORG_CLAIM
CLERK_SECRET_KEY     CLERK_WEBHOOK_SECRET                DATABASE_URL
GATEWAY_SIGNING_KEY  LIVEKIT_API_KEY   LIVEKIT_API_SECRET LIVEKIT_URL
OTEL_EXPORTER_OTLP_ENDPOINT           REDIS_URL          REDPANDA_BROKERS
RELAY_DATABASE_URL   ROLE_TOPICS_TAXONOMY_VERSION        SCHEMA_REGISTRY_URL
TEMPORAL_ADDRESS     TEMPORAL_NAMESPACE                  TEMPORAL_TASK_QUEUE
```
(There is no `BaseSettings` / pydantic-settings class anywhere — `grep -rn "BaseSettings"` over
`libs`/`services` returns nothing — so raw `os.environ` is the complete surface.)

`mytheus-platform/env/` contains: `dev.env.example`, `staging.env.example`, `prod.env.example`,
`prod.db.{app,relay,migrate,temporal}.env.example`.

**Absent from EVERY example file (3 vars):**

| Var | Read at | Behavior when unset | Severity |
|---|---|---|---|
| `GATEWAY_SIGNING_KEY` | `services/api_gateway/edge.py:59` | **Raises `ProblemDetail(503, DEPENDENCY_UNAVAILABLE, "Gateway signing key not configured")`** at `:61-63`. Every authenticated request through the gateway 503s. | **Critical** |
| `CLERK_JWT_ORG_CLAIM` | `libs/shared/auth.py:57` | Falls back to `DEFAULT_ORG_CLAIM`. `auth.py:17` records that hardcoding this claim name was a past defect — so the fallback is exactly the bug that was fixed, silently restored. | High |
| `ROLE_TOPICS_TAXONOMY_VERSION` | `services/identity_tenant_service/taxonomy/loader.py:102` | Defaults to `"v1"`. Benign today; becomes a silent-wrong-version risk once v2 exists. | Low |

**Absent from `prod.env.example` specifically (present in dev/staging) (1 var):**

| Var | Read at | Note |
|---|---|---|
| `ADMIN_DATABASE_URL` | `libs/shared/db.py:72` — `os.environ["ADMIN_DATABASE_URL"]`, a **hard KeyError** | Present in `dev.env.example` and `staging.env.example`, absent from `prod.env.example` and from all four `prod.db.*.env.example`. Used by the operator seam (`db.py:59`) and by `identity_tenant_service/seed/seed.py:10`. |

**Also missing: an example file that compose requires.**
`deploy/compose.prod.yml:381` mounts `${MYTHEUS_ENV_DIR}/prod.ui.env`, and **no
`env/prod.ui.env.example` exists**. Compose-level variables `BACKEND_IMAGE`, `UI_IMAGE`,
`MYTHEUS_ENV_DIR`, `MYTHEUS_PG_INIT` (from `grep -oE '\$\{[A-Z0-9_]+' deploy/compose.prod.yml`)
are documented only in `deploy/compose.prod.yml:14` prose, not in any `.env.example`.

**Reverse direction (declared but unread — dead config):** `SENTRY_DSN` (see #8),
`CLERK_PUBLISHABLE_KEY` (frontend-only, correctly placed), `STRIPE_SECRET_KEY` /
`STRIPE_WEBHOOK_SECRET` (`billing_metering_service/main.py` is an 11-line shell; `stripe>=10` is
declared in `pyproject.toml:22` and imported nowhere).

**Tasks:**
- `add GATEWAY_SIGNING_KEY, CLERK_JWT_ORG_CLAIM, ROLE_TOPICS_TAXONOMY_VERSION to all env examples; add ADMIN_DATABASE_URL to prod` — files: `mytheus-platform/env/*.example`, `env/README.md`. Size **S**. Depends on: nothing. **Do alongside #New-1.**
- `add env/prod.ui.env.example` — files: new. Size **S**.
- `startup env validation: fail fast at boot on a missing required var` — files: `libs/shared/app_factory.py`, new `libs/shared/config.py`. Size **M**. Depends on: nothing. This is what prevents the class of defect rather than this instance.

---

## What else is pending that this list misses

### New-1. `GATEWAY_SIGNING_KEY` may be unset in the running production stack — **verify first**
Not a documentation gap but a possible live outage. `deploy/compose.prod.yml` passes
`prod.env` to the backend anchor, and `prod.env.example` does not list the var, so whether
production works depends entirely on whether someone hand-added it to the VPS file. If it is
absent, `api_gateway` 503s every authenticated request (`edge.py:59-63`).
**Task:** `confirm GATEWAY_SIGNING_KEY is set in /etc/dokploy/mytheus-secrets/env/prod.env`
· files: none (operator check) · **S** · depends on: VPS access. **Highest priority.**

### New-2. `POST /chat` is broken in production right now
Consequence of #6, but worth stating as its own item because it is a *current outage*, not a
future risk. `chat_messages` has one partition covering `[2026-08-01, 2026-09-01)`
(`5ed978467cad...py:54-55, 88-92`) and no DEFAULT partition. Today is 2026-09-07.
**Task:** `hotfix migration adding chat_messages partitions through 2027-08` · files: new
migration · **S** · depends on: nothing.

### New-3. `consent_required` has no operator write path
Covered under #2 but missed by the original claim, and it is a compliance control, not a nicety.
ADR-0018's consent gate is enforced (`meeting_service/routes.py:186-187`) and cannot be enabled.

### New-4. `RTBFWorkflow` is a comment-only stub
`services/workflow_service/workflows/rtbf.py:12-21` — the class body is a numbered list of
comments (`# 1. Postgres soft-delete then hard purge by org_id/user_id` … `# 6. audit-service:
append RTBF completion record`) with no implementation, and it is registered as a real workflow
at `services/workflow_service/worker.py:32`. So a client can start an RTBF saga that silently
does nothing and reports success. Given `mytheus-docs/design/rtbf-contract.md` exists, this is a
documented contract with a no-op implementation.
**Task:** `implement RTBFWorkflow activities (postgres purge, qdrant, object store, audit)`
· files: `services/workflow_service/workflows/rtbf.py`, new activities module · **L**
· depends on: #7 (object store), Qdrant client in `mytheus-intelligence`.

### New-5. Four services are 11-line shells with declared dependencies
`admin_ops_service/main.py` (11 lines, `# TODO: add routers`), `billing_metering_service/main.py`
(11), `integration_service/main.py` (11), `notification_service/main.py` (11). `stripe>=10`
(`pyproject.toml:22`) and `svix>=1.20` (`:21`) are declared for services that have no routes.
This is expected phasing, but it means the `STRIPE_*` secrets in every env example are currently
unused credentials sitting in a production env file — a small blast-radius item worth noting.

### New-6. `mytheus-intelligence` and `mytheus-realtime` have **no CI at all**
`ls .github/workflows` → the directory does not exist in either repo. Only
`mytheus-backend`, `mytheus-platform`, `mytheus-events`, `mytheus-ui` have workflows. Two repos
containing the RAG/agent layer and the transcription service merge with zero automated gates —
no lint, no typecheck, no tests, no dependency scan.
**Task:** `add ci.yml to mytheus-intelligence and mytheus-realtime from the platform template`
· files: `.github/workflows/ci.yml` in each · **M** · depends on: nothing. This is a bigger hole
than most items on the original list.

### New-7. `mytheus-platform`'s own `make test` cannot fail
`mytheus-platform/Makefile`:
```
test:
	python3 scripts/check_authz.py || true
	@echo "test OK"
```
The `|| true` makes the only executable check unconditionally green, and `ci: lint typecheck test`
therefore cannot fail on it. Compare `mytheus-backend/pyproject.toml:47-49`, whose comment
explicitly argues that running a scanner with `|| true` is "a formality" rather than a control —
the same repo family applies both standards.
Also `lint` only runs `actionlint .github/workflows/ci.yml`, never `deploy.yml`.
**Task:** `drop the || true in platform's make test; actionlint deploy.yml too` · files:
`mytheus-platform/Makefile` · **S** · depends on: nothing (may surface real `check_authz.py`
failures — that is the point).

### New-8. No shared Python packaging across the four Python repos
Each of `mytheus-backend`, `mytheus-intelligence`, `mytheus-realtime` has its own
`libs/shared/` with independently-written copies of the same concerns (consumer harness, avro
codec, db/tenancy helpers). There is no published internal package and no path dependency. This
is the prerequisite that blocks #9 and will block every future cross-cutting fix.
**Task:** `decide and implement the shared-library strategy (internal package vs. git submodule
vs. accepted duplication)` · files: all four `pyproject.toml` · **L** · depends on: a decision.

### New-9. `migrate` service races the app on deploy
`deploy/compose.prod.yml:161-176` declares `migrate` with `restart: "no"`. Combined with #13
(three repos can deploy concurrently) and #4 (`make migrate` autogenerates *and applies*), the
migration path has no lock at any layer.
**Task:** covered by the #13 task; noting the coupling here.

### New-10. `pgdata` volume with no PITR and no replica, single VPS
`system.md:137` designs streaming replication + PITR; none exists. Combined with #10 (no backups
at all), the current RPO for the system of record is "whenever someone last took a Hostinger VM
snapshot" and the RTO is a full VM restore. Called out separately from #10 because the *design
doc claims otherwise*, which is how this stays invisible.

---

## Recommended order

1. **New-1** — verify `GATEWAY_SIGNING_KEY` on the VPS (possible live outage).
2. **New-2 / #6 hotfix** — `chat_messages` partitions (confirmed live outage since 2026-09-01).
3. **#4** — `include_object` in `migrations/env.py` + split `make migrate` (prevents data loss).
4. **#15** — env examples + startup validation.
5. **#11** — resource limits and log rotation (protects the shared VPS).
6. **#10** — pg_dump backups.
7. **#14** — prove the oasdiff gate (cheap).
8. **New-6** — CI for the two repos that have none.
9. Everything else.
