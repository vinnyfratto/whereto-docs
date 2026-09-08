# WhereTo Analytics — Standing Context

This is the standing context for analytics/tracking-plan work on WhereTo. Read it
before touching any PostHog instrumentation in this repo or in `WhereToTrips_Website`.
Source documents (not checked into this repo): `WhereTo_Analytics_Reporting_Strategy.xlsx`
(the tracking plan — authoritative over prose wherever they disagree),
`WhereTo_Analytics_Reporting_Strategy.docx`, `WhereTo_Analytics_Tools_Configuration_Guide.docx`.

## Product shape

Four modules: **WhereTo: Discover** (vibe-based swipe feed), **WhereTo: Explore**
(map/list browsing), **Wander Together** (collaborative planning, 2 people up to a
full group, including polls and voting), **WhereTo: Direct** (known-destination
search and booking). Three reporting categories: **A** — WhereToTrips.com, **B** —
WhereTo App, **C** — Overall Travel Metrics. 47 metrics across 3 tiers, 62 events in
the taxonomy.

## Four goals driving priority

1. Establish a reliable picture of overall usage across the website and the app —
   a standing monitoring commitment, not a project.
2. Dial in the Vibe engine.
3. Improve search result relevance.
4. Reduce and optimize booking-API search calls.

## Non-negotiable implementation rules

