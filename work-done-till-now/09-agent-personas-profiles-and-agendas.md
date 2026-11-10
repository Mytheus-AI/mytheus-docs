# Agent personas, working profiles, agendas and the MeetingAgent

**Status:** DONE WITH GAPS (merged and deployed to production on 2026-11-10; gaps in section 5). Not a plan band: an owner request of 2026-11-10, decided in [ADR-0038](../adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md). Last verification 2026-11-10.

The owner asked for three things: every person's agent has a name, a face and a personality; onboarding asks each person what they work on so their agent knows what to listen for; and a meeting can follow up an earlier one, name a host, carry an agenda, and gets one meeting-wide agent that tracks it. ADR-0038 fixed the contracts first; seven builders then worked in parallel, one worktree per repo, all on branch `feat/agents-personas-meetings`.

## 1. What landed, per repo

### mytheus-docs ([PR #13](https://github.com/Mytheus-AI/mytheus-docs/pull/13))

- `adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md`, amended twice while building (see section 3).
- Canon: `_canon/02-data-model.md` (`users` columns, `meetings.previous_meeting_id`, `meeting_agenda_items`), `_canon/06-tech-decisions.md` (seventh agent `MeetingAgent`), `_canon/03-event-taxonomy.md` and `_canon/01-service-catalog.md` (`meeting_update`, the routes, the consumer and gateway rules).
- Reader docs 00 to 24 updated where they list the roster or the affected routes and screens; glossary, ADR index, traceability matrix and diagrams in the same change; `CLAUDE.md` and `system.md` frozen-name rule includes `MeetingAgent`.

### mytheus-events ([PR #4](https://github.com/Mytheus-AI/mytheus-events/pull/4))

- `schemas/agent-results/v1.avsc`: `AgentClass` gains `MeetingAgent`, `AgentResultKind` gains `meeting_update`, both appended; `payload_json` documents the `meeting_update` shape.
- Clients regenerated, curated fixtures, `tests/test_agent_results_meeting_update.py` (17 tests, including a payload validator), packages 0.1.0 to 0.2.0.

### mytheus-backend ([PR #18](https://github.com/Mytheus-AI/mytheus-backend/pull/18))

- Migration `0b91a00079a7` (on `e2b6c4a8f913`): the six `users` columns, `meetings.previous_meeting_id`, `meeting_agenda_items` with ENABLE + FORCE RLS and `tenant_isolation`, CHECK constraints mirroring the ADR limits.
- `identity-tenant-service`: `GET /users/me`, extended `PATCH /users/{user_id}` (agent fields self-only), `GET /users` items carry `agent_name` and `agent_avatar_seed`, `get_user_context` and proto `UserContext` fields 12 to 16.
- `core-api`: host, follow-up link and agenda on create and `PATCH`; `PUT /meetings/{id}/agenda`, `PATCH /meetings/{id}/agenda/{item_id}`, `GET /meetings/{id}/carry-forward`; internal `GET /internal/meetings/{id}/agenda`; the `agent-results` consumer applies `meeting_update`.
- OpenAPI specs re-emitted.

### mytheus-realtime ([PR #9](https://github.com/Mytheus-AI/mytheus-realtime/pull/9))

- `services/realtime-gateway/src/agentOutput.ts`, `server.ts`: a `meeting_update` record becomes one `agent-agenda` event per update for the whole meeting and an optional `agent-nudge` for the host's sockets only. New `test/agent-meeting.test.ts` (13 tests).

### mytheus-intelligence ([PR #9](https://github.com/Mytheus-AI/mytheus-intelligence/pull/9))

