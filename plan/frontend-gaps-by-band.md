# Frontend Gaps by Plan Band

Status: analysis only, written 2026-08-14. Read-only throughout; `mytheus-ui` was
fetched before being judged and was not modified.

Companion to [frontend-meeting-ux-gap-analysis.md](frontend-meeting-ux-gap-analysis.md),
which lists 18 meeting-UX gaps harvested from the `mytheus-web` draft, and to
[work-done-till-now/06-frontend-and-ui.md](../work-done-till-now/06-frontend-and-ui.md),
which settles which repo is the product and records the deployment state. Neither
maps frontend work onto the plan bands. This file does.

## 1. The finding in one line

The screens exist and the plumbing exists; almost nothing connects them. Of the
14 pages in the `(app)` route group, **13 read from `src/lib/mock/data.ts` and one
calls the real API client**, and that one still imports mock data for part of its
content.

This inverts the usual reading of the status overview. The frontend is not behind
because pages are missing. Shells for bands that have not started yet already
exist. It is behind because the wiring, the realtime transport and the confirm
affordances are absent.

## 2. How this was established

- Page inventory from `git ls-tree -r origin/main`, after `git fetch`: 40 pages
  across `(marketing)` 22, `(app)` 14, `(auth)` 2, `(admin)` 2.
- Mock versus real determined by grepping the extracted tree for imports of
  `lib/mock` against `lib/api`.
- Transport capability determined by grepping `src/` for `livekit`, `WebSocket`,
  `EventSource` and `socket.io`, and by reading `package.json` dependencies.
- Plan requirements taken from the `web-app` work-item headings in each band file.

Dependencies on `origin/main` are exactly: `@clerk/nextjs`, `framer-motion`,
`lenis`, `next`, `react`, `react-dom`. There is no LiveKit client, no WebSocket
client and no data-fetching library.

## 3. The band map

| Band | Plan item | State | Gap |
|---|---|---|---|
| Plan 0 | 0E.1 Next.js scaffold plus Clerk provider | Satisfied late | None material |
| Plan 1-2 | 3.10 web-app shell: Clerk auth, nav, dashboard | Built, one path verified | Both acceptance checks outstanding |
| Plan 3-4 | 3.4 LiveKit room UI, chat, presence | Shell only | Whole item; no LiveKit dependency |
| Plan 3-4 | 3.8 Live captions, transcript, speaker mapping | Shell only | Whole item; no realtime transport |
| Plan 5-6 | 3.5 Org-chart UI and person profile | Backend ready, UI on mock | Wiring only |
| Plan 7-9 | 3.4 Worklog My Notes panel, agent sidebar shell | Not started | Whole item |
| Plan 7-9 | 3.8 Decision and action confirm cards, My Tasks, lineage | Shells on mock | The Confirm affordance |
| Plan 10-13 | 3.5 Context-fill cards, collaboration log | Not started | Whole item |
| Plan 10-13 | 3.9 Proactive alert UI, sidebar polish | Not started | Whole item |
| Plan 10-13 | 3.14 Integration settings, both consoles | Shells on mock | Wiring |
| Plan 14 | No web-app item | Not applicable | None |

## 4. Band detail

### Plan 0, step 0E.1

The Plan 0 record marks the `mytheus-ui` phase not done, deliberately, because
active UI work was in flight. What exists instead is a larger application than the
step asked for. The Clerk provider arrived later under Plan 1-2 section 3.10. No
action; the item is superseded rather than outstanding.

### Plan 1-2, section 3.10

The strongest part of the frontend. `src/middleware.ts` uses `clerkMiddleware`,
and `src/lib/api/client.ts` is server-only by construction: it imports `auth()`
from `@clerk/nextjs/server`, so importing it from a client component is a build
error rather than a token leak. It reads `API_GATEWAY_URL` and refuses to default
it, and it never sends `org_id`, because the gateway derives the tenant from the
verified token. `GET /bff/dashboard` has been exercised through the real chain of
Cloudflare, Caddy, `api-gateway` and `bff`.

Two gaps remain. The section's own acceptance checks are unmet: two users in
different tenants seeing disjoint data, and a network capture showing no `org_id`
on the wire. And `(app)/dashboard/page.tsx` imports both `@/lib/api/client` and
`mockActions` from `@/lib/mock/data`, so even the one wired screen is part
fixture.

