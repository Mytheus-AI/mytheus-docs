# Mytheus — The Build Plan, in Plain Language

> **Who this is for.** A founder, PM, or engineer who wants to understand the whole build plan in one sitting — what we're building, where we stand today, how the work is sliced up, and *why the order can't be shuffled* — without wading through 24 architecture docs first.
>
> **Codename note.** The product is **Mytheus**. The architecture (the internal name you'll see all over the design docs) is **NEXUS**. They're the same thing; "Mytheus" is the brand, "NEXUS" is the blueprint.

---

## 1. What Mytheus is (the elevator pitch)

Most meeting tools give everyone *the same* shared transcript and the same generic summary. Mytheus flips that: **every participant in a meeting gets their own personal AI agent** that attends on their behalf.

Your agent listens to the meeting *through the lens of who you are* — your role, your skills, what you care about — and does work for you in real time:

- writes **notes relevant to you** (not a generic summary),
- **suggests questions** you might want to ask, live,
- spots the **decisions, action items, and conflicts** that affect you,
- when it's missing context, it can **quietly ask other people's agents** to fill the gap, and
- it **drives the follow-ups** afterward by pushing tasks into the tools you already use (Jira, Slack, Calendar).

That gives us two things competitors don't have:

1. **Personalized, per-participant intelligence** — ten people in a meeting get ten different, tailored agents, not one shared note-taker.
2. **Agent-to-agent collaboration across meetings** — your agent can reach another person's agent to answer a question, even across separate meetings.

Under the hood it's a **multi-tenant SaaS** (one platform serving many customer organizations, each fully walled off from the others) built as **event-driven microservices** and sized for serious scale: millions of users, thousands of paying companies, tens of thousands of simultaneous meetings. The hardest engineering problem isn't the AI itself — it's making the AI *affordable*: a naive design would run an LLM bill of roughly **$224M/month**. A big part of this plan is the cost-control machinery that turns that into a viable business.

> **Jargon, defined once:**
> - **Multi-tenant** — one running system serves many separate customer organizations ("tenants"); each tenant's data is invisible to every other.
> - **Microservices** — the app is split into many small, independently-deployable services instead of one giant program.
> - **Event-driven** — services talk by publishing "something happened" messages onto a shared bus, rather than calling each other directly.
> - **LLM** — Large Language Model (the AI that reads and writes text, e.g. Claude).
> - **Tenant isolation by `org_id`** — every row of data is stamped with the organization's ID, and the database refuses to return rows that don't match the caller's org. This is the backbone of keeping customers separate.

---

## 2. Where we are today (the honest current state)

**We are near the starting line.** Across all seven code repositories, there is *almost no working application code yet*. The value so far is in the **design** — a thorough, frozen architecture — plus one genuinely useful shared library and a good-looking but off-spec mobile prototype.

Here's the repo-by-repo truth:

| Repo | What it's *for* | What actually exists today |
|---|---|---|
| **mytheus-backend** | The CRUD + control-plane tier (10 services as a "modular monolith") | The **only repo with real backend code.** A solid **shared library** (`libs/shared/`) works: auth/JWT checks, tenant isolation helpers, the outbox/inbox messaging plumbing, idempotency middleware, error handling, telemetry. But all 10 services are empty shells (`# TODO: add routers`), there are **no database models and no migrations**, so the tables the library writes to *don't exist yet*. Zero tests. |
| **mytheus-events** | The frozen, shared event contracts (Avro schemas) | **Docs only.** Excellent written spec of all 21 topics and the message envelope — but no actual schema files, no codegen, nothing importable. |
| **mytheus-realtime** | Live tier: Socket.io fan-out + Deepgram transcription | **Docs only.** No services, no code. |
| **mytheus-ui** | The Next.js 14 web dashboard | **Docs only.** No `package.json`, no app. |
| **mytheus-platform** | Infrastructure-as-code, CI/CD, deploy | **Docs only.** A well-tuned `.gitignore` is the lone real artifact. |
| **mytheus-intelligence** | The AI/inference tier (the 6 agents, RAG, embeddings) | **Does not exist on disk at all.** The entire AI repo still has to be created. |
| **mytheus-mobile** | The native mobile client | A **polished mock prototype** — but it's the *wrong product*. It implements a single-device "Meeting Pod" demo on hardcoded fake data, not the documented multi-tenant client. Real UI craft, zero backend. |

**What this means in plain terms:** the architecture is done and high-quality; the building has barely begun. The shared library in `mytheus-backend` is the one reusable asset to build *on*. The first thing the plan must produce is the **data layer** (the actual database tables) and the **event contracts** — because right now the plumbing is wired to tables and message schemas that don't exist.

> **Jargon, defined once:**
> - **Modular monolith** — the code is organized into clean, separate modules (like microservices) but deployed as *one* process for now; it can be split into true microservices later along pre-drawn lines.
> - **Avro schema** — a strict, versioned definition of a message's shape, so producers and consumers can't drift apart.
> - **Migration** — a versioned script that creates/changes database tables.
> - **RAG** — Retrieval-Augmented Generation; the agent looks up relevant documents/transcripts and feeds them to the LLM so answers are grounded in real context.

---

## 3. How the plan is organized

The original roadmap was **13 phases over 36 weeks** (P1 through P13). That's a lot of moving parts to hand to an automated builder. So the plan is regrouped into **7 themed "bands"** — each band is **one coherent theme** and lives in **its own file** that an AI agent can execute top-to-bottom.

**The naming tells you which phases a band covers:**

- `plan-0-...` → a **pre-phase**: the scaffolding everything else depends on.
- `plan-1-2-...` → covers phases **P1 and P2**.
- `plan-3-4-...` → covers **P3 and P4**.
- `plan-5-6-...` → **P5 and P6**.
- `plan-7-9-...` → **P7, P8, P9**.
- `plan-10-13-...` → **P10 through P13** (the whole MVP finish).
- `plan-14-...` → everything **after** week 36 (scale-out beyond the MVP).

So the file number = the phase range. Each band is self-contained, one theme, and is meant to be completed before the next one starts (with one exception: Plan 0 underpins the early bands rather than fully finishing before them).

> **The 7 bands at a glance:**

| File | Phases | Theme | One-line goal | Demoable at the end |
|---|---|---|---|---|
| `plan-0-scaffold-and-contracts.md` | Pre-P1 | **Scaffold** | Make every repo buildable, testable, and wired together | All repos compile + CI green; event contracts publish; first DB tables exist |
| `plan-1-2-foundation-and-tenancy.md` | P1–P2 | **Foundation** | The tenant-isolated CRUD spine | Log in, create an org, see a dashboard — with airtight tenant isolation + audit trail |
| `plan-3-4-meeting-and-transcription.md` | P3–P4 | **Meeting Platform** | Capture live meetings + transcribe them | Join a live video meeting and watch real-time captions appear |
| `plan-5-6-org-context-and-ai-infra.md` | P5–P6 | **AI Substrate** | The substrate agents need to exist | Upload a doc, search it semantically; the GPU + event backbone are live |
| `plan-7-9-agents-and-monetization.md` | P7–P9 | **Agents + Billing** | The core agent value, metered and budgeted | Your personal agent writes notes, flags decisions/actions, and answers questions — all billed |
| `plan-10-13-collab-integrations-ga.md` | P10–P13 | **MVP Completion & Go-Live** | Multi-agent + proactive + integrations + production | Agents collaborate, push tasks to Jira/Slack, and the product is **live** |
| `plan-14-scale-out.md` | Post-W36 | **Scale-Out** | Split the monolith into independently-scaling services | (Demand-driven; not part of the 36-week MVP) |

---

## 4. A walk through each band

### Plan 0 — Scaffold, Tooling & Event Contracts *(Scaffold)*

**Plain words:** Before anyone builds a feature, make the whole project *actually buildable and wired together.* Right now the repos can't talk to each other and the database tables don't exist. This band fixes the foundation under the foundation.

**What gets built (technical):** Reconcile the **Clerk → Supabase** identity drift in the docs (code already moved to Supabase Auth; docs still say Clerk). Stand up the missing **mytheus-intelligence** repo. Build **mytheus-events** for real — author the **7 frozen Avro schemas** plus the shared event envelope, wire up Schema Registry subjects, a `BACKWARD`-compatibility gate, and **codegen** for Python + TypeScript clients. In `mytheus-backend`, generate the **first Alembic migrations** and the **SQLAlchemy model layer** so the `outbox`/`inbox`/`users` tables the shared library writes to finally exist; mount the idempotency middleware; build the **outbox relay** and a **gRPC server**. Scaffold the UI/realtime repos so they build and lint. Stand up **CI/CD** (lint, test, secret-scan, schema gate, image build) and a **Railway/Render** deploy target.

**Demoable:** Every repo compiles and passes CI; a generated event client can be imported; the first real database tables exist and the shared library can write to them.

### Plan 1-2 — Foundation & Tenancy *(Foundation)*

**Plain words:** Build the skeleton of the actual product — sign-up, organizations, users, a dashboard — and make it *impossible* for one customer to see another customer's data, starting from the very first day.

**What gets built (technical):** The tenant-isolated CRUD spine: **identity-tenant-service, core-api, bff, web-app shell, api-gateway.** Critically, **Row-Level Security (RLS) and `org_id` injection from the very first migration** — tenant isolation is baked into the database from row one, not retrofitted. **`audit_logs`** (the tamper-evident trail) goes live *here* at P2, not at the end, so it captures security events from the moment isolation turns on. Plus the P1 design artifacts (C4 diagrams, ERD, threat model), **security shift-left** (dependency + secret scanning in CI), and compliance-by-construction (`data_residency`, `tenant_settings`).

**Demoable:** A user can sign in, an org gets created, and a basic dashboard renders — with provably airtight tenant separation and a working audit log.

> **Jargon:** **RLS (Row-Level Security)** — the Postgres database itself filters every query so you can only ever see rows tagged with your own org. Even a buggy service can't leak across tenants. **BFF (Backend-for-Frontend)** — a thin layer that shapes data exactly for what a given screen needs. **api-gateway** — the single front door that checks your token, figures out your `org_id`, and routes the request.

### Plan 3-4 — Meeting Platform & Real-time Transcription *(Meeting Platform)*

**Plain words:** Make meetings actually happen and get transcribed live, word by word, as people speak.

**What gets built (technical):** **meeting-service** wraps **LiveKit** behind a transport adapter (so we can swap the meeting SDK later without rewriting everything) and mints room tokens. **realtime-gateway** uses **Socket.io** for live fan-out — presence, chat, captions — to many participants at once. **transcription-service** proxies **Deepgram**, splitting interim guesses from finals and **persisting finals only** to the `transcript_chunks` table, with speaker diarization. A hard rule: the **compliance gate must be GREEN before week 9** (Deepgram data-processing agreement, PII redaction, residency pinning) — because P4 is the first time we capture real human speech.

**Demoable:** Join a live meeting and watch accurate real-time captions stream in, attributed to speakers.

> **Jargon:** **LiveKit** — the open-source real-time audio/video platform. **Socket.io** — a library for pushing live updates to browsers over WebSockets. **Deepgram** — the speech-to-text vendor. **Diarization** — figuring out *who* said each sentence. **Finals only** — speech-to-text emits rough live guesses ("interims") then a corrected "final"; we only store the finals.

### Plan 5-6 — Org Context & AI Infrastructure *(AI Substrate)*

**Plain words:** Build everything the agents will *stand on* before building the agents themselves — knowing who each person is, and a dedicated, cost-isolated home for the AI to run.

**What gets built (technical):** **Profiles & org context** — roles, skills, teams, departments, and the `get_user_context(user_id)` function that makes an agent personalized. The **dedicated GPU inference tier** (embeddings and reranking on their own GPU servers, deliberately *off* the API tier so AI load can't crush the web app). **Per-tenant Qdrant** vector collections, **document ingestion**, and **hybrid search**. And the full **Redpanda / Avro / Schema Registry** event backbone with the first real **outbox → inbox → DLQ** at-least-once consumers.

**Demoable:** Upload a document, run a semantic search and get relevant chunks back; the GPU tier and the production event bus are live and isolated.

> **Jargon:** **Embedding** — turning text into a vector of numbers so similar meanings sit near each other. **Qdrant** — the vector database that stores embeddings and finds nearest matches. **Reranker** — a second-pass model that re-sorts search hits for relevance. **Redpanda** — a Kafka-compatible event streaming platform (the "bus"). **Outbox/Inbox/DLQ** — the reliability pattern: write the message in the same DB transaction as the data (outbox), de-duplicate on the way in (inbox), and quarantine poison messages to a Dead-Letter Queue (DLQ). **At-least-once** — every message is guaranteed delivered, possibly more than once, so consumers must be idempotent.

### Plan 7-9 — Agent Intelligence & Monetization *(Agents + Billing)*

**Plain words:** Ship the actual magic — the personal agents — but *only after* the meter is running, so we don't accidentally light $224M/month on fire.

**What gets built (technical):** **agent-runtime-service** with a shared **LangGraph** `BaseAgent`, one `AgentState`, and a versioned prompt loader, running the **cost-controlled LLM pipeline**: windowed batching (run agents per 30-second window, not per raw chunk) → a cheap **relevance pre-gate** → **tiered model routing** (Haiku triage → Sonnet extract → Opus for high-stakes) → semantic + prompt **caching** → **per-tenant token budgets** with hard quotas. Billing and metering (**billing-metering-service**, `agent_budgets`) must be **live before the first agent runs** — this is a hard P7 precondition or the cost bomb detonates unmetered. Then the agents, in order: **PersonalListenerAgent** (writes `worklogs`), then **DecisionAgent + ActionAgent** feeding the **Apache AGE knowledge graph**, then a **RAG pipeline + QuestionAgent** surfacing grounded suggestion cards. A firm rule: agents **never commit silently** — they emit *candidates*; the human confirms, and only then does core-api record the decision/action.

**Demoable:** Your personal agent writes role-relevant notes during a meeting, flags decisions and action items for you to confirm, and answers questions grounded in past meetings — and every token is metered against a per-tenant budget.

> **Jargon:** **LangGraph** — a framework for building stateful, multi-step AI agents. **Token budget** — a hard cap on how much LLM usage a tenant can consume, so cost is bounded. **Tiered routing** — send easy work to the cheap fast model and only escalate to the expensive model when needed. **Knowledge graph (Apache AGE)** — decisions, actions, people, and meetings stored as connected nodes/edges so you can trace *why* a decision happened and *what* followed from it.

### Plan 10-13 — Multi-Agent, Proactive Intelligence, Integrations & Production GA *(MVP Completion & Go-Live)*

**Plain words:** Finish the differentiated product and *actually launch it* — agents that talk to each other, agents that warn you before problems happen, real connections to Jira/Slack/Calendar, and a hardened production deploy.

**What gets built (technical):** **workflow-service** (Temporal) runs the **MeetingLifecycleWorkflow**; the **ContextGapAgent** does cross-meeting agent-to-agent (A2A) collaboration over `agent-requests`/`agent-responses` with **deny-by-default authorization**. The **ProactiveAgent** raises conflict/workload/stakeholder/dependency alerts over a Redis pub/sub backbone with streamed output. **integration-service** connects **Google Calendar + Slack + one of Jira/Linear**, with envelope-encrypted OAuth tokens and an **ActionFollowUpWorkflow** (day 1/3/7 nudges, day-10 escalation). Then **P13 hardening**: Playwright E2E + k6 load tests, a final security attestation, the full Stripe billing surface, Postgres point-in-time recovery + read replica, Cloudflare, a custom domain, and a **live production deploy.**

**Demoable:** A complete, hardened product — your agent pulls context from a colleague's agent, proactively warns you of a scheduling conflict, and pushes a confirmed action item straight into Jira — running live in production.

> **Jargon:** **Temporal** — a durable workflow engine for long-running, reliable multi-step processes. **A2A (agent-to-agent)** — one user's agent securely querying another's. **Deny-by-default** — cross-agent access is forbidden unless explicitly allowed and audited. **GA (General Availability)** — the real, public production launch.

### Plan 14 — Post-W36 Scale-Out *(Scale-Out)*

**Plain words:** *After* launch, when real paying load justifies it, split the modular monolith into independently-scaling pieces and go multi-region. This is **not part of the 36-week MVP** — it's demand-driven.

**What gets built (technical):** Extract services along the **frozen Wave 1–4 seams** onto Kubernetes + KEDA — Wave 1 (GPU inference, real-time, workers), Wave 2 (CRUD core), Wave 3 (billing/integration/admin-ops), Wave 4 (knowledge-graph + search). Add **cell-based isolation** for Enterprise tenants, **multi-region** (active-passive → active-active), and the documented escalation paths (Citus sharding by `org_id`, AGE → Neptune/Neo4j Aura, dedicated Redpanda, GPU autoscaling) sized to the capacity anchors (5M users, 3,000 tenants, 20k peak meetings, 100k peak participants).

**Demoable:** N/A for MVP — this band executes only when load demands it.

> **Jargon:** **Kubernetes / KEDA** — container orchestration plus event-driven autoscaling. **Cell** — a self-contained slice of the platform serving a subset of tenants, for blast-radius isolation. **Sharding** — splitting one giant database into many by `org_id`.

---

## 5. The build order, and why it can't change

This isn't a preference ordering — it's a chain of **hard preconditions.** Each band unlocks the next; reorder them and something breaks or becomes unsafe.

1. **Data layer + event contracts first (Plan 0).** The shared library already writes to `outbox`/`inbox`/`users` tables that *have no DDL* — they don't exist. Nothing can run until the models and migrations are generated. And sibling repos are designed to *import generated event clients* — so the frozen contracts in `mytheus-events` must be real before any cross-repo event flow is safe. The Clerk→Supabase drift must also be reconciled here, or P2 builds against the wrong identity contract.

2. **Tenancy from the very first migration (Plan 1-2).** Retrofitting tenant isolation onto an existing database is a documented disaster. RLS + `org_id` must be present in migration #1. `audit_logs` is stood up here (not at the end) so the tamper-evident trail exists the moment isolation goes live.

3. **Compliance GREEN before transcription (Plan 3-4).** P4 is the first time the system captures real human speech. The compliance gate (Deepgram DPA/SCCs, PII redaction, residency pinning) **must be green before week 9** — you cannot record people first and get permission later.

4. **Substrate + budgets before agents (Plan 5-6 → Plan 7-9).** `get_user_context` (P5) is *what makes agents personalized*, and the dedicated GPU tier + per-tenant vector store + event backbone (P6) are the explicit precondition for the cost-controlled pipeline and latency budgets — running embeddings on the API tier is exactly the coupling the architecture exists to remove. Then, within Plan 7-9, **billing-metering + per-tenant token budgets must be live by P7** — the instant the first agent runs unmetered, the **R4 cost bomb detonates.** Agents are sequenced PersonalListener → Decision/Action+graph → RAG/Question because the knowledge graph needs confirmed decisions, and QuestionAgent needs RAG retrieval to exist first.

5. **Integrations + go-live last (Plan 10-13).** A2A collaboration (P10) is the least-proven feature (descope is pre-authorized). ProactiveAgent (P11) builds on *every* prior agent. Integrations (P12) need billing and admin consoles to administer them. Production hardening (P13) is a final attestation over a *complete* system. You can only harden what's already built.

6. **Scale-out only when load demands it (Plan 14).** Services peel apart along pre-drawn seams *only once their autoscaler signal diverges from the CRUD tier's.* Extracting earlier adds operational cost with no load to justify it — and re-collapsing is the rollback.

---

## 6. The end goal

The finish line of the MVP is **the end of Plan 10-13**: a complete, differentiated product that **connects to Jira, Slack, and Calendar** and **goes live in production.** Concretely — your personal agent captures a decision in a meeting, you confirm it, and it lands as a tracked task **in Jira**, with proactive follow-up nudges, agents collaborating across meetings, and the whole thing hardened, billed, and deployed on a custom domain.

**Plan 14 is the horizon beyond that** — turning the launched monolith into an independently-scaling, multi-region platform *when paying-tenant load actually requires it.* It is deliberately **not** part of the 36-week MVP.

---

## 7. Decisions — made, and still open

### Decisions we've made

- **Identity provider:** moving to **Clerk reconciliation** is being resolved in Plan 0 — the *code* has moved to **Supabase Auth** (custom access-token hook injecting `org_id`, auth→public user-sync trigger), and the plan reconciles the lingering Clerk references in the docs to match. (Confirm the canonical IdP before P2 — see open items.)
- **Mobile realignment:** `mytheus-mobile`'s "Meeting Pod" prototype is acknowledged as **off-spec**; a realignment decision is on the table (keep it as a separate surface, re-align to the web contract, or defer mobile entirely).
- **MVP deployment target:** the early-stage product runs as a **modular monolith on Railway/Render**, with Kubernetes deferred to the post-W36 scale-out (Plan 14).

### Decisions still open (the register)

These are flagged risks/questions that need a human owner and a ruling **before** the band that depends on them:

- **Consent entity — RESOLVED, schema only** ([ADR-0018](../adr/0018-consent-entity-behind-flag.md)). `consented_at`/`consent_version` on `meeting_participants` and `consent_required` on `tenant_settings` are now canon, defaulted off. The 428 token-mint gate's application code and the migration are later tasks; whether the gate must ship before launch in a given jurisdiction is still a legal/compliance call, not an open canon question.
- **Partition key — RESOLVED** ([ADR-0017](../adr/0017-transcript-chunks-composite-partition-key.md)). `transcript-chunks`/`agent-stream` are composite-keyed `{org_id}:{meeting_id}`; every other topic keeps `org_id` alone. Canon no longer contradicts itself on this.
- **Plan limits (per-seat vs per-tenant)** — `included_meeting_minutes` / `included_agent_tokens` accounting is unresolved; it changes the billing quota math and must be fixed **before metering goes live (Plan 7-9).**
- **`manager_id` / reporting edge** — there's no first-class manager relationship in the data model, so the day-10 escalation in ActionFollowUpWorkflow is best-effort. Decide whether to add `manager_id` **before P12.**
- **Jira vs Linear** — only **one** of the two ships in the MVP (the R8 guardrail). Which one is a go/no-go call needing an owner before P12.
- **The descope checkpoints — R1, R6, R8** — pre-authorized but unscheduled go/no-go decisions: **R1** (does the Apache AGE graph stay on Supabase or exit to Neo4j Aura, at P8?), **R6** (does A2A collaboration ship, or collapse to RAG-only, at P10?), **R8** (which integration ships, at P12?). Each needs an assigned owner before its band.
- Plus: **mytheus-intelligence repo boundary** (one repo or split?), **`usage_events` producer ownership** for derived dimensions, and **where `_canon/` physically lives** so naming authority is CI-enforceable.

---

## 8. How to use this folder

- **If you're a human** trying to understand the project: read **this file (`00-HUMAN-OVERVIEW.md`)** first, then **`00-PROJECT-UNDERSTANDING`** for the deeper technical grounding (the full service catalog, data model, and current-state survey). Together they give you the "why" and the "what."
- **If you're an executing agent** (or an engineer following the plan): work the **`plan-N` files in order, top to bottom** — `plan-0`, then `plan-1-2`, `plan-3-4`, `plan-5-6`, `plan-7-9`, `plan-10-13`, and finally (only when load demands it) `plan-14`. Each file is a self-contained, agent-executable band for **one theme**, with its deliverables, preconditions, and acceptance criteria spelled out.
- **The golden rule:** don't skip ahead. The order in §5 is a precondition chain, not a suggestion — the build is only safe in sequence.

> **Reference shelf** (for when you need the source of truth): the frozen architecture lives in `mytheus-docs/docs/00`–`24`, the ADRs in `docs/adr/`, the diagrams in `docs/diagrams/`, the risk register in `docs/23-risk-register.md`, and the naming authority in `docs/_canon/` + `docs/24-glossary-and-naming.md`.
