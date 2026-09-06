# Audit: `mytheus-backend/services/identity_tenant_service/`

READ-ONLY audit. No repo files modified. All citations are from source code, not from any `.md`
status/progress document. Anything not verifiable from source is marked UNVERIFIED.

Scope read in full: all 20 `.py` files of the service, plus `libs/shared/{auth,authz,tenancy,
org_resolver,cache,app_factory,errors,audit}.py`, `migrations/versions/*` (RLS + roles),
`services/api_gateway/{edge,proxy}.py`, `tests/conftest.py` and the 8 identity-related test files.

---

## IMPLEMENTED — genuinely working

### Routes (all mounted in `main.py:18-24`; every one carries a declared auth dependency)

- **`POST /clerk/webhook`** — `routes/auth.py:81`.
  Svix signature verified on the **raw body before any parsing** (`routes/auth.py:88-99`);
  tenant resolved through the shared resolver (`routes/auth.py:113`); two-layer dedupe —
  Redis `is_duplicate` (`routes/auth.py:120`) then the durable `inbox` table as the authority
  (`routes/auth.py:131-139`); apply + `record_consumed` + `audit` all inside **one** transaction
  (`routes/auth.py:128-157`). Real upsert keyed on `(org_id, clerk_user_id)` with tombstone clear
  (`routes/auth.py:169-191`).
- **`PATCH /users/{user_id}`** — `routes/users.py:73`.
  Real UPDATE with `RETURNING`. Ordering is deliberate and correct: RLS-scoped existence check
  runs *before* the self-vs-other 403 (`routes/users.py:80-91`), so a cross-tenant id 404s and the
  403 cannot be used as an existence oracle. Audit row (`routes/users.py:109`), cache eviction
  (`routes/users.py:125`).
- **`GET /context/{user_id}`** — `routes/context.py:32`, backed by `context/service.py:87`
  (cache-then-compute). Real SQL for user, teams, department, recent meetings, topics
  (`context/service.py:139-183`). Every Redis operation fails **open** to Postgres
  (`context/service.py:113-121`, `:129-133`).
- **`GET /taxonomy/role-topics`** — `routes/taxonomy.py:31`. Real YAML load + Pydantic validation;
  unknown version 404s with no silent fallback (`taxonomy/loader.py:118-129`).
- **`GET /tenant-settings`** and **`PATCH /tenant-settings`** — `routes/tenant_settings.py:102`
  and `:126`. Real upsert + audit + Redis mirror write, the mirror being the **last** statement in
  the transaction so a mirror failure rolls the column write back
  (`routes/tenant_settings.py:179`, reasoning `:165-178`).
- **`GET /healthz`** — `libs/shared/app_factory.py:34` (added by `create_app` for every service).
- **gRPC `IdentityService.GetUserContext`** — `grpc_server.py:42`. `org-id` read from gRPC metadata
  only, never the message body; fails closed on missing (`grpc_server.py:57`) and malformed
  (`:64`) metadata. `rbac_tier` is read **uncached** from `rbac_grants`
  (`grpc_server.py:34`, `:89-97`) — the cache-the-mapping-never-the-verdict rule. Infrastructure
  failure aborts UNAVAILABLE and never synthesizes a context (`grpc_server.py:98-112`).

### Auth — real, not decorative

- Clerk RS256 verification against JWKS via `PyJWKClient` (`libs/shared/auth.py:47-49`).
- `jwt.decode` with `algorithms=["RS256"]`, `issuer=`, and
  `options={"require": ["exp","iat","nbf","iss","sub"]}` (`libs/shared/auth.py:94-103`).
- **No decode-without-verify anywhere.** Grepped the whole backend: `jwt.decode` appears only at
  `libs/shared/auth.py:94`, and always with a signing key. No `verify_signature: False`,
  no `options={"verify_...": False}`.
- JWKS/IdP outage → 503 (retryable, attributable), not 500 (`libs/shared/auth.py:69-75`).
  Malformed token → 401, not an unhandled 500 (`:76-91`).
