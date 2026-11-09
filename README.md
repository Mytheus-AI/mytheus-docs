# Mytheus - Architecture Documentation (`mytheus-docs`)

The complete production architecture set for **Mytheus** - an intelligent meeting-orchestration platform built as a multi-tenant SaaS for millions of users across thousands of paying tenant organizations.

---

## About the codename

The documents in this repository are internally codenamed **NEXUS**. **Mytheus** is the product name; NEXUS is the architecture codename you will see throughout the copied docs, and that is intentional.

Certain frozen canon identifiers are kept **verbatim** and are deliberately **not** renamed to "mytheus":

- the AGE knowledge graph `nexus_knowledge`
- the table `transcript_chunks`
- the Redis key prefix `nexus:{org_id}:`
- the Redpanda topic names (`transcript-chunks`, `agent-requests`, `agent-responses`, `usage-events`, and the rest of the topic taxonomy)

These are the **frozen naming contract**. Renaming them would break the canon and every implementation repo that depends on it, so they stay as-is. New product-facing surfaces use the name "Mytheus"; the frozen internal identifiers stay NEXUS. See [`system.md`](system.md) and [`24-glossary-and-naming.md`](24-glossary-and-naming.md) for the full naming authority.

---

## How to read this

Read top-to-bottom for the narrative; the rest is reference you dip into.

1. [00 Executive Summary](00-executive-summary.md) - what Mytheus is and why the architecture looks the way it does.
2. [01 Architecture Overview & Principles](01-architecture-overview-and-principles.md) - the principles every other doc obeys.
3. [02 System Context](02-system-context.md) → [03 Container Architecture](03-container-architecture.md) - C4 zoom from external actors to deployable containers.
4. [04 Service Catalog](04-service-catalog.md) - the spine: every microservice, its data ownership, event contracts.
5. [05 Data](05-data-architecture.md) → [06 Tenancy](06-multi-tenancy.md) → [07 Events](07-event-architecture.md) - how state, isolation, and events fit together.
6. [08 Agents](08-ai-agent-architecture.md) → [09 RAG](09-rag-and-inference-serving.md) → [10 Real-time](10-realtime-and-transcription.md) - the intelligence and live-meeting tiers.
7. [11](11-security-architecture.md)–[17](17-cost-and-finops.md) cross-cutting concerns: security, compliance, scalability, reliability, observability, deployment, cost.
8. [18 Business](18-saas-business-architecture.md) → [19 API](19-api-architecture.md) → [20 Frontend](20-frontend-architecture.md) → [21 Integrations](21-integration-architecture.md).
9. [22 Roadmap](22-roadmap-and-phasing.md) → [23 Risks](23-risk-register.md).

---

## Table of contents - core documents (00–24)

