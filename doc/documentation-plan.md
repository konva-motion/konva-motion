# konva-motion Documentation Plan

Tutorial-focused docs ("how to" + "when"), not an API reference. Built on the
existing Fumadocs + React Router site in `packages/docs`, with live previews via
the `<km-player>` web component. Tone follows Remotion (imperative, frame-first)
and leans on Konva's config-object conventions.

## Locked decisions

- **Teach the real API.** Every page uses `Composition` / `Sequence` /
  `seq.register()` / `interpolate()` / `Easing`. The current
  `introduction.mdx` / `installation.mdx` teach a `Timeline.to()` API that does
  **not** exist in core — these get rewritten.
- **Studio is a full section** (4 pages), banner-marked **Preview** (it is
  `0.0.0`, React-only, lives in `demo2`).
- **Full live demo library** — one `.ts` composition per concept in
  `packages/docs/src/demos/`, rendered live in `<km-player>` and shown as
  source from the *same file* (single source of truth).
- **All demos run at 60fps.** New demos use `fps: 60`; when you need a specific
  real-time pace, size `durationInFrames` (and any frame anchors / interpolate
  ranges) for 60fps, not 30. The four ported demos were converted: frame counts,
  `from`/`trim` windows, and frame-rate-dependent oscillators were all doubled or
  halved so timing and media sync are unchanged. If a page shows a demo's code
  inline (e.g. `installation` mirrors `first-tween`), keep the inline copy in
  sync with the demo file.

## Tone of voice

- Imperative, frame-first. Lead with the smallest thing that runs.
- Reinforce one mental model everywhere: composition owns `fps` +
  `durationInFrames`; a `Sequence` is a range-gated layer; each tick runs
  updaters and `batchDraw`s active sequences. Animation = "what is this
  attribute *at frame N*" — never CSS transitions or `setInterval`.
- Config-object parity with Konva: "every `Konva.RectConfig` prop works, plus
  flex props." Link out to Konva docs instead of re-documenting raw attributes.
- Each page shape: *what it's for, smallest runnable example, live demo,
  2-3 variations, when to reach for it + gotchas.*

### Write like a human, not an LLM

The docs should read as if a developer wrote them. Avoid the tells that mark
machine-generated prose:

- **No em-dashes or en-dashes (`—`, `–`).** Use a plain hyphen `-`, a comma, a
  colon, parentheses, or just split the sentence. This is the biggest tell;
  scrub every one.
- **No "it's not just X, it's Y" / "X isn't merely Y" constructions.** Say the
  thing directly.
- **Drop filler superlatives** ("seamless", "robust", "powerful", "effortless",
  "leverage", "delve", "boasts", "unlock", "elevate"). State what it does.
- **Go easy on bold.** Bold a term on first definition, not every other phrase.
- **No "Let's dive in", "In conclusion", "It's worth noting that".** Cut throat-
  clearing; lead with the point.
- **Vary sentence length; allow short sentences.** Don't make every sentence a
  balanced two-clause structure.
- Use straight quotes/apostrophes (`'` `"`) in prose where practical.

Apply this to page prose, demo source comments, and code-block comments. When
editing existing pages, treat an em-dash as a defect to fix.

## Best practices to teach throughout

These are the recommended-way principles. Define each once on a short
**`best-practices`** page (Getting Started, after core-concepts), then reinforce
the relevant one as a `<Callout>` on the pages where it bites. Frame them as
*recommended*, not *required* — most have a working escape hatch.

