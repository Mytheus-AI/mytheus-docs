# Wireframes: sign-in, app shell, user dashboard

P1 design artifact (Plan 1-2 section 3.1 step 5). Derived from [20-frontend-architecture.md](../../20-frontend-architecture.md) section 7.

## 1. These are annotations, not proposals

The plan assumes these screens are designed here and built in section 3.10. **They are already built.** `mytheus-ui` ships 56 static pages including every authenticated screen, on a documented design system, with the route groups matching doc 20 exactly. Its own `docs/PAGES.md` already maps each route to the `bff` endpoint it will swap to.

Producing speculative wireframes for screens that exist would create a second design authority that the built UI would immediately contradict. So this document does the useful half: it fixes the **layout contract** and the **data contract** for the three screens Plan 1-2 touches, and states the delta between what is built and what the band needs.

The visual authority stays in `mytheus-ui/docs/DESIGN-SYSTEM.md`. This document is the wire contract.

## 2. Sign-in

Route `app/(auth)/sign-in/[[...rest]]`. Currently an RSC that redirects straight to `/dashboard`, with `TODO(auth)` markers where Clerk goes.

```
+------------------------------------------------------+
|                                                      |
|                    [ Mytheus ]                       |
|                                                      |
|         +--------------------------------+           |
|         |   Sign in to continue          |           |
|         |                                |           |
|         |   [ Continue with SSO      ]   |           |
|         |   [ Continue with Google   ]   |           |
|         |   ----------- or -----------   |           |
|         |   Email    [________________]  |           |
|         |   Password [________________]  |           |
|         |   [        Sign in         ]   |           |
|         |                                |           |
|         |   No account? Sign up          |           |
|         +--------------------------------+           |
|                                                      |
+------------------------------------------------------+
```

| Contract | Rule |
|---|---|
| Provider | Clerk `<SignIn />`. We do not hand-roll a credential form. |
| After sign-in | Redirect to `/dashboard`. |
| Unauthenticated on a protected route | Redirect here, via `middleware.ts` guarding `(app)` and `(admin)`. |
| **No active organization** | This is the state the band must handle explicitly. The token carries no organization claim, the API returns `401 AUTH_REQUIRED`, and the UI must show an organization picker or a "no organization" state. It must **never** surface as a generic error or a spinner that never resolves. |
| Delta to build | `@clerk/nextjs` is not installed. Replace the redirect stub with `<SignIn />`. |

## 3. App shell

`app/(app)/layout.tsx`. Built: `AppShell`, `Sidebar`, `Topbar`, with a mobile nav drawer.

```
+---------------------------------------------------------------+
| [logo]           [ org switcher v ]        [search]  [avatar v]|
+-----------+---------------------------------------------------+
| Dashboard |                                                   |
| Meetings  |                                                   |
| Tasks     |               page content                        |
| Decisions |                                                   |
| Search    |                                                   |
| Org       |                                                   |
| Settings  |                                                   |
|           |                                                   |
+-----------+---------------------------------------------------+
```

| Contract | Rule |
|---|---|
| Single resolution point | `(app)/layout.tsx` is the **only** place organization context, theming, and locale resolve. Nothing below it re-derives them. |
| Org switcher | Switching organizations **re-authenticates through Clerk to get a fresh token**. It never mutates a header or a client-side context value. A stale token after a switch is exactly the drift the tenant-claim decision exists to prevent. |
| The client never sends `org_id` | Not in a header, not in a query, not in a body. The gateway derives it from the verified token. A network capture on any screen must show only the bearer token. |
| Admin gate | `(admin)` renders only for the `owner` or `admin` RBAC tier, read from the server. A client-side-only gate is decoration, not authorization: the API enforces it regardless. |
| Delta to build | Wire the org switcher to Clerk's organization list. Everything else is built. |

## 4. User dashboard

`app/(app)/dashboard/page.tsx`, an RSC. Built, reading `src/lib/mock/data.ts`.

```
+---------------------------------------------------------------+
|  Good morning, {first_name}                                   |
+---------------------------------------------------------------+
|  UPCOMING MEETINGS                                            |
|  +---------------------------------------------------------+  |
|  | Weekly sync            Today 14:00    4 participants    |  |
|  | Design review          Tomorrow 10:00 7 participants    |  |
|  +---------------------------------------------------------+  |
+---------------------------------------------------------------+
|  ACTION ITEMS            |  RECENT DECISIONS                  |
|  +--------------------+  |  +------------------------------+  |
|  |                    |  |  |                              |  |
|  |   (empty state)    |  |  |        (empty state)         |  |
|  |  Nothing yet       |  |  |   Nothing yet                |  |
|  +--------------------+  |  +------------------------------+  |
+---------------------------------------------------------------+
|  WORKLOGS                                                     |
|  +---------------------------------------------------------+  |
|  |                    (empty state)                        |  |
|  +---------------------------------------------------------+  |
+---------------------------------------------------------------+
```

| Contract | Rule |
|---|---|
| Data source | One call: `GET /bff/dashboard`, server-side, with the forwarded JWT. Not four calls. |
| Populated in this band | `upcoming_meetings` only. |
| Shaped-empty in this band | `action_items`, `worklogs`, `decisions` return as empty arrays with frozen shapes, so the screen does not change when later bands fill them. |
| **Empty is not the same as broken** | The response carries a per-arm `arms` status of `ok`, `degraded`, or `empty`. An `empty` arm renders the empty state. A `degraded` arm renders a retry affordance. Rendering "nothing yet" when the backend actually timed out is a lie to the user, and it is the failure mode partial degradation exists to avoid. |
| Tenant proof | The band's acceptance test is two users in two organizations seeing disjoint dashboards. |
| Delta to build | Swap the mock import for the generated client. Add the `arms` handling, which has no mock equivalent. |

## 5. What section 3.10 actually does

| Task | Status |
|---|---|
| Scaffold Next.js | **Already done. Do not run `create-next-app`; it would destroy the repo.** |
| Route groups | Already match doc 20 exactly |
| App shell and nav | Built |
| Dashboard layout and empty states | Built |
| Install Clerk, wire `<ClerkProvider>` and `middleware.ts` | **To build** |
| Generate the TypeScript client from [../openapi/](../openapi/) | **To build** |
| Swap `/dashboard` from mock to `GET /bff/dashboard` | **To build** |
| Org switcher wired to Clerk | **To build** |
| Handle the no-active-organization state | **To build** |
| The other 55 pages | Stay on mock. They belong to later bands, per `mytheus-ui/docs/PAGES.md`. |

## 6. Related

- [20-frontend-architecture.md](../../20-frontend-architecture.md) is the authoritative narrative.
- `mytheus-ui/docs/PAGES.md` is the live route map and swap plan.
- `mytheus-ui/docs/DESIGN-SYSTEM.md` is the visual authority.
- [../openapi/bff.yaml](../openapi/bff.yaml) is the dashboard wire contract.
