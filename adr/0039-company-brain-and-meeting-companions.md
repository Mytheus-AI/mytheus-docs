# ADR-0039: The company brain, and ten features built on it

On 2026-11-10 the owner asked for an organisation setup that gives every agent the company's context, and for ten further features, all built at once. This ADR fixes the names, shapes and rules every repo builds against. It builds on [ADR-0038](0038-agent-personas-profiles-agendas-and-meeting-agent.md) (personas, agendas, `MeetingAgent`).

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md): every table below, in the same change.
- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md): the routes and the new gRPC method.
- [ADR-0013](0013-agent-to-agent-authorization.md): the A2A PDP that gates anything shown to someone outside a meeting.
- [ADR-0030](0030-plan-7-llm-response-cache-and-personal-listener-gating.md), [ADR-0034](0034-plan-9-question-agent-suggestions.md): bands and gating, unchanged.

---

## Status

**Accepted** (2026-11-10).

## Context

An agent knows its person (ADR-0038) and its meeting, but nothing about the company: what it sells, what its words mean, who owns what, what matters this quarter, or what must never be recorded. Transcription mishears product and customer names for the same reason. The owner also asked for: drafted agendas, pre-meeting briefs, sending your agent in your place, topic watch across meetings, "we already decided this", a promises list, status requests before follow-ups, a meeting health card, a new joiner brief, and company vocabulary in captions.

## Decision

Every table below is tenant-scoped with `org_id`, ENABLE and FORCE row-level security and the canonical `tenant_isolation` policy. Every write route that changes company-wide data requires the org role `owner` or `admin`; reads are open to members. Limits are enforced by the API (422 naming the field) and mirrored by the UI.

### Part A: the company brain (`identity-tenant-service`)

**A1. Company profile.** `org_profiles`, one row per org: `org_id` (PK), `description` (up to 2000 chars), `industry` (up to 100), `products text[]` (up to 30, each up to 80), `customers text[]` (up to 50, each up to 80), `updated_by`, `updated_at`. `GET /api/v1/org/profile`, `PUT /api/v1/org/profile`.

**A2. Vocabulary.** `org_glossary_terms`: `id`, `org_id`, `term` (1 to 80), `kind` (`product`, `codename`, `acronym`, `customer`, `person`, `other`), `definition` (up to 300, nullable), `aliases text[]` (up to 10, each up to 80), `created_at`, `updated_at`; unique per org on `lower(term)`; at most 500 per org. `GET /api/v1/org/glossary`, `POST /api/v1/org/glossary`, `PATCH /api/v1/org/glossary/{id}`, `DELETE /api/v1/org/glossary/{id}`, and `POST /api/v1/org/glossary/import` taking `{terms: [...]}` (up to 200, upsert by term) for pasting a list.

**A3. Who owns what.** `topic_owners`: `id`, `org_id`, `topic` (1 to 80), `owner_user_id` (FK `users`, ON DELETE CASCADE), `created_at`; unique per org on `lower(topic)`; at most 200. `GET`, `POST /api/v1/org/topic-owners`, `PATCH`, `DELETE /api/v1/org/topic-owners/{id}`.

**A4. Goals.** `org_goals`: `id`, `org_id`, `title` (1 to 200), `description` (up to 1000, nullable), `period` (up to 20, for example `2026-Q4`), `status` (`active`, `achieved`, `dropped`), `owner_user_id` (nullable), `position`, `created_at`, `updated_at`; at most 30 active. `GET`, `POST /api/v1/org/goals`, `PATCH`, `DELETE /api/v1/org/goals/{id}`.

**A5. Ground rules.** `tenant_settings.never_record_topics text[] NOT NULL DEFAULT '{}'` (up to 30, each up to 80). `GET /api/v1/org/ground-rules`, `PUT /api/v1/org/ground-rules` `{never_record_topics}`.

