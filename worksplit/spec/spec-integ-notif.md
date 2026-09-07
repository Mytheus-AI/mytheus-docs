# Build spec — BAND 1 `integration-service`, BAND 2 `notification-service`

Read-only investigation, 2026-09-07. Repo root `/Users/sushan/Projects/Mytheus`.
Docs cited are **intent only**. Every status claim below is a code citation.

---

## 0. Verified reality (both bands)

### 0.1 BAND 1 claims — confirmed, with two corrections

| Claim | Verdict | Evidence |
|---|---|---|
| `integration-service` is 11 lines + a TODO | **Confirmed** | `mytheus-backend/services/integration_service/main.py:9` is `app = create_app("integration-service")`; `:11` is `# TODO: add routers for /oauth/{provider} and /webhooks/{provider}`. The whole service is 11 lines; `__init__.py` is 0 bytes. No `routes.py`, `models.py`, `connectors/`, `consumer.py`. |
| Zero OAuth/connector code in any of the 8 repos | **Confirmed** | `grep -rniE "googleapis|slack_sdk|slack/oauth|linear\.app|atlassian"` across all repos (excl. `.venv`, `node_modules`, worktrees) returns **only** enum string literals: `mytheus-events/clients/python/mytheus_events/integration_events.py:12`, `mytheus-events/clients/typescript/src/integrationEvents.ts:8`, `mytheus-ui/src/lib/mock/data.ts:480`, and the intelligence consumer/test. No HTTP client, no token exchange, no signature verification exists. `mytheus-backend/pyproject.toml` pulls no provider SDK. |
| Sole designated producer of `integration-events` | **Confirmed** | `mytheus-events/schemas/integration-events/*.avsc` doc string: *"Producer integration-service"*. `mytheus-backend/CLAUDE.md:38` lists `integration-events` as an `integration-service` produce. `mytheus-docs/plan/plan-10-13-collab-integrations-ga.md` §5 topic table: produced by `integration-service`, consumed by `ingestion-service`, `core-api`. |
| `mytheus-intelligence/services/ingestion_service/consumer.py:75` is a complete, tested consumer that can never fire | **Confirmed** | `:75` is `topics = ["integration-events"]` on `IntegrationEventConsumer(BaseConsumer)`. `apply()` (`:82`–`:120+`) is complete: R8 filter on `("google_calendar","attachment_to_ingest")` (`:56`, `:87`), cross-tenant `object_key` guard `_resolve_key` (`:64`), MinIO fetch, ingest, job lifecycle. `mytheus-intelligence/tests/test_ingestion_consumer.py` (`:108`, `:165`, `:206`, `:250`) exercises it against synthetic messages. The consumer's own docstring (`:26`) states: *"`integration-service` does not exist yet in this program (P12, a later plan band)"*. **No process anywhere produces to this topic**, so in production this consumer subscribes and idles forever. |
| Gateway routes `/oauth` and `/webhooks` to this service | **Confirmed** | `mytheus-backend/services/api_gateway/edge.py:178` `"/oauth": "integration-service"`, `:179` `"/webhooks": "integration-service"`. |
| **"both on the unauthenticated public allowlist"** | **FALSE — correction** | `edge.py:41`–`:55`: `PUBLIC_PREFIXES` contains `"/clerk/webhook"`, `"/stripe/webhook"`, `"/livekit/webhook"`, `"/oauth/"` (`:54`). **`/webhooks/` is absent.** Every provider webhook (Google push, Slack Events, Jira/Linear) will be rejected by the gateway's tenant/auth middleware before reaching `integration-service`. This is defect **D-I1** below and is a one-line fix, but it is load-bearing: the entire inbound half of Band 1 is dead until it lands. |
| Second correction — `/oauth/` is *fully* public | **Noted** | `edge.py:54` makes the whole `/oauth/` subtree unauthenticated. Per `21-integration-architecture.md` §2.3 only the **callback** is unauthenticated (it authenticates by HMAC `state`); connect-initiation is `/bff/integrations/{provider}/connect` and is RBAC-gated. The current allowlist is correct *only if* `integration-service` exposes nothing but the callback under `/oauth/`. Design accordingly — see T1.7. |

### 0.2 BAND 1 — schema drift you must resolve before writing a producer

`mytheus-docs/21-integration-architecture.md` §7 prints an `IntegrationEvent` with fields
`header, provider, kind (enum), connection_id, meeting_id, external_ref, body_json`.

The **registered** schema (`mytheus-events/schemas/integration-events/*.avsc`) is:

```
header, provider (enum google_calendar|slack|jira|linear), event_type (string),
external_id (string), payload_json (string)
```

The registered schema and the already-built consumer win. The producer must emit
`event_type` (a plain string, not an enum) and pack everything else into `payload_json`.
The **only** payload contract that exists today is the one the built consumer reads
(`mytheus-intelligence/services/ingestion_service/consumer.py:92`–`:100`):

```json
{"meeting_id": "<uuid>", "object_key": "<org_id>/<meeting_id>/<file>", "filename": "<name>"}
```

`object_key` **must** start with `f"{org_id}/"` or the consumer rejects it (`:64`–`:69`, tested).
Doc 21 §7 should be corrected to match; do not "fix" the schema to match the doc — that would
break the tested consumer and requires a BACKWARD-compat schema change.

