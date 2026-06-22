# konva-motion Documentation Plan

Tutorial-focused docs ("how to" + "when"), not an API reference. Built on the
existing Fumadocs + React Router site in `packages/docs`, with live previews via
the `<km-player>` web component. Tone is imperative and frame-first, and leans on
Konva's config-object conventions.

## Locked decisions

- **Teach the real API.** Every page uses `Composition` / `Sequence` /
  `seq.register()` / `interpolate()` / `Easing`. The current
  `introduction.mdx` / `installation.mdx` teach a `Timeline.to()` API that does
  **not** exist in core — these get rewritten.
- **Don't name Remotion in page prose.** Remotion is inspirational only. Credit
  Remotion and Konva once, in a **Credits** section on the `introduction` page,
  and nowhere else. No "Remotion-style" / "API-compatible with Remotion" /
  "mirrors Remotion" phrasing in the published pages or demo comments. (Internal
  design docs under `doc/` may still reference it.)
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

## Authoring gotchas (engine reality, verified)

Practical pitfalls hit while building the demos. The contributor copy lives in
[`doc/authoring-demos.md`](authoring-demos.md); keep the two in sync. (This is an
internal plan, so em-dashes here are fine — the no-em-dash rule is for published
page prose and demo comments.)

- **Layer order = add order.** A `Sequence` is a Konva layer; later-added
  sequences paint on top. Add a full-canvas background `Sequence` **first** so it
  sits under the content, not over it. (Bit `range-gate`: cards were hidden
  behind a background added last.)
- **Update `Text` content with `setText(...)`.** The core `Text` wrapper extends
  `Konva.Group`, not `Konva.Text`, so it has no `.text()` accessor; `.text(...)`
  throws. Other wrappers (`Rect`/`Circle`/`Line`/…) extend their `Konva.Shape`,
  so `.opacity()`/`.x()`/`.points()`/etc. work as normal.
- **Don't grow a flex leaf with `.width(n)`.** `FlexShape.width(n)` sets the flex
  *size value*, not the rendered width, so it won't visibly resize a leaf from an
  updater. To make a child fill space use `flexGrow` (or a `%` width) and let the
  layout reflow. To animate a Flex *container's* own size, `setAttrs({ flexWidth })`
  (the constructor takes `width`, stored as the `flexWidth` attr).
- **Raw `Konva.*` IS laid out inside a Flex.** Verified against
  `layout/flex/flex.ts`: a raw `Konva.Rect`/`Text`/`Image` with a numeric size is
  measured, placed, and origin-corrected. The wrappers' real edge is `%` sizes,
  flex child props, nested containers, and the rich Text/Image/Block features
  (see the wrappers bullet above). Don't teach "raw won't reflow."
- **60fps scaling** when authoring or porting: see the 60fps locked decision.

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
transitions-setup     (new)   ← install @konva-motion/transitions (why + how)
transitions           (new)
---Player---
player-setup          (new)   ← install @konva-motion/player (npm + CDN/standalone)
player                (rewrite/expand)
---Rendering---
rendering-setup       (new)   ← install @konva-motion/renderer (native reqs, ./gl)
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

**Per-section setup pages (added after Step 7).** `player`, `transitions`, and
`renderer` are separate npm packages. Each add-on section leads with a short
`*-setup` page carrying the install "why + how" (exact `npm install` from the real
peer deps, peer/native notes), so the usage pages stay compact and focused on the
API. Core keeps `installation` as its Getting Started setup page; the add-on setup
pages build on it and link back instead of repeating the konva-peer rule. The
usage pages carry only a one-line setup `<Callout>` linking to their setup page.
The player's CDN/standalone build moved from `player.mdx` to `player-setup.mdx`.

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

## Step 2 — Animating ✅ DONE

Pages: `interpolation`, `easing`, `sequencing`.

- [x] **interpolation** — `interpolate(input, inputRange, outputRange,
      {easing/extrapolateLeft/Right})`; multi-stop ranges; the four extrapolate
      modes (table) with `"clamp"` as the common one; `easing` option (link to
      easing); `interpolateColors`. *`interpolate-basics` + `color-shift` demos.*
