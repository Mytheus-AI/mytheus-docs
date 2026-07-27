# Data residency decision

P1 design artifact (Plan 1-2 section 3.1 step 8). Derived from [02-system-context.md](../02-system-context.md) section 6.3, [12-compliance-and-privacy.md](../12-compliance-and-privacy.md), and [_canon/02-data-model.md](../_canon/02-data-model.md).

## 1. The decision

| Question | Answer |
|---|---|
| Region set | `us`, `eu`, `ap`. Confirmed as the frozen `data_residency.region` enum. No fourth region. |
| MVP default region | **`us`** |
| MVP deployment | **Single region.** One self-hosted VPS ([ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md)). |
| Is the table populated anyway? | **Yes.** Every tenant gets a `data_residency` row at provisioning, even though every row says `us` today. |

## 2. Why populate a table that has one value

Because the migration from single-region to multi-region is then a data change, not a schema change.

The alternative is to add residency when the second region is needed, which means backfilling a required column across every existing tenant, retrofitting the routing logic, and discovering at that moment which stores were never keyed by region. Standing the table up now with one value costs a single insert per tenant and makes the later move mechanical.

This is the same reasoning as `audit_logs` landing at P2 rather than P13: compliance-by-construction means the structure exists before the obligation arrives, not after.

## 3. The row and what each column pins

```
data_residency
  org_id                PK, references organizations(id)
  region                'us' | 'eu' | 'ap'    CHECK constrained
  db_cell               which physical Postgres cell holds the tenant
  qdrant_region         which Qdrant region holds the tenant's collections
  object_bucket_region  which bucket region holds the tenant's prefixes
  locked_at             once set, region is immutable without a migration workflow
```

MVP values for every tenant: `region = 'us'`, `db_cell = 'pool-us-1'`, `qdrant_region = 'us'`, `object_bucket_region = 'us'`, `locked_at = NULL`.

`db_cell` says `pool-us-1` rather than naming the VPS because the pooled long tail is the posture; a dedicated cell is an enterprise upgrade path, not the MVP shape.

**`locked_at` is deliberately null at MVP.** Locking a region is a commitment we cannot yet honour with one box. It is set when a tenant contracts for residency, and from that moment the region is immutable without running the region-migration workflow.

## 4. What single-region actually means for compliance

Being precise, because "we support EU residency" is a claim with legal weight.

| Claim | True at MVP? |
|---|---|
| The region set is modelled and per-tenant | Yes |
| A tenant's data is pinned to its recorded region | **No.** All data is in one region regardless of the row. |
| We can honour an EU-residency contract | **No.** Do not sell one until a second region exists. |
| The move to real residency is mechanical | Yes, that is the point of the table |

Until a second region is deployed, `data_residency.region` is a **declaration of intent**, not an enforced control. It must not be represented to a customer as enforced. The enforcement point arrives when subprocessor calls are routed to the matching regional endpoint and the tenant's stores are physically placed per the row.

## 5. Subprocessor routing, when it arrives

Each external processor that touches T4 data must be called at its regional endpoint matching the tenant's `region`, so transcript-class data does not leave the residency zone. Where cross-border transfer is unavoidable it is covered by Standard Contractual Clauses. See [subprocessors.md](subprocessors.md).

None of these are integrated in Plan 1-2, so nothing routes yet. The contract is recorded now so the integrating band inherits it rather than inventing it.

## 6. Related

- [12-compliance-and-privacy.md](../12-compliance-and-privacy.md) for the GDPR and CCPA posture.
- [rtbf-contract.md](rtbf-contract.md) for deletion, which is region-aware by construction.
- [ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md) for why there is exactly one region today.
