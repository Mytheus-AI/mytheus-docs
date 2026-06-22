# CLAUDE.md — Working guide for AI agents in `reprise-docs`

## Purpose

This repository is the **architecture documentation and naming authority** for Reprise AI (architecture codename NEXUS). It is the single source of architectural truth that every other repo cites. You are working on documentation and a frozen naming contract — not application code. Precision about names, numbers, and decisions matters more than prose.

## The canon contract

The `_canon/` folder is the **frozen source of truth**. Never silently introduce or rename a service, table, topic, store, or agent name. Changes flow in one direction:

```
_canon/  →  ADR (adr/)  →  reader docs (00–24)  →  Glossary (24-glossary-and-naming.md)
```

Reader docs and ADRs cite canon; canon never cites them. If a name/number/decision must change, change `_canon/` first, record an ADR (with an explicit `old → new` entry, superseding the prior ADR), then propagate outward to the reader docs and the Glossary.

## How to navigate the set

- **Reading order:** `00` Exec Summary → `01` Principles → `02`–`03` C4 → `04` Service Catalog → `05`–`07` Data/Tenancy/Events → `08`–`10` Agents/RAG/Real-time → `11`–`17` cross-cutting → `18`–`21` business/API/FE/integrations → `22`–`23` roadmap/risks.
- **Where to find what:** names/numbers/decisions live in `_canon/`; the published naming authority is `24-glossary-and-naming.md`; the WHY of any decision lives in the matching `adr/000X-*.md`; visuals live in `diagrams/01`–`12`; the no-gaps proof is `coverage-traceability-matrix.md`.

## Rules

- **Keep frozen identifiers verbatim.** Never rename, and never substitute "reprise" for them:
  - the AGE graph `nexus_knowledge`
  - the table `transcript_chunks`
  - the Redis prefix `nexus:{org_id}:`
  - the Redpanda topic names (`transcript-chunks`, `agent-requests`, `agent-responses`, `usage-events`, etc.)
  - the six agent class names: `PersonalListenerAgent`, `DecisionAgent`, `ActionAgent`, `QuestionAgent`, `ContextGapAgent`, `ProactiveAgent`
  - the AGE edge labels: `ATTENDED`, `PRODUCED`, `ASSIGNED_TO`, `RELATES_TO`, `FOLLOWS_FROM`
- **Mark any net-new content** (added or hardened relative to the original PDF) with the callout:
  > [SaaS-scale adjustment] &lt;what changed and why&gt;
- **Reader docs cite canon, never the reverse.**

## Do / Don't

- **Do** edit `_canon/` first when a name/number/decision changes, then propagate.
- **Do** keep relative links and Mermaid diagram sources working.
- **Do** keep the Glossary (`24`) consistent with `_canon/`.
- **Don't** modify the substance of copied doc contents without a corresponding canon/ADR change.
- **Don't** invent a new name for anything already named in `_canon/`.
- **Don't** rename frozen identifiers to match the product name.

## How implementation repos depend on this

`reprise-ui`, `reprise-mobile`, `reprise-backend`, `reprise-intelligence`, `reprise-realtime`, `reprise-platform`, and `reprise-events` all cite this repo. Service/table/topic/store names in `reprise-backend`, `reprise-intelligence`, `reprise-realtime`, and `reprise-events` MUST match the canon verbatim. A mismatch is a bug in the code repo.

## PR conventions

- A PR that changes a name/number/decision must update `_canon/` + ADR + affected reader docs + Glossary together.
- Use relative links and the `[SaaS-scale adjustment]` callout where applicable.
- **Commits in this project must NOT include a `Co-Authored-By` footer.**
