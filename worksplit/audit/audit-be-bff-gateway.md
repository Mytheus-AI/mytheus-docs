# Audit: BFF / api_gateway / relay (edge layer)

Read-only audit. No repo file modified. All claims cite source; `.md` status claims were not
trusted — where a doc and the code disagreed, the code is reported and the doc flagged.

Root: `/Users/sushan/Projects/Mytheus`

---

## BFF ROUTE INVENTORY

**BFF is not a proxy.** It makes zero HTTP calls downstream — no `httpx` import, no client, no
service URLs anywhere in `services/bff/`. Every route reads Postgres directly through the shared
`tenant_session` dependency. The "aggregation" is SQL-level, not service-level.

| Route | Downstream | Real / stub | Evidence |
|---|---|---|---|
| `GET /bff/dashboard` | direct SQL on `meetings` (NOT core-api over HTTP) | real for 1 of 4 arms | `mytheus-backend/services/bff/dashboard.py:152`; raw `SELECT` at `:165-171` |
| ↳ arm `upcoming_meetings` | direct SQL | **real** | `services/bff/dashboard.py:160-192` |
| ↳ arm `action_items` | none | **hardcoded `[]`** | `services/bff/dashboard.py:194-201`, `:206` |
| ↳ arm `worklogs` | none | **hardcoded `[]`** | `services/bff/dashboard.py:194-201`, `:207` |
| ↳ arm `decisions` | none | **hardcoded `[]`** | `services/bff/dashboard.py:194-201`, `:208` |
| `GET /bff/org/chart` | 3 direct SELECTs: `departments`, `teams`, `team_members ⋈ users` | **real** | `services/bff/org_chart.py:169`; queries `:64-74`; tree build `:117-166` |
| `GET /bff/people/{user_id}` | **in-process Python import** of identity-tenant-service + 1 direct SELECT | **real, but not a network call** | `services/bff/people.py:43` (import), `:76` (call), `:48`/`:80` (email SELECT) |

Registration: `services/bff/main.py:11-14` — `create_app("bff")` plus the three routers. All three
routes are GET; the module docstring (`main.py:1-6`) states BFF never writes, and the code holds
to that.

Degrade behaviour is genuinely implemented, not claimed:
- `dashboard.py:127-149` — `_arm()` wraps each loader in `asyncio.wait_for` (2.0s,
  `ARM_TIMEOUT_SECONDS` at `:50`) and catches broadly, returning `status="degraded", items=[]`.
- `dashboard.py:204-209` — arms run concurrently via `asyncio.gather`.
- `org_chart.py:175-182` — same pattern, whole-tree granularity.
- `people.py` deliberately has no degrade wrapper (`people.py:22-29` explains why); exceptions
  propagate to the shared RFC 7807 handler, `None` → 404 via `not_found()` (`people.py:77-78`).

### The one structural defect in BFF

`services/bff/people.py:43`:

```python
from services.identity_tenant_service.context.service import get_user_context
```

