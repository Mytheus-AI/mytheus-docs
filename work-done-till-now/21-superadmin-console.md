# 21. The superadmin console: a complete /ops

Status: `DONE WITH GAPS` (built and tested on 2026-11-13, including bug reports and feedback; pushed to `main` of mytheus-docs, mytheus-backend and mytheus-ui and deployed to the VPS the same day). Not a plan band. Decisions: [ADR-0046](../adr/0046-superadmin-ops-console.md). Canon: `support_sessions`, `tenant_migrations` and `feedback_reports` in [_canon/02-data-model.md](../_canon/02-data-model.md), the new `/admin/ops/*` and `/admin/tenant/*` routes in [_canon/01-service-catalog.md](../_canon/01-service-catalog.md).

## Why

The owner asked for `/ops` to be a complete superadmin console: everything doc 18 section 8 lists, plus an overview, a per-workspace view, service health and the staff themselves, with charts. They also asked how anyone becomes a superadmin. Plan 12 had built the directory, suspend and reinstate, flags, costs and the audit viewer; tenant migrate and support impersonation were specified but never built.

## How someone becomes a superadmin

Changed on the owner's request later the same day (ADR-0046 D1, `e2d9947`): a superadmin is a **separate Clerk account that belongs to no workspace**, and it sees only `/ops`.

1. Sign up for a new Mytheus account and stop at "join or create an organization".
2. On the server, in the admin-ops-service container: `python -m scripts.staff_lookup <email>`. It finds the account in Clerk, refuses it if it belongs to any organization or Mytheus workspace, and prints the Clerk user id and the Staff ID.
3. Append the Clerk user id to `MYTHEUS_STAFF_CLERK_USER_IDS` in admin-ops-service's environment and recreate that service.
4. Refresh the page: it sends a staff account to `/ops`, and every app page sends it back there.

A workspace member is refused as staff on every call even if listed: when its session carries an organization, or `clerk_user_in_any_tenant` finds a live `users` row for it. There is no button for any of this on purpose.

### The org-less staff change (`ca22ecf`..`7518a7b` in mytheus-backend, `d9671d5`..`9d87b11` in mytheus-ui)

- `libs/shared/auth.py`: `verify_token_without_org`, used only by admin-ops-service's staff dependency and the gateway's `/admin/ops/` branch; personal access tokens are refused there.
- Migration `d8e3f4a5b6c7`: `clerk_user_in_any_tenant(clerk_user_id)`, SECURITY INVOKER, binds each workspace in turn, returns a boolean only.
- `services/admin_ops_service/common.py`: `MYTHEUS_STAFF_CLERK_USER_IDS`, `StaffPrincipal`, `staff_id_for` (UUIDv5 Staff ID), `staff_refusal` (`not_listed`, `workspace_session`, `workspace_member`), logged with each denial. `whoami` answers any session and returns `staff_id` to staff. A global flag change no longer emits `admin-events` (no `org_id` to partition by). The own-org suspend guard is gone: staff have no workspace.
- `services/api_gateway/edge.py`: `/admin/ops/*` is verified without an organization and forwarded with no tenant headers.
- The Staff page reads names and emails from Clerk (`clerk_admin.user_summary`); `scripts/staff_lookup.py` now asks Clerk by email.
- mytheus-ui: the API client sends a pending session's token; the middleware lets a pending session into `/ops` and `/admin`; the choose-organization page and the app shell send a staff account to `/ops`; the console header has Sign out instead of Back to app; the Staff page shows each account's Clerk id and Staff ID, flags a listed account that is in a workspace, and shows your own Staff ID.
- Tests: 62 passed across the admin-ops and gateway suites (new: a listed member is refused in both session shapes; malformed list entries grant nothing; an org-less session reaches only `/admin/ops/`); 89 passed across the auth, token, self-serve, policy and RLS suites.

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

### Bug reports and feedback (ADR-0046 D6), added on the owner's request the same day

- mytheus-docs (`8512ef6`): ADR-0046 D6, the `feedback_reports` canon row, the routes, glossary and doc 18.
- mytheus-backend (`eaff1f3`..`18f039b`): migration `c7d2e3f4a5b6` (`feedback_reports`: a workspace session may insert and read its own rows through `tenant_insert` and `tenant_isolation`; only the `ops_console` policy may update); `feedback_routes.py` with `POST/GET /admin/tenant/feedback` (any member, 20 per person per day, audited as `feedback.submitted`) and `GET /admin/ops/feedback`, `POST /admin/ops/feedback/status` (staff, in `ops_audit_log`); the overview counts new reports; 4 more tests.
- mytheus-ui (`05772d6`..`64c2fc6`): Settings gains "Report a bug" (`/settings/feedback`): "Report a bug" and "Send feedback" buttons open one dialog (kind, message up to 4,000 characters, where in Mytheus), and the page lists what the person has sent with its state. `/ops/feedback` is the staff inbox with New, Seen, Resolved and All, the workspace and the reporter's email on each report, and Mark seen, Resolve, Reopen. The overview's sixth figure is new feedback.