**A6. Topic watches.** `topic_watches`: `id`, `org_id`, `user_id`, `topic` (1 to 80), `created_at`; unique per user on `lower(topic)`; at most 20 per user. `GET /api/v1/users/me/topic-watches`, `POST /api/v1/users/me/topic-watches`, `DELETE /api/v1/users/me/topic-watches/{id}`. Self only.

**A7. Readiness.** `GET /api/v1/org/readiness` (owner/admin): `{score: 0..100, checks: [{key, label, done, detail}]}` with keys `profile` (description at least 40 chars), `vocabulary` (at least 10 terms), `owners` (at least 5), `goals` (at least 1 active), `ground_rules` (at least one never-record topic: an empty list cannot be told apart from one never set), `roles` (detail names how many members have no `role`; done when none). Equal weights.

**A8. Welcome.** `GET /api/v1/org/welcome` (any member): `{profile, goals (active), owners: [{topic, owner_user_id}], glossary (first 40 by term), team: [{user_id}]}` where `team` is the caller's team mates.

**A9. Org context for services.** gRPC `IdentityService.GetOrgContext(GetOrgContextRequest) returns (OrgContext)`, `org_id` in metadata like `GetUserContext`. `OrgContext`: `description`, `industry`, `repeated string products`, `repeated string customers`, `repeated GlossaryTerm glossary` (`term`, `kind`, `definition`, `repeated aliases`), `repeated TopicOwner topic_owners` (`topic`, `owner_user_id`), `repeated Goal goals` (`id`, `title`, `period`; active only), `repeated string never_record_topics`, `repeated TopicWatch topic_watches` (`user_id`, `topic`). Cached at `nexus:{org_id}:org_ctx` for 5 minutes, evicted by every write in Part A. Callers: `agent-runtime-service` and `transcription-service`.

### Part B: meetings (`core-api` unless named)

**B1. Meeting types.** `meeting_templates`: `id`, `org_id`, `name` (1 to 80), `description` (up to 300), `default_duration_minutes` (5 to 480), `agenda jsonb` (`[{title, duration_minutes}]`, up to 20), `confidential boolean NOT NULL DEFAULT false`, `created_by`, `created_at`, `updated_at`; at most 50 per org. `GET /api/v1/meeting-templates` (members), `POST`, `PATCH`, `DELETE /api/v1/meeting-templates/{id}` (owner/admin).

**B2. Confidential meetings.** `meetings.confidential boolean NOT NULL DEFAULT false` and `meetings.template_id uuid NULL`. `MeetingCreate` accepts both; a template's `confidential` is the default. A confidential meeting is never a source for topic watch, the `related_meeting` hop, or another meeting's carry-forward shown to non-participants.

**B3. Agenda goals.** `meeting_agenda_items.goal_id uuid NULL` (an `org_goals` id; no FK across services). The agenda `PUT` accepts it.

**B4. Drafted agenda.** `POST /api/v1/meetings/agenda-draft` `{previous_meeting_id?, template_id?}` returns `{items: [{title, owner_user_id, duration_minutes, goal_id, source}]}`, deterministic, at most 20, in this order: the template's items (`source: "template"`); the previous meeting's agenda items not `covered` (`"unfinished"`); its open confirmed action items as `Status: {title}` owned by the assignee (`"open_action"`). Nothing is saved.

**B5. Send your agent instead.** `meeting_participants.attendance text NOT NULL DEFAULT 'live'` CHECK in `live`, `agent`; `proxy_update text NULL` (up to 2000); `proxy_questions text[] NOT NULL DEFAULT '{}'` (up to 5, each up to 300). `PUT /api/v1/meetings/{id}/attendance` `{attendance, proxy_update?, proxy_questions?}`: self only, a participant, not after the meeting ended (409). Participants in meeting responses carry `attendance` and `proxy_update` (visible to all participants: it is meant for the room); `proxy_questions` only to the person and the host. The internal agenda route adds `proxies: [{user_id, proxy_update, proxy_questions}]`.

