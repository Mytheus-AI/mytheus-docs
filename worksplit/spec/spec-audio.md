# BUILD SPEC — Audio capture and live transcription path

Scope: hops 1-4 of the meeting audio pipeline (browser mic → LiveKit SFU → server-side tap →
`/internal/audio` → `TranscriptionStream`). Read-only audit + task plan. No repo file modified.

---

## Part A — Reality check (each claim confirmed against source)

### Hop 1 — browser mic → LiveKit SFU: **ABSENT (confirmed, and worse than reported)**

`mytheus-ui/src/lib/realtime/useLiveKitRoom.ts:69-71` sets the initial status from the env var:

```ts
const configuredUrl = process.env.NEXT_PUBLIC_LIVEKIT_URL;
const [status, setStatus] = useState<LiveKitStatus>(
  configuredUrl ? "connecting" : "unconfigured",
);
```

and `:83` short-circuits the whole connect effect: `if (!configuredUrl) return;`. The module
docstring (`:15-21`) states plainly that `NEXT_PUBLIC_LIVEKIT_URL` "is unset in every
environment that exists today". Confirmed as described.

**Additional finding not in the audit.** Setting the env var alone would still capture no audio.
`grep -rn "setMicrophoneEnabled\|publishTrack\|createLocalAudioTrack\|getUserMedia\|audio: true"
mytheus-ui/src` returns **zero hits**. The `Room` is constructed at `:85` as
`new Room({ adaptiveStream: true, dynacast: true })` and `room.connect(...)` is called at `:120`;
nothing ever publishes a local track. The hook is a *subscriber and grid renderer*, not a
publisher. Hop 1 is therefore two gaps, not one: (a) no LiveKit URL, (b) no local-track publish
path, no device picker, no mute control, no permission-denied UI.

Server side, `mytheus-backend/services/meeting_service/routes.py:215` returns
`livekit_url=os.environ.get("LIVEKIT_URL", "")` — an empty string today, so the client falls back
to the (also unset) public env var.

### Hop 2 — LiveKit → server-side audio tap: **ABSENT (confirmed)**

`mytheus-backend/services/meeting_service/transport.py`:

- `:139-141` — `LiveKitTransport.subscribe_room_audio` body is
  `raise NotImplementedError("subscribe_room_audio lands with the transcription tap")`
- `:135-137` — `LiveKitTransport.start_egress` body is
  `raise NotImplementedError("start_egress lands with the nexus-recordings bucket (Task 4)")`
- Protocol declarations at `:54` and `:56-58`; `FakeTransport` counterparts at `:223-229`
  (returns `f"fake-egress-{org_id}-{meeting_id}"`) and `:231-233` (records the call, returns `None`).

**No caller anywhere.** A grep for `subscribe_room_audio|start_egress` across all repos returns
only these definitions plus prose in `mytheus-docs` (`10-realtime-and-transcription.md:87-88`,
`adr/0003-meeting-sdk-livekit.md:87-88`, `plan/plan-3-4-meeting-and-transcription.md:110-111`) and
the prior audit/verify files. Confirmed.

### Hop 3 — tap → transcription: **ABSENT (confirmed on all four sub-claims)**

`mytheus-realtime/services/transcription-service/src/transcription_service/main.py:119-124`:

```python
@app.websocket("/internal/audio")
async def internal_audio(websocket: WebSocket) -> None:
    # LiveKit server-side audio-hook target. Scaffold: accept then close cleanly so the transport
    # contract exists. The Deepgram WSS proxy + transcript_chunks persistence land later.
    await websocket.accept()
    await websocket.close()
```

- Accepts then immediately closes: confirmed, verbatim above.
- Zero callers: the only references to `/internal/audio` anywhere are this route, its own module
  docstring (`:3`, `:11`), `meeting_room_consumer.py:37` and `:235` (comments about sharing the
  event loop with it), `tests/test_healthz.py:17` (asserts the accept-then-close), and docs.
- No handshake validation: there is no `Authorization` read, no token verify, no query-parameter
  parse, no `meeting_id` — the function body is the four lines above.
- The three `LIVEKIT_*` settings are loaded and never read: `config.py` defines `livekit_url`,
  `livekit_api_key`, `livekit_api_secret` on `Settings` and populates them in
  `load_settings_from_env()`. Grep for `livekit` across
  `transcription-service/src` finds hits **only in `config.py`**. Confirmed dead.

