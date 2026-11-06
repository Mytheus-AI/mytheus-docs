# ADR-0025: `POST /orgs` may create tenants on the request path, gated to platform-operator owners

Migration `a1c4e7f20b18` (`mytheus-backend`, section 3.3 of Plan 1-2) granted the request-path
role `mytheus_app` column-level SELECT on `organizations` and nothing else, with the comment
"a tenant is born through the operator seam, never through a request handler". That made
`POST /orgs`, which [_canon/01-service-catalog.md](../_canon/01-service-catalog.md) assigns to
`identity-tenant-service`, unimplementable: its INSERT is refused by Postgres and the route
answers `501 NOT_IMPLEMENTED`. The user has approved letting that route create tenants. This
ADR supersedes the migration comment's decision, narrowly.

## Related documents

- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md) - `GET/POST /orgs` on
  `identity-tenant-service`; unchanged, this ADR makes the existing assignment buildable.
- [_canon/02-data-model.md section 7 rule 7](../_canon/02-data-model.md) - `organizations` is
  protected by column-level grants rather than a policy. Unchanged: the new grant is
  column-level too.
- [ADR-0002](0002-multi-tenancy-model.md) D3 - the role model. Not changed.
- [ADR-0021](0021-explicit-grants-no-schema-wide-default-privileges.md) - explicit grants.
  This ADR's grant is explicit and asserted.
- [ADR-0010](0010-auth-clerk-enterprise-sso.md) - the outbox for org-lifecycle
  `tenant-events`, which the provisioning path writes.

---

## Status

**Accepted** (approved by the user 2026-11-06). Implemented by `mytheus-backend` migration
`1afe4928b214`. Superseded decision: "tenants are born only through the operator seam
(`mytheus_admin`)", recorded in migration `a1c4e7f20b18` section 5. No canon file stated that
decision, so no canon text changes; canon already assigns `POST /orgs` to this service.

---

## Decision

**D1. Column-level INSERT on the tenant root for the request path.**
`GRANT INSERT (id, name, slug, status, plan_id, clerk_org_id) ON organizations TO mytheus_app`.
Exactly the columns `provision_tenant` writes. Nothing wider: no UPDATE, no DELETE, no INSERT
on `created_at`/`deleted_at` (defaults only), and the SELECT carve-out is unchanged, so a
blanket `SELECT *` is still refused. Suspension, renaming and deletion stay on the operator
seam. CI asserts the exact INSERT column list and the absence of UPDATE/DELETE.

**D2. The route, not the role, decides who may provision.** `POST /orgs` requires
`require_tier("owner")` in a tenant listed in `PLATFORM_OPERATOR_ORG_IDS`. Unset or empty
means nobody can provision, the same reachable surface as before. Self-serve signup (a caller
with no org claim creating their first tenant) stays an open decision and is not made here.

**D3. `PLATFORM_OPERATOR_ORG_IDS` is a PROPOSED name, flagged for the user.** It is a new
environment variable (comma-separated internal `org_id` UUIDs). No canon file names
configuration variables, so it is recorded here. If the user prefers another name or a
database-backed operator list, only the route and this ADR change.

## Consequences

- **Positive.** `POST /orgs` works and the `tenant-events` provisioned path runs outside
  hand-seeded tests.
- **Negative, accepted.** A compromised request path can now insert tenant roots. Bounded: it
  cannot modify or delete any existing tenant, cannot repoint a live `clerk_org_id` (UNIQUE,
  and the route refuses to link an existing row), and a stray org is inert until a Clerk
  organization resolves to it.
- **Watch item.** Read-modify-write on `organizations` (suspend, rename) must not drift onto the
  request path by the same route. That would need its own ADR.
