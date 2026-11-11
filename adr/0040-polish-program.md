# ADR-0040: The polish program

On 2026-11-11 the owner asked for every item of the Mytheus Polish Plan to be built, merged and deployed in one push, with no questions asked. The plan came from four reviews of the live code. This ADR records the decisions taken while building it, and why. It builds on [ADR-0038](0038-agent-personas-profiles-agendas-and-meeting-agent.md) (personas, agendas, `MeetingAgent`) and [ADR-0039](0039-company-brain-and-meeting-companions.md) (the company brain and meeting companions).

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md): every new table, column and Redis key below.
- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md): the routes, consumer groups and producers.
- [_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md): the `agent-feedback` topic, the `admin-events` retention and `erasure_completed`, the `agent-results` and `agent-stream` additions.
- [ADR-0018](0018-consent-entity-behind-flag.md) (consent), [ADR-0029](0029-plan-7-budget-scope-tier-mapping-and-metering-ownership.md) (budgets), [ADR-0030](0030-plan-7-llm-response-cache-and-personal-listener-gating.md) (bands), [ADR-0033](0033-plan-9-rag-retrieval-upgrades.md) (participant-scoped retrieval), [ADR-0037](0037-plan-12-integrations-follow-ups-and-consoles-build.md) D3 (the "not connected" pattern).
- The build record: [work-done-till-now/11-polish-program.md](../work-done-till-now/11-polish-program.md).

---

## Status

**Accepted** (2026-11-11).

## Context

Four reviews read the code that was live on 2026-11-11 (`mytheus-ui`, `mytheus-intelligence`, `mytheus-backend`, `mytheus-realtime` and these docs at `main`):

- **The meeting room.** Shared cards did not sync: one person's confirm left everyone else with a stale card whose click then failed, and one person's dismiss removed the card for the whole room. The realtime token was fetched once and expired in about a minute, so any reconnect was refused while the room still showed "Listening". The host's agenda changes were local only, Notes mode hid every confirm card, and the clock showed the viewer's time in the room, not the meeting's.
- **The seven agents.** Cards arrived 20 to 45 seconds after the words, because agents ran on fixed 30-second buckets, one window at a time per partition, against a design target under 4 seconds. Notes called people "Participant 2", renumbered every window. `DecisionAgent`, `ContextGapAgent` and three `ProactiveAgent` prompts lacked the never-record rule, and the embedding and full-text indexers stored never-record windows anyway. Sonnet ran at its default high thinking effort with 400-token limits, which risked silently dropped notes and doubled spend. Live and reload disagreed: the gateway used a fixed 0.7 threshold, `core-api` the tenant's override. No agent saw an earlier window, so dismissed cards came back, and the evals scored 1.0 on clean, hand-written windows, so they could not tell a better prompt from a worse one.
- **The UI and UX.** Seven page widths from 960 to 1600 px, about 3,000 lines of separate meeting CSS, titles shown twice, four local component kits, four success patterns, and six lists competing for "what do I owe".
- **The market.** A scan of Granola, Otter, Fireflies, Fathom, Read.ai, tl;dv, Zoom AI Companion, Google Meet, Teams Copilot, Notion, Supernormal and Krisp, from their own pages and changelogs. Mytheus is ahead on an agent per person, decisions as confirmed graph objects, sending your agent, and the loop between meetings. The market is ahead on capturing Zoom, Meet and Teams calls, recaps for people who are not users, recording playback, asking the AI live, mobile and in-person capture, an MCP server or public API, CRM sync and live translation.

The plan therefore fixed trust first (Phase 0), then made the meeting calm and fast (Phase 1), made the app one product (Phase 2), taught the agents to learn and wrap up (Phase 3), and widened the lead (Phase 4: reach, and features only Mytheus can do). It was built in three waves over one day; [the work record](../work-done-till-now/11-polish-program.md) has the waves and the PRs.

## Decision

Every new table carries `org_id` with ENABLE and FORCE row-level security, the canonical `tenant_isolation` policy and an explicit `grant_tenant_table` ([ADR-0021](0021-explicit-grants-no-schema-wide-default-privileges.md)). Every model call goes through the router, so it is budget-gated and metered. Anything that needs an external credential is built fully and ships "not connected" until the credential exists (ADR-0037 D3).