### Hop 4 — tap → `SessionTracker`: **ABSENT, and architectural (confirmed)**

The stub never references `SessionTracker`; `main.py` builds one in `lifespan` (`:75`) and hands it
only to `run_consumer` (`:78-86`). The route closure captures nothing.

The deeper claim is correct and is the load-bearing one.
`providers/base.py` defines:

```python
class TranscriptionStream(Protocol):
    def utterances(self) -> AsyncIterator[Utterance]:
        """Yields interim and final `Utterance`s for the lifetime of the stream."""
        ...

    async def close(self) -> None: ...
```

Two methods. Both are **outbound**. There is no `send_audio`, no `write`, no `feed`, no sink of any
kind. `TranscriptionProvider.open_stream(*, org_id, meeting_id, region, diarize)` takes no audio
source either. `SessionTracker._pump` (`sessions.py`) only *drains* `stream.utterances()` into
`persistence.process_utterance`. `providers/fake.py::_FakeStream.utterances` yields from a canned
script; `providers/deepgram.py::_DeepgramStream` iterates `self._ws` and parses.

So the pipeline as built is: a stream is a *source of utterances that produces them from nowhere*.
Nothing in the type system, in `SessionTracker`, or in any provider can accept a byte of audio.
This is exactly as characterised: **an interface change, not a missing credential.** A Deepgram key
and a signed DPA would not move this by one line.

### Hops 5-9: **WIRED (confirmed)**

- `sessions.py::SessionTracker.on_participant_joined` / `on_participant_left` — real state machine
  with idempotent open/close, residency + PII-toggle reads, and `record_session_decision` audit in
  the same transaction.
- `meeting_room_consumer.py` — real `meeting-room-events` consumer, poll driven via
  `run_in_executor` (`:235` comment explains why).
- `persistence.py::process_utterance` — finals-only persistence + outbox row in one transaction.
- `outbox.py`, `events/transcript_chunk.py`, `captions.py`, `redaction.py`, `residency.py`,
  `tenancy.py`, `audit.py` — all real, all with tests.
- `transcript_chunks` exists as a range-partitioned table with RLS
  (`migrations/versions/be66698dbefb_create_transcript_chunks.py`), columns: `id, org_id,
  meeting_id, speaker_id, start_ms, end_ms, text, is_final, confidence, redacted`.
- `transcript-chunks` Avro (`mytheus-events/schemas/transcript-chunks/`) matches, with
  `redacted` carrying `"default": false` — the precedent for BACKWARD-safe field addition.
- Socket.io caption fan-out in `realtime-gateway`.

Confirmed: everything downstream of "a `TranscriptionStream` yields an `Utterance`" is real,
tested, and waiting. The product is missing only its front half.

### Deepgram gate: **CONFIRMED — genuine code, legal blocker**

`providers/deepgram.py` (133 lines) is real WSS client code: `websockets.connect` at `:122`,
Deepgram's documented streaming response shape parsed at `:43-58`, deterministic `chunk_id` via
`uuid.uuid5(NAMESPACE_URL, f"deepgram:{request_id}:{speaker}:{start_ms}:{end_ms}")` with two
recorded rounds of fixes behind that key choice. Not a stub.

`compliance.py` declares `COMPLIANCE_GATE_GREEN: Final[bool] = False` as a module-level `Final`
constant, and `assert_compliance_gate_green()` raises `ComplianceGateClosedError` unless it is
`True`. `DeepgramProvider.open_stream` calls it at `:117`, before `websockets.connect` at `:122`.
The module docstring is explicit that condition 1 is "a LEGAL artifact… no PR against this repo can
make condition 1 true", and that the constant is deliberately **not** an env var or DB row so that
deploy access cannot override a fact about a signed contract.

Confirmed exactly as stated. Conditions 2 (redaction) and 3 (residency) are code and are done.

---

## Part B — The blocker triage (the important part)

### BLOCKED ON LEGAL — signed Deepgram DPA/SCC

Exactly one thing: **opening a real Deepgram WSS session against real audio.** That is it. It
blocks production-quality live ASR. It does **not** block any of hops 1-4, because none of them are
Deepgram-shaped: they are LiveKit publish, LiveKit tap, an internal WS, and a Python Protocol.

