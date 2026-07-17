# LiteAPI (Nuitee Connect) - Hotels API Reference

**Scope:** the full hotel lifecycle - static data, search/rates (incl. streaming), the rate JSON model, prebook, book, retrieve, list, cancel, and amendments - plus how it maps to WhereTo's existing hotel implementation.
**Last researched:** 2026-07-14.

> This is the **API-level** reference. The **app-level** hotel flow (wizard, staging, tiles, routing) is documented in [docs/HOTEL_BOOKING_REFERENCE.md](../HOTEL_BOOKING_REFERENCE.md). Read both.

---

## 0. Base hosts, version, auth

- **Search / static data / reviews:** `https://api.liteapi.travel/v3.0`
- **Booking lifecycle (prebook, book, retrieve, list, cancel, amend, rebook):** `https://book.liteapi.travel/v3.0`
- **Auth:** `X-API-Key: <key>` on every request. Sandbox keys prefixed `sand_`; responses carry `sandbox: true`/`1`.
- **Content:** requests `application/json`; responses `application/json` (often gzip). Most read/write endpoints accept an optional `timeout` query param (float seconds; default 4 on GETs, 30 in book/rebook examples).

> This split-host is why WhereTo has two edge functions: `liteapi-stays` (search, `api.` host) and `liteapi-book` (prebook/book, `book.` host). Both already correct.

---

## 1. Two-step search pattern

LiteAPI search is always: **(1) get hotel IDs from static data, then (2) POST those IDs to rates.** Rates are not returned by a single city query.

### 1a. `GET /data/hotels` - list hotels / hotel-ID prep

Host `api.liteapi.travel/v3.0`. Two search modes: **city** (`countryCode` + `cityName`) or **radius** (`latitude` + `longitude` + `radius` ≥ 1000m; radius mode is recommended as more precise).

Key query params (full list is long): `countryCode`, `cityName`, `latitude`, `longitude`, `radius` (meters, min 1000), `aiSearch` (natural-language), `hotelName`, `hotelIds` (comma-sep), `placeId` (Google Place ID; also returns a `place` object), `zip`, `limit` (default 200, **max 5000**), `offset`, `minRating`, `minReviewsCount`, `starRating` (comma-sep `1.0`–`5.0`), `facilityIds`, `hotelTypeIds`, `chainIds`, `strictFacilitiesFiltering`, `advancedAccessibilityOnly`, `lastUpdatedAt` (RFC3339 incremental sync), `language`, `timeout`.

**Response** `{ data[], hotelIds[], total, place? }`. Each `data[]` hotel: `id` (e.g. `lp42fec`), `name`, `hotelDescription`, `currency`, `country`, `city`, `latitude`, `longitude`, `address`, `zip`, `main_photo`, `thumbnail`, `stars` (1–5), `rating` (guest, e.g. 8.5), `reviewCount`, `hotelTypeId`, `chainId`, `chain` (name or `"Not Available"`), `facilityIds[]`, `rohId`, `score` (0–1, placeId searches only), `accessibilityAttributes{}`. `hotelIds[]` is a flat convenience array; `total` = total matches.

Source: `reference/get_data-hotels`, `docs/preparing-a-list-of-hotel-ids`.

### 1b. `GET /data/hotel` - full hotel detail

Query: `hotelId` (req), `timeout` (default 4), `language`, `advancedAccessibilityOnly`.
Returns a rich `data` object: core (`id`, `name`, `hotelDescription`, `hotelImportantInformation`, `country`, `city`, `address`, `starRating`, `chain`, `hotelType`, `phone`, `email`, `rating`, `reviewCount`, `airportCode`, `groupRoomMin`), `location{latitude,longitude}`, media (`main_photo`, `thumbnail`, `videoUrl`, `hotelImages[]{url,urlHd,caption,order,defaultImage}`), `checkinCheckoutTimes{checkin_start,checkin_end,checkout,instructions[],special_instructions}`, `hotelFacilities[]` (name strings) + `facilities[]{facilityId,name}`, `rooms[]` (`id`, `roomName`, `description`, `roomSizeSquare`, `roomSizeUnit`, `maxAdults`, `maxChildren`, `maxOccupancy`, `bedTypes[]`, `roomAmenities[]`, `photos[]`), `policies[]`, and `sentiment_analysis{pros[],cons[],categories[]}` + `poi[]{name,category,distanceKm,importance}`.

