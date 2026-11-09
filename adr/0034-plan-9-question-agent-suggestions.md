# ADR-0034: Plan 9 QuestionAgent build decisions: per-participant suggestions, the stored band, owner-only actions, and participant-scoped grounding

Plan 7-9 phase P9 builds `QuestionAgent` (plan section 3.10): real-time question suggestions, the `agent_suggestions` rows behind them, the routes that act on them and the sidebar card. The plan and canon leave several questions open. They were decided while building it on 2026-11-09, and this ADR records them.

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md) - the `agent_suggestions` row gains `confidence`, `rationale` and `citations` in the same change.
- [_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md) - `agent-results` and its idempotency key `agentresult:{meeting_id}:{window_id}:{user_id}:{agent}`, unchanged.
- [08-ai-agent-architecture.md](../08-ai-agent-architecture.md) sections 2.1 and 6 - the agent and the confidence policy.
- [ADR-0030](0030-plan-7-llm-response-cache-and-personal-listener-gating.md) - D3 (`core-api` decides the band) and D5 (per-user delivery).
- [ADR-0032](0032-plan-8-decision-and-action-candidates.md) - D2 (a list in `payload_json`), D3 (deterministic ids), D7 (lowercase status codes), which this ADR reuses.
- [ADR-0033](0033-plan-9-rag-retrieval-upgrades.md) - D1 (participant-scoped retrieval), D3 (four citation kinds).
- [plan/plan-7-9-agents-and-monetization.md](../plan/plan-7-9-agents-and-monetization.md) section 3.10.

---

## Status

**Accepted** (2026-11-09).

## Context

1. `agent_suggestions` is per `user_id`, but section 3.10 says the agent "runs per-meeting". A question worth asking depends on who is asking: what a product manager needs to know is not what an engineer needs to know.
2. The confidence policy (doc 08 section 6) stores the `0.4-0.7` band but does not show it. The canon row had no column to tell a stored-but-hidden suggestion from a shown one.
3. The frozen `AgentResult.kind` enum already has `suggestion`. Adding a kind would change a frozen schema.
4. `agent_suggestions` is partitioned by `created_at`, so its primary key is `(id, created_at)`. `ON CONFLICT (id)` cannot be used as it is for `decisions`.
5. Who may act on a suggestion is not stated. Section 3.10 says only that Save-for-later persists into the next meeting.
6. ADR-0033 D1 makes retrieval participant-scoped. A grounding call made under a service identity would either see nothing or see too much.

## Decision

**D1. `QuestionAgent` is a participant-scope agent.** One instance per participant per meeting, like `PersonalListenerAgent`. It is tuned to that participant's role, skills and bio from `get_user_context`, and the result's `user_id` is the participant. "Per-meeting" in section 3.10 is read as "runs during each meeting", not "one instance per meeting".

**D2. The pipeline is heuristic, then gate, then proposal.** Per closed 30-second window: a context-gap heuristic with no model call (the window must have at least two turns and 25 words, and after a window that surfaced something the agent skips the next one, `QUESTION_COOLDOWN_WINDOWS` = 1); then the Haiku gate `question_gate@v1`; then Sonnet `question_agent@v1`, which returns at most two suggestions, each with `priority` 1-3 (1 is most urgent), `confidence` and a one-line `rationale`. Every call goes through the router, so the budget gate and `usage-events` apply as for every agent.

**D3. Kind `suggestion`, a list in `payload_json`, deterministic ids.** The record's `kind` is the existing `suggestion` symbol, so no schema change is needed. Suggestions travel as `{"suggestions": [{id, question, priority, confidence, surfaced, rationale, citations}]}` (the ADR-0032 D2 shape). Ids are `uuid5(namespace, "QuestionAgent:{window_id}:{user_id}:{index}")`: the user is part of the name because two participants can get suggestions from the same window.

**D4. The band is stored as `confidence` and applied at read time.** `core-api` stores suggestions from `0.4` up (re-deriving `silent` from `tenant_settings.agent_confidence_overrides`, ADR-0030 D3) with their `confidence`, and the read route returns only open suggestions above the tenant's `surface` threshold. This is the same model as decision candidates (ADR-0032 D6). A `surfaced` flag was not stored, because changing a tenant's override must not need a backfill.

**D5. Idempotency is the inbox, plus a `NOT EXISTS` on the id.** The inbox claim in the consumer transaction already makes a redelivered record a no-op. Because the partitioned primary key includes `created_at`, the insert also checks that no row with the same `id` exists. Under the inbox claim that check cannot race.

**D6. Suggestions are private to their owner.** Only the suggestion's `user_id` can read it or act on it. Another user, even in the same meeting and tenant, gets 404, not 403: a suggestion is that user's private prompt, so whether one exists is not shown. Another tenant's id is 404 through RLS. `realtime-gateway` sends a surfaced suggestion only to that user's sockets, as for notes (ADR-0030 D5), not to the room (unlike candidates, ADR-0032 D4).

**D7. Status transitions.** `open` can go to `asked`, `dismissed` or `saved`. `saved` can go to `asked` or `dismissed`, because it reappears in a later meeting to be acted on. `asked` and `dismissed` are final. Repeating the current status returns 200 and writes nothing. Any other move is 409. Every change writes one audit row. Nothing is published: a suggestion is never a commitment.

**D8. Save-for-later is a read rule.** `GET /api/v1/suggestions?meeting_id=` returns the caller's open, surfaced suggestions for that meeting plus every suggestion the caller has `saved`, from any meeting. That is how a saved question reappears in the user's next meeting without copying the row.

**D9. Grounding is optional, participant-scoped and passages only.** Behind `QUESTION_RAG_GROUNDING` (off by default), the agent asks `rag-service` for passages relevant to the window before the proposal call. It always passes the participant's `user_id` (ADR-0033 D1), never a service identity, and requests passages only (`generate: false`): no Sonnet answer call is made for grounding. The passages go to the proposal prompt as numbered context, and the citations of the passages a suggestion used are stored in `citations`, in any of the four ADR-0033 D3 kinds. A grounding failure or timeout is not an error: the window is proposed without context.

## Consequences

- No Avro change: `mytheus-events` is untouched.
- One migration in `core-api` adds three nullable columns and an index on `(org_id, user_id, meeting_id, status)` per partition.
- QuestionAgent spends one Haiku call per participant per window that passes the heuristic, and one Sonnet call when the gate passes. The heuristic and the cooldown keep this under the PersonalListener cost per participant.
- A saved suggestion has no expiry. If that becomes noisy, a later change can cap the saved list or age it out. The query is bounded by a limit now.
- Another participant can never see, ask, save or dismiss your suggestion.