### Part A: the live meeting

**A1. Event-driven windows, and notes held to 30 s.** A window closes when something is said, not when a bucket ends: on a cue phrase ("I'll", "let's", "we decided", "by Friday", ...), on a turn end (a new speaker, or 1.5 s of quiet once it holds 10 s of speech), at 15 s of span, or after 15 s of wall-clock time, never closer than 4 s of meeting time to the last close (`WINDOW_*` env). Each meeting is one consumer task; a window's id is `{meeting_id}:{start_ms}`. Decision and action candidates run on every window. `PersonalListenerAgent` holds windows until they span 30 s of meeting time and `QuestionAgent` 20 s, unless a cue closes them, so faster windows do not mean more notes or more spend per person. Held windows are written when a listener leaves and when the meeting ends. *Why:* the fixed buckets were the main source of lateness, and a card per 10 s window per person would have doubled note volume and cost. Capacity A17 keeps 30 s as the per-participant sizing figure.

**A2. Memory inside the meeting.** The agents see the last two windows and a ledger of open candidates, confirmations, edits and dismissals (`nexus:{org_id}:meeting-ledger:{meeting_id}`: `c:` fields written by the agent runtime, `s:` and `q:` fields by `core-api`, one writer per field). A candidate may update an open card (`updates_candidate_id`) instead of proposing another; `core-api` applies an update only to a `proposed` row with no `edited_at`. *Why:* duplicates and returning dismissed cards were the second cause of noise, and a person's edit must never be overwritten by a later agent update.

**A3. One card per moment.** Every user-visible item carries `moment_id = uuid5(MOMENT_NS, "{meeting_id}:{chunk_id}")`, where `chunk_id` is the transcript final the item quotes, plus the quote, speaker, meeting time and a "why you're seeing this" line. A note's promise, a decision and an action about the same words share one id without the agents talking to each other, and a replay gives the same id. The UI merges them into one card; confirming an action closes the matching promise (`commitments.action_item_id`). *Why:* hashing the window id would differ between accumulating and per-window agents; the final is unique, stable and the same for every agent.

**A4. Shared candidate ledger.** A confirm, edit or dismiss stays a `core-api` REST write. `realtime-gateway` then re-reads the stored row with the actor's own token and broadcasts it (`candidate-resolution`) only if it matches: a confirm by this person, a dismiss for everyone only when this person hosts or created the meeting (otherwise dismiss is personal). Editing takes a 90-second lock (`nexus:{org_id}:candidate-locks:{meeting_id}`), renewed while editing and released on disconnect. Host agenda changes, "End for everyone" and the wrap-up are broadcast the same way. *Why:* the gateway never trusts a client's claim, and `core-api` stays the only writer.

**A5. One threshold source.** `GET /internal/agent-policy` serves the tenant's thresholds and calibration to the agent runtime, and `core-api` bands its reloads with the same function, so a card shown live is the card shown on reload.

**A6. Room reconnects and the meeting clock.** The socket fetches a fresh token on every connect and re-authenticates (`reauth`); LiveKit gets a new token after a drop; the room shows one honest status ("Agent paused, reconnecting"). `meeting_rooms.started_at` is the meeting's clock, set once by the first token mint or LiveKit `room_started`.

**A7. Consent at token mint.** The pre-join sends `consent: {version}` with its token request; the mint stamps `consented_at` and `consent_version` the first time, before the 428 gate, whether or not the tenant requires consent. `GET /api/meetings/{id}/session` says whether to ask; `GET /api/meetings/{id}/consent-receipt` shows the person what they agreed to, clause by clause. *Why:* the checkbox was never sent, so a tenant with `consent_required = true` looped people back to the pre-join forever, and the copy promised more than was stored.

