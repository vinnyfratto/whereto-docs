# Hotel Detail & Booking UX — Expedia + Booking.com Teardown vs. WhereTo Gap Analysis

**Purpose:** Document, section by section, what Expedia and Booking.com *display* and *ask the user to input/select* on a hotel detail page and through booking. Cross-reference against WhereTo's current hotel detail/booking flow, and against what LiteAPI actually returns (see `docs/liteapi/HOTELS.md` and the LiteAPI sample-pull audit, 2026-07-16), so every gap below is tagged with **whether the data already exists somewhere in our pipeline** — that's the fastest lever for prioritizing fixes.

**Scope note:** This is a gap-identification exercise, **not** a spec to replicate either platform. Companion doc to `docs/BOOKING_FLOW_GAP_ANALYSIS.md` (flights) — same method, hotel side.

**Researched:** 2026-07-16.
- **Booking.com:** live crawl of the Waldorf Astoria New York property page (search → hotel page → room/rate grid → reviews → house rules), Sep 10–12 2026, 2 adults. Content below is transcribed from that live page.
- **Expedia:** expedia.com and hotels.com both served a bot-detection interstitial ("Bot or Not?") to the automated browser and to a direct WebFetch — no live crawl was possible this session. Expedia's structure below is grounded in current (July 2026) public documentation (One Key member pricing, sustainability badges, cancellation-policy help pages — cited in §8) plus the well-established, slow-changing OTA page skeleton. Treat Expedia specifics as representative, not pixel-exact.
- **WhereTo:** read directly from `src/components/HotelDetailView.tsx`, `src/components/HotelWizard.tsx`, `src/providers/hotels/LiteApiProvider.ts`, `src/utils/hotelBooking.ts`.
- **LiteAPI data availability:** cross-referenced against `docs/liteapi/HOTELS.md` and the live sandbox pulls captured in `LiteAPI_Sample_Pull_FULL.xlsx` (2026-07-16) — 5 real hotels, `/hotels/rates` + `/data/hotel` responses fully flattened field-by-field.

---

## 1. BOOKING.COM — section by section (live crawl)

### 1.1 Overview / hero
Photo count teaser ("+98 photos"), overall review score badge ("9.2 Wonderful", review count), guest quote snippets pulled from real reviews, full address, **"Excellent location – rated 9.8/10" walking-distance-to-transit callout** ("2,150 ft from Grand Central Station"), "We Price Match" badge, **sustainability certification** badge (with detail: ISO 14001/50001/9001, certifying body named), **"Property highlights"** quick-glance icon row (Breakfast / Parking type [Valet, EV charging] / Pet policy teaser / Wellness), hotel chain/brand label, long-form description with landmark distances woven into the prose, **segment-specific rating callout** ("Couples rated the location 9.8 for a two-person trip"), a bespoke **"Perfect for a 2-night stay!"** recommendation card (location score + "very comfy beds" highlight + breakfast/parking blurbs).

### 1.2 Amenities
"Most popular amenities" top-10 icon list, full amenities tab, **on-site restaurants listed by name + cuisine** (distinct from nearby restaurants), facilities-for-disabled-guests flag.

### 1.3 Room / rate grid ("Info & prices")
Per room type: name, bed configuration, short description, room category label ("Private suite"), **size in ft²**, quick amenity icons (Bath/AC/TV/Minibar/WiFi) + "More" expander. **Multiple rate plans per room type shown as separate rows** — e.g. the same "Lexington Avenue Junior Suite" appeared 4 times with different combinations of breakfast (included/optional-$60) × cancellation (non-refundable/free-cancellation) × pay timing (pay online/pay at property). Each row: price per night AND total for stay, **taxes/fees explicitly excluded and itemized** ("Excluded: US$3.5 City tax per night, 14.75% TAX"), **explicit free-cancellation deadline date**, **scarcity messaging** ("We have 3 left"), a **quantity selector with a live running total per quantity** (1 room = $X, 2 rooms = $2X, …).

