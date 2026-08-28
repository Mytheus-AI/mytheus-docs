# ADR-0021: Tables in the shared schema are granted explicitly; no schema-wide default privilege

The `public` schema grants nothing on new TABLES by default. Every migration that creates a
tenant table issues its own GRANT, and no partition ever carries one. This closes a defect
class that produced a live cross-tenant read, write and delete, and a second arm of the same
class that had gone unnoticed because every assertion guarding it was scoped to the three
database roles this program creates rather than to the property worth holding.

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md) section 7 rule 8 - amended by this ADR
  to state the grant rule alongside the policy rule.
- [ADR-0002: Multi-tenancy model](0002-multi-tenancy-model.md) D3 - owns the role model and the
  "no `BYPASSRLS` in the request path" mandate. This ADR does not change it; it records that
  the mandate as asserted was narrower than the mandate as written.
- [06-multi-tenancy.md](../06-multi-tenancy.md) section 4 - the six isolation layers, of which
  RLS is the first and the one this concerns.
- [11-security-architecture.md](../11-security-architecture.md) - the fail-closed posture.
- [threat-model/stride.md](../threat-model/stride.md) - the elevation-of-privilege row whose
  stated control this corrects, and the defect log this adds to.
- [work-done-till-now/03-plan-3-4-meeting-and-transcription.md](../work-done-till-now/03-plan-3-4-meeting-and-transcription.md)
  section 3 and open gaps 5 and 6 - where the recorded arm of this was found and left open.

---

## Status

**Accepted, and implemented.** `mytheus-backend` migration `9b2c4d1e6f30`, 2026-08-28. Written
alongside the code rather than after it.

---

## Context

### The recorded arm