| # | Document | Description |
|---|---|---|
| 00 | [Executive Summary](00-executive-summary.md) | The product, the SaaS re-target, and the headline architecture decisions in one read. |
| 01 | [Architecture Overview & Principles](01-architecture-overview-and-principles.md) | The principles every other doc obeys: tenant isolation at every layer, at-least-once delivery, independent tier scaling. |
| 02 | [System Context (C4 L1)](02-system-context.md) | External actors, subprocessors, and the system boundary. |
| 03 | [Container Architecture (C4 L2)](03-container-architecture.md) | Deployable containers, their runtimes, and the wiring between tiers. |
| 04 | [Service Catalog & Bounded Contexts](04-service-catalog.md) | Canonical microservice list, data ownership, event contracts, sync APIs, and scaling profiles. |
| 05 | [Data Architecture](05-data-architecture.md) | Store selection, partitioning, retention, backups/PITR, read replicas, and regional placement across Postgres/Qdrant/AGE/Redis/Object. |
| 06 | [Multi-Tenancy & Tenant Isolation](06-multi-tenancy.md) | Pooled + RLS for the long tail, cell-based isolation for enterprise, and `org_id` injection at every layer. |
| 07 | [Event-Driven Architecture](07-event-architecture.md) | Redpanda topics, Avro + Schema Registry, tenant partitioning, and the outbox/inbox + DLQ delivery contract. |
| 08 | [AI Agent Architecture](08-ai-agent-architecture.md) | The six LangGraph agents, `BaseAgent` lifecycle, confidence thresholds, and the cost-controlled LLM pipeline. |
| 09 | [RAG Pipeline & Inference Serving](09-rag-and-inference-serving.md) | `RAGService` expand→retrieve→rerank→assemble→cite, hybrid retrieval, and dedicated GPU embedding/reranker serving. |
| 10 | [Real-time & Transcription](10-realtime-and-transcription.md) | LiveKit transport, Deepgram WS proxy, `transcript_chunks`, live captions, and Socket.io fan-out. |
| 11 | [Security Architecture](11-security-architecture.md) | Shift-left security, API gateway + WAF, RBAC, agent-to-agent authz, and secrets handling. |
| 12 | [Compliance & Data Privacy](12-compliance-and-privacy.md) | GDPR/CCPA, DPAs + SCCs, data residency, PII redaction, right-to-be-forgotten, audit logging, SOC2 readiness. |
| 13 | [Scalability & Performance](13-scalability-and-performance.md) | The frozen capacity model, autoscaler signals, and the SLO targets every tier is sized against. |
| 14 | [Reliability & Disaster Recovery](14-reliability-and-dr.md) | Availability tiers, RPO/RTO, multi-region active-passive→active-active, and failure-mode handling. |
| 15 | [Observability & SRE](15-observability-and-sre.md) | Structured logs, metrics/dashboards, tracing, consumer-lag and quota alerts, and on-call practice. |
| 16 | [Deployment & Infrastructure](16-deployment-and-infrastructure.md) | Kubernetes (EKS/GKE) + Terraform IaC, the single-node VPS MVP substrate, and CI/CD via GitHub Actions. |
| 17 | [Cost Architecture & FinOps](17-cost-and-finops.md) | Per-tier cost centers, the variable cost model, unit economics, and the R4 cost levers. |
| 18 | [SaaS Business Architecture](18-saas-business-architecture.md) | Tenant lifecycle, Stripe billing/seats/dunning/tax, usage metering→quota enforcement, plans, and consoles. |
| 19 | [API Architecture](19-api-architecture.md) | Sync API ownership, the `bff`/`api-gateway` edge, versioning, and the OpenAPI surface. |
| 20 | [Frontend Architecture](20-frontend-architecture.md) | Next.js 14 App Router, the meeting room, and the agent sidebar design surface. |
| 21 | [Integration Architecture](21-integration-architecture.md) | Google Calendar, Slack, and Jira/Linear OAuth + bidirectional sync, gated by guardrail R8. |
| 22 | [Roadmap & Phasing](22-roadmap-and-phasing.md) | The phase plan mapped onto the SaaS architecture and the pre-authorized descopes. |
| 23 | [Risk Register](23-risk-register.md) | PDF risks R1–R8 plus SaaS-scale risks, with owners, mitigations, and triggers. |
| 24 | [Glossary & Naming](24-glossary-and-naming.md) | The canonical naming authority - every agent, entity, service, topic, Avro schema, and AGE node/edge label, with any renames flagged. |

---

## Traceability

| Document | Description |
|---|---|
| [Coverage Traceability Matrix](coverage-traceability-matrix.md) | Maps every PDF requirement and every locked SaaS decision to the document(s) and ADR(s) that realize it - the no-gaps proof for the set. |

---

## Execution and progress

| Folder | Description |
|---|---|
| [`plan/`](plan/README.md) | The execution layer: eight band files (Plan 0 through Plan 14) turning these documents into an ordered, buildable program of work. Says *what to build, in what order*. |
| [`work-done-till-now/`](work-done-till-now/README.md) | The single, program-wide record of what has actually been *built*, one file per band. Start at [00-status-overview.md](work-done-till-now/00-status-overview.md) for the band-by-repo matrix and the live gap list. No code repo keeps its own progress log; they all record here. |

---

## Architecture Decision Records (`adr/`)

The ADRs record *why* each load-bearing decision was made.

