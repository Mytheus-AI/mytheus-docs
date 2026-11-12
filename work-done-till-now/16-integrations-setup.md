# 16. Integrations: owner setup guide and the Settings page redesign

Status: `DONE WITH GAPS` (code built and verified locally on 2026-11-12; nothing pushed or deployed; no provider app created yet). Not a plan band: a fix to the P12 / Phase 4 integrations surface.

## Why

The owner reported that Settings > Integrations looked broken and that its buttons did nothing. Both were true, for one reason: production has **no OAuth app credentials** for any provider, so integration-service lists every provider as `configured: false`, and the page drew six cards of uneven size, each with a "Not available yet" chip, a repeated paragraph and a disabled grey Connect. Nothing on the page said what to do about it.

The OAuth wiring itself was read end to end and is correct (section 4): once a provider's keys are on the server, Connect works with no code change.

## Owner guide: turn on a provider

Do this once per provider you want. The page's **Set up** button shows the same steps, with the exact URL to copy.

1. **Create the app** at the provider (table below), paste the redirect URL, add the scopes.
2. **Put the values in `mytheus-backend/.env`** on your own machine, under the env var names below. Never commit them (the repos are public).
3. **Push them:** from `mytheus-platform`, run
   `scripts/push-secrets.sh integrations --dry-run` (names only: what would be pushed, what is missing, which providers end up configured), then
   `scripts/push-secrets.sh integrations`.
   It writes only the keys that are present into `/etc/dokploy/mytheus-secrets/env/prod.env` over SSH, backs that directory up first, never prints a value, and, only when they are unset, generates `OAUTH_STATE_SECRET` on the server and sets `APP_BASE_URL=https://dashboard.mytheus.cloud` and `OAUTH_REDIRECT_BASE=https://api.mytheus.cloud`.
4. **Redeploy mytheus-backend** (integration-service reads `prod.env`). Deploy one repo at a time.
5. Open **Settings > Integrations**: the provider now says **Ready to connect**. Press **Connect** and approve at the provider.

The redirect URL is always `{APP_BASE_URL}/settings/integrations/callback/{provider}` (mytheus-backend `services/integration_service/config.py`, `redirect_uri`). In production `APP_BASE_URL` is `https://dashboard.mytheus.cloud`.

