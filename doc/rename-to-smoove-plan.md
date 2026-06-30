# Rebrand plan: `konva-motion` → `smoove`

The project is rebranding from **konva-motion** to **smoove** — a unified
change that covers both the **mechanical rename** (package scope, identifiers,
tags, files) *and* the **visual identity** (palette, type, logo marks, product
look). It ships as **one coordinated rebrand** on a single branch; the phases
below are review checkpoints, and Phase 9 is the integration gate.

- **npm org:** `smoove` (scope `@smoove/*`) — secured
- **GitHub org/repo:** `smoove-dev/smoove` — secured
- **Domain:** `smoove.dev` — secured

**Decisions locked in:**

- `km-` prefix (web-component tag + sub-element tags + CSS classes) → `smoove-`
- Full identifier rename: `KmPlayer`→`SmoovePlayer`, `KmStudio`→`SmooveStudio`,
  `KonvaMotion`→`Smoove`, `window.KonvaMotion`→`window.Smoove`
- Rename asset files, source filenames, and the working folder too
- **Per-package styling** — each package hard-codes brand values; there is **no**
  shared token module. (Values are listed once in [§ Brand reference](#brand-reference)
  for copy-paste consistency.)
- **Animated logo mark is deferred** — static marks only in this pass.

**Scale:** ~907 substring hits across ~250 files for the mechanical rename
(mostly scoped find-replace), plus the design repaint of docs / player / studio.

> **Guiding rule:** only rebrand the `konva-motion` tokens. **Leave the
> underlying Konva library untouched** — the `konva` peerDep, `Konva.Stage`,
> `window.Konva`, and `import Konva from "konva"` all stay. Replace
> `konva-motion` / `@konva-motion` / `KonvaMotion` / `km-`, never bare
> `konva` / `Konva`.

---

## Design assets (source of truth)

The visual identity was handed off as an HTML/CSS/JS design bundle from Claude
Design. **These prototypes are the spec for the repaint phases** — recreate
their visual output (colors, type, spacing, the mark), don't copy their internal
structure.

- **Bundle (committed):** [`doc/color-palette-and-product-design.zip`](./color-palette-and-product-design.zip)
- **Extracted:** `doc/_design_extracted/color-palette-and-product-design/`
  - `README.md` — handoff notes (read first)
  - `project/Smoove Brand Brief.dc.html` — **the master spec**: full palette with
    hex + usage, gradient rules, typefaces, mark variants, do/don't, and the
    "How products should look" + "In practice" (studio) product mocks
  - `project/Smoove Lockup.dc.html` — lockups, size ladder (54/34/21/14px),
    backgrounds (white / cream / ink / gradient knockout), stacked lockup
  - `project/Smoove Logo Marks.dc.html` — mark exploration gallery
  - `project/Smoove Logo Animation.dc.html` — animated mark *(deferred)*
  - `project/smoove-mark.svg` — **primary** mark, gradient
  - `project/smoove-mark-dark.svg` — one-color dark (ink on paper)
  - `project/smoove-mark-light.svg` — one-color light (paper on ink)
  - `project/smoove-mark-animated.svg` — animated mark *(deferred)*
  - `project/screenshots/` — reference renders

Companion doc: [`doc/smoove-brand-brief.md`](./smoove-brand-brief.md) — the
*why/feel* (voice, pillars, positioning) behind the visuals above.

---

## Brand reference

Copy these values into each package (per-package styling — no shared module).

**Palette**

| Token | Hex | Role |
| --- | --- | --- |
| Coral | `#FF5640` | Primary. Gradient **start**. Buttons, big moments. (Dark text `#E03A28` for small labels.) |
| Mint | `#15CDA8` | Gradient **end** / the "trail". Live states, success. (Dark `#0E9E84`.) |
| Sunshine | `#FFC23C` | The wink. Badges, the keyframe dot. **Always dark text on top.** (Dark `#B07A00`.) |
| Grape Ink | `#261733` | Type, wordmark, dark surfaces. Warm, never flat black. |
| Cream | `#FFF6EC` | Default canvas. Warm, not clinical. |
| Paper White | `#FFFFFF` | Working surfaces. |

**The Smoove gradient:** `linear-gradient(102deg, #FF5640, #15CDA8)`. Hero
accent only — the mark, key accents, **one** focal surface per screen. Never a
full-page wash. Gradient on text only at display size.

**Ratio of use:** cream + grape carry ~90%; coral/mint/sunshine are the ~10%
spark.

**Ink-surface support tints** (from the mocks): panel `#34204A`, border
`#43305A`; light borders `#ECEAF0` / `#E6E2D8` / `#F0EEE7`; body text `#3B3947`,
muted `#6A6678` / `#9A96A6` / `#8E8AA0`.

**Type** (all free on Google Fonts):

| Face | Weights | Job |
| --- | --- | --- |
| **Comfortaa** | 400–700 | Display & wordmark (rounded geometric; headlines, logo) |
| **Hanken Grotesk** | 400–700 | Body & interface (paragraphs, labels, buttons) |
| **JetBrains Mono** | 400–600 | Code & data (samples, timestamps, metrics) |

**Surfaces:** rounded corners (~10–22px, default 16), warm paper backgrounds,
crisp 1px borders, restrained shadows, generous whitespace. The gradient is the
only loud thing on screen.

---

# Group A — Mechanical rename

Foundation. The build must stay green at the end of each phase. The hard guard
throughout: **never touch bare `konva` / `Konva` / `window.Konva`** — only the
`konva-motion` family of tokens.

| Old | New |
| --- | --- |
| `konva-motion` (root) | `smoove` |
| `@konva-motion/core` | `@smoove/core` |
| `@konva-motion/player` | `@smoove/player` |
| `@konva-motion/renderer` | `@smoove/renderer` |
| `@konva-motion/transitions` | `@smoove/transitions` |
| `@konva-motion/studio` | `@smoove/studio` |
| `@konva-motion/timeline` | `@smoove/timeline` |
| `@konva-motion/docs` | `@smoove/docs` |
| `@konva-motion/google-fonts` | `@smoove/google-fonts` |
| `@konva-motion/vite` | `@smoove/vite` |

## Phase 1 — Package identity & workspace relink ✅ DONE (2026-06-30)

- Rename `"name"` in **every `package.json`**: root → `smoove`, packages →
  `@smoove/*` per the table above.
- Update all `dependencies` / `peerDependencies` / `devDependencies` entries that
  reference the old scope (the `workspace:*` cross-deps).
- Add the (currently absent) `repository`, `homepage`, and `bugs` fields to the
  publishable `package.json`s, pointing at `github.com/smoove-dev/smoove` and
  `https://smoove.dev`.
- **Do not hand-edit `pnpm-lock.yaml`** (20 hits) — it regenerates.

**Verify:** `pnpm install` relinks the workspace under `@smoove/*` and
regenerates the lockfile with no errors.

**Status / notes:**

- All 10 `package.json` names renamed (`smoove`, `@smoove/*`); all `workspace:*`
  cross-deps repointed (player, renderer, transitions, studio, docs, demo).
- `repository` / `homepage` / `bugs` added to the **7 publishable** packages
  (`core`, `player`, `renderer`, `transitions`, `studio`, `google-fonts`,
  `vite`). Skipped the **3 private** ones — root `smoove`, `@smoove/docs`,
  `demo` — since they aren't published.
- Also swapped the product name inside each package's `description` field
  (`konva-motion` → `smoove`) — it's package-identity metadata sitting on the
  same lines. Bare `konva` / `Konva` left untouched per the guard.
- **No `timeline` package on disk.** The plan's table + Group A list
  `@konva-motion/timeline`, but `packages/timeline` does not exist (only a
  placeholder mention in CLAUDE.md). Nothing to rename; if it's ever scaffolded
  it should be created directly as `@smoove/timeline`.
- `pnpm install` (v10.33.4) succeeded — lockfile regenerated, **0** residual
  `@konva-motion` entries, 20 `@smoove/` entries, all 9 projects relink. (pnpm
  flagged an optional upgrade to 11.9.0 — not applied.)
- ⚠️ **`pnpm build` will fail until Phase 2** — source files still
  `import … from "@konva-motion/*"`. That's expected; Phase 1's gate is only
  `pnpm install`. Don't run the build as a Phase 1 check.

## Phase 2 — Imports, identifiers & browser globals ✅ DONE (2026-06-30)

- Every `import … from "@konva-motion/*"` → `@smoove/*` across `packages/*`,
  `demo/`, and docs (193 `core` imports alone).
- `KmPlayer` → `SmoovePlayer` (~50), `KmStudio` → `SmooveStudio` (~21).
- `KonvaMotion` → `Smoove` (~19), incl. the `import * as KonvaMotion` alias in
  `standalone.ts`.
- `window.KonvaMotion` → `window.Smoove`, `window.KonvaMotionPlayer` →
  `window.SmoovePlayer` (standalone bundle + its JSDoc / `vite.config` comments).
  **`window.Konva` stays.**

**Verify:** `pnpm build` compiles all packages (incl. player's Vite build +
standalone bundle).

**Status / notes:**

- Applied to **compiled source only** — every tracked `.ts`/`.tsx` under
  `packages/*` + `demo/` (incl. `vite.config.ts`, `react-router.config.ts`,
  `scripts/`, `examples/`, and source `*.d.ts`). **243 source files changed**;
  `dist/` + `demo/build/` are build artifacts (regenerated by the gate) and
  `.mdx`/`.md` prose is **deferred to Phase 4** — so 29 `.mdx` files still hold
  `@konva-motion` / `konvaMotion()` references on purpose.
- Replacement order (longest-token-first, so `Konva` is never touched):
  `@konva-motion/` → `@smoove/`; `KonvaMotion` → `Smoove`; `\bkonvaMotion\b` →
  `smoove`; `konva-motion` → `smoove`; then the `Km*` identifiers. Used `perl`
  (not BSD `sed`) for reliable `\b`, and escaped `@` in the perl regex.
- `KonvaMotion` → `Smoove` as a **substring** cleanly swept the non-obvious
  cases the plan didn't enumerate: the internal stage marker
  `__KonvaMotionComposition` → `__SmooveComposition` (core
  `composition.ts`/`environment.ts`), the vite-plugin type `KonvaMotionOptions`
  → `SmooveOptions`, and `window.KonvaMotionPlayer` → `window.SmoovePlayer`.
- **Vite plugin public API renamed** (not spelled out in the plan, but required —
  `konvaMotion` is the camelCase brand token and the Phase 4 gate forbids
  residual `konva-motion`): exported fn `konvaMotion()` → `smoove()`, interface
  `KonvaMotionOptions` → `SmooveOptions`, and the runtime plugin `name:
  "konva-motion"` → `"smoove"` in `packages/vite/src/index.ts`. The legacy
  `reactMotionStudio` alias is untouched (not a brand token); its RHS now points
  at `smoove`. **mdx samples + the `default`/named export shape still need a
  prose pass in Phase 4** for the `konvaMotion()` call sites in
  `vite-plugin/*.mdx`.
- **Extra `Km*` identifiers renamed beyond the plan's `KmPlayer`/`KmStudio`** to
  honor the locked "full identifier rename" decision and avoid a half-done look:
  `KmControl` → `SmooveControl` (player base class), `KmField` → `SmooveField`,
  `KmContainer` → `SmooveContainer`, `KmSchema` → `SmooveSchema` (player/studio
  internals). ⚠️ These are **not** caught by the Phase 4/9 grep gate (which only
  scans `km-`, lowercase) — they were found by hand. **Guard:** the
  `@smoove/google-fonts` font-data files (`src/fonts/*.ts`) contain coincidental
  `Km…` substrings (e.g. `KmHesPOL`) that are **not** brand tokens; replacements
  targeted exact brand names only, so those were left intact.
- **CSS class/tag prefix `km-` left for Phase 3** — the case-sensitive subs above
  never match lowercase `km-player` tags or `km-*` classes/tag maps in the
  source `*.d.ts`; only their `@konva-motion` imports changed.
- Removed two **stale, untracked** core dist artifacts from the pre-`engine/`
  layout (`dist/composition.{js,d.ts}`, `dist/environment.{js,d.ts}`, dated May)
  that still held `KonvaMotion` and aren't referenced by the rebuilt barrel
  (which points at `dist/engine/*`). `tsc -b` incremental builds don't prune
  moved-source output; a `dist` wipe would also clear them.
- ✅ `pnpm build` green (exit 0) across all packages incl. player's Vite build +
  the standalone `player.global.js` (now pins `.Smoove`, keeps `window.Konva`).
  Grep confirms **0** residual `@konva-motion` / `KonvaMotion` / `konvaMotion` /
  `konva-motion` and **0** brand `Km*` identifiers in source, with bare
  `konva` / `Konva` imports untouched.

## Phase 3 — Web-component tags, CSS class prefix & file renames ✅ DONE (2026-06-30)

- **Tags:** `<km-player>` → `<smoove-player>` and every sub-element tag
  (`km-player-controls`, `km-player-sound-control`, `km-player-progress`, …) →
  `smoove-player-*`. Includes `customElements.define(...)` registrations and the
  player registry.
- **CSS classes/parts:** `km-studio`, `km-studio-portal`, `km-doc`, `km-spin`,
  `km-default`, `km-boot`, `km-root`, `km-renderer-demo-*` → `smoove-*`. Touches
  `player.css` (41), `studio.css` (32), `demo/src/app.css` (12), docs
  `player.mdx` (50), `doc/studio-design.md` (38). *(Pure rename here; the
  values repaint happens in Group B, keeping the two diffs clean.)*
- **Ambient typings:** `packages/studio/src/km-player-element.d.ts` and
  `packages/docs/src/km-player.d.ts` (tag-name → interface maps).
- **Source file renames** (+ update their importers):
  - `packages/player/src/km-player.ts` → `smoove-player.ts`
  - `packages/studio/src/km-player-element.d.ts` → `smoove-player-element.d.ts`
  - `packages/docs/src/km-player.d.ts` → `smoove-player.d.ts`

**Verify:** `pnpm build` + `pnpm dev` smoke — the demo renders `<smoove-player>`.

**Status / notes:**

- **Method: one blanket literal `km-` → `smoove-` substring replacement** across
  **57 files** (perl `-i`). Safe because the `km-` prefix only ever appears as a
  brand token — verified there is **no** `[a-zA-Z]km-` (i.e. `km-` is never
  embedded mid-word) anywhere in scope. A substring sub also can't touch bare
  `konva` / `Konva` / `.konvajs-content` (no `km-` in them), so the guard holds
  for free. This swept tags **and** sub-element tags **and** CSS classes/parts in
  one pass, incl. `customElements.define("smoove-player-*")` and the container
  registry (`packages/player/src/containers.ts`).
- **Scope decision — did *all* `km-` repo-wide now, not just the listed files.**
  The Phase 4 grep gate forbids residual `km-`, and the `km-` tokens also live in
  the docs live-demo `<km-player>` tags (`audio.mdx`, `components.mdx`,
  `dynamic-props.mdx`, `introduction.mdx`, `player/index.mdx`,
  `player/install.mdx`, `transitions/guide.mdx`) and several `doc/*.md`. Since the
  sub only touches `km-` substrings, the `konva-motion` / `@konva-motion` prose in
  those same files is **left intact for Phase 4**. **Excluded** from the sub: this
  plan doc itself (`doc/rename-to-smoove-plan.md` — keeps `km-` as examples) and
  `doc/_design_extracted/` (design bundle, already `smoove-`).
- **3 source files renamed via `git mv`.** `packages/player/src/index.ts` already
  pointed at `./km-player.js`, so the same blanket sub rewrote its import +
  re-export to `./smoove-player.js` — the rename and the import update agree. The
  two `.d.ts` files are **ambient** (picked up by tsconfig `include` globs, no
  path importers), so renaming them inside `src/` needs no other edits.
- **Temp-dir prefix renamed too:** `mkdtempSync(…, "km-renderer-demo-")` →
  `"smoove-renderer-demo-"` in `packages/renderer/examples/render-demo.ts` (it's
  a `km-` brand token the plan lists under CSS classes/parts; harmless fs name).
- **Guard / false-positive check:** the only odd-looking `km-*` strings
  (`km-silvery`, `km-render-end`) live **exclusively in `demo/build/` artifacts**
  (minified/mangled CSS), which are regenerated and were not edited. Bare
  `konva` / `Konva` untouched — **36** `konvajs-content` / `window.Konva` /
  `from "konva"` refs preserved.
- ✅ `pnpm build` green (exit 0) across all packages incl. player's Vite build +
  standalone `player.global.js` (which now emits `smoove-player*` tags). Grep
  confirms **0** residual `km-` in source/docs (excluding the two intentional
  exclusions above).
- ✅ `pnpm dev` smoke. **Note:** `pnpm dev` runs the **studio demo (demo2)**,
  which renders via the `<Studio>` React component + a Konva stage — it does
  **not** embed the `<smoove-player>` web component (that lives in docs/player).
  Verified the studio renders cleanly: "SmooveStudio" branding, `smoove-studio`
  root class, working `smoove-studio-portal`, a composition opens to a live Konva
  stage (2 canvases + `.konvajs-content`), **zero console errors**. Separately
  confirmed the player web component is wired: `customElements.get("smoove-player")`
  is truthy, `"km-player"` is gone, and the built bundle defines the full
  `smoove-player*` tag family.
- ⚠️ Port gotcha: the demo Vite is pinned to `:5174`; with `:5174` (and the
  preview harness's `:5173`) busy it landed on `:5175`. Drive the preview at the
  port the dev log prints, not `:5173`.

## Phase 4 — Folder, docs & prose sweep ✅ DONE (2026-06-30) — folder rename deferred

- Rename the working folder `konva-motion/` → `smoove/` (local; harmless to
  tooling).
- Sweep prose for `konva-motion` references: `README.md`, `CLAUDE.md`, all
  `doc/*.md`, `packages/docs/content/docs/*.mdx`, and the
  `docs/superpowers/specs/` + `memory/` notes.
- Update the `unpkg` / `jsdelivr` URLs and `@konva-motion/player` references in
  `packages/docs/content/docs/player.mdx`. The `exports` subpaths (`./standalone`,
  `./styles.css`, …) keep their relative names.
- Update prose links/badges to the new org + domain.

**Verify:** `pnpm check` (Biome) passes; `grep -rn` finds **zero** residual
`konva-motion` / `@konva-motion` / `KonvaMotion` / `km-` tokens (excluding bare
`konva` / `Konva`).

**Status / notes:**

- **Method: one ordered blanket `perl -i` sweep** across **55 prose/code files**
  (longest-token-first so bare `konva`/`Konva` are never touched):
  `@konva-motion/` → `@smoove/`; `KonvaMotion` → `Smoove`; `konvaMotion` →
  `smoove` (the camelCase vite-plugin token the Phase 2 notes flagged for a prose
  pass — `konvaMotion()` call sites in `vite-plugin/*.mdx` now read `smoove()`);
  `konva-motion` → `smoove`; `km-` → `smoove-` (lowercase only). Scope: every
  tracked `.md` / `.mdx` / `README` + `CLAUDE.md`, the css **header comments** in
  `player.css` / `studio.css` / docs `base.css` / `home.css`, the tracked
  `.claude/launch.json` (`@smoove/docs` debug target) and the repo
  `memory/skia-canvas-leak-root-cause.md` note. `doc/TODO.md` was swept **by hand**
  (5 prose hits) so its asset-filename lines 88–89 survive for Phase 5.
- ⚠️ **Real bug fixed (not prose):** `packages/player/src/smoove-player.ts`
  compared `c.tagName === "KM-PLAYER-CONTROLS"`. DOM `tagName` is **uppercase**, so
  Phase 3's lowercase `km-` sweep never matched it — after Phase 3 the element is
  `<smoove-player-controls>` and that check was silently **always false**, breaking
  default-control reconciliation (`_reconcileControls`). Corrected to
  `"SMOOVE-PLAYER-CONTROLS"`. A repo-wide case-sensitive `\bKM-` grep confirms no
  other uppercase tag-name comparisons remain.
- **Org / domain / CDN URLs were already correct** — Phase 1 wrote
  `github.com/smoove-dev/smoove` + `https://smoove.dev` into the publishable
  `package.json`s, and the `unpkg` / `jsdelivr` / `@konva-motion/player` doc refs
  flipped to `@smoove/player` for free via the `@konva-motion/` sub. The `exports`
  subpaths (`./standalone`, `./styles.css`, …) kept their relative names as planned.
- **Guard / preserved on purpose:** the **Remotion attribution URLs**
  (`github.com/remotion-dev/…`, `raw.githubusercontent.com/remotion-dev/…`) contain
  none of the brand tokens, so they were never at risk — upstream credit intact.
  The `picsum.photos/seed/konva-motion` demo seed → `seed/smoove` (cosmetic seed
  string only). Bare `konva` / `Konva` / `window.Konva` / `from "konva"` /
  `.konvajs-content` all untouched.
- **`pnpm check` — clean for everything in this rebrand's purview.** Phases 1–3
  gated on `pnpm build` (which ignores formatting), so Biome had never run; the
  `@smoove` rename also reordered imports (k→s) and re-wrapped some lines past the
  100-col width. Cleaned that **rename-induced drift** with `biome check --write`
  (safe fixes only) across ~18 tracked source files — incl. `smoove-player.ts`,
  `core/engine/environment.ts`, the two `vite.config.ts`, `studio/schema/kf.ts`,
  and the `packages/docs/src/**` demos/components — plus one pre-existing
  `useSelfClosingElements` nit in `docs/src/routes/home.tsx`. **Biome on every
  Phase-4-touched file is green.**
- ⚠️ **`pnpm check` still reports ~13 PRE-EXISTING errors, all unrelated to the
  rebrand**, in two buckets left untouched: **9** in the **gitignored, generated**
  `packages/docs/.source/*` (Fumadocs output — Biome shouldn't lint it; fix is a
  config gap, e.g. enable `vcs.useIgnoreFile` or ignore `.source`), and **4** in
  the **vendored** `.agents/skills/remotion-best-practices/rules/assets/*.tsx`
  (third-party example code, no brand tokens). Neither is Phase 4's job — **flag
  for Phase 9** (decide: biome-ignore both vs. accept). So `pnpm check` is *not*
  globally exit-0 yet, but no first-party rebrand file contributes to that.
- **DEFERRED to Phase 5 (so the "zero residual" grep can't be literally hit in
  Phase 4 alone):** the brand **asset files** still named/captioned `konva-motion`
  — `assets/konva-motion-{icon-afterimages,icon-circle,mark-black,mark-gradient,mark-white}.svg`
  and `packages/docs/public/favicon.svg` (each has `aria-label="konva-motion"`
  inside) — plus the two asset-filename reference lines in `doc/TODO.md:88–89`.
  Phase 5 replaces these assets and their references, so they were intentionally
  not swept here.
- **Binary media** (`demo/**` + `packages/renderer/examples/**` `.mp3`/`.mp4`)
  show up in a raw `grep` as "Binary file … matches" — coincidental byte
  sequences inside encoded audio/video, **not** brand tokens. Not editable, not
  touched.
- `.claude/settings.local.json` (gitignored, local) still holds one
  `@konva-motion/core` *permission* entry; the auto-mode classifier blocked editing
  permission rules and it's out of repo scope anyway. Harmless — it just won't
  auto-match the renamed `@smoove/core` build command. The `km-design` temp-dir
  paths in that file are not brand tokens.
- ⏳ **NOT DONE — working-folder rename `konva-motion/` → `smoove/`.** Can't be done
  from inside this session: the live working directory is
  `/Users/rotem/development/konva-motion`, and renaming it mid-session breaks the
  cwd, every absolute tool path, and the running dev server. **Do it out-of-session**
  (`mv konva-motion smoove` from the parent dir, or via the editor) when nothing
  holds the path open, then reopen. Pure-fs move — no in-repo references depend on
  the folder name.
- ✅ `pnpm build` re-run green (exit 0) after the source edits; residual
  brand-token grep clean except the deferred Phase 5 assets + binary-media false
  positives above.

---

# Group B — Visual identity

Per-package styling, landing on the same branch. Consumes the
[design assets](#design-assets-source-of-truth) above; values from
[§ Brand reference](#brand-reference).

## Phase 5 — Brand assets (logo marks + favicon) ✅ DONE (2026-06-30)

- Replace the old `konva-motion-{mark-white,mark-black,mark-gradient,…}.svg`
  assets with the new **edge-dot sunshine mark** — four timeline bars tapering to
  a play triangle, with the sunshine keyframe dot just past the last bar:
  - `smoove-mark.svg` (primary, gradient) ← `project/smoove-mark.svg`
  - `smoove-mark-dark.svg` (ink on paper) ← `project/smoove-mark-dark.svg`
  - `smoove-mark-light.svg` (paper on ink) ← `project/smoove-mark-light.svg`
- Regenerate `packages/docs/public/favicon.svg` from the mark.
- Update the brand component, `README.md`, and docs references to the new files.
- **Deferred:** `project/smoove-mark-animated.svg` / the animated lockup.

**Verify:** marks render crisp at favicon size through hero size; gradient,
dark, and light variants each keep the sunshine dot.

**Status / notes:**

- **Asset files** in `assets/`: `git rm`'d all **5** old `konva-motion-*.svg`
  (the 3 marks **plus** `icon-afterimages` / `icon-circle` — the new identity is
  just the three marks; the old play-triangle-trail explorations are dropped).
  Copied the 3 new marks verbatim from the design bundle
  (`doc/_design_extracted/.../project/smoove-mark{,-dark,-light}.svg`) →
  `assets/smoove-mark.svg` (gradient), `smoove-mark-dark.svg` (ink on paper),
  `smoove-mark-light.svg` (paper on ink). Animated mark **deferred** as planned.
- **The mark art was live in 3 code spots**, all still the **old afterimage
  trail** — each swapped to the edge-dot geometry:
  1. `packages/docs/public/favicon.svg` — full rewrite (was a teal-gradient
     afterimage on a circle, `aria-label="konva-motion"`).
  2. docs `BrandMark` in `packages/docs/src/components/icons.tsx`.
  3. studio `spark` glyph in `packages/studio/src/components/icon/paths.tsx`
     (the studio sidebar `Logo`). *Not named in the plan's bullet (which says
     "docs"), but the studio `Logo` consumes the same mark — left as old
     afterimages it'd be a half-done logo. Swapped + documented here; the
     **studio palette/surfaces repaint stays Phase 8.***
- **Favicon ≠ a bare copy of the mark.** Browser tabs are light *or* dark, so a
  transparent thin-bar mark would wash out. The favicon is the gradient mark on
  a **grape-ink (`#261733`) rounded chip** (an approved "ink" background from the
  Lockup spec — the gradient holds on dark), with bars bumped to stroke-width 10
  and the sunshine dot to r=4.5 for small-size legibility. Verified it still
  reads as a coherent mark at 16px (TODO item #4 — "logo legibility at small
  sizes" — is now largely addressed; noted in `doc/TODO.md`).
- **Phase-5 vs Phase-7/8 boundary held:** `BrandMark` and studio `spark` render
  their **bars in `currentColor`** (so each host's existing chip color still
  drives them) **+ the sunshine dot in its fixed `#FFC23C`**. Only the *mark
  geometry* changed here; the surrounding chrome (the docs `.brand__mark`
  emerald→cyan box, the studio `Logo` accent-gradient box) is **left for the
  Phase 7/8 repaint**. Interim look: edge-dot bars in white inside the old
  gradient chip — transitional, on purpose.
- **Wordmark text:** docs `brand.tsx` still rendered `konva<span>-motion</span>`
  — a residual brand token the Phase 4 grep **missed** because it's split across
  two JSX text nodes (`konva` + `-motion`), so the literal `konva-motion`
  substring never appeared on the line. Changed to `Smoove` (matching the brand
  wordmark casing); dropped the now-unused `.dim` two-tone span (Comfortaa
  wordmark styling is Phase 6/7).
- **Prose references updated:** `doc/TODO.md:87-90` (old asset filenames →
  `smoove-mark{,-dark,-light}.svg` + new consumer paths) and
  `doc/smoove-brand-brief.md` "Visual direction" — its mark section still
  prescribed the *afterimage trail* as "the signature" (now **superseded** by
  the chosen edge-dot mark); rewrote that bullet to describe the shipped mark and
  fixed the `icon-afterimages`/`mark-black`/`mark-white` filenames. `README.md`
  carries **no** mark/asset references — nothing to change there.
- **Guard:** bare `konva` / `Konva` untouched; no shared token module introduced
  (per-package values inlined per the brand reference).
- ✅ **Verified:** `pnpm --filter @smoove/studio build` + `pnpm --filter
  @smoove/docs build` both green. Browser smoke (`pnpm dev`, demo2/SmooveStudio):
  the sidebar `Logo` now renders the edge-dot mark (4 `<line>`s, `scale(0.15)`,
  dot `#FFC23C`), **zero console errors**. Visual proof rendered all four SVGs
  (favicon @64 + @16, gradient/dark/light marks): crisp, each keeps the sunshine
  dot. ⚠️ Port gotcha persists — demo Vite is pinned to `:5174` but the preview
  harness drives `:5173`; temporarily set the port to `:5173` to verify in-browser
  and **restored it to `:5174`** afterward.
- ⏳ Still pending from Group A: the working-folder rename `konva-motion/` →
  `smoove/` (Phase 4, must be done out-of-session) — unrelated to Phase 5.

## Phase 6 — Brand fonts ✅ DONE (2026-06-30)

- Wire the three faces in where text renders: **Comfortaa** (display/wordmark),
  **Hanken Grotesk** (body/UI), **JetBrains Mono** (code) — docs first, then
  player/studio chrome.
- Load via Google Fonts (`Comfortaa:wght@400..700`,
  `Hanken+Grotesk:wght@400..700`, `JetBrains+Mono:wght@400..600`).
- Check whether `@smoove/google-fonts` needs these registered for server-side
  rendering of compositions that use them.

**Verify:** docs + demo show Comfortaa headings, Hanken body, mono code blocks;
no FOUT regressions; SSR render (if applicable) embeds the faces.

**Status / notes:**

- **Starting state (key finding):** Hanken Grotesk + JetBrains Mono were
  **already** wired — both `<link>`'d in the docs (`packages/docs/src/root.tsx`)
  and demo (`demo/src/root.tsx`) Google-Fonts URLs, applied as the docs body
  (`base.css`) / studio `--font-sans` and the `…Mono` code stack. The only
  missing face was **Comfortaa** (display/wordmark), plus the player chrome was
  on bare `system-ui` / `ui-monospace`. So Phase 6 = add Comfortaa + apply the
  display face + wire the player chrome stacks. No palette/surface changes (that
  stays Phase 7/8) — the docs are still on the old violet KmStudio look here.
- **Google Fonts URL** (`Comfortaa:wght@400;500;600;700` prepended,
  `display=swap` kept) added to **both** `packages/docs/src/root.tsx` and
  `demo/src/root.tsx`. Used the explicit semicolon weight list to match the
  existing Hanken/JetBrains entries (variable-font `wght@400..700` range syntax
  also works; chose consistency).
- **Docs** (`packages/docs/src/styles/base.css`): declared one theme-independent
  token `--font-display: "Comfortaa", system-ui, sans-serif;` in `:root` (outside
  the light/dark blocks), applied it via a global `h1–h6 { font-family:
  var(--font-display) }` rule (covers the homepage hero `h1` **and** all Fumadocs
  MDX content headings — an explicit element rule beats the inherited Hanken from
  `body`), and set `.brand__word` (the header wordmark, renders "Smoove") to the
  display face.
- **Studio** (`packages/studio/src/studio.css` + `components/brand/brand.tsx`):
  added `--font-display` to the Tailwind v4 `@theme` block (which auto-generates
  a `font-display` utility — confirmed `.font-display{font-family:var(--font-display)}`
  in compiled `dist/styles/studio.css`), then put the `font-display` class on the
  "SmooveStudio" wordmark `<div>`. Body/UI stays `--font-sans` (Hanken), data
  stays `--font-mono` (JetBrains).
- **Player** (`packages/player/src/player.css`): wired the brand stacks into the
  chrome — control bar `system-ui` → `"Hanken Grotesk", system-ui, sans-serif`;
  time readout `ui-monospace` → `"JetBrains Mono", ui-monospace, monospace`. The
  player **does not load fonts itself** (it's headless/light-DOM by design); it
  inherits the faces from the host page (docs already `<link>`s all three), and
  falls back cleanly where the host hasn't loaded them. Color/surface repaint of
  the player is still **Phase 8**.
- **SSR / `@smoove/google-fonts`: no changes needed.** The package is a
  module-per-family registry (no allowlist) and **already** ships modules for all
  three faces — `@smoove/google-fonts/{comfortaa,hanken-grotesk,jetbrains-mono}`.
  A composition that wants one of these *inside the canvas* just imports the
  module and `seq.add(font)` (it's buffered + skia-loaded automatically); the
  three faces above are **chrome/HTML** fonts (loaded by the host `<link>`),
  which is a separate concern from canvas-text SSR. So nothing to register.
- **Guard held:** bare `konva` / `Konva` untouched; no shared token module
  introduced (per-package values inlined — `--font-display` declared separately
  in docs `base.css` and studio `@theme`).
- ✅ **Verified:** `pnpm build` green (exit 0) across all packages incl. player's
  Vite build + the studio Tailwind CSS step; compiled output confirms the wiring
  (Comfortaa in studio `--font-display` + the `.font-display` utility; Hanken +
  JetBrains stacks in `player.css`). Browser smoke (`pnpm dev`, demo2): computed
  styles resolve **wordmark → `Comfortaa, system-ui, sans-serif`**, **studio body
  → Hanken Grotesk**, **code → JetBrains Mono**; headings → Comfortaa. **Zero
  console errors.** ⚠️ The sandbox has **no network to `fonts.gstatic.com`**, so
  the actual woff2s don't download (`document.fonts.check('16px Comfortaa')` is
  `false`) and the rounded glyphs fall back to system-ui in the screenshot — the
  *wiring* is correct; the glyph render needs a networked environment. `display=swap`
  means the fallback is FOUT-free either way.
- ⚠️ Port gotcha persists (demo Vite pinned `:5174`, preview harness drives
  `:5173`): temporarily set the port to `:5173` to verify in-browser and
  **restored it to `:5174`** afterward.
- ⏳ Still pending from Group A: the working-folder rename `konva-motion/` →
  `smoove/` (Phase 4, must be done out-of-session) — unrelated to Phase 6.

## Phase 7 — Docs site repaint ✅ DONE (2026-06-30)

Convert the existing KmStudio look to the Smoove system (`@smoove/docs`):

- Apply the palette + the Smoove gradient as a single focal accent per view;
  honor the ~90/10 calm-vs-bright ratio.
- New lockup in the header (mark + Comfortaa wordmark), gradient hero treatment.
- Rounded surfaces, warm paper / cream backgrounds, crisp 1px borders.
- Mono for all code/data; gradient on display type only.

**Verify:** docs build + visual smoke against `Smoove Brand Brief.dc.html` and
`Smoove Lockup.dc.html` (palette, type, lockup, gradient usage).

**Status / notes:**

- **Decision (locked this session): the docs are now cream-first.** Default theme
  is the warm cream canvas (`#FFF6EC`) with grape-ink text (`#261733`); dark mode
  is **rebuilt on grape ink**. The toggle stays. Previously the docs were
  dark-default (KmStudio violet `#7c5cff`).
- **The docs CSS is token-driven**, so the core of the repaint was retokenizing.
  Two distinct theming layers, both repainted:
  1. **Home-page chrome** (`packages/docs/src/styles/base.css` +
     `home.css`) — keyed off `--accent`, `--bg-0..3`, `--ink-1..3`, `--line`,
     `--code-*`, `--tok-*`. **Swapped the two token blocks**: `:root` is now the
     cream/light theme (`color-scheme: light`, `--bg-0: #FFF6EC`,
     `--ink-1: #261733`, `--accent: #FF5640` coral / `--accent-2: #15CDA8` mint,
     `--radius-ui: 16px`, warm `--tok-*`); the old `.light` block became `.dark`
     with grape-ink surfaces (`--bg-0: #261733`, panel `#34204A`, border
     `#43305A`, accents lifted for contrast). Also flipped the `.light`-keyed
     rules to `.dark` (`::selection`, the sun/moon toggle swap), and the header
     comment now describes the Smoove system.
  2. **Fumadocs content pages** (sidebar/TOC/prose/callouts) are themed by
     `--color-fd-*` (from `fumadocs-ui/css/neutral.css` + `preset.css`), **not**
     base.css — so they're overridden in `packages/docs/src/app.css`: `:root`
     (light) → cream/grape/coral, `.dark` → grape ink, plus a `.dark #nd-sidebar`
     override (neutral.css ships a grayscale sidebar override that would otherwise
     read cold).
- **Header lockup → the colored (gradient) mark, no chip.** First pass put the
  mark in a gradient *chip* (white bars on a coral→mint tile); revised to show the
  **official colored mark directly** per the Lockup spec ("gradient mark +
  wordmark"). `BrandMark` (`components/icons.tsx`) gained a `gradient` prop that
  injects an inline `<linearGradient>` (coral `#FF5640` → mint `#15CDA8`,
  `gradientUnits="userSpaceOnUse"`, matches `smoove-mark.svg`) and strokes the
  bars with it; the sunshine dot stays `#FFC23C`. Default stays `currentColor`
  (e.g. `demo.tsx` keeps it). Applied `gradient` in all three lockups: home
  header (`brand.tsx`), footer (`home.tsx`), docs navbar (`Logo` in
  `lib/layout.shared.tsx`).
- ⚠️ **Invisible-logo bug fixed (gradient id collision).** First cut used a
  **hardcoded** `id="smoove-mark-grad"`; the lockup renders several marks per page
  (home header + footer), so the page had **duplicate `id`s** (invalid) and a
  `url(#smoove-mark-grad)` paint reference that can resolve to a duplicate/
  unmounted def — an **invisible stroke** (surfaced in the docs). Fix: the
  gradient id is now **per-instance via React `useId()`**, so every mark
  references its own def. Verified: home emits unique ids (no duplicates) and the
  docs-navbar mark's `url(#…)` resolves to a present def; marks render light +
  dark.
- **Lockup sizing verified against the design size ladder** (`Smoove
  Lockup.dc.html`): mark box ≈ **1.6× the wordmark font-size**, gap ≈ 0.4× (XL
  54px word→84px mark/18 gap; M 21→34/9; S 14→22/7). So home (16px wordmark) →
  **26px** mark box with the SVG filling it (was a 15px glyph inside a 26px chip),
  gap **8px**; docs navbar (15px wordmark) → **24px** mark (`size-6`). Removed the
  chip background/shadow from `.brand__mark`.
- **Third split-token straggler fixed:** the **footer** wordmark in `home.tsx`
  also rendered `konva<span class="dim">-motion</span>` (same split-node bug as the
  header/navbar) → `Smoove`.
- **Hero gradient moment:** `.hero h1 .grad` now uses the Smoove gradient
  (`linear-gradient(102deg,#FF5640,#15CDA8)`), display-size only — the one focal
  gradient on the page. Toned the `.hero__glow` down (it was tuned for a dark
  stage) so it's a soft warm halo on cream, not a wash.
- **Fonts:** Phase 6 wired the brand faces into base.css, but base.css **only
  loads on the home route** — docs content pages had no brand font wiring. Added
  a minimal typography block to `app.css` (global): `--font-display: Comfortaa`,
  `body` → Hanken Grotesk, `h1–h6` → Comfortaa, `code/pre/kbd/samp` → JetBrains
  Mono. Verified the docs `<h1>` computes to Comfortaa and body to Hanken.
- **Hero focal animation recolored:** `src/demos/hero-bg.ts` (the live
  `<smoove-player>` aurora behind the headline) was emerald/cyan/violet/pink — now
  a **coral↔mint field with sunshine sparks** (the Smoove gradient as
  atmosphere). The hero's displayed code sample (`home.tsx`) orb fill →
  `#FF5640`. **The in-content feature demos** (`block-card.ts`,
  `shapes-gallery.ts`, `block-borders.ts`) keep their intentionally varied
  rainbow palettes — they *demonstrate* arbitrary colors, so recoloring them
  would defeat their purpose. (One incidental old-accent `#7c5cff` remains in
  those demos; left on purpose.)
- **Straggler brand fixes found while repainting (in-scope for the lockup):**
  - `lib/layout.shared.tsx` docs-navbar `Logo` still rendered
    `konva<span>-motion</span>` — a **split-token** residue (`konva` + `-motion`
    across two JSX nodes) the earlier greps missed (same bug class fixed in
    `brand.tsx` in Phase 5). → `Smoove`.
  - **GitHub org typo in three spots:** `github.com/smoove/smoove` →
    `github.com/smoove-dev/smoove` in `lib/layout.shared.tsx`,
    `components/home-header.tsx`, `routes/home.tsx`.
- **Theme mechanism:** next-themes (via Fumadocs `RootProvider`,
  `attribute="class"`) adds `.light`/`.dark` on `<html>`. The new structure
  (`:root` = light, `.dark` = dark) matches Fumadocs' own convention, so SSR /
  first paint (no class) renders cream. Set `RootProvider theme={{ defaultTheme:
  "light" }}` in `root.tsx` so cream is the canonical default.
- **Guard held:** bare `konva` / `Konva` / `window.Konva` / `from "konva"`
  untouched; no shared token module (per-package values inlined). The
  `#FF5640`/`#15CDA8`/`#FFC23C` hexes are inlined per the brand reference.
- ✅ **Verified:** `pnpm --filter @smoove/docs build` green; `biome check` clean
  on all touched files. Browser smoke (docs dev on **:5176** — the
  `.claude/launch.json` `docs` target; note the port differs from the demo's
  :5174 / preview's :5173): **first paint = cream** (`htmlClass "light"`,
  `body` bg `#FFF6EC`, text `#261733`); home hero shows the single
  Smoove-gradient focal moment + coral/mint aurora; both lockups render the
  edge-dot mark on a gradient chip + **"Smoove"** (no "konva-motion"); toggle →
  grape-ink dark (`#261733`); `/docs/introduction` content page on the cream
  Fumadocs theme (`--color-fd-background #fff6ec`, `--color-fd-primary #ff5640`,
  `<h1>` Comfortaa, body Hanken) and grape ink in dark. **Console clean** apart
  from the **pre-existing** "Several Konva instances detected" warning (the
  standalone player bundles Konva while demos import it too — documented in
  earlier phases, not a regression).
- ⚠️ **For Phase 8/9:** the embedded `<smoove-player>` chrome (control bar,
  letterbox) is still **dark-tuned** (`player.css`) — it reads as a dark video
  frame on the cream page. That's Phase 8's job (player/studio/demo repaint), as
  scoped. The docs-page demo *content* (e.g. the introduction's teal/purple
  circles) is illustrative, not brand chrome.
- ⏳ Still pending from Group A: the working-folder rename `konva-motion/` →
  `smoove/` (Phase 4, must be done out-of-session) — unrelated to Phase 7.

## Phase 8 — Player, Studio & demo repaint ✅ DONE (2026-06-30)

- **`player.css`:** brand colors; gradient on the render/play focal accent;
  rounded control surfaces; restrained shadows.
- **`studio.css`:** match the product-look mock — warm paper frame → white
  working surface → **one** gradient focal moment (the render button + animated
  object); mono everywhere data lives; Comfortaa wordmark, Hanken UI. Reference:
  the "In practice" studio card in `Smoove Brand Brief.dc.html`.
- **`demo/src/app.css`:** inherits the player styling; align surrounding chrome
  to the palette.

**Verify:** `pnpm dev` — player controls, studio shell, and demo match the mock;
one gradient focal moment per surface.

**Status / notes:**

- **Locked decision (with the user): the studio KEEPS its dark pro-editor shell**
  — accent-only recolor, **not** the cream "warm paper" flip the mock literally
  shows. Rationale: the studio's surfaces are token-driven but a full cream flip
  would touch surface usage across ~168 components (high risk, much iteration),
  the dark canvas already matches the mock's **dark preview surface**, and a dark
  editor is a legit pro-tool pattern. So Phase 8 = swap the legacy accents to the
  Smoove palette + make each surface's one gradient focal moment coral→mint;
  surfaces stay dark-neutral. *(A future pass could still cream-flip the studio
  shell if desired — out of scope here.)*
- **Studio `studio.css:41-42` is the cascade source.** Swapped
  `--color-accent: #7c5cff → #ff5640` (coral) and `--color-accent-2: #9579ff →
  #15cda8` (mint). The `@theme` block is the single source of truth, so this
  cascades to **all 42** `bg-accent` / `text-accent` / `accent-soft` /
  `accent-line` / `from-accent`/`to-accent` usages across components, and the
  derived `--color-accent-soft` / `--color-accent-line` tints regenerate
  automatically (`studio.css:62-65`). Verified in the compiled
  `dist/styles/studio.css`: `--color-accent:#ff5640`, `--color-accent-2:#15cda8`,
  `--color-accent-soft:#ff564029` (coral @16%) + the color-mix `@supports`
  variant. Surfaces/scrollbars left dark-neutral (the optional grape-warming of
  `--color-bg-*` was **not** applied — deferred; accent swap was the required
  change). `--color-good`/`--warn`/`--danger` untouched (semantic, not brand).
- **Gradient focal moment came for free.** The `Button` primitive's `tone="primary"`
  variant is **already** `bg-gradient-to-b from-accent-2 to-accent`
  (`components/button/button.tsx:15`), so the Render button (`render-dialog.tsx:206`,
  `tone="primary"`) and every primary CTA became the mint→coral Smoove gradient
  on the token swap — no separate edit. The one visible primary CTA + the brand
  mark are the gradient moments; all other accent UI stays solid coral.
- **Sidebar `Logo` de-chipped → colored mark (`components/brand/brand.tsx`).**
  The `Logo` was a 22px gradient **square chip** with a white edge-dot glyph
  inside. Per the user, dropped the chip (bg/shadow/rounded box) and now render
  the **colored mark directly**: a self-contained `BrandMark` SVG strokes the
  four bars with an inline coral→mint `<linearGradient>` (matching
  `smoove-mark.svg`) and keeps the sunshine dot `#FFC23C`, sized 24px to match
  the 15px wordmark (the lockup's ~1.6× ratio). Mirrors the docs `BrandMark`
  (Phase 7) incl. the **per-instance `useId()` gradient id** so multiple marks
  can't collide into an invisible `url(#id)` stroke. The exported `Logo`/`Brand`
  API is unchanged (the now-unused `icon` prop is still accepted for compat); the
  old generic `spark` glyph in `icon/paths.tsx` is left in place (no longer used
  by the Logo). The favicon keeps its ink chip on purpose (small-size legibility).
- **Hardcoded violet spots the cascade missed — fixed by hand:**
  - `components/render/export-frame-dialog.tsx:87` — preview tile gradient
    `…,#7c5cff66` → `…,rgba(255,86,64,.4)` (coral).
  - `components/render/render-queue.tsx:86` — active-row (non-still) icon-chip
    gradient `#2a2550` (violet) → `#3a1f1a` (warm coral); the `isStill` branch was
    already warm `#4d2f1a` and its `color: var(--color-accent-2)` now reads mint.
  - `lib/constants.ts:11` — timeline `sequence` layer-kind color `#7c5cff` →
    `#ff5640` (coral). Other layer kinds (audio/video/group/transition) keep their
    distinct semantic colors.
  - `components/schema-form/field.tsx:96` — default color-swatch palette
    `#7c5cff` → `#ff5640`; other swatches left.
- **Player `player.css`:** kept the black video letterbox (correct for video).
  `.smoove-player__fill` (`:210`) `#5b8cff` → `linear-gradient(90deg,#ff5640,
  #15cda8)` — the scrubber is the player's natural one gradient moment, growing
  coral→mint as it plays. Loop-active (`:122`) `#5b8cff` → `#15cda8` (mint =
  "looping/live" state, distinct from the coral-start fill). Control-bar scrim,
  white icons/volume thumb, and the JetBrains-Mono time readout (already Phase-6)
  read fine over the dark stage and were left. This resolves Phase 7's flagged
  "embedded player still dark-tuned on the cream docs page" — its *accent* is now
  on-brand (the chrome stays dark, which is correct for a video frame).
- **Demo `demo/src/app.css:65`:** `.smoove-doc code` (inline code) `#9579ff` →
  `#ff5640`, matching the docs repaint. The boot/doc background gradient
  (`app.css:28`) is neutral-dark and was left as-is.
- **Deliberately NOT touched — demo *scene content* keeps its `#7c5cff`.** The
  `demo/src/compositions/**` files (Pulse, ribbon, shapes-playground, accents.ts)
  still use `#7c5cff` as default Konva fills / accent props. These are **example
  animation content** (what a user builds in the canvas), not brand chrome —
  recoloring them defeats their purpose, exactly the Phase-7 precedent for the
  in-content feature demos. The chrome stylesheet `app.css` is clean. *(If the
  old brand violet as the default scene accent ever feels off, it's a content
  refresh, separate from this rebrand.)*
- **No dark-mode lift needed** — true brand coral `#FF5640` / mint `#15CDA8` read
  fine on the dark neutral shell, so the Phase-7 lifted values
  (`#FF6B57`/`#2EE0BD`) were not used.
- **Guard held:** bare `konva` / `Konva` / `window.Konva` / `from "konva"`
  untouched (29 refs in player/demo preserved). No shared token module — brand
  hexes inlined per-package.
- ✅ **Build green** (`pnpm build`, exit 0) across all packages incl. player's
  Vite build + standalone bundle and the studio Tailwind CSS step. **Biome clean**
  on all touched files. Guard grep: **0** residual `#7c5cff`/`#9579ff`/`#5b8cff`
  in `packages/studio/src` + `packages/player/src` + `demo/src/app.css` (the only
  remaining hits are the intentional demo *composition content* above).
- ⚠️ **Live browser smoke was blocked by a concurrent session.** Both relevant
  dev-server ports were held by **another chat's running servers** — demo2 on
  `:5174` (the pinned demo Vite port) and docs on `:5176` — which this session's
  preview tools can't reach. Starting duplicates on alternate ports would require
  editing the pinned Vite/launch ports and competing with that live session, so
  it was skipped. For a pure color-token swap the build + **compiled-CSS**
  verification above is conclusive (the literal `#ff5640`/`#15cda8` values and the
  player gradient are present in the shipped `dist`). **For Phase 9:** re-confirm
  visually in-browser once the ports are free — sidebar `Logo` mint→coral, accent
  UI coral (not violet), single gradient primary CTA, coral timeline sequence
  blocks, player scrubber coral→mint + mint loop button.
- ⏳ Still pending from Group A: the out-of-session working-folder rename
  `konva-motion/` → `smoove/` (Phase 4) — unrelated to Phase 8.
- ⏳ Carried to Phase 9: the pre-existing, non-rebrand `pnpm check` errors Phase 4
  flagged (gitignored `packages/docs/.source/*` Fumadocs output + vendored
  `.agents/skills/.../assets/*.tsx`).

## Phase 9 — Integration gate (the "one big rebrand") ✅ DONE (2026-06-30)

1. `pnpm install` — clean relink under `@smoove/*`.
2. `pnpm build` — every package green (incl. player Vite + standalone bundle).
3. `pnpm check` — Biome lint/format.
4. `pnpm dev` — full smoke: `<smoove-player>` renders, new palette + fonts +
   mark present across docs / player / studio / demo.
5. Residual-token grep clean (`konva-motion` / `@konva-motion` / `KonvaMotion` /
   `km-`, excluding bare `konva` / `Konva`).
6. External links resolve: `github.com/smoove-dev/smoove`, `https://smoove.dev`,
   unpkg/jsdelivr standalone URLs.

**Status / notes:**

1. ✅ **`pnpm install`** — clean relink: all 10 workspace projects, lockfile up to
   date, **0** residual `@konva-motion` entries, **20** `@smoove/` entries.
2. ✅ **`pnpm build`** — green (exit 0) across every package incl. player's Vite
   build + standalone bundle and the studio Tailwind CSS step.
3. ✅ **`pnpm check` — now globally green (exit 0).** This phase **closed the
   pre-existing Biome gap** Phases 4 & 8 flagged: the 38 remaining errors were
   **all** in generated/vendored files Biome shouldn't lint (`packages/docs/.source/*`
   Fumadocs output, the committed `doc/_design_extracted/**` design bundle, and the
   vendored `.agents/skills/**` Remotion examples — the last seen twice because
   `.claude/skills/*` symlinks into `.agents`). **Root cause:** Biome 1.9.4's
   `vcs.useIgnoreFile` only honors the **root** `.gitignore`, but `.source` is
   ignored by a **nested** `packages/docs/.gitignore`, so Biome linted it anyway.
   **Fix:** added `**/.source/**`, `**/.agents/**`, `**/.claude/**`, and
   `doc/_design_extracted/**` to `biome.json` `files.ignore` (alongside the existing
   `dist`/`build`/`node_modules` patterns). No first-party rebrand file needed a
   code change — purely a lint-scope correction. `pnpm check` now: "Checked 414
   files. No fixes applied." (exit 0).
4. ✅ **`pnpm dev` browser smoke** (demo2 / SmooveStudio on `:5174` — the pinned
   port; the launch.json `demo` config still declares `:5173`, so drive `:5174`
   per the standing port gotcha). Verified live: title **"SmooveStudio"**,
   `.smoove-studio` root + `smoove-studio-portal`, **0** `km-` classes, no
   "konva-motion" text; computed tokens `--color-accent #ff5640` (coral) /
   `--color-accent-2 #15cda8` (mint), wordmark **Comfortaa**, body **Hanken
   Grotesk**; the sidebar `Logo` renders the gradient edge-dot mark (gradient stop
   `#FF5640`); opening "Circle + fade" navigates to `/c/basic` and mounts the live
   Konva stage (**2** canvases per-layer + `.konvajs-content`). Screenshot shows
   the coral selected-row accent, mint active-state icons, and coral/mint timeline
   scrubber. **Zero console errors.** (The standalone `<smoove-player>` web
   component itself was verified registered in Phase 3; `pnpm dev` runs the studio
   demo, not the player.)
5. ✅ **Residual-token grep clean** — `0` first-party hits for `konva-motion` /
   `@konva-motion` / `KonvaMotion` / `konvaMotion` / brand `km-` / brand `Km*`
   across `packages` / `demo` / `doc` / `README.md` / `CLAUDE.md` (excluding this
   plan, `pnpm-lock`, `dist`/`build`/`.source`, and the design bundle). Bare
   `konva` / `Konva` guard intact — **37** source files still reference
   `from "konva"` / `window.Konva` / `Konva.Stage` / `.konvajs-content`.
6. ✅ **External link references correct** — **17** `github.com/smoove-dev/smoove`
   refs (0 to the old `smoove/smoove` typo), `https://smoove.dev`, and the unpkg
   standalone `unpkg.com/@smoove/player/dist/player.global.js`. (URLs in-repo are
   correct; live DNS resolution wasn't network-tested from the sandbox.)

⏳ **Only remaining item — out-of-session working-folder rename** `konva-motion/`
→ `smoove/` (deferred since Phase 4): can't be done from inside a session rooted at
`/Users/rotem/development/konva-motion`. Run `mv konva-motion smoove` from the
parent dir when nothing holds the path open, then reopen — pure fs move, no in-repo
references depend on the folder name. **The rebrand is otherwise complete and the
gate is green.**

---

## Suggested execution order

Group A in order (1 → 4), then Group B (5 → 8), then the Phase 9 gate:

`package.json` names → `pnpm install` → imports/deps → identifiers & globals →
tags/CSS/file renames → folder + docs/prose → **then** brand assets → fonts →
docs repaint → player/studio/demo repaint → integration gate.
