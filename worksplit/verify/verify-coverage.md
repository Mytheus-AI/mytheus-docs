# Coverage matrix — 16 audit reports vs. SUSHAN/SAMAR/SHAUN work split

Adversarial read. Every distinct actionable finding across `mytheus-docs/worksplit/audit/*.md`,
classified ASSIGNED (with task id) / DEFERRED (with the list that defers it) / **UNASSIGNED**.

Source of assignments: `mytheus-docs/worksplit/{README,SUSHAN,SAMAR,SHAUN}.md`.
Source of deferrals: the "Do NOT build these" section at the foot of each work file, plus
README's "What is explicitly NOT in this plan".

Spot-checks run against live source, not against the audits:
- `mytheus-backend/services/api_gateway/edge.py:116` — `_attach_internal_headers` **is** outside
  the `try/except ProblemDetail` (closed at `:112-114`). S1's premise is correct and unfixed.
- `mytheus-backend/services/core_api/routes/transcript.py:123` — `where = ["meeting_id = ...",
  "is_final = true"]`, no `deleted_at`, no `created_at`. H2 unfixed.
- `mytheus-backend/services/core_api/routes/speakers.py:97-133` — no `users` probe; the
  "the FK is the enforcement" comment is still there verbatim. H1 unfixed.
- `mytheus-backend/libs/shared/` — **there is no `consumer.py`.** See ERROR-2.
- `mytheus-platform/deploy/compose.prod.yml` — 17 service keys, matching the audits exactly.

---

## Part 1 — UNASSIGNED findings (the answer to the question asked)

Ordered by significance. "Not deferred" below means: absent from all three "Do NOT build"
lists and from README's not-in-plan section.

### U1. `FakeProvider` writes fabricated transcripts into real tables, ungated
`audit-rt-transcription.md` §3-4, §MISSING.
`providers/factory.py:20-23` returns `FakeProvider` whenever `DEEPGRAM_API_KEY` is unset — which
is every environment. `SessionTracker.on_participant_joined` (`sessions.py:75-88`) opens that
stream on a `meeting-room-events` `joined` and `_pump` drains its canned script through the
identical `persistence.process_utterance` path a real provider uses, writing a `transcript_chunks`
row reading *"hello there, how are you"* and publishing it to the real `transcript-chunks` topic.
Nothing marks it synthetic. `devseed.py:222-229` guards the *hand-run* seeder behind
`TRANSCRIPTION_DEV_SEED=1` for exactly this reason and states it (`devseed.py:32-39`); the default
runtime path has no guard at all.
**Why this is the top item:** it is harmless today only because transcription-service is not
deployed. **M4 + M5 are the tasks that deploy it.** Samar's list arms this defect and no task
disarms it. It must be a precondition on M4/M5 the way M11 is.

### U2. The AI and realtime tiers have no ingress after M4/M5 give them containers
`audit-xcut-auth.md` MISSING PIECES #7; `audit-ai-ingest.md` (Hop 6 note).
`services/api_gateway/edge.py:146-180` `ROUTE_MAP` has **no entry for any intelligence path**, and
`deploy/caddy/mytheus.caddy` has **no WebSocket route** for `realtime-gateway`. M4 builds the
images and M5 adds the compose entries; nothing routes a request to any of them. `search-service`,
`/embed`, `/rerank`, `/ingest` and the Socket.io namespace stay unreachable from outside the docker
network. Not deferred — M4/M5 are premised on these tiers becoming usable.

### U3. `meetings.status` is never reconciled with `meeting_rooms.status`
`audit-be-meeting.md` D-2, PARTIAL.
**Nothing anywhere sets `meetings.status = 'live'`** — not the webhook (`webhook.py:230-235` moves
only `meeting_rooms.status`), not the consumer, not the token mint. Only a client
`PATCH /meetings/{id}` moves it. A meeting with people actively in the room reads `'scheduled'` to
every consumer of `GET /meetings` and `GET /meetings/{id}`. Directly undermines H5, which wires
`/meetings` to that endpoint and will render permanently-wrong statuses.

### U4. Nothing ever writes `meeting_speakers` with `assignment = 'suggested'`
`audit-be-meeting.md` MISSING; `audit-be-data.md` (schema table, `meeting_speakers` row).
`speakers.py:97-122` is the only writer in the codebase and always writes `'manual'`;
`transcription-service` explicitly never writes the table (`speakers.py:5-6`). So
`GET /meetings/{id}/speakers` returns `[]` until a human PATCHes something, `SpeakerMapping.tsx`
never renders (`speakers.length > 0` is never true — `audit-ui-pages.md` ORPHAN COMPONENTS), and
the DB trigger `meeting_speakers_manual_is_authoritative` guards a writer that does not exist.
H4 and H8 both touch the speaker surface and neither addresses the empty-by-construction list.

### U5. `bff/people.py:43` imports identity-tenant-service in-process across a container boundary
`audit-be-bff-gateway.md` "The one structural defect in BFF", finding (1) of 4.
`from services.identity_tenant_service.context.service import get_user_context`, executed inside
the **bff** container against **bff's** session. Its justification ("one deployable, every service
a Python module", `people.py:9-14`) expired: compose runs them as separate containers
(`compose.prod.yml:269-273` and `:277-286`). Two processes now execute the same code against the
same DB and both write the same Redis key `nexus:{org_id}:user_ctx:{user_id}`. Named as one of
that audit's four action items; in no work file.

