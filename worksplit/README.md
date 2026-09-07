# Mytheus - Remediation Work Split

Built from a two-pass audit of all eight repos: 16 auditors read the source, then 7 verifiers
re-checked every finding against current code and hunted for what the first pass missed. Status
docs were excluded as a source of truth throughout.

**Audit report:** https://claude.ai/code/artifact/da281c5f-d990-40b4-9487-d9f857833b08
**Raw evidence:** [`audit/`](audit/) - 16 first-pass reports · [`verify/`](verify/) - 7 verification reports

---

## The finding

The vertical slices are strong. The seams between them were never closed. Almost every component
is built to a high standard and wired to a counterpart that does not exist.

**Five defects stop the system before any of that matters**, and one of them destroys data rather
than causing an outage.

---

## Work files

| Owner | File | Non-mobile tasks | Plus |
|---|---|---|---|
| Sushan | [SUSHAN.md](SUSHAN.md) | 12 | all 25 mobile tasks, plus ~37 build tasks (~88 d) |
| Samar | [SAMAR.md](SAMAR.md) | 17 | ~46 build tasks (~82 d) |
| Shaun | [SHAUN.md](SHAUN.md) | 18 | ~45 build tasks (~84 d) |

Everyone owns backend, platform and frontend work. Nobody is "the infra person".

### On balance - read this, Sushan

Mobile was assigned to you in full as requested. That is **25 additional tasks** (13 actionable
now, 8 blocked on a scoping decision, 4 blocked on absent backend), so your real load is roughly
**2.5x the others**, even though the non-mobile split is even at 12/17/18.

If you want that evened out, these four are the cleanest to hand off - they are self-contained and
have no dependency on your auth work:

- **S7** gRPC authentication -> Samar (sits next to his deploy/network work)
- **S9** `PUBLIC_CONTAINS` anchoring -> Shaun (pairs with his D3 route-map cleanup)
- **S11** consent write endpoint -> Shaun (pairs with his D6/D7 meeting-lifecycle work)
- **S12** `tenant_settings` provisioning row -> Shaun

Your call. The files assume you keep them.

---

## ORDERING

```
 +--------------------------------------------------------------+
   |P0 - BEFORE ANYTHING ELSE |
   | |
   |SAMAR P1 make migrate footgun <- DATA LOSS, do first |
   |SUSHAN S1 gateway signing key <- every request 500s |
   |SUSHAN S2 RBAC grant writer <- every user 403s |
   |SAMAR P2 chat_messages partition <- live outage, 6 days |
   |SAMAR P3 deploy creates topics <- fresh deploy fails |
 +--------------------------------------------------------------+
                              |
      +---------------------┼---------------------+
      v v v
 SUSHAN S3-S12 SAMAR P4-P17 SHAUN D1-D18
 + all mobile
```

### Why these five come first

| # | Owner | Effect if not fixed first |
|---|---|---|
| **P1** | Samar | `make migrate` autogenerates **and applies** with no `include_object` filter. The next person adding a column drops `meeting_rooms`, `meeting_speakers`, `chat_messages`, every partition, and `mytheus-realtime`'s `transcript_chunks` - in one command, no review. **Silent data loss from the documented workflow.** Everything else here is recoverable; this is not. |
| **S1** | Sushan | Every authenticated request through the gateway returns 500. Nobody can verify any API change. |
| **S2** | Sushan | Every real Clerk signup holds tier `""` and is 403'd everywhere. Even with S1 fixed, nothing is reachable. |
| **P2** | Samar | `POST /chat` has raised "no partition of relation found" since 2026-09-01. |
| **P3** | Samar | `deploy.sh` verifies 42 topics nothing creates. A clean deploy fails by construction. |

### What can start immediately, before P0 clears

- **Samar:** P4, P5, P6, P7, P8 (Dockerfiles, CI, compose, ingress, limits) - off the request path.
- **Shaun:** D1 (the cross-tenant hole) and D2, both pure backend with tests.
- **Sushan:** the mobile tasks marked *safe now* - mobile has its own baseline and is green today
(`tsc` exit 0, jest 14/14).

### Two "must ship together" pairs

Samar's P4/P5 deploy services that currently contain armed defects. These are not follow-ups:

- **P13** (transcription-service least-privilege DB role) ships **with** P4/P5 - otherwise a
superuser connection to the shared production Postgres goes live.
- **P14** (`FakeProvider` provenance stamp) ships **with** P4/P5 - otherwise fabricated transcript
rows start entering real tables and the real topic.
- **P16** (notification consumer offset defuse) ships **before** any compose entry for it -
otherwise it commits offsets on messages it discards and destroys the backlog.

---

## Blocked on a decision from Sushan - do not assign, do not start

**Where does the authenticated product live?**

`mytheus-ui`'s branch `origin/local-mytheus-rebuild` (commit `0455cc8`, 2026-08-13) states the
authenticated product moved to **`mytheus-web`**, a separate Next.js app, and points every CTA at
`https://app.mytheus.cloud`. It rewrites sign-in/sign-up to hand off because that app *"owns the
real auth flow"*.

Canon says the opposite: `mytheus-web` is a reference-only draft - not a build target, not
deployed, not in CI, not in canon.

