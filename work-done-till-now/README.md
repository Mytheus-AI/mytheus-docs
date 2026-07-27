# work-done-till-now

The single home for the record of what has actually been **built** across the Mytheus (NEXUS) repos.

`plan/` says what to build and in what order. This folder says what is built, what is verified, and what is still open. Nothing else in the program records implementation progress: per-repo `PREVIOUS.md`, `PROGRESS.md`, and session logs are superseded by this folder.

## The single-home rule

After any session, feature, fix, or band step, in any repo, the write-up lands **here** and nowhere else.

- One file per band, numbered in band order (`01-plan-0-...`, `02-plan-1-2-...`).
- A band file covers **every repo** touched by that band, in one place, so a reader gets the whole program state without cloning eight repos.
- Do not create a new per-repo progress log. If a repo already has one (`mytheus-backend/PREVIOUS.md`, `mytheus-intelligence/PREVIOUS.md`, `mytheus-realtime/PREVIOUS.md`, `mytheus-ui/PROGRESS.md`), it is historical. Leave it in place, do not extend it, and fold anything new into the band file here.

This exists because progress was previously scattered across four repos in three different file names and two formats, so no single artifact answered "where is the program". That question now has exactly one answer.

## File index

| File | Covers |
|---|---|
| `00-status-overview.md` | Program status at a glance: the band-by-repo matrix, the repo inventory, and the live gap list. |
| `01-plan-0-scaffold-and-contracts.md` | Plan 0 (`plan/plan-0-scaffold-and-contracts.md`): what landed per repo, the Definition-of-Done audit, and the four open gaps. |
| `02-plan-1-2-foundation-and-tenancy.md` | Plan 1-2 (`plan/plan-1-2-foundation-and-tenancy.md`): P1 complete (canon amendments plus the design pack), P2 not started, the `organizations` RLS decision, and fifteen open gaps. |

## How to write a band file

Mirror the band's own structure so the plan and the record line up section for section.

1. **Status line.** Band, plan file, one of `NOT STARTED` / `IN PROGRESS` / `DONE WITH GAPS` / `DONE`, and the date of the last verification.
2. **What landed, per repo.** One subsection per repo. Concrete artifacts (files, tables, topics, routes), not adjectives.
3. **Definition of Done audit.** Copy the band's section 10 checklist verbatim and mark each line `MET`, `PARTIAL`, or `NOT MET` with the evidence or the reason.
4. **Deviations from the plan, and why.** Every place the code differs from the band text, with the rationale. A deviation with no recorded reason is a bug, not a decision.
5. **Open gaps.** What is left, who or what it blocks, and which band picks it up.
6. **Verification.** The exact commands run and their result. Distinguish what was observed from what a prior session claimed.

## Conventions

These follow the repo conventions in [CLAUDE.md](../CLAUDE.md) and apply here without exception.

- Markdown; plain-text headers; no emojis; no em-dashes or en-dashes (use a hyphen, a colon, or a restructured sentence).
- Frozen identifiers verbatim and in code spans: `org_id`, `app.current_org`, `nexus:{org_id}:`, `transcript_chunks`, `nexus_knowledge`, `get_user_context`.
- Canonical **repo** names in prose (`mytheus-backend`, `mytheus-ui`, ...), never the on-disk folder names. See the repo-name mapping in [00-status-overview.md](00-status-overview.md).
- Relative links between docs, for example [plan/plan-0-scaffold-and-contracts.md](../plan/plan-0-scaffold-and-contracts.md).
- This folder **records**; it never **defines**. It introduces no name, number, or decision. Canon (`_canon/`) remains the only source of truth, and a decision still needs an ADR. If a session changed a name or a decision, the canon and ADR edits land first and this file cites them.
- Claims are evidence-backed. "Green" means a command was run and its output seen; anything inherited from an earlier session is marked as reported, not verified.
