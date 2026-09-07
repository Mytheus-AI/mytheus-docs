# `mytheus-mobile` — full pending-work enumeration

Read-only verification, 2026-09-07. Nothing in any repo was modified.
Root: `/Users/sushan/Projects/Mytheus`. Target: `/Users/sushan/Projects/Mytheus/mytheus-mobile`.
Backend cross-checked against current source in `/Users/sushan/Projects/Mytheus/mytheus-backend`,
`/Users/sushan/Projects/Mytheus/mytheus-realtime`, `/Users/sushan/Projects/Mytheus/mytheus-intelligence`.

Baseline health, measured (not inherited):

- `npx tsc --noEmit` → exit 0, clean.
- `npx jest` → **14 passed, 14 total**, 1 suite, 2.34s.
- `git log -1` → `77290c3 feat: Clerk auth and the org_id-aware read path against bff`.
- Working tree clean except one untracked file (see C1).

---

## 0. Corrections to the brief

Three premises in the task as given are wrong, and each changes the work. Stating them first
because two of them delete a task and one replaces it with a more serious one.

**0.1 — The 84MB APK is NOT committed. It is untracked.**
`git ls-files` (66 tracked files) does not list it. `git status --short` reports `?? mytheus-0.1.0.apk`.
It is 83,717,114 bytes, dated 2026-07-30. `.gitignore:1-75` has no rule that covers it, so it is
**one `git add .` away from entering history** — which is a real task (C1), but a much cheaper one
than excising a blob from history.

**0.2 — `android/app/debug.keystore` is NOT committed either.**
`git ls-files | grep -i keystore` → empty. `mytheus-mobile/.gitignore:19-28` ignores `/android`
wholesale (line 20), so the `!debug.keystore` negation on line 28 is dead — a negation cannot
resurrect a file under an ignored directory. The whole `android/` tree on disk is prebuild output,
untracked and regenerable. There is no committed key material to rotate.

**0.3 — The real release defect is worse than a committed keystore, and it is in a generated file.**
`mytheus-mobile/android/app/build.gradle:112-115`:

```
release {
    // Caution! In production, you need to generate your own keystore file.
    signingConfig signingConfigs.debug
}
```

The **release** build type signs with `signingConfigs.debug` (`build.gradle:100-107`: `storeFile
file('debug.keystore')`, `storePassword 'android'`, `keyAlias 'androiddebugkey'`). That is the
universally-known Android debug key. The 84MB APK on disk was produced by this configuration, so
it is debug-signed. Google Play rejects debug-signed uploads outright; anyone can re-sign and
impersonate the app. Because `android/` is generated and gitignored, the fix does not belong in
that file — it belongs in `eas.json` + EAS-managed credentials (C2).

**0.4 — Screen count.** There are **14 route files** under `app/`, of which **12 are screens** and
2 are layouts. The brief says 13. Full enumeration in §2 so nothing is lost to the discrepancy.

---

## 1. Backend contract verification — is there still zero drift?

**Yes. Zero drift. Every endpoint the client calls exists, is reachable through the gateway, and
its response model matches `src/api/types.ts` field-for-field.** Verified below, not assumed.

The client calls exactly four endpoints. All four are in `mytheus-backend/services/api_gateway/edge.py:146-181`
(`ROUTE_MAP`), resolved by longest-prefix match (`edge.py:184-189`).

| Client call | Site | Gateway route | Backend handler | Verdict |
|---|---|---|---|---|
| `GET /bff/dashboard` | `src/api/hooks/index.ts:19` | `edge.py:147` → `bff` | `services/bff/dashboard.py:152` | **match** |
| `GET /api/v1/meetings?status&limit=50` | `src/api/hooks/index.ts:30` | `edge.py:148` → `core-api` | `services/core_api/routes/meetings.py:243` | **match** |
| `GET /api/v1/teams` | `src/api/hooks/index.ts:44` | `edge.py:149` → `core-api` | `services/core_api/routes/org_hierarchy.py:37` | **match** |
| `GET /api/v1/departments` | `src/api/hooks/index.ts:52` | `edge.py:150` → `core-api` | `services/core_api/routes/org_hierarchy.py:91` | **match** |

Field-level checks:

- `Dashboard` — `types.ts:31-36` (`upcoming_meetings`/`action_items`/`worklogs`/`decisions`) vs
  `dashboard.py:113-118`. Identical. `Arm<T>` (`types.ts:15-18`: `status`, `items`) vs `_ArmBase`
  (`dashboard.py:86-88`). Identical. `ArmStatus` (`types.ts:13`: `'ok' | 'degraded' | 'empty'`) vs
  `dashboard.py:53`. Identical. `DashboardMeeting` (`types.ts:22-29`, 6 fields) vs `dashboard.py:56-70`,
  6 fields. Identical, same nullability.
