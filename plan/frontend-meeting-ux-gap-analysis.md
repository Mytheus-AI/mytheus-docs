# Frontend: meeting UX gap analysis

Imported from `mytheus-web`, where it was written alongside a UI draft. It is preserved here
because that repository is a two-commit prototype that no reading path leads to, and this is
the only systematic comparison anyone has made between what the docs specify for the live
meeting experience and what a UI actually surfaces.

This is an execution INPUT, not a record of work done, and not canon: it introduces no name,
number, or decision. Where it says the draft diverges from the docs, the docs win until an ADR
says otherwise (see the DIVERGENCE entries, and gap 9 in particular, which is marked
"proposed, not canon").

Transliterated to this repo's conventions on import: the original used emoji status markers and
em-dashes, both of which are disallowed here. BUILT / PARTIAL / MISSING / BACKEND / DIVERGENCE
replace the emoji legend; the content is otherwise unchanged.

Source: `mytheus-web/meeting-gap-notes.md` at commit `82b7d96`.

---

## TL;DR - the headline gaps

The draft nails the **agent sidebar** (Needs-you cards, feed, conflict -> peek) and the **post-call artifacts** (transcript, decisions, tasks, worklog, recordings, ask). The real holes are **in the live meeting itself**:

1. **MISSING Live captions in-call** - the single biggest miss. Docs make live transcription the flagship real-time surface (interim->final, p95 < 2s). The draft in-call has **no caption strip/panel** anywhere; the transcript only exists as a post-call page.
2. **MISSING Action items get no confirm surface** - decisions & questions have Needs-you cards, but detected actions only appear as passive feed text ("captured as an action item"). Docs require Confirm/Edit before any action commits.
3. **MISSING Proactive alerts beyond conflict** - only the conflict card is built. Workload warnings, missing-stakeholder suggestions, and dependency warnings (all ProactiveAgent outputs) have no UI.
4. **MISSING In-meeting "ask your sources" (RAG) panel** - cited-source lookups during the call. Draft has "Ask Mytheus" only as a separate post-call page.
5. **MISSING Edit modal is inert** - the "Edit" button on decision cards is a dead tooltip; docs require an Edit modal pre-filled with the extracted content.
6. **MISSING No meetings list / meeting-detail hub** - you can only reach a meeting via home "Up next". No browsable `/meetings` list or per-meeting pre/post detail page.
7. **PARTIAL Progressive streaming** - agent output pops in fully-formed; docs want tokens streaming in then settling.

---

## Priority-ranked missing surfaces (the actual to-do list)

### Tier 1 - In-meeting surfaces (this IS the meeting software)
| # | Gap | Status | Docs source |
|---|-----|--------|-------------|
| 1 | **Live captions** (interim->final, click-to-jump, speaker-colored) rendered during the call | MISSING | `10-realtime-and-transcription §3.2/§4.1`; `20-frontend §7` |
| 2 | **ActionAgent card** - Confirm/Edit/Dismiss for detected action items (currently passive feed text only) | MISSING | `08-ai-agent §2.1`; `20-frontend §5.1` |
| 3 | **Workload warning** alert (assignee at 80%/95% capacity) | MISSING | `08-ai-agent §2.1`; `20-frontend §5.1` |
| 4 | **Missing-stakeholder** suggestion + **dependency warning** | MISSING | `08-ai-agent §2.1` |
| 5 | **In-meeting RAG context panel** - source cards with the frozen citation contract | MISSING | `20-frontend §7.1` |
| 6 | **Edit modal** for decisions/actions (pre-filled extracted content/rationale/assignee/due-date) | MISSING (button is inert) | `20-frontend §5.1`; `08-ai-agent §2.2` |
| 7 | **Progressive/streamed agent output** (tokens stream -> draft swaps for confirmed card) | PARTIAL (pops in whole) | `10-realtime §5`; `20-frontend §5.2` |
| 8 | **Priority strip / pre-emption** - frozen red(1)/yellow(2)/blue(3) language; urgent alerts pin above the feed | PARTIAL (kind-colored, no priority queue) | `20-frontend §5.3` |
| 9 | **Recording/transcription consent gate** at join | MISSING (proposed, not canon) | `12-compliance §3.3`; `plan-3-4 §8` |

