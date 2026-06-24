# TODO / Backlog

Tracked work for konva-motion. Each item: **what**, **why**, **where** (code
pointers), and **acceptance** notes. Ordered roughly by the original capture, not
priority.

---

## 1. Core: loop on a section (loop region)

**What** — Let a composition loop a sub-range `[in, out]` of the timeline, not
just the whole `[0, durationInFrames-1]` span.

**Why** — The studio already exposes loop-region UI (set in/out points, drag
grips, `I`/`O` shortcuts), but selecting a region does **not** actually loop on
it — playback still wraps the full composition. The region is purely cosmetic
today because core has no concept of a loop sub-range.

**Where**
- Core only knows a boolean loop and wraps the entire timeline:
  [composition.ts](../packages/core/src/engine/composition.ts) — `_loop`
  signal (`setLoop`/`loop`), and the wrap logic at the end of the tick
  (`atEnd`/`atStart` → `wrap = atEnd ? 0 : lastFrame`).
- Studio region UI that has nowhere to bind:
  [timeline-header.tsx](../packages/studio/src/components/timeline/timeline-header.tsx)
  ("Set loop start (I)" / "Set loop end (O)"),
  [region-handles.tsx](../packages/studio/src/components/timeline/region-handles.tsx),
  [scrubber.tsx](../packages/studio/src/components/timeline/scrubber.tsx),
  [transport.tsx](../packages/studio/src/components/timeline/transport.tsx)
  (`Looping ${regionActive ? "region" : "playback"}`).
- Player passthrough: [km-player.ts](../packages/player/src/km-player.ts)
  (`toggleLoop`/`isLooping`), [player-api.ts](../packages/player/src/player-api.ts).

**Acceptance** — Add a loop-region concept to `Composition` (e.g.
`setLoopRegion(in, out)` + a `loopRegion` signal; `null` = full timeline). The
play-tick wrap should clamp to the region bounds when set, and `seek`/`play`
should respect them. Thread it through player (`player-api`) and bind the studio
region UI to it. Decide whether the playhead snaps into the region when play
starts outside it.

---

## 2. Agent skill: video creation

**What** — A Claude Code skill that authors a konva-motion video end to end
(scaffold a composition, add sequences/shapes/media, render).

**Why** — Lower the barrier to producing a video with the library; give agents a
repeatable recipe instead of rediscovering the API each time.

**Where** — New skill under `.claude/skills/` (or wherever project skills live;
see `skills-lock.json`). Should lean on the public barrel API in
[packages/core/src/index.ts](../packages/core/src/index.ts) and the renderer
([packages/renderer](../packages/renderer)).

**Acceptance** — Skill produces a runnable composition file and can render it to
mp4 via `@konva-motion/renderer`. Cross-reference item **3** (API usage skill) so
they share one canonical API description.

---

## 3. Skill: how to use the motion API

**What** — A reference/usage skill documenting the konva-motion authoring API
(Composition, Sequence, Flex/Block, shapes, interpolate/easing, media, transitions).

**Why** — Companion to item **2**: the "how to create a video" skill needs an
accurate, current description of the API surface to call into.

**Where** — Source of truth is the public barrel
[packages/core/src/index.ts](../packages/core/src/index.ts) plus
[doc/README.md](README.md) and [doc/architecture.md](architecture.md). Keep it in
sync with the `exports` field (consumers never deep-import).

**Acceptance** — Skill covers the drawing vocabulary, the timeline/frame-clock
model, layout dispatch (`KMLayoutNode`), animation helpers, and media. Note in
CLAUDE.md's "update doc/README.md on API change" rule that this skill must track too.

---

## 4. Logo: legibility at small sizes

**What** — Refine the logo so it reads well at favicon / small-icon sizes.

**Why** — Current marks lose detail when scaled down.

**Where** — Existing assets in [assets/](../assets):
`konva-motion-icon-afterimages.svg`, `konva-motion-icon-circle.svg`,
`konva-motion-mark-{black,gradient,white}.svg`. Studio/docs consume them
(`packages/docs/public/`, [studio paths.tsx](../packages/studio/src/components/icon/paths.tsx)).

**Acceptance** — A simplified small-size variant (fewer strokes / higher
contrast), wired as the favicon in docs and studio. Verify at 16/32/48px.

---

## 5. Release

**What** — Publish the workspace packages.

**Why** — Get core / player / renderer / transitions usable outside the monorepo.

