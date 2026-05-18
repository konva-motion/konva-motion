# konva-motion

Remotion-style timeline-driven animation for [Konva](https://konvajs.org).

A **Composition** is a Konva.Stage that owns a frame clock (fps + duration).
A **Sequence** is a Konva.Layer scoped to a frame range — its updaters run
and its layer paints only while the playhead is in range. Composition drives
one `batchDraw()` per active sequence per frame.

The engine is agnostic: `play()` uses `requestAnimationFrame` in the browser;
on the server, step frames manually with `setFrame(n)`.

## Install

```sh
pnpm add @konva-motion/core konva
```

`konva` is a peer dependency — you pin the version.

## Quick example

```ts
import { Composition, Sequence } from "@konva-motion/core";
import Konva from "konva";

const comp = new Composition({
  id: "main",
  fps: 30,
  durationInFrames: 300,
  container: "root",
  width: 800,
  height: 600,
});

// Sequence covering the entire composition — like a "root" layer.
const main = new Sequence({ from: 0, durationInFrames: 300 });
const circle = new Konva.Circle({ x: 100, y: 300, radius: 40, fill: "tomato" });
main.add(circle);
comp.add(main);

main.register((frame) => {
  circle.x(100 + Math.sin((frame / 300) * Math.PI * 2) * 200);
});

// Sequence with its own layer — only painted while in range.
const flash = new Sequence({ from: 90, durationInFrames: 60 });
const square = new Konva.Rect({ x: 350, y: 250, width: 100, height: 100, fill: "#ff6b6b" });
flash.add(square);
comp.add(flash);
flash.register((localFrame) => {
  square.opacity(1 - localFrame / 60);
});

comp.play();
```

## API

### `new Composition(opts)`

`opts` extends `Konva.StageConfig` with:

- `id: string` — required, used for logging.
- `fps: number` — frames per second.
- `durationInFrames: number` — total length.
- `loop?: boolean` — when `true`, playback wraps to frame 0 instead of
  auto-pausing at the end. Default `false`.

The instance **is** a `Konva.Stage`. Adds itself to the stage via a
`__KonvaMotionComposition` marker; constructing a second composition on the
same underlying stage throws.

**Readonly signals** (`{ get(): T; subscribe(fn): () => void }`):

- `frame`, `isPlaying`, `durationInFrames`, `loop`
- `isStopped` — derived: `!isPlaying && frame === 0`
- `isPaused` — derived: `!isPlaying && frame > 0`

**Methods:**

- `play()` — starts the rAF loop. Throws in environments without
  `requestAnimationFrame`. Emits `"play"`.
- `pause()` — stops the loop, leaves `frame` as-is.
- `stop()` — stops the loop, resets `frame` to 0, emits `"stop"`.
- `setFrame(n)` — clamped to `[0, durationInFrames - 1]`; applies updaters
  and emits `"time"`. Works on server (no rAF needed).
- `setLoop(v)` — toggle looping at runtime; updates the `loop` signal.
- `add(sequence)` — inherited `Konva.Stage.add`; the composition will tick
  any child that is a `Sequence`. Plain `Konva.Layer` children render
  normally (Konva auto-draws), they're just not gated by frame range.
- `destroy()` — overrides `Konva.Stage.destroy` to cancel the rAF loop
  first, then tear down the stage.

**Events** (payload `{ frame, durationInFrames }`):

- `comp.on("play"|"stop"|"time", handler)` — returns an unsubscribe.
- `comp.off(event, handler)`

For Konva's own events (`"click"`, `"mousedown"`, etc.), `on`/`off` delegate
to `Konva.Node` as usual.

### `new Sequence(opts)`

`opts` extends `Konva.LayerConfig` with:

- `from: number` — start frame (inclusive).
- `durationInFrames: number` — length.

The instance **is** a `Konva.Layer`. Starts hidden; the composition toggles
`visible(true)` while `currentFrame` is in `[from, from + durationInFrames)`
and `visible(false)` otherwise. Only active sequence layers get `batchDraw`.

- `seq.register((localFrame) => void): () => void` — runs every frame in
  range. `localFrame = currentFrame - from`. Returns an unsubscribe.

### `getComposition(stage)`

Returns the `Composition` attached to a `Konva.Stage`, or `null`.

### `interpolate(input, inputRange, outputRange, options?)`

Maps `input` from `inputRange` onto `outputRange`. API-compatible with
[Remotion's `interpolate`](https://www.remotion.dev/docs/interpolate).

- `options.easing?: (n: number) => number` — applied to the normalized
  segment progress. Default identity.
- `options.extrapolateLeft?`, `options.extrapolateRight?` — `"extend"`
  (default), `"identity"`, `"clamp"`, or `"wrap"`.

```ts
import { interpolate, Easing } from "@konva-motion/core";

main.register((frame) => {
  const x = interpolate(frame, [0, 300], [100, 700], {
    easing: Easing.inOut(Easing.cubic),
    extrapolateRight: "clamp",
  });
  circle.x(x);
});
```

### `interpolateColors(input, inputRange, outputRange)`

Same idea, but for colors. Returns an `rgba(r, g, b, a)` string. Accepts
hex (`#rgb`, `#rgba`, `#rrggbb`, `#rrggbbaa`), `rgb()`/`rgba()`,
`hsl()`/`hsla()`, and CSS named colors in `outputRange`. Extrapolation is
clamped on both sides. API-compatible with
[Remotion's `interpolateColors`](https://www.remotion.dev/docs/interpolate-colors).

```ts
import { interpolateColors } from "@konva-motion/core";

main.register((frame) => {
  circle.fill(interpolateColors(frame, [0, 150, 300], ["red", "yellow", "blue"]));
});
```

### `Easing`

Easing curves. API mirrors
[Remotion's `Easing`](https://www.remotion.dev/docs/easing) (which adapts
React Native's `Easing`). Cubic-bezier support is provided by the
[`bezier-easing`](https://www.npmjs.com/package/bezier-easing) package.

- Curves: `linear`, `ease`, `quad`, `cubic`, `poly(n)`, `sin`, `circle`,
  `exp`, `elastic(bounciness?)`, `back(s?)`, `bounce`,
  `bezier(x1, y1, x2, y2)`, `step0`, `step1`
- Modifiers: `in(easing)`, `out(easing)`, `inOut(easing)`

## How-to recipes

Short snippets for things you'll do most often. Each assumes you've already
constructed a `Composition` named `comp` and a root `Sequence` named `main`
(see [Quick example](#quick-example)).

### Animate a property with easing

Drive any Konva setter from `interpolate` inside `main.register`:

```ts
import { interpolate, Easing } from "@konva-motion/core";

main.register((frame) => {
  node.opacity(
    interpolate(frame, [0, 30], [0, 1], {
      easing: Easing.out(Easing.cubic),
      extrapolateRight: "clamp",
    }),
  );
});
```

### Multi-stop keyframes

`interpolate` accepts arbitrary-length input/output ranges. Animate multiple
properties off the same playhead:

```ts
const stops = [0, 30, 60, 90];
main.register((frame) => {
  star.x(interpolate(frame, stops, [100, 400, 400, 700]));
  star.y(interpolate(frame, stops, [200, 100, 300, 200]));
  star.rotation(interpolate(frame, stops, [0, 90, 180, 360], {
    easing: Easing.inOut(Easing.quad),
  }));
});
```

### Crossfade between two images

Stack `Konva.Image` nodes and animate their `opacity`:

```ts
const a = new Konva.Image({ image: undefined, x: 0, y: 0, width, height });
const b = new Konva.Image({ image: undefined, x: 0, y: 0, width, height, opacity: 0 });
main.add(a); main.add(b);

const imgA = new window.Image(); imgA.src = "/a.jpg"; imgA.onload = () => a.image(imgA);
const imgB = new window.Image(); imgB.src = "/b.jpg"; imgB.onload = () => b.image(imgB);

main.register((frame) => {
  const t = interpolate(frame, [30, 60], [0, 1], {
    easing: Easing.inOut(Easing.sin),
    extrapolateLeft: "clamp",
    extrapolateRight: "clamp",
  });
  a.opacity(1 - t);
  b.opacity(t);
});
```

### Slide an image in from the right and out to the left

```ts
main.register((frame) => {
  image.x(
    interpolate(frame, [0, 15, 45, 60], [width, 0, 0, -width], {
      easing: Easing.inOut(Easing.cubic),
    }),
  );
});
```

The 4-stop range gives you enter → hold → exit in one call.

### Reveal a node with an animated clip

`Konva.Group` accepts a `clipFunc(ctx)` that draws the clip path. Mutate a
closure-captured state object each frame and the next `batchDraw` picks it
up:

```ts
const clip = { radius: 0 };
const group = new Konva.Group({
  clipFunc: (ctx) => {
    ctx.beginPath();
    ctx.arc(width / 2, height / 2, clip.radius, 0, Math.PI * 2);
    ctx.closePath();
  },
});
group.add(image);
main.add(group);

main.register((frame) => {
  clip.radius = interpolate(frame, [0, 60], [0, Math.hypot(width, height) / 2], {
    easing: Easing.inOut(Easing.cubic),
    extrapolateRight: "clamp",
  });
});
```

### Typewriter / progressive text reveal

Slice the target string against frames:

```ts
const CHARS_PER_FRAME = 2;
const message = "Hello from konva-motion.";
const text = new Konva.Text({ x: 24, y: 24, text: "", fontSize: 20, fill: "#fff" });
const caret = new Konva.Rect({ x: 24, y: 24, width: 2, height: 24, fill: "#fff" });
main.add(text); main.add(caret);

main.register((frame) => {
  const n = Math.min(message.length, Math.floor(frame * CHARS_PER_FRAME));
  text.text(message.slice(0, n));
  caret.x(24 + text.getTextWidth());
  caret.opacity(Math.floor(frame / 15) % 2 === 0 ? 1 : 0);
});
```

### Loop a composition forever

```ts
const comp = new Composition({ /* ... */ loop: true });
// or toggle at runtime
comp.setLoop(true);
```

### Drive the playhead from UI (scrubber)

```ts
input.addEventListener("input", () => {
  comp.pause();
  comp.setFrame(Number(input.value));
});
const unsub = comp.frame.subscribe((f) => {
  input.value = String(f);
});
```

`setFrame` runs updaters synchronously, so the canvas reflects the new frame
before the event handler returns.

### Sequences that gate visibility by range

Put work that should only appear in a window inside its own `Sequence`:

```ts
const flash = new Sequence({ from: 90, durationInFrames: 30 });
flash.add(new Konva.Rect({ /* ... */ }));
comp.add(flash);
flash.register((local) => { /* local: 0..29 */ });
```

The layer is `visible(false)` and not drawn outside its range — no manual
opacity gating needed.

## Demo app

The Vite demo (`pnpm dev`, [demo/](../demo)) ships with a sidebar of demos
and a draggable timeline scrubber:

- **Basic** — circle move + fading square sequence.
- **Bouncing ball (loop)** — `loop: true` with ease-out fall and squash on impact.
- **Staggered fade-in** — cards entering on overlapping sequences.
- **Rotate + scale** — concentric polygons plus a mid-loop burst sequence.
- **Easings — race of curves** — side-by-side comparison of every `Easing` curve.
- **Colors — fill + stroke interpolation** — `interpolateColors` driving background, rings, and a breathing blob.
- **Keyframes — multi-stop interpolate** — a star following a 6-point path with per-property easing.
- **Image — slider** — 4-stop slide-in/hold/slide-out transitions over `Konva.Image`.
- **Image — crossfade** — opacity-eased crossfade with a Ken-Burns zoom.
- **Image — circular clip reveal** — animated `clipFunc` growing a circle to reveal each image.
- **Typewriter — AI chat** — character-by-character text reveal with a blinking caret.
- **Flex layout — auto card** — `Flex` + `Block` + `Image` auto-flow: text rewraps under the image as the card animates width.
- **Flex — typewriter pushes image** — text typing inside a `Block` grows the column; the image below slides down each frame as the text height changes.
- **Flex — row of growing images** — four `Image`s in a row take turns animating `flexGrow` from 1 → 4 → 1, demonstrating that any flex attr can be mutated per frame.
- **Flex — feature showcase** — four labeled sections cycling `justifyContent`, `alignItems`, per-child `flexGrow` weights, and `gap`, all driven by `setAttr` inside `register`.

## Flex layout

`@konva-motion/core` ships `Flex`, `Block`, and `Image` — three Konva.Group
subclasses wired to a synchronous flexbox engine
([flexily](https://github.com/beorn/flexily)). When a `Flex` sits inside a
`Sequence`, layout is recomputed on every frame, so animated `width`, `gap`,
or text changes reflow without any extra wiring.

```ts
import { Flex, Block, Image } from "@konva-motion/core";
import Konva from "konva";

const card = new Flex({
  x: 40, y: 40,
  width: 360,
  flexDirection: "column",
  gap: 12,
});

const body = new Block({
  width: "100%",
  flexDirection: "column",
  padding: 16,
  gap: 10,
  background: { gradient: { type: "linear", stops: [[0, "#1f2937"], [1, "#111827"]], angle: 135 } },
  borderSize: 1,
  borderColor: "#374151",
  cornerRadius: 14,
  shadow: { color: "#000", blur: 24, offsetY: 8, opacity: 0.45 },
});

body.add(new Image({ width: "100%", height: 160, src: "/cover.jpg", objectFit: "cover", cornerRadius: 10 }));
body.add(new Konva.Text({ text: "Headline", fontSize: 22, fontStyle: "bold", fill: "#f9fafb" }));
body.add(new Konva.Text({ text: "Body copy that rewraps as the card resizes.", fontSize: 14, fill: "#d1d5db" }));

card.add(body);
main.add(card);
```

- **`Flex(config)`** — `flexDirection`, `justifyContent`, `alignItems`,
  `gap`, `padding`. Sizes accept `number` or CSS-style `"50%"`.
- **`Block(config)`** — everything `Flex` accepts, plus visual props:
  `background` (color string or `{ gradient }`), `borderSize`, `borderColor`,
  `borderStyle: "solid" | "dashed"`, `shadow`, `cornerRadius`.
- **`Image(config)`** — fixed-box image with `objectFit`
  (`"cover" | "contain" | "fill" | "none"`), `objectPosition`, and
  `cornerRadius`. `src` accepts an `HTMLImageElement` or a URL string.
- **Child props** on any flex child: `flexGrow`, `flexShrink`, `flexBasis`,
  `alignSelf`, `margin`.
- **Auto-measure** — raw `Konva.Text` and `Konva.Image` children of `Flex`
  contribute their intrinsic size to layout. Text wraps to whatever width
  the flex algorithm assigns it, so it never overlaps siblings.

### Animating layout

Inside `Sequence.register`, just mutate any flex attr — `width`, `gap`,
`flexGrow`, `justifyContent`, `alignItems`, the text content itself, etc.
The next tick recomputes layout, so positions and sizes reflow without you
calling anything explicitly:

```ts
main.register((frame) => {
  // animate the card's width — text rewraps, image below slides as needed
  card.width(320 + 240 * (0.5 - 0.5 * Math.cos(frame / 90 * Math.PI * 2)));

  // change a child's flex weight — siblings redistribute on the next draw
  images[active]?.setAttr("flexGrow", 4);

  // edit text — the column grows and pushes later children down
  title.text(`Frame ${frame}`);
});
```

Layout cadence: every active `Sequence` walks its direct children once per
frame and calls `computeLayout()` on any that are `Flex`. A `Flex` outside
a `Sequence` recomputes only when you call `computeLayout()` manually.

The scrubber ([demo/src/scrubber.ts](../demo/src/scrubber.ts)) is a small
example of wiring the engine to a UI: it subscribes to `comp.frame`, calls
`comp.setFrame()` on `<input type=range>` input (pausing while held), and
toggles `comp.setLoop()` from a checkbox.

## Server / offline rendering

`play()` is browser-only, but `setFrame(n)` works anywhere. Step manually:

```ts
for (let f = 0; f < comp.durationInFrames.get(); f++) {
  comp.setFrame(f);
  // ...export comp.toDataURL() or pipe canvas elsewhere
}
```

See [architecture.md](./architecture.md) and
[contributing.md](./contributing.md).
