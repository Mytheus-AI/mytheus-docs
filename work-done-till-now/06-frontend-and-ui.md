# Frontend and UI

Status: IN PROGRESS. Last verified 2026-11-10 (section 8); section 6 on 2026-11-07; sections 1 to 5 last verified 2026-08-08.

Written because "which repo is the actual product" was a live question, and the answer was not
recorded anywhere. Two repositories hold a Next.js application and only one of them is the
product. Nothing in the doc set said so, and one of the two is not mentioned in the doc set at
all, so the only way to answer was to read both.

## 1. Which repository is which

| Repo | What it is | Evidence |
|---|---|---|
| `mytheus-ui` | THE product. The single Next.js application the doc set specifies. | 40 pages matching doc 20 section 2.2 exactly; wired to Clerk and the real `bff`; named in the repo table in this repo's `CLAUDE.md` |
| `mytheus-web` | A UI/UX draft. Not a product, not deployable, not canon. | 2 commits; 15 pages; zero Clerk references, zero API calls, zero LiveKit references; 7 files of mock fixtures; its own notes call it "a UI mock"; absent from the doc set entirely |

`mytheus-web` is a design input to be harvested into `mytheus-ui`, not a thing to deploy or
maintain in parallel. Its most valuable artifact is preserved at
[plan/frontend-meeting-ux-gap-analysis.md](../plan/frontend-meeting-ux-gap-analysis.md).

## 2. There is no separate marketing site, and no meeting subdomain

Doc 20 section 2.2 specifies ONE application whose route groups separate the surfaces:

    (marketing)/   unauthenticated: landing, pricing, self-serve signup
    (auth)/        Clerk sign-in / sign-up
    (app)/         authenticated shell, including meetings/[meetingId]/room
    (admin)/       tenant-admin and internal ops

So the marketing site is a route group inside `mytheus-ui`, not a separate repository, and the
meeting platform is a route inside `(app)` rather than a separate product. `mytheus-ui`
already implements that tree.

No subdomain appears anywhere in the doc set. A search across every markdown file for
`app.`, `meeting.`, `www.` against the product domains returns nothing. Splitting marketing
onto its own domain or deployment is therefore a NEW decision, not a restoration of one:
per this repo's rules it needs a canon edit plus an ADR before it is built, not after.

## 3. What `mytheus-ui` already has

40 pages across the four route groups, including the meeting room, transcript view, decisions,
tasks, org chart, search, settings, billing, admin and ops.

The API contract is already correct and does not need rework:

- `src/lib/api/client.ts` is SERVER-ONLY by construction. It imports `auth()` and `getToken()`
  from `@clerk/nextjs/server`, so importing it from a client component is a build error rather
  than a token leak.
- It reads `API_GATEWAY_URL` and refuses to default it. The comment states the reason: a wrong
  default would send real session tokens to an unintended host.
- `.env.example` states that the gateway origin is the ONLY REST origin the app holds, because
  "going straight to core-api or bff would bypass the edge that derives org_id from the token".

### Verified end to end against the live gateway

The single path the app calls today, `/bff/dashboard`, was exercised through the real chain
(Cloudflare -> Caddy -> api-gateway -> bff). That is what "wired" means here: not that the code
compiles, but that the deployed edge answers the exact request this client makes, in the shape
this client parses.

It did not, at first. `Authorization: Bearer not-a-real-jwt` returned 500 INTERNAL rather than
401, because `get_signing_key_from_jwt` parses a token before looking a key up and the decode
error escaped a handler that caught only JWKS failures. That specific bug breaks THIS client:
`client.ts` redirects to sign-in only on AUTH_REQUIRED or TOKEN_EXPIRED, so a user with an
expired token would have seen an error page instead of being signed back in. Fixed in
`mytheus-backend` (`libs/shared/auth.py`); malformed, truncated and undecodable tokens all now
return 401 AUTH_REQUIRED through the deployed gateway.