### BLOCKED ON PROCUREMENT — hosted LiveKit

**Nothing. Stated plainly: a self-hosted LiveKit server in compose removes the procurement blocker
entirely, and there was never a real one for build purposes.**

The reasoning, and the caveats, honestly:

- `livekit/livekit-server` is Apache-2.0 and ships as a public container image. Run with
  `--dev` it uses the well-known dev credentials (`devkey` / `secret`) and needs no account, no
  signup, and no card. `LIVEKIT_URL=ws://livekit:7880`, `LIVEKIT_API_KEY=devkey`,
  `LIVEKIT_API_SECRET=secret` satisfies every consumer of those vars in this codebase, including
  `select_transport()` (`transport.py:250-264`), which switches off `LIVEKIT_API_KEY` alone.
- This is not a workaround against the architecture; it *is* the architecture.
  `10-realtime-and-transcription.md:70` and `:74` lock LiveKit specifically **because** it is
  self-hostable ("The self-hostable SFU is what makes `D1` = 216M minutes/month affordable; a
  per-participant-billed managed SDK does not survive the capacity math"). ADR-0003 names the
  self-host swap as Tech Decision 2.7. Building against self-hosted LiveKit is building against the
  intended production topology, not a stand-in for it.
- **Verified gap:** no compose file anywhere runs one. `mytheus-backend/docker-compose.yml` defines
  `postgres`, `redis`, `redpanda`, `temporal`, `temporal-ui` and nothing else.
  `mytheus-platform/deploy/compose.prod.yml` mentions LiveKit only in comments about which
  services hold `LIVEKIT_*` credentials. So the blocker is "nobody has added ~15 lines of YAML",
  which is a task, not a procurement dependency.
- **Honest caveats.** Self-hosted LiveKit in dev needs host networking or a correct
  `rtc.use_external_ip` / port-range config for WebRTC to traverse Docker's NAT — this is the one
  place a dev-compose LiveKit reliably costs an afternoon. Multi-participant testing across real
  networks eventually wants a TURN server. Neither is procurement; both are configuration.
  What LiveKit **Cloud** would buy is global relay, managed TURN, and SFU ops at `A5`/`A6` scale —
  a *production deployment* decision, deferrable indefinitely, and per ADR-0003 not the plan anyway.
- The egress tap (Task 4 below) additionally needs the `livekit/egress` container, which needs
  Redis — already in compose — and likewise no account.

**Conclusion: nothing in this spec is blocked on procurement.**

### ACTIONABLE TODAY with neither

Hops 1, 2, 3, and 4 in full — the entire vertical, end to end, carrying real microphone bytes from
a real browser through a real SFU to a real server-side tap into a real provider — provided the
provider at the far end is not Deepgram. Tasks 1-12 below are all in this category. The pipeline can
be made genuinely, demonstrably alive before Legal returns.

---

## Part C — Numbered task list

Sizes: **S** ≤ 1 day, **M** 1-2 days, **L** 3+ days (days given).
"Dependencies" are task numbers in this list.

---

### 1. Self-hosted LiveKit in local compose — **S** — deps: none

**Repos/files:** `mytheus-backend/docker-compose.yml`; `mytheus-backend/.env.example`;
`mytheus-realtime/docker-compose.yml`; `mytheus-ui/.env.example`;
`mytheus-docs/work-done-till-now/` band file.

Add a `livekit` service (`livekit/livekit-server:latest`, `--dev`, ports `7880`, `7881`, UDP
`50000-50100`) plus a `livekit.yaml` with `keys: {devkey: secret}`, webhook URL pointed at
`meeting-service`'s `/livekit/webhook`, and `rtc.use_external_ip: false` for Docker. Set
`LIVEKIT_URL` / `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` in the backend env and
`NEXT_PUBLIC_LIVEKIT_URL` in the UI env.

**Acceptance:** `select_transport()` returns `LiveKitTransport`, not `FakeTransport`;
`ensure_room` + `mint_join_token` succeed against the local server; a signed webhook reaches
`meeting-service`. This is the first time `LiveKitTransport` is ever executed — expect the
`with_ttl`/`WebhookReceiver` 1.1.1 adaptations at `transport.py:130` and `:147` to be validated
for real here.

---

### 2. Browser mic publish — **M** — deps: 1

**Repos/files:** `mytheus-ui/src/lib/realtime/useLiveKitRoom.ts`; the meeting-room component that
calls it; a new device-selection/mute control.

After `room.connect(...)` at `:120`, publish the local audio track
(`room.localParticipant.setMicrophoneEnabled(true)`). Add: `getUserMedia` permission request with a
denied/blocked UI state (a new `LiveKitStatus` variant — do not overload `"error"`), a mute/unmute
control bound to `setMicrophoneEnabled`, and a device picker. Keep the `"unconfigured"` degradation
path intact for environments without LiveKit — it is good design and Task 1 does not obsolete it.

**Acceptance:** two browser tabs join one `meeting_id`, each sees the other's participant tile, and
`room.remoteParticipants` shows a subscribed audio track. Mic permission denial renders a
distinguishable state.

**Note:** video publish is deliberately out of scope — the transcription vertical needs audio only,
and the participant grid already renders.

---

### 3. Spike: confirm LiveKit's WebSocket egress contract — **S** — deps: 1

**Repos/files:** throwaway script + a decision note in the band file.

Verified from the installed protobufs (`livekit-protocol 1.1.18`) that the surface exists:
`TrackEgressRequest.__slots__ == ("room_name", "track_id", "file", "websocket_url", "webhooks")`
with `file`/`websocket_url` as a `oneof` output; `StreamProtocol` enum contains `WEBSOCKET = 3`;
`EgressClient.start_track_egress` is at
`mytheus-backend/.venv/.../livekit/api/egress_service.py:79`. What the protobuf cannot tell us is
what the egress **binary** actually accepts and what it puts on the wire.

Confirm empirically, against the `livekit/egress` container: (a) that `websocket_url` is honoured
for a track egress; (b) the exact frame payload — PCM sample rate, channel count, bit depth, and
whether mute/unmute arrive as interleaved JSON text frames; (c) whether `RoomCompositeEgressRequest`
(which has `audio_only` and `audio_mixing`, including `DUAL_CHANNEL_AGENT`) can target a
`StreamOutput` with `protocol=WEBSOCKET`, or whether WebSocket output is track-egress-only.

(c) is the decision-relevant one: it determines whether the tap is one connection per meeting or one
per participant track. **Do this before Task 4** — it is the only task here whose outcome could
change the recommended design.

---

### 4. Hop 2 — the server-side tap — **L (4 days)** — deps: 1, 3

**Repos/files:** `mytheus-backend/services/meeting_service/transport.py` (replace the
`subscribe_room_audio` stub at `:139-141`); `webhook.py` (react to `track_published` /
`participant_joined`); `rooms.py`; `services/meeting_service/routes.py`; compose (`livekit/egress`
container); tests including `tests/test_livekit_import_boundary.py`.

#### The two approaches

**(A) Track subscription via a server-SDK participant.** `transcription-service` (or a new tap
worker) joins the room as a hidden participant using `livekit-rtc`, subscribes to remote audio
tracks, and receives decoded PCM frames in-process. This is how LiveKit Agents works.

**(B) Egress with a WebSocket output.** `meeting-service` calls `StartTrackEgress` with
`websocket_url` pointing at `transcription-service`'s `/internal/audio`. The separate
`livekit/egress` process pulls from the SFU and pushes audio to us over a WS we merely accept.

#### Comparison

| | (A) track subscription | (B) egress → WebSocket |
|---|---|---|
| LiveKit SDK location | `livekit-rtc` imported in `mytheus-realtime` | `livekit-api` in `meeting-service` only |
| ADR-0003 Decision 2 boundary | **violated** — needs an ADR amendment | **preserved** |
| Dependency weight | native wheel, per-arch, heavy | none new — `livekit-api 1.1.1` already installed |
| Matches `/internal/audio`'s declared shape | no — inverts it (we'd dial out, not accept in) | yes — it is literally the "audio hook" the docs describe |
| Matches docs' "mixed room audio (PCM) over an internal WS" (`plan-3-4:367`, `10-…:148`) | we would mix ourselves | yes if room-composite WS works (Task 3c); else per-track |
| Extra infrastructure | none | one `livekit/egress` container (needs Redis — already present) |
| Replica-affinity defect 8a | **solves it** — same process subscribes and tracks | does not solve it alone; see Task 6 |
| Failure blast radius | a tap crash is inside the pod holding 20k-stream state | egress is a supervised sibling process |
| Speaker attribution | per-track ⇒ true speaker identity, no diarization needed | per-track egress: same; composite: needs Deepgram diarization |

