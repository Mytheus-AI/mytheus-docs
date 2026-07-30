# 01 Plan 0 Scaffold and Contracts

| Field | Value |
|---|---|
| Band | Plan 0, pre-P1 bootstrap |
| Plan file | [plan/plan-0-scaffold-and-contracts.md](../plan/plan-0-scaffold-and-contracts.md) |
| Status | **DONE WITH GAPS** |
| Complete | `mytheus-events` (0A), `mytheus-backend` (0B, 0C), `mytheus-intelligence` (0D), `mytheus-realtime` (0E realtime half), `mytheus-platform` (0F, 0G) |
| Incomplete | `mytheus-ui` (0E web half, deliberately skipped), `mytheus-mobile` (0G) |
| Last verified | 2026-07-23, by file inspection of every clone |

Provenance note: this file consolidates records that previously lived in `mytheus-backend/PREVIOUS.md` (three sessions), `mytheus-intelligence/PREVIOUS.md`, `mytheus-realtime/PREVIOUS.md`, and `mytheus-ui/PROGRESS.md`. Those files are historical from this point forward. Everything below marked **verified** was confirmed by reading the current tree; everything marked **reported** is inherited from a session log and has not been re-run here.

---

## 1. What landed, per repo

### 1.1 `mytheus-events` (Phase 0A) - complete

Verified by directory inspection.

- **21 topic schema folders** under `schemas/`, plus `schemas/_envelope` holding the shared `EventEnvelopeHeader`.
- **22 subject files** under `subjects/`, one `{topic}-value.json` per topic plus the envelope subject.
- **21 golden fixtures** under `fixtures/`, one per topic, for round-trip assertions.
- **Generated clients in both languages**: 23 modules under `clients/python/mytheus_events/` and 23 under `clients/typescript/src/`, covering meetings, transcripts, agents (requests, responses, results, stream), billing, usage, notifications, presence, tenants, graph, embeddings, integrations, decisions, actions, document chunks, admin, and user-sync.
- **Codegen and gates** in `Makefile`: `validate`, `register`, `check-compatibility` (BACKWARD gate), `codegen-py`, `codegen-ts`, `test-py`, `test-ts`, `typecheck`, `create-topics` (21 topics plus 21 `.dlq`), and the composite `ci` and `verify-all`.
- **Tests**: `tests/test_backward_compat.py`, `tests/test_contract.py`, `tests/test_golden_roundtrip.py`, plus a TypeScript suite under `tests/ts`.

### 1.2 `mytheus-backend` (Phases 0B, 0C) - complete

This repo went through three sessions, including one reversal that matters more than anything else in the band.

**Session 1 (superseded): Clerk was replaced with Supabase Auth.** A `public.custom_access_token_hook` injected `org_id` into the JWT and an `auth.users` trigger mirrored users into `public.users`. This contradicted ADR-0010 and the frozen program decision `IDENTITY = Clerk`.

**Session 2:** scaffold trim, removing over-created structure from session 1.

**Session 3 (current state): the Supabase-Auth drift was reverted to Clerk, and Phases 0B and 0C landed.** Only Auth was reverted; Supabase-managed Postgres was not the thing reverted.

Auth, verified:

- `libs/shared/auth.py` verifies Clerk RS256 JWTs against cached JWKS via `PyJWKClient`, honoring key rotation. `sub` maps to `users.clerk_user_id`; `org_id` arrives as a Clerk session claim; a missing `org_id` fails closed with 401. The JWKS client and issuer are read lazily so importing the module never requires Clerk env to be present.
- `services/identity_tenant_service/routes/auth.py` exposes `POST /clerk/webhook` verifying the Svix signature and returning 401 on a bad signature. The body is a stub: the inbox dedupe, the `users` upsert, and the outbox write are marked `TODO(P1)`.
- `migrations/sql/custom_claims_hook.sql` and `migrations/sql/user_sync_trigger.sql` are deleted. `CLERK_SECRET_KEY`, `CLERK_PUBLISHABLE_KEY`, `CLERK_JWKS_URL`, `CLERK_ISSUER`, and `CLERK_WEBHOOK_SECRET` are restored in `.env.example`. A CI grep guard asserts no `supabase_jwt`, `custom_access_token_hook`, or `/supabase/webhook` reference returns.

