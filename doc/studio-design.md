# `@smoove/studio` — design

Status: **plan** (no code yet). This documents the intended shape of a new
`packages/studio` package. It generalizes two inputs into one reusable,
composable library:

1. the working React prototype in `demo/src/studio/` (catalog, `useComposition`,
   the `kf` props-form), and
2. the fuller **KmStudio** design handoff (a Claude-Design export: a dark
   creative-tool shell — left composition library, center stage, bottom
   timeline dock, right props panel, server-render dialogs + queue, theme
   tweaks).

…plus a runtime-agnostic render half so the *same* compositions drive both the
in-browser studio and headless video rendering.

The KmStudio design informs the **component inventory** (§10) and the
**configurability model** (§11): every region is opt-out and every part is
replaceable. Nothing here is built yet — this is the plan to review before any
code lands.

---

## 1. Goal & the one big idea

Studio lets any project stand up a "demo app" — a Remotion-Studio-like shell:
a catalog of compositions on the left, a live `<smoove-player>` preview in the
center, an auto-generated props form, a scrubber, and a **Render** button that
produces an MP4 — and then **extend or replace any piece of it**.

North star: **Storybook, but for your smoove videos, and fully
customizable.** Not a framework — an easy, themeable way to browse, tweak, and
render the compositions you already wrote.

Two halves ship from one package:

- **React UI** — fully-styled, composable components (Tailwind + CSS-variable
  theming) the user drops in and overrides.
- **Render functions** — plain, runtime/framework-agnostic functions that do
  the rendering work, wireable into *any* JS runtime (Node, Bun, Deno, edge)
  and *any* framework (Express, Hono, Next, Fastify, a queue worker, or just a
  CLI). Studio bakes in **no transport**.

**The keystone is a shared, isomorphic registry of id-keyed entries.** Each
entry is `{ id, …metadata, load(props) }`, where `load` returns a smoove
`core.Composition` (possibly async, for lazy/code-split compositions). The
**`id` is mandatory** and is the stable key shared by the frontend and the
render backend — both import the same registry module and address a composition
by `id`, which doubles as the route segment. Display/props metadata lives on the
entry (so the catalog renders before anything loads); *runtime* facts (`fps`,
`durationInFrames`, `width()`/`height()`, frame, layers) are read off the
Composition once `load` resolves. It's framework-free and Node-safe (because
`core` is: `setFrame(n)` works anywhere; only `play()` needs a browser). The
React studio and the render functions import the **same module** — one shared
artifact is what makes "share the same compositions between frontend and
backend" true by construction.

It formalizes today's `demo/src/studio/catalog.ts` (`StudioDemo` / `DemoDef`):
the `DemoDef.build(props)` builder becomes the entry's `load(props)`, and the
catalog metadata becomes entry fields — but now keyed by a mandatory `id` and
load-on-demand.

---

## 2. Package layout & subpaths

One package, granular files. Per the chosen convention: **every render
function/helper is its own file; every component is its own directory.**

