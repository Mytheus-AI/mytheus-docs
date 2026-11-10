# NEXUS Canonical Event Taxonomy

The single naming authority for every NEXUS Redpanda topic — its Avro value schema, producer/consumer services, tenant partition key, retention, and delivery/idempotency rule — distilled verbatim from [Event-Driven Architecture](../07-event-architecture.md).

## Topic catalog

All 21 topics (9 PDF-original + 12 SaaS-added). Partition key is `org_id` (the tenant UUID string) on 19 of the 21 topics. `transcript-chunks` and `agent-stream` are the exception: their partition key is the composite `{org_id}:{meeting_id}` ([ADR-0017](../adr/0017-transcript-chunks-composite-partition-key.md)) — it preserves within-meeting speech ordering while letting a hot enterprise tenant spread across partitions. `org_id` remains present and non-nullable in the envelope header on every topic, including these two; it is the tenant key and the leading component of their composite key even though it is no longer the whole key. Every other topic keeps `org_id` alone as its partition key. A paired `<topic>.dlq` (14-day retention) exists for every topic. Delivery is **at-least-once, idempotent in effect**: every consumer dedupes via the durable `inbox` table plus Redis cache `nexus:{org_id}:dedupe:{topic}:{consumer_group}:{idempotency_key}` (24h TTL, one key per consumer group, ADR-0031 D6); explicit `idempotency_key` formulas are given where [07 §6.3](../07-event-architecture.md) defines one.

| Topic | Producer service | Consumer services | Avro value schema | Partition key | Retention (cleanup) | Delivery / idempotency |
|---|---|---|---|---|---|---|
| `transcript-chunks` | `transcription-service` | `embedding-service`, `agent-runtime-service`, `search-service` | `TranscriptChunk` | `{org_id}:{meeting_id}` | 7 days (delete) | At-least-once; inbox + Redis dedupe. Key `chunk:{chunk_id}` |
| `document-chunks` | `ingestion-service` | `embedding-service`, `agent-runtime-service` | `DocumentChunk` † | `org_id` | 7 days (delete) | At-least-once; inbox + Redis dedupe. Key `docchunk:{document_id}:{element_id}` |
| `agent-results` | `agent-runtime-service` | `core-api`, `realtime-gateway`, `notification-service` | `AgentResult` | `org_id` | 3 days (delete) | At-least-once; inbox + Redis dedupe. Key `agentresult:{meeting_id}:{window_id}:{user_id}:{agent}` `kind` adds `context_fill` (ContextGapAgent: A2A collaboration entries plus the optional fill, [ADR-0035](../adr/0035-plan-10-context-gap-agent-and-a2a-build.md)). |
| `decisions` | `core-api` (outbox, on confirm) | `knowledge-graph-service`, `embedding-service`, `notification-service` | `Decision` | `org_id` | 30 days (delete) | At-least-once; inbox + Redis dedupe + `MERGE`. Key `decision:{decision_id}:confirmed`. Carries optional `supersedes` (the earlier decision replaced), the `FOLLOWS_FROM` source ([ADR-0031](../adr/0031-plan-8-graph-store-decision-supersedes-and-mark-done.md) D2, D5) |
| `actions` | `core-api` (outbox, on confirm) | `knowledge-graph-service`, `workflow-service`, `integration-service` | `Action` | `org_id` | 30 days (delete) | At-least-once; inbox + Redis dedupe + `MERGE`. Key `action:{action_id}:confirmed` |
| `notifications` | multiple | `notification-service`, `realtime-gateway` | `Notification` | `org_id` | 3 days (delete) | At-least-once; inbox + Redis dedupe. Key `notif:{source_event_id}:{channel}:{user_id}` |
| `agent-requests` | `agent-runtime-service` | `agent-runtime-service` (peer agents) | `AgentRequest` | `org_id` | 1 day (delete) | At-least-once; inbox + Redis dedupe. Key `areq:{correlation_id}` |
| `agent-responses` | `agent-runtime-service` | `agent-runtime-service` (requester) | `AgentResponse` | `org_id` | 1 day (delete) | At-least-once; inbox + Redis dedupe |
| `agent-stream` | `agent-runtime-service` (LiteLLM stream bridge) | `realtime-gateway` → WebSocket | `AgentStreamDelta` † | `{org_id}:{meeting_id}` | 1 hour (delete) | At-least-once; inbox + Redis dedupe (throwaway; shed first under backpressure) |
| `tenant-events` | `identity-tenant-service` | `billing-metering-service`, `admin-ops-service`, `workflow-service` | `TenantEvent` | `org_id` | retain (compact) | At-least-once; inbox + Redis dedupe |
| `meeting-events` | `core-api` | `meeting-service`, `workflow-service`, `integration-service` | `MeetingEvent` | `org_id` | 7 days (delete) | At-least-once; inbox + Redis dedupe |
| `meeting-room-events` | `meeting-service` | `transcription-service`, `core-api`, `knowledge-graph-service` (`ATTENDED`, [ADR-0031](../adr/0031-plan-8-graph-store-decision-supersedes-and-mark-done.md) D7) | `MeetingRoomEvent` | `org_id` | 3 days (delete) | At-least-once; inbox + Redis dedupe |
| `integration-events` | `integration-service` | `ingestion-service`, `core-api` | `IntegrationEvent` | `org_id` | 7 days (delete) | At-least-once; inbox + Redis dedupe |
| `usage-events` | `agent-runtime-service`, `rag-service` | `billing-metering-service` | `UsageEvent` | `org_id` | 7 days (delete) | At-least-once; inbox + Redis dedupe. Key `usage:{org_id}:{event_type}:{occurred_at}:{source_id}`. Carries `agent_token` only, with optional `model_tier` (`haiku`/`sonnet`/`opus`); the other three dimensions are derived inside `billing-metering-service` and never traverse this topic ([ADR-0029](../adr/0029-plan-7-budget-scope-tier-mapping-and-metering-ownership.md) D3, D4) |
| `billing-events` | `billing-metering-service` | `workflow-service`, `notification-service`, `admin-ops-service` | `BillingEvent` | `org_id` | 30 days (delete) | At-least-once; inbox + Redis dedupe |
| `graph-events` | `knowledge-graph-service` | `admin-ops-service` | `GraphEvent` | `org_id` | 7 days (delete) | At-least-once; inbox + Redis dedupe |
| `embedding-events` | `embedding-service` | (lag / metrics) | `EmbeddingEvent` | `org_id` | 1 day (delete) | At-least-once; inbox + Redis dedupe |
| `presence-events` | `realtime-gateway` | `core-api` (analytics) | `PresenceEvent` | `org_id` | 1 hour (delete) | At-least-once; inbox + Redis dedupe |
| `notification-delivery-events` | `notification-service` | `admin-ops-service`, `core-api` | `NotificationDeliveryEvent` | `org_id` | 30 days (delete) | At-least-once; inbox + Redis dedupe |
| `user-sync-events` | `identity-tenant-service` (Clerk inbox) | `identity-tenant-service` | `UserSyncEvent` | `org_id` | 7 days (delete) | At-least-once; inbox + Redis dedupe |
| `admin-events` | `admin-ops-service` | `workflow-service`, `admin-ops-service` (audit-log write) | `AdminEvent` | `org_id` | retain (compact) | At-least-once; inbox + Redis dedupe |

