# mytheus-ui — pending work, verified against source

Read-only audit. Nothing in `mytheus-ui` was modified. All line cites are against
`/Users/sushan/Projects/Mytheus/mytheus-ui` at local `main` = `2e298c5`.

---

## 0. READ THIS FIRST — `origin/local-mytheus-rebuild` conflicts with the whole premise of this task list

`git fetch` run. Facts:

| | |
|---|---|
| Branch | `origin/local-mytheus-rebuild` |
| Tip | `cca0e6a fix(landing): stop the header colliding with the page, and retire it at the end` |
| Last commit | **2026-08-13** (Yash-v-maurya) — 25 days stale |
| Merge-base with main | `3d26d63 fix(landing): uniform static CTA pills + clip-proof reveal masks` |
| Divergence | main is **11 ahead**, rebuild is **6 ahead** |
| Files changed vs merge-base | 35 (`+1475 / -836`) |

### 0a. The branch declares this repo MARKETING-ONLY and moves the product app to `mytheus-web`

This is the loud part. Commit `0455cc8 feat(marketing): send every CTA to the product app instead
of a local route`, message verbatim:

> The authenticated product lives in mytheus-web, a separate Next.js app, so getting started has
> to leave this origin. […] The sign-in and sign-up stubs redirected into the local shell. They
> now hand off to the app, which owns the real auth flow, so existing bookmarks land somewhere
> real instead of a route that is about to leave this repo.

New file on that branch, `src/lib/marketing/app-url.ts`:

> This repo is the marketing site only. The authenticated product - the shell (home, meetings,
> decisions, tasks, search, settings) and the call surfaces (prejoin, meeting) - is a separate
> Next.js app in `mytheus-web`, so every "get started" path leaves this origin.

And **`git@github.com:Mytheus-AI/mytheus-web.git` exists and has a `main`** (`ls-remote` →
`82b7d96e…  refs/heads/main`). It is not checked out at `/Users/sushan/Projects/Mytheus/`.

**Consequence for this task list:** every item below that wires an `(app)` or `(admin)` page to a
backend (tasks A1–A9, B1–B7, C1–C3) may belong to `mytheus-web`, not here. **Do not assign any
app-page wiring work until someone confirms whether `mytheus-web` is the live product app.** That
is the single highest-value question on this page, and I could not answer it from local disk.

### 0b. Direct file-level overlap with items in this list

| Item asked about | Rebuild branch touches | Overlap |
|---|---|---|
| **#10** `src/lib/mock/data.ts` removal order | `src/lib/mock/data.ts` (+17/−… ) and adds `src/lib/mock/time.ts` (94 lines) | **YES — real conflict.** It edits `mockSourcedAnswer` (drops the baked `date`, makes `source.date` optional) and rewrites the `mockAgentSidebar` story comment. Any mock-removal work on main will conflict. |
| **#5** `MeetingRoom.tsx` token refresh | `src/components/meeting/MeetingRoom.tsx` | Cosmetic only — one gradient rgba value at line ~94. No conflict with the token fix. |
| **#6** Docker build args / env | `.env.example` | **YES — destructive.** `.env.example` did **not exist** at the merge-base; main added it (Clerk keys, `API_GATEWAY_URL`, the `NEXT_PUBLIC_*` warning). The rebuild branch adds its own 7-line `.env.example` containing only `NEXT_PUBLIC_APP_URL`. A naive merge **wipes the entire Clerk + gateway env contract.** |
| **#9** marketing feature pages | `src/components/marketing/FeaturePage.tsx`, `PageShell.tsx`, `(marketing)/features/page.tsx`, `pricing`, `security`, `blog`, `enterprise` | **YES.** All CTAs rewritten to `appHref()`. Any marketing-page task collides. |
| Landing components | `Hero.tsx` (+765/−…), `Statement.tsx`, `Steps.tsx`, `SiteFooter.tsx`, `SiteHeader.tsx`, `MemoryTimeline.tsx`, `LiveCapture.tsx`, `ConnectedGraph.tsx`, `FAQ.tsx`, `FlowField.tsx`, `Loader.tsx`, `globals.css` | **YES.** Assign no landing/marketing visual work. |

### 0c. What the branch does NOT do

It contains **none** of main's last 11 commits: no LiveKit client, no live captions, no transcript
view, no generated OpenAPI types, no CI, no Dockerfile. So it fixes **none** of items #1–#8, #11,
#12. Its unique work is landing/marketing polish, theming, an `/about` page, a date-derivation
helper, and the marketing-only split.

**Recommendation:** before any assignment, (a) confirm the `mytheus-web` split with the branch
author, and (b) if the branch is landing, rebase it onto main rather than merging — the
`.env.example` clobber is silent and would break every deploy.

---

## 1. Verification of the twelve items

### #2 — `transcript/page.tsx` `notFound()` before the real fetch — **CONFIRMED**

`src/app/(app)/meetings/[meetingId]/transcript/page.tsx:56-59`

```
56    const meeting = mockMeetings.find((m) => m.id === params.meetingId);
57    if (!meeting) notFound();
58
59    const live = await fetchTranscriptSurface(params.meetingId);
```

