# Sushan — Auth & Identity, plus all of mytheus-mobile

**12 non-mobile tasks + 25 mobile tasks.** Two of the non-mobile ones block the entire team.

Audit: https://claude.ai/code/artifact/da281c5f-d990-40b4-9487-d9f857833b08
Evidence: [`audit/`](audit/) · [`verify/verify-mobile.md`](verify/verify-mobile.md)

---

## 🚨 S1 AND S2 BLOCK EVERYONE — DO THEM FIRST

Samar can proceed on P4–P8 without you; Shaun can write D1/D2 but verify nothing. Announce the
moment S1 and S2 are deployed.

## ⚖️ Your load is ~2.5× the others

Mobile is 25 tasks on top of 12. If you want it evened, hand off **S7, S9, S11, S12** — all
self-contained, none dependent on your auth work. See the README.

---

# PART 1 — AUTH & IDENTITY

## S1 — Gateway signing key (P0)

`services/api_gateway/edge.py:58-66` — `_signing_key()` raises `ProblemDetail(503)` when
`GATEWAY_SIGNING_KEY` is unset. That is correct; keep it. Two things are wrong around it:

1. The variable is set **nowhere** outside `tests/test_gateway_edge.py:78,236` — no `.env.example`,
   no compose file, no runbook.
2. The raise escapes as a **500**, not the intended 503: in `edge_auth`, the
   `try / except ProblemDetail` ends after `resolve_org_id`, and `_attach_internal_headers(...)` —
   which reaches `sign_context` → `_signing_key` — sits one line **below** it. The function's own
   docstring explains exactly this: middleware sits outside the app's exception handlers.

Nobody noticed because `/healthz` is on the public path list, so it never enters `edge_auth`.
Container healthcheck passes, both CI post-deploy curls return 200, deploy reports green.

- [ ] Move `_attach_internal_headers(...)` inside the `try/except`
- [ ] Add `GATEWAY_SIGNING_KEY` to the backend `.env.example`, `mytheus-platform/env/`, and the
      `api-gateway` service in `compose.prod.yml`
- [ ] Document generation and rotation (`openssl rand -hex 32`)
- [ ] Test asserting the missing-key path returns 503, not 500
- [ ] **Check the live VPS** — prod may be 503ing (or 500ing) every authenticated request right now

**Done when:** a valid bearer token returns real data on a fresh environment, and unsetting the key
gives a clean 503.

---

## S2 — RBAC grant writer (P0)

The highest-leverage fix in the audit.

- `libs/shared/authz.py:45` — `TIER_RANK = {"member": 1, "admin": 2, "owner": 3}`
- `:63` — `satisfies()` is `TIER_RANK.get(self.rbac_tier, 0) >= TIER_RANK[minimum]`
- The field comment says it: `rbac_tier: str  # "" when the user holds no grant, which satisfies no tier`
- The **only** `INSERT INTO rbac_grants` in the backend is `seed/seed.py:196`
- The Clerk `user.created` handler (`routes/auth.py:168-190`) writes a `users` row and nothing else

Real signup → `users` row → no grant → `0 >= 1` false → **403 on every route**. The system works
only for seeded users.

- [ ] Grant `owner` to the first user on org provisioning (`provisioning.py`)
- [ ] Grant `member` on Clerk `user.created`, **in the same transaction** as the `users` upsert,
      with `ON CONFLICT ON CONSTRAINT uq_rbac_grants_org_id_user_id DO NOTHING` (mirror `seed.py:196`)
- [ ] Decide and document the default tier for a user joining an existing org (overlaps S3)
- [ ] Integration test against real Postgres as the `mytheus_app` role — create a user via the
      webhook path, assert a `require_tier("member")` route returns 200. Follow `tests/conftest.py`.

**Done when:** a brand-new Clerk signup can load `/dashboard` and get real data.

---

## S3 — Clerk organization webhook handlers

`routes/auth.py` handles `user.created/updated/deleted` only. The six `organization.*` and
`organizationMembership.*` events are unhandled, so orgs and memberships only exist if something
else creates them.

- [ ] Handlers for all six events
- [ ] Membership create → write the `rbac_grants` row (ties to S2); membership delete → revoke
- [ ] Reuse the existing Svix verification, Redis dedupe and durable-inbox path — it is correct

---

## S4 — `user.deleted` is silently dead

`_clerk_org_of` (`routes/auth.py:68-78`) resolves the org **only** from `organization_memberships`
in the payload. Clerk's `user.deleted` payload has none, so the handler returns `"ignored"` at
`:106-109`. The covering test (`tests/test_clerk_webhook.py:222`) **fabricates memberships**, so it
passes while proving nothing.

- [ ] Resolve the org by `clerk_user_id` when the payload carries no memberships
- [ ] Fix the test to use a realistic `user.deleted` shape
- [ ] Check whether any other handler shares the blind spot

---

## S5 — `POST /orgs` and `/rbac` do not exist

The gateway routes both to identity-tenant-service, which implements neither. Without `/rbac` there
is no way for a tenant admin to grant or revoke a tier through the product — S2 fixes bootstrap,
this fixes ongoing administration.

