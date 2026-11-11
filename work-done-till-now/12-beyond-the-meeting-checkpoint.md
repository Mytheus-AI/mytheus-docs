# 12. Beyond the meeting: new features, checkpoint (2026-11-11)

This is a checkpoint, not a finished band. The owner paused the program on 2026-11-11 to save tokens. It records what is built and live, what is partly built on local branches, and what remains, so the work can be picked up cold.

The spec is the "After the meeting" proposal, version 4, at https://claude.ai/artifact/58Rzzq9xSKjZornejPZC6J. It has three layers:

1. After the meeting.
2. Beyond the meeting: the company's working memory in every tool.
3. The company that runs on agents.

It also sets trust rules:

- People own their agent.
- Everything an agent does is visible and undoable.
- Agents propose; people decide.
- No surveillance.
- Deny by default.
- An agent that isn't sure asks instead of guessing.

## 1. Live in production

Production DB head is `e4f5a6b7c8d9`. The production Schema Registry has 22 subjects, and `agent-results` includes the `room_signals` kind.

| Feature | Layer | What it does | Main pieces |
|---|---|---|---|
| Decisions that follow you | 2 | Text written anywhere is checked against confirmed decisions. On a clash it shows "This was decided otherwise on 3 Oct", with Open / Reopen / Dismiss. | rag-service `POST /internal/decisions/check` (`decision_conflict_check@v1`); core-api `POST /api/v1/decisions/check`, `/decision-challenges`, `/decision-reopen-requests`; Slack message events and `/webhooks/slack/interactions`; Chrome extension notes in Docs, Gmail and Jira (off by default, per site); MCP `check_against_decisions`. Tables `decision_challenges`, `decision_reopen_requests`, `decision_check_dismissals`. |
| One promise ledger | 2 | Promises from meetings, phone recordings, Slack (opt-in), manual entry and MCP, plus a capacity check when you say yes ("That's your seventh promise due Friday"). | `/api/v1/promises` (capacity, date proposals, hand-off, capture preferences); `commitments.channel` and `source_*`; rag-service `/internal/promises/extract` (`promise_extract@v1`); MCP `check_my_capacity`, `add_promise`. |
| The last word | 1 | People who were spoken over, whose hand was never called, or who were cut off get a private nudge afterwards to add their point. | `last_word@v2`; socket `floor-attempt`; tables `room_signals`, `last_word_nudges`, `late_contributions`. |
| Agreement check (fake-consensus detector) | 1 | Each decision gets a read of how firm the agreement was. People who hedged or stayed silent are asked privately, and the owner sees only totals. | `consensus_read@v1`; tables `decision_agreement_reads`, `agreement_checks`. |
| Pick up where we stopped | 1 | The next meeting opens on the unfinished thread, with the quote or a replay. | `resume_point@v1`; socket `resume-point`; table `resume_points`. |
| A briefing from your agent | 1 | A spoken 90-second briefing in the agent's persona, on web and mobile. | `briefing@v1`; `/api/v1/briefings`; Redis ring `nexus:{org}:briefings:ring`; tables `agent_briefings`, `briefing_preferences`. Server text-to-speech is not connected; the browser and device voices work. |
| Decisions that know why | 1 | A decision's reasons are stored as watched assumptions. When one breaks, the owner is told, with re-confirm / reopen / book a revisit. | `decision_assumptions@v1`, `assumption_check@v1`; schedule `assumption-watch`; tables `decision_assumptions`, `assumption_evidence`, `decision_why_state`. |
| Blast radius | 1 | A ring map of what a decision changes, plus settle threads when it clashes with another team's decision. | `decision_ripple@v1`; `/api/v1/decisions/{id}/blast-radius`, `/api/v1/settlements`; tables `decision_ripple_links`, `decision_clashes`, `decision_settlements`, `settlement_messages`. |
| In every meeting at once | 3 | Choose attend, "my agent listens" or skip for each meeting; interruptions come only when needed; an evening digest; a week view. | `interrupt_gate@v1`; `/api/v1/listening/*`; per-person socket channel `nexus:{org}:pubsub:user:{uid}`; schedule `listening-evening`; tables `attendance_rules`, `listening_preferences`, `standing_positions`, `agent_interruptions`, `meeting_views`, `evening_digests`. |
| Decisions as bets | 3 | Decisions record what they expect to happen, are checked on a date, and are scored (aggregate only, never per person). | `decision_bet@v1`; `/api/v1/decisions/{id}/bet`, `/api/v1/bets/*`; schedule `decision-bets-daily`; tables `decision_bets`, `decision_bet_evidence`. |
| The company time machine | 3 | The company as it stood on any date, a diff between two dates, and a topic timeline. | `/api/v1/time-machine/{snapshot,diff,timeline}`; table `record_status_history`; MCP `company_as_of`. |

