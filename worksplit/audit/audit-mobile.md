# Mobile App Audit — mytheus-mobile/

Read-only audit. Root: /Users/sushan/Projects/Mytheus. Target: `mytheus-mobile/` (Expo/React Native).
Last commit: `77290c364` — "feat: Clerk auth and the org_id-aware read path against bff" (2026-07-30 09:28:03 +0530), ~5 weeks older than the rest of the monorepo.

## SCREEN INVENTORY — screen | real/mock | endpoints called

- `app/(tabs)/index.tsx` (Home) — **REAL** (dashboard arm) + mock (weeklyStats banner) | `GET /bff/dashboard`
  - Uses `useDashboard()` (`src/api/hooks/index.ts:15-21`), renders `upcoming_meetings` arm with proper `ok`/`degraded`/`empty` states (`app/(tabs)/index.tsx:157-177`).
  - `weeklyStats` (meetings/decisions/actions counts banner) still comes from `data/mock.ts` — documented in a comment (`app/(tabs)/index.tsx:17-19`) as intentional: the decisions/action-item read models don't exist yet server-side (Plan 7-9), so faking them from real-but-empty arms would show "0 decisions" as a fabricated fact.
- `app/(tabs)/actions.tsx` — **MOCK** (`data/mock.ts: allActions`) | no endpoint calls
- `app/(tabs)/ask.tsx` — **MOCK** (multiple mock imports) | no endpoint calls
- `app/(tabs)/library.tsx` — **MOCK** (`librarySessions`, `libraryFilters`) | no endpoint calls
- `app/(tabs)/people.tsx` — **MOCK** (`commitmentStats`, `getPersonCommitments`, `people`) | no endpoint calls
  - Notable: `useTeams()` and `useDepartments()` (`src/api/hooks/index.ts:41-54`) exist, are wired to real, currently-live backend routes, and are **never imported or called from any screen** — confirmed via `grep -rn "useTeams\|useDepartments" app/ src/` returning only their own definitions. Dead code, not a wiring bug per se, but a built-and-abandoned integration.
- `app/recap.tsx` — **MOCK**
- `app/insights.tsx` — **MOCK**
- `app/live-session.tsx` — **MOCK**. Imports `liveTranscript`, `keyPoints`, `decisionText`, `actionItems`, `session` from `data/mock.ts` (`app/live-session.tsx:17-24`). The "live caption" effect is a scripted timeline driven by a 50ms tick that advances through a canned transcript array (`app/live-session.tsx:34-40` comment: "Scripted live-caption timeline... only advances while the room is being listened to").
- `app/start-session.tsx` — **MOCK** (`session` from `data/mock.ts`)
- `app/person/[id].tsx` — **MOCK**
- `app/profile.tsx` — **MOCK** (`allMeetings`, `currentUser`, `weeklyStats`)
- `app/sign-in.tsx` — **REAL**. Real Clerk `useSSO({strategy: 'oauth_google'})` flow with `Linking.createURL` redirect, no sign-up path (workspaces are company-provisioned), generic error message deliberately avoiding account-enumeration oracle.
- `app/_layout.tsx`, `app/(tabs)/_layout.tsx` — navigation shell, `ClerkProvider` + `ApiProvider` wiring, font loading, splash screen. No data.

**Net: 1 of 13 screens (Home) reads live backend data. Sign-in is real. Every other screen (12/13) is fully scripted mock data from `src/data/mock.ts`.**

## BACKEND CONTRACT DRIFT — none found

The mobile client only knows 4 endpoints total (via `src/api/hooks/index.ts`), so drift surface is small, but all 4 were cross-checked against current backend source and are unchanged:

| Client call | Defined at | Status |
|---|---|---|
| `GET /bff/dashboard` | `mytheus-backend/services/bff/dashboard.py:152` (`router = APIRouter(prefix="/bff")` at line 46) | Exists. Response model (`Dashboard`/`Arm<T>`/`DashboardMeeting`) matches `src/api/types.ts:14-34` field-for-field (`upcoming_meetings`, `action_items`, `worklogs`, `decisions`, each `{status, items}`). |
| `GET /api/v1/meetings` | `services/core_api/routes/meetings.py:243` (`router = APIRouter(prefix="/api/v1")` at line 49) | Exists. |
| `GET /api/v1/teams` | `services/core_api/routes/org_hierarchy.py:37` (`router = APIRouter(prefix="/api/v1")` at line 34) | Exists (unused by any screen — see above). |
| `GET /api/v1/departments` | `org_hierarchy.py:91` | Exists (unused by any screen). |

