# Implementation prompt — parallel rendering for `@smoove/renderer`

> Paste into a fresh Claude Code session in the `smoove` repo. Self-contained.
> Goal: add **parallel rendering to `renderComposition`** (default on) where many
> processes produce **visual frames** and a **single ffmpeg** stitches raw frames →
> encoded video + muxed audio in one pass. (A prior segmented/concat attempt was
> built and reverted — see below; build this fresh.)

---

> **⚠️ Premise update (2026-06-09): the memory justification below is OBSOLETE.**
> The "skia-canvas leaks, only process exit reclaims it" claim was root-caused and
> **fixed in-process with a one-line change**. The leak was *drawing onto an
> uncleared canvas* — skia-canvas retains a snapshot of a canvas's prior content
> on every uncleared draw. The video source now `clearRect`s before its per-frame
> `putImageData` (`FfmpegVideoSource._paint`), and memory is now **flat and
> bounded in a single process even for long full-res video** (verified: same RSS
> at 600 and 1200 frames, no `videoDecodeCap`). konva's `drawScene`/`captureCanvas`
> already cleared, so they never leaked. **Net effect for this doc: parallel
> rendering is now a pure SPEED optimization, NOT a memory necessity.** Build it
> that way — no memory-driven chunking, no need for short-lived fork-per-chunk
> workers (you may reuse long-lived workers or `worker_threads`); `framesPerChunk`
> is just a scheduling knob. The single-encode + IPC-streaming architecture below
> is still a fine *shape* for the speed feature; ignore the leak-reclaim framing.

## Why this exists (read first — but see the premise update above)

`@smoove/renderer` rasterizes a smoove `Composition` headlessly with
**skia-canvas** and encodes with **ffmpeg**. Single-process rendering works
(`renderComposition`). It *used to* leak native memory on video renders — every
uncleared per-frame blit retained an `SkImage` for the process lifetime — which
originally motivated rendering across **multiple short-lived processes** so each
exit reclaimed the leak. **That leak is now fixed** (see the premise update), so
this motivation no longer holds; what remains is the **speed** win of rendering
ranges in parallel.

## What we tried (and reverted) — learn from it

A first attempt at parallelism was built and then **reverted** (it is *not* in the
codebase; the renderer is back to clean single-process). It worked like this:
1. Render each frame range to its **own encoded mp4** in a worker.
2. `ffmpeg -f concat -c copy` the segment mp4s into one video.
3. Mux the full audio over the concatenated video.

Why it was the wrong shape:
- **Audio desync (the dealbreaker).** Concatenating independently-encoded H.264
  segments with `-c copy` is **not frame-exact** — GOP/timestamp boundaries drift a
  frame or two per segment, accumulating. The audio is muxed by absolute time, so
  it slides out of sync against the drifting video.
- **Messy API.** A parallel `SegmentedRenderOptions` / `SegmentEntry` /
  `renderInSegments` / worker-protocol surface grew next to the single-process
  path, and per-segment encoding is wasteful (N encodes + 1 concat).

Keep the *idea* (factory + worker processes + per-process memory reclaim), change
the *mechanism* to the single-encode architecture below. Build it **fresh** — there
is nothing to delete.

## Target architecture

```
                                ┌─ worker 0: render frames [0..k)   → raw0.rgba ─┐
factory module (buildComp) ─────┼─ worker 1: render frames [k..2k)  → raw1.rgba ─┤
(imported per worker)           └─ worker N: render frames [..total) → rawN.rgba ─┘
                                                         │ (each worker exits → leak reclaimed)
                                                         ▼
parent: stream raw chunk files IN ORDER ──► ONE ffmpeg ──► final.mp4
        + merged audio clips (global frames)    (raw video stdin + audio inputs,
                                                  single encode + amix + mux pass)
```

- **Workers = visuals only.** Each renders a contiguous frame range to a **raw
  RGBA file** (fixed `W*H*4` bytes/frame, byte-concatenable) and reports its
  range's **audio clips** (with global composition frames) back over IPC. Then it
  **exits** (reclaims skia memory). A live `Composition` can't cross processes, so
  each worker rebuilds a fresh comp from a **factory module**.
- **Parent = one stitching process.** Spawns a **single ffmpeg** that reads raw
  RGBA from stdin (input 0), takes the audio source files as further inputs, and
  in **one pass** encodes the video (libx264) + builds the audio graph + `amix` +
  muxes. The parent streams the raw chunk files to ffmpeg's stdin **in global
  frame order** (await chunk i, pipe its bytes, then i+1), honoring backpressure.

