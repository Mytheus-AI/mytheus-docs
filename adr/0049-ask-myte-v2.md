# ADR-0049: Ask Myte v2 (peer Mytes, proactive, actions, memory, routines, voice, attachments)

## Related documents

- [ADR-0047](0047-ask-myte.md): Ask Myte v1. Everything here extends its contracts; nothing in v1 is removed.
- [_canon/02-data-model.md](../_canon/02-data-model.md): `myte_peer_questions`, `myte_memories`, `myte_routines` (core-api), and the `proposal_states` key of `myte_messages.payload`.
- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md): the new core-api routes.

## Status

**Accepted** (2026-11-14).

## Context

Ask Myte v1 answered single questions well: whole-platform lookups, citations, confirmed proposals, in character. A review against best-in-class assistants rated it A-tier: it only responds, acts in four ways, forgets everything outside Settings, and cannot reach the one thing only Mytheus has, another person's Myte. Reliability also had open items: the 80-question set had never run against the real model, confirmed proposals lived only in a browser, metering had no user, and latency was unmeasured. The owner asked for all of it, built, with no questions.

## Decision

### Part R: reliability

- **R1 Live eval.** The 80-question set runs against the real model (`EVAL_LIVE=1`), failures are fixed in the prompt or loop, and the gate (85%) must pass before v2 ships. Run on demand, never in CI.
- **R2 Proposal state is stored.** `PATCH /api/v1/myte/messages/{id}` also accepts `{proposal_states: {<proposal_id>: "confirmed"|"dismissed"}}`, merged into `payload.proposal_states` (owner only, Myte messages only, at most 20 keys). The UI reads it on load, so Confirm never reappears on another device.
- **R3 Per-person metering and cap.** rag-service resolves the asker's `users.id` by calling identity-tenant-service `GET /users/me` with the asker's own bearer (`IDENTITY_URL`, default `http://identity-tenant-service:8000`; cached in Redis `nexus:{org_id}:myte:uid:{clerk_sub}` for 10 minutes). Every round is metered with that `user_id`, and a per-person daily cap (`MYTE_USER_DAILY_UNITS`, default 300000 budget units) is kept in `nexus:{org_id}:myte:units:{user_id}:{YYYY-MM-DD}` (TTL 2 days). Over the cap the stream answers `error {reason: "budget"}` with a plain message naming tomorrow.
- **R4 Latency.** rag-service records `myte_first_step_seconds`, `myte_first_text_seconds` and `myte_answer_seconds` histograms (Prometheus, like the agent metrics) and logs them per answer. Targets: first step < 1 s, first text < 2.5 s at p50.

### Part F: features

- **F1 Ask another person's Myte.** A person can ask a teammate's Myte a question. Asking is a write, so it is a proposal (`propose_action kind: "ask_myte"`, `{person_id, person_name, question}`) the asker confirms. core-api stores it in `myte_peer_questions` and notifies the recipient through the same path `action_status_requests` ("updates you owe") uses. The recipient sees it in Inbox ("Questions for your Myte"); opening it lets their own Myte draft an answer from the recipient's own data (an Ask Myte stream run in the recipient's session, `mode: "peer_answer"`), which the recipient edits and sends, or declines. Nothing leaves without the recipient's review. The asker's thread shows a waiting card that turns into the answer, signed by the teammate's Myte (its plush and name). Routes:

  | Route | Body / result |
  | --- | --- |
  | `POST /api/v1/myte/peer-questions` | `{to_user_id, question (1..1000), thread_id?}` -> question (201); `Idempotency-Key`; 404 for someone outside the workspace; max 20 open per asker |
  | `GET /api/v1/myte/peer-questions?box=inbox|sent&status=` | `{items: [question]}` newest first |
  | `GET /api/v1/myte/peer-questions/{id}` | question (visible to asker and recipient only) |
  | `POST /api/v1/myte/peer-questions/{id}/answer` | `{answer (1..8000)}` -> question; recipient only; once |
  | `POST /api/v1/myte/peer-questions/{id}/decline` | `{reason?}` -> question; recipient only |

  A question is `{id, from_user, to_user, question, thread_id, status: "pending"|"answered"|"declined", answer, decline_reason, created_at, answered_at}`, where `from_user`/`to_user` are `{id, name, agent_name, agent_avatar_seed}`.

