# 04 Plan 5-6 Org Context and AI Infrastructure

| Field | Value |
|---|---|
| Band | Plan 5-6, phases P5 (Profiles and Org Context, W13-14) and P6 (AI Infrastructure, W15-17) |
| Plan file | [plan/plan-5-6-org-context-and-ai-infra.md](../plan/plan-5-6-org-context-and-ai-infra.md) |
| Status | **IN PROGRESS.** Nine of the band's ten tasks are DONE (task 10, `search-service`, was still running when this record was written) and the whole-branch final review has not yet run. Every task that has completed is tested against real, live infrastructure: Postgres, Redis, Redpanda plus Schema Registry, Qdrant, and MinIO all ran locally for real, and BGE-small-en and BGE-reranker-base ran genuine CPU inference. Only the GPU tier itself, and the latency SLOs that assume it, are BLOCKED. |
| Complete | `identity-tenant-service` org model reconciliation, the role-to-topics taxonomy, `get_user_context()` with a real Redis cache, the seeder, `bff` aggregation endpoints (`mytheus-backend`); the event backbone verified GA plus a client-packaging fix (`mytheus-events`); `embedding-service`, `reranker-service`, `ingestion-service` (`mytheus-intelligence`) |
| Deliberately not built | `mytheus-ui` section 3.5 (org-chart UI, person-profile UI) - out of scope by the same kind of product-owner decision as Plan 3-4. `Topic` node seeding into `nexus_knowledge` and seed `transcript_chunks` rows - both correctly refused as second-writer traps with no legitimate write path today (section 3). |
| Blocked, not built | GPU packaging and the SLOs that assume it: `embedding-service`'s batch-32 sub-100 ms warm latency (measured ~3536 ms on CPU) and `reranker-service`'s warm-floor/cold-start behavior (per-region autoscaler topology, not expressible in a single Ray Serve deployment). Qdrant write-access control for the "only writer" acceptance check (enforced by code convention only; the dev Qdrant has no API key at all). |
| Last verified | 2026-08-06, by running each repo's own suite in its Plan 5-6 worktree against the shared dev stack (Postgres, Redis, Redpanda, Schema Registry, Qdrant, MinIO) |

The user resolved, on 2026-08-05, that this band's infrastructure would be **real and local, not faked**: Qdrant 1.19.0 on `:6333`/`:6334`, MinIO on `:9002`/`:9003`, and BGE-small-en plus BGE-reranker-base running genuinely (slowly) on CPU via `sentence-transformers`. This is the headline difference from Plan 3-4, where LiveKit and Deepgram could not be exercised at all because no credentials existed for either. Here, almost nothing is faked: only GPU packaging and the SLOs that assume GPU hardware are BLOCKED, and every task marks that distinction explicitly rather than claiming a CPU number against a GPU target.

Both of this band's `DECISION-REQUIRED` callouts (plan section 8) were closed before any task wrote code. 8.1 (the `transcript-chunks` composite partition key) was ratified in Plan 3-4 as [ADR-0017](../adr/0017-transcript-chunks-composite-partition-key.md); this band only had to wire the already-ratified key. 8.2 (Qdrant tenancy) was resolved by the product owner as **option (a): per-tenant collections**, exactly as [ADR-0006](../adr/0006-vector-store-tenancy.md) already locks them, plus the mandatory `org_id` payload filter as a second, independent isolation layer. That resolution needed no canon edit, because the ratified answer is exactly what ADR-0006 already says - the callout existed to make sure nobody quietly built the shared-collection alternative instead. The plan file's own `8.2` blockquote has not yet been edited in place to say RESOLVED the way Plan 3-4's propagation pass did for `8.1`; see the open gaps.

---

## 1. What landed, per repo

### 1.1 `mytheus-docs`: one reconciliation, no new decisions

Neither `DECISION-REQUIRED` callout needed a canon change (see above), so this band's only `mytheus-docs` commit is a naming reconciliation, not a ratification.

- **`e2a7a9a`** - `24-glossary-and-naming.md` section 10 gains one row: the data-model's informal `identity-service` label, as applied to `teams`/`team_members`/`departments`, maps to `core-api`, which is the service that actually owns their CRUD per `_canon/01-service-catalog.md` and the shipped `org_hierarchy.py`. `identity-tenant-service` keeps the identity and tenancy surface (`organizations`, `users`, RBAC, SSO/SCIM) and does not write those three tables. This follows the existing `agent-service` and `meeting-service` rows' form exactly, and changes no canon file and no ADR: `_canon/02-data-model.md`'s informal labels are deliberately informal and were left untouched, per doc 24's own stated purpose of reconciling them.
- This closed a real risk found by task 1 (section 3.1, below): without the row, a future reader could read `_canon/02-data-model.md`'s label at face value and add `teams`/`team_members`/`departments` routes to `identity-tenant-service`, creating a second writer of tables `core-api` already owns.

### 1.2 `mytheus-backend`: five tasks, one reconciliation and four real builds

