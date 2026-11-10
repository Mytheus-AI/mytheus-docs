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
| `02-plan-1-2-foundation-and-tenancy.md` | Plan 1-2 (`plan/plan-1-2-foundation-and-tenancy.md`): P1 complete (canon amendments plus the design pack); P2 backend complete, with all seven blocking defects closed, the `organizations` RLS decision, every service's routes, and CI in two repos. The two client repos are what remains. |
| `03-plan-3-4-meeting-and-transcription.md` | Plan 3-4 (`plan/plan-3-4-meeting-and-transcription.md`): `meeting-service`, `core-api` chat/speakers/consent-gate, `realtime-gateway` and `transcription-service`, all against real infrastructure; the `mytheus-ui` LiveKit and live-caption surfaces added later on the maintainer's go. Carries the Definition-of-Done audit and the largest live gap list in the folder, including everything blocked on LiveKit/Deepgram/S3 credentials. |
| `04-plan-5-6-org-context-and-ai-infra.md` | Plan 5-6 (`plan/plan-5-6-org-context-and-ai-infra.md`): `identity-tenant-service` org context, the role-to-topics taxonomy, the Redis-cached `get_user_context()`, the seeder, `bff` aggregation, and the AI tier's `embedding`/`reranker`/`ingestion`/`search` services. Both `DECISION-REQUIRED` callouts were closed before any task wrote code. |
| `05-deployment-and-cicd.md` | The deployment substrate every band depends on: image publishing, the CI/CD deploy path, the Dokploy wiring, and the nine defects found in it. Not a plan band. |
| `07-plan-7-9-agents-and-monetization.md` | Plan 7-9 (`plan/plan-7-9-agents-and-monetization.md`): the ADR-0029 and ADR-0030 decisions, then P7 one work item at a time. Items 1 to 4 are built and verified on the local stack with a faked model; the real-model check, a browser check, mobile and deployment remain. Then P8 (both parts), the P9 RAG track (section 11, with its ablation) and the P9 `QuestionAgent` track (section 12). |
| `06-frontend-and-ui.md` | Which frontend repo is the product and which is a draft, what `mytheus-ui` already has, the API contract it expects, the open frontend items, and the 2026-11-07 custom auth flows, missing screens and dev-server speed fix. |
| `08-plan-10-13-collab-integrations-ga.md` | Plan 10-13 (`plan/plan-10-13-collab-integrations-ga.md`): P10 `ContextGapAgent`, the A2A PEP/PDP and `agent_authz_scopes`, `agent_collaborations` and its log UI, the real `MeetingLifecycleWorkflow` activities, the security review and its fixes (section 1). |

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