| Provider | Create the app at | Redirect URL to register | Scopes requested | Env vars |
|---|---|---|---|---|
| Google Calendar | Google Cloud Console > APIs and Services > Credentials (`https://console.cloud.google.com/apis/credentials`): enable the Calendar API and the Drive API, create an OAuth client ID of type Web application | `https://dashboard.mytheus.cloud/settings/integrations/callback/google_calendar` | `openid`, `email`, `https://www.googleapis.com/auth/calendar.readonly`, `https://www.googleapis.com/auth/drive.readonly` | `GOOGLE_OAUTH_CLIENT_ID`, `GOOGLE_OAUTH_CLIENT_SECRET` |
| Slack | `https://api.slack.com/apps`: create an app from scratch; OAuth and Permissions (redirect URL, bot token scopes); Event Subscriptions on, request URL `https://api.mytheus.cloud/webhooks/slack`, bot events `message.channels`, `message.groups`, `message.im`, `message.mpim`; Interactivity on, request URL `https://api.mytheus.cloud/webhooks/slack/interactions`; a message shortcut "Check against decisions" with callback ID `check_against_decisions` | `https://dashboard.mytheus.cloud/settings/integrations/callback/slack` | `chat:write`, `users:read`, `users:read.email`, `im:write`, `channels:history`, `groups:history`, `channels:read`, `groups:read`, `commands`, `im:history`, `mpim:history` | `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET`, `SLACK_SIGNING_SECRET` (all three, or Slack stays "Needs setup") |
| Jira | Atlassian developer console (`https://developer.atlassian.com/console/myapps/`): OAuth 2.0 integration, Permissions > Jira API, Authorization > callback URL | `https://dashboard.mytheus.cloud/settings/integrations/callback/jira` | `read:jira-work`, `write:jira-work`, `read:jira-user`, `manage:jira-webhook`, `offline_access` | `JIRA_CLIENT_ID`, `JIRA_CLIENT_SECRET` |
| Confluence | Atlassian developer console, a separate OAuth 2.0 integration (its callback URL differs from Jira's), Permissions > Confluence API | `https://dashboard.mytheus.cloud/settings/integrations/callback/confluence` | `read:confluence-space.summary`, `read:confluence-content.summary`, `write:confluence-content`, `offline_access` | `CONFLUENCE_CLIENT_ID`, `CONFLUENCE_CLIENT_SECRET` |
| HubSpot | HubSpot developer account (`https://app.hubspot.com/developer`): a public app, Auth tab | `https://dashboard.mytheus.cloud/settings/integrations/callback/hubspot` | `oauth`, `crm.objects.companies.read`, `crm.objects.companies.write`, `crm.objects.contacts.read`, `crm.objects.contacts.write`, `crm.objects.deals.read` | `HUBSPOT_CLIENT_ID`, `HUBSPOT_CLIENT_SECRET` |
| Notion | `https://www.notion.so/my-integrations`: a public integration with Read, Update and Insert content | `https://dashboard.mytheus.cloud/settings/integrations/callback/notion` | none (Notion grants access per shared page) | `NOTION_CLIENT_ID`, `NOTION_CLIENT_SECRET` |

Scopes are copied from `services/integration_service/clients.py` (mytheus-backend, `origin/main` 81d6876). Jira's and Google's webhooks are registered by the service through the API (`{OAUTH_REDIRECT_BASE}/webhooks/{provider}`), so nothing else is pasted for them. Linear is in the canon enum but not built; the page shows it as "Coming soon".

## What landed, per repo

### mytheus-ui (`fix/integrations-ux`, worktree `wt/integ-ui`)

- `src/components/settings/IntegrationsPanel.tsx` rewritten: one card component for all six providers in one grid (`repeat(auto-fill, minmax(min(100%,236px),1fr))`: 3 columns in the 880px settings panel at desktop, 2 on tablets, 1 on phones; equal widths, equal heights per row), under "For your Mytes" (Google Calendar, Slack, Project tracker) and "Share and export" (HubSpot, Notion, Confluence). Each card: icon and one status chip, name, a two-line clamped description with More, a footer with exactly one action.
- States, one chip each: Connected (account, last sync, Disconnect with a confirm press), Ready to connect (Connect), Needs setup (admins: Set up; members: "Ask an admin to set this up"), Error (amber, Reconnect), plus "Tracker in use" when another tracker is connected. The tracker card's footer holds a Jira / Linear segmented choice; Linear is a disabled segment marked "Coming soon", never a chip.
- `IntegrationSetup.tsx` (new): the Set up dialog (the kit `Dialog`: focus moves in, Tab is trapped, Escape and the scrim close, focus returns). It shows what the integration does, a link to the provider's console, the steps, the redirect URL with Copy (Slack: also the event and interactivity URLs), the scopes with Copy, the env var names, and "ask your server operator to run `scripts/push-secrets.sh integrations`".
- `ClampedText.tsx` (new): two-line clamp with More/Less only when the text is cut.
- `src/lib/integrations.ts`: `providerView`, `VIEW_CHIP`, `appOriginFrom`, `callbackUrl`, `SETUP_GUIDES` (pure, unit tested); more scope labels.
- `settings/integrations/page.tsx`: members now see the cards (read only) instead of a lone note; the redirect URL is built from the request's own host (`x-forwarded-host`/`host`, validated, default `https://dashboard.mytheus.cloud`).
- `settings/integrations/loading.tsx`: the skeleton in the new shape; the skeleton pair in `tests/e2e/skeletons/WorkPairs.tsx` now mounts the real panel.
- Tests: `tests/integrations.test.ts` (+4), `tests/e2e/integrations-a11y.spec.tsx` (axe in both themes for every state and the dialog, keyboard trap and Escape, the form fields Connect/Disconnect submit, grid at 360/768/1024/1440/1600), `tests/e2e/integrations-shots.spec.tsx` (screenshots, skipped unless `INTEGRATIONS_SHOTS_DIR` is set). The CT config aliases `@/components/admin/FormSubmit` to a stub, because React 18 (what CT mounts) has no `useFormStatus`.

### mytheus-platform (`feat/push-integrations`, worktree `wt/integ-platform`)

- `scripts/push-secrets.sh integrations [--dry-run]` as in the owner guide. The mode is parsed apart from the shared one-line `case`, so it merges cleanly with `feat/meeting-invites` (checked with `git merge-tree`).
- `env/prod.env.example`: `OAUTH_REDIRECT_BASE`, `OAUTH_STATE_SECRET`, the Google, Slack and Jira keys (HubSpot, Notion and Confluence were already listed), and the redirect URL pattern.

## Deviations, and why

- **Card header:** icon and chip on the top row, the name under them, rather than icon, name and chip on one line. At three columns in the 880px panel a card is about 280px wide; "Google Calendar" plus "Ready to connect" does not fit on one line, and wrapping it would give cards of different shapes.
- **Error chip is amber, not red:** the red chip on its soft fill measures 4.49:1 in the light theme (axe needs 4.5). Amber (attention) is within the frozen colour language and passes in both themes.
- **Members:** integration-service's list is admin-only, so a member's page does not ask for it and cannot know each provider's state. Members see every card with "Managed by admins" and "Ask an admin to set this up".
- **Project tracker card:** a Linear connection (not built, but allowed by the canon enum) blocks Jira with "Tracker in use" and a reason, as before.
- **`OAUTH_STATE_SECRET`, `APP_BASE_URL`, `OAUTH_REDIRECT_BASE`:** none was in `prod.env.example`; without the first, Connect fails with 503, and without the second the redirect would default to `http://localhost:3000`. The integrations mode sets them only when unset, so a value already on the server is kept.

## Open gaps (owner steps)

- Create the provider apps and fill `mytheus-backend/.env` (no provider keys are there today: the dry run lists all 13 names as missing).
- Run `scripts/push-secrets.sh integrations`, redeploy mytheus-backend, then merge and deploy the UI branch.
- Google: an app in Testing mode only lets listed test users connect; publish the consent screen (Drive read-only is a restricted scope and needs Google's verification for external users).

## Verification (2026-11-12, local, worktrees above)

UI, from `wt/integ-ui`:

- `npx tsc --noEmit -p .`: clean.
- `npx next lint`: no warnings or errors.
- `npm test`: 546 pass, 0 fail.
- `CT_PORT=3641 npm run test:a11y -- --workers=4`: 548 passed, 2 failed under load (`briefing-a11y` "every state reads clearly (light)", `picker-a11y` "+N overflow"), 452 skipped (screenshot specs). Both failures are outside this change and pass when re-run alone (`... test:a11y -- briefing-a11y picker-a11y --workers=4`: 32 of 32). `integrations-a11y`: 26 of 26 (78 of 78 with `--repeat-each=3`). The integrations skeleton pair, measured: `SKELETON_SHOTS_DIR=<dir> CT_PORT=3642 npx playwright test -c tests/e2e/playwright-ct.config.ts -g "work settings-integrations" --workers=4`, 4 of 4 (skeleton busy and quiet in both themes; skeleton matches the page at 1440 and 375 px).
- `npx -y node@20 ./node_modules/next/dist/bin/next build` with `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` exported in the shell only: exit 0 (`/settings/integrations` 8.35 kB).
- Re-verified after rebasing onto main with the Myte rename (UI #69): tsc, lint, `npm test` (546 pass), `integrations-a11y` 26 of 26, the skeleton pair 4 of 4, `next build` exit 0.
- UI commit `5ce095a` (rebased); platform commits `7c8e0d2` and the doc-path fix after it.

Platform, from `wt/integ-platform`: `bash -n scripts/push-secrets.sh`; `scripts/push-secrets.sh integrations --dry-run` against the real `mytheus-backend/.env` (all names missing, nothing printed but names) and against a scratch `.env` (partial Slack reported as incomplete); the remote script run locally against a scratch directory (keys replaced, existing `APP_BASE_URL` kept, state secret generated). The VPS was not touched.
