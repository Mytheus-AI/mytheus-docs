# Meeting invitations by email (ADR-0043)

**Status:** `DONE WITH GAPS` on branches `feat/meeting-invites` in `mytheus-backend`, `mytheus-ui`, `mytheus-platform` and `mytheus-docs`; committed, not pushed, not deployed. Last verified 2026-11-12.

The owner asked for meeting invitations by email: people are sent a meeting, can join it from the email, and get a calendar schedule they can share. Every state designed, both themes, no questions asked (defaults are listed below). Decision record: [ADR-0043](../adr/0043-meeting-email-invitations.md). Canon: `meeting_invitations`, `meetings.calendar_sequence` and the wider `email_deliveries.kind` in [_canon/02-data-model.md](../_canon/02-data-model.md); the routes in [_canon/01-service-catalog.md](../_canon/01-service-catalog.md). ADR-0042 is taken by the Myte name (`feat/mytes-name`), so this is 0043.

## The guest-join finding

There is no guest join. `meeting-service` mints a LiveKit token only for a `meeting_participants` row of the caller's own tenant, and `/meetings/{id}/join` needs a Clerk session. So invitations go to workspace members only: an address that is not a member is a 422 naming it, and the UI offers the member picker (no free-typed email chips) plus Copy link. Every invitee is added to the meeting as an attendee, because an invitation whose Join button refuses them would be a dead end. A guest path (a scoped guest identity, consent, a token without a `users` row) is a separate decision.

## mytheus-backend

