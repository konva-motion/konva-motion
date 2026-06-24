# Font component + Composition buffer state — design plan

Status: **proposed** (not yet implemented). Two coupled features:

1. A new **`Font`** node (`@konva-motion/core`) that declares a font family +
   faces, loads them in an environment-aware way, and is consumed by `Text`.
2. A **buffer state** on `Composition` so a player can wait for assets (fonts,
   and later images/video) to be ready before playback starts.

---

## 1. Target API

```ts
import regularFontUrl from "./MyFont-Regular.woff2?url";
import italicFontUrl from "./MyFont-Italic.woff2?url";

const comp = new Composition({ id, fps: 30, durationInFrames: 90, ... });
const seq  = new Sequence({ from: 0, durationInFrames: 90 });

const font = new Font({
  family: "My Font",
  faces: [
    { weight: 400, style: "normal", src: regularFontUrl },
    { weight: 400, style: "italic", src: italicFontUrl },
    { weight: 700, style: "normal", src: boldFontUrl },
  ],
});

// Added to the sequence tree; registered at the Composition, which loads it
// (and buffers on it) before play.
seq.add(font);

// Face selection when passed to Text:
//   font            → preferred face (400 / normal, else first face)
//   font.face('400')        → first 400-normal, else first 400-anything
//   font.face('400-italic') → exactly 400-italic, else THROW
const title = new Text({ font, text: "Hello" });
const quote = new Text({ font: font.face("400-italic"), text: "world" });

comp.add(seq);
```

Playback gating:

```ts
comp.buffer.get();         // "idle" | "buffering" | "ready"
await comp.whenReady();    // resolves once all registered assets settle
comp.play();               // buffer-aware: if assets pending, defers the clock
```

---

## 2. Where this fits in the existing architecture

The repo already has every pattern this feature needs; we are composing them,
not inventing mechanisms.

| Existing pattern | File | Reused for |
| --- | --- | --- |
| Invisible `Konva.Group` node living in the `Sequence` tree | `media/audio/index.ts:23,42` | `Font` extends `Konva.Group`, `visible:false`, so `seq.add(font)` works |
| Marker-attr discovery (no `instanceof` in the engine) | `media/media-marker.ts`; `composition.ts:197`; `sequence.ts:63` | a new `FONT_MARK` so `Composition.add` finds fonts and registers them |
| Globally-overridable loader DI | `engine/runtime-defaults.ts:22,52` (`ImageLoader`) | a new `FontLoader`; renderer swaps in a skia implementation |
| Eager load kicked in the constructor, promise captured | `layout/image.ts:103`; `media/audio/index.ts:65` | `Font.load()` returns/caches the load promise |
| `delayRender`/`continueRender`/`renderFrame` gate (offline) | `composition.ts:306-335`; `layout/image.ts:130` | font load gates `renderFrame` during server rendering |
| Re-layout + `batchDraw` after async asset resolves | `layout/image.ts:108-113` | `Text` re-lays-out when its `Font` finishes loading |
| Server font registration via skia `FontLibrary.use` | `renderer/src/setup.ts:18-22` | server `FontLoader` reuses this |
| Reactive `Signal` + derived state on `Composition` | `composition.ts:69-73,150-157` | `buffer` state signal |

---

## 3. Composition buffer state

The render path already has a per-frame "wait for async work" gate
(`delayRender`/`renderFrame`). The buffer state is the **preview** analogue:
a session-level "are all declared assets ready?" signal a player can observe to
show a spinner, and that `play()` honors so the clock doesn't advance over
unloaded assets.

### 3.1 New surface on `Composition`

```ts
export type BufferState = "idle" | "buffering" | "ready";

class Composition {
  readonly buffer: ReadonlySignal<BufferState>;     // observable for players
  readonly isBuffering: ReadonlySignal<boolean>;    // derived: buffer === "buffering"

  /** Register an outstanding asset load. Buffer → "buffering" until it (and all
      others) settle, then → "ready". Errors are swallowed (logged) so one bad
      asset can't wedge playback forever. */
  registerAsset(load: Promise<unknown>, label?: string): void;

  /** Resolves once no assets are pending (immediately if already ready/idle). */
  whenReady(): Promise<void>;
}
```

