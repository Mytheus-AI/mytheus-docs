# Technology lock

P1 design artifact (Plan 1-2 section 3.1 step 6). **This document is a sign-off gate: it is initialled before the first Plan 1-2 migration is written.**

It restates the picks from [_canon/06-tech-decisions.md](../_canon/06-tech-decisions.md) that this band actually uses. Canon is authoritative; this is the band-scoped extract.

## 1. Correction against the plan text

> [SaaS-scale adjustment] Plan section 3.1 step 6 lists "PostgreSQL via Supabase, PgBouncer transaction pooling, Redis (Upstash)". **All three are stale.** [ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md) moved the MVP substrate to a self-hosted single-node VPS, which makes PostgreSQL, Redis, Redpanda, and Temporal self-hosted containers on one box. The plan text predates that ADR and has not been rewritten. This document follows the ADR, which is newer and load-bearing. Railway and Render are struck entirely: not the fast path, not a fallback.

## 2. The locked stack for this band

| Layer | Pick | Version or mode | Note |
|---|---|---|---|
| Language | Python | 3.12, uv-managed venv | The host Python is not what the project runs on. Always go through `uv run`. |
| API framework | FastAPI | latest 0.115+ | One app factory, `libs/shared/app_factory.create_app`, so every service gets the same middleware, error handlers, telemetry, and `/healthz`. |
| ORM | SQLAlchemy | 2.0 async plus asyncpg | One declarative `Base` across the modular monolith so one Alembic history sees every table. |
| Validation | Pydantic | v2 | Drives the `422` validation error shape. |
| Migrations | Alembic | one linear history | Invoked as `uv run alembic -c migrations/alembic.ini`. Forward-only in production; reversibility proven in CI. |
| Relational store | **PostgreSQL 15, self-hosted** | Docker container on the VPS | Was "Supabase-managed". Changed by ADR-0016. `DATABASE_URL` alone selects self-hosted versus managed, so the change is a connection string. |
| Connection pooling | **SQLAlchemy or asyncpg pool only. No PgBouncer.** | explicit bounds | See section 3. |
| Cache and pub/sub | **Redis 7, self-hosted** | Docker container, AOF on in production | Was "Upstash". Changed by ADR-0016. Namespaced `nexus:{org_id}:`. |
| Event bus | **Redpanda, self-hosted** | Kafka API plus built-in Schema Registry | Was Redpanda Cloud/BYOC. Same API. |
| Event encoding | Avro plus Schema Registry | subject `{topic}-value`, BACKWARD | Realized in `mytheus-events`. |
| Durable execution | **Temporal, self-hosted** | `temporalio/auto-setup` container | Was Temporal Cloud. |
| Identity | **Clerk** | RS256 session JWT, JWKS, Svix webhooks | Unchanged and frozen. The Supabase-Auth drift was reverted in Plan 0. |
| Frontend | Next.js | 14 App Router, React 18 | `mytheus-ui`. Uses a `src/` directory, which the plan's scaffold command contradicts. |
| Styling | Tailwind | 3.4 | The repo has its own design system in `mytheus-ui/docs/DESIGN-SYSTEM.md`. **shadcn/ui is NOT adopted**: the repo already has a component layer and adding a second would fork it. |
| Reverse proxy and TLS | Caddy | automatic HTTPS | On the VPS. |
| Container runtime | Docker Compose | single node | Kubernetes is Plan 14 only. |
| Lint, format, types | ruff, black, mypy strict | `libs/` and `services/` strict | `tests/` and root `conftest.py` excluded from strict typing: pytest fixtures are dynamically typed. |
| Tests | pytest plus testcontainers | real Postgres, Redis, Redpanda | The gate is `make ci` = lint + typecheck + test. |
| Spec governance | Spectral plus oasdiff | see `openapi/.spectral.yaml` | Breaking OpenAPI change without a major bump fails the build. |
| Observability | OpenTelemetry, Sentry | OTLP exporter | No-op unless the endpoint is set. |

## 3. Two deliberate omissions, with their exit triggers

Recording these as decisions rather than gaps, so nobody re-litigates them by accident and nobody forgets them either.

**PgBouncer is out of scope for this band.** The MVP is one node running one web process, one relay, and one workflow worker. There is no connection fan-in to absorb. Nothing structural breaks by adding it later, because the one thing that would break is already correct: `set_config('app.current_org', :org_id, true)` is transaction-local, which is the single hard requirement of transaction pooling.

*What must be done on the day PgBouncer lands, in this order:*

1. Disable asyncpg's server-side prepared-statement cache (`statement_cache_size=0`). Without it you get intermittent `prepared statement "__asyncpg_stmt_N__" does not exist` under load, which looks like corruption and is the most common asyncpg-plus-PgBouncer failure. Alternatively run PgBouncer 1.21+ with `max_prepared_statements > 0`.
2. Audit for session state spanning transactions: session-level `SET`, `LISTEN`/`NOTIFY`, session-scoped advisory locks, `SET ROLE` outside a transaction, reused temp tables. The relay's `FOR UPDATE SKIP LOCKED` pattern is already safe because its locks are taken and released inside one transaction.
3. Set `pool_mode = transaction`.
4. **Do not route Temporal through it.** Temporal shares this Postgres instance and must not sit behind a transaction pooler.

*Exit trigger:* more than one application node, or sustained use of over 60 percent of `max_connections`.

*Insurance bought now instead:* a CI check that fails the build on a non-transaction-local `set_config('app.current_org', ...)` or a bare `SET app.current_org`. That freezes the invariant PgBouncer depends on during the interval before it arrives, which is the only way deferral can hurt.

**shadcn/ui is not adopted.** Plan section 3.10 step 1 calls for `shadcn-ui init`. `mytheus-ui` already ships a documented design system and component layer built over 56 pages. Introducing a second primitive set would fork the visual language for no gain.

## 4. Sign-off

The team initials here before the first Plan 1-2 migration is written. This is the gate, not a formality: it is the last point at which a substrate decision is cheap.

| Role | Name | Date | Initials |
|---|---|---|---|
| Architecture | | | |
| Backend | | | |
| Frontend | | | |
| Security | | | |

**Status: UNSIGNED.** Section 3.3 (the first migration of the band) is blocked until this table is filled.

## 5. Related

- [_canon/06-tech-decisions.md](../_canon/06-tech-decisions.md) is the authoritative lock list.
- [ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md) for the substrate.
- [ADR-0002](../adr/0002-multi-tenancy-model.md) for the RLS posture and the three database roles.
