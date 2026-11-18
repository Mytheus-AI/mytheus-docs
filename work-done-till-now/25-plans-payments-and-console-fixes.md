# 25. Plans, payments and the console fixes

Not a plan band ([ADR-0051](../adr/0051-plans-payments-and-hard-token-stops.md)). Status: **DONE WITH GAPS** (built and verified locally on 2026-11-14; committed to `main` in backend, UI, platform and docs; deployment and a live Razorpay test payment are the gaps).

One session's list from the owner: suspension, the ops cost and audit views, the Briefings voice card, the Workspace overview, paid plans with Razorpay, staff emails, the Decisions toolbar, Home and Ask Myte tidying.

## What landed, per repo

### mytheus-backend

- Suspension: `libs/shared/org_resolver.py` still answers a suspended tenant with 401 `AUTH_REQUIRED`, but adds `reason: tenant_suspended`; `api_gateway/edge.py` passes a problem's extra fields through. Unknown and soft-deleted orgs stay byte-identical.
- Ops cost and console views read the larger of `agent_budgets.tokens_consumed` (live) and the nightly `usage_aggregates`, so usage no longer reads 0 until the rollup runs.
- `admin_ops_service/pipeline.py`: dead-letter counts (every `{topic}.dlq`, high minus low watermark) and consumer lag (high watermark minus committed offset per group and topic) from Redpanda with the Kafka admin API, in a worker thread with short timeouts; `null` only when Redpanda does not answer. Wired into `GET /admin/ops/costs` and `GET /admin/ops/health`.
- The audit search's 403 carries `reason: no_support_grant`.
- Migration `a7c3e9f1b2d4`: the catalog of ADR-0051 D1, subscriptions moved to the newest plan versions, this month's budgets rescaled with `hard_cap = token_quota`, `agent_budgets.alert_75_sent`, `alert_90_sent`, `credit_tokens`, and the `payment_orders` and `payg_settlements` tables (RLS `tenant_isolation`).
- `billing_metering_service/budgets.py`: hard stop, alerts at 75/90/95, `enforce` by default, `set_subscription_plan`, `sync_budget_to_plan` (rescales the month, re-arms alerts no longer crossed), `add_credit_tokens`, and the lazy lapse of a bought month (settling Pay as you go first).
- `billing_metering_service/payments.py`: `POST /billing/checkout`, `POST /billing/checkout/verify` (signature check, then apply in one transaction, then the Redis mirror and the staff email after commit), `GET /billing/status` (member: plan and whether AI is paused; admins also usage).
- `billing_metering_service/payg.py`: window pricing, settlement, Razorpay refunds of the remainder; the nightly job (`jobs.py`) lapses plans and retries refunds. `GET /usage` gains `paid_until`, `payg`, `balance_due`, `last_settlement`.
- `POST /admin/ops/tenants/plan` (staff): any plan, Unlimited included, no payment.
- `libs/shared/staff_mail.py`: one designed email per kind (new workspace, plan change, payment, credits, bug report, feedback) to `STAFF_ALERT_EMAILS` or each staff account's Clerk email, through the existing Resend/SMTP sender; best effort, after commit. Called from both provisioning paths, checkout, the staff plan switch and `POST /admin/tenant/feedback`.

### mytheus-ui

- `/suspended`: `api()` redirects there on `tenant_suspended` (`apiSafe` lets the redirect through); the page signs the member out and explains. Public in the middleware.
- Ops: Cost and lag and Health show dead letters and lag; the audit viewer explains a missing support grant; the workspace panel switches plans; plan names read as product names; no-cap plans show no quota.
- Settings > Briefings draws the Myte from the profile id like the rest of the app.
- Workspace overview: an AI this month card fills the right column; the plan card has an Upgrade button.
- `/workspace/billing/plans`: Free, Pro, Ultra and Pay as you go as cards, the current one glowing in `--signal-line` / `--signal-glow`; Razorpay Checkout in Mytheus colours (`components/billing/CheckoutButton.tsx`); extra credits (`CreditsCard`); a balance-due panel. Usage and billing: Upgrade, the 75% and AI-paused notes, a Payments card with the Pay as you go window and last settlement.
- `BudgetBanner` in the app shell: AI paused for everyone at 100%; admins see the 75/90/95 steps, dismissible per step.
- Marketing pricing rewritten to the four plans.
- Decisions: the meeting filter sits in the search row. Home: the starter chips and Recent decisions removed. Ask Myte: two suggestions, the composer on one line with scope as icon buttons with tooltips (`components/ask-myte/Tip.tsx`), follow-ups as an Ask next list, answer actions as one row of icon buttons.
- Wire types regenerated (`npm run generate:api`).

