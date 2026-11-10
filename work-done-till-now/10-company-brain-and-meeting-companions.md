# The company brain and ten features built on it

**Status:** DONE WITH GAPS (merged and deployed to production on 2026-11-11; gaps in section 5). Not a plan band: an owner request of 2026-11-10, decided in [ADR-0039](../adr/0039-company-brain-and-meeting-companions.md). Last verification 2026-11-11.

The owner asked for an organisation setup that gives every agent the company's context, plus ten features, built in parallel with strong UI and UX. ADR-0039 fixed the contracts; eight builders then worked at once (docs, two backend, intelligence, realtime, three UI), each owning disjoint files, and one integration pass joined them.

## 1. What landed, per repo

### mytheus-docs

- `adr/0039-company-brain-and-meeting-companions.md`, amended once while building (section 3).
- Canon: `_canon/02-data-model.md` (eight new tables and the new columns), `_canon/01-service-catalog.md` (routes, `GetOrgContext`, workflow timers, agent behaviours, transcription key terms), `_canon/03-event-taxonomy.md` (payload additions; no Avro change).
- Reader docs 03, 04, 07, 08, 10, 11, 12, 19, 20; glossary, ADR index, traceability matrix, diagrams 02, 03, 04, 06, 12 and the C4 container view.

### mytheus-backend (one branch, both halves merged)

- Migrations `c1a2b3d4e5f6` (company brain, identity) then `d2b3c4d5e6f7` (meetings, core-api). They were built in parallel off `0b91a00079a7` and first joined by an empty merge revision; CI's upgrade, downgrade -1, upgrade check cannot step down from a merge ("Ambiguous walk"), and neither was deployed, so the core-api revision was re-parented onto the identity one and the merge revision removed.
- `identity-tenant-service`: `org_profiles`, `org_glossary_terms` (with an upserting import), `topic_owners`, `org_goals`, `topic_watches`, `tenant_settings.never_record_topics`; `/api/v1/org/*` and `/api/v1/users/me/topic-watches`; readiness and welcome reads; gRPC `GetOrgContext`, cached at `nexus:{org_id}:org_ctx` for five minutes and evicted by every write.
- `core-api`: `meeting_templates`, `meetings.confidential` and `template_id`, `meeting_agenda_items.goal_id`, attendance and proxy fields on `meeting_participants`, `commitments`, `action_status_requests`; the drafted agenda, attendance, promises and status request routes; the confidential rules in carry-forward, the drafted agenda, the brief and the `related_meeting` hop; the consumer writes commitments and stores a suggested owner as a proposed action's assignee.
- `bff`: the brief, health and catch-up screens.
- `workflow-service`: `MeetingRemindersWorkflow` with durable timers at start minus 60 and minus 10 minutes and 5 minutes after the end, re-reading the schedule when a timer fires.
- `notification-service`: accepts the two new alert types.
- `api-gateway`: routes `/api/v1/org`, `/api/v1/users`, `/api/v1/meeting-templates`, `/api/v1/commitments`, `/api/v1/status-requests`. The last three were missing from both builders' work and were added at integration; without them those routes were unreachable from outside.

### mytheus-intelligence

- `libs/shared/org_context.py`: the `GetOrgContext` mirror and a cached client.
- New prompt versions: PersonalListener (note v3, gate v3), Question (v3, gate v3), Action (v2, gate v2), MeetingAgent v2, and three ProactiveAgent prompts. The company block sits in the user message so the cached system prefix stays shared; with no company brain each new version renders exactly its predecessor's user message.
- Commitments by "You", a suggested owner from topic owners, goals in the meeting agent, the `proxy_update` nudge, a listening pass for people who sent their agent, and ProactiveAgent `already_decided` (searched once per participant, as that participant) and `topic_watch` (PDP-gated, never from confidential meetings).

### mytheus-realtime

- `transcription-service`: the org's vocabulary sent to Deepgram Nova-3 as `keyterm`, at most 100 terms within the 500-token limit, cached five minutes per org, skipped quietly on any failure.
- `realtime-gateway`: accepts `already_decided`, `topic_watch` and the `proxy_update` nudge.

### mytheus-platform

- `deploy/compose.prod.yml`: `IDENTITY_GRPC_ADDR` for transcription-service.

### mytheus-ui

