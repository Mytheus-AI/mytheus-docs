# ADR-0037: Plan 12 build decisions: Jira as the one PM tool, status sync through core-api, unconfigured providers, and staff by allowlist

Plan 10-13 phase P12 builds `integration-service` (OAuth engine, token vault, webhook receiver, Google Calendar, Slack and one PM connector), `ActionFollowUpWorkflow`, and both `admin-ops-service` consoles (plan sections 3.10 to 3.14). The questions below were open; they were decided while building P12 on 2026-11-10.

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md) - `integration_id_map`, `integration_cursors`, `feature_flags`, `ops_audit_log` and `support_grants`, in the same change.
- [_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md) - `actions` stays `core-api`-only; unchanged.
- [21-integration-architecture.md](../21-integration-architecture.md) - the four primitives and the provider detail.
- [ADR-0024](0024-key-custody-and-envelope-encryption.md) - envelope encryption of `oauth_tokens`.
- [ADR-0035](0035-plan-10-context-gap-agent-and-a2a-build.md) - the internal caller-token routes reused here.
- [plan/plan-10-13-collab-integrations-ga.md](../plan/plan-10-13-collab-integrations-ga.md) sections 3.10 to 3.14 and the section 8 DECISION-REQUIRED on Jira versus Linear.

---

## Status

**Accepted** (2026-11-10).

## Context

1. Guardrail R8 allows Calendar, Slack and exactly one of Jira or Linear; section 8 leaves the choice open, and the band's stated end goal is a live Jira connection.
2. Section 3.11 says inbound Jira status changes emit `actions {status sync}`. Canon makes `core-api` the sole producer of `actions`, and the `Action` schema's status is pinned to `confirmed` by the events validator: the topic carries only user-confirmed facts.
3. OAuth needs an app registered with each provider. The platform has none yet, and a connector that cannot run should not look connectable.
4. Section 3.13 says the internal-ops console is for "NEXUS staff only", but no staff role exists in Clerk, RBAC or canon.
5. The day-10 escalation needs a manager, and the data model has no reporting edge (section 8 DECISION).

## Decision

**D1. Jira is the PM connector.** Linear is not built; the settings page shows it as an unavailable option of the single PM radio, and a second PM connection is refused (409).

**D2. Jira status enters through `core-api`, not `actions`.** `integration-service` calls the internal `POST /internal/actions/{id}/external-status`; `core-api` updates `action_items.status` (`done`, or back to `confirmed`), audits `action.status_synced`, and publishes nothing. `actions` keeps carrying only confirmations. Loop suppression uses `integration_id_map.last_synced_status`: a webhook whose status equals it is dropped. NEXUS to Jira status sync is not built in P12: it is not in the P12 Definition of Done, and it would need a non-confirmation event on `actions`.

**D3. A provider is connectable only when its app is configured.** `integration-service` reads each provider's client id and secret from its environment; without them the provider reports `configured: false` and connect returns 409.

**D4. OAuth callbacks and webhooks are public paths authenticated by their own proof.** `/oauth/{provider}` is verified by the HMAC-signed state and PKCE, `/webhooks/{provider}` by the provider's signature; the gateway passes them without a Clerk JWT. Everything else stays JWT-gated.

**D5. Mytheus staff are an allowlist.** `admin-ops-service` treats a user as staff only when their id is in `MYTHEUS_STAFF_USER_IDS`; every `/admin/ops/*` call, reads included, is appended to `ops_audit_log`; ops views are aggregates over control-plane data, never tenant content.

**D6. The follow-up escalation is best-effort.** The manager is an owner or admin in the assignee's team's department, then the meeting organizer, then an org admin.

## Consequences

- The Jira demo works once an Atlassian OAuth app is registered and its credentials are set; until then the page says "Not configured".
- A status change made in Mytheus does not move the Jira issue; the follow-up workflow still sees it because it re-reads status from `core-api`.
- Granting someone staff access is a deployment change, which is deliberate for an MVP with one operator.