### 0.3 BAND 2 claims — confirmed, with one addition

| Claim | Verdict | Evidence |
|---|---|---|
| Scaffold service | **Confirmed** | `mytheus-backend/services/notification_service/main.py` is 11 lines; `:11` `# TODO: add router for /notifications`. |
| Consumer subscribes and commits offsets after an empty `_handle`, destroying the backlog | **Confirmed** | `consumer.py:30`–`:32`: `async for msg in consumer: await _handle(msg); await consumer.commit()`. `_handle` (`:37`–`:41`) is `_ = msg  # placeholder`. `enable_auto_commit=False` (`:26`) makes the explicit commit the *only* offset advance — so every `notifications` and `agent-results` message is permanently consumed and discarded. Running this in any environment where producers exist is destructive. |
| No email/push/in-app delivery code anywhere | **Confirmed** | No SMTP/SES/Resend/Postmark/SendGrid/Expo dependency in `mytheus-backend/pyproject.toml`; no delivery module in any repo. |
| Mobile's push tier is specified against it and needs a net-new Expo token contract | **Confirmed** | `mytheus-mobile/system.md:169`–`:172` (§5.4), `:199`–`:202` (§8), `:300` (decision **N7**): *"Implies a small backend addition (device-token registration) … the one place the client touches a not-yet-existing contract."* |
| **Addition — `notifications` has zero producers** | Grep for `"notifications"` as a topic across `mytheus-backend`/`-intelligence`/`-realtime` returns only `notification_service/consumer.py:17` itself. `integration-service`, `billing-metering-service`, `workflow-service`, `identity-tenant-service` and the agent tier are all listed as producers in plan §5 and `CLAUDE.md:35,38,39,42`; none emits today. Band 2 therefore ships with **synthetic** producers until Band 1 or billing lands. |
| **Addition — `/api/v1/notifications` is unroutable** | `edge.py` `ROUTE_MAP` has `"/notifications": "notification-service"` (`:155`) but no `/api/v1/notifications`. Longest-prefix `route_for()` (`:184`) returns `None` for `/api/v1/notifications`, which is the path both `mytheus-mobile/system.md:150,151,169` and the web notification centre call. Defect **D-N1**. Pick one path and make gateway, service and both clients agree. |
| **Addition — no `push` channel in the Avro enum** | `schemas/notifications/*.avsc` `NotificationChannel` = `{in_app, email, slack}` with **no Avro `default`**. Adding a `push` symbol is *not* BACKWARD-compatible without first adding an enum default, which is itself a schema change. MVP must therefore treat push as a **delivery fan-out of `in_app`** driven by device-token rows, not as a new channel symbol. See T2.9. |
| **Addition — no `notification_log` / prefs / device-token tables** | `mytheus-backend/migrations/versions/82d44163c933_0001_*.py` creates `departments, inbox, meetings, organizations, outbox, teams, data_residency, tenant_settings, users, meeting_participants, team_members`. No notification tables in any of the 13 migrations. |
| **Addition — the Redis priority queue does not exist** | `grep -rn "queue:agent"` across all repos (excl. deps) returns **zero** code hits — only `mytheus-backend/CLAUDE.md:41` and doc prose. `realtime-gateway` has no priority-queue implementation to mirror. |
| **Addition — no `BaseConsumer` in `mytheus-backend`** | `class BaseConsumer` exists only at `mytheus-intelligence/libs/shared/consumer.py:70`. `mytheus-backend/libs/shared/` has `inbox.py` (`is_duplicate:22`, `record_consumed:33`), `outbox.py` (`enqueue:27`), `events.py` (`EventEnvelope:22`), `relay.py` — but no consumer base class. The in-repo recipe is the hand-rolled `services/core_api/consumer.py` (182 lines) / `services/meeting_service/consumer.py` (305 lines) pattern: decode → `uuid.UUID(header["org_id"])` strict parse → DLQ on parse failure → `SET app.current_org` → `is_duplicate`/`record_consumed` → apply → commit. Follow that; do not import across repos. |

---

# BAND 1 — `integration-service`

## 1. Data model

One migration, five tables, all `org_id`-scoped with `RLS FORCE` + the canonical
`tenant_isolation` policy (copy the pattern from `migrations/versions/a1c4e7f20b18_0002_*`
and `b7d9f3a55c21_0003_*`).

**`integration_connections`** — canon-frozen columns, do not extend:
`id uuid pk`, `org_id uuid NOT NULL REFERENCES organizations(id)`,
`provider text NOT NULL CHECK (provider IN ('google_calendar','slack','jira','linear'))`,
`external_account_id text`, `status text NOT NULL DEFAULT 'active' CHECK (status IN ('active','degraded','disconnected'))`,
`scopes text[] NOT NULL DEFAULT '{}'`, `installed_by uuid NOT NULL`,
`created_at/updated_at timestamptz`, `UNIQUE (org_id, provider)`.

The R8 one-PM invariant is **not** expressible by that unique constraint alone (it permits one
Jira *and* one Linear row). Enforce it as a partial unique index — the cheapest airtight form:

```sql
CREATE UNIQUE INDEX one_pm_connection_per_org
  ON integration_connections (org_id)
  WHERE provider IN ('jira','linear') AND status <> 'disconnected';
```