### U6. Backend consumer workers are still absent from compose after M5
`audit-events-matrix.md` "The deploy gap that dwarfs everything else"; `audit-be-coreapi.md`
PARTIAL 2.
M5 enumerates the *intelligence* and *realtime* workers. It omits `python -m
services.core_api.consumer` (presence-events) and `python -m services.meeting_service.consumer`
(meeting-events) — both fully built, both inside the image that is already deployed, neither
running. `meeting-service` currently ships to prod producing `meeting-room-events` with no
consumer, and `realtime-gateway` produces `presence-events` with no consumer.

### U7. `notification_service` consumer commits offsets on messages it discards
`audit-be-stubs.md` notification-service, "two distinct problems" (a).
`consumer.py:30-32` — `await _handle(msg)` (a documented no-op, `:37-41`) then
`await consumer.commit()`. Every notification it sees is destroyed rather than left on the topic.
Inert only because it has no compose service. **M5 is the task that adds compose services for
consumers.** Same shape of hazard as U1: a Samar task arms it and no task disarms it.

### U8. The whole mobile audit
`audit-mobile.md`, in full. Not one finding appears in any work file or deferred list; the README
table has no mobile column. At minimum:
- 12 of 13 screens are scripted `src/data/mock.ts`; only Home reads `/bff/dashboard`.
- `useTeams()` / `useDepartments()` (`src/api/hooks/index.ts:41-54`) are wired to live, working
  backend routes and **imported by no screen** — the same "built and not connected" pattern H6
  fixes on web.
- `app.json` declares `RECORD_AUDIO` and an iOS `NSMicrophoneUsageDescription` making a specific
  data-handling claim, with **zero recording code** in the app. App-Store-review / privacy-honesty
  risk, and the same class of defect as `/students/verify` which Shaun's list at least flags.
- `mytheus-0.1.0.apk` (83.7 MB) and `android/app/debug.keystore` are **committed to git**; no
  `eas.json`; no mobile CI of any kind; zero component/screen tests.
- The entire mobile push tier is specified against `notification-service`, including a net-new
  Expo device-token registration contract the backend must expose (`audit-be-stubs.md`,
  mobile/system.md N7) and does not.

### U9. UI has no test infrastructure at all
`audit-platform-deploy.md` §5.3; `audit-docs-spec.md` cross-cutting blocker 2.
`mytheus-ui/.github/workflows/ci.yml` runs `tsc --noEmit` + `next lint` + an image build, and its
own header states Vitest and Playwright are absent because "a job that runs zero tests and reports
green is worse than no job". H4, H5, H6, H7, H8, H9, S6 are all UI changes with no way to write a
regression test. The plan's ground rule 1 ("verify before claiming done") is unenforceable here.

### U10. `_arm`'s broad `except Exception` will mask authz errors as "degraded"
`audit-be-bff-gateway.md` Finding 3.
`dashboard.py:146-148` catches `Exception` per arm and returns `status: "degraded"`. The audit's
own words: *"Low severity today (the only real arm is a plain SELECT), worth noting before the
three empty arms get real loaders."* **H7 is the task that gives them real loaders.** An RLS or
permission failure will surface to the client as a slow panel.

### H7 is over-scoped against its own evidence — see ERROR-1 in Part 3.

### U11. Nothing anywhere reads the dead-letter queues
`audit-be-data.md` EVENT INFRA "What is missing".
Rows dead-lettered by `relay._dead_letter` and by every consumer land on `{topic}.dlq` with no
consumer and no replay tooling. M2/M8 create and count the `.dlq` topics; H10 fixes their
naming and payload format. Nothing reads them. Combined with U6 (consumers not running) and the
absence of any lag metric, a poison message is permanently invisible.

### U12. No startup env validation anywhere; `required: false` on every compose env_file
`audit-be-data.md` "Config loading mechanism"; `audit-platform-deploy.md` §7.3 risk 1.
`pydantic-settings` is a declared dependency and `BaseSettings` is subclassed nowhere; every
service reads `os.environ` at point of use, so a missing var surfaces on the first request that
touches it. Compose compounds it: all ten `env_file` entries carry `required: false`
(`compose.prod.yml:65,73,75,88,145,165,167,214,216,382`), so a missing secrets file renders a
valid compose with an empty environment and no error — documented as having already happened
(`:24-26`). **This is the generating mechanism behind S1, behind `prod.ui.env`, and behind the
Clerk-template gap.** S1 and M10 each fix one instance; nothing fixes the class.

### U13. Nothing verifies that a gateway `ROUTE_MAP` prefix corresponds to a real route
`audit-be-bff-gateway.md` caveat 5 + SPEC DRIFT closing paragraph.
Nine mapped prefixes resolve to a 404 in production (`/api/v1/worklogs`, `/decisions`, `/actions`,
`/suggestions`, `/orgs`, `/rbac`, `/billing`, `/usage`, `/plans`, `/notifications`,
`/admin/tenant`, `/admin/ops`, `/oauth`, `/webhooks`). `tests/test_gateway_edge.py:270` and
`test_gateway_proxy.py:175-176` **assert that routing as correct**, so the suite is green on a
route map to nowhere. api-gateway also has no emitted OpenAPI document, so the real public
contract is written down and machine-checked nowhere. S5 fills two of the prefixes; the missing
gate is unassigned.