- [ ] `POST /orgs`
- [ ] `/rbac` — grant / revoke / list, `require_tier("admin")` on mutations
- [ ] Audit every grant change via `libs/shared/audit.py` (it has 8 live call sites already —
      follow the existing pattern)
- [ ] Regenerate the OpenAPI spec so the generated clients pick up the new types

---

## S6 — `GET /users/me` does not exist

**Cross-cutting; both clients need it.** No endpoint anywhere returns the caller's own internal
`user_id`. The mobile client cannot learn its own identity, which blocks self-profile and any
"assigned to me" view. The web UI has the same gap — `org-context.ts:46` hardcodes `role: "member"`
because there is nothing to ask.

- [ ] `GET /users/me` returning internal `user_id`, email, role, and `rbac_tier`
- [ ] Unblocks your own S10, and mobile C-tasks

---

## S7 — gRPC identity server is unauthenticated and plaintext

`grpc_server.py:152` listens on `:50051` with no credentials; `org-id` arrives in **unsigned
metadata** (`:55-64`) and is fed straight to `set_rls`, reading any tenant's users and
`rbac_grants`. The REST edge HMAC-signs the equivalent. Latent today — not in prod compose.

- [ ] Verify the caller's JWT in an interceptor, as `libs/shared/auth.py` does for REST
- [ ] TLS, or document why plaintext-on-docker-network is accepted and bind internal-only
- [ ] Confirm the port is not published to the host

---

## S8 — The internal signature is never verified

The gateway HMAC-signs `x-mytheus-org` / `x-mytheus-user` together (correctly — the mix-and-match
reasoning is sound). But `verify_context` (`edge.py:79`) is called **only from tests**.

Not an active vulnerability: every REST service independently re-verifies the caller's JWT
(`libs/shared/authz.py:94-122`), which is a good design. But `mytheus-docs/threat-model/stride.md:25`
claims the receiving service verifies the signature, and that is false.