PRs:

| Repo | PR |
|---|---|
| backend | #25 |
| intelligence | #15 |
| realtime | #15 |
| events | #7 |
| platform | #17 |
| ui | #51 |
| mobile | #4 |

Before this program, and also live:

- MCP OAuth for Claude custom connectors: backend #24, ui #49.
- UI polish: custom dropdowns, the New meeting window, the raise-hand queue, the room header and scrollbars (ui #50, realtime #14).

### Verification (as run)

- **Backend:** 1562 tests passed on a fresh database. oasdiff exited 0 on 7 specs. Alembic upgrade, downgrade and upgrade all ran. One linear migration chain: `e7f8 → f8a9 → a0b1 → b1c2 → d3e4 → c2d3 → f5a6 → e4f5`.
- **Intelligence:** 1276 offline tests passed.
- **Live evals,** with real answers recorded:
  - Every current prompt passed its thresholds; the deliberately worse prompts failed.
  - The room-signals eval was found not to enforce its baselines. With that fixed, the last-word topic hit rate went from 0.33 to 1.0 (`last_word@v2`).
  - `assumption_check` is now gated on recall ≥ 0.9 and on F1.
- **UI:** 493 unit tests and 353 a11y tests passed, and `next build` succeeded.
- **Realtime:** 155 of 156 tests passed. The one failure also fails locally on main.
- **Production:**
  - The 3 new Temporal schedules were created.
  - The new routes answer 401 without a token.
  - The Promise ledger, Listening and Time machine pages render with real data, checked signed in on dashboard.mytheus.cloud.

## 2. Partly built (local branches, not pushed, not deployed)

Wave C was stopped mid-build. All the work is in local git worktrees under `/Users/sushan/Projects/Mytheus/wt/`. Committed work is on the branch named in each row; "uncommitted" means files changed but not yet committed.

| Feature(s) | Branch | State when stopped |
|---|---|---|
| Mandates and Rules the company follows (Layer 3) | `feat/mandates-rules` | backend: 22 uncommitted files; ui: 31 uncommitted files; intelligence: not started |
| Overtime and The standup nobody attends | `feat/overtime-standup` | intelligence: 2 commits; ui: 4 commits; backend: 15 uncommitted files |
| Do the homework | `feat/do-the-homework` | backend: 1 commit; ui: 4 commits; intelligence: not started |
| Dress rehearsal and Ask the company | `feat/rehearsal-ask-company` | intelligence: 1 commit; ui: 1 commit plus 7 uncommitted; backend: 10 uncommitted |
| Early warning, the company memo, your track record | `feat/warning-memo-record` | backend: 2 commits plus 1 uncommitted; intelligence: 1 commit; ui: 22 uncommitted |
| Inherit an agent and What we promised customers | `feat/inherit-customers` | backend: 3 commits; intelligence: 2 commits; ui: 5 commits |
| Follow-up fixes from waves A and B | `fix/beyond-followups` | backend: 3 commits; intelligence: 1 commit; ui: 4 commits; mobile: 2 commits |

The follow-up fixes branch was working on:

- The `/meetings/listening` breadcrumb, which reads "Meetings › Meeting".
- Re-emitting the actions event when a date proposal on an action is accepted. Today the graph and Jira keep the old date.
- Late-contribution and agreement chips on the `/decisions` log.
- Mobile screens for the wave A and B features.
- Pruning the unused PromisesView and usePromises.
- An admin route to revoke an OAuth client.

None of this has passed its full gates, been integrated, or been live-evaluated. Before trusting any branch:

1. Finish it.
2. Run every repo's gates.
3. Integrate the branches. The Inbox, the decision page, the agent-runtime consumer, `ROUTE_MAP`, the erasure lists and the prompt registry are where they conflict.
4. Re-chain migrations off `e4f5a6b7c8d9`. The planned revision ids are `a6b7c8d9e0f1`, `b7c8d9e0f1a2`, `c8d9e0f1a2b3`, `d9e0f1a2b3c4`, `e0f1a2b3c4d5`, `f1a2b3c4d5e6` and `a2b3c4d5e6f7`.
5. Run the live evals for each new prompt.

## 3. Not started

Wave D, the furthest reach of Layer 3:

- **The self-driving calendar:** agents book meetings only when they cannot settle something themselves, and a meeting cancels itself when the question is settled first.
- **The company twin:** simulate a reorganisation or strategy on a model of the company first.
- **Agents across company lines:** two tenants' agents settle routine work within each side's mandate. This needs the A2A layer extended between tenants, and an ADR first.

Docs for waves A and B are still to write:

- Canon: the tables, routes, events, Redis keys, prompts, consumer groups and schedules listed in section 1.
- An ADR for the "beyond the meeting" program.

## 4. Steps for the owner (blocked for the agent: writes to the production server)

1. Run `scripts/push-secrets.sh polish` again in mytheus-platform. It now also creates the core-api and integration-service entries in `MYTHEUS_RAG_CALLER_TOKENS`, which "decisions that follow you" and Slack promise capture need. Then redeploy backend and intelligence.
2. Slack app: add the scopes `channels:history`, `groups:history`, `im:history`, `mpim:history`, `channels:read`, `groups:read` and `commands`. Subscribe to `message.channels`, `message.groups`, `message.im` and `message.mpim`. Set the Interactivity URL to `{OAUTH_REDIRECT_BASE}/webhooks/slack/interactions` and add the message shortcut `check_against_decisions`. Workspace admins then reconnect Slack.
3. Optional:
   - turn on the `observability` compose profile;
   - set admin-events to `cleanup.policy=delete` with 365-day retention;
   - mobile: Clerk Native API, `EXPO_PUBLIC_*`, `eas init`, then a native build (adds expo-speech and expo-audio);
   - server text-to-speech: `TTS_PROVIDER` plus a key;
   - HubSpot `crm.objects.deals.read` for bet checks and blast radius.
4. Operator SQL to retire the test OAuth client left by the connector check: `UPDATE oauth_clients SET revoked_at = now() WHERE client_name = 'Probe (delete me)';`

## 5. How to resume

Resume wave C one branch at a time rather than all at once, to keep token use down:

1. Finish the follow-up fixes first.
2. Then finish the features that are nearly done, which are the ones with commits in every repo:
   - Inherit an agent and What we promised customers.
   - Overtime and The standup nobody attends.
   - Do the homework.
3. Integrate and ship each one on its own.
4. Wave D last.

The builder brief that every builder followed is preserved in the session scratchpad. It is reproduced here in short:

- Worktree-only work.
- Real gates.
- Additive APIs: oasdiff must pass and the CI gate is never edited.
- No Avro change unless essential.
- Every new prompt is budget-gated, with an offline eval.
- Agents propose, people decide.
- Tokens only in the UI, every state designed.