### mytheus-platform

- `env/prod.env.example`: `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_CURRENCY`, `RAZORPAY_INR_PER_USD`, `BUDGET_ENFORCEMENT_MODE`, `STAFF_ALERT_EMAILS`.

### mytheus-docs

- `_canon/02-data-model.md`: `agent_budgets` columns, `payment_orders`, `payg_settlements`. ADR-0051.

## Deviations, and why

- A suspended workspace's own members are told it is suspended (ADR-0010 D2 kept one response for every case). The caller holds a Clerk-verified token for that organization, so it already knows the workspace exists; telling it nothing let every read fail into an application error.
- The staff plan switch calls billing's module in-process rather than over a service hop, to avoid a new service secret on the server (ADR-0051 Consequences).
- Prices stay in US dollars and are charged in INR by default: the owner's prices are in dollars, and INR works on every Razorpay account without enabling international payments.
- Pay as you go price per token ($100 per million, the same rate as Ultra) and the credit pack ($10 per 100,000) were not given by the owner; chosen so Pay as you go is never cheaper per token than Ultra's allowance.

## Open gaps

- Deploy, then a Razorpay test-mode payment end to end (keys go in `prod.env`, auto-capture on in the Razorpay dashboard).
- No Razorpay webhook: a payment captured but not verified (browser closed mid-way) is not applied automatically.
- ~~The budget mirror in Redis keeps the old allowance for a workspace until its next charge or plan change after the migration.~~ Closed 14 November 2026: `GET /billing/status` republishes it when the row is blocked (see below).
- Mobile shows none of the plan screens.

## Verification

- Backend: `uv run pytest tests/test_org_resolver.py tests/test_gateway_edge.py tests/test_admin_ops_api.py tests/test_admin_ops_pipeline.py tests/test_billing_metering.py tests/test_billing_checkout_unit.py tests/test_staff_mail.py tests/test_openapi_contract.py`: all pass; the database-backed tests skip without Docker locally and run in CI. `ruff`, `black`, `mypy` clean on every touched package; `python -m scripts.emit_openapi` and `scripts.check_authz` run.
- UI: `npx tsc --noEmit` clean; `node --test tests/ops.test.ts tests/ops-console.test.ts` 13 pass; eslint clean on touched files.

## 14 November 2026: what "AI is paused" actually stops, and owner UI fixes

The owner saw the AI-paused banner while Ask Myte, its popup and New meeting all still worked. Two causes: the app never read the paused state anywhere but the banner, and the agent tier's budget gate reads the Redis mirror, which could still carry a pre-migration allowance (the open gap above).

### mytheus-backend

- `services/core_api/routes/meetings.py`: `create_meeting` reads this month's `agent_budgets.state` under the caller's RLS (read only) and answers 402 `QUOTA_EXCEEDED` when it is `blocked` and `BUDGET_ENFORCEMENT_MODE=enforce`. Covers the app, the API and Capture (the extension creates core-api meetings). `alert_only` never refuses, matching the agent gate.
- `services/billing_metering_service/payments.py`: `GET /billing/status` republishes the mirror (`write_mirror`) when the row is `blocked`; a Redis failure is logged and the status still answers. Every member's page load reads this, so the gate catches up within one page view.
- Tests: `tests/test_meetings_api.py` (refused while blocked under `enforce`, allowed under `alert_only`; database-backed, run in CI), `tests/test_billing_checkout_unit.py` (republish when blocked, not otherwise, Redis down still answers).

### mytheus-ui

