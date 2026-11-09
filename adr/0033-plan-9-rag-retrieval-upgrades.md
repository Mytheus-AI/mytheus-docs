# ADR-0033: Plan 9 RAG entry decisions: participant-scoped retrieval, filters from expansion, decision citations, small-to-big context, and a grounded answer

Plan 7-9 phase P9 builds `rag-service`. A review of the locked design against current retrieval practice, made at the P9 entry gate on 2026-11-09, found two defects and five cheap upgrades. All of them fit inside the frozen five-stage shape `expand -> retrieve -> rerank -> assemble -> cite`; none adds a service, table, topic, collection or Redis key. This ADR records them.

## Related documents

- [_canon/06-tech-decisions.md](../_canon/06-tech-decisions.md) section 2.9 - the Citations and RAG API rows are amended, and a Retrieval access row is added, in the same change.
- [09-rag-and-inference-serving.md](../09-rag-and-inference-serving.md) - the pipeline this ADR refines.
- [ADR-0031](0031-plan-8-graph-store-decision-supersedes-and-mark-done.md) - `decisions__{org_id}` is embedded on confirm and carries `meeting_id`.
- [ADR-0032](0032-plan-8-decision-and-action-candidates.md) - the participant rule (403 `NOT_A_PARTICIPANT`) that D1 extends to retrieval.
- [plan/plan-7-9-agents-and-monetization.md](../plan/plan-7-9-agents-and-monetization.md) section 3.9.

---

## Status

**Accepted** (2026-11-09).

## Context

1. **Retrieval is isolated by tenant only.** Doc 09 section 7 filters every arm by `org_id`. Inside one org, any member could retrieve the transcript of a meeting they were not in. Every other read of meeting content in `core-api` (chat, candidates) is participant-only. The Ask panel would be the one place that rule does not hold.
2. **The graph arm cannot be cited.** The graph arm returns `Decision`, `Action` and `Meeting` nodes, but the frozen citation contract knows only `transcript` and `document` passages. Under the rule "a passage that cannot bind a citation is dropped", every graph result would be dropped and the arm would contribute nothing.
3. **Meeting questions are mostly about when and who.** "What did Priya say about pricing last week" carries a date range and a person. Vectors and FTS match neither reliably.
4. **Transcript chunks are short.** Speaker turns merged and split at 600/60 characters are about 150 tokens. A hit such as "yes, let's do that" is useless without the turn before it.
5. **Chunks are embedded without their meeting.** The text alone does not say which meeting, date or speaker it came from, so "the pricing meeting" cannot match a chunk that never says "pricing meeting".
6. **The API returns passages, not an answer.** An Ask panel that only lists passages is search. A generated answer is what users expect, and it is also where unfaithful output can enter.
7. **`decisions__{org_id}` exists since P8** and holds the highest-precision facts the product has, but the locked vector arm searches only `transcript_chunks__{org_id}` and `document_chunks__{org_id}`.

## Decision

### D1. Retrieval is participant-scoped (defect fix)

A transcript, chat or decision passage is retrievable only if the asker is in `meeting_participants` for its `meeting_id`, the same rule as `core-api`'s `_require_participant`. Documents stay org-wide. The allowed meeting set is resolved once per query and applied in every arm: a `meeting_id` match-any payload filter in Qdrant (on top of the `org_id` filter), a `meeting_id` restriction in the `search-service` FTS query, and graph templates that start from meetings the asker `ATTENDED` or are filtered to the set afterwards. The answer cache is already keyed per user, so it needs no change. The source of the set is relational truth (`meeting_participants`), not the graph alone, because the `kg` profile is optional in production. Which service reads it is an implementation choice recorded in the band file.

### D2. The expansion call also returns filters

