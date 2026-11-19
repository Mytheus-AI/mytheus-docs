# 27. Every Myte speaks with Deepgram Aura-2

Not a plan band ([ADR-0052](../adr/0052-deepgram-voice.md)). Status: **DONE WITH GAPS** (live in production, follow-ups deployed; not yet heard on a phone). Last verification: 2026-11-19, in production.

The owner found the self-hosted Kokoro voice (ADR-0050) too slow and asked for Deepgram to replace it "everywhere and no questions asked". The decision, the measurements behind it and its consequences are in ADR-0052, which supersedes ADR-0050 D1 and the provider part of D2. The canon row for core-api in [_canon/01-service-catalog.md](../_canon/01-service-catalog.md) now names Deepgram Aura-2 instead of `kokoro-tts`. No table, topic, route or event changed.

Speech to text (Talk mode's `POST /api/v1/voice/transcribe`, faster-whisper in `whisper-stt`) is unchanged: the request was for text to speech only.

## 1. What landed, per repo

### mytheus-backend (PR #47, merged and deployed 2026-11-19)

- `services/core_api/briefing_audio.py`:
  - `TTS_PROVIDER` now defaults to `deepgram` (`kokoro`, `elevenlabs` and `none` stay selectable). New settings: `DEEPGRAM_API_KEY` and `DEEPGRAM_API_URL` (default `https://api.deepgram.com`).
  - `deepgram_stream` makes one `POST /v1/speak` (`model=<voice>`, `encoding=mp3`, `Authorization: Token ...`) and yields the MP3 as it arrives. `speed` is sent only for `-en` and `-es` voices. A non-200 becomes a `TtsError` carrying Deepgram's `err_code`, never the key.
  - `DeepgramSpeaker` serves live speech. `DeepgramS3` renders briefing files in pieces of at most 1,900 characters, three at a time, joined in order.
  - New helpers:
    - `DEEPGRAM_VOICES`: nine curated voices, with personality defaults.
    - `_DEEPGRAM_BY_LANGUAGE`: one voice each for es, fr, de, it, nl and ja.
    - `canonical_voice` and `known_voice`: map a stored Kokoro id to its Deepgram counterpart.
    - `speaks(language)`.
- `services/core_api/routes/voice.py`:
  - Speech in a language with no Aura-2 voice answers 503 `voice_not_connected` before any audio. An English voice never reads Hindi text.
  - The listed `chosen` voice and `PUT /api/v1/voice/preference` both go through `known_voice`.
  - The rate-limit note now says the provider bills per character.
- `services/core_api/routes/briefings.py` validates and stores `server_voice` through `known_voice`. `briefings.settings_view` shows a stored Kokoro id as its Deepgram counterpart.
- `.env.example`: `TTS_PROVIDER`, `DEEPGRAM_API_KEY`, `DEEPGRAM_API_URL`.
- `tests/test_briefings.py`. The new tests:
  - The Deepgram default, its voices and the language mapping.
  - Kokoro is still selectable.
  - The request shape checked with an `httpx.MockTransport`: path, params, `speed` only for en and es, the auth header, a 401 that does not leak the key, an unreachable host.
  - Long briefings split under the limit and joined in order, and Hindi refused.

  Changed tests:
  - The render test now proves a stored `bm_george` speaks as `aura-2-draco-en`.
  - The voice route test now covers a legacy id sent by an app and the Hindi 503.

### mytheus-ui (PR #94, merged and deployed 2026-11-19)

- `src/lib/voice/catalog.ts`: `VOICE_ID` accepted only Kokoro-shaped ids (`af_heart`). With it, every Deepgram voice would have been dropped from the catalogue and every preview refused with 422. It now accepts `aura-2-<name>-<lang>` as well as the Kokoro shape.
- Comments in the speech route, Ask Myte voice, voice mode and the voice picker now name the Aura-2 voice.
- Tests: `tests/voice-catalog.test.ts` covers Deepgram ids and near misses. The component fixtures (`tests/e2e/voice-fixture.ts`, the picker spec, the stubs, the fake speech route) use the Aura-2 voices.

### mytheus-mobile (PR #8, merged 2026-11-19)

- Comments and `system.md` N25 only. The phone never checked voice ids; it plays whatever core-api lists and streams.

### mytheus-platform (PR #29, merged and deployed 2026-11-19)

- `deploy/compose.prod.yml`: the `kokoro-tts` service is removed, which frees 4 CPUs and 3 GB. A note in its place says how to restore it, and that `docker rm -f mytheus-kokoro-tts-1` is needed once, because the deploy's `up -d` keeps orphans.
- `env/prod.env.example`: `DEEPGRAM_API_KEY` and a commented `TTS_PROVIDER`.
- `scripts/push-secrets.sh voice`: copies `DEEPGRAM_API_KEY` from the transcription service's `.env` into `prod.env` and writes `TTS_PROVIDER=deepgram`, so an older `kokoro` line cannot win. `--dry-run` prints names only.

### Follow-ups found in production (merged and deployed 2026-11-19)

- **mytheus-backend #48, connection reuse.** Production showed about 1.2 s to the first audio. A phrase-cache hit (which never reaches Deepgram) answered in about 330 ms, so roughly 900 ms was the Deepgram call. `deepgram_stream` opened a new `httpx.AsyncClient`, and so a new TLS connection, for every sentence. The fix keeps one pooled client per event loop, with 30 s keep-alive, and retries a dropped kept-alive connection once, only before any audio. Measured against the live API: 1,234 ms for the first sentence, then 392, 386 and 375 ms.
- **mytheus-ui #95, picker copy.** The voice picker still said "Spoken by Mytheus's own voice server. What your Myte says never leaves it." That has been false since this change. The line now says the voice comes from Deepgram and that the Myte's words are sent there to be read aloud.

## 2. Deviations, and why

- **Kokoro kept as an option, not deleted.** The owner said "replace". The container is gone and the default is Deepgram. The Kokoro code path, about 60 lines that are already tested, stays behind `TTS_PROVIDER=kokoro` as a zero-cost fallback if the Deepgram credit runs out.
- **No data migration for stored Kokoro voice ids.** They map at read time and are rewritten on the next save, so a rollback to Kokoro still finds the original ids.
- **Five briefing languages lose their server voice** (hi, pt, ko, zh, ar). Aura-2 has no voice for them. Text only (ADR-0050 D4) is better than an English voice reading them.
- **The key lives in `prod.env`**, as `RESEND_API_KEY` and `S3_*` do for core-api, not in a new core-api-only file. That follows the existing pattern; every backend container reads `prod.env`.

## 3. Open gaps

- The orphaned `kokoro-tts` container: the owner ran `ssh root@187.127.190.246 docker rm -f mytheus-kokoro-tts-1`, which answered "No such container". Under the deploy's project name (`mytheus`) it is gone or never ran. Any differently named Kokoro container still on the host is unconfirmed (Claude's SSH is refused in auto mode).
- The Deepgram key now serves two services. A rotation must push both files.
- Not heard on a phone yet.

