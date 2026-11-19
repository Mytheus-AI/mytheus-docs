# Slides 9 to 11: Results and Discussion

Paste-ready content for `Mytheus.pptx`. Every number below is taken from
`work-done-till-now/` and is a measured or verified fact, not an estimate.
Figures that are design projections rather than measurements are labelled as
such on the slide itself, which is what keeps the claim defensible under
questioning.

Framing note: this is Progress Presentation 2 and the agent runtime (Plan 7 to 9)
is not built yet, so these are implementation and verification results, not
end-user accuracy results. Slide 9 says that out loud in one line. A panel
punishes an overclaim far harder than an honest scope statement.

---

## SLIDE 9: Results and Discussion (1 of 3)

### Title: Results: System Built and Verified

**Sub-heading line:** Progress to date is a verified backend and AI infrastructure. Agent accuracy results follow in Phase 2.

- **Four of seven bands started** across **8 repositories**: scaffold, foundation and tenancy, meeting and transcription, and org context and AI infrastructure. The substrate is built; the agent tier is the next band and has not started.

- **Database foundation built and proven**: 31 tables created from an empty volume, all 7 services healthy; 11 identity and meeting tables carry `FORCE` row-level security with a `tenant_isolation` policy on every one.

- **Event backbone live**: 21 Redpanda topics plus 21 dead-letter queues, with 21 schema subjects gated `BACKWARD`-compatible against the live registry, not merely parsed.

- **Services running against real infrastructure**: `meeting-service`, `realtime-gateway`, `transcription-service`, `identity-tenant-service`, `embedding-service`, `reranker-service` and `ingestion-service`.

- **Test suite: 263 tests** in the backend, plus **28 of 28 AI infrastructure tests passing against real Postgres, Qdrant, Redpanda, Redis and a real BGE-small-en model.** No mocks on any load-bearing path.

- **Deployed**: CI/CD publishes container images and deploys to the production VPS, gated behind lint, test, migration, authorization and secret-scanning jobs.

**Speaker note:** Lead with the scope line. If asked why there are no accuracy numbers, the answer is that the agent tier is the next band and that measuring agents before the retrieval and tenancy substrate is verified would produce numbers nobody could trust.

---

## SLIDE 10: Results and Discussion (2 of 3)

### Title: Results: Measured Performance and Tenant Isolation

Suggested layout: the table as the centrepiece, the two bullets underneath.

| Component | Target | Measured | Status |
|---|---|---|---|
| Org chart query (`/bff/org/chart`) | Constant queries per request | **7.85 ms**, exactly 3 SQL statements regardless of tree depth | Met |
| Cross-tenant read (3 tables, org B reading org A) | 0 rows | **0 rows** on all three | Met |
| Reranker end to end (`reranker-service`) | Under 500 ms warm | **281 to 376 ms on CPU** | Under target, **not claimed** (CPU, not GPU) |
| Embedding batch of 32 (`embedding-service`) | Under 100 ms | **3536 ms on CPU** | **Blocked**: GPU packaging not built |
| Cost per meeting (capacity model) | Reduce naive path | **1,206 versus 8,640 LLM calls, an 86% reduction** | Design projection, not yet measured |

- **Isolation is proven end to end, not at the policy level.** Seeding under org A and querying under org B returns zero rows across `departments`, `teams` and `team_members`, which exercises `FORCE` RLS through the real application role.

- **Two SLOs are deliberately unclaimed.** Both inference services were measured honestly on CPU and marked blocked rather than reported as met, because the targets are GPU targets and a CPU measurement cannot establish them.

**Speaker note:** The unclaimed row is the strongest thing on this slide. Say plainly: the reranker is comfortably inside its target on CPU, and we still refuse to tick it, because the SLO is defined on hardware we have not measured. That is the difference between a number and a result.

---

## SLIDE 11: Results and Discussion (3 of 3)

### Title: Discussion

**A. Verification found four defects that testing had hidden**

