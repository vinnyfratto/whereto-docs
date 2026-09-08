# Vibe Engine — Deferred Fixes

_Living doc. Last updated: 2026-08-06._

A running list of problems found while doing other Vibe Engine work and deliberately not
fixed at the time, with the reason for leaving them. Each entry says what is wrong, how it
was found, why it was deferred, and what fixing it would cost. Nothing here is urgent
enough to derail the pass that found it, and nothing here should be rediscovered from
scratch six months from now.

Add to this file whenever a pass turns something up that it is not going to fix. Do not
delete entries when they are fixed, mark them RESOLVED with a date.

---

## Open

### 5. 157 source links across the blogs are dead

**Status:** OPEN. Audited 2026-08-06, repair not started.

Every blog carries a `## Sources` list of deep links, and the spec requires at least five
per blog from the last 24 months. Nothing checked that those URLs still resolved after the
blog was written, so nobody knew the rate.

**The audit.** All 809 blogs, **4,761 source links, 4,325 unique URLs.** A 403, 401, or 429
was treated as alive, since those are bot blocks on a page that exists.

**Result: 157 confirmed dead across 109 blogs**, out of 4,761. A 3.3 percent rot rate.

Getting to that number took two passes, and the difference between them is the useful part.
A fast concurrent pass reported 341 failures. Re-checking every non-404 failure gently, one
at a time with longer timeouts, showed **72 of them were alive** and the tooling was at
fault. Of what remained, 13 resolved to a real 404 and joined the confirmed list, and the
rest split into:

- **78 persistent connection errors.** Every one of these domains resolves in DNS. They are
  hard bot blocks, concentrated in `travel.usnews.com` (10) and `theculturetrip.com` (8),
  both of which reject automated clients outright. Almost certainly alive.
- **14 returning 406.** Content-negotiation rejection, meaning the page exists.
- **4 server errors** that are genuinely ambiguous.

So roughly 92 links cannot be verified by script at all and need a person to open them. Any
future audit should report that bucket separately rather than counting it as rot.

**A caution drawn from this.** During session 5 of the Hawaii job a `travel.usnews.com`
citation was replaced because curl returned nothing for it. On this evidence that link was
probably fine and the replacement was unnecessary. An automated failure is a reason to look,
not a reason to edit.

**The 404s are not spread evenly. Half of them are in two sub-regions.**

| Sub-region | Dead links |
| --- | --- |
| US Southwest | 47 |
| US South | 37 |
| everything else | 73 across 30 sub-regions |

**The cause is a source-selection habit, not bad luck.** Those two passes were built almost
entirely on destination-marketing sites, and those restructure constantly:

`visitsedona.com` 8, `discovermoab.com` 4, `visitlasvegas.com` 3, `santafe.org` 3,
`newmexico.org` 3, `visitalbuquerque.org` 3, `visittucson.org` 3, `texashighways.com` 3,
`texasmonthly.com` 3, `austintexas.org` 2, `visitsanantonio.com` 2, `visitokc.com` 2,
`travelok.com` 2, `visithoustontexas.com` 2.

By contrast, the sub-regions built on national park services, government agencies, UNESCO,
and established publications lost almost nothing.

**40 blogs would fall below the five-source minimum** if the dead links were simply deleted,
so deletion alone is not an option for those.

**The decision to make before repairing.** Two approaches, and they cost very differently:

1. **Patch.** Find a current equivalent for each of the 144. Preserves the existing
   argument in each blog. Several sessions of research, and it re-cites the same fragile
   class of source, so the same rot recurs in a year.
2. **Re-source.** Replace dead marketing-site citations with durable ones: park services,
   government tourism boards' stable pages, UNESCO, major publications. Slower per link,
   but the citations survive. Worth pairing with a rule in the spec that a
   destination-marketing URL is a last resort rather than a default.

Recommendation is 2 for US Southwest and US South, which are the concentrated failures and
where patching would just reset the clock, and 1 for the scattered 71 elsewhere.

**Repair policy already in use.** Homepages are replaced with deep links. When a URL is
swapped, the link title and description must be updated too, which is easy to miss.

**Tooling.** The audit is now `scripts/audit-blog-sources.py`. Two result files are
committed so the repair can be worked from a file rather than re-run:
`docs/vibes-engine/link-audit-2026-08-06.tsv` is the raw first pass, and
`docs/vibes-engine/link-audit-2026-08-06-confirmed-dead.tsv` is the verified 157 to fix.
The script still needs the gentle second pass folded into it, since as written it
over-reports by more than a factor of two. See also item 7: the QA pass does not check
sources at all today.

---

### 6b. 58 blogs keep a curated snapshot rather than a strict top-N

**Status:** OPEN by decision, 2026-08-06. The serious half is fixed, see resolved item 6.

Of the 81 blogs whose snapshot was not a top-N of the master, 23 omitted a destination the
master ranks in its top three and were fixed. The remaining **58 only skip rows further down
the list**, which misleads nobody about who leads.