### U14. `emit_openapi --check` exits 0 when run with the wrong interpreter
`audit-be-bff-gateway.md` SPEC DRIFT, parenthetical.
Running it with a Python that lacks fastapi fails with `ModuleNotFoundError` **and still exits 0**.
It is used as a CI gate. Same class as the oasdiff finding M10 covers, and cheaper to fix.

### U15. `mytheus-intelligence` and `mytheus-realtime` have no CI whatsoever; `mytheus-mobile` neither
`audit-platform-deploy.md` §5.1.
M4 asks for "CI image build + push for both", which is the *image* job only. Nothing lints,
type-checks or runs the tests of the 7-service AI tier or the 2-service realtime tier on any event
— including the sustained-load test that caught the `asyncio.run`-per-message bug. Also unassigned:
`mytheus-platform`'s own CI, where `make lint` (the **only** gate that validates
`compose.prod.yml`) is never invoked by any workflow, and `make test` is `check_authz.py || true`
(`audit-platform-deploy.md` §5.3 "Critical wiring gap").

### U16. Two of four gitleaks gates are probably dead
`audit-platform-deploy.md` §7.3 risk 7.
`mytheus-backend` replaced `gitleaks/gitleaks-action@v2` with the free binary after it began
hard-failing with "License key is required" post-org-move. `mytheus-events/ci.yml:68` and
`mytheus-platform/ci.yml:54` still use the action — same org, same licensing condition. Three more
repos have no gitleaks gate at all (no CI). Given the mobile repo commits a keystore, this matters.

### U17. `POST /api/v1/chat` has zero UI callers; there is no chat-history reader at all
`audit-ui-pages.md` MISSING APP PAGES; `audit-be-coreapi.md` MISSING.
The in-room `ChatPanel` (`MeetingRoom.tsx:333-382`) is pure local `useState` with a **hardcoded
sender name `{ from: "Yash Maurya" }`** (`MeetingRoom.tsx:343`) on every message regardless of who
is signed in. Meanwhile `POST /chat` is real, deployed and already called by `realtime-gateway`.
And there is **no `GET /chat`** anywhere — a client that misses the socket relay can never recover
chat history, unlike transcript which has an explicit re-hydrate endpoint for that exact reason.
M1 fixes the partition so the writes stop failing; nothing makes the writes readable.

### U18. `(app)/meetings/[meetingId]` detail page is mock, and `GET /api/v1/meetings/{id}` is unused
`audit-ui-pages.md` row 3; `audit-ui-integration.md` "zero callers" list.
Five mock sets (`mockMeetings`, `mockMeetingDetails`, `mockDecisions`, `mockActions`,
`mockTranscriptChunks`) and a `generateStaticParams` that enumerates mock IDs. H5 wires the list
page and H4 unbreaks the transcript page; the detail page **between** them is skipped, so the
navigation path list → detail → transcript still breaks in the middle on a real meeting id.
Same cost class as H5, and the backend already ships.

### U19. Mock fixture data is shipped to anonymous visitors on the public landing page
`audit-ui-pages.md` MOCK DATA INVENTORY.
`components/landing/Statement.tsx:18-19` imports `mockTeamCapacity` and `mockSourcedAnswer` into
the public hero animation. Not in the deferred marketing list (which covers the 11 feature pages,
not the landing hero).

### U20. `/settings` and `/settings/profile` render "Saved." while writing nothing
`audit-ui-pages.md` rows 11-12.
`settings/page.tsx:91` `onClick={() => setOrgSaved(true)}` then renders "Saved." at `:94`;
`settings/profile/page.tsx:83` the same. This is the identical class of user-facing false claim as
`/students/verify`, which Shaun's deferred list at least flags for the scope conversation — but
unlike `/students/verify`, the backends **exist**: `PATCH /tenant-settings` and
`PATCH /users/{user_id}` are both implemented and deployed. `settings/profile` also presents
hardcoded `useState("Founder")` / `useState("Product, frontend, fundraising")` as the user's own
profile.

### U21. `org-context.ts` hardcodes `role: "member"` — S6 cannot be implemented without fixing it
`audit-ui-integration.md` AUTH hole 3; `audit-ui-pages.md` supporting-infrastructure note.
`src/lib/api/org-context.ts:40,46` return `plan: "starter"` and `role: "member"` as least-privilege
placeholders. S6 says "gate the `(admin)` layout on `rbac_tier >= admin`" — there is no source of
`rbac_tier` in the UI today, and the only surface that could supply it
(`GET /context/{user_id}` on identity-tenant-service) has **zero callers** and lives in the
generated client H8 is meant to start using. This is an unflagged S6 → (S5 + H8) dependency, not
merely a missing line.

