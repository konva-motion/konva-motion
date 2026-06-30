# `@smoove/google-fonts` — design plan

Status: **proposed** (not yet implemented). A new package that lets users pull any
Google font as a typed, tree-shakeable `Font` subclass:

```ts
import NotoSans from "@smoove/google-fonts/noto-sans";

// Selected faces only; omit weights/styles to register all of them.
const font = new NotoSans({ weights: ["400", "600"], styles: ["normal", "italic"] });

seq.add(font);
seq.add(new Text({ font, text: "Hello" }));            // preferred face
seq.add(new Text({ font: font.face("600"), text: "Hi" }));
```

Each font is its own subpath module (`@smoove/google-fonts/<slug>`); there
is **no barrel**, so bundlers include only the families a project imports.

## Decisions (confirmed)

| Question | Decision |
| --- | --- |
| Font delivery | **Remote CDN URLs.** Faces' `src` are `fonts.gstatic.com` woff2 URLs. The browser fetches them (`FontFace`); the server downloads + disk-caches them via the loader added in [[font-component]] / `doc/font-component-plan.md`. Package ships only metadata + classes (tiny). |
| Catalog scope | **All families** (~1,700) — one module each. |
| Generated files | **Committed** to the repo (reviewable diffs; no generate step needed to typecheck). |
| Data source | **Google Webfonts Developer API** (`webfonts/v1/webfonts?capability=WOFF2`) — one woff2 URL per weight/style variant. Needs a free API key (maintainer/CI env var only). |
| Packaging | **TS source only — no build.** No `tsc -b`, no `dist`. `exports` point straight at `src/*.ts`; consumers (Vite/bundlers) transpile. The only generated artifact is the `.ts` font modules, written by the script. |

**Why CDN, not the CSS API:** verified that `fonts.gstatic.com` woff2 URLs return
`200` with `access-control-allow-origin: *` (CORS-safe for canvas `FontFace`). The
CSS2 API returns ~8 unicode-range-subset files for a *single* weight, which
doesn't fit core's one-`src`-per-face model. The Developer API's `capability=WOFF2`
`files` map returns **one complete woff2 per variant** — exactly one `src` per
`(weight, style)`.

## Package layout

```
packages/google-fonts/
  package.json            # wildcard subpath exports → src/*.ts (no build)
  tsconfig.json           # for editor/typecheck only (noEmit); NOT composite
  src/
    runtime.ts            # GoogleFont base class + selection logic + option types  (hand-written)
    manifest.ts           # { slug, family, weights, styles }[]  — for docs/tests, NOT a barrel  (generated)
    fonts/
      noto-sans.ts        # one module per family  (generated, committed)
      roboto.ts
      dm-sans.ts
      … ~1,700 files
  scripts/
    generate.ts           # fetches the catalog, writes src/fonts/*.ts + manifest.ts
  README.md
```

`konva` + `@smoove/core` are **peer deps** (consumers already pin them); the
package itself has no runtime deps.

## Exports & tree-shaking (no build)

The package is **TS source only** — no `tsc -b`, no `dist`. A single wildcard
entry maps every subpath straight to its `.ts` module (both the `types` and the
runtime condition resolve to the same source file). No per-font entry, no barrel:

```jsonc
// package.json — note: no "main"/"module"; nothing is built.
"exports": {
  "./package.json": "./package.json",
  "./*": "./src/fonts/*.ts"
}
```

So `@smoove/google-fonts/noto-sans` → `src/fonts/noto-sans.ts`, which the
consumer's bundler (Vite in the demo/SSR, or any TS-aware bundler) transpiles.
Generated modules import the base via a relative path (`../runtime.js` — the `.js`
specifier resolves to `runtime.ts` under `moduleResolution: bundler`/`nodenext`),
so `runtime` needs no export entry. Because each family is an isolated module
reachable only by its own subpath, importing one font never pulls the others.

> Consumed as source, so there is nothing to compile or publish-build. This is
> the first no-build package in the repo (player builds with Vite, the rest with
> `tsc -b`); it is **not** a project reference of other packages — they resolve
> its types through the `exports` `.ts` paths, like the demo already does for its
> own source.

