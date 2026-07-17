# ADR-0002: Provider abstraction for flights and hotels

- **Status:** Accepted
- **Date:** 2026-07-05 (backfilled)
- **Deciders:** thinvin

## Context

Travel supply APIs (flights, hotels) are interchangeable in principle but very different
in shape, pricing, and reliability. We started on Priceline (via RapidAPI), moved flights
to Duffel, and use LiteAPI for hotels. We did not want a vendor change to ripple through
the UI, and we wanted the freedom to switch or add suppliers as commercial terms change.

## Decision

Access all flight and hotel supply through provider interfaces in `src/providers/`. Each
vendor is a class implementing a shared interface (`DuffelProvider`, `LiteApiProvider`,
legacy `PricelineProvider`). Screens depend on the interface and the `index.ts` selector,
never on a specific vendor. Provider concerns like caching, dedup, throttle, and prefetch
live in the abstraction.

## Consequences

- Swapping or adding a supplier is a provider change, not a UI change.
- This is a genuine asset for maintainability: it shows the supply layer is not
  hard-wired, which keeps a future vendor change low-risk.
- Small ongoing cost: every new supplier must be adapted to the shared interface, and the
  interface must stay vendor-neutral.
