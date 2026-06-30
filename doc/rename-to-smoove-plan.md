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

## Phase 7 — Docs site repaint

Convert the existing KmStudio look to the Smoove system (`@smoove/docs`):

- Apply the palette + the Smoove gradient as a single focal accent per view;
  honor the ~90/10 calm-vs-bright ratio.
- New lockup in the header (mark + Comfortaa wordmark), gradient hero treatment.
- Rounded surfaces, warm paper / cream backgrounds, crisp 1px borders.
- Mono for all code/data; gradient on display type only.

**Verify:** docs build + visual smoke against `Smoove Brand Brief.dc.html` and
`Smoove Lockup.dc.html` (palette, type, lockup, gradient usage).

## Phase 8 — Player, Studio & demo repaint

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

## Phase 9 — Integration gate (the "one big rebrand")

1. `pnpm install` — clean relink under `@smoove/*`.
2. `pnpm build` — every package green (incl. player Vite + standalone bundle).
3. `pnpm check` — Biome lint/format.
4. `pnpm dev` — full smoke: `<smoove-player>` renders, new palette + fonts +
   mark present across docs / player / studio / demo.
5. Residual-token grep clean (`konva-motion` / `@konva-motion` / `KonvaMotion` /
   `km-`, excluding bare `konva` / `Konva`).
6. External links resolve: `github.com/smoove-dev/smoove`, `https://smoove.dev`,
   unpkg/jsdelivr standalone URLs.

---

## Suggested execution order

Group A in order (1 → 4), then Group B (5 → 8), then the Phase 9 gate:

`package.json` names → `pnpm install` → imports/deps → identifiers & globals →
tags/CSS/file renames → folder + docs/prose → **then** brand assets → fonts →
docs repaint → player/studio/demo repaint → integration gate.