| ADR | Decision (one line) |
|---|---|
| [ADR-0001](adr/0001-event-driven-microservices.md) | Adopt event-driven microservices with an explicit modular-monolith → services evolution path. |
| [ADR-0002](adr/0002-multi-tenancy-model.md) | Pooled + PostgreSQL RLS for the long tail; cell-based isolation for large/enterprise tenants. |
| [ADR-0003](adr/0003-meeting-sdk-livekit.md) | Lock LiveKit (self-hostable SFU) behind the transport adapter; resolves PDF R2. |
| [ADR-0004](adr/0004-llm-cost-control-pipeline.md) | Windowed batching + relevance gating + tiered routing + caching + per-tenant budgets; defeats PDF R4. |
| [ADR-0005](adr/0005-graph-store-age-with-migration.md) | Apache AGE `nexus_knowledge` now, with a Neptune/Neo4j Aura + relational MV migration path; honors R1. |
| [ADR-0006](adr/0006-vector-store-tenancy.md) | Per-tenant Qdrant collections with mandatory `org_id` payload filter. |
| [ADR-0007](adr/0007-event-delivery-idempotency.md) | Avro + Schema Registry (BACKWARD), transactional outbox/inbox, idempotency keys, DLQs. |
| [ADR-0008](adr/0008-dedicated-inference-serving.md) | BGE-small embeddings and BGE-reranker on a dedicated GPU/Ray Serve tier, off the API tier. |
| [ADR-0009](adr/0009-data-residency-multi-region.md) | Regional placement per tenant and a multi-region active-passive→active-active topology. |
| [ADR-0010](adr/0010-auth-clerk-enterprise-sso.md) | Clerk with SSO/SAML/SCIM and hardened user-sync (outbox + reconciliation). |
| [ADR-0011](adr/0011-billing-and-metering.md) | Stripe billing/seats/dunning/tax driven by the `usage-events` metering pipeline. |
| [ADR-0012](adr/0012-realtime-transport-websocket-tier.md) | A dedicated, connection-scaled WebSocket gateway tier (`realtime-gateway`). |
| [ADR-0013](adr/0013-agent-to-agent-authorization.md) | Deny-by-default, org/role-scoped, audited authorization for `agent-requests`/`agent-responses`. |
| [ADR-0014](adr/0014-kubernetes-compute-platform.md) | EKS/GKE + Terraform IaC for production; Railway/Render for the early build. |
| [ADR-0015](adr/0015-observability-stack.md) | OpenTelemetry + Prometheus/Grafana + Sentry with structured JSON logs. |
| [ADR-0016](adr/0016-self-hosted-vps-mvp-substrate.md) | MVP backend on a self-hosted single-node VPS (Docker Compose); supersedes the Railway/Render fast path in ADR-0014. |
| [ADR-0017](adr/0017-transcript-chunks-composite-partition-key.md) | `transcript-chunks` and `agent-stream` are composite-keyed `{org_id}:{meeting_id}`; the other 19 topics keep `org_id` alone. |
| [ADR-0018](adr/0018-consent-entity-behind-flag.md) | Consent entity landed schema-first behind `tenant_settings.consent_required`, defaulted off. |
| [ADR-0019](adr/0019-ai-tier-verifies-jwt-and-reads-resolved-org-claim.md) | `mytheus-intelligence` verifies the Clerk JWT itself and reads a pre-resolved internal `org_id` claim; no `org_id` from a request header. |
| [ADR-0020](adr/0020-pii-token-vault-and-ner-placement.md) | The per-tenant PII token vault (`pii_tokens`) is owned and written only by `identity-tenant-service`; the three redaction boundaries call it. NER runs on the inference tier, never in `transcription-service`. Implementation blocked on key custody. |
| [ADR-0021](adr/0021-explicit-grants-no-schema-wide-default-privileges.md) | No schema-wide default privilege grants on TABLES in `public`; every migration grants its own tenant table, no partition carries a direct grant, and no `rolbypassrls` role holds a privilege on a tenant table. |
| [ADR-0022](adr/0022-openapi-emitted-from-code.md) | The REST contract is code-first: each service's OpenAPI document is emitted from its FastAPI app and committed, governed by a regeneration check, `spectral lint` and an `oasdiff` breaking-change gate. Reverses doc 19 section 11's spec-first direction. |
| [ADR-0023](adr/0023-object-storage-cloudflare-r2.md) | Object storage is Cloudflare R2 over its S3-compatible API; chosen for zero egress on a read-heavy recordings workload, with R2's EU-jurisdiction-only residency guarantee recorded as a disclosed limit. |
| [ADR-0024](adr/0024-key-custody-and-envelope-encryption.md) | Field-level envelope encryption (AES-256-GCM, per-tenant DEKs, KEK behind a `KeyProvider` seam, AAD bound to `org_id` + purpose) for `oauth_tokens` and `pii_tokens`; the production key custodian is an open decision and the dev provider fails closed outside development. |
| [ADR-0025](adr/0025-tenant-provisioning-on-the-request-path.md) | `POST /orgs` may create tenant roots on the request path: `mytheus_app` gets column-level INSERT on exactly the provisioning columns of `organizations`, no UPDATE/DELETE; the route gates callers to owners of `PLATFORM_OPERATOR_ORG_IDS` tenants (proposed name). Supersedes the operator-seam-only rule recorded in migration `a1c4e7f20b18`. |
| [ADR-0026](adr/0026-vps-file-kek-custody.md) | Production key custody on the VPS is a local KEK file (`FileKeyProvider`, mode 0400, mounted into `identity-tenant-service` only, backed up off-box separately from DB backups); the shared-host residual risk is accepted and KMS is the explicit upgrade trigger. Decides ADR-0024 open decision 1. |
| [ADR-0027](adr/0027-anthropic-only-llm-provider.md) | Anthropic is the only LLM provider; OpenAI dropped; LiteLLM stays as the provider-agnostic router; fast-response tier is Haiku 5.5 (`claude-haiku-5-5`); other tiers: Anthropic models, mapping to be set. |
| [ADR-0028](adr/0028-object-storage-buckets-use-the-mytheus-prefix.md) | Object storage buckets are named `mytheus-recordings`, `mytheus-documents`, `mytheus-attachments`, `mytheus-invoices`, `mytheus-exports`, `mytheus-audit-archive` (was `nexus-*`) |
| [ADR-0029](adr/0029-plan-7-budget-scope-tier-mapping-and-metering-ownership.md) | Plan 7 entry decisions: plan allowances are per tenant, not per seat; extraction tier is Sonnet 5.5 (`claude-sonnet-5-5`); `billing-metering-service` derives minutes, transcription seconds and storage itself; `model_tier` added to `usage_events` and `UsageEvent`; no Anthropic DPA gate for this academic build. |
| [ADR-0030](adr/0030-plan-7-llm-response-cache-and-personal-listener-gating.md) | Plan 7 build decisions: exact-match LLM response cache `nexus:{org_id}:llmcache:{prompt_ref}:{input_hash}` (1h); PersonalListener relevance threshold 0.5 (tunable); `core-api` decides `surfaced` with `agent_confidence_overrides`; only surfaced-band notes stream; live agent output delivered per user over the meeting channel. |
| [ADR-0031](adr/0031-plan-8-graph-store-decision-supersedes-and-mark-done.md) | Plan 8 entry decisions: `nexus_knowledge` in a dedicated self-hosted `apache/age` Postgres in production (R1 fallback); optional `supersedes` on `Decision` as the `FOLLOWS_FROM` source; `POST /api/v1/actions/{id}/done` emits nothing; `Decision` node carries `meeting_id`, `content`, `confirmed_at`; `embedding-service` consumes `decisions`; the dedupe cache key is scoped by consumer group; `knowledge-graph-service` writes `ATTENDED` from `meeting-room-events`. |
| [ADR-0032](adr/0032-plan-8-decision-and-action-candidates.md) | Plan 8 part A build decisions: `core-api` fills `supersedes` on confirm; candidates as a list in `payload_json` with deterministic ids; candidates delivered to the whole meeting (`agent-candidate`); any meeting participant confirms; silent band is a never-surfaced `proposed` row. |
| [ADR-0033](adr/0033-plan-9-rag-retrieval-upgrades.md) | Plan 9 RAG entry decisions: participant-scoped retrieval; filters from the expansion call; `decisions__{org_id}` as a source with `decision`/`action` citations; small-to-big context; contextual header at embed time; optional streamed grounded answer with checked citations and abstention; every upgrade measured by an ablation. |
| [ADR-0034](adr/0034-plan-9-question-agent-suggestions.md) | Plan 9 QuestionAgent build decisions: one agent per participant; heuristic, Haiku gate, Sonnet proposal; kind `suggestion` with deterministic ids; the band stored as `confidence` and applied at read time; suggestions private to their owner and delivered to that user only; the status transitions; Save-for-later as a read rule; optional participant-scoped, passages-only grounding. |

