# Cross-cutting audit — identity, auth and multi-tenancy

Read-only source audit across all 8 repos. Root: `/Users/sushan/Projects/Mytheus`.
Every claim below is cited to `repo/path:LINE` read in this session. No status docs were trusted.
No secret values appear in this file.

---

## THE CHAIN — hop by hop

### Hop 1 — Browser sign-in — **WIRED**

Real Clerk components, not a bypass.

- `mytheus-ui/src/app/(auth)/sign-in/page.tsx:15` — `<SignIn routing="path" path="/sign-in" signUpUrl="/sign-up" fallbackRedirectUrl="/dashboard" />`. `sign-up/page.tsx` mirrors it.
- Route protection is an **allowlist, not a denylist**: `mytheus-ui/src/middleware.ts:22-49` enumerates public marketing routes; `:51-57` runs `await auth.protect()` on everything else. `(app)` and `(admin)` are absent from the allowlist on purpose, which is what protects them. A new route is protected by default.
- Matcher covers everything but Next internals and static assets (`middleware.ts:59-66`).
- Clerk instance is env-driven: `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY`, `NEXT_PUBLIC_CLERK_SIGN_IN_URL`, `NEXT_PUBLIC_CLERK_SIGN_UP_URL` (`mytheus-ui/.env.example:7-18`). `.env.local` exists and is populated.
- Mobile is equally real: `mytheus-mobile/app/_layout.tsx:65` `<ClerkProvider publishableKey={...} tokenCache={tokenCache}>`, with SecureStore-backed token cache at `mytheus-mobile/src/lib/secure-store.ts:52-57` (not AsyncStorage).

### Hop 2 — Session → token attached to backend calls — **WIRED**

- `mytheus-ui/src/lib/api/client.ts:131-135` — `const { getToken } = await auth(); const token = await getToken(); … headers.Authorization = \`Bearer ${token}\``.
- Server-only **by construction**: it imports `auth()` from `@clerk/nextjs/server` (`client.ts:21`), so a client component importing it fails the build rather than shipping a token-fetching path into the browser bundle.
- One origin only — `API_GATEWAY_URL`, with no default (`client.ts:68-77`): "a wrong default would send real session tokens to an unintended host."
- Tenant responses are `cache: "no-store"` by default (`client.ts:146-149`) because the Next data cache key does not include the tenant.
- The two same-origin route handlers hold **no credential of their own** and forward the caller's Clerk session through that same client: `app/api/livekit-token/route.ts:50-53`, `app/api/meeting-transcript/route.ts:32`.
- Mobile: `mytheus-mobile/src/api/client.ts:160-166`, token re-read per request via `useAuth().getToken()` (`src/api/provider.tsx:41-43`) rather than captured once.

### Hop 3 — Edge verification — **WIRED, genuinely**

`mytheus-backend/services/api_gateway/main.py:18-19` installs `edge_auth` as HTTP middleware; `services/api_gateway/edge.py:88-117` is the handler.

Which library, which JWKS:

- PyJWT. `mytheus-backend/libs/shared/auth.py:47-49` — `PyJWKClient(os.environ["CLERK_JWKS_URL"], cache_keys=True)`; issuer from `os.environ["CLERK_ISSUER"]` (`auth.py:52-53`).
- `auth.py:94-103` — `jwt.decode(token, signing_key, algorithms=["RS256"], issuer=_issuer(), options={"require": ["exp","iat","nbf","iss","sub"]})`. Explicit algorithm allowlist (never `none`, never HS*), `nbf` **required** rather than merely honored.
- Failure mapping is deliberate and fails closed on every branch: missing bearer → 401 (`auth.py:64-65`); JWKS unreachable → 503 `DEPENDENCY_UNAVAILABLE` (`:69-75`); malformed token during `kid` parse → 401 (`:76-91`); expired → 401 `TOKEN_EXPIRED` (`:104-105`); any other invalid → 401 without echoing `str(exc)` so no oracle for which check failed (`:106-109`); org claim absent/non-string → 401 (`:111-117`).

