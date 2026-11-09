# ADR-0032: Plan 8 part A build decisions: decision and action candidates, filling `supersedes`, and room-wide candidate delivery

Plan 7-9 phase P8 part A (`DecisionAgent`, `ActionAgent`, the `core-api` confirm path and the meeting-room confirm cards) left questions the plan text does not settle. They were decided while building it on 2026-11-09. This ADR records them.

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md) - the `decisions`, `action_items` and `agent_suggestions` rows gain the columns below in the same change.
- [_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md) - `decisions` and `actions` keep `core-api` (outbox, on confirm) as their only producer.
- [08-ai-agent-architecture.md](../08-ai-agent-architecture.md) sections 2.2 and 6 - no silent commits; the confidence policy.
- [ADR-0031](0031-plan-8-graph-store-decision-supersedes-and-mark-done.md) - Plan 8 part B's entry decisions; its D2 adds `supersedes` to `Decision`, which D1 here fills.
- [ADR-0030](0030-plan-7-llm-response-cache-and-personal-listener-gating.md) - D3 (`core-api` decides the band) and D5 (per-user delivery), which D4 here narrows for candidates.
- [plan/plan-7-9-agents-and-monetization.md](../plan/plan-7-9-agents-and-monetization.md) sections 3.5, 3.6 and 3.8.

---

## Status

**Accepted** (2026-11-09).

## Context

1. The plan's `decisions` DDL has `supersedes` and says the confirmed event feeds `FOLLOWS_FROM`, but the frozen `Decision` Avro schema has no such field, so the graph cannot learn lineage from the event.
2. The canon idempotency key for `agent-results` is `agentresult:{meeting_id}:{window_id}:{user_id}:{agent}`: one result per agent per window. A 30-second window can hold more than one decision or action.
3. A candidate card is shown live from `agent-results`, but the row it confirms is written separately by the `core-api` consumer. The card has to name that row.
4. ADR-0030 D5 delivers agent output to its owner only. A decision is the meeting's, not one participant's.
5. Who may confirm, edit or dismiss is not stated anywhere.
6. Doc 08 section 6.1 says the `0.4-0.7` band is a silent log. For candidates there is no per-user worklog to put it in.

## Decision

**D1. `core-api` fills `Decision.supersedes` on confirm.** The field itself is [ADR-0031](0031-plan-8-graph-store-decision-supersedes-and-mark-done.md) D2 (agreed between the two P8 developers). The caller of `POST /api/v1/decisions/{id}/confirm` may name a confirmed decision of the same tenant that the new one replaces; `core-api` stores it in `decisions.supersedes` and puts it on the event, and `knowledge-graph-service` writes `FOLLOWS_FROM` from it.

**D2. Several candidates per window travel as a list in `payload_json`.** `{"candidates": [{id, content, confidence, surfaced, ...}]}`; the record's `content` and `confidence` are the strongest candidate's. No key or schema change.

**D3. Candidate ids are deterministic.** `uuid5(namespace, "{agent}:{window_id}:{index}")`. The agent assigns them, `core-api` inserts the `proposed` row under that id (`ON CONFLICT DO NOTHING`), and the live card confirms by it. A replayed window produces the same ids, so it writes nothing new.

**D4. Candidates are delivered to the whole meeting.** `realtime-gateway` forwards surfaced candidates as a new socket event, `agent-candidate`, to every socket in the meeting, scoped by org and meeting as before. PersonalListener notes stay per user (ADR-0030 D5 unchanged for notes).

**D5. Any participant of the meeting may confirm, edit or dismiss its candidates.** A non-participant in the tenant gets 403 `NOT_A_PARTICIPANT`; another tenant's id is 404. The first confirm wins; a second confirm returns the row and publishes nothing; confirming a dismissed candidate is 409.

**D6. The silent band is a `proposed` row that is never surfaced.** `core-api` persists candidates from `0.4` up, re-deriving the band with `tenant_settings.agent_confidence_overrides` (ADR-0030 D3), and its candidate read returns only the surfaced band. Below `0.4` nothing is written. A `proposed` row is not a commitment: it publishes nothing and creates no edge.

**D7. `agent_suggestions.status` uses lowercase codes.** `open`, `asked`, `dismissed`, `saved`: the canon's Ask / Dismiss / Save-for-later plus the not-yet-acted state the plan's DDL adds.

## Consequences

- No silent commits holds: the only writers of the `decisions` and `actions` topics are the two `core-api` confirm endpoints, through the outbox, keyed `decision:{id}:confirmed` and `action:{id}:confirmed`.
- A confirm that arrives before the consumer has written the row gets 404; the client retries briefly.
- Another participant's confirm or dismiss is not pushed to the rest of the room; their card resolves when they act (a confirm returns the confirmed row, a dismiss of a confirmed candidate is 409) or on the next re-hydrate.
