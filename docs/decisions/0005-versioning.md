# ADR-0005: Manual SemVer plus versionCode on every change

- **Status:** Accepted
- **Date:** 2026-07-05 (backfilled)
- **Deciders:** thinvin

## Context

We need every change traceable to a released build, and we submit to the Google Play
internal track where the Android `versionCode` must increase on every upload. `eas.json`
reads the version locally (`appVersionSource: local`).

## Decision

Increment `version` and `versionCode` in `app.json` on **every** change. Version scheme:

- `0.0.x` development
- `0.x.0` beta
- `1.0.0` go-live

## Consequences

- Every build has a unique, ordered identity, and the changelog can line up with store
  builds.
- It is a manual step that is easy to forget; it is part of the definition of done and is
  a candidate for automation later.
- The repo has no git tags yet; tagging each version (`v0.3.11`) would complete the story.
  See [operations/ci-cd-and-releases.md](../operations/ci-cd-and-releases.md).
