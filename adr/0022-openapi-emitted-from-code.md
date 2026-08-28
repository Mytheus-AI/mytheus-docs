# ADR-0022: The OpenAPI contract is emitted from the code, and governed by three gates

The REST contract is code-first. Each service's OpenAPI 3.1 document is emitted from its
FastAPI app, committed, and enforced by a CI job that regenerates it and fails on any
difference. This reverses the spec-first direction stated in
[19-api-architecture.md §11](../19-api-architecture.md).

## Related documents

- [19-api-architecture.md §11](../19-api-architecture.md) - "OpenAPI governance", amended by
  this ADR. The governance it specifies (Spectral, `oasdiff`, a single reviewed document per
  service) is kept in full; only the direction of authority is reversed.
- [plan/plan-1-2-foundation-and-tenancy.md §3.1 step 4, §3.11 step 5](../plan/plan-1-2-foundation-and-tenancy.md)
  - the P1 artifact and the CI gate this finally executes.
- [work-done-till-now/02-plan-1-2-foundation-and-tenancy.md](../work-done-till-now/02-plan-1-2-foundation-and-tenancy.md)
  - gap 5, open since P1, and the P1 DoD line that has read PARTIAL since.
- [work-done-till-now/04-plan-5-6-org-context-and-ai-infra.md](../work-done-till-now/04-plan-5-6-org-context-and-ai-infra.md)
  - gap 13, `bff.yaml` stale against the shipped `dashboard.py`.
- `mytheus-backend/design/openapi/README.md` - the operational detail.

---

## Status

**Accepted, and implemented.** `mytheus-backend` `4b948d1`, 2026-08-28.

---

## Context

Doc 19 §11 says: "The REST contract is **spec-first**, not code-first. Every service publishes
an OpenAPI 3.1 document; the spec is the source of truth and the implementation conforms to
it", and "FastAPI's generated schema is reconciled against it in CI, not the other way around."

That is a coherent policy. It was never executed, and the evidence that it was not is not a
matter of opinion:

- The five specs were written in P1 before any route existed and have not been maintained
  since. `bff.yaml` describes `Dashboard` as flat arrays plus a separate `arms` object; the
  shipped `dashboard.py` returns an inline `{status, items}` arm per field, and has for months.
- `spectral lint` had never been run. Running it for the first time found the ruleset does not
  parse: two rules put `field` as a sibling of `given` rather than inside `then`. Three more
  used `(?i)`, which is PCRE, against a JavaScript engine that needs `/pattern/i`.
- `oasdiff` was never wired.
- The reconciliation step doc 19 names - "FastAPI's generated schema is reconciled against it
  in CI" - was never built at all.
- The spec set disagreed with the code about which services exist: a 6.8 KB document for
  `admin-ops-service`, which has zero routes, and none for `meeting-service`, which owns the
  only route that mints a LiveKit token.

Plan 1-2 drew the correct conclusion from the first point and refused to generate a TypeScript
client, recording that "generating from them would encode drift rather than prevent it". That
left both clients on hand-written types, which is the state B8 exists to fix and could not.

The pattern across all five is one thing: spec-first's correctness depends entirely on a
reconciliation step, and that step is the part nobody builds.

---

## Options considered

**(a) Build the reconciliation doc 19 specifies: compare the hand-written spec to FastAPI's
generated one, hand-written wins.** Rejected, and the reason is technical rather than a
preference. That comparison needs a differ that understands OpenAPI *equivalence* - two
documents describing the same API differ enormously in inlining, `$ref` use, schema naming and
ordering, and none of that is a real difference. Every such differ needs a suppression list,
and a suppression list is where genuine drift hides. The reverse direction needs no differ at
all: regenerate, compare bytes.

**(b) Emit from the code and commit the result.** Chosen. Drift is not detected, it is
impossible: there is nothing for the document to drift *from*. Governance is unchanged, because
Spectral and `oasdiff` lint a document regardless of who wrote it.

