# NEXUS Build Plan — `mytheus-docs/plan/`

This folder is the **execution layer** of the NEXUS (Mytheus) architecture: it turns the 24 architecture documents in `mytheus-docs/` into an ordered, buildable, agent-runnable program of work. The architecture docs say *what* the system is and *why*; the files here say *what to build, in what order, against which frozen names, with what entry preconditions and what demo proves each phase done*. The work is sliced into eight **bands** (one Markdown file each) that cover Plan 0 through Plan 14 and map onto the 36-week MVP plus a demand-driven post-GA scale-out. Read this README first; it is the index, the shared conventions, and the live register of decisions that are still open.

---

## File index

| File | Band id | Theme | Phases / weeks | One-line goal |
|---|---|---|---|---|
| `README.md` | — | Index | — | This index: file map, conventions, resolved + open decisions. |
| `00-PROJECT-UNDERSTANDING.md` | — | Ground truth | — | Technical "what is NEXUS" + current-state reality (drift, repos, canon) — the **canonical-name reference** for executors. |
| `00-HUMAN-OVERVIEW.md` | — | Orientation | — | Plain-language + technical walkthrough of the whole plan for a human reader. |
| `plan-0-scaffold-and-contracts.md` | Plan 0 | Scaffold | Pre-P1 bootstrap (supports P1, W1–2) | Repo/tooling/event-contract scaffold: `libs/shared`, compose stack, `EventEnvelope`, outbox/inbox, CI — so P1/P2 build on laid seams. |
| `plan-1-2-foundation-and-tenancy.md` | Plan 1-2 | Foundation | P1 Architecture & Design (W1–2) + P2 Foundation (W3–5) | The tenant-isolated CRUD spine: Clerk at the edge, `org_id` extracted once, Postgres RLS `FORCE` from migration #1, `audit_logs` live, design pack + threat model. |
| `plan-3-4-meeting-and-transcription.md` | Plan 3-4 | Meeting Platform | P3 Meeting Platform (W6–8) + P4 Real-time & Transcription (W9–12) | LiveKit rooms + token-mint, Deepgram real-time captions, `transcript_chunks` (finals only) on the bus, `realtime-gateway` fan-out. |
| `plan-5-6-org-context-and-ai-infra.md` | Plan 5-6 | AI Substrate | P5 Profiles & Org Context (W13–14) + P6 AI Infrastructure (W15–17) | User/org context for relevance + the inference tier: `embedding-service` (sole Qdrant writer), `reranker-service`, `search-service`, `ingestion-service`. |
| `plan-7-9-agents-and-monetization.md` | Plan 7-9 | Agents + Billing | P7 Personal Listener (W18–20) + P8 Decisions/Actions + Knowledge Graph (W21–23) + P9 RAG & Question Agent (W24–26) | Agent value on a cost-controlled LLM pipeline with billing/budgets **live first**: agents propose, `core-api` commits, AGE `nexus_knowledge`, hybrid RAG. |
| `plan-10-13-collab-integrations-ga.md` | Plan 10-13 | MVP Completion & Go-Live | P10 Multi-Agent (W27–29) + P11 Proactive (W30–31) + P12 Integrations (W32–33) + P13 Hardening/Production (W34–36) | A2A collaboration, proactive intelligence, Calendar + Slack + one PM tool, then testing/DevOps/security/GA on the self-hosted VPS. |
| `plan-14-scale-out.md` | Plan 14 | Scale-Out | Post-W36 (after P13 GA) — demand-driven, **not** in the 36-week MVP | Demand-driven extraction of the monolith into services, cells/multi-region, K8s, Citus sharding — each wave triggered by a real load signal. |

---

## How to use this

### If you are a human

1. **Start here** (`README.md`) for the map, conventions, and decision state.
2. Read **`00-HUMAN-OVERVIEW.md`** for the plain-language + technical narrative of the whole plan end to end.
3. Read **`00-PROJECT-UNDERSTANDING.md`** for the technical ground truth — what NEXUS actually is today, which repos exist, where the code has drifted, and what the canon freezes.
4. Open the band file for whatever phase you care about. Each band is self-contained and follows the identical section template below.

### If you are an execution agent

1. Read **`00-PROJECT-UNDERSTANDING.md` first** — it is the canonical-name reference and the current-state ground truth. Do not invent names; do not trust stale code over canon.
2. Execute the **`plan-N` files in dependency order**: Plan 0 → Plan 1-2 → Plan 3-4 → Plan 5-6 → Plan 7-9 → Plan 10-13, then Plan 14 only post-GA and only when a load signal fires.
3. Before touching any band, satisfy its **§1 Entry preconditions** — they must all be GREEN. A band assumes the prior band's data stores, topics, and invariants already exist; building on an unmet precondition re-poisons the foundation.
4. Use canonical names **verbatim** from `_canon/` (see Conventions). When a band's §8 flags a `DECISION-REQUIRED`, **do not guess** — surface it, and if it blocks your step, stop until it is resolved.
5. End every phase with the **§10 demo** built and runnable; that demo is the phase's definition of done.