- **Prefer konva-motion wrappers over raw `Konva.*`.** `Rect`/`Circle`/`Text`/
  `Image`/… from core extend their Konva equivalents *and* add full flex
  participation. NOTE (verified against `layout/flex/flex.ts`): a raw
  `Konva.Rect`/`Text`/`Image` with a fixed numeric size **is** measured and
  placed (and origin-corrected) inside a `Flex` — the old "won't reflow / won't
  be measured" framing is inaccurate. What raw nodes *can't* do is express
  percentage sizes (`width: "100%"`), flex child props
  (`flexGrow`/`flexShrink`/`flexBasis`/`alignSelf`), nested `Flex`/`Block`
  containers, or the rich `Text` (fit/highlights/typewriter) / `Image`
  (objectFit) / `Block` (background/border/shadow) features. Rule of thumb:
  import the drawing vocabulary from `@konva-motion/core`, reach for `Konva.*`
  only for something core doesn't wrap yet. *(Not never, just not the default.)*
- **Animate as a function of frame.** Drive attributes from `localFrame` via
  `seq.register()` + `interpolate()`/`Easing`. Never `setInterval`,
  `requestAnimationFrame`, `Konva.Tween`, or CSS transitions — only frame-derived
  values render deterministically and seek/scrub correctly.
- **Keep compositions pure.** A composition should be a pure function of
  `frame` + `props` — no external mutable state, no wall-clock reads. This is
  what makes preview, scrubbing, and headless render produce identical output.
- **Add a node before you animate it.** Attach to a `Sequence`/layer first;
  tweening a detached node updates attrs but never repaints (existing gotcha).
- **Pin one copy of `konva`.** It's a peer dep and core extends its classes —
  two copies break `instanceof` and layout. Dedupe in the consumer.
- **Let layout reflow; don't hand-position when you mean layout.** Use
  `Flex`/`Block` and animate `gap`/size/`padding`; the engine recomputes each
  tick. Manual `x`/`y` math is for one-off motion, not structure.