Internal state mirrors the existing `_renderHandles` gate:

```ts
private readonly _assets = new Set<Promise<unknown>>();
private readonly _buffer: Signal<BufferState> = createSignal("idle");
private _readyWaiters: Array<() => void> = [];
```

- `registerAsset(p)`: add to `_assets`, set `_buffer` → `"buffering"`. On
  `p.finally`, remove; when `_assets.size === 0` set `"ready"` and drain
  `_readyWaiters`. (If `play()` was deferred, this is also where it resumes —
  see 3.2.)
- `whenReady()`: resolve now if `_assets.size === 0`, else push to
  `_readyWaiters`.

### 3.2 `play()` becomes buffer-aware

Keep `isPlaying` meaning **"the clock is advancing."** Add a deferred-start
intent so `play()` while buffering is a queue, not an error (matches a video
player: user hit play, UI shows a spinner, playback begins when ready).

```ts
private _resumeOnReady = false;

play(): void {
  if (!raf || !caf) throw new Error(... /* unchanged */);
  if (this._isPlaying.get()) return;
  if (this._buffer.get() === "buffering") {
    this._resumeOnReady = true;       // start automatically once ready
    return;                            // do NOT spin RAF yet
  }
  this._startClock();                  // existing play() body, extracted
}
```

In the asset-drain path, when buffer flips to `"ready"`:

```ts
if (this._resumeOnReady) { this._resumeOnReady = false; this._startClock(); }
```

`pause()`/`stop()` also clear `_resumeOnReady`. `setFrame()` is unchanged and
works while buffering (so scrubbing a paused comp is fine — the visible frame
may just re-draw when the font lands, via the Text re-layout hook in §4.4).

> Scope note: `registerAsset` is generic. This plan wires **`Font`** into it.
> `Image`/`Video`/`Audio` already hold load promises (`image.ts:70`,
> `audio/index.ts:65`) and can opt in with a one-line `registerAsset` call as a
> fast follow — recommended but kept out of the core change to keep the diff
> reviewable.

---

## 4. The `Font` node

### 4.1 Types

```ts
export type FontStyleName = "normal" | "italic" | "oblique";

export type FontFace = {
  /** 400, "400", "bold", or a variable range like "100 900". Default 400. */
  weight?: number | string;
  /** Default "normal". */
  style?: FontStyleName;
  /** Font file URL (e.g. a Vite `?url` import) or, server-side, a path. */
  src: string;
};

export type FontConfig = {
  family: string;
  faces: FontFace[];
};

/** Resolved, validated face (weight/style normalized). */
type ResolvedFace = { weight: string; style: FontStyleName; src: string };

/** What `Text` consumes — a concrete family+weight+style plus readiness. */
export type FontFaceRef = {
  family: string;
  weight: string;
  style: FontStyleName;
  /** Resolves when the underlying font is loaded (for re-layout). */
  whenReady(): Promise<void>;
  readonly isLoaded: boolean;
};
```

### 4.2 Class

