# ADR-0017: Composite partition key for transcript-chunks and agent-stream

This ADR ratifies the `{org_id}:{meeting_id}` composite partition key for exactly two Redpanda
topics, `transcript-chunks` and `agent-stream`. Every other topic keeps the locked `org_id`-only
partition key. This resolves the DECISION-REQUIRED item raised in Plan 3-4 section 8 and
overturns, for these two topics only, the "not ratified and is not used" statement that
previously froze `org_id` as the sole key on every topic.

## Related documents

- [_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md) — the frozen per-topic partition
  key table this ADR changes for `transcript-chunks` and `agent-stream`.
- [_canon/02-data-model.md §7 rule 3](../_canon/02-data-model.md) — "Redpanda by tenant-keyed
  partition"; unchanged in substance, since `org_id` remains the leading component of the
  composite key.
- [Event-Driven Architecture §4.1](../07-event-architecture.md) — the reader-doc section that
  already described this key as a proposal; propagated to cite this ADR as ratifying it.
- [Real-time & Transcription §3.2, §5](../10-realtime-and-transcription.md) — the two sections
  that already assumed the composite key ahead of ratification; now cite this ADR.
- [ADR-0007: Event delivery & idempotency](0007-event-delivery-idempotency.md) — the
  inbox/dedupe machinery that makes reprocessing safe regardless of partition key shape.

---

## Status

**Accepted.** Amends [_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md) for exactly
two topics: `transcript-chunks` and `agent-stream`. Does not touch the other 19 topics, and does
not touch `agent-results`, which stays `org_id`-only despite being grouped with the other two
under the "high-volume meeting" topic class in [07 §4.1](../07-event-architecture.md).

---

## Context

Canon has frozen the partition key for every Redpanda topic as `org_id` alone, and explicitly
recorded the composite alternative as "not ratified and is not used." [07
§4.1](../07-event-architecture.md) and [10 §3.2, §5](../10-realtime-and-transcription.md) already
described a `{org_id}:{meeting_id}` composite key for the high-volume meeting topics as a sound,
if unratified, mitigation — those two reader docs were written ahead of canon on this point, which
is itself a defect this ADR closes.

### The problem `org_id`-only creates for these two topics specifically

`org_id`-only partitioning hashes every event for one tenant onto a single partition. For most
topics this is the correct trade: it gives per-tenant ordering and replay with no downside, since
per-tenant throughput on those topics is modest. `transcript-chunks` and `agent-stream` are
different: they are the two highest-*count* topics in the system (`transcript-chunks` at 360
finals/meeting, `agent-stream` at ~1,260 deltas/meeting, Capacity `E8`/`T2`), and a single
enterprise tenant running thousands of concurrent meetings can generate enough combined throughput
on either topic to saturate the one partition its `org_id` hashes to, even though the partition
count for the topic overall (180 and 120 respectively, [07 §4.2](../07-event-architecture.md)) is
otherwise ample.

### Why the fix is a composite key, not more partitions

Adding more partitions to `transcript-chunks`/`agent-stream` does not help a single hot tenant,
because Kafka/Redpanda hashes one key to exactly one partition regardless of the topic's total
partition count. The fix has to change what is hashed. The composite key `{org_id}:{meeting_id}`
spreads one tenant's meetings across partitions while preserving ordering **within a meeting** —
which is the only ordering scope either consumer of these two topics actually needs: the
embedding consumer processes one meeting's transcript in speech order, and the agent runtime
windows and renders one meeting's token stream in order. Neither consumer needs cross-meeting
ordering for the same tenant.

### Why now, not later

Adopting the composite key now is cheap: no Plan 5+ consumer has shipped yet, so no consumer
group exists to re-cut. Once `embedding-service`'s and `agent-runtime-service`'s consumer groups
are live against these topics in production, a partition-key change means re-partitioning and
carefully draining/re-consuming in-flight offsets without losing or duplicating events — materially
more expensive than landing the ratified key before anything consumes it.

---

## Options considered