- `src/lib/ai-status.tsx`: `AiStatusProvider` (set from `billing.blocked` in `(app)/layout.tsx`), `useAiPaused()`, and `AiPausedNote` (what is off, plus Buy credits or upgrade for admins, "your admin can turn it back on" for members).
- Paused: Ask Myte page and Ctrl+J sheet swap the composer for the note, starters and Talk are disabled, nothing is sent (an `/ask?q=` hand-off included); the Home ask box is disabled; the sidebar New meeting and the meetings-hub New meeting are disabled (a `?new=1` link no longer opens the form); Capture shows the note; `createMeeting` maps a 402 to a plain paused message.
- `BudgetBanner`: a card in the page flow (not sticky, no 1200px inner cap), says Mytes, Ask Myte, Talk and new meetings are off, and publishes its height as `--ai-banner-h`; the Ask Myte frame subtracts it, so the page is no longer cut off at the bottom under the banner.
- Owner fixes, one commit each: Ask Myte history says New chat / Your chats and its rail header lines up with the chat header (h-14, divided); Your Myte drops its ask box and Open Ask Myte; Save Myte sits under the Voice card (`AgentPanel` `beforeSave`); Topics you watch suggestion chips and add row tidied; Settings > Briefings drops the Voice card (the voice is set on Your Myte only) and puts speed beside language; Settings > Profile drops the workspace logo (Workspace console only); the Extra AI credits card no longer glows.

### Deviations, and why

- ADR-0051 D2 said captions and transcription continue while paused; the owner asked that no new meeting start, since meetings are where the Mytes work. Amended in the ADR; meetings already running keep captions.
- core-api reads `agent_budgets` directly (read only, under RLS) rather than calling billing over HTTP, the same in-process read admin-ops already makes; billing stays the one writer.

### Verification

- Backend: `uv run pytest tests/test_billing_checkout_unit.py tests/test_openapi_contract.py` pass; `python -m scripts.emit_openapi --check` matches; ruff, black, mypy clean on touched modules. `tests/test_meetings_api.py` needs Docker (CI).
- UI: `npx tsc --noEmit` clean, eslint clean on touched files; `npm test` 720 of 722 pass, the 2 failures (`tests/skeletons.test.ts`) fail identically without these changes (Windows path separators).

## 15 November 2026: mytheus-mobile completion pass (appended here; not part of the plans and payments band)

This section records a parity and ship-readiness pass on `mytheus-mobile`. It was appended to the newest numbered file as instructed; the subject is unrelated to plans and payments. Nothing was committed, pushed or branched in any repo. All changes sit in the `mytheus-mobile` working tree.

### Audit result

- No mobile screen imports `@/data/mock`; every screen reads through TanStack hooks in `src/api/hooks/*.ts` over `src/api/endpoints.ts`. Nothing to fix.
- Covered on the phone already: Home (dashboard), Inbox (notifications), Meetings (with join and room), My work (tasks, promises, status requests), Ask, decisions list and detail, briefing detail, capture, settings, workspace switch.
- Web only by design (mobile `CLAUDE.md`): org overview and Workspace area (agents, brain, data, members, support), billing and plans, integration install and OAuth callback, API keys, admin and purchase flows.

### What landed, per repo

#### mytheus-mobile

- CI: `.github/workflows/ci.yml` runs `npm ci`, typecheck, lint, Jest, `generate:api:check` (sparse checkout of `Mytheus-AI/mytheus-backend` main, `design/openapi`, passed as `MYTHEUS_OPENAPI_DIR`) and `expo export --platform android`. Node 20, npm cache, concurrency cancel. It needs a repo secret `BACKEND_REPO_TOKEN` (read-only fine-grained token for `mytheus-backend` contents). The export step uses placeholder public `EXPO_PUBLIC_*` values.
- Crash reporting: `@sentry/react-native ~7.2.0` (the Expo SDK 54 pin). Off unless the build has `EXPO_PUBLIC_SENTRY_DSN`.
  - `src/lib/config.ts` reads `sentryDsn` (optional; does not affect the "not connected" state).
  - `src/observability/options.ts` (pure, tested): `parseDsn` accepts only an https public-key DSN; `buildSentryOptions` returns null without a DSN and disables default personal data, screenshots and view hierarchy; `scrubEvent` keeps method, URL without query string and `user.id` only, drops headers, cookies and bodies, and removes `org_id`, `orgId`, `organization_id` at any depth; `scrubBreadcrumb` drops console breadcrumbs and strips query strings from network ones.
  - `src/observability/sentry.ts` is the only file that calls the SDK (`initObservability`, `identifyUser`, `reportError`, a test reset).
  - `src/observability/init.ts` and `entry.ts` start Sentry before `expo-router/entry`; `package.json` `main` now points at `./src/observability/entry.ts`.
