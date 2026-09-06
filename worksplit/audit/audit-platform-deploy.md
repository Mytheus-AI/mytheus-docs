# Mytheus Platform / Deploy Audit — READ-ONLY

Root: `/Users/sushan/Projects/Mytheus`
Method: every claim below comes from reading the compose/CI/env/source files directly. No status doc was trusted as evidence. No file was modified; no deploy, ssh, or docker command was run.

Primary artifact: `/Users/sushan/Projects/Mytheus/mytheus-platform/deploy/compose.prod.yml` (416 lines, project name `mytheus`, `compose.prod.yml:44`).
Root `mytheus-platform/docker-compose.yml:20-21` is a 21-line shim that `include:`s it so Dokploy can drive the repo from its root.

---

## 1. DEPLOY INVENTORY TABLE

### 1.1 Every service in `deploy/compose.prod.yml`

| # | service key | in compose.prod? | image / build source | ports | healthcheck | depends_on | volumes | env_file layering | notes |
|---|---|---|---|---|---|---|---|---|---|
| 1 | `postgres` | YES `:78` | `supabase/postgres:15.8.1.060` `:79` | none published | YES — `pg_isready -U postgres`, 5s/5s/20 `:98-102` | — | `pgdata:/var/lib/postgresql/data` `:97`; bind `${MYTHEUS_PG_INIT:-/etc/dokploy/mytheus-secrets/prod-bootstrap.sql}:/etc/postgresql.schema.sql:ro` `:96` | `prod.env` `:86-88` | Bootstrap SQL generated on host, gitignored. Password arrives via env_file not interpolation, deliberately (`:80-85`) to keep it out of Dokploy's DB |
| 2 | `redis` | YES `:105` | `redis:7-alpine` `:106` | none published | YES — `redis-cli ping`, 5s/3s/20 `:110-114` | — | `redisdata:/data` `:109` | none | `--appendonly yes` `:107` |
| 3 | `redpanda` | YES `:117` | `redpandadata/redpanda:v24.1.1` `:118` | none published | YES — `rpk cluster health \| grep -E 'Healthy:.+true'`, 10s/5s/20 `:130-134` | — | `redpandadata:/var/lib/redpanda/data` `:129` | none | `--smp 1 --overprovisioned --node-id 0` `:121-123`; Kafka 29092 internal / 9092 outside; Schema Registry `:8081` `:126`; pandaproxy `:8082` `:127` |
| 4 | `temporal` | YES `:137` | `temporalio/auto-setup:1.24` `:138` | none published | **NONE** | `postgres: service_healthy` `:152-154` | — | `prod.db.temporal.env` `:143-145` | Connects as postgres superuser; `DB: postgres12` `:147`. `workflow` gates on `service_started` only, so a temporal that never finishes schema setup does not block anything |
| 5 | `migrate` (one-shot) | YES `:161` | `${BACKEND_IMAGE:-ghcr.io/mytheus-ai/mytheus-backend:main}` `:61` | none | n/a | `postgres: service_healthy` `:172-174` | — | `prod.env` + `prod.db.migrate.env` `:163-167` | `alembic -c migrations/alembic.ini upgrade head && python -m scripts.bootstrap_roles` `:171`; `restart: "no"` `:175`. ONLY holder of schema-owner credential + the three role passwords |
| 6 | `web` (network alias `core-api`) | YES `:177` | backend image (`x-backend-app`) | `expose: 8000` `:203-204` | YES — python urllib `GET localhost:8000/healthz`, 10s/5s/10 `:194-202` | `migrate: service_completed_successfully`, `redis: service_healthy`, `redpanda: service_healthy` `:187-193` | — | `prod.env` + `prod.db.app.env` `:71-75` | Aliased `core-api` because the gateway ROUTE_MAP resolves that name `:179-182`. 16 route decorators in `services/core_api` |
| 7 | `relay` | YES `:210` | backend image (`x-backend`) | none | **NONE** | `web: service_started`, `redpanda: service_healthy` `:218-222` | — | `prod.env` + `prod.db.relay.env` `:212-216` | Only sanctioned Kafka producer; runs as `mytheus_relay` whose single `outbox_relay_drain` policy grants cross-tenant read without BYPASSRLS `:206-209` |
| 8 | `workflow` | YES `:226` | backend image (`x-backend-app`) | none | **NONE** | `temporal: service_started`, `web: service_started` `:229-233` | — | `prod.env` + `prod.db.app.env` | Temporal worker: `python -m services.workflow_service.worker` `:228` |
| 9 | `api-gateway` | YES `:252` | backend image (`x-backend-app`) | `expose: 8000` `:267-268` | YES — `/healthz`, 10s/5s/10 `:258-266` | `web: service_healthy` `:255-257` | — | `prod.env` + `prod.db.app.env` | **THE SINGLE PUBLIC INGRESS.** See blocker B1 §7 — missing `GATEWAY_SIGNING_KEY` |
| 10 | `bff` | YES `:270` | backend image (`x-backend-app`) | none | **NONE** | `migrate: service_completed_successfully` `:273-275` | — | `prod.env` + `prod.db.app.env` | 3 route decorators |
| 11 | `identity-tenant-service` | YES `:277` | backend image (`x-backend-app`) | none | **NONE** | `migrate: service_completed_successfully` `:286-288` | — | `prod.env` + `prod.db.app.env` | 11 route decorators. Publishes `internal_org_id` into Clerk `public_metadata` |
| 12 | `meeting-service` | YES `:293` | backend image (`x-backend-app`) | none | YES — `/healthz`, 10s/5s/10 `:308-316` | `migrate: service_completed_successfully` `:296-298` | — | `prod.env` + `prod.db.app.env` | 2 route decorators. Owns the only LiveKit room-token mint + LiveKit webhook. Healthcheck added because nothing else in the pipeline noticed it going dark `:299-307` |
| 13 | `integration-service` | YES `:322` | backend image (`x-backend-app`) | none | **NONE** | `migrate: service_completed_successfully` `:325-327` | — | `prod.env` + `prod.db.app.env` | **0 route decorators** — scaffold |
| 14 | `billing-metering-service` | YES `:329` | backend image (`x-backend-app`) | none | **NONE** | `migrate: service_completed_successfully` `:337-339` | — | `prod.env` + `prod.db.app.env` | **0 route decorators** — scaffold |
| 15 | `admin-ops-service` | YES `:342` | backend image (`x-backend-app`) | none | **NONE** | `migrate: service_completed_successfully` `:345-347` | — | `prod.env` + `prod.db.app.env` | **0 route decorators** — scaffold |
| 16 | `notification-service` | YES `:349` | backend image (`x-backend-app`) | none | **NONE** | `migrate: service_completed_successfully` `:352-354` | — | `prod.env` + `prod.db.app.env` | **0 route decorators** — scaffold |
| 17 | `ui` (network alias `mytheus-webapp`) | YES `:364` | `${UI_IMAGE:-ghcr.io/mytheus-ai/mytheus-ui:main}` `:365` | `expose: 3000` `:399-400` | YES — `wget -qO- http://127.0.0.1:3000/`, 10s/5s/10, `start_period: 20s` `:386-398` | `api-gateway: service_healthy` `:383-385` | — | `prod.ui.env` `:380-382` — **NO `.example` EXISTS** | Deliberately NOT on the backend anchors: holds no DB credential `:360-363`. Alias is explicit because a bare `ui` collided with the co-tenant `atithivishram-prod` stack and served their product on mytheus.cloud `:367-376`. Healthcheck hits `127.0.0.1` not `localhost` because the Next standalone server is IPv4-only `:387-389` |

