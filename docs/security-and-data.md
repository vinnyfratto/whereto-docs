# Security and Data Handling

_Living doc. Posture and controls only. This repo is public, so no secrets or
exploit-level detail belong here. Last updated: 2026-07-05._

## Secrets management

| Secret | Where it lives | In app bundle? |
| --- | --- | --- |
| Duffel API key | Supabase Edge Function secret | No (server-side only) |
| Resend key | Supabase Edge Function secret | No |
| Supabase service role | Server-side only | No |
| LiteAPI key | `EXPO_PUBLIC_LITEAPI_KEY` | Yes (public-scoped key) |
| PostHog key | `EXPO_PUBLIC_POSTHOG_API_KEY` | Yes (write-only ingest key, expected) |
| Anthropic key | Local `.env` / script env | No (tooling only) |
| Play service account | `google-service-account.json` | No (untracked, local only) |

Controls in place:

- `.env` is gitignored; `credentials.json` and `google-service-account.json` are
  untracked.
- Duffel forbids client-side calls; all Duffel traffic goes through Edge Functions, so
  the payment-capable key never ships to devices. See
  [ADR-0003](decisions/0003-server-side-travel-apis.md).
- Sessions stored in `expo-secure-store` (with 2048-byte chunking for large Google
  tokens).

Open items:

- `google-services.json` (client Firebase config) is committed to a **public** repo.
  Low severity because it is client config, but unnecessary. See GAP-REPORT G-13.
- No secret-scanning in CI (there is no CI). Recommend adding gitleaks or GitHub secret
  scanning, especially while the repo is public.

## Authentication

- Google OAuth via `expo-auth-session`. Callback relayed through a GitHub Pages page for
  deep-link return.
- Sessions persisted in SecureStore.

## Row-Level Security

RLS is enabled on most tables and **intentionally disabled** on the Wander Together
tables due to a recursion bug in the policy. This is a documented decision with
compensating access checks in the Edge Functions, not an oversight. See
[ADR-0004](decisions/0004-rls-disabled-together.md). Re-enabling RLS safely is tracked in
[risks.md](risks.md).

## Personal data (PII)

Data collected: name, email, avatar, saved passenger details (for booking), traveler
details on orders, group membership. This is meaningful PII plus travel data.

- **Legal pages live:** privacy policy, terms of service, and a data-deletion page are
  served from GitHub Pages.
- **GDPR / CCPA:** a deletion path exists. _TODO: document the full data-subject-request
  process and data-retention periods here._
- **Data residency:** Supabase project region. _TODO: confirm and record._

## Payments and PCI

Card capture and payment are handled by **Duffel**, which is the PCI-scoped party. The
app does not store card numbers. Document the exact boundary (what data touches our
systems versus Duffel's) here before go-live, because anyone assessing the payment flows
will scope PCI carefully. _TODO._

## Incident response

No formal process yet. When an incident happens, write a blameless postmortem using
[templates/postmortem-template.md](templates/postmortem-template.md) and file it under
`decisions/` or a `postmortems/` folder. The weekly bug review links any incidents.

## Priority hardening before go-live

See [GAP-REPORT.md](GAP-REPORT.md). Security-relevant items: decide repo visibility
(G-01), add error monitoring (G-04), add secret scanning, re-enable RLS on Together
tables safely, and document PCI boundary and data-retention.
