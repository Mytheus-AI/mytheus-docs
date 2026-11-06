# 02 Plan 1-2 Foundation and Tenancy

| Field | Value |
|---|---|
| Band | Plan 1-2, phases P1 (Architecture and Design, W1-2) and P2 (Foundation, W3-5) |
| Plan file | [plan/plan-1-2-foundation-and-tenancy.md](../plan/plan-1-2-foundation-and-tenancy.md) |
| Status | **IN PROGRESS. P1 COMPLETE. P2 BACKEND COMPLETE, CLIENTS NOT STARTED.** |
| Complete | `mytheus-docs` canon plus the P1 design pack; `mytheus-backend` sections 3.2, 3.2b, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9, 3.11; `mytheus-events` schema fixes plus CI; `mytheus-platform` role-split deploy |
| Not started | Section 3.10 (`mytheus-ui`), section 7 (`mytheus-mobile`) |
| Last verified | 2026-07-30, `make ci` green against a database rebuilt from an empty volume: **114 passed, 1 skipped** |

Sections 1 to 6 below record the P1 session of 2026-07-27. Sections 7 to 10 record the two foundation sessions. Sections 11 onward record the route and CI session. **All seven blocking defects D1 to D7 are closed, and the backend half of P2 is done: every planned service now has its `/api/v1` surface.**

---

## 1. What landed, per repo

### 1.1 `mytheus-docs`: canon amendments (step P1.0, net-new)

This step is not in the plan. It was added because three names the plan instructs P2 to build did not exist in `_canon/`, and the canon-first contract makes building them first a bug by construction.

**`_canon/02-data-model.md`**

- `organizations` gains `clerk_org_id` (the unique immutable Clerk-organization mapping) and `status` (`active` or `suspended`). Its row is also marked explicitly as **not** a tenant-scoped row and **not** `tenant_isolation`-policied.
- **`rbac_grants` added** to section 2.2: `id`, `org_id`, `user_id`, `tier` (`owner`/`admin`/`member`), `granted_by`, `created_at`, unique on `(org_id, user_id)`. It is the authorization store and the only thing the authz middleware reads.
- **New section 7 rule 7**, the tenant-root exemption: `organizations` carries no `org_id` column, so the canonical policy is not expressible on it. This makes rule 1 exceptionless and therefore machine-checkable: every table with an `org_id` column must carry a `tenant_isolation` policy, asserted in CI.

**`adr/0002-multi-tenancy-model.md`**, three amendments to D3

- The hardened predicate `NULLIF(current_setting('app.current_org', true), '')::uuid` is recorded. Plan 0 built it, it is stricter than the ADR's literal SQL, and it had never been written down.
- The `FORCE` mandate is scoped to tables that actually carry `org_id`, with the tenant-root carve-out and its full reasoning.
- The three database roles are named: `mytheus_app`, `mytheus_relay`, `mytheus_admin`. None holds `BYPASSRLS`. The relay reaches across tenants on `outbox` alone, through one permissive policy on that one table.

**`adr/0010-auth-clerk-enterprise-sso.md`**

- D2 amendment: the token carries Clerk's native organization id; the resolution through `organizations.clerk_org_id` **is** the authorization checkpoint. Four requirements follow: claim-absent returns `401` and never `500`; the claim is opaque and must not be parsed as a UUID; **cache the mapping, never the verdict**; all three negative outcomes return one byte-identical response.
- D5 now names `rbac_grants.tier` explicitly rather than describing an unnamed store.

**Propagated** to `06-multi-tenancy.md`, `11-security-architecture.md`, `24-glossary-and-naming.md` (three new concepts, one new entity, `organizations` row updated), and `coverage-traceability-matrix.md` (four new rows).

### 1.2 `mytheus-docs`: the P1 design pack (section 3.1, steps 1 to 10)

Seventeen files. New folders `design/` and `threat-model/`.

| File | Plan step | Content |
|---|---|---|
| `design/README.md` | index | Index, the six plan deviations, and the eight open items |
| `design/c4-l1-system-context.md` | 3.1.1 | Four trust zones, subprocessor zone, per-boundary controls, what the band actually builds |
| `design/c4-l2-container.md` | 3.1.1 | Containers by scaling family, the edge path, the MVP co-location note |
| `design/erd.md` | 3.1.2 | The whole target model in two Mermaid ERDs, the seven partition candidates, and the `meetings` deferral with its full reasoning |
| `design/avro/README.md` | 3.1.3 | Topic index pointing at `mytheus-events`, two unsatisfiable schema fields, and one free enum hardening |
| `design/openapi/core-api.yaml` | 3.1.4 | Meetings CRUD, teams, departments. Five paths. |
| `design/openapi/identity-tenant-service.yaml` | 3.1.4 | Context, Clerk webhook, orgs (deferred), SCIM (deferred). Seven paths. |
| `design/openapi/bff.yaml` | 3.1.4 | The dashboard composite with per-arm status |
| `design/openapi/admin-ops-service.yaml` | 3.1.4 | `audit_logs` read, with the tamper-evidence claim scoped honestly |
| `design/openapi/api-gateway.yaml` | 3.1.4 | The aggregate edge contract, the frozen error enum, the routing map |
| `design/openapi/.spectral.yaml` | 3.1.4 | Thirteen rules across tenant isolation, auth, RFC 7807, pagination, idempotency, house style |
| `design/wireframes/README.md` | 3.1.5 | Layout and data contracts for sign-in, the app shell, the dashboard |
| `design/tech-lock.md` | 3.1.6 | The locked stack corrected to ADR-0016, two deferrals with exit triggers, the sign-off table |
| `threat-model/stride.md` | 3.1.7 | Six surfaces by six STRIDE categories; Auth and APIs complete, four stubbed |
| `design/data-residency.md` | 3.1.8 | Region set, MVP default `us`, and an honest account of what single-region means |
| `design/subprocessors.md` | 3.1.9 | Thirteen processors classified by data tier, with per-processor owner and status |
| `design/rtbf-contract.md` | 3.1.10 | The cross-store deletion contract and three traps the band must design around |

### 1.3 Every other repo

Untouched. `mytheus-backend`, `mytheus-ui`, `mytheus-mobile`, `mytheus-platform`, and `mytheus-events` are exactly as Plan 0 left them.

---

## 2. Definition of Done audit

Against section 10 of [plan-1-2-foundation-and-tenancy.md](../plan/plan-1-2-foundation-and-tenancy.md).

### P1 (W1-W2)

| DoD line | Verdict | Evidence or reason |
|---|---|---|
| All section 3.1 artifacts exist | **MET** | Seventeen files listed in section 1.2. |
| Artifacts are PR-reviewed | **NOT MET** | Uncommitted. No PR raised. |
| `design/tech-lock.md` is signed off (initials present) | **NOT MET** | The table exists and is deliberately empty. Marked `UNSIGNED`, and section 3.3 is gated on it. |
| OpenAPI specs pass Spectral | **MET 2026-08-28**, against emitted specs rather than these. `spectral lint` ran for the first time and the ruleset **did not parse**: two rules put `field` beside `given` instead of inside `then`, and three more used PCRE `(?i)` against a JavaScript engine. Repaired, and all thirteen rules now pass with zero errors over the four emitted documents. See [ADR-0022](../adr/0022-openapi-emitted-from-code.md) and section 17. |
| Avro drafts validate BACKWARD | **MET, differently** | No drafts were produced. `mytheus-events` already ships all four band schemas, registered under a BACKWARD gate. The check is redirected there and recorded as a deviation. |
| STRIDE model authored | **MET** | Auth and APIs rows complete; the other four stubbed to canon controls, which is what the plan asks for. |
| Residency region chosen | **MET** | `us`, single-region, with the table populated per tenant regardless. |
| DPAs signed or pending-with-owner | **MET** | Thirteen processors, each with an owner and a status. Clerk flagged as the blocking one. |
| RTBF contract frozen | **MET** | Plus three structural traps found while writing it. |
| Both section 8 decisions surfaced | **MET** | Both resolved before this session; recorded in canon and both ADRs this session. |

**P1 verdict: substantively complete. Three administrative lines remain: PR review, the tech-lock signature, and an actual Spectral run.**

### P2 (W3-W5)

As of this P1 session: **NOT STARTED**. Every line of the plan's P2 DoD was outstanding.

**Superseded by section 11**, which audits the same checklist after the three build sessions. Every line except `web-app` and `mytheus-mobile` is now MET. This entry is left in place rather than edited so the record shows the band's actual progression.

---

## 3. Deviations from the plan, and why

| Deviation | Rationale |
|---|---|
| **A net-new step P1.0 was inserted before the design pack.** | `rbac_grants`, `organizations.clerk_org_id`, and `organizations.status` are all instructed by the plan and none existed in `_canon/`. The canon-first contract makes a migration that introduces an uncanonical name wrong by construction, so canon had to move first. |
| **`design/avro/` holds a reference page and zero `.avsc` files.** | Plan section 3.1 step 3 was written expecting Plan 0 to leave drafts; Plan 0 produced 21 registered schemas under a BACKWARD gate instead. A second copy in `mytheus-docs`, which has no build and no Schema Registry, could never be validated and so could only drift. The acceptance check is redirected to `make check-compatibility` in `mytheus-events`, which is a stronger check. |
| **Wireframes are annotated contracts, not proposals.** | `mytheus-ui` already ships all three screens on a documented design system, with its own `docs/PAGES.md` mapping each route to its future `bff` endpoint. Speculative wireframes would have created a second design authority that the built UI immediately contradicts. |
| **`design/tech-lock.md` locks self-hosted stores, not Supabase, PgBouncer, and Upstash.** | [ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md) supersedes the plan's step 6 text, which predates it. Following the plan literally would have produced a signed lock contradicting the newest ADR. |
| **PgBouncer explicitly deferred**, with an exit trigger and a four-step landing checklist. | One node, one web process. There is no connection fan-in to absorb. The one thing that would break under transaction pooling is already correct: `set_config(..., true)` is transaction-local. The asyncpg prepared-statement trap is documented so the later addition is not a discovery. |
| **shadcn/ui not adopted**, contrary to section 3.10 step 1. | `mytheus-ui` has its own component layer across 56 pages. A second primitive set forks the visual language for no gain. |
| **`meetings` will not be range-partitioned in this band.** | Canon says "range-partition candidate at scale"; only the reader docs state it as done, and canon wins. More decisively: PostgreSQL requires a unique constraint on a partitioned table to include every partition-key column, so partitioning by `scheduled_start` makes `UNIQUE (org_id, id)` impossible, and with it the composite foreign keys that make a cross-tenant dangling reference unrepresentable. Exit trigger recorded in `design/erd.md`. |
| **`audit_logs` gets twelve monthly partitions plus a `DEFAULT` partition**, not the plan's single partition. | The plan creates one partition and defers `pg_partman`. The band ends in W5; the partition expires at the end of the month. An insert with no matching range raises, and section 3.7 makes audited operations fail closed, so a missed maintenance window would stop meeting creation. Twelve months plus a catch-all removes the failure class. |
| **`GET /api/v1/orgs/{id}/teams` specified as `GET /api/v1/teams`.** | The plan's own form puts a tenant identifier in the path, violating the `org_id`-never-in-path invariant the same plan states. |
| **Tenant provisioning (`POST /api/v1/orgs`) deferred**, returning `501`. | Decided by the maintainer. See section 5 item 7 for the design hole that makes it non-trivial. An operator seed script running as `mytheus_admin` stands in, and also writes the `tenant-events` outbox row so the relay path is still exercised end to end. |
| **The `organizations` `tenant_isolation` policy will be REMOVED in P2.** | The largest single decision of the session. See section 4. |

