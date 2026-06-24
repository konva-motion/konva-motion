# Plan: Mediabunny on the server (`@konva-motion/renderer`)

Status: **implemented**. The renderer now encodes/decodes/muxes through Mediabunny
(`@mediabunny/server`); the ffmpeg CLI and `@ffmpeg-installer` are gone. See
[`renderer.md`](./renderer.md) for the current API. Companion to
[`mediabunny-migration.md`](./mediabunny-migration.md), which covered the *client*
migration (commit `85c5cee`). The sections below are the original design rationale,
kept for context.

## TL;DR

- The client now decodes all media through **mediabunny + WebCodecs**. The
  server renderer still shells out to the **ffmpeg CLI** (`@ffmpeg-installer`)
  for both video decode (`FfmpegVideoSource`) and the final encode/mux.
- Mediabunny ships a server extension — [`@mediabunny/server`](https://mediabunny.dev/guide/extensions/server) —
  that registers custom WebCodecs decoders/encoders backed by
  [node-av](https://github.com/seydx/node-av) (N-API bindings to the FFmpeg C
  API). After `registerMediabunnyServer()`, **the same mediabunny code paths
  that run in the browser run in Node.**
- This lets us replace the renderer's three uses of the ffmpeg *CLI* (probe,
  decode, encode/mux) with mediabunny's `Input` / `Output` API, unify the
  client and server decode code, and — for the first time — do **real audio**
  on the server (decode + mix in JS) instead of the metadata-only ffmpeg
  filtergraph hack.
- **Can we drop ffmpeg entirely?** *The ffmpeg CLI and `@ffmpeg-installer`:
  yes.* *FFmpeg the library: no* — it still lives underneath node-av as a
  prebuilt native binary. We trade a spawned CLI for in-process C bindings.
  `skia-canvas` stays (it's the Konva rasterization backend, unrelated to
  ffmpeg).

---

## 1. Where we are today

The renderer pipeline (see [`renderer.md`](./renderer.md) and
`packages/renderer/src`) uses the ffmpeg CLI in **three** places:

| Use | File | What it does |
| --- | --- | --- |
| **Probe** | `video-source-ffmpeg.ts:34` (`probe`) | `spawn(ffmpeg, ["-i", src])`, regex-parse stderr banner for `WxH`, duration, fps |
| **Decode** | `video-source-ffmpeg.ts:70` (`FfmpegVideoSource`) | One persistent ffmpeg process per video; `-ss` seek + `fps,scale` filter → raw RGBA on `pipe:1`; ~393 lines of manual chunk reassembly, backpressure, forward-seek-vs-restart heuristics |
| **Encode + Mux** | `ffmpeg.ts` | Pass 1: `spawnVideoEncoder` pipes raw RGBA frames to `libx264`. Pass 2: `buildMuxArgs` + `runFfmpeg` muxes a temp video file with N audio clips via an `atrim/atempo/volume/adelay/amix` `-filter_complex` string |

Two structural consequences:

- **Two-pass + temp file.** We render video to `/tmp/km-render-*.mp4`, then run
  a *second* ffmpeg to mux audio. `renderToStream` makes a fragmented temp file
  and reads it back through a `PassThrough`.
- **Audio is metadata-only during render.** `NullAudioSource`
  (`audio-source-null.ts`) decodes nothing. `RenderingAudioDriver` (in core)
  records per-frame `{src, mediaInSeconds, playbackRate, volume}` snapshots;
  `collectAudioTrack` (`audio-track.ts`) coalesces them into clip runs; ffmpeg's
  filtergraph reconstructs the mix. Volume envelopes are baked into a
  hand-built piecewise-linear `volume='...'` expression; playback rate into an
  `atempo` chain clamped to [0.5, 2.0].

It works, but it's a lot of brittle string/byte plumbing, it duplicates the
decode logic the client already has in mediabunny, and the audio path can only
express what we can encode as ffmpeg filter strings.

## 2. What `@mediabunny/server` provides

