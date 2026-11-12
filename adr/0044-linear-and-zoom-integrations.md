# ADR-0044: Linear and Zoom integrations

## Related documents

- [_canon/06-tech-decisions.md](../_canon/06-tech-decisions.md) section 2.13 (Project management, the R8 guardrail).
- [_canon/02-data-model.md](../_canon/02-data-model.md): `integration_connections`, `integration_id_map`, `meetings.source`.
- [21-integration-architecture.md](../21-integration-architecture.md) sections 5 and 9.
- [ADR-0037](0037-plan-12-integrations-follow-ups-and-consoles-build.md) (the integration framework, D3 reach providers outside R8), [ADR-0040](0040-polish-program.md) (captured meetings, `meetings.source`).
- Design: [docs/superpowers/specs/2026-11-12-linear-and-zoom-integrations-design.md](../docs/superpowers/specs/2026-11-12-linear-and-zoom-integrations-design.md).

## Status

**Accepted** (2026-11-12). D1 to D3 are built by the Linear plan; D4 to D6 are built by the Zoom plan that follows it.

## Context

The owner asked for two more integrations that admins connect once per workspace. Linear: many teams track work in Linear, not Jira, and the canon has listed Linear as Jira's alternative since the PDF (R8), but it was never built: the Project tracker card showed "Linear: Coming soon". Zoom: Zoom meetings reach Mytheus only through the Chrome extension during a live call; a team that records in Zoom has no way to bring those meetings in afterwards.

The owner decided: each new integration does everything its counterpart does (Linear matches Jira), and every recorded Zoom meeting hosted by a workspace member comes in automatically. Outlook Calendar, Microsoft Teams chat and server-side mail reading were discussed and deferred.

## Decision

1. **D1 Linear is built as R8's alternative PM tool.** No relaxation of R8 is needed for it: a workspace still connects at most one of Jira and Linear (`PM_PROVIDERS`, a 409 otherwise). Linear connects with OAuth 2.0 (`actor=app`, scopes `read`, `write`, `issues:create`); `external_account_id` is the Linear organization id; post-install records the organization's first team, as Jira records its first project.
2. **D2 Linear parity with Jira.** A confirmed action becomes a Linear issue (title, a back-link to the meeting, due date, assignee by email when Linear knows it). A Linear webhook on the OAuth application (resource Issue) carries status back. Everything that read Jira (promise drift, the assumption watch, decisions as bets, blast-radius links, the follow-up workflow) reads whichever tracker the tenant connected. Wire names that said "jira" (`jira_keys`, `/internal/jira/issue-status`, the bets source `jira`) keep their names and mean "the tenant's tracker"; a `provider` field says which.
3. **D3 Linear trust and status.** Inbound deliveries are verified (`Linear-Signature`, HMAC-SHA256 of the raw body with `LINEAR_WEBHOOK_SECRET`; `webhookTimestamp` within 60 seconds), routed by `organizationId`, and deduplicated by `Linear-Delivery`. An issue changes a Mytheus action only when it is in this tenant's `integration_id_map`, and its state is re-read from Linear with the tenant's token rather than taken from the webhook body. State type `completed` maps to `done`; every other type, `canceled` included, maps to `confirmed` (Jira's two-state map).
4. **D4 Zoom is a meeting-source integration, outside R8.**

   > [SaaS-scale adjustment] The PDF's R8 caps integrations at Calendar + Slack + one PM tool. Zoom is added beside the Phase 4 reach providers already outside that cap (ADR-0037 D3), because it is a source of meetings, not another tool to sync work into, and because a production SaaS cannot ask every Zoom team to run a browser extension during each call. The provider enum gains `zoom` when it is built, canon first.

5. **D5 Zoom trust, consent and media.** A Zoom meeting is imported only when its host's email belongs to a live user of the tenant. Only recordings completed after the connection are imported (no backfill). Zoom's own recording notice, shown to every participant when the host records, stands in for Mytheus's capture consent prompt; each import is audited (`zoom.meeting_imported`). Recording files are fetched with the webhook's short-lived download token, used once and never stored; only transcript text is kept.
6. **D6 Zoom transcripts go through `transcription-service`.** Zoom's own transcript is used when present, otherwise the audio is transcribed with the same key terms as a live stream. `transcription-service` stays the only writer of `transcript_chunks` and gains a batch import entry point, so search, RAG and the agents see an imported meeting exactly like a live one. An import attaches to an existing meeting of the same host and start (from Google Calendar) instead of creating a duplicate.

## Consequences

- R8 now reads "Calendar + Slack + one of Jira/Linear, plus the reach and meeting-source integrations recorded in ADR-0037 and ADR-0044".
- Three new server secrets for Linear (`LINEAR_CLIENT_ID`, `LINEAR_CLIENT_SECRET`, `LINEAR_WEBHOOK_SECRET`) and, later, three for Zoom; each provider is "not configured" until all of its values are set.
- Linear needs no webhook renewal job: the webhook belongs to the OAuth application, not to each tenant.
- Zoom adds a subprocessor (Zoom) only for workspaces that connect it; the subprocessor list changes when it is built.
- Zoom cannot be tried live by the owner (no paid Zoom account); it ships tested against recorded Zoom payloads, and the work-done note says so.