`services/api_gateway/edge.py:147-154` `ROUTE_MAP` still routes `/bff` → `bff`, `/api/v1/meetings` → `core-api`, `/api/v1/teams` → `core-api`, `/api/v1/departments` → `core-api`. Longest-prefix matching logic (`edge.py:184-188`) is unchanged and still keeps `/api/v1/meetings` (core-api CRUD) distinct from any bff aggregate.

One stale-in-spirit note: `mytheus-backend/services/bff/main.py:1-6` docstring says "The only caller of `/bff/*` is `mytheus-ui`." Mobile now also calls `/bff/dashboard`, so that comment is no longer accurate, but it's not a functional break — just documentation that wasn't updated when mobile started consuming the same aggregate.

No calls found anywhere in `app/` or `src/` to any route that has since changed or been removed. There is nothing else to drift because nothing else is wired.

## AUTH STATE — real, not mocked

- `src/lib/session.ts` — thin wrapper over `@clerk/clerk-expo`'s `useAuth`. Comment states this used to be "a module-level boolean with a listener set" (mock) and now delegates entirely to Clerk. The old no-argument `signIn()` free function was deliberately deleted rather than adapted — rationale given in the file: a no-arg sign-in can't be implemented honestly (needs a strategy + browser session), and a working-looking stub would let a screen "appear authenticated while holding no token."
- `useSignedIn()` (`src/lib/session.ts`) returns `false` while Clerk is still loading (not just "not signed in"), avoiding a flash of authenticated UI before the real state is known.
- `app/sign-in.tsx` — genuine `useSSO` OAuth flow, not a stub button.
- Tokens: `src/lib/secure-store.ts` wires Clerk's `TokenCache` (`getToken`/`saveToken`/`clearToken`) to `expo-secure-store` (iOS Keychain / Android Keystore) instead of Clerk's AsyncStorage default. Every operation swallows errors and degrades to "no cached token" rather than crashing. This matches the stated invariant in the repo's CLAUDE.md ("expo-secure-store only").
- `src/api/provider.tsx` — pulls a fresh token via `useAuth().getToken()` on every request (not captured once), correctly handling Clerk's short-lived token rotation.
- `org_id` handling: **never sent by the client.** `src/api/client.ts`'s `RequestOptions` type and `createApiClient` function have no parameter through which a tenant id could be passed — this is a structural guarantee, not just a convention. `__tests__/api-client.test.ts` includes an explicit assertion that no request's URL, headers, or body contains the org_id string. Server-side, the gateway derives tenant from the verified JWT (per code comments in `client.ts` and `meetings.py`). This was verified in code, not just claimed in a comment.

## PARITY GAP vs WEB

