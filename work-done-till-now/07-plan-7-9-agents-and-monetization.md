# Plan 7-9: Agents and Monetization

**Status:** IN PROGRESS. Phase P7 work items 1 (`billing-metering-service`, plan section 3.1) and 2 (`agent-runtime-service`, section 3.2) are built and verified; items 3 to 5 (`PersonalListenerAgent` + `worklogs`, `mytheus-ui`, `mytheus-mobile`) are not started. Last verification 2026-11-08.

Plan: [plan/plan-7-9-agents-and-monetization.md](../plan/plan-7-9-agents-and-monetization.md).

The P7 build order is the plan's own precondition chain, one work item at a time: billing first (the agent tier cannot dispatch an LLM call until there is a budget to check), then the runtime, then the agent, then the two clients.

## 1. Entry decisions

Recorded before any code, canon first, in [ADR-0029](../adr/0029-plan-7-budget-scope-tier-mapping-and-metering-ownership.md) (decided by the user 2026-11-08):

| | Decision | Closes |
|---|---|---|
| D1 | Plan allowances are per-tenant monthly pools, not per seat. `token_quota = included_agent_tokens`; `hard_cap = token_quota` on Free, `2 x token_quota` on Pro. | DECISION-REQUIRED #1 |
| D2 | Extraction tier is Sonnet 5.5, `claude-sonnet-5-5`. Triage stays Haiku 5.5. High-stakes unmapped (unused before Plan 10). | ADR-0027 D3, for the extraction tier |
| D3 | `billing-metering-service` derives `meeting_minute`, `transcription_second` and `storage_gb` itself; `usage-events` carries `agent_token` only. | DECISION-REQUIRED #2 |
| D4 | `model_tier` added to `usage_events` and to `UsageEvent` (nullable, BACKWARD-safe). | Gap found 2026-11-08: the plan wanted a per-tier split the schema could not carry |
| D5 | No Anthropic DPA gate on LLM egress for this academic build. PII redaction before egress stays. Deepgram's `COMPLIANCE_GATE_GREEN` is not changed by this. | Plan section 6 compliance precondition |
| D6 | `usage_events` is partitioned by month with a DEFAULT, not by day. | Canon said day partitions |

## 2. What landed, per repo

### mytheus-docs

- `7fdbc0b`: ADR-0029 D1 to D5; canon `_canon/02-data-model.md` (`plans`, `usage_events`, `agent_budgets`), `_canon/03-event-taxonomy.md` (`usage-events`), `_canon/06-tech-decisions.md` (LLM routing, Compliance); reader docs 12 and 18, the Glossary, the traceability matrix, the plan's DECISION-REQUIRED #1 and #2, the status row.
- `edd09fd`: ADR-0029 D6 and the matching `_canon/02-data-model.md` edit.

### mytheus-events

- `ed5b4ab`: `schemas/usage-events/v1.avsc` gains `model_tier` (`["null","string"]`, default `null`); regenerated Python and TypeScript clients and the golden fixture. Registered on the local Schema Registry (all 21 subjects in sync).

### mytheus-backend