**Named volumes** (`:413-416`): `pgdata`, `redisdata`, `redpandadata`.

**Anchors**: `x-backend` `:51-66` (image + `prod.env` + `pull_policy: always` + `restart: unless-stopped`); `x-backend-app` `:69-75` (adds `prod.db.app.env`).

**Totals**: 16 long-running containers + 1 one-shot = 17.

**Healthcheck coverage: 7 of 16 long-running services** (postgres, redis, redpanda, web, api-gateway, meeting-service, ui). The 9 without one — temporal, relay, workflow, bff, identity-tenant-service, integration-service, billing-metering-service, admin-ops-service, notification-service — are invisible both to `deploy.sh`'s gate and to any `depends_on: service_healthy`.

**No resource governance whatsoever.** `grep -nE "mem_limit|cpus|deploy:|resources|logging:" deploy/compose.prod.yml` returns **NONE**. No memory limits, no CPU limits, no log rotation, on any of the 17 services.

### 1.2 Diff against the ~21 services that exist across the repos

Backend — `mytheus-backend/services/` (11 service modules; `__init__.py` and `__pycache__` excluded):

| service module | in compose.prod? | route decorators (grep `@(app\|router).(get\|post\|put\|patch\|delete)`) | status |
|---|---|---|---|
| `core_api` | YES (as `web`) | 16 | DEPLOYED |
| `api_gateway` | YES | 0 (it is a proxy; routing is `ROUTE_MAP` + middleware) | DEPLOYED |
| `bff` | YES | 3 | DEPLOYED |
| `identity_tenant_service` | YES | 11 | DEPLOYED |
| `meeting_service` | YES | 2 | DEPLOYED |
| `relay` | YES | 0 (worker, not HTTP) | DEPLOYED |
| `workflow_service` | YES | 0 (worker, not HTTP) | DEPLOYED |
| `integration_service` | YES | **0** | **DEPLOYED-BUT-EMPTY** |
| `billing_metering_service` | YES | **0** | **DEPLOYED-BUT-EMPTY** |
| `admin_ops_service` | YES | **0** | **DEPLOYED-BUT-EMPTY** |
| `notification_service` | YES | **0** | **DEPLOYED-BUT-EMPTY** |

Intelligence — `mytheus-intelligence/services/` (7):

| service | in compose.prod? | status |
|---|---|---|
| `agent_runtime_service` | NO | EXISTS-BUT-NOT-DEPLOYED |
| `embedding_service` | NO | EXISTS-BUT-NOT-DEPLOYED |
| `ingestion_service` | NO | EXISTS-BUT-NOT-DEPLOYED |
| `knowledge_graph_service` | NO | EXISTS-BUT-NOT-DEPLOYED |
| `rag_service` | NO | EXISTS-BUT-NOT-DEPLOYED |
| `reranker_service` | NO | EXISTS-BUT-NOT-DEPLOYED |
| `search_service` | NO | EXISTS-BUT-NOT-DEPLOYED |

Realtime — `mytheus-realtime/services/` (2):

| service | language | in compose.prod? | status |
|---|---|---|---|
| `realtime-gateway` | TypeScript (`package.json`, `vitest.config.mts`) | NO | EXISTS-BUT-NOT-DEPLOYED |
| `transcription-service` | Python (`pyproject.toml`, `alembic.ini`) | NO | EXISTS-BUT-NOT-DEPLOYED |

UI — `mytheus-ui` (1): **DEPLOYED** as `ui`.

**Roll-up: 21 services + UI.** 13 deployed (of which 4 are empty scaffolds), 9 not deployed.

---

## 2. NOT DEPLOYED BUT EXISTS / DEPLOYED BUT EMPTY

### 2.1 Exists but not deployed — 9 services, and two entire repos with no build path

`find` for `Dockerfile*` across all repos (excluding `node_modules`, `.venv`, `.worktrees`) returns exactly two results: `mytheus-backend/Dockerfile` and `mytheus-ui/Dockerfile`.

- **`mytheus-intelligence` (7 services)** — no Dockerfile, no `.github/workflows/`, no deploy descriptor. Its only compose file, `mytheus-intelligence/docker-compose.yml`, declares **only** `qdrant` (`:19`) and `minio` (`:52`) for local dev, and its header (`:8-11`) explicitly states Postgres/Redis/Redpanda/Schema-Registry are *not* declared because this tier shares the backend's stack. There is no image to pull and nothing to deploy.
- **`mytheus-realtime` (2 services)** — no Dockerfile, no `.github/workflows/`, absent from `compose.prod.yml`.

This is documented as intentional at `deploy/README.md:125-126`: *"Realtime / intelligence tiers deploy as their own compose stacks/images (separate scaling families, separate cost centers); their deploy descriptors land alongside their scale-out bands."* Intentional or not, the operational fact is that **9 of 21 services have no deployable artifact today**, so the entire AI tier and the entire realtime tier are absent from production.

Both repos' `Makefile`s expose only `install / lint / typecheck / test / ci` — no `docker`, no `build`, no `deploy` target.

### 2.2 Deployed but empty — 4 services

`integration-service`, `billing-metering-service`, `admin-ops-service`, `notification-service`: 0 route decorators each. Self-declared at `compose.prod.yml:318-321`:

> "These four currently expose no routes beyond /healthz - they are scaffolds whose bands have not landed. Deployed anyway so the topology is complete and the gateway's route map has a real upstream to reach: a request to /billing gets a 404 FROM billing-metering-service, which is the truth, rather than a 404 from the gateway that looks identical to a routing bug."

The reasoning is sound. The cost is 4 idle Python containers on a resource-unbounded single node (§8).

Borderline but real: `bff` (3 routes) and `meeting-service` (2 routes) are thin. `compose.prod.yml:236-239` records that until recently exactly **one** of the nine HTTP services ran in production — core-api, as `web` — so the deployed API was 8 routes and `api-gateway` itself, the fail-closed checkpoint doc 11 depends on, was not running.

---

## 3. MISSING INFRA DEPENDENCIES

Present in `compose.prod.yml`: Postgres, Redis, Redpanda (Kafka API + Schema Registry), Temporal.

