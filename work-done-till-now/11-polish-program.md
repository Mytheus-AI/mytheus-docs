# The polish program

**Status:** DONE WITH GAPS (waves 1 to 3 and recording playback merged and deployed to production on 2026-11-11; gaps in sections 5, 7 and 9). Not a plan band: an owner request of 2026-11-11, decided in [ADR-0040](../adr/0040-polish-program.md). Last verification 2026-11-11.

The owner asked for every item of the Mytheus Polish Plan to be built, committed, merged and deployed, with no questions asked. The plan came from four reviews of the live code (the meeting room, the seven agents, the UI and UX, and a scan of twelve competing products); ADR-0040 summarises them and records every decision taken while building. The work ran in three waves. In each wave several builders worked in parallel worktrees on disjoint files, a coordinator joined their branches on one integration branch per repo, ran the gates, opened one PR per repo, merged and deployed.

- **Wave 1:** Phase 0 (fix what breaks trust), the UI foundation, reach and capture.
- **Wave 2:** the calm and fast meeting, agents that learn, minutes, evals and telemetry.
- **Wave 3:** the "only Mytheus" features, off the record, live translation, the meeting pages on the shared system, the reach UI, erasure across stores, mobile, and the wave 2 follow-ups; recording playback followed as its own pair of PRs (section 8).

## 1. What landed, per wave and per repo

### Wave 1

**mytheus-backend #20** (merged 2026-11-10 21:39 UTC). Migrations `a7b8c9d0e1f2` (`api_tokens`, `share_links`, `meeting_followups`, `email_deliveries`; `action_items.meeting_id` nullable; providers `hubspot`, `notion`, `confluence`; `integration_id_map` kind `decision`) and `b8c9d0e1f2a3` (`meeting_rooms.started_at`, `meetings.source`).
- `meeting-service`: consent recorded at token mint (`consent: {version}`), `GET /api/meetings/{id}/session`, no LiveKit room and 409 `EXTERNAL_MEETING` for a captured meeting, the meeting clock.
- `core-api`: candidates `?include_resolved` and a single-candidate read, chat history, `POST /api/v1/actions`, `GET /api/v1/actions/mine`, `GET /api/v1/search`, share links with the public `GET /api/v1/public/shares/{token}`, meeting follow-ups, the internal `GET /internal/agent-policy` (the one threshold source) and `GET /internal/meetings/{id}/index-policy`.
- `identity-tenant-service`: personal access tokens (`/api/v1/api-tokens`); `api-gateway` accepts `mytheus_pat_...` bearers with a per-token rate limit.
- `bff`: the MCP server (`POST /mcp`) and `POST /api/v1/ask`.
- `integration-service`: HubSpot meeting notes and decision-record exports to Notion, Confluence and Slack, all not connected until their OAuth apps exist. `notification-service`: the email channel, not connected until a provider is set.

**mytheus-realtime #11** (21:50 UTC).
- `realtime-gateway`: the shared candidate ledger (`candidate-resolved`, `candidate-lock`), host agenda and end broadcasts, raise hand, reactions, `reauth` with fresh tokens, persisted chat acks.
- `transcription-service`: the `/capture/audio` WebSocket for the browser capture extension (HS256 capture tokens, consent written to `audit_logs` before any audio), captured speakers labelled Speaker 1, 2.

**mytheus-intelligence #11** (21:44 UTC) and **#12** (21:52 UTC).
- #11: the router sets effort per call, `max_tokens` 8192, two cache breakpoints and dollar-weighted budget units; stable speaker labels with display names; tenant thresholds from `core-api`; the never-record rule in `decision_agent` v2, `context_gap_agent` v2 and the `ProactiveAgent` v2 prompts; the index guard in `embedding-service` and `search-service` (migration `e7c1a9d2b4f5`, the `confidential` flag on the FTS read model); `rag-service` accepts `bff` as a caller; `knowledge-graph-service` treats a nil meeting as a personal task; the never-record red-team eval.
- #12: `serve_metrics` binds once per process. The embedding worker's second thread had died on `EADDRINUSE` in production, so confirmed decisions had stopped reaching `decisions__{org_id}`.

