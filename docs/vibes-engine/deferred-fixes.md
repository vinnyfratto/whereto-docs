# Vibe Engine — Deferred Fixes

_Living doc. Last updated: 2026-08-06._

A running list of problems found while doing other Vibe Engine work and deliberately not
fixed at the time, with the reason for leaving them. Each entry says what is wrong, how it
was found, why it was deferred, and what fixing it would cost. Nothing here is urgent
enough to derail the pass that found it, and nothing here should be rediscovered from
scratch six months from now.

Add to this file whenever a pass turns something up that it is not going to fix. Do not
delete entries when they are fixed, mark them RESOLVED with a date and the commit.

---

## Open

### 1. Volcanoes are filed under Sea Kayaking in the canonical vibe map

**Status:** OPEN. Found 2026-08-06 during the Hawaii integration pass.

`src/data/vibesCanonical.ts` line 45 defines the canonical vibe `sea_kayaking_rafting`,
label "Sea Kayaking & Rafting", and its `engineKeys` array contains three volcano keys:

```
"volcanoes_hot_springs", "cascade_volcanoes", "volcanoes_craters"
```

A user who picks "Sea Kayaking & Rafting" in the app can be matched to Mount Rainier and
Kilauea. A user looking for volcanoes has no canonical vibe that finds them.

**How it was found.** The Hawaii pass renamed the US Pacific `cascade_volcanoes` section
to `volcanoes_craters`. Checking that the rename would not orphan an engine key surfaced
the mapping.

**Why deferred.** Both keys sit inside the same canonical vibe, so the rename did not
change any behaviour, and the merge was safe. Fixing it properly means deciding where
volcano keys belong, which is a taxonomy question touching every sub-region that scores a
volcano vibe, not just US Pacific. That is a global Vibe pass decision.

**Cost to fix.** Small edit, wide blast radius. One new canonical entry (a `volcanoes`
vibe under "Nature & Outdoors") plus a matching `vibeIcons.ts` entry, then move the three
keys off `sea_kayaking_rafting`. Touches `src/`, so it needs a `version` and `versionCode`
bump. Worth folding into the global Vibe pass rather than doing alone.

---

### 2. `cascade_volcanoes` is now a dead engine key

**Status:** OPEN. Created 2026-08-06 by the Hawaii integration pass.

The US Pacific master no longer has a `cascade_volcanoes` section. The key still appears
in the `sea_kayaking_rafting` engineKeys array in `src/data/vibesCanonical.ts`.

**Impact.** None at runtime. An engine key with no ranking rows behind it matches nothing.
It is dead weight that will confuse the next person auditing the taxonomy.

**Why deferred.** Removing it is a one-word deletion but it touches `src/`, which forces a
version bump. Batching it with item 1, which edits the same line, avoids bumping twice for
the same array.

---

### 3. Six US Pacific tables have rank columns that disagree with their own scores

**Status:** OPEN. Found 2026-08-06 during the Hawaii integration pass.

In `Content/VibesEngine/NorthandCentralAmerica/USPacific/vibe_destination_rankings.md`,
six sections have rows whose `Rank` column does not follow descending `Score`:

| Vibe | The break |
|---|---|
| `pch_coastal_drives` | rank 6 scores 76, rank 7 scores 79 |
| `coastal_towns_lighthouses` | rank 9 scores 68, rank 10 scores 80 |
| `skiing_snowboarding` | rank 8 scores 68, rank 9 scores 77; rank 10 scores 48, rank 11 scores 77 |
| `wine_country` | rank 8 scores 74, rank 9 scores 77 |
| `craft_beer` | rank 11 scores 62, rank 12 scores 80 |
| `gold_rush_missions` | rank 8 scores 70, rank 9 scores 82 |

Every one of these is a destination appended to an existing table without re-sorting. The
pattern is consistent: the out-of-order row is always at the bottom, and always a smaller
gateway added in a later corridor-expansion pass.

**How it was found.** A validation script run over all 304 rows after the Hawaii merge,
checking each composite against the weighting formula, each tier against its band, and
each rank against score order. The scores and tiers were all correct. Only the ordering
was wrong, and only in sections the Hawaii pass did not touch.

**Why deferred.** Re-sorting the master is trivial. The problem is that each of these six
vibes also has a blog carrying a Destination Ranking Snapshot table built from the old
order. Fixing the master alone would put the master and the blog into disagreement, which
is precisely the failure the Hawaii job exists to remove. Fixing both means six more blog
revisions in a job that already has thirty-five.

**Cost to fix.** Re-sort six tables, then update the snapshot table and any prose that
names a rank in six blogs. Best done as its own small pass once the Hawaii work is closed,
or folded into the global Vibe pass.

---

### 4. Two `hotel_tag` values are not in the controlled vocabulary

**Status:** OPEN. Found 2026-08-06 during the US Pacific blog writing.

`Content/VibesEngine/VibesEngine-Details.md` defines a closed vocabulary for the
`hotel_tag` frontmatter field and says plainly: "Do not invent new values. If a new
category is genuinely needed, add it to this table first." Two values in use were never
added to that table.

| Value | Blogs using it |
|---|---|
| `overwater` | `Asia/SouthAsia/blogs/maldives_overwater_luxury.md`, `Oceania/PacificIslands/blogs/overwater_bungalows.md` |
| `guest_ranch` | `NorthandCentralAmerica/USMountainWest/blogs/dude_ranches.md`, `NorthandCentralAmerica/USSouth/blogs/texas_ranch_stays.md` |

**Impact.** `hotel_tag` is meant to pre-filter or weight hotel search toward a property
type. A tag with no counterpart on the search side filters to nothing, so those four vibes
would silently return no weighted results rather than falling back to unfiltered. Nothing
is broken today because the tag is not yet wired to hotel search at all.

**Why deferred.** Both values describe real and distinct property types, so the right fix
is probably to add them to the vocabulary table rather than to retag the blogs. That is a
spec change affecting how `hotel_tag` maps to LiteAPI hotel filters, which is a decision
for whoever wires the field up, not for a content pass.

**Cost to fix.** Either two rows added to the vocabulary table in
`VibesEngine-Details.md`, or four blogs retagged (`overwater` to `luxury`, `guest_ranch`
to `historic_inn` or `eco_lodge`, neither of which fits well). Prefer the former.

**Related note, not a bug.** Quoting of `hotel_tag` values is inconsistent repo-wide:
bare (`hotel_tag: luxury`) and double-quoted (`hotel_tag: "luxury"`) both appear. These
parse identically, so it is cosmetic. Blog frontmatter is consistent on the empty case,
with all 711 untagged blogs using `hotel_tag: ""`. The backtick form that appears in
`vibe_destination_rankings.md` files is documentation prose, not frontmatter, and is fine.

---

## Resolved

_Nothing yet._