Source: `reference/get_data-hotel`, `docs/displaying-hotel-details`.

> WhereTo already consumes this in `LiteApiProvider.fetchHotelDetail` (strips HTML from `hotelDescription`, prefers `urlHd`, reads `checkinCheckoutTimes`, merges `hotelFacilities`/`facilities`).

---

## 2. `POST /hotels/rates` - live rates

Host `api.liteapi.travel/v3.0`. **Required:** `checkin`, `checkout` (YYYY-MM-DD), `currency`, `guestNationality` (ISO-2 - material to pricing; some rates are nationality-specific), `occupancies[]` (each `{adults (int, req), children (int[] ages), rooms?}`).

**Location selector (one of):** `hotelIds[]`, `cityName`+`countryCode`, `latitude`+`longitude`+`radius`, `iataCode`, `placeId`, `aiSearch`.

**Filtering / optimization:** `maxRatesPerHotel` (set `1` for cheapest-only), `timeout` (4–12s recommended), `limit` (default 200, max 5000), `offset`, `starRating[]`, `minRating`, `minReviewsCount`, `hotelName`, `hotelTypeIds[]`, `chainIds[]`, `facilities[]` (OR by default), `strictFacilityFiltering`, `zip`, `advancedAccessibilityOnly`.
**Rate/amenity filters:** `boardType` (e.g. `RO`,`BB`,`HB`; comma-sep), `refundableRatesOnly`, `roomAmenities[]`, `roomAmenitiesFilter` (grouped logic), `amenityFilterLogic` (`AND`|`OR`), `bedTypes[]`.
**Sorting/data options:** `sort[]{field:top_picks|price|revenue, direction:ascending|descending}`, `roomMapping` (bool → adds `mappedRoomId`/normalized rooms), `includeHotelData` (embeds static hotels in `hotels[]`), `stream` (SSE - see §4), `feed`, `margin` (override markup % - see the conflict note in §7), `sessionId` (price-consistency per search session).

**Response** top level: `{ data[], guestLevel, sandbox, hotels[]? }`.
- `data[]` per hotel: `hotelId`, `roomTypes[]`.
- `roomTypes[]`: `roomTypeId`, `name`, **`offerId`** (long opaque token for prebook - store whole), `supplier`, `supplierId`, `rates[]`, `offerRetailRate[]{amount,currency}`, `suggestedSellingPrice[]{amount,currency,source?}`, `offerInitialPrice[]`, `priceType` (`commission`), `rateType` (`standard`|`package`).
- `rates[]`: `rateId`, `occupancyNumber`, `name`, `maxOccupancy`, `adultCount`, `childCount`, `childrenAges[]`, `boardType`, `boardName`, `remarks`, `priceType`, `commission[]{amount,currency}`, `retailRate{...}`, `cancellationPolicies{...}`, `paymentTypes[]`, `perks[]`.
  - `retailRate.total[]{amount,currency}` = final price to book (incl. commission + `included:true` taxes). `retailRate.suggestedSellingPrice[]`, `retailRate.initialPrice[]`, `retailRate.taxesAndFees[]{included(bool),description,amount,currency}`, `retailRate.promotions[]{name,from,to,discount,discountType,promotionCode,currency}`.
  - `cancellationPolicies.cancelPolicyInfos[]{cancelTime, amount, currency, type("amount"|"percentage"), timezone("GMT")}`, `hotelRemarks[]`, `refundableTag` (`RFN`|`NRFN`).
  - `paymentTypes[]`: e.g. `NUITEE_PAY`, `PROPERTY_PAY`. `perks[]{perkId,name,amount,currency,level("HOTEL"|"RATE"|"ROOM")}`.

**Example request:**
```json
{
  "hotelIds": ["lp1897"],
  "occupancies": [ { "rooms": 1, "adults": 2, "children": [11] } ],
  "currency": "USD", "guestNationality": "US",
  "checkin": "2027-01-15", "checkout": "2027-01-17",
  "timeout": 6, "roomMapping": true
}
```