- `9b28dce`, migration `4c1d8e2a9b70`: `plans` (global, seeded Free/Pro/Enterprise with the plan's values), `subscriptions`, `invoices`, `usage_events`, `usage_aggregates`, `agent_budgets`, `quotas`. RLS ENABLE + FORCE with the canonical `tenant_isolation` policy on every tenant table; explicit grants per ADR-0021 (`plans` SELECT only; `usage_events` append-only, SELECT + INSERT); `usage_events` monthly partitions 2026-11 to 2027-12 plus `usage_events_default`, all owner-only; `usage_events` added to `scripts/ensure_partitions.py`.
- `734f80e`, `services/billing_metering_service/`:
  - `budgets.py`: lazy Free subscription and per-period `agent_budgets` row; the `ok -> throttled -> blocked` state machine; 80/95% flags that fire once each and independently; on a crossing, in the same transaction, a `billing-events` `quota_exceeded` outbox row (throttled, blocked) and in-app `notifications` outbox rows to every owner and admin (80%, 95%, blocked); the `nexus:{org_id}:budget:{period}` Redis mirror, written after commit, expiring at period end, carrying `enforcement` (`BUDGET_ENFORCEMENT_MODE`, `alert_only` by default).
  - `consumer.py`: the `usage-events` consumer (`python -m services.billing_metering_service.consumer`). `agent_token` only; anything else is poison and goes to `usage-events.dlq`. Exactly-once in effect: Redis dedupe, then UNIQUE `(org_id, idempotency_key, occurred_at)` (a duplicate inserts nothing and charges nothing), then the inbox row. A failed transaction deletes the dedupe key so the redelivery is metered; a transient failure seeks back instead of committing the offset.
  - `metering.py` + `jobs.py`: the nightly job (`python -m services.billing_metering_service.jobs`). One transaction per tenant under RLS: derives `meeting_minute` and `transcription_second` from ended meetings' `transcript_chunks` finals, then rolls the period's `usage_events` into `usage_aggregates` (with the haiku/sonnet/opus split) and refreshes the `meeting_minute` row in `quotas`.
  - `routes.py`: `GET /usage` (RBAC `admin`: live budget, plan, aggregates) and `GET /plans` (RBAC `member`).
  - `tests/test_billing_metering.py`: 20 tests.
- `73348fc`:
  - `storage.py`: the daily `storage_gb` snapshot per tenant, from read-only `ListObjectsV2` over the `{org_id}/` prefix of `mytheus-recordings`, `mytheus-documents` and `mytheus-attachments`. Skipped when `S3_*` is unset. The rollup reports the period's latest snapshot (a level, not a sum). Three more tests, with a fake S3 client (no network).
  - `services/workflow_service/workflows/usage_rollup.py`: `UsageRollupWorkflow` and the `usage-rollup-nightly` schedule (02:00 UTC, overlap SKIP), created once when the `workflow-service` worker starts. The workflow hands `billing.run_nightly` to the `billing-metering-service` task queue.
  - `services/billing_metering_service/temporal_worker.py`: the billing-owned worker for that activity (`python -m services.billing_metering_service.temporal_worker`), so only billing code writes billing tables.

- `60fa6b7`, `meeting-service`: `meeting-room-events` `joined`/`left` now carry `participant_id` = the LiveKit identity (`users.id` for a human) instead of the per-connection participant SID. The runtime builds its roster from it; nothing consumed the old value. The SID still keys the webhook dedupe.
- `5f26090`, `billing-metering-service` `internal.py`: `POST /billing/internal/budget`, the agent tier's budget read-through. It provisions the period's `agent_budgets` row, writes the mirror and returns the snapshot. Service-authenticated with the pii-vault header scheme (`X-Mytheus-Service`, `X-Mytheus-Service-Token`, `X-Mytheus-Org`); secrets in `MYTHEUS_BILLING_CALLER_TOKENS`, allowed callers `agent-runtime-service` and `rag-service`; unconfigured means every caller is refused. Two more tests.

### mytheus-intelligence

- `2649f21`, work item 2, `agent-runtime-service` (plan section 3.2):
  - `libs/shared/state.py`: `TranscriptWindow`, `Turn`, `AgentState` (the plan's fields plus `result`), `AgentResult`.
  - `libs/shared/base_agent.py`: `BaseAgent` on LangGraph, `spawn`/`process`/`terminate`; fails closed without `org_id`; `BudgetBlocked` makes `process` return None and marks the agent blocked.
  - `libs/shared/prompts.py`: `PromptLoader`, `{prompt_key}@{version}.md` files plus `registry.json` (`defaults`, per-tenant `pins`); a `---user---` line splits the cacheable system prompt from the per-call user message.
  - `libs/shared/budget.py`: `BudgetGate` (`check_budget`) and `BillingRefresher`.
  - `libs/shared/router.py`: `LiteLLMRouter`. `triage` to `claude-haiku-5-5`, `extract` to `claude-sonnet-5-5`, `high_stakes` refused while unmapped; budget gate before every call (`block` calls nothing, `throttle` forces Haiku); system prompt sent with `cache_control: ephemeral`; optional streaming via `on_delta`; exactly one `usage-events` record per call with its tier, emitted before the caller parses the answer.
  - `libs/shared/user_context.py` + `proto/identity.proto` (a copy of `mytheus-backend`'s): `get_user_context` over gRPC with `org-id` metadata, a descriptor mirror like `rag_proto.py`; a reply naming another tenant is dropped.
  - `services/agent_runtime_service/windowing.py`: fixed 30 s buckets, window id `{meeting_id}:{start_ms}`, same-speaker turn merging, chunk-id dedupe, close on next bucket, idle (`WINDOW_IDLE_S`, 15 s) or meeting end.
  - `runtime.py`: the spawner. Roster from `meeting-room-events` plus the window's speakers; one instance of each participant-scope agent per user, reused across windows; `left` terminates; one in-app notification per blocked participant per meeting.
  - `emitter.py`: `usage-events`, `agent-stream`, `agent-results`, `notifications` with an idempotent producer.
  - `consumer.py`: `python -m services.agent_runtime_service.consumer`. Commits an offset only once every window holding it is processed and the producer is flushed.
  - `agents/__init__.py`: the registry, empty until item 3; `prompts/registry.json`, empty.
  - `tests/test_agent_runtime.py`: 27 tests; `fakeredis` added as a dev dependency.

### mytheus-realtime

- `34501d0`: fix to `transcription-service` migration `e4a7c91b3d28`. On a database built from empty in or after 2026-11, `be66698dbefb` (which premakes months from `date.today()`) and `e4a7c91b3d28` (fixed months from 2026-11) both created `transcript_chunks_2026_11`, and the upgrade failed with `DuplicateTableError`. The migration now skips a month that already exists. Found while bringing up a fresh local database for billing.

## 3. Definition of Done audit (P7, billing items only)

From the plan's section 10 "End of P7" DoD. Lines for items 2 to 5 are not audited yet.

| DoD line | Status | Evidence |
|---|---|---|
| `billing-metering-service` shipped: plans/subscriptions/invoices/usage_events/usage_aggregates/agent_budgets/quotas migrated with RLS FORCE | MET | Migration `4c1d8e2a9b70`; `test_rls_isolates_usage_and_budgets`; `tests/test_roles_and_policies.py` grant and partition tests |
| `usage-events` consumer with idempotent dedupe | MET | `test_a_message_is_metered_once_and_mirrored` (redelivery and cache eviction); real Redpanda run below |
| Redis budget mirror | MET | Same test asserts the hash and TTL; real run read it back |
| `ok -> throttled -> blocked` state machine | MET | `test_free_goes_straight_to_blocked_at_its_cap`, `test_pro_throttles_at_quota_and_blocks_at_twice_quota` |
| alert-only -> enforce flag | MET (the flag) | `BUDGET_ENFORCEMENT_MODE`, carried in the mirror; the agent tier honouring it is item 2 |
| budget deny-by-default test | NOT MET yet | Belongs to `check_budget` in `agent-runtime-service` (item 2) |
| RLS isolation test | MET | `test_rls_isolates_usage_and_budgets`, including the no-tenant fail-closed read |
| DECISION-REQUIRED #1 + #2 resolved | MET | ADR-0029 D1, D3 |
| Usage console shipped | NOT MET yet | API is `GET /usage`; the screen is item 4 |
| `agent-runtime-service` shipped: windowing | MET | `test_360_finals_over_45_minutes_make_90_windows` |
| Haiku pre-gate | MET (mechanism) | `test_the_pre_gate_decides_whether_extraction_runs` with a two-step test agent; the real pre-gate prompt is item 3 |
| LiteLLM tiered routing | MET | `test_throttle_forces_haiku_and_ok_routes_extract_to_sonnet`, `test_high_stakes_is_refused_while_unmapped` |
| prompt caching | PARTIAL | Anthropic prompt caching on the system prompt; no semantic cache (deviation 11) |
| inline `check_budget` | MET | `test_gate_decides_from_the_mirror`, `test_a_miss_reads_through_and_every_failure_blocks` |
| budget deny-by-default test | MET | `test_block_calls_no_model_and_meters_nothing`, `test_a_blocked_budget_returns_nothing` |
| `usage-events` emit | MET | router tests; runtime round trip in section 6 |
| `agent-stream` bridge | PARTIAL | `Emitter.delta` and the router's `on_delta` are tested; no shipped agent streams yet (item 3) |
| writes zero Postgres | MET | `test_the_runtime_imports_no_database_driver` |
| pre-gate count test | NOT MET yet | Needs the real agent and its eval set (item 3) |

## 4. Deviations from the plan, and why

1. **`usage_events` monthly, not daily.** ADR-0029 D6: it matches the other partitioned tables and their existing maintenance script and coverage test; daily would mean about 365 partitions a year plus a second job for a small volume. The unique index carries `occurred_at` because Postgres requires the partition key in it; that is equivalent to the canon's key because `occurred_at` is part of the idempotency key.
2. **Alerts are evaluated independently, not as an `elif` chain.** The plan's sketch used `elif`, so one charge that crossed 80% and 95% together would set only one flag. Each threshold now fires once on its own.
3. **Budget alerts go out as `notifications`, not as `billing-events`.** `BillingEventType` has no alert or throttle symbol (`quota_exceeded`, `dunning`, `plan_changed`). Adding a symbol is not safe for consumers on the old schema, so throttled and blocked use `quota_exceeded` with `detail_json.state`, and the 80/95% alerts are in-app `notifications` to owners and admins, which `billing-metering-service` is already a canon producer of.
4. **Enforcement is an environment variable, not an `admin-ops-service` feature flag.** Plan step 11 names `BUDGET_ENFORCEMENT_MODE`; step 10 mentions the admin-ops flag. `admin-ops-service` has no feature-flag store yet, so the variable is the switch. The true state is always recorded; the mode only tells the agent tier whether to act on it.
5. **The nightly activity runs on a billing-owned worker, not on the `workflow-service` worker.** The plan says "`workflow-service` Temporal schedule". The schedule and workflow are in `workflow-service`; the activity is dispatched to the `billing-metering-service` task queue, because an activity in the `workflow-service` process writing billing tables would be a cross-service write (backend CLAUDE.md: activities do not cross-write another service's tables).
6. **`meeting_minute` measures the spoken span.** `meetings` carries scheduled times, not actual start and end, so minutes come from the first final's `start_ms` to the last final's `end_ms`, rounded up. `transcription_second` sums per-participant final durations, matching per-stream provider billing.
7. **`quotas` records meeting minutes but does not enforce them.** Over quota sets `state = throttled`; nothing stops a meeting, because durable facts are never shed.
8. **The first `GET /usage` provisions the month.** Step 9's lazy create runs on read as well as on the first charge, so the console is never empty on the 1st.

9. **The runtime publishes straight to Redpanda, not through a transactional outbox** (plan section 3.2 step 10 says outbox). An outbox makes an emit atomic with a database write; the runtime makes none, and its acceptance check is zero Postgres rows. At-least-once comes from flushing the idempotent producer before committing input offsets; duplicates are absorbed by the idempotency keys (`agentresult:...` is deduped by `core-api`).
10. **The budget read-through goes to billing, not to Postgres** (step 9 says "read through to Postgres"). The runtime writes no Postgres and cannot create the period's `agent_budgets` row, so a plain read would leave a new tenant (and every tenant on the 1st) without a budget, blocked forever. `POST /billing/internal/budget` provisions and republishes; anything else blocks.
11. **No semantic cache.** Step 8 asks for one in front of the model. It needs a Redis key canon does not define, so it waits for a canon edit. Anthropic prompt caching, the bigger lever for repeated system prompts, is on.
12. **No per-user priority queue.** `nexus:{org_id}:queue:agent:{user_id}` belongs to `notification-service` in the backend service map. Windows run one at a time per process, with users concurrent inside a window.
13. **The runtime emits no `billing-events`.** The plan's `check_budget` sketch emits `quota_exceeded`; canon makes `billing-metering-service` the only producer, and it already does so on the crossing.
14. **Meeting end is "last participant left".** `meeting-room-events` has no room-finished symbol (`joined`, `left`, `recording_ready`); idle close covers a meeting nobody leaves cleanly.
15. **Window speakers join the roster.** A runtime restarted mid-meeting missed the `joined` events; anyone who talks is served anyway.
16. **The budget can overshoot slightly.** The mirror advances when billing meters a call, which lags the call, so concurrent calls in that gap all see the older total. The overshoot is bounded by one window's calls for the tenant.

## 5. Open gaps

1. ~~`storage_gb` is not derived.~~ Closed in `73348fc`. The local `.env` points `S3_*` at MinIO; production must point it at R2. Cost there is one Class A list call per bucket per tenant per night, inside R2's 1M free a month for any small deployment.
2. ~~No Temporal schedule.~~ Closed in `73348fc`. Deploying it means running two more processes: the `workflow-service` worker (already planned) and `python -m services.billing_metering_service.temporal_worker`.
3. **Mirror read-through for the agent tier.** Plan step 9 of section 3.2 says a mirror miss falls through to Postgres. How `agent-runtime-service` does that (its own read of `agent_budgets`, or an internal billing endpoint) is decided in item 2.
4. **Stripe checkout, `/stripe/webhook`, dunning, invoices** are left to their own band, as the plan says.
6. **Deploying item 2** means one more process (`python -m services.agent_runtime_service.consumer`) with `BILLING_URL`, `BILLING_SERVICE_TOKEN` (matching an entry in billing's `MYTHEUS_BILLING_CALLER_TOKENS`), `IDENTITY_GRPC_ADDR`, `ANTHROPIC_API_KEY`, `REDIS_URL`, `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`.
7. **LLM-egress PII redaction** is part of item 3 (plan section 3.3 step 4), not built yet.
5. **Local environment only** (not code defects): `test_public_schema_grants_nothing_by_default` and `test_no_bypassrls_role_holds_privileges_on_a_public_table` fail on this machine's database (the image's `supabase_admin` default privileges, and `alembic_version_intelligence` created by `mytheus-intelligence` in the same database); six `test_crypto_file_custody.py` tests fail on Windows (no 0400 file modes); `test_relay_real_bus_roundtrip` fails because Redpanda ran with a 768M memory cap and refused new partitions.

## 5a. Next session: start here

Items 1 and 2 are done; items 3, 4 and 5 are next. Open items carried from items 1 and 2, in the order they should be picked up:

1. **Real-model check: blocked on the Anthropic workspace.** The first real call (2026-11-08, one window, 2 participants, at most 4 capped calls) was refused with 400 before any tokens were used: "This API key is not scoped to a workspace, so this request must include the anthropic-workspace-id header". `mytheus-intelligence` `6355e29` makes the router send that header when `ANTHROPIC_WORKSPACE_ID` is set. To unblock: add `ANTHROPIC_WORKSPACE_ID` (from the Anthropic console, Settings, Workspaces) to `mytheus-intelligence/.env`, or replace the key with a workspace-scoped one, then rerun the check. It has not been run successfully yet, so no real-model behavior is verified.
2. **Item 3, `PersonalListenerAgent`** (plan section 3.3): the real pre-gate and extraction prompts (`prompts/personal_listener*@v1.md`, registered in `prompts/registry.json`); confidence gating (`>0.7` surfaced, `0.4-0.7` silent, `<0.4` dropped); registration in `agents/__init__.py`; the golden-set eval; and `core-api`'s `worklogs` migration, `agent-results` consumer and `GET /api/v1/meetings/:id/worklogs?user_id=me`.
3. **Live streaming of the note's text** (`agent-stream`): wired (`Emitter.delta`, the router's `on_delta`, both tested) but unused, because no shipped agent streams yet. The real note-taker in item 3 should stream only after its confidence is known, so a note that ends up below 0.7 never reaches the screen. `realtime-gateway` has no `agent-stream` or `agent-results` consumer yet, and its Socket.io rooms are per meeting, not per user. Item 4 needs a per-user channel, or one participant's notes reach everyone in the room.
4. **Hiding personal details before text goes to Anthropic** (plan section 3.3 step 4): not built. Reuse `services/ingestion_service/pii.py` (`redact_pii` plus `pii_enabled`, which reads `nexus:{org_id}:tenant_settings:pii_redaction_enabled` and defaults to redact), or the stronger `services/ner_service/regex_detector.py`, applied to the window text inside the router or the agent before any model call.
5. **The "skip near-duplicate windows" (semantic) cache** (plan section 3.2 step 8): needs a Redis key that `_canon/02-data-model.md` does not define. Add it to canon first (plus an ADR line), then build it. Anthropic prompt caching is already on.
6. **Items 4 and 5**, `mytheus-ui` (agent sidebar, My Notes, Usage page on `GET /usage`) and `mytheus-mobile` (My Notes, sample data removed), after item 3.

## 6. Verification

All observed 2026-11-08 against the local stack (Postgres 5443, Redis 6390, Redpanda 9092 with Schema Registry 8085; Redpanda capped at 768M through a compose override kept outside the repo).

- `mytheus-events`: `uv run python codegen/validate.py schemas` OK; codegen regenerated; `uv run pytest tests -q` passed except `test_create_topics_is_idempotent` (bash cannot resolve the Windows script path on this machine); `node --test tests/ts/*.test.mjs` 1 passed; `codegen/check_compat.py` OK (21 subjects BACKWARD); `codegen/register.py` OK; `register.py --check` all 21 in sync.
- `mytheus-backend`: `MYTHEUS_REQUIRE_DB=1 uv run pytest tests/test_billing_metering.py -q` 20 passed. Full suite (two environment tests deselected): 611 passed, 7 failed, all seven environmental (gap 5). `uv run ruff check`, `uv run black --check`, `uv run mypy services/billing_metering_service` clean.
- `mytheus-realtime`: `uv run alembic upgrade head` on the local database now succeeds through `e4a7c91b3d28`.
- `73348fc`: `tests/test_billing_metering.py` 23 passed. Real Temporal (`docker compose up -d temporal`) and MinIO: a throwaway script started both workers in-process, `ensure_schedule` returned True then False (idempotent), the schedule's next action was 2026-11-09 02:00 UTC, and one manual trigger ran `billing.run_nightly` to completion (`{'tenants': 4266, 'storage_rows': 4265, 'derived_rows': 0, 'failed': 0}`; the tenants are mostly test fixtures left in the local database).
- `2649f21`: `uv run pytest tests/test_agent_runtime.py -q` 27 passed; `ruff check`, `black --check`, `mypy libs services` (80 files) clean.
- **Runtime round trip, no LLM spend** (throwaway scripts, not committed): a tenant's budget provisioned through the real `POST /billing/internal/budget` (200, quota 200000); the real `Emitter`, `AvroCodec` and `BudgetGate` on real Redis, with the test suite's two-step agent and a fake model, ran one window for 3 participants. Produced to real Redpanda against the registered schemas: 3 `agent-results`, 6 `usage-events`. The offset tracker held back the third chunk (its window still open). The real billing consumer then metered 6 rows, 720 tokens, tiers haiku and sonnet; `agent_budgets` and the mirror both read 720.
- **Real round trip** (throwaway scripts, not committed): two copies of one Avro `usage-events` message produced to Redpanda; `python -m services.billing_metering_service.consumer` run for 25 s; `python -m services.billing_metering_service.jobs` run once. Result: one `usage_events` row (1234 tokens, `haiku`); `agent_budgets` 1234 of 200000, `ok`; `usage_aggregates` `agent_token` 1234 with `{"haiku": 1234}`; the Redis mirror read back with `enforcement` `alert_only`. The first attempt sent every real message to the DLQ: the confluent deserializer returns `occurred_at` as a datetime, not an integer. Fixed in `734f80e` with a regression test; the unit tests had not caught it because they built the value by hand.