- Missing/empty org claim → explicit 401 (`libs/shared/auth.py:111-117`).
- The org claim is treated as **opaque** and resolved through `organizations.clerk_org_id`
  (`libs/shared/org_resolver.py:70-84`), never cast to a UUID
  (`libs/shared/auth.py:125-132` documents this as a hard rule).
- No session storage of its own: the Clerk JWT is the session. Nothing in this service issues,
  stores or refreshes a session.

### Multi-tenancy — enforced at the database, not by convention

- `set_config('app.current_org', :org, true)` — **transaction-local** (`libs/shared/tenancy.py:32-36`),
  which is what makes it safe on a pooled connection.
- `ENABLE ROW LEVEL SECURITY` + `FORCE ROW LEVEL SECURITY` + a `tenant_isolation` policy on the 10
  tenant tables (`migrations/versions/82d44163c933_0001_core_tables_outbox_inbox_.py:27-34`,
  applied `:156-159`) and on `rbac_grants`
  (`migrations/versions/b7d9f3a55c21_0003_rbac_grants_and_canonical_columns.py:33-44`, applied `:81`).
- Login roles created explicitly `NOSUPERUSER NOBYPASSRLS`
  (`migrations/versions/a1c4e7f20b18_0002_least_privilege_roles_tenant_root.py:88-92`), and
  `DATABASE_URL` in `.env.example` names `mytheus_app`, not `postgres`. This is the difference
  between RLS that applies and RLS that is decorative; the migration docstring records that the
  earlier state was the latter.
- `organizations` is a **deliberate, documented carve-out**: policy dropped, replaced by
  column-level grants plus one sanctioned query shape
  (`a1c4e7f20b18…:117-125`, `libs/shared/org_resolver.py` module docstring). Both readers carry a
  mandatory equality predicate (`org_resolver.py:49-55`) and all three negative outcomes
  (unknown / suspended / soft-deleted) return one byte-identical 401 (`org_resolver.py:58-67`),
  so the table cannot be used as a customer-directory oracle.
- The authorization chain is layered and each layer fails closed:
  `verify_token` → `resolve_org_id` → `tenant_session` (RLS) → `require_tier`
  (`libs/shared/authz.py:7-18`). `RequestContext` is a frozen dataclass so a handler cannot mutate
  `org_id` mid-request (`libs/shared/authz.py:48-60`). An absent grant ranks below `member` and
  satisfies nothing (`:45`, `:62-63`). Denials are audited in their **own** session so the 403's
  rollback does not discard the record (`libs/shared/authz.py:172-185`).
- Queries that *look* unscoped are covered by RLS because `set_rls` runs first in the same
  transaction: `routes/users.py:61`, `context/service.py:39-68`, `reconciliation.py:64`,
  `routes/auth.py:196-201`. See SECURITY/TENANCY HOLES #3 and #4 for why that is fragile rather
  than wrong.
- Redis keys are tenant-prefixed by construction and `redis_key` **raises** on a falsy `org_id`
  (`libs/shared/cache.py:48-60`) — closing the leak path RLS cannot see, since no SQL is issued.

### Tests — real behaviour, not 200-OK smoke

`tests/conftest.py` runs against a real migrated Postgres and, critically, connects as the
**`mytheus_app` role** (`tests/conftest.py:59-73`), explicitly refusing to prove isolation with the
schema-owner credential — the docstring at `:14-19` names that as the exact flaw of the retired
approach. Only `verify_token` is dependency-overridden in route tests
(e.g. `tests/test_user_context.py:91-94`); RLS, SQL, audit rows and Redis are all real.