**Task 1, section 3.1 (org model)** turned out to be a reconciliation, not a greenfield build: `users.role`/`skills`/`bio`/`avatar_url` and `teams`/`team_members`/`departments` with `FORCE` RLS and `tenant_isolation` all already existed from Plan 1-2 (migrations `82d44163c933` and `b7d9f3a55c21`). The genuine additions were migration `56f72ac73fa9` (the two lookup indexes the plan's outline names, `ix_departments_parent_id` and `ix_team_members_user_id` - kept in the repo's own `ix_<table>_<col>` naming convention rather than the plan's literal text) and `PATCH /users/{user_id}` in `identity-tenant-service`, the one route the plan asked for that was genuinely missing. Commit `c7b533b`, 205 passed / 0 skipped (10 deselected per the do-not-run instruction on `test_roles_and_policies.py`, disclosed below).

**Task 2, section 3.2 (role-to-topics taxonomy)**, commit `b5898b7`: `taxonomy/role_topics.v1.yaml` (verbatim from the plan), `taxonomy/loader.py` (Pydantic v2, `active_version()`, `load_taxonomy()`, `topics_for()`), `GET /taxonomy/role-topics?version=v1` (unknown version 404s, no silent fallback), and the real `topics_for` swapped in for task 3's placeholder dict (see below). 228 passed / 0 skipped (10 deselected). Deliberately did **not** fold the taxonomy version into the 300 s `user_ctx` Redis key: a version bump is a deploy-time event, not a request-scoped one, the staleness window is bounded and self-healing, and changing the key shape would have broken task 3's already-green tests.

**Task 3, section 3.3 (`get_user_context()` with Redis cache)**, commit `efe8851`: `libs/shared/cache.py` (`get_redis`, `redis_key`), `context/service.py` (`get_user_context`, `_compute`, `invalidate_user_context`, 300 s TTL), `GET /context/{user_id}`, cache invalidation wired into `PATCH /users/{id}`, and `grpc_server.GetUserContext` now calls the same cached resolver, populating the previously-empty `team_ids`/`department_id`/`cache_expires_at` proto fields while keeping `email`/`rbac_tier` as separate uncached queries per ADR-0010 D2. 218 passed / 0 skipped (10 deselected). This closes the Plan 1-2 audit finding that the cache was entirely unbuilt (zero grep hits at the time). The load-bearing property, that the cache is an optimization and never a dependency, is proven against a genuinely unreachable Redis at `127.0.0.1:1`, with the test first confirming that address actually raises `ConnectionError` before trusting the rest of the assertion. Because the plan dispatched section 3.3 before section 3.2 (contrary to the plan's own step order), task 3 shipped a small, clearly labelled inline fallback for `topics_for` covering the plan's five example roles, which task 2 then replaced with the real taxonomy.

**Task 4, section 3.4 (seeder)**, commit `2cd137f`: `make seed` (2 orgs x 10 users x 5 meetings) with deterministic `uuid5` ids so reruns are no-ops and cannot collide with another agent's data in the shared dev database, plus `provisioning.py::provision_organization()`, the first real org-creation path (an org row and a `tenant-events` outbox row in one transaction). This closes a second Plan 1-2 gap: tenant provisioning previously had no reachable path at all (`POST /api/v1/orgs` deferred, no operator script). 231 passed / 0 skipped (10 deselected). The run was interrupted mid-response by an API error and resumed cleanly (section 3). For `core-api`-owned tables (`departments`, `teams`, `meetings`, `meeting_participants`) the seeder calls that service's real functions in-process rather than raw-inserting, producing 18 genuinely published `meeting-events` rows through the ambient relay.

**Task 5, section 3.5 (`bff` aggregation, UI itself out of scope)**, commit `9bd7de7`: `GET /bff/org/chart` (department to team to member tree in exactly 3 SQL statements regardless of tree shape, proven both by a test that grows the tree mid-test and by a live measurement against real seeded data: 7.85 ms for acme's 3 departments / 3 teams / 10 members) and `GET /bff/people/{user_id}` (a thin wrapper around `get_user_context()` plus `email`, the one field the cache deliberately excludes). Cross-tenant safety proven adversarially with two orgs given identical department, team, and email labels, isolation asserted by id rather than name. `mytheus-ui` was not touched and not even read, per the scope ruling. 244 passed / 0 skipped (10 deselected) - this is the backend's final suite state for the band as of writing.

### 1.3 `mytheus-events`: verified complete, plus a genuine client-packaging fix

**Task 6, section 3.6 (event backbone GA): DONE, no commit needed.** Every acceptance check in section 3.6 already passed against the live broker and registry on the commits Plan 3-4 left in place; HEAD stayed at `1549e25` with a clean working tree before and after `make codegen`. All 21 topics plus 21 `.dlq` pairs are live with the correct partition counts (`rpk topic list`); all 21 subjects are registered BACKWARD (`curl :8081/subjects` plus `/config/{subject}`); `DocumentChunk` and `EmbeddingEvent` already existed, registered and fixture-covered by the previous band, contradicting a controller guess that they were the likely genuine addition; and the outbox/inbox/relay/DLQ mechanism was verified by reading the actual `mytheus-backend` implementation (`FOR UPDATE SKIP LOCKED`, the exact Redis dedupe key, the exact inbox primary key, `DlqEnvelope` with `raw_value_b64`) rather than assumed. All 21 subjects show as versions `[1, 2]` because the doc-string-only fix to the shared `EventEnvelopeHeader` cascades into every subject (`register.py` inlines the envelope into each subject's schema) - harmless, since doc-only diffs are BACKWARD by construction, but worth recording so the version bump is not mistaken for a field change.

**Client-packaging fix, commits `38dab78` (Python) and `3758cdb` (TypeScript).** Task 7 found the generated `mytheus_events` Python client was not `pip`-installable (no `[build-system]` table) and worked around it in `mytheus-intelligence` by writing its own `AvroCodec` directly against the live Schema Registry rather than fixing another repo from its own worktree. The fix was dispatched back to the events task and proven, not asserted: the failure was first reproduced in a throwaway virtualenv, then fixed with a Hatchling `[build-system]` (matching `mytheus-intelligence`'s existing backend) and a wheel-packages target at `clients/python/mytheus_events`, then success reproduced in a fresh venv. `fastavro` and `httpx` moved from runtime to dev dependencies after grepping to confirm the shipped client never imports them. TypeScript had the same class of problem (no `package.json` in `clients/typescript`, the root `package.json` marked `"private": true` with no `main`/`types`/`exports`) and got the same proof discipline: fixed with real `types`/`main`/`exports` at `dist/` plus a `prepare` script, then proven by a real consumer project importing the generated types. `mytheus-intelligence`'s services continue to use the hand-rolled `AvroCodec` established as this band's convention rather than switching to the now-fixed generated client, since the convention was already set before the fix landed.

### 1.4 `mytheus-intelligence`: scaffolded from documentation-only to three real services

`mytheus-intelligence` entered this band as documentation only. Tasks 7 through 9 scaffolded it for real, and independently converged on the same structural decisions (recorded as this band's repo conventions, first established by task 7 and followed by tasks 8 and 9): a dedicated Postgres schema `intelligence` for every table this repo owns (never `public`, avoiding collision with `mytheus-backend`'s own `public.outbox`/`public.inbox` in the same physical database - a worse collision than the `alembic_version` risk the brief had flagged); roles `mytheus_ai_app`/`mytheus_ai_relay`, distinct from the backend's three; no foreign key to `public.organizations`; its own `alembic_version_intelligence` table, following the `transcription-service` precedent; `libs/shared/consumer.py`'s `BaseConsumer` as the one sanctioned at-least-once base class; and one relay process per repo, mirroring the backend's single-relay pattern.

**Task 7, section 3.7 (`embedding-service`), commit `95be0db`.** 28/28 passing, all against real infrastructure: real Postgres, real Qdrant, real Redpanda plus Schema Registry, real BGE-small-en on CPU, real Redis, no fakes on any load-bearing path. Proven against real infra: 384-dim vectors with the correct payload land in `transcript_chunks__{org_id}`; redelivery idempotency via a stable point id plus dedupe (both the Redis-warm and the durable-`inbox`-only paths); cross-tenant Qdrant isolation by both the collection boundary and the payload filter (a query with the wrong `org_id` filter against the *right* collection still returns zero); and the `embedding-events` ack proven end to end through a real Redpanda relay round trip, not merely as a database row. Batch latency for a batch of 32 measured **3536 ms warm on CPU** against the plan's `< 100 ms` GPU target - the batching mechanism itself (`@serve.batch(max_batch_size=32, batch_wait_timeout_s=0.05)`) is real and exercised, the number is explicitly not claimed. "Only writer of Qdrant" is enforced by code convention only (grep confirms no other service imports `libs/shared/qdrant.py`'s `upsert()`), not by infrastructure access control: the dev Qdrant instance has no API key configured at all, flagged as a real gap for `mytheus-platform`.

**Task 8, section 3.8 (`reranker-service`), commit `2eab69c`.** 37/37 (28 baseline plus 9 new) against the real BGE-reranker-base cross-encoder on CPU, following task 7's conventions rather than inventing a second dialect. `POST /rerank` builds pairs from exactly one call's own arguments with no `@serve.batch` fan-in across requests (confirmed against the plan's own reference pseudocode, which specifies fan-in only for `embedding-service`'s ingest lane), proven cross-tenant-safe two ways: a concurrent two-org test (`asyncio.gather`, 20 disjoint passages each, zero cross-contamination) and a model-layer test that monkeypatches the model to record exactly which pairs each of two back-to-back calls was scored with. CPU measurements were 283-440 ms for `model.predict` and 281-376 ms end to end through FastAPI, comfortably under the plan's 500 ms warm p95 target - and the task still refused to claim the SLO, since it did not measure on the GPU hardware the SLO targets. Owns no Postgres table and no Qdrant credentials, proven by grep returning zero matches, not asserted; by design, the plan's own service map gives it no owned data and no events.

**Task 9, section 3.9 (`ingestion-service`), commits `4a7bd33` + `4c9b120`.** 112/112 (37 baseline plus 75 new) against real Postgres, Redis, Redpanda plus Schema Registry, Qdrant, and real MinIO. `POST /ingest` of a sample PDF writes the real blob to `nexus-documents/{org_id}/{document_id}/`, real rows into `document_chunks` with `element_type` in `{Title, NarrativeText, Table, ListItem}`, and one real `document-chunks` event per chunk. Idempotency proven at both the pipeline level (a rerun with the same `document_id` adds zero rows) and the SQL level (`UNIQUE(org_id, document_id, element_id)`). Object-key safety was proven adversarially on two separate trust boundaries: the upload path, where `org_id` is always the verified caller and `document_id` is always server-minted, so path traversal, null-byte, and absolute-path payloads never reach the key; and a second boundary the brief did not name, the calendar-attachment consumer, which reads an object *another service* names in its event payload - closed with a same-org-prefix check before any `GET` reaches MinIO. PII redaction is a real, working regex layer (email, SSN, credit card, phone), gated by a real Redis read at `nexus:{org_id}:tenant_settings:pii_redaction_enabled`; the honest gap is that nothing in the program writes that key yet, so `False` is not a fabricated default but the actual current state of every tenant. The embedding half of end-to-end ingestion (`document-chunks` reaching `document_chunks__{org_id}`) is proven as two independently-real halves - this task's outbox rows deserialize correctly per the frozen `DocumentChunk` schema, and task 7's `EmbeddingConsumer` independently proves consuming that topic - but the physical hop of one task's relay-published event being polled live by the other task's running consumer process was not exercised in a single test.

**Task 10, section 3.10 (`search-service`): pending.** See "Task 10: search-service" below.

### 1.5 `mytheus-ui`: untouched, by decision

Section 3.5's org-chart and person-profile UI (`app/(dashboard)/org/page.tsx`, `app/(dashboard)/people/[id]/page.tsx`) were deliberately not built, per the same kind of out-of-scope decision Plan 3-4 recorded for the meeting-room and captions UI. The `bff` endpoints those screens need (`GET /bff/org/chart`, `GET /bff/people/{user_id}`) were built in full, and their exact JSON shapes, status codes, and degrade semantics are documented in task 5's report for whoever builds the UI next.

### 1.6 `mytheus-mobile`: untouched, not a scoped decision

Section 7 (removing the mock for people data, adding Clerk Expo auth, and the person-profile screen reading `get_user_context`-backed data from `bff`) was not built. Unlike `mytheus-ui`, this is not a recorded product-owner exclusion for this band: the resolved-decisions note at the top of the controller's ledger names `mytheus-ui` as out of scope and says nothing about `mytheus-mobile`, and none of the band's ten dispatched tasks covered it. Recorded as an open gap, not a decision.

---

## 2. Definition of Done audit

Against section 10 of [plan-5-6-org-context-and-ai-infra.md](../plan/plan-5-6-org-context-and-ai-infra.md), reproduced verbatim per line. Lines that depend on task 10 or the final review, still pending as of writing, are marked PENDING rather than guessed.

### P5 Definition of Done (W14)

| DoD line | Verdict | Evidence or reason |
|---|---|---|
| Migrations for `teams`/`team_members`/`departments` + `users` role/skills applied with RLS `FORCE`; `make seed` is idempotent (2 orgs x 10 users x 5 meetings) | **MET** | All tables already existed from Plan 1-2 with `FORCE` RLS and `tenant_isolation` (task 1, section 1.2); `make seed` reruns are no-ops by deterministic `uuid5` ids (task 4). |
| `GET /context/{user_id}` (REST + gRPC) returns cached role/skills/team/recent-meetings/topics; key TTL <= 300 s; eviction on role/team change verified | **PARTIAL** | REST and gRPC both return the cached payload with a 300 s TTL, proven against a genuinely unreachable Redis and cross-checked for identical shape (task 3). Eviction on **role** change is proven (`PATCH /users/{id}` calls `invalidate_user_context`). Eviction on **team** change is not wired: no route in the codebase writes `team_members` yet other than the seeder, so there is nothing to invalidate against (task 3's disclosed open item, confirmed still true after task 4). |
| `GET /taxonomy/role-topics?version=v1` serves the version-pinned taxonomy; `Topic` nodes seeded per org | **PARTIAL** | The taxonomy endpoint is real, version-pinned, and 404s on an unknown version with no silent fallback (task 2). `Topic` node seeding into `nexus_knowledge` is **BLOCKED**: `knowledge-graph-service` is a bare four-line scaffold declaring itself the graph's only writer, with zero routes, so no legitimate write path exists anywhere in the program today (tasks 2 and 4 both hit this and both correctly refused to build a second writer). |
| web-app `/org` + `/people/{id}` and the mobile person profile render live `get_user_context` data; cross-org isolation proven | **NOT MET** | Deliberately out of scope (`mytheus-ui`, section 1.5) and not scoped to any task (`mytheus-mobile`, section 1.6). The `bff` contract both would consume is built and cross-tenant-proven server-side (task 5). |
| P5 demo: open the seeded org chart, click a person on web and mobile, show identical `get_user_context`-backed profile, change a role and watch the cache evict and topics update | **NOT MET** | The server-side half (identical `get_user_context` payload over REST and gRPC, cache eviction on role change, updated topics) is proven (tasks 2, 3). No browser or mobile run exists to complete the literal demo, since neither client was built this band. |

### P6 Definition of Done (W17)

| DoD line | Verdict | Evidence or reason |
|---|---|---|
| All 21 topics + 21 DLQs created; 9 Avro subjects registered BACKWARD; CI compatibility gate live; outbox relay + inbox/dedupe + DLQ routing working | **MET** | Task 6, verified against the live broker and registry, no rebuild needed. |
| `embedding-service` embeds `transcript-chunks` + `document-chunks` at batch 32 sub-100 ms into per-tenant Qdrant collections (only writer); `embedding-events` acks flow; lag < 5 s | **PARTIAL** | Embedding, upsert, redelivery-idempotency, cross-tenant isolation, and the ack are all proven against real infra (task 7). The sub-100 ms figure is a GPU target, measured at ~3536 ms on CPU and not claimed; consumer lag under load was not measured (single-message tests only); "only writer" is a code convention, not an infra-enforced control (no Qdrant API key exists in dev). |
| `reranker-service` `/rerank` top-50 to top-10 warm p95 < 500 ms with a warm floor | **PARTIAL** | `/rerank` is real and cross-tenant-safe (task 8). CPU measurements (281-376 ms end to end) are comfortably under the 500 ms figure but the task correctly declined to claim a GPU SLO from a CPU measurement. The warm floor and cold-start behavior are region/schedule topology (`num_replicas` cannot coexist with `autoscaling_config` in Ray Serve), not expressible in a single deployment config at this task's scope, so verified only that the real deployment boots and scores correctly. |
| `ingestion-service` `POST /ingest` to `nexus-documents` to `document_chunks` to `document-chunks` to embedded, end-to-end; PII redaction gated by `tenant_settings` | **PARTIAL** | The upload path, storage, chunking, and event emission are proven end to end against real MinIO and Postgres (task 9). PII redaction's mechanism is real and tested on and off; the gating `tenant_settings.pii_redaction_enabled` value has no real publisher anywhere in the program yet, so the Redis-backed default of `False` is honest but unverified against a real tenant setting. The final embedding hop is proven as two independently-real halves, not one live run connecting this task's producer to task 7's running consumer. |
| `search-service` `GET /search` returns org-scoped FTS hits under RLS | **PENDING** | Task 10 was still running when this record was written. See "Task 10: search-service" below. |
| Both section 8 `DECISION-REQUIRED` items have a recorded ratification (or an explicit deferral with the chosen default) | **MET** | 8.1 via [ADR-0017](../adr/0017-transcript-chunks-composite-partition-key.md), ratified in Plan 3-4. 8.2 resolved by the product owner as option (a), matching ADR-0006 exactly, recorded in this band's session ledger; no canon edit was needed because the ratified answer was already canon. The plan file's own `8.2` blockquote text has not yet been edited in place to say RESOLVED (see open gaps). |
| P6 demo: upload a PDF, watch it traverse ingest to bucket to `document_chunks` to `document-chunks` to embedding consumer to `document_chunks__{org_id}`; run an FTS `/search` and a Qdrant vector search side by side returning org-scoped hits; force a poison message and show it land in the DLQ while the partition keeps moving; show the embedding-lag and DLQ-rate Grafana panels green | **PARTIAL, PENDING task 10 for the FTS half** | The ingest-to-embed traversal is proven as independently-real halves (task 9's producer, task 7's consumer), not one connected live run. A poison-message DLQ test exists for `ingestion-service`'s new `IntegrationEventConsumer` (`test_missing_org_id_on_integration_events_goes_to_dlq_never_guessed`), not re-demonstrated for `embedding-service` specifically. The FTS side of the side-by-side comparison depends on task 10. Grafana panels for embedding-lag and DLQ-rate were never in this band's ten-task breakdown and were not built by any task; that is a gap, not a pending item. |

---

## 3. Deviations from the plan, and why

| Deviation | Rationale |
|---|---|
| **`mytheus-ui` section 3.5 not built.** | Explicit product-owner decision recorded before this band's tasks were dispatched, matching the pattern set in Plan 3-4. The `bff` contract it would consume is built and documented (task 5). |
| **`mytheus-mobile` section 7 not built.** | Not a recorded decision like `mytheus-ui`'s: simply absent from the band's ten-task breakdown. Recorded as an open gap, distinct from a deliberate exclusion. |
| **`Topic` nodes not seeded into `nexus_knowledge`.** | `knowledge-graph-service` declares itself the graph's only writer and is a bare four-line scaffold with zero routes (re-confirmed live, 2026-08-06, not assumed from an earlier read). Both `mytheus-backend/CLAUDE.md` and `mytheus-intelligence/CLAUDE.md` state this repo reads the graph but never writes it. Two separate tasks (2 and 4) hit the same wall independently and both correctly refused to build a second writer rather than take the plan's literal text as authorization. |
| **The seeder does not seed `transcript_chunks`, contrary to the plan's literal text.** | Self-discovered by task 4, not flagged in its brief: `transcription-service` exposes no REST, gRPC, or event write path for this table, only its own in-process pipeline and its own tests' raw superuser insert. The same class of second-writer trap as the `Topic`-node case, caught by the same discipline (checking `mytheus-backend/CLAUDE.md`'s Pitfalls section, which names this exact mistake by shape: "cross-writing another service's table because it is 'right there' in the shared Postgres"). Consequence disclosed plainly: the AI tier now has meetings to reason about but no transcript content unless `transcription-service` is driven directly or grows its own seed script keyed to this seeder's deterministic meeting ids. |
| **The seeder writes `team_members` directly, though the plan's ownership convention says `core-api` should own that write.** | A controller instruction task 4 challenged rather than followed blindly. Ruling: not a one-writer-per-store violation, because `team_members` currently has **no** writer anywhere in the product; the seeder is the first and only one, so the invariant holds trivially. The real finding is a product gap (no team-membership write API exists at all), recorded as an open gap rather than solved here, since building one would be an unrequested feature outside section 3.4's scope. |
| **`mytheus-intelligence` owns its tables in a dedicated Postgres schema `intelligence`, not `public`, with its own `outbox`/`inbox`, roles, and Alembic history.** | The brief flagged a lesser `alembic_version` collision risk; task 7 found a bigger one first: `mytheus-backend` already owns `public.outbox` and `public.inbox` in the same physical database, so two repos would have collided on table names, not just migration bookkeeping. Structural fix, not a workaround, and adopted as this band's cross-repo convention for tasks 8 to 10. |
| **`mytheus-intelligence`'s services talk to the Schema Registry through a hand-rolled `AvroCodec`, not the generated `mytheus_events` client.** | The generated Python client was not `pip`-installable at the time (no `[build-system]` table). Task 7 worked around it rather than fixing another repo from its own worktree, and the fix was dispatched and landed separately (`38dab78`, `3758cdb`). The convention was already set before the fix landed and tasks 8 to 10 followed it rather than switching mid-band. |
| **`embedding-service` and `reranker-service`'s GPU-targeted SLOs are reported as real CPU numbers, explicitly marked BLOCKED against the GPU target, never claimed green.** | Per the resolved decision that this band's infrastructure is real but GPU packaging is the one exception. `embedding-service` measured ~3536 ms per batch of 32 against a < 100 ms GPU target; `reranker-service` measured 281-376 ms end to end against a < 500 ms target it is comfortably under on CPU but still refused to claim, since it was not measured on the hardware the SLO targets. |
| **`ingestion-service`'s PII-redaction gate reads a Redis key this task itself never populates.** | `tenant_settings.pii_redaction_enabled` is owned by `identity-tenant-service` in a different repo's Postgres; `mytheus-intelligence`'s own rule is never to join into another repo's tables, and no gRPC mesh exists yet to call across. Resolved with a real Redis read at `nexus:{org_id}:tenant_settings:pii_redaction_enabled`, defaulting to `False` - not a fabricated fallback, since no producer in the program writes this key today, so `False` is the actual current state for every tenant. |
| **`ingestion-service` briefly uninstalled `torch` from the shared virtualenv while stripping `unstructured[pdf]`'s heavy transitive stack (torch, torchvision, onnxruntime, opencv, pandas, matplotlib, timm, accelerate, roughly 2 GB), before catching and reverting it.** | Disclosed and fixed within the same task, not left for someone else to find. `sentence-transformers` (tasks 7 and 8) genuinely needs `torch`. The regression was introduced before an API-error interruption (see below) and caught only because the resume instruction explicitly required re-verifying shared/cross-cutting state before continuing, rather than trusting recollection. Both flagged baselines (37/37 for `embedding-service`/`reranker-service`) were re-run green before any further work. |
| **Two task runs (4 and 9) were terminated mid-response by an API error, not by any fault of the task, and both were resumed rather than restarted.** | Both left uncommitted partial work in a shared worktree, including edits to shared modules (`libs/shared/db.py` for task 4; `services/embedding_service/splitter.py` for task 9). Both resumes were explicitly told to re-read their own partial work rather than trust recollection and to verify shared files first, since a task that dies mid-refactor can damage code it does not own. The `torch` regression above is the concrete payoff of that instruction. |
| **`design/openapi/bff.yaml` was left stale against the shipped `dashboard.py` shape, and no new route from this band was added to it.** | Pre-existing drift, not introduced by this band: the spec described `Dashboard` as flat arrays plus a separate `arms` object, while the shipped code returns each field as an inline `{status, items}` arm. No task in this band (1 through 5) touched `design/openapi/` for any of its own new routes either, so task 5 followed the established precedent rather than fixing it unilaterally for `bff` alone. Connects to the pre-existing, still-open finding that `spectral`/`oasdiff` are not actually wired into CI. |
| **The Plan 5-6 worktrees for `mytheus-backend`, `mytheus-events`, `mytheus-docs`, and `mytheus-intelligence` are nested one level deeper than intended** (for example, `mytheus-docs/mytheus-docs/.worktrees/plan-5-6` rather than `mytheus-docs/.worktrees/plan-5-6`). | A controller error: the worktrees were created with `git -C "$r" worktree add "$r/.worktrees/plan-5-6"`, and `git -C` changing into the repo first made the relative path resolve twice. All four are correctly registered in `git worktree list` and functional; not normalized mid-band because `git worktree move` under an active writer risks losing in-flight changes. |

---

## 4. Open gaps

Ordered by what blocks the most.

| # | Gap | Blocks | Picked up by |
|---|---|---|---|
| 1 | **Task 10 (`search-service`) had not finished, and the whole-band final review had not run, as of writing.** | The P6 `search-service` DoD line, the P6 demo's FTS half, and any release decision for this band. | The controller; see "Task 10: search-service" and "Final review" below. |
| 2 | **`Topic` node seeding into `nexus_knowledge` has no legitimate write path anywhere in the program.** `knowledge-graph-service` is a bare scaffold declaring itself the graph's sole writer, with zero routes. | Plan section 3.2's third acceptance check and the AI tier's later graph-traversal work. | Whichever future band first gives `knowledge-graph-service` a real write surface; `taxonomy.loader.load_taxonomy().roles` is the ready import when it does. |
| 3 | ~~**`transcript_chunks` has no write path a sibling repo can legitimately use.**~~ **CLOSED.** `transcription-service` grew `src/transcription_service/devseed.py`, a dev-only CLI (`python -m transcription_service.devseed --org-id <uuid> --finals N`). It adds no REST, gRPC or event surface to the running service, so the one-writer rule is intact and nothing new is exposed in production. Crucially it is **not** a direct INSERT: every utterance is fed through `persistence.process_utterance`, the same single call site the real provider streams drive, so seeded data is produced by the real rules rather than resembling them - interims are dropped, finals are redacted per the tenant's `pii_redaction_enabled`, and each final writes one `transcript_chunks` row and one `outbox` row in one transaction. That outbox row is what makes this useful to the AI tier at all: `mytheus-backend`'s relay publishes it to `transcript-chunks` as real Avro, so `embedding-service`, `search-service` and `agent-runtime-service` consume a real message off a real topic. A direct INSERT would have been shorter and produced rows that never reached the topic those services actually subscribe to. Idempotent by construction: `chunk_id` is `uuid5(NAMESPACE_URL, "devseed:{meeting_id}:{i}")`, so re-running hits the outbox's `(topic, idempotency_key)` guard and writes nothing rather than duplicating. Guarded by an explicit `TRANSCRIPTION_DEV_SEED=1`, and it refuses an `--org-id` absent from `organizations` rather than creating one, because that table belongs to `identity-tenant-service` and one writer per store applies to dev tooling too. | - | Done. |
| 3a | **Correction to gap 3's own instruction.** It said the seed script should be "keyed to this seeder's deterministic meeting ids". There are none: `services/identity_tenant_service/seed/seed.py` mints deterministic ids via `uuid5(_SEED_NAMESPACE, ...)` for orgs and users, but **meetings** are created through the real `create_meeting` path and re-located by title (`_SELECT_MEETING_BY_TITLE`), so their ids are whatever that path assigned and cannot be hardcoded by another repo. `devseed` therefore takes `--meeting-id` as an argument, defaulting to a fresh UUID. Harmless for a consumer that only reads `transcript_chunks` or the topic, since the table is range-partitioned and declarative partitioning precludes foreign keys, so any UUID is accepted; it matters only for a consumer that joins back to `meetings`, and the module docstring carries the SQL to look a seeded meeting up. | Nothing. Recorded so the next reader does not go hunting for deterministic meeting ids that do not exist. | - |
| 4 | **No team-membership write API exists anywhere in the product.** The seeder is currently the only writer of `team_members`, which is why writing it directly was ruled acceptable rather than a one-writer-per-store violation; the underlying product gap remains. | Any real product flow that needs to add or remove a team member outside the seeder. | A `core-api` task, since that service owns `teams`/`team_members`/`departments` per the section 1.1 reconciliation. |
| 5 | **GPU packaging is not built for `embedding-service` or `reranker-service`; every latency SLO that assumes GPU hardware is unmeasured on the target hardware.** | Claiming the plan's sub-100 ms embedding-batch and sub-500 ms warm-rerank SLOs for real; the region/schedule warm-floor topology for `reranker-service`. | Explicit go plus a real GPU runner, per the resolved decision that this is the one deliberately-not-real piece of this band's infrastructure. |
| 6 | **`embedding-service`'s "only writer of Qdrant" property is enforced by code convention only.** The dev Qdrant instance has no API key configured at all. | Turning the acceptance check from a grep-verified convention into an infrastructure-level control. | Likely `mytheus-platform`: a Qdrant API key scoped to `embedding-service` alone. |
| 7 | **No sustained-load test exists for `embedding-service`'s sub-5 s consumer-lag SLO.** Only single-message tests were run. | Confidence in the lag/DLQ Grafana panels the plan's P6 demo names, which were never built by any task this band. | A dedicated load-test and observability task, likely paired with gap 6. |
| 8 | ~~**`tenant_settings.pii_redaction_enabled` has no real publisher anywhere in the program.**~~ **CLOSED 2026-08-28** by `mytheus-backend` `590d371`: `GET`/`PATCH /tenant-settings` on `identity-tenant-service`, admin tier, audited as `tenant_settings.updated`, plus `/tenant-settings` added to the gateway `ROUTE_MAP`. The route writes **both** stores, because the column alone would have changed nothing: `ingestion-service` never queries `tenant_settings` and cannot, so `nexus:{org_id}:tenant_settings:pii_redaction_enabled` is the actual delivery path. The mirror write is the last statement inside the same transaction as the column write and its audit row, so a Redis failure rolls the column back and returns `503 DEPENDENCY_UNAVAILABLE` rather than a `200` that would report a setting as in effect when ingestion is still using the old one; the choice is driven by the two stores failing asymmetrically (mirror stale-ON over-redacts and is recoverable, mirror stale-OFF under-redacts and is not). Upsert rather than update, because `provision_organization()` creates no `tenant_settings` row and an `UPDATE` would have affected zero rows for every org that exists. 10 tests against real Postgres and real Redis; the load-bearing ones assert the Redis key rather than the column, and removing the mirror write fails 5 of them. | - | Done. See section 5.3. |
| 8a | **Postgres and the AI tier disagree about the default, and did so silently.** Found while writing gap 8's route. `tenant_settings.pii_redaction_enabled` has `server_default` **`true`**; `ingestion-service`'s `pii_enabled()` treats a missing Redis key as **`false`**. Until a tenant is explicitly written, the column says redaction is on for every organization and the tier that performs redaction believes it is off for every organization. Neither side is wrong in isolation and both were disclosed in their own records, which is why nothing caught it: the divergence only exists in the gap between them. Gap 8's route removes it for any tenant it is used on, and `test_read_reports_the_column_default_when_no_row_exists` records the discrepancy at the point a reader would hit it. | Any claim that redaction is on by default. It is not, in the only tier that performs it. | Needs a decision, not a code fix: either the column default becomes `false` to match the effective behaviour, or org provisioning writes the row and the mirror so the two agree from birth. The second is the better fix and belongs with the `tenant_settings`-row omission noted in gap 8. |
| 8b | **`provision_organization()` does not create a `tenant_settings` row.** Verified: it inserts `organizations` and a `tenant-events` outbox row, and nothing else. Every organization provisioned to date therefore has no settings row, so every `tenant_settings` column silently reports its declared default rather than a stored value. Gap 8's route works around this with an upsert; the provisioning omission itself is untouched. | Any code that reads `tenant_settings` with an `UPDATE`-then-check or an inner join, both of which see zero rows. | A `identity-tenant-service` change to `provisioning.py`, ideally in the same transaction that creates the org, together with the 8a default decision. |
| 9 | **The ingestion-to-embedding hop is proven as two independently-real halves, not one connected live run.** This task's producer and task 7's consumer were never exercised together against the same live Redpanda topic in a single test. | Full confidence in the P6 demo's ingest-to-`document_chunks__{org_id}` traversal. | A cross-task integration test, once both services' worktrees merge to a shared branch. |
| 10 | **`mytheus-mobile` section 7 (person-profile screen, Clerk Expo auth) was never assigned to a task this band and remains unbuilt.** Unlike `mytheus-ui`, no decision recorded it as out of scope. | The mobile half of the P5 demo. | Needs an explicit scoping decision the same way `mytheus-ui`'s exclusions have had one, then a task. |
| 11 | **The plan file's own `8.2` `DECISION-REQUIRED` blockquote still reads as an open question**, even though the product owner resolved it and no canon change was needed. | A reader of the plan text alone, without this record, would not know 8.2 is closed. | A small propagation edit to `plan/plan-5-6-org-context-and-ai-infra.md` section 8, mirroring how Plan 3-4's propagation pass marked `8.1` RESOLVED in place; out of this record's own scope, which records but never defines. |
| 12 | **`get_user_context()`'s cache is not invalidated on team-membership change.** `invalidate_user_context` is exported but unwired, because no route in the codebase writes `team_members` yet (gap 4). | Cache correctness the moment gap 4 is closed and a real team-membership write path exists. | Whichever task builds the `core-api` team-membership write path; wiring the existing `invalidate_user_context` call is small once that lands. |
| 13 | ~~**`design/openapi/bff.yaml` is stale against the shipped `dashboard.py` response shape.**~~ **CLOSED 2026-08-28**, by removing the possibility rather than by reconciling once. `bff.yaml` is now emitted from the app ([ADR-0022](../adr/0022-openapi-emitted-from-code.md)), so it carries the real `{status, items}` arm shape and this band's two routes because it cannot carry anything else. A reconciliation pass would have fixed the instance and left the mechanism, and the mechanism is what produced it. | - | Done. |
| 14 | **A latent lockfile hazard in `mytheus-events`**: regenerating `uv.lock` from scratch currently pulls an unpinned `ruff>=0.4` forward to a version that breaks `make lint` with unrelated import-sort errors. Found and worked around (incremental relock instead of a full regen) during the client-packaging fix, not fixed at the root. | Any future routine, well-intentioned lockfile regeneration in that repo. | Pinning `ruff` to the version currently in use (0.15.20). |
| 15 | **Every open gap the Plan 3-4 record carried forward that this band did not touch remains open** - most notably the platform-level partition-grant leak (`00-status-overview.md` live gap list item 10) and the unfixed `Idempotency-Key` cross-endpoint collision (item 3). Neither surfaced again in this band's work, and neither was newly introduced here. | See [03-plan-3-4-meeting-and-transcription.md](03-plan-3-4-meeting-and-transcription.md) section 4 and [00-status-overview.md](00-status-overview.md) for the full carried list. | Unchanged from the prior record. |

---

## 5. Verification

Every command below was run and its output observed during this band, in each repo's Plan 5-6 worktree against the shared dev stack (Postgres, Redis, Redpanda, Schema Registry, Qdrant, MinIO). Nothing here is inherited from a prior band's claim.

| Repo | Command | Result |
|---|---|---|
| `mytheus-backend` | `uv run pytest -q` (final run, after task 5) | **244 passed, 0 skipped** (10 deselected per the `test_roles_and_policies.py` do-not-run instruction) |
| `mytheus-backend` | `ruff check .` / `black --check .` / `mypy .` | Clean, project-wide |
| `mytheus-backend` | `alembic upgrade head` then `alembic downgrade -1` then `alembic upgrade head` (migration `56f72ac73fa9`) | No errors on any leg |
| Live DB (direct query) | `SELECT relrowsecurity, relforcerowsecurity FROM pg_class WHERE relname IN ('teams','team_members','departments')` | `t, t` for all three |
| Live DB (direct query, as `mytheus_app` with `app.current_org` set) | Cross-tenant read of `departments`/`teams`/`team_members` seeded under org A, queried under org B | 0 rows for all three, proving RLS `FORCE` end to end, not just at the policy-definition level |
| `mytheus-backend` (`bff`) | Live measurement against real seeded data | `GET /bff/org/chart` for acme (3 departments, 3 teams, 10 members): exactly 3 SQL statements, 7.85 ms |
| `mytheus-events` | `make validate` | 21 topics OK |
| `mytheus-events` | `make lint` | Clean |
| `mytheus-events` | `make register` + `make check-compatibility` | 21 subjects BACKWARD against the live registry at `:8081` |
| `mytheus-events` | `make codegen` | Zero diff after, working tree clean |
| `mytheus-events` | `make test` (Python and TypeScript) / `npx tsc --noEmit` | Green |
| `mytheus-events` | `rpk topic list --brokers localhost:9092` | All 21 topics + 21 `.dlq`, partition counts matching the plan's section 4.2 table exactly |
| `mytheus-events` | Python client install into a throwaway virtualenv (before and after the packaging fix) | Failed with no `[build-system]` table; succeeded after `38dab78` |
| `mytheus-events` | TypeScript client import from a real consumer project (before and after the packaging fix) | No installable package before; real types importable after `3758cdb` |
| `mytheus-intelligence` | `uv run pytest -q` (final run, after task 9) | **112 passed, 0 failed** (37 baseline plus 75 new) |
| `mytheus-intelligence` | `ruff check .` / `black --check .` / `mypy --strict` | Clean |
| `mytheus-intelligence` (`embedding-service`) | `tests/test_embedding_consumer.py::test_transcript_chunk_end_to_end_real_embedding_real_qdrant_real_outbox` | Real Avro-encoded message through real BGE-small-en embed through real Qdrant upsert, payload asserted field-by-field |
| `mytheus-intelligence` (`embedding-service`) | Real batch-of-32 latency measurement | 3536 ms warm on CPU (recorded as BLOCKED against the < 100 ms GPU target, never claimed) |
| `mytheus-intelligence` (`embedding-service`) | `test_qdrant_tenancy.py::test_cross_tenant_wrong_payload_filter_returns_zero_even_in_right_collection` | 0 points returned |
| `mytheus-intelligence` (`embedding-service`) | `test_outbox_relay.py::test_relay_publishes_embedding_events_row_to_real_redpanda` | Row Avro-serialized against the live `embedding-events-value` subject, produced to real Redpanda, read back by a real consumer |
| `mytheus-intelligence` (`reranker-service`) | Real BGE-reranker-base scoring, 50 to 10 passages, concurrent two-org test | 283-440 ms `model.predict`, 281-376 ms end to end; zero cross-tenant contamination in both the concurrent-request and model-layer tests |
| `mytheus-intelligence` (`ingestion-service`) | `tests/test_ingestion_routes.py::test_post_ingest_of_a_sample_pdf_is_the_plans_literal_acceptance_check` | Real PDF through the real route, real MinIO `head_object` at the exact key, Postgres `element_type` values and outbox row count both match `chunk_count` |
| `mytheus-intelligence` (`ingestion-service`) | `tests/test_ingestion_pipeline.py::test_rerun_with_same_document_id_is_idempotent` | Zero additional `document_chunks`/`document_elements` rows on rerun |
| `mytheus-intelligence` (`ingestion-service`) | `test_missing_org_id_on_integration_events_goes_to_dlq_never_guessed` | Poisoned message correctly routed to DLQ |
| `mytheus-intelligence` (`ingestion-service`) | Full-suite re-run after the disclosed `torch` regression | 37/37 baseline (`embedding-service` plus `reranker-service`) confirmed green again before further work |

**NOT run, and not claimed:** any GPU-hardware latency measurement for `embedding-service` or `reranker-service`; a sustained-load test for consumer lag; a single connected live run of `ingestion-service`'s producer against `embedding-service`'s running consumer; anything involving `search-service` (task 10, pending) or a browser/mobile client (neither built this band).

### 5.3 Follow-up session: gap 8, the `pii_redaction_enabled` write path

`mytheus-backend` `590d371`, run against real Postgres and real Redis on a local Compose stack.

| Command | Result |
|---|---|
| `uv run pytest tests/test_tenant_settings_api.py -q` | **10 passed** |
| Guard-fires check: delete the `_publish_pii_mirror` call, re-run | **5 of the 10 fail**, including the three that assert the Redis key. A column-only implementation cannot pass this file |
| `uv run pytest -q` (full suite) | **300 passed, 0 failed** (was 290 before this change) |
| `ruff check .` / `black --check .` / `mypy .` | Clean; mypy over 89 source files |
| `python scripts/check_authz.py` | Passes; both new routes declare `require_tier("admin")` |
| `uv run pytest tests/test_gateway_edge.py -q` | 20 passed, after adding `/tenant-settings` to `ROUTE_MAP` |

**One test-infrastructure fix rode along, recorded because it will bite the next module that touches Redis.** `libs/shared/cache.get_redis` is `lru_cache(maxsize=1)`, so the async client belongs to whichever `TestClient`'s event loop built it first. A second module-scoped client reuses a connection whose transport is attached to a closed loop, and the failure surfaces inside `send_packed_command` rather than at connect time, so it reads as a Redis outage rather than as loop reuse. It passes in isolation and fails in a full-suite run, which is the same shape as the `conftest.py` engine-URL bug in Plan 1-2 section 10.7. The fixture now clears that cache alongside the two engine caches, in both directions.

**NOT run, and not claimed:** anything against the deployed VPS. The route has not been exercised through the live `api-gateway`, and no real tenant has had redaction switched on. Turning it on for a real organization also depends on that organization existing, which it does not (`00-status-overview.md` records the live database as holding zero organizations).

---

## Task 10: search-service

Landed as commit `0d0a42f` in `mytheus-intelligence`. 134 tests pass (112 before this task, 22 new),
all against real Postgres, Redis, Redpanda and Schema Registry. `ruff`, `black` and `mypy` clean.

What shipped: the lexical arm of hybrid retrieval, `GET /search`, over Postgres full-text search.

The ownership question this task faced is the most instructive thing in it. Full-text search wants a
`tsvector` column and a GIN index on the text being searched, and the text lives in
`transcript_chunks`, which is owned by `transcription-service` in `mytheus-realtime`, and in
`chat_messages`, owned by `core-api`. Adding a column to either would have made this service a
second writer of a table it does not own.

It did not do that. It confirmed against the live database that both tables grant the
`mytheus_ai_app` and `mytheus_ai_relay` roles no access at all, then built
`intelligence.transcript_chunk_search`: this service's own read model, carrying the plan's literal
`GENERATED ALWAYS AS (to_tsvector(...)) STORED` column and its GIN index, populated by a new
`TranscriptChunkSearchConsumer` subclassing the shared `BaseConsumer` and consuming the same real
`transcript-chunks` topic that `embedding-service` already consumes. Building an owned read model
rather than reaching into another service's table is the correct resolution, and it is the third time
in this band a task refused a second-writer temptation.

The `chat_messages` arm is BLOCKED, and reported rather than faked. There is no `chat-messages` topic
anywhere in the frozen 21-subject registry, and `chat_messages` has no event production path at all,
since `core-api` writes it directly. So there is neither an event to build a read model from nor a
grant to read the table. Closing it needs either a new canon topic or a cross-repo grant, both of
which are decisions above this task.

Isolation was proven adversarially rather than by a filter assertion. Two organizations were given
content sharing a distinctive phrase, one of them with twenty times the other's matching volume and
higher-ranking text, and the query limit was set smaller than the larger tenant's match count. The
smaller tenant still received its true matches: neither starved out of the result page nor leaked
into. This is the shape of test that matters for search, because a tenant filter applied only to the
final page can leak through ranking and totals even when no foreign row is ever returned. A second
test proved that RLS alone, with no explicit predicate in the query, isolates independently.

The output shape mirrors `reranker-service`'s `RerankedPassage.passage` exactly, so `GET /search`
results feed `POST /rerank` with no adapter between them, and the citation-provenance fields
(`chunk_id`, `meeting_id`, `speaker_id`) ride alongside for the citation stage in a later band.

One shared-infrastructure finding came out of it that future work needs to know: the shared
`BaseConsumer` names its dead-letter topic after the source topic alone, not the consumer group, so
`transcript-chunks.dlq` is shared between `embedding-service` and `search-service`. This task's first
test draft reused an earlier task's hardcoded idempotency key and genuinely picked up that task's
leftover message. It was fixed with a unique key and scan-forward matching, and documented in the
test. Any future test asserting against a dead-letter topic must not assume it owns that topic.

## Final review

Run after all ten tasks completed, across both code repos, with full coverage: both diffs read, the
live database inspected directly rather than the migration source, the four contract seams verified
field by field, and `org_id` traced end to end across every repo and store boundary.

Verdict: MERGE WITH FIXES. One Important finding, one Minor, and nothing above them.

### Important: the AI tier's REST surface trusts an unauthenticated header

`ingestion-service`, `search-service`, `reranker-service` and `embedding-service` all take `org_id`
from a client-supplied `X-Org-Id` request header, failing closed only when that header is a malformed
UUID. None of them verifies a JWT or a gRPC identity.

Concretely: anyone with network reachability to `ingestion-service` can `POST /ingest` carrying
another organization's id in that header and write into that organization's object storage and vector
collection, or call `GET /search` with it and read that organization's transcript index.

This contradicts an invariant the sibling repo states explicitly. `mytheus-backend`'s own guidance
requires each service to re-verify the JWT independently, on the grounds that the edge gateway is a
denial-of-service shield rather than a trust delegation, and `identity-tenant-service` genuinely does
so. The canonical service catalog assigns JWT validation and `org_id` extraction to `api-gateway`,
which does not exist yet, so there is currently nothing upstream performing that check either.

The way this arrived is worth recording, because the code is not the interesting part. Each of the
four tasks disclosed this individually, as an accepted interim convention, and each was a locally
reasonable call at the time. What never happened was checking that local convention against the
sibling repo's stricter standard, or escalating it as a band-level risk. Four defensible local
decisions composed into an unauthenticated write path over live infrastructure. No ADR blesses the
deviation. The program has already shipped one authorization-bypass defect from precisely this
class of gap, where every component was individually correct.

It is latent rather than exploited: nothing in this band calls these routes across a repo boundary
yet. But the infrastructure is real and already holds tenant data. This needs an ADR and a fix before
`api-gateway` lands or any real caller reaches these routes, and it should be tracked as one
band-level item rather than left as four independent interim comments.

#### CLOSED. The fix shipped 2026-08-06; the ADR followed on 2026-08-28.

The finding above is left unedited, because what it describes was true when it was written and the
account of how four local decisions composed into it is the part worth keeping.

**The code fix.** `mytheus-intelligence` commit `9ea2f9b`, merged to `main` the same day. It adds
`libs/shared/auth.py`, a faithful reimplementation of `mytheus-backend`'s Clerk verifier (RS256
against cached JWKS, `nbf` required, a JWKS outage mapped to `503` rather than `500`, no error
message echoed back as an oracle), exposing one dependency, `verified_org_id`. All four services
depend on it: `services/ingestion_service/routes.py`, `services/search_service/routes.py`,
`services/embedding_service/query_embed.py`, `services/reranker_service/rerank.py`. The
`Header(..., alias="X-Org-Id")` binding is removed rather than deprecated, so `org_id` has no
unauthenticated source on this tier any more.

**The ADR.** [ADR-0019](../adr/0019-ai-tier-verifies-jwt-and-reads-resolved-org-claim.md), written
three weeks after the code and recording that lateness as a process failure rather than presenting
the canon-first ordering as having been followed. It carries the decision the docstring held: this
repo does NOT reimplement `org_resolver.py` and does not read `public.organizations`, so the claim
it reads (`CLERK_JWT_INTERNAL_ORG_CLAIM`, default `internal_org_id`) is expected to carry the
already-resolved internal `org_id`, not Clerk's opaque native id.

**Two things this does not close, both stated in ADR-0019 and repeated here so a reader of the
records alone is not misled.** First, the routes fail closed with `401` on every request until the
Clerk JWT template actually populates that claim, which is a Clerk-dashboard and
`identity-tenant-service` change outside `mytheus-intelligence`, and is the same work as
provisioning the first real organization. "Deployed" is not "reachable and authenticated": that
requires confirming the claim is present in a minted token. Second, this tier still cannot
distinguish a suspended organization from an active one, because the not-suspended check lives in
the `organizations` read this repo is not allowed to do; a token minted before a suspension stays
usable here until it expires.

**How this was found, which is the reusable part.** A planning pass on 2026-08-26 re-verified this
finding against the repos and reported it still open in all four services. It had been closed for
twenty days. The check was made against a clone that had not been fetched, which is precisely the
trap [02-plan-1-2-foundation-and-tenancy.md](02-plan-1-2-foundation-and-tenancy.md) section 14.3
already records: `git status` compares against the last fetch, not against the remote, so "clean and
up to date" means nothing on a repo nobody has pulled. A stale-clone check is not a verification,
and this record was itself the evidence used to justify the re-verification, which is why an
un-updated record is not a cosmetic problem.

### Minor: dead-letter topics are scoped by source topic, not consumer group

`embedding-service` and `search-service` both consume `transcript-chunks` and both dead-letter to
`transcript-chunks.dlq`, distinguished only by a field inside the payload. Nothing reads dead-letter
topics today, so this is not exploitable, but any future replay tooling must filter by consumer group
rather than assuming a topic belongs to one consumer. A test in this band already tripped over it by
matching an earlier task's leftover message.

### The two cross-task properties that were specifically checked, both clean

The two-repositories-one-database separation is genuinely complete, verified live rather than
inferred: the AI roles hold no role attributes granting bypass, there are zero cross-schema grants in
either direction, and the schema keeps its own migration version table.

The `intelligence` schema's row-level security matches the backend's standard exactly. All six tables
have row security both enabled and forced, and every `tenant_isolation` policy predicate is
byte-identical to the fail-closed form used elsewhere in the program. No partitions were created in
this band, so the partition-grant rule that produced a real cross-tenant leak in the previous band has
nothing to apply to here. That is an absence of risk, not an unchecked gap.

The four contract seams between independently built services were verified to match field for field
rather than assumed: search results into the reranker, the two independent consumers of
`transcript-chunks`, and ingestion's `document-chunks` output into the embedding consumer.

---

## 7. Follow-up session, 2026-08-29: the team-membership write API, and a cross-tenant write it exposed

`mytheus-backend` `9f51192` and `602bff4`. Closes the product half of gap 20 and the
`team_members` half of Plan 5-6 section 3.3's cache-invalidation rule. No canon edit was
needed, and that was checked rather than assumed: canon constrains neither this route's path
nor the audit action vocabulary, and it defines no topic that would carry a membership change.

### 7.1 What landed

`core-api` gains three routes on the table it already owns:

| Route | Tier | Notes |
|---|---|---|
| `GET /api/v1/teams/{team_id}/members` | `member` | The roster. Who is on a team is not privileged inside a tenant. |
| `PUT /api/v1/teams/{team_id}/members/{user_id}` | `admin` | Upsert. |
| `DELETE /api/v1/teams/{team_id}/members/{user_id}` | `admin` | `404` when the row is absent, never a silent `204`. |

`PUT` rather than `POST` because `(team_id, user_id)` is the primary key: adding someone
already on the team is the same end state, and `POST` would force a caller correcting a role
to `DELETE` first. Both writes call `invalidate_user_context`, which
`identity-tenant-service` has exported and unwired since Plan 5-6 for want of a write path;
the import crosses services in the direction the docstring specifies, matching the existing
`bff` to `get_user_context` precedent, so neither service writes the other's store.

**No event is emitted, deliberately.** `_canon/03-event-taxonomy.md` defines no topic carrying
team-membership changes, and `tenant-events` is produced by `identity-tenant-service` alone.
Emitting anything here would either invent a name canon does not define or add a second
producer to someone else's topic. Recorded in the module so the next reader does not treat it
as an oversight.

### 7.2 The defect this exposed, and the false claim it corrects

Removing the new `_visible_user_id` check to prove the guard fired did not produce the tidier
error message its first docstring predicted. It produced **`200` and a persisted row joining
this tenant's team to another tenant's user.**

The cause generalizes, which is why it is worth recording as a property of the substrate
rather than a detail of one route: **PostgreSQL performs referential-integrity checks as the
referenced table's owner with row-level security bypassed.** A foreign key therefore resolves
a `users` row the transaction cannot `SELECT`, and the table's own `tenant_isolation` `WITH
CHECK` does not help, because the `org_id` being written is the caller's own. The column that
crosses the boundary is `user_id`, and no policy constrains it.

Checking whether the same shape existed elsewhere found it in shipped code.
`services/core_api/routes/meetings.py` carried this comment: "The FK to users plus RLS means a
participant id belonging to another tenant is simply not insertable ... A cross-tenant
participant is unrepresentable rather than merely rejected." Verified against the live dev
database before being claimed: `POST /api/v1/meetings` with another tenant's `user_id` in
`participant_ids` returned `201` and wrote a `meeting_participants` row whose `org_id` was the
caller's and whose `user_id` was not.

**This is the third documented control in this program found not to exist**, after
[ADR-0021](../adr/0021-explicit-grants-no-schema-wide-default-privileges.md)'s arm 2 and
[ADR-0022](../adr/0022-openapi-emitted-from-code.md)'s defect 3. All three share one shape: a
comment or a ruleset asserting a guarantee that nobody had made fail on purpose.

**Scope, stated precisely rather than inflated.** It is NOT a cross-tenant read. Every read
path filters on the denormalized `org_id`, so the victim tenant never sees the row, and
`get_user_context`'s `recent_meetings` join excludes it; both were checked, not assumed. It IS
a reference resolving outside the tenant, which is the property the Plan 1-2 D6 foreign-key
work exists to guarantee and which right-to-be-forgotten depends on to find everything.
Exploiting it needs a `users.id` from another tenant, which is a UUID and not guessable.

Fixed in `602bff4` with an explicit existence check before the participant insert, returning
`404` rather than `422` so the route does not become an oracle for user ids in other tenants.

### 7.3 Verification

Every command run and its output observed.

| Command | Result |
|---|---|
| `uv run pytest -q` | **325 passed** (was 313; twelve new tests) |
| Guard-fire: both `invalidate_user_context` calls removed | Exactly the two eviction tests fail, nothing else; green again after repair |
| Guard-fire: write tier lowered `admin` to `member` | The authz test fails; green after repair |
| Guard-fire: `404` changed to `403` and the user check dropped | Both isolation tests fail; green after repair |
| Guard-fire: the participant check removed from `meetings.py` | The new cross-tenant test fails with the row-creating `201`; green after repair |
| `emit_openapi` then `--check` | Three new paths emitted; `core-api` now 11 paths; check green |
| `spectral lint --fail-severity=error` | **0 errors**, 74 warnings (71 before; three built-in style warnings, one per new route) |
| `scripts/check_authz.py` | Passes; every new route declares its tier |
| `ruff` / `black --check` / `mypy` | Clean; mypy over 93 source files |

Two claims were probed directly rather than reasoned about, and both held: `tenant_session`'s
"commit together or not at all" (a `ProblemDetail` raised after a write does roll the write
back), and the assumption that a surviving row proved otherwise. The apparent counter-example
was a row committed by the earlier RED run, because the shared dev database is never truncated
between runs. The assertion is now scoped to the run's own `org_id` so a leftover cannot
reproduce it. **Worth carrying forward: on this suite, a global predicate in an assertion can
be satisfied by a previous run's data.**

### 7.4 NOT run, and not claimed

- **No rebuild from an empty cluster.** Unchanged from the previous session and for the same
  reason: the dev stack is shared with the other track. Gap 30 still stands.
- **Nothing was exercised through `api-gateway`.** The routes were driven directly against the
  `core-api` router in-process, as every other route test in this repo is. The gateway's
  `ROUTE_MAP` already covers `/api/v1/teams` by longest-prefix match, so the new paths route by
  construction, but that was read rather than executed.
- **No client consumes these routes.** Neither `mytheus-ui` nor `mytheus-mobile` has a team
  management surface, so the product gap in item 20 is closed at the API layer only.
- **The `oasdiff` job still has not run in GitHub Actions.** These three routes are additions,
  which are not breaking, so the next pull request remains its first real execution.

---

## 8. Related

- [00-status-overview.md](00-status-overview.md) for program-wide state.
- [03-plan-3-4-meeting-and-transcription.md](03-plan-3-4-meeting-and-transcription.md) for the meeting and transcription tier this band builds on, including the `transcript_chunks` partition-grant defect and its fix, directly relevant to gap 15 above.
- [adr/0006-vector-store-tenancy.md](../adr/0006-vector-store-tenancy.md) and [adr/0017-transcript-chunks-composite-partition-key.md](../adr/0017-transcript-chunks-composite-partition-key.md) for the two decisions this band's `DECISION-REQUIRED` callouts resolved against.
- [plan/plan-5-6-org-context-and-ai-infra.md](../plan/plan-5-6-org-context-and-ai-infra.md) for the plan this record audits.
