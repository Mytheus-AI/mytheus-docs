# ADR-0024: Key custody - per-tenant envelope encryption behind a `KeyProvider` seam

[ADR-0020](0020-pii-token-vault-and-ner-placement.md) ratified the `pii_tokens` vault and then
refused to authorize its migration, on one precondition: *"The vault must not be built until key
custody exists."* [`_canon/02-data-model.md` §2.2](../_canon/02-data-model.md) makes the same
demand of `oauth_tokens`, which it describes as using "envelope encryption, per-tenant KMS data
key". Neither table can be built, and both need the same thing.

Nothing in the program provides it. There is no encryption helper in `mytheus-backend`, no KMS
or encryption environment variable anywhere in any repo, and `mytheus-platform`'s `secrets`
Terraform module - specified as managing "Cloud KMS keys (incl. per-tenant KMS data keys for
`oauth_tokens`)" - contains zero `.tf` files. This is
[00-status-overview.md](../work-done-till-now/00-status-overview.md) gap 22.

This ADR is the one that unblocks both. It decides the **shape** of key custody, not the
vendor, so that the shape can land now on the substrate the program actually runs on and the
vendor can change later without touching a single caller.

## Related documents

- [ADR-0020: PII token vault and NER placement](0020-pii-token-vault-and-ner-placement.md) -
  blocked on this ADR; unblocked by it. Its § Key custody is the requirement this answers.
- [ADR-0016: MVP compute substrate - self-hosted single-node VPS](0016-self-hosted-vps-mvp-substrate.md) -
  why there is no cloud KMS to call today, and why that is not a reason to defer.
- [ADR-0002: Multi-tenancy model](0002-multi-tenancy-model.md) - pooled + RLS, the boundary
  a per-tenant key is the second layer of.
- [ADR-0006: Vector store tenancy](0006-vector-store-tenancy.md) - the precedent for a
  per-tenant isolation primitive with a resolver chokepoint rather than call-site discipline.
- [ADR-0021: Explicit grants, no schema-wide default privileges](0021-explicit-grants-no-schema-wide-default-privileges.md) -
  why the tables below carry their own `GRANT` and cannot inherit one.
- [`_canon/02-data-model.md` §2.2](../_canon/02-data-model.md) - amended by this ADR to add
  `tenant_keys`, and to unblock the `pii_tokens` row's "not yet migrated" note.
- [12-compliance-and-privacy.md §5.2](../12-compliance-and-privacy.md) - the reversibility
  requirement all of this exists to serve.

---

## Status

**Accepted, and implementation-authorizing** - unlike ADR-0020, which this supersedes on that
one point. `tenant_keys` and `pii_tokens` may be migrated once this lands.

Authorization is narrow and conditional, and the condition is enforced in code, not by
convention: see § The production condition. A `LocalKeyProvider` deployment is authorized to
run the vault in **development**; it is not authorized to hold a real tenant's PII, and
`identity-tenant-service` refuses to mint against one unless explicitly acknowledged.

---

## Context

### What "key custody" has to mean here

ADR-0020's objection is precise and worth restating, because it is the thing that has to be
answered rather than worked around: a reversible vault *concentrates* every tenant's raw PII
behind one key, so a vault with ad-hoc key management is **strictly worse than the irreversible
redaction shipping today**. Today a leak of `transcript_chunks` yields `‹EMAIL›`. After the
vault, a leak of `pii_tokens` plus the key yields every email address in the tenant.

So the bar is not "the values are encrypted". It is that compromising the database does not by
itself compromise the values.

### What the substrate can actually offer

[ADR-0016](0016-self-hosted-vps-mvp-substrate.md) put the whole program on one Hostinger box
running Docker Compose. PostgreSQL and the application are the same machine. On that substrate a
root key in the application's environment is on the same disk as the ciphertext it protects, so
an attacker with the box has both. There is no configuration of a single node that changes this.

This is the honest reading, and it has been used - wrongly - as a reason to defer the whole
item. The inference does not hold. The vendor question ("what holds the root key") and the
design question ("how does a value get encrypted, indexed, and rotated per tenant") are
independent, and only the first one is blocked on infrastructure the program does not have.
The second is the larger body of work, it is entirely buildable today, and every day it is
deferred is a day `oauth_tokens` and `pii_tokens` both stay unbuildable.

### The forces

- **Two tables need this, not one.** Building either one's key story first means inventing a
  second one later and unifying them. ADR-0020 says this explicitly: *"land KMS + envelope
  encryption once, then both tables use it."*
