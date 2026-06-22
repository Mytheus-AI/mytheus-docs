# CLAUDE.md

> The architecture documentation set and naming authority for **Reprise AI** (codename NEXUS): the single source of architectural truth that every other repo cites.

## 🎯 Purpose

This repository is the **architecture documentation and naming authority** for Reprise AI (architecture codename NEXUS). It is the single source of architectural truth that every other repo cites.

You are working on documentation and a frozen naming contract, not application code. Precision about names, numbers, and decisions matters more than prose.

## 🗂️ How to Navigate

**Reading order (the `00`-`24` doc set):**

1. `00` Exec Summary
2. `01` Principles
3. `02`-`03` C4
4. `04` Service Catalog
5. `05`-`07` Data / Tenancy / Events
6. `08`-`10` Agents / RAG / Real-time
7. `11`-`17` Cross-cutting
8. `18`-`21` Business / API / FE / Integrations
9. `22`-`23` Roadmap / Risks

**Where to find what:**

| You need... | Look in |
| --- | --- |
| Names, numbers, decisions | `_canon/` |
| The published naming authority | `24-glossary-and-naming.md` (Glossary, doc `24`) |
| The WHY of any decision | the matching `adr/000X-*.md` |
| Visuals | `diagrams/01`-`12` |
| The no-gaps proof | `coverage-traceability-matrix.md` |

## 📛 The Canon Contract (read first)

The `_canon/` folder is the **frozen source of truth**. Never silently introduce or rename a service, table, topic, store, or agent name.

Changes flow in **one direction only**:

```
_canon/  ->  ADR (adr/)  ->  reader docs (00-24)  ->  Glossary (24-glossary-and-naming.md)
```

Reader docs and ADRs **cite canon; canon never cites them.** If a name, number, or decision must change, change `_canon/` first, record an ADR (with an explicit `old -> new` entry, superseding the prior ADR), then propagate outward to the reader docs and the Glossary.

## 🚦 Rules (non-negotiable)

- **Keep frozen identifiers verbatim.** Never rename them, and never substitute "reprise" for them:

  | Kind | Frozen identifier(s) |
  | --- | --- |
  | AGE graph | `nexus_knowledge` |
  | Table | `transcript_chunks` |
  | Redis prefix | `nexus:{org_id}:` |
  | Redpanda topics | `transcript-chunks`, `agent-requests`, `agent-responses`, `usage-events`, etc. |
  | Six agent classes | `PersonalListenerAgent`, `DecisionAgent`, `ActionAgent`, `QuestionAgent`, `ContextGapAgent`, `ProactiveAgent` |
  | AGE edge labels | `ATTENDED`, `PRODUCED`, `ASSIGNED_TO`, `RELATES_TO`, `FOLLOWS_FROM` |

- **Mark any net-new content** (added or hardened relative to the original PDF) with the callout:

  > [SaaS-scale adjustment] &lt;what changed and why&gt;

- **No silent renames.** Reader docs cite canon, never the reverse.

## ✅ Do / ❌ Don't

**✅ Do**

- Edit `_canon/` first when a name, number, or decision changes, then propagate.
- Keep relative links and Mermaid diagram sources working.
- Keep the Glossary (`24`) consistent with `_canon/`.

**❌ Don't**

- Modify the substance of copied doc contents without a corresponding canon / ADR change.
- Invent a new name for anything already named in `_canon/`.
- Rename frozen identifiers to match the product name.

## 🔗 How Code Repos Depend on This

`reprise-ui`, `reprise-mobile`, `reprise-backend`, `reprise-intelligence`, `reprise-realtime`, `reprise-platform`, and `reprise-events` all cite this repo.

Service, table, topic, and store names in `reprise-backend`, `reprise-intelligence`, `reprise-realtime`, and `reprise-events` **MUST match the canon verbatim.** A mismatch is a bug in the code repo.

## 🔀 Pull Requests

- A PR that changes a name, number, or decision must update `_canon/` + ADR + affected reader docs + Glossary **together**.
- Use relative links and the `[SaaS-scale adjustment]` callout where applicable.
- **Commits in this project must NOT include a `Co-Authored-By` footer.**