### U22. Reconciliation is built and has no caller
`audit-be-identity.md` PARTIAL, SECURITY HOLE 1.
`reconcile_tenant` (`reconciliation.py:47`) is correct and idempotent; its docstring says a
Temporal schedule in `workflow-service` supplies the roster, and **no such caller exists** (the
workflow bodies are `pass`). This is the hourly sweep that would catch the offboarded users S4's
dead `user.deleted` branch drops. S4 fixes the webhook; the backstop stays unwired.

### U23. Per-tenant rate limiting in api-gateway is a stub
`audit-be-bff-gateway.md` caveat 4.
Acknowledged at `services/api_gateway/main.py:9-11`; no rate-limit code exists in the service. Only
Cloudflare's global limit sits in front. One tenant can exhaust the single VPS for every other
tenant — which composes badly with U24 (no resource limits until M6) and the co-tenant product.

### U24. Ingress is unverifiable from source; three docs contradict the Caddy config
`audit-be-bff-gateway.md` caveats 1-2.
`deploy/README.md:16`, `README.md:62` and `compose.prod.yml:406` all say Caddy proxies
`api.mytheus.cloud` → `web:8000`, **bypassing the gateway**; only `deploy/caddy/mytheus.caddy:53`
says `api-gateway:8000`. No script in any repo deploys that Caddy file (`mytheus-deploy.sh` is not
in version control — M10 covers that script, not the Caddy question). So **whether the gateway is
in the live request path at all is UNVERIFIED**, which is the premise of S1, S8, S9 and M3.

### U25. Least-privilege / hardening residuals, individually small
All not deferred, all with a named citation:
- `libs/auth.ts:43` (realtime-gateway) passes **no `algorithms: ["RS256"]` allowlist and no
  required-claims set** to `jwtVerify`, unlike both Python verifiers. `audit-xcut-auth.md` tier table.
- `verify_token` checks neither `aud` nor `azp` (`libs/shared/auth.py:94-103`).
  `audit-be-identity.md` hole 6.
- `reconciliation.py:64,93-100` and `routes/auth.py:196-201` issue unscoped
  `SELECT/UPDATE ... users WHERE clerk_user_id = :cuid`, safe **only** via `set_rls`. One refactor
  onto a session without the GUC is a cross-tenant mass soft-delete. `audit-be-identity.md` holes 3-4.
- `X-Forwarded-For` trusted unconditionally for the audit `ip` (`libs/shared/authz.py:85`).
- LiveKit grants are **uniform for every role** — `transport.py:122` gives `can_publish=True` to an
  `attendee` and a host alike; `role_in_meeting` rides only in token metadata.
  `audit-be-meeting.md` step 5.
- `TIER_RANK = {member,admin,owner}` cannot represent a **platform-operator** role, yet `/ops` is a
  cross-tenant console. `audit-xcut-auth.md` MISSING PIECES #3. Adjacent to S2/S5/S6, in none of them.
- `tls internal` + Cloudflare SSL mode Full (not Full strict) — unauthenticated origin certificate.
  `audit-platform-deploy.md` §3, §7.3 risk 5.
- `prod-bootstrap.sql` must be mode 0644; confidentiality of the Postgres superuser password rests
  **entirely** on the parent directory staying `0700 root`. `audit-platform-deploy.md` §7.3 risk 2.

### U26. Env/config divergences not covered by M5's single bullet
M5 fixes `mytheus-intelligence/.env.example`'s Redis port. Still unassigned:
- **mytheus-backend's own code defaults are `redis://localhost:6379`** in six places
  (`app_factory.py:23`, `cache.py:39`, `routes/auth.py:115`, `webhook.py:292`,
  `consumer.py:259`, `core_api/consumer.py:131`) while the dev compose and `.env.example` publish
  **6380**. `audit-be-data.md` optional-vars table.
- `TEMPORAL_TASK_QUEUE` is `mytheus` in `env/prod.env.example:38` and `mytheus-backend` in
  `mytheus-backend/.env.example:19`. A mismatch queues workflows forever with no error.
- `staging.env.example` has **no per-role DB split**, so staging structurally cannot prove the D1
  fix that prod depends on.
- `MYTHEUS_OPENAPI_DIR` is undocumented in `.env.example` and exists only in a failure message.
- ADR-0023 says Cloudflare **R2**; the code names an **S3** bucket and describes AWS credentials
  handed to LiveKit egress (`meeting_service/storage.py`). No R2 endpoint/bucket/credential exists
  in any env example in any repo.

### U27. `tenant_settings`: five of eight columns have no read or write path
`audit-be-data.md` schema table; `audit-be-identity.md` PARTIAL.
`sso_config`, `retention_days_transcripts`, `retention_days_worklogs`, `default_locale`,
`agent_confidence_overrides` are written by nothing and read by nothing. The two retention columns
are the ones a compliance claim would rest on. (`sso_config` is fairly covered by the SSO deferral;
the other four are not.)

### U28. Smaller, genuinely unassigned
- `list_meetings` `status` filter is an unvalidated `str` — `?status=Scheduled` returns an empty
  page instead of 422 (`meetings.py:249`). `audit-be-coreapi.md` PARTIAL 4.
- No retry, no timeout, no `AbortSignal` on any UI fetch; `ApiError.isAuthFailure`
  (`client.ts:51-53`) has **zero call sites**. A hung gateway hangs an RSC render unbounded.
  `audit-ui-integration.md` ERROR HANDLING "Absent".
