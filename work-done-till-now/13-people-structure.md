# People structure (ADR-0041)

**Status:** `DONE WITH GAPS` on branches `feat/people-structure` in `mytheus-backend`, `mytheus-intelligence`, `mytheus-ui` and `mytheus-docs`; committed, not pushed, not deployed. Last verified 2026-11-11.

The owner said the People page "is lacking features and flexibility": departments and teams could only be created, nothing said what they do, and everything sat in one narrow column. Asked for: a description on every department and team that agents can read; rename, describe and delete for departments; edit, move and delete for teams; adding, removing and moving people; a full-width layout; every state designed. Decision record: [ADR-0041](../adr/0041-people-structure.md).

## mytheus-backend

- Migration `c8d9e0f1a2b3` (`departments.description`, `teams.description`, nullable text, `CHECK char_length <= 1000`, column comments). Both tables already had `org_id`, RLS `ENABLE` + `FORCE` and `tenant_isolation`. Chained off `9c8d7e6f5a4b` (the `users.display_name` migration, itself off `e4f5a6b7c8d9`) so there is one head. The ORM models carry the column and check so autogenerate stays clean.
- `services/core_api/routes/org_hierarchy.py`: creates take `description`; new `PATCH/DELETE /api/v1/departments/{department_id}`, `PATCH/DELETE /api/v1/teams/{team_id}`, `POST /api/v1/teams/{team_id}/members/{user_id}/move`. Admin-only (`require_tier("admin")`), reads unchanged. Every referenced id goes through `is_visible` first (RI bypasses RLS), so another tenant's id is 404 in the path and in the body. Department parent cycles are 422 (recursive CTE). Delete semantics per ADR-0041 D3.
- Why two evictions: the request transaction belongs to `tenant_session` and commits after the handler returns, and `BackgroundTasks` run before that commit (`routes/clips.py`). So each write evicts inside the transaction (as the membership routes always did) and again from SQLAlchemy's `after_commit` event, which closes the window where a concurrent reader re-caches the pre-commit state.
- `GetOrgContext` (`brain/context.py`, `grpc_server.py`, `proto/identity.proto`): `OrgContext.departments` (10) and `.teams` (11), new messages `OrgDepartment`, `OrgTeam` (head count, no member ids). `GET /context/{user_id}` and `GET /bff/people/{user_id}` carry `description` on the team and department refs; `GET /bff/org/chart` carries it on each node. All response additions are optional fields.
- Audit actions added: `department.created/updated/deleted`, `team.created/updated/deleted`, `team.member_moved`.
- Tests: `tests/test_org_structure_api.py` (17 tests: descriptions and the 1000 limit in schema and database, patch semantics, cycles, department delete keeps teams and reparents children, team delete removes memberships only, move is one step and keeps the label, member tier 403, every cross-tenant path 404, user context and org chart carry descriptions, `GetOrgContext` carries the structure and every structure write evicts it, the after-commit eviction fires). Updated `test_company_brain.py` (proto field numbers) and `test_team_members_api.py` (the audit assertion filters to member actions now that team creation is audited too).

Verified (throwaway `supabase/postgres:15.8.1.060` on 5497 and `redis:7-alpine` on 6497; never the dev DB on 5443):

```
uv run ruff check . && uv run black --check . && uv run mypy .     # clean, 277 files
uv run python -m scripts.emit_openapi --check                      # specs match the code
BOOTSTRAP_DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5497/mytheus \
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5497/mytheus \
REDIS_URL=redis://localhost:6497 MYTHEUS_REQUIRE_DB=1 uv run pytest -q
docker run --rm -v <dir>:/w tufin/oasdiff breaking /w/base/<spec>.yaml /w/head/<spec>.yaml --fail-on ERR
                                                                   # core-api, bff, identity-tenant-service: exit 0
```

Full suite: 1581 passed, 2 failed. After rebasing onto `main` (with `9c8d7e6f5a4b`) the People, membership, company brain, autogenerate, user context and bff suites were rerun from a fresh database: 134 passed, one alembic head (`c8d9e0f1a2b3`). The two failures are not this change: `test_bff_dashboard.py::test_a_slow_arm_degrades_and_does_not_500` fails identically on a clean `origin/main` checkout, and `test_relay.py::test_relay_real_bus_roundtrip` needs a Redpanda with room for new partitions (the shared local broker refuses: "memory limit").