#### Recommendation: **(B) egress → WebSocket**, paired with Task 6.

Three reasons, in order of weight:

1. **It is the only option that does not require amending an accepted ADR.** ADR-0003 Decision 2
   makes `meeting-service` the sole importer of the LiveKit SDK, and
   `tests/test_livekit_import_boundary.py` enforces it mechanically. That boundary is what makes the
   self-host swap "a one-interface reimplementation instead of a codebase-wide migration"
   (`transport.py:1-8`). Approach (A) puts `livekit-rtc` in `mytheus-realtime` and breaks it. That
   is a canon-first decision requiring an ADR, not a build task — and this spec should not smuggle
   an architecture change in under a plumbing task.
2. **It is the interface that already exists.** `/internal/audio` is an inbound WebSocket
   *server*. Approach (B) fills it in; approach (A) makes it dead code and replaces it with an
   outbound dialer. Every doc — `10-realtime-and-transcription.md:51`, `:148`, `:152`,
   `plan-3-4:367` — describes an inbound hook. Building (B) is finishing the declared design;
   building (A) is redesigning it.
3. **`start_egress` and `subscribe_room_audio` collapse into one mechanism.** The brief notes only
   one of the two stubs needs building. Under (B) they become two calls against the same
   `EgressClient` — the tap is `StartTrackEgress(websocket_url=…)`, the recording is a file/S3
   egress. One integration, one operational surface, one thing to learn.