### Plan 3-4, sections 3.4 and 3.8

Both are shell-only, and the blocker is below the component layer.
`(app)/meetings/[meetingId]/room/page.tsx` and
`(app)/meetings/[meetingId]/transcript/page.tsx` exist and both read mock data.
A grep of `src/` returns zero matches for `livekit`, `WebSocket`, `EventSource`
and `socket.io`, and the dependency list confirms it: there is no client capable
of joining a room or receiving a caption frame from `realtime-gateway`.

Section 3.8 is the more important of the two. Doc 10 makes live in-call captions
the flagship real-time surface, and the imported UX gap analysis names it the
largest single miss. It is also the surface every agent output attaches to, so it
gates the Plan 7-9 sidebar work as well.

### Plan 5-6, section 3.5

The cheapest remaining win in the whole frontend, and the only band item whose
backend is finished and measured. `GET /bff/org/chart` resolves a department to
team to member tree in exactly three SQL statements regardless of tree shape, at
7.85 ms against real seeded data, and `GET /bff/people/{user_id}` exists beside
it. `(app)/org/page.tsx` exists and reads mock data. Nothing needs designing or
deciding; this is a wiring task against a working, measured endpoint.

### Plan 7-9, sections 3.4 and 3.8

The band has not started, yet `(app)/decisions/page.tsx` and
`(app)/tasks/page.tsx` already exist on mock data, so the shells run ahead of the
plan. The agent sidebar does not exist in `mytheus-ui` at all, though a designed
one exists in the `mytheus-web` draft and is the intended harvest source.

Section 3.8 carries the most product-critical frontend gap in the program. The
backend enforces no silent commits: only `core-api`'s outbox emits `decisions` and
`actions`, and only on confirmation. A UI that renders detected actions as passive
text with no Confirm affordance cannot satisfy that contract, so the invariant is
currently enforced on one side only.

### Plan 10-13, sections 3.5, 3.9 and 3.14

Not started, as expected. Two notes. The context-fill card and the proactive
conflict banner both exist as finished designs in the `mytheus-web` draft, so
these are harvest-and-wire rather than design-from-nothing. And section 3.14's
surfaces already have shells: `(app)/settings/integrations`, `(admin)/admin` and
`(admin)/ops` all exist on mock data.

## 5. Cross-cutting blockers

These sit under several band items at once and should be treated as prerequisites
rather than as part of any one section.

1. **No realtime transport.** No LiveKit client and no WebSocket client. Blocks
   both Plan 3-4 items completely.
2. **No test tooling.** CI runs typecheck, lint and an image build, but there are
   no test files and no Vitest or Playwright. The marketing routes were once
   returning 404 to signed-out visitors because `src/middleware.ts` allowlisted
   `/legal(.*)`, a path this app does not have, while the real pages are
   `/privacy` and `/terms`. That was fixed, and nothing prevents its return.
3. **Clerk JWT template unmapped.** `public_metadata.internal_org_id` must be
   mapped into the token claim. Unrelated to UI code, but it blocks the AI tier
   that the sidebar work depends on.
4. **DNS.** The apex and `www` records still point at a parked page; the origin
   already serves the app.
5. **A moving target.** `origin/local-mytheus-rebuild` is the working branch and
   advanced again on 2026-08-14. Any gap list against `main` ages quickly, and one
   landing commit remains unmerged.

## 6. Suggested order

Ordered by evidence gained per unit of work, not by band number.

1. **Plan 5-6 section 3.5.** Pure wiring against a measured endpoint. Proves the
   client pattern on a second screen and removes one mock import.
2. **Plan 1-2 section 3.10 acceptance checks.** Two tenants, disjoint data, and a
   capture showing no `org_id`. This is the isolation claim demonstrated at the
   client, which is currently proven only in the database tests.
3. **Plan 3-4 section 3.8.** Add a realtime transport and land captions. Highest
   product value, and it unblocks the sidebar surfaces later.
4. **Plan 3-4 section 3.4.** LiveKit room, once the transport decision is made.
5. **Plan 7-9 section 3.8.** The Confirm affordance, which closes the no silent
   commits contract on the side that currently cannot honour it.

Items 1 and 2 need no new dependency and no new decision.
