# mytheus-ui — Real Data vs Mock Data Audit

READ-ONLY audit. No files in `mytheus-ui` were modified; no build or install was run.

- Root: `/Users/sushan/Projects/Mytheus/mytheus-ui`
- Branch audited: `main` @ `2e298c5` ("feat(api): generate the wire types from the emitted OpenAPI specs")
- `git fetch --all` run first. Remote branches: `origin/main` (same commit), `origin/local-mytheus-rebuild` @ `cca0e6a`. **The rebuild branch was NOT audited** — this report describes `main` only.
- Working tree was dirty at audit time: `package.json`, `package-lock.json`, `src/app/(marketing)/page.tsx`, `src/components/landing/Hero.tsx`, `src/components/landing/SiteFooter.tsx` modified.
- Source of truth: the `.tsx` files themselves. No status doc (PROGRESS.md, design.md, 4-july.md, LANDING.md) was consulted or trusted.

Method: traced every importer of `@/lib/mock/data` and `@/lib/api`, then read each page in full. Cross-referenced against the emitted OpenAPI specs in `src/lib/api/generated/` (bff, core-api, identity-tenant-service, meeting-service) and the backend service tree at `/Users/sushan/Projects/Mytheus/mytheus-backend/services/`.

---

## APP PAGE TABLE

All paths relative to `src/app/`. "Evidence" cites `file:line`.