## 4. Verification

Production, 2026-11-19:
- The owner ran `scripts/push-secrets.sh voice`, which wrote 2 keys to `prod.env`. Backend #47 and UI #94 then deployed through their `main` CI, with every job green.
- Signed in at `dashboard.mytheus.cloud/agent`, the voice picker listed the nine Aura-2 voices; the personality match was Pandora (detailed). Preview Thalia returned 200 from `POST /api/voice/speech`.
- From the page, `aura-2-thalia-en` returned 200 `audio/mpeg`, streamed MP3 frames (`FF F3`).
- First audio for fresh phrases took 1,144 to 1,247 ms. A repeated phrase (a phrase-cache hit) took 307 to 352 ms.
- Hindi returned 503 `voice_not_connected`, as designed.
- After platform #29 deployed, Draco (English) and Spanish both returned 200 `audio/mpeg`.
- After backend #48 (connection reuse) and UI #95 deployed: five fresh phrases took 1,378 ms to the first audio for the first one (a new connection after the deploy), then 718, 719, 725 and 738 ms. Before the fix it was 1,144 to 1,247 ms every time. The picker shows the corrected copy.

Local:

- A live Deepgram call with the existing key (2026-11-19, a scratch script, never committed):
  - `aura-2-thalia-en`: 200 `audio/mpeg`, first bytes in 0.35 s, streamed in 46 to 882 chunks.
  - `speed=1.2` and `1.4` on English, and `1.1` on Spanish: 200.
  - `speed` on de, fr and ja: 400 "Requested model does not support the 'speed' parameter".
  - French without speed: 200.
  - 2,001 characters: 413 "exceeds maximum character limit of 2000".
- `mytheus-backend`:
  - `uv run ruff check`, `black --check` and `mypy` on the changed files: clean.
  - `pytest tests/test_briefings.py` with `MYTHEUS_REQUIRE_DB=1` on a fresh `supabase/postgres:15.8.1.060`, bootstrapped as CI does: 25 passed.
  - The full suite: 1,832 passed, 3 failed. The failures were `test_cost_dashboard_reads_only_aggregate_tables`, `test_health_probes_services_and_reports_the_outbox` and `test_relay_real_bus_roundtrip`. All three fail the same way on unmodified `main` against the same local stack, so they are environmental.
- `mytheus-ui`:
  - `npm test`: 722 passed.
  - `tsc --noEmit`: clean.
  - `playwright test -c tests/e2e/playwright-ct.config.ts voice-picker briefing-a11y`: 35 passed.
  - `next lint`: one error, in `tests/e2e/stubs/clerk-errors.ts`. It is present on `main` and unrelated to this change.
- `mytheus-platform`: `docker compose -f deploy/compose.prod.yml config --quiet` passes. `bash -n scripts/push-secrets.sh` passes, and `push-secrets.sh voice --dry-run` prints the two key names.