### Tier 2 - Lifecycle / structural surfaces
| # | Gap | Status | Docs source |
|---|-----|--------|-------------|
| 10 | **`/meetings` list + `/meetings/[id]` detail** (pre/post hub) | MISSING (only home "Up next") | `20-frontend §2.2/§7` |
| 11 | **Per-participant post-meeting summary** artifact (confirmed decisions/actions/open questions + citations) | PARTIAL (transcript "Overview" only) | `diagrams/06 §3.4`; `21-integration §4.3` |
| 12 | **Server-anchored meeting clock** (draft timer is a hardcoded count-up) | PARTIAL | `plan-3-4 §7` |
| 13 | **Live presence** - real join/leave roster + in-call count (currently static) | PARTIAL | `10-realtime §4.1`; `plan-3-4 §3.2` |
| 14 | **Collapse-to-rail + mobile bottom-sheet** - sidebar collapses to a thin rail w/ unread counts; drag-up sheet on mobile; conflict auto-expands | PARTIAL (full hide/show only) | `20-frontend §5.4/§5.6` |

### Tier 3 - Config/settings the docs imply but the draft lacks
| # | Gap | Status | Docs source |
|---|-----|--------|-------------|
| 15 | **PII redaction** toggle | MISSING | `plan-3-4 §3.5`; `12-compliance §5` |
| 16 | **Data-residency region** selector (us/eu/ap) | MISSING | `plan-3-4 §3.5-3.6`; `12-compliance §6` |
| 17 | **Per-tenant agent confidence thresholds** override | MISSING | `08-ai-agent §6.1` |
| 18 | **Per-tenant token-budget alerts** (80%/95%/cap) surfaced in-app | MISSING | `08-ai-agent §4.1/§4.4` |

---

## Full diff by area

### A. Pre-join / device setup - **strong**
- BUILT Pre-join device check (camera/mic preview + device pickers)
- BUILT In-app prep brief (decisions/notes/actions/questions loaded, role-filtered)
- BUILT Countdown, lobby-status text, meeting details
- MISSING Consent gate (recording/transcription) - see Tier 1 #9
- BACKEND Participant-verified join / short-TTL LiveKit token mint (join is a plain link)

### B. In-call media - **mostly built (as mock)**
- BUILT Controls: mute / camera / screen-share / leave
- BUILT Speaker stage + participant thumbnails, pin, mic state, speaking wave
- PARTIAL Video grid: docs scope 1-9 tiles (6 nominal); draft shows 1 + 3 thumbs + "12+" and "change layout" is inert
- BACKEND SFU transport, adaptive simulcast, media reconnect/re-hydrate

### C. Participants & roster
- BUILT Roster list, speaker labels/avatars, invite/options affordances
- PARTIAL Live presence (static counts) - Tier 2 #13
- BACKEND Roster from calendar email-match

### D. Live transcription & captions - **the big hole**
- MISSING **Live captions in-call** - Tier 1 #1
- BUILT Transcript replay page (searchable, key moments, talk-time)
- BUILT **Speaker-mapping UI** (assign "Speaker 2" -> a user) - already in the transcript page
- MISSING PII redaction / data-residency config - Tier 3
- BACKEND Single diarized Deepgram stream, finals-only durable store, caption reconnect