Web (`mytheus-ui/src/app/(app)/*`) route groups found: `dashboard`, `meetings`, `meetings/[meetingId]`, `decisions`, `tasks`, `org`, `search`, `settings` (+ `settings/integrations`, `settings/profile`), `billing`, plus a separate `(admin)/admin` and `(admin)/ops` area, and a full `(marketing)` site (features/*, pricing, about, etc.) and `(auth)` sign-in/sign-up.

Mobile has screen *shapes* that echo some of these (home/dashboard, people≈org, recap≈meeting detail, actions≈tasks) but:
- No dedicated meetings-list/detail screen backed by real data (the one meetings-page equivalent — Library — is 100% mock).
- No decisions screen with real data.
- No tasks/action-items screen with real data (Actions tab is 100% mock).
- No search screen at all on mobile.
- No settings/integrations/billing on mobile.
- No org-chart screen wired to data (despite `/bff/org/chart` existing server-side per `services/bff/org_chart.py:169` — not even attempted from mobile).

Effectively, once you set aside "shows a list of upcoming meetings" and "sign in," mobile has no working parity with any web feature area.

## MOBILE-ONLY CAPABILITY (audio capture) — claimed but not implemented

- `app.json:17` (Android) declares `"permissions": ["RECORD_AUDIO"]`.
- `app.json` iOS `infoPlist.NSMicrophoneUsageDescription` (line ~14): *"Mytheus listens to your in-person meeting on this device to transcribe and summarize it. Audio stays on the device until you finish a session."* — a specific, load-bearing data-handling claim.
- No audio-capture implementation exists anywhere in the app to back this:
  - `package.json` has no `expo-av`, no `expo-audio`, no LiveKit client SDK, nothing audio-related beyond the OS permission declarations.
  - `grep -n "expo-av\|Audio\.\|Recording\|expo-audio\|LiveKit\|livekit" app/live-session.tsx app/start-session.tsx package.json` returned zero real hits (only cosmetic string matches like "Recording" as a UI label).
  - `src/components/Waveform.tsx:1-40` is a purely decorative `Animated.Value` bar-pulse loop — not driven by any audio signal.
  - `app/live-session.tsx:17-24, 34-40` — the "live transcript" is the `liveTranscript` mock array played back on a 50ms setInterval-style tick, not real-time transcription of anything.
- **Conclusion**: mobile does not do anything the web UI doesn't. The one feature that would differentiate it (in-person audio capture) is UI theater sitting on top of a real, user-facing OS permission grant with a specific privacy claim attached. This is UNVERIFIED as to intent — code comments elsewhere in the repo (Plan 3-4 / "realtime lands with Plan 3-4 and LiveKit with the media thread" per `.env.example`) suggest this is a deliberately unbuilt future band rather than an attempt to deceive — but as currently committed, granting RECORD_AUDIO with that specific usage-description string, while shipping zero recording code, is worth flagging as a compliance/App-Store-review/privacy-honesty risk if this build were ever submitted as-is.

## BUILD/RELEASE STATE — not shippable

- `mytheus-0.1.0.apk` (83,717,114 bytes ≈ 83.7 MB) is committed directly in the repo at `mytheus-mobile/mytheus-0.1.0.apk` — a binary build artifact in source control, not in an artifact store/CI release.
- No `eas.json` anywhere in `mytheus-mobile/` — there is no EAS build/submit configuration at all, despite `expo` and `expo-router` being the project's whole toolchain (EAS is the standard Expo release path).
- `android/` is a bare native project: `gradlew`, `build.gradle`, `settings.gradle`, `proguard-rules.pro`, and **`android/app/debug.keystore` is committed to git**. A debug signing key in source control isn't a leaked production secret, but it's normally gitignored and its presence suggests the APK was built locally (`expo run:android` / `gradlew assembleDebug`), not through a controlled release pipeline.
- `dist/` contains a Metro/Expo export bundle (`dist/metadata.json`, `dist/assets/*`, `dist/_expo/static/js/android/entry-0d3bf11ead0f39c1fffb34a6420299d1.hbc`) — consistent with `expo export`/local build output, not a signed release build.
- No mobile-specific CI configuration was found in a targeted pass (UNVERIFIED — did not exhaustively search the whole monorepo's CI directories for a mobile job).

**Verdict on this section**: this is a locally-built development/debug APK with no reproducible, auditable release pipeline. Even setting aside the mock-data problem, there is no path from "this repo" to "an app I'd hand to a real user" without first standing up EAS (or an equivalent) and stopping committing the APK/keystore.

## TESTS — real, but narrow

- `__tests__/api-client.test.ts` — 222 lines, 14 `it`/`test` blocks (matches the commit message's "14 jest tests pass" claim).
- These are genuine behavioral tests, not smoke tests. Confirmed coverage includes:
  - Asserting `org_id` never appears in a request's URL, headers, or body.
  - Idempotency-Key generation on mutating requests, and honoring a caller-supplied key on retry (dedup semantics).
  - RFC 7807 `application/problem+json` parsing into the typed `ErrorCode` enum, with a sane fallback when the body isn't JSON.
  - Retry policy: no retry on auth failures (`AUTH_REQUIRED`/`TOKEN_EXPIRED`) or non-retryable codes, retry allowed (up to 2x) on `DEPENDENCY_UNAVAILABLE`/`RATE_LIMITED`.
  - Query-string building drops `undefined` values instead of serializing the literal string `"undefined"`.
  - Missing `EXPO_PUBLIC_API_GATEWAY_URL` throws immediately rather than silently falling back to a default host.
- Coverage is entirely limited to `src/api/client.ts`. There are **zero** component tests, zero screen tests, zero tests of `src/api/hooks/index.ts` (including the two unused `useTeams`/`useDepartments` hooks), and zero tests exercising any of the 12 mock-data screens.
- The commit message notes plain `jest` (not `jest-expo`) was chosen because `jest-expo` requires React 19.2.3 and the app pins React 19.1.0 — and that the security-relevant layer (`client.ts`) imports nothing from React Native, so it's "fully covered" by plain jest. That's an accurate characterization of what's tested; it does mean no React Native component ever gets test-executed in this repo today.

## VERDICT

Mobile is a real, security-conscious client for exactly one feature — viewing upcoming meetings — plus genuine Clerk authentication; the plumbing that does exist (org_id tenant isolation enforced structurally and by test, secure token storage in the OS keychain, RFC 7807 typed error handling, mutation idempotency) is built carefully and has not drifted from the current backend contract despite being 5 weeks stale. But 12 of the 13 screens are entirely scripted mock data, two already-built and currently-functional backend integration hooks (teams/departments) sit completely unused, and the app's one claimed differentiator versus the web UI — in-person audio capture — has a real, user-facing OS permission grant and privacy claim backing zero actual recording or transcription code. Combined with a committed release APK, a committed debug keystore, and no EAS/CI release configuration at all, this reads as a stale early-stage prototype shell with one live wire into the backend, not a shippable or feature-complete mobile client.
