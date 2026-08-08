# Frontend and UI

Status: IN PROGRESS. Last verified 2026-08-08.

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