| File | LOC | Substance |
|---|---|---|
| `tests/test_user_context.py` | 476 | cache hit/miss, TTL ≤ 300s, cross-org JWT blocked (`:299`), unreachable Redis still correct (`:336`, `:373`), corrupt entry = miss (`:400`), gRPC and REST agree (`:432`) |
| `tests/test_clerk_webhook.py` | 339 | unsigned 401, **valid signature over tampered body rejected** (`:137`), redelivery no-op, dual-org membership, unknown org 401, reconciliation diff + idempotence |
| `tests/test_org_model_reconciliation.py` | 379 | member-may-edit-self / may-not-edit-other / admin-may (`:328-352`), cross-tenant PATCH is 404-not-403 (`:353`), audit row written (`:368`) |
| `tests/test_tenant_settings_api.py` | 305 | writes the exact key ingestion reads (`:150`), satisfies the reader's own predicate (`:165`), one tenant's write does not touch another's mirror (`:212`), member tier refused (`:230`), **failed mirror rolls the column back** (`:281`) |
| `tests/test_grpc.py` | 101 | org-id required, unknown user is NOT_FOUND not a synthesized context (`:44`), DB failure fails closed (`:82`) |
| `tests/test_role_topics_taxonomy.py` | 224 | version pinning, unknown version 404 |
| `tests/test_org_clerk_link.py` | 232 | provisioning idempotence, clerk link fills only a NULL |
| `tests/test_seed.py` | 255 | runs the real seeder end-to-end |
| `tests/test_rbac_grants.py`, `test_rls.py`, `test_roles_and_policies.py`, `test_org_resolver.py`, `test_auth.py` | — | tier/domain-role separation, one tier per user per tenant, tier CHECK constraint, `rbac_grants` tenant isolation, fail-closed with no GUC |

One test is misleading — see SECURITY/TENANCY HOLES #1 (`test_user_deleted_soft_deletes`).

---

## PARTIAL — works, with a named gap

- **`GET /tenant-settings` exposes one column of eight.** Only `pii_redaction_enabled`
  (`routes/tenant_settings.py:72`). The table also has `default_locale`,
  `retention_days_transcripts`, `retention_days_worklogs`, `agent_confidence_overrides`,
  `sso_config`, `consent_required` (`models.py:165-199`) — none has any read or write route
  anywhere in the repo. `consent_required` is excluded deliberately and the reason is stated
  (`routes/tenant_settings.py:35-38`); the other five are simply not surfaced.
- **`invalidate_user_context` is only half-wired.** Called from `PATCH /users`
  (`routes/users.py:125`) but not from any `team_members` write, because no route writes that
  table — acknowledged honestly at `context/service.py:186-196`. Team-membership changes stay
  stale for up to the 300s TTL.
- **Department resolution is arbitrary for multi-department users.** Derived transitively through
  teams, alphabetically first wins, no canon tiebreak (`context/service.py:50-61`). Documented,
  not hidden.
- **Reconciliation exists but nothing runs it.** `reconcile_tenant` (`reconciliation.py:47`) is
  correct and idempotent by construction (diff, not replay) and deliberately takes the roster as
  an argument so it is testable without Clerk (`reconciliation.py:11-13`). Its docstring says the
  Temporal schedule that supplies the roster is wired in `workflow-service` — I found **no such
  caller**. UNVERIFIED whether one exists outside this repo; within the repo it is unreachable.
- **`X-Forwarded-For` is trusted unconditionally** for the audit `ip`
  (`libs/shared/authz.py:85`). Safe only if the ingress always overwrites it; the docstring says
  so (`:82-84`). A directly-exposed service would record attacker-chosen provenance. Mitigated by
  the fact that an unparseable value becomes NULL rather than breaking the write (`:88-91`).
- **Taxonomy version bump does not invalidate warm caches** — up to 300s of old topics after a
  deploy. Explicitly decided rather than overlooked, with the two remediation levers named
  (`taxonomy/loader.py` module docstring).

---

## STUB/FAKE — exists but returns hardcoded/empty/NotImplemented

- **All five SCIM routes** — `routes/scim.py:31`, `:36`, `:41`, `:46`, `:52`. Each raises
  `NotImplementedProblem` → **501** with an RFC 7807 body
  (`libs/shared/errors.py:104-110`). Honest stubs, and correctly so: the module docstring
  (`routes/scim.py:22-27`) records that they previously raised `NotImplementedError` → 500, which
  causes SCIM clients to retry on a schedule. Router-level `Depends(verify_token)`
  (`routes/scim.py:17`) means a future route here cannot ship unauthenticated by accident.
