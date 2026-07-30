# 00 Status Overview

Program state as of 2026-07-30. Scope: all eight repos in the Mytheus (NEXUS) program.

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
| Plan 1-2 Foundation and Tenancy | [plan-1-2-foundation-and-tenancy.md](../plan/plan-1-2-foundation-and-tenancy.md) | **DONE WITH GAPS. Every section built; no end-to-end run yet.** | P1 landed 2026-07-27 (canon amendments plus the seventeen-file design pack). Five P2 sessions on 07-29 and 07-30 closed all seven blocking defects and built every section: 3.2, 3.2b, 3.3 to 3.9 and 3.11 in `mytheus-backend`, section 7 in `mytheus-mobile`, and 3.10 in `mytheus-ui`. Three least-privilege database roles, the tenant-root carve-out, `rbac_grants`, `audit_logs`, the deny-by-default authz gate, meetings CRUD, the Clerk webhook, the `bff` dashboard, CI in two repos, and both clients on the real contract. **What remains is verification, not construction:** no run against a live Clerk instance plus a deployed gateway. See [02-plan-1-2-foundation-and-tenancy.md](02-plan-1-2-foundation-and-tenancy.md) sections 7 to 15. |
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
| 3.4 identity tables | Done as of 2026-07-30. `organizations`, `users`, `tenant_settings`, `data_residency` and `rbac_grants` all exist with their canonical columns. |
| 3.4 step 5 gRPC `GetUserContext` | Done, and no longer a stub: it reads the real tier from `rbac_grants` through a LEFT JOIN, and returns `""` rather than a default for a user holding no grant. |
| 3.5 step 3 outbox relay | Done. `libs/shared/relay.py` (`FOR UPDATE SKIP LOCKED`, flush-then-mark, per-row poison isolation, DLQ) plus `services/relay/main.py`, now running as `mytheus_relay` under the `outbox_relay_drain` policy. |
| 3.6 `meetings` / `meeting_participants` tables | Tables exist with their full canonical columns and RLS. **Still no CRUD routes.** |
| 3.7 `audit_logs` | Done as of 2026-07-30, ahead of the plan's ordering. Partitioned, RLS-policied, append-only by GRANT, with `libs/shared/audit.py`. Not yet wired to any caller, because there are no routes to wire it to. |

## Blocking defects, found 2026-07-27, ALL CLOSED 2026-07-30

These were defects in code that already existed, each confirmed by reading the named source, and **none appeared in any plan file**. They were taken as the first work of Plan 1-2 P2, ahead of any feature, because until they were fixed nothing in the program could be honestly verified. Full detail in [02-plan-1-2-foundation-and-tenancy.md](02-plan-1-2-foundation-and-tenancy.md) sections 7 and 8.

| # | Defect | Closed by |
|---|---|---|
| D1 | **The application connected to PostgreSQL as a role that bypasses RLS** in every environment, including the deployed one, so the entire fail-closed foundation was inert. Every check that claimed to prove isolation proved it only for `app_rls`, a role the application never used. | Migration `a1c4e7f20b18` creates `mytheus_app` / `mytheus_relay` / `mytheus_admin`, none holding `rolsuper` or `rolbypassrls`; `scripts/bootstrap_roles.py` sets their passwords and re-asserts the property; `mytheus-platform` splits the `env_file` per process so `web` cannot read the owner credential. Two CI assertions keep it closed. |
| D2 | **`libs/shared/idempotency.py` was not tenant-namespaced**: a live cross-tenant leak of full response bodies that never touched the database, so RLS could not mitigate it. | Re-keyed to the frozen `nexus:{org_id}:idem:{key}`; a request with no resolvable tenant is passed through uncached. |
| D3 | **`grpc_server.py` failed open**: a bare `except Exception` returned a well-formed context echoing the caller's asserted `org_id` with a default tier. | Each condition now returns its own status (`INVALID_ARGUMENT` / `NOT_FOUND` / `UNAVAILABLE`) and gives the caller nothing. |
| D4 | **The outbox relay would have drained nothing, silently**, once correct roles were wired: total event loss with no error, no log, and no metric distinguishing it from idle. | One permissive `outbox_relay_drain` policy on `outbox` for `mytheus_relay` alone, not a BYPASSRLS role, so the cross-tenant reach stays visible in `pg_policies`. |
| D5 | **One poison outbox row crash-looped the relay forever**, a head-of-line block across every tenant. | Per-row isolation, `attempts` actually written, and dead-lettering to `{topic}.dlq` after `MAX_ATTEMPTS`. |
| D6 | **Seven tenant tables had no foreign key to `organizations`**, so orphan `org_id` values were insertable and RTBF could not be trusted to find everything. | Foreign keys added `ON DELETE RESTRICT`; pre-existing orphan rows deleted with a logged count (see section 8.1 of the band file). |
| D7 | **`docker-compose.yml` published 5432/6379 while `tests/conftest.py` expected 5433/6380**, so every database test skipped silently and reported green. | Ports aligned; `make ci` genuinely green and observed, 61 passed / 1 skipped against a database rebuilt from an empty volume. |

