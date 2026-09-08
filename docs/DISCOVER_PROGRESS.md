# WhereTo: Discover — Progress & Build Tracker

**Single source of truth for the Discover walkthrough (the travel-agent flow).** Keep this updated every work session. If a session ends mid-task, the **▶ PICK UP HERE** marker (bottom) says exactly where to resume.

Last updated: 2026-07-06 · App version at last update: **0.3.35 / versionCode 242**

### Fixed in 0.3.35

- 0.3.34's photo planks were broken on device (all-absolute layout collapsed: photos bled across rows, labels invisible, stepped scrim bands). Rebuilt in-flow: fixed-height touchable → `flex:1` clip (border transparent→rust) → in-flow content row; only photo + one uniform `scrimMid` overlay are absolute.
- `getVibePhoto` gained a cached keyword-root fallback (canonical labels rarely match curated keys exactly; picsum junk was showing). Pinned exact map entries for Snorkeling / Sailing & Yacht Charters / Sea Caves & Blue Lagoons / Stargazing & Dark Skies (the last also guards a "Blue Lagoons"→"Jazz & Blues" mismatch).

### Changed in 0.3.34

- Vibe planks redesigned as full-bleed photo strips (h 62 → 74, within the approved +20%): photo fills the plank, left-anchored stacked scrims carry a white monoExtraBold label, frosted circle badge fills rust on select, rust outline on the clipping container + 1.08 photo zoom on select (same mechanics as the mood tiles). Used by both the generalized list and Fine Tune.

### Changed in 0.3.27 (current)

- **Fine Tune now scopes results to the selected sub-region(s)** (was global). Selections are tracked per sub-region in `fineBySub: Record<subregion, canonicalKeys[]>`; `runSearch` fetches candidates with the sub-region-scoped `fetchCandidatesByCanonical` for each sub-region that has picks, dedups by iata, then ranks/prices as usual. Generalized (mood) search stays global. This reverses the earlier "region/sub-region are a browse aid only" behavior — the founder confirmed fine-tuning must respect the region.


Status legend: ✅ done & typecheck-verified · 🟡 partial/in-progress · ⬜ not started

---

## 0. WHAT DISCOVER IS (the locked spec)

A guided, hand-holding search flow. Not a form, a travel agent: **ask a little, show something wonderful, refine toward reality.** Origin spec: `Vibe_Engine_Fleshout/Boardroom_Session_03_UX_Workflow.md` (binding UX resolutions) plus the founder-approved adjustments from the 2026-07-05 session.

Binding rules (do not violate while iterating):

1. **Match stays pure.** The match score is vibes + lens only. Feasibility (dates, origin, budget) renders BESIDE the match, never blended into it.
2. **Every result explains itself in the user's own words.** The reason sentence is built only from what the user tapped. "We listened," never "we know you."
3. **First results land on vibes + occasion alone.** No budget, dates, or origin before the reveal.
4. **Occasion is skippable.** A required emotional question is a drop-off.
5. **New questions are earned.** Add an input only when instrumentation shows the results disappoint without it, and add it as a warm beat, never a form field.

The locked flow (founder decisions 2026-07-05):

| Beat | What happens | Locked decisions |
|---|---|---|
| Shell | Wizard with sharpening hero image + Back/Continue bar | Back grayed out on step 1; hero blur 16 → 10 → 5 → 0, crossfades to the #1 match photo at reveal |
| 1. Mood | 2x3 grid of the six most popular vacation types (Beach, City break, Mountains, Food & Wine, History & Culture, Adventure & Wildlife), full-bleed Pexels photos | Multiselect, soft cap 3, min 1 to continue. Managed in the Supabase `discover_moods` table (dashboard_cards pattern); tiles slide in from the outside on load like the dashboard |
| 2. Trip basics | Starting Point (airport picker = the `Input isAirportSearch` widget from Wander with Vibes), Budget (Total/Per Person toggle + slider), Travelers (passenger count slider) | Concrete search inputs so the reveal returns true priced results. Replaced the qualitative occasion/Trip Lens (removed 0.3.19). Writes originCode / budget / passengers to searchParams |
| 3. Vibes | Pre-stacked canonical vibe photo cards, 2-col grid | Multiselect grid (not swipe) for wizard consistency; swipe deck shelved as a future A/B; min 1 |
| 4. Reveal | Top **6** destinations, ranked globally (no region gate), rendered as the **standard `SwipeableCard`** (results screen) with full ScoreDisplay, Flight/Hotel/Car action bar, swipe-left remove (next-best backfills), swipe-right explore | Once origin + dates are known the cards are **priced in place** (reuses `confirmSearch`): real "around $X", live Flight/Hotel sheets. Before that, a rating-only preview card + a nudge to add a home airport |
| 5. Refiners | When (real date range) / From (home IATA) / Budget as chips on the results | Positioned after the reveal ("here is fine" per founder); setting From + When triggers pricing and swaps the preview for standard cards; each change re-prices/re-ranks |
| Commit | "This is the one" pill under each priced card → postcard stamp | Postcard's Explore/Save use the priced destination when available |
| Commit | Postcard stamp (like the Wander Together agreement stamp) | Rotated dashed-border postcard, match in a stamp circle → Explore / Save / Keep looking |

