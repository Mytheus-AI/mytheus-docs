# Samar — Platform, Deploy, Data Safety, Runtime

**17 tasks.** P1 is the most dangerous defect in the audit and comes before everything, including
the outages.

Audit: https://claude.ai/code/artifact/da281c5f-d990-40b4-9487-d9f857833b08
Evidence: [`audit/`](audit/) · [`verify/verify-data-platform.md`](verify/verify-data-platform.md)

---

## ⏱️ Ordering

**P1 first — it destroys data.** Then P2 and P3 (live outages). P4–P8 can run in parallel with
Sushan's blockers since they are off the request path.

**You block others on P4 + P5.** Nothing in `mytheus-intelligence` or `mytheus-realtime` runs
anywhere today, so nobody can verify transcript, embedding or search behaviour until those land.

**Three defects must ship *with* the deploy tasks, not after** — P13, P14 and P16. Your deploy work
is what arms them.

---

## P1 — `make migrate` will drop tables (P0, DATA LOSS)

```make
migrate:
	$(ALEMBIC) revision --autogenerate -m "$(msg)"
	$(ALEMBIC) upgrade head
```

Autogenerate **and immediately apply**, no review step. `migrations/env.py:27` sets
`target_metadata = Base.metadata` and defines **no `include_object` filter**.

There is no ORM model for `meeting_rooms`, `meeting_speakers` or `chat_messages` (confirmed at
`866f7eedb0e7:31`). And `transcript_chunks` lives in the same physical database but is owned by
`mytheus-realtime`, so it is not in this repo's metadata either.

**The next person who runs `make migrate` to add a column generates a revision that drops three
tables, every partition, and another repo's table — and applies it in the same command.**

Everything else in this plan is an outage. This is silent data loss triggered by the documented
workflow.

- [ ] Split `migrate` into `revision` and `upgrade`. Never autogenerate-and-apply.
- [ ] Add an `include_object` filter in `env.py` excluding tables with no model, all partitions, and
      anything owned by another repo
- [ ] Add ORM models for `meeting_rooms`, `meeting_speakers`, `chat_messages`, or exclude them
      explicitly — pick one and be consistent
- [ ] A test asserting autogenerate produces **no** DROP against a migrated database
- [ ] Check whether this has already fired anywhere

---

## P2 — `chat_messages` inserts fail (P0, live since 1 Sep)

`5ed978467cad:54-55` hardcodes `_PARTITION_YEAR = 2026, _PARTITION_MONTH = 8`; `:88-92` creates one
partition covering `[2026-08-01, 2026-09-01)` with **no DEFAULT**. `POST /chat` has raised since
2026-09-01.

The hardcoding is deliberate and correct — *"The partition set must not depend on when the
migration happens to run."* Keep that. The gap is that nothing creates later partitions.

**Correction to an earlier claim:** `audit_logs` **does** have a DEFAULT partition
(`c3e8b1d47f92:92`), so it degrades silently rather than hitting this wall. Still worth solving
both with one mechanism.

- [ ] Migration adding a DEFAULT partition to `chat_messages`
- [ ] Forward window of monthly partitions
- [ ] A rolling-partition mechanism (`pg_partman` or a scheduled task) — there is none anywhere
- [ ] Confirm rows were rejected, not silently dropped (rejection is expected)

---

## P3 — Deploy verifies topics nothing creates (P0)

`deploy/deploy.sh:87` gates on 42 topics and 21 subjects existing. Nothing on the deploy path
creates them and `mytheus-events` is not cloned on the VPS.

`mytheus-events/scripts/create_topics.sh` is good — idempotent, matches the contract list exactly.
It is simply never invoked.

- [ ] Provision topics + subjects on the deploy path, before the verify gate
- [ ] Decide how `mytheus-events` reaches the VPS (vendor the script, ship it in the image, or
      clone) and document it
- [ ] Keep the verify gate

---

## P4 — Two repos have no build path at all ▶ start now