- **Outage is indistinguishable from empty**: `meeting-transcript/route.ts:52-55` returns HTTP 200
  `{items: []}` for any non-`ApiError`, swallowing DNS failure, connection refusal and timeout
  against a *configured* gateway. Interacts directly with H4.
- `lib/api/present.ts:15`, `lib/api/org-context.ts:22` and `components/app/dashboard/format.ts:1`
  import **types from `lib/mock/data.ts`** — the real API layer depends on the mock module for its
  shapes, which blocks ever deleting it. H5 explicitly says "do not delete the module yet" without
  naming this as the reason it cannot be deleted.
- `AgentActivityStrip.tsx:14-30` presents three fabricated agent actions on the dashboard as things
  *your agent did*; `StatGrid.tsx:16-23` fabricates an open conflict. H7 names only
  `dashboard/page.tsx` line numbers, not these two component files.
- D-UI-3: four in-source comments assert "no gateway in any environment" while `.env.local` and
  `.env.example` both set `API_GATEWAY_URL`. `audit-ui-pages.md` ranked remediation #5.
- `deploy.sh:87` **counts** topics and subjects; it does not check names. 42 wrong topics pass the
  gate M2 preserves. `audit-events-matrix.md` BROKER & DEPLOY.
- 13 of 21 topics have **no canonical idempotency-key formula** in
  `mytheus-events/codegen/_schemas.py:52-61`; four producers invent their own, unreviewed by the
  contract repo. The outbox unique index is the only enforcement.
- `transcript_chunks` reaches Redpanda **only if mytheus-backend's relay is running**, because
  mytheus-realtime writes cross-repo into `public.outbox`. Documented in one docstring
  (`transcription_service/outbox.py:1-19`) and nowhere operational. Matters the moment M4/M5 land.
- `organizations.plan_id` is an unconstrained orphan uuid, documented at
  `provisioning.py:91` as an FK to a `plans` table **that no migration creates**.
- `service_role` still holds `rolbypassrls` and the image's
  `ALTER DEFAULT PRIVILEGES FOR ROLE supabase_admin` survives — both need SUPERUSER to fix, both
  self-disclosed at `9b2c4d1e6f30:96-110`. Only `test_no_bypassrls_role_holds_privileges_on_a_public_table`
  would catch a re-grant.
- `mytheus-intelligence`'s Qdrant isolation tests **silently skip** unless
  `MYTHEUS_REQUIRE_QDRANT=1` (`tests/conftest.py:84-89`). Matters the moment M4 gives that repo CI.
- `org_id` is rendered `.hex` for the Qdrant collection name and `str()` for the payload filter in
  adjacent files; a well-meaning "normalization" would make the payload backstop match nothing
  while every isolation test still passes. `audit-ai-ingest.md` "One residual fragility".
- `mytheus-ui`'s deploy job has **no post-deploy verification at all** and the repo has no tests,
  so a UI deploy is unverified end to end. `audit-platform-deploy.md` §5.4.
- `tufin/oasdiff:latest` is unpinned and the gate is `if: pull_request` only — it never runs on
  push to main. M10 says "confirm it runs"; these two are extra.
- No rollback automation and no automated pre-deploy snapshot; rollback is a documented manual
  image re-pin. M6 covers volume backups, not rollback.
- ADR-0016 justified **9 containers**; 17 run, plus a second product's whole stack and Dokploy, and
  the ADR's follow-up 3 (where the frontend is hosted) was resolved de facto without updating it.
  M6 says "sanity-check the capacity story" but nothing supersedes the ADR.
- Six stale `mytheus-ui` docs still describe auth as bypassed (`PROGRESS.md:70`, `LANDING.md:11,60`,
  `docs/PAGES.md:31`, `CLAUDE.md:5`, `README.md:15`, `docs/CONTRIBUTING.md:29`), plus
  `docs/PAGES.md:52`'s false `/billing … done` row and `bff/main.py:1-6`'s claim that the UI is the
  only `/bff` caller (mobile calls it too). Ground rule 2 asks people to fix docs "as you go"; no
  task owns them.
- `core_api/main.py:3-4` "Owns:" names `worklogs`, `decisions`, `action_items`,
  `agent_suggestions`, `agent_collaborations` — five tables no migration creates.
- No delete/cancel route for meetings anywhere, though every read filters `deleted_at IS NULL`;
  soft delete is reachable only by direct SQL. `audit-be-meeting.md` MISSING.
- `@app.on_event("shutdown")` (`proxy.py:155`) is deprecated in current FastAPI.

### Formally-undecided items (not findings to build, but decisions nobody owns)
These are real absences that no work file assigns **and** no deferred list defers. They need an
explicit "deferred, here is the ADR" rather than silence:
- **`integration-service` produces nothing**, so `mytheus-intelligence`'s complete, tested
  `IntegrationEventConsumer` (`consumer.py:75`) can never fire. The stubs audit calls this "the
  hard break, and the one to fix **or formally defer** first". Shaun's list defers the
  `/settings/integrations` *page*, not the producer.
