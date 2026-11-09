# ADR-0031: Plan 8 entry decisions: a dedicated AGE Postgres in production, `supersedes` on the `Decision` event, and the action mark-done route

Plan 8 (decisions, actions and the knowledge graph) has one open DECISION-REQUIRED (#3, risk R1) and two gaps found while starting the knowledge-graph half. The maintainer resolved all three on 2026-11-09. This ADR records the answers and closes DECISION-REQUIRED #3 of [plan-7-9-agents-and-monetization.md](../plan/plan-7-9-agents-and-monetization.md).

## Related documents

- [ADR-0005](0005-graph-store-age-with-migration.md) - AGE now, with the Neptune / Aura and relational exits. Not superseded: D1 below takes the "self-host Postgres + AGE" fallback ADR-0005 already lists.
- [ADR-0016](0016-self-hosted-vps-mvp-substrate.md) - the self-hosted VPS, which is why the managed-extension gate R1 was written about does not bind any more.
- [_canon/02-data-model.md](../_canon/02-data-model.md) - `decisions.supersedes`, the graph's physical home and the `Decision` node properties, updated in the same change.
- [_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md) - the `decisions` row, updated in the same change.
- [_canon/06-tech-decisions.md](../_canon/06-tech-decisions.md) - the "Graph store" row, updated in the same change.

---

## Status

**Accepted** (decided by the maintainer 2026-11-09).

## Context

1. **R1 is realised.** Production Postgres is `supabase/postgres:15.8.1.060`. The image ships no `age.control` and no `age.so` (checked 2026-11-06 across every layer), so `CREATE EXTENSION age` fails there. Locally the Plan 5-6 work already runs the graph in a dedicated `apache/age:release_PG16_1.6.0` container addressed by `AGE_DATABASE_URL`, with `libs/shared/age.py` enforcing `$org_id` binding, parameter-only tenant data and the frozen labels, and a passing live test.
2. **`FOLLOWS_FROM` had no source on the bus.** Plan 3.7 builds `FOLLOWS_FROM` from `evt.supersedes`, but the frozen `Decision` Avro schema has no such field, and the `decisions` canon row has no such column. Without it the lineage template has nothing to walk, and ADR-0005's promise that the graph is rebuildable from confirmed rows is false for lineage.
3. **My Tasks needs a "mark done" write** on `action_items`, and the plan names no route for it. The confirm, edit and dismiss routes are the candidate-confirmation surface; done is a progress change on an already confirmed row.

## Decision

**D1. Production runs `nexus_knowledge` in a dedicated `apache/age` Postgres container** on the same VPS, beside the system-of-record Postgres. This is ADR-0005's "AGE on self-hosted Postgres" alternative, taken at the P8 checkpoint as R1's pre-authorised fallback. `knowledge-graph-service` reaches it through `AGE_DATABASE_URL`, a variable deliberately distinct from `DATABASE_URL` so a misconfiguration cannot point graph writes at the relational core. The relational inbox and dedupe rows stay in the system-of-record Postgres.

Chosen over the relational fallback because the AGE client, its tenant guards and its live test already exist and pass, the store is self-hosted (ADR-0016) so no managed-extension gate applies, and the graph is a projection rebuildable from confirmed rows, so losing ADR-0005's "one backup domain" property costs a rebuild, not data. Chosen over Neo4j Aura because it keeps tenant data on the VPS and needs no new driver.

**D2. `Decision` gains an optional `supersedes`**, `["null", {"type": "string", "logicalType": "uuid"}]` with default `null`, appended after `confirmed_by`. This is a BACKWARD-compatible addition, the same shape as ADR-0029 D4's `model_tier`. `decisions` gains the nullable column `supersedes uuid REFERENCES decisions(id)` that plan 3.6 step 1 already drafts. `core-api` sets it on confirm when the user says the new decision replaces an earlier one; `knowledge-graph-service` writes `FOLLOWS_FROM` from it and marks the prior `Decision` node `status = superseded`. The frozen schema name, topic and idempotency key do not change.

**D3. `POST /api/v1/actions/{id}/done`** moves an action from `confirmed` to `done`, for the assignee or a tenant admin. It emits nothing: the `actions` topic carries confirmations only (its `status` enum is `{confirmed}`), and a progress change is not a new fact for the graph. It lives in its own `core-api` route module, apart from the confirm, edit and dismiss routes.

**D4. The `Decision` graph node carries `meeting_id`, `content` and `confirmed_at`** besides its canon key properties, so the lineage template can answer a lineage view without a second read of `decisions`. They are copied from the confirmed event and are rebuildable like every other property.

**D5. `embedding-service` consumes `decisions`** to embed confirmed decisions into `decisions__{org_id}`. Canon already makes `decisions__{org_id}` "embedded on confirm" from the `decisions` topic; the event taxonomy row did not list the consumer.

**D6. The consumer dedupe cache key is scoped by consumer group:** `nexus:{org_id}:dedupe:{topic}:{consumer_group}:{idempotency_key}`. The durable inbox already keys on the consumer group, but the Redis cache did not, so when two groups consume one topic (`transcript-chunks`: `embedding-service` and `search-service`; `decisions`: `knowledge-graph-service` and `embedding-service`) whichever group handled a message second within 24 hours read the other group's key as its own duplicate and skipped its effect. Found while adding the second `decisions` consumer. The `nexus:{org_id}:` prefix is unchanged.

**D7. `knowledge-graph-service` consumes `meeting-room-events`** and writes `ATTENDED` (`Person` to `Meeting`) on `joined` when `participant_id` is a user id. Canon creates `ATTENDED` "when a participant joins" and names no source; this is that source. A join is an observed fact, not an agent candidate, so the confirmation rule is not touched.

## Consequences

- **Positive.** The graph can be built now, against code that already passes its tenant-isolation tests, and lineage has a durable relational source.
- **Negative, accepted.** The graph has its own backup volume and is not in the relational PITR timeline. Recovery is a rebuild from confirmed `decisions` and `action_items` rows.
- **Negative, accepted.** One more Postgres container on the single VPS.
- **Follow-up.** `mytheus-events` `schemas/decisions/v1.avsc` gains `supersedes` in the same change set, and the subject is re-registered. The `core-api` confirm route (Part A of Plan 8) populates it.
- **Follow-up.** [Risk Register](../23-risk-register.md) R1: realised, fallback taken, owner the maintainer.

> [SaaS-scale adjustment] The PDF and ADR-0005 co-locate the graph in the managed primary. Production now runs it in a dedicated self-hosted AGE Postgres, the R1 fallback ADR-0005 already names.