- **Nothing else in the service is faked.** No hardcoded org_ids, no `return {}` placeholders, no
  `pass` handlers. The two historical fakes are genuinely gone and their removal is verifiable:
  the gRPC handler no longer synthesizes a `member` context on failure
  (`grpc_server.py:43-52` describes the removed defect; `:98-112` and `:138` are the replacement),
  and `org_id_from` no longer casts an opaque Clerk string to UUID
  (`libs/shared/auth.py:12-23` describes the removed defect; `:125-132` is the replacement).

---

## MISSING — referenced somewhere but absent

1. **`POST /orgs` / any org-lifecycle route.** `main.py:26` is a bare
   `# TODO: add routers for /orgs /rbac`. `provisioning.provision_organization`
   (`provisioning.py:65`) is a real, transaction-atomic implementation — org row plus the
   `tenant-events` `provisioned` outbox emission in one transaction
   (`provisioning.py:131-149`), idempotent on `slug`, with a `_LINK_CLERK_ORG` that only ever
   fills a NULL so a tenant cannot be repointed at another Clerk org (`provisioning.py:53-56`) —
   but its **only caller is the seeder** (`seed/seed.py`). The gateway routes `/orgs` here
   (`services/api_gateway/edge.py:163`), so it 404s;
   `mytheus-docs/design/openapi/api-gateway.yaml:166` claims it returns 501.
2. **`/rbac` — no router, no route, no writer. The largest functional hole.**
   Grepped the whole backend: the only `INSERT INTO rbac_grants` is `seed/seed.py:196`. The Clerk
   webhook creates a `users` row and **no grant** (`routes/auth.py:168-191`). Therefore a user
   onboarded through real Clerk has `rbac_tier == ""` (`libs/shared/authz.py:120`),
   `satisfies()` returns False for every tier (`:62-63`), and **every route in this service 403s
   for every genuine production user**. Only seeded users can call anything. The gateway routes
   `/rbac` here (`edge.py:164`) → 404.
3. **No `GET /users` roster endpoint.** Independently confirmed from outside this repo:
   `mytheus-mobile/src/api/hooks/index.ts:37` — "`identity-tenant-service` has no `GET /users`
   surface yet".
4. **No handler for Clerk `organization.*` or `organizationMembership.*` events.**
   `routes/auth.py:205` returns False for everything that is not `user.created` /
   `user.updated` / `user.deleted`, with the comment "organization events … are not this
   handler's business". Consequence: org creation, org deletion, org suspension, and a user
   joining or leaving an organization in Clerk never reach the database. Combined with #1 and #2,
   there is no automated path from "a customer signs up in Clerk" to "a usable tenant exists".
5. **No deployment definition for this service.** `docker-compose.yml` defines only
   postgres / redis / redpanda / temporal / temporal-ui — **no application services**.
   `Dockerfile:42` hardcodes `CMD ["uvicorn", "services.core_api.main:app", … "--port","8000"]`.
   `Makefile:72` runs identity on **port 8001**, while the gateway's default upstream is
   `http://identity-tenant-service:8000` (`services/api_gateway/proxy.py:64`, `:80`). No
   `UPSTREAM_IDENTITY_TENANT_SERVICE` is set anywhere outside `tests/test_gateway_proxy.py`.
   So the gateway → identity hop has no working configuration in any checked-in environment.
6. **`GET /orgs/{org_id}/departments|teams|members`** named in the plan are absent, but this is a
   *deliberate* omission with a stated reason (`routes/users.py:5-14`): those prefixes belong to
   `core-api`, and `{org_id}` in a path would violate the "org_id never in a path" invariant.
   Listed here for completeness, not as a defect.

---

## OUTBOUND DEPENDENCIES — each edge, and whether the far side exists

