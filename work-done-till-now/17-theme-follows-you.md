# 17. The colour theme survives navigation, follows the system, and reaches the auth pages

Status: `DONE WITH GAPS` (built and verified locally on 2026-11-12; pushed to mytheus-ui `main` as `fe48b5f`; not deployed). Not a plan band: a fix to the app's light/dark theme.

## Why

The owner reported three things:

1. In dark mode, pressing the workspace switcher in the sidebar (it goes to `/onboarding/organization?switch=1`) showed a light page, and coming back to the app it stayed light.
2. The auth pages (sign-in, sign-up, onboarding, SSO callback) had no dark mode.
3. The theme should follow the system setting by default.

## Root cause

The theme was written to the `.mx-app` root by an inline `<script>` (`THEME_INIT`) placed inside that root, in five places (AppShell, the admin and call layouts, the share page, the OAuth consent frame). That script runs when the HTML comes from the server, but React does not run an inline script it renders during a client-side navigation. Coming back from the workspace page is a client navigation (`router.replace`), so the new `.mx-app` root had no `data-theme`, and no `data-theme` means light. The saved choice in `localStorage` (`mx-theme`) was never lost; it was just never applied.

The auth pages sit under `.mx-auth`, and the dark token set only existed under `.mx-app[data-theme="dark"]`, so they had no dark mode at all.

Point 3 already held for a first visit: no saved choice means "system". The theme just did not follow a change of the OS setting while the page was open.

## What landed (mytheus-ui `fe48b5f`)

- `src/components/app/theme-init.ts`: `THEME_INIT` now writes the resolved theme to `<html data-theme>`. It runs once, in the root layout's `<head>`, before first paint. `<html>` is never re-rendered by a route change, so the theme survives every navigation. The five per-root copies are removed.
- `src/lib/prefs.ts`: `applyTheme()` writes to `<html>`; `setThemeChoice` uses it; `watchTheme()` follows the OS setting live while the choice is "system", and a choice made in another tab.
- `src/components/app/ThemeSync.tsx` (new): mounted once in the root layout; runs `watchTheme()`.
- `src/components/app/ThemeToggle.tsx`: reads the attribute from `<html>` and watches it, so the icon follows an OS flip.
- `src/styles/app-theme.css`: the dark token set applies under `:root[data-theme="dark"] :is(.mx-app, .mx-auth)`, plus the old `.mx-app[data-theme="dark"]` form for the component-test harnesses. In dark mode the page canvas behind the app and auth pages is dark too, so overscroll is not a light strip. `src/styles/meeting.css` and the scrollbar tokens in `globals.css` follow the same rule.
- The auth surfaces (`AuthShell`, `SsoCallback`, `OnboardingLoading`) import `app-theme.css` so the dark set is loaded on those routes.

## Deviations, and why

- **The marketing site stays light.** The marketing site has no dark token set; it reads the same variable names but sits outside `.mx-app` and `.mx-auth`, so `<html data-theme="dark">` changes nothing there. A dark marketing site would be a design task, not this fix.
- **The toggle still saves an explicit choice.** Pressing the sun/moon saves `light` or `dark`; Settings > Appearance has "System" to go back to following the OS. That is the usual pattern, and it was already there.

## Open gaps

- The exact reported path (signed in, dark, workspace switcher, back) was not clicked through: the local browser had no signed-in session. The mechanism was verified instead (below): a client-side navigation between auth pages keeps `<html data-theme>`, and the app shell reads the same attribute.
- Two unit tests in `tests/skeletons.test.ts` fail on `main` with or without this change (missing `loading.tsx` for `(auth)/onboarding/organization` and the two call routes; title-row checks). They are not caused by this fix.

## Verification (2026-11-12, local, dev server on :3000)

- `npx tsc --noEmit -p .`: clean (after `npm install`, which the pulled `sonner` dependency needed).
- `npx next lint`: no warnings or errors.
- `npm test`: 544 pass, 2 fail (the two pre-existing skeleton tests above; they fail on clean `main` too).
- In the browser pane:
  - OS dark, nothing saved: `/sign-in` has `<html data-theme="dark">`, `.mx-auth` background `rgb(35, 33, 32)`, body `rgb(26, 24, 22)`. Screenshot checked: the form, the brand panel and the doodle meeting all read in dark.
  - Clicking "Create an account" (a client-side navigation, checked with a window marker): `/sign-up` stays dark.
  - Switching the OS to light with nothing saved: the page flips to light live (`.mx-auth` background white).
  - Saving `dark` while the OS is light: dark at once, and dark again on a full reload of `/sign-up`.
  - The homepage with `<html data-theme="dark">`: body stays `rgb(240, 238, 230)` (unchanged).
  - No console errors.