This is an in-process import, executed inside the **bff container** against **bff's own DB
session** (`people.py:76` passes bff's `session`). The docstring (`people.py:9-14`) justifies it
with "one deployable, every service a Python module" — but that is no longer true: the deploy
splits them into separate containers (`mytheus-platform/deploy/compose.prod.yml:269-273` for
`bff`, `:277-286` for `identity-tenant-service`). So the stated precondition has already expired.

Consequence today: two independent processes execute the same `get_user_context` code against the
same database, and both write the same Redis cache key `nexus:{org_id}:user_ctx:{user_id}`. The
`identity-tenant-service` container is not consulted at all by this route.

---

## UI→BFF MATRIX

Method: grepped every `api(` / `apiSafe(` call, every `fetch(` and every `"use server"` file under
`mytheus-ui/src`, plus `mytheus-mobile/src`. The entire UI makes **6 outbound backend calls**
across **40 pages**.

### mytheus-ui

| # | Caller (file:line) | Method + path | Gateway route entry | Owning service | Route exists? | Downstream real? |
|---|---|---|---|---|---|---|
| 1 | `mytheus-ui/src/app/(app)/dashboard/page.tsx:36` | `GET /bff/dashboard` | `/bff` → `bff` (`edge.py:147`) | bff | ✓ `services/bff/dashboard.py:152` | **partial** — meetings arm real; `action_items`/`worklogs`/`decisions` hardcoded `[]` (`dashboard.py:194-201`) |
| 2 | `mytheus-ui/src/lib/api/transcript.ts:43` | `GET /api/v1/meetings/{id}/transcript` | `/api/v1/meetings` → `core-api` (`edge.py:148`) | core-api | ✓ `services/core_api/routes/transcript.py:95` | **real** |
| 3 | `mytheus-ui/src/lib/api/transcript.ts:55` | `GET /api/v1/meetings/{id}/speakers` | `/api/v1/meetings` → `core-api` | core-api | ✓ `services/core_api/routes/speakers.py:47` | **real** |
| 4 | `mytheus-ui/src/app/(app)/meetings/[meetingId]/transcript/actions.ts:33` | `PATCH /api/v1/meetings/{id}/speakers/{sid}` | `/api/v1/meetings` → `core-api` | core-api | ✓ `services/core_api/routes/speakers.py:80` | **real** |
| 5 | `mytheus-ui/src/app/api/livekit-token/route.ts:50` | `POST /api/meetings/{id}/token` | `/api/meetings` → `meeting-service` (`edge.py:161`) | meeting-service | ✓ `services/meeting_service/routes.py:153` | **real** (falls back to `FakeTransport` if LIVEKIT_* unset — `compose.prod.yml:290-292`) |
| 6 | `mytheus-ui/src/app/api/meeting-transcript/route.ts:32` | wraps `fetchTranscript` → same as #2 | via #2 | core-api | ✓ | **real** |

Internal (browser → Next route handler, not backend): `useLiveKitRoom.ts:53` → `/api/livekit-token`
(local, row #5); `useMeetingSocket.ts:83` → `/api/meeting-transcript` (local, row #6). Both are
Next route handlers, correctly used because `lib/api/client.ts` is server-only by construction
(`client.ts:11-16`).

Non-HTTP-backend: `resolveOrgContext()` (`mytheus-ui/src/lib/api/org-context.ts:24`) reads Clerk
only, no backend call. `org.plan` is hardcoded `"starter"` and `user.role` hardcoded `"member"`
(`org-context.ts:40,46`), both documented as safe-default placeholders (`org-context.ts:8-15`).

### mytheus-mobile

| # | Caller (file:line) | Method + path | Gateway entry | Route exists? | Real? |
|---|---|---|---|---|---|
| 7 | `mytheus-mobile/src/api/hooks/index.ts:19` | `GET /bff/dashboard` | `/bff` → bff | ✓ | partial (as row 1) |
| 8 | `mytheus-mobile/src/api/hooks/index.ts:30` | `GET /api/v1/meetings?status&limit` | `/api/v1/meetings` → core-api | ✓ `services/core_api/routes/meetings.py:243` | **real** |
| 9 | `mytheus-mobile/src/api/hooks/index.ts:45` | `GET /api/v1/teams` | `/api/v1/teams` → core-api (`edge.py:149`) | ✓ `services/core_api/routes/org_hierarchy.py:37` | **real** |
| 10 | `mytheus-mobile/src/api/hooks/index.ts:53` | `GET /api/v1/departments` | `/api/v1/departments` → core-api (`edge.py:150`) | ✓ `services/core_api/routes/org_hierarchy.py:91` | **real** |

**Result: 10 of 10 client calls land on a real, deployed route.** There are no broken links. The
problem is the inverse — see below.

---

## ORPHAN UI CALLS (no backend)

**Zero HTTP orphans.** Every path any client constructs resolves through the gateway's `ROUTE_MAP`
to a service that implements it. Verified prefix-by-prefix against `edge.py:146-180` and the route
decorators in `services/*/routes/`.

Three genuine gaps exist, none of them a mis-pointed HTTP call:

1. **Realtime socket has no server deployed.** `mytheus-ui/src/lib/realtime/useMeetingSocket.ts:75`
   opens a Socket.IO connection to `${NEXT_PUBLIC_REALTIME_WS_URL}/v1`. Grepping
   `mytheus-platform/deploy/` for `realtime` returns **zero hits** — it is absent from both
   `compose.prod.yml` and `deploy/caddy/mytheus.caddy`. The `mytheus-realtime` repo exists but is
   not deployed. The code is honest about it: `useMeetingSocket.ts:13` says the var is "unset in
   every environment that exists today", and `:40` defines a `"disabled"` state that falls back to
   mock captions. Not a bug; an undeployed component.

2. **LiveKit SFU likewise unconfigured.** `useLiveKitRoom.ts:69` reads `NEXT_PUBLIC_LIVEKIT_URL`;
   `:32` defines the `"unconfigured"` fallback to the mock grid. Server side,
   `compose.prod.yml:290-292` warns that without `LIVEKIT_*`, `select_transport()` returns
   `FakeTransport` and mints "obviously-fake tokens". So row #5 above returns 200 with a fake token
   in the current environment.

3. **The real integration gap: the UI mostly does not call the backend at all.** 24 of 40 pages
   import `lib/mock` (`grep -rln "lib/mock" src | wc -l` → 24; `find src/app -name page.tsx | wc -l`
   → 40). `mytheus-ui/src/lib/mock/data.ts:2-4` states this directly: mock data stands in for
   "`bff`/`api-gateway` responses until the backend exists". Even the one wired screen mixes
   sources — on the dashboard, `StatGrid.tsx:14` and `AgentActivityStrip.tsx:4` are local
   constants sitting beside the real meetings arm, and `dashboard/page.tsx:29-31` says the other
   three panels still read the mock deliberately, because the server arms are empty.

---

## DEAD BFF SURFACE (nothing calls it)

Grepped `/bff/org/chart` and `/bff/people` across `*.ts`, `*.tsx`, `*.py`, `*.dart`, `*.swift`,
`*.kt` in the whole monorepo, excluding backend tests and generated clients.

| Route | Callers | LOC |
|---|---|---|
| `GET /bff/org/chart` | **none** | `org_chart.py`, 189 |
| `GET /bff/people/{user_id}` | **none** | `people.py`, 88 |
| `GET /bff/dashboard` | ui + mobile | `dashboard.py`, 216 |

277 of BFF's 507 LOC (~55%) is unreached surface. Both dead routes are fully built — `org_chart.py`
carries a proven-constant query count and two-org tenant-isolation tests (`org_chart.py:8-11`,
`:22-24`) — they simply have no consumer.

Additionally, `mytheus-ui/src/lib/api/generated/bff.ts` is generated but only two of its schemas
are referenced (`types.ts:34,52,61` → `MeetingsArm`, `DashboardMeeting`, `Dashboard`); the
`OrgChart`/`PersonProfile`/`DepartmentNode`/`TeamNode`/`Member` types are unused.

---

## API_GATEWAY VERDICT — **live code, in the request path**

Not dead code. Four independent confirmations:

1. **Deployed.** `mytheus-platform/deploy/compose.prod.yml:251-268` — own container, own uvicorn
   command (`services.api_gateway.main:app`), healthcheck, `expose: 8000`, `depends_on: web
   (service_healthy)`.
2. **In the ingress path.** `mytheus-platform/deploy/caddy/mytheus.caddy:47-55`:
   `api.mytheus.cloud { reverse_proxy api-gateway:8000 }`, with an explicit comment at `:49-52`
   rejecting the direct-to-core-api alternative ("pointing this at a service bypasses that
   checkpoint entirely").
3. **Forwarding is real, not a stub.** `services/api_gateway/proxy.py:100-137` — builds an `httpx`
   request, streams the body (`content=request.stream()`, `:114`), strips hop-by-hop headers per
   RFC 9110 (`_HOP_BY_HOP`, `:46-62`), maps `TimeoutException`→504 and `HTTPError`→503 distinctly
   (`:119-126`), and releases the upstream connection via `BackgroundTask(...aclose)` (`:136`).
   Pooled client created once per process (`:87-97`), closed on shutdown (`:155-159`). Catch-all
   mounted last so `/healthz` and docs win (`:140-153`, called from `main.py:24`).
4. **Every route target resolves.** Checked all 25 `ROUTE_MAP` entries (`edge.py:146-180`) against
   compose service keys: `bff`, `core-api` (via the network alias on `web`,
   `compose.prod.yml:183-185`), `meeting-service`, `identity-tenant-service`,
   `billing-metering-service`, `notification-service`, `admin-ops-service`, `integration-service`
   — all present. `route_for()` is longest-prefix (`edge.py:183-188`), which is what keeps
   `/api/v1/meetings` (core-api) apart from `/api/meetings` (meeting-service) — a collision the
   code comments call out at `edge.py:156-160`.

### Gateway caveats (all real)

1. **Stale docs contradict the live config.** Three of four places describing ingress say Caddy
   proxies `api.mytheus.cloud` → `web:8000`, i.e. **bypassing the gateway**:
   `mytheus-platform/deploy/README.md:16`, `README.md:62`, `compose.prod.yml:406`. Only
   `deploy/caddy/mytheus.caddy:53` says `api-gateway:8000`. `compose.prod.yml:181` adds a fourth
   variant ("Caddy's existing `web:8000` upstream keeps working during the cutover"), implying the
   cutover may be mid-flight.
2. **UNVERIFIED: whether `mytheus.caddy` is actually installed on the box.** Its header
   (`mytheus.caddy:11`) says it is deployed by `mytheus-deploy.sh`. That script does not exist in
   this repo, and `mytheus-platform/deploy/deploy.sh` never touches Caddy (grep for
   `caddy`/`conf.d`/`validate` across all `.sh` and `.md` in the repo: zero hits). So the file is
   correct as written but has no deployer here. Combined with caveat 1, **whether the gateway is
   genuinely in the live request path cannot be resolved from this repo alone** — it needs a check
   on the host (`docker exec <caddy> cat /etc/caddy/conf.d/mytheus.caddy`, or a request to
   `api.mytheus.cloud` with no bearer token: the gateway returns 401 `AUTH_REQUIRED` before
   proxying, core-api directly would behave differently).
3. **Deploy gate skips the gateway.** `deploy/deploy.sh:58-59` waits on `mytheus-web-1` and
   `mytheus-meeting-service-1` only. `api-gateway` — the single ingress — is not gated, nor is
   `bff`. And `bff` declares **no healthcheck at all** (`compose.prod.yml:269-273`), so nothing
   anywhere notices if it fails to come up. This is precisely the failure mode
   `compose.prod.yml:299-306` documents for meeting-service ("built green, deployed green and went
   silently dark") — fixed for that one service, still open for `bff` and `api-gateway`.
4. **Per-tenant rate limiting is a stub**, acknowledged at `services/api_gateway/main.py:9-11`. No
   rate-limit code exists in the service. Only the Cloudflare global limit is in front.
5. **~60% of the route map is aspirational.** Prefixes routing to services that do not implement
   them:
   - `/api/v1/worklogs`, `/api/v1/decisions`, `/api/v1/actions`, `/api/v1/suggestions`
     (`edge.py:151-154,159`) → `core-api`, whose `main.py:16-20` includes only meetings,
     org_hierarchy, chat, speakers, transcript. Grepping `services/` for `"/worklogs`, `"/decisions`,
     `"/actions`, `"/suggestions` returns **zero** route definitions.
   - `/orgs`, `/rbac` (`edge.py:163-164`) → identity-tenant-service, whose `main.py:26` reads
     `# TODO: add routers for /orgs /rbac`.
   - `/billing`, `/usage`, `/plans`, `/notifications`, `/admin/tenant`, `/admin/ops`, `/oauth`,
     `/webhooks` → the four scaffold services, which `compose.prod.yml:318-321` admits "expose no
     routes beyond /healthz". That comment argues this is deliberate (a 404 from the owning
     service is more truthful than a 404 from the gateway), which is a defensible call — but it
     means the deployed public contract is far smaller than the route map suggests.
6. **`/scim/v2` routing works only by accident of mounting.** `edge.py:165` maps `/scim/v2` →
   identity-tenant-service; the SCIM router itself defines bare `/Users`, `/Groups`
   (`services/identity_tenant_service/routes/scim.py:31,52`) and gets its prefix from
   `main.py:20`. Correct today, but the prefix lives in a different file from the routes and
   nothing ties it to the gateway's entry.
7. `@app.on_event("shutdown")` (`proxy.py:155`) is deprecated in current FastAPI in favour of
   lifespan handlers. Works; will warn.

---

## RELAY VERDICT — **live, wired, and correctly isolated**

`services/relay/main.py`, 121 LOC. Not an HTTP service — a standalone worker process draining the
transactional outbox to Redpanda.

- **Deployed and wired:** `compose.prod.yml:210-222` — `command: ["python", "-m",
  "services.relay.main"]`, matching `relay/main.py:120-121`. `depends_on: redpanda
  (service_healthy)`.
- **Real producer, not a stub:** `ConfluentRelayProducer` (`relay/main.py:45-77`) wraps
  `confluent_kafka.Producer` with `acks=all` and `enable.idempotence=True` (`:50`), and serializes
  Confluent-wire-format Avro against Schema Registry with `auto.register.schemas: False` /
  `use.latest.version: True` (`:58-64`). Drain logic is delegated to `libs.shared.relay.relay_once`
  (`:34`, called `:105`).
- **Credential separation is real and enforced by deploy:** `relay/main.py:84` reads
  `RELAY_DATABASE_URL` (role `mytheus_relay`, whose single `outbox_relay_drain` policy on `outbox`
  is what lets it read cross-tenant *without* BYPASSRLS — `relay/main.py:10-14`).
  `compose.prod.yml:212-216` layers `prod.db.relay.env` on top of `prod.env`, so relay never
  receives the app/migrate credentials. It uses the `*backend` anchor, not `*backend-app`
  (`compose.prod.yml:211`) — deliberate and correct.
- **Fallback is loud, not silent:** `relay/main.py:85-90` warns explicitly that falling back to
  `DATABASE_URL` means draining nothing forever (names defect D4).
- **Crash-loop defect fixed:** `relay/main.py:97-112` wraps the drain in try/except with capped
  exponential backoff (5s → 60s, `:41-42`, `:109`). The comment at `:98-101` names this as defect
  D5 and describes the prior behaviour (any exception killed the process, supervisor restarted
  onto the same batch).
- Known gap, self-declared: no queue-depth metric yet (`relay/main.py:114-116`, deferred to 3.11).

No issues found in relay beyond that acknowledged gap.

---

## AUTH / TENANCY PROPAGATION

Tenancy is **not broken**. But the gateway's signing control is **inert**.

### What actually happens

1. UI never sends `org_id` — there is deliberately no parameter for one
   (`mytheus-ui/src/lib/api/client.ts:6-9`). It sends only the Clerk bearer token
   (`client.ts:131-135`). Server-only by construction (`client.ts:11-16`), so no token-fetching
   path ships to the browser bundle.
2. Gateway strips any client-supplied `X-Mytheus-Org` / `-User` / `-Signature`
   (`edge.py:101`, `_strip_internal_headers` `:128-131`) — before anything else, so a forged
   header cannot survive the edge.
3. Gateway verifies the JWT, extracts the Clerk org claim and `sub`, resolves `org_id` against
   `organizations` (`edge.py:104-111`), and fails closed on any `ProblemDetail` without touching a
   tenant-scoped store (`edge.py:112-114`).
4. Gateway attaches its own signed header set (`edge.py:116`, `_attach_internal_headers`
   `:134-140`). HMAC is over the concatenated `org:user` pair so the two cannot be mixed and
   matched (`edge.py:69-76`); comparison is constant-time (`edge.py:79-81`); a missing
   `GATEWAY_SIGNING_KEY` raises 503 rather than defaulting (`edge.py:58-66`). This design is sound.
5. Proxy forwards them intact — they live in `scope["headers"]`, which `edge_auth` rewrote, and
   `proxy.py:83-84`/`:113` forwards that list. `Authorization` also survives: it is not in
   `_HOP_BY_HOP` (`proxy.py:46-62`).
6. Each service independently re-verifies: `verify_token` → `resolve_org_id` → `set_rls`
   (`libs/shared/authz.py:37`, `:94-108`, `:125`). BFF's routes depend on
   `require_tier("member")` + `tenant_session` (`dashboard.py:154-155`, `org_chart.py:171-172`,
   `people.py:70-71`).

### Finding 1 — the signed headers have no verifier

Grepped the entire backend for `x-mytheus-org` / `X-Mytheus-Org` / `SIGNATURE_HEADER` /
`verify_context`, excluding `services/api_gateway/`. Hits are in **`tests/test_gateway_edge.py` and
`tests/test_gateway_proxy.py` only** — zero production consumers.

So the gateway signs three headers on every request and nothing downstream reads them. Tenancy
survives because it rides the forwarded JWT instead, which each service re-verifies — exactly what
`edge.py:5-7` says it intends ("It does NOT authorize: the service tier re-verifies the token
itself"). Two consequences:

- The HMAC apparatus is currently write-only: a control with no verifier, and therefore no test of
  the property it exists to guarantee in production. The day a service starts reading
  `X-Mytheus-Org` for convenience, it will be trusted-but-unchecked unless it also calls
  `verify_context` — and `verify_context` lives in the gateway package, so a downstream service
  would have to import from the gateway to use it.
- `org_id` is resolved **twice per request** — once at `edge.py:109-111`, again in each service's
  `tenant_context` — each opening its own DB session. That is a per-request round trip the signed
  header was presumably meant to save.

### Finding 2 — BFF's isolation is RLS-only, with no defence in depth

Every BFF query is raw `text()` SQL with **no `org_id` predicate at all**:
`dashboard.py:165-171`, `org_chart.py:64-74`, `people.py:48`. Correct **if and only if** RLS is
bound (`tenant_session` sets `app.current_org` transaction-locally) *and* the DB role is not
privileged. That is exactly defect D1's failure mode.

Configuration is correct today: `bff` uses the `*backend-app` anchor
(`compose.prod.yml:270`) → `prod.db.app.env` → `mytheus_app`, subject to every `tenant_isolation`
policy. But the margin is zero: one wrong env file turns all three BFF routes into cross-tenant
reads, with no second guard in the SQL and no assertion in the route. `org_chart.py:12-24` argues
at length that the walk cannot cross tenants "even in principle" — true, but the argument rests
entirely on RLS being active.

### Finding 3 — `_arm`'s broad catch can mask an authz error as a degraded panel

`dashboard.py:146-148` catches `Exception` for every arm. That is right for timeouts and
downstream 500s (and the docstring defends it), but it also swallows anything RLS- or
permission-related raised inside a loader, surfacing it to the client as
`status: "degraded"` — indistinguishable from a slow query. Low severity today (the only real arm
is a plain SELECT), worth noting before the three empty arms get real loaders.

---

## SPEC DRIFT — **none. Verified by execution, not inspection.**

Rather than eyeballing the YAML, I ran the generator's check mode:

```
$ cd /Users/sushan/Projects/Mytheus/mytheus-backend
$ .venv/bin/python -m scripts.emit_openapi --check
skipped (no routes of their own): admin-ops-service, api-gateway,
  billing-metering-service, integration-service, notification-service
OpenAPI specs match the code.
```

(Note: the system `python3.14` lacks fastapi; the repo `.venv` has it. Running with the wrong
interpreter fails with `ModuleNotFoundError` **and still exits 0** — a CI footgun worth knowing
about, since `--check` is meant to be a gate.)

`mytheus-backend/design/openapi/bff.yaml` (622 lines) describes exactly the three implemented
routes — `/bff/dashboard` (`:496`, operationId `dashboard` `:499`), `/bff/org/chart` (`:536`,
`:539`), `/bff/people/{user_id}` (`:560`, `:567`) — plus `/healthz` (`:601`). Schemas match the
Pydantic models one-for-one: `Dashboard` `:9`, `DashboardMeeting` `:26`, `MeetingsArm` `:190`,
`UnbuiltArm` `:411`, `OrgChart` `:242`, `DepartmentNode` `:79`, `TeamNode` `:367`, `Member` `:209`,
`PersonProfile` `:267`.

The spec is emitted from the live FastAPI app (`scripts/emit_openapi.py:1-32`), services are
auto-discovered rather than listed (`:21-25`), and the UI's types are **aliases onto the generated
output**, not hand-written: `mytheus-ui/src/lib/api/types.ts:25-26` imports
`./generated/bff` and `./generated/core-api`; `types.ts:14-22` documents the hand-written union
that had already drifted (missing `CONSENT_REQUIRED` and `NOT_A_PARTICIPANT`) before this
replaced it. `client.ts:35,38` likewise alias `ErrorCode` and `Problem` from the spec.

**This is the healthiest part of the edge layer** — spec, generated client, and code are provably
in agreement, with a CI gate behind it.

One honest gap the generator documents itself (`emit_openapi.py:26-32`): **api-gateway has no
spec**, because its routes live in `edge.ROUTE_MAP` rather than in decorators, so FastAPI sees only
`/healthz`. The actual public contract — the union of the per-service documents, filtered by the
route map — is written down nowhere and machine-checked nowhere. The route map's correctness is
partly covered by `tests/test_gateway_edge.py:276-286` (prefix resolution and nesting), but nothing
checks that a mapped prefix corresponds to a route that exists in the target service — which is why
caveat 5 above (nine dead prefixes) went unnoticed.

---

## VERDICT

The edge layer's mechanics are genuinely well-built — the gateway proxy streams bodies correctly
and maps upstream failures precisely, the relay is properly credential-isolated and crash-resistant,
the OpenAPI pipeline provably cannot drift, and tenancy propagates safely via re-verified JWTs — so
the integration surface is not broken in the way the brief anticipated: all 10 client calls land on
real, deployed routes and there are no orphan HTTP calls. The actual problem is that the surface is
nearly unused in both directions: 24 of 40 UI pages render mocks, two of BFF's three routes have
zero callers anywhere in the monorepo, three of `/bff/dashboard`'s four arms return a hardcoded
`[]`, and roughly 60% of the gateway's route map points at prefixes no service implements. The four
findings that warrant action are (1) `bff/people.py:43` imports identity-tenant-service in-process
while compose runs them as separate containers sharing one database and one Redis cache key,
(2) the gateway's signed `X-Mytheus-*` headers have zero production verifiers — only tests read
them, (3) `bff` and `api-gateway` have no deploy health gate and `bff` has no healthcheck at all,
reproducing the silent-dark failure mode already fixed for meeting-service, and (4) the ingress
docs contradict the Caddy config three ways, and no script in this repo deploys that Caddy file, so
whether the gateway is truly in the live path is UNVERIFIED from source alone.
