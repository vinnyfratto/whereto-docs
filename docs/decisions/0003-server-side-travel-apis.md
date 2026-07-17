# ADR-0003: Third-party travel APIs run server-side in Edge Functions

- **Status:** Accepted
- **Date:** 2026-07-05 (backfilled)
- **Deciders:** thinvin

## Context

Duffel forbids client-side API calls and issues payment-capable keys. Shipping such a key
in a mobile bundle would expose it to anyone who inspects the app, and would violate
Duffel's terms. Earlier in the project this "move the Duffel key server-side" item was a
go-live blocker.

## Decision

All Duffel traffic (search, offers, seat maps, checkout, stays, webhooks) goes through
Supabase Edge Functions that hold the key as a server-side secret
(`supabase secrets set DUFFEL_API_KEY=...`). The app bundle never carries the Duffel key.
The same pattern applies to other sensitive keys (Resend, service role).

## Consequences

- The payment-capable key is never on a device. This closes the old go-live blocker.
- Every travel operation is one network hop further (device to Edge Function to vendor),
  which the provider layer's caching and dedup help absorb.
- Edge Functions become security-critical surface and must be reviewed with that in mind.
- LiteAPI currently uses a client-scoped `EXPO_PUBLIC_LITEAPI_KEY`; revisit whether it
  should also move fully server-side.
