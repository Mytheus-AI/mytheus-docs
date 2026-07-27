# 02 Plan 1-2 Foundation and Tenancy

| Field | Value |
|---|---|
| Band | Plan 1-2, phases P1 (Architecture and Design, W1-2) and P2 (Foundation, W3-5) |
| Plan file | [plan/plan-1-2-foundation-and-tenancy.md](../plan/plan-1-2-foundation-and-tenancy.md) |
| Status | **IN PROGRESS. P1 COMPLETE, P2 NOT STARTED.** |
| Complete | `mytheus-docs`: canon amendments plus the full P1 design pack |
| Not started | `mytheus-backend`, `mytheus-ui`, `mytheus-mobile`, `mytheus-platform`, `mytheus-events` |
| Last verified | 2026-07-27, by file inspection and YAML parse |

**No code was written in this session.** Everything below is documentation, canon, and specification. The entire P2 build phase remains ahead.

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
| OpenAPI specs pass Spectral | **PARTIAL** | All five parse as valid YAML (verified). `spectral lint` has **not** been run: the CLI is a Node dependency and no repo has CI yet. |
| Avro drafts validate BACKWARD | **MET, differently** | No drafts were produced. `mytheus-events` already ships all four band schemas, registered under a BACKWARD gate. The check is redirected there and recorded as a deviation. |
| STRIDE model authored | **MET** | Auth and APIs rows complete; the other four stubbed to canon controls, which is what the plan asks for. |
| Residency region chosen | **MET** | `us`, single-region, with the table populated per tenant regardless. |
| DPAs signed or pending-with-owner | **MET** | Thirteen processors, each with an owner and a status. Clerk flagged as the blocking one. |
| RTBF contract frozen | **MET** | Plus three structural traps found while writing it. |
| Both section 8 decisions surfaced | **MET** | Both resolved before this session; recorded in canon and both ADRs this session. |

**P1 verdict: substantively complete. Three administrative lines remain: PR review, the tech-lock signature, and an actual Spectral run.**

### P2 (W3-W5)

**NOT STARTED.** Every line of section 10's P2 DoD is outstanding: the Clerk revert verification, the migration baseline, identity tables, the webhook body, meetings CRUD, `audit_logs`, the gateway, `bff`, `web-app`, `mytheus-mobile`, and every CI gate.

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

## 7. What P2 should do first

The plan's order is section 3.2 then 3.3 onward. That order stands, with one insertion and one reordering, both justified above:

1. **3.2** Verify the Clerk revert. Add the missing `CLERK_JWT_ORG_CLAIM` to `.env.example`.
2. **3.2b (net-new)** Repair the environment: align the compose ports with the test ports, move `temporal-ui` off 8080, `uv sync`, then run `make ci` and **record the real result** rather than inheriting a claim.
3. **3.3** The three roles, the `organizations` policy removal, `clerk_org_id`, `status`, the foreign keys at the mixin, and the gap 2 and gap 9 fixes. Nothing else can be verified until this lands.
4. **3.7 (partial, moved earlier)** The `audit_logs` migration and the `libs/shared/audit.py` helper, because section 3.6's routes depend on the helper existing.
5. Then 3.4, 3.5, 3.6, the rest of 3.7, 3.8, 3.9, 3.10, 3.11, and section 7 mobile as planned.

---

## 8. Related

- [00-status-overview.md](00-status-overview.md) for program-wide state.
- [01-plan-0-scaffold-and-contracts.md](01-plan-0-scaffold-and-contracts.md) for what this band inherited.
- [design/README.md](../design/README.md) for the P1 pack index and its deviations.
- [threat-model/stride.md](../threat-model/stride.md) section 5 for gaps 1 to 5 in detail.