- **`workflow-service` is deployed and idles**: `activities=[]` (`worker.py:35`), four workflow
  bodies that are `pass`, `pass`, `pass`, and four `sleep` calls that send nothing. A privileged
  DB role is being scoped for the RTBF workflow (`libs/shared/db.py:69`) that does nothing.
- **`admin-ops-service` has zero routes** while the gateway routes `/admin/tenant` and `/admin/ops`
  to it, and the roadmap puts both consoles in MVP scope at P12.
- **`start_egress` / recording**: `transport.py:135-137` raises, no caller, no bucket. The
  *receiving* half (`webhook._apply_egress_ended`) is fully built and unreachable. README's
  not-in-plan list covers audio capture, not recording.

---

## Part 2 — ASSIGNED (finding → task). Confirmed traceable.

| Finding (audit) | Task |
|---|---|
| `GATEWAY_SIGNING_KEY` unset + raise outside the guard → 500 on every authed request (`xcut-auth`, `platform-deploy` B1, `be-identity`) | **S1** |
| No `INSERT INTO rbac_grants` outside `seed.py:196`; Clerk webhook writes no grant → every route 403s (`be-identity` MISSING 2, `be-data` write-orphan) | **S2** |
| No handler for six `organization.*` / `organizationMembership.*` events (`be-identity` MISSING 4) | **S3** |
| `_clerk_org_of` dead on `user.deleted`; the test fabricates memberships (`be-identity` hole 1) | **S4** |
| `/orgs` and `/rbac` route to identity-tenant-service, which implements neither (`be-identity` MISSING 1-2, `bff-gateway` caveat 5) | **S5** |
| `(admin)/layout.tsx` has session but no role check; `OPS_CONSOLE_ENABLED = true` (`ui-pages` D-UI-2, `xcut-auth` finding 2) | **S6** |
| gRPC `add_insecure_port`, unsigned `org-id` metadata (`xcut-auth` finding 1, `be-identity` hole 5) | **S7** |
| `verify_context` has no production caller; `stride.md:25` claims otherwise (`xcut-auth` finding 4, `bff-gateway` finding 1) | **S8** |
| `PUBLIC_CONTAINS` substring match (`xcut-auth` finding 5) | **S9** |
| `chat_messages` has no partition covering today; no pg_partman (`be-data` defect 1) | **M1** |
| `deploy.sh` verifies 42 topics nothing creates; mytheus-events not on the VPS (`platform-deploy` §5.6, `events-matrix`) | **M2** |
| 7/16 healthchecks; api-gateway + ui ungated; `/healthz` public; transcription `/healthz` green with a dead consumer (`platform-deploy` §1.1, `bff-gateway` caveat 3, `rt-transcription` MISSING) | **M3** |
| No Dockerfile in mytheus-intelligence or mytheus-realtime (`ai-ingest`, `ai-rag-agents`, `rt-transcription`, `platform-deploy` §2.1) | **M4** |
| Consumer workers / Qdrant / LiveKit / MinIO absent from compose; intelligence Redis port mismatch; `CLERK_JWT_INTERNAL_ORG_CLAIM` unprovisioned (`platform-deploy` §3, `ai-ingest`) | **M5** |
| Zero `deploy:`/`resources:`/`mem_limit`/`cpus`/`logging:`; no backups; no observability; `observability/` dir does not exist (`platform-deploy` §1.1, §8, §9, `be-data`) | **M6** |
| `tenant-events` has no consumer, so org suspension never propagates (`events-matrix` dead-letter 2, `be-identity` outbound 8) | **M7** |
| `user-sync-events` is a topic ADR-0010 D4 says should not exist (`events-matrix` orphan) | **M8** |
| `migrations/_grants.py` imported by zero migrations; cross-repo GRANT gap in mytheus-realtime (`be-data` defect 2-3, `be-coreapi`) | **M9** |
| `prod.ui.env` has no example; `NEXT_PUBLIC_*` not build args; CD races; oasdiff never ran; `deploy.sh` only on the VPS (`platform-deploy` §4.4, §5.4, §6, `ui-integration`) | **M10** |
| transcription-service connects as `postgres`, bypassing its own correct RLS; no `rolbypassrls` runtime check (`xcut-auth` finding 3, `rt-transcription`, `be-data`) | **M11** |
| `assign_speaker` trusts an FK for tenancy (`be-coreapi` PARTIAL 1, `be-meeting` D-1) | **H1** |
| transcript missing `deleted_at IS NULL`; no `created_at` bound; zero tests (`be-coreapi` defects A/B + test hole) | **H2** |
| `audit_logs` is write-only; no reader, no export route; `admin_ops_service` has zero routes (`be-data`, `be-stubs`) | **H3** |
| `transcript/page.tsx:56` gates the only real page on `mockMeetings.find()` (`ui-pages` D-UI-1, `ui-integration` #6) | **H4** |
| `/meetings` renders `mockMeetings` over a working, mobile-proven endpoint (`ui-pages` row 2) | **H5** |
| `/org` renders `mockDepartments`; `/bff/org/chart` + `/bff/people` have zero callers (`ui-pages`, `bff-gateway` dead surface) | **H6** |
| Three of four `/bff/dashboard` arms are hardcoded `[]` (`bff-gateway`, `ui-pages` row 1) | **H7** (see ERROR-1) |
| `identity-tenant-service.ts` and `meeting-service.ts` have 0 importers; hand-written `TokenResponse` already drifted (`ui-integration` GENERATED TYPES) | **H8** |
| Clerk token fetched once on mount; socket + LiveKit re-handshake stale (`ui-integration` MISSING PLUMBING 1) | **H9** |
| Zero shared-schema adoption; DLQ name split; DLQ payload JSON vs Avro contract (`events-matrix` SHARED-SCHEMA ADOPTION, drifts 1-2) | **H10** |
| Room name `meeting-{id}` carries no org segment; fake-sync marks rooms permanently synced (`xcut-auth` finding 6, `be-meeting` D-4) | **H11** |

## Part 2b — DEFERRED (correctly)
SSO/SAML/SCIM (Sushan) · 428 consent check per ADR-0018 (Sushan) · ADR-0013 agent authz (Sushan) ·
`pii_tokens` per ADR-0020 key custody (Sushan + Shaun) · K8s/Terraform/Helm, Wave 1-4 extraction,
multi-region/cells/Citus, BYOC Redpanda, int8 quantization, GPU autoscaling, `nexus-deploy`
(Samar) · `/decisions` `/tasks` `/search` `/billing` `/settings/integrations` wiring, 10 of 11
marketing feature pages, `/students/verify`, knowledge-graph UI (Shaun) · no LLM anywhere,
rag/agent-runtime/knowledge-graph stubs, no `decisions`/`action_items` table, audio capture and
the Deepgram DPA (README).

All check out against `audit-docs-spec.md`'s W36 phase boundary. One caveat: README's not-in-plan
section correctly names these but does not defer U-items above that sit *next* to them —
recording, `integration-service`, `workflow-service`, `admin-ops-service` routes.

---

## Part 3 — Reverse errors

**ERROR-1 — H7 asks for work that cannot be done, on its own evidence.**
"Implement the three `[]` arms in the BFF against real queries." There is **no `decisions`,
`action_items`, `worklogs`, `agent_suggestions` or `agent_collaborations` table in any migration in
any repo** — README says so itself, and `bff/dashboard.py:94-97` names the type `UnbuiltArm` and
`:204-208` gathers them from a function literally called `not_yet_built`. H7's own third bullet
then concedes it ("return `empty` honestly rather than inventing a number"). So H7 is really two
small UI tasks (delete the mock fallbacks in `dashboard/page.tsx`, `StatGrid.tsx`,
`AgentActivityStrip.tsx`) plus a no-op. It is sized and worded as a backend task. Rewrite it.

**ERROR-2 — M7 instructs Samar to reuse a module that does not exist in that repo.**
M7: *"`libs/shared/consumer.py` already gives you Redis dedupe, durable inbox, DLQ and
single-transaction apply. Do not hand-roll."* Verified: `mytheus-backend/libs/shared/` contains no
`consumer.py`. `BaseConsumer` lives in **mytheus-intelligence**. `audit-be-data.md` states the
gap explicitly: *"No shared consumer harness. There is a `RelayProducer` Protocol but no `Consumer`
counterpart"* — each backend consumer re-implements the loop (`core_api/consumer.py:119-123`,
`meeting_service/consumer.py:247-251`, `notification_service/consumer.py:14`), and the repo carries
**two** Kafka client libraries (`confluent-kafka` and `aiokafka`). M7 is therefore materially
larger than written: Samar must either hand-roll a fourth consumer or first build the harness.