- **A blind index is required, and it interacts with rotation.** `pii_tokens.value_hmac` is
  what makes the same raw value mint the same token across boundaries (ADR-0020 D3). It is a
  lookup column, so it must be stable; it is derived from a key, so a naive "one key per
  tenant, rotate it" design silently breaks every existing lookup on the first rotation.
- **The vendor will change.** Plan 14 moves the program to Kubernetes on a cloud provider,
  where a real KMS exists. Anything that hard-codes today's answer is work to be redone.
- **A degraded key story must not be able to become production by inaction.** This is the
  failure mode that turns this ADR into the thing it was written to prevent.

## Decision

### D1. Envelope encryption, with a per-tenant data key wrapped by a root key

Values are encrypted under a **per-tenant data key (DEK)**, never under a global key. The DEK
itself is stored only in **wrapped** form, encrypted under a **root key (KEK)** that the
database never holds.

Per-tenant rather than global because the tenant is the blast radius the whole architecture is
organized around ([ADR-0002](0002-multi-tenancy-model.md)): a single global key means one
compromise is a program-wide breach, and it makes a per-tenant erasure request (GDPR Art. 17)
a row-by-row delete instead of the crypto-shredding a per-tenant key allows.

**AES-256-GCM** for both wrapping and value encryption. Authenticated encryption is not optional
here: without it, ciphertext in a database an application can write is malleable. The per-tenant
`org_id` is bound in as GCM **additional authenticated data**, so a ciphertext moved between
tenant rows fails to decrypt rather than decrypting into the wrong tenant.

### D2. The root key lives behind a `KeyProvider` seam with exactly two operations

```
wrap(org_id, plaintext_key)   -> wrapped bytes
unwrap(org_id, wrapped_key)   -> plaintext key
```

That is the entire interface, and it is the interface a cloud KMS already has. Nothing above it
knows whether the root key is a local secret, AWS KMS, GCP KMS, or Vault transit.

