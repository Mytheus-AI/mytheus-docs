# ADR-0046: The superadmin console: who is staff, what /ops must show, support sessions and tenant moves

## Related documents

- [18-saas-business-architecture.md](../18-saas-business-architecture.md) section 8: the internal ops console capability list. Expanded here into pages.
- [ADR-0037](0037-plan-12-integrations-follow-ups-and-consoles-build.md) D5 (staff are an allowlist) and D8 (support access needs a live grant). Both kept; D8 is extended to impersonation.
- [_canon/02-data-model.md](../_canon/02-data-model.md): `support_sessions` and `tenant_migrations`, added in the same change.
- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md): the new `/admin/ops/*` routes of `admin-ops-service`.
- [06-multi-tenancy.md](../06-multi-tenancy.md): no RLS-bypass role on the request path. Kept: nothing here bypasses RLS.

## Status

**Accepted** (2026-11-13).

## Context

Mytheus has two admin levels. A **workspace admin** (a customer's `owner` or `admin`) runs one organisation from `/workspace/*`. A **superadmin** (Mytheus staff, called "internal ops" in the docs) looks after the whole SaaS from `/ops`. Plan 12 built the first half of `/ops`: the tenant directory, suspend and reinstate, feature flags, the cost table and the audit viewer. Two capabilities from section 8 were never built (tenant migrate, support impersonation), and the console had no overview, no per-workspace page, no service health and no view of the staff themselves. The owner asked for `/ops` to be a complete superadmin console, with charts.

## Decision

### D1. Who is a superadmin

A superadmin is a Mytheus user whose internal `users.id` is listed in `MYTHEUS_STAFF_USER_IDS` on `admin-ops-service` (ADR-0037 D5, unchanged). There is no role, flag or button that makes someone staff from inside the product. That is deliberate: granting the most privileged access in the system is a deployment change, made by someone who can already edit the server's secrets, and it cannot be escalated to from a compromised workspace admin account.

To add someone:

1. They sign up and sign in to Mytheus once, so their `users` row exists.
2. Look up their `users.id`: `uv run python -m scripts.staff_lookup <email>` in `mytheus-backend` (read-only; runs as `mytheus_admin` through `ADMIN_DATABASE_URL`).
3. Append the id to `MYTHEUS_STAFF_USER_IDS` (comma-separated) in the admin-ops-service env file (`/etc/dokploy/mytheus-secrets/` on the VPS, `.env` locally) and restart `admin-ops-service`. The list is read on every request.
4. They open `/ops`. Anyone not on the list gets a 404.

Removing someone is the same edit in reverse. The Staff page (D3) shows who is on the list and what each person did.

### D2. Rules every /ops page keeps

1. **Deny by default.** `GET /admin/ops/whoami` must answer `{staff: true}` or the page is a 404.
2. **Everything is audited.** Every `/admin/ops/*` call, reads and denials included, writes one `ops_audit_log` row in the same transaction as its work.
3. **Aggregates, not content.** Cross-workspace views are counts and sums over control-plane tables (`organizations`, `users`, `subscriptions`, `plans`, `data_residency`, `usage_aggregates`, `agent_budgets`, `meetings` counted by week, `outbox` counted by topic). No page selects a meeting's title, a transcript, a decision or an action.
4. **RLS still applies.** Per-workspace aggregates are read with `app.current_org` bound to that workspace, one workspace at a time. Where that is done for every workspace at once, it runs inside one SECURITY INVOKER function (`ops_tenant_rollup`) that binds each workspace in turn: the database still checks every row against the policy, and only numbers come back. There is no RLS-bypass role or SECURITY DEFINER read.
5. **No `org_id` in a path or query.** A target workspace travels in a request body, so the workspace detail is `POST /admin/ops/tenants/detail`, and in the UI it opens as a panel rather than a URL.
6. **Customer consent for anything inside a workspace.** Reading a workspace's audit log, listing its members or acting as one of them needs an active `support_grants` row from that workspace (ADR-0037 D8). Acting as a member needs a grant with scope `impersonate`.

### D3. The pages of /ops

| Page | What it must show | Backend |
|---|---|---|
| **Overview** (`/ops`, the landing page) | Totals: workspaces, active, suspended, members, meetings in the last 7 days, agent tokens this billing period. Charts: new workspaces per week (12 weeks), meetings per week (12 weeks), workspaces by plan, the ten workspaces using the most tokens against their quota. Ops activity per day (14 days). | `GET /admin/ops/overview` |
| **Workspaces** (`/ops/workspaces`) | The tenant directory (plan, members, status, region and cell), search, suspend and reinstate with a reason. Clicking a row opens the workspace detail panel. | `GET /admin/ops/tenants`, existing suspend/reinstate |
| **Workspace detail** (panel) | Plan, status, created date, residency (region, database cell, vector and bucket region, whether it is locked), member count, meetings per week (12 weeks), tokens against quota this period, the feature flags that apply (workspace and global), support grants with their scope and expiry, support sessions, move requests, and the last 20 ops actions on this workspace. Actions: suspend or reinstate, request a move, start a support session. | `POST /admin/ops/tenants/detail` |
| **Feature flags** (`/ops/flags`) | Existing page, unchanged. | existing |
| **Costs** (`/ops/costs`) | Existing table, plus a chart of the top workspaces by share of quota. | existing |
| **Health** (`/ops/health`) | Each backend service's `/healthz` probed from `admin-ops-service`: up or down, and response time. Outbox backlog by topic: unpublished rows, the oldest one's age, the highest retry count. DLQ counts and consumer lag stay "not connected": they live in Redpanda and Prometheus, which this service cannot reach. | `GET /admin/ops/health` |
| **Audit** (`/ops/audit`) | Existing viewer (one workspace at a time, needs a grant), unchanged. | existing |
| **Staff** (`/ops/staff`) | Every id on the allowlist with their email and workspace (resolved from their own `ops.whoami` calls; someone who has never opened the console shows as "not seen yet"), last activity, actions in the last 30 days, a 14-day activity chart, and the 50 most recent ops actions across all staff. The page also says how to add or remove someone (D1). | `GET /admin/ops/staff` |

### D4. Support sessions (impersonation)

1. A workspace owner or admin creates a support grant with scope `impersonate` for a named staff member at `/workspace/support` (existing, at most 30 days).
2. Staff open the workspace detail. With an active `impersonate` grant they can list the workspace's members (`POST /admin/ops/support/members`) and start a session as one of them (`POST /admin/ops/support/sessions`, with a reason and a length of 15 to 60 minutes, never past the grant's expiry). Staff users cannot be impersonated.
3. `admin-ops-service` writes a `support_sessions` row, then asks Clerk for an **actor token** (Clerk's impersonation primitive) for the member, with the staff member as the actor. The response is a one-time sign-in link, which staff open in a private window. Clerk's session token then carries an `act` claim naming the staff member.
4. **The server enforces the time box, not the link.** When `tenant_context` sees an `act` claim it requires an open `support_sessions` row in that workspace for that actor and that member, unexpired, not ended, whose grant is still active. Otherwise the request is a 401. So ending the session, revoking the grant or reaching `expires_at` cuts access on the next request, whatever Clerk's session lifetime is.
5. **Every action is tagged.** During a support session, every `audit_logs` row is written with `actor_type = 'system'` and `metadata.support_session` = `{id, staff_user_id}` (the shared `audit()` helper does this from the request context, so no route has to remember). The web app shows a banner for the whole session saying who is acting.
6. Staff end a session with `POST /admin/ops/support/sessions/end`, which stamps `ended_at` and revokes the actor token. Workspace admins see their workspace's sessions at `/workspace/support` (`GET /admin/tenant/support-sessions`).

### D5. Tenant move requests

1. `POST /admin/ops/tenants/migrate` records a request to move a workspace to another region (`us`, `eu`, `ap`) and database cell, with a reason, in `tenant_migrations` (status `requested`). One open request per workspace; `POST /admin/ops/tenants/migrations/cancel` cancels it.
2. The move itself is **not automated**. Mytheus runs one cell (`pool-us-1`) today, so there is nowhere to move a workspace to. The residency-migration workflow in section 8 stays future work; until it exists, a request is the ops record that a customer asked for a move and why, and the console says so plainly. `data_residency` stays written only by `identity-tenant-service`.

## Consequences

- `/ops` becomes eight surfaces with one gate and one audit trail.
- Two new tenant-scoped tables (`support_sessions`, `tenant_migrations`), both owned by `admin-ops-service`, both under the canonical RLS policy.
- `tenant_context` gains one indexed lookup, only for requests whose token carries an `act` claim.
- Impersonation depends on Clerk actor tokens. Without a real `CLERK_SECRET_KEY` the start call answers 503 and nothing is recorded as started.
- The overview's rollup visits every workspace inside one database call. That is fine at today's scale; past a few thousand workspaces it should become a scheduled snapshot.