- `Meeting` — `types.ts:43-54` (10 fields incl. `participants`) vs
  `services/core_api/schemas.py:87-97`. Identical. `Participant` (`types.ts:38-41`) vs `schemas.py:59-61`.
  Identical. Note the client correctly consumes `Participant`, **not** `ParticipantCheck`
  (`schemas.py:64-84`), which is the only shape carrying `org_id` — and is a different resource.
- `MeetingPage` — `types.ts:57-60` vs `schemas.py:100-109`. Identical.
- `Team` — `types.ts:62-67` vs `schemas.py:149-153`. Identical. `Department` — `types.ts:69-74` vs
  `schemas.py:179-183`. Identical.
- `limit: 50` (`hooks/index.ts:30`) is within `Query(ge=1, le=MAX_LIMIT)` where `MAX_LIMIT = 200`
  (`meetings.py:53, 247`). Valid.
- `status` is sent as the query name `status`; the handler declares
  `status_filter: Annotated[str | None, Query(alias="status")]` (`meetings.py:249`). Correct.
- All four handlers gate on `require_tier("member")` — the lowest tier. No role mismatch.
- `ErrorCode` (`src/api/client.ts:21-36`, 15 values) — the client's own comment says it is
  hand-synced with `libs/shared/errors.py`. Spot-checked values all present. The generated-client
  gap that would remove the hand-sync is docs gap 5's sibling and is task B7.

**Contract drift that exists but is documentation-only, not code:** `mytheus-mobile/CLAUDE.md`
(Architecture orientation section) names five surfaces that do not exist in the backend:
`GET /bff/meeting-room/:id` (bff has only `/bff/dashboard`, `/bff/people/{user_id}`, `/bff/org/chart`
— `services/bff/main.py:11-13`); `/api/v1/actions`, `/api/v1/decisions`, `/api/v1/worklogs`,
`POST /api/v1/decisions/:id/confirm` (all four prefixes are in `ROUTE_MAP` at `edge.py:151-153` but
`core-api` registers no such router — `services/core_api/main.py:16-20` registers only meetings,
org_hierarchy, chat, speakers, transcript; a call would 404 at core-api). It also names the token
mint as `POST /api/v1/meetings/:id/token`; the real route is
`POST /api/meetings/{meeting_id}/token` (`services/meeting_service/routes.py:153-157`, gateway
prefix `/api/meetings` at `edge.py:161`). **No client code calls any of these**, so nothing is
broken today — but the file is the spec a new owner will build against, so it is task B8.

---

## 2. Screen-by-screen: mock vs real, and the endpoint each should call

Data-source scan: `grep -rn "data/mock\|useDashboard\|useMeetings\|useTeams\|useDepartments" app/`.

