# ADR-0018: Consent entity landed schema-first, behind a flag defaulted off

This ADR ratifies the recording/transcription consent schema that
[12-compliance-and-privacy.md §3.3](../12-compliance-and-privacy.md) already described as a
proposed extension: `tenant_settings.consent_required` and
`meeting_participants.consented_at`/`consent_version`. The schema and the token-mint 428
mechanism it gates are built now; the mechanism ships **defaulted off**, so no tenant's join
behavior changes until that tenant's `consent_required` flag is explicitly set to `true`. This
resolves the DECISION-REQUIRED item raised in Plan 3-4 section 8.

## Related documents

- [_canon/02-data-model.md §2.1, §2.2](../_canon/02-data-model.md) — the two frozen tables this
  ADR amends: `meeting_participants` (PDF-original) and `tenant_settings` (SaaS-scale addition).
- [12-compliance-and-privacy.md §3.3](../12-compliance-and-privacy.md) — the reader-doc section
  that already designed the token-mint 428 flow against this schema as a proposal; propagated to
  cite this ADR as ratifying it.
- [ADR-0010: Auth - Clerk enterprise SSO](0010-auth-clerk-enterprise-sso.md) — owns the
  token-mint endpoint (`POST /api/meetings/:id/token`) this consent check gates.
- [Tech Decisions §2.7](../_canon/06-tech-decisions.md) — the locked token-mint endpoint this ADR
  adds a precondition to.

---

## Status

