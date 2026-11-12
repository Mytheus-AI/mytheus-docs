# ADR-0043: Meeting invitations by email, with a calendar event

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md): `meeting_invitations`, `meetings.calendar_sequence`, the wider `email_deliveries.kind`.
- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md): the new `core-api` routes and `notification-service`'s `POST /internal/email/invitations`.
- [ADR-0036](0036-plan-11-proactive-agent-and-delivery-build.md) (email is opt-in per user for notifications), [ADR-0038](0038-agent-personas-profiles-agendas-and-meeting-agent.md) (the host, agendas), [ADR-0040](0040-polish-program.md) (the email channel, "not connected until set", share-link tokens, additive API evolution).

## Status

**Accepted** (2026-11-12).

## Context

A meeting could be created and people added to it, but nobody heard about it outside the app: no email, nothing on their calendar, no link to forward. The owner asked for meeting invitations by email that people can join from, with a shareable calendar schedule. The email channel already existed (`notification-service`, Resend or SMTP, "not connected until set"), and the owner has since connected Resend for the verified domain `mytheus.cloud`.

Joining a call needs a signed-in participant: `meeting-service` mints a LiveKit token only for a `meeting_participants` row of the caller's own tenant. There is no guest join path.

## Decision

1. **D1 Who may send, and to whom.** `POST /api/v1/meetings/{id}/invitations` `{emails?, user_ids?, message?}` is for the meeting's host, its creator, or a workspace owner/admin (403 otherwise, 404 across tenants). Recipients are people in the workspace only: a user id is checked under RLS, an address must match a user of the tenant, and an address that does not is a 422 naming it. Recipients are de-duplicated (an address and a user id for one person count once), at most 50 per call. Each recipient becomes an attendee of the meeting if they were not one, because an invitation whose Join button refuses them is worse than none.
2. **D2 No guests yet.** Because no guest join exists, external addresses are refused rather than mailed. The UI offers members only, plus Copy link (members sign in and land on the pre-join). A guest path is its own decision (a scoped guest identity, a consent notice, a token mint without a `users` row) and is out of scope here.
3. **D3 It is transactional mail, not a notification.** The host naming a person and pressing Send is the request, so the invitation is sent to that person directly, like the weekly digest (`/internal/email/user`), rather than through the `notifications` topic and its per-user opt-in (ADR-0036). `notification-service` stays the only service that talks to a provider: `core-api` renders each email (the recipient's own time zone from `listening_preferences.time_zone`, else UTC with the zone written out) and posts the batch to `POST /internal/email/invitations`.
4. **D4 Delivery.** `notification-service` claims each message in `email_deliveries` under its own idempotency key (core-api uses `invite:{meeting}:{invitation}:{send_count}`, so a resend is a new delivery and a replayed call is not), commits, then sends with up to three attempts and exponential backoff for a transient failure (unreachable, HTTP 429 or 5xx); a 4xx is final at once. `Reply-To` is the sender's address. `EMAIL_FROM` defaults to `Mytheus <noreply@mytheus.cloud>`. No `List-Unsubscribe` header: there is no unsubscribe mechanism in the email channel, and an invitation is transactional.
5. **D5 State.** `meeting_invitations` (owned by `core-api`) keeps one row per meeting and address: `status` (`pending`, `sent`, `failed` with `last_error`, `not_configured`), `last_kind`, `send_count`, `sent_at`, the note. Not configured (no provider, or no caller secret for `notification-service`) records `not_configured` and sends nothing; `notification-service` unreachable records `failed` so the host can Retry. Neither is a 5xx. Every send is audited (`meeting.invitations_sent`; participants added also `meeting.participants_added`).
6. **D6 Rate limit.** At most `MEETING_INVITE_HOURLY_LIMIT` (default 100) invitation emails per meeting per clock hour, counted in Redis under `nexus:{org_id}:ratelimit:meeting-invites:{meeting_id}:{hour}`; over it is a 429 with `Retry-After`, and Redis unreachable is a 503 (fail closed).
7. **D7 The calendar event.** RFC 5545, one VEVENT: `UID meeting-{id}@mytheus.cloud` (stable), `SEQUENCE` from `meetings.calendar_sequence`, `DTSTART`/`DTEND`/`DTSTAMP` in UTC, `SUMMARY`, `DESCRIPTION` (agenda and join link), `URL` and `LOCATION` (the join link), `ORGANIZER` (the host), one `ATTENDEE` per participant, `VALARM` ten minutes before. `METHOD:REQUEST` when attached to an invitation or update (`text/calendar; method=REQUEST`, so Gmail and Outlook show their RSVP card), `METHOD:CANCEL` with `STATUS:CANCELLED` for a cancellation, `METHOD:PUBLISH` for a download. Members download it from `GET /api/v1/meetings/{id}/calendar.ics`; the email also carries Google Calendar and Outlook web "add event" links.
8. **D8 The calendar link.** Each invitation links `GET /api/v1/public/calendar/{token}.ics`, no sign-in. The token is `<payload>.<signature>` (base64url): the tenant and meeting ids, signed with HMAC-SHA256 (16 bytes) under `MEETING_INVITE_TOKEN_SECRET`, or a key derived from `GATEWAY_SIGNING_KEY` for the purpose "meeting-calendar". The key is not the share-link key, so neither kind of token opens the other. It grants calendar data only: title, time, join link and the host's name; never the agenda, an attendee address or anything said in the meeting. No row stores it.
9. **D9 Reschedule and cancel.** `PATCH /api/v1/meetings/{id}` bumps `calendar_sequence` in the same `UPDATE` when `scheduled_start`/`scheduled_end` change or the status becomes `cancelled`. After the request's transaction commits (SQLAlchemy `after_commit`, because `BackgroundTasks` run before `tenant_session` commits), everyone with a `sent_at` gets the update or the cancellation, keyed by the new SEQUENCE so a repeat is a no-op, and only when email is configured. A title change alone sends nothing.

## Consequences

- A host can invite from the New meeting window (on by default when email is configured, with a short note) and from the meeting page's "Invite and share" card (Copy link, Google, Outlook, Download .ics, the invitation list with Retry and Resend). People who are not hosts see the share half only.
- Every API change is additive (`oasdiff breaking` exits 0 for `core-api` and `notification-service`).
- The update after a reschedule is best effort: a crash between the commit and the send loses that email (the row keeps its earlier state). A durable path (an outbox event consumed by `workflow-service`) can replace it if that ever matters.
- Owner steps: `scripts/push-secrets.sh email` (mytheus-platform) writes `EMAIL_PROVIDER`, `RESEND_API_KEY`, `EMAIL_FROM` and a generated `MEETING_INVITE_TOKEN_SECRET` into the server's `prod.env`; then redeploy `mytheus-backend`.
