# ADR-0026: Production key custody on the VPS is a local KEK file; KMS is the explicit upgrade

Decides open decision 1 of [ADR-0024](0024-key-custody-and-envelope-encryption.md): who holds
the key-encryption key (KEK) in production. The user chose option (c) in its plainest form, a
local key file on the VPS, over AWS KMS, GCP Cloud KMS and Vault Transit. This ADR records
that choice, the residual risk it accepts, and the mitigations that are a condition of it.

## Related documents

- [_canon/06-tech-decisions.md section 2.11](../_canon/06-tech-decisions.md) - "Field-level
  encryption", amended to name the custodian.
- [_canon/02-data-model.md section 2.2](../_canon/02-data-model.md) - `oauth_tokens` and
  `pii_tokens`, whose every row depends on this key.
- [ADR-0024](0024-key-custody-and-envelope-encryption.md) - the scheme and the `KeyProvider`
  seam this fills.
- [ADR-0016](0016-self-hosted-vps-mvp-substrate.md) - the single-node VPS substrate.

---

## Status

**Accepted 2026-11-06** (decision by the user). Implemented in `mytheus-backend`
`libs/shared/crypto/providers.py` (`FileKeyProvider`, `RotatingKeyProvider`) and
`mytheus-platform` `deploy/compose.prod.yml` + `deploy/mytheus-deploy.sh`.

---

## Decision

**D1. `FileKeyProvider`.** The KEK is a base64-encoded 32-byte file at `MYTHEUS_KEK_FILE`
(default `/run/secrets/mytheus_kek`). The blind-index root is a second file of the same kind
at `MYTHEUS_BLIND_INDEX_KEY_FILE` (default `/run/secrets/mytheus_blind_index_key`), kept
separate so a KEK rotation never changes a `value_hmac`. The service refuses to start if
either file is missing, empty, not a regular file, not exactly 32 bytes, group- or
world-accessible (only mode 0400 or 0600), or not owned by the service's uid.

**D2. Explicit selection, fail closed.** Production uses this provider only when
`MYTHEUS_KEY_CUSTODY=local-file` is set. Unset outside a development `ENVIRONMENT` means the
service refuses to start; no custodian is ever chosen implicitly. Under `local-file`, a key
value in an environment variable (`MYTHEUS_KEK_BASE64`, `MYTHEUS_BLIND_INDEX_KEY_BASE64`,
`MYTHEUS_KEK_RETIRED`) is refused outright: environment values leak through
`docker inspect`, `/proc/*/environ`, crash reports and child processes. `LocalKeyProvider`
(env values) stays development/test only.

**D3. Rotation by KEK id, and the swap to KMS needs no data migration.** Every envelope
already carries its KEK id. A file rotation adds the new file as active and lists the old one
in `MYTHEUS_KEK_RETIRED_FILES` (`id=/path`) until the rewrap job has visited every row.
`RotatingKeyProvider` routes unwraps by KEK id across custodians, so the upgrade is: make KMS
active, keep the file provider decrypt-only, rewrap, retire the file. The blind-index root
stays with the file provider until a separate re-index job moves it. KEK ids must be unique
across custodians for all time (`file-1`, `file-2`, `kms-...`).

## Residual risk, stated plainly

**The KEK shares a host with Postgres. A host-level compromise (root on the VPS, or a
container escape) exposes the key and the ciphertext together, and at that point the
encryption protects nothing.** What this scheme still defends against: a leaked database dump
or backup, a stolen volume snapshot, SQL-level access through the application roles, and a
compromise of any container the key is not mounted into. It does not defend the host.

## Required mitigations (conditions of this decision)

1. **Mounted into `identity-tenant-service` and the three `integration-service` processes
   only**, read-only, no other container. (Amended 2026-11-12: `integration-service`
   envelope-encrypts `oauth_tokens`, so it, `integration-worker` and `integration-consumer`
   got the same read-only mounts in `mytheus-platform` `deploy/compose.prod.yml` (platform
   PR #21). Without them the first OAuth connect in production failed with
   `KeyConfigurationError: MYTHEUS_KEY_CUSTODY is unset`. A separate KEK id was not used:
   one active KEK keeps the backup and rotation runbook single.)
2. **File mode 0400, owned by the service uid** (10001 in the `mytheus-backend` image).
   Enforced by the provider at startup and by the deploy preflight.
3. **Backed up off-box, separately from database backups.** Losing the KEK makes every
   `oauth_tokens` and `pii_tokens` row permanently unrecoverable. Storing it alongside the
   database backup would undo the one threat this scheme still covers (a leaked backup).
4. **A documented rotation runbook**: generate a new file, add it as active with a new id, list
   the old one as retired, run the rewrap, verify, retire, and re-back-up off-box. Owed as a
   document in `mytheus-platform`. Not written yet.
5. **KMS is the explicit upgrade trigger**: move to `KmsKeyProvider` before the first
   enterprise tenant, or before real PII at scale, whichever comes first.

## Consequences

- The vault and `oauth_tokens` can run in production once both key files are provisioned.
- One more pair of secrets to back up off-box, with a total-loss failure mode if neglected.
- The upgrade to KMS is a code fill-in plus a rewrap job, not a data migration.
