# Samar — Platform, Deploy, Data, Event Backbone

**10 tasks.** Three of them you can start immediately; the rest wait on Sushan.

Audit report: https://claude.ai/code/artifact/da281c5f-d990-40b4-9487-d9f857833b08

---

## ⏱️ Ordering

**Start now, no dependency:** M4, M5, M6 (Dockerfiles, compose entries, resource limits). These do
not touch the request path, so Sushan's blockers do not affect them — and M4/M5 are the largest
single piece of work in the whole plan, so starting early is the right call.

**Wait for Sushan's S1 + S2** before verifying anything that involves making an authenticated
request. He will announce when they are deployed.

**You block others on:** M4 + M5. Nothing in `mytheus-intelligence` or `mytheus-realtime` runs
anywhere today — no Dockerfile, no compose entry. Until those land, nobody can test transcript,
embedding or search behaviour end to end, no matter how correct the code is.

---

## M1 — `chat_messages` inserts have been failing since 1 September

**Repo:** `mytheus-backend` — **live production defect, 6 days old**

`migrations/versions/5ed978467cad:53-55` hardcodes:

```python
_PARTITION_YEAR = 2026
_PARTITION_MONTH = 8
```

`:87-90` creates exactly one partition, `chat_messages_2026_08`, covering `2026-08-01` →
`2026-09-01`. There is **no `DEFAULT` catch-all partition**. Today is past that range, so every
insert raises *"no partition of relation found"*.

The sibling `audit_logs` migration (`c3e8b1d47f92`) took both the wider buffer (partitions through
2027-06) *and* a DEFAULT. The comment at `5ed978467cad:18` acknowledges the difference. Chat got
neither.

The hardcoding itself is deliberate and correct — the comment says *"The partition set must not
depend on when the migration happens to run."* Keep that principle; the problem is that nothing
creates later partitions.

### Do

- [ ] New migration: add a `DEFAULT` partition to `chat_messages` so writes stop failing
      immediately.
- [ ] Add partitions covering a sensible forward window, matching how `audit_logs` did it.
- [ ] Add a rolling-partition job so this cannot recur. Either `pg_partman` or a scheduled task —
      note the audit found **no `pg_partman`** anywhere, so `audit_logs` will hit the same wall in
      June 2027. Solve both tables with one mechanism.
- [ ] Check whether any rows were lost rather than rejected (rejected is more likely — the insert
      raises rather than silently dropping).

---

## M2 — A fresh deploy fails by construction

**Repo:** `mytheus-platform`

`deploy/deploy.sh:87` **verifies** that 42 topics and 21 subjects exist. Nothing on the deploy path
**creates** them, and `mytheus-events` is not cloned on the VPS.

`mytheus-events/scripts/create_topics.sh` is good — idempotent, matches the declared contract list
exactly (21 topics + 21 `.dlq`), was recently made idempotent with a provision entry point. It is
simply never invoked by the deploy.

### Do

- [ ] Make topic + subject provisioning part of the deploy path, before the verify gate.
- [ ] Decide how `mytheus-events` reaches the VPS: vendor the script, ship it in the backend image,
      or clone the repo. Pick one and document it.
- [ ] Keep the verify gate — it is the right check, it just needs something to verify.

---

## M3 — Make a green deploy mean something

**Repo:** `mytheus-platform`

Only **7 of 16** long-running services in `compose.prod.yml` have a healthcheck. The other 9 are
invisible to both `deploy.sh`'s gate and compose's `depends_on: service_healthy`. `api-gateway` and
`ui` do have healthchecks but are not gated by `deploy.sh`.

Worse: `/healthz` is on the gateway's **public** path list, so it bypasses auth entirely. That is
exactly why Sushan's S1 outage reports green — the health signal cannot detect it.

### Do

- [ ] Add healthchecks to the 9 services without one.
- [ ] Gate `deploy.sh` on `api-gateway` and `ui` health too.
- [ ] **Add one authenticated smoke request to the post-deploy gate.** A public `/healthz` 200 is
      not evidence the system works — this audit exists partly because it looked like it was.
- [ ] Note `transcription-service`'s `/healthz` stays green even if its consumer dies (disclosed at
      `main.py:15-19`) — make the healthcheck reflect consumer liveness.
- [ ] `temporal` has no healthcheck and `workflow` gates on `service_started` only.

---

## M4 — Two entire repos have no way to run ▶ START NOW

**Repos:** `mytheus-intelligence`, `mytheus-realtime`

This is the biggest structural gap in the audit, confirmed independently by three auditors.

**Neither repo contains a Dockerfile. Neither appears in `compose.prod.yml`.** That is 9 of the 21
services, including everything genuinely well-built in the AI tier:

- `mytheus-intelligence` — embedding, ingestion, search, reranker (+ 3 stubs). Real BGE models,
  real unstructured.io parsing, uuid5-idempotent Qdrant writes, transactional outbox, per-tenant
  vector isolation. None of it runs.
- `mytheus-realtime` — transcription-service (Python) and realtime-gateway (TypeScript, 639-line
  `server.ts`, the most complete single component the audit found). Neither runs.

