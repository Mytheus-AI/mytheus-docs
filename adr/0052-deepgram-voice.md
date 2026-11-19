# ADR-0052: Every Myte speaks with Deepgram Aura-2

## Related documents

- [ADR-0050](0050-kokoro-voice.md): this ADR replaces its D1 (the self-hosted Kokoro model) and the provider and voice list in D2. D2's "one voice per Myte", D3 (the routes) and D4 (no device voice) stand.
- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md): core-api's `/api/v1/voice/*` row; `kokoro-tts` is gone.
- [work-done-till-now/27-deepgram-voice.md](../work-done-till-now/27-deepgram-voice.md): what was built and verified.

## Status

**Accepted** (2026-11-19).

## Context

ADR-0050 gave every Myte one server voice: Kokoro-82M on the VPS's CPU (`kokoro-tts`, capped at 4 CPUs). It was too slow for conversation. Kokoro renders a whole piece before sending any of it, so Ask Myte's voice mode waited seconds for each sentence. A briefing of about 1,400 characters took close to its own length to render, even with three pieces in parallel. The owner asked for Deepgram, which Mytheus already uses for live transcription, to replace Kokoro everywhere a Myte speaks.

Deepgram's Aura-2 was measured with the existing key before this decision (2026-11-19, REST `POST /v1/speak`, `encoding=mp3`). Results:

- First audio arrived in about 0.35 s, and the MP3 streamed in hundreds of chunks as it rendered.
- A 1,400-character briefing paragraph took 6.8 s in total.
- The limit is 2,000 characters per request (413 above it).
- `speed` (0.7 to 1.5) works on the English and Spanish voices; the German, French and Japanese voices answer 400 to it.

## Decision

1. **D1 Hosted model.** core-api's TTS provider defaults to Deepgram Aura-2 (`TTS_PROVIDER=deepgram`, key `DEEPGRAM_API_KEY` in `prod.env`, base `DEEPGRAM_API_URL`, default `https://api.deepgram.com`). core-api calls `POST /v1/speak` itself and streams the MP3 straight through to the app, so there is no voice container. Text over 1,900 characters goes in pieces. Live speech is sent whole (the apps send 1,500 characters at most). Briefing files render three pieces at a time and are joined in order. Without the key there is no server voice, and the apps show text only (ADR-0050 D4).
2. **D2 Voices.** The curated set is nine Aura-2 English voices, one for each Kokoro voice ADR-0050 offered, with the same spread of accents, genders and tone:
   - American female: Thalia, Helena, Callista, Harmonia.
   - American male: Orpheus, Zeus, Apollo.
   - British: Pandora (female), Draco (male).

   Personality defaults: balanced Thalia, friendly Helena, concise Callista, direct Orpheus, detailed Pandora. A Kokoro id that is still stored in `briefing_preferences.server_voice` maps to its counterpart (Heart to Thalia, Bella to Helena, Kore to Callista, Aoede to Harmonia, Michael to Orpheus, Fenrir to Zeus, Puck to Apollo, Emma to Pandora, George to Draco). It is read that way everywhere and saved that way on the next change, so no data migration is needed.
3. **D3 Languages.** Spanish, French, German, Italian, Dutch and Japanese speak with that language's own Aura-2 voice (Selena, Agathe, Elara, Livia, Daphne, Izanami). Aura-2 has no Hindi, Portuguese, Korean, Chinese or Arabic voice. Live speech in those languages answers 503 `voice_not_connected`, and a briefing's audio is marked failed. The apps then show the text, as ADR-0050 D4 already requires. An English voice never reads them.
4. **D4 Speed.** `speed` is sent only with English and Spanish voices, so a person's pace setting never breaks speech in another language.
5. **D5 The old voice.** `kokoro-tts` is removed from `deploy/compose.prod.yml`, which frees its 4 CPUs and 3 GB on the VPS. `TTS_PROVIDER=kokoro` still works if the service is restored from git history. `elevenlabs` stays available for briefing files.

## Consequences

- A Myte starts speaking in about a third of a second instead of seconds, and briefing audio is ready sooner.
- Speech text now leaves the VPS for Deepgram, the same processor that already receives meeting audio for transcription. Deepgram bills per character. The per-person limit (60 a minute) and the phrase cache for short repeated phrases bound the cost, and a Redis outage still allows speech rather than silencing every Myte.
- Five briefing languages lose their server voice and fall back to text.
- One key now serves two services: transcription (`prod.transcription.env`) and core-api (`prod.env`). Rotating it means pushing both (`scripts/push-secrets.sh all` and `scripts/push-secrets.sh voice`).
