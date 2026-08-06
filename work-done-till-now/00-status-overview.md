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
| Plan 3-4 Meeting and Transcription | [plan-3-4-meeting-and-transcription.md](../plan/plan-3-4-meeting-and-transcription.md) | **DONE WITH GAPS** | `meeting-service`, `core-api` chat/speakers/consent-gate, `realtime-gateway`, and `transcription-service` built and tested against real infra. `mytheus-ui` (sections 3.4, 3.8) and `mytheus-mobile` (section 7) deliberately not built, out of scope by decision. Every check needing a live LiveKit/Deepgram/S3 credential is BLOCKED. Two ADRs ratified (0017 partition key, 0018 consent). See [03-plan-3-4-meeting-and-transcription.md](03-plan-3-4-meeting-and-transcription.md). |
| Plan 5-6 Org Context and AI Infra | [plan-5-6-org-context-and-ai-infra.md](../plan/plan-5-6-org-context-and-ai-infra.md) | **IN PROGRESS** | Nine of ten tasks done against real, live infrastructure (Postgres, Redis, Redpanda, Schema Registry, Qdrant, MinIO; only GPU packaging is faked/BLOCKED). `identity-tenant-service` org context, the Redis-cached `get_user_context()`, the seeder, `bff` aggregation, the event backbone (verified GA, no rebuild), `embedding-service`, `reranker-service`, and `ingestion-service` are built. Task 10 (`search-service`) and the whole-band final review had not finished as of writing. See [04-plan-5-6-org-context-and-ai-infra.md](04-plan-5-6-org-context-and-ai-infra.md). |
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
3. **`Idempotency-Key` reuse with a different body replays instead of returning `409` - and the blast radius is wider than previously recorded.** `libs/shared/idempotency.py` keys purely on `nexus:{org_id}:idem:{key}`, with no route, method, or path component, so the real exposure is cross-endpoint collision (any two different mutating endpoints reusing the same key value), confirmed during Plan 3-4. The plan's `IDEMPOTENCY_KEY_REUSED` acceptance check remains unmet; the enum value exists in code and is unused.
4. **CLOSED in Plan 5-6.** ~~Tenant provisioning has no reachable path.~~ `identity-tenant-service/provisioning.py::provision_organization()` (task 4) is now the first real org-creation path: an org row and a `tenant-events` outbox row in one transaction, confirmed live with `published_at` set by the ambient relay. See [04-plan-5-6-org-context-and-ai-infra.md](04-plan-5-6-org-context-and-ai-infra.md) section 1.2.
5. **`spectral lint` and `oasdiff` are not wired**, and the OpenAPI specs in `design/openapi/` are hand-written rather than generated from the code. Now that real routes exist, the specs can drift from them silently.
6. **The outbox relay has drained real `meeting-events` since Plan 3-4**, via `meeting-service`'s live `meeting-events` consumer. Still undrained: `transcription-service`'s `meeting-room-events` consumer has real dedupe and session logic but no live `confluent-kafka` I/O loop wired (see gap 11).
7. **Stale Railway/Render references remain outside the spine.** Canon, ADR-0016, doc 16, `plan/README.md`, and Plan 1-2 are corrected. Still carrying the struck PaaS: reader docs 00, 01, 03, 04, 13, 22, and plan bands 0, 3-4, 5-6, 10-13, 14. Plan 14 needs more than a substitution - its premise is a Railway/Render-to-Kubernetes migration and the source is now the VPS.
8. **Fictional repo maps in four repos.** The "Related repos" sections in `mytheus-intelligence`, `mytheus-realtime`, and `mytheus-platform` (README and CLAUDE.md) and `mytheus-mobile` list repos that do not exist (`mytheus-core`, `mytheus-control-plane`, `mytheus-core-api`, `mytheus-gateway`, `mytheus-workflows`, `mytheus-sdk`, `mytheus-infra`) and disagree with each other on what the CRUD repo is called. The authoritative list is the table in `mytheus-docs/CLAUDE.md`.
9. **The live VPS deploy has not run.** It is **no longer blocked by D1**: the roles, the per-process credential split, and the runbook are all ready in `mytheus-platform/deploy/`. Pushing to the box remains an outward-facing action awaiting explicit go. Back up PostgreSQL off-box and rehearse a restore before any real tenant data lands.
10. **Platform-level partition-grant leak, found in Plan 3-4, not durably fixed.** `mytheus-backend` migration `a1c4e7f20b18`'s schema-wide `ALTER DEFAULT PRIVILEGES` means any table or partition created by any repo against the shared Postgres silently inherits full DML grants for the application roles unless its own migration explicitly revokes. `transcript_chunks` hit this for real (a live cross-tenant read/write/delete via direct partition access, empirically reproduced and closed per-table); the schema-wide root cause is unfixed and will recur on the next partitioned table from any repo. See [03-plan-3-4-meeting-and-transcription.md](03-plan-3-4-meeting-and-transcription.md) section 3.
11. **`transcription-service`'s `meeting-room-events` consumer has no live I/O loop.** Dedupe and session-open/close logic is real and unit-tested; nothing drives it from the real topic yet.
12. **The compliance gate's substantive conditions are unmet**: no Deepgram DPA/SCC signed, LiveKit Cloud not yet in the subprocessor register, zero-retention flag unset. The engineering control (`COMPLIANCE_GATE_GREEN = False`, unbypassable) is correct and blocks every Deepgram stream regardless.
13. **NER redaction and the per-tenant reversible token vault are not implemented.** `transcription-service` ships a deterministic regex/validator redaction layer only (email, phone, IBAN-Luhn, government ID).
14. **`mytheus-ui` room/captions UI (sections 3.4, 3.8) and `mytheus-mobile` section 7 remain deliberately unbuilt**, blocking the full browser-plus-phone meeting demo described in Plan 3-4. Same out-of-scope decision as `mytheus-ui`'s dashboard-only reconciliation in Plan 1-2.
15. **The shared dev Postgres is ahead of `main`.** `mytheus-backend`'s `main` checkout cannot run any alembic operation against the shared dev database until the Plan 3-4 branch merges (`Can't locate revision '25e7cede5672'`); services still run fine since they only query tables.
16. **`Topic` node seeding into `nexus_knowledge` has no legitimate writer anywhere in the program**, found in Plan 5-6. `knowledge-graph-service` declares itself the graph's sole writer and is a bare four-line scaffold with zero routes. Two independent Plan 5-6 tasks hit this and correctly refused to build a second writer. See [04-plan-5-6-org-context-and-ai-infra.md](04-plan-5-6-org-context-and-ai-infra.md) section 4.
17. **`transcript_chunks` likewise has no writer a sibling repo can legitimately use**, found in Plan 5-6: `transcription-service` exposes no REST, gRPC, or event write path, only its own in-process pipeline. Blocks seeding realistic transcript content for the embedding tier outside a live meeting.
18. **GPU packaging is not built for `embedding-service` or `reranker-service`**, so every SLO that assumes GPU hardware (batch-32 sub-100 ms embedding, sub-500 ms warm rerank) is unmeasured on the hardware it targets; both were measured honestly on CPU only and marked BLOCKED, never claimed.
19. **`embedding-service`'s "only writer of Qdrant" property is a code convention, not an infrastructure control** - the dev Qdrant instance has no API key configured at all. Likely `mytheus-platform` scope to close.
20. **No team-membership write API exists in the product.** Plan 5-6's seeder is currently the only writer of `team_members`, which is why that write was ruled acceptable rather than a one-writer violation; the underlying product gap (no route to add or remove a team member) remains open.
21. **`mytheus-mobile` section 7 (person-profile screen) was never assigned to a Plan 5-6 task and remains unbuilt**, unlike `mytheus-ui`'s explicitly recorded exclusion for the same band. Needs an explicit scoping decision before a future task picks it up.
