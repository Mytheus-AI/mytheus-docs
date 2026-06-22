# CLAUDE.md

This repo, `reprise-docs`, is the architecture documentation set and the naming authority for the product Reprise AI (architecture codename NEXUS). The single most important rule: the frozen `_canon/` folder is the source of truth, and every change flows in one direction only - canon first, then the ADR that records why, then the reader docs (00-24), then the Glossary (doc 24). Never the reverse. If you edit a reader doc to introduce or alter a name, number, or decision without changing canon first, the change is wrong by construction.

## Repo at a glance

This is documentation only. There is no build, no test suite, no application code here. The code lives in separate repos that consume this one (see "How code repos depend on this").

- `_canon/` - the frozen, internal source of truth. Not part of the reading order. Five files: `01-service-catalog.md` (microservices, data ownership, event contracts, sync-API ownership, scaling families), `02-data-model.md` (Postgres tables, Qdrant collections, the AGE `nexus_knowledge` graph, Redis namespaces, object-storage buckets), `03-event-taxonomy.md` (every Redpanda topic with producer/consumers, Avro schema, partition key, retention, delivery/idempotency contract), `05-capacity-math.md` (the closed, self-reconciling capacity model that every load and cost number is imported from), `06-tech-decisions.md` (the frozen lock list of every technology and architecture decision with origin and rationale).
- Reader docs `00-24` - the narrative and reference set. They explain, expand, and narrate the canon; they never invent a name or number the canon does not already define.
- `24-glossary-and-naming.md` - the published, reader-facing companion to `_canon/`. This is the naming authority that humans and dependent repos cite. Keep it consistent with canon at all times.
- `adr/` - 15 ADRs (`0001`-`0015`) recording the WHY behind each load-bearing decision: context, options, decision, consequences.
- `diagrams/` - 12 standalone, copy-pasteable Mermaid sources (`01`-`12`).
- `coverage-traceability-matrix.md` - maps every PDF requirement and every locked SaaS decision to the doc(s) and ADR(s) that realize it. The no-gaps proof for the set.

## How to navigate

Read the reader docs top-to-bottom for the narrative; treat the rest as reference you dip into.

- 00 Executive Summary -> 01 Architecture Overview and Principles: the product and the principles every other doc obeys.
- 02 System Context -> 03 Container Architecture: the C4 zoom from external actors to deployable containers.
- 04 Service Catalog: the spine - every microservice, its data ownership, event contracts, sync APIs, scaling profile.
- 05 Data -> 06 Multi-Tenancy -> 07 Events: how state, isolation, and events fit together.
- 08 Agents -> 09 RAG and Inference Serving -> 10 Real-time and Transcription: the intelligence and live-meeting tiers.
- 11-17 cross-cutting concerns: 11 Security, 12 Compliance, 13 Scalability, 14 Reliability and DR, 15 Observability, 16 Deployment, 17 Cost and FinOps.
- 18 Business -> 19 API -> 20 Frontend -> 21 Integrations.
- 22 Roadmap and Phasing -> 23 Risk Register.

Where to find a given kind of fact:

- A name (service, table, topic, Avro schema, agent, graph node/edge): `_canon/` is authoritative; `24-glossary-and-naming.md` is the reader-facing index of it.
- A number (load, cost, capacity): `_canon/05-capacity-math.md`. Reader docs import these by reference; they do not recompute them.
- Why a decision was made: the relevant file in `adr/`. The README table maps each ADR to its one-line decision.
- A locked technology choice: `_canon/06-tech-decisions.md`.
- A visual: `diagrams/`. The README table maps each diagram to its subject.
- Whether a requirement is covered: `coverage-traceability-matrix.md`.

## The canon contract (read before editing)

The five `_canon/` files are the frozen source of truth. The contract is simple and absolute:

- No document, ADR, diagram, or code repo may introduce an alternative name for anything defined in `_canon/`.
- Reader docs and ADRs CITE canon. They never redefine it, and canon never cites a reader doc as its authority.
- When a reader doc and the canon disagree, the canon wins, and the reader doc is the bug.
- The data model (`_canon/02-data-model.md`) uses informal service labels (`identity-service`, `agent-service`, `tenant-service`, `billing-service`, `graph-service`, `budget-service`, `audit-service`) in its ownership map. These are NOT canonical service names. The Service Catalog (`_canon/01-service-catalog.md`) is the single service-naming authority; doc 24 section 10 maps the informal labels to their catalog equivalents. Do not use the informal labels as canonical names.

## How to make a change (the propagation order)

Follow these steps in order. Skipping a step leaves the set internally inconsistent.

1. Change `_canon/` first. Edit the authoritative file(s) - service catalog, data model, event taxonomy, capacity math, or tech decisions - whichever owns the name, number, or decision you are changing.
2. Add or supersede an ADR in `adr/` recording the why: context, options, decision, consequences. A net-new decision gets a new ADR; a reversal supersedes the prior ADR rather than editing its decision silently.
3. Propagate to the affected reader docs `00-24`. Update every doc that cites the changed canon so the narrative matches the new truth.
4. Update the Glossary, `24-glossary-and-naming.md`, so the published naming authority matches canon. Update any affected `diagrams/` and the `coverage-traceability-matrix.md` in the same pass.
5. Flag net-new SaaS content. Anything you add or change relative to the original execution plan (the PDF) to make the product a production multi-tenant SaaS is marked inline with this exact blockquote:

   > [SaaS-scale adjustment] <what changed relative to the PDF, and why>

