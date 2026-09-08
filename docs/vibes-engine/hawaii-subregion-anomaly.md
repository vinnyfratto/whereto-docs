# Vibe Engine — The Hawaii Sub-Region Anomaly

_Living doc. Last updated: 2026-08-06. Status: **RESOLVED**, repo and database agree._

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

## State at the time the anomaly was written

| | Repo (`Content/VibesEngine/`) | Database (`vibe_destination_rankings`) |
| --- | --- | --- |
| US Pacific vibe sections | 45 | 26 |
| US Pacific rows | 265 (57 Hawaii) | 208 (0 Hawaii) |
| Pacific Islands vibe sections | 22 | 22 |
| Pacific Islands rows | 228 (0 Hawaii) | 285 (57 Hawaii) |

Final state after the fix is in [What was done](#what-was-done) below.

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

---

## What was done

Resolved 2026-08-06 across eight working sessions. The history above is left intact,
because why it happened is the part worth keeping.

### Final counts

| | US Pacific | Pacific Islands |
| --- | --- | --- |
| Vibe sections | 39 | 22 |
| Scored rows | 304 | 228 |
| Hawaii rows | 61 | 0 |
| Blogs | 39 | 22 |

US Pacific gained 61 Hawaii rows rather than 57: the original 57 plus four added when
`private_island_resorts` was widened from a single Lanai row.

### The merge, and why 45 sections became 39

Commit `5786f7c` had appended Hawaii's 19 vibes as new sections at the bottom of the US
Pacific master rather than folding them into the mainland vibes that already covered the
same ground. It had done that correctly for two, `surfing` and `spa_wellness_retreats`,
which is exactly why those two were the only ones that worked. The other 19 left the
sub-region holding the same vibe twice in five places, so scoring mainland rows into them
would have produced two beach tiles both containing California and Hawaii.

Five duplicate pairs were merged:

| Removed | Folded into |
| --- | --- |
| `iconic_beaches` | `pacific_beaches` |
| `whale_swimming` | `whale_watching` |
| `hiking_trekking` | `hiking_backpacking` |
| `island_cuisine_markets` | `food_scene_dining` |
| `volcanoes_craters` | `cascade_volcanoes`, renamed `volcanoes_craters` |

`cascade_volcanoes` had to widen and rename because Kilauea is not a Cascade.
`lagoon_motu_cruises` was folded into `sailing_yacht_charters`, which `vibesCanonical.ts`
already treated as the same vibe.

### Blogs

Thirteen new US Pacific blogs written: `scuba_diving`, `snorkeling_reefs`, `sport_fishing`,
`sailing_yacht_charters`, `waterfalls_rainforest`, `polynesian_culture`,
`ancient_sites_archaeology`, `wwii_history`, `honolulu_waikiki`, `private_island_resorts`,
`romance_honeymoon`, `family_resorts`, `eco_lodges_sustainable`.

Five existing US Pacific blogs revised to cover the Hawaii rows merged into them.

All 22 Pacific Islands blogs revised. 657 Hawaii references at the start, 12 at the end,
every one of which is a deliberate comparison or cross-reference documented in that
sub-region's `_PROGRESS.md`.

### What `honolulu_waikiki` became

Split. `honolulu_waikiki` stays in US Pacific, where it is correct. In Pacific Islands it
was renamed **`key_island_stops`**, label "Key Island Stops", because the merge left it
holding seven island capitals and main towns (Noumea, Papeete, Tumon, Suva, Apia, Port
Vila, Pago Pago) and no Honolulu. Its blog was rewritten from scratch. The engine key maps
into the existing canonical vibe `capital_cosmopolitan_cities`, so no new canonical vibe or
icon was needed.

### Seeding: why not `--fresh`

`--fresh` was rejected. Diffing all 32 sub-regions repo-against-live showed 30 matching,
two moving as intended, and **one that should not move**: British Isles, repo 300 against
live 301. The extra row is `castle_country_house_stays` / Isle of Man (IOM), `updated_at`
2026-07-22, present in the database and absent from the markdown. `--fresh` would have
deleted it silently.

The plain upsert was run instead, on `(subregion, vibe_key, destination)` with
merge-duplicates, which updates and inserts but never deletes. 7,936 ranking rows, 809
blogs, and 966 intros were synced. That left 70 stale rows an upsert cannot reach, removed
by a separate targeted delete: 57 Pacific Islands Hawaii rows, 7 Pacific Islands
`honolulu_waikiki` rows, and 6 US Pacific `cascade_volcanoes` rows, plus the two matching
`vibe_blogs` rows.

### What was actually broken in the live app

Worth recording, because it was worse than "not yet seeded". Before the fix, the six Hawaii
airports were **split across two sub-regions**: HNL, ITO, and KOA resolved to Pacific
Islands while LIH, LNY, and OGG resolved to US Pacific, because `fetchDestinationBlogs()`
picks the sub-region off the highest-scoring ranking row and Hawaii had rows in both. A
Honolulu user was being served Pacific Islands blogs about Fiji and Bora Bora. Nothing
rendered blank, which is why it went unnoticed.

After the fix all six resolve to US Pacific and every ranked vibe key has a blog behind it:

| Airport | Sub-region | Ranked vibes | With a blog | Missing |
| --- | --- | --- | --- | --- |
| HNL | US Pacific | 14 | 14 | 0 |
| OGG | US Pacific | 14 | 14 | 0 |
| KOA | US Pacific | 12 | 12 | 0 |
| LNY | US Pacific | 10 | 10 | 0 |
| LIH | US Pacific | 6 | 6 | 0 |
| ITO | US Pacific | 5 | 5 | 0 |

### Other changes

- Six intros (`honolulu`, `maui`, `kauai`, `kona`, `hilo`, `lanai`) moved with `git mv`.
  No database effect, since `seed-destination-intros.js` matches on IATA plus city. The
  seed reported 966 rows updated and 0 unmatched, confirming it.
- `src/data/vibesCanonical.ts`: `key_island_stops` added to `capital_cosmopolitan_cities`,
  dead `cascade_volcanoes` key removed. `app.json` bumped to 0.3.359 / 566.
- Pacific Islands rank columns re-sorted in 12 sections where they disagreed with their own
  scores. Ambrym rescored down after confirming its lava lakes drained in December 2018 and
  never reformed.
- Molokai (MKK) was considered and excluded: it is not in the `destinations` table, so
  adding it would have meant touching a table this job was scoped out of.

### Still open

Nothing specific to Hawaii. Related items live in [deferred-fixes.md](deferred-fixes.md).