---

## 4. The `organizations` RLS decision

Recorded at length because it reverses something Plan 0 built, and a reversal with no written reason is a bug rather than a decision.

**What Plan 0 built.** Migration `82d44163c933` applies `ENABLE` plus `FORCE ROW LEVEL SECURITY` and a `tenant_isolation` policy to `organizations`, keyed on its own `id`.

**The problem.** The resolved tenant-claim decision requires `api-gateway` to run `SELECT id FROM organizations WHERE clerk_org_id = $1` **before** `org_id` is known. Under that policy, with the GUC unset, the query returns zero rows. Always. The authorization checkpoint the decision is built around has no executable path.

**What was considered.** Two agents reviewed this adversarially from opposed starting positions. Five options were evaluated: a `SECURITY DEFINER` resolver function, removing the policy, a separate mapping table, resolving from Redis authoritatively, and a bootstrap role with column grants.

**Why the resolver function was rejected.** A `SECURITY DEFINER` function executes as its owner, and a table owner normally bypasses RLS. But `FORCE ROW LEVEL SECURITY` removes exactly that exemption. So the function is subject to the policy it was written to get past and returns nothing. Making it work requires an additional permissive policy, which means six new database objects including a policy that partly undoes the point. The agent defending this option identified the flaw itself.

**Why removing the policy is right.**

1. **Canon never required it.** ADR-0002 D3 says "Every **tenant-scoped** table enables and forces RLS." `organizations` has no `org_id` column: its `id` **is** the `org_id`. `_canon/02-data-model.md` section 7's canonical policy is keyed on `org_id` and is literally inapplicable. Canon types `organizations` as "Is the tenant root", not "Yes", unlike every other row. The policy was an implementation-time invention.
2. **It makes the real invariant machine-checkable.** The rule worth enforcing is "every table with an `org_id` column carries `tenant_isolation`". That cannot be stated exceptionlessly while a table with no `org_id` carries one anyway. The exception would have to be hardcoded into the CI assertion, and hardcoded exceptions rot.
3. **It is not on the cross-tenant leak path.** Every tenant table's policy compares its own column to the GUC, never to a value read from another query. Learning another org's UUID buys nothing, because the GUC is set only from the verified token. `organizations` holds directory metadata; tenant-confidential state lives in `tenant_settings` and `data_residency`, which carry `org_id` and stay policied.
4. **Three callers need the pre-tenant read, not one.** The gateway, the Clerk webhook, and the hourly reconciliation job the plan itself specifies. The defending agent's own stated flip condition was "more than one caller", and it is met.

**The risk accepted.** A forgotten `WHERE` on `organizations` would expose the customer directory: names and slugs, not tenant content, with no pivot to tenant data.

**The compensating controls, all required in P2.** Column-level grants rather than table-level; exactly one sanctioned query shape in `libs/shared/`; a CI assertion for the `org_id`-implies-policy rule; a second CI assertion that no application role holds `rolsuper` or `rolbypassrls`; and a `COMMENT ON TABLE` recording the reasoning at the point of use.

Recorded in `_canon/02-data-model.md` section 7 rule 7 and ADR-0002 D3.

---

## 5. Open gaps

Ordered by what blocks the most. Items 1 to 5 are defects in code that already exists, found by reading the source, and none appear in the plan. **They are the first work of P2, ahead of any feature.**

| # | Gap | Blocks | Picked up by |
|---|---|---|---|
| 1 | **The application connects to PostgreSQL as the superuser** in every environment. All four `env/*.env.example` files and `mytheus-backend/.env.example` use `postgres:`, and `compose.prod.yml`'s `x-backend` anchor shares one `env_file` across `migrate`, `web`, `relay`, and `workflow`. Superusers bypass RLS unconditionally. | Every isolation guarantee in the program. Every acceptance check that claims to prove isolation proves it only for `app_rls`, a role the application never uses. | Section 3.3, first |
| 2 | **The idempotency cache is not tenant-namespaced.** `libs/shared/idempotency.py` keys on the client-supplied header alone and caches the full response body for 24 hours, on all nine services. | A live cross-tenant leak of response bodies that never touches the database, so RLS cannot mitigate it. Also violates the frozen `nexus:{org_id}:` prefix. | Section 3.3 |
| 3 | **The identity gRPC server fails open.** `grpc_server.py` catches every exception and returns a well-formed context echoing the caller's asserted `org_id` with a default tier. | Becomes "database unreachable therefore grant `member`" the moment section 3.4 makes it read `rbac_grants`. Two lines to fix now; a security incident to fix later. | Section 3.4, before `rbac_grants` |
| 4 | **The outbox relay will drain nothing, silently**, once correct roles are wired. It selects from `outbox` with no `app.current_org` set, so the fail-closed policy returns zero rows, which `run()` reads as an empty queue and sleeps. | Total event loss with no error, no log, and no metric distinguishing it from idle. Same root cause as gap 1. | Section 3.3, with the roles |
| 5 | **One poison outbox row crash-loops the relay forever.** No per-row error handling, no `try` in `run()`, and `attempts` exists but is never written. No produce-side DLQ. | Head-of-line block across every tenant. | Section 3.5 |
| 6 | **Seven tenant tables have no foreign key to the tenant root.** `TenantMixin` adds the column with an index and no `ForeignKey`. | Orphan `org_id` values are insertable, so RTBF-by-`org_id` cannot be trusted to find everything. | Section 3.3 |
| 7 | **Tenant provisioning has no reachable path.** The gateway has two states, public or org-required. Public means anyone creates tenants; org-required means nobody can. A third state, authenticated-but-org-exempt, is missing. | `POST /api/v1/orgs`. Deferred by decision; an operator seed script stands in. | Deferred |
| 8 | **`libs/shared/auth.py` cannot implement the resolved decision.** `org_id_from` calls `UUID(claims["org_id"])` on a claim that is an opaque Clerk string, raising `ValueError` into a `500`. The claim name is hardcoded rather than read from `CLERK_JWT_ORG_CLAIM`, `nbf` is not required, and a JWKS outage returns `500` because `PyJWKClientError` does not derive from `InvalidTokenError`. | Section 3.8 entirely. | Section 3.8 |
| 9 | **RFC 7807 is not universal.** `errors.py` handles only `ProblemDetail` and `404`. Every `HTTPException` returns FastAPI's default shape. There is no `code` enum in code. `scim.py` raises `NotImplementedError`, which is a `500`, not the `501` the resolved decision requires. | The Spectral gate and two acceptance checks. | Section 3.3 |
| 10 | **No repo except `mytheus-platform` has a `.github/` directory.** No CI runs anywhere. `mytheus-events` has `make ci` and `make check-compatibility` fully defined with nothing invoking them. | Section 3.11 in full, and the mitigation that section 9 names for the worst bug class in the product. | Section 3.11 |
| 11 | **The dev stack and the test stack disagree on ports.** `docker-compose.yml` publishes 5432 and 6379; `tests/conftest.py` expects 5433 and 6380, so every database test skips silently and green. `temporal-ui` also collides with `run-gateway` on 8080. | Any honest verification of anything. | Section 3.2b, a net-new step |
| 12 | **Two `.avsc` fields are unsatisfiable.** `MeetingEvent.livekit_room` and `TenantEvent.plan_code` are non-nullable strings with no source. Free to fix now; permanent once a message is written. | Section 3.5 and 3.6 producers. | Before topic registration |
| 13 | **Eight tables are thin shells** missing canonical columns: `meetings` lacks `scheduled_start`, `scheduled_end`, `livekit_room`, `created_by`; `tenant_settings` has a generic `settings jsonb` instead of five named columns; `data_residency` has only `region`; `users.clerk_user_id` is globally unique rather than unique per `(org_id, clerk_user_id)`; `teams` lacks `department_id`; `team_members` lacks `role`; `meeting_participants` lacks `role_in_meeting`. | Sections 3.4 and 3.6, which are real migration work rather than the small additions the plan implies. | Sections 3.4, 3.6 |
| 14 | **`design/tech-lock.md` is unsigned.** | Section 3.3 by its own gate. | The team |
| 15 | **Clerk's DPA is unexecuted** and Clerk is the only processor receiving real data in this band. | Nothing technically. It is a compliance exposure the moment a non-team user signs in. | Legal |

---

## 6. Verification

**Run and observed in this session:**

| Command | Result |
|---|---|
| `git diff --stat` | 7 files changed, 47 insertions, 6 deletions across canon, both ADRs, two reader docs, the glossary, and the matrix |
| `yaml.safe_load` over all five OpenAPI files plus the Spectral ruleset | All six parse. Path counts: `core-api` 5, `identity-tenant-service` 7, `bff` 2, `admin-ops-service` 2, `api-gateway` 1; Spectral 13 rules |
| `grep` for em-dashes and en-dashes across `design/` and `threat-model/` | Zero hits |
| Mermaid fence balance across the three diagram files | Balanced |
| `git status --short` | `design/` and `threat-model/` untracked; nothing committed |

**Verified by file inspection, correcting the prior record:** every claim in section 5 gaps 1 to 13 was confirmed by reading the named source file, not inherited from a session log.

**NOT run, and not claimed:**

- `spectral lint`. The CLI is a Node dependency; no repo has CI.
- `make ci` in `mytheus-backend`. Docker is not running on this machine and there is no `.venv`. The Plan 0 record's "green" is **reported, not observed**, and given gap 11 the database tests were almost certainly skipping rather than passing.
- Any migration, any test, any build.

---

## 7. P2 session 1, 2026-07-29: the standing gate and three fail-open defects

This session did not build a feature. It made the gate mean something, because until it did, nothing else could be honestly claimed.

**The environment (step 3.2b, net-new).** `make ci` had never been green. It failed lint on two `I001` errors, could not run at all on a clean clone because `ci` never invoked `proto` and pytest failed at collection on the gitignored gRPC stubs, and six database tests were skipping silently: RLS isolation, both outbox/inbox idempotency tests, both relay tests, and the gRPC fail-closed test. Plan 0 recorded all six as passing. **None of them had ever run.**