The page is `export const dynamic = "force-dynamic"` (`:22`) and its whole reason for existing is
the real read at `:59` — but a meeting id that is not one of the five mock ids (`mtg_1`…`mtg_5`,
`data.ts:29+`) 404s at `:57`. Since real meeting ids are UUIDs (`core-api.ts` → `Id / Format: uuid`),
**every real meeting 404s on this page today.** The mock is also load-bearing for the `<h1>` at
`:66` (`meeting.title`), so the fix is not just moving the guard — the page needs a real
`GET /api/v1/meetings/{meeting_id}`.

**Same bug, two more places, not in the brief:**
- `room/page.tsx:22-23` — identical `mockMeetings.find` → `notFound()`.
- `meetings/[meetingId]/page.tsx:40-41` — identical.

And all three carry `generateStaticParams()` returning the five mock ids
(`[meetingId]/page.tsx:23-25`, `room/page.tsx:13-15`), so the detail and room routes are
**statically prerendered per-tenant-agnostic mock id** with no `dynamic` export.

### #3 — `(admin)/layout.tsx` role check — **CONFIRMED, and worse than described**

The brief says "has a session check but no role check." It has **neither**.
`src/app/(admin)/layout.tsx:16-18` is a plain sync component wrapping `OrgProvider` with
`mockOrgContext` — no `resolveOrgContext()`, no `redirect()`, no role read. Its own comment
(`:12-15`) says so:

```
12  /* RBAC gate is UX courtesy only per CLAUDE.md; real enforcement is
13     server-side. Auth isn't wired up yet, so there's no gate at all right now
14     — anyone can reach /admin. Add the Clerk role check here once auth lands. */
15  // TODO(auth): restore Clerk middleware + RBAC gate for (admin) here.
```

The comment is now **stale**: `src/middleware.ts:22-49` is an allowlist and `/admin` and `/ops`
are absent from it, so `auth.protect()` (`:56`) *does* require a session. The residual exposure is
therefore: **any signed-in user of any tenant can load `/admin` and `/ops`.** Today they see mock
tenant rows (`mockTenants`, `mockTenantOps`), so it leaks nothing real — but the moment `/ops` is
wired it becomes a cross-tenant read.

Blocking dependency: `src/lib/api/org-context.ts:46` hardcodes `role: "member"` with an explicit
comment (`:13-15`) that permissions live in `rbac_grants` behind the gateway. **A role check
cannot be written today because there is no source of role.** Wiring the check requires
identity-tenant-service's `/context/{user_id}` first.

Contrast `(app)/layout.tsx:26-27`, which does it correctly (`resolveOrgContext()` +
`redirect("/sign-in")`).

### #4 — orphan generated clients + hand-written `TokenResponse` — **CONFIRMED**

Importers of `src/lib/api/generated/`:
- `bff.ts` → `types.ts:25`
- `core-api.ts` → `types.ts:26`, `client.ts:23`, `transcript.ts:15`
- **`identity-tenant-service.ts` (924 lines) → zero importers**
- **`meeting-service.ts` (242 lines) → zero importers**

The nullability contradiction, and it is a real bug not just a duplicate type:

| | `livekit_url` |
|---|---|
| `generated/meeting-service.ts:100-107` (authoritative) | `livekit_url: string` — **required, non-nullable** |
| `app/api/livekit-token/route.ts:31-35` (hand-written) | `livekit_url?: string \| null` |
| `lib/realtime/useLiveKitRoom.ts:46-50` (**second** hand-written copy) | `livekit_url?: string \| null` |

The hand-written optional is what licenses `useLiveKitRoom.ts:85`:

```
85    if (!configuredUrl) return;      // configuredUrl = process.env.NEXT_PUBLIC_LIVEKIT_URL
```