Grep results for bypasses across **all 8 repos** (`verify_signature`, `verify=False`, `options={"verify`, `DEV_MODE`, `SKIP_AUTH`, `ALLOW_ANON`, `AUTH_DISABLED`, `DISABLE_AUTH`, `NO_AUTH`, `BYPASS`, `MOCK_AUTH`, `FAKE_AUTH`, `insecure`): **zero** hits in any verification path. Every hit is prose, an RLS role attribute (`NOBYPASSRLS`), or a stale doc. `app.dependency_overrides` appears **nowhere** in application code (tests only).

CI gate: `PYTHONPATH=. python3 mytheus-backend/scripts/check_authz.py` → *"authz check passed: every FastAPI route declares auth or is explicitly public"* (run in this session).

### Hop 4 — org_id resolution — **WIRED, and correct**

Server-derived. There is **no** client-supplied org_id that is trusted anywhere on the HTTP path.

- `edge.py:106-111` — `clerk_org_id_from(claims)` (the opaque `org_…` string, `auth.py:125-132`, documented "OPAQUE. Do not parse it, do not cast it to UUID") → `resolve_org_id(session, clerk_org)`.
- `libs/shared/org_resolver.py:49-51, 70-84` — `SELECT id, status, deleted_at FROM organizations WHERE clerk_org_id = :clerk_org_id`, rejecting unknown / suspended / soft-deleted with one **byte-identical** denial (`org_resolver.py:58-67`) so the path is not an oracle for which Clerk orgs are customers.
- Inbound copies of the internal headers are **stripped first**: `edge.py:101` → `_strip_internal_headers` at `:128-131`, removing any client-sent `x-mytheus-org` / `x-mytheus-user` / `x-mytheus-signature`.
- The gateway then attaches its own set under an HMAC that covers org and user **together** (`edge.py:69-76`), so two valid contexts cannot be mixed into tenant A's org with tenant B's user; comparison is constant-time (`:79-81`).
- The browser never learns the internal `org_id`: `mytheus-ui/src/lib/api/org-context.ts:34-41` exposes Clerk's opaque id as a display key only, and `client.ts:6-9` has no parameter for a tenant by design.

### Hop 5 — Service-to-service — **MIXED** (see Critical Finding 1)

- **BFF → core-api: no HTTP hop exists.** `bff` reads the same Postgres directly under RLS (`services/bff/dashboard.py:154-155`, `org_chart.py:171-172`, `people.py:70-71` — all `require_tier("member")` + `tenant_session`). Nothing to propagate, nothing dropped.
- **realtime-gateway → core-api: identity propagated, correctly.** The user's own bearer token is forwarded (`mytheus-realtime/services/realtime-gateway/src/coreApiClient.ts:114-123` and `:150-159`), and core-api re-verifies it. An earlier `X-Org-Id` header was deliberately **removed** because it "looks like it carries tenancy while being silently discarded, inviting a future optimization that starts trusting it" (`coreApiClient.ts:45-53`).
- **Every REST service re-verifies the JWT itself** — the gateway is genuinely not a trust delegation (`libs/shared/authz.py:94-122`).
- **Transport authn between services: ABSENT.** No mTLS, no shared secret, no internal JWT per ADR-0013. Isolation is docker-network-only, and that network is shared with an unrelated product's stack on the same box (`mytheus-platform/deploy/compose.prod.yml:366-376`; `deploy/caddy/mytheus.caddy:33-36`).

### Hop 6 — AI tier (ADR-0019) — **WIRED in code, NOT REACHABLE**