**The honest cost of choosing (B):** an extra container, and it does not by itself fix defect 8a
(recorded at `work-done-till-now/03-plan-3-4-meeting-and-transcription.md:177`) — the egress process
dials whatever replica the ingress picks, which need not be the one whose `SessionTracker` opened
the meeting's stream. Approach (A) gets that for free. That is a genuine advantage of (A) and the
strongest argument for it. Task 6 closes the gap for (B) at a cost of about a day, and closes it in
a way that is better than (A)'s: the session becomes owned by the connection that carries the audio,
which is the correct owner regardless of tap mechanism.

**Implementation.** Replace `subscribe_room_audio` with an implementation that calls
`api.EgressClient.start_track_egress(TrackEgressRequest(room_name=…, track_id=…,
websocket_url=…))`, where the URL carries a signed tap token (Task 5) and the `meeting_id`. Drive it
from `track_published` webhooks in `webhook.py`, and stop it on `track_unpublished` /
`participant_left` via `stop_egress`. Persist the egress id so a restart can reconcile. Keep
`FakeTransport.subscribe_room_audio` recording-only.

**Acceptance:** speaking into a browser tab causes bytes to arrive at `/internal/audio`; a
participant leaving stops the egress; a redelivered `track_published` does not start a second one.

---

### 5. Hop 3 — `/internal/audio`: handshake validation + frame protocol — **M** — deps: 3, 4

**Repos/files:**
`mytheus-realtime/services/transcription-service/src/transcription_service/main.py:119-124`
(replace the stub); a new `audio_in.py`; `config.py`; new tests.

Three things, all currently absent:

**(a) Handshake validation.** The connection must carry, and the endpoint must verify, an
`org_id` + `meeting_id` + a signature minted with `LIVEKIT_API_SECRET`. This is what finally makes
`config.py`'s three dead `LIVEKIT_*` settings load-bearing. Verify **before** `accept()`, and close
with a policy code on failure — an unauthenticated connection must never reach a `SessionTracker`.
Note this is our own token over the LiveKit secret, not a LiveKit-issued one; the egress process
sends whatever URL we gave it, so we sign it ourselves at Task 4's call site.

**(b) Frame protocol.** Per Task 3's findings: binary frames are raw PCM (expect 48 kHz mono
s16le from a WebRTC opus track, confirm in the spike); text frames, if any, are JSON control
messages (mute/unmute) and must be parsed separately, never fed to the provider as audio. Define
this once, in a module docstring, with the spike's evidence cited — this is the contract two repos
depend on.

**(c) Backpressure.** A slow provider must not grow an unbounded queue behind a live meeting. Use a
bounded buffer and drop oldest with a counter + log on overflow. Captions have a p95 < 2 s SLO;
silently buffering is worse than dropping.

