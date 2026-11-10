# ADR-0036: Plan 11 build decisions: ProactiveAgent on confirmed facts, alerts as notifications, and the per-user priority queue

Plan 10-13 phase P11 builds `ProactiveAgent`, the `realtime-gateway` per-user priority queue and shed order, `notification-service` delivery and the alert UI (plan sections 3.6 to 3.9). The plan leaves the questions below open. They were decided while building P11 on 2026-11-10, and this ADR records them.

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md) - `notification_log` and `notification_preferences`, in the same change.
- [_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md) - `agent-runtime-service` becomes a consumer of `decisions` and `actions`, in the same change.
- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md) - `notification-service` owns `notification_log`, delivery receipts and user channel prefs, and serves `GET /notifications` and `PATCH /notifications/:id`. Unchanged.
- [ADR-0032](0032-plan-8-decision-and-action-candidates.md) D3 (deterministic ids), [ADR-0035](0035-plan-10-context-gap-agent-and-a2a-build.md) (the core-api internal caller-token routes reused here).
- [plan/plan-10-13-collab-integrations-ga.md](../plan/plan-10-13-collab-integrations-ga.md) sections 3.6 to 3.9.

---

## Status

**Accepted** (2026-11-10).

## Context

1. Section 3.6 reasons over "a newly confirmed decision" and confirmed actions, but the agent runtime only consumed transcript windows; canon did not list it as a consumer of `decisions` or `actions`.
2. Section 3.9 says alert acknowledge and dismiss "write through to `core-api`", but no canon table holds an alert, while the service catalog already gives `notification-service` the `notification_log`, the user channel prefs and the `GET /notifications` / `PATCH /notifications/:id` surface.
3. `AgentResult` is per `user_id`, and an alert usually concerns several people.
4. `plans.features.proactive_agent` is on only for Enterprise, so a Free or Pro tenant would never see an alert.
5. Slack delivery needs the OAuth token `integration-service` holds, and that service is built in P12. Email needs a provider the platform does not configure yet.

## Decision

**D1. ProactiveAgent reacts to confirmed facts.** It consumes `decisions` and `actions` (group `agent-runtime-proactive`) for the conflict, workload and dependency checks, and runs per closed window only for missing stakeholders. It never writes `decisions`, `actions` or any table; it emits `agent-results` with `kind = alert`.

**D2. Conflict is a judged contradiction over two candidate sources.** Candidates are the AGE `FOLLOWS_FROM` lineage from `knowledge-graph-service` and a semantic search of confirmed decisions (`decisions__{org_id}`) through `rag-service`; if the graph is unreachable within 800 ms only the second source is used (the R1 hot degradation). One Sonnet call decides whether the new decision contradicts a candidate. A decision that names the candidate in `supersedes` replaces it and is never a conflict.

**D3. Workload and dependency are deterministic.** A `core-api` internal route returns the assignee's open action count and the action's prerequisites; the threshold (6 by default) and an unfinished prerequisite raise the alert with no model call.

**D4. One alert result per recipient, with a deterministic id** `uuid5(CANDIDATE_NS, "ProactiveAgent:{alert_type}:{subject_id}:{recipient}")`. Priorities follow the frozen colour language: conflict 1 (red), workload and dependency 2 (yellow), missing stakeholder 3 (blue). The confidence bands of doc 08 section 6 apply.

**D5. Alerts are notifications.** `notification-service`, already a canon consumer of `agent-results` for priority alerts, stores each surfaced alert as a `notification_log` row under the alert's id and delivers it per the user's `notification_preferences`. Read and dismiss go through `PATCH /notifications/:id`, the surface the service catalog already assigns, instead of `core-api` as section 3.9 says: canon wins, and no second store of the same alert is created.

**D6. The gateway delivers alerts through the per-user priority queue** `nexus:{org_id}:queue:agent:{user_id}`, a sorted set scored by priority then age, drained to the recipient's own sockets highest priority first, and kept until the user joins. Under backpressure `agent-stream` deltas are dropped first; captions, cards, suggestions, context fills and alerts never are.

**D7. Channels that cannot deliver yet record a skip.** Email sends through an SMTP adapter when `SMTP_URL` is set; Slack waits for P12. Each attempt, delivered or skipped, emits one `notification-delivery-events` receipt.

**D8. The plan gate has a deployment override.** `ProactiveAgent` runs when the tenant's plan enables `proactive_agent` or when `PROACTIVE_AGENT_ALL_PLANS=1` is set on `agent-runtime`. The budget gate applies either way: `throttled` or `blocked` suspends the agent. The production MVP sets the override because Mytheus has no paying tenants yet; it is removed when plan enforcement matters.

## Consequences

- Conflict detection works without the graph, at lower recall, and improves when the `kg` profile runs.
- An alert exists once, in `notification_log`, whatever channels deliver it; the UI rehydrates from it after a reconnect.
- Email and Slack are wired but silent until configured; the receipts show why.
- The override makes the Enterprise gate a configuration choice for now.
