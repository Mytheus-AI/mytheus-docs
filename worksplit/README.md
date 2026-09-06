# Mytheus — Remediation Work Split

Derived from the cross-repository integration audit of 6 September 2026 (16 parallel read-only
auditors, all findings cited to source, status docs deliberately ignored).

**Full audit report:** https://claude.ai/code/artifact/da281c5f-d990-40b4-9487-d9f857833b08

**Raw evidence:** [`audit/`](audit/) — all 16 auditor reports, 7,085 lines, every claim cited to
`file:line`. Read the relevant one before starting a task; the work files below are summaries of
these.

---

## The one-paragraph summary

The vertical slices are strong. The seams between them were never closed. Almost every component
in this system is built to a high standard and wired to a counterpart that does not exist — an
append-only audit table with no writer, a finished Kafka consumer whose producer is an 11-line
stub, a real cross-encoder model that nothing calls, a 924-line generated API client with zero
import sites. On top of that, four small defects currently prevent the deployed system from
serving a single authenticated request, while reporting itself green.

---

## Work files

| Owner | File | Tasks | Theme (not a role — just a coherent slice) |
|---|---|---|---|
| Sushan | [SUSHAN.md](SUSHAN.md) | 9 | Gateway, auth, identity, RBAC |
| Samar | [SAMAR.md](SAMAR.md) | 11 | Platform, deploy, data, event backbone |
| Shaun | [SHAUN.md](SHAUN.md) | 11 | API correctness, audit trail, UI wiring |

Everyone owns backend and frontend work. Nobody is "the infra person" or "the frontend person".

---

## ⚠️ ORDERING — read this before starting

There is exactly one hard dependency in this plan, and it belongs to Sushan.

```
        ┌─────────────────────────────────────────────┐
DAY 1   │  SUSHAN — S1 (gateway key) + S2 (RBAC)      │   ← BLOCKS EVERYONE
        │  Must land and be deployed before any       │
        │  other task can be verified end to end.     │
        └─────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   SUSHAN S3–S8       SAMAR M1–M10        SHAUN H1–H10
   (parallel)         (parallel)          (parallel)
```

### Why S1 and S2 block everything

- **S1** — `GATEWAY_SIGNING_KEY` is unset everywhere, and the code path that needs it sits outside
  its own exception guard. **Every authenticated request through the gateway returns 500.**
  Nobody can test any API change until this is fixed.
- **S2** — no code anywhere writes `rbac_grants`. A real user created through Clerk holds tier
  `""`, which satisfies no tier check. **Every route 403s for every real user.** Even after S1,
  nothing is reachable until this lands.

### What this means in practice

- **Sushan:** do S1 and S2 first, before anything else on your list. Do not start S3–S8 until both
  are merged and deployed. Post in the channel the moment they are live.
- **Samar:** you can start **M4, M5, M6** (Dockerfiles, compose, resource limits) immediately —
  they do not touch the request path. Everything else waits for Sushan's green light.
- **Shaun:** you can **write** all of H1–H10 immediately, but you cannot **verify** any of it
  until Sushan is done, because every request you make will 403. Write with tests first, verify
  after the unblock.

### One soft dependency

Samar's **M4/M5** (Dockerfiles + compose entries for `mytheus-intelligence` and
`mytheus-realtime`) should land before anyone tries to test transcript, embedding or search
behaviour end to end — neither of those repos currently has a Dockerfile or a compose entry, so
none of that code runs anywhere today. Not a blocker for writing code; a blocker for believing it
works.

---

## Ground rules

1. **Verify before claiming done.** This whole audit exists because the status docs said things
   were finished that were not. A task is done when you have run the thing and seen it work — not
   when the code is written and the tests are green.
2. **Do not trust the existing status docs.** `work-done-till-now/`, `PROGRESS.md` and
   `docs/PAGES.md` all contain claims the audit disproved. Fix them as you go if you touch the
   relevant area.
3. **`mytheus-ui` is sensitive.** There is an active rebuild on `origin/local-mytheus-rebuild`.
   `git fetch` and check with Sushan before any structural change there. The UI tasks in this plan
   are deliberately small and surgical.
4. **Do not build anything in the deferred list** (bottom of each file). Those absences are the
   roadmap being followed, not gaps.
5. **Cite file:line in PRs**, the way the audit does. It made this whole exercise checkable.

---

## What is explicitly NOT in this plan

The plan stops at "the platform works". It does not close the product gap, because that is a scope
decision rather than a bug list:

- **There is no LLM call anywhere in any of the 8 repos.** The only inference is two local BGE
  retrieval encoders, neither of which is generative. `litellm` and `langgraph` are installed and
  never imported.
- `rag-service`, `agent-runtime-service` and `knowledge-graph-service` are 13-line stubs.
- No `decisions` or `action_items` table exists in any migration, so two headline features have no
  data model.
- Audio capture is absent end to end; the Deepgram path is correctly blocked on a signed DPA,
  which is a legal dependency, not an engineering one.

Have that conversation separately, once the platform underneath it runs.
