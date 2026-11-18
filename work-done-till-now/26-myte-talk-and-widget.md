# 26. Your Myte talks you through it, and the Android widget

Not a plan band. Status: **DONE WITH GAPS** (no device run yet). Last verification: 2026-11-18, `mytheus-mobile` only.

The owner asked for a small scope: no integrations and no further web-parity screens on the phone; instead a home-screen widget showing "our Myte" which, when tapped, talks the person through their decisions and their own tasks. This file records what was built for that in `mytheus-mobile`, the review that followed, and the fixes made from it.

The decisions are recorded in `mytheus-mobile` `system.md` section 15: **N28** (the talk screen, the signed-out return and the content-free widget) and **N29** (the widget's library, manifest, permissions and storage). No canon name, route, table, topic or event was added; the screen reads only calls the app already made, so no ADR was needed and `_canon/` is unchanged.

## 1. What landed in mytheus-mobile

### The talk screen

- `app/(app)/myte/brief.tsx`, path `/myte/brief`, deep link `mytheus://myte/brief`. Opened from Home's new "Talk me through it" card (`app/(app)/(tabs)/index.tsx`, `MyteTalkCard`) and from the widget.
- Reads only existing hooks (`src/myte/useMyteTalk.ts`): `GET /api/v1/decisions`, `GET /api/v1/actions/mine?status=confirmed`, `GET /users/me`, and the briefing preferences. The contract snapshot in `__tests__/api-client.test.ts` is unchanged.
- The script is pure (`src/lib/myte-talk.ts`, `composeMyteTalk`): a hello in the Myte's own name, up to 5 confirmed decisions from the last 14 days, up to 5 open tasks soonest due first, each item one sentence cut at 160 characters, and a line when one list could not be read.
- It speaks through the existing briefing player (`src/briefing/useBriefingPlayer.ts`) and the Kokoro voice engine (`src/voice/`), so the voice, the pace and the lock-screen player are the same as a briefing's.

### Signed-out launch

- `signInReturn` in `src/lib/routes.ts` is an allowlist of exactly `/myte/brief`. The workspace shell (`app/(app)/_layout.tsx`) redirects a signed-out launch of that path to `/sign-in?next=/myte/brief`; the auth layout (`app/(auth)/_layout.tsx`) returns only to an allowed `next`, else Home.

### The widget (Android only)

- `src/widget/`: `persona.ts` (pure), `MyteWidget.tsx` (the widget tree), `store.ts` (expo-secure-store), `sync.tsx` (`<MyteWidgetSync />`, mounted in the workspace shell), `task-handler.tsx` and `register.ts` (the headless task, registered last from `src/observability/entry.ts`).
- Library `react-native-android-widget` 0.26.0, pinned exact in `package.json`, declared through its config plugin in `app.json`.
- Content-free: the Myte's doodle face, its name and the fixed line "Tap and I'll talk you through your decisions and tasks." The whole card opens `mytheus://myte/brief`. Only `{ name, seed }` is stored, in secure-store, and it is cleared on sign-out or leaving the workspace.

## 2. Review findings and the fixes (2026-11-18)

An independent review passed the core checks (content-free widget, sound deep-link allowlist, no `org_id`, entry order, N28 and N29 present, tests asserting real behaviour, nothing out of scope) and raised the items below. Each was fixed as follows.

1. **Typed routes (medium, regression).** With `experiments.typedRoutes` on, a developer who has run `expo start` gets `.expo/types/router.d.ts`, and `npx tsc --noEmit` then failed with TS2322 because `signInReturn` returned `string`. Fix: `signInReturn` now returns the literal union `SignInReturn` (`'/myte/brief'`), derived from the allowlist itself. Verified by generating `router.d.ts` with expo-router's own generator (`getTypedRoutesDeclarationFile`): the old return type reproduces TS2322 and the new one passes; it also passes without the generated file, as CI runs it.
2. **Audio held outside the screen (medium).** On blur the screen only paused, so the background session and the lock-screen player stayed active under a pushed decision or task screen. Fix: a new `release()` on `VoiceEngine` (stop, clear the lock-screen controls, remove the expo-audio player, keep the cached audio) and on `useBriefingPlayer` (the same, plus letting go of the background session). The talk screen calls it on blur and on Stop.
3. **Two voices at once (low-medium).** Home's briefing card kept playing under the talk screen, and the talk screen's unmount turned background audio off under it. Fix in `useBriefingPlayer`: one voice at a time (a player that starts pauses the one that spoke before it, which lets go of its session) and the background session is held per player and released only when the last holder lets go.
4. **Undeclared WorkManager permissions (low-medium).** `androidx.work:work-runtime:2.12.0` comes in only through the widget library, which runs its headless task through WorkManager. The 2.12.0 AAR manifest was downloaded from Google's Maven and read: `WAKE_LOCK`, `ACCESS_NETWORK_STATE`, `RECEIVE_BOOT_COMPLETED`, `FOREGROUND_SERVICE`, and `RescheduleReceiver` for `BOOT_COMPLETED` shipped disabled. Decision: not blocked, because WorkManager needs them and blocking would break the widget's redraws; declared in `app.json` `android.permissions` and recorded in N29 and `system.md` section 12. Whether the process is started at boot is inferred, not observed.
5. **Widget name differed from the app's (low).** `widgetPersona` now takes the Clerk user and falls back to the Clerk first name when identity's `display_name` is null, exactly as Home and the talk screen do.
6. **Caption and highlight drift (low).** The screen now keeps the lines as they were when the voice started (`said`) and draws the caption and the lit row from those, so a refresh while speaking cannot point them at another line. A test now covers the highlight and the refresh case.
7. **Spoken text in the cache (low, inherited from N25).** Each spoken line is an unencrypted MP3 in `Paths.cache/voice`. Fix: the first write in each process empties that directory of what a killed process left (`src/voice/useVoice.ts`); engines still delete their own files when their screen closes. N28 now states this and why it is accepted (transient playback buffering in the app-private, OS-clearable cache, never read back as data).
8. **Housekeeping.** The `dist/` folder written by `npx expo export` was removed after each export. This file is the progress narrative the review found missing.

## 3. Deviations, and why

- **iOS has no widget.** A WidgetKit extension needs a Swift target, an App Group and its own config plugin; deferred (N29).
- **A task opens the task list**, not a single task: the phone has no single-task screen (N28).
- **Stop releases rather than pauses.** "Talk me through it again" restarts from the top anyway, so there is nothing to resume, and releasing keeps the rule that the screen holds no audio it is not using.
- **`npm ci --dry-run` warnings are left as they are.** Both predate this work (the same versions are in the committed lockfile): Clerk's peer range does not include the `react` and `react-dom` 19.1.0 that Expo SDK 54 pins, and `machina@7.0.1` (through `livekit-client`) asks for Node 22.22 or later while this machine has 22.16.0. CI installs the latest Node 22, which satisfies it. Neither is a failure (exit 0), and fixing them means a React bump off the Expo SDK pin or a Node upgrade on the machine, both outside this scope.

## 4. Open gaps

- Not run on a device: a cold-start and a warm deep link, the `?next` round trip through sign-in, the widget drawing on a launcher, the Gradle manifest merge, and the audio release on the lock screen.
- It needs a new development or store build and a runtime-version bump before release (a new native module and config plugin); it must not ship as an OTA update.
- CI does not generate typed routes, so a typed-routes error is caught only on a machine that has run `expo start`.

## 5. Verification

Run in `mytheus-mobile` with Node 22.16.0 and npm 11.7.0 on 2026-11-18, output observed:

- `npx tsc --noEmit`: exit 0, both with the generated `.expo/types/router.d.ts` present and without it.
- `npm run lint`: exit 0.
- `npm test`: 23 suites, 230 tests passed.
- `npx expo export --platform android`: exit 0 (one Android bundle); `dist/` removed afterwards.
- `npm ci --dry-run`: exit 0, with the ERESOLVE peer warnings for `@clerk/react` and `@clerk/shared` and the EBADENGINE warning for `machina@7.0.1` described in section 3.
- The fixes for findings 3 and 6 were checked against their tests: with each fix reverted, its new test fails.

## 6. Widget pass: extra verification and what remains

Added for the mobile pass as a whole (talk screen plus Android widget). Everything below was observed in `mytheus-mobile` with Node 22.16.0 and npm 11.7.0 unless marked as reported.

Widget build checks:

- `react-native-android-widget` is pinned at exactly 0.26.0. Its peer range is `expo >=54.0.0` and its gradle file builds the New Architecture spec when `newArchEnabled=true`. Its README says nothing about SDK or architecture, so support is read from the peer range and the gradle file, not from a statement by the authors.
- `npx expo prebuild --platform android --no-install` on a scratch copy finished cleanly. The generated manifest has the `MyteWidget` receiver (label "Your Myte"), the provider XML (180dp by 110dp, 3x2 cells, `updatePeriodMillis` 0) and the `mytheus` scheme intent filter. The scratch copy was deleted; no `android/` or `ios/` folder was created in the repo.
- `npx expo export --platform android` produced one Hermes bundle (10.3 MB) containing `RNWidgetBackgroundTask`, `mytheus.widget-persona`, `mytheus://myte/brief` and `MyteWidget`. `npx expo export --platform ios` also bundles (run by the reviewer).
- `__tests__/widget.test.tsx` has 17 tests (persona, SVG determinism, handler on ADDED/UPDATE/RESIZED and never on DELETED, only the name and the prompt as text, click URI, Android-only registration, sync write and clear). The full suite was 225 tests before the review fixes and 230 after.

Deviations specific to the widget, with the reason:

- **Content-free.** No decisions, tasks, counts or titles are rendered or stored, only `{ name, seed }`. A home screen is visible to anyone holding the phone, and a background task has no Clerk token and would show stale counts.
- **Android only.** A WidgetKit extension needs a Swift target, an App Group and its own config plugin; deferred.
- **Tap opens the app.** The widget never speaks or fetches; speech starts on the talk screen after the data loads.
- **Lazy `require()` instead of `import()`** in `sync.tsx`, because Jest here cannot run a dynamic `import()`. iOS and Jest still never load the library.
- **Doodle SVG** reuses the existing `doodleSvgString` in `src/lib/doodle.ts` rather than a second copy of the drawing.

Gates not run, stated plainly:

- `npm run generate:api:check` was not run (it needs `MYTHEUS_OPENAPI_DIR`; no API types changed).
- No gate failed in the final runs. One earlier lint run had 2 `react-hooks/exhaustive-deps` warnings in `useMyteTalk.ts`, fixed before the recorded runs. The review's one failing check (TS2322 with generated typed routes) is fixed and re-verified (section 2).

Still unresolved or unchecked:

- **iOS widget:** not built.
- **On-device test:** nothing has run on a device or emulator. Unseen: the widget drawing and the SVG face on a launcher, the redraw after a reboot from the Keystore, a cold and a warm tap, the signed-out `?next` round trip, and lock-screen audio release.
- **New development build and runtime-version bump:** required before release and not done. `app.json` has no runtime-version policy. This must not ship as an OTA update.
- **Native merge:** the library's exported `RNWidgetImageProvider` (hardened per the review's reading of its source) and WorkManager's boot receiver are read from manifests, not confirmed in a Gradle-merged APK.
- **Autoplay over a live call or recording** is not guarded; nothing exposes that state yet.
- **Kokoro rate limit** (30 a minute): one talk is about 13 to 15 requests, so repeated widget taps can reach a 429 and show the "Voice is unavailable" note with the text kept.
- **Spoken month names** from `dueLabel` (for example "3 Nov") have not been checked against Kokoro's pronunciation.
- "Personal tasks" means confirmed actions assigned to the person; private promises and status requests are not spoken.
