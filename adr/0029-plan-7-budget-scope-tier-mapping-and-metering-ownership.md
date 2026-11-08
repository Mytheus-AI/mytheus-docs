# ADR-0029: Plan 7 entry decisions: per-tenant budgets, the Sonnet 5.5 extraction tier, in-service metering derivation, the usage tier field, and no Anthropic DPA gate

Plan 7-9 lists open decisions that block the Personal Listener and billing phase. The user resolved them on
2026-11-08. This ADR records the answers and the reasoning, and closes DECISION-REQUIRED #1 and #2 of
[plan-7-9-agents-and-monetization.md](../plan/plan-7-9-agents-and-monetization.md).

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md) - `plans`, `usage_events` and `agent_budgets`, updated in the same change.
- [_canon/03-event-taxonomy.md](../_canon/03-event-taxonomy.md) - the `usage-events` row, updated in the same change.
- [_canon/06-tech-decisions.md](../_canon/06-tech-decisions.md) - the "LLM routing" and "Compliance" rows, updated in the same change.
- [ADR-0004](0004-llm-cost-control-pipeline.md) - the tiered pipeline and the `agent_budgets` state machine. Unchanged in shape.
- [ADR-0011](0011-billing-and-metering.md) - billing and metering, and the topic-versus-table producer reconciliation that D3 adopts.
- [ADR-0027](0027-anthropic-only-llm-provider.md) - Anthropic only; its D3 left the extraction and high-stakes tiers unmapped. D2 below maps the extraction tier.

---

## Status

**Accepted** (decided by the user 2026-11-08).

## Context

1. Canon froze `plans.included_agent_tokens` and `plans.included_meeting_minutes` as columns but not whether they are per seat or per tenant, so the `agent_budgets` quota could not be computed.
2. ADR-0027 mapped only the fast tier (`claude-haiku-5-5`). The extraction tier, which does the main work of every Plan 7-9 agent, had no model.
3. Three of the four `usage_events` dimensions (`meeting_minute`, `transcription_second`, `storage_gb`) had no named producer.
4. The plan wants usage shown per model tier, but neither the `usage_events` table nor the `UsageEvent` schema carried the tier.
5. Plan 7 made Anthropic subprocessor DPAs and SCCs a precondition for the first LLM egress.

Mytheus is a final-year engineering project. It has no paying customers and is not offered for sale. That changes the weight of decisions made only to protect commercial margin or to satisfy a buyer's procurement review.

## Decision

**D1. Plan allowances are per tenant (pooled).** Mytheus is meeting software: a company's meetings, not individual seats, drive agent usage. A seat that attends no meeting uses no tokens, so scaling the budget by seats would hand out budget nobody spends.

- `agent_budgets.token_quota = plans.included_agent_tokens`, flat per tenant per `YYYY-MM`. `subscriptions.seats` does not change it.
- `plans.included_meeting_minutes` is likewise a per-tenant monthly pool.
- `hard_cap`: Free sets `hard_cap = token_quota`, so it goes straight from `ok` to `blocked`. Pro sets `hard_cap = 2 x token_quota`. The band between `token_quota` and `hard_cap` is the `throttled` (Haiku-only) overage band, and above it the tenant is `blocked`, so the 100% hard-cap SLO still holds. Enterprise (`included_agent_tokens = 0` in the catalog) sets both values on the tenant's own row.

**D2. The extraction tier is Sonnet 5.5, model id `claude-sonnet-5-5`.** The tier map is now: triage = `claude-haiku-5-5`, extract = `claude-sonnet-5-5`. The high-stakes tier stays unmapped. Only `ProactiveAgent` and `ContextGapAgent` (Plans 10-11) use it, and the routing tests reject it for every Plan 7-9 call anyway.

**D3. `billing-metering-service` derives the three non-token dimensions itself** (option (a) of DECISION-REQUIRED #2, which is ADR-0011's reading). `meeting_minute` and `transcription_second` come from `meetings` durations and `transcript_chunks` start and end timestamps. `storage_gb` comes from a nightly sweep on a `workflow-service` Temporal schedule. It writes them as `usage_events` rows directly. The `usage-events` topic keeps exactly two producers (`agent-runtime-service`, `rag-service`) and carries `agent_token` only, so the Service Catalog's producer map does not change. Option (a) was chosen because the billing service already owns the dimensions and their idempotency, and options (b) and (c) would add topic producers for data the service can read itself.

**D4. `model_tier` is added to `usage_events` and to `UsageEvent`.** It is nullable text in the table and an optional `["null", "string"]` Avro field with default `null`, which is a BACKWARD-compatible addition. It holds the tier name (`haiku`, `sonnet`, `opus`), not the model id, so a model upgrade does not split one tier's history. It is set for `agent_token` and null for the other three dimensions. `usage_aggregates.tier_breakdown` is filled from it.

**D5. No Anthropic DPA or SCC gate before LLM egress.** Subprocessor paperwork exists to serve a customer's procurement review, and this project has none. No engineering gate is built for Anthropic egress. The always-on PII redaction before egress (gated by `tenant_settings.pii_redaction_enabled`) is a technical control, not paperwork, and stays. This ADR does not change the Deepgram gate (`COMPLIANCE_GATE_GREEN`). If the project is ever offered to customers, D5 reverses and the DPA becomes a release blocker again.

**D6. `usage_events` is partitioned by month, not by day.** Canon said day partitions. Every other partitioned table here (`chat_messages`, `audit_logs`) uses monthly partitions plus a DEFAULT catch-all, kept ahead of the calendar by `scripts/ensure_partitions.py` and checked by `tests/test_partition_coverage.py`. Daily partitions would mean about 365 tables a year and a second maintenance job, for a project whose usage volume is small. The dedupe constraint is UNIQUE `(org_id, idempotency_key, occurred_at)`, because Postgres requires the partition key in a unique index on a partitioned table. Since `occurred_at` is already part of the idempotency key, this is equivalent to UNIQUE `(org_id, idempotency_key)`.

## Consequences

- **Positive.** The quota is computable, so `billing-metering-service` can provision `agent_budgets` rows and ship in alert-only mode.
- **Positive.** Every Plan 7-9 agent has a model to call, and the Usage console can show the per-tier split.
- **Negative, accepted.** Under per-tenant pooling, a tenant that adds seats gets no extra budget. The Pro margin model in [Cost and FinOps](../17-cost-and-finops.md) assumed per-seat allowances. It is not re-derived here, because nothing in this project depends on it.
- **Negative, accepted.** Pro's `2 x token_quota` hard cap is a working value, not one derived from the capacity model.
- **Follow-up.** The `mytheus-events` schema `schemas/usage-events/v1.avsc` gains the `model_tier` field in the same change set, and the subject is re-registered.
- **Follow-up.** The cost figures in [_canon/05-capacity-math.md](../_canon/05-capacity-math.md) section 9 were stated for earlier models. ADR-0027 already flagged re-costing. This ADR changes no number.

> [SaaS-scale adjustment] The SaaS overlay read plan allowances per seat and gated LLM egress on subprocessor DPAs. For this academic build, allowances are pooled per tenant and the DPA gate is dropped. The budget state machine and PII redaction are unchanged.
