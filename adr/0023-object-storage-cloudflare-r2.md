# ADR-0023: Object storage is Cloudflare R2

Every `nexus-*` bucket in [Data Model §6](../_canon/02-data-model.md) has been specified as
"S3-compatible blob storage" since the set was written, and no document ever named the provider.
[Tech Decisions](../_canon/06-tech-decisions.md) locks Postgres, Qdrant, Redis, Redpanda,
Temporal, Clerk, LiveKit, Deepgram, Stripe and LiteLLM by name, and had **no object-storage row
at all**. This ADR closes that gap: the provider is **Cloudflare R2**.

## Related documents

- [_canon/06-tech-decisions.md §2.3](../_canon/06-tech-decisions.md) - amended by this ADR to
  add the object-storage row.
- [_canon/02-data-model.md §6](../_canon/02-data-model.md) - the five `nexus-*` buckets, their
  prefixes and lifecycles, now named as R2.
- [12-compliance-and-privacy.md §6](../12-compliance-and-privacy.md) - data residency, which
  `object_bucket_region` drives and which R2's placement model constrains (see Consequences).
- [17-cost-and-finops.md](../17-cost-and-finops.md) - egress is the cost line this decision
  moves.
- [ADR-0009: Data residency, multi-region](0009-data-residency-multi-region.md) - the residency
  posture this ADR qualifies for the object-storage layer specifically.
- [ADR-0016: Self-hosted VPS MVP substrate](0016-self-hosted-vps-mvp-substrate.md) - the
  precedent for choosing the cheap, operationally simple option at this stage and recording the
  exit.

---

## Status

**Accepted.** Naming the provider changes no bucket name, no key prefix, and no application
code: the S3 API contract every document already states is unchanged, and R2 speaks it.

---

## Context

Five buckets are already frozen in canon: `nexus-recordings`, `nexus-documents`,
`nexus-audit-archive`, `nexus-invoices`, `nexus-exports`. Their contents skew heavily toward
**write-once, read-many, read-from-a-browser**: meeting recordings played back by participants,
invoice PDFs downloaded by admins, export archives pulled on request.

That access shape is what decides this. On S3 and its equivalents the dominant recurring cost of
such a workload is not storage, it is **egress** - and egress is charged per byte leaving the
provider, every time a user plays a recording back. A recordings product that succeeds is a
product whose egress bill grows with engagement, which is the wrong thing for a cost line to
correlate with.

Cloudflare is also **already in the stack** as the CDN and WAF ([Deployment
§16](../16-deployment-and-infrastructure.md), the `edge` Terraform module), so R2 is not a new
vendor relationship, a new contract, or a new subprocessor organisation - only a new service
from an existing one.

## Decision

**Object storage is Cloudflare R2**, addressed through its S3-compatible API.

Three reasons, in order of weight:

1. **Zero egress fees.** This is the decisive one. It removes the cost term that grows with
   usage from the one workload in this system that is inherently read-heavy and
   user-triggered. Storage and operations are still billed; the byte-out charge is not.
2. **No new vendor.** Cloudflare already terminates every request to this product. Adding R2
   adds no procurement, no new DPA counterparty, and one fewer subprocessor organisation than
   choosing a different cloud's object store would.
3. **S3-compatible, so nothing else moves.** Canon says "S3-compatible" in several places and
   every one stays true. The SDK, the `{org_id}/` key prefixes, the lifecycle rules and the
   RTBF prefix-delete all work unchanged.

**Rejected: Amazon S3.** The mature default, with the richer feature set (cross-region
replication, storage classes, per-region buckets). Rejected because egress on the recordings
workload is the cost this decision exists to control, and because it would add a second cloud
vendor purely for blobs.

**Rejected: MinIO self-hosted.** Already used for local development and it works. Rejected for
production because it makes durability our operational problem on a single-node VPS
([ADR-0016](0016-self-hosted-vps-mvp-substrate.md)), and recordings are the one artifact class
we cannot regenerate from anything else.

## Consequences

### The residency caveat, stated plainly

**R2's placement model is not S3's, and this is the one place the swap is not free.** S3 offers
arbitrary per-region buckets. R2 offers a hard **jurisdiction** guarantee for `eu` and *location
hints* elsewhere. So against `data_residency.object_bucket_region`:

| Value | R2 position |
| --- | --- |
| `eu` | **Enforceable.** The EU jurisdiction is a contractual placement guarantee. |
| `us` | Location hint. In practice reliable, not a contractual pin. |
| `ap` | **Best-effort only.** No matching jurisdiction exists, so an APAC pin is a hint. |

A tenant contractually requiring APAC-resident blobs therefore **cannot be served by R2 alone**
today. That is an accepted, disclosed limit, not a solved problem: it is recorded here, in
Data Model §6, and in Compliance §6 so that a future enterprise deal in APAC meets it as a known
constraint rather than a surprise during diligence. The exit if it becomes binding is a
per-jurisdiction provider split for that bucket set, which the S3-compatible contract makes
mechanical.

Note this is narrower than it sounds: it constrains **blobs only**. Postgres, Qdrant and
Redpanda residency are pinned by their own fields (`db_cell`, `qdrant_region`, `region`) and are
untouched by this ADR.

### Everything else

- **Subprocessor register.** Cloudflare must be listed for *stored content*, not only as a
  CDN/WAF transit provider. These are different processing activities and a register that names
  only the latter is incomplete.
- **No cross-region replication.** R2 has no direct S3 cross-region-replication equivalent, so
  the DR story for blobs is bucket-level durability plus, where a tenant's tier demands more, an
  explicit copy job. [14-reliability-and-dr.md](../14-reliability-and-dr.md) already treats
  object storage as asynchronously replicated; that is now an explicit job rather than a
  provider feature.
- **Local development is unchanged.** MinIO stays the dev substrate. It is S3-compatible too,
  which is precisely why the contract was written that way.