**Left alone deliberately.** A writer showing the top eight of twelve and dropping a
mid-table entry may well have meant to. Rewriting all 58 mechanically would destroy whatever
curation was intentional to fix a problem no reader would notice.

The QA pass now reports these as warnings rather than failures, so they stay visible without
blocking a run. If the policy ever changes to "every snapshot is a strict top-N", the
regeneration is a one-line script change.

---

### 8. NorthAfrica's master file uses a different structure from the other 31

**Status:** OPEN. Found 2026-08-06 while fixing item 3.

Every other sub-region writes `## vibe_key: Label` with a 16-column audit table carrying the
six sub-scores. NorthAfrica writes ``## Title (`vibe_key`)`` with an 8-column table and no
sub-scores.

**Impact.** Any script that parses masters has to special-case it, and one that does not
will either crash or silently skip NorthAfrica. The item 3 fix crashed on it before being
made shape-aware. It also means NorthAfrica's composites cannot be verified against the
weighting formula, because the sub-scores were never written down.

**Cost to fix.** Either rewrite NorthAfrica's master into the standard shape, which means
reconstructing six sub-scores per row that do not exist, or accept the variant and keep the
tooling shape-aware. The second is cheaper and is what the item 3 fix did.

---

### 9. 56 rows duplicate another row inside the same vibe

**Status:** OPEN. Found 2026-08-06 while fixing item 6.

Some vibes score the same place more than once under different names. Central America's
`rainforest_wildlife` has "Quepos (Manuel Antonio park)" at 86 and "Manuel Antonio" at 85,
plus "Corcovado (Osa)" at 84 and "Puerto Jimenez (Corcovado)" at 83: two places occupying
four of the top four slots. 56 rows across the engine look like this, worst in British Isles
(7), Mediterranean (6), and Central America (6).

**Why it matters.** It inflates a vibe's apparent breadth and lets the recommender return
the same destination twice under two names.

**Why not all of them are wrong.** "Petra" and "Petra by Night" are genuinely different
outings. "Serengeti" and "Serengeti (Ndutu)" is a region and a specific area within it. Each
of the 56 needs a look.

**Why deferred.** Deciding whether two rows are one place is an editorial and scoring call,
and merging them changes rankings. That is the user's decision, not a script's.

**Cost to fix.** An hour to review 56 pairs, then re-score and re-sort whichever get merged.

---

### 10. 39 blogs are under the 1,200-word minimum, almost all in two sub-regions

**Status:** OPEN. Surfaced by the full QA sweep 2026-08-06.

29 of 32 sub-regions now pass QA cleanly. Two of the three failures are word count:
**Caribbean 21 blogs and Mediterranean 18**, both written short of the spec's 1,200-word
narrative minimum. No other sub-region fails on this.

Like the source-link rot in item 5, the concentration points at the pass rather than at bad
luck: those two were early builds, and the standard tightened afterwards.

**Cost to fix.** Roughly 39 blogs needing 100 to 200 words each of genuine added content.
Two sessions. Worth pairing with item 5, since Caribbean also carries dead source links and
both jobs mean opening the same files.

---

## Resolved

### 6. 23 blogs told the reader a different leader than the data

**Status:** RESOLVED 2026-08-06.

A blog's Destination Ranking Snapshot is meant to be the top rows of the master. In 81 it
was not: the snapshot skipped destinations out of the middle and renumbered around the gap,
so a reader saw a ranking the data does not support.

The ambiguity was the real problem. A skipped row could be deliberate curation or could be a
master that moved after the blog was written, and the two produce an identical file.

Graded by severity, **23 were serious**: 3 omitted the master's number one outright and 20
omitted something in its top three. Those are fixed, rebuilt as true top-N tables while
preserving each row's existing hand-written "Why" text. Prose was adjusted in two of them
where it named a leader the corrected table no longer showed.

The remaining 58 only skip further down and were left alone by decision, see item 6b.

Recurrence is now blocked: `scripts/build-vibes-overview.js` hard-fails when a snapshot omits
a master top-3 destination.

---

### 7. The QA script did not validate `hotel_tag` against the controlled vocabulary

**Status:** RESOLVED 2026-08-06.

`scripts/build-vibes-overview.js` now reads the vocabulary table straight out of
`VibesEngine-Details.md` and fails on any tag not in it, so the spec and the blogs cannot
drift again. This is why item 4 hid for months.

**One thing worth recording.** The first version of this check silently did nothing. It
referenced a `ROOT` constant that does not exist in that script, and the `try/catch` around
the file read swallowed the `ReferenceError`, so the vocabulary loaded as an empty set and
every tag passed. It was only caught by deliberately setting a blog to a junk tag and
confirming the check fired. **A validation check that cannot fail is worse than no check**,
and the catch now logs instead of swallowing.

---


### 1. Volcanoes were filed under Sea Kayaking in the canonical vibe map

**Status:** RESOLVED 2026-08-06.