- **F2 Myte reaches out first.** In the UI, from data the app already reads: the floating Myte shows a nudge bubble and the Home card "Today from Myte" lists at most three openers: a meeting starting within 15 minutes ("Test starts in 10 minutes. Want a 30-second prep?"), a promise or task due today or overdue, new decisions since the last visit, a teammate's question waiting, a routine result ready. One tap starts that question. Dismissed nudges stay dismissed for the day.

- **F3 More actions.** `propose_action` kinds grow to: `task`, `promise`, `attendance`, `agenda_item` (v1), plus `schedule_meeting {title, start, end, attendee_ids}`, `invite_people {meeting_id, user_ids}`, `slack_post {text, channel?}`, `export_decisions {destination: "notion"|"confluence"|"slack", meeting_id?, decision_ids?}`, `share_decision {decision_id}`, `ask_myte` (F1) and `memory` (F4). Each confirms through the existing server action for that write (new meeting, invitations, decision export, share links); an action whose integration is not connected shows that and links to Settings.

- **F4 Memory.** Myte proposes remembering a durable preference or fact the person states (`kind: "memory"`, `{text (1..300)}`); on Confirm it is stored in `myte_memories`. The UI server sends up to 30 memories with every ask (`memories: [{id, text}]`), and rag-service puts them in the context block as "What you remember about them". Your Myte shows "What Myte remembers", with delete. Routes: `GET /api/v1/myte/memories`, `POST /api/v1/myte/memories {text, source_thread_id?}` (max 100 per person), `DELETE /api/v1/myte/memories/{id}`. Owner only.

- **F5 Routines (recurring questions).** "Every Monday at 9, tell me what changed in pricing." Stored in `myte_routines` `{id, question, scope, cadence: "daily"|"weekdays"|"weekly", weekday (0..6, weekly only), hour (0..23), timezone, enabled, next_run_at, last_run_at, last_thread_id}`. Myte has no session of its own, so a due routine runs the next time the person opens Mytheus: the app asks for due routines, runs each through the normal stream into a new thread titled with the routine, and records it (`POST /api/v1/myte/routines/{id}/ran {thread_id}`, which sets `last_run_at` and computes `next_run_at`). The nudge (F2) and the thread say "Ready". Routes: `GET/POST /api/v1/myte/routines`, `PATCH/DELETE /api/v1/myte/routines/{id}`, `GET /api/v1/myte/routines/due`, `POST /api/v1/myte/routines/{id}/ran`. Max 10 per person. Created from any answer ("Make this a routine") or Your Myte.

- **F6 Talk to Myte.** A voice mode on `/ask` and in the sheet: the large live plush listens (browser speech recognition, with the mic level), thinks while steps run, and speaks the answer in its own voice (browser speech synthesis, the briefing's voice choice), then listens again. Hidden where the browser has no speech recognition. The transcript is the same thread.

- **F7 Bring your own material.** Paste text or drop a `.txt`, `.md` or `.csv` file (at most 3, 40,000 characters in total). The request carries `attachments: [{name, text}]`; rag-service registers each as a numbered `document` source, so Myte can summarise it, answer about it, or check it against decisions with `check_against_decisions`. Attachments are not stored beyond the message payload's names.

### Contract additions to ADR-0047 D8 (the stream request)

`memories: [{id, text}]` (<= 30), `attachments: [{name, text}]` (<= 3, <= 40,000 chars total), `mode: "ask"|"peer_answer"` (default `ask`), and for `peer_answer`: `peer: {question_id, from_name, question}`. In `peer_answer` mode Myte drafts a reply to a teammate's question from the recipient's own data, shares only what answers the question, never pastes private notes wholesale, and may propose nothing. `scope.label` (already shipped) is part of the contract.

## Consequences

- Three new core-api tables and their routes; one payload key; no new deployable or topic. rag-service gains an identity call and Redis counters.
- Peer questions and routines depend on the recipient or the asker opening Mytheus; this is recorded honestly in the UI ("drafted when Samar opens Mytheus", "ready when you next open Mytheus").
- Per-person caps stop one person from spending a workspace's budget on chat.