**Acceptance:** an unsigned connection is refused before accept; a signed one is accepted and its
binary frames reach the tracked session; control frames do not; a 10-second overflow is logged and
bounded.

---

### 6. Hop 4a — lazy session open from the audio connection (closes defect 8a) — **M** — deps: 5

**Repos/files:** `sessions.py`, `meeting_room_consumer.py`, `main.py`; an ADR or canon note.

Today `SessionTracker` sessions are opened by the `meeting-room-events` consumer, on whichever
replica owns that `org_id` partition. The audio connection lands on whichever replica the ingress
picks. With one replica this is invisible; at `A5` = 20k streams it is a silent, total failure.

Change ownership: the **audio connection** opens the session (it carries `org_id` and `meeting_id`
and is the only thing that actually has audio), and the consumer's `participant.joined` /
`participant.left` become session *hints* — still driving residency/PII reads and the audit entry,
still idempotent, but no longer the sole opener. `on_participant_left` must remain a safe no-op for
a meeting this replica never saw.

This is the second of the two options already recorded against defect 8a and is the one that does
not require ingress-level `meeting_id` affinity. It is a canon-adjacent decision — record it as an
ADR or a canon note before implementing, per the repo's canon-first rule.

**Acceptance:** with two `transcription-service` replicas, audio arriving on the replica that did
*not* consume `joined` still opens a stream and produces `transcript_chunks` rows.

---

### 7. Hop 4b — `TranscriptionStream` accepts audio (the interface change) — **M** — deps: 5

**Repos/files:** `providers/base.py`, `providers/fake.py`, `providers/deepgram.py`,
`sessions.py`, and every test touching them.

**This is the architectural change the audit identified, and it is worth being precise about why no
credential fixes it.** `TranscriptionStream` today is:

```python
class TranscriptionStream(Protocol):
    def utterances(self) -> AsyncIterator[Utterance]: ...
    async def close(self) -> None: ...
```

Both methods are outbound. A stream, as typed, is a source that manufactures utterances from
nothing. There is no parameter, no method, and no constructor argument anywhere in
`TranscriptionProvider` or `TranscriptionStream` through which a byte of audio can enter. Handing
this code a valid `DEEPGRAM_API_KEY` and flipping `COMPLIANCE_GATE_GREEN` to `True` would open a
real Deepgram socket that **receives nothing and transcribes silence**. The gate is not what stands
between this code and working transcription; the missing method is.

Add to the Protocol:

```python
async def send_audio(self, frame: bytes) -> None: ...
```

and implement it in all three bindings: `_DeepgramStream` forwards the frame onto its open WSS
(`deepgram.py`, which currently only reads from `self._ws`); `_FakeStream` records frames and
optionally advances its script on receipt so tests can assert audio actually flowed; the new
provider from Task 9 buffers into its VAD segmenter.

Then wire `SessionTracker`: alongside `_pump` (stream → persistence) there must be a path from the
`/internal/audio` connection into `stream.send_audio`. Add a lookup on `SessionTracker` returning
the open stream for a `meeting_id`, or better, have the audio connection own the session (Task 6)
and hold the stream reference directly.

Keep `_assert_conforms` calls in place — mypy is what catches a binding that stops conforming, and
that guard already exists in both `fake.py` and `transport.py`.

**Acceptance:** `mypy --strict` green with the extended Protocol; a test drives real bytes through
`/internal/audio` → `SessionTracker` → `FakeProvider` and asserts the frames arrived; the existing
finals-only persistence tests still pass unchanged.

---

### 8. `LoopbackProvider` — prove the pipeline with zero ASR — **S** — deps: 7

**Repos/files:** new `providers/loopback.py`; `providers/factory.py`.

A provider whose `send_audio` measures what it receives (frame count, bytes, sample-rate sanity,
inter-arrival jitter) and emits a synthetic `Utterance` per N seconds of *actual audio received*,
with the real elapsed timing.

**Why this before Whisper.** `FakeProvider` already proves persistence, redaction, outbox, and
fan-out — it has done so since the last band. The *only* genuinely new thing the audio path needs
proving is that real bytes traverse hops 1-4 with correct timing and no backpressure collapse.
`LoopbackProvider` proves exactly that, in a day, with no model, no GPU, and no new dependency.
Reaching for an ASR engine to answer a plumbing question is the expensive way round.

