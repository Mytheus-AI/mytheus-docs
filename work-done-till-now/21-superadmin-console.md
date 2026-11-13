# 21. The superadmin console: a complete /ops

Status: `DONE WITH GAPS` (built and tested locally on 2026-11-13; committed to `main` of mytheus-docs, mytheus-backend and mytheus-ui, not pushed or deployed). Not a plan band. Decisions: [ADR-0046](../adr/0046-superadmin-ops-console.md). Canon: `support_sessions` and `tenant_migrations` in [_canon/02-data-model.md](../_canon/02-data-model.md), the new `/admin/ops/*` routes in [_canon/01-service-catalog.md](../_canon/01-service-catalog.md).

## Why

The owner asked for `/ops` to be a complete superadmin console: everything doc 18 section 8 lists, plus an overview, a per-workspace view, service health and the staff themselves, with charts. They also asked how anyone becomes a superadmin. Plan 12 had built the directory, suspend and reinstate, flags, costs and the audit viewer; tenant migrate and support impersonation were specified but never built.

## How someone becomes a superadmin

Unchanged from ADR-0037 D5, now written down in ADR-0046 D1 and on the Staff page: the person signs in to Mytheus once; `uv run python -m scripts.staff_lookup <email>` in mytheus-backend prints their `users.id`; that id is appended to `MYTHEUS_STAFF_USER_IDS` in admin-ops-service's environment and the service restarted. There is no button for it on purpose.

## What landed

### mytheus-docs (`7edd864`)

- `adr/0046-superadmin-ops-console.md`: the role, the rules every page keeps, the page list, support sessions, move requests.
- Canon rows for `support_sessions` and `tenant_migrations`; the routes in the service catalog; doc 18 section 8, the glossary, the README ADR index and the traceability matrix.

### mytheus-backend (`8e63994`..`e99b0d1`)

- Migration `b6c1d2e3f4a5`: `support_sessions` and `tenant_migrations` (canonical tenant policy, explicit grants), and `ops_tenant_rollup(weeks, period)`, a SECURITY INVOKER function that binds `app.current_org` to each live workspace in turn and returns per-workspace numbers only (plan, placement, members, meetings by week, tokens, quota, unpublished outbox rows). It restores the caller's binding at the end.
- `libs/shared/support_session.py`, `libs/shared/authz.py`: a token with an `act` claim (Clerk impersonation) is honoured only while the workspace holds an open `support_sessions` row for exactly that staff member and member, unexpired, whose `impersonate` grant is still active; otherwise 401. `RequestContext` carries the session.
- `libs/shared/audit.py`: during a support session every `audit_logs` row is written with `actor_type = 'system'` and `metadata.support_session`. No route has to remember it.
- `libs/shared/clerk_admin.py`: `create_actor_token` and `revoke_actor_token` against Clerk's Backend API.
- `services/admin_ops_service/console_routes.py`: `GET /admin/ops/overview`, `GET /admin/ops/health`, `GET /admin/ops/staff`, `POST /admin/ops/tenants/detail`, `POST /admin/ops/tenants/migrate`, `POST /admin/ops/tenants/migrations/cancel`, `POST /admin/ops/support/members`, `POST /admin/ops/support/sessions`, `POST /admin/ops/support/sessions/end`. Staff only, every call in `ops_audit_log`.
- `tenant_routes.py`: `GET /admin/tenant/support-sessions` for workspace admins; creating a support grant is refused inside a support session.
- `scripts/staff_lookup.py`; `tests/test_admin_ops_console.py` (11 tests); the OpenAPI spec regenerated.

### mytheus-ui (`c3ebec3`..`7dbbb9b`)

- Tabs: Overview, Workspaces, Feature flags, Cost and lag, Health, Audit log, Staff.
- `/ops` is now the Overview: six figures, new workspaces per week, meetings per week, workspaces by plan, the ten workspaces closest to their token quota, ops activity per day. The directory moved to `/ops/workspaces`.
- `components/ops/charts.tsx`: `ColumnChart`, `BarList`, `Meter` in plain HTML on the tokens, no chart library. One series each, so no legend; history in `--pill`, the current period in `--signal`; hover and keyboard focus show a tooltip; each column chart carries a visually hidden table.
- `components/ops/WorkspacePanel.tsx`: opened from a workspace's name. Facts, meetings per week, tokens against quota, support access (grants, open sessions, start a session, the one-time sign-in link, end), region and move requests, flags, recent ops actions.
- `/ops/health` and `/ops/staff` (the allowlist, activity chart, how to add someone, the last 50 actions).
- `SupportSessionBanner` in the app shell: shown on every page while Clerk reports an actor.
- `/workspace/support` lists the workspace's support sessions, and the Act as support help says what it allows.
- `tests/ops-console.test.ts`.

## Decisions and why

- Overview numbers come from one database function rather than a Python loop: the local database has about 4,500 workspaces left by test runs, and a loop of three round trips per workspace took seconds. The function is SECURITY INVOKER, so RLS still checks every row; a SECURITY DEFINER read would have been an RLS bypass on the request path, which doc 06 rules out.
- The workspace detail is a panel loaded with a POST, not a `/ops/workspaces/[id]` page, because `org_id` never goes in a path or query.
- Impersonation uses Clerk actor tokens (Clerk's own primitive), but the server does the time-boxing: Clerk's session lifetime is not trusted, the `support_sessions` row is.
- Tenant moves are recorded, not executed. Mytheus runs one cell (`pool-us-1`), so there is nowhere to move a workspace to, and `data_residency` stays identity-tenant-service's to write. The console says so.
- Staff emails come from each person's own `ops.whoami` rows, which already record their workspace, instead of a cross-tenant lookup of `users`.

## Verified

- `uv run pytest tests/test_admin_ops_console.py tests/test_admin_ops_api.py` against a fresh `postgres:15` (the CI image): 25 passed. The one failure, `test_set_organization_status_is_reachable_only_through_identity`, compares paths with `/` and fails on Windows before this work too.
- `uv run ruff check`, `black --check` and `mypy` on the changed backend modules: clean. `scripts/check_authz.py`: passed.
- mytheus-ui: `npx tsc --noEmit` and ESLint clean on the changed files; `npm test` 553 passed, 2 failed (`tests/skeletons.test.ts`, Windows path separators in other routes, not ops).
- The migration applied on a fresh database from scratch.

## Gaps

- Not deployed, and not looked at in a browser: the local UI talks to `api.mytheus.cloud`, which does not have the new routes until the backend is pushed.
- The local development database (`mytheus` on port 5443) is behind head and cannot be upgraded because one leftover `oauth_tokens` test row fails a foreign key; deleting it was not done.
- Support sessions have not been tried against real Clerk. Clerk's impersonation needs to be enabled for the instance; without a working key the start call answers 503 and records nothing.
- DLQ counts and consumer lag are still not readable from admin-ops-service.
- `next build` and the Playwright suite were not run.
