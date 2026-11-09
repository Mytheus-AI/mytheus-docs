# Plan 7-9: Agents and Monetization

**Status:** IN PROGRESS. Phase P7 work items 1 to 4 are built, deployed and verified; the PersonalListener pre-gate has a measured live baseline (section 8): `billing-metering-service` (section 3.1), `agent-runtime-service` (3.2), `PersonalListenerAgent` with `worklogs` and LLM-egress PII redaction (3.3), and the `mytheus-ui` agent sidebar, My Notes and Usage page (3.4), plus the `realtime-gateway` agent-output bridge they need. Item 5 (`mytheus-mobile`, section 7) is deliberately not started (excluded by the maintainer for this session). P8 Part B (knowledge graph, decision embeddings, My Tasks, decision lineage) is built and tested but not pushed (section 9); P8 Part A and P9 are not started. Last verification 2026-11-09. What is left in P7 is listed in section 5a.

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
- `eec51e9`: [ADR-0030](../adr/0030-plan-7-llm-response-cache-and-personal-listener-gating.md), the Plan 7 build decisions for items 3 and 4 (D1 the exact-match LLM response cache, D2 the relevance threshold, D3 `core-api` decides `surfaced` and the `agent_confidence_overrides` shape, D4 only surfaced notes stream, D5 per-user live delivery). Canon first: `_canon/02-data-model.md` gains `nexus:{org_id}:llmcache:{prompt_ref}:{input_hash}`; propagated to docs 05, 06, 08, the Glossary, the traceability matrix and the README index.
- `c2bac53`: `_canon/02-data-model.md` `worklogs` row carries `window_id`, `confidence`, `surfaced`, `created_at`, `deleted_at` and the monthly partitioning.

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
- `4fffbc6`, work item 3, the `core-api` half (section 3.3 step 3):
  - Migration `d5e2f1a8c370`: `worklogs` (plan DDL plus `window_id`), monthly partitions 2026-11 to 2027-12 plus `worklogs_default`, index `(org_id, meeting_id, user_id, created_at)`, RLS ENABLE + FORCE with `tenant_isolation`, explicit grants, partitions owner-only; registered in `scripts/ensure_partitions.py`.
  - `services/core_api/consumer.py` consumes `agent-results` beside `presence-events`. `handle_agent_result` claims the inbox row FIRST in the same transaction (see deviation 17), checks under RLS that the meeting and user belong to the tenant (an FK alone would accept another tenant's ids, gap 31 in the overview), reads `tenant_settings.agent_confidence_overrides` and decides `surfaced` itself; below the silent threshold nothing is written. Other agent kinds are consumed without a row until P8. A failed apply deletes the Redis dedupe key so the redelivery is applied.
  - `GET /api/v1/meetings/{id}/worklogs?user_id=me` (`services/core_api/routes/worklogs.py`): the caller's own surfaced notes, oldest first, at most 500. `user_id` accepts only `me` (anything else is 422); another tenant's meeting is 404. `api-gateway` already routed `/api/v1/meetings` to `core-api`.
  - OpenAPI re-emitted (ADR-0022). The run also wrote `design/openapi/billing-metering-service.yaml`, which the item 1 session had emitted but never committed; it is committed now.
  - `tests/test_worklogs.py`: 9 tests (now 10).
- `97f2709`: an `agent-results` record for a tenant with no `organizations` row raised `IntegrityError` from the inbox foreign key, which the loop does not catch, so the consumer died and every restart hit the same record. It is now `PoisonMessage` and dead-lettered. Found by the end-to-end run (section 6), against records left in the local broker by the item 2 session's throwaway test.

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
- `0533c14`, LLM-egress PII redaction (section 3.3 step 4). The regex redactor moved from `services/ingestion_service/pii.py` to `libs/shared/pii.py` (ingestion keeps a re-export). `EgressRedactor` reads `nexus:{org_id}:tenant_settings:pii_redaction_enabled` and redacts unless the tenant explicitly turned it off; a Redis failure redacts. `LiteLLMRouter` redacts every user message before the budget check and the call; with no redactor wired it redacts unconditionally, so a missed wiring can only over-redact. The system prompt is a reviewed template and is not redacted (it stays cacheable).
- `bac5c85`, the LLM response cache (ADR-0030 D1, plan section 3.2 step 8): `libs/shared/llm_cache.py`, `nexus:{org_id}:llmcache:{prompt_ref}:{input_hash}`, 1 hour. Callers opt in with `prompt_ref`. A hit calls no model, emits no `usage-events` record and charges no budget; the budget check still runs first. Redis errors are misses.
- `a0c51b3`, work item 3, `PersonalListenerAgent` (`services/agent_runtime_service/agents/personal_listener_agent.py`):
  - `prompts/personal_listener_gate@v1.md` (triage, Haiku) scores the window 0-1 against role, skills and bio, answering `{"relevance": x}`. Below `PERSONAL_LISTENER_RELEVANCE_THRESHOLD` (default 0.5) the graph ends with no extraction call. An unparseable score is 0.
  - `prompts/personal_listener@v1.md` (extract, Sonnet; Haiku when throttled) writes `CONFIDENCE: x` on its first line, then one to four bullets, or `NONE`.
  - Confidence bands: `< 0.4` or `NONE` emits nothing; `0.4-0.7` emits with `payload_json.surfaced = false`; `> 0.7` with `surfaced = true`. `core-api` re-derives `surfaced` (ADR-0030 D3).
  - Streaming (ADR-0030 D4): deltas are held until the confidence line is parsed; only a surfaced note goes to `agent-stream`, and the confidence line itself never does. `AgentState` gains a `stream` hook that the runtime binds to `Emitter.delta` per window and participant.
  - The model sees the participant as `You` and everyone else as `Participant N`; no user id, name or email is put in the request.
  - Registered per participant in `agents/__init__.py`; both prompts in `prompts/registry.json`.
  - Golden-set eval: `tests/eval/personal_listener_golden.json` (14 labelled windows across five roles, labels written before any model ran), `tests/eval/personal_listener_baseline.json` (targets 0.75 precision and recall, marked `measured: false`), `tests/test_personal_listener_eval.py` (`pytest -m eval`, live only with `EVAL_LIVE=1`; writes `tests/eval/personal_listener_last_run.json`, gitignored). The `eval` marker is registered in `pyproject.toml`.
  - `tests/test_personal_listener.py`: 15 tests (now 16, below). The zero-Postgres test now also scans `pii.py` and `llm_cache.py`.
- `46cdf9c`: `PromptLoader` normalizes CRLF to LF. On a Windows checkout with `core.autocrlf` the `---user---` line became `---user---\r` and no prompt would load.

### mytheus-realtime

- `34501d0`: fix to `transcription-service` migration `e4a7c91b3d28`. On a database built from empty in or after 2026-11, `be66698dbefb` (which premakes months from `date.today()`) and `e4a7c91b3d28` (fixed months from 2026-11) both created `transcript_chunks_2026_11`, and the upgrade failed with `DuplicateTableError`. The migration now skips a month that already exists. Found while bringing up a fresh local database for billing.
- `d8a6600`, `src/agentOutput.ts` (plan section 3.4 steps 2 and 4, ADR-0030 D5): one consumer group, `realtime-gateway-agent-output`, over `agent-stream` and `agent-results`, so each record is handled by one pod. That pod publishes it on the meeting's existing `nexus:{org_id}:pubsub:meeting:{meeting_id}` channel tagged with the owner's `user_id`, and every pod emits it only to local sockets whose core-api-resolved internal user id matches. New socket events `agent-delta` (volatile emit, shed first) and `agent-note` (reliable). Forwarded: stream deltas and `PersonalListenerAgent` notes with `payload_json.surfaced = true`. Skipped: silent notes, other agents and kinds, records without UUID ids. Nothing is buffered or replayed. On by default (`AGENT_OUTPUT_ENABLED=0` turns it off); off for in-process tests.
- `test/agent-output.test.ts`: mapping rules, and real Redis + Socket.io delivery to the owner but not to another participant, nor to another org whose meeting has the literal same id and whose user has the same id.
- `d38a120`: restores `package-lock.json`, which `d8a6600` had committed after a local Windows `npm install` rewrote it. No dependency changed.
- `9744bf9`, `test/agent-output-kafka.test.ts`: real Avro `agent-stream` and `agent-results` records on the local Redpanda reach only the owner's socket through the gateway's real consumer.

### mytheus-ui

- `6d7d0bf`, work item 4 (plan section 3.4):
  - `src/lib/realtime/notes.ts`: a pure reducer. Deltas build a draft per agent window, the final replaces it, a stale or late delta never overwrites a final, re-hydrate merges by window, order is by window start.
  - `useMeetingSocket` subscribes to `agent-delta` and `agent-note` and re-hydrates notes from `/api/meeting-worklogs` on every (re)connect, the same rule as captions.
  - `src/app/api/meeting-worklogs/route.ts` over `core-api`'s worklogs route through the server-only client; `src/lib/api/worklogs.ts`.
  - `AgentSidebar` renders the live notes in "My notes" (a draft shows "Writing..."), and no longer shows the empty state once a note exists. `MeetingRoom` passes them in.
  - `tenant/usage` reads `GET /usage` (`src/lib/api/usage.ts`): the token meter from the live budget, meeting minutes against the plan, transcription and storage without a plan limit, the Haiku/Sonnet/Opus split from `usage_aggregates`, and the throttled and blocked states. Sample figures only when no gateway is configured; a non-admin is told usage is admin-only.
  - Generated API types refreshed (`worklogs`, `billing-metering-service`).
  - `npm test`: `node --test` with type stripping over `tests/notes.test.ts` (4 tests). The repo had no test runner; `tests/` is excluded from `tsconfig.json` because it imports `.ts` paths.
- `a82dfbb`: a gateway that answers 404 for `/usage` (a deployed backend older than Plan 7) or 502/503 renders "not available right now" instead of crashing the page.

## 3. Definition of Done audit (End of P7)

The plan's section 10 "End of P7" DoD and its demo, line by line, as of 2026-11-08. "Faked model" means the real code path ran with the model call replaced by a scripted answer; no real-model behavior is verified (section 5a item 1).

| DoD line | Status | Evidence |
|---|---|---|
| `billing-metering-service` shipped: plans/subscriptions/invoices/usage_events/usage_aggregates/agent_budgets/quotas migrated with RLS FORCE | MET | Migration `4c1d8e2a9b70`; `test_rls_isolates_usage_and_budgets`; `tests/test_roles_and_policies.py` grant and partition tests |
| `usage-events` consumer with idempotent dedupe | MET | `test_a_message_is_metered_once_and_mirrored`; real Redpanda runs in section 6 |
| Redis budget mirror | MET | Same test; the end-to-end run read it back at 1,536 tokens |
| `ok -> throttled -> blocked` state machine | MET | `test_free_goes_straight_to_blocked_at_its_cap`, `test_pro_throttles_at_quota_and_blocks_at_twice_quota` |
| alert-only -> enforce flag | MET | `BUDGET_ENFORCEMENT_MODE` in the mirror; `test_alert_only_never_degrades` on the agent side |
| `agent-runtime-service` shipped: windowing | MET | `test_360_finals_over_45_minutes_make_90_windows` |
| Haiku pre-gate | MET | `personal_listener_gate@v1`; `test_a_window_below_the_relevance_threshold_makes_no_extraction_call` |
| LiteLLM tiered routing | MET | `test_throttle_forces_haiku_and_ok_routes_extract_to_sonnet`, `test_high_stakes_is_refused_while_unmapped`; end-to-end run routed 2 Haiku + 2 Sonnet |
| prompt caching | MET | Anthropic prompt caching on the system prompt, plus the exact-match response cache (ADR-0030 D1, `test_an_identical_call_is_answered_from_the_cache_and_meters_nothing`; the end-to-end replay made zero model calls). Embedding-similarity caching is deferred by ADR-0030 |
| inline `check_budget` | MET | `test_gate_decides_from_the_mirror`, `test_a_miss_reads_through_and_every_failure_blocks`; end-to-end run read through the real billing endpoint |
| `usage-events` emit | MET | Router tests; end-to-end run: 4 calls, 4 `usage_events` rows |
| `agent-stream` bridge | MET | Agent: `test_only_a_surfaced_note_streams_and_never_its_confidence_line`; gateway: `test/agent-output-kafka.test.ts` (real broker to socket) |
| writes zero Postgres | MET | `test_the_runtime_imports_no_database_driver` (now also scans `pii.py`, `llm_cache.py`) |
| `PersonalListenerAgent` per-participant | MET | `test_one_personal_listener_per_participant_per_meeting` (3 participants, 2 windows, 3 instances reused) |
| confidence gating | MET | Agent: `test_emitted_notes_carry_their_band`, `test_low_confidence_or_empty_notes_produce_nothing`; `core-api`: `test_the_confidence_band_decides_what_is_written`, `test_tenant_overrides_decide_the_band`; end-to-end: 0.92 stored surfaced, 0.55 stored silent |
| LLM-egress PII redaction | MET (regex scope) | `test_pii_is_redacted_before_egress_unless_the_tenant_turned_it_off`, `test_pii_is_redacted_and_no_user_id_reaches_the_model`; end-to-end: `alice@acme.com` never reached the model. Names and other free-text PII are not detected (gap 9) |
| `worklogs` migrated in `core-api` | MET | Migration `d5e2f1a8c370`; `test_rls_hides_another_tenants_worklogs`; partition coverage test |
| My Notes + agent sidebar shell + Usage console shipped | PARTIAL | Built, typechecked, linted and `next build` green; reducer tests pass. NOT exercised in a browser: every page needs a Clerk sign-in, which this session could not do (gap 8) |
| mobile My Notes wired | NOT MET | Item 5, excluded by the maintainer for this session |
| pre-gate count test | PARTIAL | The mechanism is tested (no extraction below the threshold). The live eval that measures precision and recall on the golden set has not run (section 5a item 1) |
| zero-Postgres test | MET | As above |
| budget deny-by-default test | MET | `test_block_calls_no_model_and_meters_nothing`, `test_a_blocked_budget_returns_nothing` |
| RLS isolation test | MET | `test_rls_isolates_usage_and_budgets`, `test_rls_hides_another_tenants_worklogs` |
| DECISION-REQUIRED #1 + #2 resolved | MET | ADR-0029 D1, D3 |

Demo, against the local stack: a meeting's finals on `transcript-chunks` produced one note per participant, metered on `usage-events`, persisted to `worklogs` with the right band, and the gateway delivered a real-broker note to its owner only. Not shown: the browser rendering it, a real model, an 80/95% alert firing in the same run (the alert path is covered by item 1's tests), and a Free tenant hitting `blocked` mid-meeting (covered by tests, not demonstrated).

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
11. **The semantic cache is an exact-match cache on normalized input** (superseding the item 2 entry "no semantic cache"). ADR-0030 D1: an embedding-similarity lookup would add an `embedding-service` round trip per window per participant for a hit rate nobody has measured; the concrete P7 win is that a replayed window costs nothing. Same Redis namespace if similarity is added later.
12. **No per-user priority queue.** `nexus:{org_id}:queue:agent:{user_id}` belongs to `notification-service` in the backend service map. Windows run one at a time per process, with users concurrent inside a window.
13. **The runtime emits no `billing-events`.** The plan's `check_budget` sketch emits `quota_exceeded`; canon makes `billing-metering-service` the only producer, and it already does so on the crossing.
14. **Meeting end is "last participant left".** `meeting-room-events` has no room-finished symbol (`joined`, `left`, `recording_ready`); idle close covers a meeting nobody leaves cleanly.
15. **Window speakers join the roster.** A runtime restarted mid-meeting missed the `joined` events; anyone who talks is served anyway.
16. **The budget can overshoot slightly.** The mirror advances when billing meters a call, which lags the call, so concurrent calls in that gap all see the older total. The overshoot is bounded by one window's calls for the tenant.
17. **`core-api` claims the inbox row before the effect, not after.** The backend recipe (and `libs/shared/inbox.py`) records the inbox row after the effect with `ON CONFLICT DO NOTHING`. That only dedupes the inbox row: once the 24h Redis key has expired, a redelivery would write a second note. Claiming first, in the same transaction, makes the inbox the authority; a rollback releases both. Found by `test_a_redelivered_result_writes_nothing_more` failing on the first run.
18. **`core-api` decides `surfaced`, not the agent** (plan step 2 puts confidence gating on emit). ADR-0030 D3: only `core-api` can read `tenant_settings.agent_confidence_overrides`. The agent still drops `< 0.4`, so a tenant override can only tighten.
19. **`worklogs` carries `window_id`.** Not in the plan DDL. It traces a note to its agent window, and it is the key the UI merges live and re-hydrated notes on. Canon updated first (`c2bac53`).
20. **The relevance threshold is 0.5, from an environment variable, not per tenant.** Doc 08 says "per-tenant relevance threshold" with no number; the AI tier cannot read `tenant_settings`. ADR-0030 D2. The eval harness is where the number gets tuned.
21. **The window is anonymized for the model.** Not asked for by the plan. The participant is "You" and others are "Participant N", so no user id, name or email ever reaches the provider; the PII redactor still runs on top.
22. **The extraction prompt puts confidence on the first line.** The plan sketch has the model return `note.text` and `note.confidence` as fields. A first-line header lets the agent decide whether to stream before any note text is shown (ADR-0030 D4); a JSON object would have to complete before its confidence could be read.
23. **Live notes go over the meeting's pub/sub channel with a per-user filter, not a per-user room.** ADR-0030 D5: no new Redis key and no second authz path on join.
24. **Usage page: no daily chart on live data.** `GET /usage` serves period totals; the daily burn chart is shown only for the sample figures, and a budget-state card takes its place on live data.
25. **The UI tests run on `node --test`, not Vitest.** `mytheus-ui` has no test runner installed (its CLAUDE.md describes Vitest, which was never added). Node's built-in runner with type stripping needed no new dependency.