† `DocumentChunk` and `AgentStreamDelta` are net-new names introduced by [07](../07-event-architecture.md) for the `document-chunks` "(chunk envelope)" and `agent-stream` "(token-delta envelope)"; they are **not** frozen canon. Canon freezes only seven schema names: `TranscriptChunk`, `AgentResult`, `Decision`, `Action`, `Notification`, `AgentRequest`, `AgentResponse`. The `Tenant/Meeting/…Event` names for the SaaS topics are likewise net-new.

## Avro envelope and Schema Registry

Every event value opens with the shared `EventEnvelopeHeader` (namespace `io.nexus.events`) then a topic-specific `payload`. Locked across all 21 topics. Frozen header fields:

| Field | Type | Role |
|---|---|---|
| `event_id` | `string` (uuid) | Unique per emitted event; distinct from `idempotency_key`. |
| `idempotency_key` | `string` | Business dedupe key; stable across producer retries of the same fact. |
| `org_id` | `string` (uuid) | Tenant root; RLS/cell scoping anchor; the Kafka partition key on 19 of 21 topics, and the leading component of the `{org_id}:{meeting_id}` composite key on `transcript-chunks` and `agent-stream` (see Partition key column above). |
| `schema_version` | `int` (default 1) | Producer's payload schema version (observability; Registry enforces compat independently). |
| `producer` | `string` | Owning service name (kebab-case, from the Service Catalog). |
| `occurred_at` | `long` (timestamp-micros) | When the fact happened (UTC); not enqueue time. |
| `correlation_id` | `["null","string"]` (default null) | Ties a causal chain across topics. |
| `causation_id` | `["null","string"]` (default null) | `event_id` of the direct cause; null for root events. |

- **Subject naming:** `{topic}-value` — one subject per topic. No key schemas; the partition key is the raw `org_id` UUID string.
- **Compatibility:** **BACKWARD**. Allowed: add a field with a default; remove a field that had a default; widen via union with `null` + default; append enum symbols with a default. Forbidden: add a required field without a default; remove a non-defaulted field; rename; narrow a type. Header field order is frozen by convention.
- **Enforcement:** CI `check-compatibility` against the registered subject gates every producer image build.
