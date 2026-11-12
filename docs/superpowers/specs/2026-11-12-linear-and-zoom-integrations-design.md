# Linear and Zoom integrations: design

Date: 2026-11-12. Status: draft for the owner's approval. Repos touched: `mytheus-docs`, `mytheus-backend`, `mytheus-realtime`, `mytheus-ui`, `mytheus-platform`.

## 1. Intent

Workspace admins should be able to connect two more tools on Settings > Integrations:

- **Linear**, as the second project tracker. The Project tracker card already shows "Linear: Coming soon"; this makes it real. A workspace that runs on Linear instead of Jira gets everything a Jira workspace gets.
- **Zoom**, as a meeting source. Every Zoom meeting hosted by a workspace member comes into Mytheus automatically once its cloud recording is ready, and gets the same decisions, action items, minutes and Myte follow-through as any other meeting. No browser extension and no live bot are needed.

Why: these are the two most common gaps for a team evaluating Mytheus. Linear is the default tracker at many startups, and Zoom is the most used meeting tool that Mytheus can only reach today through the Chrome extension during a live call.

The owner decided (2026-11-12): admins connect both, once per workspace; each does everything its counterpart does (Linear matches Jira); Zoom imports every recorded meeting hosted by a workspace member, automatically. Outlook Calendar, Microsoft Teams and server-side mail reading were discussed and deferred.

## 2. Goals

1. An admin can connect and disconnect Linear and Zoom from Settings > Integrations, with the same card states as the existing providers (Ready to connect, Needs setup with a Set up guide, Connected, Error) and the same OAuth flow.
2. Linear has full parity with Jira (section 5).
3. A Zoom meeting hosted by a workspace member becomes a normal Mytheus meeting with a transcript, then runs the existing post-meeting processing unchanged (section 6).
4. Both are covered by automated tests at the same depth as the Jira and Google Calendar connectors.
5. The architecture docs say what was built, canon first (section 4).

## 3. Non-goals and anti-goals

Non-goals (this round):

- Outlook Calendar, Microsoft Teams chat, Gmail or Outlook mail reading.
- Zoom live participation (a bot in the call, Zoom RTMS real-time streams). Live Zoom calls keep working through the extension as today.
- Keeping Zoom recordings. Mytheus fetches, transcribes and discards the media.
- Connecting Jira and Linear at the same time. A workspace still has at most one tracker.
- Zoom chat, Zoom Phone, Zoom Webinars.

Anti-goals (would be a failure even if it technically works):