## 5. Open gaps

1. ~~`storage_gb` is not derived.~~ Closed in `73348fc`.
2. ~~No Temporal schedule.~~ Closed in `73348fc`.
3. ~~Mirror read-through for the agent tier.~~ Closed in `5f26090` (`POST /billing/internal/budget`).
4. **Stripe checkout, `/stripe/webhook`, dunning, invoices** are left to their own band, as the plan says.
5. **Local environment only** (not code defects): `test_public_schema_grants_nothing_by_default` and `test_no_bypassrls_role_holds_privileges_on_a_public_table` fail on this machine's database; six `test_crypto_file_custody.py` tests fail on Windows (no 0400 file modes); `test_relay_real_bus_roundtrip` fails under the 768M Redpanda cap; `mypy` reports `os.geteuid` in `libs/shared/crypto/providers.py` on Windows; three `realtime-gateway` multi-process tests (`cross-pod-captions`, `cross-pod-fanout`, `presence-pod-death`) fail on Windows with `spawnSync npx ENOENT`; `prettier --check` fails on 17 `realtime-gateway` files on `main` because of CRLF checkouts.
6. ~~LLM-egress PII redaction.~~ Closed in `0533c14`.
7. **Deploying P7** means three more processes and new settings, none of which exist in `mytheus-platform` yet: `python -m services.agent_runtime_service.consumer` (`BILLING_URL`, `BILLING_SERVICE_TOKEN`, `IDENTITY_GRPC_ADDR`, `ANTHROPIC_API_KEY`, `ANTHROPIC_WORKSPACE_ID`, `REDIS_URL`, `REDPANDA_BROKERS`, `SCHEMA_REGISTRY_URL`, optionally `PERSONAL_LISTENER_RELEVANCE_THRESHOLD`); the `core-api` consumer, which now also reads `agent-results`; billing's consumer and Temporal worker; `MYTHEUS_BILLING_CALLER_TOKENS` on billing; `realtime-gateway` needs nothing new (`AGENT_OUTPUT_ENABLED` defaults on). Migration `d5e2f1a8c370` must run.
8. **The UI is not verified in a browser.** The sidebar's live notes and the Usage page were built and `next build` passes, but every page needs a Clerk sign-in, and the dev preview points at the deployed `api.mytheus.cloud`, which does not have this work. To verify: run the local stack, point `API_GATEWAY_URL` and `NEXT_PUBLIC_REALTIME_WS_URL` at it, sign in, open a meeting room and `/tenant/usage`.
9. **PII redaction is regex-only.** Emails, phone numbers, SSN- and card-shaped numbers are caught; names, addresses and other free text are not. The stronger path is `ner-service` (ADR-0020), which is built but not called from the egress gate.
10. **Per-tenant relevance threshold not built.** Needs `tenant_settings` mirrored to Redis for the AI tier (ADR-0030 D2).
11. **A tenant override can hide a note the live sidebar already showed.** The gateway forwards on the default bands; `core-api` applies the override. The note disappears on the next re-hydrate (ADR-0030 consequences).
12. **The `presence-events` handler has the same latent unknown-tenant crash** that `97f2709` fixed for `agent-results`: `record_consumed` hits the inbox foreign key for a tenant with no `organizations` row and raises `IntegrityError`, which the loop does not catch. Not changed this session (out of P7 scope); the fix is the same `PoisonMessage` wrap.
13. **The worklogs route has no `bff` path and the dashboard's `worklogs` arm is still `empty`.** The UI calls `core-api` through `api-gateway` directly, as it does for the transcript. Plan 3.4 says "through `bff`"; the `bff` dashboard arm can now read real notes and was not wired.
14. **Agent notes in the meeting room are the only live agent output.** The sidebar's decision, question and conflict cards still render nothing until P8 and P9 emit them.