```
packages/studio
  src/
    registry/                 # isomorphic, React-free, Node-safe
      defineRegistry.ts       # defineRegistry(RegistryEntry[]) -> Registry (entries/load/peek)
      loadEntry.ts            # memoized load(id): resolve entry.composition (unwrap default) once
      compositionInfo.ts      # read fps/duration/size off a (loaded) Composition
      deriveLayers.ts         # walk top-level Sequences -> StudioLayer[] (Video/Audio/Group)
      types.ts                # Registry, RegistryEntry, StudioLayer, Render* types
      schema/                 # the `kf` form DSL (from demo/src/studio/kf.ts) — type of entry.propsSchema

    render/                   # runtime/framework-agnostic — ONE function per file
      renderComposition.ts    # (registry, RenderRequest) -> RenderResult            (file output)
      renderToStream.ts       # (registry, RenderRequest) -> ReadableStream/AsyncIterable
      renderStill.ts          # (registry, StillRequest)  -> Buffer
      probe.ts                # (registry, id) -> CompositionInfo
      createRenderQueue.ts    # optional in-memory job queue w/ progress events
      handleRenderRequest.ts  # pure (req, {registry, backend}) -> result|stream — no HTTP
      defaultBackend.ts       # RenderBackend backed by @smoove/renderer (optional/peer dep)

    components/               # React — ONE directory per component, built on Base UI
      Studio/                 # root provider; owns StudioContext
      LeftPanel/              # composition library: Base UI Collapsible groups + search
      Toolbar/                # Base UI Menu (kebab) + title + Zoom (Menu/Select)
      Stage/                  # mounts the Composition's Konva stage; fit-scales it
      Timeline/               # Progress + layered modes, loop region, transport
      RightPanel/             # Base UI Tabs: Props (SchemaForm) / Info / Code
      SchemaForm/             # schema-driven auto form (Base UI Slider/Switch/Select/NumberField)
      RenderDialog/ ExportFrameDialog/   # Base UI Dialog
      RenderQueue/
      Toasts/                 # Base UI Toast
      TweaksPanel/            # Base UI ToggleGroup
      Icon/ Button/           # thin in-house primitives (no Base UI equivalent needed)

    hooks/                    # headless behavior, no markup
      useStudio.ts            # read StudioContext
      useComposition.ts       # mount/teardown a Composition's stage + drive setFrame
      usePropsForm.ts         # entry.propsSchema <-> props (signal passed to load) + comp.refresh()
      useRender.ts            # call the injected transport, expose {status, progress, result}
      useSignal.ts            # subscribe to a core ReadonlySignal in React
                              # (no useClickOutside / focus-trap — Base UI owns dismiss + a11y)

    styles/
      studio.css              # compiled Tailwind; opt-in import, CSS-variable themed

    index.ts                  # React UI barrel ("." export)
```

### Exports

```jsonc
"exports": {
  ".":          { "types": "./dist/index.d.ts",            "import": "./dist/index.js" },
  "./registry": { "types": "./dist/registry/index.d.ts",   "import": "./dist/registry/index.js" },
  "./render":   { "types": "./dist/render/index.d.ts",     "import": "./dist/render/index.js" },
  "./styles.css": "./dist/styles/studio.css"
}
```

**Hard rule (mirrors how `core`/`renderer` stay framework-agnostic):**
`./registry` and `./render` must never import React or anything DOM-only. A
render worker does `import { renderEntry } from "@smoove/studio/render"`
and `import registry from "./compositions"` without dragging in React. Only the
`.` entry pulls React + `@smoove/player`.

### Dependencies

| Dep | Kind | Why |
| --- | --- | --- |
| `@smoove/core` | peer | compositions, signals, `setFrame` |
| `react` / `react-dom` | peer | UI half only |
| `@smoove/player` | peer | `<smoove-player>` preview host |
| `@smoove/renderer` | **optional** peer | only the *default* backend uses it; users can supply their own |
| `konva` | peer | transitive, pinned by consumer |
| `@base-ui-components/react` | dep | **headless UI primitives** the components are built on (Dialog, Menu, Select, Tabs, Slider, Switch, Toast, NumberField…) — accessible + Floating-UI positioned, unstyled |
| `zod` | dep | the `kf` schema DSL |
| `tailwindcss` | devDep | styles compiled at build; consumers import the shipped CSS |

---

## 3. The registry — id-keyed entries, definition at `defineRegistry`

To support **lazy loading**, the studio metadata is **not** put on the
Composition (you can't read attributes off a composition that hasn't loaded).
Instead each composition is declared as a registry **entry** at
`defineRegistry` time — `{ id, …metadata, composition }`. The Composition object
stays clean; the studio reads only *runtime* facts off it once loaded. A
composition module **default-exports its `Composition`**; the entry's
`composition` is that instance or a lazy loader (`() => import("…")`) resolving to
it. A common layout is one directory per demo — `schema.ts`, `composition.ts`
(default-exports the `Composition`), and `index.ts` (default-exports the
`RegistryEntry`).

**Every entry has a mandatory `id`.** The `id` is the **stable shared key**: the
frontend and the render backend import the same registry module and address the
same composition by `id`. It's also the **routing key** — the studio's selected
composition maps to a route (`/:id`), and a render request carries the `id`, so
deep-links and FE↔BE addressing line up by construction.

