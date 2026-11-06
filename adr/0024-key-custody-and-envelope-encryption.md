# ADR-0024: Key custody is one envelope-encryption library behind a `KeyProvider` seam; the production custodian is an open decision

`oauth_tokens` ([_canon/02-data-model.md](../_canon/02-data-model.md) section 2.2) and
`pii_tokens` ([ADR-0020](0020-pii-token-vault-and-ner-placement.md)) both need encryption at
rest under per-tenant keys, and ADR-0020 blocked the vault on exactly that: "land KMS +
envelope encryption once, then both tables use it" (open gap 22 in
[work-done-till-now/00](../work-done-till-now/00-status-overview.md)). This ADR records the
scheme and the seam, which are decided, and the production key custodian, which is not.

## Related documents

- [_canon/06-tech-decisions.md section 2.11](../_canon/06-tech-decisions.md) - the
  "Field-level encryption" row this ADR backs.
- [_canon/02-data-model.md section 2.2](../_canon/02-data-model.md) - `oauth_tokens` and
  `pii_tokens`, both amended to cite this ADR.
- [ADR-0020](0020-pii-token-vault-and-ner-placement.md) - the vault whose key-custody
  precondition this addresses.
- [ADR-0021](0021-explicit-grants-no-schema-wide-default-privileges.md) - the grant rule both
  new tables follow.
- [ADR-0016](0016-self-hosted-vps-mvp-substrate.md) - the VPS substrate, which has no
  cloud KMS, and is why the custodian choice is not trivial.

---

## Status

**Accepted.** The scheme and interface were accepted here; the production custodian (open
decision 1 below) was decided 2026-11-06 by [ADR-0026](0026-vps-file-kek-custody.md): a local
KEK file on the VPS, with KMS as the explicit upgrade path. Implemented in `mytheus-backend` `libs/shared/crypto`, with migrations
`24e134664d84` (`oauth_tokens`) and `a2afd3bef634` (`pii_tokens`), 2026-11-06.

---

## Decision

**D1. Envelope encryption, AES-256-GCM.** A data key (DEK) per (`org_id`, purpose), generated
locally, wrapped by a key-encryption key (KEK) held by the `KeyProvider`. A purpose is a code
constant naming the column (`oauth_tokens.access_token`, `pii_tokens.value`). Two tenants
never share a DEK.

**D2. AAD binds tenant, purpose and row.** Every encryption authenticates `org_id`, purpose
and the DEK id, plus caller row context (`connection_id` for OAuth tokens; `entity_type` and
`value_hmac` for vault rows). The DEK wrap authenticates `org_id`, purpose and DEK id too, so
a blob moved to another tenant, another column or another row fails to decrypt even if RLS
were bypassed.

**D3. Self-describing ciphertext, no key table.** Each value is stored as one `bytea`
envelope carrying the KEK id, DEK id, wrapped DEK, nonce and ciphertext. A key table would be
a store shared by integration-service and identity-tenant-service, which the one-writer rule
forbids. Both tables CHECK the envelope prefix so plaintext cannot be written by mistake.

**D4. Rotation.** DEKs rotate automatically for new writes (age, use count, or KEK change).
KEK rotation is a rewrap: the DEK is re-wrapped under the new KEK and the data ciphertext is
unchanged, so `pii_tokens.dek_id` stays valid. The retired KEK stays decrypt-only until a
rewrap job has visited every row. The blind-index key is separate from the KEK and is NOT
rotated with it; rotating it is a re-index job, not yet written.

**D5. Keyed blind indexes.** Equality lookup (`pii_tokens.value_hmac`) uses HMAC-SHA256
under a deterministic per-tenant, per-purpose key derived by the provider (HKDF locally,
a KMS MAC operation in production).

**D6. The `KeyProvider` seam.** `wrap_dek`, `unwrap_dek`, `derive_tenant_key`, `active_kek_id`.
`LocalKeyProvider` reads `MYTHEUS_KEK_BASE64` / `MYTHEUS_BLIND_INDEX_KEY_BASE64` and is
**dev/test only**: it refuses to construct unless `ENVIRONMENT` is a development value (an
unset `ENVIRONMENT` is treated as production), and refuses in every environment when a key
is unset. It never generates a key. `KmsKeyProvider` documents the AWS KMS and GCP Cloud KMS
mapping for each method and raises `NotImplementedError`. The effect: **the vault and
`oauth_tokens` cannot run in production until the open decision below lands** (it landed as
[ADR-0026](0026-vps-file-kek-custody.md), which adds `FileKeyProvider`), which is the
ADR-0020 precondition enforced by code rather than by a comment.

**D7. Interim service authentication for the vault mint route.** `POST /pii-tokens/mint` is
authenticated by per-caller shared secrets (`MYTHEUS_PII_VAULT_CALLER_TOKENS`), limited to the
four redaction-boundary services. Interim until mesh identity (mTLS/SPIFFE) exists.

No key, wrapped key or plaintext is logged or placed in an exception message or `repr`.

## Open decisions (assigned to the user)

1. **Production KEK custodian. DECIDED 2026-11-06: (d), a file rather than an env value, by
   [ADR-0026](0026-vps-file-kek-custody.md).** (a) AWS KMS, (b) GCP Cloud KMS, (c) for the VPS MVP
   (ADR-0016), a self-hosted custodian such as HashiCorp Vault Transit, or (d) allowing the
   local provider in production with the KEK in the VPS secret store. (d) is the cheapest and
   the weakest: the KEK sits on the same host as the ciphertext, which is the "single
   high-value target" ADR-0020 warns about. The code deliberately has no switch for (d).
2. **Who may resolve a token.** `POST /pii-tokens/resolve` defaults to `admin` tier and up,
   audited per token. ADR-0020 says "authorized members"; widening to `member` is one
   constant, but it is a disclosure-policy decision.
3. **Mint-route authentication beyond the interim secret** (D7), once the K8s substrate lands.

## Consequences

- Gap 22's library and both tables exist; the production custodian is the remaining blocker.
- ~80 bytes of envelope overhead per encrypted value. Negligible on token tables.
- A KMS provider must cache: the envelope layer already limits KMS traffic to one wrap per
  (tenant, purpose) per DEK lifetime and one unwrap per DEK per process.
- Rotating the blind-index key is not free and is not built.
