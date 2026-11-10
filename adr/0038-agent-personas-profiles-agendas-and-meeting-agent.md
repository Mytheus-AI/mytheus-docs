# ADR-0038: Agent personas, working profiles, linked meetings with agendas, and the MeetingAgent

The owner asked for three things on 2026-11-10: every person's agent gets a name, a face and a personality; onboarding asks each person what they work on so their agent knows what to listen for; and a scheduled meeting can follow up an earlier one, name a host, carry an agenda, and gets one meeting-wide agent that tracks the agenda. This ADR fixes the names, shapes and rules every repo builds against. It amends the locked agent roster ([_canon/06-tech-decisions.md](../_canon/06-tech-decisions.md)) by adding a seventh agent.

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md) - `users` columns, `meetings.previous_meeting_id`, `meeting_agenda_items`, in the same change.
- [_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md) and [_canon/06-tech-decisions.md](../_canon/06-tech-decisions.md) - the roster and `agent-results` kinds.
- [ADR-0030](0030-plan-7-llm-response-cache-and-personal-listener-gating.md) - PersonalListener gating and surfaced bands.
- [ADR-0035](0035-plan-10-context-gap-agent-and-a2a-build.md) - the internal caller-token routes reused by the MeetingAgent.

---

## Status

**Accepted** (2026-11-10).

## Context

1. `users` already carries `role`, `skills` and `bio`, and PersonalListenerAgent scores relevance against them. A user with no role is skipped entirely, and nothing in sign-up or onboarding asks for a role: a new person's agent is silent until they find Settings.
2. There is no agent identity. The six agent names are code classes, not something a person sees.
3. A meeting has a title, a schedule, participants and an implicit host (its creator). There is no agenda and no explicit link to an earlier meeting; "related" meetings are only inferred from shared attendees.
4. Every existing agent works per participant or extracts candidates. None follows the meeting as a whole.

## Decision

### D1. Working profile (identity-tenant-service, `users`)

New columns on `users`:

| Column | Type | Rule |
| --- | --- | --- |
| `listen_for` | `text[] NOT NULL DEFAULT '{}'` | at most 15 items, each 1 to 100 chars, trimmed, de-duplicated |
| `ignore_topics` | `text[] NOT NULL DEFAULT '{}'` | same limits |

`role` (1 to 200 chars), `skills` (shown as "focus areas", at most 30, each up to 100) and `bio` (shown as "what you work on", up to 1000 chars) keep their meaning. Onboarding requires `role` and `bio` (at least 20 chars); the API does not, so existing accounts keep working.

### D2. Agent persona (identity-tenant-service, `users`)

New columns on `users` (one agent per person, so no new table):