**mytheus-platform #13** (21:55 UTC). Caddy routes `/capture/*` on `api.mytheus.cloud` to the `mytheus-transcription:8091` alias; the index guard settings for the embedding and search workers; env examples for capture, the index guard, reach and the not-connected providers; `scripts/push-secrets.sh polish` generates the wave 1 service tokens and the capture secret on the server.

**mytheus-ui #45** (21:56 UTC).
- Foundation: the Page frame and kit, Sonner toasts, the new IA (`/inbox`, `/work/*`, `/workspace/*`, `/people`, `/settings/*`) with legacy redirects, the command palette, the first-run checklist, drawn empty states, `ErrorState`.
- The room: reconnects with fresh tokens, the shared ledger, consent at token mint, hands, reactions, host agenda and end.
- Capture: the Chrome MV3 extension (`extension/`), `/api/capture/session/*`, the `/capture` pages; starting a capture sends `meetings.source`.

### Wave 2

**mytheus-events #5** (23:20 UTC). `agent-results` gains the optional `moment_id`, `why`, `updates_candidate_id`, `window_start_ms` and `window_end_ms` and the `minutes` kind; a new topic `agent-feedback` (`AgentFeedback`, 22 topics); `admin-events` gains `erasure_completed` and moves to 365-day delete retention. The production registry accepted 21 subjects as BACKWARD-compatible plus one first registration.

**mytheus-backend #21** (23:21 UTC). Migrations `c9d0e1f2a3b4` (card moments on `decisions`, `action_items`, `agent_suggestions`, `worklogs`; `edited_at`; `commitments.moment_id`, `action_item_id`), `d0e1f2a3b4c5` (the agenda clock, the wrap-up, `review_owner_id`), `f2a3b4c5d6e7` (`agent_feedback`, `agent_calibration`, `person_memory`) and `e1f2a3b4c5d6` (`meeting_minutes`, `meeting_followups.awaiting_minutes_since`).
- `core-api`: agenda advance and extend, the wrap-up, the catch-up summary, stand-in asks, `/api/v1/me/memory`, the internal guidance and person-memory routes, minutes and minutes-source, `POST /api/v1/erasures`.
- `admin-ops-service`: the admin data erasure at `/admin/tenant/erasures`. `notification-service`: email status.
- Erasure also removes agent memory and feedback, quoted speech, and review and close-out attribution.

**mytheus-realtime #12** (23:26 UTC). `realtime-gateway` forwards each card's moment fields and note commitments, sends multi-item `agenda-status` and the room's `wrap-up` event. `transcription-service` finishes admin erasure in `transcript_chunks` (group `transcription-service.erasure`).

**mytheus-platform #14** (23:29 UTC). The Grafana dashboard "Mytheus - Agent quality" (uid `mytheus-agents`), the alerts `AgentTimeToCardSlow` (time to card p50 over 6 s), `AgentParseFailureRateHigh` (over 2%), `AgentSpendPerMeetingHourOverBudget`, `NeverRecordLeakEval` and `NeverRecordOutputCaught`, the agent-runtime scrape target, and promtool unit tests in CI.

**mytheus-ui #46** (23:30 UTC). One card per moment, the Now slot, ask your agent, the agenda stage bar, the close-out ritual, stand-ins, catch-up, minutes with the share recap and the follow-up composer, the public `/s/[token]` page, the API access page (PATs, MCP), integration cards, decision share and export, tasks from `/actions/mine`, and the admin data erasure.

**mytheus-mobile #2** (23:34 UTC). The July app was a mock; it now runs on real data: Clerk sign-in, a typed REST client generated from the backend specs (34 calls pinned), the current gateway socket contract, LiveKit with consent at mint, Inbox, My work, Meetings, Ask with citations, decisions with lineage, the room.