**ERROR-3 — S1 and S8 are ordered wrong relative to each other.**
S1 is a P0 that adds `GATEWAY_SIGNING_KEY` to three env files plus a runbook plus a rotation
procedure. S8 then asks Sushan to *decide* whether the signing mechanism should exist at all, and
notes in its own text: *"Option B also removes the S1 failure mode entirely."* If he picks B, the
env and runbook work in S1 is thrown away. **The S8 decision is five minutes and belongs before
S1**; only the try/except move and the 503 test are unconditionally needed.

**ERROR-4 — M9 and M11 double-count the same work.**
M9's last bullet ("cross-repo GRANT gap … still unapplied in `mytheus-realtime`") and M11's last
bullet ("Close the related cross-repo GRANT gap from M9") are one `GRANT SELECT, INSERT, UPDATE,
DELETE ON transcript_chunks TO mytheus_app, mytheus_admin;` line. Counted twice on one person's
list, which inflates the apparent count without adding work.

**ERROR-5 — unflagged cross-person dependencies.**
- **S6 → S5 + H8.** S6 needs `rbac_tier` in the UI. `org-context.ts:46` hardcodes `"member"` and
  the only surface that could supply the real tier is on identity-tenant-service, whose generated
  client H8 owns and whose first caller S5 creates. S6 is listed as depending only on S2.
- **H7 → M5/M6? No — H7 → nothing, see ERROR-1.**
- **M7 → the missing consumer harness**, see ERROR-2.
- **The Clerk JWT template mapping** (`public_metadata.internal_org_id` → claim) appears on
  Samar's M5 only as an env var. It is **Clerk dashboard state no code can enforce**
  (`clerk_admin.py:21-25`), it 401s the entire AI tier until it exists, and it is Clerk
  configuration — i.e. Sushan's domain, not Samar's. Nobody owns it.