**Rejected: calling a cloud KMS per operation, directly from the call sites.** It is the right
production answer and it is what `ExternalKmsKeyProvider` will be, but wiring it at the call
sites means every caller learns a vendor SDK, and the program has no cloud account to call
today (the same class of blocker as gap 12's LiveKit/Deepgram accounts). The seam gets the
architecture right now and makes the vendor a deployment decision.

**Rejected: deferring the whole item until a KMS exists.** This is the status quo and it is
what has kept two tables unbuildable. It also gets the sequencing backwards: the seam is what
makes adopting a KMS a configuration change instead of a migration.

Two implementations are named:

| Provider | Root key source | Authorized for |
| --- | --- | --- |
| `LocalKeyProvider` | 32 bytes from `MYTHEUS_ROOT_KEK` (base64) | Development only. |
| `ExternalKmsKeyProvider` | A KMS that performs wrap/unwrap without releasing the root key | Production. **Not implemented** - this ADR defines the seam it plugs into. |

`LocalKeyProvider` does **not** meet ADR-0020's bar, and this ADR does not pretend otherwise:
the root key is on the same host as the ciphertext, so it provides defence against a stolen
database dump or a backup, and none at all against a compromised host. That is a real but
partial gain, and stating which half it buys is the point.

### D3. `tenant_keys`, and why `purpose` is a column rather than two tables

| Column | Purpose |
| --- | --- |
| `id` | The `dek_id` that `pii_tokens` and (later) `oauth_tokens` reference. |
| `org_id` | Tenant key. RLS `FORCE`, `tenant_isolation`, as every tenant-scoped table. |
| `purpose` | `enc` or `index`. See below - this column is the whole reason the table works. |
| `wrapped_key` | The DEK, encrypted under the root key. The plaintext DEK is never stored. |
| `wrap_key_id` | Which root key wrapped it, so the ROOT key can rotate too. |
| `status` | `active` or `retired`. |

Two partial unique indexes: one active `enc` key per tenant, and one `index` key per tenant
**ever** (no `status` predicate - see D4).

**Rejected: a single per-tenant key used for both encryption and the blind index.** It is the
obvious shape and it is wrong twice. Using one key's bytes for both AES-GCM and HMAC is key
reuse across primitives, and more concretely it makes the blind index unrotatable in practice -
which is D4.

### D4. The encryption key rotates; the blind-index key does not

`value_hmac` is a lookup column with a `UNIQUE (org_id, entity_type, value_hmac)` constraint
that exists so the same raw value always mints the same token. If the HMAC key rotated, every
existing row's `value_hmac` would become unreachable: the same email would hash to something
new, miss the index, and mint a *second* token. Coherence - the entire feature - would silently
degrade on the first rotation, and nothing would fail loudly.

So the two keys have deliberately different lifecycles:

- **`enc`** rotates freely. `pii_tokens.dek_id` records which key encrypted each row, so old
  rows stay readable under a retired key and re-encryption is a background job, not a stop-
  the-world migration. This is what `dek_id` is *for*.
- **`index`** is minted once per tenant and never retired. Rotating it requires decrypting and
  re-HMACing every row for that tenant in one transaction, which is a real operation with real
  cost. It is not forbidden - it is refused by the helper and left to a deliberate migration,
  so nobody performs it by accident through the ordinary rotation path.

Both are derived from their stored key material by **HKDF-SHA256 with distinct info labels**, so
the bytes actually fed to AES-GCM and to HMAC are never the same bytes even before the
lifecycles diverge.

### D5. `identity-tenant-service` owns `tenant_keys`, as it owns `pii_tokens`

Same reasoning as [ADR-0020](0020-pii-token-vault-and-ner-placement.md) D1, and it must be the
same service: a key table owned by one service and the ciphertext it protects owned by another
is two services that must agree about rotation forever. `integration-service` will read
`tenant_keys` for `oauth_tokens` through the same helper rather than growing its own.

The **helper** (`libs/shared/crypto.py`) is shared; the **table** has one writer. This is the
`libs/shared/audit.py` split exactly: one implementation of the dangerous thing, so there are
not nine dialects of it.

## The production condition

The failure mode this ADR most has to prevent is `LocalKeyProvider` quietly becoming the
production key story because nothing ever forced the question. Three mechanisms, in code:

1. **No default root key.** `LocalKeyProvider` raises if `MYTHEUS_ROOT_KEK` is unset or is not
   32 bytes. There is no generated-on-boot fallback, because a key that appears by itself is a
   key nobody decided to protect.
2. **Minting is refused unless the deployment is acknowledged.** `identity-tenant-service`
   returns `503 DEPENDENCY_UNAVAILABLE` from the mint path when the configured provider is
   `LocalKeyProvider` and `PII_VAULT_ALLOW_LOCAL_KEK` is not set. The redaction boundaries
   treat that exactly as they treat an unreachable vault: they fall back to the irreversible
   placeholders shipping today. **The degraded path is the status quo, never something worse
   than it.**
3. **Resolution is audited unconditionally.** Reversing a token is a PII disclosure and writes
   `audit_logs` in the same transaction, per ADR-0020's consequence list. There is no
   best-effort mode, for the reason `libs/shared/audit.py` gives.

## Consequences

- Two tables (`tenant_keys`, `pii_tokens`) and one shared helper. `oauth_tokens` gains no code
  here but stops being blocked.
- **Losing the root key destroys the data it protects.** Wrapped DEKs are unrecoverable
  without it, and a backup of PostgreSQL is therefore *not* a backup of the vault. Root-key
  backup is an operational requirement on `mytheus-platform`, called out here because the
  consequence of missing it is silent until a restore.
- Crypto-shredding becomes available: deleting a tenant's `tenant_keys` rows makes every
  `pii_tokens` row for that tenant permanently undecryptable, which is a cheaper and more
  complete answer to an erasure request than a row-by-row delete.
- The blind index gives up something real. `UNIQUE (org_id, entity_type, value_hmac)` means an
  attacker holding the database and the index key can confirm a *guess* ("is alice@acme.com in
  this tenant") even without the encryption key. That is inherent to any deterministic
  searchable encryption and is the cost of the coherence property; it is why the index key is
  keyed rather than a plain digest, and why it is a separate key from `enc` so the confirmation
  oracle does not also yield plaintext.
- One more thing must exist before a real tenant's PII is stored: an `ExternalKmsKeyProvider`
  and somewhere to run it. That is now a *deployment* item, not an architecture one.

## What this ADR does not decide

**The unavailable-NER question stays open.** ADR-0020 assigned Security/Compliance the choice
between dropping a final and persisting it regex-only-and-flagged when `ner-service` is
unreachable for a tenant with `pii_redaction_enabled`. This ADR does not answer it. The
implementation makes it a per-tenant column (`tenant_settings.ner_unavailable_policy`,
`drop` | `regex_only`) defaulting to `regex_only`, so whichever way compliance rules - and
ADR-0020 notes it may differ per tenant - the answer is a configuration change rather than a
code change. `persist raw` is not representable, because ADR-0020 rejected it outright.
