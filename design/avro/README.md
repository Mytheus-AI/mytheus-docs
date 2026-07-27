# Avro artifact reference

P1 design artifact (Plan 1-2 section 3.1 step 3).

## 1. There are no `.avsc` files here, deliberately

Plan section 3.1 step 3 was written expecting Plan 0 to leave Avro **drafts**. Plan 0 produced the real thing instead: `mytheus-events` ships 21 topic schemas, 22 registered subjects pinned to BACKWARD compatibility, 21 golden round-trip fixtures, and generated Python and TypeScript clients.

Copying those schemas into this repo would create a second definition of a frozen contract in a repo that has **no build, no test suite, and no Schema Registry**. The copy could never be validated, so it could only drift, and a drifted copy inside the naming-authority repo is worse than no copy. It would also violate this repo's own rule that reader docs cite canon and never redefine it.

**`mytheus-events/schemas/{topic}/vN.avsc` is the sole home of every schema.** This page is the index.

The step's acceptance check ("`design/avro/*.avsc` validate against a draft Schema Registry with BACKWARD compat") is therefore satisfied by a stronger check: `make validate && make register && make check-compatibility` green in `mytheus-events` for the four subjects below.

## 2. The four topics this band wires

All Avro plus Schema Registry, subject `{topic}-value`, BACKWARD compatibility, partitioned by `org_id`, each paired with a `{topic}.dlq`. Every value opens with the frozen 8-field `EventEnvelopeHeader` (`event_id`, `idempotency_key`, `org_id`, `schema_version`, `producer`, `occurred_at`, `correlation_id`, `causation_id`).

| Topic | Subject | Record | Producer in this band | Consumer in this band |
|---|---|---|---|---|
| `user-sync-events` | `user-sync-events-value` | `UserSyncEvent` | Clerk, inbound | `identity-tenant-service` via the `inbox` |
| `tenant-events` | `tenant-events-value` | `TenantEvent` | `identity-tenant-service` outbox | None. Later: `billing-metering-service`, `admin-ops-service`, `workflow-service` |
| `meeting-events` | `meeting-events-value` | `MeetingEvent` | `core-api` outbox | None. Later: `meeting-service`, `workflow-service` |
| `notifications` | `notifications-value` | `Notification` | `identity-tenant-service` | None. Later: `notification-service`, `realtime-gateway` |

`user-sync-events` is inbound-only and never an outbound topic: the Clerk path is an inbox problem, not an outbox one. `tenant-events` is log-compacted per `org_id` because it carries org-lifecycle latest-state.

The envelope is defined in [_canon/03-event-taxonomy.md](../../_canon/03-event-taxonomy.md), which is authoritative. It is cited here, not copied.

## 3. Two schema fields are unsatisfiable as written, and must be fixed before the topics are registered

These are real defects found while writing this page, not style preferences. Both are free to fix **now** and permanent if shipped: the topics do not exist yet, no message has ever been written, and no subject version carries production data.

| Field | Problem | Fix |
|---|---|---|
| `MeetingEvent.livekit_room` | Declared non-nullable `string`. The `meetings` table has no such column, and LiveKit rooms are not minted until the real-time band, so the value is genuinely absent for the whole of this band and the next. | Change to `["null", "string"]` with `"default": null`. |
| `TenantEvent.plan_code` | Declared non-nullable `string`. `plans` does not exist until the billing band. | **No schema change.** Emit the literal `"free"`. A newly provisioned tenant genuinely is on the free plan, and `free` is a canonical value of `plans.code`. This is a correct value, not a sentinel. |

`TenantEvent.org_slug` needs no change: `organizations.slug` exists and is `NOT NULL` today.

**Why an empty-string sentinel was rejected for `livekit_room`.** An empty string is a valid `string`, so the schema cannot express "absent" and every consumer must special-case it by convention across a repo boundary with nothing enforcing it. Once written to a retained topic the sentinel is indistinguishable from a genuinely empty value, forever.

**Why the nullable change passes the BACKWARD gate.** BACKWARD asks whether a reader using the new schema can read data written with the old one. Avro schema resolution matches a writer's non-union `string` against the `string` branch of a reader's `["null", "string"]` union, so it reads. The `"default": null` lets the new reader also handle a record where the field is absent. Widening a required field to a nullable union is BACKWARD-safe; the reverse is not. Confirm with `make check-compatibility` before merging rather than taking this paragraph's word for it.

## 4. One free hardening while the topics are still empty

Add an Avro `default` to both event-type enums:

```json
{ "type": "enum", "name": "MeetingEventType",
  "symbols": ["created", "updated", "ended"], "default": "updated" }

{ "type": "enum", "name": "TenantEventType",
  "symbols": ["provisioned", "suspended", "deleted"], "default": "suspended" }
```

Adding an enum symbol always passes the BACKWARD gate. What it does not protect is a **deployed consumer that is behind**: it reads a symbol it has never heard of and throws. An enum `default` makes that reader degrade to the default instead. Plan 3-4 will want `started` and `cancelled` on `MeetingEventType`. Adding the default now costs nothing; adding it after the subject is registered and consumed does not retroactively protect readers already in the field.

## 5. The delivery invariant

Transactional outbox in the same database transaction as the state change, then the relay publishes with partition key `org_id`, then consumers dedupe on `idempotency_key` via the `inbox` table plus `nexus:{org_id}:dedupe:{topic}:{key}`, then poison messages go to `{topic}.dlq`.

Never publish to Redpanda from request code. The state change and the emit must commit atomically or you get lost events on rollback and phantom events on crash.

`decisions` and `actions` are **not** produced in this band. They are emitted only by `core-api`'s outbox, and only on explicit user confirmation.

## 6. Related

- [_canon/03-event-taxonomy.md](../../_canon/03-event-taxonomy.md) is the naming authority for topics, producers, consumers, and retention.
- [07-event-architecture.md](../../07-event-architecture.md) is the narrative.
- `mytheus-events` is the sole realization.