That makes "install Linear while Jira is active" a database-level 23505, not an application
race, and satisfies the plan's acceptance check (`plan-10-13` §3.10) structurally.

**`oauth_tokens`** — canon-frozen, exactly seven columns:
`id`, `org_id`, `connection_id uuid REFERENCES integration_connections(id) ON DELETE CASCADE`,
`access_token_enc bytea NOT NULL`, `refresh_token_enc bytea`, `expires_at timestamptz`,
`token_version int NOT NULL DEFAULT 1`.
The wrapped DEK and GCM nonce are packed **inside** the bytea as
`[wrapped_dek_len(4B BE) ‖ wrapped_dek ‖ nonce(12B) ‖ ciphertext ‖ tag(16B)]` — no extra columns
(`21-integration-architecture.md` §2.2 is explicit that the schema is not extended).

**`integration_links`** — the external↔NEXUS id map. `21` §2.1 notes canon prose calls this
`integration_links` while the frozen names are the two above; it is a service-private table, so
use `integration_links` as the physical name (it is what `CLAUDE.md`/`04-service-catalog.md:120`
say) and record the reconciliation in the migration docstring.
`(id, org_id, provider, kind CHECK (kind IN ('action_item','calendar_event')), internal_id uuid,
external_key text, external_url text, last_synced_status text, last_synced_at timestamptz)`,
`UNIQUE (org_id, provider, external_key)`, `UNIQUE (org_id, provider, kind, internal_id)`.
`last_synced_status` is the loop-suppression watermark (`21` §5.4).

**`webhook_cursors`** — `(org_id, provider, resource_id)` pk-ish, plus
`channel_id text, sync_token text, history_id text, expires_at timestamptz, last_seen_at`.
One row per watched calendar / per provider stream.

**`webhook_inbox`** — the fast-ACK landing table, distinct from `libs.shared.inbox` (which
dedupes *bus* events, not HTTP deliveries).
`(id, org_id nullable-until-resolved, provider, delivery_id text, received_at, raw_headers jsonb,
raw_body bytea, processed_at timestamptz, attempts int)`, `UNIQUE (provider, delivery_id)` —
that unique index *is* the replay guard.

## 2. OAuth flow per provider

Shared engine (one `ProviderAdapter` protocol; three implementations):

- Authorization-code + **PKCE**, server-mediated. `state` = HMAC-SHA256 over
  `(org_id, user_id, nonce, exp=now+10m)` keyed by a secret from the same source as
  `GATEWAY_SIGNING_KEY` (`edge.py:65`); PKCE verifier stored in Redis at
  `nexus:{org_id}:oauth:{state}`, TTL 600s, popped once at callback.
- Initiation `GET /bff/integrations/{provider}/connect` → RBAC `owner|admin` at `bff`, **re-checked**
  in `integration-service` — do not rely on the gateway, whose `/oauth/` prefix is unauthenticated.
- Callback `GET /oauth/{provider}?code&state` → verify HMAC + exp → pop verifier → exchange →
  envelope-encrypt → upsert `integration_connections` + `oauth_tokens` → 302 to `/settings/integrations`.

Per-provider specifics:

| Provider | Authorize / token endpoints | Scopes (minimum) | Account id | Notes |
|---|---|---|---|---|
| `google_calendar` | `accounts.google.com/o/oauth2/v2/auth` → `oauth2.googleapis.com/token` | `calendar.readonly`, `calendar.events.readonly`, `drive.readonly` (attachments), `openid email` | `sub` / primary calendar id | `access_type=offline&prompt=consent` is **required** to get a refresh token; without it re-auth is manual forever. |
| `slack` | `slack.com/oauth/v2/authorize` → `oauth.v2.access` | bot: `chat:write`, `chat:write.public`, `im:write`, `users:read`, `users:read.email`, `channels:read` | `team.id` | Bot token, not user token. Slack returns a non-expiring bot token unless token rotation is enabled — if enabled, `expires_at` + refresh apply. |
| `jira` (default) | `auth.atlassian.com/authorize?audience=api.atlassian.com` → `auth.atlassian.com/oauth/token` | `read:jira-work`, `write:jira-work`, `read:jira-user`, `offline_access` | `cloudid` from `/oauth/token/accessible-resources` | `offline_access` is mandatory for refresh. All API calls go through `api.atlassian.com/ex/jira/{cloudid}`. |
| `linear` (alt) | `linear.app/oauth/authorize` → `api.linear.app/oauth/token` | `read`, `write`, `issues:create` | workspace id | GraphQL only; tokens are long-lived, no refresh flow. |

## 3. Token refresh and revocation

- **Vault module** `libs/shared/token_vault.py` (or service-local): `seal(org_id, plaintext) -> bytes`,
  `open(org_id, envelope) -> str`. AES-256-GCM via `cryptography` (already a dependency,
  `pyproject.toml:22`). **There is no KMS in this stack today** — see external decision **X-I1**.
- **Refresh**: a background task (or a `workflow-service` cron child) refreshes any token with
  `expires_at < now + 10m`. On success `token_version = token_version + 1` in the same tx.
  On the 3rd consecutive failure: `status = 'degraded'` + one `notifications` event
  (`title = "Reconnect {provider}"`, `priority` high, `link` → `/settings/integrations`).
  Refresh is `SELECT ... FOR UPDATE` on the `oauth_tokens` row so two pods cannot both burn the
  single-use refresh token (Google and Atlassian both rotate refresh tokens; a lost race
  disconnects the tenant).