A full example response (one rate) with taxes, promotions, cancellation policy and perks is preserved in the research notes; the load-bearing fields are the list above. WhereTo's `LiteApiProvider.searchHotels` already parses `offerRetailRate.amount`, sums `taxesAndFees` (included vs not), reads `refundableTag === 'RFN'` and `cancelPolicyInfos[0].cancelTime`, and dedupes room classes by `(name, boardType, refundable)`.

Source: `reference/post_hotels-rates`, `docs/step-2-requesting-room-rates`, `docs/rate-request-parameters-guide`.

---

## 3. `POST /hotels/min-rates` - cheapest rate per hotel

Lightweight: returns only the single cheapest rate per hotel (small payload, faster) for list/search-results pages. Same required fields as `/hotels/rates`.
**Response:** `{ data[]{hotelId, price, suggestedSellingPrice, offerId}, sandbox }`. The `offerId` carries into the booking flow like a full-rates offer.

Source: `reference/post_hotels-min-rates`.

> WhereTo opportunity: our search currently calls full `/hotels/rates` for the list. Switching the list view to `/hotels/min-rates` would cut payload/latency; keep full `/hotels/rates` for the detail/room-selection view.

---

## 4. Streaming rates (SSE)

Same endpoint `POST /hotels/rates` with `"stream": true`. Response `Content-Type: text/event-stream`. Each event is `data: <json>\n\n`:
- **Rates events:** `{"rates":[{"hotelId":"...","roomTypes":[...]}]}` (multiple hotels per line).
- **Hotels events:** `{"hotels":[{"id","name","photo",...}]}` (static summaries near the end).
- **Termination:** `data: [DONE]` then close.

Client: read incrementally, split on `\n\n`, strip `data: `, `JSON.parse`, stop on `[DONE]`.

Source: `docs/stream-hotel-rates`.

---

## 5. Rate JSON model - net / retail / SSP / commission / taxes

- **`retailRate.total`** - the amount due to book. Includes commission and any `included:true` taxes/fees. **Charge this.**
- **Net rate** - no separate field; achieved by setting markup/commission to `0` (`priceType` stays `"commission"`).
- **`suggestedSellingPrice` (SSP)** - public reference price with `source` (e.g. `booking.com`). Compare-at ceiling, not what you must charge. Selling below SSP publicly is a rate violation (see `COMMERCIALS.md` on Closed User Groups for member pricing).
- **`commission[]{amount,currency}`** - commission embedded in the retail rate.
- **`taxesAndFees[]`**: `included:true` → already in `total`; `included:false` → guest pays at property (surface separately). `taxesAndFees: null` → all-in, no extra liability. Buckets can be granular (per-line) or aggregated (`"OTHERS"`, `"RESORT"`).
- **`rateType`**: `standard` | `package`.

Source: `docs/hotel-rates-api-json-data-structure`.

---

## 6. Room mapping & room detail

`roomMapping: true` adds a `mappedRoomId` and normalized room data (only if the hotel is mapped in Nuitee), letting you group equivalent rooms across suppliers for clean room cards. Room-describing fields live on `roomTypes[].rates[]` (`roomTypeId`, `name`, `boardType`/`boardName`, `maxOccupancy`, counts, `rateId`, `offerId`, `mappedRoomId`); full room objects come from `/data/hotel` `rooms[]`. Always display supplier `roomName` (it carries info sent to the hotel).

**Confirmed against `docs.liteapi.travel/docs/room-details` (2026-07-16):** `mappedRoomId` joins directly to `rooms[].id` — same ID, not a separate cross-supplier namespace. Each `rooms[]` entry: `id`, `roomName`, `description`, `roomSizeSquare`/`roomSizeUnit`, `maxAdults`/`maxChildren`/`maxOccupancy`, `bedTypes[]{quantity,bedType,bedSize}`, `roomAmenities[]{amenitiesId,name,sort}`, `photos[]{url,imageDescription,failoverPhoto,mainPhoto,score}` — sort by `mainPhoto` when picking a single cover photo. If the join comes back empty for a hotel, the room-mapping coverage gap is the likely cause (confirmed empty on 2+ independently-tested properties), not a field-name/nesting bug — but normalize IDs to string on both sides regardless of which type the wire actually sends, since a strict `typeof x === 'number'` check would silently break the join if either side ever serializes as a numeric string.

