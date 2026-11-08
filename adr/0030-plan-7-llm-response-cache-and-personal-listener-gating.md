# ADR-0030: Plan 7 build decisions: the LLM response cache, PersonalListener relevance and confidence gating, streaming only surfaced notes, and per-user live delivery

Plan 7-9 work item 3 (`PersonalListenerAgent`) and work item 4 (the agent sidebar) left four questions the plan text does not settle. They were decided while building the items on 2026-11-08. This ADR records them.

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md) - the Redis namespace table gains `nexus:{org_id}:llmcache:{prompt_ref}:{input_hash}` in the same change; `tenant_settings.agent_confidence_overrides` is the column D3 gives a shape.
- [ADR-0004](0004-llm-cost-control-pipeline.md) - stage 4 asks for "semantic + Anthropic prompt caching". D1 says what the semantic half is in P7.
- [ADR-0029](0029-plan-7-budget-scope-tier-mapping-and-metering-ownership.md) - the tier map this cache keys on.
- [08-ai-agent-architecture.md](../08-ai-agent-architecture.md) sections 2.1 and 6 - the relevance pre-gate and the confidence surfacing policy.

---

## Status

**Accepted** (2026-11-08).

## Context

1. ADR-0004 stage 4 routes "semantically near-duplicate windows" through a semantic cache before any model call, but canon defined no Redis key for it, so the agent runtime shipped with Anthropic prompt caching only.
2. Doc 08 says PersonalListener extracts "above the per-tenant relevance threshold". No number is frozen, and the AI tier cannot read `tenant_settings` (it writes and reads no Postgres).
3. The confidence policy (`>0.7` surfaced, `0.4-0.7` silent log, `<0.4` dropped) is overridable per tenant through `tenant_settings.agent_confidence_overrides`, a `jsonb` column with no defined shape. The same reason as 2 means the agent cannot apply the override.
4. `agent-stream` deltas are live text. Streaming a note before its confidence is known would put a note on screen that the policy then refuses to surface.
5. `realtime-gateway` Socket.io rooms are per meeting. A PersonalListener note belongs to one participant; broadcasting it to the room would show one person's notes to everyone.

## Decision

**D1. The P7 "semantic cache" is an exact-match response cache on normalized input.** Key: `nexus:{org_id}:llmcache:{prompt_ref}:{input_hash}`, where `prompt_ref` is `{prompt_key}@{version}` and `input_hash` is the SHA-256 of the tier, the model, the system prompt and the user message after redaction, lowercasing and whitespace collapsing. Value: the completion text and its token counts. TTL 1 hour. A hit calls no model, emits no `usage-events` record and charges no budget, because no tokens were spent. The budget check still runs first, so a `blocked` tenant gets nothing from the cache either.

Why not embedding similarity now: it needs an `embedding-service` round trip per window per participant on the hot path, and two live windows are rarely near-duplicates. The concrete win in P7 is replay: a consumer restart replays every open window, and without this cache it pays for each one twice. Embedding similarity can be added later under the same namespace if a measured hit rate justifies the extra hop.

**D2. The relevance pre-gate threshold defaults to 0.5**, set by `PERSONAL_LISTENER_RELEVANCE_THRESHOLD` on `agent-runtime-service`. It is a tunable, not a capacity number: canon's gate-pass `0.35` is the share of windows expected to pass, and the eval harness measures relevance precision and recall against role labels at this threshold. A per-tenant override needs `tenant_settings` mirrored into Redis for the AI tier and is not built in P7.

**D3. `core-api` decides `surfaced`; the agent only drops.** The agent drops `<0.4` (no `agent-results` record) and emits everything else with `payload_json.surfaced` set from the default bands. `core-api`, which can read `tenant_settings`, recomputes `surfaced` when it writes `worklogs`, using `agent_confidence_overrides` when present. Its shape is keyed by agent class:

```json
{"PersonalListenerAgent": {"surface": 0.8, "silent": 0.5}}
```

`surface` replaces `0.7`, `silent` replaces `0.4`; a note below `silent` is not persisted. An override can only tighten what the agent already filtered: a `silent` below `0.4` has no effect, because those notes never left the agent.

**D4. Only notes in the surfaced band stream.** The extraction prompt makes the model write its confidence on the first line, before the note. The agent parses that line first and forwards the remaining deltas to `agent-stream` only when the confidence is above `0.7`. A silent-band note is persisted and never appears live.

**D5. Live agent output is delivered per user over the existing meeting channel.** `realtime-gateway` consumes `agent-stream` and `agent-results` in one consumer group, so each record is handled by one pod. That pod publishes it on `nexus:{org_id}:pubsub:meeting:{meeting_id}` tagged with the target `user_id`, and every pod emits it only to its local sockets whose resolved internal user id matches. No new Redis key is introduced. Only `agent-results` records with `payload_json.surfaced = true` are forwarded; on reload, `GET /api/v1/meetings/:id/worklogs` (which honours D3) is the source of truth.

## Consequences

- One Redis key is added to canon. Every write carries the `nexus:{org_id}:` prefix, so a tenant flush clears it.
- A note surfaced live under the default bands but hidden by a stricter tenant override disappears on the next reload. This is accepted until `tenant_settings` is mirrored to the AI tier and the gateway.
- The cache makes a replayed window free and deterministic. It also means a prompt change takes effect immediately, because the prompt version is part of the key.
- The window text sent to Anthropic names speakers as `You` and `Participant N`, never by user id or name, which keeps identities out of the provider request even before redaction.

## Alternatives considered

| Option | Why not |
|---|---|
| Embedding-similarity cache now | An extra network hop per window per participant for a hit rate nobody has measured. Deferred, same namespace. |
| Agent applies tenant overrides | The AI tier reads no Postgres; it would need a new mirror of `tenant_settings`. `core-api` already has the row. |
| Stream every extraction and retract below 0.7 | The note is visible before it is retracted, which is exactly what the confidence policy exists to prevent. |
| A per-user Socket.io room | Adds a second room type and a second authz path on join. Filtering on the already-authenticated meeting channel needs neither. |