**Correction to my earlier framing: this is bigger than Dockerfiles.** Neither
`mytheus-intelligence` nor `mytheus-realtime` has **any `.github/workflows`** — no CI, no image
build. There is literally nothing to deploy.

That is 9 of 21 services, including everything genuinely well-built in the AI tier and
`realtime-gateway` (a complete 639-line TypeScript service).

- [ ] Dockerfile for `mytheus-intelligence` — one image, per-service entrypoints, mirroring how
      `mytheus-backend` already does it
- [ ] Dockerfile for `mytheus-realtime` — two runtimes, Python and Node
- [ ] CI for both: lint, test, image build + push
- [ ] Bake or cache the model weights (BGE-small, bge-reranker-base) — do not download on cold start

---

## P5 — Compose entries for everything that should run ▶ start now

- [ ] **Intelligence workers:** `embedding_service/worker.py`, `search_service/worker.py`,
      `ingestion_service/worker.py`, the intelligence outbox relay (`relay_main.py`)
- [ ] **Realtime:** transcription-service, realtime-gateway
- [ ] **Backend consumers that are already in the deployed image and still not running:**
      `core_api.consumer` and `meeting_service.consumer`. Both built, neither has a process.
- [ ] **Qdrant** — required by the embedding path, absent
- [ ] **LiveKit** — absent; unset `NEXT_PUBLIC_LIVEKIT_URL` is why the UI degrades to
      `status: "unconfigured"`
- [ ] **Object storage** — ingestion does a real put; R2 (ADR-0023) is ADR text plus an unimported
      dependency
- [ ] Wire `depends_on` against the healthchecks from P9

---

## P6 — Nothing can route to the new services

Even after P4 and P5, there is **no ingress**. `edge.py:146-180` ROUTE_MAP has no entry for any
intelligence path, and `mytheus.caddy` has no WebSocket route for `realtime-gateway`.

- [ ] ROUTE_MAP entries for the intelligence services that should be publicly reachable
- [ ] Caddy WS route for `realtime-gateway`
- [ ] Confirm the ingress docs match the Caddy config — the audit found them contradicting in three
      places, and no script in `mytheus-platform` deploys that Caddy file

---

## P7 — Resource limits, backups, log rotation ▶ start now, before P5 lands

Grep found **no `deploy:`, `resources:`, `mem_limit`, `cpus` or `logging:`** anywhere in
`compose.prod.yml`. 17 containers on one VPS, alongside a second production product — and P5 adds
nine more, two of which load ML models into memory.

Also absent: **backups of any volume**, and the `observability/` directory the README documents.

- [ ] Memory and CPU limits on every container — **before** P5
- [ ] Log rotation; an unbounded json-file driver across 26 containers will fill the disk
- [ ] Backups for `pgdata`, `redisdata`, `redpandadata`, and Qdrant once P5 lands
- [ ] Sanity-check capacity: Postgres + Redis + Redpanda + Temporal + Qdrant + two model servers on
      one node

**Correction:** OTel *tracing* is genuinely wired (`libs/shared/telemetry.py`, called from
`app_factory.py:26`). Only **metrics** are zero — no Prometheus, no Grafana, no service exposes
metrics, so the lag and DLQ panels Plan 5-6 names have no datasource.

---

## P8 — Make a green deploy mean something

Only **7 of 16** long-running services have a healthcheck; the other 9 are invisible to both
`deploy.sh`'s gate and `depends_on: service_healthy`. `api-gateway` and `ui` have one but are
ungated by `deploy.sh`. `temporal` has none and `workflow` gates on `service_started` only.

- [ ] Healthchecks for the 9 without one
- [ ] Gate `deploy.sh` on `api-gateway` and `ui`
- [ ] **Add an authenticated smoke request to the post-deploy gate.** A public `/healthz` 200 is
      why Sushan's S1 outage reports green.

---

## P9 — No startup env validation anywhere

The generating mechanism behind S1, `prod.ui.env` and the Clerk gap: **nothing validates required
config at startup**, and all ten compose `env_file` entries are `required: false`, so a missing
file is silent.