Handling a rename: a rename is never silent. Record the change as an explicit `old -> new` entry in the Glossary (doc 24) and supersede or add the ADR that records the why. Then propagate the new name to every reader doc, diagram, and the traceability matrix. The old name must remain visible as a recorded rename so dependent repos can find it. Remember that some frozen identifiers cannot be renamed at all (next section).

## Rules (non-negotiable)

| Rule | Reason | Consequence if broken |
|---|---|---|
| Keep the frozen identifiers verbatim: the AGE graph `nexus_knowledge`, the table `transcript_chunks`, the Redis prefix `nexus:{org_id}:`, the Redpanda topic names, the six agent class names (`PersonalListenerAgent`, `DecisionAgent`, `ActionAgent`, `QuestionAgent`, `ContextGapAgent`, `ProactiveAgent`), and the AGE edge labels (`ATTENDED`, `PRODUCED`, `ASSIGNED_TO`, `RELATES_TO`, `FOLLOWS_FROM`). | These are the frozen naming contract that every dependent repo binds to. The NEXUS codename inside these names is intentional and stays. | Renaming any of them breaks the canon and every code repo that depends on it. |
| Reader docs and ADRs cite canon; canon never cites them. | One-directional authority keeps the set from forking into two truths. | Two competing definitions; readers and code repos cannot tell which is binding. |
| Mark net-new-versus-PDF content with the `[SaaS-scale adjustment]` blockquote, in the exact form shown above. | Lets a reviewer separate the original plan from the SaaS hardening at a glance. | SaaS additions get mistaken for PDF-original scope; traceability breaks. |
| No silent renames. Record `old -> new` in the Glossary plus an ADR. | Dependent repos search by old name; a silent rename strands them. | Code repos drift from canon and fail review against it. |
| Keep `diagrams/` (Mermaid) and `coverage-traceability-matrix.md` consistent with any change. | They are derived views of the same truth. | A diagram or the matrix shows a name or flow that no longer exists. |

## Document conventions

- Markdown throughout; one document per concern, numbered `00`-`24`.
- Mermaid for all diagrams; each `diagrams/*.md` file is a standalone, copy-pasteable source. Keep the fenced ```mermaid block valid - a broken diagram fails to render everywhere the file is viewed.
- Callout format: net-new or hardened-versus-PDF content uses the `[SaaS-scale adjustment]` blockquote shown above. Do not invent other callout forms.
- Relative links between docs, for example `[Service Catalog](04-service-catalog.md)` and `[_canon/02-data-model.md](_canon/02-data-model.md)`, so the set stays portable.
- Frozen identifiers are written verbatim and in code spans.
- Plain-text headers. No emojis. Use a hyphen, a colon, or a restructured sentence instead of em-dashes or en-dashes.

## How code repos depend on this

These docs are the contract for the implementation repos. Service, table, topic, Qdrant collection, Redis key, and AGE label names in the code MUST match the canon verbatim.

| Doc area | Implementation repo |
|---|---|
| 20 Frontend (Next.js 14, meeting room, agent sidebar) | `reprise-ui` |
| Mobile clients | `reprise-mobile` |
| 04 Service Catalog, 05 Data, 06 Tenancy, 18 Business, 19 API (CRUD/control plane) | `reprise-backend` |
| 08 Agents, 09 RAG and inference serving | `reprise-intelligence` |
| 10 Real-time and transcription (`realtime-gateway`, transcription, fan-out) | `reprise-realtime` |
| 13 Scalability, 14 Reliability, 15 Observability, 16 Deployment (platform/IaC) | `reprise-platform` |
| 07 Event-driven architecture (Redpanda topics, Avro schemas, outbox/inbox) | `reprise-events` |

`reprise-backend`, `reprise-intelligence`, `reprise-realtime`, and `reprise-events` are bound most tightly to canon. A doc change that renames anything is a breaking change for these repos: it must be coordinated with them, not landed unilaterally. A migration name, topic, collection, Redis key, or AGE label in those repos that disagrees with `_canon/` is a bug in the code repo, not in the docs.

## Pitfalls

- Editing a reader doc to change a name, number, or decision without changing `_canon/` first. The reader doc now contradicts the source of truth.
- Introducing a name that is not in the canon. Every name a reader doc uses must trace back to `_canon/` (and appear in the Glossary).
- A silent rename. Always leave an `old -> new` trail in the Glossary plus an ADR.
- Renaming a frozen identifier (the list in Rules). These cannot change without breaking every dependent repo.
- Updating canon or a reader doc but leaving the Glossary (doc 24) or `coverage-traceability-matrix.md` out of sync.
- Breaking a Mermaid diagram: an invalid fenced block, or a node/edge name that no longer matches canon after a rename.
- Treating the informal data-model service labels as canonical service names. Use the Service Catalog names.

## Commit and PR conventions

- Review is required. A PR that touches a name, number, or decision must update `_canon/` plus the ADR plus the affected reader docs plus the Glossary together, or it does not merge. Canon and Glossary changes are reviewed by the canon owners.
- Use conventional commits.
- Commits in this project must NOT include a Co-Authored-By footer.