Source: `docs/room-details`.

---

## 7. Reference data for search UIs

- **`GET /data/places`** - place autocomplete/geocode. Query: `textQuery` (req), `type` (`locality`|`airport`|`hotel`|`lodging`|…), `language`, `clientIP`, `sessionId` (Google Places billing session; or `X-Places-Session-Id` header). Response `{ data[]{placeId, displayName, formattedAddress, types[]} }`. **$0.01/request** (see `COMMERCIALS.md`).
- **`GET /data/cities`** - `countryCode` (req) → `{ data[]{city} }`.
- **`GET /data/countries`** - `{ data[]{code, name} }` (~249).
- **`GET /data/facilities`** - full catalog `{ data[]{facility_id, facility, sort, translation[]{lang,facility}} }`. Map IDs to the `facilityIds`/`facilities` filters.
- **`GET /data/reviews`** - query `hotelId` (req), `limit` (default 200, max 5000), `offset`, `timeout`, `getSentiment` (bool → AI sentiment of ~last 1000 reviews), `language`. Response `{ data[]{averageScore, country, type, name, date, headline, language, pros, cons, source}, total }`. `/data/hotel` also exposes persisted `sentiment_analysis{pros[],cons[],categories[]}`.

> Facility field casing differs by endpoint: `/data/facilities` = `facility_id`+`facility`; `/data/hotel` = `{facilityId,name}`; `/data/hotels` = `facilityIds` (int[]). Normalize on our side.

Sources: `reference/get_data-places`, `get_data-cities`, `get_data-countries`, `get_data-facilities`, `get_data-reviews`.

---

## 8. PREBOOK - `POST /rates/prebook`

Host **`book.liteapi.travel/v3.0`**. This is the confirm/verify step: it re-validates availability + price and returns a `prebookId`.

**Request:**

| Field | Type | Req | Notes |
|---|---|---|---|
| `offerId` | string | Yes | selected offer from rates |
| `usePaymentSdk` | boolean | Yes | true → returns payment-SDK fields (`transactionId`, `secretKey`) |
| `voucherCode` | string | No | discount code (e.g. a loyalty redemption voucher - see `COMMERCIALS.md`) |
| `addons` | array | No | extras folded into price (see §12) |
| `bedTypeIds` | int[] | No | preferred bed config |
| `includeCreditBalance` | boolean | No | include credit-line info |

**Response `data`** (+ root `guestLevel`): `prebookId`, `offerId`, `hotelId`, `checkin`, `checkout`, `currency`, `termsAndConditions`, `roomTypes[]` (with nested `rates[]`), `suggestedSellingPrice{amount,currency,source}`, `commission` (total), `addonsTotalAmount`, **`price`** (final, all rooms - charge this), `priceType`, **`priceDifferencePercent`**, **`cancellationChanged`**, **`boardChanged`**, `supplier`/`supplierId`, **`transactionId`** + **`secretKey`** (only when `usePaymentSdk:true`), `paymentTypes[]`, `mappedRoomId`, `voucherCode`/`voucherTotalAmount`, `addonsRequest[]`, `creditLine{remainingCredit,currency}`.

### Price-change detection - the three fields to check before BOOK
1. **`priceDifferencePercent`** should be `0` (non-zero = live price moved; show new `price`).
2. **`cancellationChanged`** should be `false`.
3. **`boardChanged`** should be `false`.
Charge `data.price`.

**Prebook errors:** `400/4002` invalid offerId; `401`; `408/4016` prebook timeout (increase `timeout`); `408/4040` outdated offerId → refetch rates.

Source: `reference/post_rates-prebook`, `docs/step-3-pre-booking-a-room`.

> WhereTo already does exactly this: `prepareHotelCheckout` re-quotes fresh (offers expire), re-matches the room by class key, prebooks the fresh offer, and surfaces `priceChanged`. Our `liteapi-book` `doPrebook` extracts `prebookId`/`transactionId`/`secretKey` defensively - tighten field names against a live sandbox response (the `data`-wrapped names above are authoritative).

---