- Migration `d4e5f6a7b8c9` (chained off `c8d9e0f1a2b3`, the single head on `origin/main` at the start and again before the commit, after `feat/mytes-name` landed): `meeting_invitations` (RLS `ENABLE` + `FORCE`, `tenant_isolation`, `grant_tenant_table`, `UNIQUE (meeting_id, email)`), `meetings.calendar_sequence` (ORM model carries it and its `CHECK`, so autogenerate stays clean), `email_deliveries.kind` widened by three values.
- `services/core_api/routes/invitations.py`: `POST/GET /api/v1/meetings/{id}/invitations`, `GET /api/v1/meetings/invitations/status`, `GET /api/v1/meetings/{id}/calendar.ics`, public `GET /api/v1/public/calendar/{token}.ics`, and `schedule_change_after_commit`, called from `PATCH /api/v1/meetings/{id}` (`routes/meetings.py`), which also bumps `calendar_sequence` in the same `UPDATE` on a new time or a cancellation.
- `services/core_api/meeting_calendar.py`: the RFC 5545 builder (CRLF, 75-octet folding that never splits a UTF-8 sequence, TEXT escaping), the HMAC calendar token (key `MEETING_INVITE_TOKEN_SECRET`, else derived from `GATEWAY_SIGNING_KEY`, purpose "meeting-calendar"), the Google and Outlook web links.
- `services/core_api/invitation_email.py`: subject, text and HTML for `invite`, `update` and `cancel`. Table layout, inline styles, the product palette (cream stage, white card, ink, terracotta `--signal-ink` button so white text passes AA), `color-scheme: light dark` plus a `prefers-color-scheme: dark` block and Outlook.com's `[data-ogsc]`/`[data-ogsb]`, a preheader, a 600 px column that shrinks on phones. Times in the recipient's zone (`listening_preferences.time_zone`), else "UTC (Coordinated Universal Time)".
- `services/notification_service/invitations.py`: `POST /internal/email/invitations` (caller `core-api`): per-message claim in `email_deliveries`, send after commit, three attempts with backoff on a transient failure, a 4xx final at once, `skipped`/`not_configured` with no provider. `email.py`: `reply_to` and `attachments` on both senders (SMTP sets `method=REQUEST` on the `text/calendar` part), `EMAIL_FROM` defaults to `Mytheus <noreply@mytheus.cloud>` when Resend has a key. No `List-Unsubscribe`: the channel has no unsubscribe mechanism and an invitation is transactional.
- `services/core_api/notification_client.py`: `send_invitations`. `libs/shared/audit.py`: `meeting.invitations_sent`.
- Tests: `tests/test_meeting_invitations.py` (21: host send with the rendered email and the attachment, UTC fallback, who may send, cross-tenant 404 on every route, another tenant's user, dedupe and the 50 cap, not configured, no caller secret and a down service, failure then resend, the 429, an ended meeting, the list for hosts and for participants, `.ics` parsed by an RFC 5545 reader, escaping and folding, the public link carries no agenda or address, forged, foreign and share-link tokens refused, no key no link, reschedule update with SEQUENCE 1, cancellation, nothing without email, the escaped dark-mode-safe email) and `tests/test_invitation_delivery.py` (7: Resend payload with `reply_to` and the attachment, the default sender, caller token, replay never resends, retries, not configured, cross-tenant 404).

Verified with a throwaway `postgres:15` on 55491 and `redis:7` on 56491 (never the dev DB on 5443):

```
export DATABASE_URL=postgresql+asyncpg://postgres:postgres@127.0.0.1:55491/mytheus \
  BOOTSTRAP_DATABASE_URL=postgresql+asyncpg://postgres:postgres@127.0.0.1:55491/mytheus \
  REDIS_URL=redis://127.0.0.1:56491/0 MYTHEUS_REQUIRE_DB=1
make proto
uv run ruff check . && uv run black --check . && uv run mypy .      # clean, 281 files
uv run python scripts/check_authz.py                                # passed
uv run python -m scripts.emit_openapi --check                       # specs match the code
npx --yes @stoplight/spectral-cli@6 lint design/openapi/core-api.yaml design/openapi/notification-service.yaml \
  --ruleset design/.spectral.yaml --fail-severity=error             # 0 errors
docker run --rm -v <dir>:/w tufin/oasdiff breaking /w/base/<spec>.yaml /w/head/<spec>.yaml --fail-on ERR
                                                                    # core-api, notification-service: exit 0
uv run pytest -q                                                    # 1617 passed, 1 skipped, on the rebased branch
                                                                    # (fresh DB on 55492, Redis on 56492, no other run)
```

## mytheus-ui

- `components/meetings/InviteShareCard.tsx` in the meeting page's people rail before the call: Join link with Copy (announced), Add to calendar (Google, Outlook, Download .ics through the route handler `app/api/meetings/[meetingId]/calendar/route.ts`), and for the host, the creator or an admin: Invite by email (the shared member picker, an optional note up to 500 characters, Send), the invitation list with status chips (Sent, Update sent, Failed with the reason, Not emailed, Sending), Retry, Resend and Resend to everyone. States: loading bones, empty, error with Retry, permission (non-hosts see the share links only), email not set up ("Email isn't set up for this workspace yet", the link and the .ics still offered, Send becomes Add to meeting), no time yet.
- `NewMeeting.tsx`: "Send email invites" under People once someone is picked, on by default when this workspace can send email (asked when the window opens), off and explained when it cannot; an optional note; the summary line says "invites by email"; the toast after creating says who was emailed.
- `app/actions/invitations.ts` (`sendInvitations`, `loadInvitations`, `invitationEmailStatus`), `lib/api/invitations.ts`, `lib/api/invitations-types.ts`, `styles/meeting-invites.css` (tokens only). The detail page's loading skeleton and its CT pair gained the card.
- CT: `tests/e2e/invites-a11y.spec.tsx` (23 tests: axe in six states and both themes, 360 px, the list, retry, keyboard invite with focus return, a refused send, not configured, not the host, copy, error Retry, both New meeting states) and `invites-shots.spec.tsx`.

Verified:

```
npx tsc --noEmit -p .                                   # clean
npx next lint                                           # no warnings or errors
npm test                                                # 542 passed
CT_PORT=3561 npm run test:a11y -- --workers=4           # rebased on main: 550 passed, 1 failed, 459 skipped
                                                        # (shots). The failure is memory-a11y "forget everything
                                                        # asks first" (dark), untouched by this work; it passes
                                                        # 3 of 3 with --repeat-each=3 alone (a load flake)
npx -y node@20 ./node_modules/next/dist/bin/next build  # compiled; /meetings/[meetingId] and
                                                        # /api/meetings/[meetingId]/calendar built
```

## mytheus-platform

- `scripts/push-secrets.sh email`: copies `EMAIL_PROVIDER`, `RESEND_API_KEY`, `EMAIL_FROM` from `mytheus-backend/.env` into the server's `prod.env` over SSH stdin (core-api, notification-service and notification-consumer all read it), generates `MEETING_INVITE_TOKEN_SECRET` on the server once (kept if set), sets `APP_BASE_URL` if unset, backs up first, prints key names only; `--dry-run` prints names and touches nothing. Exercised against a temporary directory in a Debian container (idempotent, `0600`, no values printed); not run against the VPS.
- `env/prod.env.example`: the default sender, `MEETING_INVITE_TOKEN_SECRET`, `MEETING_INVITE_HOURLY_LIMIT`, `APP_BASE_URL`, `API_PUBLIC_URL`.

## Defaults chosen (no questions asked)

- Who may send: host, creator, workspace owner/admin. Recipients: members only (no guest path). Limit: 50 per call, 100 emails per meeting per hour.
- An invitee is added to the meeting. A resend keeps the earlier note unless a new one is given.
- Join link: `{APP_BASE_URL}/meetings/{id}/join`. Calendar link: `{API_PUBLIC_URL}/api/v1/public/calendar/{token}.ics`, no expiry (it carries calendar data only and the meeting's deletion makes it 404).
- Outlook link uses `outlook.office.com` (work accounts); personal Outlook users can use the .ics.
- Updates go only to people whose invitation was delivered (`sent_at` set); a title change alone sends nothing.

## Gaps

- The reschedule and cancellation emails are sent after commit in-process; a crash in between loses that email (ADR-0043 consequences).
- No guest join, so no external invitations.
- Not deployed. Owner steps: `scripts/push-secrets.sh email` (it reads the key from the local `.env`; never printed), then deploy `mytheus-backend` (migration `d4e5f6a7b8c9` runs in `migrate`), then `mytheus-ui`. Email is live once `RESEND_API_KEY` and `EMAIL_FROM` are in `prod.env` and the `mytheus.cloud` domain is verified in Resend.