| # | What it calls | Exact dependency | Far side exists? |
|---|---|---|---|
| 1 | Postgres — own tables | `organizations`, `users`, `rbac_grants`, `tenant_settings`, `teams`, `team_members`, `departments`, `data_residency` (`models.py`) | Yes — DDL in `migrations/versions/82d44163c933…` and `b7d9f3a55c21…` |
| 2 | Postgres — core-api-owned, **read only** | `teams`, `team_members`, `departments` (`context/service.py:43-61`) | Yes — `services/core_api/routes/org_hierarchy.py` |
| 3 | Postgres — core-api-owned, **read only** | `meetings`, `meeting_participants` (`context/service.py:63-68`) | Yes — `services/core_api/routes/meetings.py` |
| 4 | Postgres — shared infra | `outbox` (`libs/shared/outbox.enqueue`), `inbox` (`libs/shared/inbox`), `audit_logs` (`libs/shared/audit.audit`) | Yes — `82d44163c933…`, `c3e8b1d47f92_0004_audit_logs_append_only.py` |
| 5 | Redis — own cache | `nexus:{org_id}:user_ctx:{user_id}`, TTL 300s (`context/service.py:37`, `:110`) | Yes |
| 6 | Redis — cross-repo contract | `nexus:{org_id}:tenant_settings:pii_redaction_enabled` (`routes/tenant_settings.py:63`, `:90-92`) | **Yes, consumer verified**: `mytheus-intelligence/services/ingestion_service/pii.py:13`, `:30` |
| 7 | Redis — webhook dedupe | `libs/shared/inbox.is_duplicate` (`routes/auth.py:120`) | Yes |
| 8 | Kafka topic `tenant-events` | via outbox, payload `{event_type: "provisioned", org_slug, plan_code}` (`provisioning.py:136-149`) | Topic/schema exist in `mytheus-events`; **no consumer found anywhere** — the emission is write-only today. UNVERIFIED outside the repos I searched |
| 9 | Clerk JWKS | `CLERK_JWKS_URL` (`libs/shared/auth.py:49`) | External, yes |
| 10 | Clerk Admin API | `libs/shared/clerk_admin.set_internal_org_id` (`provisioning.py:169`) — writes `public_metadata.internal_org_id`, which `mytheus-intelligence/libs/shared/auth.py:41-42` reads as a token claim | Yes, module exists; the consumer is verified |
| 11 | Svix | signature verification of the Clerk webhook (`routes/auth.py:90`) | External, yes |
| 12 | gRPC protobufs | `identity_pb2`, `identity_pb2_grpc` (`grpc_server.py:26-27`) | Yes — generated at `mytheus-backend/` root |
| 13 | Taxonomy artifact | `taxonomy/role_topics.v1.yaml` (`taxonomy/loader.py:124`) | Yes |

Boundary discipline is respected: this service never *writes* a core-api-owned table from a
request path. The one exception is the seeder writing `team_members`, and it is disclosed as an
exception with a citation trail rather than taken silently (`seed/seed.py:31-42`).

---

## INBOUND — who calls this

- **api-gateway** proxies eight prefixes to `identity-tenant-service`
  (`services/api_gateway/edge.py:163-170`): `/orgs`, `/rbac`, `/scim/v2`, `/context`, `/users`,
  `/taxonomy`, `/tenant-settings`, `/clerk/webhook`. `/clerk/webhook` is in `PUBLIC_PREFIXES`
  (`edge.py:51`) because it authenticates by HMAC, not bearer token. Two of the eight (`/orgs`,
  `/rbac`) have no router behind them. Proxy routing is longest-prefix (`edge.py:183-190`),
  target resolved by `upstream_base_url` (`proxy.py:68-80`).
- **mytheus-intelligence** → gRPC `GetUserContext` at `IDENTITY_GRPC_ADDR=localhost:50051`
  (`mytheus-intelligence/.env.example:43-44`).