From the [server guide](https://mediabunny.dev/guide/extensions/server) and
[node-av](https://github.com/seydx/node-av):

```ts
import { registerMediabunnyServer } from "@mediabunny/server";
registerMediabunnyServer(); // registers custom WebCodecs decoders/encoders
```

- **Implementation:** thin wrapper over node-av → N-API bindings to FFmpeg's C
  API (`libavcodec`/`libavformat`/…). *Not* the WHATWG WebCodecs polyfill, *not*
  the ffmpeg CLI. After registration, mediabunny's `Input`/`Output`,
  `CanvasSink`, `AudioBufferSink`, `VideoSampleSource`, etc. all work in Node.
- **Binaries:** node-av downloads prebuilt FFmpeg libs on `npm install`
  (GitHub releases) for **macOS / Linux / Windows × x64 / arm64**. No system
  ffmpeg, no manual setup. CLI binary is *optional* (exposed via `ffmpegPath()`
  but not needed for the C-API path).
- **Performance claims:** automatic hardware-accelerated encode/decode on all
  platforms, built-in multithreading, zero-copy decode/encode, O(1) memory via
  pipelining.
- **Codecs:** decode/encode AVC/H.264, HEVC, VP8/9, AV1 (video); AAC, MP3,
  Vorbis, Opus, FLAC, AC-3/E-AC-3 (audio).

### Relevant write API ([writing](https://mediabunny.dev/guide/writing-media-files), [sources](https://mediabunny.dev/guide/media-sources))

```ts
import { Output, Mp4OutputFormat, FilePathTarget, StreamTarget,
         CanvasSource, VideoSampleSource, AudioBufferSource } from "mediabunny";

const output = new Output({
  format: new Mp4OutputFormat(),
  target: new FilePathTarget("./out.mp4"),   // or StreamTarget for piping
});

const video = new VideoSampleSource({ codec: "avc", bitrate: 1e6 });
const audio = new AudioBufferSource({ codec: "aac", bitrate: 192_000 });
output.addVideoTrack(video, { frameRate: fps });
output.addAudioTrack(audio);

await output.start();
// frame-accurate, manually timed (NOT real-time capture):
//   canvasSource.add(timestamp, duration, { keyFrame? })  -> seconds
//   audioBufferSource.add(audioBuffer)  -> appended after previous, from t=0
await output.finalize();
```

Key API facts that shape the design:

- `CanvasSource.add(timestamp, duration, opts?)` takes **seconds** — frame-exact
  manual encoding, exactly what an offline renderer wants. But `CanvasSource`
  wants an `HTMLCanvasElement`/`OffscreenCanvas`; our frames come from
  **skia-canvas**. Safer to use **`VideoSampleSource`** + a `VideoSample` built
  from the raw RGBA buffer we already produce (`canvas.toBufferSync("raw",
  "rgba")`). See risk R1.
- `AudioBufferSource.add(audioBuffer)` has **no per-buffer timestamp** — buffers
  concatenate from t=0. So overlapping clips, per-clip delays, volume envelopes,
  and rate changes must be **pre-mixed into one PCM timeline** before feeding.
  See §4.

## 3. Target architecture

Two layers, adopt independently.

### Layer A — decode: reuse core's mediabunny sources on the server

Today core already has `MediabunnyVideoSource` / `MediabunnyAudioSource`
(browser default factories). With `registerMediabunnyServer()` active, those
same sources can run in Node. `setupServerRendering` would shrink to:

```ts
import { registerMediabunnyServer } from "@mediabunny/server";
// reuse core browser factories instead of nodeVideoSourceFactory / nullAudioSourceFactory
```

This **deletes `video-source-ffmpeg.ts` (~393 lines)** and the stderr-regex
`probe`, replacing them with the code the client already exercises daily.

Sink choice: `MediabunnyVideoSource` uses `CanvasSink` (yields a `WrappedCanvas`),
which needs canvas creation we shouldn't rely on in Node. **Server path uses
`VideoSampleSink`** instead — it yields raw `VideoSample`s that blit straight
onto the skia canvas (`sample.draw(skiaCtx, 0, 0)`, or `sample.copyTo(buf)` →
`putImageData`, the same blit `FfmpegVideoSource._paint` does today). We make
the sink environment-selectable inside core's source. See §7.R1/R2.

### Layer B — encode/mux: one mediabunny `Output`, one pass

Replace `ffmpeg.ts` (`spawnVideoEncoder` + `buildMuxArgs` + `runFfmpeg`) and the
temp-file two-pass with a single `Output`:

```ts
const video = new VideoSampleSource({ codec: "avc", bitrate: presetBitrate });
const audio = new AudioSampleSource({ codec: "aac", bitrate: audioBitrate });
output.addVideoTrack(video, { frameRate: fps });
output.addAudioTrack(audio);
await output.start();

for (let f = from; f <= to; f++) {
  await comp.renderFrame(f);
  const raw = comp.captureCanvas().toBufferSync("raw", "rgba");
  const sample = new VideoSample(raw, {
    format: "RGBX",            // skia RGBA buffer; alpha ignored on encode
    codedWidth: w, codedHeight: h,
    timestamp: f / fps,        // seconds
  });
  await video.add(sample, 1 / fps);
}
await audio.add(mixAudio(collectAudioTrack(comp, fps)));   // §4
await output.finalize();
```

- `renderToFile` → `FilePathTarget`. **No temp video, no second pass.**
- `renderToStream` → `StreamTarget`, piping chunks directly to the consumer's
  `Readable` (kills the temp-file round-trip in `render.ts:181`).
- Quality presets map to mediabunny encoder `{ codec, bitrate }`. Mediabunny is
  bitrate-oriented (no CRF), so the four presets become target bitrates or the
  built-in `QUALITY_LOW/MEDIUM/HIGH` constants. See §7.R3.

### Audio: from filtergraph to JS mix (§4)

`collectAudioTrack` already gives us clip runs with `src`, frame range,
`mediaInSeconds`, `playbackRate`, `volume`. Instead of stringifying that into a
`-filter_complex`:

1. Decode each clip's PCM with mediabunny `AudioBufferSink` (same API the client
   uses — `audio-for-preview.ts`).
2. Mix into a single timeline (apply delay = `startFrame/fps`, per-clip gain /
   volume envelope, playback-rate resample) using an `OfflineAudioContext`-style
   mixer, or a small manual Float32 accumulator.
3. Feed the mixed buffer(s) to `AudioBufferSource.add()`.

This unifies decode with the client and removes `atempo`/`adelay`/`volume`/
`amix` string construction (`ffmpeg.ts:62-150`) entirely. It also makes the
server audio *correct* rather than "whatever ffmpeg filters can express."

## 4. Audio detail — no Web Audio needed

The earlier worry was that `AudioBufferSource.add()` wants a Web Audio
`AudioBuffer` (no `AudioContext` in Node). **Mediabunny sidesteps this entirely
with `AudioSampleSource` + raw-PCM `AudioSample`:**

```ts
import { AudioSampleSource, AudioSample } from "mediabunny";

const audio = new AudioSampleSource({ codec: "aac", bitrate: 192_000 });
// build from a raw interleaved/planar Float32 buffer with an explicit timestamp:
const sample = new AudioSample({
  data: mixedFloat32,      // our mixed PCM
  format: "f32-planar",
  numberOfChannels: 2,
  sampleRate: 48_000,
  timestamp: 0,            // seconds
});
await audio.add(sample);
```

So the audio path is a **plain JS mix, no extra audio-graph dependency**:

1. Decode each clip's PCM with mediabunny `AudioBufferSink` / `AudioSampleSink`
   (the same decode the client already uses in `audio-for-preview.ts`).
2. Mix into one `Float32Array` timeline: place each clip at sample offset
   `startFrame/fps × sampleRate`, apply playback-rate resample, apply the gain
   envelope.
3. Wrap as `AudioSample`(s) and `AudioSampleSource.add()` them.

The per-frame volume keyframes already collected by `RenderingAudioDriver`
(simplified via Ramer–Douglas–Peucker in `audio-track.ts:simplifyEnvelope`)
become a per-sample gain curve applied during the mix — no behavior regression,
and we can finally express envelopes the ffmpeg filtergraph couldn't.

## 5. Can we drop ffmpeg entirely?

**Honest answer: depends what "ffmpeg" means.**

| Thing | Drop? | Notes |
| --- | --- | --- |
| `@ffmpeg-installer/ffmpeg` dep | ✅ Yes | Replaced by `@mediabunny/server` + `mediabunny` |
| ffmpeg **CLI** invocations (`spawn`/`exec`) | ✅ Yes | All 3 sites (probe/decode/encode-mux) become `Input`/`Output` API calls |
| `ffmpeg-bin.ts`, `setFfmpegPath`, `ffmpegPath` option | ✅ Yes | No external binary to point at (node-av self-contains) |
| `video-source-ffmpeg.ts` (~393 lines) | ✅ Yes | Replaced by core mediabunny source |
| FFmpeg **the library** (libav*) | ❌ No | Still present, bundled inside node-av as a prebuilt native addon |
| `skia-canvas` | ❌ No | Unrelated to ffmpeg — it's Konva's headless 2D backend |
| headless-`gl` + GL compositor (`gl.ts`/`gl-node.ts`) | ❌ No | Shader transitions; orthogonal, keep as-is |

So we **drop the ffmpeg CLI and `@ffmpeg-installer` entirely**, and remove all
the spawn/pipe/stderr-parse plumbing — but FFmpeg's *code* still does the actual
codec work, now in-process through node-av instead of via a child process.

## 6. Benefits

1. **One decode codebase.** Client and server both use core's
   `MediabunnyVideoSource`/`MediabunnyAudioSource`. Frame-accuracy parity
   between preview and final render for free.
2. **Real server-side audio.** Decode + mix in JS with full control (gain
   automation, resampling, overlap) instead of the ffmpeg filtergraph string we
   can barely extend. Closes the audio gap noted in the client migration doc.
3. **Single-pass, no temp files.** One `Output` with video+audio tracks →
   `finalize()`. `renderToStream` pipes straight to the consumer via
   `StreamTarget`.
4. **Massive code deletion.** `video-source-ffmpeg.ts` (~393), `ffmpeg.ts`
   filtergraph/encoder builders (~169), `ffmpeg-bin.ts`, the regex probe — all
   gone or shrunk. Less brittle byte/string handling.
5. **No `-i pipe` stderr-regex probing.** Mediabunny exposes
   width/height/duration/fps/codec as typed track metadata.
6. **Performance.** Automatic hardware acceleration, multithreading, zero-copy,
   O(1) memory — claimed across all platforms, vs. our hand-rolled 6-frame
   buffer cap and `clearRect` memory workaround.
7. **Self-contained install.** node-av downloads matched prebuilt FFmpeg libs;
   no reliance on the `@ffmpeg-installer` static binary version.

## 7. Resolved questions & decisions

Every item that was previously an open risk now has a concrete answer from the
mediabunny API. None of them block the migration.

- **R1 — skia → mediabunny frame handoff (encode). RESOLVED.** Don't use
  `CanvasSource` (it wants a DOM/Offscreen canvas). Use **`VideoSampleSource`**
  fed with a `VideoSample` built from the raw RGBA buffer we already extract:
  `new VideoSample(raw, { format: "RGBX", codedWidth, codedHeight, timestamp })`,
  then `source.add(sample, 1/fps)`. The constructor takes a raw `ArrayBuffer`/
  typed array directly; pixel-format conversion to the encoder's `yuv420p` is
  handled internally. No canvas-type interop at all.
- **R2 — mediabunny → skia frame handoff (decode). RESOLVED.** Don't rely on
  `CanvasSink` in Node. Use **`VideoSampleSink`**, which yields raw
  `VideoSample`s; blit each onto the skia canvas with `sample.draw(skiaCtx, 0, 0)`
  (it mirrors `drawImage` and handles rotation) or `sample.copyTo(buf)` +
  `putImageData` — the exact blit `FfmpegVideoSource._paint` already does. Make
  the sink (`CanvasSink` in browser, `VideoSampleSink` in Node) selectable by
  `environment` inside core's `MediabunnyVideoSource`.
- **R3 — quality mapping. RESOLVED (decision: bitrate presets).** Mediabunny
  encoders are bitrate-oriented (no CRF); `bitrate` accepts a bits/sec number or
  a `QUALITY_LOW | QUALITY_MEDIUM | QUALITY_HIGH` constant (resolution-aware).
  Map our four presets to target bitrates, defaulting to `QUALITY_HIGH` when
  unset (mediabunny's own default). Re-tune the `low/medium/high/max` table once
  and update [`renderer.md`](./renderer.md). The CRF→bitrate change is the one
  user-visible behavior difference.
- **R4 — audio in Node. RESOLVED (no Web Audio, no extra dep).** Use
  **`AudioSampleSource`** + `new AudioSample({ data: Float32Array, format:
  "f32-planar", numberOfChannels, sampleRate, timestamp })`. We mix clips into a
  plain `Float32Array` ourselves (§4) — no `OfflineAudioContext` polyfill, no
  `AudioBuffer`. (`AudioSample.fromAudioBuffer()` exists too, but we don't need
  it.)
- **R5 — native addon footprint. ACCEPTED (genuine cost, mitigated).** node-av
  ships per-platform prebuilt FFmpeg binaries downloaded on install. This is the
  one real tradeoff: CI must fetch the target binary (`npm install --os=… --cpu=…`),
  Docker/serverless images grow, and offline installs need the binaries cached.
  Verdict: acceptable — it replaces the `@ffmpeg-installer` static binary (also a
  platform-specific download) with one that's faster (in-process, HW-accelerated)
  and self-matched to the bindings. Document the CI flags in `renderer.md`.
- **R6 — codec/format coverage. RESOLVED.** Mediabunny encodes the formats we
  ship: MP4 (H.264/AVC video + AAC audio) and WebM (VP9/Opus). The fragmented
  output that `renderToStream` needs maps to `new Mp4OutputFormat({ fastStart:
  "fragmented" })`; for normal files, `fastStart: "in-memory"` gives a web-ready
  moov-at-front (better than today's plain output). Note: AAC *encode* depends on
  the underlying build — node-av's bundled FFmpeg provides it server-side
  (unlike some browsers), so no extension package is needed in Node.
- **R7 — GL transitions. RESOLVED (no change).** The headless-gl compositor
  (`gl.ts`/`gl-node.ts`) operates on the skia canvas *before* the encode step and
  is independent of how frames are encoded. It stays exactly as-is; we just feed
  its composited skia canvas into `VideoSample` instead of the ffmpeg encoder
  pipe.
- **R8 — dependency changes. RESOLVED.** Add `@mediabunny/server` (renderer-only)
  and `mediabunny` (already a dep of core — add to renderer too, or rely on
  core's). Remove `@ffmpeg-installer/ffmpeg`. `skia-canvas` and optional `gl`
  stay. No peerDep changes for consumers.

## 8. Phased rollout

1. **Spike (validate R1/R2/R4 in practice).** Standalone script:
   `registerMediabunnyServer()`, decode one mp4 via `Input`/`VideoSampleSink`,
   blit to skia, re-encode via `Output`+`VideoSampleSource` to a file. Then add
   one audio clip mixed into a `Float32Array` → `AudioSampleSource`. The APIs are
   confirmed (§7); this just exercises the skia⇄sample handoff and audio mux
   end-to-end before committing to the rewrite.
2. **Encode/mux (Layer B).** Rewrite `render.ts` encode path to a single
   `Output`; keep `FfmpegVideoSource` for decode initially (isolate variables).
   Wire `FilePathTarget`/`StreamTarget`. Remove the temp-file two-pass.
3. **Audio.** Replace `NullAudioSource` + `buildMuxArgs` audio with the JS
   decode+mix path. Validate against current ffmpeg output (levels, sync,
   rate/volume).
4. **Decode (Layer A).** Swap `setupServerRendering` factories to core's
   mediabunny sources; delete `video-source-ffmpeg.ts` and the regex probe.
5. **Drop ffmpeg CLI.** Remove `@ffmpeg-installer/ffmpeg`, `ffmpeg-bin.ts`,
   `setFfmpegPath`/`ffmpegPath` option. Update `package.json` exports/deps,
   [`renderer.md`](./renderer.md), and `doc/README.md`.
6. **Verify.** Render the demo comps server-side; compare frame hashes and audio
   to the ffmpeg baseline; check memory and wall-clock with HW accel on.

## 9. Decisions (made)

All previously-open choices now have a default:

- **Audio mixing:** manual `Float32Array` mix → `AudioSample` → `AudioSampleSource`.
  No Web Audio / `OfflineAudioContext` dependency (R4).
- **Decode source:** reuse core's `MediabunnyVideoSource`, with an
  environment-selected sink (`VideoSampleSink` in Node, `CanvasSink` in browser).
  Delete the renderer-local `FfmpegVideoSource` (R2).
- **Quality:** bitrate-based presets (map the existing four to target bitrates /
  `QUALITY_*` constants), accepting the CRF→bitrate behavior change (R3).
- **ffmpeg CLI:** drop `@ffmpeg-installer/ffmpeg` and all CLI plumbing; keep
  `skia-canvas` and optional `gl` (R5/R7/R8).

The only thing still warranting a real-world check (not a design decision) is the
spike in §8.1 confirming the skia⇄`VideoSample` pixel handoff and audio sync on a
non-trivial clip.