| Option | Verdict |
|---|---|
| **(a) Composite `{org_id}:{meeting_id}` on every topic** | Rejected. Every other topic already gets correct per-tenant ordering and replay from `org_id` alone, and per-tenant throughput on those topics does not approach single-partition saturation. Widening the composite key to topics that do not need it needlessly changes the locked "partitioned by tenant" semantics for topics like `decisions`/`actions` (whose per-tenant ordering guarantee matters more than partition spread) and the control-plane/metering topics (whose consumers key their own dedupe and read models on `org_id` alone). It also breaks the general rule that most consumers can reason about "one tenant, one partition" without checking a topic-by-topic exception list. |
| **(b) Keep `org_id` only, everywhere** | Rejected. Does not solve the hot-tenant partition-skew problem for `transcript-chunks`/`agent-stream`, the two highest-count topics, where the risk is real: a single enterprise tenant's peak meeting load can genuinely exceed what one partition sustains. Deferring the fix does not remove the eventual need for it, and deferring past the point where consumer groups exist multiplies the cost of fixing it later (see "Why now, not later" above). |
| **(c) Composite for `transcript-chunks` and `agent-stream` only** (chosen) | Accepted. Solves the actual hot-tenant risk, which is concentrated in exactly these two topics, without disturbing the ordering/replay guarantees the other 19 topics correctly rely on `org_id`-only for. Minimizes the blast radius of a change to locked partition-key semantics: two topics change, nineteen do not, and `org_id` remains present, non-nullable, and the tenant key on all 21. |

`agent-results` was considered and excluded even though [07 §4.1](../07-event-architecture.md)
groups it in the same "high-volume meeting" topic class: its count is lower (one per agent per
window, not per chunk or per token) and it is a durable, at-most-3-day-retention read-model
source rather than a per-chunk/per-token stream, so the same hot-tenant pressure does not apply to
it in the same way. It stays `org_id`-only. If evidence later shows otherwise, that is a separate
decision requiring its own ADR, not an extension of this one.

---

## Decision

The partition key for `transcript-chunks` and `agent-stream` is the composite
`{org_id}:{meeting_id}`. The partition key for the other 19 topics, including `agent-results`, is
unchanged: `org_id` alone. `org_id` remains present and non-nullable in the `EventEnvelopeHeader`
on every topic, including the two composite-keyed ones — it is still the tenant root and the
RLS/cell scoping anchor, and it is still the leading component of the composite key. What changes
is only the claim that `org_id` is the *whole* partition key for these two topics.

[_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md) is amended accordingly, and the
sentence stating the composite sub-key is "not ratified and is not used" is superseded precisely
for `transcript-chunks` and `agent-stream`; it continues to describe every other topic correctly.

---

## Consequences

### Accepted costs

- **Two topics now have a documented exception to "partitioned by tenant alone."** Any code or
  doc that assumed a blanket `org_id`-only rule across all 21 topics must special-case these two.
  [_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md) is the single place that lists the
  exception, so a consumer only needs to check one table.
- **The events-repo schema validator's per-topic invariant must be scoped, not removed.** It
  continues to assert `org_id` is present and non-nullable on every topic (that has not changed);
  it must no longer assert that `org_id` is the whole key for these two topics.
- **Consumer authors for these two topics must partition-key-aware their own tenant-scoped
  aggregation**, since events for one tenant across meetings no longer collapse onto one
  partition. Neither existing consumer (`embedding-service`, `agent-runtime-service`,
  `realtime-gateway`) requires cross-meeting ordering for a tenant, so this is a documentation
  change, not a rework.

### Benefits realized

- Within-meeting speech and token-delta ordering is preserved exactly as before — the ordering
  guarantee that actually matters for these two topics is unaffected.
- A hot enterprise tenant's `transcript-chunks`/`agent-stream` load spreads across up to 180/120
  partitions respectively instead of hashing to one, removing the single-partition throughput
  ceiling for exactly the two topics where it was a real risk.
- The change lands before any Plan 5+ consumer group exists against these topics, so it is a
  schema/validator change now rather than a partition re-cut and drain later.
- `07-event-architecture.md` and `10-realtime-and-transcription.md`, which already described this
  key ahead of canon, are now consistent with canon rather than contradicting it.

### What is unchanged

`org_id`-only partitioning for the other 19 topics, the BACKWARD compatibility rule, the Avro
envelope header shape, the at-least-once + inbox/dedupe delivery contract, and every retention and
partition-count number in [07 §4.2](../07-event-architecture.md). `agent-results` stays
`org_id`-only.

### Follow-ups this ADR creates

1. Propagate the ratified key to [07-event-architecture.md §4.1](../07-event-architecture.md) and
   its §11 summary table, replacing the "proposed, not ratified" language with a citation to this
   ADR.
2. Propagate the citation fix to [10-realtime-and-transcription.md §3.2 and
   §5](../10-realtime-and-transcription.md), which already assumed this key.
3. Scope the `org_id`-non-nullable invariant in `mytheus-events`
   (`codegen/validate.py`) so it keeps failing for any topic that loses `org_id`, without
   asserting `org_id` is the whole key for these two topics.
4. Any Plan 5+ work that stands up a consumer group against `transcript-chunks` or `agent-stream`
   must partition-key-aware its per-tenant aggregation logic per the accepted cost above.
