# Plan 1-2 — Foundation & Tenancy  (Foundation)

> Stand up the tenant-isolated CRUD spine — `identity-tenant-service`, `core-api`, `bff`, `web-app`, `api-gateway` — with PostgreSQL RLS and `org_id` injection live from the very first migration, `audit_logs` recording from the moment RLS goes live, and the P1 design artifacts + security shift-left + compliance-by-construction underneath it.

---

## 0. Band at a glance

| Field | Value |
|---|---|
| **Phases** | **P1 Architecture & Design (W1–W2)** + **P2 Foundation (W3–W5)** |
| **Theme** | Foundation |
| **Goal** | The tenant-isolated CRUD spine: Clerk auth at the edge, `org_id` extracted once at `api-gateway`, RLS `FORCE` on `org_id` on every tenant table from migration #1, `audit_logs` append-only live at P2, and a `web-app` dashboard that renders an org-scoped view through `bff`. Plus the P1 design pack (C4/ERD/Avro/OpenAPI/wireframes/tech-lock), the STRIDE threat model, data-residency region decision, subprocessor DPAs, and the RTBF deletion contract. |
| **Repos & folders touched** | `mytheus-backend` (`libs/shared`, `services/api-gateway`, `services/bff`, `services/identity-tenant-service`, `services/core-api`, `services/admin-ops-service`, `migrations/`, `proto/`, `docker-compose.yml`, `.env.example`, `.github/workflows`), `mytheus-ui` (net-new Next.js 14 scaffold), `mytheus-mobile` (realign off the mock prototype), `mytheus-docs` (`/design` artifacts, `adr/`, threat model) |
| **Demoable at the end** | **P1 demo:** the design pack walkthrough (C4 L1/L2, full ERD, Avro draft subjects, OpenAPI `/api/v1` draft, dashboard + sign-in wireframes, signed-off tech lock, STRIDE model, residency-region pick, executed DPAs, RTBF contract). **P2 demo:** a real user signs in through Clerk on `web-app`, lands on a dashboard that lists only their org's meetings/actions/worklogs/decisions via `bff`; creating a meeting in `core-api` writes an `audit_logs` row; a cross-tenant `GET /api/v1/meetings/:id` returns `404`; a forced-bad query with no `app.current_org` returns zero rows (RLS fail-closed proven in CI). |

This band builds **only** the CRUD/control-plane spine. No agents, no transcription, no Qdrant/AGE, no LiveKit media, no billing — those land in later bands. The seams (one writer per store, `org_id` on every hop, outbox on every produce) are laid now so nothing has to be re-cut later.

---

## 1. Entry preconditions

Everything below must be GREEN before the first P2 migration is written.

| # | Precondition | Where it is verified |
|---|---|---|
| 1 | **Plan 0 outputs exist**: the `mytheus-backend` `libs/shared` (tenancy ContextVar + `SET LOCAL app.current_org`, transactional-outbox enqueue, inbox Redis-dedupe, `EventEnvelope`, Idempotency middleware, RFC7807 errors, OTel), `docker-compose` (Postgres/Redis/Redpanda/Temporal), `Makefile`, `pyproject.toml`. All 10 `services/*/main.py` boot FastAPI (no routes yet). | `docker compose up -d && uv run uvicorn services.core_api.main:app --port 8000` returns a healthy `/healthz` (the probe is `/healthz`, mounted by `libs/shared/app_factory.create_app`). |
| 2 | **Canon is frozen and authoritative** for every name used here: services from `_canon/01-service-catalog.md`, tables from `_canon/02-data-model.md`, topics from `_canon/03-event-taxonomy.md`, tech from `_canon/06-tech-decisions.md`. | Name lookups in §2. |
| 3 | **Resolved decision: IDENTITY = CLERK.** The Supabase-Auth drift **was already reverted during Plan 0** - Clerk RS256 JWKS verification is live in `libs/shared/auth.py`, `POST /clerk/webhook` verifies Svix, `CLERK_*` is restored in `.env.example`, and the hook and trigger SQL files are deleted with a CI grep guard. §3.2 is therefore **verify-only**: confirm the acceptance checks still pass, do not redo the revert. No P2 migration may depend on the Supabase Auth hook/trigger. | §3.2 acceptance checks. |
| 4 | **Resolved decision: MVP deploy = modular monolith on a self-hosted single-node VPS** under Docker Compose ([ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md)), single region, pooled Postgres + RLS, with PostgreSQL/Redis/Redpanda/Temporal self-hosted on the same box behind Caddy. Railway and Render are struck. No Kubernetes/Terraform/cells/multi-region in this band (those are post-W36, Plan 14). | §0, §6; `mytheus-platform/deploy/compose.prod.yml`. |
| 5 | **ADR-0002 (multi-tenancy)** and **ADR-0010 (Clerk + enterprise SSO + hardened user-sync)** are accepted and read by the implementing agent. | `adr/0002-multi-tenancy-model.md`, `adr/0010-auth-clerk-enterprise-sso.md`. |
| 6 | **Gate: both DECISION-REQUIRED items in §8 are now RESOLVED** - (1) the tenant claim is Clerk's **native `org_id`**, resolved at the edge to our `org_id` via `organizations.clerk_org_id` with a Redis-cached immutable mapping; (2) `rbac_grants.tier` and `users.role` are **separate stores**. Build to these; do not reopen them. | §8. |
| 7 | **Plan 0 already landed part of P2.** Do not rebuild: §3.2 in full, the §3.2 step 5 Idempotency mount, the §3.3 RLS baseline (migration `82d44163c933` creates 11 tables with `FORCE` RLS), the §3.5 step 3 outbox relay, and the §3.4 step 5 gRPC `GetUserContext` stub. The current build state per repo is recorded in [work-done-till-now/00-status-overview.md](../work-done-till-now/00-status-overview.md); read it before starting any step. | The band-by-repo matrix in that file. |

---

## 2. Canonical names introduced in this band

Use these VERBATIM. Never the coarse `_canon/02-data-model.md` service labels (`identity-service`, `tenant-service`, `audit-service`); the Service Catalog (`_canon/01-service-catalog.md`) wins on service names.

**Services (this band):** `api-gateway`, `bff`, `identity-tenant-service`, `core-api`, `admin-ops-service` (only its `audit_logs` ownership lands here), `web-app`.

**Tables (this band):**