- `mytheus-intelligence/libs/shared/auth.py:91-128` is a faithful, independent reimplementation of the backend verifier: `PyJWKClient(os.environ["CLERK_JWKS_URL"])` (`:75-80`), `jwt.decode(…, algorithms=["RS256"], issuer=_issuer(), options={"require": ["exp","iat","nbf","iss","sub"]})` (`:112-120`), 503 on JWKS outage (`:105-109`), non-echoing 401s (`:121-126`).
- `org_id_from` (`:131-140`) fails closed on absent / non-string / non-UUID — "never guessed, never defaulted to any tenant".
- Every REST route depends on `verified_org_id`: `/search` (`services/search_service/routes.py:91`), `/ingest` and `/ingest/jobs/{id}` (`services/ingestion_service/routes.py:80, 173`), `/embed` (`services/embedding_service/query_embed.py:66`), `/rerank` (`services/reranker_service/rerank.py:64`).
- The **old trusted-`X-Org-Id` hole is closed**, with regression tests that name the exploit: `tests/test_search_routes.py:118-134`, `tests/test_ingestion_routes.py:65-72`, `tests/test_query_embed.py:78-89`, `tests/test_reranker.py:137-144`.
- It reads a pre-resolved `internal_org_id` claim instead of joining `organizations`, to respect the bounded-context boundary (`libs/shared/auth.py:25-50`). The **write side now exists**: `mytheus-backend/libs/shared/clerk_admin.py:56` `set_internal_org_id`, called post-commit from `services/identity_tenant_service/provisioning.py:154-171`.
- Two gaps: the Clerk JWT-template mapping `public_metadata.internal_org_id` → claim is dashboard configuration no code can enforce (`clerk_admin.py:21-25`) — **UNVERIFIED**, and until it exists the AI tier 401s on every request; and **the gateway ROUTE_MAP has no entry for any intelligence path** (`services/api_gateway/edge.py:146-180`), so the AI tier has no ingress at all.
- `agent_runtime_service` is a 13-line healthz stub (`mytheus-intelligence/services/agent_runtime_service/main.py`). ADR-0013 agent-to-agent authorization is **unbuilt** — no grant, authz, permit or scope logic exists in that package.
- Event consumers do carry tenancy: `libs/shared/tenancy.py:38-42` transaction-local `set_config('app.current_org', …, true)`, org_id taken from the verified envelope header, DLQ on unparseable (`libs/shared/tenancy.py:8-14`).

### Hop 7 — Realtime tier + LiveKit — **WIRED in code, NOT DEPLOYED**

WebSocket:

- Handshake verifies the Clerk JWT **before** the socket is accepted: `mytheus-realtime/services/realtime-gateway/src/server.ts:217-231` → `libs/auth.ts:34-55` (`jose` `jwtVerify` against `createRemoteJWKSet(CLERK_JWKS_URL)`, issuer checked, `org_id` and `sub` required, throws on every failure). Rejection is `next(new Error("UNAUTHENTICATED"))`.
- Room join is **deny-by-default** against core-api's participant check (`server.ts:337-370`), forwarding the user's own token; any non-200, malformed body or network error resolves `{ allowed: false }` (`coreApiClient.ts:124-147`) so a flaky core-api fails a join **closed**.
- The Socket.io room is **org-scoped** (`libs/keys.ts:78-80` → `meeting:${orgId}:${meetingId}`), with `:49-77` recording the real cross-tenant leak that scoping fixed and why `meeting_id` uniqueness alone was judged insufficient.
- Chat requires a prior successful join for that exact meeting on that socket (`server.ts:420-432`).

LiveKit:

- Tokens are minted **server-side only**, never in the browser. `mytheus-backend/services/meeting_service/routes.py:152-215`: RLS-scoped meeting-existence check → 404 (`:171-173` via `_fetch_meeting_or_404` at `:107-118`), then participant-row check → 403 `NOT_A_PARTICIPANT` (`:180-183`), then consent check → 428 (`:185-186`).
- The grant is scoped to that one room: `services/meeting_service/transport.py:122` `api.VideoGrants(room=room, room_join=True, …)`, identity = `user_id` (`:125`).
- `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` never leave the backend (`mytheus-ui/src/app/api/livekit-token/route.ts:11-14`).
- **A signed-in user cannot join a room by ID** — they cannot obtain a token for a meeting they are not a participant of.
- LiveKit webhooks verify the provider signature (`services/meeting_service/webhook.py:_invalid_signature`, `transport.py:147` `api.TokenVerifier`), and a room-metadata `org_id` is re-validated through the sanctioned second reader `resolve_org_by_id` (`libs/shared/org_resolver.py:87-100`) rather than trusted.

### Hop 8 — Database — **WIRED (RLS-primary, not application-discipline)**