```ts
interface RegistryEntry<P = Record<string, unknown>> {
  id: string;                  // REQUIRED — shared FE/BE key + route segment
  title?: string;              // catalog label; defaults to id
  group?: string;              // catalog section; default "Compositions"
  tags?: string[];
  description?: string;
  propsSchema?: KmSchema;      // kf / field descriptors → drives the auto-form
  defaultProps?: P;
  /** The default-exported Composition, or a lazy loader resolving to one
      (`() => import("…")`). Props live on the comp's `props` signal — the studio
      pushes form edits via `comp.setProps`, so edits refresh without a rebuild. */
  composition: Composition | (() => Composition | Promise<Composition>
    | Promise<{ default: Composition }>);
}

export type Registry = {
  /** Lightweight catalog rows — id + metadata + load status, no compositions built. */
  entries(): RegistryEntry[];
  /** Resolve (memoized) the composition for an id. */
  load(id: string): Promise<Composition>;
  /** The already-loaded instance for an id, if any (sync). */
  peek(id: string): Composition | undefined;
};

export function defineRegistry(entries: RegistryEntry[]): Registry;
```

Because all the catalog needs (`id`, `title`, `group`, `tags`, `propsSchema`)
lives on the entry, the sidebar renders **before** anything loads. `load(id)` is
**memoized** so the preview and an in-process render share one construction;
selecting an entry (or hitting its route) triggers the load and flips its status
`idle → loading → ready`.

```ts
// registry.ts — imported by the studio UI AND the render worker.
// With the @smoove/vite plugin, HMR is injected automatically — you never
// write `import.meta.hot`.
import { defineRegistry } from "@smoove/studio";
export default defineRegistry([
  // each module default-exports its Composition (which owns a `props` signal)
  { id: "intro", title: "Intro", group: "Brand",
    composition: () => import("./compositions/intro") },
  { id: "epic",  title: "Epic promo", group: "Brand",      // lazy / code-split
    composition: () => import("./compositions/epic") },
]);
```

**What's read off the *loaded* Composition** (runtime facts only — never
duplicated on the entry):