| # | Page | Data source | Real/Mock | Loading | Error | Empty | Evidence |
|---|---|---|---|---|---|---|---|
| 1 | `(app)/dashboard` | **HYBRID — REAL API CALL** `GET /bff/dashboard` via `apiSafe`, but only the `upcoming_meetings` arm is used. The other three panels (StatGrid, ActionItems, RecentDecisions) render mock. | ~25% real | YES — route-level `(app)/loading.tsx` | YES — `apiSafe` returns a result union; a failed fetch and a server-reported degraded arm collapse to the same `"degraded"` status | YES — `ArmStatus` distinguishes `empty` from `degraded` (`lib/api/types.ts:34`) | real read `(app)/dashboard/page.tsx:36`; arm extract `:42-45`; mock import `:10`; mock-derived headline count `:51`; mock children mounted `:78`, `:81`, `:92`, `:93` |
| 2 | `(app)/meetings` | **MOCK ARRAY IN FILE** — `mockMeetings`. `GET /api/v1/meetings` exists in the spec and is not called. | Mock | NO | NO | YES (`:89`) | `(app)/meetings/page.tsx:6` import; `:69-75` `mockMeetings.filter(...)` for live/upcoming/past |
| 3 | `(app)/meetings/[meetingId]` | **MOCK ARRAY IN FILE** — five mock sets: `mockMeetings`, `mockMeetingDetails`, `mockDecisions`, `mockActions`, `mockTranscriptChunks` | Mock | NO | NO | YES (`:179` "No outcomes captured") | `(app)/meetings/[meetingId]/page.tsx:9-15` imports; `:40` lookup; `:43-46` all four derived lists; `:24` `generateStaticParams` enumerates mock IDs |
| 4 | `(app)/meetings/[meetingId]/room` | **MOCK ARRAY IN FILE** + dormant real wiring. Page itself is pure mock; `MeetingRoom` holds real LiveKit + socket hooks behind conditions that are false everywhere today. | Mock in practice | NO | NO | NO | `(app)/meetings/[meetingId]/room/page.tsx:4`, `:22`; real-path gates `components/meeting/MeetingRoom.tsx:110` (`live.status === "connected"`) and `:172` (`socket.status === "joined"`) |
| 5 | `(app)/meetings/[meetingId]/transcript` | **REAL API CALL** — `GET /api/v1/meetings/{id}/transcript` (cursor-paginated) and `GET /api/v1/meetings/{id}/speakers`, with a deliberate mock fallback when the gateway is unreachable | Real, mock fallback | NO | PARTIAL — a real `ApiError` rethrows (404 → `notFound()`); a non-`ApiError` (unset/unreachable origin) silently returns `null` and renders mock | YES (`:78`) | page `(app)/meetings/[meetingId]/transcript/page.tsx:59`; endpoints `lib/api/transcript.ts:44`, `:56`; fallback contract `lib/api/transcript.ts:71-88`; mock adapter `page.tsx:28-49` |
| 6 | `(app)/decisions` | **MOCK ARRAY IN FILE** — `mockDecisions`. No decisions endpoint exists in any spec. | Mock | NO | NO | YES (`:38`) | `(app)/decisions/page.tsx:6` import; `:23` `[...mockDecisions].sort(...)` |
| 7 | `(app)/tasks` | **MOCK ARRAY IN FILE** — `mockActions`. Completion checkbox is `useState` only; nothing is persisted. | Mock | NO | NO | YES (`:115`) | `(app)/tasks/page.tsx:10` import; `:96` `doneMap` seeded from mock; `:99-100` mine/others split; self-declared at `:12-14` "visual state only — the Server Action + optimistic reconcile wires in when the backend lands" |
| 8 | `(app)/search` | **MOCK ARRAY IN FILE** — `mockSearchIndex`, filtered client-side with a `.includes()` substring match. Not semantic search; no network call. | Mock | NO | NO | YES (`:58`) | `(app)/search/page.tsx:5` import; `:28-32` the substring filter |
| 9 | `(app)/org` | **MOCK ARRAY IN FILE** — `mockDepartments`, `mockOrgContext`. `GET /bff/org/chart` exists in the spec and is not called. | Mock | NO | NO | NO | `(app)/org/page.tsx:5` import; `:13` member count; `:21` org name from mock; `:30` department map |
| 10 | `(app)/billing` | **MOCK OBJECT IN FILE** — `mockBilling` (plan, seats, price, invoices). "Change plan" button has no handler. | Mock | NO | NO | PARTIAL — invoice-list empty branch only (`:86`) | `(app)/billing/page.tsx:2` import; `:22` full destructure; inert button `:43-45` |
| 11 | `(app)/settings` | **STATIC + real org identity.** `org.name`/`org.plan` come from the Clerk-backed context; every notification toggle is local `useState`; "Save changes" only sets a flag. | Static/mock | NO | NO | n/a | `(app)/settings/page.tsx:49` `useOrgContext()`; `:54-56` three `useState` toggles; `:91` `onClick={() => setOrgSaved(true)}` — writes nothing, then renders "Saved." at `:94` |
| 12 | `(app)/settings/profile` | **REAL identity + HARDCODED fields.** `name`/`email`/`role` are real (Clerk); `title` and `focus` are string literals. Save is visual only. | Partial | NO | NO | n/a | `(app)/settings/profile/page.tsx:17` `useOrgContext()`; `:21` `useState("Founder")`; `:22` `useState("Product, frontend, fundraising")`; `:83` `onClick={() => setSaved(true)}`; self-declared `:14` |
| 13 | `(app)/settings/integrations` | **MOCK ARRAY IN FILE** — `mockIntegrations`. Connect/disconnect mutates local state only. | Mock | NO | NO | NO | `(app)/settings/integrations/page.tsx:8` import; `:16-17` state seeded from mock; `:22-30` local `toggle`; self-declared `:11-13` |
| 14 | `(admin)/admin` | **MOCK ARRAY IN FILE** — `mockTenants` | Mock | NO | NO | NO | `(admin)/admin/page.tsx:2` import; `:35-41` all three tiles + the sorted table derive from mock |
| 15 | `(admin)/ops` | **MOCK ARRAY IN FILE** — `mockTenantOps`, behind a hardcoded feature flag | Mock | NO | NO | PARTIAL — disabled-flag branch only (`:20`) | `(admin)/ops/page.tsx:5` import; `:12` `const OPS_CONSOLE_ENABLED = true`; `:29-30` rollups; self-declared `:9-11` "mock data until the ops plane exists" |
| 16 | `(auth)/sign-in` | **REAL** — Clerk `<SignIn>`, path routing, `fallbackRedirectUrl="/dashboard"` | Real | Clerk-owned | Clerk-owned | n/a | `(auth)/sign-in/page.tsx:1`, `:15-22` |
| 17 | `(auth)/sign-up` | **REAL** — Clerk `<SignUp>`. Note: creates a Clerk user, not a tenant; tenant provisioning (`POST /api/v1/orgs`) is deferred, so a new user authenticates and is then refused by the gateway. | Real | Clerk-owned | Clerk-owned | n/a | `(auth)/sign-up/page.tsx:1`, `:16-21`; documented at `:7-11` |