### 1.4 Reviews
Overall score + **category sub-scores** (Staff, Facilities, Cleanliness, Comfort, Value for money, Location, Free WiFi — each 0–10), **topic filter chips** (Room/Location/Breakfast/Luggage/Clean) to jump to relevant reviews, individual review cards (reviewer name, country, quote, "Read more"), **"Travelers are asking" Q&A** (auto-surfaced common questions), **"Earn a verified guest badge"** identity-verification gamification.

### 1.5 Area info
Interactive map, **top attractions with exact distances** (Rockefeller Center 1,650 ft, Empire State Building 1.1 mi, …), **restaurants & cafés nearby** (name + distance), natural features (parks/lakes), **public transit options with distance** (subway lines, train stations), **closest airports with distance**.

### 1.6 House rules
Check-in window (start–end) + **ID/credit-card-required notice**, check-out time, general cancellation/prepayment disclosure, **Children & Beds**: age policy prose + **structured crib/extra-bed pricing by age band** ($250/night for 3+ years, free crib), **minimum check-in age** (21 at this property), **pet policy** (allowed, free), **group-booking threshold** (9+ rooms → different policy), **accepted card types + cash-acceptance flag**, "The fine print" freeform legal notices.

### 1.7 FAQs
Auto-generated Q&A block at the bottom (type of room, price, restaurant on-site, distance to center, etc.).

---

## 2. EXPEDIA — section by section (grounded in current docs; not live-crawled this session)

### 2.1 Overview / hero
Photo gallery with **virtual tour** entry point, guest rating (out of 5 or 10) + review count, **review category breakdown** (cleanliness, service, amenities/condition), "Popular amenities" quick icons, description, map with distance to landmarks/airport, **"Eco-Certified" sustainability badge** — property-supplied practices (solar panels, recycling, towel-reuse program) with an explicit disclaimer that Expedia does not independently verify the claim.

