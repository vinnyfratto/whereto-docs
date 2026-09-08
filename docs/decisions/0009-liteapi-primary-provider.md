# ADR-0009: LiteAPI as the primary provider for both flights and hotels

- **Status:** Accepted
- **Date:** 2026-07-14 (backfilled)
- **Deciders:** thinvin

## Context

Duffel was the original flight provider (ADR-0002, ADR-0003) and hotel search had gone
through several providers (Duffel Stays stalled, Travelport rejected) before landing on
LiteAPI (Nuitee Connect) for hotels. Running two vendors for the two halves of a trip
added integration surface for no real benefit, and LiteAPI could quote and book both
flights and hotels through one relationship, including a hosted-widget payment flow that
makes the traveller (not WhereTo) the payer of record — see
[GO_LIVE_LITEAPI.md](../GO_LIVE_LITEAPI.md) for exactly what that proved out to.

## Decision

LiteAPI becomes the provider for both flights and hotels. New Edge Functions
(`liteapi-flights`, `liteapi-book`, `liteapi-stays`, `liteapi-data`, `liteapi-webhook`)
were built behind the existing provider-abstraction interfaces (ADR-0002), and the old
inline Duffel booking flow in `app/search/results.tsx` was removed once the Booking Hub
(`app/bookings/passengers.tsx`) covered the same flow end to end. Duffel is retired for
solo flight and hotel bookings.

## Consequences

- Solo flight and hotel booking, including traveller payment, run on LiteAPI. Duffel's
  Edge Functions (`duffel-search`, `duffel-offer`, `duffel-seat-maps`, `duffel-checkout`,
  `duffel-stays`, `duffel-webhook`) are dead code for the solo path but still exist.
- **Group checkout (Wander Together) is the one exception.** `group-checkout` still books
  through Duffel because it settles N participants' payments into a single order, and
  LiteAPI's booking model takes one payment per booking. Moving it needs a real design
  decision (for example: charge the organizer in full and reconcile participant shares
  separately), not a mechanical provider swap. Tracked as risk R-15 in
  [risks.md](../risks.md).
- Real-airline ticketing through LiteAPI is currently blocked on Nuitee's side (account
  entitlement, not a code issue) — synthetic-carrier bookings and all hotel bookings work
  end to end today. See [GO_LIVE_LITEAPI.md](../GO_LIVE_LITEAPI.md) section 3.
- The still-open item from ADR-0003 ("revisit whether LiteAPI should also move fully
  server-side") is resolved: `EXPO_PUBLIC_LITEAPI_KEY` is read only inside Edge Functions,
  never in `src/`, despite the `EXPO_PUBLIC_` prefix in its name.