### Supporting infrastructure that IS real

- **Auth/middleware** — `src/middleware.ts:22-49` is an allowlist of public routes; `(app)` and `(admin)` are protected by omission, `auth.protect()` at `:56`. Correct direction (fails closed).
- **Org context** — `src/lib/api/org-context.ts:24-48` resolves from the verified Clerk session, not `mockOrgContext`. It deliberately under-reports two fields it cannot fill honestly: `org.plan` is pinned to `"starter"` and `user.role` to `"member"` (`:36-47`), both least-privilege.
- **API client** — `src/lib/api/client.ts` is server-only (imports `@clerk/nextjs/server`, so a client import fails the build), never sends `org_id`, defaults to `cache: "no-store"` for tenant data (`:148`), and parses RFC 7807 problems into a typed `ApiError` with a generated `ErrorCode` union (`:35`).
- **Wire types** — `src/lib/api/types.ts` aliases only generated OpenAPI schemas; nothing hand-described.

### Defects found while reading

**D-UI-1 — the one real page is gated on the mock array.** `(app)/meetings/[meetingId]/transcript/page.tsx:56`:

```
const meeting = mockMeetings.find((m) => m.id === params.meetingId);
if (!meeting) notFound();
```

This runs *before* the real read at `:59`. A genuine backend meeting ID that is not one of the mock fixture IDs 404s and the real fetch never executes. The only page wired to live data is therefore unreachable for live data. `meeting.title` (`:66`) and `meeting.status` (`:83`) are also mock-sourced on the real path.

**D-UI-2 — admin console has no role check.** `(admin)/layout.tsx:15-18` still carries `// TODO(auth): restore Clerk middleware + RBAC gate for (admin) here.` and wraps everything in `<OrgProvider value={mockOrgContext}>`. The inline comment claiming "anyone can reach /admin" is stale — `middleware.ts` does require a session, since `(admin)` is absent from the allowlist. But there is still no role assertion, so **any signed-in member reaches the admin and ops consoles**, rendering under a hardcoded org identity. Real enforcement is server-side per CLAUDE.md, and these screens read no server data, so nothing is currently leaked — the gap is that the surface is reachable.

**D-UI-3 — stale "no gateway anywhere" reasoning.** `.env.local` sets `API_GATEWAY_URL=http://localhost:8080` (and `.env.example` ships the same). Multiple comments assert the opposite: `lib/api/transcript.ts:66-69` ("which is every environment today"), `dashboard/page.tsx:29-31`, `MeetingRoom.tsx:38-39`, `transcript/page.tsx:19-20`. The fallback still behaves correctly (an unreachable localhost throws a non-`ApiError`, so `fetchTranscriptSurface` returns `null` and mock renders), but the documented premise no longer matches the config. `NEXT_PUBLIC_LIVEKIT_URL` is genuinely absent from both env files.

**D-UI-4 — `/students/verify` tells the user a lie.** See MARKETING section.

---

## MOCK DATA INVENTORY

### `src/lib/mock/data.ts` — 18,852 bytes, 41 exports, imported by 14 files

Every hardcoded dataset still shipping:

| Export | Defined at | Consumed by |
|---|---|---|
| `mockOrgContext` | `data.ts:11` | `(admin)/layout.tsx:4,18`; `(app)/org/page.tsx:5,21`; `components/app/dashboard/ActionItems.tsx:3` |
| `mockMeetings` | `data.ts:29` | `(app)/meetings/page.tsx:6`; `(app)/meetings/[meetingId]/page.tsx:10`; `(app)/meetings/[meetingId]/room/page.tsx:4`; `(app)/meetings/[meetingId]/transcript/page.tsx:8`; `components/app/dashboard/StatGrid.tsx:3`; `components/app/dashboard/TodaysMeetings.tsx:6` |
| `mockDecisions` | `data.ts:96` | `(app)/decisions/page.tsx:6`; `(app)/meetings/[meetingId]/page.tsx:12`; `components/app/dashboard/StatGrid.tsx:3`; `components/app/dashboard/RecentDecisions.tsx:4` |
| `mockActions` | `data.ts:141` | `(app)/tasks/page.tsx:10`; `(app)/dashboard/page.tsx:10`; `(app)/meetings/[meetingId]/page.tsx:13`; `components/app/dashboard/StatGrid.tsx:3`; `components/app/dashboard/ActionItems.tsx:3` |
| `mockBilling` | `data.ts:184` | `(app)/billing/page.tsx:2` |
| `mockTenants` | `data.ts:208` | `(admin)/admin/page.tsx:2` |
| `mockTeamCapacity` | `data.ts:222` | **`components/landing/Statement.tsx:18` — the PUBLIC landing page** |
| `mockSourcedAnswer` | `data.ts:236` | **`components/landing/Statement.tsx:19` — the PUBLIC landing page** |
| `mockSearchIndex` | `data.ts:250` | `(app)/search/page.tsx:5` |
| `mockMeetingDetails` | `data.ts:266` | `(app)/meetings/[meetingId]/page.tsx:11` |
| `mockTranscriptChunks` | `data.ts:320` | `(app)/meetings/[meetingId]/transcript/page.tsx:8` (fallback path); `(app)/meetings/[meetingId]/page.tsx:14` |
| `mockRoomParticipants` | `data.ts:347` | `components/meeting/MeetingRoom.tsx:15` — the video-grid stand-in, `:114` |
| `mockCaptions` | `data.ts:356` | `components/meeting/MeetingRoom.tsx:16` — the caption strip, `:180` |
| `mockRoomChat` | `data.ts:363` | `components/meeting/MeetingRoom.tsx:17` — `ChatPanel` seed, `:334` |
| `mockAgentSidebar` | `data.ts:385` | `components/sidebar/AgentSidebar.tsx:9` |
| `mockDepartments` | `data.ts:441` | `(app)/org/page.tsx:5` |
| `mockIntegrations` | `data.ts:487` | `(app)/settings/integrations/page.tsx:8` |
| `mockTenantOps` | `data.ts:529` | `(admin)/ops/page.tsx:5` |

Type-only imports from the mock module (not data, but a coupling worth noting — the real API layer still depends on the mock module for its shapes):

- `lib/api/present.ts:15` — `import type { MeetingStatus as MockMeetingStatus }`
- `lib/api/org-context.ts:22` — `import type { OrgContextValue }`, i.e. the Clerk-backed resolver returns a type defined in `lib/mock/data.ts`
- `components/app/dashboard/format.ts:1` — `import type { Priority }`

### Hardcoded datasets OUTSIDE the mock module

| Data | File:line | Note |
|---|---|---|
| `const openConflicts = [...]` | `components/app/dashboard/StatGrid.tsx:16-23` | One fabricated conflict, "Pricing rework vs. Q3 launch date". Comment: "local const until /bff/dashboard exposes a conflicts rollup" |
| `const agentActivity: AgentActivity[]` | `components/app/dashboard/AgentActivityStrip.tsx:14-30` | Three fabricated agent actions presented on the dashboard as things your agent did |
| `MOCK_NOW` | `components/app/dashboard/format.ts` (referenced `StatGrid.tsx:5`) | A frozen clock so mock deltas stay deterministic; comment at `format.ts:6` |
| `useState("Founder")` / `useState("Product, frontend, fundraising")` | `(app)/settings/profile/page.tsx:21-22` | Hardcoded role/focus presented as the user's own profile |
| `{ from: "Yash Maurya" }` | `components/meeting/MeetingRoom.tsx:343` | Hardcoded sender name on every locally-sent chat message, regardless of who is signed in |
| `const OPS_CONSOLE_ENABLED = true` | `(admin)/ops/page.tsx:12` | Hardcoded feature flag |
| `const posts = [...]` | `(marketing)/blog/page.tsx:28` | Hardcoded blog index |
| `const ROLES: Role[] = [...]` | `(marketing)/careers/page.tsx:132` | Hardcoded job openings |
| `FEATURES: FeatureDef[]` | `lib/marketing/features.ts:32` | All copy for the 11 feature pages. Static by design — not a defect, listed for completeness |

