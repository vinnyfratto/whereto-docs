# Engineering Gap Report

_Prepared: 2026-07-05 by automated repo audit. Re-run quarterly._

This scores the current documentation and engineering posture against what an external
technical reviewer expects to find in a mature codebase. Each gap is rated by how badly
it would undermine confidence in the code, and how expensive it is to fix later versus
now.

App version at audit: **0.3.11 (versionCode 218)**. Stack: Expo SDK 56 / React Native
0.85.3 / Supabase. Single maintainer. Master-only workflow. Public repo.

## Severity legend

- **P0 blocker** would block a technical review, or is a live legal/security exposure.
- **P1 major** a review will flag it and ask for remediation.
- **P2 minor** easy win, looks unprofessional if missing.

## Gaps, ranked

| ID | Gap | Sev | Cost to fix now | Cost to fix later |
| --- | --- | --- | --- | --- |
| G-01 | **Repo is public** with full source, on a commercial pre-launch product | P1 | Minutes (flip to private) | High (competitors have had it) |
| G-02 | **Stock MIT `LICENSE`** attributed to "650 Industries (Expo)" sits at repo root on a public repo, implying the whole product may be MIT-licensed | P0 | Minutes (replace with proprietary notice) | Very high (IP ownership dispute) |
| G-03 | **No automated tests** (no test runner in `package.json`, README still says "if you'd like to set up testing") | P1 | Weeks (ongoing) | Very high (unverified code erodes confidence) |
| G-04 | **No error monitoring** in production (Sentry shelved; only PostHog analytics) | P1 | Days | High (you are blind to prod crashes) |
| G-05 | **No CI/CD** (no `.github/`, no pipeline); builds and submits are manual | P1 | Days | Medium |
| G-06 | **No migration tooling**: ~34 loose `.sql` files in `supabase/`, no ordering, no down-migrations, no applied-state record | P1 | Days | High (no one can reliably reproduce the DB) |
| G-07 | **No decision log (ADRs)**: rationale for key choices lives only in the founder's head and in Claude memory | P1 | Hours (backfill) | Very high (unrecoverable in year 3) |
| G-08 | **No SBOM / license inventory**: dependencies and their licenses are not catalogued; known open items (Solar icons CC BY 4.0 attribution, Google fonts) are untracked | P1 | Hours | High (license contamination is a serious legal risk) |
| G-09 | **No formal changelog and no git tags**: release history is only in commit messages | P2 | Hours | Medium |
| G-10 | **RLS intentionally disabled** on Wander Together tables is undocumented, so it reads as a vulnerability rather than a decision | P1 | Minutes (ADR) | High (looks like a breach waiting to happen) |
| G-11 | **Architecture and data model undocumented**: no diagrams, no ERD | P2 | Hours | Medium |
| G-12 | **Boilerplate README**: still the default create-expo-app text, tells a new reader nothing about the product | P2 | Minutes | Low but embarrassing |
| G-13 | **`google-services.json` committed** (client Firebase config). Low severity by itself, but on a public repo it is unnecessary exposure | P2 | Minutes | Low |
| G-14 | **Single maintainer / bus factor of one**: no second person understands the system | P1 | Structural | Very high (key-person risk priced in) |
| G-15 | **iOS not built or submitted**: EAS config is Android-only, so "cross-platform" is not yet true | P2 | Product work | Informational |

## What is already in good shape

- **Secret hygiene**: `.env` is gitignored, `credentials.json` and the Play service
  account are untracked, and the Duffel API key runs server-side in Edge Functions
  (the app bundle never holds it). This resolves the old "move Duffel key server-side"
  blocker.
- **Provider abstraction**: flights and hotels sit behind swappable provider
  interfaces (`src/providers/`), so a vendor change does not touch the UI. This is a
  genuine architectural asset and reads well under technical review.
- **Legal pages exist**: privacy, terms, and data-deletion pages are live.
- **Versioning discipline**: SemVer plus `versionCode` on every change, enforced by a
  standing rule. `eas.json` uses `appVersionSource: local`.
- **Cost tracking started**: `Wander_App_Services_Cost_Report.xlsx` exists (feeds unit
  economics).

## Recommended order of remediation

1. **This week (minutes each):** G-02 (fix LICENSE), G-12 (real README), decide G-01
   (private vs public).
2. **This scaffold (done in this pass):** G-07 (backfill ADRs), G-10 (RLS ADR), G-09
   (seed changelog), G-11 (architecture + data-model docs), G-08 (start SBOM section in
   tech-stack).
3. **Next few weeks:** G-04 (add Sentry or equivalent), G-06 (adopt Supabase migration
   ordering), G-03 (start a test suite on the highest-risk paths: checkout, auth; plan in
   [PRELAUNCH_TESTING.md](PRELAUNCH_TESTING.md)).
4. **Structural, ongoing:** G-14 (document enough that a second engineer could onboard;
   this is what the whole `docs/` folder is for), G-05 (add CI once tests exist).

_Living scores: update this table as items close. The weekly agent references it._