- EAS: `eas.json` `submit.production` gains `android { track: internal, releaseStatus: draft }` and an empty `ios {}`. No account ids or key paths.
- People: read-only directory and person profile, no new backend surface.
  - `endpoints.personProfile` (`GET /bff/people/{user_id}`, path parameter only), `qk.person(id)`, `PersonProfile` and `PersonMeeting` types, `src/api/hooks/people.ts` (`usePersonProfile`, only for UUID ids; `useDirectory` over the existing `GET /users`, sharing the `qk.members` key).
  - `app/(app)/people/index.tsx` (list, a narrowing search box above 6 people, loading, error with retry, empty) and `app/(app)/people/[personId].tsx` (header chips, `mailto:` link hidden on your own profile, about, recent meetings, skills, topic chips; 404 and NOT_FOUND show "This person is not available"; the response `org_id` is never shown).
  - Entry point: a "People" row in the Workspace card of `app/(app)/settings.tsx`.
- Tests: `__tests__/observability.test.ts` (11 new, mocks the Sentry SDK) and `__tests__/people.test.tsx` (6), plus a `GET /bff/people/u1` line in the `__tests__/api-client.test.ts` contract snapshot. The observability test file is outside the track's stated file list; it was added because Jest only finds tests under `__tests__/`.

### Deviations, and why

- The `@sentry/react-native` config plugin that `expo install` added to `app.json` was removed. With it, any native build without `SENTRY_AUTH_TOKEN` fails because the iOS source-map upload step exits 1. The native SDK still works through autolinking; the plugin only uploads source maps. `app.json` is back to its original content.
- Sentry starts from a new entry file via `package.json` `main` rather than from `app/_layout.tsx`, because the track could not edit the layout.
- Certificate pinning (N9) was not built. There are no real pins (SPKI hashes for `api.mytheus.cloud` plus a backup pin and a rotation plan must come from `mytheus-platform`); invented pins would lock every install out. The likely library covers fetch and XHR only, so on iOS the realtime-gateway WSS socket would not be pinned, and LiveKit and Clerk use separate network stacks. Library maintenance status was not checked (no network lookup).
- The web `/search` was not built: it is the Ask view, which the mobile Ask tab already covers. `GET /bff/people/{user_id}` was already in the mobile generated route table and unused, so no API regeneration was needed.
- People on the phone is a flat read-only directory, not the web org chart (departments, teams, rosters), which stays on the web.

### Verification

Run in `C:/Users/sam/Desktop/mytheus/mytheus-mobile`, observed by the building tracks and re-run by a review pass:

- `npx tsc --noEmit`: exit 0.
- `npm run lint`: exit 0 (only the notice "Using legacy ESLint config").
- `npx jest --ci` / `npm test`: 13 suites, 137 tests, 1 snapshot, all pass (the People track alone saw 12 suites, 126 tests before the Sentry tests landed). Harmless `expo-notifications` Expo Go `console.warn` in `voice-picker.test.tsx`.
- `npx expo export --platform android`: exit 0, Android `.hbc` bundle 10.1 MB; the custom entry bundles. A scan of the bundle for secret patterns found none (the `sk_` hits were string-table overlaps).
- GATE FAILED: `MYTHEUS_OPENAPI_DIR=../mytheus-backend/design/openapi npm run generate:api:check` exited 1. `core-api.ts` and `identity-tenant-service.ts` differ from the spec. Backend main (`4bcc7ed`, equal to origin/main) moved on after mobile's `src/api/generated` was last regenerated at `575c605` (identity country and state, voice STT, billing tiers). The drift predates this pass, but the new CI workflow would be red on its first run until `npm run generate:api` is run and the result landed with it. The People track did not run this check.
- The CI workflow itself has not been executed (it needs `BACKEND_REPO_TOKEN`).
- `npx expo prebuild` was not run, so the merged Android manifest (foreground-service types, `BLUETOOTH_CONNECT`) is unverified.

Review findings against the People track, reported by the review and not confirmed fixed in this record:

- Name precedence differs from `mytheus-ui`: the web uses identity `display_name`, then the Clerk name, then the email part; mobile `useDirectory` puts the Clerk name first, and `PersonScreen` ignores `display_name`, so one member can show as "Zoe Park" in the list and "zoe" on the profile.
- A code comment in `app/(app)/people/[personId].tsx` says the fallback matches the web; it does not (the web turns `raj.kumar` into "Raj Kumar", mobile shows `raj.kumar`).
- `app/(app)/people/index.tsx` calls Clerk `useOrganization()` for display; `src/api/hooks/identity.ts` already does the same, but the mobile `CLAUDE.md` says only `_layout` and Settings touch tenancy.