---

## ORPHAN COMPONENTS

**Strictly speaking: none.** Every component in `src/components/meeting/` is imported and mounted:

| Component | Mounted at |
|---|---|
| `MeetingRoom` | `(app)/meetings/[meetingId]/room/page.tsx:25` |
| `LiveVideoGrid` | `components/meeting/MeetingRoom.tsx:111` |
| `LiveCaptions` | `components/meeting/MeetingRoom.tsx:173` |
| `TranscriptView` | `(app)/meetings/[meetingId]/transcript/page.tsx:88` |
| `SpeakerMapping` | `(app)/meetings/[meetingId]/transcript/page.tsx:73` |
| `AgentSidebar` | `components/meeting/MeetingRoom.tsx:325` |

**But three are dead in practice** — mounted behind a condition that is false in every environment that currently exists. Call these *conditionally orphaned*: real code, real wiring, zero render paths today.

| Component | Gate | Why it never fires |
|---|---|---|
| `LiveVideoGrid` | `live.status === "connected" && live.room` — `MeetingRoom.tsx:110` | `useLiveKitRoom` reports `"unconfigured"` without `NEXT_PUBLIC_LIVEKIT_URL`, which appears in neither `.env.local` nor `.env.example`. The mock grid at `:113-163` renders instead. |
| `LiveCaptions` | `socket.status === "joined"` — `MeetingRoom.tsx:172` | Requires a reachable realtime gateway and a passed participant check. The mock caption strip at `:175-195` renders instead. |
| `SpeakerMapping` | `speakers.length > 0` — `transcript/page.tsx:71` | `speakers` is `live?.speakers ?? []` (`:61`). On the mock fallback path it is always `[]`, so the mapping surface never appears. |

`TranscriptView` does render on the mock path (`chunks` falls back to `mockAsChunks`), so it is the one meeting component with a live render path today.

Supporting real-time modules that exist and are wired but idle: `lib/realtime/useLiveKitRoom.ts`, `lib/realtime/useMeetingSocket.ts`, `lib/realtime/captions.ts`. Two internal route handlers exist and are also currently unexercised by any live path: `app/api/livekit-token/route.ts`, `app/api/meeting-transcript/route.ts`.

---

## MARKETING PROMISES vs REALITY

The 11 feature pages are 8-line shells — each imports `FeaturePage` and passes a slug (e.g. `(marketing)/features/knowledge-graph/page.tsx:7` → `<FeaturePage slug="knowledge-graph" />`). All copy lives in `src/lib/marketing/features.ts:32`.

Backend column is judged against the emitted specs in `src/lib/api/generated/` — the complete surface is:

- **bff**: `/bff/dashboard`, `/bff/org/chart`, `/bff/people/{user_id}`, `/healthz`
- **core-api**: `/api/v1/chat`, `/api/v1/departments`, `/api/v1/meetings`, `/api/v1/meetings/{id}`, `/api/v1/meetings/{id}/participants/{clerk_user_id}`, `/api/v1/meetings/{id}/speakers`, `/api/v1/meetings/{id}/speakers/{speaker_id}`, `/api/v1/meetings/{id}/transcript`, `/api/v1/teams`, `/api/v1/teams/{id}/members`, `/api/v1/teams/{id}/members/{user_id}`, `/healthz`
- **identity-tenant-service**: `/clerk/webhook`, `/context/{user_id}`, `/scim/v2/Groups`, `/scim/v2/Users`, `/scim/v2/Users/{user_id}`, `/taxonomy/role-topics`, `/tenant-settings`, `/users/{user_id}`, `/healthz`
- **meeting-service**: `/api/meetings/{id}/token`, `/livekit/webhook`, `/healthz`

