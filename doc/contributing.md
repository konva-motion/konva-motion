# Contributing

## Setup

Requires Node 20+ and pnpm (the repo pins `pnpm@10.33.4` via
`packageManager`, so [Corepack](https://nodejs.org/api/corepack.html) will pick
the right version automatically).

```sh
pnpm install
```

## Common commands

| Command                   | What it does                                       |
| ------------------------- | -------------------------------------------------- |
| `pnpm dev`                | Run the Vite demo at http://localhost:5173         |
| `pnpm build`              | Build all `packages/*` (tsc to `dist/`)            |
| `pnpm check`              | Biome lint + format check across the whole repo    |
| `pnpm format`             | Biome auto-format                                  |
| `pnpm --filter @smoove/timeline dev` | tsc watch for one package         |

## Layout

- `packages/core` — engine. `Composition` (extends `Konva.Stage`) +
  `Sequence` (extends `Konva.Layer`). `konva` is a peer dep.
- `packages/timeline` — planned home for React UI components (scrubber, play
  button). Currently a placeholder.
- `demo/` — Vite app that imports the packages via `workspace:*`.
- `doc/` — short design + usage docs.

When you change a package, the demo picks it up automatically thanks to pnpm
workspace symlinks — but you do need to `pnpm build` (or run `tsc -b --watch`
in the package) for the emitted `dist/` to update, since the demo imports the
package's published entry, not its source.