### E. In-meeting AI agents (the six)
- BUILT **DecisionAgent** - Needs-you card w/ confidence 0.86, Confirm/Dismiss (Edit inert)
- BUILT **QuestionAgent** - suggestion card w/ Ask now / Save-for-later / Dismiss
- BUILT **ContextGapAgent** - "context added from Milana's agent" feed entry w/ source
- PARTIAL **PersonalListenerAgent** - notes appear in the agent feed (blended, not a distinct live "My Notes" section)
- MISSING **ActionAgent** confirm surface - Tier 1 #2
- PARTIAL **ProactiveAgent** - conflict BUILT; workload / stakeholder / dependency MISSING (Tier 1 #3-4)
- PARTIAL Streamed output - Tier 1 #7
- MISSING Token-budget alerts - Tier 3 #18

### F. Real-time collaboration during the call
- BUILT In-meeting chat (typing, reactions, host/brand badges, composer)
- PARTIAL Confirm/Edit/Dismiss - decisions BUILT, actions MISSING, Edit modal MISSING (Tier 1 #2/#6)
- BUILT Save-for-later question queue (surface present; cross-meeting persistence is mock)
- MISSING In-meeting RAG source panel - Tier 1 #5
- PARTIAL Collapse-to-rail / mobile sheet - Tier 2 #14

### G. Recording & post-call artifacts - **strong**
- BUILT Decisions (log + confidence + lineage), Tasks (action items), Worklog, Recordings library, Ask (Q&A w/ citations), Transcript replay
- BUILT Record toggle in-call (mock)
- PARTIAL Per-participant personalized summary - Tier 2 #11
- BACKEND LiveKit egress -> region-pinned bucket, Slack delivery of summary

### H. Meeting lifecycle
- BUILT Prep brief (prejoin) + Slack/summary/reminder prefs shown in Settings
- BUILT Integrations surfaced (Google Calendar, Slack, Jira, Linear connect toggles)
- MISSING `/meetings` list + detail - Tier 2 #10
- PARTIAL Server-anchored clock - Tier 2 #12
- BACKEND Calendar auto-create, Temporal lifecycle/follow-up workflows, Jira/Linear two-way sync, status webhooks

### I. Notifications / status
- BUILT Toast system (ToastContext) + "Listening" live status
- PARTIAL Conflict is a card, not a top-pinned pre-emptive banner - Tier 1 #8
- PARTIAL No frozen red/yellow/blue priority strip - Tier 1 #8
- BACKEND realtime-gateway fan-out, graceful degradation under congestion

### J. Permissions / roles
- BUILT Role concept surfaced ("notes filtered for your role: product engineer")
- BACKEND role_in_meeting token grants, participant-only tenant-scoped access, A2A authz
- MISSING Per-tenant agent overrides / theming / locale - Tier 3

---

## DIVERGENCE Divergences - the draft has these, the docs DON'T spec them

These aren't "missing" - they're the reverse. Either the docs need to catch up, or the draft is showing features that won't ship. **Decision needed on each:**

| Draft feature | Docs position |
|---------------|---------------|
| **Host-admit lobby** ("Lisbon wants to join" -> Admit/Deny card) | Docs spec only a *pre-join device check* - explicitly **no** waiting-room / host-admit flow |
| **Raise-hand** control | No documented feature |
| **Emoji reactions** on chat messages | No documented feature |
| **Manual Record toggle** as a per-user call control | Docs record via LiveKit egress tied to lifecycle/consent, not a free toggle |
| **Large-meeting affordances** ("16 in call", "12+", change-layout) | Docs scope video to 1-9 tiles (6 nominal) |

---

## BACKEND Backend/infra not wired (expected - the draft is a UI mock)

Not UI gaps, but listed so nobody mistakes the mock for a working system: LiveKit SFU + token mint, Deepgram diarized streaming, Redis presence, socket reconnect/re-hydration, Temporal `MeetingLifecycleWorkflow` + `ActionFollowUpWorkflow`, event topics (`transcript-chunks`, `agent-results`, `meeting-events`…), A2A authorization, egress recording to region-pinned buckets, multi-region residency, per-tenant token metering. Latency SLOs that shape UX: token-mint p95 < 250 ms, captions p95 < 2 s, agent output p50 < 2 s.

---

_Sources: `mytheus-docs` - `08-ai-agent-architecture`, `10-realtime-and-transcription`, `12-compliance-and-privacy`, `20-frontend-architecture`, `21-integration-architecture`, `22-roadmap-and-phasing`, `adr/0003`, `adr/0012`, `diagrams/06`, `diagrams/12`, `plan/plan-3-4`. Draft: `mytheus-web/app/(call)/*`, `app/(shell)/*`, `components/*`._
