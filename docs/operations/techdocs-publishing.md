# How TechDocs Publishing Works

_Living doc. Last updated: 2026-08-01._

This describes the pipeline that turns this `docs/` folder into the public page at
wheretotrips.com/techdocs. It spans three repos, so the mechanics are not obvious from
any single one of them.

## The three pieces

```
Wander_App (this repo)              whereto-docs (public)         WhereToTrips_Website
--------------------------          --------------------          --------------------------
docs/, week-in-review/,     --push->  docs/, week-in-review/  <--fetch--  src/_data/techdocs.js
CHANGELOG.md, LICENSE               CHANGELOG.md, LICENSE               (build-time loader)
                                     "the source of truth is                    |
                                      Wander_App, do not edit"                  v
                                                                        src/techdocs.njk (page)
                                                                        wheretotrips.com/techdocs
```

1. **This repo is the source of truth.** Everyone edits `docs/` here, on `master`.
2. **`.github/workflows/docs-mirror.yml`** runs on every push to `master` that touches
   `docs/**`, `week-in-review/**`, `CHANGELOG.md`, or `LICENSE`. It copies exactly those
   paths into a fresh commit and force-pushes them to `vinnyfratto/whereto-docs`, a
   separate **public** repo whose only purpose is to be a stable, public read surface —
   this works even if `Wander_App` itself is private, and it means the techdocs page
   never has to authenticate against this repo.
3. **`WhereToTrips_Website`'s `src/_data/techdocs.js`** is an Eleventy data file. At
   build time it calls the GitHub tree API against `whereto-docs`, reads every included
   `.md` file from the raw CDN, groups them into sections, and hands the result to
   `src/techdocs.njk` to render.

## Section grouping

`techdocs.js` has a `SECTIONS` array: a list of `{ key, title, order, match(path) }`
entries. Every doc file is tested against each entry's `match` in order; the first
match decides which card group it renders under and where that group sits on the page.
Anything that matches nothing lands in a catch-all "Feature References" group at the
bottom.

**To add a new section** (this is what the Vibe Engine section did): add an entry to
`SECTIONS` in `WhereToTrips_Website/src/_data/techdocs.js` that matches a path prefix
under `docs/`, for example:

```js
{ key: "vibes-engine", title: "Vibe Engine", order: 3.5, match: (p) => p.startsWith("docs/vibes-engine/") },
```

`order` controls display position (fractional values are fine for slotting between two
existing sections without renumbering everything else). The doc's title on the page
comes from its first `# H1` line; its date badge comes from a `_Last updated: YYYY-MM-DD_`
line (or a few other patterns `techdocs.js` recognizes — see `dateOf()`).

## Freshness: when the live page actually updates

There is **no direct link** from the docs-mirror step to a website rebuild. The website
picks up new docs on whichever of these happens first:

- **Nightly cron** — `WhereToTrips_Website/.github/workflows/deploy.yml` rebuilds every
  day at 07:17 UTC regardless of whether anything changed.
- **A push to `WhereToTrips_Website`'s `main` branch** — any site change rebuilds and
  picks up whatever is currently in `whereto-docs` at that moment.
- **A manual `workflow_dispatch`** on `deploy.yml` from the Actions tab.
- **A `repository_dispatch` of type `docs-updated`** — `deploy.yml` already listens for
  this, but **nothing currently sends it**. `docs-mirror.yml` does not fire it after a
  successful mirror. This means a docs-only change can take up to a day to appear live
  unless someone also pushes to the website repo or manually dispatches the build. If
  same-day freshness matters, either add a `repository_dispatch` call to the end of
  `docs-mirror.yml`'s job (needs a token with write access to `WhereToTrips_Website`), or
  manually trigger `deploy.yml` after an important docs change.

## Access

`/techdocs/` is gated client-side to an email allowlist in
`WhereToTrips_Website/src/assets/js/wt-techdocs.js` (checked against the Supabase
session, same pattern as the marketing site's dashboard). Edit the `ALLOW` array there to
change who can see the page. This is a UI gate only — the underlying markdown is public
on `whereto-docs` and readable directly (via `raw.githubusercontent.com` or the repo
itself) regardless of the site's login check. See
[GAP-REPORT.md](../GAP-REPORT.md) G-01 and [risks.md](../risks.md) R-01 for the
consequence: nothing sensitive should ever be written into `docs/`.

Also unlisted: `noindex` meta, `robots.txt` disallow, and excluded from the sitemap.

## Local preview

Set `TECHDOCS_LOCAL=<absolute path to a Wander_App checkout>` when running the Eleventy
build in `WhereToTrips_Website` to read `docs/` straight off disk instead of fetching
`whereto-docs`, so a docs change can be previewed before it is pushed anywhere.