- **mytheus-intelligence / ingestion-service** → reads the Redis PII mirror this service writes
  (`mytheus-intelligence/services/ingestion_service/pii.py:13`, `:30`). Cross-repo table joins
  are forbidden, so Redis is the delivery path.
- **mytheus-realtime / transcription-service** → reads `tenant_settings` and `data_residency`
  directly from the shared Postgres
  (`mytheus-realtime/…/tenancy.py:2`, `…/residency.py:3`), and defers `organizations` writes to
  this service (`…/devseed.py:38`, `:165`).
- **services/bff** → consumes `get_user_context` output (`tests/test_bff_people.py:3`).
- **mytheus-mobile** → expects a member roster that does not exist yet
  (`mytheus-mobile/src/api/hooks/index.ts:37`).
- **Clerk (via Svix)** → `POST /clerk/webhook`.
- **Contract artifacts**: `mytheus-docs/design/openapi/identity-tenant-service.yaml` and
  `mytheus-backend/design/openapi/identity-tenant-service.yaml` — the latter's `paths` cover
  exactly the 12 implemented operationIds (`clerk_webhook`, `read_user_context`, `healthz`,
  5 SCIM, `read_role_topics`, `read_tenant_settings`, `update_tenant_settings`,
  `update_user_profile`) and nothing more, so the spec does **not** over-promise `/orgs` or
  `/rbac`. The gateway spec does (`mytheus-docs/design/openapi/api-gateway.yaml:166-169`).

---

## SECURITY/TENANCY HOLES

1. **`user.deleted` is almost certainly dead in production, and its test hides that.**
   `_clerk_org_of` (`routes/auth.py:68-78`) reads the organization **only** from
   `data.organization_memberships`. Clerk's real `user.deleted` payload carries just
   `{id, deleted, object}` and no memberships, so the handler returns
   `{"status": "ignored"}` at `routes/auth.py:106-109` and the soft delete at `:193-203` never
   executes. The passing test manufactures a `user.deleted` payload **with** memberships, because
   it reuses the same `_user_event` builder as `user.created`
   (`tests/test_clerk_webhook.py:222`, builder at `:78-89`) — so the green test proves nothing
   about the real wire format. Consequence: an offboarded user's mirror row survives and they
   remain resolvable as a principal indefinitely; the hourly reconciliation sweep that would
   catch it (`reconciliation.py:93-101`) has no caller either.
   *Status: the code path is verified from source; Clerk's exact payload shape is from model
   knowledge, not from a live Clerk instance — that half is UNVERIFIED.*