---

## Diagrams (`diagrams/`)

12 standalone, copy-pasteable Mermaid sources, sequenced 01–12.

| Diagram | File |
|---|---|
| System Context (C4 L1) | [01-system-context.md](diagrams/01-system-context.md) |
| Container Diagram (C4 L2) | [02-container.md](diagrams/02-container.md) |
| Agent Runtime Components (C4 L3) | [03-agent-runtime-component.md](diagrams/03-agent-runtime-component.md) |
| Entity Relationship Diagram | [04-data-erd.md](diagrams/04-data-erd.md) |
| Event Flow: transcript → agent | [05-event-flow-transcript-to-agent.md](diagrams/05-event-flow-transcript-to-agent.md) |
| Meeting Lifecycle Sequence | [06-meeting-lifecycle-sequence.md](diagrams/06-meeting-lifecycle-sequence.md) |
| Multi-Agent Collaboration Sequence | [07-agent-collaboration-sequence.md](diagrams/07-agent-collaboration-sequence.md) |
| RAG Pipeline Flow | [08-rag-flow.md](diagrams/08-rag-flow.md) |
| Multi-Tenancy Isolation | [09-multi-tenancy-isolation.md](diagrams/09-multi-tenancy-isolation.md) |
| Deployment & Multi-Region | [10-deployment-multiregion.md](diagrams/10-deployment-multiregion.md) |
| Billing & Metering Flow | [11-billing-metering-flow.md](diagrams/11-billing-metering-flow.md) |
| Real-time Fan-out | [12-realtime-fanout.md](diagrams/12-realtime-fanout.md) |

