# ADR-0046: The superadmin console: who is staff, what /ops must show, support sessions, tenant moves and feedback

## Related documents

- [18-saas-business-architecture.md](../18-saas-business-architecture.md) section 8: the internal ops console capability list. Expanded here into pages.
- [ADR-0037](0037-plan-12-integrations-follow-ups-and-consoles-build.md) D5 (staff are an allowlist): superseded by D1 here, the allowlist now holds workspace-less Clerk accounts. D8 (support access needs a live grant) is kept and extended to impersonation.
- [_canon/02-data-model.md](../_canon/02-data-model.md): `support_sessions` and `tenant_migrations`, added in the same change.
- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md): the new `/admin/ops/*` routes of `admin-ops-service`.
- [06-multi-tenancy.md](../06-multi-tenancy.md): no RLS-bypass role on the request path. Kept: nothing here bypasses RLS.

## Status

**Accepted** (2026-11-13).

## Context

Mytheus has two admin levels. A **workspace admin** (a customer's `owner` or `admin`) runs one organisation from `/workspace/*`. A **superadmin** (Mytheus staff, called "internal ops" in the docs) looks after the whole SaaS from `/ops`. Plan 12 built the first half of `/ops`: the tenant directory, suspend and reinstate, feature flags, the cost table and the audit viewer. Two capabilities from section 8 were never built (tenant migrate, support impersonation), and the console had no overview, no per-workspace page, no service health and no view of the staff themselves. The owner asked for `/ops` to be a complete superadmin console, with charts.

## Decision

### D1. Who is a superadmin

A superadmin is a **dedicated Clerk account that belongs to no workspace**, whose Clerk user id is listed in `MYTHEUS_STAFF_CLERK_USER_IDS` on `admin-ops-service`. This supersedes ADR-0037 D5's `MYTHEUS_STAFF_USER_IDS` (a list of `users.id`): a staff account has no `users` row, because `users` rows exist only inside a workspace.

1. **Staff accounts are separate accounts.** Someone who also uses Mytheus as a customer keeps a personal account for that and signs up a second, staff-only account.
2. **A workspace member can never be staff.** An account is refused as staff, even when listed, if its session carries an organization or it has a `users` row in any workspace (`clerk_user_in_any_tenant`, a SECURITY INVOKER check that binds each workspace in turn and returns only true or false). The lookup script refuses such an account before it is ever listed.
3. **Staff see only `/ops`.** A staff account signs up and stops on the choose-organization step. Once listed, it lands on `/ops` from that step, and every app page sends it back to `/ops`. A support session (D4) is different: the session is the member's, not the staff account's.
4. **Staff ID.** Each staff account has a fixed Staff ID, a UUIDv5 of its Clerk user id. It is what a workspace admin enters in a support grant, what `ops_audit_log.staff_user_id` records, and what the console shows the staff member.

To add someone:

1. They sign up for a new Mytheus account and stop at "join or create an organization".
2. Run `uv run python -m scripts.staff_lookup <email>` in mytheus-backend (on the server: inside the admin-ops-service container). It finds the account in Clerk, refuses it if it is in any organization or workspace, and prints its Clerk user id and Staff ID.
3. Append the Clerk user id to `MYTHEUS_STAFF_CLERK_USER_IDS` (comma-separated) in admin-ops-service's environment and recreate that service.
4. They refresh the page and land on `/ops`. Anyone else gets a 404 there.

The gateway lets a signed-in account with no organization reach `/admin/ops/*` only (verified, no tenant resolved, no tenant headers attached); admin-ops-service then applies the rules above on every call. A global feature flag change emits no `admin-events` record, because the topic is partitioned by `org_id` and a staff account has none; it is still in `ops_audit_log`.

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
| **Feedback** (`/ops/feedback`) | Every bug report and piece of feedback members sent from Settings (D6), newest first, filterable by state, with the workspace and the reporter's email; mark seen or resolved. | `GET /admin/ops/feedback`, `POST /admin/ops/feedback/status` |
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

### D6. Bug reports and feedback reach staff

1. Any signed-in member can send Mytheus a bug report or general feedback from Settings (`POST /admin/tenant/feedback`, tier `member`): a kind (`bug` or `feedback`), a message of up to 4,000 characters, and the page it is about if they give one. The reporter's email is stored with it, so staff can answer without reading the workspace's `users`. At most 20 reports per person per day (429 past that).
2. A member sees their own reports and whether staff have seen or resolved them (`GET /admin/tenant/feedback`).
3. Staff read every report in a Feedback inbox at `/ops/feedback` (`GET /admin/ops/feedback`) and mark each `seen` or `resolved` (`POST /admin/ops/feedback/status`). This is the one cross-workspace read of user-written text in `/ops`, and it is allowed because the member wrote it to Mytheus; it goes through the same `app.ops_console` policy as `feature_flags`, and every read and change is in `ops_audit_log`. The overview counts new reports.
4. `feedback_reports` is owned by `admin-ops-service`. A workspace session may insert its own rows and read them; only a transaction with `app.ops_console` on may update one.

## Consequences

- `/ops` becomes eight surfaces with one gate and one audit trail.
- Three new tables owned by `admin-ops-service`: `support_sessions` and `tenant_migrations` under the canonical RLS policy, and `feedback_reports` under a tenant insert/read policy plus the `ops_console` policy.
- `tenant_context` gains one indexed lookup, only for requests whose token carries an `act` claim.
- Impersonation depends on Clerk actor tokens. Without a real `CLERK_SECRET_KEY` the start call answers 503 and nothing is recorded as started.
- The overview's rollup visits every workspace inside one database call. That is fine at today's scale; past a few thousand workspaces it should become a scheduled snapshot.