Meanwhile `meeting-service` ships to production emitting `meeting-room-events` into Redpanda with
no consumer running. Prod is a relay producing into a broker nothing reads.

### Do

- [ ] Dockerfile for `mytheus-intelligence` (multi-service — one image with per-service entrypoints
      matches how `mytheus-backend` already does it).
- [ ] Dockerfile for `mytheus-realtime` — note two runtimes, Python and Node.
- [ ] CI image build + push for both, mirroring the three repos that already have real CD.
- [ ] Mind the model weights: BGE-small and bge-reranker-base need to be baked in or cached, not
      downloaded on every cold start.

---

## M5 — Compose entries for everything that should be running ▶ START NOW

**Repo:** `mytheus-platform`

### Do

- [ ] **Consumer workers** — each is a standalone process today and none are in compose:
      `embedding_service/worker.py`, `search_service/worker.py`, `ingestion_service/worker.py`,
      the intelligence outbox relay (`relay_main.py`), transcription-service, realtime-gateway.
- [ ] **Qdrant** — required by the embedding path, absent from compose.
- [ ] **LiveKit** — absent; `NEXT_PUBLIC_LIVEKIT_URL` unset is why the UI degrades to
      `status: "unconfigured"`.
- [ ] **MinIO or R2 credentials** — ingestion does a real object-store put; R2 (ADR-0023) is
      currently ADR text plus an unimported dependency.
- [ ] Wire `depends_on` correctly against the healthchecks from M3.

### Watch out

`.env.example` in `mytheus-intelligence` says Redis port **6379**; every code default is **6380**.
Fix the mismatch rather than papering over it. Also `CLERK_JWT_INTERNAL_ORG_CLAIM` is unprovisioned
— every intelligence REST route is un-callable without it.

---

## M6 — Resource limits and observability ▶ START NOW

**Repo:** `mytheus-platform`

Grep found **no `deploy:`, `resources:`, `mem_limit`, `cpus` or `logging:` anywhere** in
`compose.prod.yml`. 17 containers share one VPS — with a second production product on the same box
— and M4/M5 are about to add nine more, including two that load ML models into memory.

Also absent: **backups of any volume**, and any observability beyond `docker logs`. The
`observability/` directory the README documents does not exist.

### Do

- [ ] Memory and CPU limits on every container. Do this **before** M5 lands, not after.
- [ ] Log rotation — an unbounded json-file driver on 26 containers will fill the disk.
- [ ] Backups for `pgdata`, `redisdata`, `redpandadata` and (once M5 lands) Qdrant storage.
- [ ] Minimum viable observability. Metrics and Sentry are declared dependencies that are never
      imported — either wire them or remove them from `pyproject.toml` so the dependency list stops
      implying coverage that does not exist.
- [ ] Sanity-check the capacity story: Postgres + Redis + Redpanda + Temporal + Qdrant + two model
      servers on one node, alongside another product.

---

## M7 — Org suspension silently does nothing

**Repo:** `mytheus-backend`

`libs/shared/org_resolver.py:32-34` states that a suspension only takes effect once a
`tenant-events` consumer invalidates the cache. **That consumer does not exist.**

`identity_tenant_service/provisioning.py:139` produces `tenant-events` correctly. The declared
consumers — billing-metering, admin-ops, workflow — are all empty scaffolds. So the topic is a dead
letter, and suspending a tenant leaves them fully operational.

This is a functional security-adjacent bug, not a missing feature.

### Do

- [ ] Write a `tenant-events` consumer that invalidates the org-resolver cache on suspension and
      deletion.
- [ ] Follow the existing consumer discipline: `libs/shared/consumer.py` already gives you Redis
      dedupe, durable inbox, DLQ and single-transaction apply. Do not hand-roll.
- [ ] Add it to compose (ties into M5).
- [ ] Test that a suspended org is actually denied after the cache TTL and after invalidation.

---

## M8 — Delete the `user-sync-events` topic

**Repo:** `mytheus-events`

ADR-0010 D4 decided there is **no per-user propagation topic**. The code agrees —
`identity_tenant_service/routes/auth.py:19-21` says so explicitly, and the `TOPIC` constant there
is used only as a Postgres/Redis dedupe namespace, never as a Kafka topic.

But the topic, its `-value` subject and its `.dlq` are all still created by `create_topics.sh` and
will always be empty.

### Do

- [ ] Remove `user-sync-events` from `codegen/_schemas.py` `TOPICS`, from `subjects/`, and from
      `create_topics.sh`.
- [ ] Rename the constant in `routes/auth.py` so it no longer looks like a topic name (it is a
      dedupe namespace — call it that).
- [ ] Update the expected count in `create_topics.sh` (currently prints "42 expected") and the
      `deploy.sh:87` gate from M2 to match.

---

## M9 — Adopt the explicit-grants mechanism before it is needed

**Repo:** `mytheus-backend`

`migrations/_grants.py` was written and merged as part of the ADR-0021 explicit-grants work. Its
`grant_tenant_table` helper is imported by **zero migrations**.

Consequence: the next table-creating migration from any repo ships with an empty ACL — exactly the
failure the ADR was written to prevent. The mechanism exists and was never adopted.