### Open gaps and decisions

Not done (needs owner or credentials):

- EAS `projectId` is missing from `app.json` (`expo.extra.eas.projectId`), so no Expo push token is minted and push registration stays off. Run `eas init` against the real Expo account.
- EAS iOS submit needs `ascAppId`, `appleTeamId`, `appleId`; Android needs the Play service-account key uploaded to EAS credentials (not committed).
- Sentry: source maps and native debug files are not uploaded (plugin left out); add the plugin with organization and project, `SENTRY_AUTH_TOKEN` as an EAS secret. No DSN is set anywhere. `identifyUser` is not wired to sign-in and sign-out, the RFC 7807 `trace_id` is not attached in `src/api/client.ts`, tracing is off, and `.env.example` lacks a commented `EXPO_PUBLIC_SENTRY_DSN=` line. Adding the SDK is a native-module change: it needs a new store or development build and a runtime-version bump, and must not ship OTA.
- `system.md` entries are unwritten: the Sentry DSN gating and scrubbing rules, the N9 pinning deferral, and the People decision (proposed text: read-only directory from identity `GET /users` plus `GET /bff/people/{user_id}`, org chart and admin editing stay on the web, entry at Settings > Workspace > People, no new backend route). The review notes the mobile `CLAUDE.md` requires this entry.
- Not built, build_now candidates from the audit: a briefing-preferences screen (`briefingPreferences` GET and PUT already wired in `endpoints.ts` with no UI), a meeting transcript viewer (`GET /api/v1/meetings/{id}/transcript` exists; an in-memory view respects the no-durable-transcript rule), and the native permissions and manifest check via `expo prebuild`.
- Stale docs to fix: `mytheus-mobile/docs/superpowers/specs/2026-07-02-mytheus-brand-redesign-design.md` line 70 mentions mock data that no longer exists; `mytheus-mobile/system.md` line 41 promises view-only billing, but no billing hook or screen exists (needs_decision: build it or correct the doc).

needs_decision items:

- Phone parity for: global search beyond Ask, notification and settings sub-pages (profile, listening, memory, feedback, agent), meeting collaborations, Decisions Time Machine and Calls (L, wide-screen views), Digest versus briefings overlap, Work settle flow, listening meetings and `capture/connect`, briefings list route, and person names tappable elsewhere (recap participants, room participants, "Confirmed by"). Each needs the contract confirmed in the generated spec before building.
- `eas.json`: no development profile (`developmentClient: true`), no channels, no `runtimeVersion` or `expo-updates`, so the OTA model in `system.md` cannot work; `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY` is absent from both profiles (builds show "not connected" unless set in EAS env); the preview profile points at the production API.
- Store fields: `usesNonExemptEncryption`, privacy manifests, `buildNumber` and `versionCode` policy, and https universal and app links (only the `mytheus://` scheme exists, so web share links will not open the app).
- Pinning (N9): whether pinning fetch only is acceptable, SPKI choice, and rotation runbook with `mytheus-platform`.
- EAS Submit and release pipeline (tag, build, submit, update): blocked on Apple and Play accounts, store listings, privacy forms and an `EXPO_TOKEN` secret.
- Detox end-to-end smoke gate: not set up; decide whether Maestro is acceptable, after the CI baseline and a development build exist.

## 16 November 2026: mytheus-mobile prep fixes and the Ask Myte port (appended here; not part of the plans and payments band)

This section records two passes on `mytheus-mobile`: four prep fixes against the findings in the 15 November section above, then the Ask Myte Phase A port with its review fixes. It was appended to the newest numbered file as instructed; the subject is unrelated to plans and payments. The detailed Ask Myte record (what was ported, the transport and why) is in [24-ask-myte.md](24-ask-myte.md) under mytheus-mobile and is not repeated in full here. Nothing was committed, pushed or branched in any repo; all changes sit in working trees, and no progress log was written inside any code repo.

### What landed

#### mytheus-mobile: prep fixes (closing the 15 November findings)