- `mytheus-backend/migrations/versions/82d44163c933_0001_…py:21-35` — for `outbox, inbox, users, teams, team_members, departments, tenant_settings, data_residency, meetings, meeting_participants`: `ENABLE` **and** `FORCE ROW LEVEL SECURITY`, policy `tenant_isolation USING (org_id = NULLIF(current_setting('app.current_org', true),'')::uuid) WITH CHECK (same)`. `NULLIF` makes an unset GUC yield NULL → zero rows, never an error.
- Later migrations extend the same shape to `rbac_grants` (`b7d9f3a55c21`), `audit_logs` (`c3e8b1d47f92`), `meeting_rooms` (`3d13f46b960f`), `meeting_speakers` (`25e7cede5672`), `chat_messages` (`5ed978467cad`), plus partition-level privilege closes so a partition named directly cannot bypass the parent's policy.
- `app.current_org` is set **transaction-locally** (`libs/shared/tenancy.py:32-37`) — the property that survives PgBouncer/Supabase pooling; the docstring names non-local GUC leakage as the failure it prevents.
- Request-path role holds neither `rolsuper` nor `rolbypassrls` (`mytheus-platform/env/prod.db.app.env.example`), enforced by CI (`mytheus-backend/tests/test_roles_and_policies.py`) and by the per-role env split closing defect D1 (`env/prod.env.example:13-29`).

**Ten sampled queries, all tenant-safe.** Note these mostly carry **no** hand-written `org_id` predicate — that is deliberate: RLS is the enforcement and the app is not permitted a second, forgettable copy.

| # | Query | Scoping |
|---|---|---|
| 1 | `core_api/routes/meetings.py:136` `SELECT … FROM meetings WHERE id = :id AND deleted_at IS NULL` | RLS via `tenant_session` (`:157-158`) |
| 2 | `core_api/routes/meetings.py:112` `SELECT … FROM meeting_participants WHERE meeting_id = ANY(:ids)` | RLS |
| 3 | `core_api/routes/meetings.py:369-371` participant check joined on `users.clerk_user_id` | RLS |
| 4 | `core_api/routes/transcript.py:138` `SELECT … FROM transcript_chunks WHERE …` | RLS (`:98-99`) |
| 5 | `core_api/routes/chat.py:78` `SELECT 1 FROM meeting_participants WHERE meeting_id = … AND user_id = …` | RLS (`:91-92`) |
| 6 | `core_api/routes/speakers.py:68` `SELECT … FROM meeting_speakers WHERE meeting_id = …` | RLS (`:50-51`) |
| 7 | `bff/org_chart.py:65,68,72` departments / teams / team_members+users | RLS (`:171-172`) |
| 8 | `bff/people.py:48` `SELECT email FROM users WHERE id = :uid` | RLS (`:70-71`) |
| 9 | `identity_tenant_service/context/service.py:40,44,56,64` user / teams / departments / meetings | RLS, set by caller |
| 10 | `meeting_service/webhook.py:157,214` `SELECT meeting_id FROM meeting_rooms WHERE org_id = :org AND livekit_room = :room FOR UPDATE` | **explicit** org predicate **and** RLS |

Deliberate non-RLS readers, both justified in-source: `libs/shared/org_resolver.py` (the tenant root, column-grant-limited, one query shape per lookup key) and `libs/shared/relay.py:39` (outbox drain as `mytheus_relay` under a dedicated `outbox_relay_drain` policy). **No unscoped query on tenant data exists in the request path.**

### Hop 9 — Vector store + object storage — **WIRED (strongest layer in the system)**

- Qdrant: per-tenant **physical collection** `{logical}__{org_id.hex}` (`mytheus-intelligence/libs/shared/qdrant.py:108-112`) **and** a mandatory `org_id` payload filter on every query (`:133-135`), both always. `search_with_tenant_filter` (`:155-165`) is the only sanctioned search path; nothing else may string-format `__{org_id}` inline (`:3-8`). `org_id` is rendered as `UUID.hex`, never a client string (`:11-13`).
- Object storage: every key begins `{org_id}/` (`libs/shared/object_storage.py:42-43, 79-94`), built only from verified identity plus server-generated UUIDs and a server-validated extension from a fixed allowlist. The caller-supplied filename **never** reaches the key (`:25-35`), closing path traversal and cross-tenant prefix collision structurally.

### Hop 10 — Admin / ops routes — **BROKEN (UI), ABSENT (backend)**