- `surface` is a registered super property and is what splits reporting categories
  A and B. `posthog.reset()` **clears registered super properties**. `surface` must
  be re-registered immediately after every `reset()` on logout, on every platform.
  Miss this and every subsequent event in that session silently drops out of the
  A/B dashboards. **Status in this repo (2026-08-09):** fixed in
  `src/utils/analytics.ts` — `reset()` now calls `registerStandingProperties()`
  again after `posthog.reset()`. The property was also renamed-by-addition: the
  app registers both `platform: 'app'` (pre-existing, still relied on by the
  website's `/admin-dashboard/`) and the taxonomy's `surface` (`whereto_app_ios` /
  `whereto_app_android`), side by side. Website fixed the same way in
  `base.njk`/`partner.njk` (`surface: 'whereto_trips_web'`), no `reset()` call
  exists there today so no re-registration bug applied on that side.
- The booking funnel is **8 ordered steps**
  (`search_results_returned` → `view_item` → `option_selected` → `booking_started`
  → `passenger_info_entered` → `booking_reviewed` → `payment_info_entered` →
  `booking_submitted`) plus a **9th step defined as an Action matching
  `flight_booked` OR `hotel_booked`**. Never build it as 11 sequential steps — the
  three outcomes are mutually exclusive, so a literal 11-step funnel reads 0% past
  step 8. `booking_failed` is reported separately as an outcome rate, not as a
  funnel step.
- PostHog error tracking needs **two** switches: the project setting *and*
  an SDK-side autocapture flag (the Config Guide calls it
  `errorTrackingConfig.autoCapture`; the actual `posthog-react-native` 4.46.2
  API is `errorTracking: { autocapture: true }` in `posthog.init()` — verified
  against the SDK's own type definitions, not the doc's paraphrase).
  Symbolication requires dSYM (iOS) and mapping file (Android) upload.
  **Status (2026-08-10): both switches now on** — project toggle enabled in
  PostHog settings, SDK config added in `src/utils/analytics.ts`. Sentry was
  evaluated and deliberately shelved (`docs/decisions/0007-posthog-sentry-deferred.md`);
  this makes PostHog error tracking its replacement, closing R-04
  (`docs/risks.md`) / G-04 (`docs/GAP-REPORT.md`). Still open: dSYM/mapping-file
  upload isn't wired into the EAS build pipeline, so captured exceptions won't
  symbolicate yet — real build-config work, not a flag.
- `$screen` autocapture only covers UIViewController (iOS) and foreground Activity
  (Android). This app is Expo Router / React Native, not SwiftUI or Jetpack
  Compose — `screen()` is called explicitly from `app/_layout.tsx`'s
  `ScreenTracker` on every route change, so this gap does not apply here.
- `register()` signatures differ: **iOS takes a dictionary, Android takes a single
  key/value pair.** Not directly relevant to this repo — `posthog-react-native`'s
  `posthog.register()` takes one object on both platforms; this is a native-SDK
  distinction from the two-Config-Guide documents, called out here so nobody
  "fixes" a non-bug.
- Lifecycle event names are plain text with no `$`: `Application Opened`,
  `Application Installed`, `Application Updated`, `Application Backgrounded`.
  These are autocaptured by `posthog-react-native`; do not hand-implement them.
- `Application Installed` is **not** "new users" — it fires per install and
  reinstall and counts devices. Use PostHog's first-time-event filter for new users.
- There is **no install attribution on mobile**. `$channel_type` is web-only, so
  "acquisition channel" is not a valid app breakdown unless Play Install
  Referrer / deep-link UTMs are captured explicitly (not done).
- PostHog's native bounce rate is **not** the inverse of the GA4-style
  engaged-session rate — they are different definitions. Report one or the other
  and say which. The workbook's Summary KPIs by Category tab is the authoritative
  definition (KPI Priority Tiers tab and Strategy §2.2 both incorrectly say
  "bounce rate is its inverse").
- PostHog's built-in Stickiness insight is **not** DAU ÷ MAU — it counts
  days-active. Only a formula insight gives the ratio.
- PostHog has no "key event" concept (that is GA4 terminology). Use **Verified**
  under Data management → Events. `flight_booked` and `hotel_booked` were marked
  Verified in the live project (445316) on 2026-08-09.
- Dashboards have a date-range override, but **"compare to previous period" is a
  per-insight Trends option** — not available on funnels or retention.
  Inheritance breaks if a tile has its own date range set.
- The current feature-flag API is `getFeatureFlagResult()`.
- The Website Conversions metric is exactly `sign_up`, `app_store_click`,
  `flight_booked` and `hotel_booked`. **`booking_started` is deliberately
  excluded** — counting both the start and the completion double-counts every
  booking.
- `discover_card_save` is the canonical save event. **Status (2026-08-10):**
  renamed — `favorite_added` → `discover_card_save` in `src/utils/analytics.ts`
  (`Events.DISCOVER_CARD_SAVE`) and its one call site in `appStore.ts`
  `toggleFavorite`. This was a deliberate decision (Vinny, 2026-08-10): old
  `favorite_added` rows stay in PostHog as history, nothing new ships under
  that name going forward. `favorite_removed` is untouched (no taxonomy
  equivalent exists for "unsave").
- Flight-search overage formula, parentheses required:
  `$0.005 × max(0, flight_searches − 1,500 × (1 + flight_bookings))`.
- Cross-platform property traps: `$device_model` is a hardware ID on iOS
  ("iPhone15,2") but `Build.MODEL` on Android ("Pixel 7a"); `$device_name` is a
  generic class on iOS but an internal codename on Android; `$locale` is `"en"`
  on iOS and `"en-US"` on Android; screen dimensions are points on iOS,
  density-independent pixels on Android.
- Do **not** enable PostHog group analytics — it is a paid add-on that bills the
  moment it is enabled. Confirmed OFF in the live project on 2026-08-09
  (Organization → Billing showed "Add", not enabled). Flag to Vinny before ever
  turning it on — it's the natural fit for Wander Together's multi-person trips,
  but that's a cost decision, not an engineering one.
- `group_*` events belong to **Wander Together** — already correctly implemented
  under that module in this repo (`together_*` and `group_*` events in
  `src/utils/analytics.ts`). `conversation_*` events and `generate_lead` are
  **deleted** from the plan — confirmed not present anywhere in this repo.

## Booking API policy

Two separately metered tracks. The entire numeric structure applies to
**flights only**: 1,500 free searches/month, a hard look-to-book ceiling of 1,500
searches per completed flight booking, $0.005 per search beyond that. Hotels have
no free allowance, no numeric ratio, and no overage price — only a qualitative
"reasonable ratio" monitored by trend. Never apply the flight numbers to hotels.

## Live PostHog project (checked 2026-08-09)

- Project 445316, US Cloud region, org "WhereTo" (or similar).
- Current usage this billing cycle: ~8.6K / 1M free product-analytics events
  (projected ~19.6K) — nowhere near the free tier. Web session replay 75/5,000,
  mobile replay 0/2,500. No credit card on file.
- Group analytics: off (confirmed via Billing → Add-ons, shows "Add").
- Two existing dashboards: **My App Dashboard** and **Core Funnels**
  (auto-created) — do not touch without checking what they're built on first;
  `Core Funnels` already has at least one insight using `flight_booked`.
- Project **timezone and data region were not checked** — PostHog gates those
  settings behind a step-up re-authentication this session couldn't complete on
  Vinny's behalf. Confirm both once, in person — timezone in particular defines
  every "single day" number in the plan and can't be changed later without
  breaking historical comparability.

## api_call_logged — DEPLOYED and confirmed working (2026-08-10)

Vinny ran the 4 `supabase functions deploy` commands. Verified end-to-end
this session: `get_edge_function` on `liteapi-flights` read back version 29
containing the logging code, then a live test call to `liteapi-data`
(`action:'hotels', cityName:'Paris'`, called directly via curl with the anon
key — no app involved) returned real hotel data AND produced exactly one
`api_call_logged` row in PostHog seconds later: `distinct_id:
liteapi-data-server, search_track: hotel, action: hotels, status_code: 200`.
The pipe works. Nothing further needed here — the section below is kept for
history/context only.

## api_call_logged — written, NOT deployed (needs a manual step) [RESOLVED — see above]

`api_call_logged` (server-side, `search_track` dimension) is now implemented in
all 4 outbound LiteAPI edge functions — `liteapi-flights` (`search_track:
'flight'`), `liteapi-book`, `liteapi-stays`, `liteapi-data` (all `'hotel'`).
`liteapi-webhook` was deliberately skipped — it only receives inbound webhooks,
it never calls LiteAPI itself, so there's nothing to log. Each function POSTs a
fire-and-forget event to PostHog's HTTP capture API (`{host}/capture/`), backed
by `EdgeRuntime.waitUntil()` so it never blocks or fails the actual booking
call. Every logged call carries `action` (search/verify/prebook/book/retrieve/
quote/rates/hotels/hotel/reviews) alongside `search_track`, so a PostHog query
can later filter to just the calls that count toward the flight quota once
that's confirmed against the Nuitee/LiteAPI contract — see the code comment in
each file. `cache_hit` is hardcoded `false` (no cache layer exists on any of
these paths yet, so this is accurate, not a placeholder) and `cost` is omitted
entirely (no per-call dollar figure is known/configured anywhere in this repo —
fabricating one would corrupt the Tier 3 "Cost per API Call" metric).

**This is NOT deployed.** The Supabase MCP connection this session had is
read-only — `deploy_edge_function` exists in its tool schema but the server
rejected the actual call ("Cannot deploy an edge function in read-only mode").
The code is committed to this repo but the LIVE edge functions on Supabase
still do not have it. 🔴 **DO THIS** — deploy the 4 functions yourself:

```
supabase functions deploy liteapi-flights
supabase functions deploy liteapi-book
supabase functions deploy liteapi-stays
supabase functions deploy liteapi-data
```

(Supabase CLI, logged in and linked to this project — or redeploy each one by
hand from the Supabase dashboard's Edge Functions page, pasting in the current
`supabase/functions/<name>/index.ts` content.) No new secrets are required —
the PostHog project token is hardcoded as a safe-to-expose default (same
write-only client token already public in the website's `site.json`), so
deploying with zero config changes works immediately. Optionally set
`POSTHOG_API_KEY` / `POSTHOG_API_HOST` as Supabase secrets later if the token
ever needs rotating.

## PostHog console configuration (2026-08-10) — Phase 6, partial

Built directly in the live project (445316) via the console (no Personal API
key available — Personal API keys require PostHog's Boost plan, $250/mo,
which Vinny doesn't have, so this was clicked through the UI rather than
scripted). What exists now:

- **Action** `booking_completed` — matches `flight_booked` OR `hotel_booked`.
  Confirmed against 12 real historical events immediately after saving.
- **Funnel** "Booking Funnel (partial — 4 of 8 steps live pending app
  release)" — `booking_started → passenger_info_entered → booking_reviewed →
  booking_completed`. 14-day conversion window (PostHog's default, matches
  spec). Real data: 50% conversion, 4 entries over 90 days. The other 4 steps
  (`search_results_returned`, `view_item`, `option_selected`,
  `payment_info_entered`, `booking_submitted`) don't exist in PostHog yet —
  **confirmed by testing**, not assumed: PostHog's funnel step picker only
  accepts events that have actually been ingested at least once, and typing
  an unseen event name returns "No results" with no override. Add the
  missing steps once an app release ships them.
- **Trends** "Booking-API Trends (by search_track)" on `api_call_logged` —
  real data confirmed (`hotel: 1` from the verification test call).
- **Dashboard** "Tier 1 KPIs" — pins the funnel, the Booking-API trend, "Total
  Bookings (Flights + Hotels)" (formula `A + B` on `flight_booked` +
  `hotel_booked`, real data — a spike to 16 in early August), and "App Active
  Users (platform=app)" (unique users, filtered `platform = app` — real data,
  `platform` already has both `app` and `website` values live).
- **Error tracking**: project-level "Enable exception autocapture" toggled
  ON (Settings → Error tracking). SDK-side `errorTracking: { autocapture:
  true }` added to `posthog.init()` in `src/utils/analytics.ts` — verified
  against `posthog-react-native`'s actual type definitions
  (`dist/error-tracking/index.d.ts`), not guessed; the option is
  `errorTracking.autocapture`, NOT `errorTrackingConfig.autoCapture` as an
  earlier draft of this doc said (that was the Config Guide's paraphrase, not
  the real API in the installed SDK version, 4.46.2). dSYM (iOS) / mapping
  file (Android) upload into the EAS build pipeline is still **not** done —
  without it, captured exceptions land in PostHog but stack traces stay
  unsymbolicated. That's real CI/build-config work outside this session's
  visibility into your EAS setup.

**Deliberately NOT built tonight** — each of these is a real further session,
not a quick add:
- The three category dashboards (A — WhereToTrips.com, B — WhereTo App, C —
  Overall Travel) from the Summary KPIs by Category tab. Most of those
  metrics' underlying events aren't live in production yet.
- Module retention (`module_view` isn't live yet) and Paths starting from
  `module_view` (same reason).
- Alerts (funnel conversion rate, flight searches crossing 1,200/month) —
  didn't want to configure a threshold alert against data that's 90% not
  live yet; do this once the funnel's remaining steps are live.
- Any Look-to-Book ratio / overage-cost / free-tier-utilization insight —
  these require joining `api_call_logged` counts (PostHog) against completed
  bookings (Internal DB), which needs the internal Postgres DB linked as a
  PostHog data warehouse source. Confirmed via the SQL editor: "No data
  warehouse sources connected." That's a real setup step, not a formula.
- Vibe Match Acceptance Rate, NDCG@k, MRR — checked the internal DB directly
  (`vibe_destination_rankings`, 7,937 rows): it's a static precomputed
  region/subregion/vibe → destination scoring table, with **no** `user_id`,
  `search_id`, or `position` column — not a per-search impression log. NDCG
  and MRR need to know exactly what was shown, in what order, to whom, for
  which search, and nothing today records that. This isn't a wiring gap like
  everything else on this list — it needs a relevance-labeling decision
  (implicit vs. explicit, already flagged as open in the original plan) and
  a new persisted log before any ranking-quality metric is computable at
  all. Real, unstarted feature work.

## Known gaps not fixed in the 2026-08-09/10 pass

These are real, audited gaps — listed so nobody assumes they're covered:

- `payment_info_entered` / `booking_submitted` / `booking_failed` were **added**
  (flight + hotel), with a `booking_id` sourced from `booking_in_progress.id`
  (the staging row, one per user) threaded through the whole funnel. Not yet
  live in production — requires an app release.
- `search_initiated` / `search_results_returned` / `search_zero_results` were
  **added**, additively alongside the pre-existing `search_submitted` /
  `search_results_loaded` (kept as-is, not renamed, so nothing already built on
  them breaks).
- `search_result_click` — **added** 2026-08-10, `app/search/results.tsx`
  DestinationCard, two call sites (`onOpenDetail` → `click_type:'detail_sheet'`,
  `onExplore` → `click_type:'blog'`). Reading the blog counts as a click, not
  a separate concept — see below.
- `search_abandoned` — **added** 2026-08-10, in `results.tsx`. Design note
  (Vinny asked for a straight 30-min inactivity timer; talked through why
  that's wrong and built this instead): a plain foreground timer would almost
  never fire for the single most common abandonment case — someone looks at
  results and just leaves the app, which is exactly when RN suspends JS
  timers. Built as a mount/unmount effect instead: fires when the results
  screen goes away (back/tab-switch/into a booking) with no engagement,
  where "engagement" is opening a card's detail, reading its blog,
  favoriting, checking a flight/hotel option, or booking — deliberately NOT
  "dismiss" (that's a rejection signal, not disengagement). Keyed off
  `currentSearchId` for when a genuinely new search replaces this one.
  **Correction, same day:** an earlier version of this note claimed a
  refinement ("Update results") mints a new `currentSearchId` and so
  separately marks the old search abandoned — checked `rerunSearch()` in
  `appStore.ts` and that's wrong. It re-prices the SAME curation under the
  SAME `currentSearchId` by design (no re-curation on refine), so a
  refinement is correctly one continuous search session, not two — the
  abandoned effect doesn't fire on refine, which is the right behavior, just
  not for the reason originally written down.
  Known gap: doesn't catch OS-level backgrounding while the screen stays
  mounted (component never unmounts, cleanup never runs) — no timer
  fallback was added for that, since a wrong-but-present number reads as
  more trustworthy than it is; better to leave it as a known blind spot.
- Blog reads were already tracked before this pass (`destination_explore_viewed`
  in `destination-explore.tsx`, `blog_viewed` in `vibe-explore.tsx` — two
  events because two different screens serve the blog depending on whether
  the card came from a VibeMatch-scored search) — Vinny asked "are we
  tracking this," answer was yes, but neither carried `search_id`. Both now
  do, via a `searchId` query param on the `router.push` from results.tsx
  (chosen over reading the store directly, since these are separate routed
  screens and a param is more robust to timing than assuming store state
  hasn't moved on).
- `search_refinement` — **added** 2026-08-10, `results.tsx`'s
  `runRefinedSearch` (the "Update results" button), `changed_fields` built
  from the existing dirty-dates/budget/travelers flags. `original_search_id`
  and `new_search_id` are always equal in this app, since (see above)
  refining doesn't mint a new search_id — the taxonomy's two-distinct-IDs
  shape assumes every refinement is a brand new search, which isn't this
  app's actual model. Fires anyway; `changed_fields` is the useful part.
- `discover_card_save` — **done** (renamed from `favorite_added`, see above).
- `module_view` / `module_exit` / `module_to_module_nav` — **added**,
  centrally in `app/_layout.tsx`'s `ScreenTracker`, mapped by route prefix
  (`routeToModule()`). Verify the mapping once real navigation data is in —
  route-prefix mapping is a reasonable first cut, not verified against real
  usage.
- `sign_up` / `login` / `login_failed` / `logout` on WhereToTrips.com — **added**
  in `wt-auth.js` (signup + login) and `wt-profile.js` (the shared consumer
  logout handler, which also re-registers `platform`/`surface` after
  `posthog.reset()` — the same bug class fixed in the app). Deliberately NOT
  added to the partner-dashboard or admin-dashboard logout handlers
  (`wt-partner-shared.js`, `wt-admin*.js`) — those are internal/affiliate
  surfaces, not real website visitors, and would pollute category A metrics.
- `api_call_logged` — **deployed and confirmed working**, see the dedicated
  section above.
- PostHog error tracking — **both switches on** as of 2026-08-10, see the
  PostHog console configuration section above. dSYM/mapping-file upload into
  EAS is the one piece still outstanding.
- `analytics/taxonomy.yaml` in this repo is a first draft transcribed from the
  workbook's Event Taxonomy Reference tab — treat it as a starting point for
  Phase 2 (generated typed emitters + CI validator), not yet wired into any
  build step.

## Live-device verification (2026-08-10) — real build, real findings

Vinny installed 0.3.482 (build 2 / versionCode 687) on his phone and ran a
real Discover search. Checked PostHog directly rather than trusting the
release worked — found two real issues, both fixed:

- **Discover-originated searches never got a `search_id`.**
  `app/search/discover.tsx` deliberately doesn't call `curateWithVibeMatch`
  (it has its own candidate-assembly logic, see the code comment there) — it
  calls `confirmSearch()` directly. `search_initiated` and `currentSearchId`
  are both set inside `curateWithVibeMatch`, so neither ever ran for Discover,
  and `search_results_returned` fired with a null `search_id`. Confirmed live:
  `module_view`/`module_exit`/`module_to_module_nav` all fired correctly for
  the Discover→Explore transition, but no `search_initiated` row existed and
  `search_results_returned` had an empty `search_id` column. **Fixed** —
  `discover.tsx` now generates its own `search_id`, sets
  `useAppStore.setState({ currentSearchId })`, and fires `search_initiated`
  itself, right before the same `confirmSearch()` call.
- **`search_result_click` (blog) and the downstream `blog_viewed` each fired
  twice** for one real tap — confirmed via distinct event UUIDs, not a
  display artifact (same `destination_id`, same `position`, ~1s apart).
  Traced to `SwipeableCard` in `results.tsx`: `onExplore` is wired to both a
  swipe gesture (`runOnJS(onExplore)()`) and a plain tap target, and a tap
  that also registers as a small pan fires both. **Not fixed at the root** —
  that's real gesture-handler work (tap-vs-pan ambiguity) needing hands-on
  device testing this session can't do. **Mitigated** at the capture layer
  instead: `onExplore`'s whole handler body (capture + navigation) now
  short-circuits if the same `destination_id` fired within the last 2
  seconds (`lastExploreCaptureRef` in `results.tsx`) — fixes the double
  `search_result_click` and, since it also blocks the duplicate
  `router.push`, the downstream double `blog_viewed` too.
- Confirmed working correctly, no changes needed: `surface` present and
  correct (`whereto_app_ios`) on `$screen` and `search_result_click`.
  `api_call_logged` firing live from real searches
  (`liteapi-flights-server`/`liteapi-data-server`) with `search_track`.
  `module_view`/`module_exit`/`module_to_module_nav` all correct for the
  Discover→Explore transition.
- **Not yet re-verified**: whether the two fixes above actually work — they
  haven't been through a build yet. Version bumped to 0.3.483 / build 3 /
  versionCode 688; needs a new TestFlight build and one more live check
  before trusting `search_id` threading and click-dedup on Discover-sourced
  data.
- **Fixed 2026-08-10**: `blog_viewed`'s own `surface` property (an entry-point
  label like `'destination_results'`, set explicitly in `vibe-explore.tsx`,
  predates this project) shared its name with the taxonomy's reporting
  `surface` super property. Event-level properties win over registered super
  properties with the same key, so `blog_viewed` events were reporting
  `surface: 'destination_results'` (or similar) instead of
  `whereto_app_ios`/`whereto_app_android` — `blog_viewed` couldn't be
  correctly split into category A/B on that property. Renamed the local
  route param and event property to `entry_point` throughout
  `vibe-explore.tsx` and its 4 callers (`app/together/[code].tsx`,
  `app/saved.tsx`, `src/components/DashboardHero.tsx`, `app/group/[id].tsx`
  x2) — `blog_viewed` no longer sets `surface` at all, so the standing
  super property passes through untouched. App isn't live yet, so no backward
  compatibility was needed for the old `?surface=` param name.
- **Re-verified 2026-08-10 via Metro (`npx expo start -c`, both Android and
  iOS 0.3.483)**: both fixes above hold up on real devices. `search_result_click`
  and `blog_viewed` fired exactly once each, with `search_id` threaded
  correctly end to end, on a Discover search (Android) and an Explore search
  (iOS, "Soufrière Saint Lucia"). Confirmed via direct PostHog SQL, filtering
  each device's session by `surface`.
- **Fixed 2026-08-10**: a fresh Android hotel booking (Anchorage) surfaced a
  `hotel_booked` double-fire — one copy from `HotelPrebookView.tsx`, a second
  from `bookingInProgressStore.ts`'s `markHotelBooked` (which had no
  `booking_id` at all, since the store has no reason to know it beyond its
  own `staging.id`). Same shape as the `search_result_click` bug: two
  independent capture sites for one logical event. Consolidated into
  `confirmHotelBooking()` (`src/utils/hotelBooking.ts`) — the one function
  every hotel-booking surface (`HotelPrebookView`, `HotelWizard`) already
  funnels through — so it fires exactly once regardless of caller, and
  removed both of the old capture sites.
- **Fixed 2026-08-10**: `passenger_info_entered` and `booking_reviewed` never
  fired for hotel bookings at all — `HotelPrebookView.tsx` only had
  `payment_info_entered`/`booking_submitted`/`hotel_booked`/`booking_failed`.
  Confirmed via the same Anchorage test. Added both, mirroring
  `FlightPrebookView.tsx`'s pattern exactly (ref-guarded effect on guest-info
  completeness; capture on Continue from the Review step).
- **Fixed 2026-08-10**: `booking_started` was gated on `if (flight && ...)`
  in `app/bookings/passengers.tsx`, so it never fired for a hotel-only
  booking. Re-investigated properly this time — grepped for actual
  `<FlightPrebookView`/`<HotelPrebookView` JSX (not comment references)
  across the whole repo, and both are **only ever mounted from
  `passengers.tsx`**; `HotelDetailViewV2` and `HotelWizard` only reference
  them in comments, they don't render them (`HotelWizard.tsx`'s own header
  even says "kept only as a fallback"). So the "bypassed passengers.tsx
  entirely" theory from the Anchorage test writeup was wrong — that booking
  did go through passengers.tsx, it just never satisfied the flight-only
  gate. No double-fire risk to design around: `passengers.tsx` is the single
  real mount point for both PrebookViews. Rewrote the effect to fire once
  per screen visit whenever either leg becomes bookable (flight-only,
  hotel-only, or a package), with `has_flight`/`has_hotel`/`product_scope`
  properties instead of just `has_hotel`. The New Haven package test earlier
  tonight already showed `booking_started` firing correctly for a
  flight+hotel booking (it has a flight leg, so it always satisfied the old
  gate) — this fix specifically covers the hotel-only path that test didn't
  exercise. **Not yet re-verified with a hotel-only booking** — do that next.
- **Investigated 2026-08-10**: Vinny flagged "Active users: 9" on
  wheretotrips.com/analytics/ (App tab) as implausible — only 2 real
  testers (Vinny across 2 phones, Chris). Confirmed the 9 was real
  distinct `distinct_id`s, not a bug in what's counted (server-side
  `api_call_logged` events correctly don't set `platform: 'app'`, so
  they're excluded). Root cause: every genuinely different app install
  (5 different `$app_version` labels showed up in one day's data,
  including stale ones like `0.1.0`/`0.3.343` — old dev-client shells
  still being opened instead of the current one) gets its own fresh
  anonymous ID until `identify()` merges it — completely normal, expected
  SDK behavior, not a persistence bug. Checked the actual persistence
  config too: `posthog-react-native`'s storage fallback chain
  (`native-deps.js`) picks `expo-file-system`'s File API first, which
  this project has (SDK 56 core), so storage isn't silently falling back
  to in-memory. Checked `reset()` — only called from `authStore.ts`'s
  `logout()`, not over-firing. Checked `identify()` — called on login AND
  on `hydrateAuth()`'s cold-start session restore, exactly as it should
  be. No code-level persistence bug found. **Fixed**: the dashboard's own
  math was doing `count(DISTINCT distinct_id)` instead of
  `count(DISTINCT person_id)` for every "users" figure on the App and
  Travel Metrics tabs (`supabase/functions/admin/index.ts`) — switched
  all of them to `person_id`, which resolves through PostHog's identify()
  merges. Confirmed live: 9 raw distinct_id vs 5 person_id for the same
  day. 5 still isn't 2, but the remainder is expected too — pre-login
  anonymous testing (e.g. exercising onboarding logged out) never merges
  retroactively. The Website tab intentionally keeps `distinct_id` —
  wheretotrips.com visitors are mostly anonymous and never identify(), so
  "unique visitors" is supposed to mean distinct anonymous sessions, not
  logged-in people.
- **Fixed 2026-08-10**: Vinny noticed wheretotrips.com/analytics/'s App tab
  had no hotel data at all — its "Flights booked" stat and "Booking funnel
  (solo)" were both hardcoded to `flight_booked` with no hotel equivalent.
  Confirmed hotels WERE being tracked correctly the whole time (Travel
  Metrics tab's Bookings/Pre-Bookings cards already showed real hotel
  numbers) — this was a dashboard-layout gap on the App tab specifically,
  not a tracking gap. Added a "Hotels booked" stat card next to "Flights
  booked", and split "Booking funnel (solo)" into two real funnels:
  "Booking funnel — Flight" and "Booking funnel — Hotel"
  (`supabase/functions/admin/index.ts`, `analyticsOverviewApp`). The first
  3 steps (`booking_started`, `passenger_info_entered`, `booking_reviewed`)
  share event names between flight and hotel, so a flat per-event-name
  count would have blended both products into one misleading number — the
  new query uses `countIf` with `properties.product_scope` (booking_started
  — 'flight'/'hotel'/'package', a package counts toward both funnels' step
  1) and `properties.product_type` (the other two steps — added to the
  flight side this same commit; the hotel side already had it from the
  earlier `product_type` fix tonight). Historical booking_started events
  from before this fix have no `product_scope` at all, so the split
  funnels' step 1 will read 0 for older data — expected, not a bug; new
  bookings after this deploy populate correctly.
- **Added 2026-08-10**: Vinny wanted flight add-on revenue visible — bags
  and seats earn commission. Itemized `bag_count`/`bags_revenue`/
  `seat_count`/`seats_revenue`/`extras_total`/`fare_total` into
  `flight_booked` (the actual purchase — this is what ties to real earned
  commission), plus `booking_submitted` and `booking_reviewed` for
  funnel-stage visibility (`FlightPrebookView.tsx`, split out of the
  already-computed `extrasTotal`/`bagCount`/`seatCount` in its "Extras
  maths" `useMemo`). New "Flight Add-ons" card on the Travel Metrics tab
  (`analyticsOverviewTravel`) sums these via `sum(toIntOrZero(...))`/
  `sum(toFloatOrZero(...))` — HogQL's cast functions, confirmed by testing
  directly (ClickHouse's own `toInt64OrZero` etc. aren't valid there).
  Confirmed no hotel equivalent exists to track: LiteAPI/Nuitee's board
  type (breakfast, half-board) is a meal-plan attribute on the room rate
  itself, not a separately purchasable, separately commissionable service.

## Genuinely still open after tonight (2026-08-10)

Everything else on the original list has been closed. What's left, in
priority order:

1. **The three category dashboards (A/B/C)**, module retention, paths, and
   alerts — not a wiring gap, just not done. Most of the underlying events
   need an app release first (see "PostHog console configuration" above for
   what already exists: Tier 1 KPIs dashboard, the partial funnel,
   Booking-API Trends).
2. **Look-to-book ratio / overage-cost / free-tier-utilization metrics** —
   need the internal Postgres DB linked as a PostHog data warehouse source
   (confirmed not connected). Setup step, not a formula.
3. **Vibe Match Acceptance Rate, NDCG@k, MRR** — real, unstarted feature
   work: needs a relevance-labeling decision (implicit vs. explicit, flagged
   as open in the original plan) and a new persisted per-search impression
   log (checked `vibe_destination_rankings` directly — it's a static
   precomputed table, not a log of what was shown to whom).
4. **`search_refinement`'s two-distinct-IDs shape** doesn't match this app's
   actual "Update results" behavior (see notes above) — cosmetic mismatch
   with the taxonomy, not broken, but worth a product decision on whether to
   change `rerunSearch()` to mint a new search_id, or just accept that this
   app's refinement is a re-price-in-place rather than a new search.
5. **dSYM / mapping-file upload** into the EAS build pipeline, so captured
   exceptions symbolicate.
6. An app release, to get `payment_info_entered`, `booking_submitted`,
   `booking_failed`, `module_view`, `search_result_click`,
   `search_abandoned`, `search_refinement`, and `discover_card_save` into
   actual production traffic — nothing added in this repo since 2026-08-09
   is live yet.
