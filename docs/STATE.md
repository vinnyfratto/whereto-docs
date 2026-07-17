# Current State Snapshot

_The one-page answer to "what is this system right now?" Refreshed by the
weekly agent. Last updated: 2026-07-05._

## Product

WhereTo (slug `wander`, package `com.vcinnovationsgroup.whereto`) is a mobile travel
discovery and booking app. Users discover destinations by "vibe," search flights and
hotels, book, and plan trips solo or as a group ("Wander Together"). Parent company:
VC Innovations Group. Marketing site: wheretotrips.com.

## Version and stage

- App version **0.3.37**, versionCode **244**.
- Stage: **beta** (schema `0.x.0`). Pre-launch. Android-first.
- Distribution: EAS Build, Google Play internal track. iOS not yet built.

## Stack in one breath

Expo SDK 56, React Native 0.85.3, React 19, expo-router (file-based, typed routes,
React Compiler on), TypeScript. State via Zustand. Backend is Supabase (Postgres, Auth,
Storage, 22 Edge Functions). Flights via Duffel, hotels via LiteAPI, both behind
swappable provider interfaces. Analytics via PostHog. Email via Resend. Push via Expo.
Content and image curation tooling uses the Anthropic API.

## What works today

- Discovery by vibe (Vibes Engine: curated blogs + rankings across 30-plus sub-regions).
- WhereTo: Discover, a guided walkthrough search (mood, occasion, vibes, global ranked
  reveal with refiners). See `docs/DISCOVER_PROGRESS.md`.
- Six-tile 2x3 dashboard (Supabase-configured cards, rotating photo tiles).
- Flight search and booking (Duffel, server-side).
- Hotel search (LiteAPI).
- Auth (Google OAuth via expo-auth-session, SecureStore session).
- Booking wizard (4-step) with orders persisted and Duffel webhooks.
- Notifications and transactional email.
- Wander Together (group trips): lobby, invites, roles, per-member vibes.
- Affiliate infrastructure (accounts, attribution, dashboards).

## What is missing or in flight

- Automated tests: none yet.
- Production error monitoring: none (Sentry shelved).
- CI/CD: none (manual EAS).
- iOS build and submission.
- See [risks.md](risks.md) and [GAP-REPORT.md](GAP-REPORT.md) for the full list.

## Ownership and hosting

- Code: `github.com/vinnyfratto/Wander_App` (**public**).
- Backend: Supabase project `pvqwxphrmcvmztlkzhsg`.
- Builds: EAS (owner `thinvin`), EAS project `c7690a53-...`.
- Legal pages: GitHub Pages from repo root.

## Key numbers to keep current

- Monthly vendor cost: see `Wander_App_Services_Cost_Report.xlsx`. _TODO: summarize here._
- Active users / funnel: PostHog project 445316, "Core Funnels" dashboard 1745844.
- Test coverage: 0%.
- Open P0/P1 gaps: see GAP-REPORT.md.