`mytheus-backend` migration `a1c4e7f20b18` ends with a schema-wide default privilege:

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO mytheus_app, mytheus_admin;
```

Its stated reason is good: without it, a table added by a later migration is unreadable by the
application until somebody remembers a GRANT, and the symptom is a production permission error
rather than a failing test.

The consequence is not. PostgreSQL checks privileges against the relation **named** in a query,
and enforces a partitioned parent's row-security policy only on access **through the parent**.
A partition created by `CREATE TABLE ... PARTITION OF` is a newly created table, so it inherits
the default grant, and its own row-security state is off. `DELETE FROM audit_logs_2026_07`
therefore succeeded for `mytheus_app` while `DELETE FROM audit_logs` was correctly refused: an
append-only audit trail erasable one partition at a time. `transcript_chunks` hit the same
thing for real in Plan 3-4, where a live cross-tenant read, write and delete was reproduced
empirically before being closed.

Both were closed one table at a time. The cause was not, and Plan 3-4 recorded it as "the
single highest-leverage fix outstanding" precisely because it would recur on the next
partitioned table created by any repo against this database.

### The unrecorded arm, found while fixing the first

`supabase/postgres` installs its own default privilege at bootstrap, granting **ALL** on every
new table in `public` to `anon`, `authenticated` and `service_role`. So every table this
program has ever created - partition or not, sensitive or not - also granted the full DML set
to those three. `service_role` holds `rolbypassrls`. `authenticator` is a LOGIN role and is a
member of all three, so "none of them can log in" is not the control it appears to be.

Verified against the running database before the fix, with no `app.current_org` set at any
point:

```sql
SET ROLE service_role;
SELECT count(*) FROM audit_logs;           -- 352, every tenant
SELECT count(*) FROM audit_logs_2026_08;   -- 346, every tenant, through the partition
DELETE FROM audit_logs_2026_08 WHERE ...;  -- DELETE 1
```

That is defect D1's shape on a role nobody in this program created: the append-only guarantee
on `audit_logs` is a GRANT and was bypassed, the `organizations` column-level carve-out was
bypassed, and every `tenant_isolation` policy was inert for anything able to reach that role.
It was live on the deployed VPS, which runs the same image.

**Why nothing caught it.** Four CI assertions guard this area and all four passed throughout.
Each was scoped to `mytheus_app`, `mytheus_relay` and `mytheus_admin` - the roles this program
creates - because that is where the defect was found the first time. None of them could see a
role the database image ships. Canon rule 7's assertion is worded the same way: "no
**request-path role** holds `rolsuper` or `rolbypassrls`". The property that actually matters
is about the table, not about our roster: nothing holding `BYPASSRLS` should reach a tenant
table, whoever created it.

---

## Options considered

**(a) A `ddl_command_end` event trigger that auto-revokes on every new relation.** The first
option Plan 3-4 named. **Rejected as unavailable**, not as wrong: PostgreSQL requires SUPERUSER
to `CREATE EVENT TRIGGER`, and on `supabase/postgres` the migration runner `postgres` is not a
superuser - the image's own `10000000000000_demote-postgres.sql` demotes it, `supabase_admin`
is the only superuser, and `postgres` is not a member of it. A migration cannot install it. It
is also the weaker of the two even where it is available, because it revokes a grant after the
fact rather than preventing it being made, leaving a window and a dependency on the trigger
staying enabled.

**(b) Retire the schema-wide default privilege on TABLES and grant explicitly.** Chosen. The
grant is never made, so a partition from any repo comes out with an empty ACL by construction,
with no trigger to maintain and no privilege we do not hold.

**(c) Keep the default privilege and continue revoking per partition.** The status quo. It
depends on whoever adds the next partition remembering a convention that lives in a different
repo from the migration they are writing - which is exactly how `transcript_chunks` acquired
the defect after `audit_logs` had already been fixed for it.

**(d) Narrow the default privilege to SELECT, or to one role.** Does not help. A cross-tenant
SELECT on a transcript partition is the disclosure that matters; narrowing the verb narrows the
blast radius without closing the class.

---

## Decision

**D1. No schema-wide default privilege grants on TABLES in `public`.** Any role that creates
tables in this schema grants nothing on them by default. Asserted in CI against
`pg_default_acl`, scoped to roles that actually own a table in the schema - so the image's
`supabase_admin` entry, which the migration runner cannot alter, is treated as inert while it
owns nothing here and begins failing the assertion the moment that stops being true.

**D2. Every migration that creates a tenant table issues its own GRANT.** One call to
`migrations/_grants.grant_tenant_table`, placed with the RLS policy it belongs beside. This
applies to every repo whose migrations write to this schema, not only `mytheus-backend`.

**D3. A partition carries no direct grant to any role.** Access is through the parent, where
both the grant and the policy live. Asserted against `pg_class.relacl` for every partition, and
stated as "no grantee other than the relation's owner", not as a list of roles.

**D4. No role holding `rolbypassrls` holds any privilege on a tenant table.** Independent of
who created the role. This is the generalisation of ADR-0002 D3's mandate from the roles we
define to the property we need, and it is a standing CI assertion rather than a one-time
cleanup, because the image's superuser can re-grant at any time.

**D5. The lost convenience is replaced by a failing test, not by discipline.** ADR-0002 D3 and
migration `a1c4e7f20b18` both defend the default privilege on the grounds that its absence
turns a forgotten GRANT into a production permission error. That objection is accepted as
correct and answered directly: a CI assertion fails the build for any table carrying `org_id`
that the request role cannot read. The forgotten GRANT now fails at merge.

---

## Consequences

**Positive.** The partition defect class is closed at its cause and for every repo sharing the
schema, rather than one table at a time. The `audit_logs` append-only guarantee and the
`organizations` column-level carve-out are enforced for the first time against every role
rather than against three of them. The four guards in this area now assert properties instead
of role lists, which is what let the second arm hide behind them.

**Negative, accepted.** Adding a tenant table now takes one more line. Mitigated by D5: the
missing line is a failing test rather than a production error, which is a better failure than
the one the default privilege was avoiding.

**Cross-repo obligation.** `mytheus-realtime`'s `transcript_chunks` migration (`be66698dbefb`)
contains no GRANT and relies on the retired default privilege. Nothing breaks on a database
where that table already exists, because existing grants are untouched. A rebuild from an empty
cluster in which that migration runs after this one produces "permission denied for table
`transcript_chunks`" - a loud, immediately diagnosable failure on the disaster-recovery and
first-install paths, fixed by one `GRANT` line in that repo. It was deliberately not applied
from `mytheus-backend`: granting on another service's table across a repo boundary is the
ownership violation this program's rules exist to prevent, and it would be invisible to anyone
reading the repo that owns the table.

**Two residuals, recorded rather than claimed closed.**

1. `service_role` still holds `rolbypassrls`. `ALTER ROLE ... NOBYPASSRLS` needs superuser,
   which the migration runner does not have. What was removed is its reach, not the attribute.
2. The image's `ALTER DEFAULT PRIVILEGES FOR ROLE supabase_admin` survives for the same reason.
   It is inert while `supabase_admin` creates no table in `public`. The realistic way it stops
   being inert is an operator running DDL from inside the container, where `psql -U
   supabase_admin` is the path of least resistance; any table created that way is born granting
   the full DML set to a BYPASSRLS role. D1's assertion is scoped so that this begins failing
   at that moment.

**A note on the base image.** Both residuals exist because the substrate is `supabase/postgres`
rather than plain `postgres`, which is a hangover from the managed-Supabase era that
[ADR-0016](0016-self-hosted-vps-mvp-substrate.md) superseded. The program no longer uses
Supabase Auth, PostgREST, or the Supabase API, so `anon`, `authenticated`, `service_role`,
`authenticator` and `supabase_admin` exist solely as image artifacts with no consumer. Moving
to a plain `postgres` image would remove both residuals outright and give the migration runner
its superuser back, which would also make option (a) available as belt and braces. That is a
substrate change with its own migration and rehearsal cost and is not decided here; it is
recorded as the exit path.
