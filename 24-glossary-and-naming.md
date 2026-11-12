# Glossary & Naming Authority

This is the canonical glossary and naming reference for the NEXUS architecture set: every important term — service, entity, agent, topic, Avro schema, graph label, and concept — defined once, with its canonical spelling and any recorded `old → new` rename, so that no other document, ADR, or diagram introduces an alternative name.

## Related documents

- [Service Catalog](04-service-catalog.md) — the single naming authority for services; this glossary's service rows defer to it.
- [Data Architecture](05-data-architecture.md) — store selection, retention, and partitioning for the entities defined here.
- [Event-Driven Architecture](07-event-architecture.md) — Redpanda topics, Avro envelopes, and the topic/schema reconciliation notes this glossary cites.
- [AI Agent Architecture](08-ai-agent-architecture.md) — the seven agents, `BaseAgent`/`AgentState`, and the cost-controlled pipeline.
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
| `agent-runtime-service` | service | Hosts the seven LangGraph agents on `BaseAgent`/`AgentState` + the cost-controlled LLM pipeline + A2A collaboration. Owns no Postgres tables. | — · added (SaaS) |
| `rag-service` | service | `RAGService` pipeline expand → retrieve → rerank → assemble → cite; hybrid retrieval (Qdrant + Postgres FTS + AGE) with RRF fusion; Redis answer cache. | — · added (SaaS) |
| `knowledge-graph-service` | service | Owns the Apache AGE graph `nexus_knowledge`: node/edge labels, Cypher templates, org-partitioned, edges on confirmation. | — · added (SaaS) |
| `search-service` | service | Keyword/lexical retrieval: Postgres FTS over `transcript_chunks` and `chat_messages`; the FTS arm of hybrid retrieval. | — · added (SaaS) |
| `integration-service` | service | External integrations: Google Calendar, Slack, Jira (default)/Linear OAuth + webhooks; owns `integration_connections` and `oauth_tokens`. | — · added (SaaS) |
| `workflow-service` | service | Temporal Cloud workers: `MeetingLifecycleWorkflow`, `ActionFollowUpWorkflow`, tenant-provisioning and right-to-be-forgotten workflows, billing dunning sagas. | — · added (SaaS) |
| `billing-metering-service` | service | SaaS monetization: Stripe billing/subscriptions/seats/dunning/tax, usage metering → quota enforcement, plans/tiers, per-tenant token budgets. | — · added (SaaS) |
| `notification-service` | service | Unified delivery fabric: in-app (via `realtime-gateway`), email, Slack; templating, per-user priority queue, dedupe, delivery receipts. | — · added (SaaS) |
| `admin-ops-service` | service | Two consoles: tenant admin (org-scoped self-serve) and internal ops (cross-tenant support, feature flags, lag/cost dashboards, audit-log access, DLQ replay). | — · added (SaaS) |

---

## 3. The seven agents

The first six are PDF-original (names verbatim, phases 7 to 11); the seventh, `MeetingAgent`, is a SaaS addition ([ADR-0038](adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md)). All are hosted by `agent-runtime-service` ([AI Agent Architecture §2](08-ai-agent-architecture.md)).

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `PersonalListenerAgent` | agent | One per participant per meeting; scores each window's relevance to the user via `get_user_context`, produces relevance-scored `worklogs` content via `agent-results`; also returns the participant's own commitments in `payload_json.commitments`, and runs on every window for a participant whose attendance is `agent` ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) C3, C6). (P7) | — |
| `DecisionAgent` | agent | Detects decisions, extracts rationale/stakeholders/confidence; emits a *decision candidate* on `agent-results` — never silently commits. (P8) | — |
| `ActionAgent` | agent | Detects action items, extracts `assignee_id`/`due_date`/dependencies/confidence; emits an *action candidate* — confirmed before the `actions` event. (P8) | — |
| `QuestionAgent` | agent | Detects context gaps, proposes role-specific questions with `priority` 1–3 and status Ask/Dismiss/Save-for-later → `agent_suggestions`. (P9) | — |
| `ContextGapAgent` | agent | The cross-meeting piece (R6): asks other users' agents for missing context over `agent-requests`/`agent-responses`; fallback chain peer-agents → related-meeting agents → RAG → silent; logs `agent_collaborations`. (P10) | — |
| `ProactiveAgent` | agent | Three detectors — conflict (via `nexus_knowledge`), workload (open `action_items` per assignee), missing-stakeholder (org-graph) — output on `notifications`. Two window checks added by [ADR-0039](adr/0039-company-brain-and-meeting-companions.md) C7: `already_decided` and `topic_watch`. (P11) | — |
| `MeetingAgent` | agent | One per meeting (not per participant), only when the meeting has an agenda; tracks agenda items and nudges the host. Emits at most one `meeting_update` per meeting per window on `agent-results` (`user_id` = the host); never writes Postgres. Fast tier. ([ADR-0038](adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md) D7) Sees each item's `goal_id` and sends the deterministic `proxy_update` nudge ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) C5). | added (SaaS) |

