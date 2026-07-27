# 00 Status Overview

Program state as of 2026-07-27. Scope: all eight repos in the Mytheus (NEXUS) program.

## Decisions resolved 2026-07-23

All three items that gated Plan 1-2 entry are now settled and recorded.

| Decision | Outcome | Recorded in |
|---|---|---|
| Tenant claim shape (Plan 1-2 §8 #1) | Clerk's **native `org_id`**, resolved at the edge to our `org_id` via `organizations.clerk_org_id`, Redis-cached. The lookup is the authorization checkpoint (org exists, not suspended, not soft-deleted), not overhead. SSO/SAML/SCIM deferred. | [plan/plan-1-2 §8](../plan/plan-1-2-foundation-and-tenancy.md), [plan/README.md](../plan/README.md) |
| RBAC tier vs `users.role` (Plan 1-2 §8 #2) | **Separate stores.** `rbac_grants.tier` is the only thing authorization reads; `users.role` + `skills[]` stays a profile attribute. | [plan/plan-1-2 §8](../plan/plan-1-2-foundation-and-tenancy.md), [plan/README.md](../plan/README.md) |
| MVP compute substrate | **Self-hosted single-node VPS**, Docker Compose, whole backend plus all four stores. Railway and Render struck entirely. | [ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md), `_canon/06-tech-decisions.md` §2.1 |

Also settled: **`web-app` = `mytheus-ui`**, canon unchanged. `mytheus-web` is a reference-only Next.js draft of the meeting UI - not a build target, not deployed, not in CI. Its `meeting-gap-notes.md` feeds Plan 3-4.

## Band status

| Band | Plan file | Status | Note |
|---|---|---|---|
| Plan 0 Scaffold | [plan-0-scaffold-and-contracts.md](../plan/plan-0-scaffold-and-contracts.md) | **DONE WITH GAPS** | Backend, events, intelligence, realtime, **platform** complete. UI and mobile incomplete, both deliberately. See [01-plan-0-scaffold-and-contracts.md](01-plan-0-scaffold-and-contracts.md). |
| Plan 1-2 Foundation and Tenancy | [plan-1-2-foundation-and-tenancy.md](../plan/plan-1-2-foundation-and-tenancy.md) | **IN PROGRESS. P1 COMPLETE, P2 NOT STARTED.** | P1 landed 2026-07-27: canon amendments plus the seventeen-file design pack. **No code written.** Plan 0 also pre-empted several P2 items: the Clerk revert, RLS `FORCE` on 11 tables, the outbox relay, gRPC `GetUserContext`. See [02-plan-1-2-foundation-and-tenancy.md](02-plan-1-2-foundation-and-tenancy.md) and the section "Plan 1-2 pre-empted work" below. |
| Plan 3-4 Meeting and Transcription | [plan-3-4-meeting-and-transcription.md](../plan/plan-3-4-meeting-and-transcription.md) | NOT STARTED | |
| Plan 5-6 Org Context and AI Infra | [plan-5-6-org-context-and-ai-infra.md](../plan/plan-5-6-org-context-and-ai-infra.md) | NOT STARTED | |
| Plan 7-9 Agents and Monetization | [plan-7-9-agents-and-monetization.md](../plan/plan-7-9-agents-and-monetization.md) | NOT STARTED | |
| Plan 10-13 Collab, Integrations, GA | [plan-10-13-collab-integrations-ga.md](../plan/plan-10-13-collab-integrations-ga.md) | NOT STARTED | |
| Plan 14 Scale-Out | [plan-14-scale-out.md](../plan/plan-14-scale-out.md) | NOT STARTED (post-GA, demand-driven) | |

## Repo inventory

Eight repos in the program, plus one reference-only repo. Most on-disk clones sit inside a `Mytheus-*` wrapper folder and still carry a legacy `reprise-` inner name that predates the Mytheus rename. The canonical repo name is always the `mytheus-` form and is the only form used in prose.

| Canonical repo | GitHub | On-disk path | Plan 0 state |
|---|---|---|---|
| `mytheus-docs` | `Reprise-AI/reprise-docs` | `Mytheus-docs/reprise-docs` | Canon, 16 ADRs, diagrams, eight band plans complete. |
| `mytheus-backend` | `Reprise-AI/reprise-backend` | `Mytheus-backend/reprise-backend` | Complete (Phases 0B, 0C). |
| `mytheus-events` | `Reprise-AI/reprise-events` | `Mytheus-events/reprise-events` | Complete (Phase 0A). |
| `mytheus-intelligence` | `Reprise-AI/reprise-intelligence` | `Mytheus-intelligence/reprise-intelligence` | Complete (Phase 0D). |
| `mytheus-realtime` | `Reprise-AI/reprise-realtime` | `Mytheus-realtime/reprise-realtime` | Complete (Phase 0E, realtime half). |
| `mytheus-platform` | `Mytheus-AI/mytheus-platform` | `Mytheus-platform/mytheus-platform` | **Complete** (Phases 0F, 0G). Cloned 2026-07-23; previously unverifiable. |
| `mytheus-ui` | `Reprise-AI/reprise-ui` | `Mytheus-UI/reprise-ui` | **Not done, deliberately.** Phase 0E web half skipped while active UI work was in flight. Large mock-data site exists instead. Canonical `web-app`. |
| `mytheus-mobile` | `Reprise-AI/reprise-mobile` | `Mytheus-mobile/reprise-mobile` | **Not done.** No Clerk SDK, no `httpClient`, no `REALIGN.md`. |
| `mytheus-web` (reference only) | `Mytheus-AI/mytheus-web` | `Mytheus-Web/mytheus-web` | **Not a program repo.** Next.js 16 / React 19 draft of the meeting UI, `(call)` and `(shell)` route groups. Not a build target, not deployed, not in CI, not in canon. Its `meeting-gap-notes.md` is a docs-versus-draft gap analysis feeding Plan 3-4. |

## Plan 1-2 pre-empted work

Plan 1-2 section 1 states its entry preconditions assuming Plan 0 left "zero DDL" and a Supabase-Auth drift still in place. Both assumptions are now stale. The following Plan 1-2 items already landed during Plan 0 and must be treated as **done, verify only**, not rebuilt:

| Plan 1-2 step | Actual state after Plan 0 |
|---|---|
| 3.2 Revert the Supabase-Auth drift to Clerk | Done. `libs/shared/auth.py` is a Clerk RS256 JWKS verifier; `POST /clerk/webhook` verifies Svix; `CLERK_*` restored in `.env.example`; the Supabase `custom_access_token_hook` and `auth.users` trigger SQL files are deleted. A CI grep guard blocks their return. |
| 3.2 step 5 Mount the Idempotency middleware | Done. `libs/shared/idempotency.py` is a Starlette middleware mounted by `libs/shared/app_factory.create_app()` for all nine HTTP services. |
| 3.3 Migration baseline, RLS macro | Done differently. One migration `82d44163c933` creates 11 tables with `ENABLE` + `FORCE` + `tenant_isolation` inline, rather than the plan's `libs/shared/db/rls.py` helper plus two migrations. |
| 3.3 fail-closed RLS predicate | Done and hardened. The policy uses `NULLIF(current_setting('app.current_org', true), '')::uuid`, so an unset GUC yields zero rows rather than raising. This is stricter than the plan's literal SQL. |
| 3.4 identity tables | Partially done. `organizations`, `users`, `tenant_settings`, `data_residency` exist as models plus migration. `rbac_grants` does **not** exist. |
| 3.4 step 5 gRPC `GetUserContext` | Done as a stub. `services/identity_tenant_service/grpc_server.py` reads `org_id` from gRPC metadata, fails closed with `UNAUTHENTICATED`, sets `app.current_org`, returns a typed context with a default tier. |
| 3.5 step 3 outbox relay | Done. `libs/shared/relay.py` (`FOR UPDATE SKIP LOCKED`, flush-then-mark) plus `services/relay/main.py` (confluent-kafka Avro producer, `acks=all`, idempotent). |
| 3.6 `meetings` / `meeting_participants` tables | Tables exist as P2 stubs with RLS. No CRUD routes. |

## Blocking defects, found 2026-07-27

These are defects in code that already exists, each confirmed by reading the named source. **None appears in any plan file.** They are the first work of Plan 1-2 P2, ahead of any feature, because until they are fixed nothing in the program can be honestly verified. Full detail in [02-plan-1-2-foundation-and-tenancy.md](02-plan-1-2-foundation-and-tenancy.md) section 5 and [threat-model/stride.md](../threat-model/stride.md) section 5.

| # | Defect | Consequence |
|---|---|---|
| D1 | **The application connects to PostgreSQL as the superuser** in every environment, including the deployed one. All `env/*.env.example` files use `postgres:`, and `compose.prod.yml` shares one `env_file` across `migrate`, `web`, `relay`, and `workflow`. | Superusers bypass RLS unconditionally, so the entire fail-closed foundation is inert. Every check that claims to prove isolation proves it only for `app_rls`, a role the application never uses. |
| D2 | **`libs/shared/idempotency.py` is not tenant-namespaced.** It keys the response cache on the client-supplied `Idempotency-Key` header alone, on all nine services. | A live cross-tenant leak of full response bodies that never touches the database, so RLS cannot mitigate it. Also violates the frozen `nexus:{org_id}:` prefix. |
| D3 | **`grpc_server.py` fails open.** A bare `except Exception` returns a well-formed context echoing the caller's asserted `org_id` with a default tier. | Becomes "database unreachable therefore grant access" the moment it reads `rbac_grants`. |
| D4 | **The outbox relay will drain nothing, silently**, once correct roles are wired: it queries `outbox` with no `app.current_org` set, the fail-closed policy returns zero rows, and `run()` reads that as an empty queue. | Total event loss with no error, no log, and no metric distinguishing it from idle. Same root cause as D1. |
| D5 | **One poison outbox row crash-loops the relay forever.** No per-row error handling, `attempts` never written, no produce-side DLQ. | Head-of-line block across every tenant. |
| D6 | **Seven tenant tables have no foreign key to `organizations`**, because `TenantMixin` adds the column without one. | Orphan `org_id` values are insertable, so right-to-be-forgotten by `org_id` cannot be trusted to find everything. |
| D7 | **`docker-compose.yml` publishes 5432 and 6379; `tests/conftest.py` expects 5433 and 6380.** | Every database test skips silently and reports green. The Plan 0 record's "`make ci` green" is reported, not observed, and was almost certainly skipping. |

## Live gap list

Ordered by what blocks the most downstream work. Items 1, 7, and 8 from the 2026-07-22 list are closed: `mytheus-platform` is cloned and Phase 0F/0G is complete, the deploy decision has [ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md), and both `DECISION-REQUIRED` items are resolved.

1. **`mytheus-ui` has no Clerk and no generated API client.** Blocks Plan 1-2 section 3.10 entirely. The repo holds a 56-page mock-data marketing site plus an app shell whose route groups already match doc 20 exactly. Section 3.10 says "scaffold net-new" and instructs `create-next-app@14` in that directory, which would destroy real work: it must be read as a reconciliation, not a scaffold.
2. **`mytheus-mobile` realign has not begun.** No Clerk Expo SDK, no `src/api/client.ts`, no `REALIGN.md`. Blocks Plan 1-2 section 7.
3. **No `rbac_grants` and no `audit_logs`.** Both are Plan 1-2 tables (sections 3.4 and 3.7). `audit_logs` is the harder one: it is append-only, range-partitioned, and must be live the moment RLS is, so it cannot be deferred past the first authz denial. `grpc_server.py` hardcodes `_DEFAULT_TIER = "member"` until `rbac_grants` exists.
4. **`organizations.clerk_org_id` does not exist.** Required by the resolved tenant-claim decision: it is the join table the edge resolves Clerk's org id through. Specified in section 3.3, not built in Plan 0.
5. **Every service is route-less.** All nine HTTP services boot `/healthz` through `create_app()` and carry a `TODO` where their routers go. No `/api/v1` surface exists yet.
6. **Stale Railway/Render references remain outside the spine.** Canon, ADR-0016, doc 16, `plan/README.md`, and Plan 1-2 are corrected. Still carrying the struck PaaS: reader docs 00, 01, 03, 04, 13, 22, and plan bands 0, 3-4, 5-6, 10-13, 14. Plan 14 needs more than a substitution - its premise is a Railway/Render-to-Kubernetes migration and the source is now the VPS.
7. **Fictional repo maps in four repos.** The "Related repos" sections in `mytheus-intelligence`, `mytheus-realtime`, and `mytheus-platform` (README and CLAUDE.md) and `mytheus-mobile` list repos that do not exist (`mytheus-core`, `mytheus-control-plane`, `mytheus-core-api`, `mytheus-gateway`, `mytheus-workflows`, `mytheus-sdk`, `mytheus-infra`) and disagree with each other on what the CRUD repo is called. The authoritative list is the table in `mytheus-docs/CLAUDE.md`.
8. **The live VPS deploy has not run.** Artifacts and runbook are ready in `mytheus-platform/deploy/`; pushing to the box was held as an outward-facing action awaiting explicit go. Back up PostgreSQL off-box and rehearse a restore before any real tenant data lands.