**B6. Promises.** `commitments`: `id`, `org_id`, `user_id`, `meeting_id`, `content` (up to 500), `due_hint` (up to 100, nullable), `status` (`open`, `done`, `dismissed`), `source_window_id`, `created_at`, `updated_at`, `done_at`; unique on `(meeting_id, user_id, source_window_id, md5(content))`. Written by `core-api`'s `agent-results` consumer from `PersonalListenerAgent` records' `payload_json.commitments: [{content, due_hint}]` (private to the record's `user_id`). `GET /api/v1/commitments?status=` (the caller's own), `PATCH /api/v1/commitments/{id}` `{status}` (owner only).

**B7. Status requests.** `action_status_requests`: `id`, `org_id`, `action_id`, `user_id` (the assignee), `meeting_id` (the upcoming meeting), `state` (`pending`, `answered`, `expired`), `status_text` (up to 500, nullable), `requested_at`, `answered_at`; unique on `(action_id, meeting_id)`. Created by the internal `POST /internal/meetings/{id}/status-requests` (caller `workflow-service`): one per open confirmed action item of the meeting's `previous_meeting_id`. `GET /api/v1/status-requests?state=pending` (the caller's), `POST /api/v1/status-requests/{id}/answer` `{status_text}`. Carry-forward `open_actions` gain `latest_status: {text, answered_at} | null`; the internal agenda route's `carry_forward.open_actions` strings append the latest status.

**B8. Screen reads (`bff`).**
- `GET /bff/meetings/{id}/brief` (a participant): `{meeting: {id, title, scheduled_start}, my_agenda_items, my_open_actions (from the previous meeting, with latest_status), saved_questions (the caller's saved suggestions), proxies: [{user_id, proxy_update}], previous_meeting: {id, title} | null}`.
- `GET /bff/meetings/{id}/health` (a participant, after it ended): `{duration_minutes, people_hours, agenda: {covered, total}, decisions, actions, talk_balance: [{user_id, share}] | null, async_suggestion: bool, trend: [{meeting_id, title, scheduled_start, coverage, decisions}]}`. `trend` walks `previous_meeting_id` back up to 6 meetings. `async_suggestion` is true when the meeting had no decisions, no actions and covered under half its agenda. `talk_balance` is null when no speaker timing is stored.
- `GET /bff/meetings/{id}/catch-up` (a participant whose attendance was `agent`, or anyone who missed it): `{agenda (statuses and agent notes), decisions, assigned_to_me, my_notes (the caller's surfaced worklogs), my_questions (the caller's proxy_questions)}`.

**B9. Timers (`workflow-service`).** For a meeting with `scheduled_start`, `MeetingLifecycleWorkflow` (or a sibling workflow it starts) sleeps on durable timers, not cron:
- start minus 60 minutes, only when `previous_meeting_id` is set: `POST /internal/meetings/{id}/status-requests`, then one `notifications` record per request (`in_app`, title "Quick status on {action}?", link `/status-requests`).
- start minus 10 minutes: one `notifications` record per live participant (title "Your brief for {title}", link `/meetings/{id}#brief`).
- 5 minutes after the meeting ends: one per participant whose attendance was `agent` (title "Your agent's catch-up from {title}", link `/meetings/{id}/catch-up`).
A meeting rescheduled or cancelled before a timer fires gets no notification for it.

### Part C: agents (`agent-runtime-service`)

**C1. Company context in prompts.** PersonalListener, Question, Action and `MeetingAgent` prompts (new versions) gain a company block: description, up to 60 glossary terms with definitions, active goals. Missing pieces render nothing.

**C2. Never record.** A window mainly about a `never_record_topics` entry produces no note, suggestion, candidate or agenda note: the gate scores it 0 and the prompt says so. This is a model instruction, not a guarantee, and the UI says so.