Rejected in spec review: feeling sliders (pass), 3-result reveal (too narrow, now 6), 2-option this-or-that opener (too narrow, now 6-image grid).

---

## 1. CURRENT REALITY (post v0.3.14, built 2026-07-05)

All of section 0 is **built and typecheck-clean**, behind no flag (the dashboard tile is the gate; the `dashboard_cards.enabled` column can hide it).

### Files & their role

| File | Role |
|---|---|
| [app/search/discover.tsx](../app/search/discover.tsx) | The whole walkthrough: wizard shell, hero, all 4 steps, elimination, refiners, postcard modal |
| [src/data/discoverMoods.ts](../src/data/discoverMoods.ts) | `fetchDiscoverMoods()` (Supabase `discover_moods`, soft-fails to the bundled fallback) + 4 occasions → canonical vibe keys; `buildVibeStack()` (type-overlap count → occasion boosts → first-seen order, cap 12) |
| [src/lib/vibeRankings.ts](../src/lib/vibeRankings.ts) | New `fetchCandidatesGlobalByCanonical()` — paginated rankings query with NO subregion filter |
| [src/lib/vibeMatch.ts](../src/lib/vibeMatch.ts) | Unchanged. Discover calls `rankDestinations()` + `parseMonths()` |
| [app/search/direct.tsx](../app/search/direct.tsx) | WhereTo: Direct stub screen (tile routes here; feature TBD) |
| [app/index.tsx](../app/index.tsx) | Dashboard rebuilt as 2x3 PhotoTile grid, 6 tiles, all animations kept and generalized |
| [src/utils/analytics.ts](../src/utils/analytics.ts) | 5 new events: `discover_started/step_completed/revealed/eliminated/committed` |

### Engine wiring

moods → canonical pre-stack → user-kept keys become `searchParams.selectedCanonical` → `fetchCandidatesGlobalByCanonical` → filter to bookable pool (`loadDestinationPool`) → `rankDestinations` (obviousness index cached module-level) → top 6 shown, top 30 kept for elimination backfill. Explore pushes `/search/vibe-explore` with a minimal `Destination` (+ `scoring`). Save uses `toggleFavorite`.

### Supabase state (SQL ran 2026-07-05)

- `dashboard_cards`: 6 enabled rows, order: Discover(1), Wander with Vibes(2), Wander Together(3), Wander as a Group(4), Conversational(5), Direct(6).
- `Dashboard_Images-Discover` and `Dashboard_Images-Direct` exist (public read policy), 2 images each.
- `discover_moods` (v0.3.15): the step-1 vacation-type tiles — key, label, phrase, image_url (Pexels), canonical_keys text[], entry animation config, sort_order, enabled. Public read. Edit rows to change tiles without a release; the app falls back to the bundled list if the table is empty or unreachable.

### Fixed in 0.3.15

- Step-1 tiles rendered with zero height on device (percent width + `aspectRatio` with only absolutely-positioned children). All Discover grid tiles now use explicit pixel sizes (`TILE_W` derived from screen width).

### Fixed in 0.3.16