### 2.2 Room / rate cards
Room card: photo, bed type, **"Sleeps X"**, room size, amenity icons. **Multiple rate options per room** (refundable vs non-refundable, breakfast toggle) shown as selectable variants. **Member Price**: One Key members see a second, discounted price line automatically (10%+ off; Silver/Gold/Platinum tiers 15–20%+ off — confirmed current via Expedia's own member-price page, July 2026). Price shown with strikethrough original + "X% off" where applicable. Taxes & fees disclosed in an expandable price-breakdown drawer. **"Reserve now, pay later" / "Book now, pay later"** badge on eligible rooms — no deposit, pay at check-in, still earns OneKeyCash. **OneKeyCash rewards** earned per stay, redeemable on future bookings.

### 2.3 Reviews
Score + category breakdown, individual traveler reviews (often with **traveler-uploaded photos**), verified-stay badge.

### 2.4 Policies / "Fees & policies"
Check-in/out windows, deposit/ID requirements, pet policy, children policy, "know before you go" freeform notices — functionally similar structure to Booking's House Rules, less granular on crib/extra-bed pricing.

### 2.5 Cross-sell / trust
**"Bundle & Save"** flight+hotel package upsell (Expedia's own flights inventory), **similar-hotels carousel**, price-guarantee messaging ("Hotel Price Guarantee — find it cheaper, get the difference back"), loyalty-tier pricing baked into the room grid rather than a separate step.

---

## 3. Element inventory — union of both platforms

### 3.1 DISPLAYED
- Photo gallery with count badge / virtual tour
- Star rating (official) + guest score + review count
- Review **category sub-scores** (cleanliness / service / location / comfort / value / amenities / wifi)
- Individual reviews: name, country/date, quote, pros/cons, sometimes photos
- Review topic filter chips
- Full address + **map with distance to named landmarks, transit, restaurants, airport**
- **On-site restaurant list** (name + cuisine)
- Sustainability / eco-certification badge
- Loyalty-program discounted pricing (One Key / Genius)
- Hotel chain/brand label
- "Most popular amenities" quick list + full expandable list
- **Multiple rate plans per room type side by side** (refundable × board × pay-timing combinations)
- Per-room: photo, bed configuration, **size**, "Sleeps X", short description, amenity icons
- Price per night AND total, **taxes/fees itemized and disclosed separately from the total**
- Explicit **free-cancellation deadline date**
- **Scarcity messaging** ("Only 3 left")
- Room **quantity selector** with running total
- "Reserve — you won't be charged yet" / pay-later messaging
- Price-match / best-price guarantee messaging
- Q&A / FAQ block
- Bespoke bundle/recommendation cards ("Perfect for a 2-night stay")
- Similar/alternative hotel carousel
- Package (flight+hotel) upsell
- Structured **house rules**: check-in/out, ID+card requirement, minimum check-in age, children/crib/extra-bed pricing by age, pet policy + fees, group-booking threshold, accepted payment types, cash acceptance

### 3.2 USER-INPUT / selection elements
- Date/occupancy search bar (rooms, adults, children)
- Room + rate-plan selection (which of the N variants)
- Room quantity per rate
- Special requests free-text
- Lead guest name, email, phone
- Loyalty sign-in for member pricing
- Payment method (card / Apple-Google Pay / pay-later)
- Consent to cancellation policy / T&Cs before booking

---

## 4. WhereTo today — what we have

From `src/components/HotelDetailView.tsx`, `src/components/HotelWizard.tsx`, `src/providers/hotels/LiteApiProvider.ts`, `src/utils/hotelBooking.ts`:

**Detail screen (`HotelDetailView.tsx`):** photo carousel with a full-screen zoomable gallery (pinch/double-tap/swipe) *and* a toggle to an interactive Leaflet map showing airport ✈ + hotel 🏨 pins (a feature neither OTA offers in quite this form — a differentiator); star-rating row + rating badge (color-coded by score band) + review count; a room list (`RoomRow`) showing name, board type, "Sleeps X", refundable/non-refundable label, price per night + total; description with "Read more" expand; amenities icon-grid (keyword-mapped to Solar icons) showing first 16 with "Show all N" expander; a reviews section with overall score, **category score bars** (up to 6), **pros chips**, and up to 4 individual review cards (score, name, date, headline, pros, cons); an "Important Information" callout; sticky Book Now bar.

**Booking wizard (`HotelWizard.tsx`, 4 steps):** Step 0 room selection (`RoomCard` — name, price/night + total, board-type badge, refundable badge with cancel deadline, "Up to N adults" badge, first 5 amenities as chips); Step 1 stay preferences (arrival-time chips, smoking preference, special-requests free text, lead guest first/last name + email); Step 2 policies (check-in/out times, cancellation policy card, "What's Included" board+amenity checklist, hotel facilities grid capped at 12, Important Information callout, hotel description); Step 3 confirm (hotel + room + guest summary, **itemized price breakdown** — room rate × nights, taxes & fees, total — consent checkbox, "Book & Pay").

**Booking (`hotelBooking.ts` / `LiteApiProvider.ts`):** re-quotes fresh rates immediately before booking (LiteAPI offer IDs are short-lived), surfaces a price-change alert if the re-quote drifted, prebooks, then books with `ACC_CREDIT_CARD` (LiteAPI's own account-card method — no native Stripe UI; see `docs/liteapi/PAYMENTS.md` §12.6 for why that was tried and reverted).

---

## 5. GAP TABLE

Legend: ✅ have · 🟡 partial · ❌ missing. **LiteAPI data?** tells you the fastest lever: **Y** = already in the API response, just needs mapping/UI (cheap fix); **Y (fetched, unused)** = we already pull this field into the app and still don't show it (cheapest possible fix); **Partial** = some structured signal exists but not the full picture; **N** = not in LiteAPI's schema at all — needs a different data source or is out of scope.

| Feature | Expedia | Booking.com | WhereTo | LiteAPI data? | Notes |
|---|:---:|:---:|:---:|:---:|---|
| Photo gallery (zoomable, full-screen) | ✅ | ✅ | ✅ | Y | `HotelDetailView.tsx` gallery is arguably *nicer* than either OTA (pinch-zoom, mosaic grid) |
| Star rating + guest score + review count | ✅ | ✅ | ✅ | Y | Mapped from the `/data/hotels` search-list endpoint, not `/data/hotel` — see `HOTEL_OTA...` note in the LiteAPI audit |
| Review **category sub-scores** | ✅ | ✅ | ✅ | Y (via `/data/reviews?getSentiment=true`, a sibling call — not `/data/hotel`'s own `sentiment_analysis`) | `HotelDetailView.tsx` `catList` — genuinely close to Booking's category-bar UI |
| Individual review cards (name/date/headline/pros/cons) | ✅ | ✅ | ✅ | Y | Capped at 4 shown on detail, 8 fetched — reasonable parity |
| Review topic filter chips (Room/Location/Breakfast…) | ❌ | ✅ | ❌ | Partial | We have the review text (pros/cons/headline) to build this from; just no filter UI. Cheap-ish to build client-side |
| Interactive map | ✅ | ✅ | ✅ (airport-only) | Partial | Our Leaflet map only plots airport + hotel; no landmarks/transit/restaurants overlay |
| **Nearby attractions / restaurants / transit / airport distances ("area info")** | ✅ | ✅ | ❌ | **N — not observed in our sandbox pulls** | HOTELS.md's own reference claims `/data/hotel` returns `poi[]{name,category,distanceKm,importance}`, but **none of our 5 live captured hotel-detail responses included a `poi` key** — verify against production before relying on this; if absent there too, this needs a different data source (Google Places, etc.) |
| **On-site restaurant list (name + cuisine)** | ✅ | ✅ | ❌ | N | No structured restaurant field anywhere in `/data/hotel`; would have to be parsed out of `hotelDescription` prose, unreliable |
| Amenities: "most popular" + full expandable list | ✅ | ✅ | ✅ | Y | `HotelDetailView.tsx` amenities grid (16 shown, "Show all N") — good parity |
| **Multiple rate plans per room shown side by side** (refundable × board × pay-timing) | ✅ | ✅ | 🟡 | Y | `LiteApiProvider.ts` groups by `(name, boardType, refundable)` and keeps the *cheapest per unique combo* — so distinct refundable/non-refundable or board variants of the same room name **do** appear as separate rows today. What's missing vs. the OTAs: no "pay online vs. pay at property" distinction, and only one rate per combo (not every rate the API returned) |
| Room photo (per room type) | ✅ | ✅ | ❌ | **Y (fetched, unused)** | `/data/hotel` `rooms[].photos[]` exists (4 photos on the sample room type we pulled) — entirely unused. Cheapest visual win available |
| Room size (sqft/m²) + bed configuration | ✅ | ✅ | ❌ | **Y (fetched, unused)** | `/data/hotel` `rooms[].roomSizeSquare` + `bedTypes[]` — available, unused. `RoomCard`/`RoomRow` show board+refundable+occupancy but never size or bed type |
| Taxes/fees **itemized and disclosed separately** from the total | ✅ | ✅ | 🟡 | Y | `LiteApiProvider.ts:200-204` **adds** any not-included tax/fee straight into the displayed total rather than disclosing it as a separate "+$X due at property" line the way both OTAs do — less transparent, same underlying data |
| Scarcity messaging ("Only 3 left") | ✅ | ✅ | ❌ | **N — not observed** | No rooms-remaining count field surfaced in `/hotels/rates` (unlike flights' `fare.seatsRemaining`); would need to confirm with LiteAPI whether any inventory signal exists before promising this |
| Room quantity selector (book >1 room) | ✅ | ✅ | ❌ | Partial | LiteAPI's `book`/`rates` endpoints do support multi-room (`occupancies[].rooms`, multiple `guests[]`), but `HotelWizard.tsx` only ever books a single room — no quantity control in the UI |
| **"Compare-at" / savings price messaging** ("was $X, now $Y") | ✅ (member price) | 🟡 | ❌ | **Y (fetched, unused)** | `roomType.suggestedSellingPrice` / `offerInitialPrice` are fetched into `LiteApiProvider.ts` but never shown to the user anywhere. No loyalty program needed to use this — pure "you're saving $X vs. the reference price" messaging is sitting on the shelf |
| Loyalty/member discounted pricing | ✅ (One Key) | ✅ (Genius) | ❌ | N | No loyalty program exists in WhereTo; product decision, not a data gap |
| Sustainability / eco-certification badge | ✅ | ✅ | ❌ | **N** | No such field anywhere in the `/data/hotel` schema we captured — not fixable via better mapping |
| Segment-specific rating ("rated X for couples") | ❌ | ✅ | ❌ | N | Booking-proprietary breakdown; not in LiteAPI |
| **Structured house rules** (min check-in age, crib/extra-bed pricing by age, pet fees, group-booking threshold, accepted cards, cash acceptance) | ✅ | ✅ (very granular) | ❌ | **Y (fetched, unused)** | `hd.policies[]` (structured, typed: `POLICY_CHILDREN`, `POLICY_PETS`, etc.) plus `childAllowed`/`petsAllowed`/`groupRoomMin` booleans are all returned by `/data/hotel` and **confirmed not read anywhere** in `LiteApiProvider.ts#fetchHotelDetail()`. Everything currently shown comes from the single freeform `hotelImportantInformation` paragraph, which only *sometimes* happens to mention ID/deposit rules and never breaks out pets/children/groups into labeled sections |
| ID + credit-card-required-at-check-in notice | ✅ | ✅ | 🟡 | Y (folded into `hotelImportantInformation`) | Shown as an undifferentiated paragraph, not a labeled "Check-in requirements" section like Booking's House Rules tab |
| Price-match / best-price guarantee messaging | ✅ | ✅ | ❌ | N | Business-policy decision, not a data gap |
| Q&A / FAQ block | 🟡 | ✅ | ❌ | N | Would need its own content pipeline; not a LiteAPI field |
| Bespoke "Perfect for a 2-night stay" recommendation card | 🟡 | ✅ | ❌ | N | Proprietary recommendation logic on Booking's side |
| Similar/alternative hotel carousel | ✅ | 🟡 | ❌ | **Y (buildable today)** | We already have a working hotel search — this needs zero new API surface, just a "more hotels in this city" query + carousel UI |
| Package (flight + hotel) bundle upsell | ✅ | ❌ | ❌ | **Y (buildable today)** | WhereTo uniquely already runs **both** flight and hotel LiteAPI integrations — this is a differentiated opportunity neither OTA screenshot needed us to notice, but our own architecture makes it cheaper for us to build than it was for them |
| Itemized price breakdown at confirm | ✅ | ✅ | ✅ | Y | `HotelWizard.tsx` Step 3 — room rate × nights, taxes & fees, total. Solid parity |
| Consent checkbox before booking | ✅ | ✅ | ✅ | — | `HotelWizard.tsx` Step 3 `consentRow` — **we're ahead of the flight side here**, which has no consent gate at all per `BOOKING_FLOW_GAP_ANALYSIS.md` |
| Price-change / re-quote protection before charging | — | — | ✅ | — | `prepareHotelCheckout` + price-drift alert in `HotelWizard.tsx` — a real strength, not common to see called out explicitly on either OTA's own UI even though they surely do it server-side |

---

## 6. Prioritized recommendations

**Tier 1 — cheapest wins (data already fetched or already in the API response, just needs mapping + UI):**
1. **Room photos** — `rooms[].photos[]` from `/data/hotel` is fetched by nothing today; wire it into `RoomCard`/`RoomRow`.
2. **Room size + bed configuration** — `rooms[].roomSizeSquare` + `bedTypes[]`, same endpoint, same fix.
3. **"You're saving $X" messaging** — `suggestedSellingPrice`/`offerInitialPrice` are already sitting in `LiteApiProvider.ts`'s room objects; add one line of UI.
4. **Structured house rules section** — `hd.policies[]` + `childAllowed`/`petsAllowed`/`groupRoomMin` are fetched nowhere; this is the single biggest "we have the data, we're just not looking at it" gap (matches the room-name/occupancy mapping bug already filed separately). Would let us build a real "House Rules" section matching Booking's granularity (pets, children/cribs, groups, min age) instead of one paragraph.
5. **Itemize excluded taxes/fees separately** instead of silently folding them into the total (`LiteApiProvider.ts:200-204`) — same math, more transparent presentation.

**Tier 2 — buildable with zero new API dependency:**
6. Similar/alternative hotels carousel — reuse existing search.
7. Flight+hotel package upsell — reuse both existing LiteAPI integrations; a genuine differentiator given most competitors don't have both in one stack.
8. Review topic filter chips — derive from review text we already fetch.
9. Multi-room quantity selector in `HotelWizard.tsx` — LiteAPI's booking endpoints already support it.

**Tier 3 — needs a data source we don't have (verify before promising):**
10. Nearby attractions/restaurants/transit ("area info") — confirm whether `poi[]` is genuinely absent from LiteAPI in production, or only in this sandbox; if absent everywhere, this needs a supplemental source (Google Places API, etc.) before it can be built.
11. On-site restaurant list — same story, no structured field observed.
12. Room scarcity messaging — no rooms-remaining signal observed in `/hotels/rates`; ask Nuitee/LiteAPI directly whether one exists before designing around it.
13. Sustainability/eco badge — not in LiteAPI's schema; would need a separate certification data source if this is a priority.

**Where we already lead or are at parity:** the zoomable photo gallery + map toggle, review category bars + individual review cards, itemized confirm-step price breakdown, the pre-charge price-drift protection, and — notably — **the booking consent checkbox that the flight flow still lacks** (see `BOOKING_FLOW_GAP_ANALYSIS.md` §6). Worth preserving these rather than flattening toward either OTA's layout.

---

## 7. Sources
- Live crawl: `booking.com` search + Waldorf Astoria New York property page, 2026-07-16 (checkin 2026-09-10, checkout 2026-09-12, 2 adults)
- [Expedia — Get Member Prices](https://www.expedia.com/member-price)
- [One Key Changes on July 28 — UpgradedPoints](https://upgradedpoints.com/news/one-key-rewards-changes-2026/)
- [Expedia — Unlock instant discounts with Member Prices](https://www.expedia.com/lp/b/member-prices-education)
- [Expedia's One Key Rewards Program — UpgradedPoints](https://upgradedpoints.com/travel/expedia-one-key-rewards-program/)
- [Expedia — Book Now, Pay Later Hotels](https://www.expedia.com/lp/b/book-now-pay-later)
- [Expedia — Finding More Sustainable Travel Options](https://www.expedia.com/lp/b/more-sustainable-travel)
- `docs/liteapi/HOTELS.md` (internal — API schema reference)
- `LiteAPI_Sample_Pull_FULL.xlsx` (internal — 2026-07-16 live sandbox field audit, 5 hotels)
- `src/components/HotelDetailView.tsx`, `src/components/HotelWizard.tsx`, `src/providers/hotels/LiteApiProvider.ts`, `src/utils/hotelBooking.ts` (internal — current implementation)

**Note on Expedia:** direct crawl attempts (both the interactive browser and a server-side WebFetch) were served a bot-detection interstitial. If pixel-exact current Expedia detail is needed, that requires either an authenticated/manual session or Expedia's own partner-facing documentation, neither available in this session.
