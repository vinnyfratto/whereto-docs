# ADR-0006: Skins design-token system, no inline styles

- **Status:** Accepted
- **Date:** 2026-07-05 (backfilled)
- **Deciders:** thinvin

## Context

Consistent visual design across a growing number of screens is hard if colors, fonts, and
spacing are written inline in each component. We also want to be able to restyle the app
(themes, brand changes) without editing every component.

## Decision

All style tokens (colors, typography, spacing) live in `src/Skins/`. Components read from
Skins and must never define inline colors or fonts. This is enforced by convention and
called out in the project rules.

## Consequences

- A brand or theme change is a change in one place.
- Visual consistency is structural, not a matter of discipline per component.
- Contributors must know to reach for Skins; a component with an inline hex color is a
  defect, not a style choice.