- [ ] Fail-fast validation of required env at service startup
- [ ] Set `required: true` on `env_file` entries
- [ ] `prod.ui.env.example` — read at `compose.prod.yml:381`, no example exists anywhere
- [ ] `mytheus-intelligence/.env.example` has **no `CLERK_*` variable at all**, and `auth.py:80/84`
      uses bare `os.environ[]` — every route there **500s rather than 401s**
- [ ] Full list of code-required-but-undocumented vars is in `verify/verify-data-platform.md` §4.4

---

## P10 — Redis port drift, in both directions

⚠️ **Correction — the drift is inverted between repos. Fixing only example files leaves two of four
wrong.**

- `mytheus-intelligence`: `.env.example` says **6379**, code defaults to **6380** → fix the example
- `mytheus-realtime`: `.env.example` says **6380** (correct), the **code** defaults to **6379**
  (`transcription config.py:35`, `gateway server.ts:593`) → fix the code

- [ ] Fix each repo in the correct direction; do not batch-edit

---

## P11 — UI env vars can never work in a container

`mytheus-ui/Dockerfile:29-35` — `NEXT_PUBLIC_REALTIME_WS_URL` and `NEXT_PUBLIC_LIVEKIT_URL` are not
Docker build args. Next.js inlines `NEXT_PUBLIC_*` at build time, so containers bake them as
`undefined` and no runtime env fixes it.

- [ ] Pass them as build args
- [ ] Coordinate with Shaun — his D17 cannot be verified until this works

---

## P12 — Gates that cannot fail

- `mytheus-platform` `make test` runs `check_authz.py || true` — **the authz gate can never fail**
- `emit_openapi --check` exits 0 on the wrong interpreter
- The oasdiff breaking-change gate was fixed in-file but **has still never executed in Actions**
- `mytheus-ui` CI does not run `generate:api:check`, so the generated-types guarantee does not hold
- CD concurrency is per-repo, so the three repos with CD can race on a deploy

- [ ] Remove the `|| true`
- [ ] Fix the interpreter
- [ ] Confirm oasdiff runs and can fail a build
- [ ] Add `generate:api:check` to UI CI
- [ ] Cross-repo deploy serialisation or a lock

---

## P13 — transcription-service DB role ⚠️ ships with P4/P5

`transcription-service/db.py:13-22` connects as the `postgres` superuser. Its RLS code is
**correct** — it always issues `SET LOCAL app.current_org` — but a superuser bypasses RLS entirely,
so those calls protect nothing.

The deferral is documented and the reasoning is sound: the backend's roles are created with **no
password** by design (`a1c4e7f20b18` — *"cannot authenticate until an operator sets it"*), and a
cross-repo grant is an ops concern. It is harmless **only because the service is not deployed**.
P4/P5 change that.

- [ ] Least-privilege role mirroring `mytheus_app` — `NOSUPERUSER NOBYPASSRLS`
- [ ] Grant exactly: write on `transcript_chunks` and `outbox`; read on `tenant_settings` and
      `data_residency`. Use `grant_tenant_table` — this is the case it was built for.
- [ ] Point production `DATABASE_URL` at it; leave the local dev default alone
- [ ] Startup assertion that the connected role lacks `rolbypassrls` — `libs/shared/db.py:49` has no
      such check either
- [ ] Close the cross-repo GRANT gap documented at `9b2c4d1e6f30:73-95`, still unapplied in
      `mytheus-realtime` — it breaks the transcript endpoint on any clean-cluster rebuild

---

## P14 — `FakeProvider` writes fabricated transcripts ⚠️ ships with P4/P5

`providers/factory.py:20-23` returns `FakeProvider` whenever `DEEPGRAM_API_KEY` is unset — every
environment. A `meeting-room-events` `joined` opens that stream, and `_pump` drains its canned
script through the **same** `persistence.process_utterance` a real provider uses, writing a
`transcript_chunks` row reading *"hello there, how are you"* with `confidence=0.97`, and publishing
it to the real `transcript-chunks` topic. Nothing marks it synthetic.