| Table | Owning service | Tenant key | Notes |
|---|---|---|---|
| `organizations` | `identity-tenant-service` | tenant root (`id` = `org_id`) | never partitioned away; `clerk_org_id` maps Clerk org → our root |
| `users` | `identity-tenant-service` | `org_id` | `clerk_user_id`, `email`, `role`, `skills text[]`, `bio`, `avatar_url`; soft-delete on RTBF. `role` is a **domain** attribute, NOT a permission tier |
| `teams` | `core-api` | `org_id` | per `03-container-architecture.md` §5 / backend `system.md` §2.3 note; identity owns the identity-and-tenancy surface, `core-api` owns org-hierarchy CRUD |
| `team_members` | `core-api` | `org_id` (denorm.) | PK `(team_id, user_id)` |
| `departments` | `core-api` | `org_id` | self-referential `parent_id` |
| `tenant_settings` | `identity-tenant-service` | `org_id` (PK) | `default_locale`, `retention_days_*`, `agent_confidence_overrides jsonb`, `pii_redaction_enabled`, `sso_config jsonb` |
| `data_residency` | `identity-tenant-service` | `org_id` (PK) | `region` (`us`/`eu`/`ap`), `db_cell`, `qdrant_region`, `object_bucket_region`, `locked_at` |
| `meetings` | `core-api` | `org_id` | `title`, `scheduled_start/end`, `livekit_room`, `status`, `created_by` |
| `meeting_participants` | `core-api` | `org_id` | PK `(meeting_id, user_id)`, `role_in_meeting` |
| `audit_logs` | `admin-ops-service` | `org_id` | append-only (no UPDATE/DELETE), tamper-evident, monthly range-partition |
| **RBAC grants store** | `identity-tenant-service` | `org_id` | `owner`/`admin`/`member` tiers, **distinct from `users.role`** (see §8 decision #2) |
| `outbox` / `inbox` | per owning service | `org_id` | infra tables (transactional outbox + inbox dedupe), one history in `migrations/` |

**Topics (this band — producers land here, full consumers in later bands):** `user-sync-events` (Clerk inbox; producer = consumer = `identity-tenant-service`), `tenant-events` (producer `identity-tenant-service`; org-lifecycle only, log-compacted per `org_id`), `meeting-events` (producer `core-api`), `notifications` (producer `identity-tenant-service`). Each paired with `<topic>.dlq`. Avro + Schema Registry, subject `{topic}-value`, BACKWARD compat, partitioned by `org_id`.

**Frozen identifiers used verbatim:** `org_id`, `app.current_org`, `nexus:{org_id}:` Redis prefix, `get_user_context`. Redis keys this band: `nexus:{org_id}:user_ctx:{user_id}` (5-min TTL), `nexus:{org_id}:dedupe:{topic}:{idempotency_key}` (24h TTL).

**Canonical session GUC:** `app.current_org`, set transaction-local via `SELECT set_config('app.current_org', :org_id, true)`.

---

## 3. Work breakdown (ORDERED)

Ordered as a precondition chain. P1 produces the design contract; P2 builds against it. Within P2 the order is: revert Clerk drift → migration baseline (outbox/inbox/organizations + RLS macro) → identity tables + user-sync → core-api tables/CRUD → audit_logs → api-gateway edge + authz → bff → web-app → CI. Each step states WHY it precedes the next where it matters.

---

### PHASE P1 — Architecture & Design (W1–W2)

#### 3.1 P1 design artifacts

**Repo / folder:** `mytheus-docs/design/` (new), `mytheus-docs/adr/`, `mytheus-docs/threat-model/` (new). These are documentation deliverables that freeze the contract every P2 step builds to; they exist before code so the architecture does not calcify around an un-reviewed decision (the shift-left replacement for the PDF's Phase-13 audit, per `11-security-architecture.md` §9).

**Steps:**

1. **Finalize C4 diagrams.** Produce/confirm `design/c4-l1-system-context.md` and `design/c4-l2-container.md` as copy-pasteable Mermaid, derived from `02-system-context.md` (L1) and `04-service-catalog.md`/`_canon/01-service-catalog.md` §2 (L2). L1 must carry the four trust zones (untrusted → edge → platform → per-tenant `org_id`) and the subprocessor zone; L2 must show the three scaling families and the `web-app → Cloudflare → api-gateway → bff → services` edge plus the `web-app ↔ realtime-gateway` WSS leg.
2. **Finalize ERD.** `design/erd.md` — the full entity ERD from `05-data-architecture.md` §2 and `_canon/02-data-model.md` §2, every tenant-scoped table carrying `org_id UUID NOT NULL`. Mark the seven range-partitioned tables (`transcript_chunks`, `chat_messages`, `meetings`, `worklogs`, `agent_suggestions`, `usage_events`, `audit_logs`). This band implements only the subset in §2; the ERD is the whole target so migrations never re-cut.
3. **Avro drafts (from Plan 0).** `design/avro/` — draft `*.avsc` for the 7 FROZEN schemas (`TranscriptChunk`, `AgentResult`, `Decision`, `Action`, `Notification`, `AgentRequest`, `AgentResponse`) plus the SaaS-add envelope schemas `TenantEvent`, `MeetingEvent`, `UserSyncEvent`. Every value opens with `EventEnvelopeHeader` (`event_id`, `idempotency_key`, `org_id`, `schema_version`, `producer`, `occurred_at`, `correlation_id`, `causation_id`). Only `TenantEvent`, `MeetingEvent`, `UserSyncEvent`, `Notification` are wired in this band; the rest are drafted for traceability and land in their owning bands. Coordinate with `mytheus-events`.
4. **OpenAPI draft.** `design/openapi/` — one OpenAPI 3.1 YAML per service touched this band (`api-gateway` aggregate, `bff`, `identity-tenant-service`, `core-api`, `admin-ops-service`). Encode `19-api-architecture.md`: `/api/v1` prefix, `snake_case` JSON, cursor pagination shape, RFC 7807 `application/problem+json` with the stable `code` enum, `Idempotency-Key` on mutating POSTs, **`org_id` never in path or query**, security scheme declared on every route.
5. **Wireframes.** `design/wireframes/` — sign-in (Clerk), the app shell nav, and the **user dashboard** (upcoming meetings, action items, worklogs, decisions) per `20-frontend-architecture.md` §7. These are the screens P2 §3.10 builds.
6. **Tech-stack lock.** `design/tech-lock.md` — restate the locked picks from `_canon/06-tech-decisions.md` that this band uses: FastAPI, SQLAlchemy 2.0 async + asyncpg, Pydantic v2, Alembic, PostgreSQL via Supabase, PgBouncer transaction pooling, Redis (Upstash), Redpanda + Schema Registry, **Clerk** identity, Next.js 14 App Router, Tailwind + shadcn/ui. Sign-off gate: the team initials this doc before the first migration.
7. **Author the threat model.** `threat-model/stride.md` — the STRIDE model from `11-security-architecture.md` §8: 6 surfaces (auth, transcripts, multi-agent context, APIs, integrations, subprocessors) × 6 STRIDE categories, every control tied to a canonical mechanism. The band-relevant rows (Auth, APIs) must be complete; transcript/multi-agent/integration rows are stubbed to their canon controls for later bands.
8. **Decide data-residency regions.** `design/data-residency.md` — confirm the MVP region set is `us`/`eu`/`ap` (the `data_residency.region` enum), and that MVP deploy is **single region** (the self-hosted VPS) with the table populated per-tenant so the post-W36 cell/multi-region move is mechanical. Record which region is the MVP default.
9. **Execute subprocessor DPAs.** `design/subprocessors.md` — the five raw-T4 subprocessors (Deepgram, OpenAI, Anthropic, LiveKit) plus Qdrant/Redpanda from `02-system-context.md` §6.2 get signed DPAs + SCCs. **This band must execute Deepgram / OpenAI / Anthropic / LiveKit DPAs** even though their services are not built yet, because compliance-by-construction requires the subprocessor list before any T4 data flows. Record signed/pending status per processor.
10. **Design the RTBF deletion contract.** `design/rtbf-contract.md` — the single `org_id`-parameterized cross-store deletion contract from `05-data-architecture.md` §8.1 / `_canon/02-data-model.md` §7: Postgres (drop/purge), Qdrant (delete by filter), AGE (remove `$org_id` nodes/edges), Redis (`nexus:{org_id}:*` flush), object storage (`{org_id}/*` prefixes), with the legal-hold carve-out (`nexus-invoices`, `nexus-audit-archive`). This band implements **none** of the stores beyond Postgres, but the contract is frozen so the Temporal RTBF workflow (later band) has its spec, and the P2 tables are designed deletable (soft-delete + `org_id` keying).

**Canonical names:** C4, ERD, Avro `EventEnvelopeHeader`, OpenAPI 3.1, STRIDE, `data_residency.region` (`us`/`eu`/`ap`), DPA/SCC, RTBF.

**Acceptance checks:**
- [ ] Each artifact exists at the path above and is reviewed in a PR.
- [ ] `design/tech-lock.md` is signed off (initials present).
- [ ] `design/avro/*.avsc` validate against a draft Schema Registry (`rpk registry schema check` or local avro tooling) with BACKWARD compat.
- [ ] `design/openapi/*.yaml` pass `spectral lint` with the ruleset enforcing `org_id`-never-in-path, RFC 7807 `code` enum, cursor pagination, `Idempotency-Key`-on-mutations, security-scheme-present.
- [ ] DPA status table shows Deepgram/OpenAI/Anthropic/LiveKit as signed or explicitly pending-with-owner.
- [ ] `design/data-residency.md` records the MVP default region and confirms single-region deploy.

---

### PHASE P2 — Foundation (W3–W5)

#### 3.2 Revert the Supabase-Auth drift back to Clerk (`libs/shared` + env)

**Repo / folder:** `mytheus-backend/libs/shared/`, `mytheus-backend/.env.example`, `mytheus-backend/migrations/`. This is FIRST in P2 because every downstream step (webhook user-sync, api-gateway pre-validation, authz middleware) depends on Clerk JWT verification existing; building on the drifted Supabase Auth path would re-poison the foundation.

**Steps:**

1. **Re-add `CLERK_*` env.** In `.env.example` restore:
   ```dotenv
   # Identity = Clerk (canonical; reverts the Supabase Auth drift)
   CLERK_SECRET_KEY=sk_test_...
   CLERK_JWKS_URL=https://<your-clerk-subdomain>.clerk.accounts.dev/.well-known/jwks.json
   CLERK_WEBHOOK_SECRET=whsec_...        # Svix signing secret for /clerk/webhook
   CLERK_ISSUER=https://<your-clerk-subdomain>.clerk.accounts.dev
   CLERK_JWT_ORG_CLAIM=org_id            # see §8 DECISION-REQUIRED #1 — the claim carrying the tenant
   ```
   Remove any `SUPABASE_JWT_SECRET` / Supabase-Auth-only vars from `.env.example`.

2. **Restore Clerk JWKS verification in `libs/shared`.** Replace the Supabase-Auth verifier with a Clerk JWKS verifier:
   ```python
   # libs/shared/auth/clerk.py
   import time
   import httpx
   from jose import jwt
   from jose.exceptions import JWTError
   from functools import lru_cache
   from libs.shared.config import settings
   from libs.shared.errors import AuthRequired, TokenExpired

   _JWKS_CACHE: dict = {"keys": None, "fetched_at": 0.0}
   _JWKS_TTL = 600  # 10 min; honors Clerk key rotation without a verification outage

   async def _jwks() -> dict:
       now = time.time()
       if _JWKS_CACHE["keys"] is None or now - _JWKS_CACHE["fetched_at"] > _JWKS_TTL:
           async with httpx.AsyncClient(timeout=3.0) as c:
               resp = await c.get(settings.CLERK_JWKS_URL)
               resp.raise_for_status()
               _JWKS_CACHE.update(keys=resp.json(), fetched_at=now)
       return _JWKS_CACHE["keys"]

   async def verify_clerk_jwt(token: str) -> dict:
       """Verify signature vs Clerk JWKS, check exp/nbf/iss/aud. FAIL CLOSED."""
       try:
           claims = jwt.decode(
               token,
               await _jwks(),
               algorithms=["RS256"],
               issuer=settings.CLERK_ISSUER,
               options={"verify_aud": False},  # Clerk session tokens: aud optional
           )
       except jwt.ExpiredSignatureError:
           raise TokenExpired()
       except JWTError:
           raise AuthRequired("invalid token")
       return claims

   def extract_org_id(claims: dict) -> str | None:
       """Tenant claim per §8 decision #1. May be Clerk's org id (→ lookup organizations.clerk_org_id)
       or our org_id via a custom JWT template claim. Resolved in §3.8."""
       return claims.get(settings.CLERK_JWT_ORG_CLAIM)
   ```
   On JWKS unreachable, the verifier raises and the gateway fails **closed** for protected routes (caches keys for the TTL so a Clerk blip serves cached keys — `04-service-catalog.md` §3.1).

3. **Restore the Svix webhook verifier helper** in `libs/shared/auth/svix.py` (used by §3.5):
   ```python
   # libs/shared/auth/svix.py
   from svix.webhooks import Webhook, WebhookVerificationError
   from libs.shared.config import settings
   from libs.shared.errors import AuthRequired

   def verify_svix(payload: bytes, headers: dict[str, str]) -> dict:
       try:
           return Webhook(settings.CLERK_WEBHOOK_SECRET).verify(payload, headers)
       except WebhookVerificationError:
           raise AuthRequired("bad webhook signature")  # Spoofing → reject
   ```
   ```bash
   uv add svix python-jose[cryptography] httpx
   ```

4. **Remove/replace the Supabase Auth hook + trigger.** Add an Alembic migration that DROPS the drifted objects so they can never inject `org_id` again (Clerk does that at the edge now):
   ```python
   # migrations/versions/XXXX_revert_supabase_auth_to_clerk.py  (op.execute, idempotent)
   def upgrade() -> None:
       op.execute("DROP FUNCTION IF EXISTS public.custom_access_token_hook(jsonb) CASCADE;")
       op.execute("DROP TRIGGER IF EXISTS on_auth_user_created ON auth.users;")
       op.execute("DROP FUNCTION IF EXISTS public.handle_new_auth_user() CASCADE;")
       # users.org_id now comes from the Clerk webhook user-sync (§3.5), not an auth.users trigger.
   ```
   If the drift never reached the DB (only code), this migration is a no-op DROP IF EXISTS and still records the reversion in history.

5. **Mount the (previously NOT mounted) Idempotency middleware** in `libs/shared` so it is available to mutating routes (it ships from Plan 0 but is not mounted). Expose it as a FastAPI dependency / middleware factory the services attach in §3.5/§3.6.

**Canonical names:** Clerk, JWKS, Svix, `app.current_org`, `CLERK_*`.

**Acceptance checks:**
- [ ] `grep -ri supabase libs/shared services` returns no Auth-hook/trigger references in the request path.
- [ ] A unit test verifies a Clerk-signed JWT (test JWKS) passes and a tampered/expired one raises `AuthRequired`/`TokenExpired`.
- [ ] `uv run alembic upgrade head` runs the revert migration cleanly; `\df custom_access_token_hook` in psql shows no such function.
- [ ] `.env.example` contains `CLERK_SECRET_KEY`, `CLERK_JWKS_URL`, `CLERK_WEBHOOK_SECRET`, `CLERK_ISSUER`, `CLERK_JWT_ORG_CLAIM` and no Supabase-Auth secret.

---

#### 3.3 Migration baseline: outbox/inbox, `organizations`, and the RLS policy macro

**Repo / folder:** `mytheus-backend/migrations/`, `mytheus-backend/libs/shared/db/`. This precedes all table work because (a) the canonical RLS policy must be applied to every table without exception, (b) `organizations` is the tenant root every other table FKs to, and (c) the outbox/inbox tables are needed before any producer/consumer is wired.

> **Status: MOSTLY DONE in Plan 0.** Migration `82d44163c933` already creates 11 tables (`organizations`, `users`, `teams`, `team_members`, `departments`, `tenant_settings`, `data_residency`, `meetings`, `meeting_participants`, `outbox`, `inbox`) with `ENABLE` + `FORCE ROW LEVEL SECURITY` and a `tenant_isolation` policy on each. Two deviations from the text below, both deliberate and both kept: the RLS helper is inline in the migration rather than a `libs/shared/db/rls.py` function, and the predicate is `NULLIF(current_setting('app.current_org', true), '')::uuid` so an unset GUC returns **zero rows** instead of raising. That is stricter than the SQL in step 1 and is the form to keep. **What actually remains in this step:** add `organizations.clerk_org_id text UNIQUE` (required by the resolved §8 decision 1), and add the `mytheus_app` / `mytheus_admin` roles from step 2.

**Steps:**

1. **RLS policy helper.** Add a SQL macro used by every tenant-table migration:
   ```python
   # libs/shared/db/rls.py
   def enable_rls(table: str) -> list[str]:
       """Canonical RLS, frozen in _canon/02-data-model.md §7. FORCE applies to the table owner too."""
       return [
           f"ALTER TABLE {table} ENABLE ROW LEVEL SECURITY;",
           f"ALTER TABLE {table} FORCE ROW LEVEL SECURITY;",
           f"""CREATE POLICY tenant_isolation ON {table}
                 USING      (org_id = current_setting('app.current_org')::uuid)
                 WITH CHECK (org_id = current_setting('app.current_org')::uuid);""",
       ]
   ```

2. **Application role has no `BYPASSRLS`.** Add a migration establishing the request-path DB role (`mytheus_app`) without `BYPASSRLS`, and a separate elevated role (`mytheus_admin`) for migrations/RTBF purges that is unreachable from request handlers (ADR-0002 D3).

3. **`organizations` (tenant root).**
   ```sql
   CREATE TABLE organizations (
     id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),  -- id IS the org_id
     clerk_org_id  text UNIQUE,            -- maps a Clerk org → our tenant root (see §8 #1)
     name          text NOT NULL,
     slug          text NOT NULL UNIQUE,
     plan_id       uuid,                   -- FK added when plans lands (later band)
     created_at    timestamptz NOT NULL DEFAULT now(),
     updated_at    timestamptz NOT NULL DEFAULT now(),
     deleted_at    timestamptz
   );
   ```
   `organizations` is **not** RLS-policied on itself the same way (it is the root), but reads of it are still scoped: an org may read only its own row, enforced at the service/authz layer plus a policy `USING (id = current_setting('app.current_org')::uuid)`.

4. **`outbox` and `inbox` infra tables** (one Alembic history, per backend `CLAUDE.md` "Where things live"):
   ```sql
   CREATE TABLE outbox (
     id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
     org_id          uuid NOT NULL,
     topic           text NOT NULL,
     idempotency_key text NOT NULL,
     payload         jsonb NOT NULL,        -- Avro-encoded envelope serialized for the relay
     correlation_id  text,
     causation_id    text,
     created_at      timestamptz NOT NULL DEFAULT now(),
     published_at    timestamptz,
     UNIQUE (org_id, topic, idempotency_key)
   );
   CREATE INDEX outbox_unpublished_idx ON outbox (created_at) WHERE published_at IS NULL;

   CREATE TABLE inbox (
     org_id          uuid NOT NULL,
     topic           text NOT NULL,
     idempotency_key text NOT NULL,
     consumed_at     timestamptz NOT NULL DEFAULT now(),
     PRIMARY KEY (org_id, topic, idempotency_key)
   );
   ```
   Apply `enable_rls("outbox")` and `enable_rls("inbox")`.

5. **Generate + apply.**
   ```bash
   uv run alembic revision -m "feat: baseline organizations + outbox + inbox + rls roles"
   # hand-edit the migration: organizations, outbox, inbox, roles, enable_rls() calls
   uv run alembic upgrade head
   ```

**Canonical names:** `organizations`, `org_id`, `outbox`, `inbox`, `tenant_isolation`, `app.current_org`, `mytheus_app`/`mytheus_admin`.

**Acceptance checks:**
- [ ] `uv run alembic upgrade head` then `uv run alembic downgrade -1` then `upgrade head` all succeed (forward-only in prod, but reversibility proven in CI).
- [ ] An RLS test: as `mytheus_app`, `INSERT` into `outbox` with `org_id = A` then `SET app.current_org = B` and `SELECT * FROM outbox` returns **zero rows**.
- [ ] A no-GUC test: with `app.current_org` unset, `SELECT * FROM outbox` returns zero rows (fail-closed).
- [ ] `mytheus_app` has no `BYPASSRLS` (`SELECT rolbypassrls FROM pg_roles WHERE rolname='mytheus_app'` → `f`).

---

#### 3.4 `identity-tenant-service` — tenancy tables + RBAC grant store

**Repo / folder:** `mytheus-backend/services/identity-tenant-service/`, `migrations/`. Precedes `core-api` because `users`/`organizations` are FK targets for `meetings.created_by` and `meeting_participants.user_id`, and `get_user_context` reads here.

**Steps:**

1. **SQLAlchemy 2.0 async models** under `services/identity_tenant_service/models.py` for `users`, `tenant_settings`, `data_residency`, and the RBAC grants table. (`teams`/`team_members`/`departments` models live in `core-api` per §2 — identity does not own their CRUD.)
   ```sql
   CREATE TABLE users (
     id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
     org_id        uuid NOT NULL REFERENCES organizations(id),
     clerk_user_id text NOT NULL,
     email         text NOT NULL,
     role          text,                 -- DOMAIN role (job/skills), NOT a permission tier (§8 #2)
     skills        text[] NOT NULL DEFAULT '{}',
     bio           text,
     avatar_url    text,
     created_at    timestamptz NOT NULL DEFAULT now(),
     updated_at    timestamptz NOT NULL DEFAULT now(),
     deleted_at    timestamptz,          -- soft-delete on RTBF
     UNIQUE (org_id, clerk_user_id)
   );

   CREATE TABLE tenant_settings (
     org_id                     uuid PRIMARY KEY REFERENCES organizations(id),
     default_locale             text NOT NULL DEFAULT 'en',
     retention_days_transcripts int  NOT NULL DEFAULT 365,
     retention_days_worklogs    int  NOT NULL DEFAULT 365,
     agent_confidence_overrides jsonb NOT NULL DEFAULT '{}',
     pii_redaction_enabled      boolean NOT NULL DEFAULT true,
     sso_config                 jsonb,   -- envelope-encrypted at rest (per-tenant KMS key) — later band
     created_at  timestamptz NOT NULL DEFAULT now(),
     updated_at  timestamptz NOT NULL DEFAULT now()
   );

   CREATE TABLE data_residency (
     org_id              uuid PRIMARY KEY REFERENCES organizations(id),
     region              text NOT NULL CHECK (region IN ('us','eu','ap')),
     db_cell             text NOT NULL DEFAULT 'pool-us-1',
     qdrant_region       text NOT NULL,
     object_bucket_region text NOT NULL,
     locked_at           timestamptz       -- immutable after locked_at without a migration workflow
   );

   -- RBAC grant store: owner/admin/member, DISTINCT from users.role (§8 #2)
   CREATE TABLE rbac_grants (
     id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
     org_id     uuid NOT NULL REFERENCES organizations(id),
     user_id    uuid NOT NULL REFERENCES users(id),
     tier       text NOT NULL CHECK (tier IN ('owner','admin','member')),
     granted_by uuid REFERENCES users(id),
     created_at timestamptz NOT NULL DEFAULT now(),
     UNIQUE (org_id, user_id)            -- one tier per user per tenant
   );
   ```
   Apply `enable_rls(...)` to `users`, `tenant_settings`, `data_residency`, `rbac_grants`. `data_residency`/`tenant_settings` are the **compliance-by-construction** tables landing now, not at P13.

2. **Migration.**
   ```bash
   uv run alembic revision -m "feat: identity users, tenant_settings, data_residency, rbac_grants"
   uv run alembic upgrade head
   ```

3. **Provisioning seam.** Add `POST /api/v1/orgs` (provision a tenant) that, in one transaction, inserts `organizations` + `tenant_settings` (defaults) + `data_residency` (MVP default region from §3.1 step 8) + an `outbox` row for `tenant-events` (provisioned). This is the only place a tenant root is born; it emits org-lifecycle on `tenant-events` (log-compacted per `org_id`, ADR-0010 D4).

4. **`get_user_context(user_id)` stub.** Implement `GET /api/v1/context/{user_id}` and the gRPC `GetUserContext` returning `{user_id, org_id, role, skills, team_ids, recent_meeting_ids}` from `users` (+ `team_members` read once §3.6 lands). Cache at `nexus:{org_id}:user_ctx:{user_id}` (5-min TTL). This band returns a **stub** shape (role/skills/team; recent meetings empty until core-api participants exist); the agent tier reads it later.
   ```python
   # services/identity_tenant_service/context.py
   async def get_user_context(user_id: str, org_id: str) -> UserContext:
       cached = await redis.get(f"nexus:{org_id}:user_ctx:{user_id}")
       if cached:
           return UserContext.model_validate_json(cached)
       # ... SELECT from users (RLS-scoped); recent_meeting_ids = [] this band
       ctx = UserContext(user_id=user_id, org_id=org_id, role=row.role, skills=row.skills, team_ids=[])
       await redis.set(f"nexus:{org_id}:user_ctx:{user_id}", ctx.model_dump_json(), ex=300)
       return ctx
   ```

5. **proto + gRPC server.** Add `proto/identity.proto` (`GetUserContext(GetUserContextRequest) returns (UserContext)`), regenerate stubs, and stand up the mesh-internal gRPC server (Plan 0 had none). `org_id` travels in gRPC metadata, never the message body; the handler `SET app.current_org` from it.

**Canonical names:** `users`, `tenant_settings`, `data_residency`, `rbac_grants` (RBAC grant store), `get_user_context`, `GetUserContext`, `nexus:{org_id}:user_ctx:{user_id}`, `tenant-events`.

**Acceptance checks:**
- [ ] `POST /api/v1/orgs` creates `organizations` + `tenant_settings` + `data_residency` + an `outbox` `tenant-events` row atomically (rollback test: force a failure, assert no partial rows).
- [ ] RLS test on `users`/`rbac_grants`: tenant A cannot read tenant B's rows; no-GUC returns zero.
- [ ] `GET /api/v1/context/{user_id}` returns the stub shape and populates `nexus:{org_id}:user_ctx:{user_id}` (TTL 300s observed in Redis).
- [ ] gRPC `GetUserContext` answers over the mesh with `org_id` in metadata; a missing `org_id` metadata fails closed.
- [ ] `rbac_grants.tier` and `users.role` are demonstrably separate columns in separate tables (§8 #2 satisfied in schema).

---

#### 3.5 `identity-tenant-service` — Clerk Svix webhook user-sync (inbox dedupe + reconciliation + outbox)

**Repo / folder:** `mytheus-backend/services/identity-tenant-service/`. Depends on §3.2 (Svix verifier) and §3.4 (`users` table). This replaces the Plan-0 `pass` webhook handler and the reverted Supabase trigger as the way a `users` row is born/updated/soft-deleted.

**Steps:**

1. **Webhook receiver `POST /clerk/webhook`.** Verify Svix HMAC FIRST (reject unsigned → Spoofing), then dedupe on the Clerk `event_id` via the inbox (`nexus:{org_id}:dedupe:user-sync-events:{event_id}` + `inbox` table), then apply to `users` in ONE transaction. The inbound Clerk path is an **inbox** problem, not an outbox one (ADR-0010 D4):
   ```python
   # services/identity_tenant_service/webhooks.py
   @router.post("/clerk/webhook")
   async def clerk_webhook(request: Request):
       payload = await request.body()
       evt = verify_svix(payload, dict(request.headers))   # §3.2
       event_id = evt["data"]["id"] if evt["type"].startswith("user") else evt["data"]["id"]
       org_id = resolve_org_from_clerk(evt)                 # maps Clerk org → organizations.id (§8 #1)
       if await inbox_seen(org_id, "user-sync-events", event_id):
           return {"status": "duplicate"}                   # idempotent replay
       async with uow(org_id) as tx:                        # SET app.current_org transaction-local
           match evt["type"]:
               case "user.created" | "user.updated":
                   await upsert_user(tx, org_id, evt["data"])
               case "user.deleted":
                   await soft_delete_user(tx, org_id, evt["data"]["id"])
           await inbox_mark(tx, org_id, "user-sync-events", event_id)
       return {"status": "ok"}                              # 2xx fast; no outbound per-user event
   ```
   Per-user mirror updates do **not** write an outbox row — there is no per-user propagation topic in canon (ADR-0010 D4); seat/ops/saga state is derived downstream.

2. **Hourly reconciliation (safety net for dropped webhooks).** Add the Temporal scheduled job spec in `workflow-service` (worker stub registered in Plan 0) that pulls the Clerk user list per tenant and diffs vs the local `users` mirror: present-in-Clerk-missing-locally → create; deactivated-in-Clerk-active-locally → soft-delete. Idempotent (a diff, not a replay), runs under an elevated read role. This band wires the **activity** that does the diff against `users`; the Temporal schedule registration can be a stub trigger if the Temporal scheduler is not yet live, but the diff activity and its test land here.

3. **Transactional outbox for org-lifecycle only.** `organizations` provisioned/suspended/deleted writes an `outbox` row → `tenant-events` (log-compacted per `org_id`). Wire the **outbox relay/publisher** (Plan 0 had none): a poller that reads `outbox WHERE published_at IS NULL`, publishes to Redpanda with partition key `org_id`, and stamps `published_at`. This is the minimal relay; consumers (`billing-metering-service`, `admin-ops-service`, `workflow-service`) are built in later bands.

4. **SCIM `/scim/v2/*`.** Plan 0 raises `NotImplementedError`. Per §8 decision #1 (SCIM/SSO scope), if SCIM is **deferred** for MVP, leave the routes returning `501 Not Implemented` with an RFC 7807 body and a `# DEFERRED to Enterprise band` marker; if **in scope**, implement create/update/deactivate flowing through the same §3.5.1 user-sync path. Default assumption pending the decision: **deferred**, routes present and clearly stubbed.

**Canonical names:** `user-sync-events`, `tenant-events`, `inbox`, `outbox`, `nexus:{org_id}:dedupe:user-sync-events:{event_id}`, Svix, SCIM.

**Acceptance checks:**
- [ ] An unsigned/mis-signed webhook → `401` (Spoofing rejected); a valid `user.created` creates a `users` row scoped to the resolved `org_id`.
- [ ] Redelivery of the same `event_id` is a no-op (inbox dedupe; second call returns `{"status":"duplicate"}` and writes no second row).
- [ ] The reconciliation diff activity, given a `users` row absent from a mocked Clerk list, soft-deletes it; given a Clerk user missing locally, creates it.
- [ ] The outbox relay publishes a `tenant-events` message with partition key `org_id` and stamps `published_at`; `tenant-events.dlq` exists.
- [ ] `/scim/v2/Users` returns the agreed MVP behavior (501-deferred or implemented) with an RFC 7807 body.

---

#### 3.6 `core-api` — `meetings` + `meeting_participants` (CRUD) + outbox wiring + org-hierarchy

**Repo / folder:** `mytheus-backend/services/core-api/`, `migrations/`. Depends on §3.4 (`users`/`organizations` FK targets). `core-api` is the relational heart; this band builds the meeting CRUD subset and the org-hierarchy tables it owns.

**Steps:**

1. **Models + migration** for `meetings`, `meeting_participants`, `teams`, `team_members`, `departments`:
   ```sql
   CREATE TABLE meetings (
     id             uuid NOT NULL DEFAULT gen_random_uuid(),
     org_id         uuid NOT NULL REFERENCES organizations(id),
     title          text NOT NULL,
     scheduled_start timestamptz,
     scheduled_end   timestamptz,
     livekit_room    text,
     status          text NOT NULL DEFAULT 'scheduled',
     created_by      uuid NOT NULL REFERENCES users(id),
     created_at      timestamptz NOT NULL DEFAULT now(),
     updated_at      timestamptz NOT NULL DEFAULT now(),
     deleted_at      timestamptz,
     PRIMARY KEY (id, scheduled_start)         -- partition key in PK; range-partition by scheduled_start (month)
   ) PARTITION BY RANGE (scheduled_start);
   -- create the current + next month partitions; pg_partman automation deferred to a later band
   CREATE TABLE meetings_2026_07 PARTITION OF meetings FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
   CREATE INDEX ON meetings_2026_07 (org_id, scheduled_start DESC);   -- dashboard listing

   CREATE TABLE meeting_participants (
     meeting_id     uuid NOT NULL,
     org_id         uuid NOT NULL REFERENCES organizations(id),
     user_id        uuid NOT NULL REFERENCES users(id),
     role_in_meeting text,
     PRIMARY KEY (meeting_id, user_id)
   );
   CREATE INDEX ON meeting_participants (org_id, user_id);   -- get_user_context recent meetings

   CREATE TABLE departments (
     id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
     org_id uuid NOT NULL REFERENCES organizations(id),
     name text NOT NULL, parent_id uuid REFERENCES departments(id)
   );
   CREATE TABLE teams (
     id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
     org_id uuid NOT NULL REFERENCES organizations(id),
     name text NOT NULL, department_id uuid REFERENCES departments(id)
   );
   CREATE TABLE team_members (
     team_id uuid NOT NULL REFERENCES teams(id),
     org_id  uuid NOT NULL REFERENCES organizations(id),   -- denormalized for the simple RLS predicate
     user_id uuid NOT NULL REFERENCES users(id),
     role    text,
     PRIMARY KEY (team_id, user_id)
   );
   ```
   Apply `enable_rls(...)` to every table (and inherit the policy onto partitions). `meeting_participants` and `team_members` carry `org_id` denormalized so the RLS predicate is a column compare, not a subquery (`06-multi-tenancy.md` §3.1 rule 2).

2. **CRUD routes** (`19-api-architecture.md` §5.2): `POST /api/v1/meetings` (`Idempotency-Key` honored), `GET /api/v1/meetings` (cursor pagination, filter `status`/`from`/`to`), `GET /api/v1/meetings/:id`, `PATCH /api/v1/meetings/:id`, plus `GET /api/v1/orgs/:id/teams`, `POST /api/v1/teams`, `GET/POST /api/v1/departments`. Every write returns the **canonical persisted resource**, never a partial echo. `org_id` is never a path/query param — derived from the request context (§3.8).

3. **`meeting-events` outbox wiring.** `POST`/`PATCH /meetings` writes an `outbox` row for `meeting-events` (created/updated) in the same transaction as the row write. Reuse the §3.5 relay/publisher. `decisions`/`actions` outbox is NOT wired this band (no confirmation flow yet) — only `meeting-events`.

4. **Idempotency middleware mounted** on the mutating POSTs (the Plan-0 middleware, mounted in §3.2 step 5): `POST /api/v1/meetings` with a replayed `Idempotency-Key` + same body returns the original `201`; different body → `409 IDEMPOTENCY_KEY_REUSED`.

5. **Tenant context dependency.** Every route depends on the shared tenancy helper that `SELECT set_config('app.current_org', :org_id, true)` at transaction start from the injected `org_id` (§3.8). A route that forgets it must fail closed (RLS returns zero rows) — verified in CI.

**Canonical names:** `meetings`, `meeting_participants`, `teams`, `team_members`, `departments`, `meeting-events`, `Idempotency-Key`, `app.current_org`.

**Acceptance checks:**
- [ ] `POST /api/v1/meetings` persists a `meetings` row scoped to the caller's `org_id`, writes a `meeting-events` outbox row in the same txn (atomic rollback test), and returns the canonical resource.
- [ ] `GET /api/v1/meetings/:id` for a meeting in another tenant returns **`404`** (not `403`) — existence is tenant-confidential.
- [ ] Cross-tenant RLS test: tenant B `SELECT` over `meetings` sees zero of tenant A's rows; no-GUC returns zero.
- [ ] Replayed `Idempotency-Key` same-body → original response; different-body → `409`.
- [ ] List endpoint is cursor-paginated (no `OFFSET`), `limit` default 50 / max 200, stable ordering with `id` tiebreaker.

---

#### 3.7 `admin-ops-service` — `audit_logs` (append-only, tamper-evident) stood up NOW

**Repo / folder:** `mytheus-backend/services/admin-ops-service/`, `migrations/`. **Stood up at P2, not P13**, so RLS/authz events are recorded from the moment RLS goes live (`11-security-architecture.md` §8.3, §9). Depends on §3.3 (RLS macro) and precedes §3.8 (the authz middleware audits denials here).

**Steps:**

1. **Migration** for `audit_logs`, append-only:
   ```sql
   CREATE TABLE audit_logs (
     id          uuid NOT NULL DEFAULT gen_random_uuid(),
     org_id      uuid NOT NULL REFERENCES organizations(id),
     actor_id    uuid,
     actor_type  text NOT NULL CHECK (actor_type IN ('user','agent','system')),
     action      text NOT NULL,
     target_type text,
     target_id   text,
     metadata    jsonb NOT NULL DEFAULT '{}',
     ip          inet,
     occurred_at timestamptz NOT NULL DEFAULT now(),
     PRIMARY KEY (id, occurred_at)            -- partition key in PK
   ) PARTITION BY RANGE (occurred_at);
   CREATE TABLE audit_logs_2026_07 PARTITION OF audit_logs FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
   CREATE INDEX ON audit_logs_2026_07 (org_id, occurred_at);
   CREATE INDEX ON audit_logs_2026_07 (org_id, target_type, target_id);
   ```
   Apply `enable_rls("audit_logs")`, then **REVOKE write-back grants** so it is tamper-evident:
   ```sql
   REVOKE UPDATE, DELETE ON audit_logs FROM mytheus_app;   -- append-only: INSERT + SELECT only
   ```

2. **Append helper in `libs/shared`** so every service writes audit identically:
   ```python
   # libs/shared/audit.py
   async def audit(ctx, *, action, target_type=None, target_id=None, metadata=None, actor_type="user"):
       async with uow(ctx.org_id) as tx:
           await tx.execute(insert(audit_logs).values(
               org_id=ctx.org_id, actor_id=ctx.user_id, actor_type=actor_type,
               action=action, target_type=target_type, target_id=str(target_id) if target_id else None,
               metadata=metadata or {}, ip=ctx.ip))
   ```
   An operation that **requires** audit fails closed if the audit write fails (`04-service-catalog.md` §3.9).

3. **Wire the first audited events:** `authz.deny` (from §3.8), `meeting.created` / `meeting.updated` (from §3.6), `org.provisioned` and `user.synced` (from §3.4/§3.5). This proves audit is live the instant RLS is.

**Canonical names:** `audit_logs`, `actor_type` (`user`/`agent`/`system`), `nexus-audit-archive` (archive target, referenced not built).

**Acceptance checks:**
- [ ] `UPDATE audit_logs ...` and `DELETE FROM audit_logs ...` as `mytheus_app` both raise a permission error (tamper-evident proven).
- [ ] Creating a meeting writes an `audit_logs` row with `action='meeting.created'`, `actor_type='user'`, the actor `org_id`.
- [ ] An authz denial (§3.8) writes `action='authz.deny'`.
- [ ] RLS test: tenant B cannot read tenant A's `audit_logs`.

---

#### 3.8 `api-gateway` — JWT pre-validation + `org_id` extraction at the edge + routing + authz re-verification

**Repo / folder:** `mytheus-backend/services/api-gateway/`, `services/*/` (authz middleware in `libs/shared`). Depends on §3.2 (Clerk verifier), §3.4 (RBAC grants), §3.7 (`audit_logs` for denials). This is the single chokepoint that resolves `org_id` once at the edge.

**Steps:**

1. **Edge pre-validation + `org_id` extraction.** At `api-gateway`, verify the Clerk JWT vs cached JWKS, check `exp`/`nbf`/`iss`, extract the tenant claim → `org_id`, attach it to the forwarded request as a signed internal header. Reject malformed/expired with `401` before any service worker is touched. **Resolve the §8 #1 decision here:** if the JWT carries Clerk's org id, look it up to `organizations.clerk_org_id` → our `org_id`; if a custom JWT template injects our `org_id` directly, use it verbatim.
   ```python
   # services/api_gateway/middleware.py
   async def edge_auth(request, call_next):
       if is_public(request.url.path):                 # /clerk/webhook, /health, marketing
           return await call_next(request)
       token = bearer(request)
       claims = await verify_clerk_jwt(token)          # §3.2 — FAIL CLOSED on JWKS miss
       org_claim = extract_org_id(claims)
       org_id = await resolve_org_id(org_claim)        # §8 #1: clerk_org_id lookup OR direct claim
       if org_id is None:
           return problem(401, "AUTH_REQUIRED", "no tenant in token")
       request.headers.__dict__["_list"].append((b"x-mytheus-org", sign(org_id)))
       request.headers.__dict__["_list"].append((b"x-mytheus-user", sign(claims["sub"])))
       return await call_next(request)
   ```
   `org_id` is **never** read from a path or query param.

2. **Routing.** `api-gateway` routes screen-shaped reads to `bff` and resource-shaped/webhook routes straight to the owning service per `_canon/01-service-catalog.md` §6. Per-tenant rate-limit hook is a stub here (full WAF/rate-limit posture is `11-security-architecture.md` §5, later hardening); the band requirement is the JWT + `org_id` chokepoint and routing.

3. **Service-tier authz re-verification middleware** (`libs/shared`, mounted by every service). The gateway is a DDoS shield, not a trust delegation — the service re-verifies the JWT/signed header, resolves the RBAC tier from `rbac_grants`, and makes the permit/deny decision (`11-security-architecture.md` §3.2):
   ```python
   # libs/shared/authz.py
   async def authorize(ctx, route):
       if ctx.org_id is None:
           raise Forbidden("no tenant in context")                 # fail closed
       if route.required_tier and ctx.rbac_tier not in route.required_tier:
           await audit(ctx, action="authz.deny", target_type="route", target_id=route.path,
                       metadata={"reason": "rbac_tier", "required": list(route.required_tier)})
           raise Forbidden(f"requires one of {route.required_tier}")  # 403
       if route.self_scoped and ctx.rbac_tier == "member":
           ctx.enforce_owner_filter = True                          # WHERE user_id = ctx.user_id
       ctx.bind_db_org(ctx.org_id)   # SELECT set_config('app.current_org', org_id, true)
   ```
   `users.role` is **never** consulted for permissions (§8 #2). An unset `org_id` / unknown route / missing tier → `403`, paired with RLS where an unset GUC matches zero rows ("user sees nothing," never "everything").

4. **Cross-tenant returns 404, not 403** (`19-api-architecture.md` §7.2): a resource in another tenant is RLS-invisible, so the handler returns `404 NOT_FOUND`; existence is tenant-confidential. Wire this in the shared resource-fetch helper.

**Canonical names:** `api-gateway`, `org_id`, `app.current_org`, `rbac_grants`, `authz.deny`, `404 NOT_FOUND`, `403 FORBIDDEN_ROLE`/`CROSS_TENANT_DENIED`.

**Acceptance checks:**
- [ ] A request with no/expired/forged JWT → `401` at the gateway, no service touched.
- [ ] A valid JWT resolves to the correct `org_id` (clerk_org_id lookup or direct claim per §8 #1) and is forwarded as a signed header; tampering the header at the service tier is rejected (re-verify).
- [ ] A `member` hitting an `admin`-required route → `403` and an `audit_logs` `authz.deny` row.
- [ ] A cross-tenant resource fetch → `404` (asserted distinct from the `403` role case).
- [ ] No route accepts `org_id` as a path/query param (Spectral `org_id`-never-in-path rule green on the OpenAPI specs).

---

#### 3.9 `bff` — screen-shaped read aggregation (dashboard)

**Repo / folder:** `mytheus-backend/services/bff/`. Depends on §3.4/§3.6/§3.8. `bff` is the only caller of `/bff/*` and the only thing `web-app` reads for screens.

**Steps:**

1. **`GET /bff/dashboard`** — one composite payload for the user dashboard: upcoming `meetings` (next N by `scheduled_start`), open `action_items` assigned to the user (empty until the actions band — return `[]` with the shape), recent `worklogs` (empty shape until the agent band), recent `decisions` (empty shape). This band returns the **real** meetings arm and **shaped-but-empty** arms for the not-yet-built read models, so the screen is stable when those land.
   ```python
   # services/bff/dashboard.py
   @router.get("/bff/dashboard")
   async def dashboard(ctx = Depends(tenant_ctx)):
       meetings = await core_api.list_upcoming_meetings(ctx)      # REST/gRPC fan-out, org_id propagated
       return {
         "upcoming_meetings": meetings,
         "action_items": [],     # shape frozen; populated in the actions band
         "worklogs": [],         # shape frozen; populated in the agent band
         "decisions": [],        # shape frozen; populated in the decisions band
       }
   ```
2. **Per-call timeouts + partial-screen degradation** (`04-service-catalog.md` §3.3): if a downstream arm is slow, return the resolved arms and mark the slow one degraded — never block the whole screen.
3. **Short read-through Redis cache** keyed `nexus:{org_id}:...`; owning service is always the write authority.

**Canonical names:** `bff`, `/bff/dashboard`, `nexus:{org_id}:`.

**Acceptance checks:**
- [ ] `GET /bff/dashboard` returns the composite shape with the real `upcoming_meetings` arm populated org-scoped, and stable empty arms for the rest.
- [ ] `/bff/*` is callable only as a screen aggregate (no resource-mutation routes on `bff`).
- [ ] A forced-slow `core-api` returns a degraded-marked panel, not a 500.

---

#### 3.10 `web-app` shell (Next.js 14 App Router): Clerk auth, nav, user dashboard

**Repo / folder:** `mytheus-ui/` (currently DOCUMENTATION ONLY — scaffold net-new). Depends on §3.8 (gateway) and §3.9 (bff). Mirrors `mytheus-ui/system.md` and `20-frontend-architecture.md`.

**Steps:**

1. **Scaffold.**
   ```bash
   cd mytheus-ui
   npx create-next-app@14 . --typescript --app --tailwind --eslint --src-dir=false --import-alias "@/*"
   corepack enable && corepack prepare pnpm@latest --activate
   pnpm add @clerk/nextjs @tanstack/react-query zustand
   pnpm add -D @types/node
   npx shadcn-ui@latest init        # Button, Input, Modal(Dialog), Card, EmptyState, Avatar, Badge
   ```
2. **Route groups** per `20-frontend-architecture.md` §2.2: `app/(marketing)`, `app/(auth)/sign-in/[[...rest]]`, `app/(auth)/sign-up/[[...rest]]`, `app/(app)/layout.tsx` (the single place `org_id`/theming/locale resolve), `app/(app)/dashboard/page.tsx`, plus `app/(admin)/layout.tsx` (RBAC gate stub).
3. **Clerk wiring.** `<ClerkProvider>` in `app/layout.tsx`; `middleware.ts` protects `(app)` and `(admin)`. Env: `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY` (server only), `NEXT_PUBLIC_API_GATEWAY_URL`, `NEXT_PUBLIC_REALTIME_WS_URL` (set for later bands). The client **never** injects or trusts `org_id` — the gateway derives it from the verified JWT (`20-frontend-architecture.md` §8).
4. **Token / claim setup.** Configure the Clerk **JWT template / session claim** that carries the tenant (the §8 #1 decision) so the session token the gateway pre-validates contains the org claim `CLERK_JWT_ORG_CLAIM` resolves. Document the exact template in `mytheus-ui` README.
5. **App shell nav** (`AppShell` in `(app)/layout.tsx`): sidebar nav (Dashboard, Meetings, Tasks, Decisions, Search, Settings), top bar with org switcher (re-auths through Clerk, never mutates a header) and `Avatar`.
6. **User dashboard (RSC).** `app/(app)/dashboard/page.tsx` is an RSC that fetches `GET /bff/dashboard` server-side with the forwarded JWT and renders: upcoming meetings (`Card` list), action items / worklogs / decisions (`EmptyState` until populated — the shaped-empty arms from §3.9). Compose only `components/ui/` primitives; read tokens, never raw hex.
7. **Generated TS client.** Generate `lib/api/` from the §3.1 OpenAPI spec so wire drift is structurally impossible; the client holds only the gateway origin.

**Canonical names:** `web-app`, `(app)/layout.tsx`, `OrgContext`, `/bff/dashboard`, Clerk JWT template.

**Acceptance checks:**
- [ ] A user signs in via Clerk and lands on `/dashboard`; an unauthenticated user is redirected to `(auth)/sign-in`.
- [ ] The dashboard renders only the signed-in user's org meetings (proven by signing in as two different-tenant users and seeing disjoint data).
- [ ] The browser never sends `org_id` (network inspection: only the bearer JWT; `org_id` appears only server-side).
- [ ] `pnpm build` and `pnpm lint` are green; Playwright a11y smoke on sign-in + dashboard passes (keyboard nav, focus order).

---

#### 3.11 CI security: dependency + secret scanning enforced; authz check reviewed in every PR

**Repo / folder:** `mytheus-backend/.github/workflows/`, `mytheus-ui/.github/workflows/`, root pre-commit. The shift-left replacement for the PDF Phase-13 audit (`11-security-architecture.md` §6.3, §9). Lands now so the very first foundation PRs are gated.

**Steps:**

1. **Secret scanning.** Pre-commit `gitleaks` hook + a CI `gitleaks` job that fails the build on any key pattern. Add `.gitleaks.toml` allowlisting `*.example` placeholders.
   ```yaml
   # .github/workflows/security.yml (excerpt)
   - uses: gitleaks/gitleaks-action@v2
   ```
2. **Dependency scanning (SCA).** CI job running `pip-audit` (Python) and `pnpm audit --audit-level=high` (web); a known-vulnerable dependency fails the PR.
3. **Authz-in-every-PR gate.** CI step asserting every route declares an auth/RBAC requirement in its OpenAPI spec (Spectral rule: security-scheme-present) and that every new route has a test asserting its `required_tier`/self-scope; a route with no authz assertion fails the build (`11-security-architecture.md` §3.2). Add a PR-template checklist item: "authz check reviewed."
4. **RLS-isolation test gate.** CI runs the `testcontainers` suite (real Postgres) including: tenant A cannot read tenant B; no-`app.current_org` returns zero rows; outbox/inbox idempotency; RFC 7807 error shape. A failing RLS test fails the PR (compliance-by-construction).
5. **Spec governance.** Spectral lint + `oasdiff` breaking-change gate against `main` on the OpenAPI specs.

**Canonical names:** gitleaks, SCA, Spectral, `oasdiff`, authz-in-every-PR.

**Acceptance checks:**
- [ ] A PR that adds a hard-coded secret is blocked by `gitleaks` (pre-commit and CI).
- [ ] A PR adding a route with no declared auth fails the Spectral/authz gate.
- [ ] The RLS-isolation testcontainers suite runs in CI and is required for merge.
- [ ] `oasdiff` blocks a breaking OpenAPI change without a major bump.

---

## 4. Data stores stood up in this band

Per the roadmap store-standup schedule (`22-roadmap-and-phasing.md` section 3.1), this band stands up **only the relational + cache + event spine**; vector (Qdrant), graph (AGE), and object storage are later bands.

| Store | What stands up here | Config |
|---|---|---|
| **PostgreSQL (Supabase)** | `organizations`, `users`, `tenant_settings`, `data_residency`, `rbac_grants`, `teams`, `team_members`, `departments`, `meetings` (range-partitioned by `scheduled_start`), `meeting_participants`, `audit_logs` (range-partitioned by `occurred_at`, append-only), `outbox`, `inbox`. RLS `FORCE` on every tenant table from migration #1. | SQLAlchemy 2.0 async + asyncpg, Pydantic v2, Alembic (one history). **PgBouncer transaction pooling**; `app.current_org` set transaction-local via `set_config(..., true)`. `mytheus_app` role has no `BYPASSRLS`; `mytheus_admin` for migrations/RTBF. Single region, self-hosted on the VPS (ADR-0016). |
| **Redis (Upstash)** | `nexus:{org_id}:user_ctx:{user_id}` (5-min TTL), `nexus:{org_id}:dedupe:{topic}:{idempotency_key}` (24h TTL), short `bff` read-through cache. | Namespaced `nexus:{org_id}:`; key builder raises if `org_id` absent. |
| **Redpanda + Schema Registry** | Topics: `user-sync-events`, `tenant-events`, `meeting-events`, `notifications`, each with a `<topic>.dlq`. Avro subjects `{topic}-value`, BACKWARD compat, partitioned by `org_id`. The **outbox relay/publisher** (net-new this band) publishes from `outbox`. | `rpk topic create` (§5). Consumers for these topics are built in later bands; producers land here. |

Cells, multi-region, read replicas, `pg_partman` automation, Qdrant, AGE `nexus_knowledge`, and object buckets are explicitly **out of scope** (post-W36 / later bands).

---

## 5. Events / topics introduced

All Avro + Schema Registry, subject `{topic}-value`, BACKWARD compat, partitioned by `org_id`, each paired with a `<topic>.dlq`. Every value opens with `EventEnvelopeHeader` (`event_id`, `idempotency_key`, `org_id`, `schema_version`, `producer`, `occurred_at`, `correlation_id`, `causation_id`).

```bash
# create topics + DLQs (single-broker dev; production sizing later)
for t in user-sync-events tenant-events meeting-events notifications; do
  rpk topic create "$t"     --partitions 6 --config cleanup.policy=delete
  rpk topic create "$t.dlq" --partitions 6 --config cleanup.policy=delete
done
# tenant-events is log-compacted per org_id (org-lifecycle latest-state)
rpk topic alter-config tenant-events --set cleanup.policy=compact
```

| Topic | Avro schema | Producer (this band) | Consumer (this band) | Notes |
|---|---|---|---|---|
| `user-sync-events` | `UserSyncEvent` | Clerk webhook (inbound) | `identity-tenant-service` (inbox) | inbound-only; `idempotency_key` = Clerk `event_id`; never an outbound topic |
| `tenant-events` | `TenantEvent` | `identity-tenant-service` (outbox) | (later: `billing-metering-service`, `admin-ops-service`, `workflow-service`) | org-lifecycle only; **log-compacted** per `org_id` |
| `meeting-events` | `MeetingEvent` | `core-api` (outbox) | (later: `meeting-service`, `workflow-service`) | created/updated/ended |
| `notifications` | `Notification` (frozen schema) | `identity-tenant-service` | (later: `notification-service`, `realtime-gateway`) | only the producer side wired here |

**Delivery invariant:** transactional outbox in the same DB transaction as the state change → relay publishes with partition key `org_id` → consumers (later bands) dedupe on `idempotency_key` via `inbox` + `nexus:{org_id}:dedupe:*` → poison → `<topic>.dlq`. The `decisions`/`actions` topics are NOT produced this band (no confirmation flow yet).

---

## 6. SaaS overlay / preconditions landing here

This band lands the **security shift-left** and **compliance-by-construction** overlays. Billing+budgets and dedicated inference do **not** apply this band.

- **Security shift-left** (`11-security-architecture.md` §9): the STRIDE threat model is authored in **P1** (§3.1 step 7), not P13. Dependency + secret scanning is enforced from day one (§3.11). The authz check is reviewed in every PR (§3.11) — CI fails a route with no declared auth. The two JWT-verification points (edge pre-validation + service re-verification) and fail-closed authz land in §3.8.
- **Compliance-by-construction** (`11-security-architecture.md` §9 row "Compliance by construction"): RLS `FORCE` is load-bearing from the **first migration** (§3.3); `audit_logs` is append-only and live at **P2** (§3.7), not P13, so RLS/authz events are recorded from the moment RLS goes live; `data_residency` and `tenant_settings` (the residency/retention/PII-redaction compliance tables) land in §3.4; the subprocessor DPAs (Deepgram/OpenAI/Anthropic/LiveKit) are executed in P1 (§3.1 step 9); the RTBF deletion contract is designed in P1 (§3.1 step 10) and every P2 table is built deletable (soft-delete + `org_id` keying).
- **Identity reversion** (resolved decision #1): the Supabase-Auth drift is reverted to Clerk (§3.2) — Clerk JWKS verification restored, Svix webhook restored, `CLERK_*` re-added, the Supabase hook+trigger dropped. Enterprise SSO/SAML/SCIM is scoped in §8 #1 (default: deferred to a later Enterprise band; routes stubbed).
- **Not in this band:** per-tenant token budgets / `agent_budgets` (billing band), Stripe (billing band), Qdrant/AGE per-tenant isolation (intelligence band), LiveKit transport (real-time band), cells / multi-region / K8s / Terraform (post-W36 Plan 14).

---

## 7. Mobile (`mytheus-mobile`) work in this band

`mytheus-mobile` is currently a polished but OFF-SPEC single-device "Meeting Pod" prototype on 100% mock data (`src/data/mock.ts`). This band begins the REALIGN: keep the good UI components and design tokens; remove the mock; add Clerk auth and a real, `org_id`-aware read path to `bff`. (Media/realtime threads in later bands.)

**Repo / folder:** `mytheus-mobile/`. Mirrors `mytheus-mobile/system.md` and the parity rules.

**Steps:**

1. **Add Clerk auth.** Install and wire the Clerk Expo SDK:
   ```bash
   cd mytheus-mobile
   pnpm add @clerk/clerk-expo expo-secure-store @tanstack/react-query
   ```
   - `<ClerkProvider>` (with the `tokenCache` backed by **expo-secure-store**, never AsyncStorage) wraps `app/_layout.tsx`.
   - `app/(auth)/sign-in.tsx` / `sign-up.tsx` for Clerk sign-in/session; `app/(app)/_layout.tsx` is the auth gate and the **single place** `OrgContext` resolves (the mobile analogue of web's `(app)/layout.tsx`).
   - Session/refresh tokens live in **Keychain/Keystore** via the secure-store wrapper; the JWT carries the same `org` claim as web (the §8 #1 template).
   - Env: `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY`, `EXPO_PUBLIC_API_GATEWAY_URL` (no secrets in the bundle).

2. **`org_id`-aware `httpClient` against `bff`.** Build `src/api/client.ts`: the single base fetch wrapper that knows **only** `EXPO_PUBLIC_API_GATEWAY_URL`, injects the Clerk bearer token, parses RFC 7807 errors by the stable `code` enum, and attaches `Idempotency-Key` on mutations. It **never** sets `org_id` (the gateway derives it from the JWT). Expose endpoints as TanStack Query hooks in `src/api/hooks/` (`useDashboard`, `useMeetings`, `useTasks`).

3. **Replace the mock dashboard/home + people/library lists with real `bff` reads.** Delete `src/data/mock.ts`. Rewire:
   - the home/dashboard screen → `GET /bff/dashboard` (the same composite the web shell reads — upcoming meetings real, action-items/worklogs/decisions shaped-empty until later bands);
   - the meeting list → `GET /api/v1/meetings`;
   - the "people" list → org members via `identity-tenant-service` (`GET /api/v1/users/...` through `bff`);
   - the "library" list → meetings/documents read shape (documents empty until the ingestion band).
   Keep the existing **design-token UI** components; only the data source changes. Persisted TanStack Query cache (stale-while-revalidate) is fine for reads; durable truth is never invented client-side.

4. **Keep out of scope this band** (later mobile threads): LiveKit RN media, Socket.io realtime, push registration (the N7 device-token contract), background audio. Those land when the realtime/notification bands do.

**Acceptance checks:**
- [ ] `src/data/mock.ts` is deleted; `grep -ri mock src/` shows no mock-data imports in the home/people/library screens.
- [ ] A user signs in via Clerk on device; tokens are in secure-store (not AsyncStorage), verified by inspecting storage.
- [ ] The home screen renders real `GET /bff/dashboard` data, org-scoped, with no `org_id` ever sent by the client (network capture shows only the bearer token).
- [ ] The people/library lists render real `bff`/`core-api`/`identity-tenant-service` reads; empty states use the design-token UI.
- [ ] `pnpm typecheck` + `pnpm lint` green; a Jest/RNTL test covers the `httpClient` bearer-injection + RFC 7807 parsing.

---

## 8. DECISION-REQUIRED callouts

> **DECISION #1 - RESOLVED: option (A), Clerk's native `org_id` claim, resolved at the edge.**
> The session token carries **Clerk's** org id. `api-gateway` resolves it to our `org_id` through `organizations.clerk_org_id`, cached in Redis. The mapping is immutable once written, so the cache needs no invalidation and the lookup is a memory hit after the first request per org.
> **Why (A) over a custom JWT template (B):** the lookup is not overhead, it is the **authorization checkpoint**. It is where org-exists, not-suspended, subscription-active, and not-soft-deleted are enforced. Under (B) the claim asserts tenancy with nothing to validate it against, so any drift in Clerk metadata (a manual dashboard edit, a restore, a botched migration) becomes a silent cross-tenant read - the failure mode that cannot be detected after the fact. (A) also keeps our `org_id` out of a vendor's metadata blob and avoids the create-in-Clerk → insert-our-row → PATCH-Clerk write-back sync, which needs its own reconciliation. The Clerk webhook already syncs users into Postgres, so the mapping table comes for free in the direction the data already flows.
> **Condition that would have flipped this to (B):** anything verifying the JWT outside our code (for example Postgres RLS reading `request.jwt.claims`). Nothing does - RLS activates off `app.current_org`, set by `libs/shared/tenancy.py`. If that ever changes, revisit.
> **Two implementation requirements this creates:**
> 1. `organizations.clerk_org_id text UNIQUE` must be added in §3.3 - it is specified there but was not built in Plan 0.
> 2. Clerk emits `org_id` **only when the session has an active org**. A user with no active org, or one mid-switch between orgs, gets no claim. Treat **claim-absent as an explicit state** returning `401 AUTH_REQUIRED`, never a `500`. Multi-org users need a token refresh on org switch.
> **Also resolved:** enterprise **SSO/SAML/SCIM is deferred** to a later Enterprise band. `/scim/v2/*` routes present but returning `501` with an RFC 7807 body; `tenant_settings.sso_config` exists unused.

> **DECISION #2 - RESOLVED: both exist, as separate stores.**
> `rbac_grants.tier` (`owner`/`admin`/`member`) governs what a user may *do* - billing, member management, integration install, export/erasure - and is the **only** thing the authz middleware reads. `users.role` + `users.skills[]` is job/domain identity, feeding the role-to-topics taxonomy and `get_user_context` for agent relevance, and is **never** consulted for permissions.
> **Why:** overloading `users.role` would make a profile field a security boundary. A tenant renaming a job title from "Engineering Manager" to "Head of Engineering" would silently change access, and typing "admin" into a job-title field would grant it. It would also couple agent relevance filtering to authorization, so changing how agents match topics to people would mean touching the column the authz middleware reads (`11-security-architecture.md` §3.1, ADR-0010 D5).
> **Already half-built:** `proto/identity.proto` carries both fields separately (`role` commented "NOT an RBAC tier", plus `rbac_tier`). The `rbac_grants` table does not exist yet, so `grpc_server.py` hardcodes `_DEFAULT_TIER = "member"`. §3.4 builds the real table and removes that constant.

---

## 9. Risk checkpoints

This is a foundation band; the headline risks (R1 AGE-on-Supabase, R6 agent-to-agent cross-tenant, R8 integration scope) do not fire here because the graph, agents, and integrations are not built yet. The band-relevant checkpoints:

| Checkpoint | Risk | Mitigation verified in this band |
|---|---|---|
| **Cross-tenant leak (the most severe class)** | An application bug omitting a scope predicate is the only realistic cross-tenant path (ADR-0002 force 3). | RLS `FORCE` from migration #1 (§3.3), no-GUC → zero rows test, cross-tenant → `404`, `mytheus_app` has no `BYPASSRLS`, transaction-local `app.current_org` under PgBouncer (§3.3, §3.6, §3.8 acceptance checks). |
| **Identity reversion correctness** | The Supabase-Auth drift must be fully removed or it silently re-injects `org_id` and forks the trust model. | §3.2: drop the hook+trigger, Clerk JWKS verification restored, `grep -ri supabase` clean in the request path. |
| **R1 (AGE) — deferred** | AGE awkward on managed Supabase. | Not in scope this band; the RTBF contract (§3.1 step 10) and ERD pre-mark the graph so the later band inherits the `$org_id` contract. No action needed now. |
| **R6 (agent A2A) — deferred** | Cross-tenant agent collaboration. | Not built; the `rbac_grants` store laid here is the same grant store the agent-authz scope will use later (ADR-0010 D6). |
| **R8 (integrations) — deferred** | Integration scope creep. | Not built; `integration_connections`/`oauth_tokens` are later-band tables. No action. |
| **R2 (meeting SDK) — locked, not exercised** | SDK lock-in. | LiveKit is locked behind a transport adapter; `meetings.livekit_room` column exists but no media this band. |

---

## 10. Definition of Done & end-of-phase demo

A demo at the end of **every** phase (PDF cadence).

### P1 (W1–W2) — Definition of Done & demo
- **DoD:** all §3.1 artifacts exist, are PR-reviewed, and `design/tech-lock.md` is signed off; OpenAPI specs pass Spectral; Avro drafts validate BACKWARD; STRIDE model authored; residency region chosen; Deepgram/OpenAI/Anthropic/LiveKit DPAs signed-or-pending-with-owner; RTBF contract frozen; both §8 decisions surfaced.
- **Demo:** walk the C4 L1/L2, the full ERD (highlighting `org_id` on every tenant table and the seven range-partition tables), the `/api/v1` OpenAPI draft, the sign-in + dashboard wireframes, the STRIDE matrix (Auth + APIs rows complete), and the subprocessor/DPA + RTBF docs.

### P2 (W3–W5) — Definition of Done & demo
- **DoD:** Clerk drift reverted (§3.2); migration baseline + identity + core-api + audit_logs migrations applied; RLS `FORCE` on every tenant table; Clerk Svix webhook user-sync with inbox dedupe + reconciliation + org-lifecycle outbox; `meetings`/`meeting_participants` CRUD with `meeting-events` outbox; `audit_logs` append-only and live; api-gateway edge JWT pre-validation + `org_id` extraction + service-tier authz re-verification + cross-tenant `404`; `bff` dashboard aggregate; `web-app` shell with Clerk auth + nav + org-scoped dashboard; `mytheus-mobile` Clerk auth + `org_id`-aware httpClient + mock-free home/people/library; CI security gates (gitleaks, SCA, authz-in-every-PR, RLS-isolation testcontainers) enforced. All acceptance checks in §3 green.
- **Demo (the tenant-isolated CRUD spine):**
  1. Sign in as **User A (Org A)** on `web-app` via Clerk → land on a dashboard listing only Org A's upcoming meetings via `bff`.
  2. Create a meeting in `core-api`; show the `audit_logs` row (`meeting.created`) and the `meeting-events` outbox row + its published Redpanda message.
  3. Sign in as **User B (Org B)** → disjoint dashboard; attempt `GET /api/v1/meetings/{orgA_meeting_id}` → **`404`**.
  4. Show the CI run: RLS-isolation test (no-`app.current_org` → zero rows; A cannot read B), gitleaks, SCA, and the authz-in-every-PR gate all green.
  5. On `mytheus-mobile`: sign in via Clerk, show the home screen rendering the **same** `bff` data (mock-free), tokens in secure-store, no `org_id` sent by the client.

---

## 11. Source docs

Authoring references used, by path:

- `/Users/sushan/Projects/Mytheus/mytheus-docs/04-service-catalog.md` — per-service responsibility, owned data, API surface, degradation modes (api-gateway, bff, identity-tenant-service, core-api, admin-ops-service).
- `/Users/sushan/Projects/Mytheus/mytheus-docs/05-data-architecture.md` — RLS policy code, PgBouncer transaction pooling, range partitioning, `app.current_org`, RTBF cross-store contract.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/06-multi-tenancy.md` — the six isolation layers, `org_id` injection chokepoint, transaction-local GUC, cross-tenant prevention, context propagation.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/11-security-architecture.md` — Clerk JWT verify at edge + middleware, RBAC tiers vs `users.role`, hardened user-sync (inbox/reconciliation/outbox), audit_logs append-only, STRIDE, shift-left replacing Phase-13.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/02-system-context.md` — trust zones, subprocessor map + DPAs, data classification, residency regions.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/19-api-architecture.md` — `/api/v1`, RFC 7807 `code` enum, cursor pagination, idempotency keys, `org_id`-never-in-path, 404-for-cross-tenant, webhooks, OpenAPI governance.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/20-frontend-architecture.md` — Next.js 14 App Router, route groups, Clerk, dashboard screen, the client trust model.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/_canon/01-service-catalog.md` — frozen service names, data ownership, sync-API ownership table, topics-by-service.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/_canon/02-data-model.md` — frozen table/column names, tenant keys, RLS notes, Redis namespaces, the canonical RLS policy.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/adr/0002-multi-tenancy-model.md` — pooled+RLS / cells, `org_id` single key, fail-closed backstop, `BYPASSRLS` rule.
- `/Users/sushan/Projects/Mytheus/mytheus-docs/adr/0010-auth-clerk-enterprise-sso.md` — Clerk source of truth, edge+service JWT verify, hardened user-sync, RBAC-vs-`users.role`, agent tenant binding.
- `/Users/sushan/Projects/Mytheus/mytheus-backend/system.md` — the ten bounded contexts, edge path, one-writer-per-store, event delivery, confirmation rule, sync-API ownership.
- `/Users/sushan/Projects/Mytheus/mytheus-ui/system.md` — web-app container, RSC-first, route groups, `bff` reads, Clerk, state layering.
- `/Users/sushan/Projects/Mytheus/mytheus-mobile/system.md` — net-new mobile client, backend contracts, Clerk Expo, secure-store, the realign-from-mock posture, net-new decisions log.