## Decisions and why

- Overview numbers come from one database function rather than a Python loop: the local database has about 4,500 workspaces left by test runs, and a loop of three round trips per workspace took seconds. The function is SECURITY INVOKER, so RLS still checks every row; a SECURITY DEFINER read would have been an RLS bypass on the request path, which doc 06 rules out.
- The workspace detail is a panel loaded with a POST, not a `/ops/workspaces/[id]` page, because `org_id` never goes in a path or query.
- Impersonation uses Clerk actor tokens (Clerk's own primitive), but the server does the time-boxing: Clerk's session lifetime is not trusted, the `support_sessions` row is.
- Tenant moves are recorded, not executed. Mytheus runs one cell (`pool-us-1`), so there is nowhere to move a workspace to, and `data_residency` stays identity-tenant-service's to write. The console says so.
- Feedback is the one cross-workspace read of text a member wrote, and it is allowed because they wrote it to Mytheus. It uses the same `app.ops_console` seam as `feature_flags` rather than a new role, and the reporter's email is copied onto the row at sending time so staff never read another workspace's `users`.
- Staff emails come from each person's own `ops.whoami` rows, which already record their workspace, instead of a cross-tenant lookup of `users`.

## Verified

- `uv run pytest tests/test_admin_ops_console.py tests/test_admin_ops_api.py` against a fresh `postgres:15` (the CI image): 29 passed, feedback included. The one failure, `test_set_organization_status_is_reachable_only_through_identity`, compares paths with `/` and fails on Windows before this work too.
- `uv run ruff check`, `black --check` and `mypy` on the changed backend modules: clean. `scripts/check_authz.py`: passed.
- mytheus-ui: `npx tsc --noEmit` and ESLint clean on the changed files; `npm test` 553 passed, 2 failed (`tests/skeletons.test.ts`, Windows path separators in other routes, not ops).
- The migration applied on a fresh database from scratch.
- The charts and the feedback card were rendered with sample data on a temporary local page (deleted, never committed) and looked at in the browser pane, in dark mode: columns, the coral current period, axis labels, hover tooltips, the quota colours and the bar lists. This caught a real defect: `weekPoints`, `dayPoints` and `fmt` were exported from the `"use client"` chart module, so the server-rendered Overview and Staff pages would have thrown. They moved to `src/lib/ops-charts.ts` (`bed00cd`).

- Deploy: backend CI run `43459491583` ran 1,670 tests, 1,669 passed; `test_health_probes_services_and_reports_the_outbox` failed because it expected its workspace in the top ten backlogs while other tests leave events behind. The test now checks its workspace through `ops_tenant_rollup` as `mytheus_app`, and the overview test gives its workspace 180% of quota so it always leads (`6bd7c19`). Run `43460382242`: every job green, deployed. UI run `43459634170`: quality, build and deploy green.
- Live: `https://api.mytheus.cloud/healthz` 200; `/admin/ops/overview` answers 401 unauthenticated (the route exists). Signed in as the owner, `dashboard.mytheus.cloud/ops` shows the 404 page: the owner's `users.id` is not yet in `MYTHEUS_STAFF_USER_IDS` on the VPS.

## Gaps

- The owner is not on the production staff list yet, so `/ops` has not been seen with live data. Adding them is a VPS edit, given to the owner as commands.

- Light mode was not looked at.
- The local development database (`mytheus` on port 5443) is behind head and cannot be upgraded because one leftover `oauth_tokens` test row fails a foreign key; deleting it was not done.
- Support sessions have not been tried against real Clerk. Clerk's impersonation needs to be enabled for the instance; without a working key the start call answers 503 and records nothing.
- DLQ counts and consumer lag are still not readable from admin-ops-service.
- `next build` and the Playwright suite were not run.

## The console redesign: a sidebar, more chart types and a map (2026-11-14)

The owner found `/ops` unprofessional to look at and asked for a full redesign of its UI, for chart types beyond bar charts, and for maps. Three choices were put to them and taken as recommended: a small backend field so the overview can map workspaces, a left sidebar instead of the tab row, and hand-built SVG charts instead of a chart library.

### mytheus-backend (`caeb244`)

- `GET /admin/ops/overview` gains `regions`: one row per `data_residency.region` with `workspaces`, `active`, `members`, `meetings_7d` and `cells` (each `db_cell` with its count). `ops_tenant_rollup` already returned `region` and `db_cell`, so this is grouping in Python (`_regions` in `console_routes.py`) with no migration. A workspace with no residency row counts under `unplaced`, so the rows always add up to `totals.tenants`.
- `design/openapi/admin-ops-service.yaml` regenerated. A pure test of `_regions` (grouping, the `unplaced` bucket, largest-first order) and region assertions on the existing overview test.

### mytheus-ui (`4be7de2`, `00a1012`)

- Frame: `OpsShell` and `OpsNav` in `components/ops/`. A 256px sidebar grouped Platform (Overview, Workspaces, Health, Cost and lag), Controls (Feature flags, Staff) and Records (Audit log, Feedback), with Sign out and the theme toggle at its foot; below `lg` a header with the sections as a sideways row. `OpsTabs` is removed. Each section page now carries its own h1.
- Chart kit in `components/ops/viz/`: an area chart with crosshair and arrow-key reading, sparklines, a donut, a calendar heatmap, a squarified treemap, a latency dot plot, quota meters with 80% and 95% ticks, and `KpiStrip`. Every chart keeps a visually hidden table or prints its numbers.
- Map in `components/ops/map/`: `world-dots.ts` is a 144 by 55 dot grid baked once from world-atlas `land-110m` (Natural Earth, public domain), drawn as a single SVG path. `RegionMap` puts a hub on each region (`us`, `eu`, `ap`), sized by workspaces and labelled with its code and count; a requested move draws the coral line from the current region to the target.
- Pages. Overview: KPI strip with trends, weekly trend switching between meetings and new workspaces, plan donut, the map beside a region list (hovering either highlights both), quota meters, a 14-day console activity heatmap, and a Refresh button. Health: status banner, KPI strip, every service on one latency scale. Cost and lag: a treemap of token share coloured by quota state, then every workspace's meter. Staff: KPI strip, daily calls as an area chart, an inline bar per person. The workspace panel shows the residency map with any requested move. Workspaces, Feature flags, Audit log and Feedback take the new header. Loading skeletons follow the new layouts.
- `styles/ops.css`: two one-hue scales scoped to `.mx-ops`, plan ordinal (`--ord-1..4`, warm ink) and activity heat (`--heat-1..4`, coral), with chosen light and dark steps, plus `--land` for the map.

### Decisions and why

- The sidebar is drawn by `ops/layout.tsx` after the staff gate, not by the `(admin)` layout. A non-staff visitor gets a 404 that names nothing, as before; the `(admin)` loading screen draws only bones while the gate is checked.
- No chart or map library. The charts read the design tokens directly, so light and dark come free, and the console adds no bundle weight. The map needs only three hubs, which a baked dot grid serves without a projection library.
- The trend chart shows one series at a time behind a switch, never meetings and sign-ups on two axes.
- Treemap colour states the quota (inside, past 80%, past 95%), never identity, and the legend names each fill.
- Region hubs sit at a point inside each region. The console knows the region code and cell, nothing finer, so no city or data centre is implied.
- Both new scales were checked with the dataviz palette validator in ordinal mode against the card surface of each theme; the first light-end steps failed the 2:1 floor and were darkened until they passed.

### Verified

- mytheus-backend: the `_regions` test passes when run directly (the module skips itself without Docker, which was not running); `ruff`, `black --check`, `scripts/emit_openapi --check` clean on the rebased tree; `mypy` reports only the two Windows-only `os.geteuid` errors in `libs/shared/crypto/providers.py`, untouched here.
- mytheus-ui: `npx tsc --noEmit` and `npx next lint` clean after rebasing on main; `npm test` 633 of 635 passed, the 2 failures being the same `tests/skeletons.test.ts` cases that fail on the untouched code (routes outside ops); `next build` compiled and generated all 109 pages.
- Rendered with sample data through a temporary loader and middleware patch (reverted, never committed) and screenshotted at 1440 and 390 wide, light and dark: overview, health, costs, staff, the workspace panel with a requested US to EU move, and map hover. The first round caught a truncated KPI figure, a clipped donut legend, an axis top far above the data, a wrapping tick label and an overflowing down-service note; all were fixed and rechecked.
- The Impeccable detector reported only advisory font-size notes (13, 14 and 15px are not on the DESIGN.md ramp, though its prose uses them). No blocking findings.

### Gaps

- The map and region list need the backend deploy; until then the overview says region counts come with the next admin-ops-service deploy.
- DESIGN.md does not yet list the ops chart scales or the 13 to 15px sizes the app uses.
- Playwright and axe were not run on the new pages.