- [x] **easing** — easing reshapes the 0→1 progress inside `interpolate`'s
      `easing` option (no separate tween API); presets table; `in/out/inOut`
      modifiers; factories `bezier`/`elastic`/`back`/`poly`; same move under 5
      curves via `easing-compare`.
- [x] **sequencing** — recap `from`/`durationInFrames` + local frame (link to
      core-concepts); staggering N nodes in one sequence via `frame - i * step`
      (`stagger`); `Series` for back-to-back scenes (`series-scenes`), `offset`
      semantics (0/gap/overlap), `comp.add(series)` auto-expands.
- [x] Demos: `interpolate-basics`, `color-shift`, `easing-compare`, `stagger`,
      `series-scenes`. *All run at `fps: 60`, loop, core wrappers only.*
- [x] Nav: appended `---Animating---` group + the three slugs after Getting
      Started in `content/docs/meta.json`.

Verified facts / corrections caught while building:
- `Easing.in(fn)` is an **identity no-op** in konva-motion (a bare preset already
  eases in) — documented as such; reach for `out`/`inOut` to change the shape.
- `interpolateColors(input, inputRange, outputRange)` takes **no options arg** and
  **always clamps** both ends (unlike `interpolate`). Documented as a Callout.
- `Series.add({durationInFrames, offset}, (seq) => …)`: `offset` 0 = back-to-back,
  positive = gap, negative = overlap (cross-fade); `comp.add(series)` expands via
  `SequenceProvider`. `series-scenes` adds a base background `Sequence` first so
  scenes have a backdrop while they cross-fade.
- Sequence has **no trim props** (those are on `Video`/`Audio`) — Callout on the
  sequencing page so the distinction is explicit.
- Forward links to not-yet-authored pages (`video`/`audio`/`shapes`/`transitions`)
  left in place, matching how Step 1 linked forward to these pages before authoring.

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

## Step 4 — Layout ✅ DONE

Pages: `flex`, `block`.

- [x] **flex** — flexbox via flexily; `flexDirection`/`justifyContent`/
      `alignItems`/`gap`/`padding`; child props
      (`flexGrow`/`flexShrink`/`flexBasis`/`alignSelf`/`margin`); px/`%` sizing;
      reflows every tick - animate `gap`/size and watch layout follow. Container
      and child props each in a table; `flex-reflow` + `flex-layout` demos.
- [x] **block** ("box") — styled flex container: `background` (solid +
      `{gradient}` linear/radial), `borderSize`/`borderColor`/`borderStyle`,
      `cornerRadius`, `shadow` (`{color,blur,offsetX,offsetY,opacity}`); built a
      card with `block-card`. Note that Block is a Flex, so every flex prop
      applies.
- [x] Demos: `flex-layout` (ported, reused), `flex-reflow`, `block-card`. *Both
      new demos run at `fps: 60`, loop, core wrappers only, dark-bg Sequence
      first.*
- [x] Nav: inserted `---Layout---` group + `flex`/`block` slugs after Drawing in
      `content/docs/meta.json`.
