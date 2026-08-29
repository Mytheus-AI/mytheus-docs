# ADR-0020: The per-tenant PII token vault is owned by `identity-tenant-service`, and NER runs on the inference tier

[12-compliance-and-privacy.md §5.2](../12-compliance-and-privacy.md) already specifies that
detected PII spans are replaced with "**typed, reversible placeholders** keyed to a per-tenant
token vault so that an agent extraction referring to 'the customer' stays coherent, while the
raw value never leaves the tenant region", and that detection uses "a lightweight NER model run
on the dedicated inference tier - never on the API tier". Both halves are designed there and
neither exists anywhere in canon: `_canon/02-data-model.md` has no vault table, and no service
in `_canon/01-service-catalog.md` is assigned an NER responsibility.

This ADR closes that by naming the owner, the table, and the placement. It ratifies the data
model and the call graph. It does **not** authorize the migration or any application code, and
it records one precondition that must be met first (§ Key custody) and one question it
deliberately assigns to compliance rather than answering (§ The unavailable-NER decision).

## Related documents

- [_canon/02-data-model.md §2.2](../_canon/02-data-model.md) - amended by this ADR to add
  `pii_tokens`.
- [_canon/01-service-catalog.md §3](../_canon/01-service-catalog.md) - amended by this ADR twice:
  `identity-tenant-service`'s owned data gains `pii_tokens`, and a new `ner-service` is added to
  the AI/inference tier.
- [_canon/06-tech-decisions.md §2.5](../_canon/06-tech-decisions.md) - "Dedicated autoscaling
  inference services ... OFF the API tier", the locked decision that determines where NER runs.
- [12-compliance-and-privacy.md §5.1, §5.2](../12-compliance-and-privacy.md) - the three
  redaction boundaries and the reversibility design this ratifies.
- [ADR-0006: Vector store tenancy](0006-vector-store-tenancy.md) - the precedent for
  per-tenant isolation being the primary boundary with a second compensating control.
- [ADR-0018: Consent entity behind a flag](0018-consent-entity-behind-flag.md) - the precedent
  this ADR follows for landing a schema whose activation is a legal decision, not an
  engineering one.

---

## Status

**Accepted for the data model and the call graph. Implementation UNBLOCKED as of
[ADR-0024](0024-key-custody-envelope-encryption.md)**, which supplies the key custody the
§ Key custody section below made the precondition, and which supersedes this ADR on that one
point only. Everything else here - D1 through D5, the vault shape, and the unavailable-NER
question - stands unchanged.

Read § Key custody below as the requirement that was met, not as a live blocker. Its objection
is still the governing one: ADR-0024 is judged against it, and the reason that ADR refuses to
mint under a development-only `KeyProvider` is precisely this section's argument that a vault
with ad-hoc key management is worse than the irreversible redaction it replaces.

---

## Context

Three separate redaction boundaries exist ([12 §5.1](../12-compliance-and-privacy.md)):
transcript-at-rest in `transcription-service`, document ingestion in `ingestion-service`, and
the always-on LLM-egress pass in `agent-runtime-service`/`rag-service`. Today only the first
ships, and only its deterministic regex layer (email, phone, IBAN via Luhn, card via Luhn, a
government-ID shape). Placeholders are plain type tags (`‹EMAIL›`) and are **irreversible**: the
raw value is discarded.

Reversibility is not a nicety. Without it, a confirmed decision reading "‹PERSON› owns the
renewal" is unusable to the very humans it was extracted for, and every occurrence of the same
customer collapses into the same anonymous tag, so an agent cannot tell two people apart. §5.2's
"stays coherent" is the requirement; a vault is how it is met.

The naive implementation - each boundary keeps its own vault - fails twice. It breaks coherence
across boundaries (a name redacted in a transcript and the same name redacted in an uploaded
document must resolve to the same token, or the graph links nothing), and a shared table written
by three services violates the one-writer-per-store rule that the whole modular-monolith ->
microservices split depends on.

## Decision

### D1. `identity-tenant-service` owns `pii_tokens`, and is its only writer

It already owns `tenant_settings` (whose `pii_redaction_enabled` decides whether redaction runs
at all) and `data_residency` (whose `region` decides where a raw value may lawfully live).
Reversal is defined as happening in-region, in-tenant, for authorized members - which is
precisely the state this service already guards. Placing the vault anywhere else would mean a
second service re-deriving tenancy and residency facts it does not own.

**Rejected: a new `pii-vault-service`.** Cleanest blast radius, and the right answer eventually.
Rejected now because it adds a service to the catalog - a larger canon change - for no scaling
reason: this is a low-QPS keyed lookup, not an independent cost center. Revisit if vault traffic
ever justifies its own autoscaler.

**Rejected: a vault per boundary.** Satisfies one-writer trivially and needs no new network hop,
but destroys cross-boundary coherence, which is the feature's entire stated purpose.

### D2. The three redaction boundaries are callers, never writers

`transcription-service`, `ingestion-service`, `agent-runtime-service` and `rag-service` call a
mint/resolve API on `identity-tenant-service`. They hold no vault state and never touch the
table. This is what keeps one-writer intact while still giving every boundary one shared token
space per tenant.

### D3. `pii_tokens` shape, and why a blind index is required

| Column | Purpose |
| --- | --- |
| `org_id` | Tenant key. RLS `FORCE`, `tenant_isolation`, as every tenant-scoped table. |
| `token` | The opaque placeholder body, e.g. the `7f3a` in `‹PERSON:7f3a›`. |
| `entity_type` | `EMAIL`/`PHONE`/`IBAN`/`CARD`/`GOVID`/`PERSON`/`ORG`/`LOC`. Typed, per §5.2. |
| `value_ciphertext` | The raw value, envelope-encrypted under a per-tenant data key. |
| `value_hmac` | Keyed hash of the normalized raw value. The lookup path. |
| `dek_id` | Which per-tenant data key encrypted this row, so keys can rotate. |