Data and eventing, verified:

- `libs/shared/db.py`: SQLAlchemy 2.0 `Base` with a stable naming convention (so autogenerate is deterministic), plus `TenantMixin` (`org_id`) and `TimestampMixin`.
- `libs/shared/infra_models.py`: `Outbox` (BIGSERIAL id, `partition_key`, `schema_subject`, an `outbox_idem_uq` unique constraint, and a partial `outbox_unpublished_idx`) and `Inbox` (composite primary key `consumer_group, org_id, idempotency_key`).
- Migration `82d44163c933` creates 11 tables (`organizations`, `users`, `teams`, `team_members`, `departments`, `tenant_settings`, `data_residency`, `meetings`, `meeting_participants`, `outbox`, `inbox`), plus `CREATE EXTENSION pgcrypto`, with `ENABLE` + `FORCE ROW LEVEL SECURITY` and a `tenant_isolation` policy on every one. `organizations` is keyed on its own `id`; every other table on `org_id`.
- `libs/shared/relay.py` implements `relay_once` with `FOR UPDATE SKIP LOCKED` and flush-then-mark (at-least-once). `services/relay/main.py` is the confluent-kafka Avro producer against `{topic}-value` with `acks=all` and idempotent production.
- `libs/shared/idempotency.py` is a Starlette `IdempotencyMiddleware` mounted by `libs/shared/app_factory.create_app()`, which also installs the RFC 7807 handlers, OpenTelemetry, and `/healthz`. All nine HTTP service `main.py` files use it.
- `services/identity_tenant_service/grpc_server.py` implements `GetUserContext`: `org_id` from gRPC metadata, `UNAUTHENTICATED` when absent, `app.current_org` set from it, a typed `UserContext` returned.
- Eleven service packages exist under `services/`, each booting through `create_app()` with a `TODO` marking where its routers go. `services/workflow_service/workflows/` holds four Temporal workflow stubs (`meeting_lifecycle`, `action_followup`, `rtbf`, `dunning`).
- Tests under `tests/`: `test_rls.py`, `test_outbox_inbox.py`, `test_idempotency.py`, `test_relay.py`, `test_grpc.py`, `test_auth.py`.

### 1.3 `mytheus-intelligence` (Phase 0D) - complete

Reported in the repo session log, structure verified.

- Single hatchling workspace (`packages = ["services", "libs"]`) with forward dependencies declared: `fastapi`, `uvicorn`, `pydantic`, `langgraph`, `litellm`, `confluent-kafka[avro,schemaregistry]`, `qdrant-client`, `grpcio`, `httpx`, `redis[hiredis]`. GPU serving is an optional `gpu` extra (`ray[serve]`) so the default dev loop stays GPU-free.
- Seven services: five FastAPI apps (`agent_runtime_service`, `rag_service`, `knowledge_graph_service`, `search_service`, `ingestion_service`) each answering `/healthz`, and two GPU services (`embedding_service`, `reranker_service`) with import-guarded Ray Serve placeholders.
- `libs/shared/app.py` (`create_app`) and `libs/shared/clients.py` (a lazy importer of the published `mytheus_events` client that fails closed with an actionable message, so the AI tier never hand-rolls serializers).
- `proto/rag.proto` placeholder for the internal gRPC mesh. `org_id` rides in gRPC metadata, never the message body.
- `tests/test_healthz.py` parametrized across the five FastAPI apps, plus a fail-closed assertion on `load_events_client()`.

### 1.4 `mytheus-realtime` (Phase 0E, realtime half) - complete

Reported in the repo session log, structure verified. Two services in two stacks, fully decoupled because they scale on different signals (`realtime-gateway` on connections, `transcription-service` on streams).

