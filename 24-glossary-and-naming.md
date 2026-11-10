# Glossary & Naming Authority

This is the canonical glossary and naming reference for the NEXUS architecture set: every important term — service, entity, agent, topic, Avro schema, graph label, and concept — defined once, with its canonical spelling and any recorded `old → new` rename, so that no other document, ADR, or diagram introduces an alternative name.

## Related documents

- [Service Catalog](04-service-catalog.md) — the single naming authority for services; this glossary's service rows defer to it.
- [Data Architecture](05-data-architecture.md) — store selection, retention, and partitioning for the entities defined here.
- [Event-Driven Architecture](07-event-architecture.md) — Redpanda topics, Avro envelopes, and the topic/schema reconciliation notes this glossary cites.
- [AI Agent Architecture](08-ai-agent-architecture.md) — the six agents, `BaseAgent`/`AgentState`, and the cost-controlled pipeline.
- [`_canon/01-service-catalog.md`](_canon/01-service-catalog.md) — frozen service-naming authority (20 services).
- [`_canon/02-data-model.md`](_canon/02-data-model.md) — frozen data-naming authority (entities, Qdrant collections, AGE labels, Redis namespaces).
- [`_canon/06-tech-decisions.md`](_canon/06-tech-decisions.md) — the locked decision register and the frozen canonical-names list (§4).

---

## 1. How to read this document

This file is the **naming authority all other docs defer to**. Where a term has a canonical string in the canon documents, that string is reproduced here **verbatim** — same case, same kebab/snake convention, same back-ticking.

Columns:

- **Canonical name** — the exact string to use everywhere. Never abbreviate or re-case it.
- **Type** — one of: `entity`, `agent`, `service`, `topic`, `Avro-schema`, `graph-node`, `graph-edge`, `concept`.
- **Definition** — one line. The "what," not the "why."
- **Renamed-from-PDF?** — records an explicit `old → new` rename if one exists in the canon. A dash (`—`) means no rename. Where a term is **wholly new** versus the original NEXUS Execution Plan v2.1 PDF, it is marked **added (SaaS)**; where the PDF made a choice the SaaS posture hardened **without renaming**, it is marked **PDF→SaaS (no rename)**.

> [SaaS-scale adjustment] **The PDF named no services.** It was a 13-phase, role-agnostic, single-deployment plan. Every service below is therefore **added (SaaS)**, not renamed. Likewise, [Tech Decisions §3](_canon/06-tech-decisions.md) records that all PDF→SaaS changes were made *without renaming*. Consequently the canon contains **no `old → new` service or entity rename**, and the Renamed-from-PDF column is dashes throughout, with origin marks. The single cross-document reconciliation that resembles a rename is handled in prose in §10, not as a PDF rename.

## Recorded renames

| Old name | New name | Why |
|---|---|---|
| `nexus-recordings` | `mytheus-recordings` | The R2 buckets were created with the product name; see [ADR-0028](adr/0028-object-storage-buckets-use-the-mytheus-prefix.md). |
| `nexus-documents` | `mytheus-documents` | Same. |
| `nexus-attachments` | `mytheus-attachments` | Same. |
| `nexus-invoices` | `mytheus-invoices` | Same. |
| `nexus-exports` | `mytheus-exports` | Same. |
| `nexus-audit-archive` | `mytheus-audit-archive` | Same. |

> [Naming-authority note] [`_canon/02-data-model.md`](_canon/02-data-model.md) uses informal service labels (`identity-service`, `agent-service`, `tenant-service`, `billing-service`, `graph-service`, `budget-service`, `audit-service`) in its ownership map. These are **not canonical** — the [Service Catalog](04-service-catalog.md) ([`_canon/01`](_canon/01-service-catalog.md)) is the single service-naming authority, and [Event Architecture §2.2](07-event-architecture.md) already ruled this conflict in the catalog's favor. This glossary uses **only the 20 catalog service names**; the data-model labels are mapped to their catalog equivalents in §10 and never used as canonical service rows.

---

## 2. Services (canonical, from the Service Catalog)

