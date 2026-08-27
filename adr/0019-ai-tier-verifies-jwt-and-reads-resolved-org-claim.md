# ADR-0019: The AI tier verifies the Clerk JWT itself, and reads a pre-resolved internal `org_id` claim

Every REST route in `mytheus-intelligence` re-verifies the Clerk session token in its own
process, and takes `org_id` from a verified claim that already carries the resolved internal
UUID. It never takes `org_id` from a request header, and it never resolves Clerk's opaque
organization id by reading another repo's table. This ADR records a decision that was already
implemented and shipped; it is written after the fact, which is itself part of what it records.

## Related documents

- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md) - assigns JWT validation and
  `org_id` extraction to `api-gateway`. This ADR does not move that assignment; it records what
  the tier behind the gateway does in addition.
- [ADR-0010: Auth - Clerk enterprise SSO](0010-auth-clerk-enterprise-sso.md) - owns the Clerk
  contract, the edge-plus-service double verification, and D2's "the claim is opaque and must
  not be parsed as a UUID" rule, which this ADR deliberately inverts for a different claim.
- [ADR-0002: Multi-tenancy model](0002-multi-tenancy-model.md) - the fail-closed `org_id`
  contract this restores on the AI tier's sync path.
- [ADR-0006: Vector store tenancy](0006-vector-store-tenancy.md) - the per-tenant Qdrant
  collections that were reachable by an unauthenticated caller before this landed.
- [11-security-architecture.md](../11-security-architecture.md) - the edge-plus-service
  verification posture this implements for a second repo.
- [work-done-till-now/04-plan-5-6-org-context-and-ai-infra.md](../work-done-till-now/04-plan-5-6-org-context-and-ai-infra.md)
  - the whole-band review that found this, and the record of the fix.

---

## Status

**Accepted, and already implemented.** The code shipped in `mytheus-intelligence` commit
`9ea2f9b` on 2026-08-06 and merged to `main` the same day. This ADR is written on 2026-08-28,
three weeks later, because the review that demanded it said "this needs an ADR and a fix" and
only the fix was done. Recording it late is a deviation from this repo's canon-first ordering
and is stated plainly rather than presented as if the ordering had been followed.

---

## Context

The Plan 5-6 whole-branch review raised one Important finding. `ingestion-service`,
`search-service`, `reranker-service` and `embedding-service` each took `org_id` from a
client-supplied `X-Org-Id` request header, and failed closed only when that header was not a
well-formed UUID. None of them verified a JWT or a gRPC identity.

Concretely: anyone with network reachability to `ingestion-service` could `POST /ingest`
carrying another organization's id in that header and write into that organization's object
storage and vector collection, or call `GET /search` with it and read that organization's
transcript index. Every isolation control below the header held perfectly. The per-tenant Qdrant
collection boundary, the mandatory `org_id` payload filter, RLS `FORCE` on the `intelligence`
schema, the per-tenant object prefix: each did exactly what it was designed to do, for whichever
tenant the caller named.

Two things about how this arrived matter more than the code.

**Four defensible local decisions composed into one indefensible global one.** Each of the four
tasks disclosed the header convention individually, as an accepted interim measure, and each was
reasonable in isolation: `api-gateway` is where canon assigns JWT validation, and at the time
those tasks ran it did not exist yet. What never happened was checking that local convention
against the sibling repo's stricter standard, or escalating it as a band-level risk. Four
disclosures in four task reports are not a decision, and no reviewer was looking at all four at
once until the whole-branch review.

**The program had already shipped one authorization-bypass defect of exactly this shape.**
Defect D1 in Plan 1-2: the application connected to PostgreSQL as a role holding `rolbypassrls`,
so every `tenant_isolation` policy was decorative on the only path that mattered. In both cases
every component was individually correct and the composition was not.

The invariant this violated is stated in `mytheus-backend`'s own CLAUDE.md and is program-wide:
the gateway is a DDoS shield, not a trust delegation, so every service tier re-verifies the JWT
itself. `identity-tenant-service` genuinely does. The AI tier did not.

---

## Decision

**D1. Every REST route in `mytheus-intelligence` verifies the Clerk session token in its own
process.** `libs/shared/auth.py` provides one dependency, `verified_org_id`, and all four
services depend on it: `POST /ingest` and `GET /ingest/jobs/:id`, `GET /search`, `POST /embed`,
`POST /rerank`. The `X-Org-Id` header binding is removed, not deprecated. `org_id` has no
unauthenticated source on this tier any more.

This is a faithful reimplementation of `mytheus-backend`'s verification, not a shared library:
the two repos cannot import across each other. RS256 against cached JWKS honoring rotation,
`nbf` required rather than merely honored when present, and a JWKS outage mapped to `503` rather
than `500` so an identity-provider outage stays attributable to the dependency. The four defects
that shaped `mytheus-backend`'s verifier are inherited deliberately rather than rediscovered.