There is **no** decisions, action-items, search, knowledge-graph, insights, billing, or integrations endpoint anywhere in the exposed surface.

| Feature | Marketing page | App page? | Backend? | Verdict |
|---|---|---|---|---|
| personal-agent | `features/personal-agent` (`features.ts:35`) | Partial — `AgentSidebar`, mock only | **NO** | GAP |
| live-notes | `features/live-notes` (`:61`) | Partial — `AgentSidebar`, mock only | **NO** | GAP |
| smart-questions | `features/smart-questions` (`:87`) | Partial — `AgentSidebar`, mock only | **NO** | GAP |
| decisions | `features/decisions` (`:113`) | `/decisions`, mock | **NO** — arm returns `not_yet_built`, `services/bff/dashboard.py:208` | GAP |
| action-items | `features/action-items` (`:139`) | `/tasks`, mock | **NO** — `not_yet_built`, `services/bff/dashboard.py:206` | GAP |
| context-gaps | `features/context-gaps` (`:165`) | **none** | **NO** | GAP |
| proactive-insights | `features/proactive-insights` (`:191`) | **none** | **NO** | GAP |
| transcription | `features/transcription` (`:217`) | `/meetings/[id]/transcript` | **YES** — `/api/v1/meetings/{id}/transcript` + `/speakers` | **HONEST** |
| search | `features/search` (`:243`) | `/search`, substring filter over a mock array | **NO** | GAP — page markets "meaning, not keywords"; the code is `.includes()` |
| knowledge-graph | `features/knowledge-graph` (`:269`) | **none** | **NO** | GAP |
| integrations | `features/integrations` (`:295`) | `/settings/integrations`, mock | PARTIAL — `services/integration_service/` exists, no gateway route | GAP |

**10 of 11 feature pages market a capability with no backend behind it.** The dashboard arms confirm this from the server side: `services/bff/dashboard.py:204-208` gathers `action_items`, `worklogs` and `decisions` from a function literally named `not_yet_built`, and `UnbuiltArm` is documented at `:94-97` as "shaped-but-empty until Plan 7-9".

### Remaining 12 marketing pages, one line each

| Page | Classification |
|---|---|
| `(marketing)/page.tsx` — landing | STATIC, but `components/landing/Statement.tsx:18-19` pulls `mockTeamCapacity` and `mockSourcedAnswer` into the public hero animation — fixture data shipped to anonymous visitors |
| `features/page.tsx` — index | STATIC; links all 11 feature pages, of which 1 has a backend |
| `pricing` | STATIC tiers; no billing endpoint exists, and `/billing` renders `mockBilling` |
| `enterprise` | STATIC; promises SCIM — this one is **backed**: `identity-tenant-service` exposes `/scim/v2/Users` and `/scim/v2/Groups` |
| `security` | STATIC claims. **UNVERIFIED** — the assertions were not cross-checked against actual controls, which is out of scope for a UI audit |
| `about` | STATIC — fine |
| `blog` | STATIC with hardcoded `posts` (`:28`); no CMS. Fine as a marketing choice, listed for completeness |
| `careers` | STATIC with hardcoded `ROLES` (`:132`); apply links UNVERIFIED |
| `privacy` | STATIC legal text — fine. Correctly public via `middleware.ts:35` |
| `terms` | STATIC legal text — fine. Correctly public via `middleware.ts:36` |
| `students` | STATIC offer page |
| `students/verify` | **DEFECT D-UI-4 — the page lies.** `(marketing)/students/verify/page.tsx:30-34` sets `submitted` on a bare regex pass (`:7`, `:28`) with no network call anywhere in the file, then renders "Check your inbox — we sent a verification link to {email}" at `:48-51`. Nothing is sent. This is the only page in the app that makes a false factual claim to the user about an action taken on their behalf. |