## Runtime base class (`src/runtime.ts`, hand-written)

```ts
import { Font, type FontStyleName } from "@smoove/core";

export interface GoogleFontOptions<W extends string = string, S extends string = string> {
  /** Weights to register. Omit → all weights this family provides. */
  weights?: readonly W[];
  /** Styles to register. Omit → all styles this family provides. */
  styles?: readonly S[];
}

/** variant key `"<weight>-<style>"` → woff2 URL. */
export type FaceMap = Readonly<Record<string, string>>;

export class GoogleFont<W extends string = string, S extends string = string> extends Font {
  constructor(family: string, faces: FaceMap, options?: GoogleFontOptions<W, S>) {
    super({ family, faces: selectFaces(family, faces, options) });
  }
}

function selectFaces(family: string, faces: FaceMap, options?: GoogleFontOptions): {
  weight: string; style: FontStyleName; src: string;
}[] {
  const w = options?.weights, s = options?.styles;
  const out = [];
  for (const [variant, src] of Object.entries(faces)) {
    const [weight, style] = variant.split("-") as [string, FontStyleName];
    if (w && !w.includes(weight)) continue;
    if (s && !s.includes(style)) continue;
    out.push({ weight, style, src });
  }
  if (out.length === 0) {
    throw new Error(`@smoove/google-fonts: "${family}" — no faces match the selected weights/styles.`);
  }
  return out;
}
```

`GoogleFont extends Font`, so everything from core works unchanged: `.face()`,
the buffer/`registerAsset` gate, server disk-caching of the remote `src`, and the
transparent-while-buffering behavior.

## Generated module shape (`src/fonts/noto-sans.ts`)

```ts
// AUTO-GENERATED by scripts/generate.ts — do not edit.
import { GoogleFont, type GoogleFontOptions } from "../runtime.js";

const FAMILY = "Noto Sans";
const FACES = {
  "100-normal": "https://fonts.gstatic.com/s/notosans/v42/…woff2",
  "400-normal": "https://fonts.gstatic.com/s/notosans/v42/…woff2",
  "400-italic": "https://fonts.gstatic.com/s/notosans/v42/…woff2",
  "700-normal": "https://fonts.gstatic.com/s/notosans/v42/…woff2",
  // …
} as const;

export type NotoSansWeight = "100" | "400" | "700" /* …actual weights… */;
export type NotoSansStyle = "normal" | "italic";
export type NotoSansOptions = GoogleFontOptions<NotoSansWeight, NotoSansStyle>;

export default class NotoSans extends GoogleFont<NotoSansWeight, NotoSansStyle> {
  constructor(options?: NotoSansOptions) {
    super(FAMILY, FACES, options);
  }
}
```

**Typing:** `weights`/`styles` are constrained to literal unions of what the
family actually ships, derived per font. (A valid weight + valid style whose
*combination* doesn't exist — e.g. `400` + `italic` when only `700-italic` exists —
is silently skipped; the union types only guarantee each value individually
exists.)

## Generation script (`scripts/generate.ts`)

Run by maintainers/CI: `GOOGLE_FONTS_API_KEY=… pnpm --filter @smoove/google-fonts generate`.

1. `GET https://www.googleapis.com/webfonts/v1/webfonts?key=$KEY&capability=WOFF2&sort=alpha`.
2. For each `item` (`{ family, variants, files, subsets }`):
   - **slug**: `family.toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/^-|-$/g, "")` (fontsource-compatible, e.g. `Noto Sans JP` → `noto-sans-jp`).
   - **class name**: PascalCase of the family (`DM Sans` → `DMSans`, `Press Start 2P` → `PressStart2P`).
   - **variant → (weight, style)**: `regular`→`400/normal`, `italic`→`400/italic`,
     `<n>`→`<n>/normal`, `<n>italic`→`<n>/italic`. `files[variant]` is the woff2 URL.
   - Build `FACES` keyed `"<weight>-<style>"`, the weight/style unions, write `src/fonts/<slug>.ts`.