- Compose now publishes 5433/6380 to match `tests/conftest.py`; `temporal-ui` moves off 8080, where it collided with `make run-gateway`.
- `supabase/postgres` ships `postgres` pre-created, so its entrypoint never applies `POSTGRES_PASSWORD` to that role and TCP auth failed. Fixed through `/etc/postgresql.schema.sql`, the image's own post-bootstrap hook. Do NOT mount over `docker-entrypoint-initdb.d`: that shadows the image bootstrap and the container dies before the role exists.
- Python pinned to 3.12.13. Plain `3.12` resolves to 3.12.5, which `black` refuses to run its AST safety check against.

**Defects closed.** D3 (`GetUserContext` answered a malformed id, an unknown user and an unreachable database with one well-formed context echoing the caller's own asserted `org_id` at tier `member`; each now returns `INVALID_ARGUMENT`, `NOT_FOUND` and `UNAVAILABLE`, and not-found stays indistinguishable from hidden-by-RLS because existence is tenant-confidential). D5 (one unserializable outbox row aborted the batch, killed the process and crash-looped on restart, blocking every tenant; now per-row isolation, `attempts` actually written, and the raw payload routed to `{topic}.dlq` after `MAX_ATTEMPTS`). Gap 9 (RFC 7807 was not universal: no handler emitted the required `code` field and every bare `HTTPException`, which is the entire auth surface, returned FastAPI's default shape).

**`mytheus-events`.** `MeetingEvent.livekit_room` was a non-nullable string with no source: `meetings` had no such column and LiveKit rooms are not minted until the real-time band. Now `["null","string"]` defaulting to null. Both event-type enums gained Avro defaults so a lagging consumer degrades on an unknown symbol instead of throwing. BACKWARD compatibility was **proved rather than assumed**: the old schema was registered under a throwaway subject and the new one checked against it, because a plain `check-compatibility` run passes trivially when there is no prior version to compare against.

**Verified:** ruff clean, black 61 files, mypy 50 files, pytest 22 passed (was 5 passed / 6 skipped).

---

## 8. P2 session 2, 2026-07-30: tenancy foundation (sections 3.3, 3.4, 3.7)

The four remaining defects are closed and the schema half of the band is built. **Still no HTTP routes**: every service continues to boot `/healthz` only, and sections 3.5, 3.6, 3.8, 3.9, 3.10 and 3.11 are untouched.

### 8.1 Migration `a1c4e7f20b18`: least-privilege roles and the tenant-root carve-out (section 3.3)

This is the migration that makes every isolation claim in the program mean something.

| What | Why |
|---|---|
| Three login roles, `mytheus_app` / `mytheus_relay` / `mytheus_admin`, all `NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE` | **Closes D1.** The application connected as `postgres`, which holds `rolbypassrls` in `supabase/postgres`, so the eleven `tenant_isolation` policies from `82d44163c933` were decorative on the only path that mattered. |
| `organizations` `tenant_isolation` policy REMOVED, `NO FORCE`, RLS disabled, plus a `COMMENT ON TABLE` recording the reasoning at the point of use | The decision in section 4 of this file, implemented. The policy made the pre-tenant `clerk_org_id` lookup return zero rows forever, so the resolved tenant-claim decision had no executable path. |
| Column-level `SELECT (id, clerk_org_id, name, slug, status, deleted_at)` for `mytheus_app`, no table-level grant, no write | Compensating control 1. `SELECT *` on the tenant root is now refused by the database, which bounds the accepted risk of a forgotten `WHERE` and makes the failure loud. |
| `organizations.clerk_org_id text UNIQUE` and `organizations.status` with a CHECK | Required by the resolved tenant-claim decision (ADR-0010 D2). |
| `org_id` foreign keys to `organizations(id)` `ON DELETE RESTRICT` on the seven tables that lacked one | **Closes D6.** RESTRICT rather than CASCADE because tenant deletion is the RTBF saga's job, ordered and audited, not a side effect of one DELETE. |
| One permissive `outbox_relay_drain` policy on `outbox`, for `mytheus_relay` alone | **Closes D4.** Not a BYPASSRLS role: the cross-tenant reach stays visible in `pg_policies` and cannot extend to any other table. |
| `ALTER DEFAULT PRIVILEGES` so later tables inherit the app grants | Without it every new table is silently unreadable until someone remembers a GRANT, and the symptom is a production permission error rather than a failing test. |

**Passwords are deliberately absent from the migration.** The roles are created without one and cannot authenticate until `scripts/bootstrap_roles.py` (net-new) sets them from the environment. A password in a migration is a password in git history and in every clone of it. `make dev` now runs `make roles` after `alembic upgrade head`, and the script re-asserts that none of the three roles holds `rolsuper` or `rolbypassrls` before it exits successfully.

**Orphan cleanup, recorded because it deletes data.** The FK could not be added: orphan rows already existed (the Plan 0 tests seeded `outbox` rows with random `org_id` values). The migration deletes them, per table, with a `RAISE NOTICE` carrying the count. They are safe to delete and impossible to keep: a row whose `org_id` names a tenant root that does not exist is unreachable by every code path, undeliverable, and invisible to an RTBF sweep. That is the defect, not a consequence of fixing it. Safe now because no environment holds real tenant data; if this is ever re-run against one that does, the NOTICE count is the number of rows that were **already** unrecoverable and should be investigated before the deploy proceeds.

### 8.2 Migration `b7d9f3a55c21`: `rbac_grants` and the canonical columns (section 3.4, gap 13)

`rbac_grants` (`tier` in `owner`/`admin`/`member`, `UNIQUE (org_id, user_id)`, RLS-policied) is the authorization store, separate from `users.role` so that editing a profile can never change what someone may do. `GetUserContext` now reads the real tier through a LEFT JOIN and returns `""` for a user holding no grant; the hardcoded `_DEFAULT_TIER = "member"` is deleted. A LEFT JOIN rather than an inner one because "invited but not yet authorized" is a real state that must not be reported as "no such user".

Gap 13's eight thin shells are filled in: `meetings` gains `scheduled_start`, `scheduled_end`, `livekit_room` (nullable, for the same reason the Avro field had to become a union), `created_by` and two CHECK constraints; `tenant_settings` trades its generic `settings jsonb` for five named compliance columns plus `sso_config`; `data_residency` gains `db_cell`, `qdrant_region`, `object_bucket_region`, `locked_at` and a region CHECK; `teams` gains `department_id`; `team_members` and `meeting_participants` gain their role columns.

**`users.clerk_user_id` moves from globally unique to `UNIQUE (org_id, clerk_user_id)`.** The global constraint made one Clerk user a member of exactly one Mytheus organization for all time. Clerk does not work that way, so the second membership would have arrived at the webhook handler as a constraint violation rather than as a row.

### 8.3 Migration `c3e8b1d47f92`: `audit_logs` (section 3.7)

Partitioned by `RANGE (occurred_at)`, RLS-policied, append-only by GRANT rather than by trigger: a trigger can be dropped by whoever can write the row, a missing GRANT cannot be recovered by the role that lacks it. Twelve monthly partitions plus a `DEFAULT` catch-all, as deviated in section 3.

`libs/shared/audit.py` writes inside the **caller's** transaction, so the audit row and the change it describes commit together or not at all. It has no `best_effort` escape hatch, on purpose.

**A real hole the new CI assertion caught.** A partition is a table in its own right: privileges are checked against the relation named in the query, and the parent's RLS policy is only enforced on access through the parent. Section 8.1's `ALTER DEFAULT PRIVILEGES` grants the full DML set on every newly created table, and partitions are newly created tables. So `DELETE FROM audit_logs_2026_07` succeeded for `mytheus_app` while `DELETE FROM audit_logs` was correctly refused: an append-only audit trail erasable one partition at a time, with no tenant isolation on the way. The migration now revokes all privileges on every partition, and `test_no_application_role_can_reach_a_partition_directly` asserts it so the next partition cannot reintroduce it.

### 8.4 `libs/shared`: the request-path fixes

- **`idempotency.py` (closes D2).** Keyed on the client-supplied header alone, so two tenants sending the same `Idempotency-Key` shared one cache entry and the second received the first one's full response body, out of Redis, without a query being issued. Now `nexus:{org_id}:idem:{key}`, with the tenant resolved by an injected resolver; a request whose tenant cannot be established is passed through **uncached** rather than served from a namespace it does not own.
- **`auth.py` (closes gap 8).** `org_id_from` called `UUID()` on Clerk's opaque `org_...` string, so every real token raised into a 500. Replaced by `clerk_org_id_from`, which returns it verbatim. The claim name now comes from `CLERK_JWT_ORG_CLAIM`, `nbf` is required, and a JWKS outage returns 503 rather than 500 (`PyJWKClientError` does not derive from `InvalidTokenError`, so it escaped the handler meant to catch it). Failure messages no longer echo `str(exc)`, which was a free oracle for which check failed.
- **`org_resolver.py` (net-new).** Compensating control 2: the single sanctioned query shape against `organizations`, always with an equality predicate on `clerk_org_id`. All three negative outcomes (unknown, suspended, soft-deleted) return one byte-identical 401, so the endpoint is not an oracle for which Clerk organizations are customers.
- **`db.py`.** `get_engine()` / `get_sessionmaker()`, so one pool is shared rather than one per caller.

### 8.5 `mytheus-platform`: the second half of D1

Splitting the database roles achieves nothing if every process can still read every credential, and `compose.prod.yml`'s `x-backend` anchor handed one `env_file` to `migrate`, `web`, `relay` and `workflow` alike. The database credential now arrives from a per-role file layered on top of the shared one: `web` and `workflow` get `mytheus_app`, `relay` gets `mytheus_relay`, and only the one-shot `migrate` holds the schema-owner credential and the three role passwords. `env/prod.env.example` now carries no database credential at all. The `migrate` step runs `alembic upgrade head && python -m scripts.bootstrap_roles`, because until the second has run no application process can authenticate.

Verified with `docker compose config`: `web` and `workflow` resolve `DATABASE_URL` to `mytheus_app`, `relay` to `mytheus_relay`, `migrate` to the owner, and no long-running service sees `MYTHEUS_*_PASSWORD`.

The blocking DO-NOT-DEPLOY warning is removed from all three env templates and replaced with the standing rule that `DATABASE_URL` must never name `postgres`, a superuser, or any role holding BYPASSRLS.

### 8.6 Deviations from the plan, session 2

