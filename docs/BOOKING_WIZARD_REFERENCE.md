# WhereTo — Booking Wizard Reference
**For bug-fix / workflow threads. Read this before touching any booking code.**
Last updated: 2026-06-20 · App version: 0.1.49 / versionCode 150

---

## 1. Critical constraints

- **Duffel API key NEVER reaches the client.** All Duffel calls go through Supabase edge functions (`supabase/functions/duffel-*/index.ts`). `DUFFEL_API_KEY` lives only in Supabase secrets.
- **Skins architecture:** ALL style tokens (colors, fonts, spacing) come from `src/Skins/`. Never inline colors/fonts in components.
- **Scroll clearance rule (EVERY scroll on a tab screen):** `contentContainerStyle={{ paddingBottom: 94 + insets.bottom + 10 }}`
- **Version bump required on every change:** `version` in `app.json` + `versionCode` (android), following `0.0.x=dev / 0.x.0=beta / 1.0.0=go-live`.

---

## 2. Flow overview

```
FlightDetailView (fare-upgrade upsell)
  └─ "Book" → stages flight → router.push('/bookings/passengers')

results.tsx  (direct from search results)
  └─ stageFlightFn() → router.push('/bookings/passengers')

staging.tsx  (Booking In Progress dashboard → Checkout)
  └─ router.push('/bookings/passengers')
```

```
/bookings/passengers  (app/bookings/passengers.tsx)
  ├─ tile per passenger  →  PassengerWizard (modal, 4 steps)
  ├─ all complete?       →  ReviewConfirmSheet
  │                           └─ createOrder() → duffel-checkout edge fn
  │                                └─ BookingConfirmedSheet (PNR)
  └─ "Clear all passengers" resets progress + selected_fare
```

---

## 3. Key files

| File | Purpose |
|---|---|
| `app/bookings/passengers.tsx` | Tiles page — one tile per passenger; overlays; "Review & book" / "Clear all" footer |
| `src/components/PassengerWizard.tsx` | 4-step modal wizard (see §4) |
| `src/components/ReviewConfirmSheet.tsx` | Final review + T&C checkbox + grand total; gates `createOrder` |
| `src/components/BookingConfirmedSheet.tsx` | Full-screen PNR + itinerary after booking |
| `src/store/bookingInProgressStore.ts` | Zustand store; persists to `booking_in_progress` Supabase table |
| `src/types/index.ts` | All shared types incl. `PassengerProgress`, `FareUpgrade`, `BagService`, `SavedPassenger` |
| `src/utils/booking.ts` | `createOrder(offerId, offerAmount, currency, passengerIds, passengers, services?, servicesTotal?)` |
| `src/utils/offerExtras.ts` | `fetchBagServices(offerId)` · `fetchSeatMaps(offerId)` |
| `supabase/functions/duffel-checkout/index.ts` | Creates Duffel order; accepts `services[]`, `middle_name`, `loyalty_programme_accounts` |
| `supabase/functions/duffel-offer/index.ts` | GET single offer with `return_available_services=true` (bag prices) |
| `supabase/functions/duffel-seat-maps/index.ts` | GET seat maps array |
| `supabase/booking_passengers_columns.sql` | ⚠️ MIGRATION NOT YET RUN — adds `passengers` + `selected_fare` columns |

---

## 4. PassengerWizard — step-by-step

File: `src/components/PassengerWizard.tsx`

**Props:**
```ts
interface PassengerWizardProps {
  visible:      boolean;
  index:        number;          // 0-based; index === 0 = lead passenger
  flight:       Flight;          // base flight from staging
  selectedFare: FareUpgrade | null;  // booking-level cabin already chosen (if any)
  existing:     PassengerProgress | null;  // saved progress (resume support)
  defaultEmail: string;          // pre-fill for lead passenger
  onClose:      () => void;
}
```

**Steps (0-indexed):**

| # | Name | What happens |
|---|---|---|
| 0 | Passenger | Saved Travellers chip row (if user has any) + full identity form (title, name, DOB, gender, nationality, document, lead contact fields). Validates required fields on Next. |
| 1 | Seat class | Base fare card + upgrade cards (price delta). Choosing a card sets `fareId`; advancing commits `setSelectedFare()` to store (booking-level, shared by all passengers). |
| 2 | Bags | Calls `fetchBagServices(effOfferId)`, filtered to this passenger's Duffel pax ID. Quantity steppers. Shows per-cabin inclusion note. |
| 3 | Confirm (Regulatory) | 5 condition bullets + "I confirm" checkbox. Completing this step calls `persist('complete', 3)` then `onClose()`. |

**Close behaviour:** Any close (X or back-gesture) saves `status: 'in_progress'` if the user has entered any name, keeping the `lastStep` so the wizard resumes at the same page on re-open.

**Saved Travellers (Step 0):**
- Reads `user.savedPassengers` from `useAuthStore`.
- Horizontal chip row at top of step 0, only shown if `saved.length > 0`.
- Tapping a chip autofills the whole form from the `SavedPassenger` record.
- Active chip (matching current form) highlights in rust.
- For the lead passenger (`index === 0`): preserves the existing email if the saved record has no email.

---

## 5. Data model

### `PassengerProgress` (in `src/types/index.ts`)
```ts
interface PassengerProgress {
  index:               number;          // 0-based tile position
  status:              'empty' | 'in_progress' | 'complete';
  lastStep:            0 | 1 | 2 | 3;  // wizard step reached (for resume)
  identity?:           SavedPassenger;  // step 0 result
  seatClass?:          SeatClass;       // step 1 result
  bags?:               OrderService[];  // step 2 result (duffel service ids + qty)
  servicesTotal?:      number;          // added cost for this passenger's bags
  regulatoryAccepted?: boolean;         // step 3 result
}
```