**Acceptance:** speak into a browser tab, watch `transcript_chunks` rows appear whose `start_ms` /
`end_ms` track wall-clock speech, and captions render in the UI. **That is the vertical alive,
end to end, today, with neither a DPA nor a LiveKit account.**

---

### 9. Local `faster-whisper` provider — **L (3 days)** — deps: 7, 8

**Repos/files:** new `providers/whisper.py`; `providers/factory.py`; `config.py`;
compose (a CPU-mode whisper sidecar is fine for dev); `10-realtime-and-transcription.md` §3.3 is
already the sanctioning doc.

**Honest assessment: sound, but only if scoped correctly — and it is emphatically not the first
thing to build.**

Where it is sound:

- It is not an invention. `10-realtime-and-transcription.md:172-190` already specifies a provider
  abstraction with a self-hosted `faster-whisper` cost tier, explicitly to serve batch/Free-plan
  transcription **and** to satisfy "a subprocessor-substitution requirement: a tenant with a DPA
  constraint forbidding a US ASR subprocessor can be pinned to the self-hosted Whisper tier". A
  DPA-constrained situation is precisely the one we are in. Building it now builds a planned
  product capability, not scaffolding to be thrown away.
- The abstraction it needs already exists and is honoured. `select_provider()` switches on config;
  `SessionTracker` depends only on the Protocol; `persistence.py` is provider-agnostic.
- It gives real words, which `LoopbackProvider` cannot — needed to exercise redaction against
  genuine PII-shaped text, and to demo the product.

Where the false-economy risk is real and specific — **one trap, worth naming precisely**:

- **`faster-whisper` is not a streaming API.** It transcribes buffered segments. It has no interim
  results and no native diarization. If the audio-input interface is designed around it, the
  interface will acquire batch semantics — "hand me a segment, get a transcript" — and Deepgram's
  true-streaming shape (continuous send, asynchronous interim-then-final revisions of the *same*
  `chunk_id`) will not fit it later. That would be a genuine false economy: a day saved now, the
  interim/final contract re-cut later, and `persistence.py`'s finals-only logic and the
  `chunk:{chunk_id}` idempotency key disturbed along with it.
- **The mitigation is cheap and non-negotiable:** design `send_audio` from Deepgram's shape (Task 7
  does exactly this — `deepgram.py` already encodes it) and make the Whisper provider **adapt to
  it**: VAD-segment internally, emit only `is_final=True` utterances, `speaker_id="0"` unless
  someone wires `pyannote`. Deepgram must never be adapted to Whisper's shape. Whisper's lack of
  interims is then a documented quality limitation of that tier — which matches the routing policy
  in §3.3 (live diarized streaming stays on Deepgram) — rather than a defect in the interface.
- Second, smaller risk: nobody should mistake this for lifting the compliance gate. It does not
  touch `compliance.py`; `DeepgramProvider` stays refused. Say so in the module docstring, the way
  `devseed.py` and `compliance.py` already do.

**Verdict:** build it, second, after `LoopbackProvider`, with the interface already frozen by
Task 7. Scoped that way it is sound. Built *first*, as the thing that proves the pipeline, it is a
false economy — it puts a model on the critical path of a plumbing question and risks shaping the
interface around batch semantics.

---

### 10. Provenance marker on synthetic transcript rows — **M** — deps: none (do it early)

**Repos/files:** canon **first** — `mytheus-docs/_canon/02-data-model.md` and
`_canon/03-event-taxonomy.md`; then `mytheus-events/schemas/transcript-chunks/`;
a new Alembic migration in `transcription-service/migrations/versions/`;
`persistence.py`, `providers/base.py`, `providers/fake.py`, `providers/loopback.py`,
`providers/deepgram.py`, `devseed.py`; the read path in `core-api` / `bff`; the UI transcript view.

**Why this is urgent rather than tidy.** Today `FakeProvider` output and `devseed.py` output are
written through `persistence.process_utterance` — deliberately, so that seeded data obeys the real
rules — and land in `transcript_chunks` as rows that are **byte-for-byte indistinguishable from a
real meeting's transcript**. `devseed.py`'s own docstring names the hazard: "plausible-looking
fabricated transcripts landing in a real tenant's history, where nothing downstream could
distinguish them from a real meeting." Its only guard is `TRANSCRIPTION_DEV_SEED=1`, an env var it
itself calls "a weaker control than `compliance.py`'s source constant".