- Selected vacation types were being ignored in favor of beach vibes. Root cause: `buildVibeStack` injected the occasion's boost keys as new stack entries AND gave them tiebreak priority over the mood's own keys, so City + Food/Wine + a family-type occasion pushed `beaches_beach_towns`, `snorkeling`, `all_inclusive_beach_resorts` ahead of the actual city/food vibes. Fix: occasion boosts are filtered to keys already present in the selected types (`boosts.filter(k => count.has(k))`) — promote-only, never inject. The `discover_moods` table mapping was correct; this was purely the stack builder.

---

## 2. KNOWN v1 LIMITATIONS (each is a candidate next task)

| # | Limitation | Why it is this way |
|---|---|---|
| L1 | ✅ RESOLVED 0.3.17 | Reveal prices in place via `confirmSearch` once origin + dates are set; standard cards show real "around $X" and live Flight/Hotel sheets |
| L2 | ✅ RESOLVED 0.3.17 | Standard card's ScoreDisplay + priced flight/hotel are the feasibility signals now |
| L8 | 🟡 Booking from Discover is browse-only | Flight/Hotel sheets show priced options, but the full booking (passenger form → order) still lives in the results screen; Discover's sheets don't wire `onSelectFlight`/`onAddToBooking` yet |
| L9 | 🟡 Default trip window when dates unset | Pricing seeds the 1st of next month + 6 nights so cards can price before the user picks dates; the When refiner overrides it |
| L3 | ⬜ Postcard is not shareable | v1 is an in-app success stamp only |
| L4 | ✅ RESOLVED 0.3.15 | Step-1 tiles are now Pexels photos managed in the `discover_moods` table |
| L5 | ⬜ WhereTo: Direct is a stub | Founder: "we will tackle that later" |
| L6 | 🟡 Occasion lens only re-orders the pre-stack | It does not yet weight `VibeHit.weight` (the lens re-skin from the boardroom spec is not wired). As of 0.3.16 it only PROMOTES vibes a selected type already carries — it never injects unrelated vibes (see the "Fixed in 0.3.16" note) |
| L7 | ⬜ No device verification yet | Founder tests on phone; tooling side is typecheck-only |

---

## 3. ROADMAP (next couple of days)

Ordered proposal; re-order freely.

- ⬜ **P1. Device pass + polish.** Founder runs the flow on phone. Expected rough edges: hero crossfade when results land (fade only fires on step change), mood-photo load time, wizard bar overlap on small screens, LayoutAnimation jank on elimination.
- ✅ **P2. Priced handoff (done 0.3.17).** Reveal prices in place via `confirmSearch`; standard cards show real prices and live Flight/Hotel sheets.
- ⬜ **P3. Bridge to booking (partial; see L8).** Flight/Hotel sheets browse priced options in Discover; still need to wire the sheet's `onSelectFlight`/`onAddToBooking` into the passenger form → order flow (or route into results carrying the priced dest) so a booking can be completed from Discover.
- ⬜ **P4. Occasion as a real lens (kills L6).** Map occasion to `VibeHit.weight` multipliers before `rankDestinations`, and re-skin the reason sentence per lens.
- ⬜ **P5. Postcard share (kills L3).** Render-to-image + native share sheet, styled like the Wander Together agreement stamp.
- ⬜ **P6. Funnel dashboard.** Add the 5 `discover_*` events to the PostHog "Core Funnels" dashboard (project 445316) and watch step drop-off to decide the next question (spec rule 5).
- ✅ **P7. Mood photography (done 0.3.15).** `discover_moods` Supabase table with Pexels images; copy/photos/ordering editable without a release.

## Open decisions

| # | Question | Leaning |
|---|---|---|
| D1 | Where do refiners eventually live? | Founder: "not sure at what point is best yet"; v1 keeps them post-reveal. Revisit after P6 data |
| D2 | Should Discover replace Conversational long-term? | Both tiles live today; Conversational still has no VibeMatch wiring. Decide after funnel comparison |
| D3 | Mood grid A/B: multiselect grid vs 2-round adaptive pick | Grid shipped; adaptive shelved |
| D4 | `feature_discover` remote flag? | Currently gated only by the dashboard tile. Add an `app_config` flag if a kill switch is wanted before wider beta |

---

## ▶ PICK UP HERE

