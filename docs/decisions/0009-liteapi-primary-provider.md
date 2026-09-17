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

> **Update 2026-09-17. The retirement is complete and there is no exception left.** The
> `duffel-*` Edge Functions, `DuffelProvider.ts`, the wizard pieces that only it used
> (`PassengerWizard`, `ReviewConfirmSheet`, `ExtrasSheet`, `SeatMapView`, `offerExtras`)
> and the `orders` table are all deleted, so the "swap two lines to roll back" fallback
> no longer exists. Group checkout was settled by deleting the split-pay route rather
> than porting it: it marked members "paid" without charging anything, so every trip now
> books the same LiteAPI path through `useGroupPackageBooking` into the Booking Hub.
> Bookings live in `hotel_orders` and `flight_orders`. The LiteAPI key is now the
> `LITEAPI_KEY` secret, without the misleading `EXPO_PUBLIC_` prefix.

## Consequences

- Flight and hotel booking, including traveller payment, run on LiteAPI, for group trips
  as well as solo ones.
- Real-airline ticketing through LiteAPI is currently blocked on Nuitee's side (account
  entitlement, not a code issue) — synthetic-carrier bookings and all hotel bookings work
  end to end today. See [GO_LIVE_LITEAPI.md](../GO_LIVE_LITEAPI.md) section 3.
- The still-open item from ADR-0003 ("revisit whether LiteAPI should also move fully
  server-side") is resolved: `EXPO_PUBLIC_LITEAPI_KEY` is read only inside Edge Functions,
  never in `src/`, despite the `EXPO_PUBLIC_` prefix in its name.