## 9. BOOK - `POST /rates/book`

Host **`book.liteapi.travel/v3.0`**. Query `timeout` (e.g. 30).

**Required:** `prebookId`, `holder`, `guests[]`, `payment`.
- **`holder`** (pays): `firstName`, `lastName`, `email`, `phone`.
- **`guests[]`** (one primary contact per room): `occupancyNumber` (int, req - **room number starting at 1, NOT a headcount**), `firstName`, `lastName`, `email`, optional `remarks`, `phone`.
- **`payment`** variants:
  - Payment-SDK / transaction: `{ "method": "TRANSACTION_ID", "transactionId": "tr_ct_..." }` (see the enum caveat below).
  - Simple: `{ "method": "ACC_CREDIT_CARD" }` (sandbox-safe simulated card), `{ "method": "WALLET" }`, `{ "method": "CREDIT" }` (account credit).

**Optional top-level:** `clientReference` (idempotency - see §10), `metadata{ip,country,language,platform,device_id,user_agent,utm_*}` (fraud), `customTags` (≤5 KV, keys `^[A-Z0-9_-]+$`), `guestPayment{method,phone,payee_first_name,payee_last_name,last_4_digits,address{}}` (recommended for fraud detection).

**Response `data`** (+ `guestLevel`, `sandbox`): required `bookingId`, `status` (`CONFIRMED`|`CANCELED`), `checkin`, `checkout`, `currency`, `price`, `createdAt`. Plus `clientReference`, `supplierBookingId`, `hotelConfirmationCode` (hotel's own code, may be assigned later), `hotel{hotelId,name}`, `bookedRooms[]`, `holder{}`, `commission`, `guestId`, `prebookId`, `sellingPrice`, `tag` (`RFN`/`NRFN`), `lastFreeCancellationDate`, `cancelledAt`/`refundedAt` (nullable), `addons[]`, etc.

**Treat `status: "CONFIRMED"` as success.**

**Book errors:** `4000` missing/unsupported payment method; `4002` invalid prebookId/missing txn id; **`4005` duplicate clientReference (idempotency guard)**; `4010` prebook validation failed; `2010/2013/2014` not confirmed / provider unsuccessful / incomplete; `5000` internal / payment failed; `401`.

Source: `reference/post_rates-book`, `docs/step-4-booking-a-room`, `docs/adding-guests-during-the-booking-step`.

> ⚠️ **Payment-method enum - must verify against sandbox.** The current book *reference example* shows `payment.method: "TRANSACTION_ID"`. WhereTo's code (`src/providers/hotels/types.ts`, `liteapi-book`) and older LiteAPI usage use `"TRANSACTION"`. Flights confirmed `"TRANSACTION_ID"`. This exact string is load-bearing for a successful charge. Confirm the accepted value (`TRANSACTION` vs `TRANSACTION_ID`) against the live sandbox/Postman before the first live booking. Same caution for `ACC_CREDIT_CARD` vs the `guestPayment.method` example value `CREDIT_CARD`.

---

## 10. Idempotency - `clientReference`

Associates a booking with a unique reference from our system for tracking + dedup. "It is not meant to be just the user's ID, as it needs to be unique." A duplicate BOOK/REBOOK with a used `clientReference` returns `4005`. Generate a unique value per attempt, reuse on retries.

> WhereTo derives it deterministically as `userId:prebookId` in `hotelBooking.ts` - good, this makes retries idempotent.

Source: `docs/step-4-booking-a-room`.

---

## 11. Retrieve / list bookings

- **Retrieve prebook - `GET /prebooks/{prebookId}`** - mirrors the prebook `data` object exactly (+ `guestLevel`, `sandbox`).
- **Retrieve booking - `GET /bookings/{bookingId}`** - the richest booking view. Adds `paymentStatus` (`succeeded`/`refunded`/`requires_capture`), `paymentTransactionId`, refund fields (`amountRefunded`, `refundType`), `checkinInstructions{idRequired,propertyContact{email,phone},instructions,...}`, `customTags`, `rebookFrom`, etc. `204 No Content` = not found.
- **List bookings - `GET /bookings`** has **two documented shapes** (a real doc inconsistency - code defensively):
  - **`listbookings`** (camelCase, current): query `guestId`, `clientReference`, `customTags` (`KEY:VALUE`), `timeout`. Items use a `rooms[]` array; statuses `CONFIRMED`|`CANCELED`|`CANCELLED_WITH_CHARGES`.
  - **`get_bookings`** (snake_case, with date filters): query `startDate`/`endDate` (stay), `bookingStartDate`/`bookingEndDate` (created), `status`, `paymentStatus`, `Sandbox`, `customTags`. Response `{count, data[]}` with flat snake_case fields (`booking_Id`, `payment_status`, `hotel_confirmation_code`, …). Note the misspelled `voucher_transation_id`.

Sources: `reference/get_prebooks-prebookid`, `get_bookings-bookingid`, `listbookings`, `get_bookings`.

> WhereTo pending: "My Bookings" reads only the Duffel `orders` table; add a `hotel_orders` fetch + union. Retrieval by our stored `liteapi_booking_id` or by `clientReference` (via `listbookings`) both work.

---

## 12. Add-ons / amenities

Attached **at prebook** via the `addons` array, folded into `price` and embedded in `prebookId` (nothing extra at BOOK; the BOOK response echoes `addons[]` with fulfillment data). Supported: **Uber voucher** (`addon:"uber"`, `value` $10–$100 increments, USD only) and **eSIM** (`addon:"esimply"`, `value` = eSIM API price, USD, plus `addonDetails{package_id,destination_code,start_date,end_date}`). BOOK response `addons[]` carries `addonVoucherCode`, `addonVoucherId`, `expiryDate`, `status`.

Source: `docs/attaching-add-ons-to-user-payment`.

> WhereTo opportunity: Uber credit + eSIM are natural trip add-ons for a discovery app - an ancillary revenue lever with zero extra booking calls.

---

## 13. Cancel - `PUT /bookings/{bookingId}`

Host `book.liteapi.travel/v3.0`. No body. Query `timeout` (default 4).
**Response `data`:** `bookingId`, `status`, `cancellation_fee`, `refund_amount`, `currency`.
- **Refundable (`RFN`):** `status → CANCELLED`, `refund_amount > 0` (price minus any fee).
- **Non-refundable (`NRFN`):** cancel still terminates the booking but does not refund. `status → CANCELLED_WITH_CHARGES`, `refund_amount: 0`.
- Refund is governed by `refundableTag` + the cancellation-policy timeline; when multiple `cancelPolicyInfos` exist, the policy whose `cancelTime` most recently passed applies (`amount: 0` = fully refundable). `lastFreeCancellationDate` marks the free-cancel cutoff.
- Errors: `204` not found; `304` unable to process; `401`.

Sources: `reference/put_bookings-bookingid`, `docs/canceling-a-booking`.

> WhereTo: `HotelProvider.cancel?()` is defined but optional/unimplemented. This endpoint is what wires it up. Needed for a real "My Bookings → Cancel" flow.

---

## 14. Amendments

**Soft (name only) - `PUT /bookings/{bookingId}/amend`:** holder name/email/phone only ("other guest details cannot be amended"). Body `{ holder{firstName,lastName,email,phone?}, remarks? }`. Creates an async **PENDING** amendment (`type:UPDATE`, `subType:UPDATE_NAME_CHANGE`), actioned by ops/hotel.

**Hard (dates/occupancy) - two steps:**
1. **`POST /bookings/{bookingId}/alternative-prebooks`** - body requires `occupancies[]`; optional `checkin`, `checkout`, `refundableRatesOnly`, `boardType`, `maxPrebooks` (default 3, max 10). Returns an array of ready-to-book alternative prebook sessions (each with its own `prebookId`, `price`, `priceDifferencePercent`, etc.) at the **same hotel**. Original must be CONFIRMED.
2. **`POST /rates/rebook`** - body requires `prebookId` (from step 1), `existingBookingId`, `holder`, `guests[]`, `payment` (schema-required but ignored - server forces `method:"NONE"`, no payment collected). Optional `clientReference` (idempotency; `4005` on dup). **Auto-cancels the old booking on success** ("Do not need to call the cancel endpoint"). Refundable original → `200` new booking CONFIRMED immediately (`rebookFrom` = old id). Non-refundable original → `202 Accepted` queued to ops (`type:REBOOK`, `status:PENDING`, with an `audit_trail` of old/new values). Errors: `400` (`4002`/`4000`), `409/4009` "rebook already completed".

Sources: `reference/put_bookings-bookingid-amend`, `post_bookings-bookingid-alternative-prebooks`, `post_rates-rebook`.

---

## 15. Booking statuses & timelines

- **`CONFIRMED`** - active. **`CANCELED`/`CANCELLED`** - cancelled (both spellings appear; retrieve/list use `CANCELLED`). **`CANCELLED_WITH_CHARGES`** - non-refundable cancel, `refund_amount:0`.
- Amendment/rebook async records: `PENDING` with `type` `UPDATE`/`REBOOK`, `subType` `UPDATE_NAME_CHANGE`/`CHECKIN_CHANGE`/`OCCUPANCY_CHANGE`.
- `paymentStatus`: `succeeded`, `refunded`, `requires_capture`. `refundableTag`/`tag`: `RFN`/`NRFN`.
- **Prebook TTL is not published**; the practical constraint is offer expiry (`408/4040` → refetch). Prebook immediately before booking and re-check the three price-change flags. Cancellation timeline is driven by `cancelPolicyInfos[].cancelTime` (always GMT).

---

## 16. Gotchas / doc inconsistencies

1. Older guide pages write `/hotels-rates` + header `AUTHORIZATION` + `checkIn`/`checkOut`. Authoritative: `POST /hotels/rates`, `X-API-Key`, `checkin`/`checkout`.
2. **`margin` conflict:** the `/hotels/rates` reference lists a `margin` body field; the parameters guide says none exists at search time. Verify against our account (markup is normally account-side).
3. **Payment-method enum:** `TRANSACTION` vs `TRANSACTION_ID` (see §9) - verify before first live booking.
4. `offerId`/`rateId` are long opaque tokens - store whole, never truncate.
5. Sum `included:false` taxes and show as "payable at property"; `taxesAndFees:null` = all-in.
6. Two `GET /bookings` shapes (camelCase vs snake_case + `count`) - code defensively.
7. Radius min 1000m; `limit` max 5000 (default 200).
8. Misspelled field `voucherTransationId`/`voucher_transation_id` as returned by the API.

---

## 17. WhereTo hotel implementation - current state & gaps

**Built and correct:** `LiteApiProvider.ts` (two-step search + detail + reviews/sentiment + room-class dedup), `liteapi-stays` (server search proxy, `api.` host), `liteapi-book` (quote/prebook/book, `book.` host, key server-side, writes `hotel_orders`, idempotent `clientReference`), the 4-step `HotelWizard`, and the unified `hotelBooking.ts` workflow (re-quote → prebook → price-check → book).

**Pending (from [HOTEL_BOOKING_REFERENCE.md](../HOTEL_BOOKING_REFERENCE.md) §11, still open):**
1. Run the `hotel_orders` SQL migration; set `LITEAPI_KEY` secret; deploy `liteapi-book`.
2. **Payment UI** (Model A, the deferred fork) - Stripe SDK in a WebView after prebook, returning `transactionId` → `confirmHotelBooking({ payment: { method:'TRANSACTION'…, transactionId } })`. See `PAYMENTS.md`. **Verify the `TRANSACTION`/`TRANSACTION_ID` enum here.**
3. Route search through `liteapi-stays` and drop the client key `EXPO_PUBLIC_LITEAPI_KEY`.
4. `send-notification` `hotel_confirmation` branch.
5. "My Bookings" `hotel_orders` union.
6. Tighten prebook/book field extraction against a live sandbox response (this doc's `data`-wrapped names are authoritative).
7. Wire `cancel()` via `PUT /bookings/{id}` for a real cancellation flow.

---

_Cross-refs: [docs/HOTEL_BOOKING_REFERENCE.md](../HOTEL_BOOKING_REFERENCE.md) (app-level flow), `PAYMENTS.md` (Stripe/MoR), `COMMERCIALS.md` (rates, commission, vouchers, loyalty), `PLATFORM.md` (hosts, auth, webhooks), `README.md` (transition plan)._
