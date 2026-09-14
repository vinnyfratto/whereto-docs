# Booking Workflow — Progress & Build Tracker

**Single source of truth for the flight booking/checkout workflow.** Keep this updated every work session. If we run out of tokens mid-task, the **▶ PICK UP HERE** marker (bottom) says exactly where to resume.

Last updated: 2026-06-20 · App version at last update: **0.1.47 / versionCode 148**

Status legend: ✅ done & verified-by-typecheck · 🟡 partial/in-progress · ⬜ not started · ♻️ to be replaced/deprecated

---

## 0. CURRENT REALITY (what exists today, post Bucket A + B)

The **old** flow (now being redesigned) is:
`FlightDetailView (fare upgrade) → PassengerFormModal → ExtrasSheet (seats+bags) → ReviewConfirmSheet → createOrder → BookingConfirmedSheet`.

Wired in **two** call-sites:
- [app/search/results.tsx](../app/search/results.tsx) (book directly from results)
- [app/bookings/staging.tsx](../app/bookings/staging.tsx) ("Booking in Progress" dashboard → Checkout)

### Files & their role today
| File | Role | Fate under redesign |
|---|---|---|
| [app/bookings/staging.tsx](../app/bookings/staging.tsx) | "Booking in Progress" dashboard — FLIGHT / HOTEL / CAR tiles, Checkout btn | 🟡 keep as the entry; **its Checkout now navigates to the new Passengers page** |
| [app/search/PassengerFormModal.tsx](../app/search/PassengerFormModal.tsx) | Single modal, paged per passenger; identity/doc/contact/loyalty fields | ♻️ **fold into Wizard Step 1** (reuse field set; drop the paging) |
| [src/components/ExtrasSheet.tsx](../src/components/ExtrasSheet.tsx) | Seats (seat map) + per-passenger bag steppers | ♻️ **bags → Wizard Step 3**; seat-MAP not in the new 4-step spec (see Open Decision #2) |
| [src/components/SeatMapView.tsx](../src/components/SeatMapView.tsx) | Tappable cabin grid | 🟡 orphaned by new spec — keep for an optional seat sub-step or later |
| [src/components/ReviewConfirmSheet.tsx](../src/components/ReviewConfirmSheet.tsx) | Final review + T&C consent + grand total | ✅ keep (final checkout, after all passengers complete) |
| [src/components/BookingConfirmedSheet.tsx](../src/components/BookingConfirmedSheet.tsx) | PNR + confirmation | ✅ keep |
| [src/components/FlightDetailView.tsx](../src/components/FlightDetailView.tsx) | Flight detail + **fare-upgrade upsell** (per-offer cabin) | 🟡 keep; note Open Decision #2 (per-passenger seat class vs per-offer cabin) |

### Data layer today
- **Store:** [src/store/bookingInProgressStore.ts](../src/store/bookingInProgressStore.ts) — Zustand, one row per user.
- **Table:** `booking_in_progress` (Supabase). Columns: `id, user_id, flight_data (JSONB), flight_status, flight_order_id, hotel_data, hotel_status, hotel_booking_ref, car_data, car_status, car_booking_ref, created_at, updated_at`.
- **Methods:** `loadStaging, addFlight, addHotel, removeFlight, removeHotel, markFlightBooked, markHotelBooked, clearStaging`.
- **⚠️ No per-passenger progress storage exists yet** — this is the main schema gap for the redesign.

### Types today ([src/types/index.ts](../src/types/index.ts))
`Flight` (incl. `baseAmount, taxAmount, fareBrand, cabinClass, conditions, slices, upgradeOffers`), `FareUpgrade`, `BagService`, `SeatSelection`, `OrderService`, `BookingExtras`, `SavedPassenger` (incl. `middle_name, loyalty_airline, loyalty_account_number`).

### Edge functions (ALL DEPLOYED to project `pvqwxphrmcvmztlkzhsg`)
- ✅ `duffel-search` — offers (verbatim passthrough)
- ✅ `duffel-offer` — single offer w/ `return_available_services` (bag prices)
- ✅ `duffel-seat-maps` — seat maps
- ✅ `duffel-checkout` — creates order; accepts `services[]`, `middle_name`, `loyalty_programme_accounts`; payment = offer + services
- ✅ `createOrder` ([src/utils/booking.ts](../src/utils/booking.ts)) takes `(offerId, offerAmount, currency, passengerIds, passengers, services?, servicesTotal?)`
- Fetchers: [src/utils/offerExtras.ts](../src/utils/offerExtras.ts) — `fetchBagServices`, `fetchSeatMaps`

---

## 1. TARGET DESIGN (the new per-passenger wizard) — authoritative spec

When the user touches **Book**, navigate to a **NEW page** with **Dashboard-style tiles, one per passenger** (1 tile solo, 2 for two pax, etc.).

**Each empty tile:** shows `**Passenger #,**` (newline) `Touch to assign passenger.`

**Tapping a tile** opens a **multi-step wizard** for that passenger:
1. **Select/enter passenger** identity. Closable at ANY point → returns to the tiles page; **progress persists** to the user's `booking_in_progress` (or sibling) table.
   - If closed before completion → overlay a **rust** variation of the completion overlay: a Solar **person** icon + message **"Needs Completion"**.
2. **Select seat class** — affects price; communicate clearly.
3. **Select bags** — communicate what baggage the chosen seat class already includes; affects price; communicate clearly.
4. **Display all regulatory info** the user must accept on behalf of this passenger.
5. On completing step 4 → **passenger is complete**.
6. If multiple passengers → completed tile gets the **standard green overlay + center checkmark**.
7. **Bottom button: "Clear all passengers"** — visible once ≥1 passenger has any data.

(When all tiles complete → proceed to existing **ReviewConfirmSheet → createOrder → BookingConfirmedSheet**.)

---

## 2. DATA MODEL (new — needed for the redesign)

Add per-passenger progress to the staging row. **Proposed schema change:**

```sql
ALTER TABLE booking_in_progress
  ADD COLUMN IF NOT EXISTS passengers JSONB DEFAULT '[]'::jsonb;
```

**`PassengerProgress` shape** (store in that JSONB array, one per seat index):
```ts
interface PassengerProgress {
  index:        number;                       // 0-based tile position
  status:       'empty' | 'in_progress' | 'complete';
  lastStep:     0 | 1 | 2 | 3;                // wizard step reached (for resume)
  identity?:    SavedPassenger;               // step 1 (reuse existing field set)
  seatClass?:   SeatClass;                    // step 2 ('economy'|'premium_economy'|'business'|'first')
  bags?:        OrderService[];               // step 3 (duffel service ids + qty)
  servicesTotal?: number;                     // running added cost for this pax
  regulatoryAccepted?: boolean;               // step 4
}
```

New store methods on `bookingInProgressStore`:
- `savePassengerProgress(userId, index, partial: Partial<PassengerProgress>)` — upsert into the `passengers` array.
- `clearPassengers(userId)` — reset array to `[]` (for the "Clear all passengers" button).
- `loadStaging` already returns the row; surface `staging.passengers`.

**Passenger count source:** `staging.flight_data.passengerIds.length` (already used as `paxCount` in staging.tsx).

---

## 3. FILE PLAN

### NEW files
- ⬜ `app/bookings/passengers.tsx` — the tiles page (route `/bookings/passengers`). Renders `paxCount` tiles; each shows empty prompt / rust "Needs Completion" / green "complete" overlay. Hosts the "Clear all passengers" footer button. Opens `PassengerWizard` on tile tap. When all complete → triggers ReviewConfirmSheet → createOrder.
- ⬜ `src/components/PassengerWizard.tsx` — modal wizard, 4 steps, closable (persists), resumes at `lastStep`.
  - Step 1 identity: lift the field set + `ChipRow`/`FieldLabel` helpers + saved-traveller quick-fill from `PassengerFormModal.tsx`.
  - Step 2 seat class: chip/card selector; show price delta (Open Decision #2).
  - Step 3 bags: reuse per-passenger bag logic from `ExtrasSheet.tsx` (`fetchBagServices` filtered to this passengerId) + "included in <class>" messaging.
  - Step 4 regulatory: render `flight.conditions` + Secure Flight/APIS notices + airline conditions of carriage; require explicit accept.
- ⬜ `src/components/overlays` — a **NeedsCompletionOverlay** (rust bg + Solar `user` icon + "Needs Completion"). Mirror the green `BookedOverlay` already in `staging.tsx` (lines ~29-45). Reuse that green one for the complete state.

### MODIFY
- ⬜ [src/store/bookingInProgressStore.ts](../src/store/bookingInProgressStore.ts) — add `passengers` to `StagingRow`, add `savePassengerProgress` + `clearPassengers`.
- ⬜ "Book" entry points → navigate to `/bookings/passengers` instead of opening `PassengerFormModal`:
  - [app/bookings/staging.tsx](../app/bookings/staging.tsx) Checkout button.
  - [app/search/results.tsx](../app/search/results.tsx) — decide whether direct-from-results also routes here (recommended: yes, for consistency) or keeps its inline flow.
- ⬜ Supabase migration for the `passengers` column (write SQL file under `supabase/` and run, OR add via dashboard).

### REUSE (no change)
- `ReviewConfirmSheet`, `BookingConfirmedSheet`, `createOrder`, `duffel-checkout`, `offerExtras.ts`.

### DEPRECATE / ORPHAN
- ♻️ `PassengerFormModal.tsx` — superseded by Wizard Step 1 (keep file until wizard proven, then remove).
- ♻️ `ExtrasSheet.tsx` — bags move into Wizard Step 3; seat-map portion orphaned (see Open Decision #2).
- 🟡 `SeatMapView.tsx` — orphaned by the 4-step spec; keep for optional seat sub-step / future.

---

## 4. DECISIONS (resolved 2026-06-20)

1. ✅ **All book paths route to `/bookings/passengers`** — both staging Checkout AND results direct-book. (Results must ensure the flight is staged in `booking_in_progress` via `addFlight` before navigating.)
2. ✅ **Seat class = booking-level cabin choice** surfaced on each passenger but applied to the whole booking, driving the existing `upgradeOffers` swap. Stored ONCE as `selected_fare` (a `FareUpgrade`, or null = base) on the staging row. First passenger to choose sets it; others display/inherit. Checkout uses `selected_fare` if present, else base `flight_data`.
3. **Regulatory content (Step 4):** fare rules (`flight.conditions`) + Secure Flight/TSA notice + APIS + airline conditions of carriage + 24-hr cancellation. (Wording can be refined later; ship sensible defaults.)
4. **Migration:** SQL file under `supabase/` run via dashboard SQL editor (matches `vibes_table.sql` pattern). Two columns added — see §2.

---

## 5. BUILD ORDER (recommended)

1. Schema: add `passengers` JSONB column (migration) + extend `StagingRow` type + store methods.
2. `app/bookings/passengers.tsx` shell: render tiles from `paxCount`, empty prompt, "Clear all" footer. Wire navigation from staging Checkout.
3. Overlays: green complete (reuse) + rust "Needs Completion".
4. `PassengerWizard` Step 1 (identity, persist on close) → tile reflects in_progress/needs-completion.
5. Wizard Steps 2–4 (seat class, bags, regulatory) with price communication.
6. Completion → green overlay; all-complete → ReviewConfirmSheet → createOrder → BookingConfirmedSheet.
7. Remove deprecated PassengerFormModal/ExtrasSheet usage once verified. Bump version + versionCode.

---

## 6. REDESIGN BUILD LOG (v0.1.48)

✅ **Schema** — [supabase/booking_passengers_columns.sql](../supabase/booking_passengers_columns.sql) adds `passengers` + `selected_fare` JSONB. **⚠️ MUST RUN in Supabase SQL editor** (project `pvqwxphrmcvmztlkzhsg`) — not yet executed.
✅ **Types** — `PassengerProgress` added to [src/types/index.ts](../src/types/index.ts).
✅ **Store** — [bookingInProgressStore.ts](../src/store/bookingInProgressStore.ts): `StagingRow` gains `passengers` + `selected_fare`; new methods `savePassengerProgress`, `clearPassengers`, `setSelectedFare`.
✅ **Tiles page** — [app/bookings/passengers.tsx](../app/bookings/passengers.tsx): per-pax tiles, rust "Needs Completion" + green complete overlays, "Clear all passengers", "Review & book" when all complete → ReviewConfirmSheet → createOrder → BookingConfirmedSheet. Aggregates per-passenger bags into extras.
✅ **Wizard** — [src/components/PassengerWizard.tsx](../src/components/PassengerWizard.tsx): 4 steps (identity → seat class → bags → regulatory), closable w/ persist + resume at `lastStep`, seat-class drives `selected_fare` (booking-level), bags via `fetchBagServices` filtered to the passenger's Duffel id.
✅ **Entry wiring** — staging Checkout + results direct-book both `router.push('/bookings/passengers')` (results stages the flight first).
✅ Typechecks clean (only pre-existing tsconfig deprecation). Version 0.1.48 / 149.

### Still TODO
- ⬜ **RUN THE SQL MIGRATION** (above) — until then `passengers`/`selected_fare` reads/writes will fail.
- ⬜ **Test on device** — esp. close-and-resume persistence, overlays, seat-class price propagation to review, multi-pax.
- ⬜ **Deprecate old flow** — remove now-unused `PassengerFormModal`, `ExtrasSheet`, and the staging.tsx/results.tsx Extras+Review+PassengerForm modal wiring + their dead state vars (kept for now, harmless, but should be cleaned). `SeatMapView` orphaned.
- ⬜ Optional: regulatory wording review; per-passenger bag "included allowance" is currently inferred from cabin, not exact Duffel allowance.

## ▶ PICK UP HERE
**Status: redesign BUILT (v0.1.48), typechecks clean. NOT yet device-tested; SQL migration NOT yet run.**
Next concrete steps, in order:
1. **Run** [supabase/booking_passengers_columns.sql](../supabase/booking_passengers_columns.sql) in the Supabase SQL editor.
2. Device-test the full path: results/staging → Book → tiles → wizard (close mid-way → rust overlay → reopen resumes) → complete all → Review & book → confirmation.
3. Once verified, clean up deprecated `PassengerFormModal`/`ExtrasSheet` usage in [staging.tsx](../app/bookings/staging.tsx) + [results.tsx](../app/search/results.tsx) (remove the old Extras/Review/PassengerForm modals + dead state).
