# Vibe Engine — The Hawaii Sub-Region Anomaly

_Living doc. Last updated: 2026-08-06. Status: **OPEN**, repo and database disagree._

Hawaii is the only place in the Vibe Engine where the content repo and the live database
hold two different, mutually exclusive answers to the question "which sub-region is this
destination in". This doc records what the split is, how it happened, everything it
touches, and what closing it actually costs. See [README.md](README.md) for how the
pipeline works normally.

## The short version

The `destinations` table has always filed Hawaii as United States: `region NA`,
`subregion north_america`, the same as Portland or Denver. The Vibe Engine scored it
somewhere else, under `Oceania / Pacific Islands`, alongside Fiji and French Polynesia.

On 2026-07-31, commit `5786f7c` moved all 57 Hawaii ranking rows out of Pacific Islands
and into US Pacific **in the markdown**. That change was never seeded to Supabase. The
database still holds the pre-merge arrangement. Both states are internally consistent,
which is why nothing has visibly broken, and also why it is easy to miss.

## Current state, by the numbers

| | Repo (`Content/VibesEngine/`) | Database (`vibe_destination_rankings`) |
| --- | --- | --- |
| US Pacific vibe sections | 45 | 26 |
| US Pacific rows | 265 (57 Hawaii) | 208 (0 Hawaii) |
| Pacific Islands vibe sections | 22 | 22 |
| Pacific Islands rows | 228 (0 Hawaii) | 285 (57 Hawaii) |

The 57 Hawaii rows cover 6 airports (OGG Maui, HNL Honolulu, KOA Kona, LIH Kauai, ITO
Hilo, LNY Lanai) across 21 vibe keys.

## How it happened

The stock-image ingest script walks the Envato folder tree, which is organised the way
the `destinations` table is, so it looked for Hawaii under US Pacific and reported those
destinations as uncovered. The fix chosen was to move the rankings to match the folders,
which is the right call: the app files Hawaii as US, so the engine should too. What did
not follow was the rest of the move.

## What a move actually touches

The rankings are the smallest part. Five surfaces depend on the sub-region string.

### 1. Blogs are the real cost

`fetchDestinationBlogs()` in `src/lib/vibeBlogs.ts` reads the sub-region off the
destination's highest-scoring ranking row, then requires a `vibe_blogs` row at that same
`subregion` + `vibe_key`. Seed the repo's rankings as they stand today and Hawaii's
Explore screen goes blank, because 19 of its 21 vibe keys have no US Pacific blog:

`ancient_sites_archaeology`, `eco_lodges_sustainable`, `family_resorts`, `hiking_trekking`,
`honolulu_waikiki`, `iconic_beaches`, `island_cuisine_markets`, `lagoon_motu_cruises`,
`polynesian_culture`, `private_island_resorts`, `romance_honeymoon`,
`sailing_yacht_charters`, `scuba_diving`, `snorkeling_reefs`, `sport_fishing`,
`volcanoes_craters`, `waterfalls_rainforest`, `whale_swimming`, `wwii_history`

Only `surfing` and `spa_wellness_retreats` exist on both sides, because those two US
Pacific sections already existed and the Hawaii rows were merged into them.

Those 19 blogs cannot be copied across as-is. They are written from a Pacific-wide point
of view, and several rank Bora Bora, Fiji, or Tonga above Hawaii in their own in-body
snapshot tables. Dropped into US Pacific they would read as though the reader had opened
the wrong page.

### 2. Pacific Islands blogs still describe a Hawaii that has left

All 22 Pacific Islands blogs mention Hawaii. In 16 of them it is heavy, 11 or more
mentions, and it appears in the frontmatter `top_destinations`, in the prose, and in the
Destination Ranking Snapshot tables. The repo's master file no longer scores any of it.
Until those blogs are revised, the reader-facing text and the ranking data contradict
each other.

### 3. One vibe key is now a misnomer

`honolulu_waikiki` still exists as a Pacific Islands vibe, but the merge removed Honolulu
from it. Its seven remaining rows are Noumea, Papeete, Tumon, Suva, Apia, Port Vila, and
Pago Pago. The vibe is now "island capital towns" wearing a Honolulu label, and its blog
is framed entirely around Waikiki. It needs a rename plus a rewrite, or it needs to move
to US Pacific whole and be replaced in Pacific Islands with a properly named equivalent.

### 4. Intros sit in the wrong folder

Six Hawaii intros are still under `Content/VibesEngine/Oceania/PacificIslands/intros/`:
`honolulu.md`, `maui.md`, `kauai.md`, `kona.md`, `hilo.md`, `lanai.md`. This one is
cosmetic for the app, because `seed-destination-intros.js` matches on IATA plus city and
does not care which folder the file came from, but leaving them there hides the move from
anyone reading the tree.

### 5. Sub-region size after the move

Pacific Islands drops to 228 rows across 22 vibes, and thins out at the bottom:
`volcanoes_craters` falls to 5 rows, `surfing` to 6, `family_resorts` and
`spa_wellness_retreats` to 7. Those are workable but no longer comfortable. US Pacific
grows to 45 vibes, above the 15 to 40 band the engine spec sets, so some consolidation is
likely warranted.

## What is not affected

- `destinations.region` / `subregion` are already United States. Nothing to change.
- The in-app vibe gate reads `destinations.region` / `subregion`, so Hawaii's allowed
  regional vibe keys are the `NA` / `NA/north_america` set either way. The move does not
  change which tags Hawaii can carry.
- `destination_image_edits` keys on the destination id, not the sub-region.
- `vibe_image_edits` has no rows for either sub-region, so there is nothing to re-point.

## Definition of done

1. The repo and the database agree on one sub-region for all 57 rows.
2. Every Hawaii vibe key has a blog at its own sub-region, so Explore renders for all six
   airports.
3. No blog's prose or snapshot table contradicts its own master ranking file.
4. No vibe key names a destination that is no longer scored under it.
5. `docs/vibes-engine/regions/oceania.md` and `regions/north-and-central-america.md`
   carry the corrected counts and no longer describe Pacific Islands as covering Hawaii.
6. This doc is updated to Status: RESOLVED, with what was done and when.

## Known stale text

`docs/vibes-engine/regions/oceania.md` still reads "this one covers Hawaii, French
Polynesia, Fiji, and the wider tropical Pacific island nations", and gives Pacific Islands
240 scored rows. Neither the repo (228) nor the database (285) matches that number. It was
last touched the day after the merge commit and did not pick the change up.
