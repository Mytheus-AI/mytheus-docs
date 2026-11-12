# ADR-0042: The personal agent is called a Myte

## Related documents

- [_canon/06-tech-decisions.md](../_canon/06-tech-decisions.md): §2.8 (the frontend row) and §4 (the frozen names, unchanged).
- [24-glossary-and-naming.md](../24-glossary-and-naming.md): the "Myte / Mytes" concept row.
- [ADR-0038](0038-agent-personas-profiles-agendas-and-meeting-agent.md) (agent personas, the default display name), [ADR-0040](0040-polish-program.md) (the versioned consent notice).

## Status

**Accepted** (2026-11-12).

## Context

Every participant gets a personal AI agent. In the product it was called "your agent", "Samar's agent" or "a personal agent". That is generic: it competes with "user agent", integration agents, the room-level meeting agent and the agent token budget, and it does not match the plush characters people now pick as their agent's face. The owner chose a product name: **Myte** (plural **Mytes**), from Mytheus plus "mites", little helpers.

Options considered: keep "agent" (clear to search engines, but generic in the product); rename the canon agent and every identifier (a breaking change across nine repos, the event contracts and the database for a word only people read); or name it only in user-facing copy.

## Decision

1. **D1 The name.** In every user-facing word (web, mobile, email and notification text, Slack messages, the briefing's spoken persona name), the personal agent is a **Myte**: "your Myte", "Samar's Myte", "Meet your Myte"; plural "Mytes"; capital M always, as a product noun.
2. **D2 Copy only.** Code identifiers, routes (`/agent`), API fields (`agent_name`, `agent_*`), enums (`attendance = 'agent'`), events and topics (`agent-results`, `agent-stream`), tables, prompt keys and the canon agent names (`PersonalListenerAgent` and the rest, [§4](../_canon/06-tech-decisions.md)) do not change. No contract changes, so no oasdiff break and no migration.
3. **D3 Explained once.** Where a newcomer first meets the word (the marketing hero, sign-up, onboarding "Meet your Myte", the agent page, mobile Settings and the join notice) the copy says what it is: your personal AI agent. "AI agent" stays in that phrase and in SEO descriptions so search still works.
4. **D4 What keeps "agent".** Generic uses (user agent, integration agents), the admin metering terms (agent tokens, the agent budget, the ops cost console), and the room-level `MeetingAgent`, which is shown as "the meeting agent" because it is not anyone's Myte.
5. **D5 Defaults.** The default display name (ADR-0038 D2) becomes "{first name}'s Myte" in mytheus-ui, core-api's briefing persona and agent-runtime's briefing and delegated-answer lines.
6. **D6 Consent.** The consent notice's words change ("Your Myte's notes are yours", "every Myte pauses"), so it gets new versions that meeting-service keeps verbatim: `room-notice-2026-11-12` (web, current) and `room-notice-2026-11-12-mobile` (the phone's notice, which has no off-the-record clause). Receipts for older versions keep their old words.

## Consequences

- People see one consistent name for their agent; search and the glossary still connect it to "personal AI agent".
- Developers keep reading `agent` in code, APIs and canon. The glossary row records that the two names mean the same thing.
- Prompts were left as written: the briefing writer already speaks as the person's agent in the first person and takes its name as input, so its output says "Myte" without a prompt version change, and the offline eval recordings stay valid.
- Existing personas named by their owner keep their names; only the default changes.
