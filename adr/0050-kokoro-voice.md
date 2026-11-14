# ADR-0050: Every Myte speaks with Kokoro-82M, self-hosted

## Related documents

- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md): `kokoro-tts` (a third-party model server, no data) and core-api's `/api/v1/voice/*`.
- [ADR-0047](0047-ask-myte.md), [ADR-0049](0049-ask-myte-v2.md) (Ask Myte read aloud and voice mode), the briefing voice in core-api `briefing_audio`.

## Status

**Accepted** (2026-11-14).

## Context

A Myte spoke in three places: Ask Myte's read aloud, Ask Myte's voice mode, and the after-meeting briefing (web and phone). All of them used the device's built-in voice (the browser's `speechSynthesis`, `expo-speech` on the phone), so a Myte sounded different on every device and often robotic. A hosted voice (ElevenLabs) existed for briefing files only, not connected. The owner asked for Kokoro-82M, deployed on the VPS, as the voice everywhere agents speak, with a few good voices to choose from.

## Decision

1. **D1 Self-hosted model.** `kokoro-tts` runs Kokoro-82M (open-weight, Apache-2.0) through Kokoro-FastAPI on CPU (`ghcr.io/remsky/kokoro-fastapi-cpu`, pinned), internal only, capped at 2 CPUs and 2 GB, health-checked. No text leaves the server.
2. **D2 One voice per Myte.** core-api's TTS provider defaults to Kokoro (`TTS_PROVIDER=kokoro`, `KOKORO_URL`). The person's choice, `briefing_preferences.server_voice`, is their Myte's voice everywhere; with none chosen, the Myte's personality picks one. A curated set of Kokoro's best-graded English voices is offered: Heart, Bella, Kore, Aoede (American, female), Michael, Fenrir, Puck (American, male), Emma, George (British). Other briefing languages use that language's own Kokoro voice.
3. **D3 Routes (core-api).** `GET /api/v1/voice/voices`, `PUT /api/v1/voice/preference {voice}`, `POST /api/v1/voice/speech {text 1..1500, voice?, speed 0.7..1.4, language?}` streaming `audio/mpeg`, 30 per minute per person (`nexus:{org_id}:ratelimit:voice:{user_id}:{minute}`), 503 `voice_not_connected` when the service is off or unreachable.
4. **D4 No device voice.** The web app and the phone play server audio for every Myte utterance (sentence by sentence, so the first sound comes quickly). When the server voice is unavailable they show the text and say so; they do not fall back to the built-in voice.

## Consequences

- One recognisable voice per Myte on every device; no per-character API cost.
- CPU load on the VPS grows with speech; the cap and the per-person limit bound it.
- ElevenLabs remains selectable (`TTS_PROVIDER=elevenlabs`) for briefing files.
