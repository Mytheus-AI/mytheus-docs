# Linear Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** An admin connects Linear as the workspace's project tracker, and Linear then does everything Jira does in Mytheus.

**Architecture:** A `linear` connector in `integration-service` mirrors `connectors/jira.py` (OAuth, outbound issue creation from the `actions` consumer, inbound status by verified webhook, re-read before acting). A small `tracker.py` module answers "which tracker does this tenant use" so the internal reads (issue status, watch sources) and their core-api and workflow consumers work for Jira or Linear. The UI enables the Linear segment on the Project tracker card.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy async, httpx, pytest (`mytheus-backend`); Next.js 14, TypeScript, node test runner, Playwright CT (`mytheus-ui`); bash (`mytheus-platform`); Markdown (`mytheus-docs`).

**Spec:** `docs/superpowers/specs/2026-11-12-linear-and-zoom-integrations-design.md` (sections 4, 5, 7, 8, 9). Zoom (spec section 6) is a separate plan written after this one lands.

## Global Constraints

- Every repo pushes straight to `main`; no branches, no PRs. Commit after every task (one small conventional commit per task, more if a task has natural sub-steps). Push only when the owner says so.
- `mytheus-ui` and `mytheus-docs` commits carry no `Co-Authored-By` footer.
- Docs change before code: Task 1 lands first.
- One tracker per workspace: Jira and Linear are never both connected (existing `PM_PROVIDERS` 409).
- Secrets: `LINEAR_CLIENT_ID`, `LINEAR_CLIENT_SECRET`, `LINEAR_WEBHOOK_SECRET`. All three, or Linear is "not configured".
- Linear status map: workflow state type `completed` -> `done`; every other type (including `canceled`) -> `confirmed`.
- Inbound webhooks are verified on the raw body before parsing; `webhookTimestamp` must be within 60 seconds; delivery id = `Linear-Delivery` header.
- Only issues in this tenant's `integration_id_map` (provider `linear`, kind `action`) change a Mytheus action. The status is re-read from Linear with the tenant's token, never taken from the webhook body.
- No token or provider secret is ever logged (use `redaction.get_logger`).
- Wire names stay stable: the `jira_keys` request field, the `/internal/jira/issue-status` path and the bets `source = "jira"` value keep their names and now mean "the tenant's tracker". New fields are additive.
- Docs conventions: plain headers, no emojis, no em or en dashes, frozen identifiers in code spans.

## Review Focus