The hook refuses to attempt a connection unless `NEXT_PUBLIC_LIVEKIT_URL` is set, then at `:120`
does `minted.livekit_url ?? configuredUrl`. But the server **always** returns `livekit_url`. So the
env var is a hard gate on a value the server would supply anyway — and it is set in no environment
(see #6/#7). Typing against the generated schema makes `:120`'s `??` a compile error and forces the
gate at `:85` to be re-examined.

`generated/identity-tenant-service.ts` unused paths that map to unwired UI:
`/context/{user_id}`, `/users/{user_id}`, `/tenant-settings`, `/taxonomy/role-topics`.

### #5 — `MeetingRoom.tsx` token fetched once — **CONFIRMED, with a second, worse instance**

`src/components/meeting/MeetingRoom.tsx:45-53`:

```
45    useEffect(() => {
46      let cancelled = false;
47      void getToken().then((t) => {
48        if (!cancelled) setSocketToken(t);
49      });
50      return () => { cancelled = true; };
51    }, [getToken]);
54    const socket = useMeetingSocket(meeting.id, socketToken);
```

`getToken` is referentially stable, so this runs once per mount. Clerk session JWTs are short-lived
(60s default). The token is then frozen into the Socket.io client at
`useMeetingSocket.ts:75-78`:

```
75    const socket = io(`${url.replace(/\/+$/, "")}/v1`, {
76      auth: { token },
77      transports: ["websocket"],
78    });
```

Socket.io's automatic reconnect **reuses that same `auth` object**. So any reconnect after the TTL
presents an expired JWT, the gateway handshake rejects it, `connect_error` fires
(`useMeetingSocket.ts:117-119`) and status latches to `"error"` permanently — the room silently
stops receiving captions for the rest of the meeting with no recovery path. The hook's deps
(`[url, token, meetingId]`, `:140`) are already correct; they just never see a new token.

**Second instance, not in the brief.** `useLiveKitRoom.ts` documents at `:6-11`:

> Plan 3-4 §3.4 step 2: mint a token, connect, render the participant grid, **"re-mint the token on
> reconnect"**. […] a full disconnect needs a fresh token, which is why `connect` always mints
> rather than caching one in state.

**That is not implemented.** `mintToken` is called exactly once inside a `useEffect` that runs on
`[meetingId, configuredUrl, syncParticipants]` (`:113-137`). There is no `RoomEvent.Reconnecting`
re-mint handler. The documented behaviour and the code disagree; the doc comment is the bug report.

Minor, same file: `:139` returns `roomRef.current` — a ref read during render. It happens to work
because `setStatus("connected")` re-renders after the ref is set, but it is not concurrent-safe.

### #6 — realtime/LiveKit URLs are not Docker build args — **CONFIRMED, and broader**

`Dockerfile:29-35`:

```
29  ARG NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY
30  ARG CLERK_SECRET_KEY
31  ARG API_GATEWAY_URL
32  ENV NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=$NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY \
33      CLERK_SECRET_KEY=$CLERK_SECRET_KEY \
34      API_GATEWAY_URL=$API_GATEWAY_URL \
35      NEXT_TELEMETRY_DISABLED=1
```

`NEXT_PUBLIC_*` is inlined at build time. Missing build args:
- `NEXT_PUBLIC_REALTIME_WS_URL` — read at `useMeetingSocket.ts:60`
- `NEXT_PUBLIC_LIVEKIT_URL` — read at `useLiveKitRoom.ts:69`
- `NEXT_PUBLIC_CLERK_SIGN_IN_URL` / `NEXT_PUBLIC_CLERK_SIGN_UP_URL` — in `.env.example:15-16`, so
  containerised Clerk redirects fall back to Clerk's hosted pages, leaving the domain
- `NEXT_PUBLIC_APP_ENV` — `.env.example:34`

**No container build can ever enable LiveKit or the realtime socket.** This is the mechanical cause
of #7.

### #7 — `LiveVideoGrid` / `LiveCaptions` / `SpeakerMapping` are conditionally orphaned — **CONFIRMED**

None is an unreferenced file; all three are behind gates that are false everywhere.

| Component | Gate | Why it is false |
|---|---|---|
| `LiveVideoGrid` | `MeetingRoom.tsx:110` — `live.status === "connected" && live.room` | needs `NEXT_PUBLIC_LIVEKIT_URL` (`useLiveKitRoom.ts:69,85`) |
| `LiveCaptions` | `MeetingRoom.tsx:172` — `socket.status === "joined"` | needs `NEXT_PUBLIC_REALTIME_WS_URL` (`useMeetingSocket.ts:60,69`) |
| `SpeakerMapping` | `transcript/page.tsx:71` — `speakers.length > 0` | `speakers` is `live?.speakers ?? []` (`:61`); `live` is `null` without a reachable `API_GATEWAY_URL` |

Environment evidence: `.env.example:31-32` has both realtime vars **commented out** under a
"`--- Later bands ---` […] unused today" heading; `.env.local` contains only
`NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY`, `NEXT_PUBLIC_CLERK_SIGN_IN_URL`,
`NEXT_PUBLIC_CLERK_SIGN_UP_URL`, `API_GATEWAY_URL`, `NEXT_PUBLIC_APP_ENV`; and the Dockerfile
cannot pass them (#6).

**Genuine zero-importer orphan, not in the brief:** `src/components/ComingSoon.tsx` — grep for
`ComingSoon` across `src` returns only its own definition.

### #8 — `/students/verify` claims an email was sent — **CONFIRMED**

`src/app/(marketing)/students/verify/page.tsx:30-34`:

```
30    function onSubmit(e: FormEvent<HTMLFormElement>) {
31      e.preventDefault();
32      if (!valid) return;
33      setSubmitted(true);
34    }
```

No `fetch`, no Server Action, no import of `lib/api/client`. Then `:48-51`:

> Check your inbox — we sent a verification link to **{trimmed}**.

Nothing is sent. The email is never transmitted anywhere; it is discarded on unmount. **There is no
backend for this at all** — `notification_service/main.py` is an 11-line stub whose only body is
`# TODO: add router for /notifications`, and there is no student-verification route in any service.
The route is public by design (`middleware.ts:39`), and the form has no honeypot, no rate limit and
no captcha.

This is the one item on the list that is a **truthfulness defect visible to users today**, not just
unfinished wiring.

### #9 — 10 of 11 marketing feature pages have no backend — **CONFIRMED**

All 11 pages are 8-line shells calling `<FeaturePage slug=… />`; copy lives in
`src/lib/marketing/features.ts:32+`.

| Slug | Backing endpoint today |
|---|---|
| `personal-agent` | none |
| `live-notes` | none |
| `smart-questions` | none |
| `decisions` | none — gateway maps `/api/v1/decisions` (`api_gateway/edge.py:152`) but `core_api` registers no such router |
| `action-items` | none — same, `/api/v1/actions` (`edge.py:153`) unrouted in `core_api/main.py:16-20` |
| `context-gaps` | none |
| `proactive-insights` | none |
| **`transcription`** | **`GET /api/v1/meetings/{id}/transcript` + `/speakers`** — real |
| `search` | none — no search route in any service |
| `knowledge-graph` | none |
| `integrations` | none — `integration_service/main.py` is an 11-line stub |

10 of 11 confirmed. Note this is a marketing-claim / launch-gating issue, not a wiring task: these
are static copy pages and correctly need no backend to render. The pending work is a decision about
what may be claimed pre-launch, plus the overlap flagged in §0b.

### #11 — backends with no UI — see task list D below
### #12 — loading/error/empty coverage — see §4

---

## 2. Backend inventory (the basis for wireable-now vs blocked)

Verified from the committed generated specs (`src/lib/api/generated/*.ts`, emitted from
`mytheus-backend/design/openapi/`) and cross-checked against `mytheus-backend` source.

**Endpoints that exist today:**

| Service | Path | Method |
|---|---|---|
| bff | `/bff/dashboard` | GET |
| bff | `/bff/org/chart` | GET |
| bff | `/bff/people/{user_id}` | GET |
| core-api | `/api/v1/meetings` | GET (cursor), POST |
| core-api | `/api/v1/meetings/{meeting_id}` | GET, PATCH |
| core-api | `/api/v1/meetings/{meeting_id}/participants/{clerk_user_id}` | GET |
| core-api | `/api/v1/meetings/{meeting_id}/transcript` | GET |
| core-api | `/api/v1/meetings/{meeting_id}/speakers` | GET |
| core-api | `/api/v1/meetings/{meeting_id}/speakers/{speaker_id}` | PATCH |
| core-api | `/api/v1/teams`, `/api/v1/teams/{id}/members[/{user_id}]` | GET/POST/PUT/DELETE |
| core-api | `/api/v1/departments` | GET, POST |
| core-api | `/api/v1/chat` | POST |
| meeting-service | `/api/meetings/{meeting_id}/token` | POST |
| identity-tenant | `/context/{user_id}`, `/users/{user_id}`, `/tenant-settings`, `/taxonomy/role-topics`, `/scim/v2/*` | various |

**Services that are stubs — zero routes.** Verified: `grep -E '@app\.(get|post|…)|@router\.…'`
returns nothing for any of them, and each `main.py` is 11 lines ending in a `# TODO: add routers`:

- `admin_ops_service` — `# TODO: add routers for /admin/tenant/* and /admin/ops/*`
- `billing_metering_service` — `# TODO: add routers for /billing /usage /plans /stripe/webhook`
- `integration_service` — `# TODO: add routers for /oauth/{provider} and /webhooks/{provider}`
- `notification_service` — `# TODO: add router for /notifications`

The gateway's `ROUTE_MAP` (`api_gateway/edge.py:146-180`) lists prefixes for all of these plus
`/api/v1/decisions`, `/api/v1/actions`, `/api/v1/worklogs`, `/api/v1/suggestions` — **a routed
prefix is not an implemented endpoint.** Do not read the ROUTE_MAP as an availability list.

The three `Dashboard` arms `action_items`, `decisions`, `worklogs` are typed `UnbuiltArm`
(`generated/bff.ts`, `Dashboard` schema) — shaped-but-empty on the server by design.

---

## 3. THE TASK LIST

Sizes: **S** ≤ half a day · **M** ~1–2 days · **L** ≥ 3 days.
Every task is blocked on the §0 `mytheus-web` question if it touches `(app)` or `(admin)`.

### Group A — wireable now (backend exists today)

**A1. Wire `/meetings` list to `GET /api/v1/meetings`** — **M**
Files: `src/app/(app)/meetings/page.tsx` (currently `mockMeetings` at `:6,69-75`),
`src/lib/api/present.ts`, `src/lib/api/types.ts`.
Endpoint: `GET /api/v1/meetings` → `MeetingPage` (already aliased, `types.ts:68`), cursor-paginated.
Notes: page is currently a sync component with no `async`; needs RSC + `apiSafe`. Mock groups by
`live/upcoming/past`; wire status is `scheduled|live|ended|cancelled` — `toMockStatus`
(`present.ts:43`) already maps this. Needs cursor pagination UI (mock renders all rows).
Deps: none. Unblocks A2, A3, A4.

**A2. Wire meeting detail to `GET /api/v1/meetings/{id}`, and delete the mock 404 gate** — **M**
Files: `src/app/(app)/meetings/[meetingId]/page.tsx:40-41` (the `notFound()`), `:23-25`
(`generateStaticParams` over mock ids — must be removed; route must be dynamic).
Endpoint: `GET /api/v1/meetings/{meeting_id}` → `Meeting`.
Partial only: `detail.agenda` / `detail.recap` (`mockMeetingDetails`), `decisions`, `actions`
(`:43-46`) have **no** backend — leave them on mock or render honest empties (see B2, B3).
Deps: A1 (shared presenter).

**A3. Fix the transcript-page 404 — item #2** — **S**
Files: `src/app/(app)/meetings/[meetingId]/transcript/page.tsx:56-59`.
Replace `mockMeetings.find` with the real `GET /api/v1/meetings/{id}`; the title at `:66` needs it.
Keep the mock-transcript fallback at `:60` (it is deliberate and documented at `:16-20`).
Deps: **A2** (needs the meeting fetch A2 introduces). Highest severity-to-effort ratio in Group A.

**A4. Same 404 fix for the room route** — **S**
Files: `src/app/(app)/meetings/[meetingId]/room/page.tsx:13-15, 22-23`.
Deps: A2.

**A5. Wire `/org` to `GET /bff/org/chart`** — **M**
Files: `src/app/(app)/org/page.tsx` (mock at `:5,13-15,30`), `src/lib/api/types.ts`.
Endpoint: `GET /bff/org/chart` → `OrgChart { status, departments: DepartmentNode[], unassigned_teams }`,
where `DepartmentNode { id, name, children, teams }` and `TeamNode { id, name, members: Member[] }`.
Shape maps almost 1:1 onto `mockDepartments`. Two real differences: the wire has **recursive**
`children` (the mock is flat) and `unassigned_teams` (the mock has no such concept) — both need UI.
`Member` carries `role` (domain) and `role_in_team` (`lead`/`member`), which the schema comment
explicitly warns must not be conflated; `mockOrgContext.org.name` at `:21` should come from
`resolveOrgContext()`. Also needs an `empty`/`degraded` branch on `status`.
Deps: none.

**A6. Type `livekit-token` against the generated schema — item #4** — **S**
Files: `src/app/api/livekit-token/route.ts:31-35`, `src/lib/realtime/useLiveKitRoom.ts:46-50,85,120`.
Delete both hand-written `TokenResponse`/`MintedToken`; import
`components["schemas"]["TokenResponse"]` from `generated/meeting-service.ts`. The `?? configuredUrl`
at `:120` becomes a type error, which forces the real decision at `:85`: should the hook attempt a
connect when `NEXT_PUBLIC_LIVEKIT_URL` is unset, given the server always supplies `livekit_url`?
Deps: none. **Do this before A7** — it changes the gate A7 depends on.

**A7. Re-mint the LiveKit token on reconnect — item #5, second half** — **M**
Files: `src/lib/realtime/useLiveKitRoom.ts:103-137`.
Add a `RoomEvent.Reconnecting` / `SignalReconnecting` handler that calls `mintToken` and reconnects,
delivering what `:6-11` already claims. Also fix the render-phase `roomRef.current` read at `:139`
(hold the room in state, or expose it via the status union).
Deps: A6.

**A8. Refresh the Clerk token for the meeting socket — item #5, first half** — **M**
Files: `src/components/meeting/MeetingRoom.tsx:44-54`, `src/lib/realtime/useMeetingSocket.ts:66-140`.
Options: (a) re-fetch on an interval shorter than the JWT TTL and let the existing
`[url, token, meetingId]` deps recreate the socket — simple, but drops the socket periodically,
which conflicts with the "ONE SOCKET, EVER" rule at `:5-10`; (b) pass a token *provider* into
`io({ auth: cb })` so Socket.io calls it on each reconnect — correct, and does not churn the
connection. Prefer (b). Add a reconnect/backoff path out of the latched `"error"` state at `:117-119`.
Deps: none, but pairs naturally with A7.

**A9. Consume `check_participant` for a real pre-join gate** — **S**
Files: new; `src/app/(app)/meetings/[meetingId]/room/page.tsx`.
Endpoint: `GET /api/v1/meetings/{meeting_id}/participants/{clerk_user_id}`.
Today the room renders for anyone who can guess a URL, and only the token mint enforces membership.
Deps: A4.

### Group B — blocked, no backend at all

**B1. `/tasks`** — **M when unblocked**
`src/app/(app)/tasks/page.tsx:10` (`mockActions`), plus `src/components/app/dashboard/ActionItems.tsx:3`.
No endpoint. `/api/v1/actions` is a ROUTE_MAP prefix (`edge.py:153`) with no `core_api` router.
Dashboard arm `action_items` is `UnbuiltArm`. Checkbox at `tasks/page.tsx:28-57` is visual-only, as
its own comment (`:12-14`) says. **Blocked on: core-api actions router + Server Action.**

**B2. `/decisions`** — **M when unblocked**
`src/app/(app)/decisions/page.tsx:6`, `src/components/app/dashboard/RecentDecisions.tsx:4`.
No endpoint; `/api/v1/decisions` prefix only. Arm is `UnbuiltArm`. The `FOLLOWS_FROM` lineage the
page renders (`:34-35`) has no wire representation anywhere. **Blocked on: core-api decisions router.**

**B3. Meeting outcomes on the detail page** — **S when unblocked**
`meetings/[meetingId]/page.tsx:43-46,117-186` — recap, agenda, decisions, actions.
**Blocked on: B1 + B2, plus an agenda/recap endpoint that does not exist in any spec.**

**B4. `/search`** — **L when unblocked**
`src/app/(app)/search/page.tsx:5,26-33` — client-side `.includes()` over `mockSearchIndex`.
**No search endpoint exists in any service.** Not merely unwired: unspecified. Needs a backend
design decision first (semantic vs lexical, what is indexed, tenant scoping).

**B5. `/billing`** — **M when unblocked**
`src/app/(app)/billing/page.tsx:2,22-23`. `billing_metering_service/main.py` is an 11-line stub.
"Change plan" button (`:43-45`) is inert. **Blocked on: billing-metering-service routers.**

**B6. `/settings/integrations`** — **M when unblocked**
`src/app/(app)/settings/integrations/page.tsx:8,16-30`. Toggles mutate local state only (comment at
`:10-13` admits it). `integration_service/main.py` is a stub; OAuth flow does not exist.
**Blocked on: integration-service `/oauth/{provider}`.**

**B7. `/admin` and `/ops`** — **L when unblocked**
`src/app/(admin)/admin/page.tsx:2`, `src/app/(admin)/ops/page.tsx:5,12`.
`admin_ops_service/main.py` is a stub. `/ops` is gated by a hardcoded
`const OPS_CONSOLE_ENABLED = true` (`ops/page.tsx:12`) — a flag with no role behind it.
**Blocked on: admin-ops-service + a real role source (see C1).**

### Group C — auth, security, correctness

**C1. Role-gate `(admin)` — item #3** — **M**
Files: `src/app/(admin)/layout.tsx:12-18`, `src/lib/api/org-context.ts:34-48`.
Two halves: (i) add `resolveOrgContext()` + `redirect()` to the admin layout mirroring
`(app)/layout.tsx:26-27`, and replace `mockOrgContext` at `:18`; (ii) get a **real role**, since
`org-context.ts:46` hardcodes `role: "member"` on purpose.
**Blocked on: identity-tenant-service `/context/{user_id}` being reachable through the gateway** —
that endpoint exists in the spec, so this may be wireable now; verify the gateway proxies `/context`
(`edge.py:166` says it does). Also delete the now-false comment at `layout.tsx:12-15`.
Also replace `OPS_CONSOLE_ENABLED` (`ops/page.tsx:12`) with the role check.

**C2. `/students/verify` — item #8** — **S** (UI) / **M** (with backend)
`src/app/(marketing)/students/verify/page.tsx:30-34,48-51`.
Minimum honest fix today: stop claiming an email was sent. Either say "we'll email you when student
verification opens", or capture the address somewhere real. Add a honeypot and rate limiting before
this becomes a live public endpoint. **Full fix blocked on: notification-service.**

**C3. Remove `generateStaticParams` from tenant-scoped routes** — **S**
`meetings/[meetingId]/page.tsx:23-25`, `room/page.tsx:13-15`. Prerendering five mock ids on
per-tenant routes is wrong independent of the 404 bug.
Folded into A2/A4 if those are done first.

### Group D — missing pages for backends that exist (item #11)

**D1. Person profile page — `GET /bff/people/{user_id}`** — **M**
`PersonProfile { user_id, org_id, email, role, avatar_url, bio, department, team[], topics[],
skills[], recent_meetings[] }` is fully specified and served, with **no UI anywhere**. Nothing in
`src` references `/bff/people`. Natural route: `/people/[userId]`, linked from `/org` (A5) and from
participant avatars on the meeting detail page.
Deps: none. Best backend-exists-but-no-UI ratio on the list.

**D2. In-meeting chat persistence — `POST /api/v1/chat`** — **M**
`MeetingRoom.tsx:333-346` (`ChatPanel`) pushes to local state and drops on unmount. The endpoint
exists and is documented as the canonical writer of `chat_messages`; the realtime gateway relays
for latency. Needs the socket path for reads (A8) plus a Server Action for the durable write.
Deps: A8.

**D3. Tenant settings — `GET/PATCH /tenant-settings`** — **M**
`src/app/(app)/settings/page.tsx` is entirely local `useState`. The endpoint exists in
`identity-tenant-service`. Also `/taxonomy/role-topics` would back the "focus" field at
`settings/profile/page.tsx:22`, and `/users/{user_id}` would back the profile save at `:19-23`
(currently visual-only per its comment at `:12-14`).
Deps: A6-style generated-type adoption for `identity-tenant-service.ts`.

**D4. Team/department management — `POST /api/v1/teams`, `/departments`, member PUT/DELETE** — **M**
Write endpoints exist; `/org` is read-only even after A5.
Deps: A5.

### Group E — mock removal (item #10)

`src/lib/mock/data.ts` — 534 lines, **41 exports** (25 values + 16 types), imported by **24 files**
(the brief said 14; `grep -rn "lib/mock" src` returns 24 file-level import sites, including
type-only imports).

**Two exports must NOT be deleted — they are permanent marketing fixtures:**
`src/components/landing/Statement.tsx:18-19` imports `mockTeamCapacity` and `mockSourcedAnswer` and
renders them at `:314, 1037-1065, 1214, 1397-1421`. That is landing-page illustration, not unwired
product data. `lib/api/org-context.ts:22` and `lib/api/present.ts:15` also import **types** from
this module, so it can never be deleted outright without moving those types first.

**Safe removal order — each step compiles on its own:**

**E0 (do first, S).** Split the module. Move `mockTeamCapacity`, `mockSourcedAnswer`,
`mockAgentSidebar` (and their types) to `src/lib/marketing/fixtures.ts`; move `OrgContextValue`
(`data.ts:6`) and `MeetingStatus` (`:16`) to `src/lib/api/types.ts` so `org-context.ts` and
`present.ts` stop importing from `lib/mock`. **⚠ Conflicts with `origin/local-mytheus-rebuild`,
which edits `mockSourcedAnswer` and `mockAgentSidebar` — coordinate before starting.**

**E1 (S).** `mockTenantOps` — 1 importer (`ops/page.tsx:5`). Delete with B7.
**E2 (S).** `mockBilling` — 1 importer (`billing/page.tsx:2`). Delete with B5.
**E3 (S).** `mockIntegrations` — 1 importer (`settings/integrations/page.tsx:8`). Delete with B6.
**E4 (S).** `mockTenants` — 1 importer (`admin/page.tsx:2`). Delete with B7.
**E5 (S).** `mockSearchIndex` — 1 importer (`search/page.tsx:5`). Delete with B4.
**E6 (S).** `mockDepartments` — 1 importer (`org/page.tsx:5`). Delete with A5.
**E7 (S).** `mockOrgContext` — 2 importers (`(admin)/layout.tsx:4`, `org/page.tsx:5`). Delete with C1 + A5.
**E8 (M).** `mockDecisions` — 3 importers (`decisions/page.tsx:6`, `RecentDecisions.tsx:4`,
`StatGrid.tsx:3`). Delete with B2.
**E9 (M).** `mockActions` — 4 importers (`tasks/page.tsx:10`, `dashboard/page.tsx:10`,
`ActionItems.tsx:3`, `StatGrid.tsx:3`). Delete with B1.
**E10 (M).** `mockMeetingDetails`, `mockTranscriptChunks` — `[meetingId]/page.tsx:11,14`,
`transcript/page.tsx:8`. Delete with A2 + A3 + B3. `mockTranscriptChunks` also backs
`mockAsChunks()` (`transcript/page.tsx:28-49`), the deliberate no-gateway fallback — keep it until
a gateway is reachable in dev.
**E11 (L, last).** `mockMeetings` — 6 importers: `meetings/page.tsx:6`, `[meetingId]/page.tsx:10`,
`room/page.tsx:4`, `transcript/page.tsx:8`, `TodaysMeetings.tsx:6`, `StatGrid.tsx:3`. Delete only
after A1–A4. This is the keystone: three routes' `notFound()` gates and both dashboard widgets
depend on it.
**E12 (M).** `mockRoomParticipants`, `mockCaptions`, `mockRoomChat` — `MeetingRoom.tsx:15-17`. These
are the *fallbacks* for LiveVideoGrid/LiveCaptions (#7). Delete only once
`NEXT_PUBLIC_LIVEKIT_URL` and `NEXT_PUBLIC_REALTIME_WS_URL` are set in a real environment (E1 of
Group F), otherwise the room renders empty.

**Ordering rule:** every `mockX` deletion is downstream of its wiring task. Doing E-work first
produces empty screens, not progress.

### Group F — infrastructure & environment

**F1. Add the missing Docker build args — item #6** — **S**
`Dockerfile:29-35`: add `NEXT_PUBLIC_REALTIME_WS_URL`, `NEXT_PUBLIC_LIVEKIT_URL`,
`NEXT_PUBLIC_CLERK_SIGN_IN_URL`, `NEXT_PUBLIC_CLERK_SIGN_UP_URL`, `NEXT_PUBLIC_APP_ENV`; thread them
through `.github/workflows/ci.yml`'s `docker/build-push-action` `build-args`.
Deps: none. **Gates #7 entirely** — until this ships, no container can run live video or captions.

**F2. Provision the realtime/LiveKit environment** — **M**, external dependency
Un-comment `.env.example:31-32`, set the vars in `.env.local` and in deploy. `useLiveKitRoom.ts:13-19`
records the blocker: *"there is no LiveKit account anywhere in the program yet (the A1 blocker)."*
Deps: F1 + a LiveKit account. Gates A7, E12.

**F3. Add `generate:api:check` to CI** — **S**
`.github/workflows/ci.yml` runs `npx tsc --noEmit` and `npx next lint` only. The generation script
(`scripts/generate-api-types.mjs:19-22`) states that `--check` "is what stops a committed copy from
silently becoming a second source of truth" — and it is not wired into CI, so the guarantee does not
hold. Note the script needs `MYTHEUS_OPENAPI_DIR` or a sibling checkout (`:46-50`), so CI must check
out `mytheus-backend` or the job must be conditional.

**F4. Add test tooling** — **L**
`package.json:5-14` has no test script; no vitest, jest or playwright in dependencies. The CI file
says so itself: *"Vitest and Playwright are deliberately absent: there is no test tooling and no
test files."* Zero tests exist in the repo. Also no `typecheck` npm script (CI calls `npx tsc`
directly).

---

## 4. Item #12 — loading / error / empty state coverage

`find src/app -name 'error.tsx' -o -name 'loading.tsx' -o -name 'not-found.tsx' -o -name 'global-error.tsx'`
returns exactly **two** files:

- `src/app/(app)/loading.tsx`
- `src/app/not-found.tsx`

**There is no `error.tsx` anywhere and no `global-error.tsx`.** Every thrown `ApiError` from
`api()` (`client.ts:152`) in any RSC produces Next's default error page. This matters most on
`transcript/page.tsx`, which **deliberately rethrows** `ApiError` (`transcript.ts:81-85`) so a real
404 is not papered over — and there is no boundary to catch it.

Per-page state coverage:

| Page | Loading | Error | Empty |
|---|---|---|---|
| `/dashboard` | segment `loading.tsx` | ✗ no `error.tsx`; uses `apiSafe` so reads degrade (`:36,45`) | partial — only `TodaysMeetings` handles `degraded` vs `empty` (`TodaysMeetings.tsx:150,175-191`); `ActionItems`, `RecentDecisions`, `StatGrid`, `AgentActivityStrip` take no status prop at all |
| `/meetings` | segment | ✗ | ✓ `EmptyState` (`:89-94`) |
| `/meetings/[id]` | segment | ✗ | ✓ outcomes empty (`:179-185`); agenda empty (`:111-113`) |
| `/meetings/[id]/transcript` | segment; `force-dynamic` with **no `<Suspense>`** around the awaited fetch (`:59`) | ✗ — and it rethrows `ApiError` | ✓ (`:78-86`) |
| `/meetings/[id]/room` | segment | ✗ | n/a — degrades to mock (#7) |
| `/decisions` | segment | ✗ | ✓ (`:38-43`) |
| `/tasks` | segment | ✗ | ✓ `EmptyState` imported `:8` |
| `/search` | client-side, instant | ✗ | ✓ (`:58-61`) |
| `/org` | segment | ✗ | ✗ no empty branch |
| `/billing` | segment | ✗ | ✗ |
| `/settings`, `/settings/profile`, `/settings/integrations` | segment | ✗ | ✗ |
| `/admin`, `/ops` | ✗ **no `(admin)/loading.tsx`** | ✗ | `/ops` has one (`:20-27`), `/admin` none |
| `(marketing)/*` | ✗ | ✗ | n/a (static) |

**Pending work, F5–F7:**
- **F5 (M)** — add `error.tsx` to `(app)`, `(admin)` and the root, branching on `ApiError.code`
  (`FORBIDDEN_ROLE` vs `NOT_FOUND` vs `DEPENDENCY_UNAVAILABLE` are different screens; `client.ts:16-18`
  says exactly this and nothing acts on it).
- **F6 (S)** — add `(admin)/loading.tsx`.
- **F7 (M)** — thread `ArmStatus` into `ActionItems`, `RecentDecisions` and `StatGrid` so
  `degraded ≠ empty` holds across the dashboard, not just in one widget. Currently blocked by B1/B2
  (those arms are `UnbuiltArm`), so do it as part of B1/B2.

---

## 5. What else is pending that the brief misses

**G1. The dashboard renders two different "now"s simultaneously.** — **S**
`dashboard/page.tsx:47` uses `const now = new Date()` (real clock) for the greeting, `formatDayLong`
and `todayCount`. But `StatGrid`, `TodaysMeetings` and `ActionItems` compute against
`MOCK_NOW = new Date("2026-07-01T14:30:00Z")` (`components/app/dashboard/format.ts:9`), which is the
day `mtg_1` is live (`data.ts:33`). Today being 2026-09-07, the header reads
"…, September 7 · **0 meetings today**" directly above a stat grid showing the July-1 week as
current. This is visible on every load and is not a wiring gap — it is a bug introduced by wiring
half the page. Fix as part of A1, or immediately as a standalone.
*Related:* `origin/local-mytheus-rebuild` solves the same class of problem for the landing page with
`src/lib/mock/time.ts` (`useLandingTime`) — worth reusing rather than reinventing.

**G2. `ComingSoon.tsx` is a true dead file.** — **S**
`src/components/ComingSoon.tsx`, zero importers. Delete, or use it for the B-group screens.

**G3. Two hand-written duplicates of one generated type.** — folded into A6
`livekit-token/route.ts:31-35` and `useLiveKitRoom.ts:46-50` both restate `TokenResponse`. The repo
went to the trouble of generating types precisely to stop this (`types.ts:15-19` documents the last
time hand-written types drifted and cost the `CONSENT_REQUIRED`/`NOT_A_PARTICIPANT` distinction).

**G4. The `(admin)` comment is actively misleading.** — folded into C1
`(admin)/layout.tsx:12-15` says "Auth isn't wired up yet, so there's no gate at all right now."
Auth **is** wired (`middleware.ts`), and `(app)/layout.tsx:14-16` says so. A reader trusting the
admin comment would conclude `/admin` is unauthenticated. Fix the comment even before the role check.

**G5. `.env.example` omits `NEXT_PUBLIC_SENTRY_DSN`'s consumer.** — **S**
`.env.example:33` lists `NEXT_PUBLIC_SENTRY_DSN` but nothing in `src` reads it and Sentry is not in
`package.json`. Either wire error reporting (which pairs with F5) or drop the line.

**G6. No accessibility or performance gate.** — **M**
Two Three.js/canvas landing components (`FlowField.tsx`, `ConnectedGraph.tsx`, `three@0.185.1`) with
no bundle budget, no Lighthouse check, no axe run in CI.

**G7. `next@14.2.35` on React 18, pinned to Node 20.** — **M**
`Dockerfile:6-11` documents that the build breaks on Node 25 because `routes-manifest.json` is never
emitted. That is a pin against a known-broken combination, not a fix. A Next 15 / React 19 upgrade
is unscheduled work that will collide with `origin/local-mytheus-rebuild`'s framer-motion and Lenis
usage.

**G8. `useLiveKitRoom` reads a ref during render.** — folded into A7
`:139` returns `roomRef.current`. Works today by accident of render ordering; not concurrent-safe.

---

## 6. Suggested sequencing

1. **§0 decision** — is `mytheus-web` the product app? Everything in A/B/C/D/E hangs on this.
2. **F1** (Docker build args) and **A6** (type the token) — small, unblock others, zero conflict
   with the rebuild branch.
3. **G1** (the two-clocks bug) and **G4** (the misleading admin comment) — visible defects, minutes each.
4. **A1 → A2 → A3/A4** — the meetings spine; this is what makes the transcript work land for real.
5. **A5 + D1** — org chart and person profile; the largest backend-exists/no-UI gap.
6. **C1** (admin role gate), then **C2** (students/verify honesty).
7. **F5/F6** error boundaries, **F3** generate-check in CI, **F4** test tooling.
8. **E0 → E1…E12** mock removal, each step trailing its wiring task.
9. **B1–B7** as their backends land.