- **Revocation / disconnect**: `DELETE /bff/integrations/{provider}` → best-effort provider
  revoke (Google `oauth2/revoke`, Slack `auth.revoke`, Atlassian token invalidate) → stop the
  Google push channel (`channels.stop`) / delete the Jira webhook → `status='disconnected'` →
  delete `oauth_tokens` rows → keep `integration_links` (history) but stop syncing.
- **Logging**: add a redaction filter to `libs/shared/telemetry.py` for field names matching
  `*token*`, `*secret*`, `code`, `client_secret`, `refresh*`. Assert it in a test — this is the
  single cheapest control and the easiest to lose in a later refactor.
- **No API returns token material.** `GET /bff/integrations` returns
  `provider, status, scopes, external_account_id, installed_by, last_synced_at` only.

## 4. Webhook receivers

`POST /webhooks/{provider}` — one handler, four steps, in this order:

1. **Signature verify over the raw body** (read `await request.body()` before any JSON parse):
   Slack `v0=HMAC(signing_secret, "v0:{ts}:{body}")` + `|now-ts| < 300s`;
   Jira/Linear shared-secret HMAC (`X-Hub-Signature`/`Linear-Signature`);
   Google push carries no HMAC — verify the per-channel token you set at `events.watch`
   (`X-Goog-Channel-Token`) against `webhook_cursors`, and resolve `org_id` from
   `X-Goog-Channel-ID`. **Google is the reason `org_id` on `webhook_inbox` is nullable until
   resolved**: the tenant is derived from the channel, not from a header.
2. **Replay guard**: `INSERT INTO webhook_inbox ... ON CONFLICT (provider, delivery_id) DO NOTHING`.
   Zero rows affected → already seen → return 200 immediately.
3. **Fast ACK**: return `200` right after the insert. Slack's deadline is 3 s, Google's ~10 s.
   No provider call, no bus produce, no DB work beyond the insert on this path.
4. **Async drain**: a worker polls unprocessed `webhook_inbox` rows (`FOR UPDATE SKIP LOCKED`,
   same shape as `libs/shared/relay.py:39`), applies the connector handler idempotently,
   advances `webhook_cursors`, sets `processed_at`. Retry budget then a DLQ column/topic.

Bus produces all go through the existing transactional outbox (`libs.shared.outbox.enqueue`)
in the same transaction as the state change — never a direct producer call.

## 5. Calendar attachment path — **do this first**

This is the highest value-per-unit-of-work item in either band, because the consumer side is
already built, tested and idle. Producing one correctly-shaped `integration-events` message
turns `mytheus-intelligence/services/ingestion_service/consumer.py` from dead code into a live
end-to-end path (calendar attachment → MinIO → unstructured.io → `document_chunks` →
embeddings → per-tenant Qdrant), with **zero** new code in the intelligence repo.

Sequence: Google push → `events.list(syncToken)` delta → for each `attachments[]`, download bytes
with the Drive scope → `PUT` to `nexus-attachments` under key `{org_id}/{meeting_id}/{filename}`
(the prefix is mandatory: `consumer.py:64` rejects anything else) → outbox-enqueue
`integration-events` with `provider="google_calendar"`, `event_type="attachment_to_ingest"`,
`external_id=<drive file id>`, `payload_json={"meeting_id","object_key","filename"}`.

Note the ordering dependency: `meeting_id` must exist before the attachment event, so the
`calendar_event_upserted` → `core-api` → `meetings` upsert lands first. If `core-api` has no
`integration-events` consumer yet (it does not — `services/core_api/consumer.py:2` consumes
`presence-events` only), the attachment path can still be demoed against a manually-created
meeting; the calendar→meeting arm is T1.9/T1.10.

## 6. Bidirectional action sync (Jira default)

