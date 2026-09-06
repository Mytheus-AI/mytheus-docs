# Sushan — Gateway, Auth, Identity, RBAC

**8 tasks.** Two of them block the entire team. Read the ordering note before you start.

Audit report: https://claude.ai/code/artifact/da281c5f-d990-40b4-9487-d9f857833b08

---

## 🚨 DO S1 AND S2 FIRST — THE WHOLE TEAM IS BLOCKED ON THEM

Nobody else can verify their work until these two are merged and deployed.

- Samar can start M4/M5/M6 without you; everything else on their list waits.
- Shaun can write all their code but cannot test any of it — every request 403s until S2.

**Announce in the channel the moment S1 and S2 are live.** Do not start S3–S8 before then.

---

## S1 — Fix the gateway signing key (P0, blocks everyone)

**Repo:** `mytheus-backend` + `mytheus-platform`

### The defect

`_signing_key()` at `services/api_gateway/edge.py:58-66` raises `ProblemDetail(503)` when
`GATEWAY_SIGNING_KEY` is unset. That is correct and deliberate — the comment explains it well:
*"A default key is worse than no signing at all, because it looks like a control while being
public knowledge."* Keep that behaviour.

Two things are wrong around it:

1. **The variable is set nowhere.** Grep across all 8 repos returns hits only in
   `tests/test_gateway_edge.py:78` and `:236`. It is in no `.env.example`, no compose file, no
   runbook.
2. **The raise escapes as a 500.** In `edge_auth`, the `try / except ProblemDetail` block ends
   after `resolve_org_id`. The call to `_attach_internal_headers(...)` — which reaches
   `sign_context` → `_signing_key` — sits one line *below* the `except`, outside the guard. The
   function's own docstring says why this matters: *"middleware sits outside the exception
   handlers the app factory installs, so a raise here would escape as an unhandled 500."*

**Net effect: every authenticated request through the gateway returns 500.**

### Why nobody caught it

`/healthz` is on the gateway's public path list, so it never enters `edge_auth`. The container
healthcheck passes, both CI post-deploy curls return 200, and the deploy reports fully green.
The health signal is structurally incapable of detecting this outage.

### Do

- [ ] Move `_attach_internal_headers(...)` inside the `try / except ProblemDetail` block so a
      missing key surfaces as the intended 503, not a 500.
- [ ] Add `GATEWAY_SIGNING_KEY` to `mytheus-backend/.env.example`, `mytheus-platform/env/`, and
      `deploy/compose.prod.yml` for the `api-gateway` service.
- [ ] Document generation + rotation in the platform runbook (it is an HMAC key — `openssl rand
      -hex 32` is fine).
- [ ] Add a test asserting the missing-key path returns 503, not 500.

### Done when

A request with a valid bearer token returns real data through the gateway on a fresh environment,
and unsetting the key gives a clean 503.

---

## S2 — Write RBAC grants so real users can do anything (P0, blocks everyone)

**Repo:** `mytheus-backend`

### The defect

This is the single highest-leverage fix in the audit.

- `libs/shared/authz.py:45` — `TIER_RANK = {"member": 1, "admin": 2, "owner": 3}`
- `libs/shared/authz.py:63` — `satisfies()` is `TIER_RANK.get(self.rbac_tier, 0) >= TIER_RANK[minimum]`
- The `RequestContext` field comment says it plainly: `rbac_tier: str  # "" when the user holds no
  grant, which satisfies no tier`
- The **only** `INSERT INTO rbac_grants` in the entire backend is
  `services/identity_tenant_service/seed/seed.py:196`
- The Clerk `user.created` webhook at `services/identity_tenant_service/routes/auth.py:168-190`
  inserts a `users` row and nothing else

So: a real person signs up → webhook mirrors them into `users` → they hold no grant → `rbac_tier`
is `""` → `0 >= 1` is false → **every `require_tier` route 403s**. The system works only for users
created by the seed script.

Everything else in this codebase that was built correctly sits behind a door that cannot be opened
for a real signup.

### Do

- [ ] On org provisioning (`services/identity_tenant_service/provisioning.py`), grant the first
      user `owner`.
- [ ] On Clerk `user.created`, insert a `member` grant **in the same transaction** as the `users`
      upsert. Match the existing idempotency discipline — `ON CONFLICT ON CONSTRAINT
      uq_rbac_grants_org_id_user_id DO NOTHING`, as `seed.py:196` already does.