**One further defect was found while fixing these, by the new CI assertion**: partitions of `audit_logs` inherited the full DML grant from `ALTER DEFAULT PRIVILEGES`, so `DELETE FROM audit_logs_2026_07` succeeded for `mytheus_app` while `DELETE FROM audit_logs` was correctly refused. An append-only audit trail erasable one partition at a time. Closed in the same migration and now asserted.

## Live gap list

Ordered by what blocks the most downstream work. Closed since 2026-07-27: `rbac_grants` and `audit_logs` exist with RLS; `organizations.clerk_org_id` is built and resolved through `libs/shared/org_resolver.py`; every HTTP service has its routes; CI runs in `mytheus-backend` and `mytheus-events`; and [threat-model/stride.md](../threat-model/stride.md) is up to date with the fixes.

1. **No end-to-end run exists.** Both clients are wired to the contract and both build clean, but nothing has been exercised against a live Clerk instance and a deployed gateway together. Section 3.10's two acceptance checks are specifically the ones outstanding: two different-tenant users seeing disjoint data, and a network capture confirming the browser never sends `org_id`. The property is structurally enforced (no parameter for it exists in either client) and unit-tested on mobile, but "structurally enforced" is not "observed".
2. **No generated TS client in either client repo.** The OpenAPI specs in `design/openapi/` are hand-written rather than emitted from the FastAPI app, so generating from them would encode drift rather than prevent it. Both clients keep a small hand-written types file as the single place drift can hide. Closing this properly means emitting the specs from code first.
3. **`Idempotency-Key` reuse with a different body replays instead of returning `409`.** The middleware caches by key without fingerprinting the body, so the plan's `IDEMPOTENCY_KEY_REUSED` acceptance check is unmet. The enum value exists in code and is unused.
4. **Tenant provisioning has no reachable path.** `POST /api/v1/orgs` is deferred by decision, and the operator seed script standing in for it has not been written. Nothing can create a tenant today except a manual INSERT as `mytheus_admin`.
5. **`spectral lint` and `oasdiff` are not wired**, and the OpenAPI specs in `design/openapi/` are hand-written rather than generated from the code. Now that real routes exist, the specs can drift from them silently.
6. **The outbox relay has never drained a real event.** The routes write `meeting-events` rows; nothing has consumed them, because the real-bus round-trip is the one skipped test in the suite.
7. **Stale Railway/Render references remain outside the spine.** Canon, ADR-0016, doc 16, `plan/README.md`, and Plan 1-2 are corrected. Still carrying the struck PaaS: reader docs 00, 01, 03, 04, 13, 22, and plan bands 0, 3-4, 5-6, 10-13, 14. Plan 14 needs more than a substitution - its premise is a Railway/Render-to-Kubernetes migration and the source is now the VPS.
8. **Fictional repo maps in four repos.** The "Related repos" sections in `mytheus-intelligence`, `mytheus-realtime`, and `mytheus-platform` (README and CLAUDE.md) and `mytheus-mobile` list repos that do not exist (`mytheus-core`, `mytheus-control-plane`, `mytheus-core-api`, `mytheus-gateway`, `mytheus-workflows`, `mytheus-sdk`, `mytheus-infra`) and disagree with each other on what the CRUD repo is called. The authoritative list is the table in `mytheus-docs/CLAUDE.md`.
9. **The live VPS deploy has not run.** It is **no longer blocked by D1**: the roles, the per-process credential split, and the runbook are all ready in `mytheus-platform/deploy/`. Pushing to the box remains an outward-facing action awaiting explicit go. Back up PostgreSQL off-box and rehearse a restore before any real tenant data lands.