- [x] Verified: docs `pnpm build` passes; both pages load and every demo paints
      (checked the per-layer Konva canvases directly - flex-reflow's four chips
      with the grow-chip widened, block-card's gradient/border/shadow/text).

Verified facts / corrections caught while building:
- **One Sequence, one Konva layer, one canvas.** Each `Sequence` is its own
  `Konva.Layer`, so the player has one `<canvas>` per active sequence. Sampling
  "the canvas" only sees the bottom layer; the content layer is a separate
  canvas. (Bit verification, not the page - noting so the next author doesn't
  chase a "blank" demo that is actually painting on layer 2.)
- **Layout roots must be DIRECT children of the Sequence.** `Sequence._apply`
  calls `_kmComputeLayout()` only on its own top-level children
  (`sequence.ts:73-75`), after running updaters. So the `Flex`/`Block` you
  animate must sit directly under the Sequence that holds its `register` updater,
  or it never reflows. Both demos put the layout root directly in its Sequence.
- **`flexWidth`/`flexHeight`, not `.width()`.** Confirmed live: the Flex/Block
  constructor stores `width`/`height` config as the `flexWidth`/`flexHeight`
  attrs (`flex.ts:54-55`, `block.ts:115-116`); `layoutRoot` reads those, not
  Konva's bbox `width()`. Animate the container's own size with
  `setAttrs({ flexWidth })`. Documented as a `<Callout type="warn">` on the flex
  page.
- **Block always sizes to content.** `computeLayout()` calls `layoutRoot(this,
  true)` so the background rect is never 0x0 even when the Block hugs its
  children. The bg rect is `moveToBottom()`'d so children render on top
  (`block.ts:129-167`).
- Forward links from the Drawing pages (`shapes.mdx`, `text.mdx`) to `/docs/flex`
  and `/docs/block` now resolve.

## Step 5 — Media ✅ DONE

Pages: `video`, `audio`.

- [x] **video** — `Video({src, trimBefore/After, loop, muted, volume,
      objectFit})`; frame-clock sync; multi-clip sync; server-render Callout
      (skia backend). Props table + `setVolume`/`setMuted`/`setPlaybackRate`;
      `objectFit`/`objectPosition` cross-link to `images`.
- [x] **audio** — `Audio({src, name, volume, muted, trim…})`; `comp.mixer`
      master volume/mute; layering music + voiceover; frame-driven volume
      automation via `register` + `interpolate` + `setVolume`. Dropped the
      "markers mention" - there is no public marker API (see below).
- [x] Demos: `video-sync` (ported), `audio-mixer` (ported). Both already lived in
      `src/demos/` from Step 0; this step was page prose + nav + verification.
- [x] Nav: inserted `---Media---` group + `video`/`audio` slugs after Layout in
      `content/docs/meta.json`.
- [x] Verified: docs `pnpm build` passes; both pages serve 200 and render; the
      `video-sync` clip paints a real video frame on its content layer (canvas 1
      colored, base layer separate); the `audio-mixer` visual layer animates
      between frames (Music A ducked to 25%, Voice at 100% at frame 300). New nav
      group + prev/next links resolve.

Verified facts / corrections caught while building:
- **No public marker API.** `media/media-marker.ts` (`MEDIA_MARK`/`VIDEO_MARK`/
  `AUDIO_MARK`/`TICK_MARK`) is internal node-discovery only - nothing is exported
  from the barrel. The planned "markers mention" was dropped; media nodes are
  auto-discovered by the `Sequence`, no manual registration.
- **`.volume`/`.muted` are read-only signals.** Both `Video` and `Audio` expose
  `volume`/`muted` as `ReadonlySignal`s; set them with `setVolume(0..1)` /
  `setMuted(bool)` (and `setPlaybackRate(n)`), never by assignment. Documented as
  a `<Callout type="warn">` on the audio page.
- **`Video` reuses `Image`'s fit logic.** `objectFit`/`objectPosition` are the
  exact `layout/image.ts` controls, so the video page links to `images` rather
  than re-documenting the fit modes.
- **Effective level = master × track.** `comp.mixer` scales every channel; a
  track is silent when either the master or the track is muted. The audio demo
  drives intrinsic levels from the frame and the mixer applies master on top.
- **Audio renders nothing.** An `Audio` node is `visible:false`/`listening:false`,
  so its `Sequence` produces a blank canvas; the demo's visuals live on a
  separate base `Sequence` (one Konva layer/canvas per sequence - relevant when
  verifying, sample all canvases not just the first).
- **Audio sequences added before the visual layer.** `audio-mixer` adds the five
  audio sequences first, then the base visual `Sequence` last so visuals paint on
  top (Konva layer order = add order).
- Forward links to not-yet-authored `/docs/rendering` and `/docs/rendering-media`
  left in place, matching how earlier steps linked forward before authoring.

## Step 6 — Transitions ✅ DONE

Page: `transitions`.

- [x] **transitions** — `TransitionSeries({composition, from?})`; `.scene(...)` +
      `.transition({presentation, timing})`; geometric presentations (`fade`,
      `slide`, `wipe`, `clockWipe`, `iris`, `flip`, `none`); `linearTiming`
      vs `springTiming` (option tables); Callout on shader presentations
      (`dissolve`, `crosswarp`, `bookFlip`, …) and the renderer `./gl` path; a
      second Callout on the structural rules the series enforces.
- [x] Demos: `transition-fade`, `transition-slide`, `transition-spring`. *All
      run at `fps: 60`, loop, core wrappers only; each is a `TransitionSeries`
      whose composition `durationInFrames` equals the net (Σ scenes − Σ
      transitions) so the loop is seamless.*
- [x] Dep + nav: added `@konva-motion/transitions` as a `workspace:*` dependency
      of `@konva-motion/docs` (it was missing); inserted `---Transitions---` +
      `transitions` between Media and Player in `content/docs/meta.json`.
- [x] Verified: `pnpm build` (root) and `pnpm --filter @konva-motion/docs build`
      pass; `/docs/transitions` serves 200 and renders with the new nav group;
      all three players load (correct scene/canvas counts 3/3/2), no errors
      beyond the benign multi-Konva dev warning. Stepped through each transition
      window by forcing synchronous layer draws (see below).

Verified facts / corrections caught while building:
- **`bookFlip` is Tier B (shader), not geometric.** The plan text listed it
  under geometric presentations; `book-flip.ts` is built on `glTransition`
  (a WebGL2 fragment shader). The page lists it with the shader set. Confirmed
  geometric (Tier A) set is exactly: `fade`, `slide`, `wipe`, `clockWipe`,
  `iris`, `flip`, `none` (`presentations/index.ts:1-8`).
- **`@konva-motion/transitions` was not a docs dependency.** The demos import it,
  so it had to be added to `packages/docs/package.json` (`workspace:*`) before the
  page would build. The package was already built (`dist/` present).
- **`TransitionSeries` requires the composition up front** (`{composition, from?}`)
  because spring timings read `fps` and presentations read stage `width`/`height`
  from it. `comp.add(series)` expands it (it implements `SequenceProvider`).
- **Structural rules throw**: can't start/end with a transition, no two adjacent
  transitions, and a transition must be ≤ each neighbour scene's duration (it
  overlaps both). Documented as a `<Callout type="warn">`.