- [ ] Decide and document the default tier for a user who joins an existing org via Clerk
      organization membership (this overlaps S3 — do them together if easier).
- [ ] Write an integration test against real Postgres as the `mytheus_app` role: create a user via
      the webhook path, then assert a `require_tier("member")` route returns 200. The existing
      `tests/conftest.py` already runs as the unprivileged role — follow that pattern.

### Done when

A brand-new Clerk signup can load `/dashboard` and get real data. Right now they cannot.

---

## S3 — Missing Clerk organization webhook handlers

**Repo:** `mytheus-backend`

`routes/auth.py` handles `user.created`, `user.updated`, `user.deleted`. It does not handle
`organization.created`, `organization.updated`, `organization.deleted`,
`organizationMembership.created`, `organizationMembership.updated`, or
`organizationMembership.deleted`.

Consequence: orgs and memberships only exist if something else creates them. Combined with S2,
this is the whole tenant-lifecycle surface.

### Do

- [ ] Add handlers for the six `organization.*` / `organizationMembership.*` events.
- [ ] Membership creation should write the matching `rbac_grants` row (ties into S2).
- [ ] Membership deletion should revoke it.
- [ ] Reuse the existing Svix verification, Redis dedupe and durable-inbox path — it is already
      correct, do not rebuild it.

---

## S4 — `user.deleted` is silently dead

**Repo:** `mytheus-backend`

`_clerk_org_of` at `routes/auth.py:68-78` resolves the org **only** from
`organization_memberships` in the webhook payload. Clerk's `user.deleted` payload does not carry
memberships, so the function returns nothing and the handler returns `"ignored"` at `:106-109`.

The test that covers this (`tests/test_clerk_webhook.py:222`) **fabricates a payload with
memberships**, so it passes while proving nothing. A real deletion from Clerk is dropped on the
floor.

### Do

- [ ] Fix org resolution for the delete path — look the user up by `clerk_user_id` when the
      payload has no memberships.
- [ ] Fix the test to use a realistic `user.deleted` payload shape (no memberships).
- [ ] Check whether any other handler depends on `_clerk_org_of` and has the same blind spot.

---

## S5 — `POST /orgs` and `/rbac` do not exist

**Repo:** `mytheus-backend`

`services/api_gateway/edge.py` routes both prefixes to `identity-tenant-service`, which implements
neither. Both 404 today. Without `/rbac`, there is no way for a tenant admin to grant or revoke a
tier through the product at all — S2 fixes the bootstrap, this fixes ongoing administration.

### Do

- [ ] `POST /orgs` — org creation surface.
- [ ] `/rbac` — grant, revoke and list tiers for a tenant. Enforce `require_tier("admin")` on
      mutations.
- [ ] Audit-log every grant change (coordinate with Shaun's H3, which builds the audit writer — if
      H3 is not merged yet, leave a clearly-marked TODO rather than a second implementation).
- [ ] Regenerate the OpenAPI spec (`scripts/emit_openapi.py`) so Shaun's H8 picks up the new types.

---

## S6 — Admin console has no role check

**Repo:** `mytheus-ui` ⚠️ *sensitive repo — `git fetch` first, and check the active rebuild branch*

`src/app/(admin)/layout.tsx:15-18` carries a `TODO(auth)` and a `mockOrgContext`. The stale comment
claims "anyone can reach /admin" — that part is no longer true, `src/middleware.ts` does require a
session. But there is still **no tier check**, so any signed-in member of any tenant reaches the
admin console.

### Do

- [ ] Gate the `(admin)` layout on `rbac_tier >= admin`. This depends on S2 existing, which is why
      it is yours.
- [ ] Remove `mockOrgContext` and the stale TODO comment.
- [ ] Same gate on `(admin)/ops` — note `ops/page.tsx` has `const OPS_CONSOLE_ENABLED = true`
      hardcoded; decide whether that stays and make it deliberate either way.

---

## S7 — gRPC identity server is unauthenticated and plaintext

**Repo:** `mytheus-backend`

`services/identity_tenant_service/grpc_server.py:152` — the server listens on `:50051` with no
credentials. `org-id` arrives in **unsigned metadata** (`:55-64`). Any process that can reach the
port can read any tenant's user context.

The REST edge HMAC-signs the equivalent header. gRPC has no counterpart. Today the exposure is
limited to the docker network, but it is the weakest link in an otherwise strong auth story.

### Do