v0.3.17 swapped the reveal to the standard `SwipeableCard` with in-place pricing (reuses `confirmSearch`), keeping the elimination round and postcard. Nothing is mid-task. Next session: **P1 device feedback** (watch: pan-gesture card swipe nested inside the reveal ScrollView, pricing spinner timing, sheet mounts), then **P3 booking bridge** (L8 — let a booking complete from Discover's Flight sheet). Before any code, re-read section 0's binding rules.

### Changed in 0.3.23 (current)

- Mood tile selection: full rust outline + 15% image zoom (animated, image only via `MoodTile`), dropped the rust shadow that read as an underline.
- Trip-basics reordered: Include → Starting Point → Budget → Dates → Travelers; nights label ("— five nights —", singular for one) under the dates via a local `nightsLabel` (mirrors traditional + `typography.nightsLabel`).
- Fine Tune shows "Vibes Specific to[ the] <sub-region>" header (exported `articleFor` from `ResultsHero`, same set as the results hero).

### Changed in 0.3.22

- Step 1 mood tiles shortened (`MOOD_TILE_H = TILE_W * 0.8`) so all six fit on screen.
- Step 2 vibes render as single-column full-width planks (`VibePlank`: photo thumb + label + check), ~6 visible, replacing the 2-col photo tiles.
- Step 2 "Fine Tune" pill: swaps the generalized planks for Region → Sub-Region dropdowns (`VIBE_REGIONS` / `getVibeSubRegions`) + the canonical vibes available in that sub-region (`fetchSubregionCanonicalCoverage`), rendered as planks. `fineVibes` multiselect persists across region/sub-region changes. Search uses `effectiveVibes = fineVibes.size ? fineVibes : vibeSel` — fine-tune vibes win, else generalized. Region/sub-region are a browsing aid only; the search stays global (fine-tune vibes can span regions).

### Changed in 0.3.20

- Discover is now 3 steps (mood → trip basics → vibes) and **hands off to the standard results screen** (`/search/results?from=discover`) instead of rendering its own reveal. Removed: the sharpening hero image, in-app SwipeableCard reveal, elimination round, postcard, reveal refiners, and the Flight/Hotel detail sheets mounted inside Discover.
- Trip basics now has: Starting Point, **Travel Dates (`DateRangeInput`)**, Budget, Travelers, and **component stamps (Flights / Hotel / Rental Car-soon)** using the results-page icons. Flights + Hotel default on, deselectable, one must remain.
- New `searchParams.includeFlights` / `includeHotels` (default true). `confirmSearch` prices + budget-counts only toggled-on components (wrapped so all other workflows are byte-identical). `confirmSearch` also clears + sets loading before its first await (no empty-state flash on handoff).
- Results hero shows "Destinations to Discover" (Discover in rust) when `from=discover` (new `discoverMode` prop on `ResultsHero`).
- `runSearch` ranks the whole map (`fetchCandidatesGlobalByCanonical` → `rankDestinations`), builds `CurationSuggestion[]`, primes the store, pushes results, calls `confirmSearch`. Postcard/elimination are gone (superseded by the standard results UX).

### Changed in 0.3.19

- Step 2 became "trip basics" (Starting Point + Budget + Travelers), replacing the occasion/Trip Lens. The reveal now prices immediately (origin captured up front + default dates), so users see true `Destination` results (score, in-budget, flights + hotel) from the start rather than the rating-only preview. Reveal refiners reduced to When (dates) + tap-to-edit chips for starting point / budget. `buildVibeStack` is now called with `null` occasion (mood-driven only). The bespoke preview card only shows if no starting airport is set.

### Fixed / built in 0.3.17

- Reveal renders `SwipeableCard` (exported from `app/search/results.tsx`) instead of the bespoke card: full ScoreDisplay, Flight/Hotel/Car bar, swipe remove/explore.
- Pricing reuses the store: Discover builds `CurationSuggestion[]` from its ranked items, calls `setCurationSuggestions` + `confirmSearch`, and renders `store.destinations`. `confirmSearch`'s scoring gate was relaxed from the global flag to `if (suggestion.vibeScore)` so Discover scores render flag-off.
- `SwipeableCard` (and `SwipeableCardProps`) exported; Flight/Hotel use the same `FlightDetailSheet` / `HotelDetailSheet` the results screen mounts.
- The bespoke rating card remains as the pre-pricing preview (shown until a home airport is set).
