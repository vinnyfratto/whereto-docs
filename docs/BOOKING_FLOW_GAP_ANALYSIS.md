# Flight Review & Booking — Competitive Teardown + Gap Analysis

**Purpose:** Document, screen by screen, what Expedia and Booking.com *display* and what they *ask the user to input/select* during flight review → checkout → confirmation. Then cross-reference against WhereTo's current booking flow to surface our gaps.

**Scope note:** This is a gap-identification exercise, **not** a spec to replicate either platform. Companion doc to `docs/HOTEL_DETAIL_BOOKING_GAP_ANALYSIS.md` — same method, flight side.

**Researched:** 2026-06-20 (original pass, Expedia + Priceline). **Refreshed 2026-07-16** — swapped Priceline for Booking.com (to match the hotel teardown's platform pair) and re-audited WhereTo from scratch, since flight booking changed dramatically in the interim: `feat(flights): wire real LiteAPI booking checkout end-to-end (v0.3.127)`, shipped the day before this refresh, replaced most of what the original pass flagged as missing. **Read this version, not the June one** — the gap table below is current; §7's prioritized list reflects what's *actually* still open today.

- **Booking.com:** flight search form live-crawled (`flights.booking.com`, one-way AUS→EWR and JFK→LHR, Aug 20 2026); both searches returned "We can't find any flights that match your search" in this automated session (likely a bot-detection/session gate on the results call, not a real route gap), so the search-form structure (round-trip/one-way/multi-city, cabin selector, direct-only toggle, traveler count) is live-verified but the results/detail/booking screens below are grounded in well-established, stable Booking.com Flights UX rather than a fresh live capture.
- **Expedia:** bot-blocked (same "Bot or Not?" interstitial as the hotel teardown). Grounded in current public documentation + the original June research, re-verified against the hotel teardown's confirmed-current One Key / member-pricing sourcing.
- **WhereTo:** read directly, 2026-07-16, from `src/components/FlightDetailView.tsx`, `src/components/PassengerWizard.tsx`, `src/components/ReviewConfirmSheet.tsx`, `src/components/BookingConfirmedSheet.tsx`, `app/bookings/passengers.tsx`, `src/utils/flightBooking.ts`, `supabase/functions/liteapi-flights/index.ts`.

---

## 1. The canonical OTA booking funnel

Both Expedia and Booking.com follow the same high-level skeleton. Use this as the column spine for the gap table in §6.

1. **Search** → 2. **Results list** → 3. **Flight Details / Fare selection** → 4. **Review (trip summary)** → 5. **Fare upgrade / branded-fare upsell** → 6. **Seats** → 7. **Bags & extras** → 8. **Traveler details** → 9. **Add-ons (insurance, price-drop, etc.)** → 10. **Payment** → 11. **Final review & legal consent** → 12. **Confirmation**

WhereTo today spans steps 3–12 across four real surfaces: `FlightDetailView` (detail + fare upgrade + bag picker), `PassengerWizard` (per-traveler identity → seat class → bags → regulatory), `ReviewConfirmSheet` (consolidated review + consent), and `BookingConfirmedSheet` (PNR + next steps). That's a much closer match to the OTA skeleton than the June pass found — see §5.

---

## 2. EXPEDIA — screen-by-screen

### 2.1 Results list (step 2)
**Displayed:** airline + logo, depart/arrive times, total duration, stop count + layover airport/duration, cabin, price (per person), "included/not included" hints, basic-economy flags, sort & filter rail (stops, price, duration, airline, departure window, airports), "Bundle + Save" (flight+hotel) nudges, One Key/rewards messaging.
**User inputs:** sort selection, filter selections, pick an outbound; for round trips a second list for the return; "Show more fares."

### 2.2 Flight Details / fare picker (step 3)
**Displayed:** full segment-by-segment itinerary (operating carrier, flight number, aircraft, layover detail, overnight/next-day +1 markers), cabin/fare-brand cards (Basic / Main / Refundable etc.) with a **feature comparison matrix** (seat choice, carry-on, checked bag, changes/cancel, mileage earning), per-fare price delta, baggage-allowance summary, CO₂ estimate.
**User inputs:** select fare brand; expand "what's included"; continue.

### 2.3 Review / trip overview (step 4)
**Displayed:** chosen flights recap, right-rail **Price Summary** (per-traveler fare, taxes & fees, total), "what's included/not included" panel, fare rules/cancellation summary.
**User inputs:** confirm and continue; edit/remove.

### 2.4 Fare upgrade upsell (step 5)
**Displayed:** "Upgrade your flight" cards with perks per tier and a **"Show more"** detail expander, price difference per tier.
**User inputs:** keep current fare or choose an upgrade tier.

### 2.5 Seat selection (step 6)
**Displayed:** interactive **seat map** per segment, seat categories (Standard, Preferred, Extra-legroom, Window/Aisle), per-seat fee, availability/occupied state, "standard seat included / choose later" option.
**User inputs:** tap seat per passenger per segment, or skip ("choose at check-in"); pay seat fees if applicable.

### 2.6 Bags & extras (step 7)
**Displayed:** carry-on / checked-bag allowance and per-bag fees, ability to prepay bags, airline baggage policy link.
**User inputs:** add checked bags (count), per passenger.

### 2.7 Traveler details (step 8)
**Displayed:** form per traveler; signed-in users get **saved-traveler autofill**.
**User inputs:** title, **first / middle / last name** (must match government ID), **date of birth**, **gender**, **frequent-flyer program + number** (optional), **Known Traveler / TSA PreCheck number** (optional), **redress number** (optional), passport/**travel-document details for international** (number, issuing country, expiry, nationality), **contact email + phone**, sometimes billing/contact address.

### 2.8 Add-ons (step 9)
**Displayed:** **AIG/Trip Protection** insurance card with coverage summary + premium; **Price Drop Protection** toggle (refunds difference as OneKeyCash); sometimes refundable-ticket upsell, airport-ride / activities cross-sell.
**User inputs:** opt in/out of insurance; toggle price-drop; accept/decline each.

### 2.9 Payment (step 10)
**Displayed:** payment-method selector (card, PayPal, **Affirm/"book now pay later,"** Apple/Google Pay where available), **OneKeyCash / rewards-points redemption**, **promo/coupon code** field, billing-address form, full price breakdown with taxes & fees.
**User inputs:** card number/expiry/CVV/name, billing address, apply points, apply coupon, choose installment plan.

### 2.10 Final review & consent (step 11)
**Displayed:** complete itinerary, traveler list, grand total, **fare rules / cancellation & change policy**, **terms & conditions consent**, "prices/seats not guaranteed until ticketed" warnings.
**User inputs:** check consent box(es); press **Complete Booking**.

### 2.11 Confirmation (step 12)
**Displayed:** **itinerary number + airline confirmation/PNR**, full itinerary, travelers, price paid, what's next (check-in window, manage-booking links), confirmation email sent.

---

## 3. BOOKING.COM FLIGHTS — screen-by-screen

Booking.com's flight product is newer than its hotel product (launched ~2021) and noticeably leaner than Expedia's. The search form (live-verified 2026-07-16): Round-trip / One-way / Multi-city toggle, cabin selector (Economy / Premium economy / Business / First-class), a **"Direct flights only" checkbox right on the search form** (Expedia buries this in post-search filters), origin/destination with full airport-name autocomplete, travel date, traveler count.

### 3.1 Results list (step 2)
**Displayed:** airline, times, duration, stops/layovers, price, cabin, filter/sort rail, **"Genius" member-discount pricing** baked into the list (same 10%+/tier-based model as Booking's hotel product), **"Self-transfer" risk warning** on itineraries that combine separately-ticketed airlines (you must re-check-in and re-clear security yourself — Booking discloses this explicitly since it's a real risk unique to mix-and-match itineraries).
**User inputs:** sort/filter, pick outbound (and return).

### 3.2 Flight details & fare upgrade (steps 3–5)
**Displayed:** departure/return recap, baggage information panel, fare-class options (generally fewer branded tiers than Expedia — often just cabin class rather than a full Basic/Main/Flex ladder), "what's included."
**User inputs:** keep fare or upgrade; continue.

### 3.3 Seat selection (step 6)
**Displayed:** seat map when the marketing airline supports it through Booking's API; otherwise deferred to the airline's own check-in.
**User inputs:** select seat or defer.

### 3.4 Baggage & add-ons (step 7)
**Displayed:** baggage fee detail, add-ons.
**User inputs:** add bags; continue.

### 3.5 Traveler details (step 8)
**Displayed:** passenger form, contact fields.
**User inputs:** name (matching ID), DOB, gender, passport details for international, **email + phone**. Booking's flight product captures less loyalty-program detail than Expedia/Priceline by default (no prominent frequent-flyer field on the base form).

### 3.6 Payment (step 10)
**Displayed:** card, **Genius discount applied automatically for signed-in members**, taxes & fees breakdown, grand total.
**User inputs:** payment details, billing address.

### 3.7 Final review & confirmation (steps 11–12)
**Displayed:** itinerary recap, total, fare rules, terms consent, then book. Confirmation: booking reference + airline PNR, itinerary emailed.
**User inputs:** consent; confirm & pay.

---

## 4. Element inventory — what they show vs. what the user inputs

### 4.1 Platform-DISPLAYED elements (union of both)
- Segment-level itinerary: operating carrier, flight #, aircraft type, layover duration & airport, overnight/+1-day markers
- Fare-brand comparison matrix (Basic vs Main vs Flex/Refundable) with per-feature checkmarks (richer on Expedia than Booking)
- Cabin/seat-class label per segment
- Baggage allowance + per-bag fee schedule (carry-on, 1st/2nd checked)
- CO₂ / emissions estimate (Expedia)
- Interactive **seat map** with categories, fees, occupied state
- **Price Summary breakdown**: base fare per traveler, taxes & fees, add-ons, grand total
- Fare rules: change/cancellation policy, refundability, 24-hr cancellation notice
- Trip-protection/insurance offer with coverage + premium (Expedia; less prominent on Booking)
- Price-drop protection / best-price guarantee messaging (Expedia)
- Rewards balance & redemption (One Key / Genius)
- **Self-transfer risk warning** on combined-airline itineraries (Booking)
- "Price & seat not guaranteed until ticketed" risk warnings
- Terms & conditions / consent text
- Confirmation: itinerary number **+ airline PNR/confirmation code**, check-in guidance, email receipt

### 4.2 USER-INPUT / selection elements (union of both)
- Sort & filter selections on results, including a direct-only toggle
- Outbound + return flight choice
- Fare-brand / upgrade-tier choice
- **Seat selection** per passenger per segment (or defer)
- Checked-bag count per passenger
- Per traveler: title, first / **middle** / last (ID-matching), DOB, gender
- Per traveler: **frequent-flyer program + number** (Expedia)
- Per traveler: Known Traveler/TSA PreCheck #, redress #
- International: passport number, issuing country, expiry, nationality
- Lead contact: email + phone (+ sometimes address)
- Insurance / trip-protection opt-in (Expedia)
- Price-drop toggle (Expedia)
- Payment method incl. **pay-over-time (Affirm)**, points redemption
- **Promo / coupon / VIP code**
- Billing address
- Terms & conditions consent checkbox
- Final "Complete booking" / "Confirm & Pay"

---

## 5. WhereTo today — what we have (re-verified 2026-07-16)

From `FlightDetailView.tsx`, `PassengerWizard.tsx`, `ReviewConfirmSheet.tsx`, `BookingConfirmedSheet.tsx`, `app/bookings/passengers.tsx`, `flightBooking.ts`:

**Detail screen (`FlightDetailView.tsx`):** airline banner (logo, aircraft, "PREFERRED" badge), a curated aircraft hero photo, **full per-segment itinerary when the offer carries `slices[]`** — each segment's date, times, flight number, duration, aircraft, and an explicit **layover chip** ("47m layover in ORD") between segments; itemized price breakdown (per-person + total, base fare vs. taxes & fees when the offer provides them); a real **"Upgrade Your Fare"** section listing every `upgradeOffers[]` entry as a selectable card with price delta; a Flight Details card (aircraft, CO₂ with above/below-average context); a Bags & Add-ons section (personal item always-included, carry-on included-by-cabin, checked-bag picker); a Fare Conditions card showing real per-fare changeable/refundable status with penalty amounts when the offer has `conditions`; sticky Select / Add-to-Booking bar.

**Per-passenger wizard (`PassengerWizard.tsx`, 4 steps):** Step 1 identity — title, first/**middle**/last name, DOB, gender, nationality, document type/number/issuing-country/expiry, **Known Traveler # + redress #**, **frequent-flyer program + number** (`loyalty_airline`/`loyalty_account_number`), lead-only email+phone, saved-traveler autofill chips. Step 2 seat class — a real fare-tier picker (`FareCard`) showing every fare option with per-person/total price, quick-glance inclusions (bags, meals, priority boarding, lounge, lie-flat), and a **"Full details & fare rules" expandable sheet** that breaks out baggage/seat-comfort/services as "confirmed by airline" vs. "typical for this cabin," plus real change/refund terms with fees. Step 3 bags — **real per-passenger bag pricing** fetched live (`fetchBagServices`) with a quantity stepper, or an honest "not available through this booking" fallback card. Step 4 regulatory — five acceptance statements (ID-match, fare rules, Secure Flight/TSA, APIS, conditions of carriage) with a consent checkbox gating "Complete passenger."

**Review (`ReviewConfirmSheet.tsx`):** flight recap, full traveler list (with frequent-flyer badge if present), **itemized price breakdown** (base fare × pax, bags, taxes & fees, grand total), fare-rules summary (change/refund text with fees), an explicit **"prices and seats aren't guaranteed until ticketed"** warning card, and a **terms-of-service consent checkbox** gating "Confirm & book."

**Confirmation (`BookingConfirmedSheet.tsx`):** success screen with the **real airline booking reference (PNR)**, order ID, itinerary recap, and a "What's Next" card (check-in reminder, "find it in My Bookings" pointer). Promises "a confirmation and e-ticket are on the way to your email" — **this claim is currently false**: `liteapi-flights/index.ts`'s `fireNotification` sends `type: "flight_confirmation"`, but its own inline comment says `send-notification` "needs a `flight_confirmation` branch to render this; harmless no-op until then." The UI shipped ahead of the email wiring.

**Booking dashboard (`app/bookings/passengers.tsx`):** three-tile dashboard (Flight / Hotel / Car), a live 24-hour countdown from `staging.expires_at`, per-traveler completion tiles that expand to launch `PassengerWizard`, and **real multi-passenger handling** — `paxCount` is derived from `flight.passengerIds.length`, not hard-coded.

**Booking (`flightBooking.ts` / `liteapi-flights/index.ts`):** `prepareFlightCheckout` re-verifies the staged offer's price before locking it (surfaces a "price updated, continue?" alert on drift — mirrors the hotel side's `prepareHotelCheckout`), then prebooks; `confirmFlightBooking` books with the prebook's `TRANSACTION_ID` and persists. (**Corrected 2026-07-17:** it sent `ACC_CREDIT_CARD` until v0.3.143 — LiteAPI *flights* reject that with `45009 "payment method must be CREDIT, TRANSACTION_ID, or THIRD_PARTY"`, unlike hotels, which do accept it. The bug was invisible while `53099` masked it.) **Bags collected in the wizard are priced and saved to `booking_in_progress.passengers[].bags`, and rolled up for display in `ReviewConfirmSheet` — but never actually attached to the LiteAPI booking.** `passengers.tsx`'s own comment confirms: *"Bags/extras aren't attached yet (no `attachServices` wiring); this only carries the base fare through."* ~~Hard external blocker, unchanged since June: LiteAPI sandbox rejects flight prebook with error `53099`~~ — **RESOLVED 2026-07-17.** Flight booking/payments were enabled on the Nuitee account (via the dashboard Flights onboarding), and the full path is now **verified end-to-end in sandbox**: verify → prebook → book(`TRANSACTION_ID`) → `CONFIRMED` with a real airline PNR (`FH-267-KOFL5XA0` / PNR `SGQG7R`). Two caveats found once it unblocked: (1) flights reject `ACC_CREDIT_CARD` (see above); (2) sandbox inventory is flaky — many offers return `502 / 53010 "offer is sold out"`, so the flow should fall through to another offer rather than erroring.

