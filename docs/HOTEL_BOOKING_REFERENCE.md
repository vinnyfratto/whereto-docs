# WhereTo — Hotel Booking Reference
**For bug-fix / detail threads. Read this before touching any hotel booking code.**
Last updated: 2026-06-21 · App version: 0.1.58 / versionCode 159

---

## 1. Critical constraints

- **LiteAPI key is client-side (`EXPO_PUBLIC_LITEAPI_KEY`).** Unlike Duffel, LiteAPI is called directly from the app — no edge function proxy.
- **Skins architecture:** ALL style tokens (colors, fonts, spacing) come from `src/Skins/`. Never inline colors/fonts.
- **Scroll clearance rule (every scroll on a tab screen):** `contentContainerStyle={{ paddingBottom: 94 + insets.bottom + 10 }}`
- **Version bump required on every change:** `version` in `app.json` + `versionCode` (android).
- **Provider abstraction:** swap hotel APIs via `src/providers/hotels/index.ts` — never import `LiteApiProvider` directly except in `HotelSelectSheet.tsx`.

---

## 2. Flow overview

```
results.tsx  (HotelDetailSheet → "Book" on a hotel)
  └─ handleAddHotelToBooking → stageHotelFn(hotel, userId)   // adds to booking_in_progress
       └─ router.push('/bookings/passengers?openHotelWizard=1')  // → Booking Dashboard

app/bookings/passengers.tsx — Booking Dashboard [ Flight ] [ Hotel ] [ Car ]  (canonical, v0.3.56)
  Green completion tiles; works with any mix of items (a hotel-only booking needs no flight).
  └─ FLIGHT tile → tap to expand per-traveler entry (PassengerWizard) → Review & book
  └─ HOTEL tile → auto-opens HotelWizard (openHotelWizard=1); tap to reconfigure
       └─ HotelWizard (4-step) → onSave → saveHotelProgress()  // writes hotel_progress JSONB
            └─ tile shows the selected room + green completion overlay
  └─ CAR tile → coming soon
```

> **v0.3.56 routing change:** every "Add to Booking" (flight or hotel), plus the profile
> "Booking in Progress" link, now lands on the green-tile [ Flight ] [ Hotel ] [ Car ] dashboard at
> `app/bookings/passengers.tsx`, which works with any mix of items and needs no flight. The Flight tile
> expands to reveal per-traveler entry. `app/bookings/staging.tsx` is re-deprecated. This fixes
> hotel-only bookings dead-ending on the old flight-first screen ("no travelers").

---

## 3. Key files

| File | Purpose |
|---|---|
| `app/bookings/passengers.tsx` | Hotel tile + auto-open wizard logic; "Add Hotel" → navigate to results |
| `app/search/results.tsx` | Auto-opens `HotelDetailSheet` via `openHotelIata` URL param; after staging → navigates back to passengers |
| `src/components/HotelWizard.tsx` | 4-step hotel booking modal (see §5) |
| `src/components/HotelDetailSheet.tsx` | Hotel search, sort, card list, detail view; entry point from results screen |
| `src/components/HotelSelectSheet.tsx` | Standalone hotel search sheet (built but not used in the main flow — reserved for future use) |
| `src/store/bookingInProgressStore.ts` | `addHotel`, `removeHotel`, `saveHotelProgress`, `clearHotelProgress` |
| `src/providers/hotels/index.ts` | Active provider export — swap here to change API |
| `src/providers/hotels/LiteApiProvider.ts` | LiteAPI implementation; room deduplication by `(name, boardType, refundable)` |
| `src/providers/hotels/types.ts` | `HotelProvider`, `HotelSearchParams`, `HotelResult`, `HotelDetail` |
| `src/types/index.ts` | `HotelRoomType`, `HotelBookingProgress`, `BoardType` |

---

## 4. Navigation params

