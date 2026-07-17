# Architecture Overview

_Living doc. Last updated: 2026-07-05._

## The shape of the system

```
  Mobile app (Expo / React Native)
        |
        |  expo-router screens in app/
        |  UI styled from src/Skins/
        |  state in Zustand
        |
        |  provider interfaces (src/providers/)
        |   - flights: Duffel (active), Priceline (legacy)
        |   - hotels:  LiteAPI (active)
        |
        v
  Supabase  ------------------------------------------
        |  Postgres (app data, orders, groups, vibes)
        |  Auth (Google OAuth, sessions)
        |  Storage (avatars, images)
        |  22 Edge Functions (Deno/TS)  <----- hold all third-party API keys
        |
        +--> Duffel   (flights, stays, seat maps, checkout, webhooks)
        +--> LiteAPI  (hotel stays)
        +--> Resend   (transactional email)
        +--> Expo Push (device notifications)
        |
  PostHog (product analytics, client + funnels)
  Anthropic API (content and image curation, tooling only, not in the app runtime)
```

## Key architectural decisions

The reasoning behind each of these lives in [decisions/](../decisions/). The load-bearing ones:

- **Provider abstraction** ([ADR-0002](../decisions/0002-provider-abstraction.md)):
  flights and hotels are accessed through interfaces, so a vendor swap does not touch
  screens.
- **Third-party calls run server-side** ([ADR-0003](../decisions/0003-server-side-travel-apis.md)):
  Edge Functions hold the keys. The app bundle never carries a Duffel or LiteAPI secret.
- **Supabase as the single backend** ([ADR-0001](../decisions/0001-supabase-backend.md)):
  Postgres, Auth, Storage, and Edge Functions from one vendor.
- **Skins design tokens** ([ADR-0006](../decisions/0006-skins-design-tokens.md)): no
  inline colors or fonts in components.

## Client structure

- `app/` expo-router screens (tabs: index, saved, search, profile; nested stacks).
- Search entry points (dashboard tiles): WhereTo: Discover (guided walkthrough, see
  [DISCOVER_PROGRESS.md](../DISCOVER_PROGRESS.md)), Wander with Vibes (traditional
  form), Wander Together, Wander as a Group, Conversational, and WhereTo: Direct
  (stub). All but Conversational rank through the VibeMatch engine.
- `src/Skins/` all style tokens (colors, type, spacing).
- `src/providers/` flight and hotel provider interfaces plus PostHog provider.
- `src/components/` shared UI.
- `scripts/` content and data tooling (icon codegen, destination seeding, image curation).
- `server.js` optional local Express dev server (`@expo/ngrok` for tunneling). Not part
  of production; production talks to Supabase directly.

## Backend structure

- `supabase/functions/` 22 Edge Functions. Grouped by domain in
  [integrations.md](integrations.md).
- `supabase/*.sql` schema and feature migrations (loose files, see
  [data-model.md](data-model.md) and GAP-REPORT G-06).

## Environments

- **Dev:** local Expo dev client against the live Supabase project.
- **Prod:** EAS build against the same Supabase project. There is currently **no staging
  environment**; dev and prod share one backend. Note this for the
  eventual go-live hardening.

## Diagrams

_TODO: add a C4 container diagram (PNG or Mermaid) here. The ASCII sketch above is the
interim version._