All 20 services are kebab-case, named exactly once in [`_canon/01-service-catalog.md` §3](_canon/01-service-catalog.md). None existed in the PDF.

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `api-gateway` | service | Single ingress: TLS termination, WAF, global rate limiting, Clerk JWT pre-validation, `org_id` extraction, routing to `bff`/services. | — · added (SaaS) |
| `web-app` | service | Next.js 14 App Router UI: dashboard, meeting room, agent sidebar, transcript view, RAG context panel, admin/tenant consoles. | — · added (SaaS) |
| `bff` | service | Backend-for-frontend: aggregates/denormalizes reads for screens, fans out to services, shapes payloads for `web-app` only. | — · added (SaaS) |
| `identity-tenant-service` | service | Identity & tenancy control plane: Clerk user-sync, org/tenant provisioning & lifecycle, RBAC, SSO/SAML/SCIM, agent-to-agent authz grants, `get_user_context` source-of-record. | — · added (SaaS) |
| `core-api` | service | Domain CRUD: owns the relational heart — meetings, teams, departments, participants, speakers, worklogs, decisions/action_items read models, chat, agent_suggestions, agent_collaborations. Publishes `decisions`/`actions` via outbox on confirm. | — · added (SaaS) |
| `meeting-service` | service | LiveKit transport adapter: mints room tokens, receives LiveKit webhooks, bridges server-side audio hooks to `transcription-service`. | — · added (SaaS) |
| `realtime-gateway` | service | WebSocket/Socket.io fan-out for non-AV events: chat, presence, live captions, agent sidebar outputs, streamed tokens, priority alert queue. | — · added (SaaS) |
| `transcription-service` | service | Deepgram WebSocket proxy: diarization, interim vs final separation, persists only finals to `transcript_chunks`, emits `transcript-chunks`. | — · added (SaaS) |
| `ingestion-service` | service | Document ingestion via unstructured.io → typed elements, LangChain splitting, PII redaction, emits `document-chunks`. | — · added (SaaS) |
| `embedding-service` | service | Dedicated autoscaling inference for BGE-small-en 384-dim via ONNX; embed-on-ingest + query-embed; upserts per-tenant Qdrant. Off the API tier. | — · added (SaaS) |
| `reranker-service` | service | Dedicated autoscaling inference for the BGE-reranker-base cross-encoder; reranks top-50 → top-10 for `rag-service`. Off the API tier. | — · added (SaaS) |
| `ner-service` | service | Dedicated autoscaling inference for the lightweight NER model behind the PII redaction pass ([ADR-0020](adr/0020-pii-token-vault-and-ner-placement.md)). Detects the unstructured entities (`PERSON`, `ORG`, `LOC`) the deterministic regex layer cannot; it detects only and does not redact, so each boundary keeps its own pass. Off the API tier, and specifically not in `transcription-service`. | — · added (SaaS) |
| `agent-runtime-service` | service | Hosts the six LangGraph agents on `BaseAgent`/`AgentState` + the cost-controlled LLM pipeline + A2A collaboration. Owns no Postgres tables. | — · added (SaaS) |
| `rag-service` | service | `RAGService` pipeline expand → retrieve → rerank → assemble → cite; hybrid retrieval (Qdrant + Postgres FTS + AGE) with RRF fusion; Redis answer cache. | — · added (SaaS) |
| `knowledge-graph-service` | service | Owns the Apache AGE graph `nexus_knowledge`: node/edge labels, Cypher templates, org-partitioned, edges on confirmation. | — · added (SaaS) |
| `search-service` | service | Keyword/lexical retrieval: Postgres FTS over `transcript_chunks` and `chat_messages`; the FTS arm of hybrid retrieval. | — · added (SaaS) |
| `integration-service` | service | External integrations: Google Calendar, Slack, Jira (default)/Linear OAuth + webhooks; owns `integration_connections` and `oauth_tokens`. | — · added (SaaS) |
| `workflow-service` | service | Temporal Cloud workers: `MeetingLifecycleWorkflow`, `ActionFollowUpWorkflow`, tenant-provisioning and right-to-be-forgotten workflows, billing dunning sagas. | — · added (SaaS) |
| `billing-metering-service` | service | SaaS monetization: Stripe billing/subscriptions/seats/dunning/tax, usage metering → quota enforcement, plans/tiers, per-tenant token budgets. | — · added (SaaS) |
| `notification-service` | service | Unified delivery fabric: in-app (via `realtime-gateway`), email, Slack; templating, per-user priority queue, dedupe, delivery receipts. | — · added (SaaS) |
| `admin-ops-service` | service | Two consoles: tenant admin (org-scoped self-serve) and internal ops (cross-tenant support, feature flags, lag/cost dashboards, audit-log access, DLQ replay). | — · added (SaaS) |

---

## 3. The six agents