---

## The `_canon/` folder

`_canon/` is the **frozen, internal source of truth** for the whole set. It is not a reader document and is intentionally absent from the reading order above. Every other doc derives its service names, table/column names, event-topic names, capacity numbers, and locked decisions from these files - no document may introduce an alternative name for anything defined here.

- [`_canon/01-service-catalog.md`](_canon/01-service-catalog.md) - the single naming authority for microservices, data ownership, event contracts, sync-API ownership, and scaling families.
- [`_canon/02-data-model.md`](_canon/02-data-model.md) - the frozen data-naming authority across PostgreSQL tables, Qdrant collections, the AGE `nexus_knowledge` graph, Redis namespaces, and object-storage buckets.
- [`_canon/03-event-taxonomy.md`](_canon/03-event-taxonomy.md) - the frozen event-naming authority: every Redpanda topic with its producer/consumers, Avro schema, partition key, retention, and delivery/idempotency contract.
- [`_canon/05-capacity-math.md`](_canon/05-capacity-math.md) - the closed, self-reconciling capacity model; every load and cost number is imported from here by reference.
- [`_canon/06-tech-decisions.md`](_canon/06-tech-decisions.md) - the frozen lock list of every technology and architecture decision with its origin and rationale.

The published, reader-facing companion to this folder is [24 Glossary & Naming](24-glossary-and-naming.md). If a number, name, or decision needs to change, it changes in `_canon/` first and propagates outward; reader docs and ADRs cite canon, never the reverse.

---

## How this maps to the code repos

| Doc area | Implementation repo |
|---|---|
| 20 Frontend (Next.js 14, meeting room, agent sidebar) | `mytheus-ui` |
| Mobile clients | `mytheus-mobile` |
| 04 Service Catalog · 05 Data · 06 Tenancy · 18 Business · 19 API (CRUD/control plane) | `mytheus-backend` |
| 08 Agents · 09 RAG & inference serving | `mytheus-intelligence` |
| 10 Real-time & transcription (`realtime-gateway`, transcription, fan-out) | `mytheus-realtime` |
| 13 Scalability · 16 Deployment · 14 Reliability · 15 Observability (platform/IaC) | `mytheus-platform` |
| 07 Event-driven architecture (Redpanda topics, Avro schemas, outbox/inbox) | `mytheus-events` |

Service, table, topic, and store names in these repos MUST match the canon verbatim - including the frozen identifiers `nexus_knowledge`, `transcript_chunks`, the `nexus:{org_id}:` Redis prefix, and the Redpanda topic names.

---

## License

Proprietary © Mytheus. All rights reserved.