| Column | Type | Rule |
| --- | --- | --- |
| `agent_name` | `text NULL` | 1 to 40 chars. NULL means the UI shows "{first name}'s agent" |
| `agent_avatar_seed` | `text NULL` | a doodle seed, `^[a-z0-9]{4,32}$` (the UI's `isDoodleSeed`). NULL means the UI derives one from the user id |
| `agent_personality` | `text NOT NULL DEFAULT 'balanced'` | CHECK in `balanced`, `concise`, `detailed`, `direct`, `friendly` |
| `agent_instructions` | `text NULL` | up to 500 chars, free text from the person ("always note numbers") |

The persona changes how the agent words what it writes for its person (notes, suggestions). It never changes what is captured, the confidence bands, or who can see what. `agent_name` is a display label only; it is not an agent class and never appears in events, logs or metrics as an agent identifier.

### D3. Profile API

- `GET /api/v1/users/me` (new): the caller's own `UserProfile`.
- `PATCH /api/v1/users/{user_id}` accepts, in addition to `role`, `skills`, `bio`: `listen_for`, `ignore_topics`, `agent_name`, `agent_avatar_seed`, `agent_personality`, `agent_instructions`. The existing rule stands for profile fields (self, or admin and above). The `agent_*` fields are **self only**: anyone else gets 403. An empty string for `agent_name` / `agent_instructions` / `agent_avatar_seed` stores NULL. Validation failures are 422 naming the field.
- `UserProfile` gains all six fields. `GET /api/v1/users` (`MemberList` items) gains `agent_name` and `agent_avatar_seed` so a meeting room can label everyone's agent.
- Any change evicts `nexus:{org_id}:user_ctx:{user_id}`, as today.

### D4. User context

`get_user_context` and the gRPC `UserContext` gain, as new fields numbered after the existing ones: `repeated string listen_for = 12; repeated string ignore_topics = 13; string agent_name = 14; string agent_personality = 15; string agent_instructions = 16;`. The dict returned by the Python function uses the same keys. The avatar seed is not part of the context: no agent needs it.

### D5. Meetings: host, follow-up link, agenda

- `meetings.previous_meeting_id uuid NULL`, FK `meetings(id) ON DELETE SET NULL`. It must name a meeting the caller can see in the same tenant and not the meeting itself; otherwise 422 (`previous_meeting_id`). One link per meeting; a chain of follow-ups is a chain of links. No recurring-series entity.
- The host is the participant with `meeting_participants.role_in_meeting = 'host'`. There is exactly one. `MeetingCreate.host_user_id` is optional (default: the creator) and must be the creator or one of `participant_ids` (422 otherwise). The creator is always a participant (`attendee` when not host).
- New table `meeting_agenda_items`, owned by the service that owns `meetings`, RLS by `org_id` with the canonical `tenant_isolation` policy:

| Column | Type |
| --- | --- |
| `id` | `uuid PK` |
| `org_id` | `uuid NOT NULL` |
| `meeting_id` | `uuid NOT NULL` FK `meetings(id) ON DELETE CASCADE` |
| `position` | `int NOT NULL` (0-based order) |
| `title` | `text NOT NULL` (1 to 200 chars) |
| `owner_user_id` | `uuid NULL` (a participant) |
| `duration_minutes` | `int NULL` (1 to 240) |
| `status` | `text NOT NULL DEFAULT 'pending'`, CHECK in `pending`, `in_progress`, `covered`, `skipped` |
| `status_source` | `text NULL`, CHECK in `agent`, `host` |
| `agent_note` | `text NULL` (up to 500 chars) |
| `created_at`, `updated_at` | `timestamptz NOT NULL DEFAULT now()` |

At most 20 items per meeting.

### D6. Meeting API

- `MeetingCreate` gains `host_user_id`, `previous_meeting_id`, and `agenda: [{title, owner_user_id?, duration_minutes?}]` (at most 20; owners must be participants).
- `Meeting` (create, get, list) gains `host_user_id` and `previous_meeting_id`. `GET /api/v1/meetings/{id}` also returns `previous_meeting: {id, title, scheduled_start} | null` and `agenda: AgendaItem[]` ordered by `position`.
- `PATCH /api/v1/meetings/{id}` accepts `host_user_id` (the new host must be a participant) and `previous_meeting_id` (same rules as create). Only the current host or the creator may change either; otherwise 403.
- `PUT /api/v1/meetings/{id}/agenda` `{items: [{id?, title, owner_user_id?, duration_minutes?}]}` replaces the agenda (host or creator; not after the meeting has ended, 409). Items sent with an existing `id` keep their status; new ones start `pending`; items left out are deleted. Returns `AgendaItem[]`.
- `PATCH /api/v1/meetings/{id}/agenda/{item_id}` `{status}` (host or creator): sets `status`, `status_source = 'host'`. Returns `AgendaItem`.
- `GET /api/v1/meetings/{id}/carry-forward`: for a caller who can see the meeting, what the previous meeting left open: `{previous_meeting: {id, title, scheduled_start} | null, decisions: [{id, content}], open_actions: [{id, title, owner_user_id, due_date}], saved_questions: [{id, question}]}`. `decisions` are confirmed decisions of the previous meeting; `open_actions` its action items not done; `saved_questions` the caller's own suggestions with status `saved` from that meeting. Empty lists, and `previous_meeting: null`, when there is no link.
- `AgendaItem` = `{id, position, title, owner_user_id, duration_minutes, status, status_source, agent_note, updated_at}`.
- Internal, caller-token (same scheme as `related-attendees`, caller `agent-runtime-service`): `GET /internal/meetings/{id}/agenda` returns `{host_user_id, previous_meeting_id, scheduled_start, scheduled_end (ISO-8601 or null), items: [{id, position, title, duration_minutes, status, status_source}], carry_forward: {decisions: [str], open_actions: [str]}}`.

### D7. The MeetingAgent

A seventh agent, `MeetingAgent`, in `agent-runtime-service`: one per meeting, not per participant. It runs on the same 30-second windows as the others, and only when the meeting has at least one agenda item.

- Input: the window transcript (speakers as "Participant N"), the agenda with current statuses and how long the in-progress item has run, and the previous meeting's open items as plain strings.
- Output: at most **one** record per meeting per window on `agent-results`, with `agent = MeetingAgent`, `user_id` = the host's user id and `kind = meeting_update`. One record, because the canonical `agent-results` idempotency key is `agentresult:{meeting_id}:{window_id}:{user_id}:{agent}` (ADR-0032): a second record in the same window would be deduplicated away. This is the same batching pattern as ADR-0032 and ADR-0034. No record is emitted for a window with nothing to say.
  - `payload_json = {"updates": [{"item_id": uuid, "status": "in_progress" | "covered", "note": str (up to 300 chars)}], "nudge": {"id": uuid, "type": "off_agenda" | "over_time" | "unaddressed", "item_id": uuid | null, "content": str (up to 200 chars)} | null, "surfaced": true}`. At most one update per item per window.
  - `over_time` is computed, not inferred: an item in progress longer than its `duration_minutes`, once per item. `off_agenda` needs confidence at least 0.75. `unaddressed` names a carried-forward open item nobody has raised after 70% of the scheduled time. At most one nudge per meeting per 5 minutes.
- Fast tier model (Haiku 5.5), metered against the tenant budget like every agent; skipped when the budget is blocked.
- It never writes Postgres. `core-api`'s `agent-results` consumer applies each entry of `updates` in a `meeting_update` record: only to an item of that meeting, never when `status_source = 'host'` (the host wins), and never from `covered` back to `in_progress`. It sets `status_source = 'agent'` and `agent_note`. Idempotent under redelivery. The `nudge` is not persisted.

### D8. Events

`agent-results` `v1` gains, BACKWARD compatibly: `AgentClass` symbol `MeetingAgent`; `AgentResultKind` symbol `meeting_update`. Every consumer must tolerate them before the first producer ships them. The idempotency key is unchanged.

### D9. Realtime

`realtime-gateway` splits each `MeetingAgent` `meeting_update` record:

- each entry of `updates` as `"agent-agenda"` `{meetingId, payload: {itemId, status, note}}` to **every** socket on the meeting channel (the agenda is shared);
- a non-null `nudge` as `"agent-nudge"` `{meetingId, payload: {id, type, itemId, content}}` **only** to the host's sockets (the record's `user_id`).

