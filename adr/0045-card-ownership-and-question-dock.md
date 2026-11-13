# ADR-0045: A card is settled by its owner, and a question comes up over the room

## Related documents

- [ADR-0031](0031-decision-and-action-candidates.md): decision and action candidates; confirm is the only commit. Amended here: who may confirm.
- [ADR-0034](0034-question-agent-suggestions.md): `QuestionAgent` suggestions, private to their owner. Unchanged; only where the room shows them.
- [ADR-0038](0038-agent-personas-profiles-agendas-and-meeting-agent.md) D6: the meeting's host or creator.
- [ADR-0040](0040-polish-program.md): the shared candidate ledger, the calm sidebar and the close-out ritual, all amended here.
- Record: [work-done-till-now/19-card-ownership-and-question-dock.md](../work-done-till-now/19-card-ownership-and-question-dock.md).

## Status

**Accepted** (2026-11-13).

## Context

Decision and action candidates are the meeting's shared ledger: every participant sees the same card. Until now every participant could also edit, confirm and dismiss every card, and in practice the host did it for everyone, including accepting tasks on other people's behalf. The owner asked for three things:

1. A task for a particular person is accepted, declined or changed by that person only.
2. Decisions stay with the host, together with any task that is the host's own.
3. A suggested question must get its owner's attention in the room, because it has to be asked while the topic is live; a decision or a task can wait until the end.

## Decision

1. **D1 Who settles a card.** "Settle" means edit, confirm or dismiss a `proposed` row. `core-api` enforces it, 403 `FORBIDDEN_ROLE` otherwise:
   - a task (`action_items`) whose `assignee_id` is a participant of the meeting: that person only. Not the host.
   - a decision, and a task with no owner in the meeting (no `assignee_id`, or an assignee who is not a participant): the meeting's host or creator (ADR-0038 D6), or the person the close-out handed it to (`review_owner_id`).
2. **D2 Reassigning.** The host or creator may still change who a proposed task is for (a `PATCH` that carries only `assignee_id`), so a task the agent gave to the wrong person is not stuck with them. They cannot change its text or settle it.
3. **D3 Visibility is unchanged.** Every participant still sees every surfaced card (the shared ledger). A card that is not yours shows who it waits for and offers only "Hide for me". It does not count toward "Needs you" and sorts after everything that does.
4. **D4 Close-out.** At the close-out each open task gets `review_owner_id` = the person it is for (when they are a participant); decisions and tasks with no owner go to the named owner as before. The review queue (`GET /bff/dashboard` `needs_review`) lists only the cards the caller may settle.
5. **D5 Announcing a dismissal.** `realtime-gateway` announces a dismissal to the room when the stored row is dismissed and the actor is the host, the creator, or the task's assignee.
6. **D6 The question dock.** In the web room a suggested question is docked at the bottom left of the room, over the stage, for its owner only, in Call and Notes mode and whatever the panel shows. It waits while its owner is speaking, draws the eye once on arrival (one ring in the priority colour; nothing with reduced motion), and stays until it is asked, saved for later or dismissed. It can be minimised to a pill that keeps the count. The sidebar still lists the same suggestion. No contract change: same events, same routes, same privacy (ADR-0034 D6).

## Consequences

- Nobody can accept work for someone else; the audit trail's `confirmed_by` on a task is its owner (or the host for an unowned one).
- No schema, topic or route is added or renamed. Existing routes answer 403 where they answered 200, which is a behaviour change for clients: `mytheus-ui` is updated; `mytheus-mobile` still offers confirm and dismiss to everyone and will show its generic error for a card that is not the caller's (open gap).
- A task whose owner never opens their review queue stays `proposed`. The host can reassign it (D2).
- The dock was placed bottom left because the panel is on the right and may be closed or on another tab; the owner left the exact design open.