- `libs/shared/user_context.py`: the `UserContext` mirror gains fields 12 to 16.
- Prompts `personal_listener@v2`, `question_agent@v2` (profile adds listen for and ignore; a voice line from the personality and the person's instructions, which may change wording only), `meeting_agent@v1`; registry defaults advanced.
- Gate prompts `personal_listener_gate@v2` and `question_gate@v2` see the topic lines but never the voice: a listen-for topic outside the role is no longer cut by the pre-gate, and an ignored topic scores low. Thresholds unchanged; empty lists render exactly as v1. Two new golden cases in `tests/eval/personal_listener_golden.json`.
- `services/agent_runtime_service/agents/meeting_agent.py` and `agenda.py`: the `MeetingAgent`, one per meeting per window, only with an agenda; over time computed, off agenda and unaddressed from the model; one `meeting_update` record per window; nudge rate limits in Redis under `nexus:{org_id}:`.
- Tests: `tests/test_agent_personas.py` (20), `tests/test_meeting_agent.py` (33).

### mytheus-ui ([PR #41](https://github.com/Mytheus-AI/mytheus-ui/pull/41))

- Onboarding: `/onboarding/profile` ("About you") and `/onboarding/agent` ("Meet your agent") after the workspace step; a switch of workspace still returns straight to the app.
- Settings: Profile gains what you work on, listen for and ignore; new `/settings/agent` with a live preview.
- Dashboard card for anyone whose role is empty.
- New meeting dialog: follow-up to, host, agenda editor. Meeting page: host, follow-up link, agenda with statuses and agent notes, carried forward.
- Room: your agent heads the sidebar, agent badges on participant tiles, shared live agenda panel, host-only nudges, host can mark covered, skip or reopen.
- API types regenerated from the backend branch's specs.

## 2. Definition of Done audit

There is no plan band for this work, so the audit is against ADR-0038's decisions.

| Decision | State | Evidence |
| --- | --- | --- |
| D1, D2 profile and persona columns | MET on branch | migration `0b91a00079a7`; backend tests `tests/test_user_persona.py` |
| D3 profile API | MET on branch, with deviation 4.1 | backend tests; UI `tests/profile-types.test.ts` |
| D4 user context | MET on branch | backend context tests; intelligence mirror test against the proto |
| D5, D6 host, follow-up, agenda, carry-forward | MET on branch, with deviations 4.2 to 4.4 | `tests/test_meeting_agendas.py` |
| D7 MeetingAgent | MET on branch | `tests/test_meeting_agent.py`; not yet run against a real model |
| D8 events | MET on branch | `make ci`, `make check-compatibility` |
| D9 realtime | MET on branch | `test/agent-meeting.test.ts` |
| D10 persona in prompts | MET on branch | `tests/test_agent_personas.py`; paid live evals not run |
| D11 product surface | PARTIAL | built and unit-tested; not seen in a browser while signed in |

## 3. Decisions taken while building

- **One record per meeting per window.** The first ADR text had the `MeetingAgent` emit one record per agenda item plus one per nudge. The canonical `agent-results` idempotency key is `agentresult:{meeting_id}:{window_id}:{user_id}:{agent}` ([ADR-0032](../adr/0032-plan-8-decision-and-action-candidates.md) pattern), so every record after the first in a window would have been deduplicated away. Found by the docs pass; the ADR now has a single batched `meeting_update` kind and the key is unchanged.
- **Scheduled times on the internal agenda route.** The 70 percent rule for unaddressed items needs the meeting's length; the route now returns `scheduled_start` and `scheduled_end`. Without them the agent falls back to the sum of item minutes, and without either it never sends that nudge.
- **`GET /users/me`** has been in doc 19 since the PDF; it is first built here. The PDF's `PUT /users/:id/profile` stays documented and unbuilt.
- **No default on `AgentClass`.** Avro uses the reader's default, so a default added now cannot help any older reader, and it would silently relabel a future agent's output as an existing one.

## 4. Deviations, and why

1. The 30 focus-area and 1000-character limits are enforced by the UI only: tightening the API is a breaking change by the `oasdiff` gate.
2. Editing an agenda after the meeting has ended is a 409 with the existing `VERSION_CONFLICT` code: a new code would need a frozen error enum change.
3. Participants who are not the host are now stored as `attendee` rather than NULL, matching the calendar path.
4. Exactly one host is enforced in code (demote, then promote, in one transaction), not by a unique index, because existing rows may not satisfy one.
5. Carry-forward "open actions" are confirmed action items not yet done.
6. Prompt defaults for `personal_listener` and `question_agent` (and both gates) advanced to v2 before the paid live evals ran; the evals ran after the deploy and passed (section 6).
7. No golden-set eval for the `MeetingAgent` yet: there is no labelled data.

## 5. Open gaps

- Deployed 2026-11-10 in this order: `mytheus-backend` (CI deploy green; production `alembic_version` is `0b91a00079a7`, `meeting_agenda_items` exists with RLS enabled and forced, the six `users` columns and `meetings.previous_meeting_id` present), `mytheus-realtime` (images and deploy green), then the production Schema Registry through an SSH tunnel from a `mytheus-events` checkout matching main (`make check-registered` showed only `agent-results-value` differing; `make register` moved it to v3 with `MeetingAgent` and `meeting_update`; all 21 subjects then in sync), then `mytheus-intelligence` (the running `agent-runtime` has `meeting_agent.py`, prompt defaults `personal_listener`, `personal_listener_gate`, `question_agent`, `question_gate` at v2 and `meeting_agent` at v1, `CORE_URL` and the caller token set, zero restarts; the internal agenda route answers 404 for an unknown meeting, so the caller token is accepted), then `mytheus-ui` (CI deploy green; `/onboarding/profile` and `/onboarding/agent` serve the new steps).
- Signed-in browser check of every new screen in both themes and at 360 to 1600 px.
- A golden set for the `MeetingAgent` (there is no labelled data yet).
- The local developer Postgres could not run the migrations (existing data breaks migration `7c3e9a1f5b20`), so a fresh `mytheus-agents-pg` container now serves port 5443; the old container is stopped with its volume kept.

## 6. Verification

Run in each worktree on 2026-11-10:

- mytheus-events: `make ci`: 31 Python tests pass, 2 skipped (need a sibling docs checkout), 2 TypeScript tests pass; `make check-compatibility`: all subjects BACKWARD.
- mytheus-backend: `uv run ruff check .`, `uv run black --check .`, `uv run mypy .` clean; `uv run pytest`: 989 passed, 1 failed (`test_relay_real_bus_roundtrip`, the local Redpanda's partition memory limit); `oasdiff breaking` against main: no breaking change. Spectral not run locally.
- mytheus-realtime: `npm run lint`, `npm run typecheck`, `npm run build` pass; new tests 13 of 13; the full suite's 2 failures in `agent-output-kafka.test.ts` reproduce on a clean main.
- mytheus-intelligence: ruff, black, mypy clean; non-database `pytest`: 534 passed, 106 skipped (database, AGE, Qdrant, MinIO and paid evals), 1 failed (`tests/test_context_gap_bus_live.py`, fails the same on a clean main).
- mytheus-ui: `npx tsc --noEmit -p .` clean, `next lint` clean on changed files, `npm test` 140 of 140, `next build` succeeds.
- Paid live evals after the deploy, 2026-11-10, `EVAL_LIVE=1 uv run pytest -m eval tests/test_personal_listener_eval.py tests/test_question_eval.py` on main's code: 2 passed. `personal_listener_gate@v2` on 16 golden cases: precision 1.0, recall 1.0 at threshold 0.5; the listen-for topic outside the role scored 0.8 (kept) and the ignored topic inside the role 0.2 (cut). `question_agent@v2` with `question_gate@v2`: 7 surfaced, 7 useful, gap recall 1.0, noise 0.0, 21 calls.