- A Zoom meeting whose host is not a member of the workspace ends up in Mytheus. Anyone can be in a Zoom account the admin connected; only the workspace's own people bring meetings in.
- The same meeting appears twice: once from Google Calendar and once from Zoom.
- A replayed or forged Zoom or Linear webhook changes anything.
- A Linear issue that Mytheus did not create (or is not in this tenant's id map) changes a Mytheus action, whatever its contents claim.
- A provider token, a recording download token or a recording file is logged, stored in plain text, or kept after use.

## 4. Docs first: the canon and the ADR

Per `mytheus-docs/CLAUDE.md`, the docs change before the code:

1. **ADR-0044** (new): "Linear and Zoom integrations". Records: Linear is built as the alternative PM tool R8 already allowed; Zoom is added as a meeting-source integration, which is a deliberate relaxation of guardrail R8 (Calendar + Slack + one PM tool), alongside the Phase 4 reach providers already outside it (ADR-0037 D3); why imported transcripts go through `transcription-service`; the host-is-a-member trust rule; media is never kept. Marked with the `[SaaS-scale adjustment]` callout where it departs from the PDF.
2. `_canon/02-data-model.md`: `integration_connections.provider` gains `zoom`. `meetings.source` already allows `zoom`. Add `meetings.external_meeting_ref` (or the builder's equivalent, see 6.4) if needed for idempotent Zoom import.
3. `_canon/01-service-catalog.md`: `transcription-service` gains a batch import entry point (6.5); `integration-service` lists Linear and Zoom.
4. `_canon/06-tech-decisions.md` section 2.13: the R8 row is amended to point at ADR-0044.
5. Reader doc `21-integration-architecture.md` (Linear and Zoom sections, the R8 section), `24-glossary-and-naming.md`, `coverage-traceability-matrix.md`, and any diagram that lists the providers.
6. A work-done note in `work-done-till-now/` once built.

## 5. Linear: parity with Jira

Every place Jira is used today gets a Linear path. The code already has `LINEAR` in `integration_service/config.py`, `PM_PROVIDERS`, a `linear` column in `watch_source_status`, `linear` in the `decision_why` source literal, and `/oauth/linear` routed in `api_gateway/edge.py`; most of the work is filling these in.

| Jira today | Linear |
|---|---|
| OAuth 2.0 3LO connect; pick the site and its first project (`jira_site` cursor) | Linear OAuth 2.0 connect, actor `app`; pick the organization (external account id = Linear organization id) and its first team (a `linear_team` cursor). Scopes: `read`, `write`, `issues:create`. |
| Dynamic webhook registered per tenant, renewed every 30 days | Webhook configured once on the Linear OAuth application (resource type Issue). Linear delivers events for every installing organization, so there is no per-tenant registration and no renewal job. |
| Webhook auth: Atlassian JWT naming the cloud | `Linear-Signature` = HMAC-SHA256 of the raw body with `LINEAR_WEBHOOK_SECRET`, plus `webhookTimestamp` within 60 seconds. Tenant = the connection whose external account id is the payload's `organizationId`. Delivery id = the `Linear-Delivery` header. |
| `actions` consumer: confirmed action -> create issue (id map, then label search for the crash window, then create) | Same three steps. Create with `issueCreate`: title = content, markdown description with the back-link to `{APP_BASE_URL}/meetings/{meeting_id}`, `dueDate`, assignee by email (Linear `users` lookup) when Linear knows the email. The crash-window lookup uses an issue attachment whose URL is the action's Mytheus link (Linear can find issues by attachment URL); the builder may use a label instead if that proves simpler. |
| Inbound status: re-read the issue with the tenant's token; `done` category -> done, else confirmed; loop suppression by `last_synced_status` | Same. Re-read the issue; workflow state type `completed` -> done, every other type -> confirmed (including `canceled`, matching Jira's two-state map). Same watermark loop suppression. Only issues in this tenant's id map are acted on. |
| `jira_status.py` and `watch_sources._jira` (assumption watch, decisions as bets) | A Linear equivalent: issue state, due date, assignee for the issue keys a decision watches; `watch_source_status.linear` reports `connected` / `not_connected` / `error` instead of always `not_connected`. |
| `decision_why` blast radius links (`source = jira`) | `source = linear` links with Linear issue URLs. |
| core-api, workflow-service and bff call sites that name Jira (action follow-up, promise drift, only-Mytheus, bets, assumption watch, the MCP tools) | Each becomes tracker-neutral: it asks "the tenant's tracker" and works for whichever of Jira or Linear is connected. The builder lists every call site (`grep -i jira` over `services/`) and covers each with a test. |
| UI: Project tracker card, Jira segment, Set up guide | The Linear segment is enabled; choosing it shows Linear's Connect and Set up guide (console link, redirect URL, scopes, env var names). "Tracker in use" blocks the other tracker as today. |

Secrets: `LINEAR_CLIENT_ID`, `LINEAR_CLIENT_SECRET`, `LINEAR_WEBHOOK_SECRET` (all three, or Linear stays "Needs setup", like Slack's signing secret).

## 6. Zoom: recorded meetings come in automatically

### 6.1 Connect

- A Zoom "admin-managed" OAuth app, installed by a Zoom account admin, so Mytheus can see every user's cloud recordings in that Zoom account. External account id = Zoom `account_id`.
- Scopes (Zoom granular scope names to be confirmed by the builder against Zoom's current docs): read cloud recordings and their files account-wide, read users, read past meeting participants.
- One card, "Zoom", under "For your Mytes". Secrets: `ZOOM_CLIENT_ID`, `ZOOM_CLIENT_SECRET`, `ZOOM_WEBHOOK_SECRET_TOKEN`.

### 6.2 Webhooks

- Event subscriptions on the Zoom app: `recording.completed` and `recording.transcript_completed`, delivered to `{OAUTH_REDIRECT_BASE}/webhooks/zoom`.
- Verification: `x-zm-signature` = `v0=` HMAC-SHA256(`ZOOM_WEBHOOK_SECRET_TOKEN`, `v0:{x-zm-request-timestamp}:{raw body}`), timestamp within 5 minutes. The `endpoint.url_validation` event is answered with the `plainToken` and its HMAC, as Zoom requires.
- Tenant = the connection whose external account id is the payload's `account_id`. Delivery id = the event's id, or `{event}:{meeting uuid}` if absent.
- As with every provider: verify, write one inbox row, answer 200 at once; the worker does the work.

### 6.3 The trust rule

A Zoom meeting is imported only when its **host's email belongs to a live user of this workspace** (the same RLS-bound, case-insensitive match the calendar connector uses for organizers). Otherwise the inbox row is recorded as skipped with reason `host_not_member`. This is the owner's "every meeting hosted by a workspace member".

### 6.4 One meeting, not two

If the Zoom meeting was also on a connected Google Calendar, Mytheus already created a meeting for it. The import attaches to that meeting instead of creating another:

- **Owner-decided intent:** never a duplicate.
- **My call on the mechanism (builder may refine):** match an existing meeting of this tenant with the same host, a scheduled start within 15 minutes of the recording start, and no transcript yet. If the calendar connector can be extended to keep the Zoom meeting id from the event's conference data or location, match on that first and fall back to the time window. Otherwise create a meeting with `source = zoom`, title = Zoom topic, scheduled start and end from the recording, host = the matched user, participants = workspace members found among Zoom's past-meeting participants (by email).
- Import is idempotent on the Zoom meeting UUID: a redelivered or second event (recording then transcript) never creates a second meeting or a second transcript.

### 6.5 The transcript

- **Preferred:** Zoom's own transcript (a VTT file, present when the Zoom account has audio transcripts on). Its cues carry speaker names and times.
- **Fallback:** when there is no transcript file, the audio-only recording file is transcribed by Mytheus.
- **Where it is written:** `transcription-service` owns `transcript_chunks` and emits `transcript-chunks`. It gains one internal batch import entry point, authenticated with an internal caller token, that takes either parsed cues or an audio file reference, writes final chunks (no interims) with speaker labels, and emits them on the bus exactly as a live stream does. Audio goes to Deepgram's pre-recorded API with the same key terms the live stream gets (ADR-0039 D1). This keeps one owner of transcripts and lets search, RAG and the agents work unchanged.
- **Speakers:** a Zoom speaker name is matched to a workspace participant by display name, then by email when Zoom gives one; anyone unmatched stays a named external speaker.
- **Then:** the meeting is marked ended with its real start and end, which starts the existing post-meeting processing (the same `ended` path a captured call uses).
- **Media:** recording files are streamed with the short-lived download token from the webhook, used once, and never stored. Only transcript text is kept.
- **Audit:** an audit row `zoom.meeting_imported` with the host as actor and the Zoom meeting UUID; the off-record, confidential and erasure rules apply to the imported meeting as to any other.

## 7. UI

- Settings > Integrations, "For your Mytes": add a **Zoom** card; enable the **Linear** segment on the Project tracker card. Same card component, states and Set up dialog as today (`IntegrationsPanel.tsx`, `IntegrationSetup.tsx`, `SETUP_GUIDES` in `src/lib/integrations.ts`).
- A Zoom-imported meeting shows its source as Zoom wherever the meeting source already shows (meeting header, lists). No new screens.
- Types regenerated from the backend OpenAPI, not hand-edited.

## 8. Platform

- `mytheus-platform`: `scripts/push-secrets.sh integrations` and `env/prod.env.example` learn the six new env vars.
- The work-done note 16 owner guide gains Linear and Zoom rows (where to create the app, redirect URL, webhook URL, scopes, env vars).

## 9. Testing and verification

- **Linear:** unit and integration tests at the depth of the Jira tests (signature, tenant routing, replay, id map, crash window, loop suppression, status map, watch sources, every tracker-neutral call site). Then a **live test** by the owner on a free Linear workspace: connect, confirm an action, see the issue, move it to Done, see the action done.
- **Zoom:** tests against recorded Zoom payloads and a sample VTT: URL validation, signature, tenant routing, replay, the host-not-member skip, matching an existing calendar meeting, creating a new one, the transcript-first and audio-fallback paths, idempotency across `recording.completed` then `recording.transcript_completed`, and that no media or token is persisted. **Not tried live:** the owner has no paid Zoom account. This is stated in the work-done note.
- The existing quality gates for each repo (lint, types, tests, `next build`, a11y for the UI cards).

## 10. Order of work

1. Docs: ADR-0044 and the canon edits (section 4).
2. Linear, end to end (backend, UI, platform), then the owner's live test.
3. Zoom: integration-service connector, then the `transcription-service` import, then the UI card and platform.
4. Work-done note.

## 11. Left to the builder

- Exact Linear GraphQL queries and Zoom endpoints and scope names, checked against the providers' current docs.
- Retry, backoff and inbox `MAX_ATTEMPTS` behaviour (reuse the worker's).
- Whether the Zoom import is driven by internal calls from the worker or a new `integration-events` kind consumed by core-api, as long as each step is idempotent and the canon names it.
- Schema details for idempotent import (a column on `meetings` or an id-map row), canon first.
- Copy on the cards and Set up guides, in the existing voice.