**Where** — Per-package `package.json` `exports` + `dist/`. Build = `tsc -b` per
package except `player` (Vite); see CLAUDE.md "Build" notes.

**Acceptance** — Version bump, changelog, `pnpm build` green for all packages,
verify each package's `exports`/`types` resolve from a clean consumer, publish.
Confirm `konva` peerDep ranges and the player standalone bundle
([player-remote-src.md](player-remote-src.md)).

---

## 6. Frame-rate mismatch: 30fps video in a 60fps composition

**What** — A source video whose frame rate differs from the composition fps does
not play correctly.

**Why** — Media time is mapped in composition frames; a 30fps clip in a 60fps
comp needs the source frame resampled (each source frame held for 2 comp frames),
which isn't handled.

**Where**
- [media-time.ts](../packages/core/src/media/media-time.ts) — frame↔time mapping.
- [video/index.ts](../packages/core/src/media/video/index.ts) (`fps: comp.fps`),
  [video/types.ts](../packages/core/src/media/video/types.ts).

**Acceptance** — A video source carries its own native fps; the time mapping
converts comp-frame → source-time using the source rate (and back), so 30fps and
60fps sources both play at correct speed in any-fps composition. Add the
equivalent check for audio. Cover both browser playback and server render.

---

## 7. Export: audio missing from rendered video

**What** — Exported/rendered video has no audio track.

**Why** — Audio clips are collected and meant to be muxed by ffmpeg, but the
final output is coming out silent.

**Where**
- [audio-track.ts](../packages/renderer/src/audio-track.ts) (`collectAudioTrack`).
- [render.ts](../packages/renderer/src/render.ts) (`const clips = opts.mute ? []
  : collectAudioTrack(comp, fps)`).
- [ffmpeg.ts](../packages/renderer/src/ffmpeg.ts) (mux: `delayMs`, `compDur`,
  per-clip filters).

**Acceptance** — Render a comp with at least one audio asset and confirm the
output mp4 has a correct audio track (right offset, duration, volume envelope).
Check whether `collectAudioTrack` returns clips, whether asset URLs resolve to fs
paths server-side (see memory: server-render asset URLs), and whether ffmpeg's
filtergraph wires the inputs. Related to item **6** for fps-correct audio timing.

---

## 8. SSR fonts: first-party font handling

**What** — A built-in, first-class way to load and use fonts during server-side
rendering (headless renderer + docs SSR).

**Why** — Text currently relies on whatever fonts the host environment has;
custom/web fonts don't reliably register for skia-canvas rendering or SSR,
producing wrong glyphs/metrics offline.

**Where**
- Renderer text path: [packages/renderer](../packages/renderer) (skia-canvas
  backend; `./register` = `setupServerRendering()`).
- Core text: [layout/text](../packages/core/src/layout/text) — note core `Text`
  extends `Konva.Group`, update via `setText()` (see memory: core Text setText).
- Docs SSR: [packages/docs](../packages/docs).

**Acceptance** — An API to register font files/families that works in both
browser and Node/skia render, so a composition declaring a font renders identical
glyphs and metrics in preview and export. Document font loading in the authoring
docs.

---

## 9. Performance: disable Konva event listeners

**What** — Provide a way to turn off all Konva event handling (hit graph + click/
tap listening) so playback and rendering don't pay for hit detection.

**Why** — konva-motion is timeline-driven, not interactive: the stage is drawn
each tick but nothing needs pointer hit-testing. Konva builds and maintains a hit
graph and pointer listeners by default, which is wasted CPU/memory per
`batchDraw`. Disabling it should speed up playback (especially many shapes) and
server render.

**Where**
- Internal sub-nodes already opt out ad hoc with `listening: false`
  ([block.ts](../packages/core/src/layout/block.ts),
  [image.ts](../packages/core/src/layout/image.ts),
  [text.ts](../packages/core/src/layout/text/text.ts)) — but there's no global
  switch and top-level wrappers/shapes still listen.
- [composition.ts](../packages/core/src/engine/composition.ts) (`Stage`) and
  [sequence.ts](../packages/core/src/engine/sequence.ts) (`Layer`) are the right
  place to flip listening off by default.

**Acceptance** — Default to no listeners for playback/render (e.g. stage
`listening(false)` and/or `Konva.listenClickTap = false`, skip hit-graph drawing
on layers). If the studio needs interaction (selection/editing), make it opt-in
so authoring still works. Measure `batchDraw` cost before/after on a
many-shape composition.