### Do

- [ ] Use `grant_tenant_table` in the migration template / contributor docs.
- [ ] Backfill it into the existing migrations where it applies, or add a test asserting every
      tenant table has the expected grants.
- [ ] Add a CI check: a new table without grants should fail the build.
- [ ] Related open item — `9b2c4d1e6f30:73-95` documents a cross-repo GRANT gap that is still
      unapplied in `mytheus-realtime`. It breaks the transcript endpoint on any clean-cluster
      rebuild. Close it while you are here.

---

## M10 — Environment and CI hygiene

**Repos:** `mytheus-platform`, `mytheus-ui`, all

### Do

- [ ] `prod.ui.env` is read at `compose.prod.yml:381` and **no example file exists anywhere**.
      Create one.
- [ ] `NEXT_PUBLIC_REALTIME_WS_URL` and `NEXT_PUBLIC_LIVEKIT_URL` are **not Docker build args**
      (`mytheus-ui/Dockerfile:29-35`), so containers bake them as `undefined` and no amount of
      runtime env fixes it. Next.js inlines `NEXT_PUBLIC_*` at build time. Coordinate with Shaun —
      his H9 depends on this working.
- [ ] CD concurrency is per-repo, so the three repos with real CD can race each other on a deploy.
      Add cross-repo serialisation or a deploy lock.
- [ ] The oasdiff OpenAPI breaking-change gate was fixed in-file but **has still never actually
      executed in Actions**. Confirm it runs and can fail a build.
- [ ] `deploy.sh` exists **only on the VPS** — production deploy behaviour is unverified from the
      repo. Get it into version control.

---

## M11 — transcription-service must not reach production as `postgres` ⚠️ *gates M4/M5*

**Repo:** `mytheus-realtime` + `mytheus-backend`

**Do this as part of M4/M5, not after.** It is a deployment precondition, not a follow-up.

`services/transcription-service/src/transcription_service/db.py:13-22` connects as the `postgres`
superuser. The service's RLS code is **correct** — it always issues `SET LOCAL app.current_org` —
but a superuser connection bypasses row-level security entirely, so those calls protect nothing.

### This is a documented deferral, not a hidden bug

The docstring is explicit and the reasoning is sound: `mytheus-backend`'s roles (`mytheus_app`,
`mytheus_relay`, `mytheus_admin`) are created with **no password** by design (`a1c4e7f20b18` —
*"cannot authenticate until an operator sets it"*), and granting a new cross-repo role SELECT on
tables this service does not own is an ops/bootstrap concern rather than something a migration in
this repo should arrogate to itself.

It is harmless today only because **the service is not deployed anywhere**. The moment M4/M5 give
it a compose entry, a superuser connection to the shared production Postgres becomes a live
cross-tenant hole in a service that writes `transcript_chunks` and the shared `outbox`.

### Do

- [ ] Create a least-privilege role for transcription-service mirroring `mytheus_app` /
      `mytheus_relay` — `NOSUPERUSER NOBYPASSRLS`, explicit grants only.
- [ ] Grant it exactly what it needs: write on `transcript_chunks` and `outbox`, read on
      `tenant_settings` and `data_residency`. Use `grant_tenant_table` from M9 — this is precisely
      the case that mechanism was built for.
- [ ] Point the production `DATABASE_URL` at that role; leave the local dev default alone.
- [ ] Add a startup assertion that the connected role does **not** have `rolbypassrls`. The audit
      noted `libs/shared/db.py:49` has no such runtime check either — worth adding in both places.
- [ ] Close the related cross-repo GRANT gap from M9 at the same time; they are the same
      conversation.

---

## Do NOT build these (deliberately deferred)

Per the roadmap phase boundary at W36. Their absence is the plan being followed — do not "fix"
them:

- **Kubernetes, Terraform, Helm.** Plan 0 states it explicitly: the MVP band ships a single-node
  VPS target only. The lack of `.tf` files is correct.
- **Wave 1–4 service extraction** — the monolith is supposed to still be a monolith at W36.
- **Multi-region, cell-based isolation, Citus sharding.**
- **BYOC / dedicated multi-AZ Redpanda, int8 Qdrant quantization, GPU autoscaling.**
- **The `nexus-deploy` GitOps repo.**

---

## Your checklist

- [ ] **M4** — Dockerfiles for intelligence + realtime ▶ *start now, blocks others*
- [ ] **M5** — Compose entries, Qdrant, LiveKit ▶ *start now, blocks others*
- [ ] **M11** — Least-privilege DB role for transcription-service ⚠️ *must ship with M4/M5*
- [ ] **M6** — Resource limits, backups, log rotation ▶ *start now, before M5 lands*
- [ ] *— wait for Sushan's S1 + S2 —*
- [ ] **M1** — `chat_messages` partition (live defect)
- [ ] **M2** — Deploy creates topics
- [ ] **M3** — Real healthchecks + authenticated smoke test
- [ ] **M7** — `tenant-events` consumer (org suspension)
- [ ] **M8** — Delete `user-sync-events`
- [ ] **M9** — Adopt `_grants.py`
- [ ] **M10** — Env and CI hygiene