- [ ] **Decide:** enforce it (real defence in depth) or remove it (also removes S1's failure mode)
- [ ] Either way, correct `stride.md:25` to describe the code

---

## S9 — `PUBLIC_CONTAINS` uses substring matching

`libs/shared/public_routes.py:31,37` — `any(c in path for c in PUBLIC_CONTAINS)`. Any path merely
*containing* `/oauth` or `/callback` counts as public.

**Scope it correctly:** this is **not** a runtime bypass. The module feeds only `check_authz.py`
and the OpenAPI emitter; the runtime gateway uses its own stricter exact+prefix list
(`edge.py:40-54,85`). The risk is a **false negative in the CI gate** that exists to catch missing
auth — which matters more given Samar's P12 found `make test` runs that gate with `|| true`.

- [ ] Anchor the match (prefix or path-segment, not `in`)
- [ ] Adversarial test: `/api/v1/x/oauth-notes` must not be public
- [ ] Diff `check_authz.py`'s public route set before and after

---

## S10 — Admin console has no gate

`(admin)/layout.tsx:12-18` has **no session check and no role check** — only middleware protects
it. Worse, `org-context.ts:46` hardcodes `role: "member"`, so a role gate is **unwritable today**
without a backend source of truth.

**Depends on S6.** ⚠️ *`mytheus-ui` is sensitive and its scope is unsettled — see the README. This
task is safe under either outcome because it is a security gate, not feature wiring.*

- [ ] Land S6 first, then read the real tier from it
- [ ] Gate `(admin)` and `(admin)/ops` on `rbac_tier >= admin`
- [ ] Remove `mockOrgContext` and the stale TODO
- [ ] `ops/page.tsx` hardcodes `OPS_CONSOLE_ENABLED = true` — make it deliberate either way

---

## S11 — `consent_required` is a trap with no key

ADR-0018's consent gate is **enforced** (`meeting_service/routes.py:186`) but has **no write path
anywhere**. `meeting_participants.consented_at` is written only by a test
(`tests/test_meeting_token_route.py:355`). If any tenant sets `tenant_settings.consent_required =
true`, **every meeting in that tenant becomes permanently unjoinable** — 428 forever, no API to
clear it. Inert only because the flag defaults false.

Note ADR-0018 puts the *migration and application code* out of scope. The gate shipping without a
write path is the part that is not deliberate.

- [ ] Either add a consent-write endpoint, or make `consent_required` unsettable until one exists
- [ ] Whichever you choose, record it — a flag that bricks a tenant needs the decision written down

---

## S12 — `tenant_settings` is barely written

- Provisioning creates **no** `tenant_settings` row at all (self-documented at
  `routes/tenant_settings.py:29-33`)
- Of 8 columns, only `pii_redaction_enabled` is ever written. `sso_config`,
  `retention_days_transcripts`, `retention_days_worklogs`, `default_locale` and
  `agent_confidence_overrides` have no writer
- `GET /tenant-settings` returns only `pii_redaction_enabled` (`:72`)

- [ ] Create the row on provisioning
- [ ] Expose and write the columns that have a consumer; drop or document the ones that do not
      (`sso_config` is deferred with SSO — leave it, note why)

---

# PART 2 — MYTHEUS-MOBILE (all 25 tasks)

Baseline is green today: `tsc` exit 0, jest 14/14. **Contract drift with the backend is zero** —
all four called endpoints verified field-by-field, including query params, limits and role tiers.

## ⚠️ Corrections to what the first audit reported

- The 84 MB APK is **untracked** (`?? mytheus-0.1.0.apk`), not committed
- `debug.keystore` is **untracked** too — `.gitignore:20` ignores `/android` wholesale, which kills
  the `!debug.keystore` negation
- **The real defect is worse:** `android/app/build.gradle:112-115` — the **release** build signs
  with `signingConfigs.debug` (`storePassword 'android'`). The shipped APK is debug-signed.

## C — Safe now (13)

- [ ] **C1** Remove `RECORD_AUDIO` + `NSMicrophoneUsageDescription`. No audio dependency exists;
      `app.json:15` and `start-session.tsx:104` both state a privacy claim no code produces.
      Implementing capture is blocked anyway — `transcription-service`'s `/internal/audio`
      accepts then immediately closes (`main.py:119-124`). Factual risk: policy exposure under
      Apple 5.1.1 and Play permissions policy. No iOS build has ever existed. **Zero functional loss.**
- [ ] **C2** `eas.json` + EAS credentials so the release build stops signing with the debug key
- [ ] **C3** Fix cold-start auth flash — `(tabs)/_layout.tsx:8-9` bounces signed-in users to
      sign-in; `useSessionLoaded` exists at `session.ts:34` and is imported by nothing
- [ ] **C4** `profile.tsx:74-88` shows a **fabricated name and email** to a genuinely signed-in
      user while Clerk's `useUser()` is already in the app. Use the real identity.
- [ ] **C5** Wire the People screen to `GET /bff/org/chart` (`bff/org_chart.py:169`) — it returns a
      real member roster (`user_id`, `email`, `role`, `role_in_team`) under member tier.
      **Note:** `useTeams()`/`useDepartments()` are the *wrong* hooks for this; the comment at
      `hooks/index.ts:36-39` saying no roster exists is stale. Teams/departments are the grouping
      affordance only.
- [ ] **C6** Wire the library/meetings screen to `GET /api/v1/meetings`
- [ ] **C7** Fix `recap.tsx` — it takes no meeting id and both callers drop it. Must land before
      recap can be wired to anything.
- [ ] **C8** Remove the untracked APK from the working tree; add it to `.gitignore` properly
- [ ] **C9** Fix `.gitignore:20` so the `/android` wholesale ignore stops masking real files
- [ ] **C10** Mobile CI — there is none. Add `tsc` + jest on PR.
- [ ] **C11** Screen and hook tests — `__tests__/api-client.test.ts` is genuinely good (14 real
      behavioural tests) but covers only `src/api/client.ts`
- [ ] **C12** Fix mobile `CLAUDE.md` — names 5 surfaces that do not exist and the wrong token-mint
      path (shipped is `/api/meetings/:id/token`)
- [ ] **C13** Delete or wire the dead `useTeams`/`useDepartments` hooks once C5 settles their role

## B — Blocked on the gap-21 scoping decision (8)

`mytheus-mobile` realign is deferred in Plans 0 and 1-2, and **Plan 5-6 §7 was never assigned a
task** — status gap 21 calls for an explicit scoping decision before anyone picks this up.

**Get that decision before starting B1–B8.** They are the screen-by-screen mock removal: actions,
ask, insights, live-session, start-session, person detail, home's mock `weeklyStats` banner, and
the full mock module retirement.

## X — Blocked on absent backend (4)

Nothing to wire these to. Do not start.

- Ask / agent surface — no `rag-service`
- Actions / tasks — no `action_items` table anywhere
- Insights — no backend
- Push notifications — `notification-service` has zero routes, and the Expo token contract is net-new

---

## Your checklist

**P0 — blocks everyone**
- [ ] **S1** Gateway signing key
- [ ] **S2** RBAC grant writer
- [ ] 📣 Announce S1 + S2 deployed

**Auth & identity**
- [ ] **S3** Clerk org webhook handlers
- [ ] **S4** `user.deleted` org resolution
- [ ] **S5** `POST /orgs` + `/rbac`
- [ ] **S6** `GET /users/me` *(unblocks S10 and mobile)*
- [ ] **S7** gRPC authentication *(hand-off candidate)*
- [ ] **S8** Settle the internal signature + fix `stride.md:25`
- [ ] **S9** Anchor `PUBLIC_CONTAINS` *(hand-off candidate)*
- [ ] **S10** Admin gate *(needs S6)*
- [ ] **S11** Consent write path *(hand-off candidate)*
- [ ] **S12** `tenant_settings` provisioning + columns *(hand-off candidate)*

**Mobile**
- [ ] **C1–C13** safe now — start with C1, C2, C4
- [ ] **B1–B8** hold for the gap-21 scoping decision
- [ ] **X** four items blocked on backend — do not start
