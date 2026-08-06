# ADR-0007: PostHog for analytics; error monitoring deferred

- **Status:** Accepted
- **Date:** 2026-07-05 (backfilled)
- **Deciders:** thinvin

## Context

We need to understand user behavior and conversion funnels. We also, separately, need to
know when production crashes. Analytics and crash/error monitoring are different tools.
We prioritized analytics first.

## Decision

Adopt PostHog for product analytics and funnels (instrumented, with a "Core Funnels"
dashboard). Defer crash and error monitoring (Sentry was evaluated and shelved).

## Consequences

- We can see behavior and conversion.
- We are **blind to production errors**. There is no aggregated crash reporting, so a
  silent failure in the app or an Edge Function may go unnoticed until a user reports it.
  This is [risks.md](../risks.md) R-04 and a known gap (GAP-REPORT G-04).
- Adding an error monitor (Sentry or equivalent), wired into both the app and Edge
  Functions, is the recommended next observability step before go-live.