3. Write `src/manifest.ts` (`{ slug, family, weights, styles }[]`) for docs/tests.
4. Print a summary (counts, any slug collisions → fail loudly).

The script is the only thing that needs network/key; the published package is static.

## Monorepo integration (no build)

- Add `packages/google-fonts` to `pnpm-workspace.yaml` (already globs `packages/*`).
- **No build step.** `package.json` has no `build` script (or a no-op); there is
  no `dist`. The only script is `generate`.
- `tsconfig.json` exists for editors + an optional `typecheck` (`tsc --noEmit`),
  **not** `composite` and **not** a root project reference. The ~1,700 files
  affect only typecheck/editor indexing, not any emit — and `tsc --noEmit` over
  tiny modules is cheap. (If editor indexing ever drags, the `typecheck` can scope
  to `src/runtime.ts` + a sample, since the generated modules are mechanical.)
- Consumers resolve the package as source via the `exports` `.ts` paths; Vite
  transpiles for both browser and SSR. The `@smoove/vite` `serverAssets`
  rewrite is irrelevant here — `src` values are already remote URLs, which the
  core loaders handle on both sides.
- Demo: add a `compositions/google-font` scene importing one family to exercise it
  end-to-end (browser FontFace + the buffer gate).

## Selection semantics

- `new NotoSans()` → **all** faces (no filter).
- `new NotoSans({ weights: ["400"] })` → every style at weight 400.
- `new NotoSans({ styles: ["italic"] })` → every weight, italic only.
- `new NotoSans({ weights: ["400","700"], styles: ["normal"] })` → the 2 existing combos.
- `new NotoSans({ subset: "cyrillic" })` → all faces, cyrillic glyphs (default `"latin"`).
- Empty selection (impossible combo) → throws a clear error (rather than core's
  generic "at least one face"). Unknown `subset` warns and falls back to `latin`.

## Subsets (implemented)

Each generated module groups faces by subset — `FACES = { latin: { "400-normal":
url, … }, cyrillic: { … }, … }` — and the typed `subset?` option (default
`"latin"`) picks **one** subset. One subset = one `src` per `(weight, style)`,
which fits core's `Font` model and skia's first-wins `FontLibrary` exactly, so
the browser and headless renders load the same file (no core change, no
`unicodeRange`). Per-subset URLs come from the **CSS2 API** (one throttled
request per family) since the Developer API only returns full files; the
Developer API still supplies the family catalog.

## Implementation checklist

1. Scaffold `packages/google-fonts` — `package.json` (wildcard `exports` → `src/*.ts`,
   peer deps `konva` + `@smoove/core`, no `build`, a `generate` script), and a
   non-composite `tsconfig.json`. Add to `pnpm-workspace.yaml` if not auto-globbed.
2. `src/runtime.ts` — `GoogleFont` base + `selectFaces` + option types.
3. `scripts/generate.ts` — Developer API → `src/fonts/*.ts` + `src/manifest.ts`.
4. Run `generate` with a key; commit the generated modules + manifest.
5. Verify a sample import (`noto-sans`) types + constructs (consumed as source, no
   build); confirm importing one font doesn't pull others (tree-shaking).
6. Demo scene + a small test that constructs a few fonts and asserts face counts.
7. `README.md` for the package; link from `doc/README.md`.

## Deferred / open

- **Variable fonts**: the Developer API lists discrete named variants; we treat
  each as a static face. True variable-axis (`wght` range) support is a later
  enhancement.
- **Single-subset only**: one subset is loaded per font (default `latin`). Loading
  *multiple* subsets simultaneously (per-glyph fallback across ranges) would need
  the browser `unicodeRange` descriptor — and is fundamentally limited server-side
  by skia's first-wins-per-slot `FontLibrary` — so it's intentionally out of scope.
  Pick the subset matching your text.
- **URL drift**: gstatic URLs are versioned (`…/v42/…`). Regenerating refreshes
  them; old URLs keep working, so no runtime breakage between regenerations.
- **`display` / `text` options**: not available via the Developer API source;
  would need the CSS API. Deferred.
```