All six are PDF-original (names verbatim, phases 7–11), hosted by `agent-runtime-service` ([AI Agent Architecture §2](08-ai-agent-architecture.md)).

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `PersonalListenerAgent` | agent | One per participant per meeting; scores each window's relevance to the user via `get_user_context`, produces relevance-scored `worklogs` content via `agent-results`. (P7) | — |
| `DecisionAgent` | agent | Detects decisions, extracts rationale/stakeholders/confidence; emits a *decision candidate* on `agent-results` — never silently commits. (P8) | — |
| `ActionAgent` | agent | Detects action items, extracts `assignee_id`/`due_date`/dependencies/confidence; emits an *action candidate* — confirmed before the `actions` event. (P8) | — |
| `QuestionAgent` | agent | Detects context gaps, proposes role-specific questions with `priority` 1–3 and status Ask/Dismiss/Save-for-later → `agent_suggestions`. (P9) | — |
| `ContextGapAgent` | agent | The cross-meeting piece (R6): asks other users' agents for missing context over `agent-requests`/`agent-responses`; fallback chain peer-agents → related-meeting agents → RAG → silent; logs `agent_collaborations`. (P10) | — |
| `ProactiveAgent` | agent | Three detectors — conflict (via `nexus_knowledge`), workload (open `action_items` per assignee), missing-stakeholder (org-graph) — output on `notifications`. (P11) | — |

---

## 4. Data entities

PDF-original tables (names verbatim) appear first, then the SaaS-scale additions. Source: [`_canon/02-data-model.md` §2.1/§2.2](_canon/02-data-model.md). Owning service is stated using the **canonical catalog name** (see §10 for the data-model-label mapping).