- **Net duration = Σ scenes − Σ transitions.** Each incoming transition pulls its
  scene back over the previous one by the transition length. The demo composition
  lengths are sized to this so a looped player has no empty tail.
- **`seq.register()` stacks** (Set of updaters); the series adds its presentation
  updater after the scene `build`, so the layer transform it applies wins last.
  Scene content can still animate child-node attrs.
- Verification was done by pixel/position sampling, not screenshots: the preview
  screenshot tool does not capture the player's `<canvas>` content in this headless
  setup. Forcing `comp.getLayers().forEach(l => l.draw())` after `setFrame(n)` gives
  a synchronous read. Confirmed fade cross-fades opacity (50/50 at the midpoint),
  slide pushes the incoming layer in from the edge while the outgoing leaves, and
  the spring overshoots past the target (negative x) then settles back.
- Forward link to not-yet-authored `/docs/rendering-media` left in place (shader
  Callout), matching how earlier steps linked forward before authoring.

## Step 7 — Player ✅ DONE

Page: `player` (rewrite/expand).

- [x] **player** — `<km-player>` attributes (`src`, `controls`, `loop`,
      `autoplay`, `muted`, `volume`, `playbackrate`, `initialframe`, plus the
      `no-*` / `double-click-fullscreen` flags); the `src` module contract (a
      `?url` module the player `import()`s, default export = Composition / factory
      / `{default}`); keyboard + click controls; the full imperative API (play/
      pause/toggle/stop/seekTo/stepBy/volume/loop/fullscreen/setProps) and the
      full event set (`frameupdate`/`timeupdate`/`play`/`pause`/`ended`/`seeked`/
      `ratechange`/`volumechange`/`mutechange`/`fullscreenchange`/`scalechange`/
      `loadstart`/`loaded`/`error`); composing custom controls (overlay +
      `<km-player-controls>` building blocks, `getPlayerApi`/`playerContext`,
      `createDefaultControls`); the standalone `<script type="module">` CDN path.
      Scrubbed the em-dashes left over from the old page.