**Accepted, schema and mechanism design only.** This ADR ratifies the data model and the shape of
the 428 gate. It does **not** authorize writing the migration or any application code — those are
later tasks that depend on this one landing in canon first. The behavior stays inert (every
tenant's `consent_required` is `false`) until a tenant is explicitly flipped to `true`, which this
ADR records as a legal/compliance decision, not an engineering one.

---

## Context

[12-compliance-and-privacy.md §3.3](../12-compliance-and-privacy.md) identified a genuine gap:
NEXUS records and transcribes meetings, several jurisdictions require an affirmative, recorded
consent acknowledgment before a participant is transcribed, and canon had no consent entity at
all. That section designed a mechanism against the token-mint boundary that already gates meeting
join (`POST /api/meetings/:id/token`, owned by `meeting-service`) — the participant cannot receive
a join token until they acknowledge the meeting is recorded and transcribed by AI agents — but
flagged the whole design as "a PROPOSED extension, not settled canon," because it required adding
columns to two tables canon had frozen: `meeting_participants` (PDF-original, frozen to exactly
`meeting_id`, `user_id`, `role_in_meeting`, PK `(meeting_id, user_id)`) and `tenant_settings`
(SaaS-scale, frozen to `default_locale`, `agent_confidence_overrides`,
`retention_days_transcripts`, `retention_days_worklogs`, `pii_redaction_enabled`, `sso_config`).

### Why land the schema now instead of waiting

Both tables are load-bearing and already in active use by other Plan 3-4 and later work
(`meeting_participants` drives `get_user_context`; `tenant_settings` drives per-tenant retention
and confidence overrides). Reopening either table to add a column later, under the time pressure
of an actual compliance deadline or a customer's legal team asking "where is consent recorded,"
is materially riskier than adding two nullable columns and one defaulted boolean now, while the
tables are still being actively developed against. The columns are inert until used: nullable
`consented_at`/`consent_version` and a `false`-defaulted `consent_required` change nothing about
existing behavior.

### Why the mechanism, not just the schema

Landing only the columns without also ratifying the 428 mechanism shape would leave the "how do
these columns get populated" question open, inviting a second ad hoc design later. Ratifying both
now, while keeping the flag off, gets the shape settled without forcing early rollout.

### Why defaulted off, and why that is a legal decision

Turning consent enforcement on changes what participants must do to join a meeting for a given
tenant. Whether that requirement, its wording, its versioning, and its jurisdictional scope are
correct is a legal/compliance determination specific to each tenant's regulatory exposure, not
something engineering should decide by shipping a default. Defaulting `consent_required` to
`false` means no tenant is affected by this ADR landing; a tenant is only affected when someone
with the authority to make that legal determination flips the flag for that tenant.

---

## Options considered

| Option | Verdict |
|---|---|
| **Defer the schema entirely until the migration/application-code task** | Rejected. Leaves the "PROPOSED extension" gap in [12 §3.3](../12-compliance-and-privacy.md) open indefinitely and risks the two frozen tables being reopened later under real time pressure once a tenant's legal team asks for consent capture. |
| **Land schema and application code together, flag defaulted on** | Rejected. Changes join behavior for every tenant immediately, which is exactly the kind of default this program does not get to set unilaterally — flipping consent enforcement on is a legal/compliance call per tenant, not an engineering rollout decision. |
| **Land schema and the 428 mechanism shape now, defaulted off; migration and application code as later tasks** (chosen) | Accepted. Closes the frozen-table gap while it is still cheap, keeps behavior inert for every tenant until a legal/compliance decision explicitly flips it, and gives the later migration/application-code tasks a settled schema to build against instead of a proposal. |

---

## Decision

[_canon/02-data-model.md](../_canon/02-data-model.md) is amended to add:

- `tenant_settings.consent_required boolean NOT NULL DEFAULT false`
- `meeting_participants.consented_at timestamptz NULL`
- `meeting_participants.consent_version text NULL`

The token-mint 428-Precondition-Required mechanism described in [12
§3.3](../12-compliance-and-privacy.md) is ratified as the design these columns support: when a
tenant's `consent_required` is `true` and a participant has no matching `consented_at` for the
current `consent_version`, `POST /api/meetings/:id/token` returns 428 instead of minting a join
token, until the participant acknowledges.

**Out of scope for this ADR and this task:** the Alembic migration adding these columns, and any
`meeting-service` application code implementing the 428 check. Both are later tasks that depend on
this ADR and the canon amendment landing first.

---

## Consequences

### Accepted costs

- Two previously-frozen tables now carry columns with no consumer yet. This is deliberate: the
  cost is paid once, now, while both tables are already under active development, rather than
  paid again later as a reopening.
- Every future reader of `meeting_participants`/`tenant_settings` must know these columns are
  inert until a tenant's flag is set — [12 §3.3](../12-compliance-and-privacy.md) and the Glossary
  entries for both tables carry that caveat forward.

### Benefits realized

- The "PROPOSED extension, not settled canon" gap in [12
  §3.3](../12-compliance-and-privacy.md) is closed: the schema is now canon, cited rather than
  proposed.
- The migration and application-code tasks that depend on this one have a settled, ADR-recorded
  schema to build against instead of having to re-litigate the shape.
- No tenant's meeting-join behavior changes as a result of this ADR landing.

### What is unchanged

Every other column on `meeting_participants` and `tenant_settings`, the token-mint endpoint's
existing behavior for every tenant with `consent_required = false` (i.e., every tenant today), and
the rest of [12-compliance-and-privacy.md](../12-compliance-and-privacy.md)'s DSR/GDPR/CCPA
machinery.

### Follow-ups this ADR creates

1. Write the Alembic migration adding the three columns (later task, depends on this ADR).
2. Implement the `meeting-service` token-mint 428 check and the consent-acknowledgment write path
   (later task, depends on this ADR and the migration).
3. Propagate the ratified schema to [12-compliance-and-privacy.md
   §3.3](../12-compliance-and-privacy.md), replacing "PROPOSED extension" language with a citation
   to this ADR.
4. Update the Glossary entries for `meeting_participants` and `tenant_settings` to list the new
   columns.
5. A tenant-facing decision, owned outside engineering, on which tenants and jurisdictions get
   `consent_required` flipped to `true`, and on what `consent_version` text/versioning scheme is
   used.