That contract matches the deployed topology exactly: `api.mytheus.cloud` terminates at
`api-gateway`, which is the component that fails closed before any tenant-scoped store is
reached. Wiring the app is therefore configuration - point `API_GATEWAY_URL` at
`https://api.mytheus.cloud` - and not a code change.

## 4. Open items

| Item | Detail |
|---|---|
| Unmerged landing work in `mytheus-ui` | Less alarming than the raw counts suggest. Four of `main`'s five extra commits are merges FROM `local-mytheus-rebuild`, so that branch is the working branch and `main` absorbs it by PR. Exactly one commit is unmerged - `refactor(landing): cut redundant copy, fix the story canon, derive dates` - and it is landing-page work, so the newest marketing content is not on `main`. A dry-run merge is CLEAN, no conflicts, and preserves `main`'s only unique change (the progress-docs section of `CLAUDE.md`). Not merged here: this repo is not modified without asking. |
| 18 meeting-UX gaps | See the imported gap analysis. Tier 1 is the live-meeting surfaces, and the largest single miss is live in-call captions, which doc 10 makes the flagship real-time surface. |
| Action-item confirm surface | Gap 2. The backend enforces the confirmation rule (only `core-api`'s outbox emits `decisions`/`actions`, only on confirm). A UI that shows detected actions as passive text with no Confirm affordance cannot satisfy it. |
| Clerk JWT template | Unrelated to the UI but blocks the AI tier: `public_metadata.internal_org_id` must be mapped into the token claim. See `mytheus-platform` `deploy/README.md`. |
| Deployment | DONE except DNS. `mytheus-ui` is containerised, published to `ghcr.io/mytheus-ai/mytheus-ui`, and running as the `ui` service behind the shared Caddy at `mytheus.cloud` and `www.mytheus.cloud`. The origin serves it today; the apex and www A records still point at Hostinger's parked page and must be repointed to `187.127.190.246` (proxied) in Cloudflare. |
| No tests in `mytheus-ui` | CI now exists (typecheck, `next lint`, a real image build on every PR, publish and deploy on main). Vitest and Playwright are still absent because the repo has no test tooling and no test files; a job running zero tests and reporting green would read as coverage. Adding that stack is separate work. |
| Node 25 breaks the `mytheus-ui` build | On Node 25 the build compiles and generates all 57 static pages, then fails writing standalone output because `routes-manifest.json` is never emitted. `next`'s `engines` allows `>=18.17.0`, so nothing warns. The Dockerfile pins Node 20. |

### The marketing site was not publicly reachable

Found while verifying the container, and it is the reason the marketing site could not simply
be switched on. `src/middleware.ts` allowlists `/legal(.*)`, which matches no route in this
app; the pages that exist are `/privacy` and `/terms`. Both were therefore protected by Clerk
and returned 404 to a signed-out visitor - the two documents that must be publicly reachable,
and both linked from the site footer. `/students` and `/students/verify` were gated the same
way, so the campaign they exist for could not convert.

Fixed in `mytheus-ui` PR #5. Verified against a running container: all 13 `(marketing)` routes
return 200 signed out, while `/dashboard`, `/meetings` and `/settings` stay protected. Nothing
guards it from regressing, because that repo has no tests and no CI.

### A generic service name served the wrong product

The `ui` service came up healthy and Caddy reached it on `ui:3000` with a 200 - and the page
returned was "Atithivishram: Property Management for Indian hotels". Compose derives a network
alias from the service key, `atithivishram-prod` also has a container aliased `ui`, and the
Caddy that fronts 443 is attached to both stacks' networks, so the bare name was ambiguous
there and resolved to theirs.

Every check short of reading the response body passed: container running, healthcheck green,
upstream reachable, HTTP 200. It was caught by looking at the title.

Fixed with an explicit namespaced alias, `mytheus-webapp`. `redis` and `postgres` are ambiguous
across the two stacks for the same reason and are harmless only because Caddy never proxies to
them - every other container resolves names inside its own network. The general rule this
established: any upstream the shared Caddy must reach needs a name unique across all three
networks, not merely unique within its own stack.

## 5. Verification

- Read-only throughout. `mytheus-ui` was fetched before being judged and was not modified.
- Page counts taken from `git ls-tree` against `origin/main` and `origin/local-mytheus-rebuild`.
- `mytheus-web` characterised by grep across its tree: 0 matches for clerk, 0 for
  fetch/axios/useSWR/NEXT_PUBLIC_API, 0 for livekit, 7 files matching mock/fixture.
- Subdomain claim verified by grepping every markdown file in this repo; zero matches.

## 6. Auth pages, the missing screens, and dev-server speed (2026-11-07)

All in `mytheus-ui`, branch `feat/custom-auth-pages`, commits `a561629`, `dce7bf2`, `53b0c1e`,
`bcf3a00`.

### 6.1 Custom auth flows (`a561629`)

Sign-up crashed with Clerk's "the SignUp component is not configured correctly": `/sign-in` and
`/sign-up` were plain pages, not catch-all routes, so Clerk could not render its sub-steps.
Both are now `[[...rest]]` routes, which is the layout doc 20 section 2.2 already specifies. The
boxed Clerk components were replaced with full-page custom flows built on `useSignIn` and
`useSignUp`, in the site's design language:

| Route | What it does |
|---|---|
| `/sign-in` | email and password, or Google |
| `/sign-in/factor-one` | one-time code by email |
| `/sign-in/factor-two` | two-step verification (authenticator, email or SMS code, backup code) |
| `/sign-in/forgot-password` | reset by emailed code, then signs in |
| `/sign-up` | details, or Google |
| `/sign-up/verify-email-address` | six-digit email code |
| `/sign-up/continue` | anything Clerk still requires, for example after Google |
| `/sso-callback` | the Google return page and Clerk's bot check (`#clerk-captcha`) |
| `/onboarding/organization` | Clerk's `choose-organization` session task, mapped by `taskUrls` |

The middleware allowlist gained `/sso-callback` and `/onboarding`. A new account is signed in
but "pending" until it belongs to an organization, and `auth.protect()` treats pending as
signed out, so gating the onboarding page would loop the user back to sign-in.

Creating a workspace there creates a Clerk organization only. It does not provision a tenant:
`POST /api/v1/orgs` stays operator-only, so a self-created workspace signs in but its data reads
are refused until an operator provisions it. That is unchanged from the prebuilt Clerk screen.

### 6.2 The screens doc 20 and the plans name but the app lacked (`dce7bf2`)

| Route | Source in the doc set | Data |
|---|---|---|
| `/people/[id]` | Plan 5-6 section 3.5 | real `GET /bff/people/{user_id}` for a `user_id`; mock for org-chart name slugs |
| `/meetings/[meetingId]/decisions/[decisionId]` | Plan 7-9 section 3.8 (decision lineage, `FOLLOWS_FROM`) | mock |
| `/meetings/[meetingId]/collaborations` | Plan 10-13 section 3.5 (`agent_collaborations` log) | mock |
| `/tenant` | doc 20 section 7, Plan 10-13 section 3.14 (tenant admin console) | mock |
| `/tenant/usage` | Plan 7-9 sections 3.1 and 3.4 (usage and agent budget) | mock |

Mock pages say so on the page, and buttons with no service behind them (invite, export,
erasure) are disabled rather than faked.

Deviation: doc 20 files the tenant admin console under `(admin)/tenant`. It lives in the
`(app)` shell instead, because in `mytheus-ui` the `(admin)` group is the platform-staff surface
(every tenant, ops) with its own header and no app navigation, while a workspace admin
managing their own team belongs in the app they already use. The URL, `/tenant`, is unchanged.

The workspace role in the org context now comes from the Clerk organization role
(`org:admin` maps to `admin`), so workspace admins see an Admin section in the navigation. It
decides only what the UI offers; the API enforces the RBAC tier. The platform-staff consoles
(`/admin`, `/ops`) left the workspace navigation and stay reachable by URL.

### 6.3 Dev-server speed (`53b0c1e`, `bcf3a00`)

Clicking around under `next dev` was slow on every machine, not one. The production build was
fast: on public pages it answered in 0.006 to 0.08 s, against 1.4 to 2.3 s for a first visit on
the dev server.

Root cause: `optimizePackageImports` listed only `@phosphor-icons/react`. The 16 Server
Component files that render icons import from the separate `@phosphor-icons/react/dist/ssr`
entry, which re-exports about 1,500 icons and their 1,500 definitions and was not optimized.
Every dev compile of a page importing one icon from it processed all of them, and
`settings/primitives` plus the dashboard parts use it, so nearly every signed-in screen paid
the cost. Production was never affected because it tree-shakes the unused icons.

Fix: both `ssr` entries added to `optimizePackageImports`. Measured with a throwaway page
importing five icons, cold dev server: 4,976 modules in 8.2 s before, 1,861 modules in 4.4 s
after, two more modules than the page it sits in. Bundle sizes in `next build` are unchanged.

Ruled out by measurement: Turbopack (`next dev --turbo`) was slower here, 13.1 s against 7.2 s
to first-load ten pages, and repeat requests 0.13 to 0.27 s against 0.04 to 0.09 s. Next's idle
page disposal was not the cause either: pages left idle for 75 s still answered in 0.06 to
0.10 s.

What remains is inherent to `next dev`: it compiles each page on first visit and after every
edit, never prefetches, and runs React's development build. `mytheus-ui`'s
`docs/CONTRIBUTING.md` and `README.md` now say to use `npm run build && npm start` to click
around, demo, or judge speed, and `npm run dev` for editing only.

### 6.4 Verification

- `npx tsc --noEmit` and `npx next lint` clean after each change.
- `npm run build` succeeds; route table and bundle sizes recorded from its output.
- Auth pages rendered and checked in a browser at 1024x768 and 375x812, with no horizontal
  overflow on the phone size. The invalid-email error path was exercised with a made-up
  address. The real sign-up, Google and workspace-creation paths were exercised by the
  maintainer.
- The five new screens were not viewed by this session (the browser it drove was not signed
  in); only the typecheck, lint and build evidence above covers them.
- Speed numbers are `curl` timings against `localhost:3000` and the dev server's own
  "Compiled ... (N modules)" log lines.

## 7. Doodle avatars, UI polish, and the auth sketchbook (2026-11-07, later)

All in `mytheus-ui`, pushed to `main`: commits `d8bcecf`, `4c4d049`, `de3159a`.

### 7.1 Doodle avatars (`d8bcecf`)

Every person now has a hand-drawn doodle face instead of two initials.

- `src/lib/doodle.ts` draws a doodle from a short seed: hair or hat, eyes, mouth, extras and colours, with tens of thousands of combinations. It is a pure function, so the server render and the browser always draw the same face.
- `Avatar` prefers, in order: the chosen doodle, a real photo, then a default doodle keyed by email. Email is the key every surface can see for a person, so their own sidebar and a teammate's view of `/people/[id]` agree. Clerk's generic placeholder image counts as no photo.
- Storage. No backend change and no new route. The seed goes to Clerk `unsafeMetadata.doodle`. The rendered PNG becomes the Clerk profile image, and the existing `user.updated` webhook in `identity-tenant-service` copies `image_url` into `users.avatar_url`. That is how the doodle reaches Postgres on the VPS.
- Existing accounts. `DoodleImageSync`, mounted in the app shell, gives an account with no seed its default doodle on its next signed-in visit and uploads the picture. It also finishes a sign-up doodle, whose picture could not upload before the session existed.
- Settings, Profile gained a doodle picker. A save drops the tagged profile cache through the `refreshMyProfile` Server Action.

Deviation: a Google sign-in photo is replaced by the default doodle on the next visit unless the user picks one. This was requested: doodles everywhere, existing accounts included.

Not verified: the webhook round trip into `users.avatar_url`. It needs a signed-in browser plus a webhook endpoint Clerk can reach, and neither was available to this session.

### 7.2 Polish (`4c4d049`)

- A new themed `Select` primitive replaces the native selects on Decisions (meeting filter) and Search (sort). The operating-system menu ignored the dark theme.
- Segmented controls on Search, Decisions and Tasks are now full pills inside their pill track, with even count badges.
- Tasks: assignee, due date and priority sit in fixed-width cells, so the rows line up.
- Command palette: the `.mx-app :focus-visible` ring drew an orange box around the bare input. It is removed.
- Usage: "View as table" is a styled disclosure row with a scrollable table and a sticky header.

### 7.3 Auth redesign (`de3159a`)

Every auth page is redrawn as a sketchbook.

- The stage is a doodled meeting around an ink-drawn table: people speak in hand-drawn bubbles, and the agent pencils the decision into a margin note.
- Type is the app's own pair (Newsreader and Hanken Grotesk) plus Caveat for handwriting, scoped by `src/styles/auth.css` under `.mx-auth`.
- Sign-up lets the visitor pick their doodle, which takes the empty seat at the table. The seed is sent with `signUp.create` and with `authenticateWithRedirect`.
- Organization onboarding has two paths, join a team or start a workspace. Join lists:
  - memberships and invitations;
  - verified-domain suggestions (`useOrganizationList` `userSuggestions`), with a "Requested" state.

  An empty state explains how to get invited. Clerk does not let anyone browse organizations they are not in, so joining by name search is not possible by design.
- The SSO callback ("making sure you're human") shows a doodle under a magnifying glass and a handwritten checklist. The checklist is pacing, not progress, and never claims completion. The Clerk challenge slot hides while empty.

Joining an existing workspace is the path that works end to end today. A self-created workspace still needs operator provisioning (`POST /api/v1/orgs`) before its data reads succeed.

### 7.4 Verification

- `npx tsc --noEmit` and `npx next lint` are clean, and `next build` succeeds (69 pages).
- Checked in a browser at 1440x900 and 375x812, with no horizontal overflow at the phone size:
  - sign-in and sign-up;
  - the doodle grid;
  - Tasks, Decisions, Search and the command palette, rendered with mock data on a temporary page that has since been deleted.
- Not viewed: the onboarding page, which needs a pending session, and the SSO callback past its first frame, because Clerk redirects without an OAuth handshake.
- The meetings and call surfaces were not modified. `/meetings/[id]/collaborations` renders through the shared `Avatar`, so it shows doodles.

## 8. Every authenticated screen on real data (2026-11-10)

Goal: no authenticated screen in `mytheus-ui` shows invented data. Before this pass the dashboard's tasks, decisions, recent meetings and "Needs you" panels, the decision log, billing, the workspace admin overview, integrations, the platform ops consoles, and parts of the settings screens read `src/lib/mock/data.ts` or hard-coded sample rows; Tasks, Usage and decision lineage fell back to sample data whenever no gateway was configured. Branches: `mytheus-backend` `feat/real-data-endpoints`, `mytheus-ui` `feat/real-data-everywhere`, this file on `docs/real-data-everywhere`. Not merged, not deployed.

### 8.1 What landed in `mytheus-backend`

- `GET /api/v1/decisions` (`core-api`, new module `services/core_api/routes/decision_log.py`, RBAC `member`). Confirmed and superseded decisions from meetings the caller is a participant of, newest first, each with its meeting's title and start, `supersedes`, and `superseded_by` (the decision that replaced it). `proposed` and `dismissed` rows never appear. The prefix was already `core-api`'s in `_canon/01-service-catalog.md` and already routed by `api-gateway`, so no canon or gateway change. `org_id` comes only from the edge; RLS hides other tenants, so the route has no tenant parameter.
- `GET /bff/dashboard`: every arm is now real. `action_items` (the caller's open confirmed tasks, same rows and order as `GET /api/v1/actions?assignee_id=me`) and `decisions` (recent confirmed decisions from the caller's meetings) replace the open-map `UnbuiltArm`; two arms are added: `recent_meetings` (the caller's ended meetings with participant, confirmed-decision and confirmed-or-done-task counts) and `needs_review` (proposed candidates in the caller's meetings above the tenant's per-agent surface threshold, the same rule `GET /meetings/:id/candidates` applies). The arms share one `AsyncSession`, so each query now takes an `asyncio.Lock`; the arms are still gathered for per-arm timeout and failure isolation.
- Tests: `tests/test_decision_log.py` (4: confirmed-only and newest first, superseded linkage, cross-tenant invisibility, a colleague sees only shared meetings) and 6 new cases in `tests/test_bff_dashboard.py` (each new arm, all six arms `ok` together, cross-tenant invisibility of tasks, decisions and candidates). `tests/test_openapi_contract.py` now asserts every arm has a typed item and that `UnbuiltArm` is gone, replacing the two tests that asserted the placeholder.
- `design/openapi/bff.yaml` and `core-api.yaml` regenerated. `oasdiff breaking --fail-on ERR` against `origin/main` reports no breaking change for either.

### 8.2 What landed in `mytheus-ui`, screen by screen

| Screen | Now reads | Before |
|---|---|---|
| `/dashboard` header and hero | `/bff/dashboard` `upcoming_meetings`; header counts from `action_items` and `needs_review`, a dash when that arm failed | counts from mock tasks and a hard-coded conflict |
| `/dashboard` Needs you | `needs_review`, with Confirm and Dismiss through the existing `confirmCandidate` / `dismissCandidate` Server Actions, then a refresh | one hard-coded "pricing vs launch" conflict |
| `/dashboard` Your tasks | `action_items`; due labels against the request's clock | `mockActions` against a fixed 2026-07-01 "now" |
| `/dashboard` Recent decisions | `decisions`, each linking to its lineage page | `mockDecisions` |
| `/dashboard` Recent meetings | `recent_meetings` with real counts | `mockMeetings` |
| `/dashboard` My notes | `worklogs` (unchanged, already real) | |
| `/decisions` | `GET /api/v1/decisions`; filters by status (in force, replaced) and meeting; lineage hops resolved within the log | `mockDecisions` with an invented priority |
| `/meetings/[id]/decisions/[decisionId]` | `bff` lineage (unchanged); with no gateway it now says history is unavailable | a sample chain of one |
| `/tasks` | `GET /api/v1/actions` (unchanged); the no-gateway sample mode is removed | `mockActions` when no gateway |
| `/tenant/usage` | `GET /usage`; the sample view and its invented daily chart are removed, no gateway says unavailable | `mockUsage` when no gateway |
| `/billing` | `GET /usage` plan (name, seat price, inclusions) and `GET /users` member count; Payments says not connected, no invoices | `mockBilling` invoices, seats and next charge |
| `/tenant` (workspace admin) | members and invites (unchanged), plan from `GET /usage`; Integrations and the export and erasure requests say not available yet | `mockBilling` seats and plan, `mockIntegrations` |
| `/settings/integrations` | static list of the four supported integrations, all "Not available yet", Connect disabled | three shown as connected to sample accounts, toggles that saved nothing |
| `/settings` | workspace rename saved to the Clerk organization by `renameWorkspace` (admins only, `orgTag` revalidated); plan row links to Billing; Notifications says not available yet | local-only save, a hard-coded "Starter plan" badge, three toggles that saved nothing |
| `/settings/profile` | name saved to Clerk; role and focus areas read from `GET /context/{id}` and saved by `saveAgentProfile` through `PATCH /users/{id}` | hard-coded "Founder" and three sample focus areas, local-only save |
| App sidebar workspace chip | the caller's access ("Workspace admin" or "Member") | "starter plan" for every workspace |
| `/admin`, `/ops` (platform staff) | no data; a "not available yet" page | `mockTenants`, `mockTenantOps`, and `mockOrgContext` in the layout |

Already real before this pass and untouched: meetings hub, detail, transcript, room and join, collaborations, Ask, org chart, people, the agent admin tab.

Structural changes: `src/lib/mock/data.ts` holds no data any more, only type re-exports for three meeting-room files that the Phase 11 session is editing (`components/sidebar/AgentSidebar.tsx`, `components/meeting/MeetingRoom.tsx`, `components/room/ChatPanel.tsx`); shared view types moved to `src/lib/types.ts`; the public marketing vignettes' copy moved to `components/landing/sample-content.ts`. `OrgContextValue` lost its `org.plan` field, which was always `"starter"`; the `OrgProvider` default is a blank value instead of a sample person. Removed as unused: `components/app/dashboard/data.ts`, `components/billing/SeatMeter.tsx`, `components/settings/Toggle.tsx`.

### 8.3 Deviations and decisions, with reasons

- Decision visibility is participant-scoped (log and dashboard), not org-wide. It is the rule `core-api` already applies to candidates and that ADR-0033 D1 applies to RAG citations; an org-wide log would have been a new, wider visibility rule decided in a UI pass.
- "Needs you" shows unconfirmed candidates, not cross-meeting conflicts. Conflict detection belongs to `ProactiveAgent` (Phase 11, being built in parallel); the candidates queue is real today and acting on it is the confirmation rule the product is built around.
- No `confirmed_at` column exists on `decisions`, so the log dates entries by `created_at` and labels it "Recorded", not "Confirmed".
- Billing shows the plan from `subscriptions` but no seat meter: `subscriptions.seats` defaults to 1 until the Stripe band sets it, so a seat meter would present a placeholder as a measurement.
- Integrations, payments, notification preferences, data-rights requests and the platform consoles show a not-available state rather than a new endpoint, because their owning services (`integration-service`, the Stripe part of `billing-metering-service`, `notification-service`, `admin-ops-service`) are unbuilt bands, and `notification-service` is Phase 11's.
- The plan is fetched where it is shown (`GET /usage`, admin) instead of being put back into the session context; a non-admin sees "only admins can see the plan".

### 8.4 Open gaps

- `components/sidebar/AgentSidebar.tsx` still labels a saved context card "Context from Sarah's agent". It is a Phase 11 file, so it was left alone; the Phase 11 session should use the real source name.
- Once Phase 11 lands, point the three remaining `@/lib/mock/data` type imports at `@/lib/types` and delete the file.
- No live browser check: the authenticated screens need a signed-in Clerk session and a gateway running this branch's backend. Reading the running local stack's environment to wire one was refused by the permission system and not pursued.

### 8.5 Verification

All observed in this session on 2026-11-10.

- `mytheus-backend`, throwaway `postgres:15` and `redis:7-alpine` containers, `MYTHEUS_REQUIRE_DB=1`: `make proto`, `uv run ruff check .`, `uv run black --check .`, `uv run mypy .` and `uv run python -m scripts.emit_openapi --check` clean; `uv run pytest -q`: 799 passed, 1 skipped, 0 failed.
- `oasdiff breaking --fail-on ERR` (`tufin/oasdiff:latest`) for `bff.yaml` and `core-api.yaml` against `origin/main`: "No breaking changes to report", exit 0.
- `mytheus-ui`: `npx tsc --noEmit` and `npx next lint` clean; `npm run generate:api:check` matches the branch's specs; `npm test` 60 pass, 0 fail (new `tests/dashboard-format.test.ts`); `npx next build` succeeds (51 static pages) with the test Clerk publishable key, on Node 25 locally (CI builds the Docker image on Node 20).


## 9. The doodle meeting back on the auth pages, in the one-line style (2026-11-10)

The one-line redesign (`mytheus-ui` `fab728d`, then `a9ffe5a` and `f46aec9`) replaced the auth sketchbook of section 7.3 with a coral "memory line" drawing on a peach brand panel, now on the right of the form. The doodle meeting scene (table, people, speech bubbles, the agent's note) and the SSO page's magnifying glass were gone. This pass puts them back, redrawn to the rules in `mytheus-ui/docs/DESIGN-ONELINE.md`. Commit: `mytheus-ui` `220d9ee`.

### 9.1 What changed

- `src/components/auth/AuthRoom.tsx` (new) is the brand-panel scene on every auth page: sign-in, email code, two-step, forgot password, sign-up (start, verify, continue) and workspace onboarding.
  - Five doodle people sit around a table and take turns talking. A decision line is kept on a card by the agent.
  - The visitor gets a seat on sign-up and onboarding: a dashed open seat first, then their doodle with a "You" chip.
  - During onboarding, the workspace name is written on the table top.
  - Each page passes its own lines and note through `AuthShell`'s new `scene` prop, which replaces `art` / `artLabel`.
- Below `lg`, the slim band above the form shows a row of the doodle faces ("your team, remembered", or "and you" once the visitor has a doodle). It replaces the short line drawing there.
- `SsoCallback` shows a doodle drawing itself in under a sweeping magnifying glass. This replaces the looping logo mark. The status copy and the `#clerk-captcha` card are unchanged.
- `AuthArt.tsx` (the memory-line drawings) is deleted, because nothing imports it any more. Git history keeps it.

### 9.2 How it blends with the one-line language, and why

- **The coral line:** the only line in the scene is the coral table top, drawn once on arrival. The coral line also loops round the decided words on the agent's card, the way it circles the hero's key phrase on the marketing site. This keeps to "one bold line, everything else quiet".
- **Styling:** bubbles and the agent's card are white `rounded-card` surfaces with `shadow-warm-*` and hairline rings. The decision bubble uses `signal-soft` / `signal-ink`. Cups and the pad use `--tint-*` fills. There are no raw hex values, and only the 20px and full radii are used.
- **Type:** speech is in Hanken and the kept note is in Newsreader. The Caveat handwriting face and its tilts were dropped, because the redesign retired it.
- **People:** the five people are the sample team (Maya, Raj, Priya, Sarah, Diego), drawn through `defaultDoodleSeed`, so they have the same faces `Avatar` gives them elsewhere.
- **Motion:** the old talking bob and wobble loops are gone; the speaker leans in and their ring turns coral. The rotating conversation is the view's one orchestrated moment. With reduced motion, the scene holds on its decision line with the kept card already showing.

### 9.3 Verification

All checks below ran in this session on 2026-11-10:
- `npx tsc --noEmit -p .` was clean.
- `npx next lint` was clean on the six changed auth files.
- The dev server was run on :3000 and pages were checked in the browser at 1440x900:
  - `/sign-in`, `/sign-up` and `/sign-in/forgot-password` rendered the scene.
  - Measured bubble placement clears the speaker's face.
  - The decision bubble and kept card appear.
- `/sso-callback` rendered the doodle and glass before Clerk redirected (no real OAuth handshake locally).
- At 375px, the sign-up band showed the crowd with the visitor's doodle. There was no horizontal scroll and no console errors.
- Not checked live: `/onboarding/organization` needs a signed-in session, so the table-top workspace label was not seen in the browser.

### 9.4 Second pass: fuller, and no frame (`mytheus-ui` `e3eafaa`)

The owner reviewed the first pass and found it too empty: a small scene in a mostly bare peach panel. A version wrapped in a white product window was then rejected, because the box boundary looked bad. What shipped:
- **Size and frame:** the room is bigger, up to 620px wide, and has no frame; it sits straight on the peach stage.
- **Above the room:** the meeting's name and when it is, as plain text. A running Live timer was tried and removed at the owner's request (`mytheus-ui` commit after `e3eafaa`).
- **The room:**
  - A whiteboard on the wall behind the back row, and a soft floor shadow under the table.
  - Each bubble names its speaker. Who speaks is scripted per line (`Line.who`), so owners in the kept items match the speaker.
  - The room is shorter when there is no visitor seat, so sign-in has no empty floor.
- **"Your agent kept":** a strip under the room. Three items (Decision, Action, Question or Done, in the frozen chip tones) land one by one on a coral thread with beads, then the meeting loops. This replaces the single kept card and reuses the memory-line idea from `fab728d`.

`SceneProps` is now `{ title, when, lines: { who, text, keep? }[], you, label }`, with three keeps per page. During workspace onboarding, the typed name is also the meeting's name.

Verified on 2026-11-10:
- `tsc` and `next lint` were clean.
- In the browser at 1440x900, the room renders unframed on `/sign-in` (529 x 536px) and `/sign-up`, with no console errors.