- Company brain under Workspace admin (About, Vocabulary with paste-a-list import, Who owns what, Goals, Meeting types, Ground rules) with a readiness meter; a first-run setup for the admin of a new workspace; `/welcome`.
- New meeting (meeting types, confidential, drafted agenda, a goal per item); meeting page (send my agent instead, your brief, latest statuses, the health card); `/meetings/{id}/catch-up`; the call screen (agent tiles, the proxy update nudge, the already-decided card).
- `/promises`, `/status-requests`, `/notifications`, topic watch on the agent page; dashboard cards (welcome, readiness, brief, status requests, promises); a Company brain tab; nav entries.

## 2. Definition of Done audit (against ADR-0039)

| Part | State | Evidence |
| --- | --- | --- |
| A, company brain | MET on branch | `tests/test_company_brain.py` (71) in mytheus-backend |
| B, meetings | MET on branch | `tests/test_company_brain_meetings.py`, `test_bff_meeting_screens.py`, `test_meeting_reminders_workflow.py` |
| C, agents | MET on branch, not run against a real model | `tests/test_company_brain.py` (50) in mytheus-intelligence; paid evals not run |
| D, realtime | MET on branch | `tests/test_org_context.py` (15), `test/agent-brain-types.test.ts` (12) |
| E, product surface | PARTIAL | built and unit-tested; key screens seen with sample data; not seen signed in |

## 3. Decisions taken while building

- **`already_decided` is searched once per participant, as that participant**, so a participant who cannot see the earlier decision gets no alert (the retrieval rule of ADR-0033 and ADR-0036 D9). Priorities: `already_decided` 2, `topic_watch` 3.
- **The readiness ground-rules check** is done when there is at least one never-record topic: an empty list cannot be told apart from one never set.
- **The reminder workflow** is started by the `meeting-events` consumer on `created` and signalled on `updated` and `ended`, because the lifecycle workflow ignores updates, and reschedules and cancellations arrive as updates.
- **An imported term that already exists keeps its kind**: the import updates definitions and spellings; kinds are changed in the table. Found in the browser check.

## 4. Deviations, and why

1. `DecisionAgent` does not yet apply the never-record rule: it was outside the agents builder's scope. Decision candidates on a never-record topic are still proposed (they still need a person to confirm).
2. The health card's length is the scheduled length: no actual start or end time is stored.
3. Members cannot open the company brain pages: Workspace admin returns 404 to non-admins. `/welcome` is their read view.
4. Meeting types and status requests have their own types in two UI modules; they are compatible and can be consolidated on the generated client.

## 5. Open gaps

- Deployed 2026-11-11: backend (production `alembic_version` `d2b3c4d5e6f7`; new routes answer 401 without a token), realtime and platform (transcription-service runs `org_context.py` with `IDENTITY_GRPC_ADDR=identity-grpc:50051`), intelligence (agent-runtime on PersonalListener v3, Question v3, Action v2, MeetingAgent v2 and the two ProactiveAgent prompts, no restarts), then the UI. Backend, realtime and platform deploys ran at the same moment and the realtime one failed removing a container another run had replaced; production was checked consistent and the realtime deploy re-run on its own succeeded. Deploys from different repos should be run one at a time.
- A signed-in walk-through of every new screen in both themes.
- The paid live evals for the new prompt versions.
- `DecisionAgent` and the never-record rule.

## 6. Verification

- mytheus-backend (merged branch, a fresh `supabase/postgres:15.8.1.060` on a throwaway port): `alembic heads` shows a single head `d2b3c4d5e6f7`; upgrade, downgrade -1, upgrade passes; ruff, black, mypy (183 files), `scripts.emit_openapi --check` and `scripts/check_authz.py` clean (the authz gate needed a router-level `Depends(verify_token)` on the company brain routes, found by CI); `uv run pytest -x --ignore=tests/test_relay.py`: 1090 passed. `test_relay.py` was left out because the shared local Redpanda refuses new partitions (memory limit); each builder saw only that failure in a full run.
- mytheus-intelligence: ruff, black, mypy clean; non-database pytest 585 passed, 24 skipped, 1 failed (`test_context_gap_bus_live`, needs a live Redpanda, fails the same on main).
- mytheus-realtime: transcription-service 131 passed, 1 skipped; realtime-gateway lint, typecheck and build clean, 104 of 105 (the Kafka test that fails on main too).
- mytheus-ui: `tsc` clean, `next lint` clean on changed files, `npm test` 202 of 202, `next build` succeeds with every new route. The readiness meter, vocabulary table with import, promises page and health card were looked at in a browser at 1440 px through a temporary, uncommitted preview route with sample data.
- mytheus-platform: `docker compose --profile realtime config` renders `IDENTITY_GRPC_ADDR=identity-grpc:50051` for transcription-service.