**Until this is settled**, only UI work that is safe under either outcome is assigned (Shaun's D16,
D17). Everything else - wiring `/meetings`, `/org`, the dashboard arms, the generated types - is
held. `mytheus-web` has never been audited; if it wins, it needs its own audit before anything is
assigned there.

Shaun's **D18** covers reconciling the branch itself (6 ahead / 11 behind; its merge would clobber
`.env.example`, which main added after the merge-base).

---

## Deliberately out of scope - not gaps, do not build

Per the roadmap's W36 phase boundary. Their absence is the plan being followed:

- Kubernetes, Terraform, Helm - Plan 0 says the MVP ships a single-node VPS target only
- Wave 1-4 service extraction; the monolith is meant to still be a monolith at W36
- Multi-region, cell isolation, Citus, BYOC Redpanda, int8 Qdrant, GPU autoscaling
- SSO / SAML / SCIM - deferred at the tenant-claim decision
- `pii_tokens` migration - ADR-0020 accepted for the model, *blocked for implementation* on key custody
- `ner-service` - named in canon, assigned to no band
- A second issue tracker - the R8 guardrail working as intended

**Blocked on external dependencies, not engineering:**

- **Deepgram ASR** - `COMPLIANCE_GATE_GREEN = False` is a source constant gating on a signed DPA.
A legal artifact; no PR can make it true.
- **LiveKit** - *procurement*, not legal. Distinct from the above. Browser mic capture (audio hop 1)
needs no account and is actionable whenever it is scheduled.

---

## The build program, now assigned

The first version of this split parked the entire product build under "needs a scope conversation"
and assigned none of it. That was wrong: it is the scope the roadmap already allocates, and leaving
it unwritten is how it stays unbuilt. Six specs were produced and the work is now distributed.

| Chunk | Owner | Tasks | Effort |
|---|---|---|---|
| Agent runtime and the four MVP agents | Sushan | ~18 | ~40 d |
| Notification delivery | Sushan | 19 (16 MVP) | ~34 d |
| Audio vertical | Samar | 12 | 24 d, 13-day critical path |
| Billing, metering, quota | Samar | 32 (26 MVP) | ~46 d |
| Serving and test substrate | Samar | ~2 | ~12 d |
| Confirmation spine | Shaun | 7 | ~20 d |
| Knowledge graph (R1 fallback) | Shaun | ~3 | M |
| RAG pipeline | Shaun | ~6 | ~20 d |
| Delivery to the browser | Shaun | ~5 | ~8 d |
| Integrations | Shaun | 19 (16 MVP) | ~36 d |

Roughly 123 tasks and 250 to 280 engineer-days, split about 88 / 82 / 84.

Specs live in [`spec/`](spec/). Read the relevant one before starting a task.

## Decisions only Sushan can make

Everything below blocks real work. Several are pure waiting-time and should start on day one.

| Decision | Blocks | Note |
|---|---|---|
| LLM provider and spend | B1, B2 completion | No LLM call exists in any repo. Everything else can be built against a stub completion. |
| Knowledge graph: take R1? | E2 | Recommendation is yes, now. The migration cost is zero today and rises weekly. |
| `action_items.status`: one axis or two | E1 migration | Canon-frozen field list; spec recommends one column. |
| `supersedes_id`, `blocked_on_decision_id` | E1 migration | Net-new versus canon, and load-bearing for `FOLLOWS_FROM`. |
| Plan limits: per-seat or per-tenant | C2 quota arithmetic, go-live | Needs Pricing and GTM plus a canon amendment. |
| Derived usage producer ownership | 3 of 4 metered dimensions | |
| Stripe account, products, prices, tax | C2 P13 surface | Longest lead time, near-zero cost. Start now. |
| Jira or Linear | E5 action sync | R8 caps this at exactly one. |
| Google verification, Slack app | E5 | Waiting-time. Start now. |
| Email provider and sending DNS, APNs and FCM | B3 | Waiting-time. Start now. |
| `mytheus-web` scope | most UI wiring | See above. |
| mytheus-mobile gap-21 scoping | 8 of Sushan's 25 mobile tasks | |

## Still deliberately unbuilt

Only these remain genuinely out of scope, and each has a stated reason:

- Everything in the roadmap's Stage 2 list above
- `ContextGapAgent` and A2A are P10, `ProactiveAgent` is P11, and R6 pre-authorises descoping A2A
  to RAG-only. In the 36 weeks, but later, and not to be built as if mandatory.
- Ad-hoc Cypher, given up by the R1 recommendation, with no caller today

## Ground rules

1. **Verify before claiming done.** Run it and see it work. This audit exists because status docs
 claimed completion that source disproved.
2. **Cite `file:line` in PRs.** It is what made this exercise checkable.
3. **Do not trust `work-done-till-now/`, `PROGRESS.md`, or `docs/PAGES.md`** - all three contain
 claims the audit disproved. `mytheus-ui/docs/PAGES.md:52` marks `/billing` "done" against a
 service with zero routes. Fix them as you go.
4. **`mytheus-ui` is sensitive** - active rebuild branch, unsettled scope. `git fetch` first.
5. **If a task turns out to be already done, say so and close it.** Two first-pass findings did not
 survive verification; that is a good outcome, not a failure.
