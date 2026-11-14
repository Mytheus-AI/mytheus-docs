# ADR-0047: Ask Myte

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md): `myte_threads`, `myte_messages` (core-api).
- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md): core-api `/api/v1/myte/*`, rag-service `POST /api/rag/myte`, the new bff MCP tools.
- [ADR-0033](0033-plan-9-rag-retrieval-upgrades.md) (participant-scoped retrieval), [ADR-0038](0038-agent-personas-profiles-agendas-and-meeting-agent.md) (persona: `agent_name`, `agent_avatar_seed`, `agent_personality`, `agent_instructions`), [ADR-0040](0040-polish-program.md) (the MCP server, "Ask your Myte, live"), [ADR-0042](0042-myte-user-facing-name.md) (the name), [ADR-0029](0029-plan-7-budget-scope-tier-mapping-and-metering-ownership.md) (budget and metering).
- The approved plan with mockups: "Ask Myte Plan" (owner-approved 2026-11-14).

## Status

**Accepted** (2026-11-14).

## Context

Mytheus can answer questions in three narrow places: the search page's Ask (`POST /api/rag/query`), the meeting room's private ask box (`POST /api/v1/ask`) and the MCP server Claude and IDEs use (`/mcp`, 11 tools). There is no place to simply talk to your Myte about everything, and the answers there are single-shot retrieval: they cannot look at your tasks, your schedule and a decision together, cannot act, and do not speak as your Myte. The owner asked for an "Ask Myte" feature covering the whole platform, in character, to an S-grade standard, with no open questions.

## Decision

### D1 One assistant, three entry layers

`/ask` (a full page with history and a source panel), a sheet on every app page (`⌘J`, page-aware), and entry points on Your Myte, Home, meeting and decision pages, the meeting room and the command palette. All of them read and write the same threads.

### D2 The loop lives in rag-service

`POST /api/rag/myte` in rag-service (mytheus-intelligence), behind the gateway's existing `/api/rag` route. rag-service already verifies Clerk sessions, owns "ask", and has the model router (budget gate, PII redaction, prompt caching, metering). No new deployable and no gateway change for the loop. The router gains `converse()`: a multi-turn, tool-calling, streaming call with the same budget, redaction and metering as `complete()`, one `usage-events` record per model round. Tier `extract` (Sonnet 5.5, adaptive thinking, effort `medium`) answers; tier `triage` (Haiku 5.5) writes the thread title and the three follow-ups.

*Why not agent-runtime-service:* it runs as a consumer worker with no HTTP surface in production; adding one is a new container for no gain. *Why not bff:* bff aggregates and owns no model calls (CLAUDE.md); the router and its guards live in intelligence.

### D3 Tools are the MCP tools, called as the person

The loop is an MCP client of Mytheus's own server: JSON-RPC `tools/call` on bff `POST /mcp` (`BFF_URL`, default `http://bff:8000`) with the asker's own `Authorization` header. So Myte can never see more than the person can through REST, and every tool added for Myte also serves Claude. Read tools only: `search_meetings`, `ask_meetings`, `get_meeting_outcome`, `get_decision_record`, `company_as_of`, `list_my_open_actions`, `list_my_promises`, `check_my_capacity`, `check_against_decisions`, plus four new bff tools (D6). `create_action` and `add_promise` are never exposed to the model.

At most 8 tool rounds and 45 s of tool time per answer (the Clerk session token lives 60 s; the UI server mints a fresh one per request with `skipCache`). Tool calls in one round run concurrently.

### D4 Writes are proposals the person confirms

Two pseudo-tools exist only inside the loop and call nothing:
- `propose_action` `{kind: "task"|"promise"|"attendance"|"agenda_item", title, due_date?, meeting_id?, choice?: "attend"|"agent"|"skip", note?}` becomes a `proposal` event.
- `show_card` `{kind: "meeting"|"decision"|"tasks"|"promises"|"capacity", source_ns: [n]}` becomes a `card` event built from those sources.

The UI's Confirm calls the same server actions the rest of the app uses (create task, add promise, set attendance, add agenda item), so the write, its audit row and its events are identical to a manual one. No silent writes.

### D5 Citations

Every item a tool returns is registered as a numbered source `n` (1-based, stable within one answer) and given to the model as `[n] kind | title | text`. The model cites inline as `[n]`. The system prompt forbids factual claims without a source; with nothing relevant the answer abstains in plain words and `done.abstained` is true.

### D6 New bff MCP tools (read-only)

- `list_my_schedule` `{days: 1..14 = 7}`: the caller's upcoming meetings (id, title, start, end, host, attendee count, the caller's attendance choice, live/scheduled).
- `find_people` `{query, limit: 1..10 = 5}`: members by name or email, with role, team and department.
- `list_my_inbox` `{limit: 1..20 = 10}`: unread notifications and detections waiting for review.
- `get_my_myte` `{}`: the caller's Myte persona and listening settings, and how many notes it kept in the last 7 days.