| # | Screen | File | Lines | Data today | Endpoint it SHOULD call | Backend exists? |
|---|---|---|---|---|---|---|
| 1 | Home | `app/(tabs)/index.tsx` | 283 | **PARTIAL REAL** — `useDashboard()` at :69; `weeklyStats` mock at :20,:137 | `GET /bff/dashboard` (done). Banner needs decisions + action-item counts | **Partly.** Dashboard: yes. `action_items`/`decisions` arms are `UnbuiltArm`, always `empty` (`dashboard.py:94-110`, `:186-193`). **No read model exists** |
| 2 | Ask | `app/(tabs)/ask.tsx` | 714 | **MOCK** — `askAnswers`/`askFallbackAnswer` at :33-34, canned via `setTimeout` at :131,:155; fake attachment picker at :59-60,:104-109 | An agent/RAG query endpoint | **NO BACKEND.** `mytheus-intelligence` has `GET /search` (`services/search_service/routes.py:86`) but it is **not in `ROUTE_MAP`** — unreachable from a client. No agent surface anywhere. `POST /api/v1/chat` (`core_api/routes/chat.py:88`) is meeting-scoped participant chat, not Ask |
| 3 | Actions | `app/(tabs)/actions.tsx` | 147 | **MOCK** — `allActions` at :10 | `GET /api/v1/actions` | **NO BACKEND.** Prefix mapped (`edge.py:153`), no router. Plan 7-9 |
| 4 | People | `app/(tabs)/people.tsx` | 127 | **MOCK** — `people`, `getPersonCommitments`, `commitmentStats` at :13 | `GET /bff/org/chart` | **YES — and this is the finding in §3** |
| 5 | Library | `app/(tabs)/library.tsx` | 96 | **MOCK** — `librarySessions` at :12, via `fromMockMeeting` at :67 | `GET /api/v1/meetings?status=ended` + `next_cursor` paging | **YES** (`meetings.py:243`). The `durationMin`/`speakers`/`chips` the card shows have no backend (Plans 3-4, 7-9) |
| 6 | Person detail | `app/person/[id].tsx` | 312 | **MOCK** — `getPerson`/`getPersonCommitments`/`getPersonQuotes`/`getPersonMeetings` at :15-20 | `GET /bff/people/{user_id}` | **YES** (`services/bff/people.py:67`, returns `role`, `skills`, `bio`, `avatar_url`, `team`, `department`, `recent_meetings`, `topics`, `email`). Quotes and commitments have **no backend** |
| 7 | Recap | `app/recap.tsx` | 218 | **MOCK** — `actionItems`, `decisionText`, `overviewText`, `fullTranscript` at :14-19. Title hardcoded at :50-51 | `GET /api/v1/meetings/{id}` + `GET /api/v1/meetings/{id}/transcript` | **Transcript: YES** (`core_api/routes/transcript.py:95`, cursor + `from_ms` seek). **Summary/decisions/actions: NO BACKEND** (Plan 7-9) |
| 8 | Live session | `app/live-session.tsx` | 462 | **MOCK, scripted** — `liveTranscript`/`keyPoints`/`decisionText`/`actionItems` at :18-25; a 50ms character-typing script at :39-94,:119-130; elapsed seeded to 767s at :100 | `POST /api/meetings/{id}/token` → LiveKit; Socket.io `/v1` for captions | **Transport: YES.** Token mint at `meeting_service/routes.py:153`; `realtime-gateway` is real Socket.io `/v1` with `join`/`chat`/`presence`/caption relay (`mytheus-realtime/services/realtime-gateway/src/server.ts:213,337,411`). **Captions: NO.** `transcription-service` `/internal/audio` accepts then immediately closes (`main.py:119-124`) — a stub |
| 9 | Start session | `app/start-session.tsx` | 191 | **MOCK** — `session` at :14. "3 voices nearby" hardcoded at :80 | `POST /api/v1/meetings` | **YES** (`meetings.py:153`) |
| 10 | Insights | `app/insights.tsx` | 446 | **MOCK** — `weeklyStats`, `weeklyActivity`, `topCollaborators`, `weekHighlights`, `commitmentStats` at :15-21 | An analytics/aggregate read | **NO BACKEND.** Nothing aggregates per-user weekly activity |
| 11 | Profile | `app/profile.tsx` | 259 | **MOCK identity** — `currentUser`, `allMeetings`, `weeklyStats` at :20, rendered at :74-88. Sign-out IS real (`useSessionActions` :43, Clerk) | Clerk `useUser()` for identity; `PATCH /users/{user_id}` for edits | **Identity: YES via Clerk, already in the app** (`app/(tabs)/index.tsx:68` proves it). Server profile: `PATCH /users/{user_id}` exists (`identity_tenant_service/routes/users.py:73`) but see §4 blocker |
| 12 | Sign-in | `app/sign-in.tsx` | 132 | **REAL** — Clerk `useSSO` + `oauth_google` at :25,:34-42 | — | Done |
| — | Root layout | `app/_layout.tsx` | 94 | **REAL** — ClerkProvider + ApiProvider :65-66 | — | Done |
| — | Tabs layout | `app/(tabs)/_layout.tsx` | 26 | **REAL** — auth gate :8-9 | — | Done, but has defect A1 |

Totals: **1 screen real** (sign-in), **1 partially real** (home), **10 fully mock**. `src/data/mock.ts`
is 472 lines and is imported by 9 screens.

---

## 3. `useTeams()` / `useDepartments()` — wired, working, consumed by nobody

`src/api/hooks/index.ts:41-47` and `:49-55`. Both hit real, matching routes (§1). `grep -rn
"useTeams\|useDepartments" app/` → **zero hits**. They are dead code today.

**The comment above `useTeams()` is now false**, and this is the substantive finding. `hooks/index.ts:36-39`:

> *"Teams, not an org-member roster: `identity-tenant-service` has no `GET /users` surface yet, so
> there is nothing to read a member list from."*

That was true when written (it restates the deviation recorded in
`mytheus-docs/work-done-till-now/02-plan-1-2-foundation-and-tenancy.md:14.2`). It is not true now.
`GET /bff/org/chart` (`services/bff/org_chart.py:169`) returns the full nested hierarchy:
`DepartmentNode { id, name, teams, children }` → `TeamNode { id, name, members }` →
`Member { user_id, email, role, role_in_team }` (`org_chart.py:77-115`), plus `unassigned_teams`,
under `require_tier("member")`, with a `status` field for degradation. **That is a real org-member
roster, delivered in one call, and it is in `ROUTE_MAP` via the `/bff` prefix.**