`volcanoes_craters` and `volcanoes_hot_springs` sat in the `sea_kayaking_rafting` canonical
vibe, so a user picking "Sea Kayaking & Rafting" could be matched to Kilauea.

The original estimate here was wrong in two ways, both worth recording.

**First, a `volcanoes_geothermal` canonical vibe already existed** under Nature & Outdoors
with twelve volcano keys in it. This was never a missing-vibe problem, only two keys in the
wrong bucket, so no new canonical entry was needed.

**Second, and more important: `src/data/vibesCanonical.ts` is auto-generated.** Its header
says so. It is produced by `scripts/canonicalize-vibes.mjs`, which reads engine keys from
Supabase and assigns each to the first canonical vibe whose substring rule matches.
Hand-editing the output is futile, and an earlier hand-edit in this same job was silently
lost the first time the generator ran. The fix has to go in the ruleset.

The root cause was substring matching without word boundaries. Fixing it caught two
unrelated mis-mappings of exactly the same kind:

| Engine key | Was in | Because | Now |
| --- | --- | --- | --- |
| `volcanoes_craters`, `volcanoes_hot_springs` | Sea Kayaking & Rafting | stale generated output | Volcanoes & Geothermal |
| `romance_honeymoon` | Ancient Ruins & Archaeology | `roman` is inside `romance` | its own Romance & Honeymoon vibe |
| `luxury_romance` | Ancient Ruins & Archaeology | same | Luxury Escapes |
| `bbq_comfort_food` | Castles & Forts | `fort` is inside `comfort` | Barbecue & Grill |

Four guards added to the ruleset and the taxonomy regenerated: 171 canonical vibes, every
one with an icon. `key_island_stops` is grouped with the capitals by rule rather than by
hand, so it survives regeneration. Six icons added for canonical keys that had none, five of
them NorthAfrica vibes that only appeared once NorthAfrica was seeded.

**Worth watching:** the same substring trap will catch any future short rule token that is a
fragment of a longer word.

---

### 2. `cascade_volcanoes` was a dead engine key

**Status:** RESOLVED 2026-08-06.

The US Pacific master renamed its `cascade_volcanoes` section to `volcanoes_craters`,
leaving `cascade_volcanoes` with no ranking rows behind it. It is gone from the regenerated
taxonomy and from the emoji map in `scripts/build-vibes-engine-page.js`.

---

### 3. Rank columns disagreed with their own scores

**Status:** RESOLVED 2026-08-06, and much larger than logged.

Logged as six US Pacific tables. The real number, once all 32 sub-regions were audited, was
**346 ordering breaks across 16 sub-regions**, worst in EastAsia (64), Mediterranean (52),
and SoutheastAsia (46). Plus three genuine data faults the check surfaced:

- EastAsia `historic_old_towns`: rank jumped 14 to 16, a row deleted without renumbering.
- NordicBaltic `wildlife_watching`: two rows both numbered 12.
- Mediterranean `volcano_hiking`: Methana scored 65 but was tiered C. 65 is the B threshold.

Every composite was correct, so this was presentation rather than data. Fixed by re-sorting
all masters by score, renumbering, and correcting tiers from scores. 179 sections reordered,
178 blog snapshots updated to match, preserving each row's existing hand-written "Why" text
and falling back to the master note only for a destination newly promoted into view.

Verified afterwards: **7,936 rows across 32 sub-regions, zero score, tier, sequence, or
ordering issues.** Confirmed against a stashed baseline that the change introduced no new QA
failures in any sub-region.

---

### 4. Two `hotel_tag` values were not in the controlled vocabulary

**Status:** RESOLVED 2026-08-06.

`overwater` and `guest_ranch` were in use across four blogs without being declared. Both
describe genuinely distinct property types with no good substitute in the existing list, so
the vocabulary in `Content/VibesEngine/VibesEngine-Details.md` was extended to 13 terms
rather than the blogs being retagged. Every `hotel_tag` in use is now declared. See item 7
for why this went unnoticed.

---

### 11. Two US Pacific blogs ranked Hawaii and never mentioned it

**Status:** RESOLVED 2026-08-06. Found while fixing item 3.

`surfing` and `spa_wellness_retreats` were the two vibes commit `5786f7c` merged Hawaii
rows into correctly, and the Hawaii anomaly doc called them "the only two that worked". That
was true of the rankings and false of the blogs. Neither blog mentioned Hawaii at all.

`surfing` was the bad one: its master ranks **North Shore (Oahu) first at 90**, above San
Diego at 88, and the blog's snapshot omitted it entirely and showed San Diego as rank 1. A
Hawaii user picking Surfing on Explore got a guide to California.

Both rewritten with Hawaii sections, snapshots regenerated to the full master, frontmatter
and seasons corrected. `surfing` now leads on the North Shore and covers Waikiki as the
teaching wave; `spa_wellness_retreats` covers lomilomi and Sensei Lanai.

**The lesson:** a merge that gets the rankings right still leaves the prose wrong, and
nothing in the QA pass compares the two. Items 6 and 6b are the general form of this problem.