**A8. The calm room.** A Now slot with a queue and "Needs you" pinned on top, notes folded under the current agenda item, suggestions held while you speak, a stage bar for everyone (item, elapsed versus planned, next item; `extended_minutes`, "Start next"), a private "Ask your agent" box, stand-ins (a proxied person's update on their tile, their questions raised when their item starts, "Ask Sarah's agent" over the permission-gated context fill), a 60-second wrap-up for the host that assigns every card still `proposed` a `review_owner_id`, and a catch-up summary for late joiners.

### Part B: agents you can trust

**B1. Stable labels.** Speakers are "Participant N" numbered once per meeting, and display names are substituted after the model answers, line by line for streamed notes. The model still never sees a name, id or email.

**B2. Effort and cache breakpoints.** Every extract-tier call names its effort: `low` for extraction (notes, decisions, actions), `medium` for synthesis (questions, context fills, conflict and already-decided checks); triage keeps thinking off. `max_tokens` is 8192, a ceiling only, and a response cut at the ceiling is logged, never parsed as "nothing to say". The system prompt is cache breakpoint 1; the org-stable head of the user message (the company block, the never-record line, topic owners) is breakpoint 2, so it is read at 0.1x on every window of every participant. *Why:* default high effort spent thinking tokens against a 400-token limit.

**B3. Dollar-weighted budget units.** One budget unit is the list price of one uncached Sonnet input token. Usage is weighed by the tier's price, cache writes at 1.25x, reads at 0.1x, thinking as output. The names (`usage-events` `agent_token` `quantity`, `agent_budgets.tokens_consumed`) and their integer semantics are unchanged. *Why:* raw tokens made a Haiku gate token cost the tenant as much as a Sonnet output token, a hundredfold difference.

**B4. Never record everywhere, and the confidential index flag.** Every agent prompt carries the ground rules, and a post-processing filter drops any output text that names a never-record topic or its alias, live or durable. `embedding-service` and `search-service` check each chunk before storing it: an outright mention marks that 30-second window and the next (`nexus:{org_id}:never_record:{meeting_id}:{window}`) and purges what was already stored; an ambiguous one gets one cached triage check (`never_record_check@v1`); if the check or the company brain cannot be read, the chunk is not indexed. A confidential meeting is indexed with `confidential = true`: participant-scoped retrieval still finds it for its own participants, org-wide search excludes it. A red-team eval must stay at zero leaks. *Why:* a prompt rule is an instruction; the filter and the index guard make the cases a machine can see a guarantee. Skipping confidential meetings would have taken them away from the people who were in them.

**B5. Calibration bounds.** Each agent's surface threshold is calibrated per workspace and, for the agents that run for one person (`PersonalListenerAgent`, `QuestionAgent`, `ContextGapAgent`), per person: from the latest action per (person, item) over 90 days with a 14-day half-life, the decayed dismiss share moves the base by 0.4 x (share minus 0.35), shrunk for small samples, at most 0.10 down and 0.15 up, bounded to 0.6 to 0.85 and quantised to 0.025, after at least 20 workspace or 8 personal samples. Decision and action candidates are one card for the whole room, so they use the workspace value only. *Why:* bounded, slow-moving thresholds keep live and reload in step under a 60-second policy cache and stop one person's habits from silencing a shared card.

**B6. Person memory, with learning on or off.** `agent_feedback` records every confirm, dismiss, edit, ask and save, with an `agent-feedback` event in the same transaction. `person_memory` keeps weighted topics (30-day half-life), recurring questions and muted topics, used by notes, suggestions and briefs. The person sees it under Settings, "What your agent remembers", can mute or pin topics, turn learning off (nothing new learned or served) or forget it (feedback text nulled, counts kept for calibration). Open actions and promises are read live, never copied. *Why:* agents never heard whether they helped, and memory a person cannot see or stop would not be trusted.

**B7. Translation on Haiku.** A viewer picks a caption language; the agent runtime translates each final into the languages the meeting's viewers chose, on the triage tier (`caption_translate@v1`), budget-gated and metered, and sends it on `agent-stream` as `caption_translation` to those viewers only. A budget refusal or failure leaves the original caption, with at most one notice per language a minute. *Why:* a caption is short and frequent; Sonnet would cost about twenty times more for no visible gain.

**B8. Evals that can fail, with live recordings.** A corpus of 144 windows with recognition noise and crosstalk, persona pairs that must capture the same facts, golden sets for `MeetingAgent` and the proactive checks, a note-quality judge (`note_quality_judge@v1`), a never-record red-team set, and deliberately worse prompts (`@v0-worse`) that must fail every gate. Live answers are recorded (`EVAL_LIVE=1 EVAL_RECORD=1`) and CI replays them offline, so the gate is free to run and still measures real model output. *Why:* evals that score 1.0 on clean windows cannot catch a regression.

**B9. Telemetry.** Per agent and workspace: time to card, gate pass rate, parsed results and parse failures, tokens, spend, meeting seconds, never-record skips, off-the-record skips and caption translations, on one Grafana dashboard ("Mytheus - Agent quality") with alerts on slow cards, parse failures, spend per meeting-hour and never-record leaks. Agent series carry `org_id` by design; HTTP series stay tenant-free.

### Part C: minutes, recaps and reach

**C1. Minutes per participant plus shared.** After `meeting-events` `ended`, one extract-tier call per participant (their decisions, open items, unanswered questions and promises) and one shared call, with the meeting record as a cached prefix shared by all of them. Records go out on `agent-results` as `minutes`; `core-api` keeps one private row per person and one shared row. With no transcript, or a refused or failed shared call, a shared `unavailable` row is written so nothing waits forever. *Why:* each person's minutes are about what they own; the shared set is what can leave the company.

**C2. Recap via share links.** A recap is the shared minutes behind a signed, expiring, revocable link (`share_links`, HMAC over org and link id), served at `/s/{token}` with no sign-in and a view count. The host's follow-up email sends the link to guests who are not users, and waits for the shared minutes before it goes. Decision records share the same way.

**C3. Reach in a not-connected state.** Personal access tokens (`mytheus_pat_<org hex>_<secret>`; only the SHA-256 stored; `read` or `read_write`; expiry mandatory) for the public API and an MCP server at `POST /mcp` whose tools call the public API as the caller. Decision records export to Notion, Confluence or Slack with lineage links, and meeting notes to HubSpot. HubSpot, Notion and Confluence ship "not connected" until their OAuth apps exist; email until a provider key is set; push until `PUSH_PROVIDER=expo`. *Why:* the token names its tenant, so it resolves under RLS with no exception; an MCP server inside `bff` needed no new deployable.

**C4. Capture via the Chrome extension.** A Manifest V3 extension captures a Meet, Teams or Zoom tab's audio and streams it to `transcription-service` `/capture/audio` under a 15-minute HS256 capture token the app's server mints from the person's session; the person confirms everyone knows, and that consent is written to `audit_logs` before audio is accepted. The meeting has `source` `meet`, `teams` or `zoom`, so no Mytheus room is opened and the same agents and graph run on it. The phone records in-person meetings the same way (`in_person`). *Why:* a botless companion needs no calendar bot account, and without `source` LiveKit's empty-room timeout ended captured meetings mid-call.

**C5. Recording playback and clips.** `meeting-service`, which owns the `mytheus-recordings` bucket, keeps one `meeting_recordings` row per LiveKit egress segment, holding an object key, never a URL. `GET /api/meetings/{id}/recording` mints 600-second presigned URLs for participants only, placed on the meeting's clock so the transcript plays in sync, and every issue is audited. A clip (`meeting_clips`, at most 10 minutes, `core-api`) is a span of the meeting clock and may not cross an off-the-record gap (409). A shared clip (`share_links` kind `meeting_clip`, video only when the sharer chose `include_video`) gets its own object cut with ffmpeg (`-c copy`) into `{org_id}/{meeting_id}/clips/{clip_id}.mp4`. `meeting_rooms.recording_skip_reason` says why a room was not recorded (`consent_missing`). Recording stays not connected until `RECORDING_ENABLED` and the storage credentials are set. *Why:* the full recording is never shared, and a presigned URL that expires in ten minutes cannot outlive a revoked share.

### Part D: only Mytheus

**D1. Off the record, enforced server-side in four places.** The host (or creator) starts and ends a segment; any participant can ask. `core-api` records it (`meeting_off_record_segments`) and sets `nexus:{org_id}:off-record:{meeting_id}`; `transcription-service` persists no final, `agent-runtime-service` runs no agent, and `embedding-service` and `search-service` index nothing while it is set. The gap shows in place in the transcript, the minutes and the replay, and the recording's egress stops for the segment and starts again after it. *Why:* a pause button is only a promise if every writer obeys it, and Mytheus owns every writer because it owns the room.

**D2. Delegated answers only from statements.** A person who sends their agent may pre-approve up to 10 statements of up to 300 characters for that meeting. Asked in the room, their agent answers only from those (`delegated_answer@v1`), otherwise exactly "{First name} didn't pre-approve an answer to that.", and every answer or refusal is logged for that person to read. *Why:* an agent speaking for someone must never improvise a commitment.

**D3. The rest of the list.** Decision reversal: a superseding decision asks the owner of each linked open action to keep, re-scope or close it. Should this be a meeting: a read-only precheck shows already-decided matches and status updates that would cover the agenda, with one-click status requests. Promise drift: a daily job compares "done by Friday" against a still-open Jira issue, nudging the owner privately first and the host two days later. A weekly "what changed for you" digest from the graph, filtered by listen-for topics.

### Part E: contracts and data

**E1. Additive API evolution.** The oasdiff gate treats a published enum as frozen. So `in_person` is stored in `meetings.source` but published as `source = mytheus` plus `recorded_in_person = true`; an erasure keeps `status` as the constant `completed` and reports across stores in a new `progress` field (`in_progress`/`completed`) with `stores[]`; the agenda PATCH keeps `status` a plain optional enum while adding `extend_minutes`. *Why:* deployed clients and the mobile app parse those enums, and an additive field breaks nobody.

**E2. Erasure across stores, with completion events.** An admin erasure of a person or a meeting runs through `admin-ops-service`: `core-api` erases its share in one transaction (including feedback, memory, quotes and close-out fields), tokens are revoked, and `erasure_requested` goes out on `admin-events`. `transcription-service`, `embedding-service`, `search-service` and `knowledge-graph-service` each finish their store and emit `erasure_completed` (key `erasure_completed:{erasure_id}:{store}`); `admin-ops-service` records each completion, and the console shows a per-store checklist. A 30-day tombstone keeps a late chunk of an erased target out of the indexes. A meeting erasure also deletes its clips and, through `meeting-service`, its recording objects.

**E3. `admin-events` retained, not compacted.** The topic moves from compaction to 365-day delete retention. *Why:* compaction keeps one record per key, so a store that was down could find the request it still had to apply already compacted away; a year covers any outage and matches the audit horizon.

**E4. Mobile on real contracts.** `mytheus-mobile` was rebuilt on real data: Clerk, a typed REST client generated from the backend OpenAPI specs, the gateway socket and LiveKit with consent, Expo push registration (`push_devices`), in-person recording, and Undo through the reopen route. No mobile-only endpoint was added.

### Part F: one coherent app

`mytheus-ui` moved to one information architecture (Home, Inbox, Meetings, My work, Decisions, People; Workspace for admins; Settings personal only), one Page frame with two widths, one component kit, Sonner toasts that use the button's verb with Undo, a command palette over meetings, people, decisions and actions, one first-run checklist, an accessibility pass with room a11y tests, and the meeting pages on the shared tokens. Old routes redirect.

## Consequences

- One new topic (`agent-feedback`, 22 in all) and BACKWARD-compatible appends on `agent-results`, `agent-stream` and `admin-events`; every consumer tolerates the new symbols and ignores what it does not handle.
- Eighteen new tables and nullable or defaulted columns on nine existing ones; one linear migration chain in `mytheus-backend` (`a7b8c9d0e1f2` to `d6e7f8a9b0c1`) and one in `mytheus-intelligence` (`e7c1a9d2b4f5`).
- More model calls per meeting-hour for candidates (shorter windows), offset by low effort, the second cache breakpoint and the held note and suggestion spans; the live numbers come from the new telemetry, not from the capacity model.
- Agents still never write Postgres; `core-api` persists cards, minutes, feedback, memory and delegated answers.
- Everything that needs a credential works end to end in tests and lights up when the credential is set.

## Follow-ups still open

- Turn recording on in production (`RECORDING_ENABLED`, the recording storage credentials) so playback has something to play.
- Recompute Capacity A17 and the spend per meeting-hour from production telemetry once the observability profile runs in production.
- Paid evals for `delegated_answer@v1` and `caption_translate@v1`.
- The never-record index marks still use 30-second buckets (`start_ms // 30000`) while agent windows are event-driven; aligning them is open.
- The `NeverRecordLeakEval` alert needs a Prometheus pushgateway for the eval job.
- Owner steps that the build could not run (production secrets, the observability profile, the `admin-events` retention change on the running cluster, the mobile native build): see the work record.
</content>
</invoke>