## mytheus-intelligence

- `libs/shared/org_context.py` mirrors the two new messages; `CompanyContext.departments` / `.teams` (`OrgDepartment`, `OrgTeam`); malformed entries are dropped; an older identity server sends neither field, which reads as no structure.
- `agents/company.py`: `render_structure` adds, inside the existing `<company_context>` block of `company_values` (the per-participant prompts: PersonalListener, Question, Action, Decision, ContextGap), "Where the person you work for sits" (their teams and department, descriptions in full) and "How the company is organised" (departments depth-first with their teams, descriptions clipped to 200 characters, at most 30 departments and 60 teams, a team with someone in this meeting marked "here"). `render_company(org)` without a context (minutes, bets, decision why) is unchanged, and an org with no structure renders exactly as before, so no prompt version changed and the system prompts are untouched.
- `runtime.py` puts `meeting_team_ids` (the union of the live roster's `team_ids`) beside `org` in every context.
- Why the brain and not an event: the structure changes rarely, every agent already fetches and caches `GetOrgContext` per window, and a person's own team ids already arrive through `GetUserContext`.
- Tests: `tests/test_org_structure.py` (parse, render, caps, cycles, no ids, unchanged output without structure, the runtime marking teams) and the offline golden set `tests/eval/org_structure_golden.json` (5 cases run through each of the 5 participant prompts). No paid eval was run.

Verified: `uv run ruff check . && uv run black --check . && uv run mypy .` clean; `uv run pytest -q`: 1337 passed, 73 skipped.

## mytheus-ui

- `app/(app)/people/page.tsx` renders `components/org/PeopleBoard.tsx`: each department a section (serif name, team count, "Part of", what it does, clamped with More, and an admin menu: Rename, Edit description, Move under another department, Add a team here, Delete department), then team cards in a grid of one, two (md) and three (xl) columns, then No department and Not on a team.
- `components/org/TeamCard.tsx`: head count and lead, the description clamped with More, a menu (Edit name and description, Move to another department, Delete team), a menu per person (Make lead or Remove as lead, Move to another team, Remove from team) and Add a person through the shared `MemberPicker` (public props only; the file is untouched).
- `components/org/OrgAdmin.tsx`: create and edit dialogs on the shared `Dialog`, department choices on `Dropdown`, "What does it do?" with a live count, helper copy saying agents read it to tune notes to each role and to suggest missing stakeholders, and worked example placeholders. Delete dialogs say where the teams go and focus "Keep" first. `components/org/ActionMenu.tsx`: the menu-button pattern; picking an item returns focus to the button before the dialog opens, and Enter is handled on keydown so it cannot reopen the menu.
- `app/actions/org.ts`: `updateDepartment`, `deleteDepartment`, `updateTeam`, `deleteTeam`, `addTeamMember`, `removeTeamMember`, `moveTeamMember`, `setTeamLead`, plus `description` on the creates; each revalidates `/people` and the client calls `router.refresh()`.
- States: empty (both create buttons), loading (`people/loading.tsx` redrawn to the new frame), failed read (`ErrorState` with Retry), members unavailable (a warning note), and read-only for members (a lock line, no menus or pickers).
- Tests: `tests/org-chart.test.ts` (descriptions, depth, `descendantsOf`), `tests/e2e/people-a11y.spec.tsx` with `PeopleHarness.tsx` and `people-fixture.ts` (axe in both themes for every state and the dialogs, keyboard menu paths, focus return, the 1000 limit, three columns at 1440, one at 375, no horizontal scroll at 360, 375, 768, 1024 and 1600), and the People skeleton pair now mounts the real `PeopleBoard`.

Verified: `npx tsc --noEmit -p .`, `npx next lint`, `npm test` (500 pass), `npm run test:a11y` (487 passed, 396 skipped by design; one unrelated skeleton test, `admin onboarding-agent`, failed under load and passed on rerun), and `npx -y node@20 ./node_modules/next/dist/bin/next build` with the publishable key set in that shell only.

## Gaps

- Not deployed. Deploy order: backend (migration), then intelligence (it reads the new proto fields and tolerates their absence), then UI.
- `_canon/02-data-model.md` still names identity-service as the owner of `teams` / `departments` while `core-api` writes them (pre-existing; ADR-0041 leaves it).