---

## MISSING APP PAGES

Endpoints that exist in the emitted specs with no UI consuming them:

| Endpoint | Status | Note |
|---|---|---|
| `GET /bff/org/chart` | **Unused** | `/org` renders `mockDepartments` instead. A working backend the UI is simply not pointed at. |
| `GET /bff/people/{user_id}` | **Unused** | No person/profile page consumes it |
| `GET /api/v1/meetings` | **Unused** | `/meetings` is on `mockMeetings` |
| `GET /api/v1/meetings/{id}` | **Unused** | The detail page is on `mockMeetings` + `mockMeetingDetails` |
| `POST /api/v1/chat` | **Unused — no UI at all** | The in-room `ChatPanel` (`MeetingRoom.tsx:333-382`) is pure local `useState` with a hardcoded sender |
| `GET /api/v1/departments` | **Unused** | No admin/org-editing UI |
| `GET /api/v1/teams`, `/teams/{id}/members`, `/teams/{id}/members/{user_id}` | **Unused** | Full team CRUD with no UI |
| `/api/v1/meetings/{id}/participants/{clerk_user_id}` | **Unused** | No participant-management UI |
| `/api/v1/meetings/{id}/speakers/{speaker_id}` | Partially used | `SpeakerMapping` targets it, but the component never renders (see ORPHAN COMPONENTS) |
| `/tenant-settings`, `/taxonomy/role-topics`, `/users/{user_id}` (identity) | **Unused** | No settings surface reads or writes them; `/settings` is entirely local state |
| `/api/meetings/{id}/token` (meeting-service) | Wired, dormant | `app/api/livekit-token/route.ts` calls it, but nothing reaches that route without LiveKit configured |

Backend services with **no reachable UI whatsoever**: `billing_metering_service`, `notification_service`, `workflow_service`, `admin_ops_service` (`/Users/sushan/Projects/Mytheus/mytheus-backend/services/`). Their maturity was not assessed — only that no page calls them.

**The largest gap runs opposite to the usual direction.** Meetings, org chart, chat and teams are surfaces where a working backend already exists and the UI has simply not been pointed at it. That is a much cheaper class of work than the ten marketing-promised features that need a backend built from nothing.

---

## VERDICT

Two of seventeen app pages read real data — the transcript page and the sign-in/sign-up pair — with the dashboard a third at roughly one panel in four, so call it **~15% of the product UI actually live**. Everything else renders `src/lib/mock/data.ts`, which is still imported by fourteen files and even leaks fixture data into the public landing page's hero animation. The infrastructure is markedly better than the screens: Clerk auth, a fail-closed middleware allowlist, a server-only typed client generated from the backend's OpenAPI documents, and genuine LiveKit and socket components are all built and correct — but three of those components sit behind conditions no current environment satisfies, and the one page wired to the real API gates its own existence on `mockMeetings.find()` at `transcript/page.tsx:56`, so the real path cannot be reached with real data. The sharpest product gap is that ten of eleven marketing feature pages sell capabilities with no backend at all, while the endpoints that *do* exist — meetings, org chart, chat, teams — have no UI pointed at them.

### Ranked remediation

1. **D-UI-1** — remove the `mockMeetings` gate at `transcript/page.tsx:56`. One-line class of fix that unblocks the only live page.
2. **D-UI-4** — `/students/verify` claims to have sent an email it never sends. User-facing false statement, on a public page.
3. **D-UI-2** — add the RBAC gate at `(admin)/layout.tsx:15`; any signed-in member currently reaches the admin and ops consoles.
4. **Point existing UI at existing backends** — `/meetings` → `GET /api/v1/meetings`, `/org` → `GET /bff/org/chart`. Backend already ships; highest value per unit of work in the whole list.
5. **D-UI-3** — reconcile the "no gateway in any environment" comments with `.env.local`, which sets `API_GATEWAY_URL`.
6. **Marketing honesty** — 10 of 11 feature pages need either a backend or a change in tense.