## 5a. What is left in P7, and the next session's start list

In the order it should be picked up:

1. **Real-model check: blocked on the Anthropic workspace.** `mytheus-intelligence/.env` has `ANTHROPIC_API_KEY` but no `ANTHROPIC_WORKSPACE_ID`, and the key is not workspace-scoped, so Anthropic refuses every call with 400 before any tokens are used. Add `ANTHROPIC_WORKSPACE_ID` (Anthropic console, Settings, Workspaces) or swap in a workspace-scoped key. Then run, in order: the capped one-window check from the item 2 session, and `EVAL_LIVE=1 uv run pytest -m eval` (14 Haiku calls). Record the measured precision and recall in `tests/eval/personal_listener_baseline.json` and set `measured: true`. Until then no real-model behavior, prompt quality or streaming from a real provider is verified.
2. **Browser check of the UI** (gap 8): the meeting room's My Notes with the local stack and a signed-in user, and `/tenant/usage` as an admin and as a member.
3. **Item 5, `mytheus-mobile`** (plan section 7, steps 2 and 7): My Notes in the live-session screen over the same REST route and socket events, the recap and insights screens off `src/data/mock.ts`, and the budget-state messaging. Excluded from this session by the maintainer.
4. **Deployment** (gap 7): code, routes and migrations are deployed (section 7). Left: stage the agent secrets and enable the `workers`, `agents` and `realtime` profiles on the VPS.
5. **Smaller follow-ups:** the presence-events unknown-tenant fix (gap 12), the `bff` dashboard `worklogs` arm (gap 13), NER on the egress path (gap 9).

After that, P7 is done and P8 (`DecisionAgent`, `ActionAgent`, confirm endpoints, `knowledge-graph-service`) is next, starting with DECISION-REQUIRED #3 (AGE).

## 6. Verification

All observed 2026-11-08 against the local stack (Postgres 5443, Redis 6390, Redpanda 9092 with Schema Registry 8085; Redpanda capped at 768M through a compose override kept outside the repo).

- `mytheus-events`: `uv run python codegen/validate.py schemas` OK; codegen regenerated; `uv run pytest tests -q` passed except `test_create_topics_is_idempotent` (bash cannot resolve the Windows script path on this machine); `node --test tests/ts/*.test.mjs` 1 passed; `codegen/check_compat.py` OK (21 subjects BACKWARD); `codegen/register.py` OK; `register.py --check` all 21 in sync.
- `mytheus-backend`: `MYTHEUS_REQUIRE_DB=1 uv run pytest tests/test_billing_metering.py -q` 20 passed. Full suite (two environment tests deselected): 611 passed, 7 failed, all seven environmental (gap 5). `uv run ruff check`, `uv run black --check`, `uv run mypy services/billing_metering_service` clean.
- `mytheus-realtime`: `uv run alembic upgrade head` on the local database now succeeds through `e4a7c91b3d28`.
- `73348fc`: `tests/test_billing_metering.py` 23 passed. Real Temporal (`docker compose up -d temporal`) and MinIO: a throwaway script started both workers in-process, `ensure_schedule` returned True then False (idempotent), the schedule's next action was 2026-11-09 02:00 UTC, and one manual trigger ran `billing.run_nightly` to completion (`{'tenants': 4266, 'storage_rows': 4265, 'derived_rows': 0, 'failed': 0}`; the tenants are mostly test fixtures left in the local database).
- `2649f21`: `uv run pytest tests/test_agent_runtime.py -q` 27 passed; `ruff check`, `black --check`, `mypy libs services` (80 files) clean.
- **Runtime round trip, no LLM spend** (throwaway scripts, not committed): a tenant's budget provisioned through the real `POST /billing/internal/budget` (200, quota 200000); the real `Emitter`, `AvroCodec` and `BudgetGate` on real Redis, with the test suite's two-step agent and a fake model, ran one window for 3 participants. Produced to real Redpanda against the registered schemas: 3 `agent-results`, 6 `usage-events`. The offset tracker held back the third chunk (its window still open). The real billing consumer then metered 6 rows, 720 tokens, tiers haiku and sonnet; `agent_budgets` and the mirror both read 720.
- **Real round trip** (throwaway scripts, not committed): two copies of one Avro `usage-events` message produced to Redpanda; `python -m services.billing_metering_service.consumer` run for 25 s; `python -m services.billing_metering_service.jobs` run once. Result: one `usage_events` row (1234 tokens, `haiku`); `agent_budgets` 1234 of 200000, `ok`; `usage_aggregates` `agent_token` 1234 with `{"haiku": 1234}`; the Redis mirror read back with `enforcement` `alert_only`. The first attempt sent every real message to the DLQ: the confluent deserializer returns `occurred_at` as a datetime, not an integer. Fixed in `734f80e` with a regression test; the unit tests had not caught it because they built the value by hand.

Items 3 and 4, observed 2026-11-08 on the same local stack:

- `mytheus-intelligence`: `uv run pytest tests/test_agent_runtime.py tests/test_personal_listener.py tests/test_personal_listener_eval.py tests/test_pii.py -q`: 66 passed, 1 skipped (the live eval, by design). `ruff check`, `black --check`, `mypy libs services` (83 files) clean.
- `mytheus-backend`: `MYTHEUS_REQUIRE_DB=1 uv run pytest tests/test_worklogs.py tests/test_partition_coverage.py tests/test_roles_and_policies.py tests/test_openapi_contract.py tests/test_presence_events_consumer.py -q`: all passed except the two known local-database tests (gap 5); `test_worklogs.py` 10 passed. `uv run alembic -c migrations/alembic.ini upgrade head` applied `d5e2f1a8c370` (15 `worklogs` partitions). `python -m scripts.emit_openapi --check`: specs match. `ruff`, `black --check` clean; `mypy` clean apart from the Windows-only `os.geteuid` (gap 5).
- `mytheus-realtime`: `npx vitest run test/agent-output.test.ts test/agent-output-kafka.test.ts test/captions.test.ts test/gateway.test.ts`: all passed, the Kafka test against the real Redpanda and Schema Registry. Full suite: 48 passed, 3 skipped; 3 files fail only on `spawnSync npx ENOENT` (gap 5). `tsc --noEmit` and `eslint` clean.
- `mytheus-ui`: `npx tsc --noEmit`, `next lint`, `npm run generate:api:check`, `npm test` (4 passed) and `npx next build` all green. Not opened in a browser (gap 8).
- **End-to-end, faked model, everything else real** (throwaway scripts in the session scratchpad, not committed): one tenant, two users (roles "Backend engineer" and "Product designer") and one meeting seeded; `billing-metering-service` run on 8006 with a throwaway caller token; three finals produced as real Avro to `transcript-chunks`, one carrying `alice@acme.com`. The real `Loop`, `AgentRuntime`, `PersonalListenerAgent`, prompts, router, redactor, response cache, budget gate (read through the real billing endpoint) and emitter processed them: 2 Haiku gate calls and 2 Sonnet extractions, no request contained the email or any user id. `python -m services.core_api.consumer` then wrote two `worklogs` rows: the engineer's at confidence 0.92 `surfaced = true` with the email shown as `[REDACTED_EMAIL]`, the designer's at 0.55 `surfaced = false`, each with 3 source chunk ids. `python -m services.billing_metering_service.consumer` metered 4 rows (haiku 616, sonnet 920 tokens); `agent_budgets` and the Redis mirror both read 1,536. Replaying the same three finals made zero model calls (4 `llmcache` keys, one per prompt per participant) and `core-api` still held exactly 2 notes. The first `core-api` run crashed on a stale record for a deleted tenant, which is how `97f2709` was found.

## 7. Deployment to api.mytheus.cloud (2026-11-08)

**Why nothing had deployed.** `mytheus-backend`'s `deploy` job runs only when every CI gate is green on a push to `main`. From the first billing commit onward the `test` job failed on one test, `test_meeting_usage_is_derived_once_and_rolled_up_with_the_tier_split`, with `relation "transcript_chunks" does not exist`: that table is created by `mytheus-realtime`'s migrations, which CI's database never runs. So every Plan 7 push skipped `deploy`, and production stayed on the 2026-11-06 build. The same assumption would have failed the nightly rollup on a server without the transcription tier.

What landed to fix and extend the deploy path:

- `mytheus-backend` `6fcde90`: `derive_meeting_usage` derives nothing when `transcript_chunks` does not exist (`to_regclass`) and the rollup still runs; the test covers both cases. CI run `42496296920` was fully green and its `deploy` job ran: the server pulled the new image, recreated every backend container, `migrate` exited cleanly (the billing tables and `worklogs`), and the public `/healthz` check passed. `/usage` and `GET /api/v1/meetings/:id/worklogs` are therefore live.
- `mytheus-backend` `a2f9393`: the image now generates `identity_pb2` / `identity_pb2_grpc` (they were never in it, so the `get_user_context` gRPC server could not start in production). Verified by building the image locally and importing the server; CI run `42497095456` green and deployed.
- `mytheus-platform` `b93016b`: `billing-consumer` and `billing-temporal-worker` (profile `workers`), `identity-grpc` and `agent-runtime` (profile `agents`) in `deploy/compose.prod.yml`. All opt-in: `docker compose config --services` lists exactly the same default services as before. `scripts/push-secrets.sh agents` generates one billing caller token and writes it to `prod.env` and `prod.agents.env` with the Anthropic key (and workspace id when set), validating every value before touching the server.
- `mytheus-intelligence` `8d7704c`: the CPU image workflow (PR #3, never merged) merged to `main` at the maintainer's instruction, so CI publishes `ghcr.io/mytheus-ai/mytheus-intelligence:main` with the agent runtime and its prompts.
- `mytheus-ui`: its own pipeline redeployed the `ui` container from `a82dfbb` (seen running in the backend deploy log), so My Notes and the Usage page are live.

**What is NOT running in production, and why.** The deploy log's `docker ps` shows only the default services: no `realtime-gateway`, no Kafka consumers, no agent. The server's `COMPOSE_PROFILES` does not enable `workers`, `realtime` or `agents`, and changing it, or staging secrets, needs a shell on the VPS, which this session did not have (a read-only SSH probe was refused by the session's permission guard). Until those are enabled, production has the routes and tables but nothing produces notes, and the live socket is not served.

To switch the agent tier on, from the maintainer's machine:

1. `MYTHEUS_INTELLIGENCE_ENV=<path to mytheus-intelligence/.env> scripts/push-secrets.sh agents` (add `ANTHROPIC_WORKSPACE_ID` to that `.env` first, see section 5a item 1).
2. On the VPS, add `workers,agents,realtime` to `COMPOSE_PROFILES` in the deploy environment (the `workers` services also need their `prod.db.*-consumer.env` role files, see `deploy/README.md`).
3. Redeploy (any push to `mytheus-backend` `main`, or the platform deploy workflow).

## 8. Production run and follow-ups (2026-11-09)

The first real meetings on `dashboard.mytheus.cloud` (three people) surfaced the issues below. Each fix is its own PR; this records what and why.

**Agent tier, found in production.**

- The real-model check (section 5a item 1) is still blocked, now for a different reason: `ANTHROPIC_WORKSPACE_ID` is set, but Anthropic answers `authentication_error: invalid x-api-key` for the key in `mytheus-intelligence/.env`, the same key staged on the VPS. The live eval (`EVAL_LIVE=1 uv run pytest -m eval`) fails on the first call. A new key is needed; nothing else is.
- That bad key exposed a real defect: `mytheus-agent-runtime-1` had 3,149 restarts. A provider error escaped `process_window`, the consumer died before committing, and replayed the same window on every restart. Fixed in `mytheus-intelligence` PR #4: a failing agent call is logged and skipped for that window, so the window completes and its offset commits. Test first.
- The pipeline itself is confirmed live: the test call produced 137 `transcript_chunks`, the runtime built windows and called the model; 0 `worklogs` only because every call was refused.

**Plan 7 gaps 12 and 13** (`mytheus-backend` PR #9, `mytheus-ui` PR #25).

- Gap 12: the `presence-events` unknown-tenant crash is now `PoisonMessage`, as `agent-results` already was.
- Gap 13: `/bff/dashboard` `worklogs` is a typed, real arm (the caller's surfaced notes across meetings, newest first), and the dashboard has a Your notes panel. `action_items` and `decisions` stay `UnbuiltArm` until P8.

**Transcription** (`mytheus-realtime` PR #4).

- The Deepgram request named no model, so it ran on Deepgram's older default. Now `model=nova-3`, `language=multi` (English/Hindi code-switching, chosen by the maintainer), `punctuate`, `smart_format`, `endpointing=300`, `utterance_end_ms=1000`; `DEEPGRAM_MODEL` / `DEEPGRAM_LANGUAGE` override it.
- Results with no words were saved as blank `transcript_chunks` rows (the call's transcript showed 137 lines, many empty); they are now dropped before persistence and captions.
- Every line's time was relative to its own participant's stream, so all lines sat between 00:00 and 00:04. A participant's offset is now fixed when they join, on top of where the meeting's saved transcript already ends.

**Meeting lifecycle** (`mytheus-backend` PR #8, `mytheus-platform` PR #7, `mytheus-ui` PR #24).

- Nothing moved a meeting off `scheduled`, so a finished call still offered Join. LiveKit rooms are now created with `empty_timeout` and `departure_timeout` of 300 s; `meeting-service` reports `room_started` and `room_finished` to a new service-authenticated core-api route (`POST /internal/meetings/{id}/room-state`, `MYTHEUS_CORE_CALLER_TOKENS`), which moves the meeting to `live` and then `ended` with the usual `meeting-events` `ended` record. Transitions only go forward, and the token route refuses an ended meeting (409).
- No canon change: `meetings.status` and its four values already exist; the route is internal and not in `api-gateway`'s map. The new audit row uses `actor_type = 'system'`.
- The meeting page was reorganised (status by the title, people by name, one action card; recap and talk time by name after the call), and a scheduled meeting 5+ minutes past its end with nobody joining shows as "Didn't happen" with no Join.

**Names.** Every identity in a call is an internal user id and LiveKit tokens carry no name, so the room, chat, captions and transcript showed "Guest", raw ids or "Speaker 07b3...". The UI now resolves names from `GET /users` plus Clerk (`mytheus-ui` PRs #20, #24). The sidebar shows the workspace's Clerk name instead of its slug.

**Still open after this.** Mobile (item 5) and NER on the egress path (gap 9). Nothing above is a P8 or P9 agent; those are next.

**Correction, later the same day.** The Anthropic key blocker above was resolved after this section was written:

- The live eval ran: `personal_listener_gate@v1` on `claude-haiku-5-5`, threshold 0.5, over the 14-window golden set, precision 1.0 and recall 1.0 for 6,034 tokens. Recorded in `mytheus-intelligence` `tests/eval/personal_listener_baseline.json` (`measured: true`, `d2dc3b8`). Section 5a item 1 is closed.
- Production's key is valid: a `GET /v1/models` made from inside `mytheus-agent-runtime-1` with its own environment answered 200 (status only; no key value was read or printed).
- Production runs the fixes: `transcription-service`, `realtime-gateway` and `agent-runtime` restarted at 06:45 UTC on 2026-11-09, after `mytheus-realtime` PR #4 merged at 06:43 UTC, and the running transcription image contains the `nova-3` request; the `agent-runtime` has 0 restarts since. `worklogs` is still empty only because no meeting has run since; the next real call is the first end-to-end check of My Notes against a real model in production.

## 9. P8 Part B: the knowledge graph, decision embeddings, My Tasks and lineage (2026-11-09)

Phase 8 is split between two people working in parallel. Part A owns `DecisionAgent` / `ActionAgent`, the `decisions` / `action_items` / `agent_suggestions` migrations, the confirm, edit and dismiss endpoints that produce the `decisions` / `actions` events, and the sidebar confirm cards. Part B, recorded here, owns everything downstream of a confirmed event: `knowledge-graph-service`, the `decisions__{org_id}` embeddings, the My Tasks read path and the decision-lineage read path, and their two UI pages. Part A had landed nothing when this was built, so every Part B piece was developed and tested against fixture events encoded with the registered Avro schemas and, in `core-api`, against a stand-in `action_items` table with the plan's DDL.

Status of Part B: built, tested against real infrastructure, committed to each repo's `main`, **not pushed and not deployed**.

### 9.1 Entry decisions

Recorded canon first in [ADR-0031](../adr/0031-plan-8-graph-store-decision-supersedes-and-mark-done.md), decided by the maintainer 2026-11-09:

| | Decision | Why |
|---|---|---|
| D1 | Production runs `nexus_knowledge` in a dedicated self-hosted `apache/age` Postgres, addressed by `AGE_DATABASE_URL`. Closes DECISION-REQUIRED #3 (R1), owner the maintainer. | The production image `supabase/postgres:15.8.1.060` ships no `age.control`. The AGE client, its `$org_id` guards and a live test already existed (Plan 5-6 gap 16 work) against exactly this shape locally. Chosen over the relational fallback (it would discard working code) and over Neo4j Aura (tenant data would leave the VPS). |
| D2 | `Decision` gains an optional `supersedes` (`["null", uuid]`, default null). `decisions` gains `supersedes`. | The frozen `Decision` schema had no lineage field, so `FOLLOWS_FROM` had no source on the bus and the graph could not be rebuilt from rows for lineage. BACKWARD-safe, same shape as ADR-0029 D4. |
| D3 | `POST /api/v1/actions/{id}/done`, confirmed to done, emits nothing. | My Tasks needs mark-done and the plan names no route. The `actions` topic carries confirmations only (`status` enum `{confirmed}`). |
| D4 | The `Decision` graph node carries `meeting_id`, `content`, `confirmed_at`. | The lineage view can be answered from one template call. Rebuildable like every other property. |
| D5 | `embedding-service` is a `decisions` consumer. | Canon already said `decisions__{org_id}` is embedded on confirm; the taxonomy row did not list the consumer. |
| D6 | The Redis dedupe key is `nexus:{org_id}:dedupe:{topic}:{consumer_group}:{idempotency_key}`. | Found while adding the second `decisions` consumer: see deviation 1 below. |
| D7 | `knowledge-graph-service` consumes `meeting-room-events` and writes `ATTENDED` on `joined`. | Canon creates `ATTENDED` "when a participant joins" and names no source. |

### 9.2 What landed, per repo

**mytheus-docs**

- `c95c019`: ADR-0031 D1 to D5; `_canon/02-data-model.md` (`decisions.supersedes`, the `action_items` done transition, the graph's physical home, the `Decision` node properties), `_canon/03-event-taxonomy.md` (`decisions` consumers and `supersedes`), `_canon/06-tech-decisions.md` (Graph store row), the Glossary, the ADR index, the R1 status note in the Risk Register, and the RESOLVED marker on the plan's DECISION-REQUIRED #3.
- `74da45e`: ADR-0031 D6 and D7; the canon Redis row and the `meeting-room-events` consumers; the new dedupe key propagated to the reader docs, the Glossary and diagrams 05 and 12 (plans and audits left as written).

**mytheus-events**

- `783bf4f`: `schemas/decisions/v1.avsc` gains `supersedes`; Python and TypeScript clients and the golden fixture regenerated; checked BACKWARD against the local Schema Registry and re-registered (all 21 subjects in sync).

**mytheus-intelligence**

- `5cb000b`, `libs/shared/inbox.py` + `consumer.py`: the dedupe cache key now carries the consumer group (D6). `tests/test_inbox_dedupe.py` is the regression; the two existing consumer tests use `dedupe_key()`.
- `3d75236`, `knowledge-graph-service` (plan section 3.7):
  - `confirmed.py`: the edge writers. `apply_decision` MERGEs `Meeting`, `Decision` (status, confidence, content, meeting_id, confirmed_at) and `PRODUCED`; with `supersedes` it MERGEs the prior decision, marks it `superseded` and writes `FOLLOWS_FROM`. `apply_action` writes `Action`, `PRODUCED` and, with an assignee, `Person` and `ASSIGNED_TO`. `apply_attended` writes `Person -ATTENDED-> Meeting`. Every node and edge carries `org_id` in its MERGE pattern; one transaction per fact under the existing per-org advisory lock; a node already `superseded` stays so if its own confirmation arrives later. Plus `soft_delete` (sets `deleted_at`) and `purge_org` (the RTBF step).
  - `facts_consumer.py`: `ConfirmedFactsConsumer`, group `knowledge-graph-service.confirmed-facts`, topics `decisions`, `actions`, `meeting-room-events`. Refuses any non-`confirmed` status, takes `org_id` from the header only, skips non-UUID room identities (the audio tap), logs and skips an unusable payload instead of blocking the partition, and enqueues one `graph-events` record per edge (`graph:{edge_label}:{from}:{to}`) in the same transaction as the inbox row.
  - `templates.py`: `decision_lineage` (ancestors deepest first, the decision, descendants), `decisions_by_person` (via `ATTENDED` and `PRODUCED`), `actions_blocked_on_decision`, `related_decisions_for_topic`. `$org_id` on every node and every relationship pattern, `FOLLOWS_FROM*1..10`, at most 100 rows, soft-deleted nodes filtered, each run in a `READ ONLY` transaction with a 2 s statement timeout. `run_adhoc` backs `POST /graph/cypher`: refused without `$org_id`, refused on any write clause, and read-only by transaction anyway.
  - `routes.py` + `main.py`: `POST /graph/query/{template}` and `POST /graph/cypher`. `libs/shared/service_auth.py`: the backend's service-token scheme (`X-Mytheus-Service`, `X-Mytheus-Service-Token`, tenant in `X-Mytheus-Org`), secrets in `MYTHEUS_KG_CALLER_TOKENS`, allowed callers `bff`, `rag-service`, `agent-runtime-service`, constant-time compare, unconfigured refuses everyone.
  - `proto/knowledge_graph.proto` (`KnowledgeGraphService.QueryTemplate`), its runtime mirror `libs/shared/kg_proto.py` (no protoc in the toolchain, same approach as `rag_proto.py`), `grpc_server.py`, `grpc_main.py` (`KG_GRPC_BIND`, default `0.0.0.0:50104`). Same auth, read from metadata.
  - `worker.py` runs the existing Topic seeder and the new consumer on two threads with separate clients (`KG_CONSUMERS` selects). `rebuild.py` re-applies confirmed rows exported from `core-api` as JSON lines through the same writers, with `--purge`.
- `ad2c144`, `embedding-service`: `decision_consumer.py`, `DecisionEmbeddingConsumer` (group `embedding-service.embed-decisions`). One point per decision (`uuid5` of `decision:{id}`, so a redelivery overwrites), vector of content plus rationale, payload exactly canon's (`org_id`, `decision_id`, `meeting_id`, `status`, `confidence`); a superseding decision re-labels the prior point `superseded` by payload filter. The worker runs embed-on-ingest and embed-decisions on two threads.

**mytheus-backend**

- `84a8ead`:
  - `services/core_api/routes/my_tasks.py`, in its own module so it cannot collide with Part A's routes: `GET /api/v1/actions?assignee_id=me&status=confirmed|done` (the caller's own, with the meeting title, soonest due first, undated last; `assignee_id` accepts only `me`) and `POST /api/v1/actions/{id}/done` (assignee or admin; idempotent; `confirmed` only, else 409; audit `action.completed`; no outbox row). `libs/shared/audit.py` gains `ACTION_ACTION_COMPLETED`.
  - `services/bff/lineage.py`: `GET /bff/meetings/{meeting_id}/decisions/{decision_id}/lineage`. RLS check on the meeting first, then the graph's `decision_lineage` template with `bff`'s token and the caller's org; 404 for an unknown decision or one from another meeting; 503 `DEPENDENCY_UNAVAILABLE` when the graph is down or unconfigured (`KNOWLEDGE_GRAPH_URL`, `MYTHEUS_KG_SERVICE_TOKEN`).
  - OpenAPI re-emitted (`bff.yaml`, `core-api.yaml`). `api-gateway` already routed `/api/v1/actions` and `/bff`.

**mytheus-ui**

- `573b1d2`: `/tasks` reads `GET /api/v1/actions` (open and done) and completes through the `markTaskDone` Server Action, optimistic with rollback and the server's reason; done is final when live. Real items may have no due date and no priority, so the tasks view model (`src/lib/tasks.ts`) makes both nullable; undated tasks group as Later and sort last; the dates run on the real clock (the mock's fixed `TASKS_NOW` is gone); the High priority filter hides when no task has a priority. The lineage page `(app)/meetings/[meetingId]/decisions/[decisionId]` reads the `bff` route and renders the `FOLLOWS_FROM` chain oldest first, each earlier or later decision linking to its own page. Both pages keep sample data when no gateway is configured. Generated API types refreshed (this also picked up `identity-tenant-service` changes from the 2026-11-08 self-serve work that had not been regenerated). `tests/tasks-dates.test.ts`: 3 tests.

**mytheus-platform**

- `03f8442`: profile `kg` now runs `age` (`apache/age:release_PG16_1.6.0`, volume `agedata`, no published port), `knowledge-graph-service` (read API; `prod.kg.env` only, no Qdrant or relational credential, which the old scaffold had) and `knowledge-graph-worker` (adds `prod.db.ai.env` for its inbox and outbox). `bff` gets `KNOWLEDGE_GRAPH_URL`. `scripts/enable-tiers.sh` generates, only when absent, `prod.age.env`, `prod.kg.env` (`AGE_DATABASE_URL`, `MYTHEUS_KG_CALLER_TOKENS=bff=...`) and `MYTHEUS_KG_SERVICE_TOKEN` in `prod.env`. The default service set is unchanged.

### 9.3 Definition of Done audit (End of P8, Part B lines)

| DoD line | Status | Evidence |
|---|---|---|
| `knowledge-graph-service` shipped: AGE `nexus_knowledge` | MET | `ensure_graph` (Plan 5-6); live tests on `apache/age:release_PG16_1.6.0` |
| per-label `org_id` index | MET | `ensure_graph` creates `{label}_org_idx` on all ten label tables (Plan 5-6 work, re-run here) |
| `$org_id` on every template | MET | Every template binds it on every node and edge pattern; `test_a_cross_tenant_traversal_returns_nothing`, `test_bad_requests` (no `$org_id` is a 422) |
| edge-on-confirmation consumer | MET | `test_confirmed_decision_creates_nodes_edges_and_graph_events`, `test_confirmed_action_writes_produced_and_assigned_to`, `test_a_join_writes_attended_only_for_a_user_identity` |
| four named templates | MET | All four exercised in `test_a_cross_tenant_traversal_returns_nothing`; lineage in `test_the_lineage_template_walks_a_supersession_chain_in_any_order` |
| graph-events | MET (enqueued) | One outbox row per edge asserted in the tests above. Publishing is `embedding-relay`'s existing drain of `intelligence.outbox`, not re-run here |
| `decisions__{org_id}` embedded on confirm | MET | `test_a_confirmed_decision_is_embedded_once_and_supersession_relabels` (real BGE, real Qdrant) |
| My Tasks + lineage UI shipped | PARTIAL | Built, typechecked, linted, unit-tested and `next build` green. Not opened in a browser (Clerk sign-in, same as Plan 7 gap 8), and no real confirmed rows exist until Part A lands |
| cross-tenant graph isolation test | MET | `test_a_cross_tenant_traversal_returns_nothing`, `test_lineage_over_http_is_scoped_to_the_header_org` |
| edge-on-confirmation test | MET | As above, plus the consumer refuses a non-`confirmed` status |
| R1 go/no-go recorded with owner | MET | ADR-0031 D1, Risk Register R1 note, plan DECISION-REQUIRED #3 marked RESOLVED |
| mobile My Tasks + lineage wired | NOT MET | Out of Part B's scope (mobile is Sushan's lane) |
| Part A lines (agents, migrations, confirm endpoints, confirm cards, no-silent-commits test) | NOT MINE | Part A |

The brief's own "done when" list: a confirmed decision/action fixture event creates the right nodes and edges (MET); a redelivered event is a no-op, through both the Redis and the inbox path and even past both (MET); a cross-tenant traversal returns nothing (MET); the lineage template walks a supersession chain, delivered out of order (MET); My Tasks and the lineage page render real data (PARTIAL: the full path is tested in pieces, but rendering real data needs Part A's tables and a browser session); tests cover each (MET).

### 9.4 Deviations from the plan, and why

1. **The dedupe cache key changed** (ADR-0031 D6). A real, pre-existing bug: the Redis key had no consumer group while the inbox did, so the second of two groups on one topic silently skipped every message the first had handled within 24 hours. `embedding-service` and `search-service` on `transcript-chunks` were already exposed. Fixed in the shared consumer, so every group in the repo benefits.
2. **The graph is a separate database from the inbox, so the two are not one transaction.** Graph MERGE commits first, then the inbox and `graph-events` rows. A crash between redelivers and re-MERGEs nothing new. ADR-0005's "edge in the same transaction as the confirmed row" cannot hold with D1; rebuildability replaces it.
3. **`actions_blocked_on_decision` means "still-open actions PRODUCED by the decision's meeting".** Canon has no Action-to-Decision edge label, and adding one would break the frozen label set. A real blocking relation needs a source field (`action_items.blocked_on_decision_id`, worksplit E1 decision 2).
4. **`decisions_by_person` walks `ATTENDED`.** "By a person" has no edge of its own; meetings the person attended that produced a decision is the graph-native reading.
5. **The graph never learns `done`.** Mark-done emits nothing by design (D3), so an `Action` node stays `confirmed`, and `actions_blocked_on_decision` can list an action already done. Acceptable until a progress event exists.
6. **Read surfaces use service tokens, not the user's Clerk JWT.** `search-service`'s gRPC arm verifies a Clerk JWT with an internal-org claim that production tokens may not carry. `bff` has already resolved the tenant, so the backend's own service-token scheme is the reliable fit.
7. **`/graph/cypher` is read-only.** ADR-0005 only requires `$org_id`; a write clause is also refused, and the statement runs in a READ ONLY transaction. There is no legitimate ad-hoc writer.
8. **`RELATES_TO` is not written.** The brief lists `PRODUCED`, `ASSIGNED_TO`, `FOLLOWS_FROM`, `ATTENDED`; topic extraction is not built, so `related_decisions_for_topic` returns nothing until it is.
9. **The Tasks page keeps the existing design** and swaps in real data, rather than a new page. Priority is not a column, so it is hidden instead of invented.
10. **No gRPC process in production yet.** `grpc_main.py` exists and is tested; nothing calls it until `rag-service` (P9), so no container was added.

### 9.5 Open gaps and handover

For **Part A** (no Part B code needs to change if these hold):

1. Populate `supersedes` on the `Decision` event when the user says a decision replaces an earlier one, and store it in `decisions.supersedes` (ADR-0031 D2). Without it the graph has no lineage.
2. Keep the event exactly as frozen: `status = confirmed`, idempotency key `decision:{id}:confirmed` / `action:{id}:confirmed`, header `org_id`, `meeting_id` set, `assignee_id` a `users.id` UUID or null, `due_date` an Avro `date` or null.
3. Create `action_items` with at least the plan DDL columns `my_tasks.py` reads (`id`, `org_id`, `meeting_id`, `content`, `assignee_id`, `due_date`, `status`, `confidence`, `created_at`, `deleted_at`) and the `(org_id, assignee_id, status)` index. The status domain must include `confirmed` and `done`. `tests/test_my_tasks.py` then uses the real table automatically and drops its stand-in.
4. Your list route for `/api/v1/actions` must not register a second `GET /api/v1/actions`; extend `my_tasks.py` or use a different query shape, and tell Part B.

Other gaps:

5. **Not pushed, not deployed.** Seven commits across six repos sit on local `main`.
6. **Browser check** of `/tasks` and the lineage page with a signed-in user and the local stack.
7. **Enabling `kg` in production**: run `enable-tiers.sh` with `kg` in `MYTHEUS_PROFILES` (it generates the credentials), then the next `bff` deploy picks up the token. Until then the lineage page answers "history is unavailable".
8. **Pre-existing, not touched:** `tests/test_bff_dashboard.py::test_a_slow_arm_degrades_and_does_not_500` fails on `main` without these changes.
9. **Local AGE port.** `mytheus-intelligence/docker-compose.yml` maps AGE to 5434; on this machine another project held 5434, so the tests ran against a container on 5436 (`AGE_DATABASE_URL` override). No repo change.

### 9.6 Verification

All observed 2026-11-09 on the local stack: Postgres 5443, Redis 6390, Redpanda 9092 with Schema Registry 8085, Qdrant 6333, and `apache/age:release_PG16_1.6.0` on 5436.

- `mytheus-events`: `codegen/validate.py schemas` OK; codegen regenerated; `uv run pytest tests -q` passed except the known `test_create_topics_is_idempotent` (Windows path); `check_compat.py` 21 subjects BACKWARD; `register.py` and `register.py --check` all 21 in sync.
- `mytheus-intelligence`, with `AGE_DATABASE_URL=postgresql://nexus_graph:nexus-graph-dev@localhost:5436/nexus_graph MYTHEUS_REQUIRE_AGE=1`:
  - `uv run pytest tests/test_knowledge_graph_facts_live.py` 9 passed; `tests/test_knowledge_graph_api.py tests/test_knowledge_graph_writer.py tests/test_knowledge_graph_age_live.py tests/test_knowledge_graph_topics.py` 31 passed, 1 skipped (the backend taxonomy copy check).
  - `tests/test_embedding_decisions.py` 2 passed; `tests/test_search_consumer.py tests/test_embedding_consumer.py` 8 passed after the dedupe change; `tests/test_inbox_dedupe.py` 2 passed.
  - `ruff check .`, `black --check --target-version py312 .`, `mypy libs services` (93 files) clean.
- `mytheus-backend`: `MYTHEUS_REQUIRE_DB=1 uv run pytest tests/test_my_tasks.py` 5 passed; `tests/test_bff_lineage.py` 4 passed; `tests/test_openapi_contract.py tests/test_gateway_edge.py tests/test_bff_dashboard.py` 49 passed, 1 failed (gap 8, also failing without these changes); `python -m scripts.emit_openapi --check` matches; `ruff`, `black --check`, `mypy` on the new modules clean.
- `mytheus-ui`: `npx tsc --noEmit`, `next lint`, `npm run generate:api:check`, `npm test` (23 passed) and `npx next build` all green. Not opened in a browser (gap 6).
- `mytheus-platform`: `docker compose -f deploy/compose.prod.yml config --services` lists the same default services as before; with `COMPOSE_PROFILES=kg` it adds `age`, `knowledge-graph-service`, `knowledge-graph-worker`; `bash -n scripts/enable-tiers.sh` OK.

## 10. Phase P8 part A: decision and action candidates (2026-11-09)

P8 is split between two developers working in parallel. Part A (this section) is `DecisionAgent` and `ActionAgent`, the `core-api` confirm path and the meeting-room confirm cards (plan sections 3.5, 3.6 and 3.8 step 1). Part B is `knowledge-graph-service`, the `decisions__{org_id}` embedding, My Tasks (`GET /api/v1/actions` and mark done) and the lineage view, and is recorded by its owner. The seam between them is the `decisions` and `actions` topics, whose schemas Part B consumes.

Canon first: `_canon/02-data-model.md` gains the `decisions`, `action_items` and `agent_suggestions` columns the plan DDL has, and [ADR-0032](../adr/0032-plan-8-decision-and-action-candidates.md) records the build decisions (D1 to D7).

### What landed, per repo

- **mytheus-docs** `42566cf`: the canon rows, ADR-0032 (numbered after part B's ADR-0031, which merged first), the README index, the Glossary row for `agent_suggestions`.
- **mytheus-events**: nothing merged from part A. `Decision.supersedes` landed from part B (`783bf4f`, [ADR-0031](../adr/0031-plan-8-graph-store-decision-supersedes-and-mark-done.md) D2); part A's identical change (`aee3060`, PR #2) was closed as redundant. `core-api` fills the field on confirm (ADR-0032 D1).
- **mytheus-intelligence**:
  - `55aba99`: `agents/candidates.py` (the shared `CandidateAgent` graph: a Haiku gate, then Sonnet extraction to strict JSON, defensive parsing, the 0.7 / 0.4 bands per candidate, deterministic ids `uuid5(ns, "{agent}:{window_id}:{index}")`, "Participant N" labels mapped back to speaker ids, only UUIDs passed through); `agents/decision_agent.py` and `agents/action_agent.py`, both `scope = "meeting"`; four prompts (`decision_gate@v1`, `decision_agent@v1`, `action_gate@v1`, `action_agent@v1`); `tests/test_decision_action_agents.py`. The action prompt carries today's UTC date so "by Friday" becomes an ISO date.
  - `8e3477b`: the eval harness for both agents (doc 08 section 6.3): `tests/eval/{decision,action}_golden.json` (14 windows each: 7 with no item, 5 with one, 2 with two), greedy keyword matching to precision, recall and F1, and the Brier score of emitted confidence against matched.
  - `33486b2`: measured baselines from the first live run (below).
- **mytheus-backend**:
  - `fc12466`, migration `e7a3c9d14b52`: `decisions`, `action_items` (plain tables) and `agent_suggestions` (monthly partitions plus default, registered in `scripts/ensure_partitions.py`), RLS ENABLE + FORCE with `tenant_isolation`, `grant_tenant_table` on each.
  - `822b52a`: the `agent-results` consumer persists `decision_candidate` / `action_candidate` as `status = 'proposed'` under the candidate's id (`ON CONFLICT DO NOTHING`), re-deriving the band from `agent_confidence_overrides`; a foreign or invalid assignee is nulled. No outbox row, ever.
  - `1b6fe1b`, `services/core_api/routes/candidates.py`: `GET /api/v1/meetings/{id}/candidates` (proposed, surfaced band only); `PATCH`, `POST .../confirm` and `POST .../dismiss` for `/api/v1/decisions/{id}` and `/api/v1/actions/{id}`. Rows are read `FOR UPDATE`. Confirm enqueues exactly one outbox row (`decision:{id}:confirmed` / `action:{id}:confirmed`) and an audit row in the same transaction; a second confirm returns the row and enqueues nothing; dismiss enqueues nothing; confirm after dismiss and edit or dismiss after confirm are 409. Participants only (403 `NOT_A_PARTICIPANT`), another tenant 404. OpenAPI re-emitted.
  - `bf87161`: a record the consumer cannot decode (for example a non-UUID string in a `uuid` field) is now `PoisonMessage` and goes to the DLQ. Before, the `ValueError` escaped the loop, the consumer died, and every restart re-read the same offset. Found by the end-to-end run below, on records left in the local broker by earlier test runs.
- **mytheus-realtime**:
  - `16d5cb7`: `agentOutput.ts` forwards surfaced decision and action candidates as a new pub/sub message `agent-candidate`; `server.ts` emits it to every socket of the meeting (ADR-0032 D4), on the org-scoped channel. PersonalListener notes stay per user.
  - `202f919`: the three spawned-process tests set `AGENT_OUTPUT_ENABLED=0`; their gateways had joined the shared agent-output consumer group, and the pod-death test left a dead member holding partitions.
- **mytheus-ui** `94f014a`: `src/lib/realtime/candidates.ts` (a pure reducer merging live and re-hydrated candidates by id; a confirmed or dismissed one never comes back), `useMeetingSocket` subscribes to `agent-candidate` and re-hydrates from `/api/meeting-candidates` on every (re)connect, `CandidateCard` in the agent sidebar with Confirm, Edit (inline) and Dismiss as server actions in `src/app/actions/candidates.ts`. Confirm retries a 404 briefly, because a card can arrive a moment before `core-api` has stored its row. Generated API types refreshed.

### Deviations, and why

1. **Candidates are a list in `payload_json`** (ADR-0032 D2). The canon idempotency key allows one `agent-results` record per agent per window.
2. **The agent assigns the row id** (D3), so the live card and the stored row agree without a round trip.
3. **Candidates go to the whole meeting** (D4), not to the user the runtime ran the meeting-scope agent as.
4. **The silent band is a `proposed` row** (D6), not a worklog: there is no per-user note for a meeting's decision.
5. **`agent_suggestions` uses canon's `question` column**, not the plan DDL's `content`, and lowercase status codes (D7). Created now, unused until P9.
6. **Confirming with `supersedes` does not mark the older decision `superseded`.** That would change a confirmed row without an event; the graph learns the lineage from `supersedes`.
7. **A realtime-gateway change was needed** that the Part A brief did not list: the gateway forwarded only PersonalListener notes, so no card could reach a live call without it.

### Verification (2026-11-09, local stack)

- `mytheus-backend`: `MYTHEUS_REQUIRE_DB=1 uv run pytest tests/test_candidates.py tests/test_worklogs.py -q` all passed (24 candidate tests, including RLS isolation for the three tables, the consumer writing zero outbox rows, exactly one outbox row per confirm, rollback of row and outbox together, and `test_only_the_confirm_routes_enqueue_decisions_or_actions`). Full suite: 685 passed, 1 failed (`test_relay_real_bus_roundtrip`, the known Redpanda memory cap, gap 5). `ruff`, `black --check`, `mypy` clean; `python -m scripts.emit_openapi --check` matches.
- `mytheus-intelligence`: `uv run pytest -q` 287 passed, 39 skipped; `ruff`, `black --check`, `mypy libs services` clean.
- **Live eval**, `EVAL_LIVE=1 uv run pytest -m eval tests/test_decision_action_eval.py`, real models: `DecisionAgent` F1 1.0 (9 of 9, no false positive), Brier 0.012, 21 calls, 13,254 tokens; `ActionAgent` F1 1.0, Brier 0.007, 21 calls, 13,095 tokens. The Haiku gate stopped 7 of 14 windows before Sonnet. Recorded as baselines (`min_f1` 1.0, `max_brier` measured plus 0.02); a second run asserting them passed. The first attempt failed with 401 because the shell exported an older `ANTHROPIC_API_KEY` that overrode the valid key in `.env` (the harness uses `setdefault`).
- `mytheus-realtime`: `npx vitest run` 57 passed (18 files), including real Redis plus Socket.io delivery to every participant but not another meeting or another org with the same meeting id, and a real Avro `decision_candidate` through the real Redpanda consumer; `tsc --noEmit` and `eslint` clean.
- `mytheus-ui`: `npx tsc --noEmit`, `next lint`, `npm test` (25 passed, 5 of them the candidates reducer) green. Not opened in a browser (no Clerk sign-in, gap 8).
- `mytheus-events`: `make validate`, `make codegen`, `make test-py`, `make check-compatibility` (21 subjects BACKWARD) green.
- **Producer grep** across the four code repos: the only producers of topic `decisions` / `actions` are `services/core_api/routes/candidates.py` lines 261 and 401.
- **End to end, scripted model, everything else real** (throwaway scripts, not committed): one tenant, two participants, one meeting. The real `DecisionAgent` and `ActionAgent` graphs (gate on Haiku, extraction on Sonnet, both answered by a script) emitted real Avro to `agent-results`; the real `core-api` consumer stored a 0.86 decision and a 0.5 decision as `proposed`, and a 0.82 action with its assignee and due date; zero outbox rows. `GET .../candidates` returned only the two surfaced ones. Confirm (twice) on the decision and once on the action, dismiss on the silent decision: exactly two outbox rows, one `decisions` and one `actions`, both relayed and read back from Redpanda as valid Avro with the right ids, content, assignee and due date.

### Open items

1. **Deploy order for `supersedes`:** register the new `decisions-value` schema on the production Schema Registry, then restart the relay. The end-to-end run's decision event had no `supersedes` because a long-running relay had cached the older schema.
2. **Browser check** of the cards in a live call with a signed-in user (gap 8).
3. **A meeting-scope agent runs only when the lowest-sorted roster user's context fetch succeeds** (`runtime.py` `_run_user`); if that one fetch fails, the window gets no decision or action pass. A budget block notifies only that user, in wording about notes.
4. **Another participant's confirm or dismiss is not pushed to the room**; the other cards settle on their next action (200 or 409) or on re-hydrate (ADR-0032 consequences).
5. **Mobile** confirm cards (plan section 7) are not started.

## 11. Phase P9 RAG: `rag-service`, participant-scoped hybrid retrieval and the Ask page (2026-11-09)

**Status:** P9 RAG track DONE WITH GAPS, verified 2026-11-09 on the local stack (Postgres 5443, Redis 6390, Redpanda 9092, Qdrant 6333) with real BGE-small-en and BGE-reranker-base and, for the live rows, real Haiku and Sonnet. Not deployed, not opened in a signed-in browser.

P9 is split between two developers working in parallel. This section is the RAG track (plan section 3.9 and the Ask panel). The `QuestionAgent` track (plan section 3.10) is recorded by its owner in section 12. The seam between them is `libs/shared/rag_client.py` in `mytheus-intelligence`. Mobile is out of scope for this phase.

Entry decisions: [ADR-0033](../adr/0033-plan-9-rag-retrieval-upgrades.md) (D1 to D7), written before the build after a review of the locked design found two defects (retrieval isolated by tenant only; graph results could not be cited). The ADR's "Amendments from the build" records every decision the measurements changed. Canon: `_canon/06-tech-decisions.md` section 2.9, the Citations, Retrieval access and RAG API rows.

### 11.1 What landed, per repo

- **mytheus-docs**
  - `31dc0b0`: ADR-0033, the canon rows, plan section 3.9 steps 12 to 18 with their acceptance checks, doc 09 sections 3.5 and 7, the README index, the traceability matrix.
  - This commit: the build amendments in ADR-0033, the RAG API canon row (answer returned after its check, not streamed), plan step 17 and doc 09 to match, this section.
- **mytheus-backend** `c89de7b`
  - `services/core_api/routes/internal_retrieval.py`: `GET /internal/retrieval/scope` (the asker's `users.id` by `user_id` or `clerk_user_id`, the meetings they are a participant of with title and start, the org's people by email) and `POST /internal/retrieval/records` (confirmed or superseded decisions and confirmed or done actions by id, restricted again to the asker's meetings). Both RLS-scoped; an unknown or foreign user is 404.
  - `internal_meetings.py`: the caller check is per route (`authenticate_caller(request, allowed)`). `MYTHEUS_CORE_CALLER_TOKENS` may now hold `rag-service`, which is allowed on the retrieval routes only, not on room-state.
  - `api_gateway/edge.py`: `/api/rag` routes to `rag-service`. `design/openapi/core-api.yaml` re-emitted.
  - `tests/test_internal_retrieval.py` (7 tests).
- **mytheus-intelligence**
  - `b4f43ce`, search and auth:
    - `proto/rag.proto` and its runtime mirror `libs/shared/rag_proto.py`: `FtsSearchRequest` gains `meeting_ids`, `speaker_ids`, `scoped` (an allowlist; empty means nothing, never "unfiltered"); new `GetPassages` RPC returning transcript text with neighbouring chunks (`before`, `after`, same meeting, by `start_ms`) and document element text.
    - `services/search_service/fts.py` and `grpc_server.py`: the scoped query, `get_passages`, and a second auth path for `rag-service`'s service credentials (`MYTHEUS_SEARCH_CALLER_TOKENS`). `GetPassages` accepts the service path only.
    - `libs/shared/auth.py` `verified_or_service_org_id`: `/internal/embed` and `/internal/rerank` accept `rag-service` (`MYTHEUS_EMBED_CALLER_TOKENS`, `MYTHEUS_RERANK_CALLER_TOKENS`) beside a user token. A present but bad service header never falls back to the bearer path.
  - `4a70a84`, `services/rag_service/`:
    - `main.py`: `POST /api/rag/query`. Callers are a user (Clerk token, re-verified) or `agent-runtime-service` (service credentials in `MYTHEUS_RAG_CALLER_TOKENS` plus `X-Mytheus-User`, the participant it asks for).
    - `pipeline.py` `RAGService`: scope, cache (`nexus:{org_id}:rag:{user_id}:{query_hash}`, 10 minutes), expansion in parallel with raw-query retrieval, the vector, FTS and graph arms in parallel, the scope rule re-applied after the arms whatever they returned, RRF k=60 to 50, hydration, rerank, assembly (3,000 tokens, halved past 80% of quota; near-duplicates dropped; at most 4 passages per meeting), four citation kinds, the checked answer. Every addition sits behind a `RagConfig` flag.
    - `clients.py`: core-api scope and records, `/internal/embed`, `/internal/rerank` (rebinding echoed passage text to indexes), Qdrant reads with the read-only key over `transcript_chunks__{org_id}`, `document_chunks__{org_id}` and `decisions__{org_id}` (status confirmed or superseded), the gRPC FTS and passage client, the graph arm (`decisions_by_person`, `related_decisions_for_topic`; empty when the `kg` profile is off).
    - `expand.py` (Haiku, `rag_expand@v1`: variants, date range, person resolved by email local part, source, taxonomy topics), `answer.py` (Sonnet, `rag_answer@v1`, sentence-level marker check, NOT_FOUND abstention), `metering.py` (`usage-events` `agent_token` with producer `rag-service`; a budget probe on the Redis mirror).
    - `libs/shared/rag_client.py`: `RagClient`, `HttpRagClient` (asks as a participant, passages only, never raises), `FakeRagClient`. This is the contract the `QuestionAgent` track builds against.
  - `d22fcb4`, the eval harness and the fixes it drove: `tests/test_rag_eval.py`, `tests/eval/rag_corpus.json` (six meetings, one private to two other people, a document, five confirmed decisions), `rag_golden.json` (20 answerable questions, 3 permission traps, 2 no-answer), `rag_baseline.json`, `rag_ablation.json`. FTS any-term mode (`match_any`), context-aware rerank, heading skip, rerank fusion, the source filter made advisory, larger output caps, the `/internal` mount in the default URLs.
- **mytheus-platform** `9c48e31`: `deploy/compose.prod.yml` adds `rag-service` (8107, profile `intelligence`) and `search-grpc` (50103, the gRPC server that had never been deployed), the gateway's `UPSTREAM_RAG_SERVICE`, and `prod.reranker.env` for the reranker. `scripts/enable-tiers.sh` generates `RAG_SERVICE_TOKEN` and upserts `rag-service` into every callee's allowlist (core-api, billing, graph, search, embed, rerank) with an `add_caller` helper that replaces a stale entry and leaves other callers alone; it admits `agent-runtime-service` to rag-service (`RAG_CALLER_TOKEN`, `RAG_URL` in `prod.agents.env`) and copies the Anthropic and Qdrant read-only keys into `prod.rag.env`. `env/prod.rag.env.example`, `env/prod.reranker.env.example`.
- **mytheus-ui** `1b3cc17`: `/search` is now Ask. `src/app/actions/ask.ts` calls `POST /api/rag/query` with the user's session; `src/lib/rag/answer.ts` holds the contract types and the pure helpers (answer segmentation, citation links, provenance); `AskPanel` renders the checked answer with clickable `[n]` markers and a source list (transcript opens `/meetings/{id}/transcript#t={seconds}`, decision opens its lineage page, action opens My Tasks, document shows its text); `AskView` holds the input, `/` focus and the command palette's `?q=`. The mock search view, its records and `mockSearchIndex` are removed. The nav entry reads "Ask". `tests/rag-answer.test.ts`. Generated core-api types refreshed.

### 11.2 Definition of Done audit (End of P9, RAG lines)

| DoD line | State | Evidence |
|---|---|---|
| `rag-service` shipped: five-stage `RAGService` | MET (not deployed) | `services/rag_service/pipeline.py`; `tests/test_rag_pipeline.py` |
| `org_id` in metadata + mandatory Qdrant payload filter | MET | `clients.py` `QdrantVectorSearch` (collection plus `org_id` plus `meeting_id` filters); gRPC org in metadata |
| RRF k=60 | MET | `fusion.py`; `test_rrf_rewards_agreement_across_lists` |
| Citations with uncited dropped | MET | `_citation` binding; `test_a_candidate_that_cannot_be_hydrated_is_dropped`; answer sentence check |
| Answer cache | MET | `test_cache_hit_is_per_user` |
| Expansion skipped on throttle | MET | `test_short_queries_and_degraded_tenants_skip_expansion` |
| `usage-events` emit | PARTIAL | `metering.UsageEmitter` through `LiteLLMRouter`; not observed on a broker in this session |
| Ask panel | MET (not browser-checked) | `mytheus-ui` `1b3cc17`; `npm test`, `tsc`, `next lint`, `next build` green |
| Five-stage/citation, parallel-arms, cache-hit, degrade-not-fail, cross-tenant isolation tests green | MET | `tests/test_rag_pipeline.py` (22), `tests/test_search_scope_live.py` (3, real Postgres), `tests/test_search_grpc.py` |
| Eval gates: citation validity and retrieval quality | MET | `tests/test_rag_eval.py`, offline and live rows, baseline asserted |
| Mobile Ask screen | NOT MET (out of scope) | Mobile excluded from this phase by the maintainer |
| ADR-0033 additional checks: permission trap; filters in every arm; decision citation; cited answer; ablation non-negative | MET | trap leaks 0 in every row; `test_expansion_filters_reach_every_arm`; `test_a_decision_is_cited_as_a_decision`; `test_answer_keeps_only_cited_sentences`; final row above vector-only on all three metrics |

### 11.3 The ablation (D7)

`tests/eval/rag_ablation.json`, one run, 25 questions. Offline rows: real embeddings, reranker and Postgres FTS. The last row adds real Haiku expansion and Sonnet answers.

| Row | Recall@10 | MRR | nDCG@10 | Trap leaks |
|---|---|---|---|---|
| vector only | 0.729 | 0.775 | 0.733 | 0 |
| + FTS and RRF | 0.729 | 0.775 | 0.733 | 0 |
| + rerank (plain) | 0.779 | 0.696 | 0.693 | 0 |
| + rerank with context | 0.842 | 0.600 | 0.622 | 0 |
| + skip headings | 0.842 | 0.608 | 0.628 | 0 |
| + fuse rerank with first stage | 0.842 | 0.778 | 0.758 | 0 |
| + decisions source | 0.896 | 0.794 | 0.797 | 0 |
| + small-to-big | 0.896 | 0.794 | 0.797 | 0 |
| + expansion, filters, graph (live) | 0.929 | 0.846 | 0.845 | 0 |

Live answer metrics: answered 95% of answerable questions, abstained on 100% of no-answer questions, every answer cited a gold passage. Small-to-big changes the answer context, not the ranking, so its row matching the one above is expected. Passage-path p95 on this CPU-only machine: about 0.9 s offline (the cross-encoder dominates; the `< 500 ms` SLO assumes the GPU reranker), about 5 s live with two real LLM calls.

### 11.4 Deviations from the plan, and why

1. **rag-service has no gRPC surface.** Plan 3.9 step 1 lists `proto/rag.proto` with a RAG service. Users reach it through `api-gateway` over REST, and agents call the same REST route with service credentials; a second transport would be a second auth surface for no caller. `proto/rag.proto` now holds search-service's two RPCs only.
2. **Embed, rerank, graph and core-api are called over REST, not gRPC.** They already serve REST; only search-service had a gRPC surface. Recorded in ADR-0033's amendments.
3. **Scope comes from core-api, not the graph.** `meeting_participants` is relational truth, the `kg` profile is optional in production, and the AI tier may not read core-api's tables, hence the internal scope route.
4. **D5 contextual header deferred**, **D6 score gate off**, **D6 answer not streamed**, **`source` advisory**, **rerank context and fusion**, **FTS any-term**: each with its measurement in ADR-0033's amendments and section 11.3.
5. **Expansion is not role-tuned.** Plan 3.9 step 3 says variants are role-tuned through `get_user_context`. The expansion prompt carries the taxonomy topics and today's date but not the asker's role: one more gRPC hop on the critical path for a gain the eval could not measure (the corpus has one asker). Left for when a multi-role golden set exists.
6. **A person is matched by email local part.** `users` has no display-name column; "Priya" matches `priya.nair@...`. More than three matches is treated as ambiguous and not filtered on.
7. **Documents are excluded when a date, person or single meeting narrows the question.** They carry none of the three, so including them would break the filter's meaning.

### 11.5 Open gaps and handover

1. **Not deployed.** Order: register nothing new on the Schema Registry (no schema changed); deploy `mytheus-intelligence` (image), run `scripts/enable-tiers.sh` with the `intelligence` profile so `prod.rag.env` and the allowlists are written, then deploy `mytheus-backend` (the core-api routes and the gateway route) and `mytheus-ui`. Re-run `enable-tiers.sh` after any `push-secrets.sh agents`, which rewrites the billing allowlist.
2. **Browser check** of the Ask page with a signed-in user: the page sits behind Clerk sign-in and was not opened signed in.
3. **No end-to-end run through the gateway**: the pieces were verified separately (core-api routes on real Postgres, the gRPC client against a real `search-grpc` over real Postgres, the pipeline with real models in the eval). A run with every service up is still to do.
4. **`usage-events` from rag-service** not observed on a broker.
5. **D5** needs the meeting title and speaker name on `transcript-chunks` (a `mytheus-events` change) before `embedding-service` can build the header.
6. **Multilingual risk (ADR-0033 consequences)**: not measured; the golden set has no non-English question yet.
7. **For the `QuestionAgent` track:** call `HttpRagClient.retrieve(org_id=..., user_id=<the participant>, query=...)`; `RAG_URL` and `RAG_CALLER_TOKEN` are written to `prod.agents.env` by `enable-tiers.sh`. Citations have four kinds.

### 11.6 Verification (2026-11-09, local stack)

- `mytheus-backend`: `MYTHEUS_REQUIRE_DB=1 uv run pytest tests/test_internal_retrieval.py tests/test_meeting_room_state.py -q` 12 passed; `tests/test_gateway_edge.py tests/test_openapi_contract.py` 42 passed; `python -m scripts.emit_openapi --check` matches; `ruff`, `black --check`, `mypy` on the changed modules clean.
- `mytheus-intelligence`:
  - `uv run pytest tests/test_rag_pipeline.py tests/test_rag_api.py tests/test_search_grpc.py tests/test_service_caller_auth.py -q` 59 passed; `tests/test_query_embed.py tests/test_reranker.py tests/test_auth.py` 26 passed.
  - `MYTHEUS_REQUIRE_DB=1 uv run pytest tests/test_search_scope_live.py tests/test_search_routes.py -q` 14 passed.
  - `MYTHEUS_REQUIRE_DB=1 uv run pytest -m eval tests/test_rag_eval.py` passed (offline gate); with `EVAL_LIVE=1` and the `.env` key (shell `ANTHROPIC_API_KEY` unset so `.env` wins), passed with the live gate, producing the table in 11.3.
  - A wire smoke with throwaway scripts: rag-service's `GrpcLexicalSearch` against a real in-process `search-grpc` over real Postgres. Any-term FTS found the PLAT-4412 line for "what is the SSO ticket about", an allowlist of another meeting returned nothing, `GetPassages` returned the neighbouring lines, a wrong service token was refused (UNAUTHENTICATED).
  - `ruff check`, `black --check --target-version py312`, `mypy libs services` (106 files) clean.
- `mytheus-platform`: `docker compose -f deploy/compose.prod.yml config` valid; with `COMPOSE_PROFILES=intelligence` the services list includes `rag-service` and `search-grpc`; `bash -n scripts/enable-tiers.sh` OK; `add_caller` exercised in isolation (add, no-op, replace a stale entry, new key; other callers untouched).
- `mytheus-ui`: `npx tsc --noEmit`, `npx next lint`, `npm test` (32 passed, 4 of them the answer helpers), `npm run generate:api:check`, `npx next build` green. `/search` redirected to sign-in in the dev server, as expected unauthenticated.