### `StagingRow` additions (in `src/store/bookingInProgressStore.ts`)
```ts
passengers:    PassengerProgress[] | null;   // one entry per completed/started tile
selected_fare: FareUpgrade | null;           // booking-level cabin choice
```

### Supabase table: `booking_in_progress`
Project ID: `pvqwxphrmcvmztlkzhsg`

Columns added (⚠️ SQL migration NOT yet run):
```sql
ALTER TABLE booking_in_progress
  ADD COLUMN IF NOT EXISTS passengers    JSONB DEFAULT '[]'::jsonb,
  ADD COLUMN IF NOT EXISTS selected_fare JSONB;
```
File: `supabase/booking_passengers_columns.sql`
**Must be run in Supabase SQL editor before passenger save/load will work.**

### Store methods
```ts
savePassengerProgress(userId, index, partial: Partial<PassengerProgress>): Promise<void>
clearPassengers(userId): Promise<void>          // also resets selected_fare
setSelectedFare(userId, fare: FareUpgrade | null): Promise<void>
```

---

## 6. Fare / cabin model

- **Cabin choice is booking-level**, not per-passenger (Duffel offers cabin at offer level).
- Base flight comes from `staging.flight_data`. Upgrade offers come from `flight.upgradeOffers[]`.
- The first passenger to advance through Step 1 commits `selected_fare` to the staging row via `setSelectedFare()`.
- Subsequent passengers open to the same selection pre-selected.
- `passengers.tsx` computes `effFlight`:
  ```ts
  const effFlight = selected_fare
    ? { ...flight, id: sf.id, passengerIds: sf.passengerIds, totalAmount: sf.totalAmount, ... }
    : flight;
  ```
- `effFlight` is what gets passed to `createOrder` and `ReviewConfirmSheet`.

---

## 7. Tile overlays (passengers.tsx)

| State | Overlay | Colour |
|---|---|---|
| Empty | none | — |
| `in_progress` | `NeedsCompletionOverlay` | rust rgba(193,87,53,0.55) + person icon + "Needs Completion" text |
| `complete` | `CompleteOverlay` | green rgba(0,162,27,0.52) + checkRead icon |

Logic:
```ts
const complete = p?.status === 'complete';
const started  = !!p && p.status !== 'complete';
```

---

## 8. Order of operations at checkout

1. All tiles `status === 'complete'` → "Review & book" appears.
2. `ReviewConfirmSheet` opens — shows flight recap, passenger list, itemized price, extras subtotal, grand total, fare rules, T&C checkbox.
3. User ticks checkbox and presses confirm → `handleConfirm()` in `passengers.tsx`:
   ```ts
   createOrder(effFlight.id, effFlight.totalAmount, currency, effFlight.passengerIds,
               bookingPassengers, aggregatedExtras.services, aggregatedExtras.servicesTotal)
   ```
4. `createOrder` → `duffel-checkout` edge fn → Duffel `/air/orders`.
5. `markFlightBooked(orderId)` updates staging row.
6. `BookingConfirmedSheet` shows PNR.

---

## 9. Known pending items

1. **⚠️ SQL migration NOT run.** Without it, `savePassengerProgress` / `setSelectedFare` fail with column-not-found errors. Must be run manually in the Supabase SQL editor.
2. **Not yet device-tested.** Priority test path:
   - Search → Book (from results) → tiles page → tap tile → wizard (close mid-way = rust overlay) → reopen (resumes at saved step) → complete all 4 steps → green overlay → all complete → "Review & book" → ReviewConfirmSheet → confirm → BookingConfirmedSheet.
3. **Dead code to clean up** (harmless for now):
   - Old `PassengerFormModal` / `ExtrasSheet` / `ReviewConfirmSheet` modal JSX + dead state vars still in `app/bookings/staging.tsx` and `app/search/results.tsx`.
   - `src/components/SeatMapView.tsx` — orphaned (seat-map not in the 4-step spec).

---

## 10. Entry-point wiring (both navigate to `/bookings/passengers`)

### `app/search/results.tsx` (direct from search)
```ts
// inside onBook():
await stageFlightFn(flight, userId);   // addFlight to staging
router.push('/bookings/passengers');
```

### `app/bookings/staging.tsx` (Booking In Progress → Checkout)
```ts
// Checkout button handler:
router.push('/bookings/passengers');
```

---

## 11. Icon system

Solar bold-duotone icons via `src/components/Icon.tsx`. Generate with `npm run generate:icons`.
Icon names used in booking flow: `checkRead`, `user`, `chevronRight`, `doubleAltArrowLeft`, `doubleAltArrowRight`, `plane`, `bag`, `shieldCheck`, `check`, `close`, `add`.

---

## 12. Supabase edge functions (all deployed, project `pvqwxphrmcvmztlkzhsg`)

| Function | Route | Used for |
|---|---|---|
| `duffel-search` | POST `/search` | Offer search |
| `duffel-offer` | GET `/offer/:id` | Single offer + `return_available_services=true` (bag prices) |
| `duffel-seat-maps` | GET `/seat-maps?offer_id=` | Seat map data |
| `duffel-checkout` | POST `/checkout` | Create order; accepts `services[]`, `middle_name`, `loyalty_programme_accounts` |

Edge functions live in `supabase/functions/`. Client calls go through `src/utils/booking.ts` and `src/utils/offerExtras.ts`.