### results.tsx ← passengers.tsx
```
/search/results?openHotelIata=<IATA>
```
- `openHotelIata`: last-leg IATA of the booked flight's destination (e.g. `MIA`)
- On load, `results.tsx` scans `destinations` for a match and calls `setHotelDest(match)` to auto-open `HotelDetailSheet`
- One-time trigger guarded by `autoHotelOpenedRef` (won't re-trigger on re-render)

### passengers.tsx ← results.tsx
```
/bookings/passengers?openHotelWizard=1
```
- `openHotelWizard`: signals passengers page to auto-open `HotelWizard`
- Fires only once `staging.hotel_data` is populated (useEffect dependency)

---

## 5. HotelWizard — step-by-step

File: `src/components/HotelWizard.tsx`

**Props:**
```ts
interface HotelWizardProps {
  visible:          boolean;
  hotel:            HotelResult;       // staged hotel (has roomTypes[])
  existingProgress: HotelBookingProgress | null;
  leadGuestName?:   string;            // pre-fill from lead passenger
  leadGuestEmail?:  string;
  onClose:          () => void;
  onSave:           (progress: HotelBookingProgress) => void;
}
```

**Steps (0-indexed):**

| # | Title | What happens |
|---|---|---|
| 0 | Choose Your Room | `RoomCard` list from `hotel.roomTypes[]`; shows board type, free cancel badge, occupancy, amenities chips, per-night + total; one card selectable (navy border + check); Next disabled until a room is chosen |
| 1 | Stay Preferences | Arrival time chips (Morning / Afternoon / Evening / Late Night), smoking preference chips (Non-smoking / Smoking / No preference), special requests multiline TextInput, lead guest name + email |
| 2 | Policies & Inclusions | Calls `provider.fetchHotelDetail(hotel.id)` on wizard open; shows check-in/out times, cancellation policy card (green left border = refundable / rust = non-refundable), what's included (board type + amenities), hotel facilities grid (12 items), important info, description |
| 3 | Confirm Stay | Hotel name + address + dates, selected room name + board type, guest info, price breakdown (per-night × nights + taxes = total), policy consent checkbox, "Save Hotel" CTA — calls `onSave()` |

**Board type labels:**
```
RO → Room Only
BB → Bed & Breakfast
HB → Half Board
FB → Full Board
AI → All Inclusive
```

**Close / back behaviour:** Pressing back on step 0 calls `onClose()`. Back on steps 1–3 goes to previous step. No partial-progress save on close (unlike PassengerWizard) — hotel progress is only written on "Save Hotel".

---

## 6. Data model

### `HotelRoomType` (in `src/types/index.ts`)
```ts
interface HotelRoomType {
  offerId:         string;    // LiteAPI prebook token for this rate
  name:            string;    // e.g. "Deluxe King Room"
  totalAmount:     number;    // all-in total for full stay
  currency:        string;
  perNight:        number;    // totalAmount / nights
  boardType:       BoardType; // 'RO' | 'BB' | 'HB' | 'FB' | 'AI' | string
  refundable:      boolean;
  cancelDeadline?: string;    // ISO datetime — free cancel until this point
  amenities:       string[];
  maxAdults:       number;
  maxChildren:     number;
  taxAmount?:      number;    // taxes already included in totalAmount (display only)
}
```

### `HotelBookingProgress` (in `src/types/index.ts`)
```ts
interface HotelBookingProgress {
  status:           'empty' | 'room_selected' | 'details_added' | 'complete';
  selectedRoom?:    HotelRoomType;
  guestName?:       string;
  guestEmail?:      string;
  guestPhone?:      string;
  arrivalTime?:     string;   // e.g. "14:00–16:00"
  specialRequests?: string;
  smokingPref?:     'non-smoking' | 'smoking' | 'no-preference';
  policyAccepted?:  boolean;
}
```

### `StagingRow` hotel fields (in `src/store/bookingInProgressStore.ts`)
```ts
hotel_id:       string | null;              // indexed copy of hotel_data.id
hotel_data:     HotelResult | null;         // full hotel object from LiteAPI
hotel_progress: HotelBookingProgress | null; // wizard state, persisted to Supabase
```

### Supabase table: `booking_in_progress`
Project ID: `pvqwxphrmcvmztlkzhsg`

Required columns (⚠️ SQL migration — run in Supabase SQL editor if not already applied):
```sql
ALTER TABLE booking_in_progress
  ADD COLUMN IF NOT EXISTS hotel_progress JSONB;
```

### Store methods
```ts
addHotel(hotel: HotelResult, userId: string): Promise<void>
removeHotel(userId: string): Promise<void>
saveHotelProgress(userId: string, progress: HotelBookingProgress): Promise<void>
clearHotelProgress(userId: string): Promise<void>
```

---

## 7. LiteAPI provider — room deduplication

`LiteApiProvider.ts` maps raw LiteAPI `roomTypes[]` (can be 100s of entries — one per physical room slot) into deduplicated room classes using:

```ts
const classKey = `${name}||${boardType}||${String(refundable)}`;
// keeps cheapest entry per key
```

Result is sorted ascending by `totalAmount`. This reduces 100–200 raw entries to ~3–8 distinct selectable room types in the wizard.

---

## 8. Hotel tile overlays (passengers.tsx)

| State | Tile appearance |
|---|---|
| No hotel staged | Dashed "Add a hotel to your trip" tile — taps to navigate to results |
| Hotel staged, wizard incomplete | Hotel photo + name + city + dates + "Tap to configure" — opens HotelWizard |
| Hotel staged, wizard complete | Same tile + green rgba(0,162,27,0.52) overlay + checkCircle icon |

Hotel photo is shown as a 66×66 circle (matching passenger avatar size). Fallback: hotel icon on navy background. City is extracted from `hotel.address` (last 2 comma-separated segments).

---

## 9. HotelDetailSheet (entry point from results)

File: `src/components/HotelDetailSheet.tsx`

- Triggered by `setHotelDest(destination)` in `results.tsx`
- Auto-triggered when `openHotelIata` URL param is set and a matching destination is found
- Contains its own hotel search (LiteAPI) + sort (price / rating / stars) + HotelCard list + tap-to-detail-view
- "Book" button on a hotel card calls `onAddToBooking(hotel)` → `handleAddHotelToBooking` in `results.tsx`
- When called from passengers flow (`openHotelIata` is set), `handleAddHotelToBooking` navigates to `/bookings/passengers?openHotelWizard=1` instead of showing an alert

---

## 10. Booking workflow (added v0.3.54)

**Single shared workflow for ALL WhereTo surfaces** (Discover, Explore, Direct, Together, group). No surface talks to the provider or edge functions directly — everything goes through `src/utils/hotelBooking.ts`.

```
prepareHotelCheckout({ userId, guests?, source })   // re-quote fresh → prebook → price-check
   └─ <payment UI collects money — DEFERRED, Model A: LiteAPI = merchant of record>
confirmHotelBooking({ userId, prep, holder, guests, payment })   // book → persist → markHotelBooked
```

| Piece | File |
|---|---|
| Unified workflow (the single entry point) | `src/utils/hotelBooking.ts` |
| Provider booking methods (`quoteHotel`/`prebook`/`book`) | `src/providers/hotels/LiteApiProvider.ts` |
| Server-side booking edge fn (key hidden) | `supabase/functions/liteapi-book/index.ts` |
| Booking persistence table | `hotel_orders` (migration `supabase/migrations/20260708_hotel_orders.sql`) |

- **Booking host is `book.liteapi.travel/v3.0`** (prebook/book) — different from the search host `api.liteapi.travel/v3.0`.
- **Stale offer handling:** the staged `offerId` is NEVER booked directly (LiteAPI offers expire). `prepareHotelCheckout` re-quotes the hotel live, re-matches the room by class key, and prebooks the FRESH offer. `priceChanged` is surfaced when the fresh price differs.
- **Idempotency:** `clientReference` is derived deterministically (`userId:prebookId`) so retries can't double-book.
- **Attribution:** every booking carries a `source` tag persisted to `hotel_orders.source`.

## 11. Remaining pending items

1. **⚠️ SQL migration for `hotel_orders`** — run `supabase/migrations/20260708_hotel_orders.sql` in the SQL editor (creates the table + RLS). (`hotel_progress` column on `booking_in_progress` is already applied.)
2. **Payment UI (the deferred fork).** Model A chosen: LiteAPI merchant of record via their Payment SDK (Stripe Elements). In Expo this needs a WebView that loads the SDK after prebook and returns a `transactionId`, which is passed to `confirmHotelBooking` as `payment: { method: 'TRANSACTION', transactionId }`. Until this exists, `confirmHotelBooking` cannot complete a live booking.
3. **Search path still uses the client key** (`EXPO_PUBLIC_LITEAPI_KEY`). The booking/money path is now fully server-side; migrating browse/search into the edge function is the remaining key-exposure fix.
4. **`send-notification` needs a `hotel_confirmation` branch** — `liteapi-book` fires it best-effort; it's a no-op until the branch is added.
5. **"My Bookings" UI** — `bookingsStore` reads only `orders` (flights). Add a `hotel_orders` fetch + union for display.
6. **Deploy secret + function:** `LITEAPI_KEY` must be set as a Supabase secret; deploy with `npx supabase functions deploy liteapi-book`.
7. **Prebook/book response field extraction is defensive** (raw passed through). Confirm exact field names against a live sandbox prebook/book response and tighten.
8. **Not yet device-tested** end to end.

---

## 11. Icon names used in hotel flow

From Solar bold-duotone set via `src/components/Icon.tsx`:

| Icon | Used for |
|---|---|
| `hotel` | Hotel tile fallback (no photo) |
| `clockCircle` | Check-in / check-out times in wizard policies step |
| `arrowRight` | Check-out label in policies step |
| `star` | Board type / meal plan in policies step |
| `checkCircle` | Complete overlay on hotel tile |
| `close` | Remove hotel × on staged tile; wizard close |
| `add` | "Add a hotel" dashed tile icon |
| `mapPoint` | Address / city display |
| `buildings` | Hotel photo placeholder in HotelSelectSheet |
| `doubleAltArrowLeft` | Back button in wizard header |
| `doubleAltArrowRight` | Next button in wizard header |
| `shieldCheck` | Free cancel badge in RoomCard |
| `dangerTriangle` | Error state in hotel search |