- `services/realtime-gateway/`: Node, TypeScript strict, Socket.io with the Redis adapter for cross-instance fan-out. `src/server.ts` serves `/healthz` and keeps the process up when Redis is down (`lazyConnect` plus error handlers). `src/keys.ts` builds the verbatim canonical channel `nexus:{org_id}:pubsub:meeting:{meeting_id}`. No Redpanda consumption yet.
- `services/transcription-service/`: standalone uv project, `src/`-layout package, Python 3.12. FastAPI with `/healthz` and the internal WebSocket route `/internal/audio` (the LiveKit server-side audio-hook target), which accepts then closes so the transport contract exists. The Deepgram proxy, the interim-versus-final split, `transcript_chunks` writes, and the `transcript-chunks` producer are all deferred to the realtime band. The Deepgram key stays server-side.

### 1.5 `mytheus-ui` (Phase 0E, web half) - NOT DONE

Phase 0E for the web repo was deliberately skipped: active UI work was in flight and the maintainer instructed that the repo not be touched.

What exists instead is substantial but off the Plan 0 track: a Next.js marketing site plus every authenticated screen, built entirely on `src/lib/mock/data.ts`. Contributor docs live in `docs/` (`DESIGN-SYSTEM.md`, `MOTION.md`, `PAGES.md`, `MOCK-DATA.md`, `CONTRIBUTING.md`).

What Plan 0 asked for and is missing: the Clerk provider and the API-client placeholder. `@clerk/nextjs` is not in `package.json`. Auth is bypassed and every screen reads mock data.

### 1.6 `mytheus-mobile` (Phase 0G) - NOT DONE

The repo holds a polished single-device "Meeting Pod" prototype on 100 percent mock data (`src/data/mock.ts`), rebuilt around a brand redesign. Plan 0 asked for the Clerk SDK, an `httpClient`, an env base URL, and a `REALIGN.md` backlog. None are present: `@clerk/clerk-expo` is not in `package.json`, and there is no `REALIGN.md`.

### 1.7 `mytheus-platform` (Phases 0F, 0G) - complete

Cloned 2026-07-23, previously unverifiable. Verified by file inspection.

**The key adaptation: self-host on a VPS, not Railway or Render.** Phase 0F's deploy target was rewritten from the plan's `render.yaml` / `railway.json` to a Docker Compose deploy of the modular monolith plus self-hosted PostgreSQL, Redis, and Redpanda on an already-owned Hostinger VPS (8 vCPU / 32 GB / 400 GB, Ubuntu 24.04). Everything else in 0F is unchanged in spirit. Kubernetes and Terraform remain out of scope until Plan 14. This decision is now recorded in [ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md); it had no ADR when it was made, which is why it appeared as a gap on 2026-07-22.

Phase 0F:

- `.github/workflows/ci.yml` - the canonical CI template. Jobs: `python` (uv sync, ruff, black, mypy, pytest), `authz-check` (backend), `avro-compat` (events, BACKWARD gate against a Schema Registry service), `security` (gitleaks + pip-audit), `docker` (build the backend image). Jobs self-select by repo via `if: contains(github.repository, ...)`. `actionlint`-clean. `mytheus-ui` is intentionally out of scope; it owns its own Next.js CI.
- `scripts/check_authz.py` - the shift-left authz gate. AST-walks `services/**` and fails if a FastAPI route declares no auth and is not explicitly public. Recognizes router-level and decorator-level `dependencies=[Depends(...)]`, per-parameter `Depends`/`Security`, a PUBLIC allowlist (health, webhooks, oauth), and a `# authz: public` marker. A runnable copy ships in `mytheus-backend/scripts/`, where its CI runs; this is the reference copy.
- `deploy/compose.prod.yml` - nine services: self-hosted `postgres`, `redis`, `redpanda` (Kafka plus Schema Registry), `temporal`; one backend image running three roles (`web` = uvicorn, `relay` = outbox drain, `workflow` = Temporal worker); a one-shot `migrate` running `alembic upgrade head` that `web` waits on; and `caddy` for automatic HTTPS.
- `deploy/Caddyfile`, `deploy/deploy.sh` (idempotent bring-up), `deploy/README.md` (first-time runbook).
- `env/{dev,staging,prod}.env.example` plus `env/README.md`. Only `.example` files are committed. Documents the per-store split: `DATABASE_URL` alone chooses self-hosted PostgreSQL versus managed Supabase.
- `mytheus-backend/Dockerfile` and `.dockerignore` - multi-stage (uv builder resolving from the lockfile, slim `python:3.12-slim` runtime, non-root `appuser` uid 10001, `tini` as PID 1). One image, role selected by the compose `command`.