### Wave 3

**mytheus-events #6** (2026-11-11 00:30 UTC). `agent-stream` gains `kind` (`delta`, `caption_translation`), `chunk_id`, `language` and `status`, all defaulted. The production registry accepted all 22 subjects as BACKWARD-compatible.

**mytheus-backend #22** (00:31 UTC). Migrations `a9b0c1d2e3f4` (`push_devices`, `in_person` source), `c5d6e7f8a9b0` (`meeting_off_record_segments`) and `b4c5d6e7f8a9` (`decision_reversals`, `delegated_answer_statements`, `delegated_answer_log`, `promise_drift_nudges`, `weekly_digests`, `digest_preferences`; `status_changed_at` with the trigger `mx_touch_status_changed_at`; `action_status_requests.requested_by`; `email_deliveries.kind` gains `weekly_digest`).
- `core-api`: off the record; decision reversal; delegated answers; promise drift; the weekly digest; in-person capture sessions; task reopen; memory calibration counts; Ask citations carry `document_id` and `start_ms`.
- `meeting-service`: the consent receipt. `bff`: the should-this-be-a-meeting precheck.
- `workflow-service`: the Temporal schedules `promise-drift-daily` and `weekly-digest`.
- `admin-ops-service`: erasure status per store (group `admin-ops-service.erasure-status`). `notification-service`: Expo push devices.
- Every API change is additive (section 4).

**mytheus-realtime #13** (00:36 UTC). Off the record in `transcription-service` and the gateway (`off-record`, `off-record-request`, `off-record-answer`), caption translation delivery (`caption-language`, `caption-translation`), and in-person capture tokens from the phone.

**mytheus-platform #15** (00:38 UTC). The `admin-erasure-status-consumer` service (workers profile); `push-secrets polish` also sets `CAPTURE_TOKEN_SECRET` and `CAPTURE_INGEST_URL` for `core-api` and allows the phone's origin; env examples for `PUSH_PROVIDER`.

**mytheus-ui #47** (00:39 UTC). Off the record and the consent receipt (`/meetings/[id]/record`), live caption translation, the meeting pages on the shared tokens and kit with real breadcrumbs, decision reversals, pre-approved answers and "what my agent said", the precheck hint, `/digest`, "What your agent remembers" (`/settings/memory`), the per-store erasure checklist, the reopen Undo, Ask citations that open the moment, in-person meetings.

**mytheus-mobile #3** (00:43 UTC). Expo push registration, in-person recording to `/capture/audio` (consent first), the reopen Undo, in-person meetings read from `recorded_in_person`.

### Intelligence for waves 2 and 3

**mytheus-intelligence #13** (01:00 UTC) joined seven builder branches and the live-eval fixes.
- Event-driven windows, in-meeting memory with the ledger, and one card per moment.
- Agents that learn: guidance from feedback, per-person calibration, person memory.
- Minutes (`minutes@v1`), delegated answers (`delegated_answer@v1`) and live caption translation (`caption_translate@v1`).
- Off the record enforced in the runtime and the indexers, and the erasure consumers for vectors, FTS and the graph.
- `BaseConsumer` releases its dedupe mark when an apply fails, for every consumer.
- Evals that can fail, with live recordings; the held-notes fix (section 6).
- Prompt defaults on main: `decision_agent` v5, `action_agent` v5, `question_agent` v4, `personal_listener` v5, `personal_listener_gate` v6, `meeting_agent` v3, `proactive_stakeholder` v3, `minutes` v1, `delegated_answer` v1; in the shared registry `never_record_check` v1 and `caption_translate` v1; `note_quality_judge@v1` is eval-only.

**mytheus-intelligence #14** (01:06 UTC) records the first live baselines for `minutes@v1` and in-meeting memory; it changes test fixtures only.