**(c) Emit and do not commit - generate at build time for the client.** Rejected. An
uncommitted spec cannot be diffed in review, so a contract change becomes invisible in the one
place a human would catch it, and `oasdiff` loses the base document it compares against.

**(d) Keep spec-first and accept the drift.** This was the status quo for eight months. It
produced a spec set that no one could safely generate a client from, which is the entire
purpose of having one.

---

## Decision

**D1. Each service's OpenAPI document is emitted from its FastAPI app** by
`scripts/emit_openapi.py`, committed under `mytheus-backend/design/openapi/`, and marked
generated. Services are discovered by walking `services/*/main.py`, not listed: the
hand-maintained list is what produced a spec for a service with no routes and no spec for one
with two.

**D2. Three CI gates, because they catch three different failures.**

| Gate | Catches |
|---|---|
| `emit_openapi --check` | a route changed without regenerating |
| `spectral lint` | `org_id` in a path, `offset` pagination, a bare `application/json` error |
| `oasdiff breaking` | a deleted path, a newly-required request field |

None subsumes another. A document can be perfectly in sync and still put `org_id` in a path;
it can pass every house rule and still delete a path a client depends on.

**D3. The emitter documents what the code does and never more.** FastAPI cannot see middleware,
exception handlers or dependency-injected auth, so the emitter adds the RFC 7807 error shape,
the `Idempotency-Key` header, and the document-level bearer scheme - all three of which the
running app genuinely implements. The rule for anyone extending it: never add to the document
something the code does not do. A spec that overstates is worse than one that understates,
because the understatement is visible at the call site and the overstatement is not.

**D4. The `ErrorCode` enum is emitted from `libs.shared.errors.ErrorCode`.** `errors.py` had
carried the instruction "Keep the two in sync: a value here with no counterpart there is a
bug". Sourcing the enum from the code deletes that instruction rather than restating it.

**D5. The P1 specs are superseded, not deleted.** They carry a banner pointing here and remain
the record of what was designed. They are a design artifact; they are not the contract.

**D6. `api-gateway` publishes no document.** It proxies, so its routes live in
`edge.ROUTE_MAP` rather than its own decorators, and emitting it produced a zero-path document
that failed the ruleset for an accurate reason. Its real contract is the union of the others
under its own prefixes, which nothing computes; clients generate per service and point at the
gateway origin, which is what both already do.

---

## Consequences

**Positive.** The P1 DoD line "OpenAPI specs pass Spectral" is met for the first time, with the
ruleset repaired. Plan 1-2 gap 5 and Plan 5-6 gap 13 are closed. B8 is unblocked: a client
generated from these documents inherits the real error shape, the real security scheme, and an
`Idempotency-Key` parameter that the hand-written specs never carried, so a generated client
could not have sent the one header the write contract is built on.

**Negative, accepted.** An endpoint can no longer be designed in YAML and reviewed before it is
written. That was spec-first's real benefit and it was genuinely collected in P1, when the
design pack was the point. It is a smaller loss now that routes exist: a new endpoint is
reviewed as a diff to the code plus a diff to the emitted document in the same pull request,
which is more information than the YAML alone carried.

**A residual worth naming.** Three of the thirteen Spectral rules were non-functional and the
documents passed anyway - two because the ruleset would not load, three because a regex threw.
A gate that has never been run is not a gate, and the only way to know it works is to make it
fail on purpose. Every gate here was proven to fire against a deliberate defect before being
called done: a hand-edited spec for `--check`, an `org_id` path and an `offset` parameter for
Spectral, a deleted path for `oasdiff`.

**One documented control turned out not to exist.** The ruleset claimed `oasdiff` enforced the
no-removal half of the frozen `ErrorCode` enum. It does not: the enum is reachable only from a
response schema, and narrowing a response enum is not breaking by `oasdiff`'s definition -
verified by deleting `NOT_FOUND` from a copy of `core-api.yaml`, which `oasdiff` passed while
flagging the two paths the same probe deleted. Removing a code is breaking in *this* program
because both clients branch on it, so it is now asserted by
`test_no_published_error_code_is_ever_removed`, and the false claim is corrected in both places
it appeared.
