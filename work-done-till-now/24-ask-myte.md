# 24. Ask Myte

Not a plan band ([ADR-0047](../adr/0047-ask-myte.md), [ADR-0049](../adr/0049-ask-myte-v2.md)). Status: **DONE WITH GAPS** (v1 live in production since 2026-11-14; v2 merged and deploying 2026-11-14; the open items are listed under Open gaps).

One place to ask your Myte anything across Mytheus: the `/ask` page, a sheet on every app page (`Cmd+J` / `Ctrl+J`) and entry points on the sidebar, Your Myte, Home, meeting and decision pages, the meeting room and the command palette. Answers stream with numbered citations, in the person's own Myte character (its plush, name, personality, instructions and voice). Every write is a proposal the person confirms. v2 adds asking a teammate's Myte, Myte reaching out first, seven more actions, memory, routines, voice mode and attached text files, plus the reliability items below.

## What landed, per repo

### mytheus-docs

- [ADR-0047](../adr/0047-ask-myte.md) (PR #24) and [ADR-0049](../adr/0049-ask-myte-v2.md) (PR #26); canon rows `myte_threads`, `myte_messages` (v1), `myte_peer_questions`, `myte_memories`, `myte_routines` and `payload.proposal_states` (v2); service catalog entries for core-api, bff and rag-service; the Glossary row "Ask Myte".

### mytheus-backend

- v1 (PR #32): migration `d927f2c093f5` (`myte_threads`, `myte_messages`, RLS ENABLE + FORCE, `tenant_isolation`, `grant_tenant_table`); `services/core_api/routes/myte.py` with the owner-only `/api/v1/myte/threads*` and `/api/v1/myte/messages/{id}` routes; gateway prefix `/api/v1/myte` to core-api; four bff MCP read tools in `services/bff/myte_tools.py` (`list_my_schedule`, `find_people`, `list_my_inbox`, `get_my_myte`), read in-process under the caller's workspace because personal access tokens are refused on the routes they would otherwise call; erasure coverage.
- Fix (PR #31, same day, found while testing): an empty pre-opened LiveKit room had marked meetings live and then ended before they began. Rooms now open at the first join, `live` follows a person joining, and only a `live` meeting can end; migration `3a87a7db84a0` reopened the meetings already hit.
- v2 (PR #34): migration `b6d1e4a7c902` (`myte_peer_questions`, `myte_memories`, `myte_routines`); peer question routes with asker and recipient visibility, notifications through the `notifications` outbox (`mytepeer:{id}`, `mytepeer:{id}:answered|declined`), `DELETE` for the asker to withdraw; memories (dedupe ignoring case, max 100); routines with the pure scheduler `services/core_api/myte_schedule.py` (timezones, DST, skipped and repeated hours) and `GET /routines/due`, `POST /routines/{id}/ran`; `PATCH /api/v1/myte/messages/{id}` accepts `proposal_states`; `list_my_inbox` returns pending peer questions.

### mytheus-intelligence

- v1 (PR #19): `LiteLLMRouter.converse()` (multi-turn, tool calling, streaming, with the budget gate, redaction, caching and per-round metering of `complete()`); `services/rag_service/myte/` and `POST /api/rag/myte` (SSE) calling bff `POST /mcp` with the asker's own bearer; pseudo-tools `propose_action` and `show_card`; a local `search_help` over `help.json`; prompts `myte_answer@v1`, `myte_followups@v1`; an 80-case golden set with an offline harness.
- Fixes (PRs #20, #21): production answers failed after the first tool round because the router sent `thinking: disabled` mid-turn, which Sonnet 5.5 refuses. Ask Myte now runs with no thinking at all (owner's decision, token cost): `effort=None` sends `thinking: {type: "between_tools"}`, `max_tokens` 2048. A person scope now carries its label, so "his work" with the chip "Samar Rihan" is answered instead of "who is his?". A stray `.venv` symlink committed by mistake was removed and ignored.
- v2 (PR #22): per-person `users.id` from identity `GET /users/me` (cached `nexus:{org_id}:myte:uid:{clerk_sub}`), metering with it and a daily cap `MYTE_USER_DAILY_UNITS` (300000) in `nexus:{org_id}:myte:units:{user_id}:{date}`; histograms `myte_first_step_seconds`, `myte_first_text_seconds`, `myte_answer_seconds`; seven new proposal kinds with validation; memories and attachments in the request; `mode: "peer_answer"` with `myte_peer_answer@v1`; `myte_answer@v2` as the default prompt; a 240-character text holdback that drops tool-round preambles.

### mytheus-ui

- v1 (PR #79): the stream route `src/app/api/ask-myte/route.ts` (fresh Clerk token, persistence after `done`, `saved` event); `src/lib/ask-myte/*` (SSE parser, reducer, citations, cards, starters, page context); `/ask`, the sheet, every entry point; tests and the `ask-myte-a11y` component spec.
- Fixes (PRs #80, #81): the latest answer is signed by a live 56 px Myte and the header Myte is 48 px (the owner saw only a tiny header Myte move); the page sits in its own rounded panel; the scope chip's label is sent upstream.
- v2 (PR #82): stored proposal states; cards and edit forms for every new action, confirmed through the existing server actions; the asker's waiting card and the recipient's review sheet in Inbox; nudges from the floating Myte and the Home card "Today from Myte"; Your Myte cards for memory and routines; due routines run on the next visit; voice mode (lazy-loaded, 200 px live plush driven by the mic level, auto-send after 1.2 s of silence); attachments (`.txt`, `.md`, `.csv`, paste); `ask-myte-v2-a11y` spec (21 tests).

### mytheus-platform

- PR #21: the KEK and blind-index key files are mounted into `integration-service`, `integration-worker` and `integration-consumer` (Google Calendar connect had failed with `KeyConfigurationError`; ADR-0026 amended in docs PR #23).
- PR #23: Prometheus scrapes `rag-service:8107` for the Ask Myte latency histograms (applies when observability is next deployed).

## Definition of Done audit (ADR-0047 and ADR-0049)

- Whole-platform answers with citations, in character: MET (production answers verified by the owner on 2026-11-14).
- Writes only through confirmed proposals: MET.
- Threads private to their owner, admins included: MET (route filters plus tests in CI).
- Live eval gate of 85%: MET, 92 of 95 (97%) on the second full live run; the first run scored 62% and drove the prompt and holdback fixes.
- Stored proposal states: MET.
- Per-person metering and daily cap: MET in code; not yet observed in production.
- Latency targets (first step under 1 s, first text under 2.5 s): PARTIAL, measured in code; not observed until Prometheus scrapes rag-service.
- Peer Mytes, proactive nudges, actions, memory, routines, voice, attachments: MET in code and component tests; not yet exercised end to end on production.

## Deviations, and why

- The loop lives in rag-service, not agent-runtime-service: rag-service already verifies Clerk sessions, owns "ask" and is routed by the gateway; agent-runtime has no HTTP surface in production.
- Tools are reached through bff `POST /mcp` as the asker, so Myte can never see more than the person can, and every tool also serves Claude.
- No thinking: thinking tokens cost too much for chat (owner). `between_tools` is Sonnet 5.5's "off" in a tool conversation.
- Peer answers are drafted in the recipient's session and reviewed before sending, and routines run on the next visit: Myte has no session of its own, and running as a person while they are signed out was not built.
- Slack posts copy the text and open Slack: there is no public Slack post route.
- The new bff tools read in-process rather than over HTTP, because personal access tokens are refused on `/users`, `/notifications` and `/api/v1/listening`.

## Open gaps

- Observe per-person caps and latency in production once observability is redeployed with the new scrape target.
- End-to-end checks on production of peer questions, routines, voice mode and the seven actions (component-tested only).
- Three live-eval cases short of 100% (`myte-07` cites nothing for a self-question, `help-07` answers rather than abstains because the export now exists, `actions-03` needs its fixture updated).

## Verification

- mytheus-backend: CI on PRs #32 and #34 (lint, mypy, the full pytest suite with testcontainers, migrations up/down/up, oasdiff, Spectral, authz, security, docker), all green; production probes of `/api/v1/myte/threads`, `/peer-questions`, `/memories`, `/routines/due` answer 401 without a session.
- mytheus-intelligence: `uv run pytest` 1529 passed; `ruff`, `black --check`, `mypy` clean; `EVAL_LIVE=1` 92/95. A live two-round call against Sonnet 5.5 with `between_tools` returned a tool call then a cited answer (625 and 664 units).
- mytheus-ui: `npx tsc --noEmit`, `npx next lint`, `npm test` (635 passed), `ask-myte-a11y` (11) and `ask-myte-v2-a11y` (21) component specs, `next build` compiles; screenshots of every surface in both themes and at phone width reviewed.
- Production: the owner asked real questions on `/ask` and received answers after PRs #20 and #21.