- `mytheus-ui/src/app/(admin)/layout.tsx:12-15`, verbatim: *"RBAC gate is UX courtesy only per CLAUDE.md; real enforcement is server-side. Auth isn't wired up yet, so there's no gate at all right now — anyone can reach /admin. Add the Clerk role check here once auth lands." / "// TODO(auth): restore Clerk middleware + RBAC gate for (admin) here."*
- The Clerk middleware **does** now require a session (the `(admin)` group is absent from the public allowlist in `middleware.ts:22-49`), but **no role check exists anywhere** in the group. Any signed-in user of any tenant reaches `/admin` (all-tenants list) and `/ops`.
- `/ops` is the cross-tenant internal console (support / flags / cost), gated only by a hardcoded `const OPS_CONSOLE_ENABLED = true` (`mytheus-ui/src/app/(admin)/ops/page.tsx:12`).
- Both render from `@/lib/mock/data` (`admin/page.tsx:2`, `ops/page.tsx:5`), and `layout.tsx:18` still passes `mockOrgContext` — so no real tenant data leaks **today**.
- Backend: `mytheus-backend/services/admin_ops_service/main.py` is a healthz stub — `# TODO: add routers for /admin/tenant/* and /admin/ops/*`. The gateway already routes `/admin/tenant` and `/admin/ops` to it (`edge.py:176-177`).

---

## TIER-BY-TIER CONTRAST — is identity-tenant-service the only one doing it properly?

**No. The discipline is shared library code, not an identity-service local virtue.** `identity_tenant_service` is not special; it is one of five consumers of the same two modules. There are, however, two genuine outliers — and one of them is inside identity itself.

| Tier | JWT verification | Tenancy enforcement | Verdict |
|---|---|---|---|
| **api_gateway** | `libs/shared/auth.py:60-118` — the same RS256/JWKS/`nbf`-required verifier (`edge.py:105`) | Resolves org once via `org_resolver.py:70-84`; strips + re-signs internal headers (`edge.py:101,116`) | **Same discipline** |
| **core_api** | Same shared verifier, via `authz.tenant_context` (`libs/shared/authz.py:96`) | `tenant_session` + `require_tier` on all 16 routes: `meetings.py:157,245,294,310`; `org_hierarchy.py:39,67,93,114`; `transcript.py:98`; `chat.py:91`; `speakers.py:50,88` | **Same discipline** |
| **bff** | Same shared verifier | `require_tier("member")` + `tenant_session` on all 3 routes (`dashboard.py:154`, `org_chart.py:171`, `people.py:70`) | **Same discipline** |
| **meeting_service** | Same shared verifier (`routes.py:160` `Depends(tenant_context)`) | Manages its own transaction with explicit `set_rls` (`routes.py:171`) rather than `tenant_session`, documented at `routes.py:32-40` as needed to close the transaction before a LiveKit HTTP call and `BackgroundTasks`. Webhook path: `webhook.py:301` `set_rls` + explicit `org_id` predicates | **Same discipline**, different plumbing for a stated reason |
| **identity_tenant_service (REST)** | Same shared verifier | `require_tier` + `tenant_session` (`users.py:77`, `context.py:35`, `tenant_settings.py:104,129` at `admin`, `taxonomy.py:33`); SCIM routes are 501 stubs behind `dependencies=[Depends(verify_token)]` (`scim.py:17`); Clerk webhook authenticates by Svix HMAC then `set_rls` (`routes/auth.py:99,129`) | **Same discipline** |
| **identity_tenant_service (gRPC)** | **NONE** | `org-id` taken from caller metadata and fed to `set_rls` (`grpc_server.py:54-64, 75`); `add_insecure_port` (`:152`) | **OUTLIER — see Critical Finding 1** |
| **mytheus-intelligence** | Its **own** parallel implementation, equally rigorous: `libs/shared/auth.py:112-120` RS256 + JWKS + `nbf` required + explicit algorithm allowlist | `FORCE ROW LEVEL SECURITY` + identical `NULLIF(current_setting('app.current_org', true))` predicate on its schema (`migrations/versions/9c2b7e4a1d03…:83-89`, `b3f7a1c9e402…:63-66`, `d415f023220e…:72-75`); app role `mytheus_ai_app` created `NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE` **explicitly stated, not defaulted** (`9c2b7e4a1d03…:60-78`), and that is the role in `DATABASE_URL` (`.env.example:33`); `set_rls` before every read (`search_service/routes.py:113`, `ingestion_service/routes.py:120,132,158,183`) | **Equal or stronger** — it is the only tier with a documented dual-layer story (RLS *and* Qdrant collection+filter). Weaker in one respect only: it reads a pre-resolved claim instead of resolving, so it cannot see a suspension (`auth.py:44-50`, self-disclosed) |
| **mytheus-realtime / realtime-gateway** | Real RS256 JWKS via `jose` (`libs/auth.ts:34-55`), fail-closed | Has **no database at all**; delegates org resolution and the participant decision to core-api under the user's own token (`coreApiClient.ts:114-123`), and org-scopes its Redis keys and Socket.io rooms (`libs/keys.ts:36-80`) | **Same discipline, appropriate to a stateless tier.** Two smaller gaps than the Python tiers: no explicit `algorithms: ["RS256"]` allowlist and no required-claims set passed to `jwtVerify` (`libs/auth.ts:43`) — safe today because the key comes from a JWKS RSA key set, but the Python verifiers state it and this does not |
| **mytheus-realtime / transcription-service** | n/a (consumer, no HTTP surface taking user tokens) | `FORCE ROW LEVEL SECURITY` on `transcript_chunks` with the identical predicate (`migrations/versions/be66698dbefb…:119-122`), plus a partition-privilege close (`migrations/_partition_privileges.py:14`); code always issues transaction-local `SET app.current_org` (`src/transcription_service/db.py:47-64`). **But the connecting role is `postgres`** — `db.py:13-22` states plainly that a least-privilege role mirroring `mytheus_app`/`mytheus_relay` is **deferred** as an ops/bootstrap concern, and that "the local superuser connection bypasses RLS" | **OUTLIER — the only tier whose RLS is aspirational rather than enforced by its role.** The code is correct; the credential is not yet |