### D7 Threads are private, stored by core-api

`myte_threads` and `myte_messages` (canon 02). Only their owner reads or writes them, admins included: every route filters on `user_id = caller`, a thread of anyone else is `404`. Kept until the owner deletes one thread or all. Routes under `/api/v1/myte` (gateway entry to core-api):

| Route | Body / result |
| --- | --- |
| `GET /api/v1/myte/threads?limit=30&cursor=&q=` | `{items: [{id, title, pinned, created_at, last_message_at}], next_cursor}`, pinned first, then `last_message_at` desc; `q` matches title or message text |
| `POST /api/v1/myte/threads` | `{title?, scope?}` -> thread (201) |
| `GET /api/v1/myte/threads/{id}` | thread + `messages` ascending (max 200) |
| `PATCH /api/v1/myte/threads/{id}` | `{title?, pinned?}` -> thread |
| `DELETE /api/v1/myte/threads/{id}` | 204; the thread and its messages are hard-deleted |
| `DELETE /api/v1/myte/threads` | 204; all the caller's threads |
| `POST /api/v1/myte/threads/{id}/messages` | `{messages: [{role: "user"|"myte", content, payload?}]}` (1 to 2 items, `Idempotency-Key`) -> `{items: [message]}` (201); bumps `last_message_at` |
| `PATCH /api/v1/myte/messages/{id}` | `{feedback: 1|-1|0, feedback_reason?}` -> message |

A message is `{id, thread_id, role, content, payload, feedback, feedback_reason, created_at}`. `payload` (jsonb, max 64 KB) holds what the UI renders besides text: `steps`, `sources`, `cards`, `proposals` (with their confirmed state), `followups`, `abstained`, `degraded`.

### D8 The stream

`POST /api/rag/myte`, `Content-Type: text/event-stream`. Request:

```json
{
  "question": "string, 1..2000",
  "history": [{"role": "user|myte", "content": "string"}],
  "scope": {"kind": "all|meeting|person|as_of", "meeting_id": null, "person_id": null, "as_of": null},
  "persona": {"name": "Pip", "personality": "balanced|concise|detailed|direct|friendly", "instructions": "string|null"},
  "page": {"kind": "meeting|decision|person|null", "id": null, "title": null},
  "want_title": true
}
```

`history` is at most the last 12 messages; `persona` comes from the asker's own profile (it only shapes tone, never permissions). Events, each `event: <name>` + `data: <json>`:

| Event | Data |
| --- | --- |
| `step` | `{id, label, state: "running"|"done"|"failed"}`: plain words, e.g. "Looking through your meetings" |
| `delta` | `{text}` |
| `source` | `{n, kind: "transcript"|"decision"|"action"|"promise"|"meeting"|"document"|"person"|"notification", title, text, meeting_id?, meeting_title?, decision_id?, action_id?, occurred_at?, speaker?, start_ms?}` |
| `card` | `{kind, source_ns, data}` |
| `proposal` | `{id, kind, title, due_date?, meeting_id?, choice?, note?}` |
| `followups` | `{items: [string, string, string]}` |
| `title` | `{text}` (only when `want_title`) |
| `done` | `{abstained, degraded, cited: [n]}` |
| `error` | `{reason: "budget"|"not_connected"|"unavailable"|"invalid"|"session", message}` |

The UI server (`mytheus-ui` route handler `POST /api/ask-myte`) relays the stream, then creates the thread if needed and appends the user and Myte messages through core-api, and ends with its own `saved` event `{thread_id, message_ids}`. Persisting from the UI server, with a fresh token, keeps rag-service a non-writer and survives a token that expired during a long answer.

### D9 In character

Answers speak as the person's own Myte: first person, by its name, in its personality (balanced, concise, detailed, direct, friendly), following its instructions. The face is its plush (`agent_avatar_seed`), live where motion is allowed: `listening` while the person types (eyes on the caret), `thinking` while `step`s run, `speaking` while `delta`s stream (and to its own voice when read aloud), `happy` once when a proposal is confirmed, `idle` otherwise. People keep their doodle avatars. Character never changes facts: citations and abstention rules win.

### D10 States

Nothing found (abstain, say where it looked), partial (answer and name the gap), outside access (never hint at hidden meetings), not connected (`error.reason = not_connected`, who can fix it), budget (`budget`, when it comes back), session, stopped (keep the streamed text, Retry), offline.

## Consequences

- One new pair of tables (core-api), one new route group (core-api), one new streaming route (rag-service), four MCP tools (bff, also available to Claude), the router's `converse()`. No new deployable, topic or secret.
- Every answer is metered per round against the workspace budget, and redacted on the way out.
- An 80-question eval set across the ten areas (including refusal, abstention and cross-user privacy cases) gates prompt changes in CI.