1. A Linear webhook for an issue this tenant never created (or naming another tenant's `organizationId`) must change nothing. Test in Task 3 and Task 4.
2. A replayed delivery, or one older than 60 seconds, must be a no-op (401 for stale, 200 no-op for replay). Test in Task 4.
3. An expired Linear access token must refresh and carry on; a revoked one marks the connection errored and never surfaces a 5xx to workflow callers. Test in Task 2 (refresh) and Task 5 (internal reads stay 200).
4. An assignee whose email Linear does not know: the issue is still created, unassigned. Test in Task 3.
5. Switching trackers: with Jira connected, connecting Linear answers 409; after disconnecting Jira and connecting Linear, a confirmed action creates exactly one Linear issue and no Jira call. Test in Task 4.

---

### Task 1: ADR-0044 and the canon note for Linear

**Files (mytheus-docs):**
- Create: `adr/0044-linear-and-zoom-integrations.md`
- Modify: `_canon/06-tech-decisions.md` (section 2.13, the R8 row), `_canon/02-data-model.md` (`integration_connections` row: note Linear is built), `21-integration-architecture.md` (section 5 Linear column becomes built; section 9 R8 cites ADR-0044), `24-glossary-and-naming.md` (Linear entry), `coverage-traceability-matrix.md` (R8 row), `README.md` (ADR table)

**Interfaces:** Produces ADR number `0044`, cited by every later commit message that touches Linear.

- [ ] **Step 1:** Write ADR-0044 in the existing ADR format (Context, Options, Decision, Consequences). Decisions: D1 Linear is built as R8's alternative PM tool (no relaxation needed); D2 Zoom is added as a meeting-source integration outside R8, beside the Phase 4 reach providers (ADR-0037 D3), with a `[SaaS-scale adjustment]` callout; D3 the Linear status map and the "re-read, id map only" inbound rule; D4 Zoom's host-is-a-member trust rule, no media kept, no backfill, Zoom's recording notice as consent (spec 6.3 to 6.6). Status: Accepted, 2026-11-12.
- [ ] **Step 2:** Edit the canon and reader docs listed above so each cites ADR-0044. Do not add `zoom` to the provider enum here; the Zoom plan does that with its migration.
- [ ] **Step 3:** Check: `grep -n "0044" _canon/06-tech-decisions.md 21-integration-architecture.md README.md` shows a hit in each; `grep -nP "[\x{2013}\x{2014}]" adr/0044-linear-and-zoom-integrations.md` (or a Python one-liner) finds no dashes.
- [ ] **Step 4:** Commit: `docs(adr): ADR-0044 Linear and Zoom integrations; R8 cites it`

### Task 2: Linear OAuth and API clients

**Files (mytheus-backend):**
- Modify: `services/integration_service/config.py` (add `LINEAR` to `_CREDENTIAL_ENV` with `LINEAR_CLIENT_ID`/`LINEAR_CLIENT_SECRET`; `credentials()` also requires `LINEAR_WEBHOOK_SECRET`, like Slack; add `linear_webhook_secret() -> bytes | None`; `R8_PROVIDERS = (GOOGLE_CALENDAR, SLACK, JIRA, LINEAR)`; update the module docstring)
- Modify: `services/integration_service/clients.py` (new `LinearOAuth`, `LinearApi`; `Clients.linear: LinearApi = field(default_factory=LinearApi)`; `default_clients()` registers `config.LINEAR: LinearOAuth()`)
- Modify: `tests/_integration_fakes.py` (`FakeLinear`, wired into `fake_clients()`; `install_env()` sets the three Linear vars)
- Test: `tests/test_linear_clients.py`

**Interfaces:**
- Produces `LinearOAuth(_Http)`: `provider = config.LINEAR`; `scopes = ("read", "write", "issues:create")`; `authorize_url(*, state, code_challenge, redirect_uri) -> str` to `https://linear.app/oauth/authorize` with `response_type=code`, `scope` comma-joined, `actor=app`, `prompt=consent`, PKCE S256; `exchange(...) -> TokenGrant` posts to `https://api.linear.app/oauth/token`, then queries `organization { id urlKey }` and returns `external_account_id = organization.id`, `extra = {"url_key": urlKey}`; `refresh(refresh_token) -> TokenGrant`; `revoke(token) -> None` posts to `https://api.linear.app/oauth/revoke`.
- Produces `LinearApi(_Http)`, all against `https://api.linear.app/graphql` with `Authorization: Bearer {token}`, raising `ProviderError(config.LINEAR, op, reason, status)` on GraphQL `errors` or a non-2xx:
  - `first_team(token) -> dict[str, str] | None` returns `{"id", "key"}`
  - `user_id_for_email(token, email) -> str | None`
  - `create_issue(token, *, team_id, title, description, due_date: str | None, assignee_id: str | None) -> dict[str, Any]` returns `{"id", "identifier", "url", "state": {"type"}}`
  - `attach_url(token, *, issue_id, url, title) -> None`
  - `issues_for_url(token, url) -> list[dict[str, Any]]` (issues carrying an attachment with that URL)
  - `get_issue(token, ref) -> dict[str, Any] | None` (`ref` = id or identifier; returns `{"id", "identifier", "url", "title", "dueDate", "state": {"name", "type"}}`; `None` when Linear says not found)
- `FakeLinear` implements the same methods in memory, plus `set_state(identifier, type_)`.

- [ ] **Step 1: Write the failing tests** in `tests/test_linear_clients.py` using `httpx.MockTransport`:
  - `test_authorize_url_asks_for_app_actor_and_pkce`: the URL host is `linear.app`, query has `actor=app`, `scope=read,write,issues:create`, `code_challenge_method=S256`.
  - `test_exchange_returns_organization_id`: token response then organization response; `grant.external_account_id == "org-1"`, `grant.extra["url_key"] == "acme"`.
  - `test_graphql_errors_raise_provider_error`: a 200 with `{"errors": [...]}` raises `ProviderError` with `provider == "linear"`.
  - `test_get_issue_not_found_is_none`.
  - `test_linear_not_configured_without_webhook_secret`: with only client id and secret set, `config.is_configured("linear") is False`.
- [ ] **Step 2:** Run `uv run pytest tests/test_linear_clients.py -q`. Expected: FAIL (no `LinearOAuth`).
- [ ] **Step 3:** Implement the config and client changes above, following `JiraOAuth`/`JiraApi` (reuse `_Http.json`, `_creds`).
- [ ] **Step 4:** Run `uv run pytest tests/test_linear_clients.py tests/test_integrations_api.py -q`. Expected: PASS.
- [ ] **Step 5:** Commit: `feat(integration): Linear OAuth and GraphQL clients (ADR-0044)`

### Task 3: The Linear connector

**Files (mytheus-backend):**
- Create: `services/integration_service/connectors/linear.py`
- Test: `tests/test_linear_sync.py`

**Interfaces:**
- Consumes: Task 2 `LinearApi`, `store.get_connection/access_token/get_cursor/put_cursor/link_for_nexus/link_for_external/put_link/touch_synced`, `CoreApiClient.external_status` (Task 5 adds its `source` argument; until then pass nothing and let Task 5 update the call).
- Produces, in `connectors/linear.py`:
  - `PROVIDER = config.LINEAR`, `TEAM_KIND = "linear_team"`
  - `map_status(issue: dict) -> str` (`"done"` when `state.type == "completed"`, else `"confirmed"`)
  - `action_url(action_id: uuid.UUID) -> str` = `{APP_BASE_URL}/work/tasks?action={action_id}` (the attachment URL used for the crash-window lookup)
  - `async post_install(session, conn, clients) -> None`: `first_team`; none -> `ProviderError(PROVIDER, "post_install", "no_team")`; `put_cursor(..., TEAM_KIND, {"team_id", "team_key", "url_key"}, None)`
  - `async push_action(session, org_id, action, clients) -> str` returning `created|linked|exists|not_connected`, the same three steps as `jira.push_action` (id map; `issues_for_url(action_url)`; create with markdown description `"{content}\n\nConfirmed in Mytheus. [Open the meeting]({source_url})"`, `dueDate`, assignee by email, then `attach_url`). Stored external id = the issue `identifier`; url = the issue `url`.
  - `async process_webhook(session, org_id, payload, core, clients) -> str` returning `processed|skipped`: only `type == "Issue"` and `action == "update"`; id map by `data.identifier`; re-read with `get_issue`; compare `map_status` to `last_synced_status`; call core then `put_link`.

- [ ] **Step 1: Write the failing tests** in `tests/test_linear_sync.py` (mirror the Jira cases in `tests/test_integrations_sync.py`):
  - `test_confirmed_action_creates_one_issue_with_back_link_and_attachment`
  - `test_redelivered_action_is_exists_not_a_second_issue`
  - `test_crash_window_links_issue_found_by_attachment_url`
  - `test_unknown_assignee_email_creates_unassigned_issue` (Review Focus 4)
  - `test_webhook_for_issue_not_in_id_map_is_skipped` (Review Focus 1)
  - `test_webhook_takes_status_from_reread_not_payload`: payload says completed, re-read says started -> no core call
  - `test_completed_maps_done_and_canceled_maps_confirmed`
  - `test_echo_of_known_status_is_suppressed`
- [ ] **Step 2:** Run `uv run pytest tests/test_linear_sync.py -q`. Expected: FAIL (module missing).
- [ ] **Step 3:** Implement `connectors/linear.py` with the interfaces above. Reuse `jira._due` and `jira._source_url` by moving them to a shared `connectors/_tracker_common.py` (`due_iso(raw) -> str | None`, `source_url(meeting_id) -> str`) and importing them from both connectors.
- [ ] **Step 4:** Run `uv run pytest tests/test_linear_sync.py tests/test_integrations_sync.py -q`. Expected: PASS (Jira unchanged after the move).
- [ ] **Step 5:** Commit: `feat(integration): Linear connector, issues from confirmed actions and status back`

### Task 4: Wire Linear in: webhook door, worker, consumer, connect and disconnect

**Files (mytheus-backend):**
- Modify: `services/integration_service/webhooks.py` (`verify_linear(headers, body, *, now=None) -> None`; a `LINEAR` branch in `receive`; docstring)
- Modify: `services/integration_service/worker.py` (`_dispatch`: `config.LINEAR` -> `linear.process_webhook`)
- Modify: `services/integration_service/consumer.py` (`handle_action` pushes to the tenant's tracker: Task 5's `tracker.active(session)`; until Task 5 lands, try `jira.push_action` and, when it answers `not_connected`, `linear.push_action`)
- Modify: `services/integration_service/routes.py` (post-install `elif provider == config.LINEAR: await linear.post_install(...)`; disconnect revokes the Linear token; no webhook cleanup is needed)
- Modify: `services/api_gateway/edge.py` only if `/webhooks/linear` is not already routed to integration-service (check; `/oauth/linear` already is)
- Test: `tests/test_linear_webhooks.py`

**Interfaces:**
- `verify_linear`: `Linear-Signature` must equal hex HMAC-SHA256(`LINEAR_WEBHOOK_SECRET`, raw body), compared with `hmac.compare_digest`; parsed body's `webhookTimestamp` (ms) within 60 s of now; else `_unauthorized()`.
- Tenant = `_route(config.LINEAR, payload["organizationId"])`; delivery id = `Linear-Delivery` header or the body hash; event type = `f"{type}:{action}"`.

- [ ] **Step 1: Write the failing tests:**
  - `test_valid_signature_writes_one_inbox_row`
  - `test_bad_signature_is_401`
  - `test_stale_timestamp_is_401` (Review Focus 2)
  - `test_replayed_delivery_is_200_noop` (Review Focus 2)
  - `test_unknown_organization_is_ignored` (Review Focus 1)
  - `test_connect_linear_while_jira_connected_is_409` and `test_after_switching_trackers_action_goes_only_to_linear` (Review Focus 5)
  - `test_disconnect_revokes_linear_token`
- [ ] **Step 2:** Run `uv run pytest tests/test_linear_webhooks.py -q`. Expected: FAIL.
- [ ] **Step 3:** Implement the wiring above.
- [ ] **Step 4:** Run `uv run pytest tests/test_linear_webhooks.py tests/test_integrations_api.py tests/test_integrations_sync.py -q`. Expected: PASS.
- [ ] **Step 5:** Commit: `feat(integration): Linear webhooks, worker, consumer and connect flow`

### Task 5: Tracker-neutral internal reads in integration-service

**Files (mytheus-backend):**
- Create: `services/integration_service/tracker.py`
- Modify: `services/integration_service/jira_status.py` (use `tracker`; result gains `provider`)
- Modify: `services/integration_service/watch_sources.py` (`LinearSource` gains `issues`; `_linear(org_id, keys, clients)`; `WatchIssue` gains `provider: Literal["jira", "linear"] = "jira"`)
- Modify: `services/integration_service/core_client.py` (`external_status(..., source: str = "jira")`), and `connectors/linear.py` passes `source="linear"`
- Modify: `services/integration_service/consumer.py` (use `tracker.active`)
- Test: `tests/test_tracker_reads.py`

**Interfaces:**
- `tracker.active(session) -> tuple[str, store.Connection] | None`: the one active connection among `config.PM_PROVIDERS`, provider name first.
- `tracker.issue_open(provider, token, ctx: dict, key) -> bool | None`: re-read via the provider's `get_issue` and `map_status`; `None` when not found. `ctx` is the Jira site dict or the Linear team cursor.
- `IssueStatusResult` gains `provider: Literal["jira", "linear"] | None = None`; `IssueStatus.source` becomes `"jira" | "linear" | "watermark"`.
- `/internal/watch/sources` answers `linear: {connected, issues: [WatchIssue(provider="linear")]}` when Linear is the tracker, reading the same `jira_keys` (Linear identifiers have the same `ABC-123` shape); `jira` answers `{connected: false}` in that case, and the reverse when Jira is the tracker.

- [ ] **Step 1: Write the failing tests:**
  - `test_issue_status_reads_linear_when_linear_is_the_tracker` (provider `linear`, `open` from the re-read)
  - `test_issue_status_falls_back_to_watermark_when_linear_unreachable` (Review Focus 3: still 200)
  - `test_watch_sources_reads_keys_from_linear`
  - `test_watch_sources_jira_unchanged_when_jira_is_the_tracker`
  - `test_external_status_names_linear_as_source`
- [ ] **Step 2:** Run `uv run pytest tests/test_tracker_reads.py -q`. Expected: FAIL.
- [ ] **Step 3:** Implement the interfaces above.
- [ ] **Step 4:** Run `uv run pytest tests/test_tracker_reads.py tests/test_linear_sync.py tests/test_integrations_sync.py tests/test_reach_integrations.py -q`. Expected: PASS.
- [ ] **Step 5:** Commit: `feat(integration): issue status and watch sources work for Jira or Linear`

### Task 6: core-api and workflow-service follow the tracker

**Files (mytheus-backend):**
- Modify: `services/core_api/routes/internal_integrations.py` (`ExternalStatus.source: Literal["jira", "linear"]`)
- Modify: `services/core_api/routes/internal_assumptions.py` (the `Issue` model gains `provider: Literal["jira", "linear"] = "jira"`; a break is produced when the issue's own provider is `connected` in `body.sources`; break `source` and the ref prefix use the provider: `f"{provider}:{key}:due:..."`)
- Modify: `services/core_api/routes/internal_bets.py` (`JiraOutcome` gains `provider: Literal["jira", "linear"] = "jira"`; evidence titles and the not-connected note say "Jira" or "Linear" from it; stored `kind`/`source` values stay `"jira"`)
- Modify: `services/core_api/routes/internal_promise_drift.py` (the "still open in Jira" copy uses the provider name)
- Modify: `services/core_api/routes/decision_why.py` (validator message: `"not an issue key (like SSO-142)"`)
- Modify: `services/workflow_service/assumption_activities.py` (send `issues` = Jira issues + Linear issues), `bet_activities.py` and `scheduled_activities.py` (pass `provider` through from issue-status)
- Test: extend `tests/test_decision_bets.py`, `tests/test_decision_why_blast.py` and the promise-drift and assumption-watch tests (find with `grep -ln "issue-status\|watch-results" tests/`)

**Interfaces:** Consumes Task 5's `provider` fields. Produces nothing new for later tasks.

- [ ] **Step 1: Write the failing tests:**
  - `test_linear_issue_breaks_reason_with_linear_source`
  - `test_linear_issue_ignored_when_linear_not_connected`
  - `test_bet_evidence_says_linear_when_provider_linear`
  - `test_promise_drift_copy_names_linear`
  - `test_external_status_accepts_linear`
  - `test_assumption_activity_forwards_linear_issues`
- [ ] **Step 2:** Run those tests. Expected: FAIL.
- [ ] **Step 3:** Implement the changes above.
- [ ] **Step 4:** Run `make test` (the full backend suite) and `make lint`. Expected: PASS, no new lint findings.
- [ ] **Step 5:** Commit: `feat(core): decisions, bets and promise drift follow Linear as well as Jira`

### Task 7: UI: the Linear segment and its Set up guide

**Files (mytheus-ui):**
- Modify: `src/lib/integrations.ts` (`CONNECTABLE` includes `"linear"`; `trackerOptions` makes Linear selectable, with the same reasons as Jira: `connected`, `other_tracker_connected`, `not_configured`; `SETUP_GUIDES.linear` with console `https://linear.app/settings/api/applications`, redirect URL `{origin}/settings/integrations/callback/linear`, webhook URL `{api}/webhooks/linear` with resource type Issue, scopes `read`, `write`, `issues:create`, env vars `LINEAR_CLIENT_ID`, `LINEAR_CLIENT_SECRET`, `LINEAR_WEBHOOK_SECRET`; update the comments that say Linear is not built)
- Modify: `src/components/settings/IntegrationsPanel.tsx` (the tracker card's segmented choice selects Jira or Linear; the selected tracker's view, body and action come from the existing `viewFor/bodyFor/actionFor` with the chosen key; `CardMeta.key` no longer excludes `"linear"`; the selection defaults to the connected tracker, else Jira; "Coming soon" is removed)
- Modify: `src/app/(app)/settings/integrations/loading.tsx` only if the skeleton names "Coming soon"
- Test: `tests/integrations.test.ts`, `tests/e2e/integrations-a11y.spec.tsx` (and its harness)

- [ ] **Step 1: Write the failing tests** in `tests/integrations.test.ts`:
  - `linear is selectable and connectable when configured`
  - `linear connected blocks jira with other_tracker_connected, and the reverse`
  - `linear not configured shows Needs setup with the Linear guide`
  - a11y spec: the tracker radio group is keyboard operable (arrow keys move between Jira and Linear) and passes axe in both themes.
- [ ] **Step 2:** Run `npm test` and `CT_PORT=3641 npm run test:a11y -- integrations-a11y`. Expected: FAIL on the new cases.
- [ ] **Step 3:** Implement the changes above.
- [ ] **Step 4:** Run `npx tsc --noEmit -p .`, `npx next lint`, `npm test`, the a11y spec. Expected: clean, all pass.
- [ ] **Step 5:** Commit: `feat(integrations): Linear is a real choice on the Project tracker card`

### Task 8: Platform secrets and the owner guide

**Files:**
- Modify (mytheus-platform): `scripts/push-secrets.sh` (integrations mode knows `LINEAR_CLIENT_ID`, `LINEAR_CLIENT_SECRET`, `LINEAR_WEBHOOK_SECRET`; Linear counts as configured only with all three), `env/prod.env.example` (the three names, empty)
- Modify (mytheus-docs): `work-done-till-now/16-integrations-setup.md` owner table gains a Linear row (where to create the app, redirect URL, webhook URL and resource type, scopes, env vars)

- [ ] **Step 1:** Edit the script and example. Run `bash -n scripts/push-secrets.sh` (expected: no output) and `scripts/push-secrets.sh integrations --dry-run` against a scratch `.env` holding only two of the three Linear names (expected: Linear reported incomplete, no values printed).
- [ ] **Step 2:** Commit (platform): `feat(secrets): integrations mode pushes the Linear app values`
- [ ] **Step 3:** Add the Linear row to note 16. Commit (docs): `docs(integrations): Linear owner setup row`

### Task 9: Live test with the owner, then the work-done note

**Files (mytheus-docs):**
- Create: `work-done-till-now/19-linear-integration.md`

- [ ] **Step 1:** Ask the owner to create a free Linear workspace and an OAuth application (Task 8's guide), and put the three values in `mytheus-backend/.env`. The owner pastes them; never echo them.
- [ ] **Step 2:** With the local stack running and a public tunnel for `/webhooks/linear` (the owner chooses the tunnel), have the owner: connect Linear; confirm an action in a meeting; see the issue appear with the back-link; move it to Done in Linear; see the action become done in Mytheus. Record each observation.
- [ ] **Step 3:** Write note 19 in the work-done README's structure: what landed per repo with commit hashes, deviations with reasons, open gaps, and the exact verification commands and observations.
- [ ] **Step 4:** Commit: `docs(work-done): Linear integration (note 19)`. Push every repo only when the owner says push.