- [ ] Add authentication — verify the caller's JWT in the interceptor, matching what
      `libs/shared/auth.py` already does for REST, rather than trusting metadata.
- [ ] TLS or, at minimum, document explicitly why plaintext-on-docker-network is accepted and bind
      to the internal interface only.
- [ ] Confirm the port is not published to the host in `compose.prod.yml`.

---

## S8 — The service-to-service signature is never verified

**Repo:** `mytheus-backend` + `mytheus-docs`

The gateway HMAC-signs `x-mytheus-org` / `x-mytheus-user` together (correctly — the comment about
mix-and-match attacks is right). But `verify_context` at `edge.py:79` is called **only from
`tests/test_gateway_edge.py`**. Nothing in production verifies the signature.

Tenancy currently survives this because every REST service independently re-verifies the caller's
JWT (`libs/shared/authz.py:94-122`) rather than trusting the gateway — which is a genuinely good
design. So this is not an active vulnerability.

It is, however, a **false claim in the threat model**: `mytheus-docs/threat-model/stride.md:25`
states "The receiving service verifies the signature before trusting it." That is not true.

### Do — pick one and be decisive

- **Option A:** enforce it. Add signature verification to the shared service dependency, so the
  defence-in-depth layer actually exists.
- **Option B:** remove it. Delete `sign_context`/`verify_context` and the header attachment,
  document that JWT re-verification is the sole mechanism.

Either way:

- [ ] Correct `stride.md:25` to describe what the code actually does.
- [ ] Note that Option B also removes the S1 failure mode entirely — worth weighing.

---

## S9 — `PUBLIC_CONTAINS` uses substring matching, weakening the CI authz gate

**Repo:** `mytheus-backend`

`libs/shared/public_routes.py:31,37`:

```python
PUBLIC_CONTAINS = ("/webhook", "/oauth", "/clerk/", "/stripe/", "/livekit/", "/callback")
return prefixed or any(c in path for c in PUBLIC_CONTAINS)
```

That is a **substring** test, not a prefix test. Any route whose path merely *contains* `/oauth`
or `/callback` anywhere — `/api/v1/meetings/{id}/oauth-notes`, say — is treated as public.

### Scope it correctly before fixing

This is **not** a runtime auth bypass. This module is consumed by only two things:

- `scripts/check_authz.py`, the CI gate that fails any route declaring no auth dependency
- `libs/shared/openapi.py`, which marks operations as requiring the bearer scheme

The runtime gateway uses its own stricter list (`edge.py:40-54,85` — exact match plus
`startswith`), so a mis-matched route is still protected in production.

The real risk is a **false negative in the safety net**: a future route with an unlucky path
silently passes the gate that exists to catch missing auth, and is advertised as public in the
emitted OpenAPI spec. Given this gate is the main systemic defence in the authz-review process,
that is worth closing.

### Do

- [ ] Anchor the match — prefix or path-segment matching rather than `in`.
- [ ] Add a test with a deliberately adversarial path (`/api/v1/x/oauth-notes`) asserting it is
      **not** public.
- [ ] Confirm no existing route was silently exempted: run `check_authz.py` before and after and
      diff the route set it considers public.

---

## Do NOT build these (deliberately deferred)

Per the roadmap phase boundary at W36. Their absence is the plan being followed:

- **SSO / SAML / SCIM** — deferred at the tenant-claim decision, even though ADR-0010 and
  `19-api-architecture.md` §5.1 still specify the surface.
- **The 428 consent check in meeting-service** — ADR-0018 puts the migration and application code
  explicitly out of scope; the flag defaults to `false`.
- **Agent-to-agent authorization (ADR-0013)** beyond what S8 decides.
- **`pii_tokens` migration** — ADR-0020 is accepted for the data model but *blocked for
  implementation* on key custody.

---

## Your checklist

- [ ] **S1** — Gateway signing key ⚠️ *blocks everyone*
- [ ] **S2** — RBAC grant writer ⚠️ *blocks everyone*
- [ ] 📣 **Tell the team S1 + S2 are deployed**
- [ ] **S3** — Clerk organization webhook handlers
- [ ] **S4** — `user.deleted` org resolution
- [ ] **S5** — `POST /orgs` and `/rbac`
- [ ] **S6** — Admin console role check
- [ ] **S7** — gRPC authentication
- [ ] **S8** — Decide and settle the internal signature
- [ ] **S9** — Anchor `PUBLIC_CONTAINS` matching