**C3. Promises.** PersonalListener (new version) also returns commitments the participant ("You") made in the window, in its existing record: `payload_json.commitments: [{content, due_hint}]` (up to 5). Only commitments by "You".

**C4. Suggested owner.** ActionAgent receives `topic_owners` and may add `suggested_owner_user_id` to a candidate when the action's topic matches; core-api stores it on the candidate if the candidate model has an assignee suggestion, otherwise passes it through to the UI.

**C5. Meeting agent.** Each agenda item's `goal_id` is in its prompt. New nudge type `proxy_update`: when an item owned by a participant whose attendance is `agent` becomes `in_progress`, the host is nudged with that person's `proxy_update` (once per item, no model call).

**C6. Proxied participants.** A participant with attendance `agent` gets a PersonalListener pass on every window even though they never speak, so their catch-up has notes.

**C7. ProactiveAgent, two new window checks** (meeting-scoped, same gating, budget and rate limits as `missing_stakeholder`):
- `already_decided`: a triage gate decides whether the window re-opens a settled question; if so, the org's confirmed decisions are searched (rag-service, as the alert's recipient); the search runs once per participant (cap 10), as that participant, so the ADR-0033 retrieval rule and ADR-0036 D9 hold: a participant who cannot see the earlier decision gets no alert. Each participant with a match at confidence 0.8 or more is alerted with `{alert_type: "already_decided", priority: 2, decision_id, decision_content, decided_at, meeting_id, meeting_title}`. At most once per (meeting, decision, participant).
- `topic_watch`: the window's topic is matched against the org's `topic_watches` (exact or alias match first, then a triage-tier check); each watcher not in the meeting, authorized by the ADR-0013 PDP for `meeting_context` on that meeting, gets `{alert_type: "topic_watch", priority: 3, topic, meeting_id, meeting_title, snippet (up to 200 chars)}`. Watchers are not in the meeting, so this arrives through the notification surfaces (`notification_log`), not the room. Never from a confidential meeting. At most once per (meeting, watcher, topic).

### Part D: realtime

**D1. Vocabulary in captions.** `transcription-service` fetches `GetOrgContext` when a meeting's stream opens and passes up to 100 key terms (terms and aliases, products, customers) to Deepgram Nova-3 as `keyterm`. A failed fetch opens the stream without them.

**D2. Gateway.** `realtime-gateway` accepts the new alert types (`already_decided`, `topic_watch`) and the nudge type `proxy_update` in its validators, unchanged otherwise.

### Part E: product surface (`mytheus-ui`)

- **Company brain** under Workspace admin: About, Vocabulary (with paste-a-list import), Who owns what, Goals, Meeting types, Ground rules, and a readiness meter; a first-run setup for the admin who creates a workspace.
- **New joiner brief** at `/welcome`, offered on the dashboard for 14 days after joining.
- **New meeting**: meeting type picker (fills length, agenda, confidential), confidential toggle, "Draft the agenda" for a follow-up, a goal on each agenda item.
- **Meeting page**: "Send my agent instead" with an update and questions; your brief; status of carried-forward actions; the health card after it ends; `/meetings/{id}/catch-up`.
- **Room**: proxied participants as agent tiles carrying their update; the `proxy_update` nudge; the `already_decided` card showing the original decision.
- **Promises** at `/promises` and on the dashboard; **status requests** answered inline from the dashboard and `/status-requests`; **topic watch** on the agent page; `topic_watch` and `already_decided` alerts in the notification surfaces.

## Consequences

- No Avro change: commitments ride in PersonalListener's `payload_json`, the two alerts in ProactiveAgent's, and every reminder is a `notifications` record with a link.
- Agents still never write Postgres; `core-api` persists commitments as it does notes.
- Everything is opt-in by data: an org with no brain, a meeting with no template or agenda, a person with no watches behaves as before.
- Not built now: speaker-time capture where none is stored, a separate org-wide document picker (key documents stay in Workspace admin, Data), cost in money (people-hours only), and an agent that speaks.
