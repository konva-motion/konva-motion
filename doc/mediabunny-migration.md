# Browser media migration: Mediabunny + Web Audio

Status: **done** (all 5 phases landed & verified) · Scope: **browser preview only** (the
Node renderer keeps ffmpeg) · Rollout: **replaced `BrowserVideoSource`/`BrowserAudioSource`
as the browser defaults**

## Why

[Mediabunny](https://mediabunny.dev) is a pure-TypeScript, zero-dependency (no WASM)
library that drives the browser's **WebCodecs** API for demux + decode/encode. Swapping
the browser video/audio decoders to it buys us:

- **Frame-accurate seeking.** Today `BrowserVideoSource` leans on
  `HTMLVideoElement.currentTime` + `fastSeek()`, which lands on the nearest keyframe and
  drifts. The driver tolerates this with thresholds (0.45s play / 0.01s scrub). Mediabunny
  decodes the *exact* frame at a timestamp — deterministic scrubbing instead of approximate.
- **One decode model for preview and (eventually) render** — same seek semantics, so what
  you scrub is what you render.
- **Richer metadata + format reach** — real rotation, color space, HDR, per-track codec
  info; MKV / `.ts` / FLAC / Ogg decoded via WebCodecs even when `<video>` refuses them.

Reference implementation: Remotion's `packages/media/src` (`remotion-dev/remotion`) — a
Mediabunny/WebCodecs player that replaced their old HTMLMediaElement approach. This plan
borrows its proven techniques.

## The core shift: push → pull

Current sources model `HTMLMediaElement`: **self-playing** (`play()` advances on its own
clock, audio reaches the speakers for free), driver only corrects drift. Mediabunny is
**pull-based**: you ask for the exact frame/samples at timestamp `t`.

- For **video** this fits the per-frame `driver.tick()` model perfectly — pull the frame
  for the current timestamp each tick.
- For **audio** it does not: audio must be **scheduled ahead through Web Audio** (you can't
  re-pull every 16ms without clicks). This is the heaviest new piece.

Two techniques carried over from Remotion:

1. **Forward-only iterator + "can this satisfy time T?" predicate.** Keep one decode
   iterator moving forward; reseek/teardown *only* on backward or large-jump (>~3s) seeks.
   This single idea is the entire seek-accuracy strategy, and maps directly onto
   `setFrame(n)` ≡ `seekTo(n/fps)`.
2. **`audioSyncAnchor`** — the indirection mapping `AudioContext.currentTime` ↔ timeline
   seconds. Keeps audio locked to the frame clock across play/pause/seek/rate. The piece
   most easily gotten wrong.

## Architecture changes

### 1. Shared audio context — `media/audio/shared-audio-context.ts` (new)
Singleton owned by the `Composition`: the `AudioContext`, a master `GainNode`, the
`audioSyncAnchor`, and the user-gesture unlock (`resume()` on first `play()`).
`Composition.play()/pause()/setFrame()` update the anchor. The spine everything schedules
against.

### 2. `MediabunnyVideoSource` — replaces `BrowserVideoSource`
Implements the existing `VideoSource` interface, semantics adjusted to pull:
- `Input` + `UrlSource`/`BlobSource` → `CanvasSink` (poolSize 2); internal canvas becomes
  `element`.
- Forward iterator with read-ahead buffer; `seek(t)` satisfies forward by decoding ahead,
  reseeks only on backward / >~3s jumps.
- `play()/pause()` → state flags only (frames pulled per tick); `onFrame` fires after paint.
- Meticulous `VideoFrame.close()` lifecycle (decoder pools exhaust fast otherwise).
- `canDecode()` gating with graceful fallback to a retained `HTMLVideoElement` source for
  codecs WebCodecs can't handle.

### 3. Web Audio scheduler — replaces the self-playing `AudioSource` model
Driven off the shared context + `audioSyncAnchor`:
- `AudioBufferSink` + **0.5s priming iterator** (decoders need lead-in before the seek point).
- Continuous schedule-ahead: each decoded buffer → fresh `AudioBufferSourceNode` →
  per-instance `GainNode` → master. `node.start(scheduledTime, offset, duration)`.
- Per-instance gain for volume/mute; `playbackRate` on the node + divided out of `targetTime`.
- On seek/rate/anchor-change: tear down queued nodes and reschedule.

### 4. Video's own soundtrack
A `Video` opens **one** Mediabunny `Input` exposing both a `CanvasSink` (frames) and an
`AudioBufferSink` (audio track) → routed through the same scheduler. No double-loading.

### 5. Rewrite the preview drivers
- `PreviewVideoDriver`: collapses to "seek to exact frame every tick" — drift/self-play
  branches deleted.
- `PreviewAudioDriver`: drives the scheduler (anchor + schedule-ahead) instead of
  `play()/pause()/currentTime` on an element.

## Critical constraint: don't break the renderer

`AudioSource`/`VideoSource` are shared with the Node renderer (`NullAudioSource`,
`RenderingAudioDriver`, `FfmpegVideoSource`). Any change must keep those compiling and
rendering.

- Keep `VideoSource` intact (semantics-only change).
- For audio, **add** the scheduler as the new browser path while leaving the `AudioSource`
  interface (and `NullAudioSource`) compatible — the rendering audio path keeps collecting
  `AudioAsset` metadata exactly as today.
- Verify `pnpm build` across all packages and a server render still works.

## Dependencies
- Add `mediabunny` as a regular dep of `@konva-motion/core` (pure JS, no WASM,
  tree-shakable, ~30–70 kB gzipped). No `@mediabunny/*` encoder add-ons needed — browser
  decode uses native WebCodecs.

## Rollout
`MediabunnyVideoSource`/`MediabunnyAudioSource` + the Web Audio scheduler are the browser
defaults in `runtime-defaults`. `BrowserVideoSource`/`BrowserAudioSource` are retained and
exported for opt-in use via a node's `sourceFactory`, but are **not** auto-wired as a
`cannot-decode` fallback (see Phase 4 — the pull driver can't drive self-playing elements).

## Phasing

1. ✅ **Mediabunny dep + `MediabunnyVideoSource`** (frames only, no audio) — wired as the
   browser default; `PreviewVideoDriver` rewritten to the pull model.
2. ✅ **Shared `AudioContext` + Web Audio scheduler** for standalone `Audio` nodes.
3. ✅ **Video soundtrack** through the same scheduler.
4. ✅ **Looping audio**; playback-rate / volume curves / codec-fallback decisions.
5. ✅ **Cross-package build + server-render regression check**, then docs.

### Phase 1 status (done)

Landed:
- `packages/core/src/media/video/video-source-mediabunny.ts` — `MediabunnyVideoSource`:
  `Input` + `UrlSource` → `CanvasSink` forward iterator with one-frame lookahead, restart
  only on backward / >3s-forward seeks, blitting each frame onto one owned canvas exposed as
  `element`. Seek-coalesced (one decode in flight, latest target wins). `play()/pause()/
  set{Muted,Volume,PlaybackRate}` are no-ops (pull model).
- `PreviewVideoDriver` collapsed to "seek to the exact media time every tick" — drift /
  self-play branches and the `isPlaying` subscription removed.
- `runtime-defaults.ts` now defaults the browser video factory to `MediabunnyVideoSource`.
  `BrowserVideoSource` is retained (unwired) for the future `cannot-decode` fallback.
- `mediabunny@^1.49.0` added as a regular dep of `@konva-motion/core`.

Verified in the demo (`/c/video-sync`, `pnpm dev`): the decoded top clip's burnt-in SMPTE
timecode reads exactly `00:00:02:00` at frame 60 (30fps); pixel hashes show stepping
60→90 changes the frame and re-seeking 90→60 reproduces it byte-for-byte (deterministic,
frame-accurate); backward / large-jump seeks and two simultaneous videos render cleanly;
`play()` advances at a true 30fps; no console errors. Full `pnpm build` passes across all
packages.

**Known Phase-1 limitation:** a `Video`'s own soundtrack is now **silent** in preview
(WebCodecs decodes frames only; nothing plays the audio track yet). Addressed in Phase 3.

### Phase 2 status (done)

Standalone `Audio` nodes now play through a Web Audio scheduler instead of
`HTMLAudioElement`.

Landed:
- `packages/core/src/media/audio/shared-audio-context.ts` — `SharedAudioContext`, one per
  `Composition` (cached in a `WeakMap`): lazily creates the single `AudioContext` + master
  `GainNode`, hands out per-clip channel gains, and exposes `resume()` for the user-gesture
  unlock. Browser-only; never constructed in Node.
- `packages/core/src/media/audio/audio-source-mediabunny.ts` — `MediabunnyAudioSource`:
  `Input` + `UrlSource` → `AudioBufferSink`. Pure decode handle (exposes `sink` +
  `firstTimestamp` via the `SchedulableAudioSource` interface); the old self-playing
  `play()/seek()/volume` surface is inert.
- `audio-for-preview.ts` — `PreviewAudioDriver` rewritten as a **schedule-ahead pump**:
  while playing it decodes buffers from the sink and schedules them on the shared
  `AudioContext`, anchoring timeline media-time to the context clock
  (`scheduledCtx = anchorCtx + (mediaTime − anchorMedia) / effectiveRate`). Stays ~1s
  ahead (throttled), primes 0.25s of lead-in, re-anchors on a seek (>0.15s playhead
  divergence) or rate change, and stops all nodes on pause/scrub/deactivate. Per-clip gain
  tracks `effectiveVolume/Muted` each tick (so volume automation + the mixer master apply).
  Web Audio auto-resamples the decoded `AudioBuffer`s — no manual resampling in preview.
- `runtime-defaults.ts` defaults the browser audio factory to `MediabunnyAudioSource`.

Verified in the demo (`/c/audio-mixer`, 5 clips with volume automation/ducking/crossfade):
one shared `AudioContext` (running), per-channel gains, **113** buffer-source nodes for a 2s
play + ~1s lookahead — i.e. the anchor tracks a moving playhead with no false-restart node
explosion; pausing schedules **0** further nodes; a large mid-play seek (frame 30→450)
re-anchors the pump to exactly 15.0s and schedules fresh nodes from there; no console
errors. Full `pnpm build` passes (renderer interface unchanged — `NullAudioSource` still
satisfies `AudioSource`).

### Phase 3 status (done)

A `Video`'s own soundtrack now plays in preview (the Phase-1 silence is fixed).

Landed:
- `video-source-mediabunny.ts` — `MediabunnyVideoSource` now opens the file's **audio
  track off the same `Input`** (`getPrimaryAudioTrack` → `AudioBufferSink`) alongside the
  `CanvasSink`, so an audible clip is demuxed once, not twice. It additionally satisfies
  `SchedulableAudioSource` (`sink` + `firstTimestamp` getters); a clip with no decodable
  audio just reports `sink === null` and stays silent.
- `media/video/index.ts` — in preview, `Video._ensureDriver()` also builds a
  `PreviewAudioDriver` (the Phase-2 scheduler) with the video's own `effectiveVolume/Muted`
  and the **same `VideoTiming`** as the frames, so audio and video share trims/rate and stay
  locked together. `_kmTick`/`_kmDeactivate`/`destroy` drive both drivers. Rendering env is
  unchanged (no audio driver — see gap below).

Verified in the demo (`/c/video-sync`, clips with `muted: false`): video stays
frame-accurate (timecode reads `00:00:03:00` at frame 90; stepping changes the frame and
re-seeking reproduces it); the soundtrack schedules through the **one** shared
`AudioContext` (127 buffer-source nodes over a 2s play, **0** after pause); no console
errors. Full `pnpm build` passes (renderer + player included).

**Server-side video audio gap:** the Node renderer still strips video audio (`-an`) and
collects no `AudioAsset` for `Video` (only standalone `Audio` nodes are muxed) — this
predates the migration. Closing it means giving `Video` a `RenderingAudioDriver` in the
rendering env (with server-side audio-presence detection so audio-less clips don't break
the ffmpeg mux). Tracked as a follow-up, not blocking this browser migration.

### Phase 4 status (done)

- **Looping audio.** `PreviewAudioDriver` now schedules in **unwrapped** (monotonic) media
  time, and its buffer stream replays the `[trimBefore, trimAfter)` segment forever for a
  looping clip (offsetting each iteration), so the scheduler just keeps consuming. Unwrapped
  time also makes seek-detection correct across the loop seam (no false re-anchor when the
  wrapped playhead jumps back to 0). Verified: non-loop scheduling unchanged (113 nodes /
  2s, 0 after pause); the loop stream produces buffers monotonically past the segment end
  (`startUnwrapped` reached 3.05s over a forced 1s loop — ~4 iterations — with no gap at the
  0.99s→1.02s seam). **Video** looping already worked via the per-tick wrapped seek (each
  loop boundary is just a backward seek that restarts the frame iterator).
- **Volume curves** already work from Phase 2 (per-clip gain re-reads `effectiveVolume`
  every tick), confirmed by the audio-mixer automation/ducking/crossfade.
- **Decisions (accepted, documented):**
  - *Reverse playback* (negative comp rate) produces no audio — the pump guards `effRate ≤ 0`
    and skips. Reverse audio isn't meaningful for preview.
  - *`playbackRate ≠ 1`* shifts pitch — `AudioBufferSourceNode` has no `preservesPitch` and
    Web Audio offers no built-in time-stretch. Acceptable for preview.
  - *`cannot-decode` fallback* is **descoped**: the new pull driver can't drive a
    self-playing `<video>`/`<audio>` element, so a fallback to `BrowserVideoSource`/
    `BrowserAudioSource` would be silent/awkward — and WebCodecs already covers every codec
    those elements do. Undecodable input fails with a clear console error; the browser
    sources stay exported for manual `sourceFactory` opt-in.
  - *Backgrounded tab*: the rAF composition clock and the `setTimeout` audio pump both track
    real time (`performance.now` / `AudioContext.currentTime`), so on refocus the comp jumps
    to the realtime-correct frame and realigns — not a true desync.

### Phase 5 status (done)

- **Cross-package build:** `pnpm build` passes (core, player, renderer, studio, docs); the
  new sources are exported from the core barrel (`MediabunnyVideoSource`,
  `MediabunnyAudioSource`, `SchedulableAudioSource`).
- **Server-render regression:** both renderer examples pass headlessly —
  `example` (shapes + Image + Audio → mp4, `hasAudio: true`, + still + frames) and
  `example:mixer` (looping `Video` bed + 5 audio clips + master automation → 780-frame mp4,
  `hasAudio: true`). The renderer's `AudioSource` contract is unchanged, so `NullAudioSource`
  and the ffmpeg mux path are intact.

> Preview-tooling note: the demo's Vite dev server is pinned to port **5174**
> (`demo/vite.config.ts`), but the preview harness drives port **5173**. To verify in the
> browser, temporarily set that port to `5173`, then restore it.

## Verification

Per phase, in the demo via the preview tools:
- Frame-exact scrub (step to frame N, confirm the rendered frame matches).
- Audio sync (a clap/beat lands on the right frame).
- Play / pause / seek, looping, 2× rate, volume automation.
- `pnpm build` + `pnpm check` + a headless render to prove the renderer is intact.

## Risk outcomes

- **Audio sync anchor** (the highest-risk piece) — resolved: the unwrapped-media anchor
  tracks a moving playhead with no false re-anchors (113 nodes / 2s, not thousands), and a
  large seek re-anchors to the exact target. Verified in Phases 2 & 4.
- **Frame lifecycle leaks** — sidestepped by using `CanvasSink` (yields already-drawn
  canvases, no `VideoFrame.close()` lifecycle) blitted onto one owned canvas.
- **Codec fallback** — descoped (see Phase 4): WebCodecs covers the codecs `<video>`/
  `<audio>` do, and the pull driver can't drive self-playing elements; undecodable input
  fails with a clear error.
- **`AudioContext` autoplay unlock** — the scheduler calls `resume()` on first play; when
  `play()` is invoked from the player's button (a user gesture) the context unlocks.
  `SharedAudioContext.resume()` is the single seam if `@konva-motion/player` ever needs to
  wire it more explicitly.

## Files changed

| Change | File |
| --- | --- |
| **New** — Mediabunny video source (frames + audio sink) | `packages/core/src/media/video/video-source-mediabunny.ts` |
| **New** — Mediabunny audio source (decode handle) | `packages/core/src/media/audio/audio-source-mediabunny.ts` |
| **New** — per-composition Web Audio bus | `packages/core/src/media/audio/shared-audio-context.ts` |
| Video preview driver → pull model | `packages/core/src/media/video/video-for-preview.ts` |
| Audio preview driver → schedule-ahead pump (+ looping) | `packages/core/src/media/audio/audio-for-preview.ts` |
| Video node — preview audio driver alongside frames | `packages/core/src/media/video/index.ts` |
| Default browser factories → Mediabunny sources | `packages/core/src/engine/runtime-defaults.ts` |
| Public barrel — export new sources | `packages/core/src/index.ts` |
| `mediabunny` dependency | `packages/core/package.json` |
| Unchanged (verified intact) | renderer: `audio-source-null.ts`, `video-source-ffmpeg.ts`, `audio-track.ts`, `ffmpeg.ts` |

`BrowserVideoSource`/`BrowserAudioSource` and the rendering drivers are untouched and still
exported. The `Composition` and `AudioSource`/`VideoSource` interfaces are unchanged.