2. **No RBAC grant issuance path** (MISSING #2). This fails **closed**, so it is an availability
   hole rather than a privilege-escalation hole — but it means the service cannot serve a single
   real user end-to-end without manual SQL.
3. **`reconciliation.py:64` and `:93-100`** — `SELECT clerk_user_id FROM users` and
   `UPDATE users SET deleted_at = now() WHERE clerk_user_id = :cuid`, both with **no `org_id`
   predicate**. Correct today only because `set_rls` runs at `reconciliation.py:61`. One refactor
   that moves these onto a session without the GUC turns them into a cross-tenant mass
   soft-delete. Defence-in-depth would add the redundant `AND org_id = :org`.
4. **`routes/auth.py:196-201`** — same shape: `UPDATE users … WHERE clerk_user_id = :cuid`,
   unscoped, safe only via `set_rls` at `:129`.
5. **The gRPC server has no transport security and no caller authentication.**
   `server.add_insecure_port(f"[::]:{port}")` (`grpc_server.py:152`), and `org-id` is read from
   plain metadata with no signature or token (`grpc_server.py:54-55`). Any process that can reach
   :50051 can assert any `org_id` and read any tenant's user context, role, skills, bio, team and
   RBAC tier. Contrast the REST edge, which HMAC-signs its forwarded org/user headers precisely
   so a forged value is detectable (`services/api_gateway/edge.py:69-81`, rationale `:8-13`).
   The gRPC path has no equivalent control. **This is the most serious live exposure.**
6. **`verify_token` does not check `aud` or `azp`** (`libs/shared/auth.py:94-103`). A token minted
   for a different Clerk application under the same issuer would pass every check. Low severity
   for a single-application deployment, real if Clerk instance sharing ever happens.
7. **`GET /taxonomy/role-topics` discards its context** (`routes/taxonomy.py:36`, `del ctx`).
   This is correct — the response carries no tenant data and the dependency is there purely to
   require authentication — noted only so it is not mistaken for a forgotten check.
8. **Residual disclosed window on `PATCH /tenant-settings`**: if the Redis mirror write succeeds
   and the COMMIT then fails, on *disable* the mirror is OFF while the column says ON, i.e.
   under-redaction until the next successful write. Disclosed in-source rather than papered over
   (`routes/tenant_settings.py:171-178`), with the correct fix named (deliver via `tenant-events`).

---

## CONFIG/ENV REQUIRED

| Variable | Used at | Failure if absent |
|---|---|---|
| `CLERK_JWKS_URL` | `libs/shared/auth.py:49` | `KeyError` → 500 on first authenticated request |
| `CLERK_ISSUER` | `libs/shared/auth.py:53` | `KeyError` → 500 |
| `CLERK_JWT_ORG_CLAIM` | `libs/shared/auth.py:57` | defaults to `"org_id"` |
| `CLERK_WEBHOOK_SECRET` | `routes/auth.py:50` | clean 503 `DEPENDENCY_UNAVAILABLE` |
| `CLERK_SECRET_KEY` | `libs/shared/clerk_admin` (via `provisioning.py:169`) | `sync_internal_org_id_to_clerk` returns False and logs |
| `DATABASE_URL` | `libs/shared/db` | **must** name `mytheus_app`; pointing it at `postgres` silently disables all tenant isolation (`.env.example` warns explicitly) |
| `ADMIN_DATABASE_URL` | seeder only (`seed/seed.py`) | seeder cannot run |
| `REDIS_URL` | `cache.py:41`, `app_factory.py:22`, `routes/auth.py:115` | **inconsistency**: `.env.example` uses port **6380**, but all three code defaults are **6379**. Unset env → silently wrong Redis |
| `ROLE_TOPICS_TAXONOMY_VERSION` | `taxonomy/loader.py:112` | defaults to `"v1"` |
| `GATEWAY_SIGNING_KEY` | `services/api_gateway/edge.py:59-66` | gateway fails closed with 503 (correct — no default key) |
| `UPSTREAM_IDENTITY_TENANT_SERVICE` | `services/api_gateway/proxy.py:77` | falls back to `http://identity-tenant-service:8000`, which matches nothing that is checked in |
| `OTEL_*` | `libs/shared/telemetry` | telemetry is a no-op |

Ports: HTTP **8001** per `Makefile:72`; gRPC **50051** per `grpc_server.py:145` and
`mytheus-intelligence/.env.example:44`.

---

## VERDICT

The identity half of this service is genuinely built and unusually well defended: JWT signature
verification is real with no decode-without-verify anywhere, RLS is real and enforced by a role
that provably lacks BYPASSRLS, the Svix webhook has correct verify-then-dedupe-then-apply
ordering with a durable inbox, and the test suite runs against a live database as the same
least-privilege role production uses — so the isolation claims are backed by evidence rather than
by comments. The tenancy half is where it stops: there is no route to create an organization and
no route or webhook path to issue an RBAC grant, so a tenant onboarded through real Clerk ends up
with users holding no tier who are 403'd out of every endpoint this service serves, leaving the
whole thing reachable today only via the seeder. Add the likely-dead `user.deleted` branch whose
test uses a fabricated payload, the unauthenticated plaintext gRPC port on 50051, the unhandled
`organization.*` / `organizationMembership.*` events, and the complete absence of a deployment
definition (compose carries infra only, and the gateway's default upstream port disagrees with the
Makefile), and the honest read is roughly **60% complete**: a solid, correct core with the entire
tenant-lifecycle and role-assignment surface missing.
