# The personal agent is called a Myte (ADR-0042)

Built 2026-11-12 on branch `feat/mytes-name` in mytheus-ui, mytheus-backend, mytheus-intelligence, mytheus-mobile and mytheus-docs. Not deployed.

## What changed

- **mytheus-ui:** every user-facing "your agent", "X's agent" and "personal agent" now says Myte (plural Mytes): sidebar, agent page, onboarding, settings, briefings, the meeting room (tab, people panel, listening chip, off the record, stand-ins), Inbox, notifications, empty states, toasts, aria labels, page titles, the command palette, the company brain and Myte collaboration pages, and the marketing site. `agentDisplayName` defaults to "{first name}'s Myte". The hero, sign-up, onboarding "Meet your Myte" and the agent page explain it once as your personal AI agent; SEO descriptions keep "AI agent". Also in this branch: the "How it writes" personalities as a 3 x 2 grid with a "Your own" button that focuses the instructions.
- **mytheus-backend:** core-api text people read (the briefing persona default, digest lines, notification titles, promise and bet alerts, the MCP pending note, reminder email text, two error messages) says Myte. meeting-service gains the consent notice versions `room-notice-2026-11-12` (web, current) and `room-notice-2026-11-12-mobile`, verbatim; older versions keep their words.
- **mytheus-intelligence:** no prompt changed. The briefing persona default and the delegated-answer chat line say "{first name}'s Myte".
- **mytheus-mobile:** the same copy pass, and its consent label is now `room-notice-2026-11-12-mobile`.
- **mytheus-docs:** ADR-0042, the `_canon/06` frontend row and §4 note, and the glossary row "Myte / Mytes".

## Kept as "agent" on purpose

Code identifiers, routes, API fields, enums, events, tables, prompt keys and canon agent names; OpenAPI descriptions and code comments that cite an ADR; billing and ops metering ("agent tokens", "the agent budget"); the room-level meeting agent; generic uses ("AI / Agents Engineer", user agent).

## Gaps

- Not deployed. Ship backend first (meeting-service must know `room-notice-2026-11-12` before the UI and mobile send it), then intelligence, then UI; mobile with its next build.
- The intelligence eval golden inputs still say "Priya's agent"; they are inputs, and the evals pass offline.
