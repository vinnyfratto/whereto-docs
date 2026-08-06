# Vibe Engine

_Living doc. Last updated: 2026-08-06._

Related: [The Hawaii Sub-Region Anomaly](hawaii-subregion-anomaly.md) for the one place
where the repo and the database disagree, and [Deferred Fixes](deferred-fixes.md) for the
running list of problems a pass found and deliberately left alone.

## What it is

The Vibe Engine is the content and ranking pipeline behind WhereTo's "discover by vibe"
search. For every travel sub-region (roughly a continent broken into bookable corridors,
for example "US Pacific" or "Southeast Asia") it produces two linked outputs:

1. **Reader-facing blog content** — one long-form post per vibe (Reef Snorkeling, Ski
   Resorts, Street Food, and so on) that gives a traveler confidence in the destination
   and in booking it through WhereTo.
2. **A 0-100 destination ranking per vibe** — a scored, tunable dataset that is meant to
   drive booking recommendations: pick "Reef Snorkeling," get back the destinations that
   are actually best for it, in order.

Everything is curated from real travel sources published in the last 24 months. Nothing
is invented and nothing is pulled from the app's in-app vibe list. See
[Where this fits in the app](#where-this-fits-in-the-app) below for how (and how much)
of this is wired into the running product today.

**Do not confuse this with `src/data/vibesCanonical.ts`.** That file is the small,
in-app taxonomy of vibe tags used for filtering and matching in the live app. The Vibe
Engine is a much larger, separate content system that will eventually feed that taxonomy
and the app's recommendation scoring, not a duplicate of it.

## How it's constructed

Each sub-region goes through the same fixed pipeline:

1. **Define the corridor.** Scope is drawn as a traveler would actually book it, not by
   strict geography. A sub-region can flag in a cross-border destination (for example
   Belize's cayes inside the Caribbean pass) when it is genuinely the best expression of
   a vibe, noted inline as a flag rather than silently folded in.
2. **Discover the vibes.** 15-40 distinct, bookable vibes per sub-region (15-25 is the
   sweet spot). Quality over count — a sub-region stops at however many vibes the
   research actually supports. Sub-regions with a major city (NYC, Bangkok, New Orleans)
   get a dedicated multi-vibe "Big City" group instead of one catch-all city vibe.
3. **Write the blogs.** 1,200-2,200 words each, a fixed 8-section structure (Quick Read,
   Why the Sub-Region Delivers This, Where to Go, When to Go, What to Watch Out For, Who
   This Is (and Isn't) For, Destination Ranking Snapshot, Sources), identical frontmatter
   across every blog, and at least 5 dated, deep-linked sources. Plain, well-traveled
   voice; no marketing language, no AI-flourish phrasing, honest about downsides.
4. **Score every destination.** Six weighted dimensions roll up into one 0-100 composite,
   tiered S through E (see [Scoring](#scoring)). Scores are calibrated relative to the
   sub-region, not globally, and the ceiling is deliberately left short of 100 so later
   destinations have room to slot in above the current best.
5. **QA the pass.** A script checks composite math, frontmatter completeness, the 8-section
   structure, word count, and source count before a sub-region is called done, plus a
   manual grep pass for banned phrasing (em dashes, AI antithesis flourishes like "it's
   not X, it's Y").
6. **Publish the review page.** A static page at wheretotrips.com/vibes-engine renders
   every sub-region's vibe cards and destination rankings for internal review.

## Folder structure

```
Content/VibesEngine/
  VibesEngine-Details.md          <- full build spec: methodology, prose rules, QA checklist
  <Region>/
    <SubRegion>/
      README.md                   <- scope, corridor rules, headline stats
      VIBE_RANKING_SYSTEM.md      <- this pass's scoring methodology (isolated, tunable)
      vibe_destination_rankings.md <- master audit table, every scored row, all 6 sub-scores
      RANKINGS_OVERVIEW.md        <- auto-generated leaderboard + cross-vibe coverage
      _PROGRESS.md                <- build log (not a reliable "done" signal on its own;
                                      see the note in each region doc)
      blogs/
        <vibe_key>.md              <- one file per vibe
```

Six regions, each broken into sub-regions: Africa, Asia, Europe, North & Central America,
Oceania, South America. Per-region status and sub-region tables are in
[`regions/`](regions/):

- [Africa](regions/africa.md)
- [Asia](regions/asia.md)
- [Europe](regions/europe.md)
- [North & Central America](regions/north-and-central-america.md)
- [Oceania](regions/oceania.md)
- [South America](regions/south-america.md)

## Scoring

```
score = 0.30 x Signature + 0.25 x Quality + 0.15 x Breadth + 0.15 x Access
      + 0.10 x Reliability + 0.05 x Value
```

| Dimension | Weight | What it measures |
| --- | --- | --- |
| Signature | 30% | How distinctively/authentically the vibe is expressed here |
| Quality | 25% | Condition of the core experience (reef health, food quality, etc.) |
| Breadth | 15% | Number of sites/venues/options within the destination |
| Access | 15% | Flight connectivity from major US/EU hubs |
| Reliability | 10% | Consistency across seasons, weather, operational uptime |
| Value | 5% | Cost-quality ratio |

| Tier | Score | Meaning |
| --- | --- | --- |
| S | 90+ | Best in class for this sub-region |
| A | 78-89 | Excellent, recommended |
| B | 65-77 | Good, worth considering |
| C | 50-64 | Adequate, notable caveats |
| D | 35-49 | Weak fit for this vibe |
| E | <35 | Not recommended |

## Hotel tag

Some vibes are really about the stay, not the activity (Luxury Escape, All-Inclusive
Resort, Spa & Wellness Retreat). Those blogs carry a `hotel_tag` value from a small
controlled vocabulary (`luxury`, `all_inclusive`, `boutique`, `spa_wellness`, `eco_lodge`,
`historic_inn`, `family_resort`, `beach_resort`, `safari_camp`, `ryokan`, `desert_camp`).
The intent is for the app's hotel search to pre-filter or weight results toward the right
property type when a user picks one of these vibes. Activity vibes (snorkeling, hiking,
street food) leave the field empty. This is defined but **not yet wired into hotel search**
— see [Where this fits in the app](#where-this-fits-in-the-app).

## Tooling

| Script | What it does |
| --- | --- |
| `scripts/build-vibes-overview.js [subregionDir]` | Regenerates `RANKINGS_OVERVIEW.md` and runs the QA pass: composite-math check, frontmatter, section structure, word count, source count. |
| `scripts/build-vibes-engine-page.js` | Parses every sub-region and regenerates the static review page. Run from `Wander_App`, then commit + push the output from the `WhereToTrips_Website` repo. |

## Where this fits in the app

Today the Vibe Engine is a **content and scoring pipeline that feeds a separate internal
review page**, not yet a live input to the app's search or recommendation logic. What
exists:

- ~784 scored vibes and roughly 6,300 scored destination rows across 31 completed
  sub-regions, all as local markdown in `Content/VibesEngine/`.
- A public-facing review page (wheretotrips.com/vibes-engine) for browsing the content.
- A defined Supabase import target (`vibe_rankings` keyed by sub-region + vibe + IATA, a
  future `vibe_blogs` table) that has not been built yet.

What is **not** done yet: importing this data into Supabase, wiring vibe rankings into
the app's actual recommendation/search scoring, and connecting `hotel_tag` to hotel
search filtering. The in-app "Wander with Vibes" and "WhereTo: Discover" search flows
run on the smaller `src/data/vibesCanonical.ts` taxonomy today, independent of this
pipeline.

## What is deliberately not published here

`Content/VibesEngine/` (the actual blog text, scores, and per-sub-region methodology
notes) is **not** mirrored to the public `docs/` tree or to wheretotrips.com/techdocs.
That folder is curated product content, not engineering documentation, and it is the
asset the ranking/recommendation feature is meant to be built on. This section of
techdocs documents how the pipeline works and where it stands, not the content itself.