- API types regenerated from `mytheus-backend/design/openapi`: five files under `src/api/generated/` changed (`bff.ts`, `core-api.ts`, `identity-tenant-service.ts`, `meeting-service.ts`, `notification-service.ts`). `tsc` had no fallout. This closes the failed `generate:api:check` gate recorded above.
- Name precedence now matches the task: identity `display_name`, then the Clerk name, then the email local part with `[._-]` replaced by spaces and each word capitalised ("raj.patel@" becomes "Raj Patel"). The rule lives in `src/lib/people.ts` (`personName`, `nameFromEmail`) and is applied in `joinPeople`, so the directory, the person profile, assignees and "Confirmed by" all read through it. The wrong code comments were rewritten. `mytheus-ui` was not opened in this pass, so it is not confirmed that its `personName` also humanises the email.
- `mytheus-mobile` `system.md` section 15 gained N26: People as a flat, read-only directory over identity `GET /users` plus the `display_name` join, with the reasoning (this closes the unwritten People entry listed above).
- `app/(app)/people/index.tsx` no longer calls Clerk `useOrganization()`. There is no `OrgContext` in the repo, so `usePeople` in `src/api/hooks/identity.ts` (which already reads the Clerk org) returns `orgName`, and `useDirectory` passes it through.
- Tests: `__tests__/lib.test.ts` (new precedence test) and `__tests__/people.test.tsx` (names such as "Maya (you)" and "Raj").

#### mytheus-mobile: Ask Myte Phase A

- The Ask tab streams rag-service `POST /api/rag/myte` through the gateway and keeps conversations in core-api `/api/v1/myte/threads*`, replacing the single-shot `POST /api/v1/ask`. It shows steps, text with `[n]` markers, numbered sources, follow-ups, the D10 failure copy per reason, Stop with Retry, History (cursor-paged sheet, delete after a confirm) and New chat.
- Pure modules ported verbatim from `mytheus-ui` main @ `093129f` into `src/lib/ask-myte/` and `src/lib/rag/ask-live.ts`; `runAskMyte` in `src/api/ask-myte.ts` is the device port of the web route; `ApiClient.stream()` and the `streamFetch` option (`expo/fetch` injected in `app/(app)/_layout.tsx`); hooks in `src/api/hooks/ask-myte.ts`; `MyteAnswer.tsx` and `ThreadsSheet.tsx`. `useAsk` and `src/lib/citations.ts` were deleted. No dependency was added.
- `system.md` section 15 gained N27 (what was ported, what is deferred, how the stream is read and why).
- Review fixes, after a review pass that failed on the points below: answer text now matches the web (paragraphs, lists, bold, dead `[n]` dropped; `src/lib/ask-myte/answer-text.ts`); deleting an already-deleted thread (404) counts as success; each save write is bounded at 15 s and each read at 6 s (`RequestOptions.timeoutMs`), with one same-key retry then "unsaved"; an expired session shows "Sign in again"; leaving the Ask tab stops a streaming answer (`useFocusEffect`); `endpoints.ask` removed and the mobile `CLAUDE.md` gateway list names `POST /api/rag/myte`; Jest now fails any test that writes to `console.error` or `console.warn`, `expo-notifications` is stubbed in `jest.setup.ts`, `testTimeout` is 30 s, and ESLint moved to the flat config (`eslint.config.js`).
- Tests added: ported web `ask-myte-sse`, `ask-myte-reducer` and `ask-myte-upstream` suites, plus `ask-myte-run`, `ask-myte-hook`, `app-layout` and `ask` (screen) tests. Three old Ask tests in `__tests__/screens.test.tsx` moved to `__tests__/ask.test.tsx`.

#### mytheus-docs

- [ADR-0047](../adr/0047-ask-myte.md) amended: D8 now says a client with no server runs the same steps itself, with the same keys and time limits; D10 records the `offline` state as not produced on mobile, with the reason. This was an amendment of an existing ADR; no canon name, table, route or event changed.
- [24-ask-myte.md](24-ask-myte.md) carries the mytheus-mobile section for Ask Myte.

### Deviations, and why