A client re-hydrates the agenda from `GET /api/v1/meetings/{id}`; nudges are not replayed.

### D10. Persona and profile in prompts

PersonalListenerAgent (new prompt version) renders the profile block with `listen_for` ("Listen especially for") and `ignore_topics` ("Ignore"), and a voice line from `agent_personality` plus `agent_instructions` for how its notes are worded. QuestionAgent uses the same profile block and voice. Nothing else changes: the confidence thresholds, gating and bands are those of ADR-0030 and ADR-0034. `agent_instructions` is user text: it is placed in a delimited block and the prompt states it may change wording only, never what is captured or who sees it.

### D11. Product surface

- Onboarding, after the workspace is chosen: "About you" (`role`, "what you work on" = `bio`, focus areas = `skills`, listen for, ignore), then "Meet your agent" (name, doodle face, personality preset, optional instructions; skippable with defaults), then the dashboard. A signed-in person with no `role` sees a dashboard prompt that opens "About you".
- Settings: Profile gains "what you work on", listen for and ignore; a new "Your agent" page edits the persona.
- New meeting: "Follow-up to" (a recent meeting), host, and an agenda editor (title, owner, minutes; add, remove, reorder).
- Meeting page: host, follow-up link, the agenda with statuses and agent notes, and what was carried forward.
- Meeting room: your agent's face and name head your sidebar; each participant tile carries a small badge of their agent's face; a shared live agenda panel; the host can mark items covered or skipped; nudges reach the host only.

## Consequences

- The roster is seven agents. Canon and every list of agent names are updated in this change.
- Existing accounts and meetings are unaffected: every new column is nullable or defaulted, and a meeting without an agenda never invokes the MeetingAgent.
- Deploy order: events, then consumers (`core-api`, `notification-service`, `realtime-gateway`), then `agent-runtime-service`, then the UI.
- Not built now: a recurring-series entity, agents that attend without their person, an agent that speaks, and recap generation beyond the per-item agent notes.
