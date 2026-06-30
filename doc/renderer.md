# `@smoove/renderer`

Headless server renderer for smoove. It rasterizes a live `Composition`
frame-by-frame with [skia-canvas](https://skia-canvas.org) (via konva 10's
official `konva/skia-backend`), then encodes and muxes video + audio with
[Mediabunny](https://mediabunny.dev) — all in Node, no browser, no bundler, **no
ffmpeg CLI**.

```
build comp (you) → renderer drives frames → RGBA → Mediabunny Output → file | stream
                      renderFrame(f)      captureCanvas()    VideoSample +
                   (awaits delayRender)  .toBufferSync("raw")  mixed AudioSample
```

Encoding goes through `@mediabunny/server`, which wraps
[node-av](https://github.com/seydx/node-av) (N-API bindings to the FFmpeg C API).
So FFmpeg still does the codec work, but **in-process** — there is no spawned
`ffmpeg` binary and no `@ffmpeg-installer` dependency.

## Install

```sh
pnpm add @smoove/renderer @smoove/core konva
```

`skia-canvas` (the konva rasterization backend) and `@mediabunny/server` (node-av
→ FFmpeg) are both native (N-API) addons with per-platform prebuilt binaries
downloaded on install. In a pnpm workspace you must allow their build scripts
(`onlyBuiltDependencies` / `pnpm approve-builds`). For cross-platform CI, fetch
the right node-av binary with npm's `--os` / `--cpu` flags. Node-first; Bun is
best-effort.

## Setup (call before building any composition)

The renderer must be wired up **before** you construct a `Composition`, because
media nodes (`Image`/`Audio`/`Video`) build their source/loader eagerly at
construction. Setup installs the skia backend, sets the global rendering flag,
and registers Node-safe defaults.

```ts
import "@smoove/renderer/register"; // sugar === setupServerRendering()
// or:
import { setupServerRendering } from "@smoove/renderer";
setupServerRendering({ fonts: ["./Inter.ttf"] }); // idempotent
```

Always also pass `mode: "rendering"` to the `Composition` (the flag is resolved
once at construction; the renderer validates `comp.environment.isRendering` and
throws a clear error otherwise).

## High-level API

### `renderComposition(comp, opts): Promise<RenderResult>`

Render frames + audio to a muxed video file.

```ts
const result = await renderComposition(comp, {
  output: "out.mp4",
  resolution: { width: 1920, height: 1080 }, // default: comp native size
  fit: "contain",                            // "contain" (letterbox) | "cover" (crop)
  quality: "high",                           // preset name or { videoBitrate, audioBitrate }
  fps: 30,                                   // default: comp.fps
  format: "mp4",                             // "mp4" (H.264/AAC) | "webm" (VP9/Opus)
  range: { from: 0, to: 89 },                // inclusive; default: whole comp
  mute: false,
  signal: abortController.signal,
  onProgress: (p) => {/* { frame, total, fps, etaSeconds } */},
});
// RenderResult: { output, width, height, frames, durationInSeconds, hasAudio }
```

The stage renders at the comp's native size; frames are resampled in skia to
`resolution` before encode (`contain` → letterbox, `cover` → crop).

### `renderToStream(comp, opts): { stream, done }`

Render to a fragmented container (`Mp4OutputFormat({ fastStart: "fragmented" })`)
written straight into a `Readable` — **no temp file** — plus a `done` promise
that resolves with the `RenderResult` (or rejects), so errors/results aren't lost.

```ts
const { stream, done } = renderToStream(comp, { quality: "medium" });
stream.pipe(res); // e.g. an HTTP response
const result = await done;
```

### `renderStill(comp, opts): Promise<Buffer>`

Render one frame to a PNG/JPEG buffer (optionally written to `output`).

```ts
const png = await renderStill(comp, { frame: 30, output: "thumb.png", type: "png" });
```

## Building blocks

- **`renderFrames(comp, opts?): AsyncGenerator<RenderedFrame>`** — the primitive
  everything above is built on. Yields `{ index, frame, data /* raw RGBA */,
  width, height }` at the comp's native size. Use it for custom encoders, GIFs,
  image sequences, or per-frame upload.

  ```ts
  for await (const { frame, data, width, height } of renderFrames(comp)) {
    // data is width*height*4 bytes of RGBA
  }
  ```

- **`collectAudioTrack(comp, fps): AudioClip[]`** — the audio coalescing logic.
  Groups the per-frame samples from `comp.getAudioAssets()` by source, drops
  muted ones, and splits each into runs of consecutive frames → clips
  (`{ id, src, startFrame, endFrame, mediaInSeconds, playbackRate, volume }`,
  where `volume` is a constant or a time-keyed envelope).

- **`probeComposition(comp): CompositionInfo`** — `{ fps, durationInFrames,
  width, height, durationInSeconds, isRendering }`, no render.

## Wiring seams

- **`setupServerRendering(opts?)`** — install backend + flag + Node defaults.
  `opts.video` overrides the default `VideoSource` factory; `opts.fonts`
  registers fonts via skia-canvas `FontLibrary.use`.
- **`installSkiaBackend()`** — just the konva skia backend (subset of setup).
- **`nodeVideoSourceFactory` / `MediabunnyVideoSource`** — the decoder-backed
  video source (Mediabunny `VideoSampleSink` → skia canvas).
- **`nullAudioSourceFactory` / `NullAudioSource`** — no-op audio source so the
  `Audio` node constructs in Node; audio is decoded + mixed separately at finalize.
- **`mixAudio(clips, fps, totalSeconds, fromFrame?)`** — decode + mix the
  coalesced audio clips into one interleaved-stereo `Float32Array` (48 kHz).
- **`loadImageNode(src)`** — the skia-canvas image loader.
- **`registerServerMedia()`** — register `@mediabunny/server` (idempotent;
  `setupServerRendering` calls it).
- **`import "@smoove/renderer/register"`** — `setupServerRendering()` at
  import time.

## Quality presets

Mediabunny encoders are bitrate-oriented (no CRF): `videoBitrate` is a bits/sec
number or one of Mediabunny's resolution-aware quality constants.

| preset | video bitrate | audio bitrate |
| --- | --- | --- |
| `low` | `QUALITY_LOW` | 96k |
| `medium` | `QUALITY_MEDIUM` | 128k |
| `high` | `QUALITY_HIGH` | 192k |
| `max` | `QUALITY_VERY_HIGH` | 256k |

Pass a `QualityConfig` (`{ videoBitrate, audioBitrate }`) — each a bits/sec
number or a Mediabunny `Quality` constant — for full control.

## Custom fonts

There is no DOM `@font-face` in Node. Register fonts with skia-canvas before
drawing text, via the `fonts` option on `setupServerRendering`/`SetupOptions`
(or the per-call `fonts` option on render/still). Accepts an array of font file
paths or a `{ family: paths }` record.

## How it works (notes)

- **Frame loop.** `renderFrames` validates rendering mode, then loops
  `await comp.renderFrame(f)` — which applies the frame to every sequence and
  awaits all outstanding `delayRender` handles (async image/video loads) before
  resolving — and captures pixels via `comp.captureCanvas().toBufferSync("raw")`.
- **One pass, one Output, no temp files.** A single Mediabunny `Output` holds a
  `VideoSampleSource` and (when the comp has audio nodes) an `AudioSampleSource`.
  Each rendered frame is wrapped as a `VideoSample` (`format: "RGBA"`) and added;
  the audio track is mixed and added after the frame loop, then one `finalize()`
  muxes the container. The audio-track existence is decided up front by scanning
  the comp for `Audio`/`Video` nodes (audio timing is only known after rendering,
  and re-rasterizing or buffering raw frames just to learn it would be wasteful).
- **Audio.** During rendering `Audio` (and `Video`, for its soundtrack) records
  one sample per frame (`getAudioAssets()`); `collectAudioTrack` coalesces them
  into clips; `mixAudio` decodes each clip with a Mediabunny `AudioSampleSink`,
  resamples for `playbackRate` + source rate, applies the volume envelope, delays
  to the start frame, and sums everything into one interleaved-stereo
  `Float32Array` fed to the encoder as `AudioSample`s.
- **Video.** `MediabunnyVideoSource` pulls decoded frames from a forward-only
  `VideoSampleSink` iterator at the exact media time, copies each `VideoSample`
  (RGBA) onto a reused skia canvas the konva backend draws, and reseeds only on a
  backward jump or large skip. Seeks are coalesced (one decode in flight, latest
  target wins).

## Core changes this package depends on

- A globally-overridable default source/loader registry in `@smoove/core`
  (`setDefaultVideoSourceFactory` / `setDefaultAudioSourceFactory` /
  `setDefaultImageLoader`) so Node-safe sources can replace the browser ones
  before comps are built.
- `Image` gained an injectable loader and gates its async load with
  `delayRender` during rendering (so `renderFrame(0)` isn't captured blank).
- `Composition.captureCanvas()` composites visible layers into one canvas for a
  full-frame capture.

## Performance notes

- **In-process, hardware-accelerated codecs.** Encode and decode run through
  node-av's FFmpeg C bindings (zero-copy paths, multithreading, automatic HW
  acceleration where available) instead of piping raw bytes to a child process.
- **Video decode is streaming.** `MediabunnyVideoSource` advances one
  `VideoSampleSink` iterator forward and reseeds only on a backward jump or large
  skip — the common forward render never pays a re-seek.
- **Capture reuses one canvas.** `captureCanvas()` composites into a single
  reused scratch canvas; allocating a fresh skia `Canvas` per frame leaks native
  memory and collapses throughput.
- **Clear before each per-frame blit.** skia-canvas retains a native,
  GC-invisible snapshot of a canvas's *prior* content every time you draw onto it
  without first clearing — so an uncleared per-frame `putImageData`/`drawImage`
  leaks ~one frame of RSS per frame for the process lifetime. The video source
  `clearRect`s its blit canvas before each frame, so video memory stays **flat
  and bounded — independent of render length**. konva's `Layer.drawScene` and
  `Composition.captureCanvas` already clear before drawing. **`videoDecodeCap`**
  (a `setupServerRendering` option, or `setVideoDecodeCap(w, h)`) is an *optional*
  throughput/size knob — decode a dimmed/background bed at a smaller size Konva
  upscales anyway.

## Out of scope (v1)

- Multi-process / parallel rendering. A live `Composition` isn't serializable
  across workers (Konva nodes aren't serializable); the renderer runs in one
  process. With the per-frame clear in place, **memory is bounded single-process
  even for long video**, so parallel is now purely a *speed* optimization (a comp
  *factory* builds one comp per worker, each renders a range) — not a memory
  necessity. Future work.
- GPU rendering (`canvas.gpu`), image-sequence file output.
