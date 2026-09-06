# mytheus-ui — Integration / Plumbing Layer Audit

READ-ONLY audit. No file in `mytheus-ui` was modified; no build, no install, no test run.
All claims below are from source, cited as `path:LINE`. Repo state at time of audit:
branch `main`, HEAD `2e298c5 feat(api): generate the wire types from the emitted OpenAPI specs`.

Scope covered: `src/lib/api/` (incl. `generated/`), `src/lib/realtime/`, `src/app/api/`,
server actions, `src/middleware.ts`, auth, env config, `scripts/generate-api-types.mjs`,
`next.config.mjs`, `Dockerfile`.

---

## BACKEND BASE URLS THE UI TARGETS

Exactly **three** distinct outbound origins, all env-driven. **Zero hardcoded URLs anywhere in
`src/`** — verified by grepping `https?://|wss?://` across `src` with `generated/` excluded:
no hits.

| # | Origin | Env var | Read at | Protocol | Caller side |
|---|---|---|---|---|---|
| 1 | api-gateway | `API_GATEWAY_URL` ?? `NEXT_PUBLIC_API_GATEWAY_URL` | `src/lib/api/client.ts:69` | HTTPS REST | server only |
| 2 | realtime-gateway | `NEXT_PUBLIC_REALTIME_WS_URL` | `src/lib/realtime/useMeetingSocket.ts:60` | Socket.io WS | browser |
| 3 | LiveKit SFU | `NEXT_PUBLIC_LIVEKIT_URL`, overridden by server-returned `livekit_url` | `src/lib/realtime/useLiveKitRoom.ts:69`; override applied `useLiveKitRoom.ts:120` | WebRTC | browser |

There is **one** REST client — `src/lib/api/client.ts` — and no ad-hoc `fetch` to any backend.
Only three `fetch(` call sites exist in the entire tree:

- `src/lib/api/client.ts:141` — the single gateway wrapper
- `src/lib/realtime/useLiveKitRoom.ts:53` — same-origin, to this app's own route handler
- `src/lib/realtime/useMeetingSocket.ts:83` — same-origin, to this app's own route handler

`client.ts:68-77` refuses to default the origin and throws with an explicit rationale
("a wrong default would send real session tokens to an unintended host"). The base URL is
trailing-slash-normalised at `client.ts:76`.

`next.config.mjs` contains only `output: "standalone"` (`next.config.mjs:5`) — no rewrites,
no proxy, no `env` block. So there is no hidden fourth origin behind a rewrite.

---

## EVERY OUTBOUND CALL

Complete list. 8 call sites total, of which **4 cross the trust boundary** to a real backend
and 4 are same-origin/internal.

| # | file:line | method + path | target service | typed? | auth attached? |
|---|---|---|---|---|---|
| 1 | `src/app/(app)/dashboard/page.tsx:36` | `GET /bff/dashboard` | bff (via api-gateway) | **YES** — `DashboardResponse` = `Bff["Dashboard"]`, `src/lib/api/types.ts:60` | YES — `Authorization: Bearer <clerk>` (RSC) |
| 2 | `src/lib/api/transcript.ts:43-46` | `GET /api/v1/meetings/{meeting_id}/transcript?limit=500[&cursor=…]` | core-api (via gateway) | **YES** — `CoreApi["TranscriptPage"]`, `transcript.ts:23` | YES — Bearer |
| 3 | `src/lib/api/transcript.ts:55-57` | `GET /api/v1/meetings/{meeting_id}/speakers` | core-api (via gateway) | **PARTIAL** — items are `CoreApi["MeetingSpeaker"]` (`transcript.ts:27`), but the `{ items: … }` envelope is hand-written inline at `transcript.ts:55` | YES — Bearer |
| 4 | `src/app/(app)/meetings/[meetingId]/transcript/actions.ts:33-36` | `PATCH /api/v1/meetings/{meeting_id}/speakers/{speaker_id}` body `{ user_id }` | core-api (via gateway) | **NO** — `api()` called with no type argument; request body hand-shaped | YES — Bearer + auto `Idempotency-Key` (`client.ts:137-139`) |
| 5 | `src/app/api/livekit-token/route.ts:50-53` | `POST /api/meetings/{meeting_id}/token` | meeting-service (via gateway) | **NO** — hand-written `TokenResponse` at `route.ts:31-35`; ignores `generated/meeting-service.ts` | YES — Bearer + auto `Idempotency-Key` |
| 6 | `src/lib/realtime/useLiveKitRoom.ts:53` | `POST /api/livekit-token` body `{ meeting_id }` | this app's own Next route handler | hand-written `MintedToken`, `useLiveKitRoom.ts:46-50` | Clerk session cookie (same-origin) |
| 7 | `src/lib/realtime/useMeetingSocket.ts:83` | `GET /api/meeting-transcript?meeting_id=…` | this app's own Next route handler | `{ items?: CaptionPayload[] }`, `useMeetingSocket.ts:87` | Clerk session cookie (same-origin) |
| 8 | `src/lib/realtime/useMeetingSocket.ts:75` | Socket.io connect `${NEXT_PUBLIC_REALTIME_WS_URL}/v1` | realtime-gateway | hand-written payload/event types (`captions.ts:29-37`, `useMeetingSocket.ts:53-57`) | **YES** — raw Clerk JWT in `handshake.auth.token` |

**Summary of real product surface:** two REST reads (`/bff/dashboard`, transcript+speakers),
one REST write (speaker assignment), one token mint. That is the whole thing.

### Endpoints that exist in the generated specs but have ZERO callers

