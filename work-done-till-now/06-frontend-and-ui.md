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
| Branch divergence in `mytheus-ui` | `main` is 5 commits ahead of `local-mytheus-rebuild`; the rebuild branch is 1 ahead of `main`; 18 files differ. Both sides moved, so this is a merge, not a fast-forward. Resolve before more work lands. |
| 18 meeting-UX gaps | See the imported gap analysis. Tier 1 is the live-meeting surfaces, and the largest single miss is live in-call captions, which doc 10 makes the flagship real-time surface. |
| Action-item confirm surface | Gap 2. The backend enforces the confirmation rule (only `core-api`'s outbox emits `decisions`/`actions`, only on confirm). A UI that shows detected actions as passive text with no Confirm affordance cannot satisfy it. |
| Clerk JWT template | Unrelated to the UI but blocks the AI tier: `public_metadata.internal_org_id` must be mapped into the token claim. See `mytheus-platform` `deploy/README.md`. |
| Deployment | `mytheus-ui` is not deployed anywhere yet. No container, no compose service, no ingress route. |

## 5. Verification

- Read-only throughout. `mytheus-ui` was fetched before being judged and was not modified.
- Page counts taken from `git ls-tree` against `origin/main` and `origin/local-mytheus-rebuild`.
- `mytheus-web` characterised by grep across its tree: 0 matches for clerk, 0 for
  fetch/axios/useSWR/NEXT_PUBLIC_API, 0 for livekit, 7 files matching mock/fixture.
- Subdomain claim verified by grepping every markdown file in this repo; zero matches.