```ts
export class Font extends Konva.Group {
  readonly family: string;
  readonly faces: ResolvedFace[];
  private _loadPromise: Promise<void> | null = null;
  private _loaded = false;
  private _registered = false;   // guards one-time comp registration

  constructor(config: FontConfig) {
    super({ listening: false, visible: false });
    this.setAttr(FONT_MARK, true);   // discovered by Composition.add
    this.family = config.family;
    this.faces = config.faces.map(normalizeFace);  // validates non-empty
  }

  /** Env-aware, idempotent. Loads every face via the runtime FontLoader. */
  load(): Promise<void> {
    if (this._loadPromise) return this._loadPromise;
    const loader = getDefaultFontLoader();
    this._loadPromise = Promise.all(
      this.faces.map((f) => loader(this.family, f)),
    ).then(() => { this._loaded = true; }).catch((err) => {
      console.error("[konva-motion] Font load failed:", err);
    });
    return this._loadPromise;
  }

  get isLoaded(): boolean { return this._loaded; }
  whenReady(): Promise<void> { return this.load(); }

  /** Select a face. See selection rules in §4.3. */
  face(selector?: string): FontFaceRef { ... }

  /** @internal — Composition.add walks FONT_MARK and calls this once: kick the
      load and register it with the buffer (preview) or render gate (offline). */
  _kmRegister(comp: Composition): void {
    if (this._registered) return;
    this._registered = true;
    const p = this.load();
    if (comp.environment.isRendering) {
      const h = comp.delayRender(`load font ${this.family}`);
      p.finally(() => comp.continueRender(h));
    } else {
      comp.registerAsset(p, `font ${this.family}`);
    }
  }
}
```

### 4.3 Face selection rules

Selector grammar: `"<weight>"` or `"<weight>-<style>"`.

| Call | Behavior |
| --- | --- |
| `font` (passed bare to `Text`) / `font.face()` | prefer weight `400` + `normal`; else first declared face |
| `font.face("400")` | first `400`+`normal`; else first `400` of any style |
| `font.face("400-italic")` | exactly `400`+`italic`; **throws** if absent |

Rule: **when a style is named explicitly, it must exist (throw).** When only a
weight is given, prefer `normal`, else any matching weight. Throw with a clear
message listing available faces (`My Font: no 400-italic face; have 400-normal, 700-normal`).

### 4.4 `FontLoader` (runtime-defaults DI)

Add alongside `ImageLoader` in `engine/runtime-defaults.ts`. The loader is keyed
by `family|weight|style|src` and **deduped at the core level** so repeated
registration (multiple `Text`s, multiple `Font` instances sharing a family,
re-renders) loads each face once:

```ts
export type FontLoader = (family: string, face: ResolvedFace) => Promise<void>;

let fontLoader: FontLoader = domLoadFont;
const fontCache = new Map<string, Promise<void>>();          // family|weight|style|src
export const setDefaultFontLoader = (l: FontLoader) => { fontLoader = l; };
export const loadFontFace = (family: string, face: ResolvedFace): Promise<void> => {
  const key = `${family}|${face.weight}|${face.style}|${face.src}`;
  let p = fontCache.get(key);
  if (!p) { p = fontLoader(family, face); fontCache.set(key, p); }
  return p;
};
```

`Font.load()` calls `loadFontFace` per face (not the raw `fontLoader`), so the
cache is honored everywhere.

**Client default — `FontFace` API (reliable for canvas).** Canvas text does
*not* trigger CSS `@font-face` lazy-loading, so the JS `FontFace` API is the
correct path — its `load()` promise resolves only once glyphs are parsed:

```ts
function domLoadFont(family: string, face: ResolvedFace): Promise<void> {
  // face.src may be a Vite ?url, a same-origin path, or a remote URL — the
  // browser HTTP cache handles remote caching automatically.
  const ff = new FontFace(family, `url(${face.src})`, {
    weight: face.weight, style: face.style,
  });
  return ff.load().then((loaded) => { (document as any).fonts.add(loaded); });
}
```

> Client reliability caveats (mechanism is sound; these are the real failure
> modes): **CORS** — a remote font fetch is anonymous-CORS, so the host must
> send `Access-Control-Allow-Origin` or `load()` rejects; **duplicates** — the
> `fontCache` above prevents adding the same descriptor twice to `document.fonts`.

**Server default — skia `FontLibrary` with a local disk cache.** `FontLibrary.use`
accepts **only local file paths** (no URL, no Buffer — confirmed against
skia-canvas 3.0.8), so remote `src` must be downloaded and cached on disk. Lives
in `renderer/src/font-loader.ts`, wired in `setup.ts`:

```ts
// renderer/src/font-loader.ts
const registered = new Set<string>();        // family|weight|style|path — process-level

async function resolveToLocalPath(src: string, cacheDir: string): Promise<string> {
  if (!/^https?:\/\//.test(src)) return src;                 // already a path
  const file = path.join(cacheDir, `${sha256(src)}${path.extname(src) || ".font"}`);
  if (!existsSync(file)) {                                     // cache miss → download once
    const buf = Buffer.from(await (await fetch(src)).arrayBuffer());
    await fs.mkdir(cacheDir, { recursive: true });
    await fs.writeFile(file, buf);                            // persists across processes
  }
  return file;
}

export function makeSkiaFontLoader(cacheDir: string): FontLoader {
  return async (family, face) => {
    const localPath = await resolveToLocalPath(face.src, cacheDir);
    const key = `${family}|${face.weight}|${face.style}|${localPath}`;
    if (registered.has(key)) return;                          // dedup the use() call
    registered.add(key);
    FontLibrary.use(family, [localPath]);
  };
}
```

`setupServerRendering({ fontCacheDir })` (default `os.tmpdir()/konva-motion-fonts`)
calls `setDefaultFontLoader(makeSkiaFontLoader(cacheDir))`. The existing
`registerFonts(opts.fonts)` path stays for setup-time fonts.

> **`family` is a process-global key in skia** (verified): repeated
> `use('Fam', [...])` *accumulates* faces, and each weight/style slot is
> first-wins — a second file for an already-filled slot is silently dropped. So
> two `Font`s sharing a `family` with **conflicting** faces (same weight/style,
> different file) is a latent bug. `Font` should `console.warn` when it detects a
> duplicate weight/style within its own `faces`, and we document that distinct
> fonts need distinct `family` names. The `?url`→path mapping precedent is the
> existing `mediaSrc` helper (memory: *Server-render asset URLs*).

---

## 5. `Text` integration

`TextConfig` gains one field:

```ts
font?: Font | FontFaceRef;
```

When present it **overrides** `fontFamily`/`fontStyle` (and the weight encoded
in `fontStyle`). Resolution in the `Text` constructor:

```ts
const ref = config.font instanceof Font ? config.font.face() : config.font;
const fontFamily = ref ? ref.family : config.fontFamily;
// Konva builds "<fontStyle> <fontSize>px <fontFamily>"; numeric weight + style
// both live in fontStyle, e.g. "400 italic" / "700 normal" / "italic".
const fontStyle = ref ? konvaFontStyle(ref.weight, ref.style) : config.fontStyle;
```

`konvaFontStyle("400","normal")` → `"normal"`; `("700","normal")` → `"700"` (or
`"bold"`); `("400","italic")` → `"italic"`; `("700","italic")` → `"700 italic"`.

**Re-layout on load** (the font may not be ready when `Text` is constructed —
Konva would measure with a fallback face and lay out wrong):

```ts
if (ref && !ref.isLoaded) {
  ref.whenReady().then(() => {
    this._layoutText();
    this.getLayer()?.batchDraw();
  });
}
```

This is the same "resolve → re-layout → batchDraw" shape as `Image`
(`image.ts:108-113`). Combined with the buffer gate, a player that waits for
`whenReady()` (or uses the buffer-aware `play()`) starts playback only after the
font is measured correctly, so there is no glyph reflow flash mid-playback.
A bare scrubbing/paused comp still self-corrects via this hook.

---

## 6. Discovery & registration wiring

Mirror media discovery exactly.

- **`media-marker.ts`**: add `export const FONT_MARK = "__kmIsFont";`
- **`Composition.add`** (`composition.ts:193-202`): in the same per-layer loop
  that registers media, also walk `find(FONT_MARK)` and call
  `font._kmRegister(this)`. (Eager — so buffering starts before `play`.)