So:

- **The People screen (#4) should consume `GET /bff/org/chart`, not `useTeams()`.** One call replaces
  the mock `people` array with real members carrying `email` and `role`. What it will *not* supply is
  the "you owe / owes you" commitment model the screen is built around (`people.tsx:22-44`) — that
  needs action items, which have no backend. So the screen migrates to real people with the
  commitment filters removed or shown as unavailable, not silently zeroed.
- **`useTeams()` / `useDepartments()` have exactly one honest consumer today: the People screen's
  filter/grouping affordance** (group the roster by department, filter by team). They are the
  supporting reads, not the primary one.
- If neither is used after the org-chart migration, delete them rather than leave them dead —
  `org/chart` already embeds team and department names.

---

## 4. Backend gaps that block mobile work — stated, not invented

1. **No route returns the caller's own internal `user_id`.** There is no `GET /users/me` and no
   `me` alias. `identity_tenant_service/routes/users.py` exposes only `PATCH /users/{user_id}`
   (:73). `GET /context/{user_id}` (`routes/context.py:32`) and `GET /bff/people/{user_id}`
   (`bff/people.py:67`) both take an internal UUID. The client holds a **Clerk** user id, and
   `RequestContext.user_id` is resolved server-side but never returned. Consequence: **the app
   cannot open its own profile, cannot filter "assigned to me", and cannot address itself in the
   org chart.** This blocks part of tasks B2 and B4. Workaround inside the org-chart response:
   match on `email` against Clerk's `primaryEmailAddress` — usable, but a heuristic, and it should
   be flagged to the backend owner rather than adopted silently.
2. **No agent / Ask surface anywhere.** Blocks the Ask screen entirely (Plan 7-9).
3. **No action-items, decisions, worklogs read models.** The three `bff` arms are `UnbuiltArm` and
   deliberately always `empty` (`dashboard.py:55-71`). Blocks Actions, the home banner, Insights,
   and the commitment half of People and Person detail.
4. **No live captions.** `transcription-service` `/internal/audio` is a stub. Blocks the live
   transcript in #8 even after LiveKit is wired.
5. **No analytics aggregate.** Blocks Insights.
6. **Push:** `notification-service` exists but registering an Expo device token is flagged in
   `mytheus-mobile/CLAUDE.md` as net-new backend surface (N7). Not built on either side.

---

## 5. `RECORD_AUDIO` / `NSMicrophoneUsageDescription` — the precise situation

**Facts.**

- `app.json:20-22` requests `"RECORD_AUDIO"` for Android.
- `app.json:14-16` declares `NSMicrophoneUsageDescription`: *"Mytheus listens to your in-person
  meeting on this device to transcribe and summarize it. Audio stays on the device until you finish
  a session."*
- `android/app/src/main/AndroidManifest.xml:4` carries `android.permission.RECORD_AUDIO` in the
  built artifact.
- **There is no audio code.** `package.json:15-41` has no `expo-av`, no `expo-audio`, no
  `@livekit/react-native`, no `react-native-audio-record`. `grep -in "audio\|microphone\|record"`
  across `src/` matches only the `RecordButton` component name. `RecordButton`'s only behaviour is
  `router.push('/live-session')` (`app/start-session.tsx:65`), and `Waveform` is a decorative
  animation over a fixed height array.
- The user-facing privacy copy repeats the claim: `app/start-session.tsx:104-106` renders *"Audio
  stays on this device until you finish."*
- The manifest also carries `SYSTEM_ALERT_WINDOW`, `READ_EXTERNAL_STORAGE` and
  `WRITE_EXTERNAL_STORAGE` (`AndroidManifest.xml:3,5,6`). These are Expo autolinking defaults, not
  deliberate, and `SYSTEM_ALERT_WINDOW` (draw-over-other-apps) is itself a review-sensitive
  permission on an app that never draws an overlay.

**The two options, and what each costs.**

**Option A — remove the permission and the claim until capture is implemented. (Recommended.)**
Delete `"permissions": ["RECORD_AUDIO"]` from `app.json:20-22` and the `NSMicrophoneUsageDescription`
block at `:14-16`; soften the on-screen copy at `start-session.tsx:104-106`. Cost: **S, under an
hour, zero functional loss** — nothing reads the microphone, so nothing regresses. Add
`android.permission.SYSTEM_ALERT_WINDOW` etc. to a `blockedPermissions` list in the same pass if
the team wants a minimal manifest. Reversible: the permission returns in the same commit that adds
capture, which is where the contextual-request rule in `CLAUDE.md` ("request contextually — mic at
meeting join, never all at launch") says it belongs anyway.

**Option B — implement capture now.** Add `expo-audio` (or the LiveKit RN SDK, which is what
`CLAUDE.md` actually specifies for the media plane), a runtime permission request at session start,
recording lifecycle, and a destination for the bytes. Cost: **L, and it is blocked** — the
destination does not exist. LiveKit ingest requires `POST /api/meetings/{id}/token` plus a meeting
id the app never creates, and `transcription-service` closes the socket immediately (§4.4). Building
capture now produces audio with nowhere to go, and would make the privacy claim *more* wrong, not
less, because audio would then genuinely exist on the device with no defined retention.

**App-store review risk, stated factually and without overstatement.** Both stores require that a
declared permission be used by the app, and iOS requires the purpose string to describe actual
behaviour. Apple's review guideline 5.1.1 covers data-collection purpose strings; Google Play's
permissions policy requires declared permissions to be necessary for implemented features. An app
that declares microphone access, states in the system prompt that it transcribes meetings, and
contains no code that opens the microphone is a plausible rejection under both — the purpose string
describes a feature that does not exist. This is a **factual policy exposure, not a certainty**:
reviewers do not always catch unused permissions, and neither store has been engaged here. Note
also that no iOS build has ever been produced (no `ios/` directory, no EAS config), so the iOS
purpose string has never been reviewed by anyone. The exposure is cheap to remove (Option A) and
expensive to keep.

**Verdict: Option A is unambiguously safe to do now.** It is not blocked on the Plan 5-6 §7 scoping
decision, it removes a false statement from user-facing copy, and it costs nothing.

---

## 6. Build and release — what a reproducible signed release needs

Current state, verified:

- No `eas.json` (`ls eas.json` → no such file). `mytheus-mobile/CLAUDE.md` lists it in the intended
  tree; it was never written.
- No `.github/` in `mytheus-mobile` (`ls .github` → no such file). No mobile job exists in
  `mytheus-platform/.github/workflows/ci.yml` or `deploy.yml` either — those are backend/compose.
  **There is zero mobile CI**: `tsc`, `eslint` and `jest` all pass, and nothing runs them.
- Release signs with the debug key (§0.3).
- `android/app/build.gradle:95-96`: `versionCode 1`, `versionName "0.1.0"` — generated from
  `app.json:5`. No versionCode strategy, no `ios.buildNumber`.
- `app.json` has no `extra.eas.projectId`, no `owner`, no `runtimeVersion`, and
  `expo.updates` is disabled in the manifest (`AndroidManifest.xml:19`) — so no OTA channel exists.
- `android/` is prebuild output and gitignored (`.gitignore:19-20`); the checked-in tree is
  managed-workflow. That is the correct posture and should be preserved.
- No iOS build has ever existed.

What is needed, in order: `eas.json` with `development`/`preview`/`production` profiles; EAS-managed
release credentials (an upload keystore generated and held by EAS, never in the repo, which also
makes §0.3 moot without editing generated Gradle); an explicit versioning policy (`autoIncrement`
or `remote` versionSource) so two builds are never the same versionCode; `extra.eas.projectId` and
`owner` in `app.json`; a CI workflow running `typecheck`/`lint`/`test` on every push plus an
`eas build --profile preview` on tags; and `EXPO_PUBLIC_*` values supplied per profile rather than
from a developer `.env`.

---

## 7. Test coverage — what exists and what is missing

`__tests__/api-client.test.ts` is genuine and well-targeted: 14 tests across bearer injection (3),
`org_id` never sent (1), idempotency (3), RFC 7807 parsing (5), URL building (2). It covers
`src/api/client.ts` and nothing else.

`jest.config.js:11-18` uses the bare `ts-jest` preset with `testEnvironment: 'node'` and
`testMatch: ['**/__tests__/**/*.test.ts']` — note `.ts` only, so a `.tsx` test file would not even
be collected. The header comment (`jest.config.js:1-9`) explains the deliberate choice: `jest-expo`
needs React 19.2.3 and the app is on 19.1.0 (`package.json:33`).

Untested, in priority order: `src/api/present.ts` (**highest value and testable today — pure TS, no
React**: `relativeWhen()` at :39-54 has day-boundary, invalid-date and null branches, and
`toMeetingCard()` at :56-65 has the title-null default); the four hooks in `src/api/hooks/index.ts`
(query keys, the `status`/`limit` params actually sent, error passthrough); `src/api/provider.tsx`'s
retry predicate (`:24-29` — that auth failures are not retried is a security-relevant behaviour and
is currently unverified); `src/lib/secure-store.ts` (all three swallow-and-degrade paths, :22-47);
`src/lib/session.ts`'s loading-state semantics; and every component and screen (0% — needs the
`jest-expo` + RNTL unblock, which needs the React version to line up).

---

## 8. Things the audit missed

**A1 — Auth-gate flash on cold start. Real defect, cheap fix.**
`app/(tabs)/_layout.tsx:8-9` calls `useSignedIn()` and redirects to `/sign-in` when it is false.
`useSignedIn()` (`src/lib/session.ts:27-30`) returns `isLoaded && Boolean(isSignedIn)` — **false
while Clerk is still restoring the cached session.** So an already-signed-in user is bounced to the
sign-in screen on every cold start until Clerk finishes loading. `useSessionLoaded()` exists at
`session.ts:34-36` precisely for this, with a docstring saying screens should use it to hold the
splash — and `grep -rn "useSessionLoaded" app/ src/` shows it is **imported by nothing**. The fix
is to hold rather than redirect while `!isLoaded`.

**A2 — Profile screen shows a fabricated identity to a really-signed-in user.**
`app/profile.tsx:74-88` renders `currentUser.initials`, `.name`, `.role`, `.org`, `.email` from
`src/data/mock.ts:369`. The user is genuinely authenticated at that point and Clerk has the real
values — `app/(tabs)/index.tsx:68` already uses `useUser()` for exactly this. Showing a fictional
name and email on the profile of a signed-in user is the clearest instance of the
"never invent durable truth client-side" invariant being broken, and the data to fix it is already
in the app. Independent of every plan band.

**A3 — Recap takes no meeting id.**
`app/recap.tsx` has no `useLocalSearchParams`; the title is the literal `"Q3 Roadmap Sync"` at :50
and the meta `"Today · 24 min · 3 speakers"` at :51. Both `app/(tabs)/index.tsx:166` and
`app/(tabs)/library.tsx:66` navigate with a bare `router.push('/recap')`, dropping `m.id`. So the
one screen that already has a real backend behind it (`GET /api/v1/meetings/{id}`) cannot be wired
without a routing change first. This makes A3 a **prerequisite** for B5, not a cosmetic issue.

**A4 — Home always greets "Good afternoon".** `app/(tabs)/index.tsx:82`, hardcoded. Trivial.

**A5 — `REALIGN.md` was never written.** Plan 0 phase 0G asked for it; recorded as still missing in
`mytheus-docs/work-done-till-now/01-plan-0-scaffold-and-contracts.md:83` and `:126`. Still absent
(`ls REALIGN.md` → no such file). If the repo is being handed to one owner, this is the file that
tells them what is mock and why — much of §2 above is its content.

**A6 — `app/(tabs)/ask.tsx` is 714 lines**, the largest file in the repo, and every line of its
answer path is scaffolding around canned data (`:131` `askAnswers[q] ?? askFallbackAnswer`). When
the agent backend lands this is a rewrite, not a rewiring. Size accordingly.

**A7 — Fabricated facts still rendered as measured.** `present.ts:1-14` documents the rule carefully
and `toMeetingCard()` honours it — but `start-session.tsx:80` renders "3 voices nearby" with no
microphone; `live-session.tsx:100` seeds elapsed time to 767 seconds; `recap.tsx:51` states
"24 min · 3 speakers". These are the same class of invention the mapper refuses to commit. Worth a
single sweep.

**A8 — No `.eslintcache`/lint in CI, but `.eslintrc.js` and `expo lint` exist** and pass. Nothing
enforces them (see §6).

**A9 — Mock attachment picker in Ask.** `ask.tsx:58-60` fakes file names from `MOCK_IMAGES` /
`MOCK_DOCS`; its own comment says a real build wires `expo-image-picker`/`expo-document-picker`.
Neither is in `package.json`. Attachments also have no upload endpoint —
`mytheus-intelligence`'s `POST /ingest` (`services/ingestion_service/routes.py:78`) is not in
`ROUTE_MAP` and is not client-reachable.

---

## 9. The ordered task list

Sizes: **S** ≤ half a day · **M** ≈ 1-3 days · **L** ≥ 1 week.

### Group C — Safe now, no scoping decision needed, no backend dependency

Do these first. None of them touch the realign question.

| ID | Title | Files | Size | Depends on |
|---|---|---|---|---|
| **C1** | **Stop the 84MB APK from entering git.** Add `*.apk`/`*.aab` to `.gitignore` and move the artifact out of the tree. It is untracked today (§0.1) — this is prevention, not history surgery. | `.gitignore`, `mytheus-0.1.0.apk` | S | — |
| **C2** | **Remove `RECORD_AUDIO` + `NSMicrophoneUsageDescription` until capture exists.** Option A in §5. Also soften the "audio stays on this device" copy, and consider `blockedPermissions` for the autolinked `SYSTEM_ALERT_WINDOW` / storage permissions. | `app.json:14-22`, `app/start-session.tsx:104-106` | S | — |
| **C3** | **Fix the cold-start auth flash.** Hold on `useSessionLoaded()` instead of redirecting while Clerk is loading. Defect A1. | `app/(tabs)/_layout.tsx:8-9`, `src/lib/session.ts:34-36` | S | — |
| **C4** | **Profile shows the real signed-in user.** Replace `currentUser` with Clerk `useUser()`, as the home screen already does. Defect A2. Leave the mock-derived stats alone — they belong to B6. | `app/profile.tsx:20,74-88` | S | — |
| **C5** | **Pass the meeting id into Recap.** `router.push('/recap/[id]')` (or a query param) from home and library; read it with `useLocalSearchParams`. Behaviour-neutral on mock data, and it unblocks B5. Defect A3. | `app/recap.tsx`, `app/(tabs)/index.tsx:166`, `app/(tabs)/library.tsx:66` | S | — |
| **C6** | **`eas.json` + EAS-managed release credentials + versioning policy.** Fixes the debug-signed release (§0.3) without editing generated Gradle. Add `extra.eas.projectId`/`owner`. | `eas.json` (new), `app.json` | M | — |
| **C7** | **Mobile CI.** A workflow running `typecheck`, `lint`, `test` on push; `eas build --profile preview` on tag. All three gates pass today (§0) so this starts green. | `.github/workflows/ci.yml` (new) | S | C6 for the build step |
| **C8** | **Tests for `present.ts`.** Pure TS, no React, runs under the existing jest config unchanged. `relativeWhen()` boundaries + null/invalid dates; `toMeetingCard()` title default. §7. | `__tests__/present.test.ts` (new) | S | — |
| **C9** | **Tests for the hooks and the retry predicate.** Assert the params actually sent, the query keys, and — security-relevant — that auth failures are *not* retried (`provider.tsx:24-29`). Needs a light `@tanstack/react-query` harness; still no React Native import. | `__tests__/hooks.test.ts`, `__tests__/provider.test.ts` (new) | M | C8 |
| **C10** | **Tests for `secure-store.ts`.** All three swallow-and-degrade paths. | `__tests__/secure-store.test.ts` (new) | S | — |
| **C11** | **Sweep fabricated facts.** "3 voices nearby", the 767s seed, "24 min · 3 speakers", "Good afternoon". Remove or mark as sample. Defects A4, A7. | `app/start-session.tsx:80`, `app/live-session.tsx:100`, `app/recap.tsx:51`, `app/(tabs)/index.tsx:82` | S | — |
| **C12** | **Write `REALIGN.md`.** The Plan 0 0G deliverable, still missing. §2 of this document is most of its content. Highest leverage per hour for a single new owner. | `REALIGN.md` (new) | S | — |
| **C13** | **Correct `CLAUDE.md`'s endpoint list.** Five named surfaces do not exist and the token-mint path is wrong (§1). It is the spec a new owner builds from. | `mytheus-mobile/CLAUDE.md` | S | — |

### Group B — Blocked on the Plan 5-6 §7 scoping decision (status gap 21)

`mytheus-docs/work-done-till-now/00-status-overview.md:116` records gap 21: *"`mytheus-mobile`
section 7 (person-profile screen) was never assigned to a Plan 5-6 task and remains unbuilt...
Needs an explicit scoping decision before a future task picks it up."*
`plan/plan-5-6-org-context-and-ai-infra.md:610-630` is the section: step 2 (Clerk) already landed in
`77290c3`; steps 1, 3 and 4 (person profile from `GET /bff/people/{user_id}`, org context on the
header, de-mock the people path) did not. **Note the plan names the route `app/people/[id].tsx`;
the repo has `app/person/[id].tsx`** — the rename is part of the scoping decision, not a silent call.

Every task below has a real, verified backend behind it. **They are blocked by governance, not by
engineering.** B1-B3 are the actual Plan 5-6 §7 scope.

| ID | Title | Files | Size | Depends on |
|---|---|---|---|---|
| **B1** | **People screen onto `GET /bff/org/chart`.** Replace the mock roster; group/filter by department and team. Drop or explicitly disable the "you owe / owes you" filters — no action-item backend (§4.3). This is the §3 finding made real. | `app/(tabs)/people.tsx`, `src/api/hooks/index.ts`, `src/api/types.ts`, `src/api/queryKeys.ts` | M | scoping decision |
| **B2** | **Person detail onto `GET /bff/people/{user_id}`.** Renders `role`, `skills`, `bio`, `avatar_url`, `team`, `department`, `recent_meetings`, `topics`, `email`. Quotes and commitments have no backend and must be removed, not zeroed. Self-profile is blocked by §4.1. | `app/person/[id].tsx`, `src/api/hooks/index.ts`, `src/api/types.ts` | M | B1, scoping decision |
| **B3** | **Retire or repoint `useTeams()`/`useDepartments()`.** After B1, either they support the roster's grouping or they are dead — delete them if `org/chart` already carries the names. Correct the stale comment at `hooks/index.ts:36-39` either way. | `src/api/hooks/index.ts:34-55`, `src/api/queryKeys.ts` | S | B1 |
| **B4** | **Raise the missing self-identity route with the backend owner.** No `GET /users/me` exists (§4.1); it blocks self-profile, "assigned to me", and self-highlighting in the org chart. This is a request and a decision, not mobile code. | — (cross-repo) | S | — |
| **B5** | **Library onto `GET /api/v1/meetings?status=ended`, with cursor paging.** `useMeetings()` exists but ignores `next_cursor` (`hooks/index.ts:27-31`) — convert to `useInfiniteQuery`. Duration/speakers/chips stay omitted per `present.ts`'s rule. | `app/(tabs)/library.tsx`, `src/api/hooks/index.ts` | M | C5, scoping decision |
| **B6** | **Recap's real half: meeting header + transcript tab.** `GET /api/v1/meetings/{id}` and `GET /api/v1/meetings/{id}/transcript` (cursor + `from_ms`) both exist. The summary/decisions/actions tab stays mock — no backend (§4.3). Ship the screen half-real and say so in the UI. | `app/recap.tsx`, `src/api/hooks/index.ts`, `src/api/types.ts` | M | C5, B5 |
| **B7** | **Generated OpenAPI client.** The specs are now emitted from code and gated (`mytheus-docs` gap 5, closed 2026-08-28, ADR-0022). `src/api/types.ts:1-8` and `client.ts:19-20` both name themselves as the hand-sync that should be replaced. Removes the whole drift surface permanently. | `src/api/types.ts`, `src/api/client.ts`, build config | M | scoping decision |
| **B8** | **`POST /api/v1/meetings` from Start session.** Creates a real meeting so downstream screens have an id. Endpoint exists (`meetings.py:153`); needs a title input, since the screen currently shows a non-editable mock title chip (`start-session.tsx:47-50`). | `app/start-session.tsx`, `src/api/hooks/index.ts` | M | B5 |

### Group A — Blocked on backend that does not exist

Do not start these. Each is listed so the owner can see the shape of the repo's remaining work and
not mistake a mock screen for an oversight.

| ID | Title | Blocked by | Size when unblocked |
|---|---|---|---|
| **A-live** | Live session: LiveKit media + Socket.io captions. Transport exists (token mint, `realtime-gateway` `/v1`); **captions do not** — `transcription-service` is a stub (§4.4). Needs `@livekit/react-native` + `socket.io-client`, neither in `package.json`. Depends on B8 for a meeting id. | Plan 3-4 transcription | L |
| **A-ask** | Ask screen against a real agent/RAG surface. No such endpoint exists anywhere (§4.2). A 714-line rewrite (A6), plus real attachment pickers and an upload path (A9). | Plan 7-9 | L |
| **A-actions** | Actions tab against `GET /api/v1/actions`. Prefix is mapped, no router exists. | Plan 7-9 | M |
| **A-insights** | Insights against a real aggregate. Nothing computes it. | Plan 7-9 | M |
| **A-home-banner** | Home "week in review" from real decision/action counts, removing the last mock import from the one wired screen (`index.tsx:20`). | Plan 7-9 | S |
| **A-push** | Expo push registration. Net-new backend surface (N7), unbuilt on both sides. | a decision + `notification-service` | M |
| **A-rntl** | Component and screen tests via `jest-expo` + RNTL. | React 19.1.0 → 19.2.3 (`jest.config.js:1-9`) | L |
| **A-mock-delete** | Delete `src/data/mock.ts` (472 lines, 9 importers). | every screen above | S |

---

## 10. Bottom line

- **Contract drift: zero.** All four called endpoints verified against current backend source,
  field by field, including query params, limits and role tiers.
- **13 of the 14 tasks in Group C are unambiguously safe to start today** and none of them wait on
  gap 21. The permission removal (C2) and the release-signing fix (C6) are the two with real
  external exposure.
- **The committed-artifact premises were wrong** (§0.1, §0.2) — but the release build is
  debug-signed (§0.3), which is worse and is fixed by C6 rather than by anything in the repo.
- **`useTeams()`/`useDepartments()` are the wrong hooks for the People screen.** `GET /bff/org/chart`
  is the roster, it exists, and the comment saying otherwise is stale (§3).
- **Group B is blocked by a governance decision, not by engineering.** Every endpoint it needs is
  live. Getting that decision made is the highest-leverage unblock in the repo.