- **`play()` is browser-only; `setFrame(n)` works anywhere.** Use `setFrame`
  for offline/server paths (it's also what the renderer drives).
- **Use `interpolate` extrapolation, not manual clamping.** Reach for
  `extrapolateLeft/Right: "clamp"` instead of `Math.min/max` around a ratio.

## Target information architecture (`meta.json`)

```
---Getting Started---
introduction          (rewrite)
installation          (rewrite)
core-concepts         (new)
best-practices        (new)
---Animating---
interpolation         (new)
easing                (new)
sequencing            (new)
---Drawing---
shapes                (new)
text                  (new)
images                (new)
---Layout---
flex                  (split from layout-and-shapes)
block                 (split — "box")
---Media---
video                 (new)
audio                 (new)
---Transitions---
transitions           (new)
---Player---
player                (rewrite/expand)
---Rendering---
rendering             (new)   ← headless MP4 export via @konva-motion/renderer
rendering-media       (new)   ← server-side assets, fonts, shader transitions (./gl)
---Studio---  (Preview)
studio-overview       (new)
studio-setup          (new)
studio-interface      (new)
studio-rendering      (new)
```

The existing `components.mdx` (design-system kitchen-sink) drops out of public
nav; its role is replaced by the focused Drawing/Layout pages. Keep the file as
an internal authoring reference.

---

# Build steps

## Step 0 — Foundations (do first, unblocks everything) ✅ DONE

- [x] Restructure `content/docs/meta.json` to the IA above (group headers +
      ordered slugs). *Applied the new group headers (Getting Started, Player)
      with only the pages that exist today; later steps append their slugs as
      pages are authored. `components`/`layout-and-shapes` dropped from public
      nav (files kept on disk as internal reference).*
- [x] Formalize the **demo authoring contract**: each demo is
      `src/demos/<name>.ts` whose default export is a `Composition` (or a
      `() => Composition` factory). `<Demo>` renders it live via `?url` into
      `<km-player src>` and shows source via `?raw` — from the same file.
- [x] Decide + (optionally) implement the `<Demo name="...">` single-prop
      helper that resolves both `?url` and `?raw` from one name, replacing the
      two-import-per-page boilerplate. *Implemented via two eager
      `import.meta.glob` maps in `src/components/demo.tsx`; `name` resolves both
      views (throws on unknown name). `src`/`source` props kept as an override.
      `src/demos/registry.ts` deleted (superseded). `player.mdx` migrated to
      `<Demo name="orbit" />` and verified live.*
- [x] Port the 4 demos that already exist in core into `src/demos/`:
      `flex-layout`, `text-highlight`, `video-sync`, `audio-mixer`. *Ported from
      `demo/src/compositions/*/composition.ts`; media copied into
      `src/demos/assets/` and imports rewritten to `./assets/...`.*
- [x] Add a short contributor note (`doc/authoring-demos.md`) describing the
      contract.

## Step 1 — Getting Started (priority: current docs are actively wrong) ✅ DONE

Pages: `introduction`, `installation`, `core-concepts`, `best-practices`.

- [x] **introduction** — pitch; frame-clock mental model (ASCII playhead diagram
      over `[from, from+duration)`); when to reach for it vs plain Konva /
      single CSS tween; `hero` live demo. *Next-steps links repointed to
      installation/core-concepts/best-practices (dropped the `components` link).*
- [x] **installation** — `npm install konva @konva-motion/core`; konva-as-peer
      Callout; **rewritten** first composition (`Composition` + `Sequence` +
      `register` + `interpolate`); mount via `setContainer` + `play()`; `<Steps>`
      walkthrough rebuilt around the real API. The `first-tween` demo IS the page
      example (View source). *Replaced the old non-existent `Timeline`/`tl.to`
      API.*
- [x] **core-concepts** — `Composition({id, fps, durationInFrames, width,
      height, loop})`; `Sequence({from, durationInFrames})` + range-gating
      (`range-gate` demo); local vs global frame; the tick loop (updaters → media
      → layout → `batchDraw`, `frame-clock` demo); `play()` (browser) vs
      `setFrame(n)` (anywhere).
- [x] **best-practices** — the recommended-way principles (see "Best practices
      to teach throughout"); leads with "prefer the core wrappers over raw
      `Konva.*`" (`wrapper-vs-raw` demo) and "animate as a function of frame."
      Each principle has a one-line *why* + a do/don't snippet. Later pages link
      back here via `<Callout>`s.
- [x] Demos: `hero`, `first-tween`, `frame-clock`, `range-gate`,
      `wrapper-vs-raw`. *All five added to `src/demos/`, use core wrappers, run at
      `fps: 60`, verified live in `<km-player>`.*
      - `wrapper-vs-raw` shows **flexGrow**: as a Flex widens, the core `Rect`
        (`flexGrow: 1`) fills the row while a raw `Konva.Rect` can't and leaves a
        gap. *(NOT the original "raw won't reflow" idea — that was found false;
        see the corrected wrappers bullet under "Best practices to teach
        throughout".)*
      - Gotchas caught: (1) the core `Text` wrapper extends `Konva.Group`, so
        update its content with `setText(...)`, not Konva's `.text(...)`; (2) the
        `FlexShape` `.width()` setter changes the flex size value, not the render
        width — to grow a leaf use `flexGrow`/layout, not `.width(n)` in an
        updater; (3) add the full-canvas background `Sequence` **first** so it
        sits under the content layers (Konva layer order = add order).
- [x] Nav: appended `core-concepts` + `best-practices` to the Getting Started
      group in `content/docs/meta.json`.

## Step 2 — Animating

Pages: `interpolation`, `easing`, `sequencing`.

- [ ] **interpolation** — `interpolate(input, inputRange, outputRange,
      {extrapolateLeft/Right})`; multi-stop ranges; `interpolateColors`.
- [ ] **easing** — `Easing.*`, `Easing.in/out/inOut`, `Easing.bezier`,
      elastic/back/bounce; spring feel; same move under 4 curves.
- [ ] **sequencing** — `from`/`durationInFrames`; staggering N nodes; `Series`
      for back-to-back scenes; trim-in/out.
- [ ] Demos: `interpolate-basics`, `color-shift`, `easing-compare`, `stagger`,
      `series-scenes`.

## Step 3 — Drawing

Pages: `shapes`, `text`, `images`.

- [ ] **shapes** — the `FlexShape` wrapper story; full shape list (`Rect`,
      `Circle`, `Ellipse`, `Line`, `Arrow`, `Star`, `Ring`, `Arc`, `Wedge`,
      `RegularPolygon`, `Path`, `TextPath`, `Sprite`); Konva config parity
      (link out); the "delete absent width/height" gotcha Callout; `getSelfRect`
      intrinsic size.
- [ ] **text** — `Text` config; typewriter; animated `highlights.progress`;
      `fitText`/`wrap`/`maxLines`.
- [ ] **images** — `src`, `objectFit`, `objectPosition`, `cornerRadius`,
      custom `loader` (note the server-render asset-URL helper).
- [ ] Demos: `shapes-gallery`, `shape-morph`, `typewriter`, `text-highlight`
      (ported), `image-objectfit`.

## Step 4 — Layout

Pages: `flex`, `block`.

- [ ] **flex** — flexbox via flexily; `flexDirection`/`justifyContent`/
      `alignItems`/`gap`/`padding`; child props
      (`flexGrow`/`flexShrink`/`flexBasis`/`alignSelf`); reflows every tick —
      animate `gap`/size and watch layout follow.
- [ ] **block** ("box") — styled flex container: `background` (solid +
      `{gradient}`), `borderSize`/`borderColor`/`borderStyle`, `cornerRadius`,
      `shadow`; build a card.
- [ ] Demos: `flex-layout` (ported), `flex-reflow`, `block-card`.

## Step 5 — Media

Pages: `video`, `audio`.

- [ ] **video** — `Video({src, trimBefore/After, loop, muted, volume,
      objectFit})`; frame-clock sync; multi-clip sync; server-render Callout
      (skia backend).
- [ ] **audio** — `Audio({src, name, volume, muted, trim…})`; `comp.mixer`
      master volume/mute; layering music + voiceover; markers mention.
- [ ] Demos: `video-sync` (ported), `audio-mixer` (ported).

## Step 6 — Transitions

Page: `transitions`.

- [ ] **transitions** — `TransitionSeries({composition})`; `.scene(...)` +
      `.transition({presentation, timing})`; geometric presentations (`fade`,
      `slide`, `wipe`, `clockWipe`, `iris`, `flip`, `bookFlip`); `linearTiming`
      vs `springTiming`; Callout on shader presentations (`dissolve`,
      `crosswarp`, …) and the renderer `./gl` path.
- [ ] Demos: `transition-fade`, `transition-slide`, `transition-spring`.

## Step 7 — Player

Page: `player` (rewrite/expand).

- [ ] **player** — `<km-player>` attributes (`src`, `controls`, `loop`,
      `autoplay`, `muted`, `volume`, `playbackrate`, `initialframe`); the `src`
      module contract (a `?url` module the player `import()`s); composing custom
      controls (`KmPlayerPlayButton`, `Progress`, `SoundControl`,
      `createDefaultControls`); imperative API + events (`time`, `play`,
      `seekTo`, `setProps`); the standalone `<script type="module">` CDN path.
- [ ] Demos: `orbit` (exists), `player-custom-controls`.

## Step 8 — Rendering

Pages: `rendering`, `rendering-media`. The renderer is Node/headless and
**API-only (no CLI)** — these pages use annotated code blocks, not
`<km-player>` previews. Where helpful, embed the resulting MP4 as a `<video>`.

- [ ] **rendering** — "your composition → MP4 file." `import
      "@konva-motion/renderer/register"` (= `setupServerRendering()` at import)
      before building the comp; `renderComposition(comp, {output, fps,
      resolution, fit, quality, range, mute, onProgress, signal})`; quality
      presets (`low`/`medium`/`high`/`max`); `RenderResult`; progress +
      `AbortSignal`. Single frame with `renderStill({frame, type})`; raw frames
      with the `renderFrames()` async generator. Call out the no-CLI note (run
      a `.ts` file with `tsx`/`node`).
- [ ] **rendering-media** — server-side assets & advanced: the `mediaSrc`
      asset-URL helper (Vite URL → fs path) so media comps render headlessly;
      custom image/video source factories and `setupServerRendering({video,
      fonts, videoDecodeCap})`; font registration (`registerFonts`); how audio
      is collected + muxed (`collectAudioTrack`, ffmpeg mix); shader (Tier B)
      transitions via `import "@konva-motion/renderer/gl"` + the headless-gl
      optional dep and `fade()` fallback when `gl` is absent.
- [ ] No live demos — annotated code + optional embedded result `<video>`.

## Step 9 — Studio (Preview)

Pages: `studio-overview`, `studio-setup`, `studio-interface`,
`studio-rendering`. Preview banner on each.

- [ ] **studio-overview** — what it is; when to use; screenshot/embed;
      `0.0.0` preview Callout.
- [ ] **studio-setup** — `<Studio registry render>`; a registry entry
      (`{id, title, propsSchema, render}`); the `kf` DSL (`text`/`number`/
      `color`/`select`/`boolean`/`object`/`array`/`divider`) → props form;
      `selectedId`/`onNavigate` routing.
- [ ] **studio-interface** — compound parts: `Studio.Stage`, `Studio.Timeline`
      (`TimelineHeader`/`Transport`/`Scrubber`/`LayeredBody`), `Studio.Panel`/
      Tabs (inspector), `Studio.Header`/`Zoom`; hooks (`useStudio`,
      `useComposition`, `useSignalValue`, `usePlayback`, `useShortcuts`).
- [ ] **studio-rendering** — `Studio.RenderDialog`, `Studio.ExportFrameDialog`,
      `Studio.RenderQueue`; render backend prop; how the in-app render queue
      submits to `@konva-motion/renderer` (link back to the Rendering section).
- [ ] Assets: Studio is React (not a `<km-player>` composition) — use static
      screenshots for v1; revisit live React embed/iframe of demo2 later.

## Step 10 — Verify & ship

- [ ] `pnpm build` the docs.
- [ ] Load every page; confirm each `<km-player>` demo imports and plays (use
      preview tools); fix failures.
- [ ] Screenshot proof of key pages.
- [ ] Update `doc/README.md` if any public API examples changed.

---

# Demo library map

Authoring contract: `src/demos/<name>.ts`, default export = `Composition` or
`() => Composition`. Live preview via `?url`; displayed source via `?raw`.

| Page | Demo files |
| --- | --- |
| introduction | `hero` |
| installation | `first-tween` |
| core-concepts | `frame-clock`, `range-gate` |
| best-practices | `wrapper-vs-raw` |
| interpolation | `interpolate-basics`, `color-shift` |
| easing | `easing-compare` |
| sequencing | `stagger`, `series-scenes` |
| shapes | `shapes-gallery`, `shape-morph` |
| text | `typewriter`, `text-highlight`¹ |
| images | `image-objectfit` |
| flex | `flex-layout`¹, `flex-reflow` |
| block | `block-card` |
| video | `video-sync`¹ |
| audio | `audio-mixer`¹ |
| transitions | `transition-fade`, `transition-slide`, `transition-spring` |
| player | `orbit` (exists), `player-custom-controls` |
| studio | screenshots + one registry snippet |

¹ Already exists as a core demo — port into `src/demos/`.

# Open questions

- Fast-track the `introduction` + `installation` rewrite as a standalone fix
  (current published docs teach a non-existent API), or bundle into the full
  build?
- Adopt the `<Demo name="...">` single-prop helper, or keep explicit
  two-import-per-page?
- Studio live previews: static screenshots for v1 (default), or invest in an
  embedded React island / iframe of demo2?