- `/bff/org/chart` (`generated/bff.ts:35`)
- `/bff/people/{user_id}` (`generated/bff.ts:55`)
- `/api/v1/chat` (`generated/core-api.ts:15`)
- `/api/v1/departments` (`generated/core-api.ts:32`)
- `/api/v1/meetings` list + `/api/v1/meetings/{id}` detail (`core-api.ts:50`, `:74`)
- `/api/v1/meetings/{id}/participants/{clerk_user_id}` (`core-api.ts:92`)
- `/api/v1/teams`, `/api/v1/teams/{id}/members`, `/api/v1/teams/{id}/members/{user_id}` (`core-api.ts:235,261,281`)
- The **entirety** of identity-tenant-service: `/context/{user_id}`, `/users/{user_id}`, `/scim/v2/*`, `/taxonomy/role-topics`, `/tenant-settings` (`generated/identity-tenant-service.ts:32-161`)

### Path-routing verification (source, not docs)

`src/app/api/livekit-token/route.ts:16-24` carries a comment claiming the documented path
`POST /api/v1/meetings/:id/token` is **wrong** and the shipped path is `/api/meetings/:id/token`.
Verified against the backend rather than accepted:

- `mytheus-backend/services/api_gateway/edge.py:148` → `"/api/v1/meetings": "core-api"`
- `mytheus-backend/services/api_gateway/edge.py:161` → `"/api/meetings": "meeting-service"`
- `edge.py:156-160` documents the near-collision as deliberate
- Longest-prefix resolution at `edge.py:186-189`
- `generated/meeting-service.ts:15` confirms the only token path is `/api/meetings/{meeting_id}/token`

**The UI is correct and the plan doc / CLAUDE.md is the bug.** Following the documented path
would have produced a client that never mints a token at all.

---

## GENERATED TYPES: USED OR IGNORED?

**Mixed — half connected, half dead.** Not "generated and ignored" wholesale; the typed-client
work IS wired up for the two services the UI actually calls today.

| generated file | LOC | import sites | status |
|---|---|---|---|
| `src/lib/api/generated/core-api.ts` | 1552 | **3** | USED |
| `src/lib/api/generated/bff.ts` | 527 | **1** | USED |
| `src/lib/api/generated/identity-tenant-service.ts` | 924 | **0** | GENERATED AND IGNORED |
| `src/lib/api/generated/meeting-service.ts` | 242 | **0** | GENERATED AND IGNORED |

**Total usage count: 4 import statements** in the whole codebase, all `import type`:

1. `src/lib/api/types.ts:25` — `import type { components as BffSchemas } from "./generated/bff"`
2. `src/lib/api/types.ts:26` — `import type { components as CoreApiSchemas } from "./generated/core-api"`
3. `src/lib/api/client.ts:23` — `import type { components as CoreApiSchemas } from "./generated/core-api"`
4. `src/lib/api/transcript.ts:15` — `import type { components as CoreApiSchemas } from "@/lib/api/generated/core-api"`

Everything downstream re-exports through `src/lib/api/types.ts` (`ErrorCode`, `Problem`,
`ArmStatus`, `DashboardMeeting`, `DashboardResponse`, `MeetingStatus`, `Participant`,
`ApiMeeting`, `MeetingPage`, `ApiTeam`) or through `transcript.ts` (`TranscriptChunk`,
`TranscriptPage`, `MeetingSpeaker`). Consumers found at `components/app/dashboard/TodaysMeetings.tsx:7-8`,
`components/meeting/TranscriptView.tsx:6`, `components/meeting/SpeakerMapping.tsx:21`.

### The one real drift defect

`src/app/api/livekit-token/route.ts:31-35` hand-writes:

```ts
interface TokenResponse {
  token: string;
  room: string;
  livekit_url?: string | null;
}
```

while `src/lib/api/generated/meeting-service.ts:100-107` declares:

```ts
TokenResponse: {
  livekit_url: string;   // REQUIRED, non-nullable
  room: string;
  token: string;
};
```

This is exactly the drift class the generator exists to prevent, reintroduced in the one file
that had a generated type sitting right there and did not use it.

**Downstream consequence:** `src/lib/realtime/useLiveKitRoom.ts:120` does
`room.connect(minted.livekit_url ?? configuredUrl, minted.token)`. Since the server always
sends a URL, the `?? NEXT_PUBLIC_LIVEKIT_URL` fallback is **dead code**. The comment at
`useLiveKitRoom.ts:117-119` describing the public env var as "the fallback for a single-region
dev setup" describes a path that can never execute.

### What the generation pipeline does right

- Output is committed and git-tracked (`git ls-files src/lib/api/generated` returns all four).
- A staleness gate exists: `scripts/generate-api-types.mjs:139-159`, run via
  `npm run generate:api:check` (`package.json` scripts block).
- The gate compares content, not bytes-as-written-by-platform: LF normalisation on both write
  (`generate-api-types.mjs:119`) and read (`:131`), backed by `.gitattributes`
  (`src/lib/api/generated/*.ts text eol=lf`) so `core.autocrlf=true` cannot make the check red
  on one machine and green in CI.
- The output directory is `rmSync`'d and rewritten whole (`generate-api-types.mjs:163-164`), so
  a spec that stops being emitted cannot leave a stale module behind for an import to resolve against.

---

## scripts/generate-api-types.mjs — SPEC SOURCE AND CROSS-REPO COUPLING

**Confirmed fragile cross-repo coupling.** It reads from a **local sibling checkout path** —
not a URL, and not a committed copy of the YAML specs.

Candidate paths, `scripts/generate-api-types.mjs:46-50`:

```
join(REPO, "..", "..", "mytheus-backend", "design", "openapi")
join(REPO, "..", "..", "Mytheus-backend", "reprise-backend", "design", "openapi")
join(REPO, "..",       "mytheus-backend", "design", "openapi")
```