Phase 0F verification, reported: the backend image built and booted, answering `/healthz` with `200 {"status":"ok","service":"core-api"}` as non-root. `actionlint` clean, `docker compose config` valid, `check_authz.py` passing (it correctly flagged the SCIM stubs until they were guarded with a router-level `Depends(verify_token)`).

**Deliberately not done:** the live deploy to the production VPS. Artifacts and runbook are ready; pushing to the running box was held as an outward-facing, hard-to-reverse action awaiting explicit go.

Phase 0G, cross-repo conventions:

- Uniform `make` targets (`install`, `lint`, `typecheck`, `test`, `ci`) in all five code repos, including a new repo-level `mytheus-realtime/Makefile` delegating to both its Node and Python services.
- `.pre-commit-config.yaml` in all five: local ruff/black/mypy plus gitleaks; events adds `make validate`, backend adds the authz check, realtime scopes Python hooks to `transcription-service` plus a gateway `tsc` hook, platform runs gitleaks plus actionlint.
- `CONTRIBUTING.md` in all five, each stating: names come from `_canon/` and are renamed only via Glossary plus ADR; every PR includes an `org_id` authz review; Conventional Commits, trunk-based, no `Co-Authored-By` footer; the repo's CI gate and `pre-commit install`.
- Canon-location decision recorded: canon authority lives in `mytheus-docs`, and for MVP the contract-binding repos consume it as a sibling checkout (all repos cloned under one parent). Revisit submodules or vendoring post-W36.

---

## 2. Definition of Done audit

Against section 10 of [plan-0-scaffold-and-contracts.md](../plan/plan-0-scaffold-and-contracts.md).

| DoD line | Verdict | Evidence or reason |
|---|---|---|
| `mytheus-events`: `make ci` green; 7 frozen + 14 net-new schemas; 21 subjects under BACKWARD; Python and TypeScript clients; golden fixtures round-trip; a breaking change fails `check-compatibility` | **MET** (`make ci` reported, not re-run) | 21 topic schema folders, 22 subject files, 21 fixtures, 23 modules per language, `check-compatibility` target present. |
| `mytheus-backend`: `alembic upgrade head` creates `outbox`, `inbox`, `organizations`, `users` plus P2 stubs with RLS FORCE; `target_metadata` wired; Idempotency middleware on all services; relay drains to Redpanda; gRPC `GetUserContext` boots; `pytest` green | **MET** (reported green) | Migration `82d44163c933` creates 11 tables with `FORCE` RLS. `alembic check` reported clean. Middleware mounted via `create_app()`. Relay and gRPC server present. Six test modules cover RLS, outbox/inbox, idempotency, relay, gRPC, auth. |
| Auth reconciliation: Clerk JWKS restored; Svix `/clerk/webhook` restored; `CLERK_*` in `.env.example`; Supabase hook and trigger deleted; docs updated; grep guard clean | **MET** | All five sub-items verified in the tree. |
| `mytheus-intelligence`: 7 skeletons plus shared lib plus Ray placeholder build; `ruff` and `mypy` clean | **MET** (reported clean) | Seven service packages, shared `libs/`, guarded Ray import. |
| `mytheus-ui`: `next build`, lint, and `tsc` clean; Clerk provider and API client placeholders wired | **PARTIAL** | Build gates reported clean. The Clerk provider and API client placeholders are absent; Phase 0E web half was deliberately skipped. |
| `mytheus-realtime`: both skeletons boot `/healthz` | **MET** (reported green) | Both services present with `/healthz`; `transcription-service` also carries `/internal/audio`. |
| `mytheus-platform`: CI workflow valid; deploy descriptor present; `.env` templates; no Kubernetes or Terraform | **MET** | `ci.yml` (`actionlint` clean), `deploy/compose.prod.yml` (9 services, `docker compose config` valid), three env templates, no K8s or Terraform. Deploy target is the VPS, not Railway or Render (ADR-0016). |
| `mytheus-mobile`: builds; Clerk SDK, `httpClient`, env base URL added; screens still on mock; `REALIGN.md` written | **NOT MET** | Builds and is on mock data, but the Clerk SDK, `httpClient`, and `REALIGN.md` are all absent. |
| Conventions: uniform `make ci`, pre-commit, `CONTRIBUTING.md`, canon-location decision recorded | **MET** for the five code repos | Present in `mytheus-backend`, `mytheus-events`, `mytheus-intelligence`, `mytheus-realtime`, `mytheus-platform`, with the canon-location decision recorded in each `CONTRIBUTING.md`. Absent in `mytheus-ui` and `mytheus-mobile`, both of which are excluded from 0G by the same decision that skipped 0E. |