Why this fixes sync: the video is **one continuous encode** of an exact,
gapless raw stream → frame-exact. The audio is muxed once over the whole timeline
by global-frame `adelay` → it lines up perfectly. No concat, no drift.

## Detailed design

### Composition factory (unchanged idea)
`entry: string | { module: string; export?: string }`. The module exports
`() => Composition | Promise<Composition>` with **side-effect-light top level**
(don't render at import). Workers `import()` it; the parent imports it once to
read metadata (fps, durationInFrames, width, height) then `comp.destroy()`s.
Keep `examples/mixer-comp.ts` (already a clean factory) for the demo.

### Worker (`render frames → raw file + clips`)
Replace the current `renderRangeToVideo` (which encodes a segment mp4) with a
`renderRangeToRaw(comp, { fps, range, fonts, signal, onFrame }) →
{ rawFile, clips, frameCount }`:
- `comp.clearAudioAssets()`, loop `framesBetween(comp, from, to)` (exists in
  `render.ts`), write each frame's `data` (raw RGBA from `comp.captureCanvas()
  .toBufferSync("raw")`) to a `*.rgba` file (a plain `fs.createWriteStream`,
  honoring `write()===false` → `drain`).
- After the loop, `clips = collectAudioTrack(comp, fps)` (exists; global frames).
- Return `{ rawFile, clips, frameCount }`. Worker posts `{type:"result", rawFile,
  clips, frameCount}` over IPC and `process.exit(0)`.
- No per-worker encoding, no per-worker ffmpeg. Workers are pure visual producers.

### Parent (`one ffmpeg`)
1. Setup + import factory + build once for metadata; compute total frames; split
   `[from,to]` into chunks (size = `framesPerChunk`, default ~150 — this bounds
   per-process memory). Each chunk = one worker invocation = one process.
2. Run the chunk pool with `processes` workers in flight (default `min(cpus-1, 4)`).
   Collect `{rawFile, clips}` per chunk index. Fork with
   `{ execArgv: process.execArgv }` so `.ts` factories load under `tsx`.
3. **Merge audio clips** across all chunks (sorted by index). A clip split across a
   chunk boundary (same `id`, `src`, `playbackRate`, contiguous `mediaInSeconds`,
   `endFrame+1 === next.startFrame`) **should be merged** back into one clip so the
   `volume` envelope is continuous and you don't emit redundant ffmpeg inputs.
4. Spawn **one** ffmpeg: raw video on stdin + the merged clips' source files as
   inputs; build the **combined** filtergraph (see below); encode + mux; write the
   final output.
5. **Stream the raw chunk files to ffmpeg stdin in order.** For chunk index 0,1,2…:
   await that chunk's worker result, then pipe `createReadStream(rawFile)` into
   `ffmpeg.stdin` (do not `end()` until the last chunk), honoring backpressure.
   Delete each raw file after it's streamed.
6. On all done, `ffmpeg.stdin.end()`, await ffmpeg close, cleanup, return
   `RenderResult`.

### The single ffmpeg command (the key new ffmpeg builder)
Merge today's `spawnVideoEncoder` (raw→H.264) and `buildMuxArgs` (audio graph)
into ONE invocation. Today they're two passes; now it's one:

```
ffmpeg -y -f rawvideo -pix_fmt rgba -s WxH -framerate FPS -i pipe:0        # input 0 = raw video
       [ -ss <clip.mediaInSeconds> -i <clip.src> ]*                        # inputs 1..N = audio
       -filter_complex "
          [0:v] <scale/pad or crop if resolution set> [v];                 # omit → map 0:v
          [1:a] atrim=0:<srcDur>,asetpts=PTS-STARTPTS,<atempo*>,<volume>,adelay=<startFrame/fps ms>:all=1 [a0];
          ...
          [a0][a1]... amix=inputs=N:normalize=0:dropout_transition=0 [aout]
       "
       -map "[v]" (or 0:v) -map "[aout]"
       -c:v libx264 -pix_fmt yuv420p -preset <P> -crf <C> -c:a aac -b:a <B>
       out.mp4
```
- The per-clip audio chain is exactly today's `buildMuxArgs` chain (`atrim` /
  `atempo` / `volume` envelope / `adelay`) — **reuse `atempoChain` and
  `volumeFilter` from `ffmpeg.ts` verbatim**. `adelay` uses the **global** start
  frame (the whole timeline is in one pass; `frameOffset` is 0 here).
- If `resolution`/`fit` is set, the video scale/pad/crop goes **inside**
  `filter_complex` as `[0:v]...[v]` and you `-map "[v]"` (you can't combine `-vf`
  with `-filter_complex`). If not, `-map 0:v`. Reuse `buildScaleFilter`'s
  expression, just emitted as a filter_complex node.
- No audio (mute / no clips) → drop the audio inputs/graph, `-map 0:v`, `-an`.

### Memory & disk
- Per-process memory ≈ `framesPerChunk × W×H×4` (bounded; reclaimed on exit). Peak
  ≈ `processes × that`. Tune via `framesPerChunk` (memory) and `processes` (speed).
- Disk: total raw ≈ `W×H×4 × totalFrames` (e.g. 1080² × 780 ≈ 3.5 GB), written to
  `os.tmpdir()` and **deleted as each chunk is streamed** (so at most ~`processes`
  chunk files + already-produced-not-yet-streamed ones exist at once). If that's a
  concern for very long renders, a follow-up can stream worker stdout → parent
  directly, but disk-backed raw is the simplest correct v1 and keeps full overlap.

## Public API — ONE entry, parallel by default

`renderComposition` is the single entry; `parallel` defaults to **true**. Don't
add a second `renderParallel` / `SegmentedRenderOptions` surface.

```ts
renderComposition(
  source: Composition | CompositionRef,
  opts: RenderOptions & {
    parallel?: boolean;       // default TRUE
    processes?: number;       // workers in flight; default min(cpus-1, 4)
    framesPerChunk?: number;  // per-worker memory bound; default ~150
  },
): Promise<RenderResult>;

// Parallel workers run in separate processes and must REBUILD the comp
// (a live Composition instance cannot cross a process boundary — Konva nodes
// aren't serializable), so parallel requires a module-importable factory:
type CompositionRef = { module: string; export?: string }; // export defaults to "default"
```

**The unavoidable constraint:** a live `Composition` can only be rendered in the
process that built it. So `parallel` actually engages based on what `source` is:

- `source` is a **`CompositionRef`** and `parallel !== false` → **multi-process
  parallel** (the blessed, default path). Workers `import()` the module, build a
  fresh comp, and render raw-frame chunks; one ffmpeg stitches. The demo uses this.
- `source` is a **live `Composition`** → **single-process** (can't fork a live
  instance). `parallel` is ignored when left at its default; if the caller
  *explicitly* set `parallel: true` with a live comp, **throw a clear error**
  telling them to pass `{ module, export }` for parallel.
- `source` is a `CompositionRef` with `parallel: false` → single-process via the
  factory (parent builds once and renders) — handy for debugging.

Default-true means: pass a factory ref → you get parallel + leak-proof + fast with
zero extra flags; pass a live comp → it just works single-process. You only set
`parallel: false` for debugging or tiny renders.

Notes:
- Reuse `RenderOptions` for everything shared (output, resolution, fit, quality,
  fps, range, fonts, mute, ffmpegPath, signal, onProgress). Only `parallel`,
  `processes`, `framesPerChunk` are new.
- `videoDecodeCap` is a *setup* option — forward it to the workers' setup; usually
  unneeded with parallel (process exit reclaims the leak), so the parallel demo
  runs **full-res, no cap** to prove it. `onProgress` aggregates across workers.
- Keep `renderStill` / `renderFrames` / `renderToStream` single-process,
  instance-based — they don't benefit from forking.
- Internally, put the orchestrator in `parallel.ts` and the worker entry in a
  `*-worker.ts`; `renderComposition` just dispatches to it when `source` is a ref
  and `parallel` is on.

## What exists to build on (all present in the repo today)

Reuse as-is:
- `framesBetween`, `renderFrames`, `renderToFile`, `capture` (`render.ts`); the raw
  capture is `comp.captureCanvas().toBufferSync("raw", { colorType: "rgba" })`.
- `collectAudioTrack` (`audio-track.ts`) — clips with **global** frames + envelope
  simplification (RDP) already done.
- `atempoChain`, `volumeFilter`, `buildScaleFilter`, `buildMuxArgs`,
  `spawnVideoEncoder`, `runFfmpeg` (`ffmpeg.ts`) — the per-clip audio chain in
  `buildMuxArgs` (`atrim`/`atempo`/`volume`/`adelay` from the global `startFrame`)
  is exactly the graph the single-pass builder needs.
- `setupServerRendering` / `videoDecodeCap` (`setup.ts`), the streaming
  `FfmpegVideoSource` (`video-source-ffmpeg.ts`), reused-canvas
  `Composition.captureCanvas()` (core), `resolveQuality` (`render.ts`).

New things to add:
- `renderRangeToRaw(comp, { fps, range, fonts, signal, onFrame }) → { rawFile,
  clips, frameCount }` in `render.ts` (or a new module) — loops `framesBetween`,
  writes each frame's raw RGBA to a `.rgba` file, returns `collectAudioTrack(comp,
  fps)`. **No encode.**
- A single-pass ffmpeg builder/spawner in `ffmpeg.ts`, e.g.
  `spawnEncodeAndMux({ width, height, fps, resolution, fit, quality, clips,
  output, ffmpegPath }) → ChildProcess` — raw RGBA on stdin (input 0) + audio
  source files (inputs 1..N) + the combined filtergraph + libx264 encode + `amix`
  + mux, **all in one process**. It merges what `spawnVideoEncoder` and
  `buildMuxArgs` do today into a single invocation. Reuse `atempoChain` /
  `volumeFilter` for the audio chains and `buildScaleFilter`'s expression (emitted
  as a `filter_complex` `[0:v]...[v]` node when `resolution` is set).
- A new `parallel.ts` orchestrator (forks workers, streams raw chunk files to the
  single ffmpeg in order, merges clips) + a `*-worker.ts` entry. `renderComposition`
  dispatches to it when `source` is a `CompositionRef` and `parallel` is on.
- An `examples/mixer-comp.ts` factory (extract the comp builder out of
  `examples/audio-mixer.ts`) + a runner that calls
  `renderComposition({ module: ".../mixer-comp.ts", export: "buildMixerComp" }, …)`.
- Update the `renderComposition` types/overloads (accept `Composition |
  CompositionRef`; add `parallel`/`processes`/`framesPerChunk`) + a parallel
  section in `doc/renderer.md`.

Note: the single-pass path needs **no frame offset** — the whole timeline is one
continuous encode and `buildMuxArgs`'s `adelay = startFrame/fps` (global frame) is
already correct. (Single-process *ranged* audio renders, `range.from > 0`, have a
pre-existing latent offset bug; out of scope here, fix separately if needed.)

## Gotchas (hard-won — honor these)

- **Even dimensions.** libx264 + `yuv420p` rejects odd width/height (EPIPE on the
  raw write). Comp size must be even; if `resolution` is odd, round.
- **Frame ordering is mandatory.** The single encoder reads a sequential raw
  stream; stream chunk files strictly in index order. Out-of-order = garbled video.
- **Backpressure** both when workers write raw to disk and when the parent pipes
  raw to ffmpeg stdin (`write()===false` → await `"drain"`).
- **tsx workers:** `fork(worker, [params], { execArgv: process.execArgv })` so the
  `.ts` factory loads. `process.execArgv` under tsx carries `--import .../loader`.
- **Audio clips already carry global frames** (`collectAudioTrack`). In the single
  pass `adelay = startFrame/fps` (frameOffset 0). Merge boundary-split clips.
- **Streaming video decoder** (`FfmpegVideoSource`) restarts on backward jumps;
  each worker rendering a forward range is the happy path. Loop wrap = a restart.
- **Don't reintroduce concat.** The whole point is one continuous encode.
- **skia leak is per-process** — never reuse a worker for multiple chunks without
  letting it exit between them, or the leak accumulates within that process.

## Definition of done

- `renderComposition({ module: ".../mixer-comp.ts", export: "buildMixerComp" },
  { output, processes: 4, framesPerChunk: 150, quality: "medium" })` (parallel by
  default) produces the 26s 1080² mixer demo **at full-res video, no decode cap**,
  with **perfect A/V sync** and bounded per-process memory.
- **Verify sync rigorously** (this is the bug being fixed): the demo has known
  audio events at known frames (ducking under voice ~f120–404, master MUTE
  f468–498, A→B crossfade f540–630). Extract video frames at those timecodes and
  confirm the on-screen captions/meters match, AND check the output audio RMS
  windows line up with those frames (no drift vs. the single-process render).
- The single-process paths (a live `Composition`, or `parallel: false`),
  `renderStill`, and `renderFrames` still work and produce identical output.
  `pnpm build` + `pnpm check` clean. Parallel is faster than single-process.
- `doc/renderer.md` updated: document `renderComposition`'s parallel mode (one
  ffmpeg, raw frames, why it's frame-exact), fold it into the skia-leak Performance
  note's mitigation list, and update the "Out of scope" note (which currently lists
  a segmented render as future work).

## Repo conventions (quick)
pnpm workspaces; `core` extends Konva (konva is a peerDep, now `>=10`); build =
`tsc -b` per package; Biome (`pnpm check`/`pnpm format`); public API via the
barrel `src/index.ts`; `./register` subpath = `setupServerRendering()`. Examples
run via `tsx`; build before running (`pnpm --filter @smoove/renderer build`).
