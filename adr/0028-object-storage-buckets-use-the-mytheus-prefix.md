# ADR-0028: Object storage buckets use the `mytheus-` prefix, not `nexus-`

Canon named the six object-storage buckets `nexus-recordings`, `nexus-documents`, `nexus-attachments`,
`nexus-invoices`, `nexus-exports` and `nexus-audit-archive`. The maintainer created the Cloudflare R2
buckets as `mytheus-recordings`, `mytheus-documents` and `mytheus-attachments` and decided the code and
canon follow those names. This ADR records the rename and supersedes the bucket names in earlier ADRs.

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md) - the object-storage bucket table, updated in the same change.
- [ADR-0023](0023-object-storage-cloudflare-r2.md) - Cloudflare R2 as the object store. Its bucket names are now the `mytheus-` forms below.
- [Glossary and Naming](../24-glossary-and-naming.md) - records the `old -> new` rename.

---

## Status

**Accepted** (decided by the maintainer 2026-11-06). Existing ADRs and the `work-done-till-now/` records keep the
`nexus-` names as the record of what was true when they were written.

## Decision

| Old name | New name |
|---|---|
| `nexus-recordings` | `mytheus-recordings` |
| `nexus-documents` | `mytheus-documents` |
| `nexus-attachments` | `mytheus-attachments` |
| `nexus-invoices` | `mytheus-invoices` |
| `nexus-exports` | `mytheus-exports` |
| `nexus-audit-archive` | `mytheus-audit-archive` |

Key layout is unchanged: every object key is prefixed `{org_id}/`.

## Consequences

- The `nexus_knowledge` graph, the `nexus:{org_id}:` Redis prefix and the Redpanda topic names are frozen identifiers and are NOT renamed by this ADR.
- Code that hardcodes a bucket name (`mytheus-intelligence` `libs/shared/object_storage.py`, `mytheus-backend` `meeting_service`) changes to the new names in the same release. A bucket whose name differs from the code fails closed with a missing-bucket error, never a wrong-bucket write.
- Existing buckets are not migrated: no object was written to a `nexus-` bucket in any deployed environment.
- Bucket names are permanent in R2, so a future rename means a new bucket and a copy.