That is tolerable **only while no real path exists**. The moment Tasks 1-8 land, real and synthetic
rows coexist in the same partitioned table, feed the same `transcript-chunks` topic, and reach the
same downstream consumers — `embedding-service` will vectorise them into Qdrant, `search-service`
will index them into FTS, and the agent tier will reason over them and surface decision candidates
from fabricated speech. Retrofitting provenance after those vectors exist means a reindex; adding it
before means a default value.

**Design.** Add `provider text NOT NULL DEFAULT 'unknown'` (or a narrower
`synthetic boolean NOT NULL DEFAULT false` — the canon edit decides which; `provider` is more useful
downstream and costs the same) to `transcript_chunks`, and the matching field to the
`TranscriptChunk` Avro record **with a default**, so the change is BACKWARD-compatible. The
`redacted` field is the exact precedent: it exists on both the row and the Avro record with
`"default": false`, added under this same reasoning. Populate it in `process_utterance` from the
provider that produced the utterance — carry it on `Utterance` or pass it alongside; do not infer it
at the write site.

Then make it visible: the UI transcript view and any export must mark non-production provenance.
A marker nobody can see is not a marker.

**Acceptance:** an Avro BACKWARD-compatibility check passes; a `FakeProvider`-produced row and a
`devseed` row are distinguishable from a Deepgram-produced one by a SQL predicate and by a field on
the topic; the UI shows it.

---

### 11. End-to-end smoke test — **M** — deps: 2, 4, 5, 6, 7, 8

**Repos/files:** a new integration test spanning `mytheus-backend` + `mytheus-realtime`; CI wiring.

Browser (or a headless publisher feeding a WAV) → local LiveKit → egress → `/internal/audio` →
`SessionTracker` → `LoopbackProvider` → `transcript_chunks` + `transcript-chunks` topic →
`realtime-gateway` → a caption on a socket. Assert: finals only are persisted; the row carries the
right `org_id` under RLS; redelivery is a no-op; the provenance marker is set; end-to-end latency is
measured and recorded against the p95 < 2 s budget.

This is the test that would have caught all four absent hops. It should exist before anyone claims
the vertical is done.

---

### 12. (Adjacent, explicitly out of scope) Recording egress — **M** — deps: 4

`start_egress` at `transport.py:135-137` and the `nexus-recordings` bucket. Listed for completeness
because Task 4 builds the `EgressClient` integration that this needs, making it cheap immediately
afterwards. It is a recording feature, not part of the live transcription vertical, and nothing in
Tasks 1-11 depends on it. Do not let it expand Task 4's scope.

---

## Part D — Effort summary

| # | Task | Size | Deps |
|---|---|---|---|
| 1 | Self-hosted LiveKit in compose | S (1d) | — |
| 2 | Browser mic publish | M (2d) | 1 |
| 3 | Spike: LiveKit WS egress contract | S (1d) | 1 |
| 4 | Hop 2 — server-side tap (egress → WS) | L (4d) | 1, 3 |
| 5 | Hop 3 — handshake + frame protocol | M (2d) | 3, 4 |
| 6 | Hop 4a — lazy session open (defect 8a) | M (2d) | 5 |
| 7 | Hop 4b — `TranscriptionStream.send_audio` | M (2d) | 5 |
| 8 | `LoopbackProvider` | S (1d) | 7 |
| 9 | Local `faster-whisper` provider | L (3d) | 7, 8 |
| 10 | Provenance marker | M (2d) | — |
| 11 | End-to-end smoke test | M (2d) | 2,4,5,6,7,8 |
| 12 | Recording egress (out of scope, adjacent) | M (2d) | 4 |

**12 tasks; 24 engineer-days total (22 excluding the out-of-scope task 12).**
Critical path 1 → 3 → 4 → 5 → 7 → 8 → 11: **13 days**, after which the vertical is alive end to end.
Tasks 2 and 10 parallelise off the critical path. Task 9 is a follow-on, not a gate.

**Blocked on LEGAL:** production-quality Deepgram ASR only (`compliance.py:39`). Zero tasks above.
**Blocked on PROCUREMENT:** nothing.
**Actionable today:** all 12.