**Answer to the question asked:** identity-tenant-service is *not* the only service doing this properly — its REST tier is unremarkable, using exactly the same `libs/shared/auth.py` + `libs/shared/authz.py` + `libs/shared/tenancy.py` as api_gateway, core_api, bff and meeting_service, and mytheus-intelligence independently matches it. The contrast that matters runs the other way: **the two places where the discipline breaks are (a) identity-tenant-service's own gRPC server, which has no authentication at all, and (b) transcription-service, which writes correct RLS code and then connects as a superuser.** Both are outliers *from* a standard the rest of the estate actually meets.

---

## CRITICAL FINDINGS

Ranked by severity. Only what is evidenced.

### 1. CRITICAL (by design; not currently deployed) — unauthenticated gRPC identity service trusts a caller-supplied `org-id`

`mytheus-backend/services/identity_tenant_service/grpc_server.py`

- `:54-56` — `metadata = dict(context.invocation_metadata()); raw_org_id = metadata.get("org-id")`
- `:60` — `org_id = UUID(raw_org_id)`
- `:75` — `await set_rls(session, org_id)`
- `:34-35, 80-100` — then reads `users` (email, role, skills, bio, avatar) and `rbac_grants` (tier) for that tenant
- `:152` — `server.add_insecure_port(f"[::]:{port}")`

There is **no token, no interceptor, no mTLS, no signature**. Any caller that can reach port 50051 names any tenant and reads that tenant's user directory, emails and RBAC tiers. This is precisely the client-supplied-org_id-that-is-trusted pattern, one layer below the one the HTTP path guards so carefully — and the file's own docstring frames metadata as arriving "from the mesh", a mesh that does not exist.

Stated precisely, so this is not overcalled: the gRPC server is **absent from `mytheus-platform/deploy/compose.prod.yml`** (only `mytheus-backend/Makefile:92` `make run-identity-grpc` runs it), and no client code calls it — `mytheus-intelligence/.env.example:44` declares `IDENTITY_GRPC_ADDR=localhost:50051` but no stub usage exists in that repo. It is a loaded gun, not a live breach. It ships in the image and is one compose entry from being live.

### 2. HIGH (latent today) — `/admin` and `/ops` have authentication but no authorization

`mytheus-ui/src/app/(admin)/layout.tsx:12-15`; `mytheus-ui/src/app/(admin)/ops/page.tsx:12`.

Any signed-in user of any tenant reaches the cross-tenant internal ops console. Real tenant data is not exposed **today** only because both pages read `@/lib/mock/data`. The moment either is wired to `admin-ops-service` — for which the gateway route already exists (`edge.py:176-177`) — this becomes a live cross-tenant read **with no change to the route**. The gap is a missing check, not a missing page.

### 3. MEDIUM — transcription-service enforces RLS in code but connects as a superuser

