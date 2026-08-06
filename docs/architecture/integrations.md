# External Integrations and Blast Radius

_Living doc. Every third-party service, what it does, where its key lives, and what
breaks if it goes down. Last updated: 2026-08-01._

| Service | Purpose | Key location | If it goes down |
| --- | --- | --- | --- |
| **Supabase** | Postgres, Auth, Storage, Edge Functions | Project secrets + publishable key | The app is down. Single point of failure. |
| **LiteAPI (Nuitee Connect)** | Flight and hotel search, offers, seat maps, prebook, checkout, traveller payment (hosted widget) | Supabase secrets (server-side only) | No flight or hotel search/booking. Primary travel provider — see [GO_LIVE_LITEAPI.md](../GO_LIVE_LITEAPI.md) for what is proven vs. still blocked (real-airline ticketing is blocked on Nuitee's side, not ours). |
| **Duffel** | Legacy flight/stay provider, retired for solo bookings. Still the only path for group checkout (`group-checkout`) pending a payment-splitting redesign for LiteAPI | Supabase secret `DUFFEL_API_KEY` (server-side only) | No group-trip booking. Solo flight/hotel booking is unaffected (runs on LiteAPI). |
| **Resend** | Transactional email (welcome, booking confirmations, share-by-email) + bounce/complaint webhook | Edge Function secret | No emails; bookings still work. |
| **Expo Push** | Device push notifications | Expo credentials | No push; email fallback for some flows. |
| **PostHog** | Product analytics and funnels | `EXPO_PUBLIC_POSTHOG_API_KEY` | No analytics; app unaffected. |
| **Google OAuth** | Sign-in | Client config in `google-services.json` | New and returning sign-ins fail. |
| **Anthropic API** | Content and image curation (tooling only) | `ANTHROPIC_API_KEY` (dev/scripts) | Content pipeline stalls; runtime app unaffected. |
| **Pixabay / Openverse / Pexels** | Image sourcing for the curation pipeline | Script env | Curation degraded; runtime app unaffected. |
| **GitHub Pages** | Serves legal pages and OAuth callback relay from repo root | n/a | Legal pages and OAuth deep-link relay unavailable. |

## Edge Functions, by domain

- **Flights / hotels (LiteAPI, active):** `liteapi-flights`, `liteapi-book`,
  `liteapi-stays`, `liteapi-data`, `liteapi-webhook`.
- **Flights / stays (Duffel, retired except group checkout):** `duffel-search`,
  `duffel-offer`, `duffel-seat-maps`, `duffel-checkout`, `duffel-stays`, `duffel-webhook`.
- **Groups (Wander Together):** `group-checkout` (still Duffel-backed, see risk R-15),
  `validate-invite`.
- **Notifications / email:** `send-notification`, `welcome-email`, `friend-notify`,
  `together-notify`, `send-match-notification`, `send-group-invite`,
  `send-group-vibes-ready`, `nudge-participant`, `send-curation-email`, `share-email`,
  `resend-webhook`, `test-email-preview`.
- **Affiliates:** `create-affiliate`, `get-affiliate-stats`, `track-click`.
- **Content tooling:** `curate-image`, `upload-stock-image`, `capture-submission`.
- **Admin:** `admin`.

## Notes

- **No second region and no staging backend.** Supabase is a single point of failure.
  A reviewer will ask about backups and disaster recovery; document the Supabase backup
  posture here once confirmed. _TODO._
- Vendor cost per service is tracked in `Wander_App_Services_Cost_Report.xlsx`. Summarize
  monthly totals into [STATE.md](../STATE.md).
