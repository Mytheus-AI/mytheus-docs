# Shaun — API Correctness, Audit Trail, UI Wiring

**10 tasks.** You can write all of them immediately. You cannot verify any of them until Sushan
lands S1 + S2 — every request will 403 until then.

Audit report: https://claude.ai/code/artifact/da281c5f-d990-40b4-9487-d9f857833b08

---

## ⏱️ Ordering

**Write now, verify later.** Nothing on your list is blocked from being *written*. But every one of
your tasks involves making an authenticated request, and today those return 500 (Sushan's S1) and
then 403 (Sushan's S2). Write test-first, verify the moment he announces.

**Soft dependency on Samar's M4/M5** for H10 — the Avro clients matter most once the intelligence
consumers actually run somewhere.

**Start with H1.** It is a live cross-tenant hole and it is small.

---

## H1 — Cross-tenant speaker assignment (HIGH severity, start here)

**Repo:** `mytheus-backend`

`services/core_api/routes/speakers.py:97-133` accepts a caller-supplied `user_id` and relies solely
on a foreign key to enforce tenancy. Its comment claims the FK blocks another tenant's `user_id`.

**That claim is false, and two sibling routes already document it as false:**

- `routes/meetings.py:186-192` — runs an explicit `SELECT 1 FROM users` RLS visibility probe
  *precisely because* the FK does not enforce tenancy
- `routes/org_hierarchy.py:172` — same probe, same documented reasoning

`create_meeting` got the fix. `assign_speaker` never did. No test covers it.

Two independent auditors flagged this — it is the most credible security finding in the audit.

### Do

- [ ] Add the same `SELECT 1 FROM users` RLS-scoped probe before the assignment write.
- [ ] Correct the misleading comment so it does not re-mislead the next reader.
- [ ] Add a test: tenant A attempts to assign a speaker to a tenant B `user_id`, expect a clean
      404 (match the 404-before-403 ordering the rest of the codebase uses — see
      `routes/users.py:80-91`).
- [ ] Grep for any other route trusting an FK for tenancy. There may be a fourth.

---

## H2 — Soft-deleted transcript chunks are served

**Repo:** `mytheus-backend`

`services/core_api/routes/transcript.py:123` is missing a `deleted_at IS NULL` filter, so
soft-deleted chunks are returned to clients.

While you are in this file, two more things the audit found:

- `transcript.py:134` has no `created_at` bound, so the monthly range partitioning on
  `transcript_chunks` is forfeited entirely — every query scans all partitions.
- `routes/transcript.py` has **zero tests**, despite being the one endpoint the live UI actually
  depends on.

### Do

- [ ] Add the `deleted_at IS NULL` filter.
- [ ] Add a `created_at` bound so partition pruning works.
- [ ] Write tests for the transcript endpoint — pagination, the `from_ms` seek, tenant scoping,
      soft-delete exclusion.

---

## H3 — Nothing writes the audit log

**Repo:** `mytheus-backend`

`audit_logs` is genuinely well built: partitioned monthly through 2027-06 plus a DEFAULT,
`FORCE ROW LEVEL SECURITY`, `UPDATE`/`DELETE` revoked per partition, an `AuditLog` model at
`services/admin_ops_service/models.py:25-57`. The append-only guarantee the security architecture
depends on is real at the database level.

`libs/shared/audit.py` exists as the writer helper. But the table has **no reader and no export
route**, and `admin_ops_service` has **zero routes** — so the audit trail is write-only, and much
of what should be audited never calls the helper.

Per the roadmap this is in MVP scope from P2, not P13.

### Do

- [ ] Audit the audit: grep which mutating operations call `libs/shared/audit.py` and which do not.
      Several routes already do it correctly (`routes/users.py:109`, tenant-settings, meeting
      create) — find the gaps.
- [ ] Add audit writes to the mutations that lack them.
- [ ] Add a read/export route so the trail is usable. Gate it on `require_tier("admin")`.
- [ ] Coordinate with Sushan — his S5 (`/rbac`) needs to audit grant changes and will call your
      helper.

---

## H4 — The one working page 404s on real data

**Repo:** `mytheus-ui` ⚠️ *sensitive repo — `git fetch` first, check the active rebuild branch*

`src/app/(app)/meetings/[meetingId]/transcript/page.tsx:56` does:

```ts
const meeting = mockMeetings.find(...)   // :56
if (!meeting) notFound()
// ...
const data = await getTranscript(...)     // :59  ← the real fetch, never reached
```

This is the **only** app page with a genuine backend read, and it gates its own existence on a mock
array. A real backend meeting ID 404s before the live fetch at `:59` ever runs. The real path is
unreachable for real data.

### Do

- [ ] Remove the mock existence gate; let the real fetch determine 404 (`lib/api/transcript.ts`
      already throws `ApiError` and the page already maps 404 → `notFound()`).
- [ ] Add a loading state — the page currently has none.
- [ ] Verify against a real meeting ID once Sushan unblocks and Samar's transcription service runs.

---

## H5 — `/meetings` renders a mock array over a working endpoint

**Repo:** `mytheus-ui` ⚠️ *sensitive*

`src/app/(app)/meetings/page.tsx:6,69-75` renders `mockMeetings`, while `GET /api/v1/meetings`
exists, is deployed, is keyset-paginated, and is already called successfully by the mobile app.

This is the cheapest visible progress available in the whole plan.

### Do

- [ ] Wire the page to `GET /api/v1/meetings` through the existing `src/lib/api/client.ts`.
- [ ] Use the generated `CoreApi` types — do not hand-write the response shape.
- [ ] Add loading, error and empty states (the page has none of the three).
- [ ] Keep the mock import removal clean; `src/lib/mock/data.ts` is imported by 14 files, so do not
      delete the module yet.

---

## H6 — `/org` renders a mock array over a working endpoint

**Repo:** `mytheus-ui` ⚠️ *sensitive*

`src/app/(app)/org/page.tsx:5,13,30` renders `mockDepartments`. `GET /bff/org/chart` exists and is
deployed — and has **zero callers anywhere**, in either client.

`GET /bff/people/{user_id}` is in the same position. Teams and departments CRUD exist in
`core_api/routes/org_hierarchy.py` and are equally unused.

Note for context: mobile has `useTeams()` / `useDepartments()` hooks already wired to these working
routes, and no screen imports them. The plumbing keeps getting built and not connected.

### Do

- [ ] Wire `/org` to `GET /bff/org/chart`.
- [ ] Add loading, error and empty states.
- [ ] Decide whether `/bff/people/{user_id}` gets a caller now or is removed — roughly 55% of the
      BFF's lines are currently dead surface.

---

## H7 — Fill the dashboard's three empty arms

**Repo:** `mytheus-ui` + `mytheus-backend`

`/bff/dashboard` is the UI's main live call, but `services/bff/dashboard.py:194-201` returns
hardcoded `[]` for three of its four arms. Only the meetings arm is real. On the UI side,
`dashboard/page.tsx:10,51,78,81,92,93` fills the gaps with mock values including a mock
`openCount`.

The `ArmStatus` degradation model is well designed — it distinguishes `empty` from `degraded` — so
the shape is right, the data is missing.

### Do

- [ ] Implement the three `[]` arms in the BFF against real queries.
- [ ] Remove the corresponding UI mock fallbacks.
- [ ] Where an arm has no backing data model at all (actions/decisions — see the note at the
      bottom), return `empty` honestly rather than inventing a number. A truthful empty state beats
      a fake count.

---

## H8 — Two generated type modules have zero importers

**Repo:** `mytheus-ui` ⚠️ *sensitive*

The recent OpenAPI work generated four client modules. Two are used, two are dead:

| Module | LOC | Import sites |
|---|---|---|
| `generated/core-api.ts` | 1552 | 3 ✅ |
| `generated/bff.ts` | 527 | 1 ✅ |
| `generated/identity-tenant-service.ts` | 924 | **0** ❌ |
| `generated/meeting-service.ts` | — | **0** ❌ |

Worse, `src/app/api/livekit-token/route.ts:31-35` hand-writes a `TokenResponse` that **already
contradicts** `generated/meeting-service.ts:100-107` on `livekit_url` nullability. The generated
types exist specifically to prevent that.

### Do

- [ ] Replace the hand-written `TokenResponse` with the generated type and resolve the nullability
      disagreement.
- [ ] Type the speakers envelope in `lib/api/transcript.ts` (items are typed, the `{items: …}`
      wrapper is hand-written).
- [ ] Type the speaker PATCH in
      `app/(app)/meetings/[meetingId]/transcript/actions.ts:33-36` — currently untyped with a
      hand-shaped body.
- [ ] `identity-tenant-service.ts` will get its first caller from Sushan's S5 — coordinate.
- [ ] Check `scripts/generate-api-types.mjs`: if it reads specs from a sibling-repo filesystem
      path, that is a fragile cross-repo coupling worth flagging to Samar for CI.

---

## H9 — Realtime auth breaks on reconnect

**Repo:** `mytheus-ui` ⚠️ *sensitive*

`src/components/meeting/MeetingRoom.tsx:45-53` fetches the Clerk token **once on mount** and never
refreshes it. Socket.io and LiveKit both reconnect automatically — and re-handshake with a stale
JWT, which will fail once the token expires.

Mobile already solved this correctly: `src/api/provider.tsx` pulls a fresh token per request via
`useAuth().getToken()`. Follow that pattern.

### Do

- [ ] Refresh the token on reconnect rather than caching the mount-time value.
- [ ] Confirm with Samar that M10 landed — `NEXT_PUBLIC_REALTIME_WS_URL` and
      `NEXT_PUBLIC_LIVEKIT_URL` are not Docker build args today, so they bake as `undefined` and
      this cannot be tested in a container until that is fixed.
- [ ] The client-side realtime contract is fully documented in the audit's UI-integration report if
      you need the exact message shapes — the good news is the UI and gateway contracts **match
      field for field**, so no protocol work is needed here.

---

## H10 — Import the generated Avro clients

**Repos:** `mytheus-events` consumers across `mytheus-backend`, `mytheus-intelligence`,
`mytheus-realtime`

`mytheus-events` generates Python and TypeScript clients from the 21 Avro schemas. **Shared-schema
adoption is zero** — the only reference anywhere is a test asserting the package is *absent*.

Producers currently hand-roll dicts. The audit found no payload-level drift today (both Python
tiers encode through the live Schema Registry, which catches drift at encode time — a genuinely
good safeguard), but hand-rolled payloads are the standing risk that safeguard is compensating for.

The packaging blocker cited in the code as the reason for non-adoption **was already fixed** in
`pyproject.toml`. There is no remaining obstacle.

### Do

- [ ] Import the generated Python client in the backend and intelligence producers/consumers.
- [ ] Import the generated TS client in `realtime-gateway`'s `presenceEventsProducer.ts`.
- [ ] Delete the stale "package is absent" test.
- [ ] Two structural drifts worth fixing while you are here: intelligence writes
      `{topic}.{group}.dlq` but `create_topics.sh` only creates `{topic}.dlq`; and DLQ payloads are
      JSON while `contracts/DlqEnvelope.avsc` specifies Avro. Coordinate the topic-name half with
      Samar's M2/M8.

---

## H11 — LiveKit room names carry no org segment

**Repo:** `mytheus-backend`

`services/meeting_service/rooms.py:51-96` names rooms `meeting-{meeting_id}`. There is no tenant
segment in the name.

### Severity: low — defence in depth, not an open hole

Tenant resolution for LiveKit webhooks reads the org from **room metadata**, not the room name, and
that path fails closed six ways (`webhook.py:107-137`). Meeting IDs are UUIDs, so accidental
collision is not a realistic concern either. The room-name convention is simply carrying less
information than it could.

The reason to fix it is that it is the one identifier in the meeting path that is not
tenant-qualified, so any future code that reasonably parses a room name — a debugging tool, a
dashboard, an egress filename, a third-party integration — inherits an ambiguity it has no way to
detect.

### Do

- [ ] Add an org segment: `{org_id}-meeting-{meeting_id}` or similar.
- [ ] Coordinate with the metadata-sync path — the audit flagged that rooms "synced" via
      `FakeTransport` are marked `livekit_metadata_synced_at` **permanently**, so at real-LiveKit
      cutover they never re-sync and webhook tenant resolution breaks for every pre-existing room.
      A room-naming change and that cutover defect want handling together, in one migration of
      existing rows.
- [ ] Keep the `ON CONFLICT (org_id, meeting_id)` upsert semantics intact.

---

## Do NOT build these (deliberately deferred, or no data model exists)

- **`/decisions`, `/tasks`, `/search`, `/billing`, `/settings/integrations`** — do not wire these.
  **No backend exists for any of them.** There is no `decisions` or `action_items` table in any
  migration; `/search` has no BFF or gateway route; billing and integrations are 11-line scaffolds.
  Wiring a page to nothing is worse than an honest mock. Flag them for the scope conversation
  instead.
- **10 of 11 marketing feature pages** promise features with no backend (only transcription has
  one). That is a product/marketing decision, not a task here.
- **`/students/verify:48-51`** tells users "we sent a verification link" while making no network
  call at all. Worth raising — it is a truthfulness issue, not a wiring one.
- **`pii_tokens` / PII vault work** — ADR-0020 is blocked for implementation on key custody.
- **Knowledge graph UI** — no AGE, no graph store, no backend of any kind exists.

---

## Your checklist

- [ ] **H1** — Cross-tenant speaker assignment ⚠️ *security, do first*
- [ ] **H2** — Transcript `deleted_at` + partition bound + tests
- [ ] **H3** — Audit log writer and reader
- [ ] **H4** — Unbreak the transcript page's mock 404 gate
- [ ] **H5** — Wire `/meetings` to the real endpoint
- [ ] **H6** — Wire `/org` to the real endpoint
- [ ] **H7** — Fill the three empty dashboard arms
- [ ] **H8** — Use the two unused generated type modules
- [ ] **H9** — Refresh the realtime token on reconnect
- [ ] **H10** — Import the generated Avro clients
- [ ] **H11** — Org segment in LiveKit room names + the fake-sync cutover fix