Seven of nine met, one partial (`mytheus-ui`), one not met (`mytheus-mobile`). Both shortfalls are the two client repos, and both were deliberate.

---

## 3. Deviations from the plan, and why

| Deviation | Rationale as recorded |
|---|---|
| **One migration instead of the plan's two** (infra split from identity). | Autogenerating a single migration keeps model and migration names from drifting. The infra-versus-identity split was a separability nicety, not load-bearing. |
| **RLS predicate uses `NULLIF(current_setting('app.current_org', true), '')::uuid`** instead of the plan's bare `current_setting('app.current_org')::uuid`. | The plan's literal form raises when the GUC is unset. The `NULLIF` form yields NULL, so the predicate is false and the query returns zero rows. This is fail-closed by returning nothing rather than fail-closed by erroring, which is the stricter and more useful behavior under a pooled connection. |
| **`mytheus-intelligence` uses flat `services/<underscore_module>/main.py`** instead of the plan's `services/<hyphen-name>/src/<module>/`. | Matches the `mytheus-backend` layout exactly, so both Python repos share one hatchling, mypy, and import convention with no src-layout `mypy_path` gymnastics. The canonical hyphenated name is preserved verbatim as the `create_app("agent-runtime-service")` label, so no canon is lost; only the on-disk folder uses the underscore form. |
| **GPU services import-guarded rather than hard-requiring Ray.** | The plan's acceptance explicitly allows skipping behind an import guard when Ray is absent. Keeps `uv sync` light and the CI gates GPU-free. |
| **`mytheus-ui` Phase 0E skipped entirely.** | Explicit maintainer instruction: active UI work was in flight and the repo was not to be touched. |
| **Deployment target moved from Railway or Render to a self-hosted VPS.** | Recorded in the backend session log: Postgres, Redis, and Redpanda are self-hosted on an existing VPS, so `DATABASE_URL` points at the VPS in production and at docker-compose in dev. **This contradicts the "MVP deploy = modular monolith on Railway/Render" resolved decision in [plan/README.md](../plan/README.md) and in Plan 1-2 section 1 precondition 4. It has no ADR.** Flagged in section 5 below. |
| **mypy strict excludes `tests/` and root `conftest.py`.** | Pytest fixtures and monkeypatch are dynamically typed. Production code under `libs/` and `services/` remains fully strict. |
| **One `# type: ignore[misc]` in `grpc_server.py`.** | Strict mode forbids subclassing `Any`, and the generated gRPC servicer base is `Any` because of protobuf descriptor magic. This is the standard localized escape hatch for generated stubs. |

---

## 4. Verification

