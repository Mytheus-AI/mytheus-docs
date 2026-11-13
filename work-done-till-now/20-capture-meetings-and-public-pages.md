# 20. Capture and Meetings tidy-up, a dark public site, a real blog and an honest careers page

Status: `DONE WITH GAPS` (built and verified locally on 2026-11-13; committed and pushed to `main` of mytheus-ui on 2026-11-13). Not a plan band and no ADR: presentation and copy only, no contract, name or number changed.

## Why

The owner asked for four things on 2026-11-13:

1. Capture: remove the "What it records" card under the setup card and open it as a popup from a last line in "Set it up once".
2. Meetings: the "This week" card hung under the calendar as an outgrowth of the right column; move it somewhere it belongs.
3. The pages the marketing footer links to should be redesigned and filled with proper, realistic content.
4. Those pages must work in both light and dark mode.

## What landed (mytheus-ui, `fe5b3d2`..`0571f3a`)

- `src/app/(app)/capture/WhatItRecords.tsx` (new client island) and `capture/page.tsx`: the four points and the Chrome note moved into the shared `Dialog`; the setup card ends in a "What it records" link button (`fe5b3d2`).
- `src/components/meetings/MeetingsHub.tsx`: the week's four figures are a single strip above the tabs (label and range, then Meetings, Still ahead, Held, Time booked; two columns under `sm`). The rail holds the calendar only (`e0b6d20`).
- `src/styles/app-theme.css`, `src/app/(marketing)/layout.tsx`, `src/components/site/SiteNav.tsx`: the marketing root carries `mx-site`, which is added to the dark token scope next to `.mx-app` and `.mx-auth`; the nav gets the app's `ThemeToggle`. `CloseCTA`'s white inset ring became a token mix. `docs/DESIGN-ONELINE.md` no longer says dark mode is app only (`a394f72`).
- Blog: `src/lib/marketing/posts.ts` holds seven full posts; `/blog` lists them and links each; `/blog/[slug]` is a statically generated reading page with two further posts under it (`9e94e9f`, `e4ac47d`, `7c3d193`).
- Careers: rewritten around the four-person team: how we work, the stack, areas where help is wanted, how a conversation goes, and an FAQ (`f1b6a70`).
- Footer and contact: the LinkedIn and X links still pointed at the old Reprise handles and were replaced by the GitHub organisation; every `@reprise.ai` address became `@mytheus.cloud` (`fe3b9fc`, `0571f3a`).

## Decisions and why

- The marketing site had no dark set on purpose (the first redesign scoped dark to the app). The owner now wants the public pages in both themes, and the marketing components already read tokens only (one raw colour, in `CloseCTA`), so the same token set was extended instead of writing a second one.
- The old careers page listed full-time roles, equity, a learning stipend and hardware. None of that exists; Mytheus is an academic project. The new page says the team is four people without outside funding and that work starts as a collaboration.
- Blog copy makes no measured claims and names no customers. Every product statement follows what the product does (confirm before write, Capture's consent step, captions under two seconds, what is dropped first on a slow link).
- This week went above the list, not into the calendar card: the figures describe the list, and the rail then ends with the calendar on every screen size.

## Verified

- `npx tsc --noEmit -p .` clean; `npx next lint --file` clean on the changed files.
- Signed in on `localhost:3000`: `/meetings` shows the strip and a calendar-only rail (dark); on `/capture` the link opens the dialog with the four points.
- Signed out, dark: `/`, `/about`, `/blog`, a blog post, `/careers`, `/enterprise`, `/security`, `/pricing`, `/students`, `/features`, `/features/live-notes`, `/privacy` have no light-filled element left and no console errors. `/blog` and `/careers` also checked in light.

## Gaps

- Not run: the unit tests, the Playwright a11y suite and `next build`.
- The Capture dialog and the Meetings strip were not looked at in light mode or at phone width.
- The landing page in dark mode was checked at the top and by a background-colour sweep, not section by section.
- The `@mytheus.cloud` mailboxes (careers, enterprise, security, privacy, legal) are assumed, not confirmed to exist.
- About, Enterprise, Security, Students, Pricing, Privacy and Terms kept their layout and copy; only theme support and addresses changed.