`devseed.py` guards the hand-run seeder behind `TRANSCRIPTION_DEV_SEED=1` for exactly this reason.
The fake provider has no guard.

**The provider *selection* is deliberate and correct** — the factory docstring argues persuasively
that defaulting to fake fails toward "obviously not talking to Deepgram". The gap is provenance
only. **P5 is the task that arms this.**

- [ ] Add a provenance marker — a `source` column, or a reserved confidence sentinel — so synthetic
      finals are distinguishable downstream
- [ ] Refuse to run `FakeProvider` in a production environment, or make it loudly obvious

---

## P15 — Liveness lies

`transcription-service`'s `/healthz` stays green if its consumer dies — disclosed honestly at
`main.py:15-19`; `_log_consumer_exit` logs loudly but `/healthz` still returns ok, so a liveness
probe would not restart the pod.

- [ ] Make the healthcheck reflect consumer liveness
- [ ] Check the same pattern in the other consumer services as P5 lands them

---

## P16 — Defuse the notification consumer ⚠️ before it gets a compose entry

`notification_service/consumer.py:30-32,37-41` subscribes, runs an **empty `_handle`**, and
**commits the offset anyway** — destroying the backlog rather than leaving it.

Inert today only because it has no runner and no producer. **P5 is the task that would add one.**
One line to defuse.

- [ ] Stop committing offsets for messages the handler discards — or do not deploy the consumer
      until it has a real handler
- [ ] Prefer the latter: do not add a compose entry for it in P5

---

## P17 — `/stripe/webhook` is a public 404 Stripe will retry

`edge.py:52` and `libs/shared/public_routes.py:31` put `/stripe/webhook` on the unauthenticated
allowlist, and `ROUTE_MAP` routes it to `billing-metering-service`, which has **zero routes**.

Stripe retries a failing webhook and then **auto-disables the endpoint**. This is a live external
integration failure sitting in the route map.

- [ ] Remove `/stripe/webhook` from `ROUTE_MAP` and `PUBLIC_PREFIXES` until billing is real
- [ ] Same judgment for the other dead public prefixes — coordinate with Shaun's D3

---

## Do NOT build (deferred)

Kubernetes, Terraform, Helm · Wave 1–4 extraction · multi-region · cell isolation · Citus · BYOC
Redpanda · int8 Qdrant · GPU autoscaling · the `nexus-deploy` GitOps repo.

**Corrections — two first-pass claims did not survive verification:**
- `migrations/_grants.py` is **not** dead code. Migration 0005 runs last in the chain, so every
  existing table already inherited its grant. It is a helper for the *next* migration — so the only
  work is a CI check that new tables use it, folded into P12.
- **`deploy.sh` is in version control** — 110 lines with real git history. The "exists only on the
  VPS" claim was wrong.

---

## Your checklist

- [ ] **P1** `make migrate` footgun ⚠️ *data loss — first*
- [ ] **P2** `chat_messages` partition *(live outage)*
- [ ] **P3** Deploy creates topics
- [ ] **P4** Dockerfiles **+ CI** for intelligence & realtime ▶ *start now, blocks others*
- [ ] **P5** Compose: all workers, Qdrant, LiveKit ▶ *blocks others*
- [ ] **P6** Ingress — ROUTE_MAP + Caddy WS route
- [ ] **P7** Resource limits, backups, log rotation ▶ *before P5*
- [ ] **P8** Healthchecks + authenticated smoke test
- [ ] **P9** Startup env validation + missing env files
- [ ] **P10** Redis port drift *(opposite direction per repo)*
- [ ] **P11** UI env vars as build args
- [ ] **P12** Gates that cannot fail
- [ ] **P13** transcription DB role ⚠️ *with P4/P5*
- [ ] **P14** `FakeProvider` provenance ⚠️ *with P4/P5*
- [ ] **P15** Consumer liveness in `/healthz`
- [ ] **P16** Defuse notification consumer ⚠️ *before P5*
- [ ] **P17** Remove `/stripe/webhook` from the public route map
