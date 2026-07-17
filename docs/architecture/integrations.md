# External Integrations and Blast Radius

_Living doc. Every third-party service, what it does, where its key lives, and what
breaks if it goes down. Last updated: 2026-07-05._

| Service | Purpose | Key location | If it goes down |
| --- | --- | --- | --- |
| **Supabase** | Postgres, Auth, Storage, Edge Functions | Project secrets + publishable key | The app is down. Single point of failure. |
| **Duffel** | Flight and stay search, offers, seat maps, checkout, order webhooks | Supabase secret `DUFFEL_API_KEY` (server-side only) | No flight search or booking. |
| **LiteAPI** | Hotel search | `EXPO_PUBLIC_LITEAPI_KEY` | No hotel search. |
| **Resend** | Transactional email (welcome, invites, notifications) | Edge Function secret | No emails; bookings still work. |
| **Expo Push** | Device push notifications | Expo credentials | No push; email fallback for some flows. |
| **PostHog** | Product analytics and funnels | `EXPO_PUBLIC_POSTHOG_API_KEY` | No analytics; app unaffected. |
| **Google OAuth** | Sign-in | Client config in `google-services.json` | New and returning sign-ins fail. |
| **Anthropic API** | Content and image curation (tooling only) | `ANTHROPIC_API_KEY` (dev/scripts) | Content pipeline stalls; runtime app unaffected. |
| **Pixabay / Openverse / Pexels** | Image sourcing for the curation pipeline | Script env | Curation degraded; runtime app unaffected. |
| **GitHub Pages** | Serves legal pages and OAuth callback relay from repo root | n/a | Legal pages and OAuth deep-link relay unavailable. |

## Edge Functions (22), by domain

- **Flights / stays (Duffel):** `duffel-search`, `duffel-offer`, `duffel-seat-maps`,
  `duffel-checkout`, `duffel-stays`, `duffel-webhook`.
- **Hotels:** `liteapi-stays`.
- **Notifications / email:** `send-notification`, `welcome-email`, `friend-notify`,
  `together-notify`, `send-match-notification`, `send-group-invite`,
  `send-group-vibes-ready`, `nudge-participant`, `send-curation-email`.
- **Groups (Wander Together):** `group-checkout`, `validate-invite`.
- **Affiliates:** `create-affiliate`, `get-affiliate-stats`, `track-click`.
- **Admin:** `admin`.

## Notes

- **No second region and no staging backend.** Supabase is a single point of failure.
  A reviewer will ask about backups and disaster recovery; document the Supabase backup
  posture here once confirmed. _TODO._
- Vendor cost per service is tracked in `Wander_App_Services_Cost_Report.xlsx`. Summarize
  monthly totals into [STATE.md](../STATE.md).