| Studio needs | Read straight off the Composition |
| --- | --- |
| fps | `comp.fps` |
| total frames | `comp.durationInFrames` |
| width / height | `comp.width()` / `comp.height()` (it's a `Konva.Stage`) |
| **the preview** | mount `comp`'s stage into the DOM; drive `comp.setFrame(round(t · duration))` — the Composition renders itself, no preview slot needed |
| current frame | `comp.frame` (signal) |
| **timeline layers** | derived from `comp`'s **top-level `Sequence`s** (`from` + `durationInFrames` → normalized `start`/`end`, name from the layer) |

(`comp.id` should equal the entry `id` — the studio uses the entry `id` as the
source of truth and can assert they match.)

**Props & layers:**
- *Props* — the Composition owns a live `props` signal (seeded from its
  constructor `props` option). The form pushes edits with `comp.setProps`, which
  re-applies the current frame (the frame-preserving pattern), with the form built
  from the entry's `propsSchema` + `defaultProps`. The bare `<smoove-player>` can drive
  props too via `player.setProps(...)`.
- *Layers* — top-level `Sequence`s only, one track each; **kind** recognized as
  **Video / Audio / Group** (else generic `sequence`/`transition`) from an
  optional `kind` tag on the Sequence, falling back to node-content inspection.
  An entry may supply an explicit `layers` override.

The `kf` schema DSL (`demo/src/studio/kf.ts`) is the type behind
`entry.propsSchema`, shared by the form and any server-side prop validation
(`safeParse` works per field).

---

## 4. The central store, and the React UI on top

Everything in the studio reads from **one central store** (`StudioContext` +
a small reactive store). It is the single source of truth — the catalog (each
entry's `id` / eager label / **load status**: idle·loading·ready·error), the
selected id and its loaded `Composition`, props values + signal, playback state,
and the **render job state** (queued / rendering / progress / done / error /
output URL). Every prebuilt component and every headless hook reads from this
store; nothing holds its own copy of app state. Selecting a lazy composition
flips its entry to `loading` and the store resolves `registry.load(id)` before
mounting it on the stage.

This is what makes the render side clean: the **render functions stay
transport-free** (§5), and *whatever* the user's backend does — SSE, short
polling, a websocket, an in-process callback — it just **dispatches updates
into the store**, and the UI re-renders. The store is the one place the two
halves meet on the client.

```ts
// shape the store exposes (read via useStudio(); mutate via actions/hooks)
type StudioStore = {
  registry: Registry;
  selectedId: string | undefined;
  composition: Composition | null;
  props: Record<string, unknown>;          // current form values
  // render job state — the "central place" the backend feeds:
  jobs: Record<string, RenderJob>;         // { status, progress, result?, error? }
  actions: {
    select(id: string): void;
    setProps(next: Record<string, unknown>): void;
    startRender(req: RenderRequest): string;        // returns jobId, calls injected transport
    updateJob(jobId: string, patch: Partial<RenderJob>): void;  // backend pushes progress here
  };
};
```

A user wiring short-polling, for example, just calls
`store.actions.updateJob(jobId, { progress })` on each tick — no studio API
beyond that. The studio neither knows nor cares how the bytes arrived.

**Routing is id-based and optional.** `selectedId` is the registry entry `id`,
which is exactly the route segment (`/:id`). The store exposes it and accepts an
initial id; the consumer (or a built-in adapter) syncs it to their router
(`react-router`, Next, hash, or none). Because the same `id` keys the backend,
a deep-link, a preview, and a render request all address one composition.

The React layer mirrors the player's compound-component model (`<smoove-player>` +
slotted control elements), in React over this store. **Default is fully
styled** (Tailwind); users restyle by **overriding CSS variables**, not by
rewriting markup. Controls are **Studio's own React components built on Base
UI** by default — accessible and keyboard-navigable out of the box, themed with
the same `--smoove-*` variables as the rest of the shell; the player's
`<smoove-player-*>` control elements remain available for anyone who prefers them.

```tsx
import { Studio } from "@smoove/studio";
import "@smoove/studio/styles.css";
import registry from "./compositions";

<Studio registry={registry} render={myTransport}>
  <Studio.Catalog />
  <Studio.Preview />        {/* mounts <smoove-player>, sets .composition */}
  <Studio.Controls />
  <Studio.Timeline />
  <Studio.PropsPanel />
  <Studio.RenderButton />
</Studio>;
```

- `<Studio>` owns `StudioContext`: the registry, the selected entry, the live
  `Composition`, the props values + signal, and the injected `render`
  transport. Selection/build/teardown logic is lifted straight from
  `demo/src/studio/useComposition.ts` (props `Signal` → `comp.refresh()` on
  edit, no rebuild, playhead preserved).
- Every prebuilt piece is a thin wrapper over a headless hook
  (`useStudio`, `useComposition`, `usePropsForm`, `useRender`). A user who
  wants a totally different layout ignores the components and builds on the
  hooks — same "drop the defaults, keep the behavior" escape hatch the player
  offers.
- **Theming:** components consume `--smoove-*` CSS variables (colors, radii,
  spacing, font). `studio.css` ships sensible defaults; overriding a handful of
  vars re-skins the whole shell. No Tailwind config required in the consumer —
  the CSS is precompiled and scoped under a `.smoove-studio` root.
- `<Studio.Preview>` reuses the React-wraps-`<smoove-player>` pattern already
  proven in `demo/src/studio/App.tsx` (a `ref`, set `.composition`, player
  handles letterbox/scale). The player's own control custom-elements remain
  available if a user prefers them over `<Studio.Controls>`.

---

## 5. Render half: functions, no transport

The user said: *functions that do all the work, wireable into any runtime/any
framework.* So `render/` exposes **pure orchestration functions** plus a
**pluggable backend** — and ships **zero** HTTP/server code. The transport is
the user's to write (and is symmetric on the client via an injected function).

```ts
// @smoove/studio/render

export interface RenderBackend {
  render(comp: Composition, opts: RenderOptions): Promise<RenderResult>;
  renderStill?(comp: Composition, opts: StillOptions): Promise<Buffer>;
  // progress flows through opts.onProgress (RenderProgress), same as renderer
}

/** Default backend wraps @smoove/renderer (optional peer dep). */
export function defaultBackend(): RenderBackend;

/** The core "do the work" function: look up the entry, build it with props,
    render it. Runtime-agnostic — no Request/Response, no fs assumptions beyond
    what the backend needs. */
export function renderEntry(
  registry: Registry,
  req: RenderRequest,
  backend?: RenderBackend,           // defaults to defaultBackend()
): Promise<RenderResult>;

export function renderEntryToStream(registry: Registry, req, backend?): Promise<ReadableStream>;
export function renderEntryStill(registry: Registry, req, backend?): Promise<Buffer>;
export function probeEntry(registry: Registry, id: string): Promise<CompositionInfo>;

/** Optional in-memory queue: enqueue jobs, subscribe to RenderProgress,
    collect results. Lets a "demo app" show a render queue with no DB. */
export function createRenderQueue(registry: Registry, backend?): RenderQueue;
```

```ts
// RenderRequest — the shared FE<->BE contract (lives in registry/types.ts)
export type RenderRequest = {
  id: string;                          // composition id in the registry
  props?: Record<string, unknown>;     // validated against entry.schema if present
  range?: { from: number; to: number };
  resolution?: { width: number; height: number };
  quality?: "low" | "medium" | "high" | "max";
  output?: string;                     // backend-specific sink hint
};
```

### Wiring is the user's, and stays symmetric

Because Studio injects a `render` transport into the UI and exposes plain
functions on the server, the same `RenderRequest` shape crosses whatever
boundary the user picks:

- **In-process** (Electron, a local tool): `render={(req) => renderEntry(registry, req)}` — no network at all.
- **Web `Request`/`Response`** (Hono, Next route, Bun.serve, Deno, edge):
  ```ts
  // user's route — Studio provides the work, user provides the transport
  app.post("/render", async (c) => Response.json(await renderEntry(registry, await c.req.json())));
  ```
- **Node `http`/Express**, a queue worker, SSE for progress, etc. — all just
  call `renderEntry` / `createRenderQueue`.

On the client, `<Studio.RenderButton>` calls `store.actions.startRender(req)`,
which invokes the injected transport and tracks the job in the central store
(§4). The transport's job is only to **push progress/result back into the store
via `updateJob`** — by SSE, polling, websocket, or an in-process callback. The
UI (button, a render-queue panel, progress bar) re-renders off the store. The
render *functions* themselves stay transport-free; `onProgress` is how a
server-side caller observes progress before it forwards it however it likes.

This is the "implement the render backend as you like" requirement: the default
backend gets you rendering immediately via `@smoove/renderer`; a custom
`RenderBackend` (Lambda, a render farm, a different encoder) plugs in without
touching the UI or the registry.

---

## 6. What moves out of the demo

The demo's `src/studio/` becomes a thin *consumer* of the package — proof the
extraction works and the live example:

| Today (`demo/src/studio/`) | Becomes |
| --- | --- |
| `catalog.ts` (`StudioDemo`, `DemoDef`) | `registry/` + per-project `compositions.ts` |
| `kf.ts` (Zod form DSL) | `registry/schema/` |
| `useComposition.ts`, `useSignal.ts` | `hooks/` |
| `SchemaForm.tsx`, `RightPanel.tsx` | `components/PropsPanel/` |
| `Sidebar.tsx` | `components/Catalog/` |
| `Player.tsx`, `ZoomControl.tsx`, `Icon.tsx` | `components/Controls,Zoom,Icon/` |
| `App.tsx` shell | `components/Studio/` (compound root) + a demo that composes it |
| `styles.css` | `styles/studio.css` (Tailwind, CSS-var themed) |

The render half is new (the demo has no render button today); it builds on the
existing `@smoove/renderer` API (`renderComposition`, `renderToStream`,
`renderStill`, `probeComposition`, `RenderOptions`/`RenderResult`).

---

## 7. Build & conventions (matches existing packages)

- `package.json`: `type: module`, `exports` map above, `files: ["dist"]`,
  scripts `build: tsc -b`, `dev: tsc -b --watch`, `clean`.
- `tsconfig.json`: `extends ../../tsconfig.base.json`, `composite: true`,
  `rootDir: src`, `outDir: dist`, references to `core` / `player` /
  (optionally) `renderer`.
- Register in root `tsconfig.json` references and `pnpm-workspace.yaml`
  (already globs `packages/*`, so just the tsconfig reference + `pnpm build`).
- Tailwind compiles to a static `studio.css` at build time so consumers need no
  Tailwind setup — they `import "@smoove/studio/styles.css"`, exactly the
  opt-in-styles pattern `@smoove/player` uses.
- Biome via the root config; no per-package config.
- Update `doc/README.md` once the public API stabilizes.

---

## 8. Open questions

1. **`./render` subpath name** — `/render` vs `/server`. `render` reads better
   for "runtime-agnostic functions" (it's not necessarily a server). Leaning
   `render`.
2. **Progress transport** — RESOLVED. `render/` stays 100% transport-free;
   progress flows through the central store's `updateJob` action (§4). The
   backend implements SSE / short-polling / ws / in-process however it wants
   and pushes into the store. The demo shows one wiring as a copyable example.
3. **Controls** — RESOLVED. Studio ships its own React controls by default
   (themed via `--smoove-*`); the player's `<smoove-player-*>` elements remain
   available for anyone who prefers them.
4. **Store implementation** — hand-rolled reactive store vs. a tiny dependency
   (e.g. zustand). Leaning hand-rolled over `core`'s existing `Signal`
   primitive to avoid a new dep and stay consistent with the rest of the repo.
5. **Schema dep** — `zod` becomes a hard dep of `registry`. Acceptable, since
   the form needs it and server-side prop validation reuses it.
6. **Registry sugar** — RESOLVED (§3). Catalog metadata (title/group/tags/
   description) lives on the id-keyed registry **entry**; the studio derives
   fps/duration/size/layers from the loaded Composition. No separate
   catalog-view layer; grouping reads from `entry.group`.
7. **Where does the metadata live?** — RESOLVED (§3): catalog metadata lives on
   the **registry entry**, defined at `defineRegistry` time — *not* on the
   Composition (reverted, to support lazy load). Each entry has a **mandatory
   `id`** (shared FE/BE key + route segment). The Composition stays clean except
   for its `props` signal; other runtime facts are read off it once loaded.
8. **Compositions eager or lazy?** — RESOLVED (§3): each entry's
   `load(props): Composition | Promise<Composition>` may be **sync or async**,
   so compositions can be code-split and loaded on demand. `load(id)` is
   memoized so preview and render share one construction.
9. **Layer derivation** — RESOLVED (§3): **top-level `Sequence`s only**, one
   track each; kind recognized as **Video / Audio / Group** (else generic), from
   an optional `kind` tag or node-content inspection. A composition may supply an
   explicit `layers` override.

---

## 9. Suggested phasing

1. **Scaffold + registry** — package, tsconfig/exports, `registry/` (+ move
   `kf` schema). No UI yet; unit-build green.
2. **Render functions** — `renderEntry` + `defaultBackend` over the existing
   renderer; `probeEntry`; a Node script in the demo proving headless render
   from the shared registry.
3. **Headless hooks** — lift `useComposition`/`useSignal`, add
   `useStudio`/`usePropsForm`/`useRender`.
4. **Components + styles** — build on **Base UI** primitives: `Studio` root,
   `LeftPanel`, `Stage`, `Timeline`, `RightPanel` (Tabs), `SchemaForm`
   (Slider/Switch/Select/NumberField), render `Dialog`s, `RenderQueue`, `Toast`
   `Toasts`, `TweaksPanel`; wire a studio-owned portal container so Base UI
   popups inherit the `.smoove-studio` scope + theme; author `studio.css` (Tailwind).
5. **Demo swap** — re-point `demo/src/studio` at the package; delete the
   duplicated prototype; wire a real Render button end-to-end.

---

## 10. Component inventory (from the KmStudio design)

Extracted from the design handoff. Convention: **one component per directory**
under `src/components/`, **one render-function/helper per file** under
`src/render/`. Every component is a thin reader of the central store and is
individually droppable.

**Built on Base UI.** Behavior, accessibility, focus management and popup
positioning come from **Base UI** (`@base-ui-components/react`) — we own only
the look. We don't ship hand-rolled `Dropdown`/`Modal`/`Toggle`/click-outside
(the spike's versions are discarded). Base UI parts are unstyled and take
`className` + expose `data-*` state attributes (`data-open`, `data-checked`,
`data-highlighted`…), which our `.smoove-studio`-scoped, CSS-variable-themed
stylesheet targets. The `render` prop lets us swap the underlying element where
the design needs a specific tag. This keeps the "fully styled by default,
themeable via CSS variables" decision while getting a11y for free.

| KmStudio control | Base UI primitive |
| --- | --- |
| kebab menu, context actions | `Menu` |
| zoom selector, format/quality/resolution selects | `Select` (or `Menu`) |
| render / export-frame modals | `Dialog` |
| right-panel Props/Info/Code | `Tabs` |
| boolean prop toggle | `Switch` |
| segmented controls (range Full/Region; Tweaks Mood/Canvas/Corners) | `Toggle` + `ToggleGroup` |
| number prop slider; volume | `Slider` |
| resolution W×H, fps inputs | `NumberField` |
| multiselect chips | `ToggleGroup` (or `Combobox` multiple) |
| sidebar collapsible groups | `Collapsible` / `Accordion` |
| toasts | `Toast` |
| control tooltips | `Tooltip` |
| scroll regions (sidebar/panel/menus) | `ScrollArea` |

**In-house primitives** (no Base UI equivalent needed)
| Component | Role |
| --- | --- |
| `Icon` | inline-SVG icon set (40+ glyphs: play/pause, layer-type icons, render/queue, chevrons…). `name` + `size`. |
| `Button` | `variant` = `default | ghost | primary | danger`, `size`, optional `icon`. Wraps a `<button>` (or Base UI part via `render`). |

**Shell / regions**
| Component | Maps to design | Notes |
| --- | --- | --- |
| `Studio` | `App` | compound root; owns provider; renders `Studio.Default` or composed children. |
| `LeftPanel` | `Sidebar` | brand/logo, search, collapsible composition groups (closed by default), top-level nav items + badges. Whitelabel (no per-item type icons). |
| `Toolbar` | `Toolbar` + `TopMenu` | kebab menu (Render… / Export frame… / Render Queue), composition title block, zoom control. |
| `ZoomControl` | `ZoomControl` | Fit + 25–200% steps dropdown. |
| `Stage` | stage-wrap/`video-frame` | centers + fit-scales the composition preview; fullscreen. |
| `Timeline` | `Timeline` | dock with **Progress / Layered** modes, ruler, loop region (in/out + drag-snap), named layers w/ enable toggles, transport (play/step/volume/time/loop/fullscreen), readout (resolution · realtime/target fps · frame). |
| `RightPanel` | `RightPanel` + `PanelHandle` | collapsed handle by default; tabs Props (SchemaForm) / Info / Code. |
| `SchemaForm` | `form.jsx` + `schema.jsx` (`kf`) | zod-driven auto form; all field types. Lives under `registry/schema` + a `components/SchemaForm`. |
| `RenderDialog` | `RenderDialog` | format/quality/resolution(W×H)/fps/range → enqueue. |
| `ExportFrameDialog` | `ExportFrameDialog` | export current frame (format + scale). |
| `RenderQueue` | `RenderQueue` | job list w/ live progress, download/cancel/remove. |
| `Toasts` | `Toasts` | transient notifications. |
| `TweaksPanel` | `tweaks-panel.jsx` | optional theme tweaks (Mood / Canvas / Corners) — pure CSS-var reshaping. |

The design's `LAYER_KINDS` (Sequence · Audio · Video · Group · Transition,
each with icon + color) is data the Timeline consumes.

---

## 11. Configurability — "change all things"

Two complementary mechanisms; use whichever fits.

### (a) Feature flags — quick opt-outs

`<Studio config={{ features: {...} }}>`. Every region/control defaults **on**;
set `false` to remove it (the component isn't rendered, not hidden with CSS).

```ts
interface StudioFeatures {
  leftPanel: boolean;       // the composition library sidebar
  rightPanel: boolean;      // props/info/code panel
  timeline: boolean;        // bottom dock
  toolbar: boolean;         // top bar
  search: boolean;          // sidebar search field
  zoom: boolean;            // scale dropdown            ← "hide the scale dropdown"
  render: boolean;          // server render (kebab Render…, dialog, queue) ← "disable render sections"
  exportFrame: boolean;     // export-frame button/dialog ← "never show the render frame button"
  renderQueue: boolean;
  layeredTimeline: boolean; // Progress/Timeline mode toggle (off ⇒ progress only)
  loopRegion: boolean;      // in/out controls
  volume: boolean; loop: boolean; fullscreen: boolean;
  tweaks: boolean;          // theme tweaks panel (default OFF)
}
```

Examples mapping the user's asks:
- never show the export-frame button → `features.exportFrame: false`
- hide the scale dropdown → `features.zoom: false`
- disable render entirely → `features.render: false` (also drops the queue + kebab items)
- drop the left/right panel → `features.leftPanel: false` / `features.rightPanel: false`

### (b) Slots — replace any part

`config.slots` (or props on the compound children) accept a `ReactNode` or a
render function and **override** the default. This is "build the bottom bar
yourself" / "change the logo".

```ts
interface StudioSlots {
  logo?: ReactNode;                                   // ← "change the logo"
  toolbar?: ReactNode | ((s: StudioApi) => ReactNode);
  transport?: ReactNode | ((s: StudioApi) => ReactNode);  // ← "build the bottom bar control"
  leftPanel?: ...; rightPanel?: ...; stage?: ...; emptyState?: ...;
}
```

### (c) Compound composition — full control

Power users skip `Studio.Default` and assemble regions themselves; anything
omitted simply doesn't render:

```tsx
<Studio compositions={comps} renderBackend={backend} config={{ features: { tweaks: true } }}>
  <Studio.LeftPanel logo={<MyLogo/>} />
  <Studio.Main>
    <Studio.Toolbar />
    <Studio.Stage />
    <MyOwnBottomBar />        {/* replaces Studio.Timeline */}
  </Studio.Main>
  {/* no Studio.RightPanel → no right panel */}
</Studio>
```

All three read the same store via `useStudio()`, so custom parts stay in sync
with selection, playback, props and render jobs. Theming is still CSS-variable
driven (`--accent`, `--bg-*`, `--radius`…), so `config.theme` / a style override
re-skins everything (the Tweaks panel is just a built-in driver of those vars).

---

## 12. Implementation notes (de-risked, not yet built)

A throwaway spike validated the riskier mechanics before committing the plan.
The spike was reverted; these are the concrete findings worth carrying into the
real build:

- **Central store as a single provider works.** One `StudioProvider` holding all
  state (selection, playback time in **seconds**, props-per-composition,
  timeline/region, jobs, toasts) and exposing a `StudioApi` via context kept
  every component a thin reader. `frame` derives from `time`; `durationSeconds =
  durationInFrames / fps`.
- **Preview = mount the Composition's stage (decided, §3).** The composition is
  a `Konva.Stage`, so the Stage component mounts it and drives `comp.setFrame(
  round(t · durationInFrames))` from the playhead — no preview slot, no per-comp
  React component. (The spike used a `{ t, p } => ReactNode` slot with CSS
  stand-ins only because it had no real core Compositions to mount; that slot is
  *not* the plan — it stays available as an optional escape hatch for non-konva
  visuals.)
- **Stage fit-scaling gotcha (important).** A CSS `transform: scale()` is
  *visual only* — the element's layout box stays native size (e.g. 1280×720), so
  `place-items: center` centers the **unscaled** box and the scaled visual drifts
  off to one side. Fix: wrap the frame in a box whose width/height are the
  *scaled* dims and scale the inner frame from `transform-origin: top left`. The
  spike hit this exact bug; bake the wrapper into the Stage from the start.
- **Pluggable backend shape that fit cleanly.**
  `RenderBackend.start(req, onUpdate) => Promise<Partial<RenderJob>|void>`, where
  the backend calls `onUpdate(patch)` to push progress into the store
  (`updateJob`). A mock backend (ticking progress on a timer) and the central
  store proved the FE↔store↔backend loop without any transport baked in.
- **Primitives = Base UI, not hand-rolled.** The spike hand-rolled
  `Dropdown`/`Modal`/`Toggle` + a `useClickOutside`; the real build uses Base UI
  (`@base-ui-components/react`) for those instead — Menu/Select/Dialog/Tabs/
  Switch/Slider/NumberField/Toast/ToggleGroup — for a11y, focus traps and popup
  positioning. We style Base UI parts via `className` + their `data-*` state
  attributes; the spike's CSS (class names + tokens) largely carries over by
  re-targeting it at those parts.
- **Styles scope.** Ship the CSS under a `.smoove-studio` root wrapper (vars +
  scoped selectors) so the opt-in `styles.css` doesn't touch the host page —
  same opt-in pattern as `@smoove/player`. Base UI portals (menus,
  dialogs, toasts) render to `document.body` by default — render them into a
  studio-owned portal container (or add the `.smoove-studio` class to the portal)
  so the scoped styles + theme vars still apply.
- **Repo fit.** `tsconfig.base.json` sets `noUncheckedIndexedAccess` — array
  indexing needs guards/`!`. Package builds with `tsc -b`; CSS is copied to
  `dist/` post-build (mirrors the player). React is a peer dep; `jsx:
  "react-jsx"` in the package tsconfig.
- **The `kf` zod form was *not* spiked.** The spike used a lightweight
  `StudioField` descriptor list instead of porting the full `kf` zod DSL. Decide
  in §8.5 whether v1 ships the lightweight descriptors, the full `kf` port, or
  both.

