# Current State Snapshot

_The one-page answer to "what is this system right now?" Refreshed by the
weekly agent. Last updated: 2026-08-01._

## Product

WhereTo (slug `wander`, package `com.vcinnovationsgroup.whereto`) is a mobile travel
discovery and booking app. Users discover destinations by "vibe," search flights and
hotels, book, and plan trips solo or as a group ("Wander Together"). Parent company:
VC Innovations Group. Marketing site: wheretotrips.com.

## Version and stage

- App version **0.3.305**, versionCode **512**.
- Stage: **beta** (schema `0.x.0`). Pre-launch. Android-first.
- Distribution: EAS Build, Google Play internal track. iOS not yet built.

## Stack in one breath

Expo SDK 56, React Native 0.85.3, React 19, expo-router (file-based, typed routes,
React Compiler on), TypeScript. State via Zustand. Backend is Supabase (Postgres, Auth,
Storage, Edge Functions). Flights and hotels both via LiteAPI (Duffel retired except
group checkout, see [ADR-0009](decisions/0009-liteapi-primary-provider.md)), behind
swappable provider interfaces. Analytics via PostHog. Email via Resend. Push via Expo.
Content and image curation tooling uses the Anthropic API.

## What works today

- Discovery by vibe (Vibe Engine: curated blogs + 0-100 destination rankings, 31 of 32
  sub-regions complete across 6 regions — see [vibes-engine/README.md](vibes-engine/README.md)).
  Feeds an internal review page today; not yet imported into Supabase or wired into the
  app's live search/recommendation scoring.
- WhereTo: Discover, a guided walkthrough search (mood, occasion, vibes, global ranked
  reveal with refiners). See `docs/DISCOVER_PROGRESS.md`.
- Six-tile 2x3 dashboard (Supabase-configured cards, rotating photo tiles).
- Flight and hotel search and booking, including traveller payment, via LiteAPI
  (server-side). See [GO_LIVE_LITEAPI.md](GO_LIVE_LITEAPI.md) for what is proven vs.
  still blocked.
- Auth (Google OAuth via expo-auth-session, SecureStore session).
- Booking Hub (`app/bookings/passengers.tsx`) with orders persisted server-side.
- Notifications and transactional email (Resend dashboard templates, bounce/complaint
  tracking, share-by-email).
- Wander Together (group trips): lobby, invites, roles, per-member vibes. Group checkout
  still runs on Duffel pending a payment-splitting redesign for LiteAPI (risk R-15).
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
