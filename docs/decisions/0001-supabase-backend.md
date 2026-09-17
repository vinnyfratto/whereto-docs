# ADR-0001: Supabase as the single backend

- **Status:** Accepted
- **Date:** 2026-07-05 (backfilled)
- **Deciders:** thinvin

## Context

The app needs a database, authentication, file storage, and somewhere to run
server-side logic that holds third-party API keys. As a solo team we wanted one managed
vendor rather than stitching together separate database, auth, storage, and function
providers.

## Decision

Use Supabase for everything server-side: Postgres for data, Supabase Auth for sign-in,
Supabase Storage for images and avatars, and Supabase Edge Functions (Deno/TypeScript)
for server logic and secret-holding.

## Consequences

- One vendor, one dashboard, one bill. Fast to build with.
- Supabase is a **single point of failure**. If it is down, the app is down. There is no
  second region and currently no separate staging project (dev and prod share one).
- Postgres gives us real SQL and row-level security, which matters for a data-heavy,
  multi-user product.
- Lock-in is moderate: Postgres is portable, but Auth, Storage, and Edge Functions would
  need rework to move off Supabase.