**D2. The claim carries the ALREADY-RESOLVED internal `org_id`, a UUID, not Clerk's opaque
native organization id.** It is read from `CLERK_JWT_INTERNAL_ORG_CLAIM`, defaulting to the
claim name `internal_org_id`.

This is deliberately a different environment variable and a different claim name from
`mytheus-backend`'s `CLERK_JWT_ORG_CLAIM`, which holds the opaque `org_...` string. The two
carry different things. Reusing one name across both repos with two meanings would be a silent
trap for anyone reading the two `.env` files side by side, and the failure it produces is a
`UUID(...)` parse of an opaque string, which is precisely the defect ADR-0010 D2 exists to
prevent.

**D3. This repo does NOT reimplement `org_resolver.py`, and does not read
`public.organizations`.** `mytheus-backend` resolves the opaque Clerk id to the internal
`org_id` with one query against that table. `mytheus-intelligence` shares the same physical
Postgres and could issue the same query, and must not. Migration `9c2b7e4a1d03`'s own docstring
records the boundary: `org_id` in this repo is an opaque tenant UUID carried by the event
envelope header, never a row this repo looks up by joining into another repo's table. Reading
`public.organizations` from the sync path would reopen exactly the boundary that migration
closed for the async path.

**D4. Every degenerate shape fails closed, and no failure is an oracle.** A missing bearer, an
absent claim, a non-string claim, and a claim that is not a UUID all return `401`. The response
does not echo the underlying exception message, which varies by failure mode and would hand a
caller a free signal for which check failed.

---

## Options considered

**(a) Leave the header until `api-gateway` lands, then trust it.** Rejected on two grounds. It
contradicts the program-wide invariant that the service tier re-verifies regardless of the
gateway, so it would have to be un-done rather than extended. And it treats an unbounded wait as
a mitigation: the tier was already deployed against real infrastructure holding real tenant
data, and "latent because nothing calls it across a repo boundary yet" is a property of the
current callers, not of the exposure.

**(b) Reimplement `org_resolver.py` here, reading `public.organizations`.** Rejected. It gives
the strictly better security property, since resolving through that table is also where
org-exists, not-suspended and not-soft-deleted are enforced, and this repo therefore cannot
enforce them. But it buys that by dissolving the bounded-context boundary between two repos
sharing one database, which is the boundary that keeps the future service split mechanical. The
cost is recorded as an accepted consequence below rather than paid here.

**(c) Verify via gRPC to `identity-tenant-service.GetUserContext` instead of verifying locally.**
Rejected for the REST tier. It makes every AI-tier request synchronously dependent on another
repo's service being up, converting an identity-provider outage into a hard dependency chain,
and it does not remove the need to verify the caller's token in the first place.

---

## Consequences

**The unauthenticated write path is closed.** `org_id` on this tier now originates only from a
genuine, current, correctly signed Clerk session token.

**The AI tier is unreachable until the Clerk JWT template claim exists, and that is not this
repo's change to make.** The claim `internal_org_id` must be populated on the token, which means
a Clerk dashboard template mapping the organization's `public_metadata.internal_org_id`, written
by `identity-tenant-service` at provisioning time. Until that is configured, every route here
returns `401` on every request. That is the correct direction to fail, and it is a real
operational precondition rather than a detail: the work that creates the claim is the same work
that provisions the first real organization. Anyone marking the inference tier "reachable and
authenticated" must confirm the claim is actually present in a minted token, not merely that
this code is deployed.

**This tier cannot tell a suspended organization from an active one.** D3's consequence, and it
is inherited rather than new: `intelligence.outbox` and `intelligence.inbox` already accept an
`org_id` from the event envelope with the same limitation, recorded as decision 3 of migration
`9c2b7e4a1d03`. A token minted before a suspension remains usable on this tier until it expires.
The exposure is bounded by Clerk session lifetime, and the mitigation, if it is ever needed, is a
`tenant-events` consumer in this repo holding a suspension set, not a cross-repo read.

**Two org claim names now exist across two repos.** `CLERK_JWT_ORG_CLAIM` (opaque, in
`mytheus-backend`) and `CLERK_JWT_INTERNAL_ORG_CLAIM` (resolved UUID, here). This is a real
cost, accepted because the alternative is one name with two meanings. Both are documented at
their point of use.

**A late ADR is a process failure, recorded as one.** The fix shipped without this document, so
for three weeks the only account of the reasoning lived in a module docstring, and the program
records continued to describe the finding as open. The concrete harm was not hypothetical: a
later planning pass read the records, re-verified against a stale checkout, and scheduled the
already-completed work again. Canon-first ordering exists to prevent exactly that.