`mytheus-realtime/services/transcription-service/src/transcription_service/db.py:13-22`. `transcript_chunks` has `FORCE ROW LEVEL SECURITY` and the service always issues transaction-local `SET app.current_org`, but the connecting role is `postgres`, which bypasses RLS unconditionally. The file discloses this and defers the least-privilege role to ops. This is the same class of defect as D1 (`mytheus-platform/env/prod.env.example:13-29`), unclosed for this one service. Not currently exploitable via any external path — the service has no user-facing HTTP surface — but a bug in its own `org_id` threading would not be caught by the database, which is exactly the backstop the rest of the estate relies on.

### 4. MEDIUM — the signed internal-header control is inert, and the threat model claims otherwise

`edge.py:69-81` signs `x-mytheus-org` / `x-mytheus-user`; `verify_context` has **no production caller** — I grepped all 8 repos and every hit is `mytheus-backend/tests/test_gateway_edge.py:199,225,239-241`. Meanwhile `mytheus-docs/threat-model/stride.md:25` asserts "The receiving service verifies the signature before trusting it."

Currently harmless — nothing downstream reads the header either, and every service re-verifies the JWT instead — but the documented defense-in-depth layer does not exist, and anyone building a new service against that document will assume protection that is not there.

### 5. LOW (latent) — `PUBLIC_CONTAINS` uses substring matching

`mytheus-backend/libs/shared/public_routes.py:31,37` — `any(c in path for c in ("/webhook","/oauth","/clerk/","/stripe/","/livekit/","/callback"))`. A future route such as `/api/v1/meetings/{id}/callback` would be silently excused by the `check_authz.py` CI gate **and** marked as requiring no bearer token in the emitted OpenAPI, so a generated client would omit the token. No such route exists today.

### 6. LOW — LiveKit room name carries no org segment

`mytheus-backend/services/meeting_service/rooms.py:51-53` → `meeting-{meeting_id}`. Isolation rests on meeting-UUID uniqueness plus mint-time participation, not on structure — the exact reasoning `mytheus-realtime/libs/keys.ts:49-77` explicitly **rejected** for socket rooms after a real shipped leak. Room metadata does carry the internal `org_id` (`transport.py:74-107`). Low residual risk; inconsistent with the sibling decision.

---

## DEV BYPASSES PRESENT IN CODE

**None in any service, in any repo.** No `verify=False`, no decode-without-verification, no `SKIP_AUTH` / `DEV_MODE` / `ALLOW_ANON` / `AUTH_DISABLED`, no env flag that weakens verification, no `app.dependency_overrides` reachable from production code.

The historical bypasses are gone; only their tombstones remain, and those are **stale docs that contradict shipped code**: `mytheus-ui/PROGRESS.md:70`, `mytheus-ui/LANDING.md:11,60`, `mytheus-ui/docs/PAGES.md:31`, `mytheus-ui/CLAUDE.md:5`, `mytheus-ui/README.md:15`, `mytheus-ui/docs/CONTRIBUTING.md:29` all still describe the 4-July "sign-in redirects to /dashboard, auth bypassed" state as current. It is not — `src/middleware.ts` and `src/app/(auth)/sign-in/page.tsx` are real.

The one live shortcut is **not** env-gated, which is worse than if it were: `(admin)/layout.tsx:12-15` and `ops/page.tsx:12`'s `OPS_CONSOLE_ENABLED = true` are unconditional in every environment, dev and prod alike.

Two fail-closed-but-unconfigured items — deploy blockers, not security holes, but they mean the posture above has never actually run in prod:

- **`GATEWAY_SIGNING_KEY` is defined nowhere outside code and tests.** `edge.py:58-66` raises 503 `DEPENDENCY_UNAVAILABLE` when unset ("a default key is worse than no signing at all"). It appears in **no** compose file and **no** `mytheus-platform/env/*.example`. With `deploy/compose.prod.yml` as written, every authenticated request returns 503.
- **`env/prod.ui.env` has no `.example` and is `required: false`** (`compose.prod.yml:381-382`), so the UI container can boot with no Clerk keys; `mytheus-platform/env/` contains only `dev`, `prod`, `staging` and the four `prod.db.*` templates.

---

## SERVICE-TO-SERVICE AUTH — verdict

**ABSENT as a transport control; compensated at the application layer, with one hole.**