| dependency | required by (evidence) | in compose.prod? | consequence |
|---|---|---|---|
| **Qdrant** | intelligence tier — `QDRANT_URL`, `QDRANT_API_KEY`, `QDRANT_READ_ONLY_API_KEY`; `libs/shared/qdrant.py` refuses to build a client if the two keys match | **NO** — dev only, `mytheus-intelligence/docker-compose.yml:19` | Vector search unavailable in prod. Moot only because the tier isn't deployed |
| **MinIO / object storage** | `ingestion_service` — `MINIO_ENDPOINT_URL`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY` | **NO** — dev only, `mytheus-intelligence/docker-compose.yml:52` | No document/attachment storage. ADR-0023 names Cloudflare R2; **no R2 endpoint, bucket or credential appears in any env example in any repo** |
| **LiveKit server** | `meeting-service` (token mint + webhook), `transcription-service`, UI | **NO** — assumed LiveKit Cloud via `LIVEKIT_URL` | `compose.prod.yml:290-292`: without `LIVEKIT_URL`/`LIVEKIT_API_KEY`/`LIVEKIT_API_SECRET`, `select_transport()` falls back to `FakeTransport` and mints obviously-fake tokens. All three are **blank** in `prod.env.example:52-54` |
| **Deepgram** | `transcription-service` — `DEEPGRAM_API_KEY` | **NO** (tier not deployed) | No STT |
| **identity gRPC (:50051)** | intelligence tier — `IDENTITY_GRPC_ADDR=localhost:50051` (`mytheus-intelligence/.env.example:44`) | **NO** — nothing in compose exposes 50051; `identity-tenant-service` runs uvicorn on 8000 only `:279-285` | gRPC mesh has no listener in prod |
| **Reverse proxy / TLS** | all ingress | **NOT IN THIS STACK, BY DESIGN** | See below |
| **Ray Serve / GPU / LLM serving** | ADR-0008, `EMBEDDING_GPU`/`RERANKER_GPU` | **NO** | Nothing GPU-shaped is in the prod stack. LiteLLM routes to hosted providers |
| **Observability collector** | `OTEL_EXPORTER_OTLP_ENDPOINT` in every env example | **NO** | §9 |
| **Backups** | ADR-0016 follow-up 1 | **NO** | §9 |

**TLS / ingress detail.** `compose.prod.yml:403-411` records that a `caddy` service used to be declared here publishing 80/443, never ran because those ports were already bound, and was removed so the file describes what really runs. The box's single Caddy belongs to the `atithivishram-prod` stack and is the only listener the host's `cf-origin-lock` admits Cloudflare traffic to.

Mytheus' ingress config is `deploy/caddy/mytheus.caddy`, spliced into that other product's Caddyfile via one permanent `import /etc/caddy/conf.d/*.caddy` line plus a read-only mount (`mytheus.caddy:9-11`). Three site blocks:

- `mytheus.cloud, www.mytheus.cloud` → `reverse_proxy mytheus-webapp:3000` `:31-39`
- `vps.mytheus.cloud` → `reverse_proxy dokploy:3000` `:41-45`
- `api.mytheus.cloud` → `reverse_proxy api-gateway:8000` `:47-55`

All three use `tls internal` `:24`, which **requires the Cloudflare zone to stay on SSL mode Full — Full (strict) rejects an internally-issued cert**. The Cloudflare↔origin hop is encrypted but the origin certificate is unauthenticated.

Two coupling risks are documented in the file itself: a syntax error here fails validation of the **whole** Caddyfile including Atithi's five domains (`:16-19`), and any upstream name used here must be unique across every network that Caddy is attached to (`:33-36`) — the `ui`/`mytheus-webapp` collision already caused mytheus.cloud to serve the other product's app.

---

## 4. ENV MATRIX + MISSING VARS

### 4.1 Example files found (all 8 repos)

| repo | example files |
|---|---|
| `mytheus-platform` | `env/dev.env.example`, `env/staging.env.example`, `env/prod.env.example`, `env/prod.db.app.env.example`, `env/prod.db.migrate.env.example`, `env/prod.db.relay.env.example`, `env/prod.db.temporal.env.example` |
| `mytheus-backend` | `.env.example` |
| `mytheus-intelligence` | `.env.example` |
| `mytheus-ui` | `.env.example` |
| `mytheus-mobile` | `.env.example` |
| `mytheus-realtime` | `services/realtime-gateway/.env.example`, `services/transcription-service/.env.example` |
| `mytheus-events` | none (needs none) |
| `mytheus-docs` | none (needs none) |

### 4.2 Which file reaches which prod service

From `compose.prod.yml` env_file entries:

| env file | consumed by |
|---|---|
| `prod.env` (`:64, 72, 87, 164, 213`) | postgres, migrate, web, relay, workflow, api-gateway, bff, identity-tenant-service, meeting-service, integration-service, billing-metering-service, admin-ops-service, notification-service |
| `prod.db.app.env` (`:74`) | web, workflow, api-gateway, bff, identity-tenant-service, meeting-service, integration-service, billing-metering-service, admin-ops-service, notification-service (all via `x-backend-app`) |
| `prod.db.relay.env` (`:215`) | relay only |
| `prod.db.migrate.env` (`:166`) | migrate only |
| `prod.db.temporal.env` (`:144`) | temporal only |
| `prod.ui.env` (`:381`) | ui only — **NO `.example`, NOT IN `deploy/README.md`'s copy list, NOT IN `env/README.md`'s table** |

All entries carry `required: false`, deliberately, so a checkout without secrets still renders (`:24-26`). See §7 risk 1.

### 4.3 Union of variables, grouped

**`prod.env` — shared, non-database** (`env/prod.env.example`):

| var | line | default in example |
|---|---|---|
| `POSTGRES_PASSWORD` | 29 | `CHANGE_ME_strong_random` |
| `POSTGRES_DB` | 30 | `mytheus` |
| `REDIS_URL` | 31 | `redis://redis:6379/0` |
| `REDPANDA_BROKERS` | 32 | `redpanda:29092` |
| `SCHEMA_REGISTRY_URL` | 33 | `http://redpanda:8081` |
| `TEMPORAL_ADDRESS` | 36 | `temporal:7233` |
| `TEMPORAL_NAMESPACE` | 37 | `default` |
| `TEMPORAL_TASK_QUEUE` | 38 | `mytheus` |
| `CLERK_SECRET_KEY` | 41 | blank |
| `CLERK_PUBLISHABLE_KEY` | 42 | blank |
| `CLERK_JWKS_URL` | 43 | blank |
| `CLERK_ISSUER` | 44 | blank |
| `CLERK_WEBHOOK_SECRET` | 45 | blank |
| `STRIPE_SECRET_KEY` | 48 | blank |
| `STRIPE_WEBHOOK_SECRET` | 49 | blank |
| `LIVEKIT_API_KEY` | 52 | blank |
| `LIVEKIT_API_SECRET` | 53 | blank |
| `LIVEKIT_URL` | 54 | blank |
| `PUBLIC_HOSTNAME` | 57 | `srv1756037.hstgr.cloud` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | 60 | blank |
| `SENTRY_DSN` | 61 | blank |

Explicitly **no** database credential in this file (`prod.env.example:11-28`) — that is the closed half of defect D1.

**Per-role database files:**

| file | vars | lines |
|---|---|---|
| `prod.db.app.env` | `DATABASE_URL` (mytheus_app) | 12 |
| `prod.db.relay.env` | `RELAY_DATABASE_URL`, `DATABASE_URL` (both mytheus_relay — pointing the fallback at mytheus_app is defect D4) | 14-15 |
| `prod.db.migrate.env` | `BOOTSTRAP_DATABASE_URL`, `DATABASE_URL` (both postgres owner), `MYTHEUS_APP_PASSWORD`, `MYTHEUS_RELAY_PASSWORD`, `MYTHEUS_ADMIN_PASSWORD` | 16-21 |
| `prod.db.temporal.env` | `POSTGRES_PWD` (same secret as `POSTGRES_PASSWORD`, different name because the image demands it) | 22 |

**`staging.env.example`** flattens all of the above into one file (`:10-18`) — i.e. staging does **not** have the per-role split that prod has. That is a divergence: staging cannot prove the D1 fix.

**`dev.env.example`** — same shape as staging, localhost ports 5433/6380/9092/8081 (`:16-25`).

**Other repos' vars:**

| repo | vars |
|---|---|
| `mytheus-backend/.env.example` | `CLERK_SECRET_KEY`, `CLERK_PUBLISHABLE_KEY`, `CLERK_JWKS_URL`, `CLERK_ISSUER`, `CLERK_WEBHOOK_SECRET`, `CLERK_JWT_ORG_CLAIM`, `DATABASE_URL`, `RELAY_DATABASE_URL`, `ADMIN_DATABASE_URL`, `BOOTSTRAP_DATABASE_URL`, `MYTHEUS_APP_PASSWORD`, `MYTHEUS_RELAY_PASSWORD`, `MYTHEUS_ADMIN_PASSWORD`, `REDIS_URL`, `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`, `TEMPORAL_ADDRESS`, `TEMPORAL_NAMESPACE`, `TEMPORAL_TASK_QUEUE`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `LIVEKIT_URL`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `SENTRY_DSN`, `ENVIRONMENT`, `LOG_LEVEL` |
| `mytheus-intelligence/.env.example` | `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`, `REDIS_URL`, `QDRANT_URL`, `QDRANT_API_KEY`, `QDRANT_READ_ONLY_API_KEY`, `MINIO_ENDPOINT_URL`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`, `DATABASE_URL`, `RELAY_DATABASE_URL`, `BOOTSTRAP_DATABASE_URL`, `EMBEDDING_GPU`, `RERANKER_GPU`, `IDENTITY_GRPC_ADDR`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `SENTRY_DSN` |
| `mytheus-realtime/services/realtime-gateway/.env.example` | `PORT`, `WS_BIND_HOST`, `REDIS_URL`, `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`, `CLERK_JWKS_URL`, `CORE_API_URL`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `PRESENCE_HEARTBEAT_INTERVAL_MS`, `PRESENCE_STALE_AFTER_MS`, `PRESENCE_SWEEP_INTERVAL_MS` |
| `mytheus-realtime/services/transcription-service/.env.example` | `REDIS_URL`, `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`, `DATABASE_URL`, `TRANSCRIPTION_WS_BIND`, `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `DEEPGRAM_API_KEY`, `OTEL_EXPORTER_OTLP_ENDPOINT` |
| `mytheus-ui/.env.example` | `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY`, `NEXT_PUBLIC_CLERK_SIGN_IN_URL`, `NEXT_PUBLIC_CLERK_SIGN_UP_URL`, `API_GATEWAY_URL`, `NEXT_PUBLIC_APP_ENV` |
| `mytheus-mobile/.env.example` | `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY`, `EXPO_PUBLIC_API_GATEWAY_URL`, `EXPO_PUBLIC_ENV` |

### 4.4 REQUIRED BY CODE, ABSENT FROM EVERY EXAMPLE FILE

Method: grepped `os.environ(.get)?\(|os.getenv\(` across `services/`, `libs/`, `scripts/`, `migrations/` in the four Python repos and `process.env.` across `mytheus-ui/src`, then checked each name against every `*.example`.

| var | required by | absent from | severity |
|---|---|---|---|
| **`GATEWAY_SIGNING_KEY`** | `mytheus-backend/services/api_gateway/edge.py:59` | **EVERY example file, every compose file, every deploy doc** | **DEPLOY-BREAKING** |
| **`NEXT_PUBLIC_API_GATEWAY_URL`** | `mytheus-ui/src/lib/api/client.ts:69` | every example file | HIGH |
| **`NEXT_PUBLIC_LIVEKIT_URL`** | `mytheus-ui/src/lib/realtime/useLiveKitRoom.ts:69` | every example file **and** the Dockerfile's build args | HIGH |
| **`NEXT_PUBLIC_REALTIME_WS_URL`** | `mytheus-ui/src/lib/realtime/useMeetingSocket.ts:60` | every example file **and** the Dockerfile's build args | HIGH |
| **`prod.ui.env` (the whole file)** | `compose.prod.yml:381` | no `.example`; not in `deploy/README.md:47-50`; not in `env/README.md` | HIGH |
| `ROLE_TOPICS_TAXONOMY_VERSION` | `services/identity_tenant_service/taxonomy/loader.py:102` | every example file | LOW (defaults `"v1"`) |
| `CLERK_JWT_INTERNAL_ORG_CLAIM` | intelligence tier | `mytheus-intelligence/.env.example` | LOW (tier not deployed) |
| R2 / object-storage config (ADR-0023) | ingestion path | every example file — only MinIO dev values exist | MEDIUM (tier not deployed) |

#### B1 — `GATEWAY_SIGNING_KEY`: the deploy-breaking one

`mytheus-backend/services/api_gateway/edge.py:58-65`:

```python
def _signing_key() -> bytes:
    key = os.environ.get("GATEWAY_SIGNING_KEY")
    if not key:
        # Fail closed and loudly. A default key is worse than no signing at all, because it
        # looks like a control while being public knowledge.
        raise ProblemDetail(
            503, ErrorCode.DEPENDENCY_UNAVAILABLE, "Gateway signing key not configured"
        )
    return key.encode()
```

Grepped across all repos: the name appears **only** at `edge.py:59` and in `tests/test_gateway_edge.py:78,236`. It is in no `.env.example`, no compose file, and no runbook.

It is worse than a clean 503. The call chain is `edge_auth` → `_attach_internal_headers` (`edge.py:120`) → `sign_context` (`edge.py:139`) → `_signing_key`. Line 120 sits **outside** the `try:` block, which is closed by `except ProblemDetail` at `edge.py:112-114`. The middleware's own docstring (`edge.py:88-91`) explains why that matters:

> "Returns a response directly rather than raising, because middleware sits outside the exception handlers the app factory installs, so a raise here would escape as an unhandled 500 carrying whatever the exception said."

So with the variable unset, every authenticated request through `api-gateway` returns an **unhandled 500**, not a 503.

The failure is invisible to every gate in the pipeline. `/healthz` is in `PUBLIC_PATHS` (`edge.py:42`), and `is_public()` short-circuits before `_attach_internal_headers` is ever reached (`edge.py:96-97`). Therefore:
- the container healthcheck (`compose.prod.yml:258-266`) hits `/healthz` and reports **healthy**;
- `deploy.sh`'s `wait_healthy` passes;
- `mytheus-backend/.github/workflows/ci.yml:333-340` and `mytheus-platform/.github/workflows/deploy.yml:70-79` both `curl https://api.mytheus.cloud/healthz` and get **200**.

**The deploy reports fully green while every real API call 500s.**

Related finding: `verify_context` (`edge.py:79-81`) is defined and **never called by any upstream service** — grep for `verify_context|sign_context` outside `edge.py` returns nothing. `services/api_gateway/proxy.py:11-14` states the signed headers "must reach the upstream intact or downstream cannot tell a gateway-resolved tenant from a forged one", but no downstream verifies them. The HMAC is currently minted and discarded; the actual protection is `_strip_internal_headers` (`edge.py:128-131`), which does work.

#### B2 — UI runtime variables: the meeting room is silently dark

`mytheus-ui/src` reads exactly four env vars. Three of them are in no example file, and two of those three **cannot be made to work at all with the current build**:

`NEXT_PUBLIC_*` values are inlined at build time. `mytheus-ui/Dockerfile:29-35` declares only three build args:

```dockerfile
ARG NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY
ARG CLERK_SECRET_KEY
ARG API_GATEWAY_URL
```

matched exactly by `mytheus-ui/.github/workflows/ci.yml:61-64`. So `NEXT_PUBLIC_LIVEKIT_URL` and `NEXT_PUBLIC_REALTIME_WS_URL` are `undefined` in the shipped client bundle **regardless of what `prod.ui.env` contains** — putting them in the runtime env file cannot fix it; the image must be rebuilt with new build args.

And the code degrades silently rather than erroring:
- `useLiveKitRoom.ts:13,32` — "DEGRADES BY DESIGN when LiveKit is not configured… `unconfigured` // no NEXT_PUBLIC_LIVEKIT_URL: fall back to the mock grid"
- `useMeetingSocket.ts:13,40` — "`NEXT_PUBLIC_REALTIME_WS_URL` is unset in every environment that exists today… `disabled` // caller falls back to mock captions"

Production would serve a mock video grid and mock captions with no error anywhere.

### 4.5 Cross-file divergences (not missing, but will bite)

- **`TEMPORAL_TASK_QUEUE`**: `mytheus` in `env/prod.env.example:38` vs `mytheus-backend` in `mytheus-backend/.env.example:19`. If the worker and the workflow starter disagree, workflows queue forever with no error. UNVERIFIED which value the code treats as authoritative.
- **`REDIS_URL`**: `redis://localhost:6379/0` in `mytheus-intelligence/.env.example:9` vs `redis://localhost:6380` in `mytheus-backend/.env.example` and the realtime examples. The backend stack publishes 6380 (`env/dev.env.example:6-7`), so the intelligence dev default points at the wrong port.
- **`staging.env.example`** has no per-role split (§4.3), so staging cannot exercise the D1 fix that prod depends on.
- **Clerk dashboard step**: `deploy/README.md:71-88` requires a manual "Customize session token" claim `"internal_org_id": "{{org.public_metadata.internal_org_id}}"`. Nothing verifies it. Absent, the AI tier 401s on every request while the backend looks healthy — the symptom reads as an AI outage rather than a config gap.

---

## 5. CI/CD REALITY

### 5.1 Which repos have CI at all

| repo | `.github/workflows/` |
|---|---|
| `mytheus-backend` | `ci.yml` |
| `mytheus-ui` | `ci.yml` |
| `mytheus-events` | `ci.yml` |
| `mytheus-platform` | `ci.yml`, `deploy.yml` |
| **`mytheus-intelligence`** | **none** |
| **`mytheus-realtime`** | **none** |
| **`mytheus-mobile`** | **none** |
| `mytheus-docs` | none (acceptable) |

Three code repos — including the entire 7-service AI tier and the 2-service realtime tier — have **no CI whatsoever**. Nothing lints, type-checks, tests, or builds them on any event.

### 5.2 `mytheus-backend/.github/workflows/ci.yml` — the substantial one

Triggers: `pull_request: {}` and `push: branches: [main]` (`:11-14`). Env pins ports 5433/6380 to match `docker-compose.yml` and `conftest.py`, with `BOOTSTRAP_DATABASE_URL` and `DATABASE_URL` deliberately set to **different roles** so isolation tests prove something (`:16-29`).

| job | what it runs | notes |
|---|---|---|
| `lint` `:32` | `uv sync`, `ruff check`, `black --check`, `mypy` | |
| `test` `:42` | Postgres 15 + Redis 7 service containers; `make proto`; `pytest -q` with `MYTHEUS_REQUIRE_DB: "1"` `:78` so an unreachable DB is a hard failure, not a skip | Then `:80-102` re-runs **six named** isolation tests and greps `"6 passed"`, failing the build if the count changes. Genuinely strong |
| `migrations` `:104` | `upgrade head` → `downgrade -1` → `upgrade head`; then runs `bootstrap_roles`, deliberately grants `BYPASSRLS` to `mytheus_app`, and asserts the script **rejects** it `:130-145` | A negative test on a guard — rare and good |
| `authz-check` `:147` | `scripts/check_authz.py` — every FastAPI route must declare auth or be explicitly public | |
| `openapi` `:157` | (a) `scripts.emit_openapi --check` drift gate; (b) `spectral lint --fail-severity=error`; (c) oasdiff breaking-change gate — see §6 | `fetch-depth: 0` for the merge base |
| `security` `:221` | gitleaks **binary** v8.30.1 `git . --redact --verbose` (full history), then `pip-audit --skip-editable` — neither `\|\| true` | `:227-236` documents that the `gitleaks-action` wrapper began hard-failing with "License key is required" after the repo moved into the Mytheus-AI org, so the gate had been dead |
| `docker` `:253` | buildx build; on `push` only, logs into GHCR with the built-in `GITHUB_TOKEN` (`packages: write`) and pushes `:${{ github.sha }}` **and** `:main` `:286-288` | PRs build but never push, to close the fork supply-chain hole `:281-282` |
| `deploy` `:292` | `needs: [lint, test, migrations, authz-check, openapi, security, docker]` `:297`; `if: push && ref == refs/heads/main` `:298`; `environment: production` | See §5.4 |

### 5.3 The other three

**`mytheus-platform/.github/workflows/ci.yml`** is the canonical *template*. Its `python` `:14`, `authz-check` `:25`, `avro-compat` `:35` and `docker` `:57` jobs are all gated on `contains(github.repository, 'mytheus-backend' | 'mytheus-events')`, so **in mytheus-platform itself only the `security` job runs**. Worse, its `python` job would run `uv sync` in a repo that has no `pyproject.toml`. Its `security` job still uses `gitleaks/gitleaks-action@v2` `:54` and `pip-audit || true` `:55`.

**Critical wiring gap**: `mytheus-platform/Makefile:6-9` defines
```make
lint:
	actionlint .github/workflows/ci.yml
	POSTGRES_PASSWORD=validate PUBLIC_HOSTNAME=:80 \
		docker compose -f deploy/compose.prod.yml config >/dev/null
```
— the only gate that would catch a broken `compose.prod.yml` — **and nothing in CI invokes `make lint`**. `make test` is `python3 scripts/check_authz.py || true` (`Makefile:16`), i.e. a no-op gate.

**`mytheus-events/.github/workflows/ci.yml`** — `schemas` (`make ci`), `compatibility` (Redpanda service container; registers the **base-branch** schemas first, then runs `make check-compatibility` against the PR's — `:48-60`, which closes the trap where a fresh registry makes the BACKWARD check pass trivially), `security` (`gitleaks/gitleaks-action@v2` `:68`). Well-constructed.

**`mytheus-ui/.github/workflows/ci.yml`** — `quality` (`tsc --noEmit`, `next lint`; Node pinned to 20 with a documented reason `:25-29`), `build` (Docker build; push to GHCR on main), `deploy`. Its header `:8-11` is explicitly honest that Vitest and Playwright are absent because there is no test tooling and "a job that runs zero tests and reports green is worse than no job". So **the UI has zero automated tests**.

### 5.4 Is there a real CD path to the VPS?

**Yes — three of them, and they can race.**

| workflow | trigger | verification after deploy |
|---|---|---|
| `mytheus-backend/ci.yml:292-340` | push to main, after all 7 gates | `curl https://api.mytheus.cloud/healthz`, 10 × 10s `:333-340` |
| `mytheus-platform/deploy.yml:38-79` | push to main touching `deploy/**`, `docker-compose.yml`, or the workflow itself `:25-28`; plus `workflow_dispatch` | same healthz loop `:70-79` |
| `mytheus-ui/ci.yml:71-92` | push to main, after `quality` + `build` | **NONE** — the job ends at the ssh step `:87-92` |

All three configure ssh identically and **pass no command**:
```
ssh -i ~/.ssh/deploy_key -o BatchMode=yes -o StrictHostKeyChecking=yes \
    -o UserKnownHostsFile=~/.ssh/known_hosts -o ConnectTimeout=30 \
    root@${{ secrets.VPS_HOST }}
```
The `authorized_keys` forced command (`/usr/local/sbin/mytheus-deploy.sh`) decides what runs (`deploy.yml:15-20`, `mytheus-backend/ci.yml:318-320`). `StrictHostKeyChecking=yes` with a pinned `VPS_KNOWN_HOSTS` rather than `accept-new` — correct.

Two problems:

1. **Concurrency does not actually serialize.** All three use `concurrency: group: deploy-vps, cancel-in-progress: false`, but GitHub scopes concurrency **per repository** — which `mytheus-backend/ci.yml:301-303` openly admits. A backend deploy and a UI deploy can therefore run `docker compose up` against the same project simultaneously.
2. **`mytheus-ui`'s deploy never verifies anything.** It has no test suite, and its deploy job has no post-deploy check, so a UI deploy is unverified end to end.

### 5.5 The deploy script itself is not in any repository — everything below is UNVERIFIED

`deploy/deploy.sh:4-7` states plainly it is the **manual fallback** ("Dokploy is the normal way to deploy… Keep this script working: it is what you reach for when Dokploy itself is the thing that is broken"). The script CI actually triggers — `/usr/local/sbin/mytheus-deploy.sh` — exists only on the VPS.

This is already on the project's own defect list, at `mytheus-docs/work-done-till-now/00-status-overview.md:121` (gap 26) and `mytheus-docs/work-done-till-now/03-plan-3-4-meeting-and-transcription.md:182` (defect 10b), which notes the `wait_healthy` gate therefore applies only to the documented fallback, not to the merge path.

**Consequence for this audit: everything in §5.6 describes `deploy/deploy.sh` only. What production deploys actually do is UNVERIFIED and unverifiable from source control.** The one property that does hold on all paths is that Dokploy, the server script and the manual script all run the same `compose.prod.yml`, so the healthchecks in that file apply everywhere.

### 5.6 `deploy/deploy.sh` — step by step (fallback path)

110 lines, `set -euo pipefail` (`:13`).

1. `:15-16` — resolve own dir, `cd` into it.
2. `:18-22` — require `../env/prod.env`, else `exit 1`.
3. `:25-28` — `set -a; source "$ENV_FILE"; set +a`, for compose **interpolation** only (env_files are read separately by compose).
4. `:31` — `git -C .. pull --ff-only || true`. **Failure swallowed**: a deploy can silently run stale descriptors.
5. `:37` — `docker compose -f compose.prod.yml up -d --pull always`. Deliberately `--pull`, not `--build` (`:33-35`): building here would produce an image matching no identifiable commit.
   - **Migrations run here**, as the `migrate` one-shot. Because `web` declares `depends_on: migrate: service_completed_successfully` (`compose.prod.yml:187-189`), a failed migration fails the `up` and `set -e` catches it. So **yes, migrations run and are gated.**
6. `:42-54` — `wait_healthy()`: 30 iterations × 5s = 150s, polls `docker inspect .State.Health.Status`, dumps the last 50 log lines and `return 1` on timeout. `:39-41` records that the previous version looped and then fell through to "Done." regardless — `set -e` cannot catch a loop that exits 0.
7. `:58-59` — gates **only** `mytheus-web-1` and `mytheus-meeting-service-1`.
   - **`api-gateway` and `ui` both have healthchecks and are NOT gated.** Given that `api-gateway` is the sole origin for `api.mytheus.cloud`, that is the most consequential omission.
   - The 9 services with no healthcheck cannot be gated at all.
8. `:77-106` — `verify_event_backbone()`: **VERIFIES, DOES NOT CREATE.** Counts topics via `rpk topic list` excluding `_schemas|__consumer_offsets|_redpanda` (expects ≥ 42 = 21 topics + 21 `.dlq`) and Schema Registry subjects via `curl localhost:8081/subjects` (expects ≥ 21). Hard-fails with a long explanatory message otherwise (`:87-103`). Rationale at `:61-76`: a second copy of the topic list in this repo would drift silently from `mytheus-events/scripts/create_topics.sh`.
   - **Topic creation is manual**, from a mytheus-events checkout: `BROKERS=<broker>:9092 bash scripts/create_topics.sh` and `make register SR_URL=…` (`:97-98`).
   - **`mytheus-events` is not cloned on the VPS** — `deploy/README.md:33-37` clones mytheus-platform only. So **a from-scratch deploy fails this gate by construction.** (`mytheus-events/scripts/create_topics.sh` and 22 files under `subjects/` do exist in that repo.)
9. `:108-109` — `docker compose ps`; print the healthz URL.

**Rollback: none in the script.** Documented as a manual re-pin (`deploy/README.md:127-135`):
```
BACKEND_IMAGE=ghcr.io/mytheus-ai/mytheus-backend:<previous-sha> docker compose -f compose.prod.yml up -d
```
with the correct note that `main` is a moving pointer and never a rollback target, and that a Hostinger VPS snapshot is the fallback for already-applied forward-only migrations. **Nothing takes that snapshot automatically.**

---

## 6. THE oasdiff BREAKING-CHANGE GATE

`mytheus-backend/.github/workflows/ci.yml:186-219`.

The most recent commit in the repo is `27d4599` — *"fix(ci): oasdiff exits 0 on a breaking change, so the gate never failed a build"* (2026-08-30), touching only `.github/workflows/ci.yml`, +9/−1.

**Is it correctly failing builds now? The fix is correct in the file; whether it fails builds is still unproven.**

The code as it now stands (`:213`):
```
docker run --rm -v /tmp/oasdiff-base:/base:ro -v "$PWD/design/openapi:/head:ro" \
  tufin/oasdiff:latest breaking --fail-on ERR "/base/$name" "/head/$name" || failed=1
```
`--fail-on ERR` is present, and `:215-219` exits 1 if `failed` is set. The mechanism is sound.

Three caveats, two of which the commit message states itself:

1. **`if: github.event_name == 'pull_request'` (`:201`)** — the gate never runs on push to main.
2. **It has still never executed in GitHub Actions.** The commit message: *"Still not run in Actions. The next PR carrying a spec change is its first real run."* The in-file comment `:199-200` says the same. So the fix is verified only by the author's local reproduction against `tufin/oasdiff:latest`, described in the commit as proven both ways (deleting a path reports `api-path-removed-without-deprecation`, exits 0 without the flag and 1 with it). **UNVERIFIED in CI.**
3. **`tufin/oasdiff:latest` is unpinned** (`:213`) — an upstream change to flag semantics or exit codes would silently alter the gate.

The commit message is also worth relaying verbatim as a governance signal: *"This is the fourth documented control in this program found not to exist, after ADR-0021 arm 2, ADR-0022 defect 3 and gap 31, and it is the second false belief about this same tool."*

---

## 7. SECRETS HANDLING (locations + risk; no values printed)

### 7.1 Nothing secret is committed — verified

`git ls-files | grep -iE '\.env|secret|\.pem|\.key$'` across all six code repos returns **only** `*.env.example` files:
- `mytheus-platform/env/{dev,prod,staging,prod.db.app,prod.db.migrate,prod.db.relay,prod.db.temporal}.env.example`
- `mytheus-backend/.env.example`, `mytheus-ui/.env.example`, `mytheus-intelligence/.env.example`
- `mytheus-realtime/services/{realtime-gateway,transcription-service}/.env.example`

All contain placeholders (`CHANGE_ME_*`, `sk_test_...`, blanks). `mytheus-backend/.env` exists on disk locally but is **untracked**. `mytheus-platform/.gitignore` covers `.env`, `.env.*` with `!.env.example`, `env/*.env` with `!env/*.env.example`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `secrets.y*ml`, `*-secret*.yaml`, plus `deploy/postgres-init/prod-bootstrap.sql` on the final line.

### 7.2 How secrets reach the VPS

Manual `.env` files on the host, nothing else. They live at `/etc/dokploy/mytheus-secrets/` (directory mode `0700 root`) and are read by the compose CLI at bring-up. **No SOPS, no sealed-secrets, no vault, no External Secrets Operator** — `env/README.md:13-18` states ESO + KMS is a post-W36 / Plan 14 item and that the key names will stay the same so the migration is mechanical.

Why `/etc/dokploy` specifically is documented at length in `compose.prod.yml:20-39`: Dokploy runs `docker compose` **inside** the dokploy container with the docker socket bind-mounted, so env_file paths resolve against the container's filesystem while containers are created on the host daemon. `/srv` is not mounted into that container; `/etc/dokploy` is bind-mounted at the same path in both namespaces, so it is the only location that satisfies both the container-side env_file read and the host-side `MYTHEUS_PG_INIT` bind mount. It also keeps secrets out of Dokploy's database.

The five-way per-role split (§4.2) is genuinely good least-privilege design: `web` cannot read the schema-owner or operator credential, and `temporal` gets a one-line file rather than every Clerk/Stripe/LiveKit key (`prod.db.temporal.env.example:5-12`).

### 7.3 Risks (locations only)

1. **`required: false` on every single env_file entry** — `compose.prod.yml:65, 73, 75, 88, 145, 165, 167, 214, 216, 382`. A missing secrets file renders a valid compose with an empty environment and **no error at compose level**. The file header `:24-26` documents exactly this having happened: the backend started with no `DATABASE_URL`, surfacing only as `migrate` dying on `KeyError: 'DATABASE_URL'`. The trade-off (a checkout without secrets still renders) is understood, but the failure mode is live for any file missing on a new host — and now includes `prod.ui.env`, which no document tells anyone to create.
2. **`prod-bootstrap.sql` must be mode 0644, not 0600** — `deploy/postgres-init/README.md:16-39`. `psql` reads it as the non-root `postgres` user inside the container; a root-only file produces `Permission denied`, which the entrypoint does **not** treat as fatal, so `public` stays owned by `pg_database_owner` and the failure surfaces much later as `alembic upgrade head` dying on "permission denied for schema public". Confidentiality therefore rests **entirely** on the parent directory being `0700 root`. Correct as documented, fragile in practice: any drift in that directory's mode exposes the Postgres superuser password to every local user. The file also notes production was once repaired by hand, hiding the fact that the hook still did not work — every rebuild and any DR would have failed identically.
3. **A GHCR read token sits on the server** — `deploy/README.md:39-41`, scoped `read:packages` only. Correctly minimized, but it is a long-lived credential on the box.
4. **`VPS_DEPLOY_KEY` is a root SSH key held in three separate repos' Actions secrets** (mytheus-backend, mytheus-ui, mytheus-platform). Well mitigated by the `authorized_keys` forced command (`deploy.yml:15-20`) so a leak buys a redeploy of current main and nothing more — but that mitigation lives in a file that is in no repository and **cannot be verified**.
5. **`tls internal` + Cloudflare SSL mode Full (not Full strict)** — `mytheus.caddy:22-24`. The Cloudflare↔origin hop is encrypted with an unauthenticated certificate.
6. **UI build secrets flow through Actions build args** — `mytheus-ui/.github/workflows/ci.yml:61-64` passes `CLERK_SECRET_KEY` as a build arg. `Dockerfile:24-36` confines it to the builder stage and `:38-48` copies only the standalone output into the runtime image, so it is not in the shipped layers — correct. It would be exposed by `docker history` on the builder stage if that stage were ever pushed.
7. **Two of the four gitleaks gates are probably still dead.** `mytheus-backend/ci.yml:227-242` documents that `gitleaks/gitleaks-action@v2` began hard-failing with "License key is required" once the repo moved into the Mytheus-AI org, and replaced it with the free binary. `mytheus-events/ci.yml:68` and `mytheus-platform/ci.yml:54` **still use the action**. Same org, same licensing condition — so those two repos' secret-scanning gates are very likely non-functional. UNVERIFIED without run logs.
8. **`mytheus-intelligence`, `mytheus-realtime`, `mytheus-mobile` have no gitleaks gate at all** (no CI).

---

## 8. CAPACITY PLAUSIBILITY (ADR-0016)

`mytheus-docs/adr/0016-self-hosted-vps-mvp-substrate.md`: Hostinger KVM, **8 vCPU / 32 GB RAM / 400 GB disk**, Ubuntu 24.04.

### What the ADR justified vs what is actually deployed

| ADR-0016 assumption | reality |
|---|---|
| Force 1: the node carries "the FastAPI process, the relay, the Temporal worker, and all four stores" | **11 backend Python containers** + 4 stores + Temporal + Next.js |
| Force 2: "the MVP stack is **9 containers** including four stateful services" | **17 containers** |
| Not mentioned anywhere in the ADR | The box **also runs the entire `atithivishram-prod` stack** (a different product, five domains) **and Dokploy** — `compose.prod.yml:20-22, 366-376, 404-406` |

### Is it physically plausible?

**Yes, arithmetically — but the deploy no longer matches its own justification, and it is completely unbounded.**

The genuinely expensive tiers are absent: no Qdrant, no MinIO, no Ray Serve, no GPU, **no LLM serving on this box** (LiteLLM routes to hosted OpenAI/Anthropic, and the intelligence tier isn't deployed at all). Redpanda is pinned to `--smp 1 --overprovisioned` (`compose.prod.yml:121-122`), which is the right single-node setting. A rough steady state for the Mytheus stack — 11 uvicorn/worker processes at a few hundred MB each, Postgres, Redpanda ~1–2 GB, Temporal ~0.5 GB, Redis, Next.js — lands on the order of 8–12 GB, which fits 32 GB alongside the co-tenant.

So the answer to "Qdrant + Kafka + Postgres + LLM serving on one box?" is: **no — only Kafka and Postgres are on the box.** The plausibility problem is structural, not arithmetic:

1. **Zero resource limits on 17 containers.** No `mem_limit`, no `cpus`, no `deploy.resources` anywhere in `compose.prod.yml`. A single runaway Python process, or Redpanda under load, can OOM the host and take down **a second, unrelated production product**. On a single node with no HA (ADR-0016 "Accepted costs": *"One node is one failure domain. A host failure is a full outage, and there is no failover until Plan 14"*), that is a simultaneous outage of two products.
2. **No log rotation.** No `logging:` driver config anywhere; the default json-file driver is unbounded against a 400 GB disk.
3. **Disk is the medium-term constraint** — `pgdata` + `redpandadata` + Postgres WAL + unrotated logs, growing with nothing watching.
4. **ADR-0016's own follow-up 2 is not implemented**: *"Monitor CPU, memory, disk, and Redpanda lag as Plan 14 trigger signals rather than as mere health checks."* Nothing monitors any of them (§9).
5. **ADR-0016 follow-up 3** ("Decide where the `web-app` frontend is hosted. This ADR covers the backend only") has been resolved *de facto* by putting the UI on the same box, without the ADR being updated.

**Verdict: physically plausible today; not safely plausible, because nothing bounds it and nothing watches it.**

---

## 9. OBSERVABILITY / BACKUPS IN THE DEPLOY

### Observability — ABSENT

- `OTEL_EXPORTER_OTLP_ENDPOINT` and `SENTRY_DSN` are present-but-**blank** in `env/prod.env.example:60-61` (and blank in every other example except backend dev).
- **No** OTel Collector, Prometheus, Grafana, Loki, Tempo, or Jaeger container in `compose.prod.yml` — a grep for `prometheus|grafana|loki|tempo|otel-collector|jaeger` across every `.yml` in mytheus-platform returns **zero** matches in any compose file; all hits are prose in `README.md`, `system.md`, `CLAUDE.md`.
- `mytheus-platform/README.md:58` documents an `observability/` directory — *"OTel Collector, Prometheus rules, Grafana dashboards, alert routing"* — **that does not exist.** The repo contains only `.github/`, `deploy/`, `env/`, `scripts/`, and top-level docs. The full Prometheus/Grafana/Loki/Tempo/Thanos design in `README.md:26-27,39` and `system.md:179-181` is Kubernetes-era aspiration, not deployed infrastructure.
- **Effective production observability today: `docker logs` and container healthchecks on 7 of 16 services.** No metrics, no traces, no log aggregation, no alerting, no SLO dashboards, no error tracking.

### Backups — ABSENT

- A grep for `backup|pg_dump|restic|pgbackrest|wal-g|barman` across every `.yml`, `.sh`, and `.md` in mytheus-platform returns **no implementation** — only prose.
- No backup service, no sidecar, no cron, no volume snapshot automation. Three named volumes (`pgdata`, `redisdata`, `redpandadata`) have **no backup path of any kind**.
- **ADR-0016 follow-up 1 is unambiguous and unmet**: *"Back up PostgreSQL off-box, and rehearse a restore, before any real tenant data lands."* Neither the backup nor the drill exists.
- The only recovery mechanism mentioned anywhere is a **manual** Hostinger VPS snapshot suggested in `deploy/README.md:133-135`, taken by hand before a deploy, offered as the fallback for forward-only migrations. Nothing automates or schedules it.
- Consequence: an accidental `docker compose down -v`, a disk failure, or a host loss is **total, unrecoverable data loss** — including every tenant's data, the outbox, and the event log.
- `env/README.md:20-25` and `deploy/README.md:121-124` note that moving Postgres to managed Supabase (which would bring backups and PITR) is a one-line `DATABASE_URL` change. That option exists and is not taken.

---

## 10. VERDICT

**No — someone could deploy this today and it would report success while being substantially broken.**

`api-gateway` is the sole origin for `api.mytheus.cloud` (`deploy/caddy/mytheus.caddy:53`) and requires `GATEWAY_SIGNING_KEY` (`services/api_gateway/edge.py:59`), which appears in no env example, no compose file, and no runbook. Because the `sign_context` call at `edge.py:120` sits outside the middleware's `except ProblemDetail` block (closed at `:112-114`), every authenticated request returns an **unhandled 500** — while `/healthz` is a public path that bypasses signing entirely (`edge.py:42, 96-97`), so the container healthcheck passes, `deploy.sh`'s gate passes, and both CI post-deploy `curl` checks return 200. **The pipeline goes fully green over a completely non-functional API.** A first deploy on a fresh box would not even reach that point: `deploy.sh:87` hard-fails because it *verifies* 42 topics and 21 subjects that nothing on the deploy path *creates*, and mytheus-events is not cloned on the VPS to create them.

Beyond those two blockers, the deploy is considerably thinner than the topology suggests. Nine of twenty-one services — all of `mytheus-intelligence` (7) and all of `mytheus-realtime` (2) — have no Dockerfile, no CI, and no deploy descriptor, so the AI and realtime tiers do not exist in production at all. Four of the thirteen deployed backend services expose zero routes. The UI's `prod.ui.env` is read at `compose.prod.yml:381` with no example file and no mention in any runbook, and its two `NEXT_PUBLIC_*` realtime variables can never take effect because they are not build args in `mytheus-ui/Dockerfile:29-31` — so the meeting room silently serves a mock video grid and mock captions with no error anywhere.

**What is genuinely strong** is the security engineering *around* the deploy, and it should not be lost in the list of defects: the five-way per-role env split that closes both halves of defect D1; the forced-command SSH key that reduces a leaked deploy credential to "redeploy current main"; the six tenant-isolation assertions gated in CI with an explicit `grep "6 passed"`; the migration reversibility job with a negative test proving `bootstrap_roles` rejects a BYPASSRLS role; and the base-branch schema registration in mytheus-events that stops the BACKWARD gate passing trivially. These are real controls, not paperwork, and the inline comments explaining *why* each exists are unusually good.

**What is missing is everything that keeps a running system alive.** No backups of any volume and no restore drill, against ADR-0016's own stated precondition for real tenant data. No observability beyond `docker logs` — the `observability/` directory the README documents does not exist. No resource limits and no log rotation on seventeen containers sharing a single 8 vCPU / 32 GB node with a second production product, so one runaway process takes down two products at once. Healthchecks on seven of sixteen services, with `api-gateway` — the single ingress — ungated by the deploy script. Three repos with no CI at all. And most importantly, the script that performs every real production deploy exists only on the VPS and is in no repository (already logged as gap 26 / defect 10b in the project's own status docs), so **nothing in this audit can confirm what a production deploy actually does.**

---

## APPENDIX — files read (absolute paths)

**mytheus-platform**
- `/Users/sushan/Projects/Mytheus/mytheus-platform/deploy/compose.prod.yml`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/deploy/deploy.sh`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/deploy/caddy/mytheus.caddy`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/deploy/README.md`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/deploy/postgres-init/README.md`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/env/dev.env.example`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/env/staging.env.example`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/env/prod.env.example`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/env/prod.db.app.env.example`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/env/prod.db.migrate.env.example`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/env/prod.db.relay.env.example`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/env/prod.db.temporal.env.example`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/env/README.md`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/docker-compose.yml`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/Makefile`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/.gitignore`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/.github/workflows/ci.yml`
- `/Users/sushan/Projects/Mytheus/mytheus-platform/.github/workflows/deploy.yml`

**mytheus-backend**
- `/Users/sushan/Projects/Mytheus/mytheus-backend/.github/workflows/ci.yml`
- `/Users/sushan/Projects/Mytheus/mytheus-backend/Dockerfile`
- `/Users/sushan/Projects/Mytheus/mytheus-backend/.env.example`
- `/Users/sushan/Projects/Mytheus/mytheus-backend/services/api_gateway/edge.py`
- `/Users/sushan/Projects/Mytheus/mytheus-backend/services/api_gateway/proxy.py`
- `/Users/sushan/Projects/Mytheus/mytheus-backend/services/identity_tenant_service/taxonomy/loader.py`

**mytheus-ui**
- `/Users/sushan/Projects/Mytheus/mytheus-ui/.github/workflows/ci.yml`
- `/Users/sushan/Projects/Mytheus/mytheus-ui/Dockerfile`
- `/Users/sushan/Projects/Mytheus/mytheus-ui/.env.example`
- `/Users/sushan/Projects/Mytheus/mytheus-ui/src/lib/api/client.ts`
- `/Users/sushan/Projects/Mytheus/mytheus-ui/src/lib/realtime/useLiveKitRoom.ts`
- `/Users/sushan/Projects/Mytheus/mytheus-ui/src/lib/realtime/useMeetingSocket.ts`

**mytheus-events / intelligence / realtime / mobile / docs**
- `/Users/sushan/Projects/Mytheus/mytheus-events/.github/workflows/ci.yml`
- `/Users/sushan/Projects/Mytheus/mytheus-intelligence/docker-compose.yml`
- `/Users/sushan/Projects/Mytheus/mytheus-intelligence/.env.example`
- `/Users/sushan/Projects/Mytheus/mytheus-intelligence/Makefile`
- `/Users/sushan/Projects/Mytheus/mytheus-realtime/services/realtime-gateway/.env.example`
- `/Users/sushan/Projects/Mytheus/mytheus-realtime/services/transcription-service/.env.example`
- `/Users/sushan/Projects/Mytheus/mytheus-realtime/Makefile`
- `/Users/sushan/Projects/Mytheus/mytheus-mobile/.env.example`
- `/Users/sushan/Projects/Mytheus/mytheus-docs/adr/0016-self-hosted-vps-mvp-substrate.md`
- `/Users/sushan/Projects/Mytheus/mytheus-docs/work-done-till-now/00-status-overview.md` (gap 26 reference only)
- `/Users/sushan/Projects/Mytheus/mytheus-docs/work-done-till-now/03-plan-3-4-meeting-and-transcription.md` (defect 10b reference only)