Two uniqueness constraints: `(org_id, token)` so a token resolves to exactly one value, and
`(org_id, entity_type, value_hmac)` so the same raw value in the same tenant always mints the
same token. The second is what makes "the customer" coherent across boundaries and across
meetings.

`value_hmac` is an **HMAC under a per-tenant key, never a plain hash**. Direct identifiers have
catastrophically low entropy - the set of plausible email addresses or phone numbers is small
enough to enumerate - so an unkeyed digest of an email is equivalent to storing the email. The
keyed hash is what makes the lookup column safe to index.

### D4. NER runs on the inference tier, called over the network

[Tech Decisions §2.5](../_canon/06-tech-decisions.md) locks model inference to dedicated
services off the API tier, and [12 §5.2](../12-compliance-and-privacy.md) says NER specifically
"never on the API tier". `transcription-service` is not the API tier, but the rule's purpose -
keeping model inference off a latency-sensitive request path - applies to it with equal force:
it scales on anchor `A5` (20k concurrent live streams) and carries a p95 < 2 s caption SLO. An
NER model in that process couples GPU/CPU-bound inference to the live audio proxy, which is the
exact coupling ADR-0008 and §2.5 exist to prevent.

NER therefore lives in `mytheus-intelligence` and is called over the internal mesh.

### D5. The NER model gets its own service, `ner-service`

"The inference tier" is not a name a code repo can build against, and the Service Catalog is the
service-naming authority, so this ADR adds one rather than leaving the responsibility
unassigned. `ner-service` follows `reranker-service`'s shape exactly: a dedicated model server,
owning no data, called synchronously and internally only (`POST /ner`), scaling on its own
signal.

One service per model is the pattern §2.5 already locks (`embedding-service` and
`reranker-service` are separate for precisely this reason), and the two models it names have
opposite cost profiles and independent autoscalers. Folding a third model into
`embedding-service` would couple three lifecycles - a model swap, a batch-size change or a
scale event in one would move the others - for no benefit beyond one fewer deployment.

Note the deliberate split of concerns this creates: `ner-service` **detects**, it does not
redact. Each boundary keeps its own redaction pass, calls `ner-service` for spans and
`identity-tenant-service` for tokens, and assembles the result itself. Canon already places the
"PII redaction pass" responsibility on `ingestion-service` for the document path, and that
stays true - what moves to `ner-service` is the model, not the pass.

## Consequences

- One new table, one new owning service responsibility, one new cross-tier call.
- `transcription-service` gains a synchronous dependency on the inference tier in the path that
  persists a final. That is a real availability coupling and is the cost of D4.
- The regex layer does **not** go away. It stays as the first, dependency-free pass; NER is
  additive, covering the unstructured entities (`PERSON`, `ORG`, `LOC`) regex cannot.
- Reversal becomes an audited operation: resolving a token is a disclosure of PII and must
  write `audit_logs`, exactly as the session-open residency/redaction decision already does.

## Key custody: the precondition, met by ADR-0024

**The vault must not be built until key custody exists.** A reversible vault concentrates every
tenant's raw PII behind one key; a vault with weak or ad-hoc key management is strictly worse
than the irreversible redaction shipping today, because it converts a scatter of redacted
strings into a single high-value target.

**This was true when this ADR was written and is no longer.** At the time: no migration for
`oauth_tokens`, no encryption helper in `mytheus-backend`, no KMS or encryption environment
variable anywhere, and `mytheus-platform`'s `secrets` Terraform module - specified as managing
"Cloud KMS keys (incl. per-tenant KMS data keys for `oauth_tokens`)" - containing zero `.tf`
files. [ADR-0024](0024-key-custody-envelope-encryption.md) closes it with `tenant_keys` plus a
`KeyProvider` seam, and deliberately separates the vendor question (still open, still needing
infrastructure the program does not have) from the design question (now built).

So `pii_tokens` and `oauth_tokens` need the same substrate. Building the vault first would mean
inventing a second, parallel key story that the `oauth_tokens` implementation would later have
to unify with. The dependency runs the other way: land KMS + envelope encryption once, then
both tables use it. **That is the order ADR-0024 followed** - it builds no vault code at all,
only the shared substrate both tables sit on.

## The unavailable-NER decision, assigned to compliance

When the NER service is unreachable and the tenant has `pii_redaction_enabled`, a final can be:

1. **not persisted** - no PII lands at rest, but the transcript is lost, and `transcript_chunks`
   is the system of record the entire AI tier reads;
2. **persisted regex-only, flagged** - structured PII (email, phone, card, IBAN, government ID)
   is still removed by the dependency-free layer; unstructured entities are not, and the row
   records that it was only partially redacted;
3. **persisted raw** - never acceptable when the flag is on.

Option 3 is rejected outright here. The choice between 1 and 2 is a compliance judgment about
whether losing a meeting's record is preferable to storing names and organizations that the
regex layer never covered anyway, and it may differ per tenant. This ADR does not decide it, in
the same way [ADR-0018](0018-consent-entity-behind-flag.md) leaves the consent-enforcement call
to legal. **Assigned to: Security/Compliance. Still open.**

It is no longer a blocker, because it did not need to be one: implementation makes it the
per-tenant column `tenant_settings.ner_unavailable_policy` (`drop` | `regex_only`), defaulting
to `regex_only` and holding no `raw` value at all, so option 3 stays unrepresentable and
compliance's answer - which this ADR notes may differ per tenant - is a configuration change
rather than a code change.