---

## 4. Data entities

PDF-original tables (names verbatim) appear first, then the SaaS-scale additions. Source: [`_canon/02-data-model.md` §2.1/§2.2](_canon/02-data-model.md). Owning service is stated using the **canonical catalog name** (see §10 for the data-model-label mapping).

### 4.1 PDF-original entities (16, names carried verbatim)

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `organizations` | entity | The tenant root (`id` = `org_id`); `clerk_org_id`, name, slug, plan_id, `status` (active/suspended). Never partitioned away; deletion cascades a right-to-be-forgotten workflow. Not a tenant-scoped row and deliberately **not** `tenant_isolation`-policied (it carries no `org_id` column). Owned by `identity-tenant-service`. | — |
| `users` | entity | Org member: `clerk_user_id`, `email`, `role`, `skills text[]`, `bio`, `avatar_url`, plus the working profile (`listen_for`, `ignore_topics`) and the agent persona (`agent_name`, `agent_avatar_seed`, `agent_personality`, `agent_instructions`), added by [ADR-0038](adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md). RLS by `org_id`. Owned by `identity-tenant-service`. | — |
| `teams` | entity | Named team within a department. RLS by `org_id`. Owned by `identity-tenant-service`. | — |
| `team_members` | entity | Join table `(team_id, user_id)` with role. Owned by `identity-tenant-service`. | — |
| `departments` | entity | Self-referential org tree (`parent_id`). Owned by `identity-tenant-service`. | — |
| `meetings` | entity | A meeting: title, scheduled_start/end, `livekit_room`, status, created_by, and `previous_meeting_id` (the follow-up link, [ADR-0038](adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md)), plus `template_id` and `confidential` ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B1, B2). The host is the one participant with `role_in_meeting = 'host'`. Monthly range-partition candidate. Owned by `core-api`. | — |
| `meeting_participants` | entity | Join table `(meeting_id, user_id)` with `role_in_meeting`; drives `get_user_context`. Also carries `consented_at timestamptz NULL` / `consent_version text NULL`, the recording/transcription consent acknowledgment ([ADR-0018](adr/0018-consent-entity-behind-flag.md)); both NULL until enforcement is enabled and a participant consents. Owned by `core-api`. Also `attendance` (`live`/`agent`), `proxy_update` and `proxy_questions`, the send-your-agent columns ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B5). | — |
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
| `notification_log` | entity | One row per delivered in-app item, including `ProactiveAgent` alerts; status `unread`/`read`/`dismissed`; private to its user ([ADR-0036](adr/0036-plan-11-proactive-agent-and-delivery-build.md)). Owned by `notification-service`. | - |
| `notification_preferences` | entity | Per-user channel prefs (`in_app`/`email`/`slack`, enabled, `max_priority`). Owned by `notification-service`. | - |
| `integration_id_map` | entity | `integration-service`'s external to NEXUS id map with `last_synced_status` for loop suppression ([ADR-0037](adr/0037-plan-12-integrations-follow-ups-and-consoles-build.md)). | - |
| `integration_cursors` | entity | `integration-service`'s sync cursors and push-channel state. | - |
| `ops_audit_log` | entity | Append-only record of every internal-ops console call. Owned by `admin-ops-service`. | - |
| `support_grants` | entity | Time-boxed, tenant-granted support access. Owned by `admin-ops-service`. | - |
| `context_fill` | event kind | `AgentResult.kind` for `ContextGapAgent`: the window's A2A collaboration entries plus the optional fill ([ADR-0035](adr/0035-plan-10-context-gap-agent-and-a2a-build.md)). | - |
| `meeting_update` | event kind | `AgentResult.kind` for `MeetingAgent`: one record per meeting per window batching the agenda `updates` (`item_id`, `status`, `note`) and an optional host `nudge` (`off_agenda`/`over_time`/`unaddressed`, plus the deterministic `proxy_update` from [ADR-0039](adr/0039-company-brain-and-meeting-companions.md) C5). `core-api` applies the updates to `meeting_agenda_items`; `realtime-gateway` splits the record into `agent-agenda` and `agent-nudge` ([ADR-0038](adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md) D7 to D9). | added (SaaS) |
| `chat_messages` | entity | In-meeting chat (non-AV via Socket.io). Range-partition by `created_at`. Owned by `core-api`. | — |
| `document_chunks` | entity | unstructured.io typed elements: document_id, element_id, element_type, text, position. Mirrors the Qdrant `document_chunks` payload. Owned by `rag-service`. | — |

### 4.2 SaaS-scale entities (38, wholly added vs PDF)

