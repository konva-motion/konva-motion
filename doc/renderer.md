# `@konva-motion/renderer`

Headless server renderer for konva-motion. It rasterizes a live `Composition`
frame-by-frame with [skia-canvas](https://skia-canvas.org) (via konva 10's
official `konva/skia-backend`), pipes raw RGBA to ffmpeg, and muxes the
composition's audio — all in Node, no browser, no bundler.

```
build comp (you) → renderer drives frames → raw RGBA → ffmpeg → file | stream
                      renderFrame(f)      captureCanvas()      audio from
                   (awaits delayRender)  .toBufferSync("raw")  getAudioAssets()
```

## Install

```sh
pnpm add @konva-motion/renderer @konva-motion/core konva
```

`skia-canvas` is a native (N-API) addon and `@ffmpeg-installer/ffmpeg` ships a
per-platform ffmpeg binary; both are dependencies of the package. In a pnpm
workspace you must allow their build scripts (`onlyBuiltDependencies` /
`pnpm approve-builds`) so the native binary downloads. Node-first; Bun is
best-effort.

## Setup (call before building any composition)

The renderer must be wired up **before** you construct a `Composition`, because
media nodes (`Image`/`Audio`/`Video`) build their source/loader eagerly at
construction. Setup installs the skia backend, sets the global rendering flag,
and registers Node-safe defaults.

```ts
import "@konva-motion/renderer/register"; // sugar === setupServerRendering()
// or:
import { setupServerRendering } from "@konva-motion/renderer";
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
  quality: "high",                           // preset name or { crf, preset, audioBitrate }
  fps: 30,                                   // default: comp.fps
  range: { from: 0, to: 89 },                // inclusive; default: whole comp
  mute: false,
  ffmpegPath: "/usr/bin/ffmpeg",             // override the bundled binary
  signal: abortController.signal,
  onProgress: (p) => {/* { frame, total, fps, etaSeconds } */},
});
// RenderResult: { output, width, height, frames, durationInSeconds, hasAudio }
```

The stage renders at the comp's native size; ffmpeg resamples to `resolution`
(`contain` → `scale=…:decrease,pad=…`; `cover` → `scale=…:increase,crop=…`).

### `renderToStream(comp, opts): { stream, done }`

Render to a fragmented mp4 (`-movflags frag_keyframe+empty_moov`) exposed as a
`Readable`, plus a `done` promise that resolves with the `RenderResult` (or
rejects), so errors/results aren't lost.

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
- **`nodeVideoSourceFactory` / `FfmpegVideoSource`** — the decoder-backed video
  source (single-frame ffmpeg extraction → skia canvas).
- **`nullAudioSourceFactory` / `NullAudioSource`** — no-op audio source for
  Node (audio is never decoded during render; it's collected as metadata).
- **`loadImageNode(src)`** — the skia-canvas image loader.
- **`setFfmpegPath(path)` / `resolveFfmpegPath(opt?)`** — ffmpeg binary control.
- **`import "@konva-motion/renderer/register"`** — `setupServerRendering()` at
  import time.

## Quality presets

| preset | crf | x264 preset | audio |
| --- | --- | --- | --- |
| `low` | 28 | veryfast | 96k |
| `medium` | 23 | fast | 128k |
| `high` | 18 | slow | 192k |
| `max` | 14 | slower | 256k |

Pass a `QualityConfig` (`{ crf, preset, audioBitrate }`) for full control.

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
- **Two ffmpeg passes, one render pass.** Audio samples are only known *after*
  the render pass (they're collected per frame), so a single ffmpeg invocation
  can't know its audio inputs up front. The renderer encodes video in one render
  pass, then does a fast `-c:v copy` mux to lay the coalesced audio over it (no
  re-encode). The public API and result are unchanged.
- **Audio.** During rendering `Audio` records one sample per frame
  (`getAudioAssets()`); `collectAudioTrack` coalesces them into clips, each
  mapped to an ffmpeg input with `-ss` in-point, `adelay` start offset, `atempo`
  for `playbackRate`, and a `volume` filter (constant or time-keyed). Clips are
  `amix`ed into the output track.
- **Video.** `FfmpegVideoSource` extracts single frames with ffmpeg (`-ss`
  before `-i`, accurate by default) and blits them into an offscreen skia canvas
  the konva backend draws. Seeks are coalesced (one decode in flight, latest
  target wins).

## Core changes this package depends on

- A globally-overridable default source/loader registry in `@konva-motion/core`
  (`setDefaultVideoSourceFactory` / `setDefaultAudioSourceFactory` /
  `setDefaultImageLoader`) so Node-safe sources can replace the browser ones
  before comps are built.
- `Image` gained an injectable loader and gates its async load with
  `delayRender` during rendering (so `renderFrame(0)` isn't captured blank).
- `Composition.captureCanvas()` composites visible layers into one canvas for a
  full-frame capture.

## Performance notes

- **Video decode is streaming.** `FfmpegVideoSource` keeps one ffmpeg process
  decoding the clip sequentially and pulls frames off the pipe; it only restarts
  (a `-ss` seek) on a backward jump or large skip. Forward renders (the common
  case) avoid the ~200ms-per-frame cost of spawning a fresh ffmpeg per frame.
- **Capture reuses one canvas.** `captureCanvas()` composites into a single
  reused scratch canvas; allocating a fresh skia `Canvas` per frame leaks native
  memory and collapses throughput.
- **skia-canvas retains decoded video pixels.** skia-canvas holds the native
  pixels of every *distinct* frame fed to it (via `putImageData`/`Image`) for the
  process lifetime — V8's GC can't reclaim it. So a video render's memory scales
  with `frame area × distinct frames`. Mitigation: **`videoDecodeCap`** (a
  `setupServerRendering` option, or `setVideoDecodeCap(w, h)`) decodes clips at a
  smaller size — ideal for a dimmed/background bed Konva upscales anyway. For
  very long, full-resolution, unique-frame video, render in shorter ranges
  (`opts.range`) across separate processes and `concat`, or composite the video
  in ffmpeg. Non-video renders (shapes/text/images) are unaffected and stay flat.

## Out of scope (v1)

- Multi-process / parallel rendering. A live `Composition` isn't serializable
  across workers (Konva nodes aren't serializable); the renderer runs
  sequentially. A future **segmented render** (a comp *factory* builds one comp
  per worker, each renders a range, ffmpeg `concat`s) can drop into the loop, but
  needs a factory entry — not a live instance.
- GPU rendering (`canvas.gpu`), image-sequence file output.