Override: `MYTHEUS_OPENAPI_DIR` (`generate-api-types.mjs:57`). If it is set but does not exist,
the script exits 2 rather than falling through to the guesses (`:60-63`) — the right call, and
documented as such (`:53-56`). If nothing resolves, it exits 2 listing every path tried
(`:68-75`), in the same fail-loud spirit as `API_GATEWAY_URL` refusing to default.

Generation itself shells out to the local `openapi-typescript` CLI
(`generate-api-types.mjs:107-111`), reading every `*.yaml` in the resolved directory
(`:95-97`), sorted, and writing `<name>.ts` per spec (`:119`).

### Problems

1. **The staleness gate cannot run where it matters.** `generate:api:check` needs the backend
   on disk or `MYTHEUS_OPENAPI_DIR` set. The `Dockerfile` provides neither and never invokes
   the check (`Dockerfile:36` runs only `npm run build`). So the gate that stops the committed
   copy becoming a second source of truth is **unenforced in the container build**, and
   unenforceable in this repo's own CI without checking out a second repo.
2. **A workstation-shaped legacy path is hardcoded** into candidate 2:
   `Mytheus-backend/reprise-backend`. The script's own comment (`:39-44`) acknowledges this
   ("on a real workstation the clones sit under wrapper directories with legacy inner names,
   recorded as gap 33").
3. **Today it resolves by falling through to the third guess.** Candidate 1 resolves to
   `/Users/sushan/mytheus-backend` (does not exist); candidate 3 resolves to
   `/Users/sushan/Projects/mytheus-backend` (exists). So it works here by luck of ordering, not
   by the documented primary convention.
4. `MYTHEUS_OPENAPI_DIR` is **not documented in `.env.example`** — it exists only in the
   script's failure message.

The committed output is what lets a clean clone, CI and Docker compile with no sibling
checkout (`generate-api-types.mjs:19-22`). That part genuinely works.

---

## AUTH & ORG PROPAGATION

**Verdict: correct, fail-closed, and unusually disciplined about the tenant question. I found
no route reachable unauthenticated that should not be.**

### Session and token

- Clerk. Middleware is `clerkMiddleware` at `src/middleware.ts:51-57`.
- The route list at `src/middleware.ts:22-49` is an **allowlist of public routes**, not a
  denylist of protected ones. `middleware.ts:5-8` states the reasoning: with a denylist every
  new authenticated route is public until someone remembers it, and the failure is silent;
  with an allowlist a new route is protected by default and the mistake is a visible redirect.
- `(app)` and `(admin)` are deliberately absent from the allowlist — that absence is what
  protects them (`middleware.ts:19-20`).
- `await auth.protect()` at `middleware.ts:56` for everything not matched.
- Matcher at `middleware.ts:60-65` covers everything except Next internals and static assets,
  **plus an explicit `"/(api|trpc)(.*)"` entry**. So both Next route handlers are authenticated.

### Public routes (the complete allowlist, `middleware.ts:22-49`)

`/`, `/about(.*)`, `/blog(.*)`, `/careers(.*)`, `/enterprise(.*)`, `/features(.*)`,
`/pricing(.*)`, `/security(.*)`, `/privacy(.*)`, `/terms(.*)`, `/students(.*)`,
`/contact(.*)`, `/changelog(.*)`, `/docs(.*)`, `/sign-in(.*)`, `/sign-up(.*)`.

All are marketing/legal/auth surfaces. `middleware.ts:31-34` records a real prior bug: the list
said `/legal(.*)`, which matches no route in this app, so `/privacy` and `/terms` — the two
documents that must be publicly reachable and are both linked from the footer — were protected
and 404'd to signed-out visitors. Fixed.

### Token attachment

`src/lib/api/client.ts:129-139`:

```ts
const { getToken } = await auth();
const token = await getToken();
const headers: Record<string, string> = { Accept: "application/json" };
if (token) headers.Authorization = `Bearer ${token}`;
```

Server-only **by construction**, and `client.ts:10-14` states this is a security property
rather than a style choice: importing `@clerk/nextjs/server` from a client component fails the
build instead of shipping a token-fetching path into the browser bundle.

Client-side token for the socket only: `src/components/meeting/MeetingRoom.tsx:43-53`, via
`useAuth().getToken()` from `@clerk/nextjs`.

### org_id — never sent, deliberately

- `src/lib/api/client.ts:6-9`: "IT NEVER SENDS `org_id`. There is deliberately no parameter for
  one. `api-gateway` derives the tenant from the verified JWT and resolves it through
  `organizations.clerk_org_id`; a client-supplied tenant is a value the server must either
  ignore or trust, and trusting it is a cross-tenant read." Confirmed — `RequestOptions`
  (`client.ts:56-64`) has no tenant field and nothing in `buildUrl`/`api` adds one.
- `src/middleware.ts:10-11` repeats it: nothing tenant-related for the middleware to attach.
- `src/lib/api/org-context.ts:17-19`: `org.id` is **Clerk's opaque organization id, display
  key only** — explicitly not the internal `org_id`. Confirmed at `org-context.ts:25` and `:39`.

This is the right design and it is actually implemented, not merely commented.

### Holes

1. **No token refresh on the socket — highest-severity finding.**
   `src/components/meeting/MeetingRoom.tsx:45-53` fetches the Clerk token **once** on mount into
   `useState`. `useMeetingSocket`'s effect dependency array is `[url, token, meetingId]`
   (`useMeetingSocket.ts:140`), so the token value never changes and the effect never re-runs.
   Clerk session tokens are short-lived (~60s). Socket.io's automatic reconnect re-handshakes
   with `auth: { token }` captured at connect time (`useMeetingSocket.ts:75-78`) — i.e. a
   **stale token** — which the gateway will reject. The room then silently degrades with no
   recovery path. Needs a token *supplier* invoked per handshake, not a captured string.
2. **LiveKit has the same shape of gap.** `useLiveKitRoom.ts:113-115` mints fresh on every
   effect run and never caches — correct — but the effect deps are
   `[meetingId, configuredUrl, syncParticipants]` (`:137`), and neither
   `RoomEvent.Reconnecting` nor `RoomEvent.Disconnected` re-mints (`:103-111`). The file's own
   header (`:6-11`) names "re-mint the token on reconnect" as the requirement. `livekit-client`
   does refresh a *live* connection internally, so this only bites on a full drop past the TTL —
   but that is precisely the case the header says it is solving.
3. **`org.plan` and `user.role` are hardcoded constants.** `src/lib/api/org-context.ts:40`
   returns `plan: "starter"`; `:46` returns `role: "member"`. Both are documented as
   least-privilege placeholders (`org-context.ts:8-16`) because plan needs `subscriptions` and
   role is a permission living in `rbac_grants`. Not a security hole — it under-grants, so a
   gated feature hides rather than leaks — but any role- or plan-gated UI is currently gating
   on a constant.
4. `src/lib/api/org-context.ts:26` returns `null` when signed out with the comment "middleware
   should already have redirected". True given the allowlist, but it is a second-line assumption
   rather than a check.

---

## NEXT API ROUTES — SECRETS HELD, LOGIC DUPLICATED?

Only two route handlers exist (`find src/app/api -type f`: `.gitkeep`,
`meeting-transcript/route.ts`, `livekit-token/route.ts`).

**Neither holds a secret. Neither fakes data. Both are thin authenticated proxies. This is the
correct architecture.**

### `src/app/api/livekit-token/route.ts`

- **Holds no LiveKit secret.** `route.ts:11-14` states `LIVEKIT_API_KEY`/`LIVEKIT_API_SECRET`
  live in `meeting-service` and never leave the backend. Verified: grepping `process.env` across
  the entire repo returns only four hits — `client.ts:69`, `useLiveKitRoom.ts:69`,
  `useMeetingSocket.ts:60`, `generate-api-types.mjs:57`. **There is no `LIVEKIT_API_*` read
  anywhere in this repo.**
- Reason it exists (`route.ts:4-9`): `lib/api/client.ts` is server-only by construction, and the
  meeting room is a client island that needs a token at connect and on every reconnect, so it
  needs a same-origin endpoint. Legitimate.
- Validates the body: JSON parse failure → 400 `MALFORMED_BODY` (`:39-43`); non-string or empty
  `meeting_id` → 400 `VALIDATION_FAILED` (`:45-47`).
- **Duplicates no backend logic.** It forwards and passes the gateway's own status and stable
  `code` through unchanged (`:61`), explicitly preserving the 403 (not a participant) /
  404 (another tenant's meeting — existence is confidential) / 428 (consent required)
  distinction (`:57-60`). Collapsing those would have lost exactly what the backend went to
  trouble to express.
- Only flaw: the hand-written `TokenResponse` (`:31-35`) discussed above.

### `src/app/api/meeting-transcript/route.ts`

- **Proxying to the real backend, not faking.** `route.ts:32` calls `fetchTranscript()` from
  `src/lib/api/transcript.ts`, which walks the real cursor against
  `GET /api/v1/meetings/{id}/transcript` through the gateway.
- Holds no credential of its own; forwards the caller's Clerk session (`route.ts:12-13`).
- Does one legitimate transformation (`:37-47`): reshapes `snake_case` `TranscriptChunk` rows
  into the socket's camelCase `caption` payload so the reducer takes both sources through one
  code path. `chunkId` is set to the row's `id` (`:39`) — the same identity the producer minted
  it under — so a final arriving from both REST and socket dedupes instead of rendering twice.
  `isFinal: true` is hardcoded (`:44`) and that is correct: only finals are persisted in
  `transcript_chunks`; interims never are (`route.ts:15-18`).
- Why it exists: `realtime-gateway` buffers and replays nothing to a reconnecting client, so a
  socket drop loses every caption spoken during it unless the client re-reads the durable
  record (`route.ts:5-9`).

### One thing to flag on both

`route.ts:52-55` returns `{ items: [] }` with **HTTP 200** for any non-`ApiError` failure,
justified as "No gateway configured, which is every environment today." Defensible for a
*missing* gateway — but it also swallows DNS failures, connection refusals and timeouts against
a *configured* one, making a genuine backend outage indistinguishable from "no transcript yet."
Same shape at `src/lib/api/transcript.ts:80-87`: `ApiError` rethrows (a real server answer),
everything else returns `null` meaning "fall back to mock". The distinction drawn is deliberate
and documented (`transcript.ts:61-69`), but the non-`ApiError` bucket is broader than "unset
`API_GATEWAY_URL`".

---

## REALTIME CLIENT CONTRACT

This is the precise **client-side expectation**, to be diffed against the server implementation.
Everything below is what the UI will actually do / require at runtime.

### A. Socket.io — `src/lib/realtime/useMeetingSocket.ts`

#### Connection

- **URL:** `` `${NEXT_PUBLIC_REALTIME_WS_URL.replace(/\/+$/, "")}/v1` `` — `useMeetingSocket.ts:75`.
  Namespace is **`/v1`**, appended after trailing-slash stripping.
- **Transports:** `transports: ["websocket"]` — `useMeetingSocket.ts:77`. **WebSocket only, no
  HTTP long-poll fallback.** If the gateway or any proxy in front of it requires the polling
  upgrade handshake, this client will never connect.
- **Auth:** `auth: { token }` — `useMeetingSocket.ts:76`. Server must read
  `socket.handshake.auth.token` (stated at `:72-74`). It is the **raw Clerk session JWT**.
  Not a header — the comment notes the WebSocket upgrade cannot carry one portably.
- **Gating:** the effect returns early unless both `url` and `token` are present
  (`useMeetingSocket.ts:69`). A null token means Clerk has not resolved a session; connecting
  anonymously would just be denied, so it waits rather than burning an attempt.
- **One socket, ever.** Created inside the effect, torn down by its cleanup
  (`useMeetingSocket.ts:134-139`: `removeAllListeners()` then `disconnect()`), so a remount
  cannot leak one. Rationale at `:8-11` — the gateway scales on anchor A6 = 100k concurrent
  connections, so a stray second connection per client doubles the tier's headline cost driver.

#### Client → server

```
emit "join"
  payload: { meetingId: string }              // camelCase
  ack:     { ok: boolean,
             presence?: string[],             // internal users.id UUIDs
             error?: "FORBIDDEN" | "INVALID_PAYLOAD" }
```

- Emitted at `useMeetingSocket.ts:101`; ack type at `:53-57`.
- The **ack callback is mandatory**. The client passes one (`:101-111`) and has no timeout, so a
  server that never acks leaves the UI pinned at `connecting` forever.
- A denial must be an **ack value, not a thrown exception** (`:107-110`): "The gateway always
  acks rather than throwing, so a denial is a value to render, not an exception."
- `ack.error === "FORBIDDEN"` → status `"forbidden"`; any other falsy-`ok` → status `"error"`
  (`:109`).
- **Re-emitted on every `connect`**, therefore on every automatic reconnect
  (`useMeetingSocket.ts:116`), because Socket.io reconnects with a **new socket id** and room
  membership does not survive it (`:114-115`).

#### Server → client

```
"caption"
  { meetingId: string,                        // client filters on this
    payload: {
      chunkId:     string,                    // STABLE across interims AND the final
      speakerId:   string,                    // opaque diarization label ("0", "1", …)
      startMs:     number,
      endMs:       number,
      text:        string,
      isFinal:     boolean,
      confidence?: number                     // optional
    } }

"presence"
  string[]                                    // BARE array — NOT { members: [...] }

"connect_error"                               // → status "error"
```

- `caption` handler `useMeetingSocket.ts:121-128`. Envelope shape
  `{ meetingId, payload }` is required; the client drops any event where
  `event?.meetingId !== meetingId || !event.payload` (`:126`).
- `presence` handler `useMeetingSocket.ts:130-132`: the argument is the array **itself**.
  `Array.isArray(members) ? members : []` — a wrapped object silently yields empty presence.
- `CaptionPayload` field names are declared verbatim from the gateway's own event docs
  (`src/lib/realtime/captions.ts:29-37`, sourced per `:8-13` from
  `realtime-gateway/src/server.ts`).

#### Reducer invariants the client depends on — `src/lib/realtime/captions.ts`

1. **`chunkId` is stable across an utterance's interim revisions and its own final**
   (`captions.ts:10-13`). This is what permits replace-in-place until `isFinal`, then append.
   Violate it and the strip appends half-sentences and flickers — the exact layout shift the
   plan forbids (`:15-17`).
2. **An interim is a REVISION, never a new line** (`captions.ts:15`).
3. **Ordering is by `startMs`, not arrival** (`captions.ts:18-20`, implemented `:50-56`).
   Rationale: Deepgram finalizes an utterance after later interims from a *different* speaker
   may already have arrived, so arrival order is not utterance order.
4. **Tie-break is `chunkId.localeCompare`** (`captions.ts:54`) — two speakers can genuinely
   begin on the same millisecond, and without the tiebreak their relative order flips between
   renders.
5. **`applyCaption` is idempotent on `chunkId`** (`captions.ts:58-76`) — a final arriving from
   both the REST hydrate and the socket must render once.
6. **A final never regresses to an interim** (`captions.ts:70`): if the stored line is final and
   an incoming payload is not, the state is returned unchanged. Redelivery is expected on this
   program's buses.
7. **`startMs` may be revised between an interim and its final** (`captions.ts:72-75`) — the
   superseded revision is removed and re-inserted in order, not patched in place.
8. **Speaker colour is a hash of `speakerId`**, not arrival order (`captions.ts:97-103`), so the
   same speaker keeps the same colour even after a hydrate reorders lines.

#### Re-hydration

- **The gateway replays nothing.** Stated twice (`captions.ts:21-25`, `useMeetingSocket.ts:18-21`).
  Finals are durable and re-read from core-api's REST read model.
- On **every** `connect` — hence every automatic reconnect — the client calls `rehydrate()`
  *before* emitting `join` (`useMeetingSocket.ts:100`).
- `rehydrate()` GETs `/api/meeting-transcript?meeting_id=…` (`:83-85`), and **merges** into
  current state rather than replacing (`:89-91`). Rationale at `:25-27`: a naive
  `setCaptions(hydrate(...))` would discard any caption that arrived between the fetch starting
  and finishing, which on a busy meeting is a real window.
- A failed re-hydrate is swallowed (`:92-96`) — it must not break the live stream.

#### Status machine

`"disabled" | "connecting" | "joined" | "forbidden" | "error"` (`useMeetingSocket.ts:39-44`).
Initial state is `"connecting"` if `NEXT_PUBLIC_REALTIME_WS_URL` is set, else `"disabled"` (`:61`).

#### Degradation

`NEXT_PUBLIC_REALTIME_WS_URL` unset → status `"disabled"`, **zero network activity**, caller
keeps rendering mock captions (`useMeetingSocket.ts:12-16`, `:61`, `:69`).

### B. LiveKit — `src/lib/realtime/useLiveKitRoom.ts`

#### Token mint

```
POST /api/livekit-token
  headers: { "Content-Type": "application/json" }
  body:    { meeting_id: string }
  200:     { token: string, room: string, livekit_url?: string | null }
  non-2xx: { code?: string }   → thrown as Error(problem.code ?? `token mint failed (${status})`)
```

`useLiveKitRoom.ts:52-66`. The error path deliberately preserves the backend's distinctions
(`:60-63`): never-admitted participant vs. another tenant's meeting vs. consent-required are
three different things to show a user, not one "could not join".

#### Connect

- `new Room({ adaptiveStream: true, dynacast: true })` — `useLiveKitRoom.ts:88`
- `await room.connect(minted.livekit_url ?? configuredUrl, minted.token)` — `:120`.
  **Server-returned `livekit_url` wins**, because a tenant pinned to a regional SFU by
  `data_residency.region` must connect to that one and only the server knows which (`:117-119`).
  As noted, the spec makes `livekit_url` required, so the `??` fallback never fires.
- Token is minted fresh on every effect run and **never cached in state** (`:113-115`) — correct
  for short-lived tokens.

#### Events subscribed (`useLiveKitRoom.ts:96-111`)

`ParticipantConnected`, `ParticipantDisconnected`, `TrackSubscribed`, `TrackUnsubscribed`,
`TrackMuted`, `TrackUnmuted`, `ActiveSpeakersChanged` → all re-sync the participant array.
`ConnectionStateChanged` → `Connected` sets `"connected"`, `Reconnecting` sets `"reconnecting"`.
`Disconnected` → clears participants only; explicitly **not** an error state (`:108-110`).

`syncParticipants` (`:77-82`) always builds a **new array identity** because LiveKit mutates
participant objects in place — otherwise React has nothing to diff and the grid never re-renders
on a camera/mic change.

#### Status machine

`"unconfigured" | "connecting" | "connected" | "reconnecting" | "error"` (`useLiveKitRoom.ts:31-36`).

#### Degradation

`NEXT_PUBLIC_LIVEKIT_URL` unset → `"unconfigured"`, effect returns immediately, **no network
work at all** (`:70-72`, `:85`), grid keeps showing the mock composition (`:13-19`).

### C. Consumer wiring — `src/components/meeting/MeetingRoom.tsx`

- `useLiveKitRoom(meeting.id)` — `MeetingRoom.tsx:40`
- `useAuth().getToken()` once on mount into `socketToken` — `:43-53`
- `useMeetingSocket(meeting.id, socketToken)` — `:54`

The one-shot token here is the refresh gap described under AUTH.

### UNVERIFIED (server side — another agent's scope)

Whether realtime-gateway actually serves namespace `/v1`; reads `handshake.auth.token`; accepts
websocket-only transport without a polling upgrade; always acks `join`; emits `presence` as a
bare array; wraps captions as `{ meetingId, payload }`; and honours `chunkId` stability across
interim→final. The above is the exact client-side expectation to diff against.

---

## SERVER ACTIONS

**Exactly one exists** in the whole app — `grep -rln '"use server"' src` returns a single file.

`src/app/(app)/meetings/[meetingId]/transcript/actions.ts` — `assignSpeaker(meetingId, speakerId, userId)`.

**Real backend call, not a stub.** `actions.ts:33-36`:

```ts
await api(
  `/api/v1/meetings/${encodeURIComponent(meetingId)}/speakers/${encodeURIComponent(speakerId)}`,
  { method: "PATCH", body: { user_id: userId } },
);
```

- Carries an auto-generated `Idempotency-Key` via `client.ts:137-139` (`actions.ts:7-9`).
- **`assignment` is deliberately NOT a parameter** (`actions.ts:12-17`): core-api sets
  `assignment = 'manual'` as a SQL literal and never from the request body, and a DB trigger
  (`meeting_speakers_manual_is_authoritative`) is the second guard. Passing it from here would
  invent a client-controlled path into a property the backend deliberately does not expose.
  Correct restraint.
- Error handling returns `{ ok: false, error: error.code }` for `ApiError` and
  `"DEPENDENCY_UNAVAILABLE"` otherwise (`:37-45`) — no throw into the client island.
- `revalidatePath(\`/meetings/${meetingId}/transcript\`)` on success (`:50`) so relabelled turns
  actually appear.

**Not typed** — `api()` is called with no type argument, and the request body is hand-shaped
even though `generated/core-api.ts:187` describes the endpoint.

---

## FULL UI ENV VAR LIST

Complete `process.env` read sites in the repo — only **four**, everything else is SDK-internal:
`client.ts:69`, `useLiveKitRoom.ts:69`, `useMeetingSocket.ts:60`, `generate-api-types.mjs:57`.

| Var | Public? | Read at | In `.env.example`? | Present in local `.env.local`? |
|---|---|---|---|---|
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | public | Clerk SDK; `Dockerfile:29,32` | YES `.env.example:9` | SET |
| `CLERK_SECRET_KEY` | **server** | Clerk SDK; `Dockerfile:30,33` | YES `.env.example:11` | SET |
| `NEXT_PUBLIC_CLERK_SIGN_IN_URL` | public | Clerk SDK | YES `.env.example:15` | SET |
| `NEXT_PUBLIC_CLERK_SIGN_UP_URL` | public | Clerk SDK | YES `.env.example:16` | SET |
| `API_GATEWAY_URL` | **server** | `src/lib/api/client.ts:69`; `Dockerfile:31,34` | YES `.env.example:25` | SET |
| `NEXT_PUBLIC_API_GATEWAY_URL` | public | `src/lib/api/client.ts:69` (fallback) | commented `.env.example:26` | not set |
| `NEXT_PUBLIC_REALTIME_WS_URL` | public | `src/lib/realtime/useMeetingSocket.ts:60` | commented `.env.example:31` | **NOT SET** |
| `NEXT_PUBLIC_LIVEKIT_URL` | public | `src/lib/realtime/useLiveKitRoom.ts:69` | commented `.env.example:32` | **NOT SET** |
| `NEXT_PUBLIC_SENTRY_DSN` | public | **never read anywhere** | commented `.env.example:33` | not set |
| `NEXT_PUBLIC_APP_ENV` | public | **never read anywhere** | YES `.env.example:34` | SET |
| `MYTHEUS_OPENAPI_DIR` | build-time | `scripts/generate-api-types.mjs:57` | **NO — undocumented** | n/a |
| `NEXT_TELEMETRY_DISABLED` | build/runtime | `Dockerfile:35,40` | n/a | n/a |
| `NODE_ENV` / `PORT` / `HOSTNAME` | runtime | `Dockerfile:40` | n/a | n/a |

`.env.example` **exists** and is good: it leads with the warning that `NEXT_PUBLIC_*` values are
inlined into the browser bundle at build time and are therefore public forever, and that a secret
in a `NEXT_PUBLIC_` var is exfiltratable from the shipped JS (`.env.example:3-5`).
`.env.local:25` marks `CLERK_SECRET_KEY` "SERVER ONLY. Never prefix this with NEXT_PUBLIC_."

Because `NEXT_PUBLIC_REALTIME_WS_URL` and `NEXT_PUBLIC_LIVEKIT_URL` are unset locally, **both
realtime paths are inert in this checkout** — `"disabled"` and `"unconfigured"` respectively,
by design.

### Docker gap — blocks any containerised realtime deploy

`Dockerfile:29-35` declares exactly three build args:

```
ARG NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY
ARG CLERK_SECRET_KEY
ARG API_GATEWAY_URL
```

`NEXT_PUBLIC_REALTIME_WS_URL` and `NEXT_PUBLIC_LIVEKIT_URL` are `NEXT_PUBLIC_*`, which Next
**inlines at build time**. They are not build args and not present during `npm run build`
(`Dockerfile:36`). Therefore a container built from this Dockerfile has realtime and LiveKit
**permanently hard-disabled**, and setting them in the deployment's runtime environment will
have **no effect** — the values are already baked in as `undefined`.

`Dockerfile:24-28` correctly explains why the Clerk keys must be present at build time (Clerk
validates the publishable key at build, its provider wraps the root layout, Server Components
call Clerk during prerender) and that neither is baked into the final image because the runtime
reads its own environment. The same reasoning was simply not extended to the two realtime vars.

Other Dockerfile notes: Node pinned to 20 with a documented reason (`Dockerfile:6-11` — on
Node 25 the build completes every phase then fails writing standalone output because
`routes-manifest.json` is never emitted); `npm ci` not `npm install` (`:15-17`); non-root
`nextjs` user (`:41,46`); exec-form `CMD` so node is PID 1 and SIGTERM drains cleanly (`:48-49`).
`generate:api:check` is **never invoked** in the image build.

---

## ERROR HANDLING

Shared and better than typical for this stage — with one structural absence.

### Present

- **One error type.** `ApiError extends Error` with `status` + stable `code`
  (`src/lib/api/client.ts:40-54`).
- **RFC 7807 parsing** with a status-derived fallback when the body is not a Problem
  (a proxy error page, a truncated body): `client.ts:92-117`. Fallback map at `:106-115` —
  401→`AUTH_REQUIRED`, 403→`FORBIDDEN_ROLE`, 404→`NOT_FOUND`, ≥500→`INTERNAL`,
  else `MALFORMED_BODY`.
- **Callers branch on `code`, never on message strings** (`client.ts:16-18`, `:37`) — 403 (wrong
  role) and 404 (another tenant's resource, confidential) must not collapse.
- **`ErrorCode` is generated, not hand-kept** (`client.ts:35` ← `CoreApiSchemas["schemas"]["ErrorCode"]`).
  `client.ts:27-33` records that the hand-written union had already drifted: 15 of 17 codes,
  missing `CONSENT_REQUIRED` and `NOT_A_PARTICIPANT` — precisely the two outcomes the
  livekit-token path exists to keep apart.
- **`apiSafe()` result union** (`client.ts:164-177`) so one failed panel degrades itself rather
  than the whole page hitting an error boundary; mirrors bff's per-arm status. Used at
  `dashboard/page.tsx:36`, with `:44-45` treating a failed fetch and a server-reported degraded
  arm as the same thing.
- **Auto `Idempotency-Key`** on every POST/PATCH/PUT (`client.ts:66`, `:137-139`).
- **`cache: "no-store"` by default** (`client.ts:148`) with the rationale at `:145-147` that the
  Next data cache keys on the URL, which is identical for every tenant, so caching would serve
  one tenant's dashboard to the next visitor. **A real cross-tenant leak, actively prevented.**
- **204 handled** (`client.ts:153`) rather than crashing on an empty-body JSON parse.
- **Undefined query params omitted** rather than serialised as the string `"undefined"`, which
  the server would reject (`client.ts:84-86`).
- **Bounded pagination** — `MAX_PAGES = 20` (`transcript.ts:34`) stops a pathological meeting
  from looping forever.
- Socket-side: `connect_error` → status `"error"` (`useMeetingSocket.ts:117-119`); failed
  re-hydrate is swallowed so it cannot break the live stream (`:92-96`); LiveKit mint failure →
  `status: "error"` with a human-readable reason (`useLiveKitRoom.ts:124-128`).

### Absent

1. **No retry, anywhere.** No backoff, no retry-on-5xx, no jitter. Every REST call is
   single-shot. (Socket.io's own reconnect is the only retry in the system, and it is the one
   that hits the stale-token problem.)
2. **No 401 refresh.** `ApiError.isAuthFailure` is defined at `client.ts:51-53` and **has zero
   call sites** — grep confirms. Nothing re-fetches a token or redirects on
   `AUTH_REQUIRED`/`TOKEN_EXPIRED`. In an RSC the token is minted fresh per request so this
   rarely bites; on the socket it is the primary failure mode.
3. **No timeout and no `AbortSignal`** on any fetch. A hung gateway hangs the RSC render with no
   upper bound.
4. **Serial pagination inside a render.** `fetchTranscript` (`transcript.ts:40-50`) walks up to
   20 pages sequentially with no per-request timeout — worst case 20 serial round trips during
   page render.
5. **Outage indistinguishable from empty** in two places: `meeting-transcript/route.ts:52-55`
   (200 + `{items: []}`) and `transcript.ts:80-87` (`null` → render mock).
6. **No observability.** `NEXT_PUBLIC_SENTRY_DSN` is documented in `.env.example:33` but there
   is no Sentry dependency in `package.json` and no error reporting anywhere in the tree.

---

## MISSING PLUMBING

Ordered by severity.

1. **Socket / LiveKit token refresh.** `MeetingRoom.tsx:45-53` captures the Clerk token once;
   `useMeetingSocket.ts:140` deps make it immutable; Socket.io auto-reconnect re-handshakes with
   a stale JWT. `useLiveKitRoom.ts:137` likewise never re-mints on
   `RoomEvent.Reconnecting`/`Disconnected` despite `:6-11` naming that as the requirement.
   **This is the highest-severity gap and will fail in production the first time a meeting runs
   longer than a token TTL across a network blip.**
2. **`NEXT_PUBLIC_REALTIME_WS_URL` and `NEXT_PUBLIC_LIVEKIT_URL` are not Docker build args**
   (`Dockerfile:29-35`). Realtime and LiveKit cannot be enabled in any container built from this
   file; runtime env will not help because Next inlines `NEXT_PUBLIC_*` at build.
3. **Two of four generated modules have zero importers**, and the one file with a generated type
   available hand-writes it instead and has already drifted
   (`livekit-token/route.ts:31-35` vs `generated/meeting-service.ts:100-107`, on
   `livekit_url` nullability).
4. **No retry / timeout / 401-refresh layer.** `isAuthFailure` (`client.ts:51-53`) is dead code.
5. **`generate:api:check` is unenforceable where it matters** — needs a sibling backend checkout
   or `MYTHEUS_OPENAPI_DIR`; the Dockerfile provides neither and never runs it.
   `MYTHEUS_OPENAPI_DIR` is undocumented in `.env.example`.
6. **Most of the app is still mock-backed.** Only the dashboard's meetings arm
   (`dashboard/page.tsx:42-45`) and the transcript surface read real data. The other three
   dashboard panels deliberately read mocks (`dashboard/page.tsx:29-31`) because `action_items`,
   `worklogs` and `decisions` are shaped-but-empty server-side until Plan 7-9 — that restraint
   is correct, not a defect. But the **transcript page still resolves the meeting itself from
   `mockMeetings`** (`transcript/page.tsx:56-57`): a real backend transcript would render under
   a mock title, and a meeting absent from the fixtures 404s at `:57` **before the API is ever
   consulted**. That is a genuine wiring gap, not a deliberate deferral.
7. **`org.plan` and `user.role` are constants** (`org-context.ts:40,46`) — role-gated UI gates
   on `"member"` forever, and plan-gated UI on `"starter"`.
8. **No observability plumbing at all** (see Error Handling #6).
9. **Two documented env vars are read by nothing** — `NEXT_PUBLIC_SENTRY_DSN`,
   `NEXT_PUBLIC_APP_ENV`.
10. **Whole services unreachable from the UI** — identity-tenant-service has a 924-line generated
    client and zero calls; `/bff/org/chart` and `/bff/people/{user_id}` likewise.

---

## VERDICT

The integration layer is small, coherent and genuinely well-reasoned — one server-only REST
client with no ad-hoc fetches, no hardcoded URLs, an origin that refuses to default,
`no-store` caching with a real cross-tenant rationale, a fail-closed middleware allowlist, and a
deliberate refusal to ever send `org_id` from the client; the two Next route handlers hold no
secrets and proxy rather than fake. The typed-client work **is** connected for the two services
the UI calls (4 import sites, `core-api.ts` and `bff.ts`), but `identity-tenant-service.ts` and
`meeting-service.ts` have zero importers, and the one file with a generated type available —
`livekit-token/route.ts` — hand-writes its response type and has already drifted from the spec
on `livekit_url` nullability. The two hard blockers for shipping realtime are that the Clerk
token is fetched **once** on mount and never refreshed for socket or LiveKit reconnects, and that
`NEXT_PUBLIC_REALTIME_WS_URL`/`NEXT_PUBLIC_LIVEKIT_URL` are not Docker build args, so a container
build inlines them as undefined and permanently disables both features. Beyond that the gap is
coverage rather than quality: four real backend calls total, most screens still on mocks
(including a transcript page that 404s on mock fixtures before consulting the API), no
retry/timeout/401-refresh (`isAuthFailure` is dead code), and a spec-generation script coupled to
a sibling checkout path that CI and Docker cannot satisfy.