| Deviation | Rationale |
|---|---|
| The RLS helper stays inline in each migration; `libs/shared/db/rls.py` is still not created | Plan 0 already made this choice and the band record kept it. A helper importable by application code invites application code to call it. |
| Migrations run as the schema owner through `BOOTSTRAP_DATABASE_URL`, not as `mytheus_admin` | `mytheus_admin` is the operator seam (seed script, RTBF purge), not the migration runner. Moving table ownership to it in the same migration that creates it adds a failure mode and buys nothing for the D1 property, which is entirely about the request path. |
| Role passwords live in `scripts/bootstrap_roles.py`, not in the migration the plan describes | Keeps secrets out of git history while still satisfying "a migration establishing the request-path DB role without BYPASSRLS". |
| `org_resolver.py` implements **no** cache, despite ADR-0010 D2's "cache the mapping, never the verdict" | `clerk_org_id`, `id` and `status` are in the same row, so one indexed lookup already returns both the mapping and the verdict. A mapping cache in front of it saves nothing and adds a poisoning surface. The ADR's caching only pays once a suspension arrives as a `tenant-events` message that can invalidate an entry, because only then can the status read be skipped safely. **Exit trigger:** build it in the same change that adds the `tenant-events` consumer, never before. |
| `tenant_settings.settings jsonb` is dropped rather than retained alongside the named columns | It was never read, and leaving it invites writes that shadow the named compliance columns. |
| `POST /api/v1/orgs` still returns nothing; no routes were added at all | Unchanged from the earlier decision (section 3). This session was schema and shared libraries; section 3.6 onward is the next unit of work. |

### 8.7 Verification, session 2

Every command below was run and its output observed on 2026-07-30. Nothing here is inherited from a prior session's claim.

| Command | Result |
|---|---|
| `docker compose down -v` then `up -d postgres redis` | Database rebuilt from an **empty volume**, so the whole chain is proved to replay rather than to depend on accumulated local state |
| `alembic upgrade head` | All four migrations apply clean: `82d44163c933` -> `a1c4e7f20b18` -> `b7d9f3a55c21` -> `c3e8b1d47f92` |
| `alembic downgrade -1` then `upgrade head` (on `a1c4e7f20b18` and `c3e8b1d47f92`) | Reversibility proven, which is the plan's section 3.3 acceptance check |
| `python -m scripts.bootstrap_roles` | Sets all three passwords; re-asserts no role holds `rolsuper`/`rolbypassrls`; exits 2 with a clear message when a password variable is missing (verified by running it without them) |
| `make ci` (proto, ruff, black, mypy, pytest) | **61 passed, 1 skipped.** Was 22 passed at the start of the session, 5 passed / 6 skipped before session 1 |
| `docker compose -f compose.prod.yml config` | Valid; per-service credential scoping confirmed by parsing the rendered config |

The one skip is `test_relay_real_bus_roundtrip`, which needs Redpanda and the Schema Registry running; it is skipped rather than silently passing, and it is the only skip.

Tests added this session (39 net-new assertions across five files): `test_roles_and_policies.py` (10), `test_audit.py` (6), `test_org_resolver.py` (6), `test_rbac_grants.py` (5), plus rewrites of `test_idempotency.py` (7), `test_auth.py` (9) and `test_rls.py` (3).

---

## 9. Open gaps as of the end of session 2

Superseded by section 12. Kept because the ordering it argued for is what session 3 then followed: CI first, because assertions nothing enforces are not controls.

---

## 10. P2 session 3, 2026-07-30: CI, and every route (sections 3.11, 3.5, 3.6, 3.8, 3.9)

The nine HTTP services stop being route-less. Everything below runs on the section 3.3 foundation rather than around it: every route resolves its tenant through `org_resolver`, binds `app.current_org` transaction-locally, and is denied by default.

### 10.1 Section 3.11: CI, first, because the assertions were not yet controls

`.github/workflows/ci.yml` in `mytheus-backend` and `mytheus-events`. Six jobs in the backend: `lint`, `test`, `migrations`, `authz-check`, `security`, `docker`.

Three things in it are load-bearing rather than boilerplate:

- **`MYTHEUS_REQUIRE_DB=1` turns an unreachable database from a skip into a hard failure.** This is defect D7's own failure mode, so leaving it to chance would be leaving the door open behind us. A second step then greps the summary line to assert the four isolation tests actually ran and passed, in case one is later renamed, xfailed, or dropped from collection.
- **`BOOTSTRAP_DATABASE_URL` and `DATABASE_URL` are set to DIFFERENT roles.** The suite seeds cross-tenant fixtures as the owner and asserts isolation as `mytheus_app`. Pointing both at the owner would make every isolation test pass while proving nothing, because the owner is not subject to the policies. That is the exact shape D1 took.
- **The `migrations` job deliberately grants `BYPASSRLS` and asserts `bootstrap_roles` refuses.** A guard nobody has seen fail is a guard nobody knows works.

`pip-audit` runs without `|| true` and immediately found a real transitive vulnerability: `setuptools` 82.0.1, PYSEC-2026-3447. Floored to `>=83.0.0` rather than allowlisted, which is the difference between a scanner and a formality.

**A false-negative in the authz gate, found by running it.** `scripts/check_authz.py` inspected only parameter DEFAULTS, so it failed all nine new routes even though each declares `Annotated[RequestContext, Depends(require_tier(...))]` - the style FastAPI documents and every route here uses. That is worse than having no gate: the fastest way to a green build is to mark a route `# authz: public`, and once one route lies the marker means nothing anywhere. The checker now also walks annotations. Verified in both directions: it passes the real routes, and it rejects a deliberately unsecured probe route.

### 10.2 Section 3.8: the authz gate and the edge