Stage 1 stays one Haiku call, but it returns strict JSON: the 2-3 variants plus optional filters `date_from`, `date_to`, `person` (resolved to `users.id` against the asker's org), and `source` (`transcripts`, `documents`, `decisions`). Today's date is in the prompt, as in `action_agent@v1`. Filters are applied in all arms. When expansion is skipped (tenant `throttled` or `blocked`, or a query of three words or fewer), retrieval runs unfiltered on the raw query. Raw-query retrieval starts at once, in parallel with the expansion call; variant arms join when it returns, so expansion is off the critical path.

### D3. Decisions are a retrieval source and a citation kind

The vector arm also searches `decisions__{org_id}` (status `confirmed` only). Decision and action nodes from the graph arm, and decision hits from Qdrant, cite as kind `decision` (`decision_id`, `meeting_id`) or kind `action` (`action_id`, `meeting_id`). The click target is the decision lineage page or the meeting. This closes Context item 2: the graph arm now produces citable passages.

### D4. Small-to-big context at assembly

Retrieval and reranking stay on single chunks. At Stage 4, each surviving transcript hit is widened to its neighbouring chunks in the same meeting (one before, one after, by `start_ms`) for the context sent to the answer model, inside the same 3,000-token budget. The citation still points at the hit chunk.

### D5. Contextual header at embed time

`embedding-service` embeds `"{meeting title} | {meeting date} | {speaker}: {chunk text}"` for transcript chunks and `"{document title} | {chunk text}"` for document chunks. The stored text and the Qdrant payload are unchanged; only the vector differs. No LLM call per chunk. Existing points are re-embedded once with the existing rebuild path.

### D6. An optional grounded answer, with checked citations and abstention

`POST /api/rag/query` keeps returning cited passages. With `generate: true` it also returns `answer`: a short answer from the Sonnet tier in which every sentence carries one or more `[n]` markers that index into the returned passages. Before return, a sentence with no marker or with a marker outside the returned passages is removed. If the best rerank score is below a threshold, the answer is a fixed "not found in your meetings or documents" with no model call. Before abstaining, there is one corrective retry with the date and person filters dropped. The answer is streamed. The `< 500 ms` warm p95 applies to the passage path. Throttled or blocked tenants get passages only, never an answer, which keeps the degrade-not-fail rule. Answer tokens are metered on `usage-events` as `agent_token`.

### D7. Every upgrade is measured

A RAG eval harness ships with `rag-service`: a golden set of questions over seeded meetings and documents, each with the gold `chunk_id`s, including permission-trap questions (the answer sits in a meeting the asker did not attend) and no-answer questions. Retrieval metrics are Recall@10, MRR and nDCG@10. Answer metrics are citation precision, faithfulness (LLM judge, offline only) and abstain accuracy. The harness also reports passage-path p95. The band file records an ablation table: vector only, then + FTS and RRF, + rerank, + expansion, + filters, + contextual header, + small-to-big. A step that lowers Recall@10 or nDCG@10 is switched off by flag, not shipped.

## Options considered and not taken

- **GraphRAG community summaries.** Needs an LLM pass over the whole corpus to build, and the corpus is small. `nexus_knowledge` already holds structured, confirmed facts.
- **LLM-written context per chunk (contextual retrieval in full).** One LLM call per chunk on ingest, on the hottest topic. D5 gets most of the benefit at no model cost.
- **HyDE.** It writes a fake answer and embeds that. It is weak on names, dates and identifiers, which are what meeting questions contain. D2 covers recall.
- **Agentic multi-hop retrieval.** Each hop adds an LLM round trip and breaks the latency budget. D6's single corrective retry is the bounded version. A "deep" mode can come later.
- **Semantic answer cache.** Answering a different question from a cache risks a wrong answer, and with D1 it would have to respect per-user scope anyway. The exact-match cache stays.
- **Swapping the embedding model or adding a late-interaction model now.** The models are locked (`06-tech-decisions.md` section 2.5) and the GPU path is not live. See Consequences.

## Consequences

- The Ask panel cannot leak a meeting the asker was not in. The permission-trap questions in D7 are the regression test.
- The graph arm contributes for the first time. Citations gain two kinds, so the UI renders four citation types.
- Any client of `rag_client` (the `QuestionAgent` grounding call, `ContextGapAgent` in P10) passes the asking user, because scope is per user. An agent asking on behalf of a participant sees only that participant's meetings.
- D5 needs a one-time re-embed of `transcript_chunks__{org_id}` and `document_chunks__{org_id}`.
- D6 adds one Sonnet call per generated answer. It is metered and gated by `agent_budgets` like every other LLM call.
- **Open risk, not decided here:** `transcription-service` now runs Deepgram Nova-3 multilingual, but `embedding-service` uses `BGE-small-en`, an English-only model. Non-English chunks will retrieve poorly through the vector arm; FTS and the graph still match them. The D7 harness should include a few non-English questions so the size of the problem is measured before anyone decides to change the model.
