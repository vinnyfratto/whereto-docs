# CI/CD and Releases

_Living doc. Last updated: 2026-07-05._

## Current reality

There is **no CI/CD pipeline**. There is no `.github/` directory and no automated build,
test, or deploy. Builds and submissions are run manually from the developer machine with
EAS. This is honest to record; a reviewer will ask, and "manual" is a fine answer for a
solo pre-launch product as long as it is documented.

## How a release happens today

1. Make changes on `master` (single-branch workflow, no PRs).
2. Increment `version` and `versionCode` in `app.json` (standing rule; see
   [ADR-0005](../decisions/0005-versioning.md)).
3. Add a `CHANGELOG.md` entry.
4. Build with EAS: `eas build --profile production --platform android`.
5. Submit: `eas submit --profile production --platform android` to the Google Play
   internal track.

Build profiles (`eas.json`): `development` (APK + dev client), `preview` (internal
app-bundle), `preview-apk`, `production` (app-bundle). `appVersionSource: local`.

## Edge Function deploys

Edge Functions are deployed to Supabase separately (via the Supabase CLI or MCP), not
through EAS. _TODO: document the exact deploy command and cadence._

## What is missing (and the target state)

- **CI:** once a test suite exists (GAP-REPORT G-03), add a GitHub Actions workflow that
  runs lint and tests on every push. This is the natural first pipeline.
- **Release tags:** the repo has no git tags. Tag each released version (`v0.3.11`) so
  the changelog and store builds line up. The weekly agent can create the tag at a
  version bump.
- **Staging:** there is one shared Supabase backend for dev and prod. A staging project
  is the pre-go-live target.
- **Automated Edge Function deploy:** currently manual.

## Rollback

There is no automated rollback. For the app, roll back by submitting a previous build.
For the backend, Edge Functions and schema changes are manual, so rollback is manual and
should be rehearsed. Record any real rollback in the weekly bug review as a regression.