---

## 6. GAP TABLE

Legend: ✅ have · 🟡 partial/cosmetic · ❌ missing · **↑** improved since the June 2026 pass · **NEW** finding this pass

| Funnel element | Expedia | Booking.com | WhereTo | Gap notes |
|---|:---:|:---:|:---:|---|
| Sort/filter results | ✅ | ✅ | ✅ | Unchanged — in `results.tsx` |
| Segment-level itinerary (flight #, aircraft, layover detail) | ✅ | ✅ | ✅ **↑** | Was 🟡 in June ("not per-segment flight numbers/layover breakdown"). **Now real** — `SliceItinerary` shows per-segment flight #, duration, aircraft, and explicit layover chips, when `flight.slices[]` is populated by `liteapi-flights` |
| Fare-brand comparison matrix (Basic/Main/Flex) | ✅ | 🟡 (leaner than Expedia) | ✅ **↑** | Was ❌ in June ("No branded-fare selection at all"). **Now real** — `FareCard` in `PassengerWizard` + "Upgrade Your Fare" in `FlightDetailView`, both backed by real `upgradeOffers[]` with an expandable amenities/fare-rules sheet |
| Fare-upgrade upsell | ✅ | ✅ | ✅ **↑** | Same fix as above |
| **Interactive seat map / seat selection** | ✅ | ✅ | ❌ | **Still the single largest functional gap.** No seat selection anywhere, unchanged since June |
| Checked-bag *purchase* (persisted, priced, per-pax) | ✅ | ✅ | 🟡 **↑** | Was 🟡-cosmetic in June (local state, not priced, not passed to booking). **Now genuinely priced and persisted** (`fetchBagServices`, real per-passenger quantities, saved to progress, shown in review total) — but **still not attached to the actual LiteAPI booking** (`attachServices` unwired, per `passengers.tsx`'s own comment). Went from "fake" to "real but disconnected at the last mile" |
| Per-passenger seat/bag (multi-traveler) | ✅ | ✅ | ✅ **↑** | Was ❌ in June (hard-coded `passengerCount={1}`). **Now dynamic** — `paxCount` derives from `flight.passengerIds.length`, dashboard iterates every traveler |
| Trip protection / insurance | ✅ (AIG) | 🟡 (less prominent) | ❌ | Unchanged — no insurance offer |
| Price-drop / best-price guarantee | ✅ | 🟡 | ❌ | Unchanged |
| Frequent-flyer number capture | ✅ | 🟡 (not on base form) | ✅ **↑** | Was ❌ in June. **Now captured** — `loyalty_airline` + `loyalty_account_number` in `PassengerWizard`, shown as a badge in `ReviewConfirmSheet` |
| Middle name field | ✅ | ✅ | ✅ **↑** | Was ❌ in June. **Now captured**, ID-match hint included |
| Detailed price breakdown (base/taxes/fees lines) | ✅ | ✅ | ✅ **↑** | Was 🟡 in June (total + "taxes included," no itemization). **Now itemized** in both `FlightDetailView` and `ReviewConfirmSheet` (base × pax, bags, taxes, total) |
| Promo / coupon / rewards code | ✅ | ✅ (Genius auto-discount) | ❌ | Unchanged — no coupon field, no loyalty program |
| Payment screen (card/Apple-Google Pay) | ✅ | ✅ | ❌ | **Unchanged, still the other biggest gap.** Same `ACC_CREDIT_CARD` placeholder pattern as hotels — no real payment UI collected client-side |
| Pay-over-time (Affirm) | ✅ | 🟡 | ❌ | Unchanged |
| Billing address capture | ✅ | ✅ | ❌ | Unchanged |
| Fare rules / cancellation policy detail | ✅ | ✅ | ✅ **↑** | Was 🟡 in June (generic 3-line blurb). **Now real** — actual `conditions.changeable`/`.refundable` + penalty amounts, surfaced in three places (detail view, wizard amenities sheet, review sheet) |
| Terms & conditions consent checkbox | ✅ | ✅ | ✅ **↑** | Was ❌ in June ("No explicit consent gate"). **Now two gates**: `PassengerWizard`'s regulatory-acceptance step (per passenger) *and* `ReviewConfirmSheet`'s T&C consent (booking-level) — arguably more thorough than either OTA |
| "Price not guaranteed until ticketed" warning | ✅ | ✅ | ✅ **↑** | Was ❌ in June. **Now shown** explicitly in `ReviewConfirmSheet` |
| Multi-step review before pay | ✅ | ✅ | ✅ **↑** | Was ❌ in June ("we jump form → order"). **Now a real consolidated review** — `ReviewConfirmSheet` |
| Confirmation w/ airline PNR + next steps | ✅ | ✅ | 🟡 **↑ / NEW finding** | Was 🟡 in June ("just an alert"). **Visually now ✅** — `BookingConfirmedSheet` shows the real PNR, itinerary, and a "what's next" card. But it **promises an emailed confirmation that isn't actually sent yet** — `send-notification`'s `flight_confirmation` type is a documented no-op. Downgraded to 🟡 for that reason, not the UI itself |
| Saved-traveler autofill | ✅ | ✅ | ✅ | Unchanged — still matches well |
| CO₂ estimate | ✅ | ❌ | ✅ | Unchanged — we're ahead of Booking.com here too |
| Self-transfer / combined-airline risk warning | ❌ | ✅ | — | N/A — not established whether LiteAPI ever returns mix-and-match itineraries; flag for later if it does |
| Direct-flights-only search toggle | 🟡 (post-search filter) | ✅ (on the search form itself) | 🟡 | Out of this doc's scope (results-list, not detail/booking) — noted for the results-page owner |

---

## 7. Prioritized gaps (recommendation) — refreshed 2026-07-16

**Tier 1 — blocks a credible booking (do first):**
1. **Real payment capture** (card / Apple Pay / Google Pay) + billing address — still the single biggest gap, shared with hotels (`ACC_CREDIT_CARD` is a placeholder on both).
2. **Seat selection** (at minimum a "choose at check-in" vs. seat-map fork) — still the single most-expected feature we lack; unchanged priority from June.
3. **Wire `attachServices`** so the bags a traveler already picked and paid for in the wizard actually reach the LiteAPI booking — this is now a much smaller lift than "build bag selection from scratch" (June's framing): the UI, pricing, and persistence are all real today. Only the last API call is missing.
4. **Fix the confirmation-email promise** — either wire `send-notification`'s `flight_confirmation` branch, or soften `BookingConfirmedSheet`'s copy so it doesn't promise an email that never arrives.
5. **Remove the stale "confirmed by the airline via Duffel" line** in `PassengerWizard`'s `AmenitiesDetail` disclaimer — leftover copy from the pre-LiteAPI integration; harmless but a red flag worth a quick sweep for other stale Duffel references in booking-adjacent copy.

**Tier 2 — parity & trust:**
6. Trip protection / insurance offer.
7. Promo/coupon code + any loyalty concept (same opportunity as hotels' `suggestedSellingPrice`/member-pricing gap — consider solving both together).
8. Real payment UI is Tier 1 above; billing-address capture rides along with it.

**Tier 3 — revenue & differentiation:**
9. Price-drop protection.
10. Pay-over-time (Affirm-style) — likely post-launch, same as June.
11. Package (flight + hotel) bundle upsell — flagged in the hotel teardown as a differentiator WhereTo is uniquely positioned to build (both LiteAPI integrations already exist); applies equally from the flight side.

**Where we already lead or are at parity (confirmed 2026-07-16, several newly closed since June):** CO₂-with-context, saved-traveler autofill, real fare-brand/upgrade selection with a genuine amenities comparison sheet, itemized pricing throughout, a genuinely consolidated review + dual consent gates (arguably more thorough than either OTA), and real per-fare change/refund terms surfaced in three places. The October redesign risk flagged in June — "flattening toward Expedia's layout" — didn't happen; the build went the other direction and closed most of the credibility gaps organically.

---

## 8. Sources
- [Expedia flight booking process (step-by-step)](https://customerservice12.zohodesk.com/portal/en/kb/articles/what-is-expedia-s-flight-booking-process-step-by-step-15-7-2025)
- [Going.com — How to use Expedia](https://www.going.com/guides/how-to-use-expedia-to-find-cheap-flights)
- [Expedia — All about seats](https://www.expedia.com/helpcenter/?product=Flight&productId=flight&articleId=12369)
- [Expedia — Price Drop Protection](https://www.expedia.com/why/price-drop-protection)
- [Expedia — Flight insurance](https://www.expedia.com/helpcenter/?product=Flight&productId=flight&articleId=16505)
- [Expedia — Get Member Prices (One Key)](https://www.expedia.com/member-price)
- Live crawl: `flights.booking.com` search form, 2026-07-16 (AUS→EWR and JFK→LHR one-way, Aug 20 2026) — results calls did not return in this automated session
- `docs/liteapi/FLIGHTS.md`, `LiteAPI_Sample_Pull_FULL.xlsx` (internal — 2026-07-16 live sandbox field audit)
- `src/components/FlightDetailView.tsx`, `PassengerWizard.tsx`, `ReviewConfirmSheet.tsx`, `BookingConfirmedSheet.tsx`, `app/bookings/passengers.tsx`, `src/utils/flightBooking.ts`, `supabase/functions/liteapi-flights/index.ts` (internal — current implementation, 2026-07-16)

**Note on Expedia and Booking.com Flights:** neither could be live-crawled to full depth this session (Expedia bot-blocked entirely; Booking.com Flights' results call didn't return for this automated browser). Treat both platforms' results/detail/booking specifics as representative of stable, well-documented UX rather than pixel-exact captures. The **search-form structure** for Booking.com Flights (§3, top) *is* a live capture.
