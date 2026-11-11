# ADR-0041: People structure that agents can read

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md): `departments.description`, `teams.description`, the delete rules.
- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md): the new `core-api` routes, `OrgContext.departments` / `.teams`, the agents' company block.
- [ADR-0039](0039-company-brain-and-meeting-companions.md) (the company brain and `GetOrgContext`), [ADR-0040](0040-polish-program.md) (additive API evolution).

## Status

**Accepted** (2026-11-11).

## Context

The owner found the People page "lacking features and flexibility": departments and teams could be created but not renamed, described, moved or deleted; a person could be added or removed only through a whole-roster editor; and nothing said what a department or team does. Agents already read a person's team ids and department through `GetUserContext`, but knew only names, so a note could not be tuned to what the person's team owns, and nothing told an agent which team is responsible for a topic being discussed without them.

## Decision

1. **D1 What it does.** `departments` and `teams` gain a nullable `description`: the unit's purpose and responsibilities, written by an owner or admin, at most 1000 characters (request schema and a database `CHECK`). Blank is stored as `NULL`.
2. **D2 Edit, move, delete.** `core-api`, the tables' only writer, adds `PATCH /api/v1/departments/{id}` (name, description, parent; a parent may not be the department itself or one of its descendants: 422), `DELETE /api/v1/departments/{id}`, `PATCH /api/v1/teams/{id}` (name, description, department; `null` is "No department"), `DELETE /api/v1/teams/{id}` and `POST /api/v1/teams/{team_id}/members/{user_id}/move` (one transaction; the label comes along unless one is sent). Writes are owner/admin; reads stay any member. Every id of another tenant, in the path or the body, is 404.
3. **D3 Deleting never loses a team or a person.** A department delete soft-deletes the department, moves its teams to No department and its child departments to its own parent. A team delete soft-deletes the team and removes its memberships only. The UI confirms both and says exactly this.
4. **D4 Agents read it through the company brain, not an event.** `OrgContext` gains `departments` (`OrgDepartment`) and `teams` (`OrgTeam`, with a head count and no member ids), proto fields 10 and 11; `UserContext`'s REST shape carries the description on its team and department refs. No topic is added: the structure changes rarely and every reader already caches `GetOrgContext`. Every structure write evicts `nexus:{org_id}:org_ctx` and the affected members' `user_ctx`, inside the transaction and again after commit.
5. **D5 What a participant's agent sees.** In the `{company}` block of the per-participant prompts (PersonalListener, Question, Action, Decision, ContextGap), when the org has a structure: where the person sits (their teams and department, descriptions in full) and how the company is organised (every department and team with a description clipped to 200 characters; teams with someone in the meeting marked "here", from the live roster's team ids; at most 30 departments and 60 teams). No person is named and no id is rendered. Org-level writers (minutes, bets, decision why) and an org with no structure render exactly as before, so no prompt version changed; an offline golden set covers the block.
6. **D6 Audit.** New `audit_logs` actions: `department.created`, `department.updated`, `department.deleted`, `team.created`, `team.updated`, `team.deleted`, `team.member_moved`.

## Consequences

- An admin can reshape the org from the People page, and the agents pick it up within one in-process cache period (60 s by default) of the eviction.
- The structure adds up to about 1k tokens to a participant's user message for a mid-size org; it sits after the cached system prompt, so caching is unchanged.
- The `teams` / `departments` rows in canon still name identity-service as owner while `core-api` is the writer (recorded since Plan 5-6); this ADR does not change that.