- [x] Demos: `orbit` (exists, default controls) + `player-custom-controls` (new,
      its own composition) shown live with a hand-built control bar.
- [x] `<Demo>` enhancement: added an optional `children` prop
      (`src/components/demo.tsx`) so a page can render custom control markup inside
      `<km-player>` (drops the `controls` attr and the View-source toggle).
      Extended the ambient JSX typing (`src/km-player.d.ts`) with all the
      control-element tags so the MDX type-checks.
- [x] Verified: docs `pnpm --filter @konva-motion/docs build` passes; `/docs/player`
      serves 200 and both players mount independent compositions and paint (orbit
      `#ffd166` core, the custom-controls scene's puck); the custom bar upgrades
      with all six controls + the overlay play button in order; only the benign
      multi-Konva dev warning in the console; `/docs/transitions` still renders its
      three demos with the default bar (backward-compatible). No nav change needed
      (the `---Player---` group + `player` slug already existed in `meta.json`).

Verified facts / corrections caught while building:
- **Custom controls are page markup, not a composition.** The `<Demo>` system loads
  a composition module; control elements (`<km-player-controls>`, etc.) wrap the
  player. So the "demo" for custom controls is the control markup passed as
  `<Demo>` children, not a `player-custom-controls.ts` that *is* the controls.
- **Two `<Demo>` instances cannot share one demo module.** A demo file that
  default-exports a singleton `Composition` (like `orbit`) mounts into only one
  player at a time: `setContainer` moves the stage, so a second `<Demo name="orbit">`
  on the same page steals the canvas from the first (left it blank). Fixed by giving
  the custom-controls example its **own** module (`player-custom-controls.ts`). A
  demo meant to appear twice on one page would instead need a factory default export
  (`export default () => buildComp()`), which the player unwraps per mount.
- **Only six attributes are reactive.** `observedAttributes` is exactly `loop`,
  `controls`, `muted`, `volume`, `playbackrate`, `src`; `autoplay` and
  `initialframe` are read once at mount. Documented in a Callout.
- **`controls` with user-supplied `<km-player-controls>` suppresses the default
  bar** (`_reconcileControls` checks for a non-`data-km-default` controls child), so
  the custom-controls demo omits `controls` and the player keeps the hand-built bar.
- **Player uses light DOM**, so controls render into light DOM (no shadow root) and
  the opt-in `styles.css` (loaded globally in docs `root.tsx`) styles them.
- **CDN note:** jsdelivr/unpkg resolve files by path, so the inline `<script>` loads
  `…/dist/player.global.js` directly; the `@konva-motion/player/standalone` export
  subpath is for bundler resolution, mentioned separately.
- Pre-existing, out of scope: `pnpm typecheck` (`tsc -b`) fails on Step 6's
  `transition-fade.ts` (and siblings) under `noUncheckedIndexedAccess` for
  `scenes[i]` access; unrelated to Step 7 (flagged as a follow-up task). The Step 7
  files type-check clean.

## Step 8 — Rendering ✅ DONE

Pages: `rendering`, `rendering-media`. (`rendering-setup` already exists, authored
in the post-Step-7 setup-page split: it carries the `@konva-motion/renderer`
install, native requirements, the `/register` import, and the optional `gl` shader
path. Step 8's pages link back to it and focus on the API, not install.) The
renderer is Node/headless and **API-only (no CLI)** — these pages use annotated
code blocks, not `<km-player>` previews. *Result `<video>` embeds were skipped
(code-only): there is no static-asset/`<video>` pattern in the docs and it would
mean committing a binary; the headless nature is conveyed in prose instead.*

- [x] **rendering** — "your composition → MP4 file." `import
      "@konva-motion/renderer/register"` (= `setupServerRendering()` at import)
      before building the comp; `renderComposition(comp, {output, fps,
      resolution, fit, quality, range, mute, onProgress, signal, fonts,
      ffmpegPath})` (options table); quality presets table
      (`low`/`medium`/`high`/`max`, crf/x264-preset/audio-bitrate, `medium`
      default, custom `QualityConfig`); `RenderResult` table; `onProgress`
      (`{frame,total,fps,etaSeconds?}`) + `AbortSignal` snippets. Single frame
      with `renderStill({frame, output?, type, quality})` (returns a Buffer);
      raw frames with the `renderFrames()` async generator + a one-line
      `renderToStream` mention. No-CLI Callout (run a `.ts` with `npx tsx`).
- [x] **rendering-media** — server-side assets & advanced: the
      `@konva-motion/vite` `konvaMotion({serverAssets})` import rewrite (Vite URL
      → fs path in the SSR build) so media comps render headlessly **(NOT a
      `mediaSrc` helper — see correction below)**; font registration
      (`registerFonts` + the `fonts` option, why: no DOM `@font-face`); how audio
      is collected + muxed (`collectAudioTrack`, ffmpeg `amix`, `mute`, master
      mixer carry-through); `setupServerRendering({videoDecodeCap})` for memory;
      custom `video` source factory; shader (Tier B) transitions via `import
      "@konva-motion/renderer/gl"` + the optional `gl` dep and `fade()` fallback.
- [x] No live demos — annotated `ts`/`bash` code blocks only.
- [x] Nav: appended `rendering` + `rendering-media` after `rendering-setup` in the
      `---Rendering---` group in `content/docs/meta.json`.
- [x] Verified: `pnpm --filter @konva-motion/docs build` passes;
      `/docs/rendering`, `/docs/rendering-media`, `/docs/rendering-setup` all serve
      200 and render with the new nav group (all three slugs present), every
      section heading, the three rendering tables, the `#quality-presets` anchor,
      and no error overlay. Biome clean on touched files. Every snippet checked
      against the verified signatures and `renderer/examples/render-demo.ts`.

Verified facts / corrections caught while building:
- **No `mediaSrc` helper exists.** The plan named a `mediaSrc` asset-URL helper
  (Vite URL → fs path); `grep` finds no such export anywhere and the old
  `demo/src/media-src.ts` was deleted. The behavior now lives in the
  `@konva-motion/vite` plugin (`packages/vite/src/index.ts`): `konvaMotion({
  serverAssets })` is a `load` hook (`enforce: "pre"`) that rewrites media/image
  imports to an absolute fs path **in the SSR build only** (client keeps the URL),
  so `new Video({ src: clip })` works in both preview and render with no per-`src`
  helper. Default `serverAssets` covers mp4/webm/mov/m4v/mkv/mp3/wav/ogg/m4a/aac/
  flac/png/jpg/jpeg/webp/avif/gif. The page documents the plugin, not `mediaSrc`.
- **`FrameRange` is `{ from, to }` inclusive** (`renderer/types.ts`), used by
  `range` on `renderComposition`/`renderFrames` — e.g. `{ from: 0, to: 59 }` is
  the first second at 60fps.
- **`renderStill` returns a `Buffer`**; `output` is optional (write-and-return).
  `type` is `"png"` (default) or `"jpeg"`; `quality` 0..1 applies to JPEG.
- **Server audio is metadata-only.** `NullAudioSource` is used during frame draw;
  ffmpeg does the real decode/trim/volume-envelope/`amix`/mux in a separate pass.
  Frame-driven `setVolume` becomes a keyframed envelope; `comp.mixer` scales on
  top; `RenderResult.hasAudio` reports whether a track was written.
- **`videoDecodeCap`** (`setupServerRendering`) caps decode resolution to bound
  memory on long, video-heavy renders (skia retains every decoded frame's pixels);
  oversized clips are scaled down and Konva upscales into the display box.
- The page set kept it API-focused and linked back to `rendering-setup` for
  install/native reqs rather than repeating them.

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
| player | `orbit` (exists, default bar), `player-custom-controls` (new, own comp; custom bar = `<Demo>` children) |
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
