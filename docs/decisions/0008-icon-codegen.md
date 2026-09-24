# ADR-0008: Icons via Solar/Iconify codegen

- **Status:** Accepted
- **Date:** 2026-07-05 (backfilled)
- **Deciders:** thinvin

## Context

The app needs a large, visually consistent icon set. Pulling icons at runtime from a font
or remote source adds weight and dependencies. We wanted local, tree-shakeable icon
components with one visual style (Solar bold-duotone).

## Decision

Generate icon components at build time from the Solar set using Iconify tooling
(`@iconify-json/solar`, `@iconify/utils`), via `npm run generate:icons`. Hand-written SVG
icons that are not from Solar are suffixed `_custom` to keep them distinct from generated
ones.

## Consequences

- Icons are local, consistent, and tree-shakeable.
- The Solar set is **CC BY 4.0**, which requires visible attribution in the app. This is
  an outstanding go-live obligation, tracked in [risks.md](../risks.md) R-11.
- Adding an icon means regenerating, not hand-copying SVG, except for intentional
  `_custom` icons.
