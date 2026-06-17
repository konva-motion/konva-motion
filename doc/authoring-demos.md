# Authoring docs demos

Every live demo in the documentation is a real composition that runs in the
`<km-player>` web component and is shown as source from the *same file* — one
source of truth, no copy-paste drift.

## The contract

A demo is a single file:

```
packages/docs/src/demos/<name>.ts
```

whose **default export is a `Composition`** (a `() => Composition` factory —
sync or async — also works; the player's `resolveComposition` unwraps it).

```ts
// packages/docs/src/demos/<name>.ts
import { Composition, Sequence } from "@konva-motion/core";
import Konva from "konva";

const comp = new Composition({
  id: "<name>",
  fps: 30,
  durationInFrames: 90,
  width: 1280,
  height: 720,
  loop: true, // loop so the embedded player replays
});

const main = new Sequence({ from: 0, durationInFrames: 90 });
main.add(new Konva.Rect({ x: 0, y: 0, width: 1280, height: 720, fill: "#0d1117" }));
// …build the scene; drive attributes from the frame via main.register(...)…
comp.add(main);

export default comp;
```

That's it — no registry to edit, no manual wiring.

## Embedding it in a page

In any `.mdx` page, reference the demo by its file name:

```mdx
<Demo name="<name>" />
```

`<Demo>` ([src/components/demo.tsx](../packages/docs/src/components/demo.tsx))
resolves two views of the file from its name via `import.meta.glob`:

- the served **`?url`** module the `<km-player src=…>` dynamically `import()`s at
  runtime (the same path a genuinely remote composition takes), and
- the **`?raw`** source string behind the "View source" toggle.

Optional props: `lang` (code-block language, default `"ts"`), `label` (caption
text), and `src`/`source` to override the resolved values for a composition that
lives outside `src/demos/`.

An unknown `name` throws at render with the list of available demos, so a typo
fails loudly instead of rendering an empty player.

## Assets (images, video, audio)

Demos that need local media import it from a sibling `assets/` folder:

```
packages/docs/src/demos/assets/<file>
```

```ts
import clipUrl from "./assets/sync-test-1.mp4";
import musicUrl from "./assets/music-a.mp3";
```

Because the demo is emitted as its own `?url` chunk, Vite rewrites these imports
to served URLs that resolve when the player loads the module. `.mp4`/`.mp3`/
`.wav`/image imports are typed via `vite/client` (already in the docs tsconfig).

## Checklist

- [ ] File at `packages/docs/src/demos/<name>.ts`, default-exports a `Composition`.
- [ ] `loop: true` so the embedded player replays.
- [ ] Any media lives under `src/demos/assets/` and is imported relatively.
- [ ] Page embeds it with `<Demo name="<name>" />`.
