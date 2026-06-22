# `reprise-docs` — Operating Document

This is the meta/operating guide for the documentation repository itself. It describes how the set is organized, who owns what, and how it changes.

---

## Purpose

`reprise-docs` is the **single source of architectural truth** for Reprise AI. It is the canonical home of the full architecture documentation set (codename NEXUS). Every other repository — `reprise-ui`, `reprise-mobile`, `reprise-backend`, `reprise-intelligence`, `reprise-realtime`, `reprise-platform`, `reprise-events` — cites this repo rather than redefining architecture, names, or numbers locally.

---

## The documentation hierarchy

The set has a strict authority order. Higher levels bind lower levels.

1. **The frozen `_canon/` files are authoritative.** They define every name, number, and decision.
   - [`_canon/01-service-catalog.md`](_canon/01-service-catalog.md) — microservices, data ownership, event contracts, sync-API ownership, scaling families.
   - [`_canon/02-data-model.md`](_canon/02-data-model.md) — PostgreSQL tables, Qdrant collections, the AGE `nexus_knowledge` graph, Redis namespaces, object-storage buckets.
   - [`_canon/03-event-taxonomy.md`](_canon/03-event-taxonomy.md) — every Redpanda topic with producer/consumers, Avro schema, partition key, retention, delivery/idempotency contract.
   - [`_canon/05-capacity-math.md`](_canon/05-capacity-math.md) — the closed, self-reconciling capacity model; every load and cost number is imported from here.
   - [`_canon/06-tech-decisions.md`](_canon/06-tech-decisions.md) — the frozen lock list of every technology/architecture decision with origin and rationale.
2. **Reader docs `00`–`24` derive from the canon.** They explain, expand, and narrate, but they never invent a name or number the canon does not already define.
3. **ADRs (`adr/0001`–`0015`) record the WHY.** Each load-bearing decision has an ADR documenting context, options, decision, and consequences.
4. **Diagrams (`diagrams/01`–`12`) visualize.** Standalone Mermaid sources that picture what the docs describe.
5. **The Glossary, [`24-glossary-and-naming.md`](24-glossary-and-naming.md), is the published naming authority.** It is the reader-facing companion to `_canon/` and is kept consistent with it.

---

## Naming authority & the canon contract

- No document or code repo may introduce an **alternative name** for anything defined in `_canon/`.
- If a name, number, or decision must change, it changes in **`_canon/` first**, then propagates outward to the reader docs, ADRs, Glossary, and finally the implementation repos.
- Reader docs and ADRs **cite canon, never the reverse.** Canon does not reference reader docs as authority.
- The frozen identifiers are kept verbatim and are never renamed to "reprise": the AGE graph `nexus_knowledge`, the table `transcript_chunks`, the Redis prefix `nexus:{org_id}:`, the Redpanda topic names, the six agent class names, and the AGE edge labels (`ATTENDED` / `PRODUCED` / `ASSIGNED_TO` / `RELATES_TO` / `FOLLOWS_FROM`). These constitute the frozen naming contract; renaming them would break the canon and every dependent repo.
- **The `[SaaS-scale adjustment]` callout convention:** anything these documents add or change relative to the original execution plan (the PDF) to make the product a production multi-tenant SaaS is flagged inline with a blockquote in this exact form:
  > [SaaS-scale adjustment] &lt;what changed relative to the PDF, and why&gt;

---

## How code repos consume these docs

Service, table, topic, and store names used in `reprise-backend`, `reprise-intelligence`, `reprise-realtime`, and `reprise-events` **MUST match the canon verbatim**. A migration name, Kafka/Redpanda topic, Qdrant collection, Redis key, or AGE label that disagrees with `_canon/` is a bug in the code repo, not in the docs. When in doubt, the canon and the Glossary ([`24-glossary-and-naming.md`](24-glossary-and-naming.md)) win.

---

## Change process

To propose a change to architecture, a name, a number, or a decision:

1. **Edit `_canon/`** — change the authoritative file(s).
2. **Add or supersede an ADR** in `adr/` recording the WHY. Never silently rename; record an explicit `old → new` entry and supersede via ADR.
3. **Propagate to reader docs** `00`–`24` so they stay consistent with the canon.
4. **Update the Glossary** [`24-glossary-and-naming.md`](24-glossary-and-naming.md) (and diagrams if affected).
5. **Open a PR.** Review gates below apply; merge only after canon, ADR, reader docs, and Glossary are mutually consistent.

---

## Document conventions

- **Markdown** throughout; one document per concern, numbered `00`–`24`.
- **Mermaid** for all diagrams; each `diagrams/*.md` file is a standalone, copy-pasteable source.
- **Callout format:** net-new/hardened-versus-PDF content uses the `[SaaS-scale adjustment]` blockquote shown above.
- **Relative links** between docs (e.g. `[Service Catalog](04-service-catalog.md)`, `[`_canon/02-data-model.md`](_canon/02-data-model.md)`) so the set is portable.
- **Frozen identifiers** are written verbatim and in code spans.

---

## Contributing

- **Canon owners** review any change to `_canon/` and to `24-glossary-and-naming.md` — these carry the naming authority.
- **Domain owners** review the reader doc(s) for their area (data, events, agents, real-time, security, business, etc.).
- **ADR authors** own the decision record they introduce or supersede.
- Review gate: a PR touching a name/number/decision must update `_canon/` + ADR + affected reader docs + Glossary together, or it does not merge.

---

## Status & versioning

This set is the canonical, frozen-contract version of the Reprise AI architecture. The `_canon/` files are the version anchor; the reader docs, ADRs, diagrams, and Glossary are versioned together with them. Changes are tracked via PRs following the change process above. Commits in this project do **not** include a Co-Authored-By footer.