---

## Shared conventions (every band file)

- **Section template — identical across all eight bands.** Every `plan-N` file is structured as:
  `0. Band at a glance` (phases/theme/goal/repos/demoable table) · `1. Entry preconditions` (what must be GREEN to start) · `2. Canonical names introduced in this band` (used verbatim) · `3. Work breakdown` (ORDERED, phase → component, each step states *why* it precedes the next) · `4. Data stores stood up` · `5. Events / topics introduced` (Avro subjects, producers/consumers, DLQs) · `6. SaaS overlay / preconditions landing here` · `7. Mobile (`mytheus-mobile`) work` · `8. DECISION-REQUIRED callouts` · `9. Risk checkpoints` · `10. Definition of Done & end-of-phase demo` · `11. Source docs`.
- **Canonical-name authority = `mytheus-docs/_canon/`.** Service names come from `_canon/01-service-catalog.md` (it wins over the coarser labels in the data model), tables from `_canon/02-data-model.md`, topics from `_canon/03-event-taxonomy.md`, capacity math from `_canon/05-capacity-math.md`, locked tech from `_canon/06-tech-decisions.md`. Frozen identifiers (`org_id`, `app.current_org`, the `nexus:{org_id}:` Redis prefix, `EventEnvelopeHeader`, the per-tenant Qdrant collection naming) are used exactly as written. If canon and a doc disagree, that conflict is logged as a `DECISION-REQUIRED`, not silently resolved.
- **Precondition chain.** Bands are a strict dependency chain. Each §1 enumerates the prior-band outputs it requires (e.g. RLS `FORCE` + `audit_logs`, the at-least-once outbox/inbox + DLQ + `EventEnvelopeHeader`, the inference tier's three servers) and *where each is verified*. A later band never re-cuts an earlier band's seam — the seams (one writer per store, `org_id` on every hop, outbox on every produce) are laid up front.
- **A demo at the end of every phase.** Per the original roadmap cadence, **every** phase closes with a concrete, runnable demo that is the phase's definition of done — not a slide. §0's "demoable at the end" and §10's DoD describe the exact thing a reviewer can watch run.

---

## Resolved decisions (do not re-litigate)

These are settled across the program. Bands assume them; an executor should ratify, not reopen.

- **Identity = Clerk.** Clerk is the canonical auth at the edge; `org_id` is extracted once at `api-gateway`. The Supabase-Auth drift (a `custom_access_token_hook`, an `auth.users → public.users` trigger, `CLERK_*` removed) **has already been reverted** during Plan 0: `libs/shared/auth.py` verifies Clerk RS256 against cached JWKS, `POST /clerk/webhook` verifies Svix, and the hook and trigger SQL are deleted with a CI grep guard blocking their return. Plan 1-2 §3.2 is therefore **verify-only**, not work.
- **Tenant claim shape = Clerk's native `org_id`, resolved at the edge** (Plan 1-2 §8 decision 1, RESOLVED). The session token carries *Clerk's* org id; `api-gateway` resolves it to our `org_id` via `organizations.clerk_org_id`, cached in Redis against an immutable mapping. The lookup is deliberate: it is the checkpoint where org-exists, not-suspended, and not-soft-deleted are also enforced, so a drifted or forged claim cannot assert tenancy unchallenged. Our `org_id` is never written into Clerk metadata, so there is no write-back sync to reconcile. **"Claim absent" is an explicit state** (a user with no active org) and must return `401`, never a `500`. Note: `organizations.clerk_org_id` is specified in §3.3 but was not built in Plan 0 - it lands with the §3.3 migration.
- **RBAC tier and `users.role` are separate stores** (Plan 1-2 §8 decision 2, RESOLVED). `rbac_grants.tier` (`owner`/`admin`/`member`) is the only thing authorization reads. `users.role` plus `users.skills[]` is a domain/profile attribute feeding agent relevance, and is **never** consulted for permissions - otherwise renaming a job title would grant or revoke access. `proto/identity.proto` already carries both fields separately.
- **Mobile realign.** `mytheus-mobile` is realigned off its mock prototype to consume the same Clerk claim and the same `/api/v1` contracts as `web-app`; each band carries a §7 describing the mobile slice for that phase.
- **`web-app` = `mytheus-ui`.** Canon is unchanged on this. `mytheus-web` is a **reference-only** Next.js draft of the meeting UI: it is not a build target, not in the deploy path, and not in CI. Its `meeting-gap-notes.md` is a useful docs-versus-draft gap analysis of the live-meeting surface and feeds Plan 3-4, not Plan 1-2.
- **MVP deploy = modular monolith on a self-hosted single-node VPS** ([ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md)). Docker Compose; the whole backend plus self-hosted PostgreSQL, Redis, Redpanda, and Temporal on one box behind Caddy. Single region, pooled Postgres + RLS. The relay and the workflow worker run as worker processes in the same deployable. Container seams are kept real but co-located so later extraction is mechanical. **Railway and Render are struck** - not the fast path, not a fallback.
- **Kubernetes only in Plan 14.** No K8s / Terraform / cells / multi-region / Citus in the 36-week MVP. All of that is post-W36, demand-driven, and lives exclusively in `plan-14-scale-out.md`.

---

## Open-decisions register

These are the live `DECISION-REQUIRED` items raised across the bands. Each is unresolved (or pending ratification) and **blocks** the listed work; surface and resolve them at or before the band that bites. Do not code past a blocker by guessing.

| Decision | Band it bites | What it blocks |
|---|---|---|
| **`mytheus-intelligence` repo boundary** — one repo for all 7 AI services, or split GPU inference from Python services? | Plan 0 (DR-1) | Phase 0D scaffolding (can't `mkdir` the tree) and the `mytheus-platform` CI matrix (one vs two AI repos). |
| **Canon physical location for CI enforcement** — git submodule vs vendored-with-sync vs published `@mytheus/canon` package. | Plan 0 (DR-2) | The name/schema-enforcement CI step in Phases 0A/0D/0E (`§0G.3`). **Must resolve before Phase 0A.** |
| **Partition-key contradiction** — `transcript-chunks` / `agent-stream`: canon's `org_id`-only vs the proposed composite `{org_id}:{meeting_id}` for hot-tenant spread. | Plan 3-4 (DR#1, before W9); re-surfaces Plan 5-6 (DR 8.1) | The `transcription-service` producer key function + Avro subject config, then the §3.7 embedding consumer group and all later agent consumers (re-cut if chosen after they ship). |
| **Consent entity** — `consented_at`/`consent_version` on `meeting_participants`, `consent_required` on `tenant_settings`, the `428` token-mint gate (PROPOSED, not canon). | Plan 3-4 (DR#2, decide W6) | If consent is legally required at launch, the **P3 token-mint and the P4 Deepgram stream** — you cannot lawfully record without it. Needs an ADR + amendment of two frozen tables. |
| **Plan limits per-seat vs per-tenant** — `plans.included_*` semantics not frozen by canon. | Plan 7-9 (DR#1, before W18) | The `agent_budgets` quota math (`token_quota`/`hard_cap`) and therefore **P7 metering go-live** — you cannot set a quota you cannot compute. |
| **`usage_events` derived-dimension producers** — which component owns `meeting_minute` / `transcription_second` / `storage_gb`. | Raised Plan 3-4 (DR#3); bites Plan 7-9 (DR#2, before W19) | P7 aggregation/quota for those three dimensions; if unresolved, only `agent_token` meters and minutes/storage bill nothing. |
| **`manager_id` / reporting edge** — the data model encodes no reports-to relationship. | Plan 10-13 (DR-3, before P12 §3.12) | The **authoritative** day-10 manager escalation in `ActionFollowUpWorkflow` (best-effort over the `departments` tree ships regardless; first-class `manager_id` is the canon-change upgrade). |
| **Jira vs Linear (R8 guardrail)** — ship Calendar + Slack + **exactly one** PM tool. | Plan 10-13 (DR-2, decide before W32) | The §3.11 PM connector (provider-specific code), the P12 headline demo, and the bidirectional-sync acceptance checks. Shipping both is out of scope by policy. |
| **R1 — AGE checkpoint** — AGE on managed Supabase vs self-host vs Neptune/Aura vs relational MV fallback. | Plan 7-9 (DR#3, go/no-go at P8 entry / W21); at-scale in Plan 14 (DR-2) | The `knowledge-graph-service` writer (§3.7) and therefore the lineage UI and the RAG graph arm. Record the decision + owner in the Risk Register. |
| **R6 — A2A checkpoint** — full agent-to-agent collaboration vs canary vs descope `ContextGapAgent` to RAG-only. | Plan 10-13 (DR-1, go/no-go at the P10 demo / W29) | The P10 "context added from X's agent" demo and the A2A authz attestation. Descope is a config flip, not a redesign; P11 proactive alerts do not block on it. |

---

## Provenance

These plans are **derived from the architecture docs in `mytheus-docs/`** — they do not invent architecture, they sequence it. The **time axis** (phases, week numbers, the demo-per-phase cadence, the risk-checkpoint placement) comes from **`22-roadmap-and-phasing.md`**. The **naming authority** is **`_canon/`** (service catalog, data model, event taxonomy, capacity math, tech decisions); every canonical name in a band traces back to a `_canon/` file, and every band's §11 lists the specific source docs and ADRs it was authored against.