`libs/shared/authz.py` is the chain every route depends on: `verify_token` -> `resolve_org_id` -> `tenant_session` (binds `app.current_org` for the request's one transaction) -> `require_tier`. RLS and the tier check are belt and braces for each other: a route that forgets the tier check still cannot cross a tenant boundary, and a route that somehow escaped RLS still cannot be called by the wrong tier.

A denial writes `authz.deny` on its OWN transaction, not the request's. The request transaction is about to be discarded by the 403, and the denial is precisely the fact worth keeping; writing it into the doomed transaction would record every denial except the ones that happened.

`services/api_gateway/edge.py` pre-validates the token, resolves the tenant once, and forwards `X-Mytheus-Org` / `X-Mytheus-User` under an HMAC. Two details that are the point rather than decoration: inbound copies of those headers are **stripped** before anything else runs, so a client cannot assert its own tenant; and the HMAC covers org and user **together**, so two valid contexts cannot be mixed into tenant A's org with tenant B's user.

**A real bug this surfaced.** `audit_logs.ip` is `inet`, and audited operations fail closed, so a value Postgres cannot parse fails the audit write and therefore the operation being audited. A caller-supplied `X-Forwarded-For: not-an-ip` was enough to stop meeting creation. The IP is now validated with `ipaddress` and recorded as NULL when unparseable: provenance is worth less than availability here.

### 10.3 Section 3.6: meetings CRUD and the org hierarchy

`POST/GET/PATCH /api/v1/meetings`, plus `/api/v1/teams` and `/api/v1/departments`. The row write, the `meeting-events` outbox row and the `audit_logs` row commit in one transaction because they are one fact.

Cursor pagination over `(created_at, id)`, never `OFFSET`: `OFFSET` re-scans every skipped row and a concurrent insert shifts every later page, silently dropping a row from the caller's view. `id` is in the key because `created_at` alone is not unique, and two meetings created in the same microsecond would make one unreachable.

The `updated` outbox key carries `updated_at`. Keyed on the meeting id alone, the outbox's unique index would have swallowed every update after the first and the bus would have lost them silently.

### 10.4 Section 3.5: the Clerk webhook and reconciliation

`POST /clerk/webhook` verifies the Svix signature on the raw body **before parsing**, resolves the tenant, dedupes, then applies and records in one transaction.

**Dedupe is on the Svix MESSAGE id, not the Clerk user id.** The message id is what is stable across Clerk's retries of one delivery. Keyed on the user id, the first event about a person would have been applied and every legitimate later update dropped as a duplicate, permanently.

`reconciliation.py` is the hourly sweep the plan asks for, and it is a diff rather than a replay, so it is idempotent by construction. It takes the roster as an argument instead of fetching it, which makes the diff testable with no Clerk account. It deliberately does NOT update users present on both sides: the webhook owns edits, the sweep owns membership, and overwriting here would let a slow sweep silently revert a just-delivered update with the winner decided by timing.

### 10.5 Section 3.9: the bff dashboard

`GET /bff/dashboard`, one composite with four arms gathered concurrently, each under a 2s budget. Every arm carries its own `status`, because a client must distinguish "you have no meetings" from "we could not load your meetings" - those are different empty states and guessing from an empty list gets it wrong half the time. The three unbuilt arms report `empty`, not `degraded`: they genuinely have nothing in them, as opposed to having failed.

### 10.6 Deviations, session 3

| Deviation | Rationale |
|---|---|
| `GET /api/v1/teams`, not the plan's `GET /api/v1/orgs/{id}/teams` | The plan's own form puts a tenant identifier in the path, violating the `org_id`-never-in-path invariant the same plan states. The tenant is already known from the token, so the path parameter could only agree with it or be an attempted cross-tenant read. |
| `meetings` is NOT range-partitioned, so the plan's `PARTITION BY RANGE (scheduled_start)` DDL in section 3.6 step 1 is not implemented | Already decided and recorded in section 3. PostgreSQL requires every partition-key column in a unique constraint, so partitioning by `scheduled_start` makes `UNIQUE (org_id, id)` impossible and with it the composite foreign keys that make a cross-tenant dangling reference unrepresentable. |
| No `Idempotency-Key` conflict detection on `POST /meetings` (plan section 3.6 step 4's `409 IDEMPOTENCY_KEY_REUSED`) | The middleware replays the cached response for a repeated key but does not fingerprint the body, so same-key-different-body currently replays instead of returning 409. `ErrorCode.IDEMPOTENCY_KEY_REUSED` exists unused. **Recorded as an open gap rather than quietly skipped**; it needs a body hash alongside the cached response. |
| The gateway does not reverse-proxy | Every service still runs in one process, so there is nothing to proxy to. `edge.route_for()` and the routing map are built and tested so the split is a deployment change, not a code change. |
| Per-tenant rate limiting is still a stub | The full posture belongs with the WAF work in doc 11 section 5, and a token bucket in one process is wrong the moment there are two replicas. |
| `tests/test_auth.py`'s webhook test was deleted rather than fixed | It built a bare `FastAPI()` instead of `create_app()`, so it had no RFC 7807 handlers and asserted against an app shaped unlike any deployed one. The same assertion now lives in `test_clerk_webhook.py` against a real app. |
| Route tests use one module-scoped `TestClient` and psycopg2 for assertions | Each `TestClient` runs its own event loop while the process-wide async engine pools connections, so a second client reuses a connection bound to a dead loop. Switching the signed-in principal on one client avoids it; synchronous assertions keep the test bodies off the loop entirely. |

### 10.7 Verification, session 3

Every command run and observed on 2026-07-30, against a database rebuilt from an empty volume.

| Command | Result |
|---|---|
| `docker compose down -v`, `up -d`, `alembic upgrade head` | All four migrations apply clean from empty |
| `python -m scripts.bootstrap_roles` | Three roles set; none holds `rolsuper` or `rolbypassrls` |
| `make ci` with `DATABASE_URL` naming `mytheus_app` and `BOOTSTRAP_DATABASE_URL` the owner | **114 passed, 1 skipped.** ruff and black clean, mypy clean over 62 source files |
| `python scripts/check_authz.py` | Passes on the nine real routes; **exits 1** on a deliberately unsecured probe route (both directions verified) |
| `pip-audit --skip-editable` | No known vulnerabilities, after flooring `setuptools` to close PYSEC-2026-3447 |
| `MYTHEUS_REQUIRE_DB=1` against an unreachable port | Errors instead of skipping, verified deliberately |
| `python -c "yaml.safe_load(...)"` on both new workflows | Valid |

**A configuration bug this verification caught.** `tests/conftest.py` derived the owner URL from `DATABASE_URL`, which since section 3.3 names `mytheus_app`. Every database test errored for anyone whose `.env` was correct, while passing in CI where `DATABASE_URL` happened to be the owner. It now reads `BOOTSTRAP_DATABASE_URL` first, and parses the URL to swap credentials rather than string-replacing `postgres:postgres`, which would have silently left every role URL pointing at the owner against a real database. That is a false green of exactly the D1 kind.

Test count by file: `test_gateway_edge.py` 20, `test_meetings_api.py` 15, `test_clerk_webhook.py` 12, `test_roles_and_policies.py` 10, `test_auth.py` 8, `test_idempotency.py` 7, `test_bff_dashboard.py` 7, `test_org_resolver.py` 6, `test_errors.py` 6, `test_audit.py` 6, `test_rbac_grants.py` 5, `test_relay.py` 4, `test_grpc.py` 4, `test_rls.py` 3, `test_outbox_inbox.py` 2. Total 115 collected, 114 passing, 1 skipped (`test_relay_real_bus_roundtrip`, which needs Redpanda; skipped rather than silently passing).

---

## 11. Definition of Done, P2

Against section 10 of the plan.

| DoD line | Verdict | Evidence |
|---|---|---|
| Clerk revert verified | **MET** | `libs/shared/auth.py` is a Clerk RS256 JWKS verifier; a CI grep guard blocks the Supabase SQL returning |
| Migration baseline with RLS | **MET** | Four migrations; every `org_id` table `ENABLE`+`FORCE` with `tenant_isolation`, asserted in CI |
| Identity tables plus `rbac_grants` | **MET** | Section 8.2 |
| Clerk webhook body | **MET** | Section 10.4, 12 tests |
| Meetings CRUD | **MET** | Section 10.3, 15 tests |
| `audit_logs` live from the moment RLS is | **MET** | Section 8.3; wired to `meeting.created`, `meeting.updated`, `authz.deny`, `user.synced` |
| Gateway JWT pre-validation and `org_id` extraction | **MET** | Section 10.2, 20 tests |
| `bff` dashboard | **MET** | Section 10.5, 7 tests |
| `web-app` shell | **NOT MET** | Section 3.10 untouched; `mytheus-ui` has no Clerk and no generated client |
| `mytheus-mobile` realign | **NOT MET** | Not begun |
| CI gates | **MET** | Seven jobs in `mytheus-backend`, three in `mytheus-events`. `spectral lint` and `oasdiff` landed 2026-08-28 in the net-new `openapi` job, alongside the regeneration check; `deploy` is gated on it. See section 17. |

**P2 verdict at the end of session 3: the backend is complete. Section 14 records the mobile realign that followed.**

---

## 12. What is still open

Gap 2 is superseded by section 14: the mobile realign landed.

| # | Gap | Blocks | Picked up by |
|---|---|---|---|
| 1 | **`mytheus-ui` Clerk wiring** - LANDED in session 5 (see section 15). Still open within it: no generated TS client (the specs are hand-written, gap 5), and no browser run against a live Clerk instance or gateway, so section 3.10's two end-to-end acceptance checks (two tenants seeing disjoint data, a network capture showing no `org_id`) are unverified. | The end-to-end demo | A session with real Clerk keys and a deployed gateway |
| 2 | **`mytheus-mobile` realign** - LANDED in session 4. See section 14. | - | Done |
| 3 | **`Idempotency-Key` reuse with a different body replays instead of returning 409.** The middleware caches by key without fingerprinting the body. `ErrorCode.IDEMPOTENCY_KEY_REUSED` exists and is unused. | Plan section 3.6 step 4's acceptance check | Next backend session |
| 4 | **Tenant provisioning still has no reachable path.** `POST /api/v1/orgs` is deferred by decision and the operator seed script standing in for it is still unwritten, so a tenant can only be created by a manual INSERT as `mytheus_admin`. | Onboarding anyone | Deferred, needs a decision |
| 5 | ~~**`spectral lint` and `oasdiff` are not wired.**~~ **CLOSED 2026-08-28** by `mytheus-backend` `4b948d1` and [ADR-0022](../adr/0022-openapi-emitted-from-code.md). The specs are emitted from the FastAPI apps into `mytheus-backend/design/openapi/` and gated three ways: a regeneration check, `spectral lint`, and an `oasdiff` breaking-change diff against the base branch. Running the two Node tools for the first time found four latent defects, three of them in the ruleset itself. See section 17. | - | Done. |
| 6 | **The outbox relay has never run against these events.** `meeting-events` rows are being written by the routes and nothing has drained them, because the real-bus test is the one skipped test. | Nothing yet; the first thing a consumer needs | Plan 3-4 |
| 7 | **Stale Railway/Render references** in reader docs 00, 01, 03, 04, 13, 22 and plan bands 0, 3-4, 5-6, 10-13, 14. Plan 14 needs rewriting, not substituting: its premise is a PaaS-to-Kubernetes migration and the source is now the VPS. | Nothing technically | Docs pass |
| 8 | **`design/tech-lock.md` is unsigned**, and section 3.3 was gated on it. The gate was not honored: 3.3 landed unsigned. Recorded rather than left implicit. | Nothing now | The team |
| 9 | **Clerk's DPA is unexecuted**, and Clerk is the only processor receiving real data in this band. | Nothing technically | Legal |
| 10 | **The live VPS deploy has not run.** No longer blocked by D1: roles, the per-process credential split and the runbook are ready. Back up PostgreSQL off-box and rehearse a restore before any real tenant data lands. | Nothing | Explicit go |

---

## 13. What to do next

1. **Section 3.10, `mytheus-ui`**, once the maintainer confirms it is safe to touch (see 14.3). A reconciliation, not a scaffold: add Clerk, generate the API client, and point the existing `(app)/dashboard` page at `GET /bff/dashboard`, whose shape is now fixed and tested. The mobile client in section 14 is the working reference for the whole client-side contract.
2. ~~**Generate the OpenAPI specs from the code**~~ **DONE 2026-08-28** ([ADR-0022](../adr/0022-openapi-emitted-from-code.md), section 17). This also unblocks B8, the generated TypeScript client, which Plan 1-2 correctly refused to build against the hand-written specs.
3. **Close gap 3**, the idempotency body fingerprint. Small, and a stated acceptance check.
4. **Write the operator seed script** (gap 4) so a tenant can be created without a manual INSERT.

---

## 14. P2 session 4, 2026-07-30: the mobile realign (section 7)

`mytheus-mobile` moves off mock data onto the real backend for the arms that have one. Its git divergence (one local commit against one remote) was rebased first: it is now one commit ahead, clean.

### 14.1 What landed

> **Audit 2026-11-06 (static, not executed):** the 00 gap 32 correction of this section is itself partly wrong. The API layer, sign-in, secure-store and an api-client test did land, in `mytheus-mobile` commit `77290c3`; mobile is still not mock-free. See section 20.

| Artifact | What it does |
|---|---|
| `src/api/client.ts` | The single base fetch wrapper. Knows one address and one credential, injects the Clerk bearer, attaches `Idempotency-Key` to every mutation, and parses RFC 7807 into a typed `ApiError` carrying the stable `code`. |
| `src/api/types.ts` | Wire types for the `bff` and `core-api` reads, including the per-arm `status` the dashboard returns. |
| `src/api/present.ts` | Wire to view model, and the one place `snake_case` becomes `camelCase`. |
| `src/api/provider.tsx` | Binds the client to Clerk's session and supplies the TanStack Query client with a retry policy that does not retry auth failures. |
| `src/api/hooks/index.ts`, `src/api/queryKeys.ts` | One hook per endpoint. Components never call `fetch`. |
| `src/lib/secure-store.ts` | The Keychain/Keystore wrapper, and Clerk's `tokenCache`. |
| `src/lib/session.ts` | Rewritten from a module-level mock boolean to a Clerk delegate, keeping the same surface so the three consuming screens did not need rewriting. |
| `app/_layout.tsx`, `app/sign-in.tsx`, `app/profile.tsx`, `app/(tabs)/index.tsx` | Providers wired, real Clerk SSO sign-in and sign-out, and the home screen reading `GET /bff/dashboard`. |
| `__tests__/api-client.test.ts`, `jest.config.js`, `.env.example` | 14 tests, and the env contract. |

Three decisions worth stating outright, because each is a place where the obvious implementation would have been wrong:

- **The client has no parameter through which `org_id` could be passed.** Not "we remember not to send it": there is no argument for it, and a test asserts no request's URL, headers or body contains the string. The gateway derives the tenant from the token, and a client-supplied one is a value the server must either ignore or trust.
- **Tokens go to `expo-secure-store`, wired as Clerk's `tokenCache`.** Left on Clerk's default they would sit in AsyncStorage, which is plain JS-readable storage that survives on a lost device. This is a stated invariant in the repo's own CLAUDE.md and it is one line to get wrong.
- **`signIn()` was deleted rather than adapted.** The mock exported a no-argument `signIn()` that flipped a boolean. Real sign-in needs a strategy and a browser session, so a no-argument version cannot be implemented honestly, and leaving a working-looking stub invites a screen to call it and appear authenticated while holding no token.

### 14.2 Deviations, session 4

| Deviation | Rationale |
|---|---|
| **`src/data/mock.ts` is NOT deleted**, contrary to step 3 | It is imported by 12 screens, and four of them (`live-session`, `recap`, `insights`, `ask`) render transcripts and agent output whose backends land in Plan 3-4 and Plan 7-9. Deleting it now breaks four working screens in exchange for nothing. The mock is gone from the home screen, which is the one with a real backend, and each remaining import is a screen waiting on a named later band. |
| **The people screen still reads the mock** | It needs an org-member roster and `identity-tenant-service` has no `GET /users` surface. `useTeams()` exists and works, but teams are not people. Faking a roster with a client-side join would invent membership data. **Recorded as a gap, not quietly skipped.** |
| **`MeetingCard` gained optional `durationMin`, `speakers` and `chips`** rather than being fed defaults | Those come from transcription and agents, neither of which exists. Defaulting them to `0` renders "0 min - 0 speakers", which states a fabricated fact as a real one; the card now omits what it does not have. `fromMockMeeting()` bridges the screens still on mock so the component keeps ONE prop type. |
| **`weeklyStats` on the home banner stays on the mock** | It aggregates decisions and action items. The `bff` arms for both are shaped-but-empty, so computing it from real data would display "0 decisions" as though that were measured. |
| **Route groups were not restructured to `(auth)`/`(app)`** as step 1 describes | The app already ships a working `(tabs)` shell with an auth gate. Moving every route to satisfy the plan's directory names would be a large destructive refactor of a working app for no functional gain. |
| **Plain jest, not jest-expo** | `jest-expo` requires React 19.2.3 and the app is on 19.1.0. Bumping the app's React to satisfy a test harness is the wrong way round. The security-relevant layer imports nothing from React Native, so it is fully covered; component/RNTL tests wait for the versions to line up. |
| **Home shows UPCOMING, not RECENT** | `GET /bff/dashboard` returns `upcoming_meetings` (status `scheduled` or `live`). Labelling upcoming meetings "recent" would mislabel the data to preserve a heading. |

### 14.3 `mytheus-ui`: a correction, and why 3.10 did not proceed

**An error worth recording.** This session first concluded that `mytheus-ui` contained only four documentation files and no application code, and that the gap-1 description above was therefore false. That conclusion was wrong. The local clone was **34 commits behind** and its `origin/main` remote-tracking ref had never been fetched, so `git status` reported "up to date" against a stale reference and `git ls-files` listed only what the stale checkout held.

The repo really does contain what the record said: **40 pages** across `(marketing)`, `(app)`, `(auth)` and `(admin)`, plus `src/lib/mock/data.ts` and `src/lib/org-context.tsx`. It is 1301KB on the remote, last pushed 2026-07-23, and it carries a second branch, `local-mytheus-rebuild`, which suggests in-flight work. The clone has since been fast-forwarded so it no longer misreports.

The lesson is specific and worth keeping: **`git status` compares against the last fetch, not against the remote.** On a repo deliberately excluded from the pull sweep, that makes "clean and up to date" meaningless. Fetch before drawing any conclusion about a repo's contents.

Section 3.10 was paused at this point for the maintainer's go, because the plan's literal instruction is `create-next-app@14` in a directory holding 40 real pages, and because the standing instruction is not to modify that repo while work is active in it. **The maintainer authorised proceeding additively on `main`**, and section 15 records what landed.

---

## 15. P2 session 5, 2026-07-30: `mytheus-ui` reconciliation (section 3.10)

Done additively on `main` with the maintainer's explicit go. **No scaffold**: `create-next-app` was never run, and none of the 40 pages was replaced.

Footprint: **6 files modified, 3 added**. The modified files are the root layout, the `(app)` layout, the two `(auth)` pages, the dashboard page, and one dashboard component. Every other page is untouched.

### 15.1 What landed

| Artifact | What it does |
|---|---|
| `src/middleware.ts` (new) | Clerk middleware gating everything outside an explicit public ALLOWLIST. |
| `src/lib/api/client.ts` (new) | The server-only gateway wrapper: Clerk bearer, `Idempotency-Key` on mutations, RFC 7807 to a typed `ApiError`, plus `apiSafe` for per-panel degradation. |
| `src/lib/api/types.ts`, `present.ts` (new) | Wire shapes, and wire to the view models the existing components already expect. |
| `src/lib/api/org-context.ts` (new) | Resolves `OrgContextValue` from the Clerk session, replacing `mockOrgContext`. |
| `src/app/layout.tsx` | `ClerkProvider` at the root. |
| `src/app/(app)/layout.tsx` | Session-derived `OrgContext`. Closes the `TODO(auth)` that stood there. |
| `src/app/(auth)/sign-in`, `sign-up` | Real `<SignIn />` / `<SignUp />`, replacing the 4-July bypass that redirected straight to `/dashboard`. |
| `src/app/(app)/dashboard/page.tsx` | Now an async Server Component reading `GET /bff/dashboard`. |
| `src/components/app/dashboard/TodaysMeetings.tsx` | Optional `meetings` and `status` props, defaulting to the mock. |

Four decisions worth stating, because in each case the obvious implementation would have been wrong:

- **The middleware is an allowlist, not a denylist.** With a denylist, every new authenticated route is public until somebody remembers to add it, and the failure is silent. Here a new route is protected by default and the mistake is a redirect to sign-in, which is noticed immediately.
- **`client.ts` is server-only, and that is a security property.** It imports `auth()` from `@clerk/nextjs/server`, so importing it from a client component fails the build rather than shipping a token-fetching path into the browser bundle.
- **Every gateway fetch is `cache: "no-store"`.** The Next data cache keys on the URL, which is identical for every tenant, so a cached `/bff/dashboard` would serve one tenant's dashboard to the next visitor. The build confirms `/dashboard` renders as `ƒ` (dynamic), not static.
- **`apiSafe`, not `api`, on the dashboard.** One failed read degrades its own panel instead of replacing the screen with an error boundary, mirroring the per-arm status `bff` already returns.

### 15.2 Deviations, session 5

| Deviation | Rationale |
|---|---|
| **No `create-next-app@14`**, contrary to step 1 | It would have destroyed 40 pages. The repo already has the exact route groups step 2 asks for, so the scaffold step was satisfied before the band started. |
| **No shadcn/ui init**, contrary to step 1 | Already recorded as a deviation in section 3: the repo has its own component layer, and a second primitive set forks the visual language. |
| **Clerk 6, not 7** | Clerk 7 requires Next 15 or 16; this app is on Next 14.2.35, which is what the plan itself specifies. Clerk 6 declares `next ^14.2.25`, so it fits with no framework upgrade. Upgrading Next under 40 pages to satisfy an auth library is the wrong way round. |
| **No generated TS client yet** (step 7) | The OpenAPI specs in `design/openapi/` are hand-written and not emitted from the code, so generating from them would encode drift rather than prevent it. `src/lib/api/types.ts` is deliberately small and separate so it is the only place drift can hide. Unchanged open gap. |
| **Only the meetings panel reads real data** | `action_items`, `worklogs` and `decisions` are shaped-but-empty arms until Plan 7-9. Wiring them now would replace a populated demo with three permanently empty boxes and no new information. The same reasoning as the mobile client. |
| **`org.plan` reports `starter` and `user.role` reports `member`** | Both are placeholders in `OrgContextValue` that cannot be filled honestly: plan needs `subscriptions` (billing band) and role is a permission living in `rbac_grants`, readable only through the gateway. Both default to the LEAST privileged value, so a UI gating on either hides a feature rather than offering something the tenant has not got. |
| **`org.id` is Clerk's opaque organization id, not our `org_id`** | Display key only. The browser never learns our tenant identifier and never needs to; the gateway derives it from the token. |
| **`TodaysMeetings` gained optional props rather than a rewrite** | `meetings`/`status` default to the mock, so the component keeps one prop type and any not-yet-reconciled caller renders exactly as before. `durationMinutes` and `participants` became optional for the same reason as on mobile: rendering "0 attendees" states a fabricated fact as a measured one. |

### 15.3 Verification, session 5

| Command | Result |
|---|---|
| `npx tsc --noEmit` | Clean |
| `npm run lint` | No ESLint warnings or errors |
| `npm run build` | Succeeds. All 40 pages build; `Middleware 78.4 kB` compiled; `/dashboard` is `ƒ` (dynamic), confirming no cross-tenant static caching |
| `git status --porcelain` | 6 modified, 3 added. No page deleted or replaced |

**Not verified, and not claimed:** no browser run against a live Clerk instance or a live gateway, because that needs real Clerk keys and a deployed backend. The end-to-end acceptance checks in step 3.10 (two different-tenant users seeing disjoint data, and a network capture showing no `org_id`) remain outstanding and are listed as an open gap. The `org_id`-never-sent property is structurally enforced (no parameter exists for it) and is unit-tested on the mobile client, which shares the contract.

---

## 17. Follow-up session, 2026-08-28: the OpenAPI specs are emitted, and the gates finally run

`mytheus-backend` `4b948d1`. Ratified as [ADR-0022](../adr/0022-openapi-emitted-from-code.md),
with the reversal propagated into [19-api-architecture.md](../19-api-architecture.md) section
11, the ADR index and the matrix. No canon edit was needed: canon names no OpenAPI artifact and
takes no position on spec-first, so this changes no name, number or canonical decision.

**What changed.** Each service's document is now emitted from its FastAPI app by
`make openapi` and committed under `mytheus-backend/design/openapi/`: four documents, 22 paths,
services discovered by walking `services/*/main.py` rather than listed. The five hand-written
specs here are superseded and carry a banner saying so; they remain the P1 design record.

**Why the direction reversed** is in the ADR, but the short version is that doc 19's stated
mechanism - "FastAPI's generated schema is reconciled against it in CI" - was never built, and
is the harder direction by construction. Comparing a hand-written document to a generated one
needs a differ that understands OpenAPI equivalence, and every such differ needs a suppression
list, which is where real drift hides. Emitting needs no differ: regenerate and compare bytes.

**Four latent defects, every one found by running something that had never been run.** This is
the part worth carrying forward, because it is the same shape as the partition-grant finding in
[03](03-plan-3-4-meeting-and-transcription.md) section 5.3 four hours earlier.

| # | Defect | How long it had been there |
|---|---|---|
| 1 | The Spectral ruleset **did not parse**. `org-id-never-in-path` and `snake-case-properties` put `field` as a sibling of `given` instead of inside `then`. | Since P1, 2026-07-27 |
| 2 | Three rules used `(?i)` for case-insensitivity. That is PCRE; Spectral is JavaScript and needs `/pattern/i`. They threw "Invalid regular expression". | Since P1 |
| 3 | The ruleset claimed "oasdiff enforces the removal half in CI" for the frozen `ErrorCode` enum. **It does not.** The enum is reachable only from a response schema, and narrowing a response enum is not breaking by oasdiff's definition. | Since P1 |
| 4 | FastAPI 0.138 does not flatten included routers into `app.routes`; it appends one `_IncludedRouter` per router, holding its child on `original_router`. A flat scan saw only `/healthz`. | Introduced this session, found by looking at the output |

Defect 3 is the one to read twice. It is a **documented control that did not exist**, and it
was believed for eight months because nobody ran the tool it named. Proven by deleting
`NOT_FOUND` from a copy of `core-api.yaml` and running `oasdiff breaking`: it reported the two
paths the same probe deleted and said nothing about the enum. Removing a code IS breaking in
this program, because both clients branch on `code`, so it is now asserted directly by
`test_no_published_error_code_is_ever_removed`, which holds the published list. The false claim
is corrected in both places it appeared.

Defect 4 is worth recording for a different reason: it was mine, it was silent, and the only
thing that surfaced it was reading the generated file instead of trusting that the code had
worked. The first cut shortened exactly one operationId - `healthz`, the one route mounted on
the app itself - and left every real route untouched, which reads as "the uniqueness check
rejected everything" rather than "the walk never saw them".

**The emitted document says three things FastAPI cannot know**, all of which the running app
genuinely does: the RFC 7807 error shape (FastAPI documents `422` as `application/json` and no
other error at all), the `Idempotency-Key` header the middleware reads, and the document-level
bearer scheme that arrives through a `Depends` chain. Before this, a client generated from
these documents could not have parsed a single error the service emits, nor sent the one header
the write contract is built on. The `ErrorCode` enum is emitted from `libs.shared.errors`, which
deletes the "keep the two in sync" instruction `errors.py` had been carrying rather than
restating it.

**Verification.** Every command run and its output observed.

| Command | Result |
|---|---|
| `uv run pytest -q` | **313 passed** (was 304; nine new contract tests) |
| `emit_openapi --check` | Green; then **exit 1** on a hand-edited spec, then green again after regeneration |
| `spectral lint --fail-severity=error` | **0 errors**, 71 warnings (all built-in `spectral:oas` style: missing descriptions, tags, `servers`) |
| Spectral guard-fires: add `/api/v1/orgs/{org_id}/teams` and an `offset` query parameter | `org-id-never-in-path` and `list-endpoints-are-cursor-paginated` both fire, plus two built-in rules |
| `oasdiff breaking` guard-fires: delete a path from a copy | 2 errors, `api-path-removed-without-deprecation` |
| `oasdiff breaking`: delete `NOT_FOUND` from the enum in the same copy | **Not detected.** Defect 3 above |
| `actionlint` on the new `openapi` job | Clean |
| `scripts/check_authz.py` after moving its allowlist to `libs/shared/public_routes.py` | Passes, and still **exits 1** on a deliberately unguarded probe route |
| `emit_openapi --check` with `DATABASE_URL`, `REDIS_URL` and the broker vars unset | Green - the CI job has no database, and this confirms it does not need one |
| `ruff` / `black --check` / `mypy` | Clean; mypy over 93 source files |

**NOT run, and not claimed.** The `oasdiff` job itself has never executed in GitHub Actions; it
was exercised locally through the same `tufin/oasdiff` container the workflow uses, on the same
inputs, but the `pull_request`-only path in CI has not fired. It fires on the next PR.

**One thing deliberately not done.** `api-gateway` publishes no document. It proxies, its routes
live in `edge.ROUTE_MAP`, and emitting it produced a zero-path document that failed the ruleset
for an accurate reason - there is no contract in it. Recorded in ADR-0022 D6 rather than left
looking like an oversight.

## 16. Related

- [00-status-overview.md](00-status-overview.md) for program-wide state.
- [01-plan-0-scaffold-and-contracts.md](01-plan-0-scaffold-and-contracts.md) for what this band inherited.
- [design/README.md](../design/README.md) for the P1 pack index and its deviations.
- [threat-model/stride.md](../threat-model/stride.md) section 5, updated in the same pass: all six defects closed, with the residual risk stated.

---

## 18. Follow-up session, 2026-08-30: the dashboard arms are typed (B8 prerequisite N2)

`mytheus-backend`. Closes the prerequisite the B8 scoping pass identified: until this landed,
generating a TypeScript client from the emitted specs would have made `mytheus-ui` worse than
the hand-written types the client exists to replace. No canon edit, and that was checked
rather than assumed: `_canon/` names no wire model and takes no position on the `bff`
dashboard's response shape, only that `bff` serves `GET/POST /bff/*`. This is the fifth time
in this program a session has checked whether canon needed an edit and found it did not.

### 18.1 The defect, stated as a client would meet it

`services/bff/dashboard.py` declared one arm model with `items: list[dict[str, Any]]`, so the
emitted `design/openapi/bff.yaml` carried an unconstrained object map and a generated client
received `items?: { [key: string]: unknown }[]` beside the `status` union.

Against `mytheus-ui`'s hand-written `Arm<DashboardMeeting>`, that is a strict regression: the
six fields, their nullability, and the four-value meeting `status` union all disappear.
**None of the three OpenAPI gates can see this.** `emit_openapi --check` only asks whether the
spec matches the code, and it did. `spectral lint` has no rule about item types. `oasdiff`
treats loosening a RESPONSE schema as non-breaking. So the loosening is invisible to the whole
gate stack and surfaces only as a worse frontend, months later, in a different repo.

### 18.2 What landed

- **`DashboardMeeting`**, the six-column projection the arm actually returns. `status` is
  IMPORTED from `services/core_api/schemas.py`, which owns `meetings`, rather than restated.
  A second copy of the four values would be a second place for them to drift, and gap 35 is
  this program's standing example of two copies of one thing diverging unnoticed. Those values
  are also exactly the `ck_meetings_status_valid` CHECK constraint, so the `Literal` is
  faithful to the database rather than a guess.
- **`_ArmBase[ItemT]`** with concrete named subclasses `MeetingsArm` and `UnbuiltArm`. The base
  is generic so `items` is typed per panel; the subclasses are concrete so the emitted schema
  carries readable names instead of the `_ArmBase_DashboardMeeting_` Pydantic derives for a
  parameterized generic. PEP 695 syntax, which is what `ruff`'s `UP047` requires on this Python
  floor; there was no existing `TypeVar` in the repo to follow either way.
- **`items` lost its default and is now `required`.** With `Field(default_factory=list)` the
  field is optional in the schema and a generated client types it `items?:`, forcing every
  caller to handle an absence the server never produces. The server always sends it, so the
  schema now says so.
- **The loader builds models rather than dicts**, so the columns are checked at the one place
  that knows what the `SELECT` returned. The two degrade paths pass `items=[]` explicitly.
- `_arm()` now takes the arm class as a parameter. It computes degrade policy; which model
  wraps the result is the caller's concern. Two test wrappers were updated for the new arity.

### 18.3 A wire-format divergence found on the way

The old loader hand-built its rows and called `.isoformat()`, emitting a UTC
`meetings.scheduled_start` as `...+00:00`. `core-api`'s `Meeting` model returns that same
column through Pydantic, which emits `...Z`. **The same column of the same table serialized two
different ways depending on which endpoint answered.** Both are valid RFC 3339 and both parse
in a browser, so nothing was broken, and that is exactly why it survived: a client that
normalises on one spelling is simply wrong against the other endpoint, and only for UTC values.

Typing the model removed the divergence, since both now go through one serializer.
`test_the_dashboard_and_core_api_serialize_the_same_column_identically` asserts the two models
agree field for field rather than asserting a literal spelling, so the pair stays consistent
even if the serializer changes under both at once.

### 18.4 The three unbuilt arms keep their open item type, deliberately

`action_items`, `worklogs` and `decisions` have no read model until Plan 7-9.

Widening them to `list[Any]`, so a client would read `unknown[]`, was implemented and then
reverted: `oasdiff breaking` reports `object` to `any` as `response-property-type-changed` on
all three, and no client can observe the difference because those arrays are always empty.
Loosening a live response type to make an always-empty array read better is not worth a
breaking-change entry.

Inventing a record shape for them was rejected for the opposite reason: it would ship a
fabricated contract to every generated client as though it had been measured. So they keep
exactly the wire type they have today, and `test_the_unbuilt_arms_do_not_invent_an_item_type`
asserts that no item type is invented for them.

### 18.5 The `oasdiff` gate does not fail the build, and never has

Found while checking whether this change was breaking, and it is the more serious finding of
the session. **`oasdiff breaking` prints its findings and exits 0.** The CI step relies
entirely on a non-zero exit (`... breaking "/base/$name" "/head/$name" || failed=1`), so
`failed` is never set and the job passes a build it has just finished describing as broken.

Proven in both directions against `tufin/oasdiff:latest`: deleting `GET /bff/dashboard` from a
copy of the spec reports `api-path-removed-without-deprecation` and exits **0** without the
flag, and exits **1** with `--fail-on ERR`. The flag is now in `ci.yml` with the reasoning
recorded at the step.

This is the fourth documented control in this program found not to exist, after
[ADR-0021](../adr/0021-explicit-grants-no-schema-wide-default-privileges.md) arm 2,
[ADR-0022](../adr/0022-openapi-emitted-from-code.md) defect 3, and gap 31. It shares their
shape exactly: a guard nobody had made fail on purpose. It also shares a subject with defect 3,
which was itself a false claim about what `oasdiff` enforces, so **the same tool has now been
believed twice about something it does not do.** Why it stayed hidden is already on the record
and unchanged: the step is `pull_request`-only and has still never executed in GitHub Actions,
so its exit code had never been observed by anyone.

### 18.6 Verification

Every command run and its output observed, against the shared dev stack.

| Command | Result |
|---|---|
| `uv run pytest -q` | **328 passed, 2 failed.** Both failures pre-existing, see 18.7 |
| `uv run pytest tests/test_openapi_contract.py tests/test_bff_dashboard.py -q` | 21 passed |
| Guard-fire: revert `items` to `list[dict[str, Any]]` | Exactly 3 of the new tests fail, nothing else; green again on repair |
| Guard-fire: restore `Field(default_factory=list)` on `items` | Only `test_every_arm_marks_items_required` fails; green again on repair |
| Guard-fire: give `UnbuiltArm` a fabricated item model | Only `test_the_unbuilt_arms_do_not_invent_an_item_type` fails; green again |
| Guard-fire: re-add an `.isoformat()` serializer to `DashboardMeeting` | Only the cross-endpoint agreement test fails, on `+00:00` against `Z`; green again |
| `oasdiff breaking` guard-fire: delete a path, no flag | Reports the removal, **exits 0**. The defect in 18.5 |
| `oasdiff breaking` guard-fire: same probe, `--fail-on ERR` | Reports the removal, **exits 1** |
| `oasdiff breaking --fail-on ERR`, this change against `main` | No breaking changes |
| `emit_openapi --check` | Specs match the code |
| `spectral lint --fail-severity=error` | **0 errors**, 74 warnings, unchanged from the previous session |
| `scripts/check_authz.py` | Passes |
| `ruff` / `black --check` / `mypy` | Clean; mypy over 93 source files |
| `actionlint` on the edited `ci.yml` | Clean |
| `openapi-typescript@7` over the emitted `bff.yaml`, before and after | `items?: { [key: string]: unknown }[]` becomes `items: DashboardMeeting[]`, with `status` narrowed to the four-value union and `format: uuid` / `format: date-time` carried through |

The generated-client comparison is the acceptance check that matters here, and it was run
rather than reasoned about: the whole point of the task is what a client receives, so a client
was generated from both specs and the two outputs compared line by line.

### 18.7 Two pre-existing failures, and what they are

`test_public_schema_grants_nothing_by_default` and
`test_no_bypassrls_role_holds_privileges_on_a_public_table` fail on the shared dev database.
**They are not from this change**, proven by stashing it and running the file against `main`:
both fail identically. The database is at head (`9b2c4d1e6f30`) and holds accumulated state
(3,884 `organizations`), so this is not a fresh-volume artifact.

The cause is gap 29's residual: `pg_default_acl` carries 17 entries granted by `supabase_admin`,
the `supabase/postgres` image's own bootstrap
`ALTER DEFAULT PRIVILEGES ... GRANT ALL ON TABLES TO anon, authenticated, service_role`, which
a non-superuser migration runner cannot alter. Gap 29 records both residuals as inert and
"asserted so they stop being silent"; on this cluster the assertion is currently red rather
than green, which the 2026-08-29 record's 325-passed run implies it was not. Left untouched and
reported rather than worked around, because closing it is the substrate decision gap 29 already
frames (move off the `supabase/postgres` base image), not a change to make in passing.

### 18.8 NOT run, and not claimed

- **Nothing was run against the deployed VPS**, and no client was regenerated into
  `mytheus-ui`. B8 itself is untouched: this is only its prerequisite. The client compared in
  18.6 was generated into a scratch directory, never into that repo.
- **The `oasdiff` fix has still not executed in GitHub Actions.** It was exercised locally
  through the same `tufin/oasdiff:latest` container the workflow uses, on the same command
  shape, in both directions. The `pull_request`-only path fires on the next PR, and that will
  be its first real run, exactly as recorded before this session.
- **No rebuild from an empty cluster.** Unchanged, and for the unchanged reason: the dev stack
  is shared with the other track.

---

## 19. Follow-up session, 2026-08-30: the generated TypeScript client (B8)

`mytheus-ui` `2e298c5`. Closes section 3.10 step 7, which Plan 1-2 recorded as a deliberate
deviation ("No generated TS client yet"), and closes live gap 2. No canon edit: `_canon/` names
no wire model and takes no position on client codegen, checked rather than assumed.

**Scope decision, taken by the maintainer.** The original B8 assumed two client repos. Gap 32
established that `mytheus-mobile` has no API layer to regenerate into, so the maintainer scoped
B8 to `mytheus-ui` alone and excluded `mytheus-mobile` from this session outright. Mobile's
client is therefore folded into gap 32 rather than stretching B8 to cover a repo with nothing
to replace. `mytheus-mobile` was not opened, read or modified.

**Cross-repo note.** `mytheus-ui` is the other track's repo. The work is additive, no existing
screen was changed, and every exported type name is unchanged, so nothing in their meeting,
LiveKit or caption surfaces was touched. It was fetched before and after; `main` was at
`0f757ca` throughout with no in-flight work.

### 19.1 What the defect actually was

Three hand-written type sets, kept in sync by attention:

| Where | What it described |
|---|---|
| `src/lib/api/types.ts` | `Arm<T>`, `DashboardMeeting`, `DashboardResponse`, `ApiMeeting`, `MeetingPage`, `ApiTeam`, `Participant` |
| `src/lib/api/client.ts` | The `ErrorCode` union and `Problem` |
| `src/lib/api/transcript.ts` | `TranscriptChunk`, `TranscriptPage`, `MeetingSpeaker`, inline |

Attention had already lapsed, in the direction that matters. The `ErrorCode` union listed 15 of
the 17 codes the server emits; the two missing were `CONSENT_REQUIRED` and `NOT_A_PARTICIPANT`,
which are exactly the two outcomes `app/api/livekit-token/route.ts` exists to keep apart. That
route relays `error.code` rather than naming it, so the shortfall was a type that had quietly
stopped describing the server rather than a crash: the value arrived at runtime while the type
said it could not. This also closes the item recorded separately as the `ErrorCode` drift.

It only became safe to generate now. Plan 1-2 refused, correctly, because the specs were
hand-written and generating from them would have encoded drift rather than prevented it
([ADR-0022](../adr/0022-openapi-emitted-from-code.md) changed that by emitting each document
from its FastAPI app), and the emitted `Arm.items` was weaker than the hand-written `Arm<T>`
until the dashboard arms were typed in section 18.

### 19.2 What landed

- **`scripts/generate-api-types.mjs`** writes `src/lib/api/generated/` from `mytheus-backend`'s
  `design/openapi/*.yaml`, one module per service document, and carries a `--check` mode that
  fails when the committed output and the spec disagree. That is the same
  regenerate-and-compare shape ADR-0022 uses on the specs themselves, applied one layer out.
- **`types.ts` becomes an assembly layer**: every type is now an alias onto a generated schema.
  Every exported name is unchanged, which is why no screen needed editing.
- **`client.ts`'s `ErrorCode` and `Problem`** are taken from the spec. The drift closes by
  construction rather than by being noticed.
- **`transcript.ts`'s three inline interfaces** likewise.

**The output is committed, deliberately.** This app has to build with no sibling checkout: CI,
the Docker build and a fresh clone all compile without `mytheus-backend` on disk. Committing
buys that; `--check` is what stops the committed copy from quietly becoming a second source of
truth, which is gap 35's hazard.

**Vendoring the specs into `mytheus-ui` was considered and rejected** for the same reason B4
refused to vendor the topic list into `mytheus-platform`: a second copy of a canonical artifact
drifts, and the drift surfaces as a wrong client rather than a failing test.

### 19.3 Two defects found by running the script rather than reading it

Both were mine, introduced in the first cut, and both are the shape this program keeps finding.

1. **An explicitly set but wrong `MYTHEUS_OPENAPI_DIR` silently fell through to a guessed
   directory.** The override sat in the same candidate list as the fallbacks, so naming a path
   that did not exist produced a correct-looking client generated from a different server. That
   is a silent fallback where a loud failure belongs, the same reasoning B3 applied to Qdrant's
   `write_client()` refusing to build without a key. An override is now used or it fails.
2. **The generator emits CRLF on Windows and LF on Linux**, so a byte comparison would have been
   red for a Windows developer and green in CI on identical content. This is precisely what cost
   `mytheus-realtime` a whole lint gate in Plan 3-4 (gap 14, no `.gitattributes` plus
   `core.autocrlf` against `prettier --check`). Fixed in the comparison itself through one
   shared normaliser on both the write and read paths, so the gate cannot be platform-dependent
   whatever git does, with a `.gitattributes` as well so a checkout cannot undo it. Fixing it in
   the script rather than only in config is the difference between the property holding and the
   configuration happening to be right.

### 19.4 Verification

Every command run and its output observed.

| Command | Result |
|---|---|
| `npx tsc --noEmit` | Clean, before and after |
| `npm run lint` | No ESLint warnings or errors |
| `npm run build` with a Clerk key | **Compiled successfully, 54/54 static pages, exit 0** |
| Positive control: rename `scheduled_start` in the generated spec output | `present.ts` fails to typecheck. The app is genuinely bound to the spec |
| Positive control: drop `cancelled` from `Meeting.status` | `present.ts` fails twice, on the switch arm and on the call. Both restore clean |
| Guard-fire: hand-edit a generated file | `--check` reports `differs from the spec` and **exits 1** |
| Guard-fire: delete a generated file | `--check` reports `missing from the committed output` |
| Guard-fire: `MYTHEUS_OPENAPI_DIR=/nonexistent` | **exits 2** with the path it was given, after the fix; exited 0 before it |
| Guard-fire: rewrite a committed file to CRLF, content identical | `--check` **passes**, which is the point of the normaliser |
| Same file, one real content change | `--check` fails again. The line-ending tolerance did not blunt it |
| `npm run generate:api:check` on the committed tree | Matches |

The exit codes were checked without a pipe, deliberately: `$?` after `npm ... | tail` is the
exit of `tail`. That is not a hypothetical worry, it is the defect found in the `oasdiff` gate
the same day (gap 39), where a tool printed its findings and exited 0 and the CI step depended
on the exit code.

**A build failure that is NOT from this change:** `next build` without a Clerk publishable key
fails at prerender, because `ClerkProvider` wraps the root layout and validates its key at build
time. That is what this repo's own CI comment already documents and why CI passes build args.
Proven pre-existing by stashing the change and rebuilding on `main`, which fails identically.
With a syntactically valid key supplied, the full build passes.

### 19.5 Open, and stated rather than papered over

- **`generate:api:check` is not wired into `mytheus-ui`'s CI.** It cannot be, honestly, without
  a decision that is not this session's to take: the workflow checks out one repo, and the check
  needs `mytheus-backend`'s specs. Vendoring them here would create the second copy 19.2
  rejects, and a step that silently passes when the specs are absent would be worse than no step
  at all, which is the reasoning that workflow's own author used to leave Vitest out rather than
  run zero tests green. Closing it needs either a cross-repo checkout in that workflow or the
  check moving into `mytheus-backend`'s pipeline, and it needs the other track's agreement
  because it is their pipeline.
- **Nothing was regenerated into `mytheus-mobile`**, per the maintainer's scoping. Gap 32 owns
  that repo's client.
- **No screen consumes a newly available type yet.** This change makes the existing screens
  honest; it adds no surface. `UserContext`, `OrgChart`, `TenantSettings` and the rest are now
  available to the org-chart and person-profile UI when that is built, which is where B9 starts.
- **Nothing was run against a deployed backend.** The types are checked against the specs, and
  the specs are emitted from the code, but no request was made through a live gateway.

## 20. Audit 2026-11-06

Static code audit, **not executed**. Other agents' code changes were in flight; the lead will fold in final results. Earlier sections are kept as history.

- **Idempotency `409` (observed in source).** `libs/shared/idempotency.py` returns `409` on key reuse with a different body, using a request fingerprint. The "`IDEMPOTENCY_KEY_REUSED` remains unmet" claim in 00 gap 3 is stale for the behavior; no `409` test existed at audit time. See 00 gap 43.
- **`mytheus-mobile` API layer.** Landed in commit `77290c3` (`src/api`, sign-in, secure-store, api-client test). 00 gap 32's "never landed" is wrong for those artifacts. Not mock-free: `library`, `people`, `actions`, `profile`, `recap`, `insights` and `live-session` import `@/data/mock`. The band-status line "clients not started" at the top of this file is also stale. See 00 gap 44.
- **`POST /api/v1/orgs`** returns `501`. See 00 gap 45.