| Canonical name | Type | Definition | Renamed-from-PDF? |
|---|---|---|---|
| `plans` | entity | Global plan catalog: `code` (free/pro/enterprise), seat price, included minutes/tokens (per-tenant monthly pools, not per seat; [ADR-0029](adr/0029-plan-7-budget-scope-tier-mapping-and-metering-ownership.md)), features. Non-tenant. Owned by `billing-metering-service`. | added (SaaS) |
| `subscriptions` | entity | One active subscription per tenant: plan_id, stripe_subscription_id, seats, status, period, trial_ends_at. Owned by `billing-metering-service`. | added (SaaS) |
| `usage_events` | entity | Append-only metered units: event_type (meeting_minute/agent_token/transcription_second/storage_gb), quantity, model_tier (haiku/sonnet/opus, `agent_token` only), idempotency_key. `agent_token` rows come from the `usage-events` topic; the other three are derived inside `billing-metering-service` ([ADR-0029](adr/0029-plan-7-budget-scope-tier-mapping-and-metering-ownership.md)). Owned by `billing-metering-service`. | added (SaaS) |
| `invoices` | entity | Immutable finalized invoices: stripe_invoice_id, amounts, tax, status, pdf_url. Retained 7 years. Owned by `billing-metering-service`. | added (SaaS) |
| `audit_logs` | entity | Append-only, tamper-evident audit trail: actor_id/type, action, target, metadata, ip, occurred_at. Owned by `admin-ops-service` (the catalog folds audit-log access into it — see §10). | added (SaaS) |
| `integration_connections` | entity | One row per installed integration: provider (google_calendar/slack/jira/linear), external_account_id, status, scopes, webhook cursors. Owned by `integration-service`. | added (SaaS) |
| `oauth_tokens` | entity | Per-connection OAuth tokens, encrypted at rest (envelope/KMS); never returned by API. Owned by `integration-service`. | added (SaaS) |
| `tenant_settings` | entity | Per-tenant config: locale, `agent_confidence_overrides`, retention days, pii_redaction_enabled, sso_config, `consent_required boolean NOT NULL DEFAULT false` (gates the meeting-join consent flow, [ADR-0018](adr/0018-consent-entity-behind-flag.md); default off). Owned by `identity-tenant-service` (the tenant control plane; data-model labels this `tenant-service` — see §10). Also `never_record_topics text[]` (the ground rules, [ADR-0039](adr/0039-company-brain-and-meeting-companions.md) A5). | added (SaaS) |
| `data_residency` | entity | Pins a tenant to region/cell (us/eu/ap), db_cell, qdrant_region, object_bucket_region; immutable after `locked_at`. Owned by `identity-tenant-service` (data-model labels this `tenant-service` — see §10). | added (SaaS) |
| `agent_budgets` | entity | Per (tenant, month) token budget: token_quota, tokens_consumed, hard_cap, 80/95% alert flags, state (ok/throttled/blocked). `token_quota` = the plan's per-tenant `included_agent_tokens`. Enforces R4 budgets. Owned by `billing-metering-service` (data-model labels this `budget-service` — see §10). | added (SaaS) |
| `rbac_grants` | entity | The authorization store: `tier` (owner/admin/member) per (org_id, user_id), plus `granted_by`. The **only** thing the authz middleware reads. Strictly distinct from `users.role`, which is a job/domain attribute with no permission meaning. Owned by `identity-tenant-service`. | added (SaaS) |
| `meeting_agenda_items` | entity | A meeting's agenda, at most 20 items: `position`, `title`, `owner_user_id`, `duration_minutes`, `status` (`pending`/`in_progress`/`covered`/`skipped`), `status_source` (`agent`/`host`), `agent_note`. RLS by `org_id`. Owned by `core-api` (the data model labels it `meeting-service`, like `meetings`; see §10) ([ADR-0038](adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md) D5). Also `goal_id` (an `org_goals` id, [ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B3). | added (SaaS) |
| `org_profiles` | entity | The company profile, one row per org: `description`, `industry`, `products[]`, `customers[]`. Part of the company brain. Owned by `identity-tenant-service` ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) A1). | added (SaaS) |
| `org_glossary_terms` | entity | The company vocabulary: `term`, `kind` (`product`/`codename`/`acronym`/`customer`/`person`/`other`), `definition`, `aliases[]`; at most 500 per org. Not to be confused with this Glossary document. Owned by `identity-tenant-service` ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) A2). | added (SaaS) |
| `topic_owners` | entity | Who owns what: a free-text `topic` (not an AGE `Topic` node) and its `owner_user_id`; at most 200 per org. Owned by `identity-tenant-service` ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) A3). | added (SaaS) |
| `org_goals` | entity | The org's goals: `title`, `description`, `period`, `status` (`active`/`achieved`/`dropped`), `owner_user_id`, `position`; at most 30 active. Owned by `identity-tenant-service` ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) A4). | added (SaaS) |
| `topic_watches` | entity | A person's watched topics, at most 20, self only; source of `topic_watch` alerts. Owned by `identity-tenant-service` ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) A6). | added (SaaS) |
| `meeting_templates` | entity | Meeting types: `name`, `description`, `default_duration_minutes`, `agenda jsonb`, `confidential`; at most 50 per org. Owned by `core-api` (the data model labels it `meeting-service`; see §10) ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B1). | added (SaaS) |
| `commitments` | entity | Promises: what a participant said they would do (`content`, `due_hint`, `status` `open`/`done`/`dismissed`), from `PersonalListenerAgent` `payload_json.commitments`, private to the person. Owned by `core-api` (data-model label `agent-service`) ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B6). | added (SaaS) |
| `action_status_requests` | entity | Status requests: one per open confirmed action of the previous meeting, asked of the assignee 60 minutes before the follow-up (`state` `pending`/`answered`/`expired`, `status_text`). Owned by `core-api` (data-model label `agent-service`) ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B7). | added (SaaS) |
| `api_tokens` | entity | Personal access tokens for the public API and the MCP server: `name`, `token_hash` (SHA-256), `token_hint`, `scope` (`read`/`read_write`), mandatory `expires_at`. Owned by `identity-tenant-service` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `share_links` | entity | Signed, expiring, revocable public links to a meeting recap or a decision record, with a view count. Owned by `core-api` (data-model label `meeting-service`) ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `meeting_followups` | entity | The host's follow-up email for a meeting: guests who are not users, the recap link, the last dispatch, and `awaiting_minutes_since`. Owned by `core-api` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `email_deliveries` | entity | One row per email `notification-service` sent (`meeting_followup`, `weekly_digest`), unique per idempotency key. Owned by `notification-service` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `push_devices` | entity | Expo push tokens of signed-in phones. Owned by `notification-service` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `agent_feedback` | entity | A person's confirm, dismiss, edit, ask or save of something an agent proposed; written with the `agent-feedback` outbox row. Owned by `core-api` (data-model label `agent-service`) ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `agent_calibration` | entity | Decayed confirm and dismiss counts per agent, per workspace and per person, from which the surface threshold is computed (bounded 0.6 to 0.85). Owned by `core-api` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `person_memory` | entity | What a person cares about across meetings: weighted topics, recurring questions, muted topics and `learning_enabled`; private to the person. Owned by `core-api` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `meeting_minutes` | entity | Minutes after a meeting: one private row per participant and one shared row (`user_id` NULL). Owned by `core-api` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `meeting_off_record_segments` | entity | The stretches of a meeting that were off the record. Owned by `core-api` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `decision_reversals` | entity | An open action whose decision was superseded, and its owner's choice (`kept`/`rescoped`/`closed`). Owned by `core-api` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `delegated_answer_statements` | entity | Answers a person who sent their agent pre-approved for one meeting (at most 10, each up to 300 chars). Owned by `core-api` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `delegated_answer_log` | entity | Every answer, or refusal, that person's agent gave in the meeting. Owned by `core-api` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `promise_drift_nudges` | entity | Nudges about a promise whose Jira issue is still open after its due date: owner first, host two days later. Owned by `core-api` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `weekly_digests` | entity | One "what changed for you" digest per person per week. Owned by `core-api` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `meeting_recordings` | entity | One row per LiveKit egress segment of a meeting's recording: `status`, `object_key` (never a URL), timing on the meeting clock. Owned by `meeting-service` ([ADR-0040](adr/0040-polish-program.md) C5). | added (SaaS) |
| `meeting_clips` | entity | A named span of a meeting (at most 10 minutes) on its clock, made by a participant; shareable as a `meeting_clip` link. Owned by `core-api` ([ADR-0040](adr/0040-polish-program.md) C5). | added (SaaS) |
| `digest_preferences` | entity | The weekly digest toggle and its email opt-in, per person. Owned by `core-api` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |

---

## 5. Redpanda topics

Kebab-case, Avro + Schema Registry (`{topic}-value`, BACKWARD). 9 PDF-original + 13 SaaS-added = 22. Partitioned by `org_id` alone on 20 of 22 topics; `transcript-chunks` and `agent-stream` use the composite `{org_id}:{meeting_id}` ([ADR-0017](adr/0017-transcript-chunks-composite-partition-key.md)). Source: [Event Architecture §2](07-event-architecture.md), [`_canon/01` §5](_canon/01-service-catalog.md), [`_canon/03`](_canon/03-event-taxonomy.md).

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
| `agent-stream` | topic | Streamed LLM token deltas to the UI, and live caption translations (`kind` `caption_translation`, [ADR-0040](adr/0040-polish-program.md)). Partition key `{org_id}:{meeting_id}` ([ADR-0017](adr/0017-transcript-chunks-composite-partition-key.md)). | — |

### 5.2 SaaS-scale topics (13, added vs PDF)

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
| `admin-events` | topic | Flag changed / tenant suspended / erasure requested / erasure completed (one per store, from `transcription-service`, `embedding-service`, `search-service`, `knowledge-graph-service`). Producer `admin-ops-service`. Retained 365 days with delete cleanup since [ADR-0040](adr/0040-polish-program.md) (was log-compacted). | added (SaaS) |
| `agent-feedback` | topic | A person's confirm / dismiss / edit / ask / save of something an agent proposed. Producer `core-api` (outbox); consumers `agent-runtime-service`, `core-api`. 12 partitions, 7 days ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |

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
| `AgentFeedback` | Avro-schema | Value of `agent-feedback` ({A}). | added — not frozen canon |
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
| `get_user_context(user_id)` | concept | Utility returning role/skills/team/recent meetings/topics, plus `listen_for`, `ignore_topics`, `agent_name`, `agent_personality`, `agent_instructions` ([ADR-0038](adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md) D4); Redis-cached 5-min TTL; per-user personalization source. | — |
| `clerk_org_id` | concept | The unique, immutable mapping from a Clerk organization to our tenant root, held on `organizations`. The edge resolves the token's native Clerk organization claim through it, and that resolution is the authorization checkpoint where org-exists, not-suspended, and not-soft-deleted are enforced. Our `org_id` is never written into Clerk metadata. | added (SaaS) |
| `mytheus_app` / `mytheus_relay` / `mytheus_admin` | concept | The three PostgreSQL roles this program creates. None holds `BYPASSRLS`. `mytheus_app` is the request path; `mytheus_relay` drains `outbox` across tenants through one permissive policy on that one table; `mytheus_admin` owns the tables and runs migrations and RTBF purges, unreachable from request handlers. These three are not the only roles in the database: the `supabase/postgres` image ships `anon`, `authenticated`, `service_role`, `authenticator` and `supabase_admin`, none of which this program uses and one of which (`service_role`) holds `rolbypassrls`. The isolation rule is therefore stated over roles, not over this list ([ADR-0021](adr/0021-explicit-grants-no-schema-wide-default-privileges.md)). | added (SaaS) |
| Tenant-root exemption | concept | `organizations` and `plans` are the two un-tenanted tables. Because `organizations` has no `org_id` column, the canonical RLS policy is not expressible on it and it is deliberately unpolicied, which makes the rule "every table with an `org_id` column carries `tenant_isolation`" exceptionless and machine-checkable in CI. | added (SaaS) |
| RRF (Reciprocal Rank Fusion) | concept | The fusion method combining Qdrant vector + Postgres FTS + AGE topic-relation results in the `RAGService` hybrid retrieval. | — |
| `RAGService` | concept | The fixed five-stage retrieval pipeline expand → retrieve → rerank → assemble → cite. | — |
| `BaseAgent` | concept | Shared LangGraph lifecycle (spawn/process/terminate) for all seven agents. | — |
| Myte / Mytes | concept | The user-facing product name for a person's personal AI agent (plural Mytes, capital M): "your Myte", "Samar's Myte", default display name "{first name}'s Myte". Copy only: the code and canon identifiers (`PersonalListenerAgent` and the other agents, `agent_name` and the `agent_*` fields, `/agent`, `agent-results`, `attendance = 'agent'`) are unchanged, and it is not a rename. The room-level `MeetingAgent` is not a Myte ([ADR-0042](adr/0042-myte-user-facing-name.md)). | added (SaaS) |
| Agent persona | concept | A person's agent's display name (`agent_name`), doodle face (`agent_avatar_seed`), personality preset (`agent_personality`: `balanced`/`concise`/`detailed`/`direct`/`friendly`) and free-text instructions (`agent_instructions`). Changes only how notes and suggestions are worded; a display label, never an agent class or an event identifier ([ADR-0038](adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md) D2). | added (SaaS) |
| Working profile | concept | `role`, "what you work on" (`bio`), focus areas (`skills`), listen for (`listen_for`) and ignore (`ignore_topics`): what a person's `PersonalListenerAgent` listens for. Asked during onboarding ("About you") ([ADR-0038](adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md) D1). | added (SaaS) |
| Host | concept | The one participant with `meeting_participants.role_in_meeting = 'host'`; default the creator. Edits the agenda, overrides agent statuses, and alone receives `MeetingAgent` nudges ([ADR-0038](adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md) D5). | added (SaaS) |
| Follow-up link / carry-forward | concept | `meetings.previous_meeting_id` links a meeting to the one it follows up; carry-forward is what that meeting left open (confirmed decisions, open actions, the caller's saved questions), served by `GET /api/v1/meetings/{id}/carry-forward`. No recurring-series entity ([ADR-0038](adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md) D5, D6). | added (SaaS) |
| `agent-agenda` / `agent-nudge` | concept | The Socket.io events `realtime-gateway` derives from a `meeting_update`: each agenda update to every socket on the meeting channel, a nudge to the host's sockets only; nudges are not replayed ([ADR-0038](adr/0038-agent-personas-profiles-agendas-and-meeting-agent.md) D9). | added (SaaS) |
| Company brain | concept | What every agent knows about the company: the company profile (`org_profiles`), vocabulary (`org_glossary_terms`), who owns what (`topic_owners`), goals (`org_goals`) and ground rules (`tenant_settings.never_record_topics`), managed in Workspace admin by an org `owner` or `admin` and read by services through `GetOrgContext` ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) Part A). | added (SaaS) |
| `GetOrgContext` | concept | The internal gRPC method `IdentityService.GetOrgContext(GetOrgContextRequest) returns (OrgContext)` on `identity-tenant-service` (`org_id` in metadata): the company brain plus every topic watch, active goals only. Cached at `nexus:{org_id}:org_ctx` for 5 minutes, evicted by every company-brain write. Callers `agent-runtime-service` and `transcription-service` ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) A9). | added (SaaS) |
| People structure | concept | Departments and teams with what each does (`departments.description`, `teams.description`, up to 1000 characters), edited on the People page by an org `owner` or `admin` through `core-api`, and carried to agents in `OrgContext.departments` (`OrgDepartment`) and `OrgContext.teams` (`OrgTeam`) of `GetOrgContext`, evicted by every structure write ([ADR-0041](adr/0041-people-structure.md)). | added (SaaS) |
| Vocabulary / key terms | concept | The company vocabulary (`org_glossary_terms`). Its terms and aliases, with the profile's products and customers, are passed to Deepgram Nova-3 as up to 100 `keyterm`s when a meeting's stream opens, and up to 60 terms with definitions go into agent prompts ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) C1, D1). | added (SaaS) |
| Topic owners (who owns what) | concept | `topic_owners`: a topic and the person who owns it. `ActionAgent` may suggest that person as an action's owner (`suggested_owner_user_id`) ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) A3, C4). | added (SaaS) |
| Goals | concept | `org_goals`. An agenda item may carry a `goal_id`, and active goals go into agent prompts ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) A4, B3). | added (SaaS) |
| Ground rules / never-record topics | concept | `tenant_settings.never_record_topics`: a window mainly about one produces no note, suggestion, candidate or agenda note. A model instruction, not a guarantee ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) A5, C2). | added (SaaS) |
| Readiness | concept | `GET /api/v1/org/readiness`: a 0 to 100 score over six equal checks (`profile`, `vocabulary`, `owners`, `goals`, `ground_rules`, `roles`) shown as the company-brain readiness meter. Not SOC2 readiness and not a Kubernetes readiness probe ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) A7). | added (SaaS) |
| Welcome (new joiner brief) | concept | `GET /api/v1/org/welcome` and the `/welcome` page: profile, active goals, owners, the first 40 vocabulary terms and your team mates, offered on the dashboard for 14 days after joining ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) A8). | added (SaaS) |
| Topic watch | concept | A person's watched topic (`topic_watches`). When a non-confidential meeting the watcher is not in discusses it, `ProactiveAgent` sends a `topic_watch` alert if the A2A PDP allows `meeting_context` on that meeting ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) A6, C7). | added (SaaS) |
| Meeting templates (meeting types) | concept | `meeting_templates`: a named type that fills a new meeting's length, agenda and `confidential` default. Distinct from prompt templates and Cypher templates, and from the proposed consent-exempt meeting type in [Compliance §3.3](12-compliance-and-privacy.md) ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B1). | added (SaaS) |
| Confidential meeting | concept | `meetings.confidential = true`: never a source for topic watch, the `ContextGapAgent` `related_meeting` hop, or carry-forward shown to non-participants ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B2). | added (SaaS) |
| Drafted agenda | concept | `POST /api/v1/meetings/agenda-draft`: a deterministic agenda from a template's items, the previous meeting's unfinished items and its open actions (`source` `template`/`unfinished`/`open_action`); nothing is saved ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B4). | added (SaaS) |
| Attendance / send your agent | concept | `meeting_participants.attendance` (`live`/`agent`). With `agent` the person does not join; their agent carries `proxy_update` (shown to the room) and `proxy_questions` (to the person and the host), gets a `PersonalListenerAgent` pass on every window, and the host gets a `proxy_update` nudge when their item starts ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B5, C5, C6). | added (SaaS) |
| Commitments / promises | concept | `commitments`: what a participant said they would do, captured by `PersonalListenerAgent` and listed at `/promises`. Only commitments the person made, private to them ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B6, C3). | added (SaaS) |
| Status requests | concept | `action_status_requests`: before a follow-up meeting the assignee of each open action is asked for a quick status; the answer shows as `latest_status` on carry-forward ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B7, B9). | added (SaaS) |
| Brief | concept | The pre-meeting brief, `GET /bff/meetings/{id}/brief`: your agenda items, your open actions with their latest status, your saved questions, and proxies' updates; reminded 10 minutes before start. Distinct from the Slack prep briefs `integration-service` sends ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B8, B9). | added (SaaS) |
| Health card | concept | `GET /bff/meetings/{id}/health` after a meeting ends: duration, people-hours, agenda coverage, decisions, actions, talk balance (null when no speaker timing is stored), an async suggestion and a trend over up to 6 earlier meetings in the follow-up chain. Not a service health check ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B8). | added (SaaS) |
| Catch-up | concept | `GET /bff/meetings/{id}/catch-up` and `/meetings/{id}/catch-up`: what an absent or proxied participant missed (agenda, decisions, what was assigned to them, their notes, their questions); reminded 5 minutes after the end ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B8, B9). | added (SaaS) |
| `already_decided` / `topic_watch` | concept | The two `ProactiveAgent` alert types added by [ADR-0039](adr/0039-company-brain-and-meeting-companions.md) C7, carried in its existing `kind = alert` `payload_json` (no Avro change). | added (SaaS) |
| Moment (`moment_id`) | concept | The words a card is about: `uuid5` over the meeting and the quoted transcript final's `chunk_id`, shared by every agent's item about the same words, so one sentence is one card ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| In-meeting ledger | concept | `nexus:{org_id}:meeting-ledger:{meeting_id}`: the open candidates and the room's confirmations, edits and dismissals the agents read, so they update a card instead of repeating it ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Shared candidate ledger | concept | A confirm, edit or dismiss broadcast to the whole room by `realtime-gateway` after it re-reads the stored row, with a 90-second edit lock; a dismiss is personal unless the host's ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| One threshold source | concept | `GET /internal/agent-policy`: the thresholds and calibration that both the agent runtime and `core-api` reloads use, so live and reload agree ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Calibration | concept | A per-workspace and per-person surface threshold from recent confirms and dismisses, bounded 0.6 to 0.85 ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Person memory / learning | concept | What a person's agent remembers across meetings (`person_memory`), shown under "What your agent remembers"; learning can be turned off or forgotten ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Budget unit | concept | One budget unit is the list price of one uncached Sonnet input token; usage is weighed by tier price, cache writes 1.25x, reads 0.1x. The names `agent_token`, `tokens_consumed` are unchanged ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Minutes | concept | After a meeting, one private set per participant and one shared set (`meeting_minutes`), produced by `MeetingAgent` as `agent-results` kind `minutes` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Recap / share link | concept | A public, signed, expiring page of the shared minutes or a decision record at `/s/{token}` (`share_links`) ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Personal access token (PAT) | concept | `mytheus_pat_<org hex>_<secret>`: a bearer for the public API and the MCP server (`api_tokens`) ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| MCP server | concept | `POST /mcp` on `bff`: tools that ask and act through the public API as the caller ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Capture | concept | Audio from a Meet, Teams or Zoom tab (the Chrome extension) or a phone in a room, streamed to `transcription-service` `/capture/audio`; the meeting's `source` says where it came from ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Off the record | concept | A stretch of a meeting with no transcript, no agents and no index, enforced by `transcription-service`, `agent-runtime-service`, `embedding-service` and `search-service` (`meeting_off_record_segments`) ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Consent receipt | concept | `GET /api/meetings/{id}/consent-receipt`: what a person agreed to for a meeting, clause by clause ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Delegated answers | concept | An absent person's agent answering in the room only from statements they pre-approved, logged for them ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Decision reversal | concept | When a decision is superseded, the owner of each linked open action is asked to keep, re-scope or close it ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Promise drift | concept | A promise whose Jira issue is still open after its due date; the owner is nudged first, the host two days later ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Weekly digest | concept | One "what changed for you" summary per person per week (`weekly_digests`) ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Should this be a meeting? (precheck) | concept | `POST /bff/meetings/precheck`: already-decided matches and status updates that would cover the agenda, shown while scheduling ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Recording playback / clips | concept | Transcript-synced playback of a meeting's recording through 600-second presigned URLs, and clips that can be shared with their own cut video ([ADR-0040](adr/0040-polish-program.md) C5). | added (SaaS) |
| Live translation | concept | Caption finals translated on the triage tier into the languages a meeting's viewers chose, on `agent-stream` as `caption_translation` ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| Not connected | concept | A feature built end to end that stays visibly off until its credential exists (an OAuth app, an email or push provider), the ADR-0037 D3 pattern ([ADR-0040](adr/0040-polish-program.md)). | added (SaaS) |
| `AgentState` | concept | Shared dataclass carried through an agent graph (meeting_id, user_id, chunk window, context, retrieved_facts + SaaS tenant/authz/budget fields). | — |
| Confidence thresholds | concept | Surfacing policy: `> 0.7` surface · `0.4–0.7` silent-log to `worklogs` · `< 0.4` drop. | — |
| Windowed batching | concept | Agents run on speaker-merged windows, not per raw `transcript_chunks` row. Since [ADR-0040](adr/0040-polish-program.md) a window closes on a cue phrase, a turn end or a 10 to 15 s cap (at least 4 s apart); notes keep at least 30 s and suggestions 20 s; `A17` = 30 s remains the sizing figure. | added (SaaS) |
| Relevance pre-gating | concept | A cheap Haiku triage call per window scoring relevance; ~35% pass, the rest dropped before any extraction call. | added (SaaS) |
| Tiered routing (LiteLLM) | concept | Cheapest-capable model routing: Haiku triage/score → Sonnet extract → Opus high-stakes, via the LiteLLM router. Provider is Anthropic only ([ADR-0027](adr/0027-anthropic-only-llm-provider.md)); fast-response tier is Haiku 5.5 (`claude-haiku-5-5`); extraction tier is Sonnet 5.5 (`claude-sonnet-5-5`, [ADR-0029](adr/0029-plan-7-budget-scope-tier-mapping-and-metering-ownership.md)); high-stakes tier mapping to be set. | — (LiteLLM is PDF; per-tier mix is hardened) |
| LLM response cache | concept | Exact-match completion cache at `nexus:{org_id}:llmcache:{prompt_ref}:{input_hash}` (1h): a hit calls no model and meters nothing. The P7 form of the pipeline's semantic cache ([ADR-0030](adr/0030-plan-7-llm-response-cache-and-personal-listener-gating.md)). | added (SaaS) |
| Prompt caching | concept | Semantic + Anthropic prompt caching of stable prefixes (role taxonomy, system prompt, prior-window context). | added (SaaS) |
| Per-tenant token budgets / hard quotas | concept | `agent_budgets`-backed per (tenant, month) caps; 80/95% alerts; deny-by-default at `hard_cap`. PDF had per-user budgets; SaaS adds tenant-level hard caps. | PDF→SaaS (no rename) |
| `MeetingLifecycleWorkflow` | concept | Temporal workflow orchestrating a meeting's lifecycle. Also sleeps on durable timers for the status requests (start minus 60 minutes), the brief reminder (minus 10) and the agent catch-up (5 minutes after the end) ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md) B9). | — |
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
| `identity-service` | `identity-tenant-service` | Identity + tenancy control plane folded into one catalog service. Includes the company-brain tables `org_profiles`, `org_glossary_terms`, `topic_owners`, `org_goals` and `topic_watches` ([ADR-0039](adr/0039-company-brain-and-meeting-companions.md)). |
| `identity-service` (data model: `teams`/`team_members`/`departments`) | `core-api` | Per [`_canon/01-service-catalog.md`](_canon/01-service-catalog.md), `core-api` owns `teams`, `team_members`, and `departments` CRUD and their `/teams` / `/departments` prefixes. `identity-tenant-service` retains the identity and tenancy surface (`organizations`, `users`, RBAC, SSO/SCIM) and does not write these three tables. |
| `tenant-service` | `identity-tenant-service` | `tenant_settings` / `data_residency` are owned by the identity/tenancy control plane. |
| `meeting-service` (data model: meetings/participants/speakers/chat/agenda/templates) | `core-api` (meetings, participants, chat, `meeting_speakers`, `meeting_agenda_items`, `meeting_templates`) + `transcription-service` (`transcript_chunks`) + `meeting-service` (LiveKit transport adapter only) | The catalog splits the data-model "meeting-service" across three services; `meeting_speakers` is in `core-api`'s owned-data list and `meeting-service` proper is the LiveKit adapter. |
| `agent-service` | `core-api` | `worklogs`, `decisions`, `action_items`, `agent_suggestions`, `agent_collaborations`, `commitments`, `action_status_requests` are owned by `core-api`; `agent-runtime-service` owns no tables. |
| `rag-service` (data model) | `rag-service` | Same name; `document_chunks` ownership matches. |
| `graph-service` | `knowledge-graph-service` | Owns `nexus_knowledge`. |
| `billing-service` | `billing-metering-service` | Owns plans/subscriptions/usage_events/invoices. |
| `budget-service` | `billing-metering-service` | `agent_budgets` is owned by the billing/metering service in the catalog. |
| `audit-service` | `admin-ops-service` | The catalog folds audit-log access ("ops audit log", "audit-log access") into `admin-ops-service`; there is no `audit-service` in the catalog. |

> [Naming-authority note] No alternative names for any term in this glossary are permitted anywhere in the architecture set. Any future rename must land here as an explicit `old → new` row **and** supersede the relevant row in [`_canon/06-tech-decisions.md` §4](_canon/06-tech-decisions.md) via an [ADR](adr/).