- **Lazy fallback**: if a `Font` is added to a sequence *after* `comp.add(seq)`,
  it isn't in that walk. Give `Font` a `TICK_MARK` too and register on first
  `_kmTick` as a safety net (same lazy-fallback philosophy as audio's
  `_ensureDriver`, `audio/index.ts:88`). Eager path is the documented norm;
  recommend `seq.add(font)` before `comp.add(seq)`.

Because `_kmRegister` is idempotent (`_registered` guard), eager + lazy paths
can't double-register.

---

## 7. Public exports (`core/src/index.ts`)

```ts
export { Font, type FontConfig, type FontFace, type FontFaceRef, type FontStyleName }
  from "./layout/text/font.js";
export { type BufferState } from "./engine/composition.js";
export { type FontLoader, setDefaultFontLoader, getDefaultFontLoader }
  from "./engine/runtime-defaults.js";
```

`TextConfig` gains `font?` (already exported via `types.ts`).

File placement: `packages/core/src/layout/text/font.ts` (adjacent to `Text`,
which imports the `Font` type). The marker stays in `media/media-marker.ts`,
which already holds the non-media `TICK_MARK`.

---

## 8. Implementation checklist

1. `media-marker.ts`: add `FONT_MARK`.
2. `runtime-defaults.ts`: add `FontLoader` + `domLoadFont` (FontFace API) +
   `loadFontFace` (the deduped, cached entrypoint) + `setDefaultFontLoader`.
3. `layout/text/font.ts`: `Font` class, `normalizeFace` (+ warn on duplicate
   weight/style in `faces`), `face()` selection, `_kmRegister`. `load()` calls
   `loadFontFace` per face.
4. `engine/composition.ts`: `buffer`/`isBuffering` signals, `_assets`,
   `registerAsset`, `whenReady`, buffer-aware `play()` (`_startClock` extract +
   `_resumeOnReady`), and `FONT_MARK` walk in `add`.
5. `layout/text/types.ts` + `text.ts`: `font?` field, resolution to
   family/fontStyle, re-layout-on-ready hook.
6. `index.ts`: exports.
7. `renderer/src/font-loader.ts` + `setup.ts`: `makeSkiaFontLoader(cacheDir)`
   with remote-URL→disk-cache + process-level dedup; `setupServerRendering`
   gains `fontCacheDir`; wire `setDefaultFontLoader`.
8. `pnpm build` + `pnpm check`. Add a demo scene using a local `?url` font, and a
   render smoke test using a **remote** font URL (exercises the disk cache).
9. Update `doc/README.md` target-API example (per CLAUDE.md gotcha).

## 9. Open questions

- **Buffer scope now vs later**: wire only `Font` into `registerAsset` in this
  change, or also `Image`/`Video`? (Plan: Font now, others as a fast follow.)
- **`play()` semantics**: auto-defer-and-resume (proposed) vs. `play()` throws
  while buffering and the caller must `await whenReady()`. Proposed favors the
  video-player UX; confirm before building.
- **Variable fonts**: `weight: "100 900"` is accepted by `FontFace` but face
  *selection* by exact weight string needs a rule (treat a range as matching any
  contained weight?). Deferred unless needed now.
- **`face.src` preview vs server**: ✅ handled. A build-local `?url` import is
  rewritten to an absolute fs path in the SSR build by `@konva-motion/vite`
  (`serverAssets` regex now covers `woff2?|ttf|otf`; the `?url` query is stripped
  before matching), which the skia loader reads directly. Remote `http(s)` srcs
  also "just work" on both sides — the browser fetches them, the skia loader
  downloads + disk-caches. So no `{ preview, server }` src-pair is needed.
- **Cache invalidation**: the server disk cache is keyed by URL hash, so a font
  served from a stable URL is cached forever. Fine for immutable/CDN-hashed URLs;
  add a TTL or `?v=` convention if mutable URLs are expected.
- **Family-conflict policy**: warn-only (proposed) vs. throw when two faces /
  two `Font`s collide on `family|weight|style`. skia silently first-wins, so
  warn-only matches its behavior; throwing would be stricter but could break
  hot-reload re-registration.
