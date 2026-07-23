# 00 Status Overview

Program state as of 2026-07-22. Scope: all eight repos in the Mytheus (NEXUS) program.

## Band status

| Band | Plan file | Status | Note |
|---|---|---|---|
| Plan 0 Scaffold | [plan-0-scaffold-and-contracts.md](../plan/plan-0-scaffold-and-contracts.md) | **DONE WITH GAPS** | Backend, events, intelligence, realtime complete. UI, mobile, platform incomplete. See [01-plan-0-scaffold-and-contracts.md](01-plan-0-scaffold-and-contracts.md). |
| Plan 1-2 Foundation and Tenancy | [plan-1-2-foundation-and-tenancy.md](../plan/plan-1-2-foundation-and-tenancy.md) | **NOT STARTED** (partially pre-empted) | Plan 0 already landed several P2 items early: the Clerk revert, RLS `FORCE` on 11 tables, the outbox relay, gRPC `GetUserContext`. Section "Plan 1-2 pre-empted work" below. |
| Plan 3-4 Meeting and Transcription | [plan-3-4-meeting-and-transcription.md](../plan/plan-3-4-meeting-and-transcription.md) | NOT STARTED | |
| Plan 5-6 Org Context and AI Infra | [plan-5-6-org-context-and-ai-infra.md](../plan/plan-5-6-org-context-and-ai-infra.md) | NOT STARTED | |
| Plan 7-9 Agents and Monetization | [plan-7-9-agents-and-monetization.md](../plan/plan-7-9-agents-and-monetization.md) | NOT STARTED | |
| Plan 10-13 Collab, Integrations, GA | [plan-10-13-collab-integrations-ga.md](../plan/plan-10-13-collab-integrations-ga.md) | NOT STARTED | |
| Plan 14 Scale-Out | [plan-14-scale-out.md](../plan/plan-14-scale-out.md) | NOT STARTED (post-GA, demand-driven) | |

## Repo inventory

Eight repos. The on-disk clone folders use a legacy `reprise-` prefix that predates the Mytheus rename; the canonical repo name is always the `mytheus-` form and is the only form used in prose.

| Canonical repo | GitHub | On-disk folder | Plan 0 state |
|---|---|---|---|
| `mytheus-docs` | `Reprise-AI/reprise-docs` | `reprise-docs/reprise-docs` | Canon, ADRs, diagrams, and the eight band plans complete. |
| `mytheus-backend` | `Reprise-AI/reprise-backend` | `reprise-backend/reprise-backend` | Complete (Phases 0B, 0C). |
| `mytheus-events` | `Reprise-AI/reprise-events` | `reprise-events/reprise-events` | Complete (Phase 0A). |
| `mytheus-intelligence` | `Reprise-AI/reprise-intelligence` | `reprise-intelligence/reprise-intelligence` | Complete (Phase 0D). |
| `mytheus-realtime` | `Reprise-AI/reprise-realtime` | `reprise-realtime/reprise-realtime` | Complete (Phase 0E, realtime half). |
| `mytheus-ui` | `Reprise-AI/reprise-ui` | `reprise-ui/reprise-ui` | **Not done.** Phase 0E web half was deliberately skipped. Large mock-data UI exists instead. |
| `mytheus-mobile` | `Reprise-AI/reprise-mobile` | `reprise-mobile/reprise-mobile` | **Not done.** No Clerk SDK, no `httpClient`, no `REALIGN.md`. |
| `mytheus-platform` | not cloned | `reprise-platform/` holds a second clone of `reprise-events` | **Unknown.** Phase 0F cannot be verified until the real repo is cloned. |

### Clone hygiene issue

`reprise-platform/` contains a duplicate clone of `mytheus-events`, not `mytheus-platform`. Two consequences: `mytheus-platform` Phase 0F is unverifiable locally, and an edit made in the duplicate diverges silently from the real `mytheus-events` working copy. Resolve by removing the duplicate and cloning the real platform repo (or confirming it does not exist yet).

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

## Live gap list

Ordered by what blocks the most downstream work.

1. **`mytheus-platform` is unaccounted for.** Phase 0F (CI workflow, deploy descriptor, `.env` templates) is unverified. Blocks the Plan 1-2 section 3.11 CI security gates, which assume workflow files exist.
2. **`mytheus-ui` has no Clerk and no generated API client.** Blocks Plan 1-2 section 3.10 entirely. The repo instead holds a large mock-data marketing site and app shell that must be reconciled with, not discarded by, the section 3.10 scaffold.
3. **`mytheus-mobile` realign has not begun.** No Clerk Expo SDK, no `src/api/client.ts`, no `REALIGN.md`. Blocks Plan 1-2 section 7.
4. **No `rbac_grants` and no `audit_logs`.** Both are Plan 1-2 tables (sections 3.4 and 3.7). `audit_logs` is the harder one: it is append-only, range-partitioned, and must be live the moment RLS is, so it cannot be deferred past the first authz denial.
5. **Every service is route-less.** All nine HTTP services boot `/healthz` through `create_app()` and carry a `TODO` where their routers go. No `/api/v1` surface exists yet.
6. **Both Plan 1-2 `DECISION-REQUIRED` items are open.** The Clerk `org_id` claim shape and the RBAC-tier vs `users.role` split. The first blocks the `api-gateway` edge extraction and both client repos.
