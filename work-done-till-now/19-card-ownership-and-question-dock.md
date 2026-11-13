# 19. A card is settled by its owner, and a question comes up over the room

Status: `DONE WITH GAPS` (built and verified locally on 2026-11-13; pushed to `main` in three repos and deployed by each repo's CI on 2026-11-13). Not a plan band: [ADR-0045](../adr/0045-card-ownership-and-question-dock.md).

## Why

The owner reported that in a meeting everything was being accepted by the host (and possibly others), and asked for:

1. A task for a particular person is accepted, declined or acted on by that person only.
2. Decisions are taken by the host, as are tasks assigned to the host.
3. A suggested question must attract its owner's attention in the room (bottom left was suggested, the exact design left open), because it has to be asked there and then, unlike a decision or a task, which can be settled at the end.

Before this, `core-api` let every participant of a meeting edit, confirm and dismiss every proposed decision and action; only "Dismiss for everyone" was limited to the host, and only in the room's UI and in `realtime-gateway`'s announcement check.

## What landed, per repo

### mytheus-backend (`fc30a11`)

- `services/core_api/routes/candidates.py`: `_require_card_owner`, called by the six settle routes (`PATCH`, `/confirm`, `/dismiss` for `/decisions/{id}` and `/actions/{id}`) after the status check, so an already settled card still answers as before (200 on a repeat, 409 otherwise). 403 `FORBIDDEN_ROLE` when the caller is not the card's owner (ADR-0045 D1). A `PATCH /actions/{id}` whose body is only `assignee_id` is also allowed for the host or creator (D2).
- `services/core_api/routes/room_closeout.py`: on close, an open task's `review_owner_id` is its assignee when that person is a participant, otherwise the named owner; decisions go to the named owner (D4).
- `services/bff/dashboard.py`: `needs_review` lists only the cards the caller may settle (D4).
- Tests: `tests/test_candidates.py` (three new tests: owner only; an absent owner falls to the host; the review owner may settle), `tests/test_room_stage.py` (close-out routing). `tests/test_chat_api.py` gains `make_creator`; the meeting helpers of `test_candidates`, `test_agent_learning` and `test_only_mytheus` use it, because their callers confirm decisions and so must steer the meeting.

### mytheus-realtime (`91eb37e`)

- `realtime-gateway` `src/roomEvents.ts` `verifyResolution`: a dismissal is announced when the stored row is dismissed and the actor is the host, the creator or the task's assignee (D5). Test in `test/room-events.test.ts`.

### mytheus-ui (`79a406f`)

- `src/lib/realtime/ownership.ts` (new, pure): `cardOwner(candidate, viewer)` mirrors the `core-api` rule from what the room already knows (your id, whether you steer, the participant ids). Test: `tests/ownership.test.ts`.
- `CandidateCard`: takes `canAct` and `waitingFor` instead of `canDismissForEveryone`. Your own task says Accept and Decline; a card that is not yours shows "Waiting for {name} to accept it" (or "the host to confirm it") and only "Hide for me".
- `AgentSidebar`: the ledger carries `owner(c)`; "Needs you" counts only your cards; `calm.ts` `needsYouQueue` takes `mine` and ranks a card that only waits for others last (`RANK.watching`).
- `WrapUp`: someone else's task is listed with "Stays with {name} to accept or decline." and no buttons; copy updated; the ended screen says tasks are in their owners' review queues.
- `src/app/actions/candidates.ts`: a 403 `FORBIDDEN_ROLE` gets its own message.
- `src/components/room/QuestionDock.tsx` (new) and its styles in `room-polish.css`, mounted by `MeetingRoom` once the meeting link is joined (D6).

## Deviations, and why

- **The question is not only bottom left; it is also still in the sidebar.** The owner left the design open. The dock is the attention surface; the sidebar keeps the full card (moment, sources). Settling in either place settles both, since both read the same room state.
- **The dock's primary button is "I asked it", not "Ask".** In the dock the point is to say it out loud now. It still copies the question, for someone who prefers chat.
- **A task with an owner who is not in the meeting falls to the host.** Otherwise nobody in the room could ever settle it (settling requires being a participant).
- **The host may reassign but not settle someone else's task.** Without reassigning, a task the agent gave to the wrong person could only be declined by that person.
- **No sound.** The room's motion rule is calm: one entrance and one ring. A chime in a call was judged more annoying than useful.

## Open gaps

- `mytheus-mobile` still offers confirm and dismiss on every card (`src/api/endpoints.ts`); a card that is not the caller's now fails with its generic error. It needs the same ownership rule in its UI.
- Not checked against a live meeting with two signed-in people; the room was verified in the component harness with the gateway faked (below).
- Pre-existing failures seen and left alone: `tests/test_bff_dashboard.py::test_a_slow_arm_degrades_and_does_not_500` (fails without this change too), two tests in `mytheus-ui` `tests/skeletons.test.ts` (Windows path checks, already recorded in file 17).

## Verification (2026-11-13, local)

mytheus-backend, against a fresh database on the compose Postgres (`BOOTSTRAP_DATABASE_URL=postgresql+asyncpg://supabase_admin:postgres@localhost:5443/mytheus_ownertest`; the local `mytheus` database could not be migrated to head because of an orphaned `oauth_tokens` row, unrelated):

- `pytest tests/test_candidates.py tests/test_card_moments.py tests/test_agent_learning.py tests/test_only_mytheus.py tests/test_bff_dashboard.py tests/test_room_stage.py`: 97 passed, 1 failed (the pre-existing dashboard test above).
- `ruff check services tests`, `black --check services tests`, `mypy` on the two route files: clean.
- The full suite was started and stopped after about fifteen minutes without finishing; its result is not claimed.

mytheus-realtime (`services/realtime-gateway`): `npx tsc --noEmit` clean; `npx vitest run test/room-events.test.ts`: 26 passed.

mytheus-ui:

- `npx tsc --noEmit`: clean. `npx next lint`: no warnings or errors.
- `npm test`: 549 pass, 2 fail (the pre-existing skeleton tests).
- `npx playwright test -c tests/e2e/playwright-ct.config.ts room-sidebar-a11y room-a11y`: 21 passed, including three new tests (host and guest ownership; the dock at the bottom left with the panel closed, minimise, ask, dismiss) and axe with the dock on screen in light and dark.
- A screenshot of the room as a guest (dock, own task with Accept, others' cards waiting) was looked at.

## Deployment (2026-11-13)

- mytheus-ui `79a406f` and mytheus-realtime `91eb37e`: CI green, `deploy` job succeeded.
- mytheus-backend: CI on `fc30a11` failed in `test` and `openapi`, so `deploy` was skipped. Both failures came from the earlier Linear commit (`b0cbf44`, ADR-0044), not from this work: two assertions in `tests/test_only_mytheus_services.py` did not expect the new `provider` field, and `design/openapi/core-api.yaml` and `integration-service.yaml` were stale. Fixed in `6e86b64` and the commit before it; CI run 43407662523 was green on every job (1655 tests) and `deploy` succeeded, including the `https://api.mytheus.cloud/healthz` check.
- `tests/test_bff_dashboard.py::test_a_slow_arm_degrades_and_does_not_500` fails only on the local Windows machine; it passes in CI.