Intelligence waited for the live eval: backend, realtime, UI and mobile for waves 2 and 3 were deployed first. Until #13 deployed, the features fed by agent output (moments on cards, minutes, learning, translation, delegated answers) had no producer and showed nothing new.

### mytheus-docs

`adr/0040-polish-program.md`; canon `_canon/01-service-catalog.md` (routes, consumer groups, producers of `admin-events`), `_canon/02-data-model.md` (eighteen tables, the new columns, thirteen Redis keys), `_canon/03-event-taxonomy.md` (22 topics, `agent-feedback`, `admin-events` retention and `erasure_completed`, the `agent-results` and `agent-stream` additions, idempotency keys), a note on Capacity A17 in `_canon/05-capacity-math.md`; reader docs 06, 07, 08, 10, 12, 13 and 20, diagrams 07 and 09, the glossary, the ADR index and the traceability matrix; this file.

## 2. Definition of Done audit (against the plan)

The plan's "How we will know" table, then its items by phase.

| Target | State | Evidence |
| --- | --- | --- |
| Speech to card p50 under 6 s | NOT MEASURED | `mytheus_agent_time_to_card_seconds` and the alert exist; the observability profile is off in production (section 7) |
| Cards per person in 30 min under 15 | NOT MEASURED | same |
| Never-record leaks 0 on the red-team eval | MET | live run: `never_record.leaks` 0, `outright_skip_rate` 1.0; the worse prompts leak 10 |
| Spend per meeting-hour under $3.5 | NOT MEASURED | `mytheus_agent_spend_usd_total` and `mytheus_agent_meeting_seconds_total` exist; same blocker |
| Meetings ending with minutes 100% | PARTIAL | every `ended` meeting gets minutes or a shared `unavailable` row; the production share is not measured |
| Two page widths | MET (reported) | the Page frame (ui #45) |

| Phase | State | Note |
| --- | --- | --- |
| 0, fix what breaks trust (reconnects, shared ledger, never-record everywhere, consent, names, model settings, agenda and Notes mode, one threshold source) | MET | waves 1 and 2 |
| 1, calm and fast (event-driven windows, memory, cards tied to the moment, one card per moment, calm sidebar, stage bar, ask live, stand-ins, close-out) | MET | live latency not measured |
| 2, one coherent app (page frame, kit, meetings on the shared system, feedback, decorations, accessibility, palette and first run, the simpler sidebar) | MET (reported) | not walked through signed in for this record |
| 3, learn and wrap up (minutes, learning, evals that can fail, telemetry, memory across meetings) | MET | dashboard not running in production |
| 4, table stakes (capture, recaps, playback, MCP and API, mobile, CRM, translation) | PARTIAL | playback is built and deployed but recording is not enabled in production (section 8); capture is a browser extension, not a desktop app; CRM is HubSpot notes; mobile has no native build yet |
| 4, only Mytheus (reversal, delegated answers, off the record, precheck, promise drift, digest, shareable decision records) | MET | promise drift needs a Jira connection; exports need their OAuth apps |

## 3. Decisions taken while building

All are recorded with their reasons in [ADR-0040](../adr/0040-polish-program.md):

- event-driven windows with notes held to 30 s;
- one card per moment;
- the shared candidate ledger;
- consent at token mint;
- stable labels;
- effort and cache breakpoints;
- dollar-weighted budget units;
- never record everywhere with the confidential index flag;
- calibration bounds;
- person memory with learning on and off;
- minutes per participant plus shared;
- recaps through share links;
- reach in a not-connected state;
- capture through the Chrome extension;
- off the record enforced in four services;
- translation on Haiku;
- delegated answers only from statements;
- additive API evolution;
- erasure across stores;
- `admin-events` retained, not compacted;
- evals that can fail;
- telemetry;
- mobile on real contracts.

## 4. Deviations, and why

1. **Capture is a Chrome extension, not a desktop companion.** The plan asked for a botless desktop companion. There is no signing or update infrastructure for a native app, and the web clients of Meet, Teams and Zoom cover the common case. Calls in the native Zoom and Teams apps are not captured.
2. **`in_person` is stored but not published.** `meetings.source` allows `in_person`, but the API publishes such a meeting as `source = mytheus` plus `recorded_in_person = true`. The first version added the enum value and the oasdiff gate flagged a breaking change. A builder's edit to the CI gate was blocked as a bypass, so the change was made additive instead (backend `b08a6d4`, UI `2d2ee0e`, mobile `60a0dc5`).
3. **An erasure's `status` stays the constant `completed`.** Per-store progress is a new `progress` field with `stores[]`, for the same oasdiff reason.
4. **The agenda PATCH keeps `status` a plain optional enum** while adding `extend_minutes`. The wave 2 room-stage branch was merged without oasdiff and fixed at integration (`563e275`, `e309444`).
5. **`POST /api/v1/erasures` is not routed by `api-gateway`.** `admin-ops-service` is the single public entry point, so every erasure carries the umbrella audit row and the `erasure_requested` event. `admin-ops-service` emits the event, not `core-api`.
6. **The never-record index marks keep 30-second buckets** (`start_ms // 30000`, in the index guard and in the minutes writer), while agent windows are now event-driven. Both indexers share the marks, so they still agree with each other. The index guard's docstring still calls the bucket "the agent runtime's window"; it is a follow-up.
7. **Capacity A17 was not recomputed.** Notes keep a 30 s span, so A17 stays the per-participant sizing figure; candidates run more often, and the real cost comes from telemetry once it runs in production.
8. **The MCP server lives in `bff`, not a new service.** It is a client-shaped aggregation over the public API that calls `core-api` as the caller, so it needed no deployable or secret of its own.
9. **CRM sync is one way, to HubSpot only.** A meeting's confirmed outcome becomes a note on the matched companies and contacts. A confidential meeting is never sent. Salesforce is not built.
10. **Live translation is captions only.** One triage-tier call per final per requested language, twelve languages; there is no audio dubbing.
11. **Migration chains were re-parented at integration.** Wave 1's three migrations were built in parallel off `d2b3c4d5e6f7`, then joined into one line. Waves 2 and 3 were chained in merge order (`c9d0` to `d0e1` to `f2a3` to `e1f2`, then `a9b0` to `c5d6` to `b4c5`), so revision ids are not in alphabetical order. No merge revisions remain.
12. **Intelligence deployed last.** Wave 2's intelligence branch waited for the live eval to pass (section 6). The other repos' wave 2 and 3 features that depend on agent output were therefore inert in production until intelligence #13 deployed.
13. **A fix outside the plan.** `BaseConsumer` released no dedupe mark when an apply failed, so a redelivery was skipped as a duplicate. This was found while building the erasure consumers and fixed for every consumer (`e9c7a99`).
14. **The `admin-events` Avro doc string still says the topic is log-compacted.** The topic config and the canon say 365-day delete. The schema doc is a comment, not a contract, and is left for the next schema change.
15. **Recording playback shipped after wave 3, as its own PRs** (backend #23, UI #48), because it builds on the wave 3 off-the-record segments and share links. It ships without recording turned on in production (section 5).

## 5. What ships "not connected", and why

Each of these is built end to end, tested, and visibly off until its credential or setting exists (the ADR-0037 D3 pattern):

- **HubSpot, Notion and Confluence:** no OAuth apps exist. The integration cards say "not connected", and exports and CRM notes are refused.
- **Email** (meeting follow-ups to guests, the weekly digest by email): `EMAIL_PROVIDER`, `RESEND_API_KEY` and `EMAIL_FROM` are unset.
- **Expo push:** `PUSH_PROVIDER=expo` and an EAS project id are unset.
- **Capture** (extension and phone): `CAPTURE_TOKEN_SECRET` and `CAPTURE_INGEST_URL` arrive only with `push-secrets polish` (section 7). Until then the capture routes answer 503 `capture_not_configured` and `GET /api/v1/capture/status` says `configured: false`.
- **Promise drift:** needs a Jira connection; without one there are no issues to compare.
- **The agent quality dashboard and alerts:** the `observability` compose profile is off in production.
- **`NeverRecordLeakEval`:** needs a Prometheus pushgateway for the eval job.
- **Recording playback:** `RECORDING_ENABLED`, the R2 recording credentials and a `core-api` entry in `MYTHEUS_MEETING_CALLER_TOKENS` are unset, so nothing is recorded yet; the meeting page shows "recording off".

## 6. Verification

Rows marked *reported* come from the PR descriptions and the coordinator's integration runs; rows marked *observed* were run or read for this record.

| Repo and wave | Gates | Result |
| --- | --- | --- |
| backend #20 | ruff, black, mypy (201 files), `scripts.emit_openapi --check`, `scripts/check_authz.py`, alembic upgrade/downgrade -1/upgrade, pytest | 1196 passed (reported) |
| backend #21 | same, mypy 211 files | 1254 passed (reported) |
| backend #22 | same, mypy 230 files; oasdiff | 1323 passed; oasdiff exit 0 on `core-api`, `admin-ops-service`, `notification-service`, `bff`, `meeting-service`, `integration-service` (reported) |
| realtime #11 | gateway tsc, eslint, prettier, vitest; transcription ruff, black, pytest | transcription 170 passed; vitest green except one local failure from a stale local schema registry (reported) |
| realtime #12 | same | vitest 127, pytest 176 (reported) |
| realtime #13 | same | vitest 135, pytest 181 (reported) |
| intelligence #11 | ruff, black, mypy, non-database pytest | green; the red-team eval offline and live (reported) |
| intelligence #13 | ruff, black, mypy (155 source files); `pytest -m "not slow"` with infrastructure URLs on dead ports; `pytest -m eval tests/test_eval_harness.py` | 1021 passed, 89 skipped, 0 failed; eval replay 3 passed, 1 skipped (live) (reported) |
| events #5, #6 | validate, compatibility against the production registry | 21 BACKWARD plus 1 first registration; then 22 BACKWARD (reported) |
| platform #14 | promtool unit tests in CI | green (reported) |
| ui #45, #46, #47 | `tsc`, `next lint`, `npm test`, `test:a11y`, `next build` | 260 and 5 a11y; 310 and 26; 356 and 67; every build OK (reported) |
| mobile #2, #3 | tsc, expo lint, jest, expo export | jest 73, then 90 (reported) |
| backend #23 (playback) | pytest, oasdiff, alembic upgrade/downgrade -1/upgrade, `scripts.emit_openapi --check` | 1344 passed; oasdiff exit 0 on `core-api`, `meeting-service`, `bff` (reported) |
| ui #48 (playback) | `tsc`, `npm test`, `test:a11y`, `next build` | 363 and 103 a11y; build OK (reported) |
| events canon test, this docs branch | `uv run pytest tests -q` in a worktree of `mytheus-events` `origin/main`, with `mytheus-docs` pointing at this branch | 63 passed, including `test_topics_consistency.py` 4 of 4 (22 topics, partitions, retention with `admin-events` at 365 days) (observed) |
| events canon test, docs `main` | same test against `mytheus-docs` `origin/main` | 2 failed (`test_partitions_match_canon`, `test_retention_matches_canon`: 21 topics in the docs), which is what this branch fixes (observed) |
| backend migration chain | revision and `down_revision` read from every polish migration on `origin/main` | one line, `d2b3c4d5e6f7` to `a7b8c9d0e1f2` to `b8c9d0e1f2a3` to `c9d0e1f2a3b4` to `d0e1f2a3b4c5` to `f2a3b4c5d6e7` to `e1f2a3b4c5d6` to `a9b0c1d2e3f4` to `c5d6e7f8a9b0` to `b4c5d6e7f8a9` to `d6e7f8a9b0c1` (observed) |

**Live evals.** Run with `EVAL_LIVE=1 EVAL_RECORD=1`, about 800 model calls; reports and recordings are under `tests/eval/` in `mytheus-intelligence`.

| Metric (threshold) | First full live run (before the fixes) | After the fixes |
| --- | --- | --- |
| persona `fact_overlap` (0.8) | 0.0 | 0.958 |
| persona `shared_coverage` (0.8) | 0.0 | 0.938 |
| meeting `off_agenda_recall` (0.75) | 0.333 | 1.0 |
| proactive conflict `parse_failure_rate` (at most 0.02) | 0.05 | 0.0 |
| proactive stakeholder recall (0.95) | 0.833 | 1.0 |
| proactive stakeholder `band_accuracy` (1.0) | 0.5 | 1.0 |
| judge coverage (3.5) | 2.10 | 4.06 |
| judge attribution (3.5) | 2.73 | 4.40 |
| judge overall (3.8) | 2.06 | 4.06 |
| never record leaks (0) | 0 | 0 |

- A partial re-run of persona and the judge after the first prompt change still failed (fact overlap 0.125, judge 1.0), which led to the outcome-first `personal_listener_gate@v6` and `personal_listener@v5`.
- The final full run passed every threshold (28 m 39 s). Decision F1 was 1.0 and action F1 0.977.
- The deliberately worse prompts failed 15 checks, among them decision F1 0.40, action F1 0.41 and never-record leaks 10. This is the evidence that the gate can fail.
- Fixes (`2ef6c69`): `personal_listener_gate@v6` puts a meeting-outcome floor first; `personal_listener@v5`; `meeting_agent@v3` calibrates `off_agenda`; `proactive_stakeholder@v3`. `libs/shared/json_answer` reads the last JSON object holding the key, so a self-corrected answer parses. Live answers are now shared across variants, so recordings replay.
- #14 baselines: `minutes@v1` scored 1.0 on every metric over 3 golden meetings (10 calls); in-meeting memory with decision and action v5 scored accuracy 1.0 over 12 cases (24 calls).

**The production bug found: held notes were lost at meeting end.**
- What happened: event-driven windows let `PersonalListenerAgent` and `QuestionAgent` hold windows until they span 30 s or 20 s. A meeting's last window is usually closed by silence before the last participant leaves, so `close_meeting` had nothing left to flush, and `end_meeting` dropped whatever the agents still held. A participant who left mid-meeting lost theirs the same way, and a last leaver who had not spoken never got the final window.
- Fix (`2625971`, in intelligence #13 before it shipped): `flush_pending` runs every held stretch as final when a listener leaves and when the meeting ends. The consumer runs the open window before applying the last `left`.

**Production state.**
- Backend `alembic_version` was `b4c5d6e7f8a9` after #22 and is `d6e7f8a9b0c1` after #23; intelligence schema `e7c1a9d2b4f5`.
- After #23: `/usr/bin/ffmpeg` is present in the `meeting-service` container, `GET /api/meetings/{id}/recording` answers 401 without a token, and the UI deployed.
- The Temporal schedules were created. `admin-erasure-status-consumer` is up.
- 22 subjects are registered in the production schema registry.
- After intelligence #13 deployed, every intelligence worker was up with no restarts, and the logs show these consumers started: embedding (`embed-on-ingest`, `embed-decisions`, erasure), search (transcript index, erasure) and knowledge graph (topic seed, confirmed facts, erasure). Agent-runtime logged no errors. All reported by the coordinator, not observed by this record.

## 7. Owner steps blocked for the agent

The build session may not make remote writes on the VPS, so these wait for the owner:

1. **Secrets:** run `scripts/push-secrets.sh polish` in `mytheus-platform`, then redeploy backend, intelligence, realtime and UI, one repo at a time. This sets the wave 1 service tokens (the `bff` caller for `rag-service`, `core-api` to `notification-service`, the share-link key) and the capture secret and ingest URL for transcription, UI and `core-api`. Until then capture is not connected and some cross-service calls are refused.
2. **Observability:** enable the `observability` profile in `COMPOSE_PROFILES` and fill `prod.grafana.env`, so the agent quality dashboard and alerts run and the section 2 targets can be measured.
3. **`admin-events` retention:** set `cleanup.policy=delete` and `retention.ms=31536000000` on the production topic; `create_topics.sh` only applies to a new topic. While at it, create `agent-feedback` with 12 partitions and 7-day retention: it auto-creates on first produce with broker defaults otherwise.
4. **Mobile:** turn on Clerk's Native API for the instance, set the `EXPO_PUBLIC_*` env, run `eas init` and the native build. `PUSH_PROVIDER=expo` and the EAS project id then light up push.

## 8. Recording playback

**mytheus-backend #23** (merged 2026-11-11 01:17 UTC) and **mytheus-ui #48** (01:22 UTC), decided in ADR-0040 C5. Both are deployed. Recording itself is not connected in production (section 5).

- Migration `d6e7f8a9b0c1` after `b4c5d6e7f8a9`:
  - `meeting_recordings`, written by `meeting-service`: one row per egress segment, status `recording`, `processing`, `ready` or `failed`, holding an object key, never a URL;
  - `meeting_rooms.recording_skip_reason` (`consent_missing`);
  - `meeting_clips`, written by `core-api`, at most 10 minutes;
  - `share_links` gains kind `meeting_clip`, `clip_id` and `include_video`.
- `meeting-service`:
  - `GET /api/meetings/{id}/recording`: status `not_enabled`, `recording`, `processing`, `ready`, `failed` or `none`, presigned URLs valid for 600 s, and each segment's offset on the meeting clock; participants only, audited;
  - internal routes, caller `core-api` only: pause, resume and erase the recording, and cut a shared clip's video with ffmpeg (`-c copy`) into `{org_id}/{meeting_id}/clips/{clip_id}.mp4`;
  - egress webhooks now resolve the tenant from the object path, which fixes a 400 on real LiveKit egress events;
  - the backend image includes ffmpeg.
- `core-api`: `GET/POST /api/v1/meetings/{id}/clips`, `DELETE /api/v1/clips/{id}`, `POST /api/v1/clips/{id}/share-links`, public `GET /api/v1/public/clips/{token}`; `api-gateway` routes `/api/v1/clips`.
- Off the record stops the egress and starts a new segment afterwards; a clip that crosses a gap is refused with 409. A meeting erasure deletes its clips and recording objects.
- Audit actions `recording.playback_issued`, `recording.clip_video_issued`, `recording.erased`, `clip.created`, `clip.deleted`.
- `mytheus-ui`: the recording card on the meeting page (a player on the meeting clock with a synced transcript, keyboard controls, captions and off-the-record gaps), clips from a selection or the playhead, a share dialog (expiry, include video, a confirmation for confidential meetings), calm states (recording off, processing, failed, no consent); `/s/[token]` renders clip shares.
- Canon: `meeting_recordings` and `meeting_clips` in `_canon/02-data-model.md`, the routes in `_canon/01-service-catalog.md`.

## 9. Open gaps and follow-ups

- The four owner steps in section 7.
- Measure the plan's targets (speech to card, cards per person, spend per meeting-hour, share of meetings with minutes) once the dashboard runs, and recompute Capacity A17 and the spend model from them.
- Paid evals for `delegated_answer@v1` and `caption_translate@v1`.
- Move the never-record index marks off the 30-second grid, or document the grid in the index guard (deviation 6).
- A Prometheus pushgateway for `NeverRecordLeakEval`.
- Correct the `admin-events` Avro doc string at the next schema change (deviation 14).
- A signed-in walk-through of every new screen in both themes, and the native mobile build.
- Turn recording on in production: `RECORDING_ENABLED`, the R2 recording credentials and the `core-api` caller token for `meeting-service`.
</content>
</invoke>
