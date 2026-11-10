# ADR-0035: Plan 10 build decisions: ContextGapAgent per participant, a signed agent identity, the PDP over identity gRPC, and agent_collaborations as the fill's home

Plan 10-13 phase P10 builds `ContextGapAgent`, the A2A protocol around it, the `agent_collaborations` log and the real `MeetingLifecycleWorkflow` activities (plan sections 3.1 to 3.5). [ADR-0013](0013-agent-to-agent-authorization.md) fixes the authorization model; it leaves the questions below open. They were decided while building P10 on 2026-11-10, and this ADR records them.

## Related documents

- [ADR-0013](0013-agent-to-agent-authorization.md) - the PEP/PDP split, the three rules, the `agent_authz_scopes` schema and the 60 s decision cache. Unchanged by this ADR.
- [_canon/02-data-model.md](../_canon/02-data-model.md) - `agent_collaborations` gains its full column list, `agent_authz_scopes` and the `nexus:{org_id}:a2a:grant:*` key are recorded, in the same change.
- [_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md) - `agent-results` `kind` gains `context_fill`; `agent-requests` / `agent-responses` unchanged.
- [ADR-0032](0032-plan-8-decision-and-action-candidates.md) D2 and D3, [ADR-0033](0033-plan-9-rag-retrieval-upgrades.md) D1 and D3, [ADR-0034](0034-plan-9-question-agent-suggestions.md) D4 and D6 - patterns reused here.
- [plan/plan-10-13-collab-integrations-ga.md](../plan/plan-10-13-collab-integrations-ga.md) sections 3.1 to 3.5.

---

## Status

**Accepted** (2026-11-10).

## Context

1. ADR-0013 says identity is "derived from `requester_agent_id`'s bound principal", held "in process". The requester and the responder may run in different `agent-runtime-service` processes, so an in-process binding alone cannot be checked by the responder.
2. "A context gap" is not defined beyond "a reference to context a participant lacks". Running the full chain for every participant on every window would exceed `L4` (12 synthesis calls per meeting).
3. Retrieval is participant-scoped (ADR-0033 D1), so the RAG hop sees only what the requester could already see, and "related-meeting agents" needs someone to say which meetings are related.
4. The frozen `AgentResult.kind` enum has no value for a context fill, and the agent tier owns no tables, so the collaboration log needs a carrier to `core-api`.
5. The plan shows a "Context added from B's agent" card but names no table for it.
6. ADR-0013 names `grantee_role` "the requester's RBAC role" but `get_user_context` returns both a free-text domain `role` and the RBAC tier.

## Decision

**D1. The agent identity is a signed id.** `requester_agent_id` and `responder_agent_id` are `cga:{org_id}:{user_id}:{sig}`, where `sig` is an HMAC-SHA256 over `cga:{org_id}:{user_id}` with a secret only `agent-runtime-service` holds (`AGENT_IDENTITY_SECRET`). The PEP derives org and user only from an id whose signature verifies; the envelope `org_id` is cross-checked against it (ADR-0013 rule 1). A bad signature has no derivable identity: the request goes to `agent-requests.dlq` and is logged, never served.

**D2. `grantee_role` is the RBAC tier** (`owner`/`admin`/`member`, `UserContext.rbac_tier`). The domain role is free text and cannot key a grant. `target_scope` is resolved by the PEP from both users' `get_user_context`: same user `self`, a shared team `same_team`, same department `department`, otherwise `org`.

**D3. The PDP is a method on the identity gRPC service** (`IdentityService.Authorize(org_id, grantee_role, query_class, target_scope) -> allow|deny`), next to `get_user_context`, which the runtime already calls. Any error or unknown value is a deny. Grants are managed over `identity-tenant-service` REST (`/api/v1/agent-authz-scopes`, owner and admin only), including a template endpoint that adopts the platform default at a chosen scope. Every change deletes the tenant's `nexus:{org_id}:a2a:grant:*` keys.

**D4. One agent per participant, one gate per window.** A deterministic detector looks for references to earlier context; one Haiku gate per window (shared by all participants) turns it into a standalone query. A participant "lacks" the context when their own participant-scoped retrieval returns nothing above a threshold; at most two such participants per window become requesters. Participants who already have the context get nothing.

**D5. The chain is peer, related meeting, RAG, silent**, inside one 1.5 s deadline. Peers are the other participants of the meeting. Related-meeting responders are the people the requester met most in their recent ended meetings who are not in this one, from a `core-api` internal route; without it the hop is skipped. The RAG hop uses the requester's own passages. A responder answers from its own user's participant-scoped retrieval, with passages and citations, no model call, so the round-trip stays inside the budget; the requester makes the one synthesis call (`L4`). Budget `throttled`, or `CONTEXT_GAP_A2A=0` (the R6 descope as configuration), skips straight to RAG.

**D6. `agent-results` carries the log.** `AgentResult.kind` gains `context_fill`, appended last with enum default `note` (BACKWARD-safe). One result per window and requester carries every hop attempt and the optional fill in `payload_json`. Responder-side blocks (spoof, foreign org) are reported under the derived requester's org.

**D7. `agent_collaborations` is the fill's home.** One row per hop attempt, grants and denials alike, with `success`, `result` and `latency_ms`. The fill (content, confidence, rationale, citations) sits on the row that produced it and is private to `requester`; the collaboration log shown to participants and admins never includes it. The band is re-derived in `core-api` from the stored confidence, like ADR-0034 D4. Dismiss sets `dismissed_at`; only the requester may dismiss (404 to anyone else).

**D8. Each attempt is audited** by `core-api` in `audit_logs` with `actor_type = 'agent'`: `a2a.request.allowed`, `a2a.request.denied`, `a2a.request.blocked`.

## Consequences

- A forged envelope cannot cross a tenant: the org comes from a signature only the runtime can make, and topics are tenant-partitioned besides.
- The responder answers without a model call, so a peer hop costs one retrieval; the meeting's synthesis calls stay at most two per window and are metered and budget-gated like every other agent call.
- A2A is off for every tenant until an admin grants scopes. The template makes that one action.
- Rotating `AGENT_IDENTITY_SECRET` invalidates in-flight requests only; agent ids are minted per request.
- The related-meeting hop is a heuristic (people met most recently), not a semantic search over other people's meetings, which participant scoping would forbid without a grant anyway.