### 4.1 PDF-original entities (16, names carried verbatim)

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `organizations` | entity | The tenant root (`id` = `org_id`); `clerk_org_id`, name, slug, plan_id, `status` (active/suspended). Never partitioned away; deletion cascades a right-to-be-forgotten workflow. Not a tenant-scoped row and deliberately **not** `tenant_isolation`-policied (it carries no `org_id` column). Owned by `identity-tenant-service`. | — |
| `users` | entity | Org member: `clerk_user_id`, `email`, `role`, `skills text[]`, `bio`, `avatar_url`. RLS by `org_id`. Owned by `identity-tenant-service`. | — |
| `teams` | entity | Named team within a department. RLS by `org_id`. Owned by `identity-tenant-service`. | — |
| `team_members` | entity | Join table `(team_id, user_id)` with role. Owned by `identity-tenant-service`. | — |
| `departments` | entity | Self-referential org tree (`parent_id`). Owned by `identity-tenant-service`. | — |
| `meetings` | entity | A meeting: title, scheduled_start/end, `livekit_room`, status, created_by. Monthly range-partition candidate. Owned by `core-api`. | — |
| `meeting_participants` | entity | Join table `(meeting_id, user_id)` with `role_in_meeting`; drives `get_user_context`. Also carries `consented_at timestamptz NULL` / `consent_version text NULL`, the recording/transcription consent acknowledgment ([ADR-0018](adr/0018-consent-entity-behind-flag.md)); both NULL until enforcement is enabled and a participant consents. Owned by `core-api`. | — |
| `pii_tokens` | entity | The per-tenant reversible PII token vault ([ADR-0020](adr/0020-pii-token-vault-and-ner-placement.md)): `token`, `entity_type`, `value_ciphertext` (envelope-encrypted), `value_hmac` (keyed, never a plain digest), `dek_id`. UNIQUE `(org_id, token)` and `(org_id, entity_type, value_hmac)`, the latter being what makes one raw value mint one token across all three redaction boundaries. Owned and written only by `identity-tenant-service`; the redaction boundaries call its mint/resolve API. Ratified but **not yet migrated** - ADR-0020 blocks it on key custody. | — |
| `meeting_rooms` | entity | The LiveKit room ↔ meeting map ([ADR-0003](adr/0003-meeting-sdk-livekit.md)): `livekit_room`, `status` (pending/active/ended), `egress_id`, `recording_ref`, `livekit_metadata_synced_at`. UNIQUE `(org_id, meeting_id)` and `(org_id, livekit_room)`; RLS `FORCE`. One row per meeting. Owned by `meeting-service`: here the catalog service and the data model's `meeting-service` label agree, unlike the split described in §10. | — |
| `transcript_chunks` | entity | Finalized diarized utterances: speaker_id, start_ms, end_ms, text, is_final, confidence. Index on `(meeting_id, start_ms)`; only finals persisted. Owned by `transcription-service`. | — |
| `meeting_speakers` | entity | Diarized `speaker_id` → `user_id` mapping with confidence and `assignment` (manual/suggested) — the R7 source of truth. Owned by `core-api` (the catalog lists it under `core-api`'s owned data; the data model labels its owner `transcription-service` — catalog wins, see §10). | — |
| `worklogs` | entity | Per-user notes (PersonalListenerAgent): content, relevance_score, created_by_agent, source_chunk_ids. Owned by `core-api`. | — |
| `decisions` | entity | Confirmed decisions: content, rationale, confidence, status, confirmed_by, supersedes. On confirm → AGE `Decision`→`Meeting` `PRODUCED` edge; `supersedes` → `FOLLOWS_FROM` (ADR-0031). Owned by `core-api`. | — |
| `action_items` | entity | Confirmed action items: content, assignee_id, due_date, confidence, status. On confirm → AGE `Action`→`Person` `ASSIGNED_TO` edge. Owned by `core-api`. | — |
| `agent_suggestions` | entity | QuestionAgent output: question, `priority` 1–3, `status` `open`/`asked`/`dismissed`/`saved` (Ask/Dismiss/Save-for-later; persists into next meeting, [ADR-0032](adr/0032-plan-8-decision-and-action-candidates.md) D7), `confidence`, `rationale`, `citations`. Private to its `user_id`; the band is applied at read time ([ADR-0034](adr/0034-plan-9-question-agent-suggestions.md)). Owned by `core-api`. | — |
| `agent_collaborations` | entity | ContextGapAgent A2A logs: one row per hop attempt (requester, responder, hop, query, query class, target scope, result, success, `latency_ms`), grants and denials alike; the context fill sits on the row that produced it and is private to the requester ([ADR-0035](adr/0035-plan-10-context-gap-agent-and-a2a-build.md)). Owned by `core-api`. | - |
| `agent_authz_scopes` | entity | The A2A grant store: `grantee_role` (RBAC tier) x `query_class` x `target_scope` x `effect`, deny by default ([ADR-0013](adr/0013-agent-to-agent-authorization.md), [ADR-0035](adr/0035-plan-10-context-gap-agent-and-a2a-build.md)). Owned by `identity-tenant-service`. | - |
| `context_fill` | event kind | `AgentResult.kind` for `ContextGapAgent`: the window's A2A collaboration entries plus the optional fill ([ADR-0035](adr/0035-plan-10-context-gap-agent-and-a2a-build.md)). | - |
| `chat_messages` | entity | In-meeting chat (non-AV via Socket.io). Range-partition by `created_at`. Owned by `core-api`. | — |
| `document_chunks` | entity | unstructured.io typed elements: document_id, element_id, element_type, text, position. Mirrors the Qdrant `document_chunks` payload. Owned by `rag-service`. | — |

### 4.2 SaaS-scale entities (11, wholly added vs PDF)

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `plans` | entity | Global plan catalog: `code` (free/pro/enterprise), seat price, included minutes/tokens (per-tenant monthly pools, not per seat; [ADR-0029](adr/0029-plan-7-budget-scope-tier-mapping-and-metering-ownership.md)), features. Non-tenant. Owned by `billing-metering-service`. | added (SaaS) |
| `subscriptions` | entity | One active subscription per tenant: plan_id, stripe_subscription_id, seats, status, period, trial_ends_at. Owned by `billing-metering-service`. | added (SaaS) |
| `usage_events` | entity | Append-only metered units: event_type (meeting_minute/agent_token/transcription_second/storage_gb), quantity, model_tier (haiku/sonnet/opus, `agent_token` only), idempotency_key. `agent_token` rows come from the `usage-events` topic; the other three are derived inside `billing-metering-service` ([ADR-0029](adr/0029-plan-7-budget-scope-tier-mapping-and-metering-ownership.md)). Owned by `billing-metering-service`. | added (SaaS) |
| `invoices` | entity | Immutable finalized invoices: stripe_invoice_id, amounts, tax, status, pdf_url. Retained 7 years. Owned by `billing-metering-service`. | added (SaaS) |
| `audit_logs` | entity | Append-only, tamper-evident audit trail: actor_id/type, action, target, metadata, ip, occurred_at. Owned by `admin-ops-service` (the catalog folds audit-log access into it — see §10). | added (SaaS) |
| `integration_connections` | entity | One row per installed integration: provider (google_calendar/slack/jira/linear), external_account_id, status, scopes, webhook cursors. Owned by `integration-service`. | added (SaaS) |
| `oauth_tokens` | entity | Per-connection OAuth tokens, encrypted at rest (envelope/KMS); never returned by API. Owned by `integration-service`. | added (SaaS) |
| `tenant_settings` | entity | Per-tenant config: locale, `agent_confidence_overrides`, retention days, pii_redaction_enabled, sso_config, `consent_required boolean NOT NULL DEFAULT false` (gates the meeting-join consent flow, [ADR-0018](adr/0018-consent-entity-behind-flag.md); default off). Owned by `identity-tenant-service` (the tenant control plane; data-model labels this `tenant-service` — see §10). | added (SaaS) |
| `data_residency` | entity | Pins a tenant to region/cell (us/eu/ap), db_cell, qdrant_region, object_bucket_region; immutable after `locked_at`. Owned by `identity-tenant-service` (data-model labels this `tenant-service` — see §10). | added (SaaS) |
| `agent_budgets` | entity | Per (tenant, month) token budget: token_quota, tokens_consumed, hard_cap, 80/95% alert flags, state (ok/throttled/blocked). `token_quota` = the plan's per-tenant `included_agent_tokens`. Enforces R4 budgets. Owned by `billing-metering-service` (data-model labels this `budget-service` — see §10). | added (SaaS) |
| `rbac_grants` | entity | The authorization store: `tier` (owner/admin/member) per (org_id, user_id), plus `granted_by`. The **only** thing the authz middleware reads. Strictly distinct from `users.role`, which is a job/domain attribute with no permission meaning. Owned by `identity-tenant-service`. | added (SaaS) |

---

## 5. Redpanda topics

Kebab-case, Avro + Schema Registry (`{topic}-value`, BACKWARD). 9 PDF-original + 12 SaaS-added = 21. Partitioned by `org_id` alone on 19 of 21 topics; `transcript-chunks` and `agent-stream` use the composite `{org_id}:{meeting_id}` ([ADR-0017](adr/0017-transcript-chunks-composite-partition-key.md)). Source: [Event Architecture §2](07-event-architecture.md), [`_canon/01` §5](_canon/01-service-catalog.md), [`_canon/03`](_canon/03-event-taxonomy.md).

### 5.1 PDF-original topics (9, names verbatim)

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `transcript-chunks` | topic | Finalized diarized utterances from a live meeting. Producer `transcription-service`. Partition key `{org_id}:{meeting_id}` ([ADR-0017](adr/0017-transcript-chunks-composite-partition-key.md)). | — |
| `document-chunks` | topic | Typed, split document elements from ingestion. Producer `ingestion-service`. | — |
| `agent-results` | topic | Agent output (notes, scores, candidates, suggestions). Producer `agent-runtime-service`. | — |
| `decisions` | topic | Confirmed decisions only. Producer `core-api` (outbox, on confirm). | — |
| `actions` | topic | Confirmed action items only. Producer `core-api` (outbox, on confirm). | — |
| `notifications` | topic | Cross-channel notification intents. Multiple producers. | — |
| `agent-requests` | topic | A2A context-fill requests (ContextGapAgent). | — |
| `agent-responses` | topic | A2A context-fill responses. | — |
| `agent-stream` | topic | Streamed LLM token deltas to the UI. Partition key `{org_id}:{meeting_id}` ([ADR-0017](adr/0017-transcript-chunks-composite-partition-key.md)). | — |

### 5.2 SaaS-scale topics (12, added vs PDF)

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `tenant-events` | topic | Tenant provisioned/suspended/deleted. Producer `identity-tenant-service`. Log-compacted. | added (SaaS) |
| `meeting-events` | topic | Meeting created/updated/ended. Producer `core-api`. | added (SaaS) |
| `meeting-room-events` | topic | LiveKit joined/left/recording-ready. Producer `meeting-service`. | added (SaaS) |
| `integration-events` | topic | Calendar→meeting, attachment-to-ingest, external status → action update. Producer `integration-service`. | added (SaaS) |
| `usage-events` | topic | Metered units (tokens/minutes/seconds/storage). Producers `agent-runtime-service`, `rag-service`. | added (SaaS) |
| `billing-events` | topic | Quota-exceeded/dunning/plan-changed. Producer `billing-metering-service`. | added (SaaS) |
| `graph-events` | topic | Edge created / lineage updated. Producer `knowledge-graph-service`. | added (SaaS) |
| `embedding-events` | topic | Upsert acks for lag tracking. Producer `embedding-service`. | added (SaaS) |
| `presence-events` | topic | Socket.io presence join/leave. Producer `realtime-gateway`. | added (SaaS) |
| `notification-delivery-events` | topic | Delivery receipts. Producer `notification-service`. | added (SaaS) |
| `user-sync-events` | topic | Clerk webhook user-sync inbox. Producer `identity-tenant-service`. | added (SaaS) |
| `admin-events` | topic | Flag changed / tenant suspended / erasure requested. Producer `admin-ops-service`. Log-compacted. | added (SaaS) |

---

## 6. Avro value schemas

Only **seven** schema names are frozen canon ([Tech Decisions §2.4/§4](_canon/06-tech-decisions.md)). The remaining schema names are net-new, introduced by [Event Architecture](07-event-architecture.md), and are **not frozen canon** — they must be ratified back into Tech Decisions §4 if they become load-bearing.

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `TranscriptChunk` | Avro-schema | Value of `transcript-chunks`. Frozen canon. | — |
| `AgentResult` | Avro-schema | Value of `agent-results`. Frozen canon. | — |
| `Decision` | Avro-schema | Value of `decisions`. Frozen canon. Optional `supersedes` appended, BACKWARD-safe (ADR-0031 D2). | — |
| `Action` | Avro-schema | Value of `actions`. Frozen canon. | — |
| `Notification` | Avro-schema | Value of `notifications`. Frozen canon. | — |
| `AgentRequest` | Avro-schema | Value of `agent-requests`; carries requester_agent_id, responder_target, query, deadline, correlation_id. Frozen canon. | — |
| `AgentResponse` | Avro-schema | Value of `agent-responses`. Frozen canon. | — |
| `EventEnvelopeHeader` | Avro-schema | Shared header on every event value (event_id, idempotency_key, org_id, schema_version, producer, occurred_at, correlation_id, causation_id). | added (SaaS) — not frozen canon (introduced in 07) |
| `DocumentChunk` | Avro-schema | Value of `document-chunks` (the PDF's "chunk envelope"). | added — not frozen canon (introduced in 07) |
| `AgentStreamDelta` | Avro-schema | Value of `agent-stream` (the PDF's "token-delta envelope"). | added — not frozen canon (introduced in 07) |
| `TenantEvent` · `MeetingEvent` · `MeetingRoomEvent` · `IntegrationEvent` · `UsageEvent` · `BillingEvent` · `GraphEvent` · `EmbeddingEvent` · `PresenceEvent` · `NotificationDeliveryEvent` · `UserSyncEvent` · `AdminEvent` | Avro-schema | Values of the 12 SaaS-scale topics in §5.2, in order. | added (SaaS) — not frozen canon (introduced in 07) |
| `DlqEnvelope` | Avro-schema | Wraps a failed message with replay metadata (original topic/partition/offset, failure_class, raw_value_b64) for every `<topic>.dlq`. | added (SaaS) — not frozen canon (introduced in 07) |

---

## 7. Qdrant collections

384-dim BGE-small-en, cosine. Logical names mirror their relational counterparts; the physical per-tenant collection appends `__{org_id}`. Source: [`_canon/02` §3](_canon/02-data-model.md).

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `transcript_chunks` (Qdrant) | concept | Vector collection over transcript chunks; payload org_id, meeting_id, chunk_id, speaker, start_ms, is_final. Physical `transcript_chunks__{org_id}`. | — (PDF had one shared store; per-tenant split is added) |
| `document_chunks` (Qdrant) | concept | Vector collection over document chunks; payload org_id, document_id, element_id, element_type. Physical `document_chunks__{org_id}`. | — (per-tenant split added) |
| `decisions` (Qdrant) | concept | Vector collection over confirmed decisions; enables ProactiveAgent semantic conflict detection. Physical `decisions__{org_id}`. | — (per-tenant split added) |

---

## 8. Apache AGE graph labels

Graph `nexus_knowledge` (verbatim, PDF). Five node labels and five edge labels, org-partitioned by `$org_id`, edges created on confirmation only. Source: [`_canon/02` §4](_canon/02-data-model.md), [Tech Decisions §2.10](_canon/06-tech-decisions.md).

### 8.1 Node labels (5, verbatim)

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `Person` | graph-node | Backed by `users`; props user_id, org_id, role. | — |
| `Meeting` | graph-node | Backed by `meetings`; props meeting_id, org_id, title. | — |
| `Decision` | graph-node | Backed by `decisions`; props decision_id, org_id, status, confidence. | — |
| `Action` | graph-node | Backed by `action_items`; props action_id, org_id, status. | — |
| `Topic` | graph-node | Role→topics taxonomy; props name, org_id. | — |

### 8.2 Edge labels (5, verbatim)

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `ATTENDED` | graph-edge | `Person` → `Meeting`; created when a participant joins. | — |
| `PRODUCED` | graph-edge | `Decision` → `Meeting` (also `Action` → `Meeting`); created on confirmation. | — |
| `ASSIGNED_TO` | graph-edge | `Action` → `Person`; created when an action is confirmed with an assignee. | — |
| `RELATES_TO` | graph-edge | `Decision`/`Action`/`Meeting` → `Topic`; created on topic extraction. | — |
| `FOLLOWS_FROM` | graph-edge | `Decision` → `Decision`; decision lineage / supersession. | — |

---

## 9. Concepts and key terms

Cross-cutting concepts referenced across the set. PDF-original concepts are dashes; SaaS additions are marked.

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `nexus_knowledge` | concept | The single Apache AGE graph (Cypher-over-Postgres), org-partitioned. | — |
| Cloudflare R2 | concept | The object store behind every `nexus-*` bucket ([ADR-0023](adr/0023-object-storage-cloudflare-r2.md)), addressed over its S3-compatible API, so the "S3-compatible" contract stated throughout the set stays literally true. Chosen for zero egress fees on a read-heavy recordings workload, from a vendor already terminating every request. Residency caveat: R2's **EU jurisdiction** is a hard placement guarantee while `us`/`ap` are location hints, so an `ap` blob pin is best-effort. | — · added (SaaS) |
| `get_user_context(user_id)` | concept | Utility returning role/skills/team/recent meetings/topics; Redis-cached 5-min TTL; per-user personalization source. | — |
| `clerk_org_id` | concept | The unique, immutable mapping from a Clerk organization to our tenant root, held on `organizations`. The edge resolves the token's native Clerk organization claim through it, and that resolution is the authorization checkpoint where org-exists, not-suspended, and not-soft-deleted are enforced. Our `org_id` is never written into Clerk metadata. | added (SaaS) |
| `mytheus_app` / `mytheus_relay` / `mytheus_admin` | concept | The three PostgreSQL roles this program creates. None holds `BYPASSRLS`. `mytheus_app` is the request path; `mytheus_relay` drains `outbox` across tenants through one permissive policy on that one table; `mytheus_admin` owns the tables and runs migrations and RTBF purges, unreachable from request handlers. These three are not the only roles in the database: the `supabase/postgres` image ships `anon`, `authenticated`, `service_role`, `authenticator` and `supabase_admin`, none of which this program uses and one of which (`service_role`) holds `rolbypassrls`. The isolation rule is therefore stated over roles, not over this list ([ADR-0021](adr/0021-explicit-grants-no-schema-wide-default-privileges.md)). | added (SaaS) |
| Tenant-root exemption | concept | `organizations` and `plans` are the two un-tenanted tables. Because `organizations` has no `org_id` column, the canonical RLS policy is not expressible on it and it is deliberately unpolicied, which makes the rule "every table with an `org_id` column carries `tenant_isolation`" exceptionless and machine-checkable in CI. | added (SaaS) |
| RRF (Reciprocal Rank Fusion) | concept | The fusion method combining Qdrant vector + Postgres FTS + AGE topic-relation results in the `RAGService` hybrid retrieval. | — |
| `RAGService` | concept | The fixed five-stage retrieval pipeline expand → retrieve → rerank → assemble → cite. | — |
| `BaseAgent` | concept | Shared LangGraph lifecycle (spawn/process/terminate) for all six agents. | — |
| `AgentState` | concept | Shared dataclass carried through an agent graph (meeting_id, user_id, chunk window, context, retrieved_facts + SaaS tenant/authz/budget fields). | — |
| Confidence thresholds | concept | Surfacing policy: `> 0.7` surface · `0.4–0.7` silent-log to `worklogs` · `< 0.4` drop. | — |
| Windowed batching | concept | Agents run on 30-second speaker-merged windows (`A17`), not per raw `transcript_chunks` row — the core R4 cost fix. | added (SaaS) |
| Relevance pre-gating | concept | A cheap Haiku triage call per window scoring relevance; ~35% pass, the rest dropped before any extraction call. | added (SaaS) |
| Tiered routing (LiteLLM) | concept | Cheapest-capable model routing: Haiku triage/score → Sonnet extract → Opus high-stakes, via the LiteLLM router. Provider is Anthropic only ([ADR-0027](adr/0027-anthropic-only-llm-provider.md)); fast-response tier is Haiku 5.5 (`claude-haiku-5-5`); extraction tier is Sonnet 5.5 (`claude-sonnet-5-5`, [ADR-0029](adr/0029-plan-7-budget-scope-tier-mapping-and-metering-ownership.md)); high-stakes tier mapping to be set. | — (LiteLLM is PDF; per-tier mix is hardened) |
| LLM response cache | concept | Exact-match completion cache at `nexus:{org_id}:llmcache:{prompt_ref}:{input_hash}` (1h): a hit calls no model and meters nothing. The P7 form of the pipeline's semantic cache ([ADR-0030](adr/0030-plan-7-llm-response-cache-and-personal-listener-gating.md)). | added (SaaS) |
| Prompt caching | concept | Semantic + Anthropic prompt caching of stable prefixes (role taxonomy, system prompt, prior-window context). | added (SaaS) |
| Per-tenant token budgets / hard quotas | concept | `agent_budgets`-backed per (tenant, month) caps; 80/95% alerts; deny-by-default at `hard_cap`. PDF had per-user budgets; SaaS adds tenant-level hard caps. | PDF→SaaS (no rename) |
| `MeetingLifecycleWorkflow` | concept | Temporal workflow orchestrating a meeting's lifecycle. | — |
| `ActionFollowUpWorkflow` | concept | Temporal workflow: reminders day 1/3/7, manager escalation via Slack DM at day 10. | — |
| RLS (Row-Level Security) | concept | PostgreSQL policies `USING (org_id = current_setting('app.current_org')::uuid)` — the default isolation backstop for the pooled long tail. | added (SaaS) |
| Cell-based isolation | concept | Dedicated cell (DB/Qdrant) for large/enterprise tenants; the isolation tier above pooled + RLS. | added (SaaS) |
| tenant / org | concept | A customer org. `organizations` is the tenant root (`id` = `org_id`); `org_id UUID` is the tenant key on every tenant-scoped row, the Redpanda partition key, and the Redis/object-storage namespace prefix. | added (SaaS) |
| BACKWARD compatibility | concept | The Schema Registry compatibility mode: a new consumer schema can read data written by the previous producer schema. One subject per topic, `{topic}-value`. | — |
| Transactional outbox / inbox | concept | Producer writes state + outbox row in one transaction; consumer dedupes via an inbox table — the at-least-once → exactly-once-in-effect machinery. | added (SaaS) |
| DLQ (dead-letter queue) | concept | Per-topic `<topic>.dlq` (21 total) for poison messages; 14-day retention; idempotency-preserving replay via `admin-ops-service`. | added (SaaS) |
| Idempotency key | concept | Stable per-logical-fact dedupe key carried in the envelope header and the Redis `nexus:{org_id}:dedupe:{topic}:{consumer_group}:{idempotency_key}` cache (24h). | added (SaaS) |
| SSO / SAML / SCIM | concept | Enterprise identity via Clerk for Enterprise-tier tenants. | added (SaaS) |
| Dunning | concept | Stripe failed-payment recovery flow, run via `billing-metering-service` and Temporal sagas. | added (SaaS) |
| Seats | concept | Per-subscription seat count billed via Stripe. | added (SaaS) |
| Plans / tiers (Free / Pro / Enterprise) | concept | Packaging that gates features and quotas; the `plans` catalog codes. | added (SaaS) |
| Usage metering | concept | `usage-events` → aggregation → quota enforcement; meters tokens/minutes/seconds/storage. | added (SaaS) |
| RTBF (right-to-be-forgotten) | concept | A single Temporal workflow deleting by `org_id`/`user_id` across all six stores (Postgres, Qdrant, AGE, Redis, object storage, audit). | added (SaaS) |

---

## 10. Cross-document service-label reconciliation

These are **not PDF renames** — they are reconciliations between the informal service labels in [`_canon/02-data-model.md`](_canon/02-data-model.md) and the canonical names in the [Service Catalog](04-service-catalog.md). Per the catalog's naming rule and [Event Architecture §2.2](07-event-architecture.md), the **catalog name always wins**. Some data-model labels do not map 1:1 (e.g. the data model's "agent-service" owns `worklogs`/`decisions`/`action_items`, which the catalog assigns to `core-api`; `agent-runtime-service` owns no Postgres tables).

| Data-model label (non-canonical) | Canonical service (use this) | Note |
|---|---|---|
| `identity-service` | `identity-tenant-service` | Identity + tenancy control plane folded into one catalog service. |
| `identity-service` (data model: `teams`/`team_members`/`departments`) | `core-api` | Per [`_canon/01-service-catalog.md`](_canon/01-service-catalog.md), `core-api` owns `teams`, `team_members`, and `departments` CRUD and their `/teams` / `/departments` prefixes. `identity-tenant-service` retains the identity and tenancy surface (`organizations`, `users`, RBAC, SSO/SCIM) and does not write these three tables. |
| `tenant-service` | `identity-tenant-service` | `tenant_settings` / `data_residency` are owned by the identity/tenancy control plane. |
| `meeting-service` (data model: meetings/participants/speakers/chat) | `core-api` (meetings, participants, chat, `meeting_speakers`) + `transcription-service` (`transcript_chunks`) + `meeting-service` (LiveKit transport adapter only) | The catalog splits the data-model "meeting-service" across three services; `meeting_speakers` is in `core-api`'s owned-data list and `meeting-service` proper is the LiveKit adapter. |
| `agent-service` | `core-api` | `worklogs`, `decisions`, `action_items`, `agent_suggestions`, `agent_collaborations` are owned by `core-api`; `agent-runtime-service` owns no tables. |
| `rag-service` (data model) | `rag-service` | Same name; `document_chunks` ownership matches. |
| `graph-service` | `knowledge-graph-service` | Owns `nexus_knowledge`. |
| `billing-service` | `billing-metering-service` | Owns plans/subscriptions/usage_events/invoices. |
| `budget-service` | `billing-metering-service` | `agent_budgets` is owned by the billing/metering service in the catalog. |
| `audit-service` | `admin-ops-service` | The catalog folds audit-log access ("ops audit log", "audit-log access") into `admin-ops-service`; there is no `audit-service` in the catalog. |

> [Naming-authority note] No alternative names for any term in this glossary are permitted anywhere in the architecture set. Any future rename must land here as an explicit `old → new` row **and** supersede the relevant row in [`_canon/06-tech-decisions.md` §4](_canon/06-tech-decisions.md) via an [ADR](adr/).
