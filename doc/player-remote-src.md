# Plan: load a Composition from a remote file in `<km-player>`

## Goal

Add a `src` attribute to `<km-player>` so it can load and run a Composition
from a remote ESM module, mirroring `<video src>`:

```html
<km-player src="https://cdn.example/my-comp.js" controls></km-player>
```

## Resolution contract

The remote file is a bundled ESM module whose **default export** is a
`CompositionInput` — the same shape the studio registry already accepts:

- a `Composition` instance, **or**
- a factory `() => Composition | Promise<Composition>`, **or**
- a factory resolving to `{ default: Composition }`.

The player dynamically `import()`s the URL, resolves the default export through
that ladder, then assigns `this.composition = comp` — reusing the **entire
existing mount/bind path** in `packages/player/src/km-player.ts` (the
`composition` setter at line 67). No engine / core changes are needed.

## Changes

### 1. `packages/player/src/km-player.ts`

- Add `"src"` to `observedAttributes`.
- Add `src` getter/setter (attribute-backed, like `loop` / `controls`).
- Add a private `_loadFromSrc()`:
  - Resolve the URL against `document.baseURI` so consumer-relative paths
    behave like `<video src>`.
  - `import(/* @vite-ignore */ url)` → take `.default` → resolve through the
    factory / `{ default }` ladder. Detect a factory by `typeof === "function"`
    (a `Composition` is not callable), so **no runtime import of core** is
    required — keeps `import type { Composition }` as-is and avoids bundling
    concerns.
  - Guard against races with an incrementing `_loadSeq` token — a stale `src`
    resolving late must not clobber a newer one.
  - Toggle a reflected `loading` attribute during the fetch (CSS hook).
  - On success: `this.composition = comp` + emit `loaded`.
    On failure: emit the existing `error` event.
- Call `_loadFromSrc()` from `connectedCallback` (when `src` is present and no
  composition was set imperatively) and from `attributeChangedCallback` for
  `"src"`.
- Emit a `loadstart` event when a load begins.
- An explicitly-assigned `composition` property still takes precedence over
  `src`.

### 2. `packages/player/src/events.ts`

- Add `LoadStartDetail { src }` and `LoadedDetail { src; composition }`; add
  both to `KmPlayerEventMap`. Reuse the existing `error` event for failures.

### 3. `packages/player/src/index.ts`

- Export the two new event detail types.

### 4. `doc/README.md`

- Document the `src` attribute, the expected default-export contract,
  relative-URL resolution, and the new `loadstart` / `loaded` events.
- **Security note:** dynamic import executes arbitrary code, so only load
  trusted URLs (and CSP `script-src` must allow the origin).

## Out of scope / decisions

- **No imperative `load()` method** — the `src` attribute is the only surface.
- **No JSON / serialization layer** — compositions keep their updater closures.
- Setting `src=""` / removing it does **not** tear down a running composition;
  a new `src` swaps it.
- Verification: build the player package
  (`pnpm --filter @konva-motion/player build`) and typecheck. An optional
  end-to-end demo (serving a composition as a standalone module) can be added.
