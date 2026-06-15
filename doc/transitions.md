# `@konva-motion/transitions`

Remotion-style transitions for konva-motion. Chain scenes that **overlap** while
a transition plays — a geometric move (slide, wipe, iris…) or a WebGL shader
(dissolve, crosswarp, swap…). Built on the same offset engine as core's
[`Series`](./README.md#new-seriesopts).

```
TransitionSeries  — sequencer; scenes overlap by each transition's duration
Presentation      — how a transition looks (Tier A geometric · Tier B shader)
Timing            — how progress moves over the overlap (linear · spring)
```

Install: it's a workspace package. `@konva-motion/core` and `konva` are peer
deps.

```ts
import {
  TransitionSeries,
  slide,
  dissolve,
  linearTiming,
} from "@konva-motion/transitions";
```

## Quick example

```ts
import { Composition } from "@konva-motion/core";
import { TransitionSeries, slide, dissolve, linearTiming } from "@konva-motion/transitions";
import Konva from "konva";

const comp = new Composition({ id: "demo", fps: 30, durationInFrames: 180, width: 1280, height: 720 });

// 4 scenes × 60f − 3 transitions × 20f = 180 frames.
const series = new TransitionSeries({ composition: comp });

const scene = (fill: string) => (seq: import("@konva-motion/core").Sequence) =>
  seq.add(new Konva.Rect({ x: 0, y: 0, width: 1280, height: 720, fill }));

series.scene({ durationInFrames: 60 }, scene("#0d1117"));
series.transition({ presentation: slide({ direction: "from-right" }), timing: linearTiming({ durationInFrames: 20 }) });
series.scene({ durationInFrames: 60 }, scene("#161b22"));
series.transition({ presentation: dissolve(), timing: linearTiming({ durationInFrames: 20 }) });
series.scene({ durationInFrames: 60 }, scene("#1b1030"));

comp.add(series); // adds every scene + GL overlay layer the series yields
```

> The runnable version chaining four scenes (slide + two shader transitions)
> lives in the demo: **Transitions → Slide · dissolve · crosswarp**
> ([demo/src/demos/transitions.ts](../demo/src/demos/transitions.ts)). The same
> **Transitions** group also has one isolated A → B demo per presentation
> ([transition-gallery.ts](../demo/src/demos/transition-gallery.ts)), each
> exposing its props in the studio form — edits apply live (the presentation
> re-reads them every frame, no rebuild).

## `new TransitionSeries(opts)`

- `opts.composition: Composition` — **required**. Spring timings resolve their
  length from `fps`, and presentations are sized from the stage
  `width`/`height`, so the composition is passed up front. (Remotion reads these
  from React context; konva-motion is imperative.)
- `opts.from?: number` — frame the series starts at (default `0`).

Methods:

- `series.scene({ durationInFrames }, (seq) => void): this` — append a scene.
- `series.transition({ presentation, timing }): this` — overlap the previous and
  next scene by `timing.getDurationInFrames(fps)` frames.
- `series.sequences(): Sequence[]` — build the scene layers (overlaps resolved)
  plus a GL overlay layer for each shader transition. Usually you don't call
  this — `comp.add(series)` does it for you (a `TransitionSeries` is a
  `SequenceProvider`).
- `series.durationInFrames` — `Σ scene durations − Σ transition durations`.

The incoming scene's `from` starts `transition.duration` frames before the
outgoing scene ends — the overlap, implemented as a negative offset through the
shared `computeOffsets` engine.

### Validation (throws descriptive errors)

- A transition must not be first or last.
- Two transitions must not be adjacent.
- A transition must not be longer than either neighbouring scene.

## Timings

A `Timing` maps a transition's local frame onto `[0, 1]` and reports its length.

### `linearTiming({ durationInFrames, easing? })`

Linear (optionally eased) progress over a fixed frame count. `easing` accepts
core's [`Easing`](./README.md#easing).

```ts
import { Easing } from "@konva-motion/core";
linearTiming({ durationInFrames: 15, easing: Easing.inOut(Easing.cubic) });
```

### `springTiming({ config?, durationInFrames?, durationRestThreshold?, reverse? })`

Physics-spring progress. With no `durationInFrames`, the spring's natural settle
time is used (measured from `fps`). `config` is `{ damping, mass, stiffness,
overshootClamping }` (defaults `{ 10, 1, 100, false }`).

```ts
springTiming({ config: { damping: 200 }, durationInFrames: 30 });
```

`spring(...)`, `measureSpring(...)` and `defaultSpringConfig` are also exported.

## Presentations

A `Presentation` defines how a transition looks. Two kinds:

```ts
type Presentation = {
  enter?(layer, progress, dims): void;   // Tier A — mutate the incoming layer
  exit?(layer, progress, dims): void;     // Tier A — mutate the outgoing layer
  gl?: { fragment, uniforms? };           // Tier B — a fragment shader
};
```

### Tier A — geometric (Konva-native, no WebGL)

| Factory | Effect |
| --- | --- |
| `fade()` | Opacity cross-dissolve. At `progress 0.5` both layers are at 0.5 opacity. |
| `slide({ direction })` | Translate the incoming layer in, push the outgoing out. `direction`: `from-left \| from-right \| from-top \| from-bottom`. |
| `wipe({ direction })` | Rectangular `clipFunc` reveal of the incoming layer. |
| `clockWipe({ width?, height? })` | Angular pie-sweep clip. Defaults to stage size. |
| `iris({ width?, height? })` | Expanding circular clip. Defaults to stage size. |
| `flip({ direction })` | `scaleX`/`scaleY` card-flip. **Fake** — Konva has no real 3D; it squashes rather than rotating in perspective. |
| `none()` | Hard cut (no transform). |

### Tier B — GLSL shaders (shared WebGL compositor)

Each frame of the overlap, the two scene layers are captured with
`layer.toCanvas()`, uploaded as textures, and blended by a fragment shader
(ported verbatim from [gl-transitions](https://gl-transitions.com/), license
headers preserved). The output is drawn into a `Konva.Image` on an overlay layer
above the two scenes.

| Factory | Params |
| --- | --- |
| `dissolve({ lineWidth?, spreadColor?, hotColor?, pow?, intensity? })` | Burning dissolve with a glowing edge. |
| `crosswarp()` | Warping cross-dissolve. |
| `crossZoom({ strength? })` | Zoom-blur cross-dissolve. |
| `dreamyZoom({ rotation?, scale? })` | Spiralling zoom. |
| `filmBurn({ seed? })` | Fiery film-burn. |
| `linearBlur({ intensity? })` | Directionless blur dissolve. |
| `ripple({ amplitude?, speed? })` | Concentric ripple. |
| `zoomBlur({ rotation? })` | Counter-rotating radial zoom blur. |
| `zoomInOut()` | Punch-in / punch-out crossfade. |
| `swap({ reflection?, perspective?, depth? })` | Perspective card swap with floor reflection. |
| `bookFlip({ direction? })` | Page-turn / book flip. |

**WebGL requirement.** Tier B needs a WebGL context. In the browser the shared
compositor uses WebGL2 directly. When none is available, every shader transition
falls back to `fade()` and a single `console.warn` is emitted. Shaders are
deterministic in `progress`, so a frame renders identically given the same
inputs.

**Server (Node) rendering.** skia-canvas has no WebGL, so by default offline
renders use the fade fallback. To render shader transitions headlessly, add the
optional [`gl`](https://github.com/stackgl/headless-gl) (headless-gl) package and
import the GL wiring once, **before** building any `TransitionSeries`:

```ts
import "@konva-motion/renderer/register"; // skia backend + factories
import "@konva-motion/renderer/gl";        // route Tier B through headless-gl
```

This installs a headless-gl + skia compositor (`setCompositorFactory`): the
`#version 300 es` fragments are transpiled to GLSL ES 1.00 for WebGL1, scenes are
uploaded as raw RGBA and the blended frame is read back into a skia canvas. If
`gl` isn't installed, transitions still fall back to fade and a one-time hint is
logged.

### Custom shaders

`glTransition(fragment, uniforms?)` wraps any fragment shader into a
`Presentation`. The compositor supplies `u_time` (progress), `u_prev` (the
**incoming** scene) and `u_next` (the **outgoing** scene); `uniforms(progress,
dims)` returns everything else (a `number` → `uniform1f`, a `number[]` of length
2/3/4 → `uniform2f`/`3f`/`4f`).

## How it fits together

`Sequence` (range-gated layer) → `Series` (auto-offsets, in core) →
`TransitionSeries` (offsets with overlap). The same `computeOffsets` engine
powers `Series` and `TransitionSeries`; a transition is just a sequential step
with a negative (overlap) delta.