- **Outbound**: consume `actions` (from `core-api`'s outbox, on confirm) → id-map lookup by
  `(org_id,'jira','action_item',action_item_id)` → if absent, provider-side search on label
  `nexus:{action_item_id}` → if still absent, `POST /rest/api/3/issue` with the §5.2 field map →
  write `integration_links` row. Carries `header.idempotency_key`; the id-map row is the real
  dedupe.
- **Inbound**: webhook → verify/dedupe (§4) → id-map lookup by `external_key` → compare mapped
  status to `last_synced_status`; **equal ⇒ drop (echo suppression)**; different ⇒ outbox-enqueue
  `actions {status sync}` and set `last_synced_status` in the *same* transaction.
- Status map (`21` §5.3): `open ↔ To Do`, `in_progress ↔ In Progress`, `done ↔ Done`,
  `cancelled ↔ Won't Do`. Unmapped external state ⇒ `open` + a low-priority
  "review status mapping" `notifications`.
- **Blocker**: `action_items` has no table and no model — `services/core_api/models.py` declares
  only `meetings` (`:28`) and `meeting_participants` (`:58`). The whole outbound arm depends on
  `core-api` owning `action_items` and producing `actions`. Treat as an external dependency on
  the core-api band, not as work in this band.

## 7. `/settings/integrations` UI

Today `mytheus-ui/src/app/(app)/settings/integrations/page.tsx` is 93 lines of `useState` over
`mockIntegrations` (`src/lib/mock/data.ts:487`); the toggle at `:22`–`:30` fakes the one-PM rule
client-side, and the file comment at `:10`–`:13` says so. What it needs:

1. Replace `mockIntegrations` with `GET /bff/integrations` → `{provider, status, scopes,
   external_account_id, installed_by, last_synced_at}[]`. Keep the exact `IntegrationConnection`
   shape (`data.ts:479`) so the components do not change.
2. `Connect` becomes a full-page navigation to `/bff/integrations/{provider}/connect` (a 302 to
   the provider) — **not** `fetch`, or the consent redirect will be swallowed by CORS.
3. Return from the callback lands on `/settings/integrations?connected={provider}` — show a
   toast, invalidate the query.
4. PM section renders as a radio; the unselected option's Connect is `disabled` when the other is
   `active`, and the server 409s regardless (defence in depth, matching the partial unique index).
5. `status='degraded'` renders a distinct badge + "Reconnect" CTA — this is the user-visible half
   of the refresh-failure path.
6. Page is RBAC-gated to `owner|admin`; a `member` sees read-only state.
7. Calendar sub-section: watched calendars + unmatched-attendee mapping (later band).

## 8. Task list — BAND 1

MVP-band = the minimum for the P12 headline demo (live calendar attachment + live Jira issue).

| # | Title | Repo + files | Size | Depends on | External decision? | Band |
|---|---|---|---|---|---|---|
| **T1.1** | **D-I1: add `/webhooks/` to `PUBLIC_PREFIXES`** | `mytheus-backend/services/api_gateway/edge.py:41-55` + a test asserting a `/webhooks/slack` POST is not auth-rejected | **S** | none | no | MVP |
| T1.2 | Migration: 5 tables + RLS + the `one_pm_connection_per_org` partial unique index | `mytheus-backend/migrations/versions/*_integration_tables.py`, `services/integration_service/models.py` | **M** | T1.1 (no), independent | no | MVP |
| T1.3 | Token vault (`seal`/`open`, AES-256-GCM, packed envelope) + log-redaction filter + tests | `mytheus-backend/libs/shared/token_vault.py`, `libs/shared/telemetry.py` | **M** | T1.2 | **YES — X-I1** (KEK source) | MVP |
| T1.4 | OAuth engine: HMAC `state`, Redis PKCE store, `ProviderAdapter` protocol, callback route | `services/integration_service/{oauth.py,routes.py}` | **M** | T1.2, T1.3 | no | MVP |
| T1.5 | `bff` connect/disconnect/list endpoints, RBAC `owner|admin`, re-checked in-service | `services/bff/`, `services/integration_service/routes.py` | **M** | T1.4 | no | MVP |
| T1.6 | Webhook receiver: raw-body signature verify (4 providers), `webhook_inbox` replay guard, fast-ACK, `SKIP LOCKED` drain worker | `services/integration_service/{webhooks.py,worker.py}` | **L — 3d** | T1.1, T1.2 | no | MVP |
| T1.7 | Audit the `/oauth/` public subtree: assert only the callback is reachable unauthenticated | `services/integration_service/routes.py` + gateway test | **S** | T1.4 | no | MVP |
| **T1.8** | **Google Calendar attachment → `integration-events` (unblocks the built consumer)** | `services/integration_service/connectors/google_calendar.py`, `libs/shared/object_storage`-equivalent for `nexus-attachments` | **M** | T1.3, T1.4, T1.6 | **YES — X-I2** (Google Cloud project + OAuth client + verified domain) | MVP |
| T1.9 | Calendar `events.watch` channel registration + `syncToken` incremental sync + cursor renewal | same connector + `webhook_cursors` | **L — 3d** | T1.8 | X-I2 | MVP |
| T1.10 | `calendar_event_upserted` producer + email→`user_id` resolution via `identity-tenant-service` | connector + an identity client | **M** | T1.9 | no | MVP (needs a `core-api` consumer to land elsewhere) |
| T1.11 | Correct `21-integration-architecture.md` §7 to the registered Avro schema | `mytheus-docs/21-integration-architecture.md` | **S** | none | no | MVP (do with T1.8) |
| T1.12 | Slack connector: bot OAuth, workspace/channel/user id maps, Block Kit **builders** only | `connectors/slack.py` | **M** | T1.4 | **YES — X-I3** (Slack app + signing secret + distribution) | MVP |
| T1.13 | Prep-brief + post-meeting-summary builders → `notifications` producer | `connectors/slack.py`, outbox | **M** | T1.12, Band 2 T2.3 | no | Later (needs `rag-service` + `MeetingLifecycleWorkflow` timers) |
| T1.14 | Jira connector outbound: consume `actions`, create issue, write id map | `connectors/jira.py`, `consumer.py` | **L — 3d** | T1.4, `core-api` `action_items` + `actions` producer | **YES — X-I4** (Jira vs Linear = DR-2) | MVP |
| T1.15 | Jira inbound: status webhook → id-map → loop suppression → `actions` produce | `connectors/jira.py`, `webhooks.py` | **M** | T1.14, T1.6 | X-I4 | MVP |
| T1.16 | Token refresh loop + `degraded` transition + re-auth `notifications` | `services/integration_service/refresh.py` | **M** | T1.3, T1.4 | no | MVP |
| T1.17 | `/settings/integrations` wired to real `bff` data (items 1–6 of §7) | `mytheus-ui/src/app/(app)/settings/integrations/page.tsx`, `src/lib/api/` | **M** | T1.5 | no | MVP |
| T1.18 | Unmatched-attendee mapping UI + watched-calendar list | `mytheus-ui` settings | **M** | T1.10, T1.17 | no | Later |
| T1.19 | Linear connector (only if DR-2 picks Linear) | `connectors/linear.py` | **L — 3d** | T1.4 | X-I4 | Later / alternative to T1.14–15 |

**Band 1: 19 tasks. MVP band = T1.1–T1.12, T1.14–T1.17 (16 tasks). Effort ≈ 34–38 person-days**
(4×S≈2d, 10×M≈20d, 4×L≈12d for the MVP set; T1.13/T1.18/T1.19 add ≈8d later).

**External decisions blocking Band 1** (not engineering):
- **X-I1** — KEK source. There is no KMS in this stack (`grep` for KMS/AESGCM across
  `mytheus-backend/libs`+`services` finds only prose; deployment is a single-node VPS).
  Choose: (a) a real cloud KMS (adds a provider + cost + a network dependency on every token
  read), or (b) a per-tenant DEK wrapped by a single KEK from the environment/secret store,
  with the envelope format unchanged so (a) is a drop-in later. **(b) is the right MVP call**;
  it keeps the `[wrapped_dek_len ‖ …]` layout and the RTBF story (delete the tenant DEK).
  Blocks T1.3 and therefore everything.
- **X-I2** — Google Cloud project, OAuth client (web), verified domain + a published consent
  screen. Calendar+Drive **read** scopes are sensitive; unverified apps cap at 100 test users.
  Start the verification early — it is calendar-time, not engineering-time. Blocks T1.8–T1.10.
- **X-I3** — Slack app creation, signing secret, bot scopes, distribution decision. Blocks T1.12.
- **X-I4** — **DR-2: Jira or Linear.** Documented as decide-before-P12
  (`plan-10-13-collab-integrations-ga.md` §8). Blocks T1.14/T1.15 vs T1.19.

**Dependency Band 1 does not own**: `core-api` must own `action_items` and produce `actions`
(neither exists), and must consume `integration-events` for `calendar_event_upserted`.

---

# BAND 2 — `notification-service`

## 9. Data model

**`notification_log`** — the durable read side behind `GET /notifications`:
`id uuid pk`, `org_id uuid NOT NULL`, `user_id uuid NOT NULL`,
`source_event_id text NOT NULL`, `channel text NOT NULL`, `priority int NOT NULL`,
`title text`, `body text`, `link text`,
`created_at timestamptz`, `read_at timestamptz`, `dismissed_at timestamptz`,
`UNIQUE (org_id, source_event_id, channel, user_id)`.
That unique key **is** the idempotency key from `plan-10-13` §3.8 step 1
(`notif:{source_event_id}:{channel}:{user_id}`) — make it a constraint, not a Redis-only check,
so a Redis flush cannot cause a double send. RLS FORCE + `tenant_isolation`.
Index `(org_id, user_id, created_at DESC) WHERE dismissed_at IS NULL` for the cursor-paginated
notification centre.

**`notification_deliveries`** — one row per attempt, the receipt ledger:
`id`, `org_id`, `notification_id REFERENCES notification_log(id)`, `channel text`,
`status text CHECK (status IN ('pending','sent','failed','bounced'))`, `detail text`,
`attempts int`, `next_attempt_at timestamptz`, `provider_message_id text`, `created_at/updated_at`.
`status` deliberately mirrors the Avro `NotificationDeliveryStatus` enum
(`sent|failed|bounced`) plus a local `pending`.

**`notification_preferences`** — per user per channel:
`(org_id, user_id, channel)` pk, `enabled bool NOT NULL DEFAULT true`,
`min_priority int NOT NULL DEFAULT 0`, `quiet_hours_start/end time`, `timezone text`.
Absent row ⇒ default-on for `in_app`, default-on for `email`, default-off for `slack`.

**`device_push_tokens`** — the net-new N7 contract:
`id`, `org_id`, `user_id`, `expo_push_token text NOT NULL`, `platform text CHECK (platform IN ('ios','android'))`,
`device_id text`, `created_at`, `last_seen_at`, `revoked_at`,
`UNIQUE (org_id, user_id, expo_push_token)`.

## 10. The three channels

- **In-app**: write `notification_log`, then publish to the `realtime-gateway` Redis backbone so
  a connected client gets it live. Priority ordering lands in
  `nexus:{org_id}:queue:agent:{user_id}` — a Redis **sorted set** scored
  `priority * 10^12 + occurred_at_micros` so lower priority number sorts first and ties break
  by time. **Nothing implements this today** (zero `queue:agent` hits repo-wide), and
  `realtime-gateway` has no queue to "mirror" — Band 2 defines the structure and
  `realtime-gateway` consumes it. Coordinate the key shape before either side writes code.
- **Email**: a provider adapter behind one interface (`send(to, subject, html, text) ->
  provider_message_id`). No email dependency exists in `pyproject.toml`; see **X-N1**.
  Templates: Jinja2 (already transitively present via FastAPI's ecosystem — verify) with a
  plain-text alternative per template. Per-org from-address and a working unsubscribe/prefs link
  are launch requirements, not polish.
- **Slack**: `chat.postMessage`. `notification-service` **must not** read `oauth_tokens`.
  Per `21` §4.1 and `plan-10-13` §3.8 it calls a narrow internal `integration-service` endpoint
  that returns a short-lived, scope-checked handle. Concretely:
  `POST /internal/integrations/slack/token` `{org_id}` → `{token, expires_at}` with a ≤60 s TTL,
  gateway-signed service-to-service (reuse `edge.py:sign_context`'s HMAC discipline), never
  persisted, never logged. **This makes Band 2's Slack channel depend on Band 1 T1.12.**
  Until then the Slack channel is implemented and unit-tested against a fake handle provider.

## 11. Delivery-receipt loop

Every attempt writes `notification_deliveries` **and** outbox-enqueues a
`notification-delivery-events` message in the same transaction:
`{notification_id, channel, status: sent|failed|bounced, detail}` — exactly the registered
schema's four payload fields. Consumers are `admin-ops-service` and `core-api` (plan §5).
Bounces: email-provider bounce webhooks and Expo `DeviceNotRegistered` receipts both land as
`bounced` and **revoke** the offending address/token (`device_push_tokens.revoked_at`), which is
the only thing that keeps a push sender from being throttled by Expo.

## 12. Idempotency and retry

- **The first fix is to stop destroying the backlog.** `consumer.py:30`–`:32` must not commit
  until `_handle` has durably recorded the message. Rewrite the loop in the in-repo shape
  (`services/core_api/consumer.py`): decode → strict `uuid.UUID(header["org_id"])` (DLQ on
  failure) → `SET LOCAL app.current_org` → `is_duplicate`/`record_consumed`
  (`libs/shared/inbox.py:22,33`) → apply → **then** commit the offset.
- Dedupe key `notif:{source_event_id}:{channel}:{user_id}` in Redis **and** as the
  `notification_log` unique constraint; the DB constraint is the authority.
- Retry: exponential backoff on `next_attempt_at` (1m/5m/30m/2h/6h), max 5 attempts, then
  `failed` + a receipt. A drain worker picks up due rows `FOR UPDATE SKIP LOCKED`.
- Provider 429 (Slack `Retry-After`, Expo rate limits) sets `next_attempt_at` from the header
  rather than the backoff curve.
- Poison messages → `notifications.dlq` / `agent-results.dlq` after the retry budget.

## 13. Expo push token contract (N7)

The one net-new client-facing contract. Modeled as a channel-preference write so
`notification-service` stays the single delivery owner (`mytheus-mobile/system.md:300`).

```
POST   /api/v1/notifications/devices     { expo_push_token, platform, device_id }  -> 201 {id}
DELETE /api/v1/notifications/devices/{id}                                          -> 204
GET    /api/v1/notifications?cursor=&limit=            (cursor-paginated, unread-first)
PATCH  /api/v1/notifications/{id}        { read: true } | { dismissed: true }      -> 200
```

`org_id`/`user_id` come from the gateway-injected `x-mytheus-org`/`x-mytheus-user` headers
(`edge.py:34,35`) — **never** from the body; the mobile client never sets `org_id`
(`system.md` acceptance check). Registration is idempotent on
`(org_id, user_id, expo_push_token)`; re-registering refreshes `last_seen_at` and clears
`revoked_at`. Sign-out calls DELETE.

Delivery: send to `https://exp.host/--/api/v2/push/send`, batched ≤100 tickets, then poll
`/push/getReceipts` after ~15 min; a `DeviceNotRegistered` receipt revokes the token.
Payload carries `{title, body, data:{link}}` where `link` is one of `mytheus://tasks`,
`mytheus://meetings/{id}/room`, `mytheus://notifications` (`system.md:201`).

**Schema constraint (see §0.3):** `NotificationChannel` is `{in_app, email, slack}` with no Avro
default, so a `push` symbol is not a BACKWARD-compatible addition. **MVP treats push as a
fan-out of `in_app`**: an `in_app` notification for a user with active `device_push_tokens` and
`preferences.push.enabled` also sends an Expo push. If push must become a first-class channel
later, that is a two-step schema change (add an enum `default`, then the symbol) and belongs in
`mytheus-events` — flag it, do not sneak it in.

## 14. Task list — BAND 2

| # | Title | Repo + files | Size | Depends on | External decision? | Band |
|---|---|---|---|---|---|---|
| **T2.1** | **Stop the offset-destroying loop** — rewrite `run_consumer`/`_handle` to the `core_api/consumer.py` recipe (org re-assert → inbox dedupe → apply → commit) | `mytheus-backend/services/notification_service/consumer.py:21-45` | **M** | none | no | MVP |
| T2.2 | Migration: `notification_log`, `notification_deliveries`, `notification_preferences`, `device_push_tokens` + RLS + indexes | `migrations/versions/*_notification_tables.py`, `services/notification_service/models.py` | **M** | none | no | MVP |
| T2.3 | In-app channel: persist + publish to the realtime backbone; define + write `nexus:{org_id}:queue:agent:{user_id}` (sorted set) | `services/notification_service/{delivery/in_app.py,queue.py}` | **M** | T2.1, T2.2 | no | MVP |
| **T2.4** | **D-N1: make `/api/v1/notifications` routable** — add to `ROUTE_MAP` (or change the service+clients to `/notifications`); pick one and align mobile + web | `services/api_gateway/edge.py:146-180` | **S** | none | no | MVP |
| T2.5 | REST surface: `GET /api/v1/notifications` (cursor), `PATCH /{id}` read/dismiss | `services/notification_service/routes.py` | **M** | T2.2, T2.4 | no | MVP |
| T2.6 | Channel-preference model + enforcement (enabled / min_priority / quiet hours) + prefs REST | `services/notification_service/{prefs.py,routes.py}` | **M** | T2.2 | no | MVP |
| T2.7 | Delivery-receipt loop: `notification_deliveries` + outbox-enqueue `notification-delivery-events` | `services/notification_service/delivery/receipts.py`, `libs/shared/outbox.py` | **M** | T2.2, T2.3 | no | MVP |
| T2.8 | Retry/backoff worker (`SKIP LOCKED`, `next_attempt_at`, 429 `Retry-After`) + DLQ path | `services/notification_service/worker.py` | **M** | T2.7 | no | MVP |
| T2.9 | Expo push: `device_push_tokens` REST, batched send, receipt polling, `DeviceNotRegistered` revoke, fan-out-of-`in_app` semantics | `services/notification_service/delivery/push.py`, `routes.py` | **L — 3d** | T2.2, T2.3, T2.7 | **YES — X-N2** (Apple dev account + APNs key; FCM project) | MVP |
| T2.10 | Mobile: register/deregister the token, permission prompt, tap→deep-link routing, foreground reconcile | `mytheus-mobile/` (`expo-notifications`, `app/_layout.tsx`, `(app)/_layout.tsx`) | **M** | T2.9, T2.4 | X-N2 | MVP |
| T2.11 | Email channel: provider adapter + Jinja2 templates (HTML + text) + per-org from-address + unsubscribe link | `services/notification_service/delivery/email.py`, `templates/` | **L — 3d** | T2.2, T2.6, T2.7 | **YES — X-N1** (email provider + sending domain + SPF/DKIM/DMARC) | MVP |
| T2.12 | Email bounce/complaint webhook → `bounced` receipt + address suppression | `services/notification_service/webhooks.py`, gateway allowlist entry | **M** | T2.11, T2.4 | X-N1 | MVP |
| T2.13 | Slack channel: `chat.postMessage` against a short-lived scoped handle; never persist the token | `services/notification_service/delivery/slack.py` | **M** | T2.7, **Band 1 T1.12** | X-I3 | MVP |
| T2.14 | `integration-service` internal short-lived-token endpoint (`POST /internal/integrations/slack/token`, ≤60 s TTL, service-signed) | `mytheus-backend/services/integration_service/internal.py` | **M** | Band 1 T1.3, T1.12 | no | MVP (Band 1 work, listed here because Band 2 blocks on it) |
| T2.15 | `agent-results` handling: priority alerts pre-empt routine notifications in the queue | `services/notification_service/consumer.py` | **M** | T2.1, T2.3 | no | MVP |
| T2.16 | Synthetic producer + integration test harness (no real producers exist today) | `mytheus-backend/tests/test_notification_consumer.py` | **M** | T2.1 | no | MVP |
| T2.17 | Web notification centre wired to the real REST surface | `mytheus-ui/src/app/(app)/` (no notifications route exists today) | **M** | T2.5 | no | Later |
| T2.18 | `realtime-gateway` reads the priority queue and fans in-app deltas in priority order | `mytheus-realtime/` | **M** | T2.3 | no | Later |
| T2.19 | If push must become a first-class Avro channel: enum `default` then `push` symbol, two BACKWARD-safe steps | `mytheus-events/schemas/notifications/` | **S** | T2.9 | no | Later |

**Band 2: 19 tasks. MVP band = T2.1–T2.16 (16 tasks). Effort ≈ 32–36 person-days**
(1×S≈0.5d, 13×M≈26d, 2×L≈6d for the MVP set; T2.17–T2.19 add ≈5d later).

**External decisions blocking Band 2** (not engineering):
- **X-N1** — email provider (Resend / Postmark / SES) + a sending domain with SPF, DKIM and
  DMARC. Domain warm-up and DNS are calendar-time; start before the code. Blocks T2.11, T2.12.
- **X-N2** — Apple Developer account + APNs auth key, and an FCM project, both registered with
  EAS. Expo push cannot be tested end-to-end without them. Blocks T2.9, T2.10.
- **X-I3** (shared with Band 1) — the Slack app; blocks T2.13.

---

## 15. Ordering advice

1. **T1.1 and T2.4 first** — two small gateway edits that unblock the entire inbound half of
   Band 1 and the entire client half of Band 2. They are S-sized and have no external blockers.
2. **T2.1 immediately after** — the current notification consumer is actively destructive the
   moment a producer exists. Do not let Band 1 produce `notifications` before this lands.
3. **T1.2 → T1.3 → T1.4 → T1.6 → T1.8** is the critical path to the highest-leverage
   demo in the program: a real calendar attachment becoming searchable through the already-built,
   already-tested `ingestion-service` consumer, with zero new intelligence-repo code.
4. Start **X-I2** (Google verification) and **X-N1** (sending-domain DNS) on day one; both are
   waiting-time, not working-time, and both sit on the critical path of their band.