- **M4/M5 → U1 and U7.** Two tasks on Samar's list arm two latent data-integrity defects that no
  task disarms. M11 is correctly flagged as gating M4/M5; U1 and U7 deserve identical treatment.

**Invented work: none found.** All 31 tasks trace to at least one cited audit finding.

**Already-done: none found.** S1, H1 and H2 were spot-checked against live source and are all
still open. S6's work file already corrects the audits' stale "anyone can reach /admin" framing.
H10's note that the packaging blocker "was already fixed in `pyproject.toml`" is accurate.
One caution: S4 rests on Clerk's `user.deleted` payload shape, which `audit-be-identity.md` marks
**UNVERIFIED** (model knowledge, not a live Clerk instance) — verify against a real webhook before
building.

**Cross-person duplication: none.** S5/H3 (audit-logging grant changes), M2/M8/H10 (topic counts
and DLQ names), M5/M11 (role before deploy) and M10/H9 (`NEXT_PUBLIC_*` build args) are all
overlaps that the files already flag with explicit coordination notes. That part was done well.

---

## Part 4 — Balance

Counting only genuinely pending work, **9/11/11 is not an even split.** Task *count* is a poor
proxy here because M4, M5 and M6 are each larger than several of the other lists' tasks combined.

Rough effort (1 unit ≈ a focused day):

| | Large / very large | Medium | Small | Total |
|---|---|---|---|---|
| **Sushan** S1-S9 | S5 | S2, S3, S7, S8 | S1, S4, S6, S9 | **~10** |
| **Samar** M1-M11 | **M4, M5** (very large), M6, M11, M7 | M1, M2, M3, M9, M10 | M8 | **~22** |
| **Shaun** H1-H11 | H3, H10 | H2, H5, H6, H8, H9, H11 | H1, H4, H7 | **~15** |

Notes behind those numbers: M4 is two repos, two runtimes, ML weight baking and CI; M5 is nine
services plus three infra dependencies plus `depends_on` rewiring; M6 is limits + rotation + four
volume backup paths + observability; M7 is bigger than written (ERROR-2). Against that, S1/S4/S6/S9
are genuinely one-sitting tasks and H7 collapses to almost nothing (ERROR-1).

**Samar is carrying roughly twice Sushan and 1.5× Shaun.** Mobile has not landed yet and the split
is already lopsided the other way from what the head-count suggests.

### With mobile added to Sushan
`audit-mobile.md` plus the mobile half of `audit-be-stubs.md` is realistically ~7 units: EAS/release
pipeline, purge the committed APK + `debug.keystore` and rewrite history or accept it, resolve the
`RECORD_AUDIO` honesty problem, wire `useTeams`/`useDepartments` into a screen, mobile CI, some
screen tests, and scope the Expo push-token contract against `notification-service`.

That puts Sushan at ~17 and leaves Samar at ~22.

### What should move — and it is not off Sushan's plate
With mobile, Sushan lands near the mean. **The imbalance to fix is Samar's.** Total pending work is
~47 units across three people ≈ 16 each, so there is no slack to redistribute into: Samar must
*shed* ~6 units, and only ~1 can go to Shaun.

Recommended, in order:

1. **Move M1 (`chat_messages` partition + rolling-partition mechanism) to Shaun.** ~3 units. It is
   a data-correctness task adjacent to H3, which already puts him inside the partitioned
   `audit_logs` migration, and the rolling mechanism must solve both tables anyway.
2. **Move M8 (delete `user-sync-events`) to Shaun.** ~1 unit. H10 already puts him in
   `mytheus-events`; doing both in one pass is cheaper than two people entering that repo.
3. **Descope M6's observability bullet** to "remove `sentry-sdk` and the unimported metrics deps
   from `pyproject.toml` so the dependency list stops implying coverage that does not exist."
   Standing up metrics/tracing on a single VPS is a Plan-14-shaped item; keep limits, log rotation
   and backups, which are the parts that stop the box dying. Saves ~2 units.
4. **Cut M5's LiveKit and MinIO/R2 bullets.** There is **no LiveKit account and no R2
   credential anywhere in the program** (`platform-deploy` §3, `be-data` storage section) — a
   compose entry for either is unbuildable today, not merely unbuilt. Replace with "record the
   two blocked dependencies and their owners." Saves ~1-2 units.
5. **Split M4's CI work out as its own task and give it to Sushan or Shaun**, expanded to cover
   U15 (mytheus-intelligence and mytheus-realtime have **no CI at all**, not just no image build;
   mytheus-platform's own `make lint` is never invoked). ~2 units off Samar. If it goes to Sushan,
   it pairs naturally with the mobile CI he is about to inherit — one person, one CI pass across
   the three repos that have none.

After those five: roughly **Sushan ~17-19, Samar ~15-16, Shaun ~17-19.** That is even.

The unassigned items in Part 1 are on top of all of it. U1, U2, U6 and U7 in particular are not
optional additions — U1 and U7 are defects that Samar's own M4/M5 arm, and U2 and U6 are the
difference between M4/M5 producing running containers and M4/M5 producing *reachable* ones.