- Four blocking defects were found by reading source rather than by running tests, and all four are now closed.
- The most serious, **D1**: the application connected to PostgreSQL as a role that **bypasses row-level security**, in every environment including production. Every isolation test that had been passing was proving isolation only for a role the application never used.
- Fixed with three least-privilege database roles, none holding `BYPASSRLS`, plus two CI assertions that keep it closed.
- **The lesson, and the one to say aloud: a passing test suite proved nothing here, because the test and the application did not share an identity.**

**B. Positioning against the literature**

- 25 papers were surveyed and independently verified against live APIs (129 of 130 candidates resolved).
- Sorting them on two axes, static corpus versus live stream, and one shared output versus per-recipient output, leaves **one quadrant empty**: streaming input with per-recipient output. That is the quadrant this project occupies.
- **Not one of the 25 reports a percentile latency for its own method.** The three that report absolute wall-clock time are all above our 1.5 second peer budget before any tail effect.
- Honest finding: role-conditioned per-participant relevance appears to be **anticipated by existing patents**, so our defensible claim is the measurement, not the mechanism.

**C. Limitations**

- Agent tier, and therefore all accuracy results, not yet built.
- GPU inference path unmeasured.
- No end-to-end run against live Clerk, LiveKit and Deepgram credentials yet.

**D. What follows**

- Build the agent runtime, then run the per-participant evaluation against a single-shared-summary baseline and a swapped-profile control.

**Speaker note:** Part A is the answer to "what did you actually learn". Part B is the answer to "how is this different from AutoGen and LaMP", which your own literature survey slides already set up. Keep part C on the slide; volunteering the limitation is what makes the rest credible.

---

## If you are told to cut to two slides

Merge 9 and 10: keep the six bullets from slide 9, drop the table to three rows
(org chart, cross-tenant, embedding blocked), and move the cost projection to
slide 11 part B. Never cut part A of slide 11; the D1 finding is the most
substantial engineering result the project has.

---

# SLIDE 12: Progress Overview

Six rows per column, one line each. Bold the lead-in, keep the rest short.
Both columns run in build order, so the last completed item and the first
pending item sit adjacent in the pipeline.

| Work Completed | To Be Completed |
|---|---|
| **Contract scaffold** across 8 repositories, 21 topics and 21 dead-letter queues | **`search-service`**, the last task of the current band |
| **Multi-tenant foundation**: 11 tables under `FORCE` RLS, least-privilege database roles | **Agent runtime**: the six LangGraph agents |
| **Identity and meetings**: Clerk auth, RBAC, audit log, meeting CRUD, consent gate | **Peer protocol** for `ContextGapAgent`, with deadlines and fallback |
| **Real-time and AI infrastructure**: transcription, embedding, reranker, ingestion | **RAG pipeline** and the AGE knowledge graph |
| **Deployed** to the production VPS through gated CI/CD | **GPU packaging**, so the inference targets can be claimed |
| **Verified**: 263 tests passing, four blocking security defects closed | **Evaluation harness** and the per-participant benchmark |

**Speaker note:** If asked why bands are marked done when items remain, the
honest answer is that each band is complete in construction and the remainder
is verification against live third-party credentials, tracked separately.

**Optional footer line:**

> Substrate built and verified. Remaining work is the agent tier and its evaluation.

---

# SLIDE 15: Conclusions

Five bullets, one line each.

- A **multi-tenant, event-driven backend** is built and verified across 8 repositories, with tenant isolation proven end to end, not just at the policy level.
- **Reading the source found four blocking defects that a passing test suite had hidden**, the worst being a database role that bypassed row-level security in every environment.
- Every measurement is reported against its target, and **two GPU targets are deliberately left unclaimed** because they were measured on CPU.
- **25 verified papers** place this work in the one quadrant the literature leaves empty: live streaming input with per-recipient output.
- Remaining work is the **agent tier and its evaluation**, tested against a single-shared-summary baseline and a swapped-profile control.

**Speaker note:** Land on bullet 2. It is the only claim on the slide that no
other team can make, because it came from reading code rather than running it.