- Layout: ports sit in `src/lib/ask-myte/` and orchestration in `src/api/ask-myte.ts`, not a top-level `src/ask-myte/`, so that all REST goes through `src/api/client.ts`.
- Delete asks for a confirm rather than showing a Toast with Undo, because the server deletes for good and there is nothing to undo.
- One `useOpenMyteThread` hook instead of a `useMyteThread(id)` query hook.
- The REST client is rebuilt per person and workspace (with an eslint-disable and its reason), so an in-memory conversation cannot survive a workspace switch.
- The D10 `offline` state is not produced: the phone has no connectivity signal without a new native module (NetInfo or expo-network), which needs a new development build and a runtime-version bump. A network failure shows as "unavailable". Recorded in N27, in 24-ask-myte.md and in ADR-0047 D10.
- `org_id` is never a parameter of `ApiClient.stream()`; the gateway injects it, as for all clients.
- `POST /api/rag/myte` has no OpenAPI in `mytheus-backend/design/openapi`, so its types are hand-ported from `mytheus-ui` (as the web does) and the path is pinned as `ASK_MYTE_STREAM` with a contract snapshot row. They can drift until a rag OpenAPI exists.

### Verification

Run in `C:/Users/sam/Desktop/mytheus/mytheus-mobile`.

Prep pass (observed by that track):

- `npx tsc --noEmit`: clean, run after the regenerate and again at the end.
- `npm run lint` (`expo lint`): no errors or warnings.
- `npm test`: 13 of 13 suites, 137 of 137 tests, 1 snapshot passed; `expo-notifications` Expo Go warning noise was present in the output.
- `MYTHEUS_OPENAPI_DIR=... npm run generate:api:check`: "Generated API types match". The earlier failed gate is closed.
- `npx expo export --platform android` was not run in the prep pass.

Ask Myte pass, final gates after the review fixes:

- `npx tsc --noEmit`: exit 0.
- `npm run lint`: exit 0, no findings, no legacy-config notice.
- `npm test`: exit 0; 20 suites, 187 tests, 2 snapshots passed in 13.0 s with nothing written to the console. Before the fixes, the build pass had 17 suites, 174 tests, 2 snapshots (baseline 13 suites, 137 tests, 1 snapshot).
- Cold-cache runs (`npx jest --no-cache`): with the default 5 s timeout one run had 6 failures (5 first-test timeouts plus 1 act() warning in the Join consent test); with `testTimeout` 30000, three cold runs passed 187 of 187.
- `npx expo export --platform android`: exit 0; one Android Hermes bundle of 10.2 MB written to `dist/` (git-ignored), no warnings.
- `generate:api:check` was not re-run after the Ask Myte pass; no generated types changed in it (the myte routes were already in `core-api.ts`).
- The first review pass confirmed the ported modules are byte-identical to `mytheus-ui` @ `093129f` apart from the header and the `.ts` import suffixes. The second review pass failed on the points listed under "Review fixes" above; those were fixed and the gates above were run afterwards. A re-review after the fixes was not run.

### Open gaps

- Not verified on a device: nobody has seen the SSE stream render progressively, without buffering, on a real iOS or Android development build. It is verified from source and by Jest with fake `ReadableStream` bodies only; the `_layout` `expo/fetch` wiring is tested in Jest, native streaming is not. If a device shows buffering, the fallback is an XHR `onprogress` reader behind the same `streamFetch` seam (documented, not built). Hermes `TextEncoder` (used by `reducer.toPayload` for the 64 KB cap) has not been run on a device.
- The VirtualizedList act() warning seen in the original report never reproduced in about ten warm and cold runs; the console gate will fail any test that prints it, but no known cause was removed.
- The D10 `offline` state is not produced (see Deviations).
- Deferred to Phase B: proposal cards and confirm writes (proposals are collected and stored, so they stay confirmable on the web, but a person asking from the phone sees no Confirm), answer cards, thumbs feedback, peer questions, routines, voice mode, attachments, memory management UI, thread rename, pin, search and delete-all, person and `as_of` scopes, page-aware entry points, and the Myte face states.
- Each ask is metered per model round against the workspace and per-person budget; Retry is a deliberate tap and there is no automatic re-ask.
- Backgrounding or killing the app after `done` but before the save leaves the answer shown but not stored; the save retries once with the same key, then shows "unsaved".
- Still open from the 15 November section: EAS `projectId`, store submit fields, Sentry source maps and `identifyUser` wiring, pinning (N9), CI secret `BACKEND_REPO_TOKEN`, and the `needs_decision` list. The three People review findings and the failed `generate:api:check` gate recorded above are fixed by the prep pass, and the unwritten People `system.md` entry is N26.
- Commits in `mytheus-mobile` must not carry a Co-Authored-By footer (mobile rule); the work is uncommitted and awaits the owner.