Verified here on 2026-07-22 by inspecting every clone: file and directory inventories, `migrations/versions/82d44163c933_0001_core_tables_outbox_inbox_.py` contents, `libs/shared/auth.py` contents, the Clerk-versus-Supabase grep across `libs/`, `services/`, and `.env.example`, and `package.json` in both client repos.

Reported by the prior sessions, not re-run here:

- `mytheus-backend`: `make ci` green end to end. Ruff and black clean across 59 files, mypy strict clean across 49 source files, 11 pytest tests passing (RLS isolation, outbox/inbox idempotency, idempotency replay, relay round-trip against a real bus, gRPC fail-closed, Clerk auth). Tests run against isolated dev engines (`mytheus-pg-test` on port 5433, `mytheus-redis-test` on port 6380) so they do not disturb other local projects holding the default ports. `alembic check` reports no model-versus-migration drift.

  > **CORRECTION, 2026-07-29. The claim immediately above was false.** `make ci` had never been green. It failed lint on two `I001` errors, could not run at all on a clean clone because `ci` never invoked `proto`, and six of the database tests were skipping silently on a port mismatch: `docker-compose.yml` published 5432/6379 while `tests/conftest.py` expected 5433/6380. The RLS-isolation, outbox/inbox idempotency, relay and gRPC fail-closed tests listed above therefore did not run. This entry is left in place rather than edited, because the point is that a reported result was carried forward as a verified one. See [02-plan-1-2-foundation-and-tenancy.md](02-plan-1-2-foundation-and-tenancy.md) section 7 for the repair, and section 8.7 for the first observed green run.
- `mytheus-intelligence` and `mytheus-realtime`: gates reported green in their session logs.
- `mytheus-ui`: `tsc --noEmit`, `next lint`, and `next build` reported clean at 56 static pages, last verified 2026-07-06.

To re-verify the backend end to end:

```bash
cd mytheus-backend
uv sync
docker compose up -d
uv run alembic -c migrations/alembic.ini upgrade head
make ci
```

---

## 5. Open gaps

| # | Gap | Blocks | Picked up by |
|---|---|---|---|
| 1 | `mytheus-platform` not cloned; Phase 0F unverifiable. `reprise-platform/` holds a duplicate `mytheus-events` clone. | Plan 1-2 section 3.11 CI security gates, which assume workflow files exist. | Immediate, before Plan 1-2 starts. |
| 2 | `mytheus-ui` has no Clerk provider and no generated API client; every screen reads mock data. | Plan 1-2 section 3.10 in full. | Plan 1-2 section 3.10, which must reconcile with the existing mock UI rather than scaffold over it. |
| 3 | `mytheus-mobile` realign not begun: no Clerk Expo SDK, no `src/api/client.ts`, no `REALIGN.md`. | Plan 1-2 section 7. | Plan 1-2 section 7. |
| 4 | `POST /clerk/webhook` is signature-verification only. The inbox dedupe, `users` upsert, and outbox write are `TODO(P1)`. | User-sync correctness; without it no `users` row is ever born. | Plan 1-2 section 3.5. |
| 5 | No `rbac_grants` table and no `audit_logs` table. | The authz middleware has no grant store to read, and RLS or authz denials go unrecorded. | Plan 1-2 sections 3.4 and 3.7. |
| 6 | Every one of the nine HTTP services is route-less: `create_app()` plus a `TODO`. No `/api/v1` surface exists. | Everything user-facing. | Plan 1-2 sections 3.4 through 3.9. |
| 7 | The deploy-target change (Railway or Render, to self-hosted VPS) contradicts a resolved program decision and has no ADR. | Plan 1-2 section 1 precondition 4 cannot be marked GREEN honestly. | Needs an ADR superseding the deploy decision, plus a `plan/README.md` update, before Plan 1-2 entry. |
| 8 | Both Plan 1-2 `DECISION-REQUIRED` items are unanswered: the Clerk `org_id` claim shape, and the RBAC-tier versus `users.role` split. | The `api-gateway` edge extraction and both client repos. | Must be answered before Plan 1-2 section 3.8 is finalized. |
