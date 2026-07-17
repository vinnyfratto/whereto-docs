# Risk and Tech-Debt Register

_Living doc. Append-mostly. The weekly agent adds new debt it detects and marks items
resolved. Last updated: 2026-07-14._

Each item: what it is, why it matters, and the plan. Severity mirrors
[GAP-REPORT.md](GAP-REPORT.md).

| ID | Risk / debt | Sev | Status | Plan |
| --- | --- | --- | --- | --- |
| R-01 | Repo is public with full product source | P1 | Open | Decide private vs public (founder call). |
| R-02 | Stock MIT LICENSE (Expo) on a public repo | P0 | Open | Replace with proprietary "All Rights Reserved" notice. |
| R-03 | No automated tests | P1 | Open | Start with checkout, auth, provider layer. |
| R-04 | No production error monitoring | P1 | Open | Add Sentry or equivalent; wire into Edge Functions and app. |
| R-05 | No CI/CD | P1 | Open | Add GitHub Actions once tests exist. |
| R-06 | Loose SQL, no migration ordering | P1 | Open | Adopt `supabase/migrations/` timestamped convention. |
| R-07 | RLS disabled on Wander Together tables | P1 | Accepted (documented) | Re-enable safely by fixing the recursive policy. See ADR-0004. |
| R-08 | Single maintainer / bus factor of one | P1 | Open | This docs folder mitigates; second engineer is the real fix. |
| R-09 | Shared backend for dev and prod (no staging) | P1 | Open | Stand up a staging Supabase project before go-live. |
| R-10 | No git tags for releases | P2 | Open | Tag at each version bump. |
| R-11 | Solar icon CC BY 4.0 attribution not yet in app | P2 | Open | Add attribution screen before go-live. |
| R-12 | Duffel key was previously a go-live blocker | P1 | Resolved | Now server-side in Edge Functions. |
| R-13 | Boilerplate README, no SBOM automation | P2 | Open | Rewrite README; automate license-checker in weekly job. |
| R-14 | LiteAPI client-side 5 req/s throttle (sandbox only) | P2 | Open (go-live blocker to remove) | Sandbox key caps at ~5 req/s; `callEdgeFunction` spaces `liteapi-*` calls to avoid 429. Production is 500 req/s — raise or remove `LITEAPI_SANDBOX_RPS` in `src/utils/edgeApi.ts` when on a production LiteAPI key. |

## How to use this file

- Add a row when you knowingly take a shortcut. Debt you record reads as maturity; debt
  someone else discovers reads as risk.
- Move items to Resolved with the date rather than deleting, so the history is visible.
- The weekly bug review references this register when a regression maps to known debt.