There is no mTLS, no shared secret, no internal JWT, and no signature verification between services. ADR-0013's agent-to-agent authorization has no implementation at all.

What actually holds the line is that every REST service independently re-verifies the end user's Clerk JWT and derives `org_id` itself (`libs/shared/authz.py:94-122`; `mytheus-intelligence/libs/shared/auth.py:143-149`). A pod that calls `core-api:8000` directly therefore still needs a real user token and still gets only that user's tenant. `realtime-gateway` propagates the user's token rather than re-minting or dropping it (`coreApiClient.ts:114-123`), so the identity survives the hop intact.

Network isolation is docker-compose-network only. No service publishes a host port — `compose.prod.yml` uses `expose:` throughout, and the sole ingress is Caddy → `api-gateway:8000` (`deploy/caddy/mytheus.caddy`, which explicitly refuses to point at `core-api`). But that Caddy is attached to **two** stacks' networks (`compose.prod.yml:366-376`), and the file itself records a name-collision incident where `mytheus.cloud` served another product.

The hole is that the one interface that is **not** JWT-verified — the identity gRPC service — is precisely the one that accepts an asserted tenant (Finding 1).

---

## MISSING PIECES

1. **gRPC authentication.** An interceptor requiring a verified caller identity (mTLS or an internal token) before `org-id` metadata is honored — or delete `add_insecure_port` until there is one.
2. **A server-side RBAC gate on `(admin)`**, landed *before* `admin-ops-service` gains routers.
3. **`admin-ops-service` routers** with `require_tier`, plus the platform-operator tier that the current three-tier `TIER_RANK = {"member":1,"admin":2,"owner":3}` (`authz.py:45`) does not model — an internal ops console has no representable role today.
4. **A least-privilege database role for transcription-service** mirroring `mytheus_app`, and a `DATABASE_URL` that uses it.
5. **Downstream verification of `x-mytheus-signature`**, or removal of the header and correction of `stride.md:25`.
6. **`GATEWAY_SIGNING_KEY`** in `compose.prod.yml` / `env/prod.env.example`, and a **`prod.ui.env.example`**.
7. **Gateway ROUTE_MAP entries and a Caddy WS route** for the AI and realtime tiers. Neither appears in `compose.prod.yml` — no `realtime-gateway`, no `transcription-service`, no intelligence services; the deployed set is postgres, redis, redpanda, temporal, migrate, web(core-api), relay, workflow, api-gateway, bff, identity-tenant-service, meeting-service, integration-service, billing-metering-service, admin-ops-service, notification-service, ui.
8. **The Clerk JWT template mapping** `public_metadata.internal_org_id` → the `internal_org_id` claim. **UNVERIFIED** — it is dashboard state invisible to code, and the AI tier 401s on every request until it exists.
9. **ADR-0021 explicit grants and ADR-0013 agent authorization.** `rbac_grants.tier` is the only authorization input in the system; no per-resource grant model and no agent-to-agent authz exist.
10. **Substring→prefix fix** in `public_routes.py:37`.
11. **Refresh the six stale UI docs** that still describe auth as bypassed.

---

## VERDICT

The user-facing chain — browser → Clerk → gateway → service → RLS-scoped row — is genuinely and carefully wired end to end, with real RS256/JWKS signature verification at three independent tiers, server-derived tenancy the client cannot influence, stripped-then-re-signed internal headers, and `FORCE ROW LEVEL SECURITY` under a `NOBYPASSRLS` role; I found no dev bypass anywhere and no unscoped tenant query in any request path. That discipline is shared library code rather than one service's virtue — api_gateway, core_api, bff, meeting_service and identity's REST tier all use the same three modules, and mytheus-intelligence independently matches them with a stronger dual-layer story for vectors and object keys. The two real defects are outliers from that standard, and both sit below the HTTP path: identity-tenant-service's own gRPC server accepts an unauthenticated, caller-asserted `org-id` and would hand over any tenant's user directory (dangerous by design, absent from the prod compose and currently uncalled), and transcription-service writes correct RLS code while connecting as a superuser that ignores it. Deployment lags the code substantially — the AI and realtime tiers are correct but have no ingress, `/admin` and `/ops` check for a session and never for a role, and with `GATEWAY_SIGNING_KEY` defined in no env file the gateway as configured would 503 every authenticated request before any of this matters.
