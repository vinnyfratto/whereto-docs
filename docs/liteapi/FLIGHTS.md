# LiteAPI (Nuitee Connect) - Flights API Reference

**Scope:** the full flight booking lifecycle (search, flex matrix, verify, prebook, attach services, book, retrieve), reference data, error/retry rules, and the Duffel to LiteAPI migration map.
**Last researched:** 2026-07-14 against the flight `.md` reference pages and guide pages.

---

## 0. TL;DR for the migration

- **Base host + version:** `https://api.liteapi.travel/v3.0` for **every** flight endpoint. Same host and same `X-API-Key` as the Hotels data/search API.
  - **Important asymmetry vs hotels:** hotel *booking* calls use `book.liteapi.travel`. **Flights do NOT** - flight booking (`/flights/bookings`) is on the `api.` host. Our future `liteapi-flight-book` edge function must point at `api.liteapi.travel/v3.0`, not `book.`.
- **The book endpoint (the critical unknown, now resolved):** `POST /flights/bookings`. There is no `/flights/book` or `/rates/book` for flights (`post_flights-book.md` 404s). Finalize with `{ prebookId, payment: { method, transactionId } }`.
- **Flow:** `POST /flights/rates` (search) → `POST /flights/verify` → `POST /flights/prebooks` (mints a Stripe PaymentIntent: `transactionId` + `secretKey`) → optional `POST /flights/prebooks/{prebookId}/services` (mints a **new** `transactionId`/`secretKey`) → confirm Stripe client-side → `POST /flights/bookings` → `GET /flights/bookings/{bookingId}`.
- Every response is wrapped in `{ "data": [ ... ] }`.
- **Flights are not enabled by default (even sandbox)** - you must request access in the dashboard.
- Payment specifics (Stripe SDK, merchant of record) live in `PAYMENTS.md`; this doc covers the flight-side contract.

---

## 1. Base host, version, auth, environments

- **Base URL:** `https://api.liteapi.travel/v3.0` (single host for all flight endpoints).
- **Auth:** `X-API-Key: <key>`. "Flights uses the same API key and authentication method as the Hotels API. No additional credentials are required once access is enabled." Environment is determined by the key, not a different host.
- **Environment hostnames appear only inside asset URLs** (airline/provider logos), not as API bases:
  - Sandbox assets: `https://sandbox.nuitee.flights/static/images/airlines/DE.png`
  - Production assets: `https://production.nuitee.flights/static/...`
  - The booking response also carries `booking.providerEnvironment: "sandbox" | "production"`.

Source: `docs/getting-access-to-flights` + every flight reference page.

---

## 2. Access / entitlement

- **Flights are NOT enabled by default, including sandbox.** Must be explicitly requested.
- **Workflow:** submit via the dashboard "Request Assistance" feature; state use case + access type (sandbox / production / Whitelabel). A sandbox integration must be completed **before** production is granted.
- **Sandbox:** non-production data, limited pricing/availability accuracy; for validating request/response structure.
- **Production:** activated by the Nuitee team (provider restrictions, stricter rate limits, commercial review). **Exact rate-limit numbers are not published** ("stricter rate limits").
- **Whitelabel:** flights can be enabled there too but are off by default (error `"Flights has not been enabled for your Whitelabel configuration"`).

### Content sourcing (GDS / NDC / LCC)
- Flights aggregate **GDS + NDC + low-cost carriers** into one uniform schema.
- Providers surface in payloads: `verify` returns `provider.code: "SABRE"`; the prebook `loyaltyPrograms` field is "Sabre/Travelport only." Expect at least **Sabre** and **Travelport** GDS backends behind the abstraction.

Source: `docs/getting-access-to-flights`, liteapi.travel / nuitee.com marketing pages (sourcing claims).

> WhereTo action: request flight access (sandbox first) in the LiteAPI dashboard before any flight build can be tested. This is a hard prerequisite and a lead-time item.

---

## 3. End-to-end flow, state machine, TTLs

**State machine:** `SEARCHING → VERIFIED → PREBOOKED → SERVICES_ATTACHED → BOOKING → CONFIRMED` (or `FAILED` at BOOKING). Valid shortcut: `PREBOOKED → BOOKING` (skip services). From `FAILED`, restart at `SEARCHING`.

**TTLs (server-side; expired offers bounce the user back to SEARCHING):**

| State | TTL | Note |
|---|---|---|
| SEARCHING | 30 min | user browsing |
| VERIFIED | 5 min | offer has `expiration` |
| PREBOOKED | 15 min | provider hold varies |
| SERVICES_ATTACHED | 15 min | same as prebook |
| BOOKING (in-flight) | 2 min | network/provider timeout |

Offer-level: search offers carry `expiration` (ISO 8601) and "expire in minutes, sometimes seconds." `servicesAttachable.expiresAt` gives the seat/bag attach deadline.

**Step-by-step:**

| # | Step | Method + Path | Key request | Key response |
|---|---|---|---|---|
| 1 | Search | `POST /flights/rates` | `legs[]`, `adults`, `currency` | `data[].journeys[].offers[].offerId`, `.expiration`, `journeyKey` |
| 1b | Flex matrix | `POST /flights/rates/matrix` | `legs[]`, `adults`, `currency`, `flexDays` | `cells[]{outboundDate, price}`, `cheapest` |
| 2 | Verify | `POST /flights/verify` | `offerId` | `journey.expiration`, `journey.pricing`, `changes{}` |
| 3 | Prebook | `POST /flights/prebooks` | `offerId`, `usePaymentSdk`, `contact{}`, `passengers[]` | `prebookId`, `transactionId`, `secretKey`, `paymentTypes[]`, `servicesAttachable{}` |
| 4 | Attach services (opt) | `POST /flights/prebooks/{prebookId}/services` | `selectedServices[]` | **NEW** `transactionId` + `secretKey`, updated `price` |
| 5 | Pay | Stripe, client-side | confirm PaymentIntent with latest `secretKey` | `paymentIntent.status === 'succeeded'` |
| 6 | Book | `POST /flights/bookings` | `prebookId`, `payment{method, transactionId}` | `booking.bookingId`, `.bookingRef`, airline PNR |
| 7 | Retrieve | `GET /flights/bookings/{bookingId}` | path `bookingId` | full `booking{}` incl. `status` |

Source: `docs/flight-booking-architecture`, `docs/build-a-flight-booking-experience`, `docs/booking-flow-recipes`.

---

## 4. Search - `POST /flights/rates`

**Request (required):** `legs` (array, min 1), `adults` (int ≥1), `currency` (ISO 4217).

`legs[]`: `origin` (IATA), `destination` (IATA), `date` (YYYY-MM-DD), `direction` (`OUTBOUND`|`INBOUND`, optional), `filters` (per-leg override, optional).
- **One-way** = 1 leg; **round-trip** = 2 legs; **multi-city** = 3+ ordered legs.

**Passengers:** `children` (ages 2–11), `infants` (under 2), `childrenAges[]`, `infantAges[]`.
**Global:** `cabinClass` (`ECONOMY`|`PREMIUM_ECONOMY`|`BUSINESS`|`FIRST`), `country` (ISO alpha-2 point-of-sale), `filters`, `sort`.

`filters` (global or per-leg): `maxStops` (-1/omit=any, 0=nonstop, 1=≤1, 2=≤2), `maxPrice`, `minPrice`, `maxDuration` (min), `cabinClass`, `cabinClassMatch` (`exactly`|`at_least`), `refundableOnly`, `changeableOnly`, `includesCarryOnBag`, `includesCheckedBag`, `excludeOvernight`, `excludeConnectionAirports[]`, `flightNumbers[]`, `flightNumbersMatch` (`any`|`all`), `departureTimeBefore/After`, `arrivalTimeBefore/After` (HH:MM 24h), `legDurations[]{direction,maxMinutes}`, `showCheapestOfferOnly`.

`sort`: `sortBy` (`price`|`duration`|`departure`|`arrival`|`stops`), `sortOrder` (`asc`|`desc`).

**Example request:**
```json
{
  "legs": [
    { "origin": "JFK", "destination": "CDG", "date": "2026-07-01", "direction": "OUTBOUND" },
    { "origin": "CDG", "destination": "JFK", "date": "2026-08-02", "direction": "INBOUND" }
  ],
  "adults": 1, "children": 0, "infants": 0,
  "currency": "USD", "country": "US", "cabinClass": "ECONOMY",
  "filters": { "maxStops": 1, "maxPrice": 1500, "refundableOnly": false },
  "sort": { "sortBy": "price", "sortOrder": "asc" }
}
```

**Response** - `{ "data": [ resultSet ] }`:
- `resultSet.journeys[]`, `resultSet.sortMetadata` (nullable).
- **journey:** `journeyKey`, `isCheapest`, `cheapestOffer`, `offers[]` (price asc), `segments[]`, `parameters{adults,children,infants}`, `timestamp`, `legDurations[]`, `totalDuration`, `connections[]`.
- **segment:** `segmentKey`, `originCode`, `originName`, `destinationCode`, `destinationName`, `departureTime`, `arrivalTime`, `direction`, `duration{iso8601,minutes}`, `flight{marketingNumber, operatingNumber}`, `carrier{marketingCode, marketingName, marketingLogo, operatingCode, operatingName, operatingLogo}`.
- **FlightOffer:** `offerId`, `expiration`, `pricing`, `fare{family, mixedCabin, seatsRemaining}`, `terms`, `baggage`, `segmentFares[]`, `segmentAmenities[]`.
  - `pricing.display{total, currency, base, fees, taxes, perPassenger{adult,child,infant{base,currency,fees,taxes,total}}}`, `pricing.converted` (bool = FX-converted).
  - `terms{refundable, changeable, summary[]{level,message}, changeFee, refundFee, hasChangeFee, hasRefundFee}` (fees: `{pricing{display{amount,currency},converted}, percent, applicability, label}` or null; `applicability` ∈ `beforeDeparture|afterDeparture|anytime|noShow`).
  - `baggage{hasCarryOnBag, hasCheckedBag, included[]{bagType, description, passengerType, pieces, pricing, unit, weightKg}}` (`bagType` ∈ `cabin|checked|personal`; `passengerType` ∈ `ADT|CHD|INF|ALL`).
  - `segmentFares[]{segmentKey, bookingCode, cabin, fareBasisCode, fareFamily, seatsRemaining}`.
  - `segmentAmenities[]{segmentKey, aircraftType, amenities[]{available, category, chargeable, name, details}}` (categories e.g. `wifi`, `entertainment`).
- **connection (layover):** `arrivalAirportCode/Name`, `arrivalTime`, `changeAirport`, `departureAirportCode/Name`, `departureTime`, `direction`, `duration`, `overnight`.

> `offerId` is an opaque **msgpack-encoded base64 blob** (e.g. `h6NwaWTZJDAx...`), not a UUID. Store whole, never parse or truncate.

**Status codes:** 200 / 400 / 401 / 502 (provider) / 503.
**Streaming (SSE):** send `Accept: text/event-stream` on the POST, or use `POST /flights/rates/stream`. Events: `rates-chunk`, `rates-complete`, `error`, `message`.

Source: `reference/post_flights-rates`.

---

## 5. Flexible-date matrix - `POST /flights/rates/matrix`

> Doc slug is `searchflightsmatrix`, but the actual path is `/flights/rates/matrix`.

**Request:** `legs` (1 = one-way, 2 = round-trip), `adults` (≥1), `currency` (all required); optional `children`, `infants`, `country`, `flexDays` (default 3; ±days window).
**Response** `{ "data": [ FlightMatrixData ] }`:
- `FlightMatrixData{baseOutboundDate, baseReturnDate (null one-way), cells[], cheapest, currency, flexDays, roundTrip}`.
- `FlightMatrixCell{outboundDate, outboundOffset (−flex..+flex), returnDate, returnOffset, price (float|null), currency, success, cached}`.

**Status:** 200 / 400 / 401 / **403 (matrix not enabled)** / 500. SSE: `matrix-start`, `matrix-chunk`, `matrix-complete`.

Source: `reference/searchflightsmatrix`.

> WhereTo fit: this maps well to our date-flex feature (`DateFlex`) - a cheapest-across-nearby-dates grid.

---

## 6. Verify - `POST /flights/verify`

**Request:** `{ "offerId": "<from search>" }`.
**Response** `{ "data": [ { journey, changes } ] }`:
- **journey:** `journeyKey`, **`expiration`** (guaranteed-price deadline), `timestamp`, `provider{code, logo}`, `pricing`, `baggage`, `fare`, `passengers{adults,children,infants}`, `segments[]`, `segmentFares[]`, `terms`.
- **changes** (present when the offer drifted, else null): `cabinChanged`, `fareChanged`, `priceChanged` (bools), `messages[]`, `oldCabin/newCabin`, `oldFareClass/newFareClass`, `oldFareFamily/newFareFamily`, `pricing{old, new}`.

Use `changes` to detect price/cabin/fare movement before charging. **Status:** 200 / 400 / 401 / **404 (offer expired)** / 502 / 503.

Source: `reference/post_flights-verify`.

---

## 7. Prebook - `POST /flights/prebooks` (creates the Stripe PaymentIntent)

**Request body:**

| Field | Type | Req | Notes |
|---|---|---|---|
| `offerId` | string | Yes | from search/verify |
| `usePaymentSdk` | boolean | No (default **true**) | true → creates Stripe PaymentIntent; false → credit-line/bypass path |
| `includeCreditBalance` | boolean | No | include credit-line info |
| `payment.descriptorSuffix` | string | No | Stripe statement-descriptor suffix |
| `voucherCode` | string | No | discount voucher |
| `contact` | object | Yes | `firstName`, `lastName`, `email`, `phoneCountryCode` (no `+`, e.g. `"33"`), `phoneNumber`; optional `middleName` |
| `passengers[]` | array | Yes | length must match search counts |

`passengers[]`: `firstName`, `lastName`, `birthday` (YYYY-MM-DD), `gender` (`M`|`F`), `documentType` (`passport`|`id_card`|…), `documentNumber`, `documentIssueCountry` (ISO), `documentExpiry` (YYYY-MM-DD), `nationality` (ISO), **`passengerType` (integer: 0=adult, 1=child, 2=infant)**, optional `middleName`, optional `loyaltyPrograms[]{airlineCode, programNumber}` (**Sabre/Travelport only**).

**Response** `{ "data": [ { ... } ] }` - key fields:
- **`prebookId`** (UUID).
- **`price`**, **`currency`**.
- **`transactionId`** (Stripe transaction id, when `usePaymentSdk: true`).
- **`secretKey`** (Stripe PaymentIntent client secret, e.g. `pi_..._secret_...`).
- `publishableKey` (string|null), `offerId`, `voucherCode`, `voucherTotalAmount`.
- **`paymentTypes[]`:** `TRANSACTION_ID`, `CREDIT`, `ACC_CREDIT_CARD`, `WALLET`.
- **`servicesAttachable`:** `{ expiresAt, groups[] }`; group = `{ available, category ("seat"|"baggage"), label, services[] }`; service = `{ serviceId, name, category, passengerType, segmentKey, pricing{display{amount,currency},converted}, metadata }`.
  - `metadata.seat = { available, position (window|middle|aisle), seatColumn, seatNumber, seatRow, seatType (standard|extra_legroom|exit_row) }`.
  - `metadata.baggage = { bagType (checked|cabin), pieces, weightKg }`.
- **`booking`** (provisional): `bookingId`, `bookingRef`, `status` (`PENDING_CONFIRMATION`), `paymentStatus` (`pending`), `journey{}`, `passengers[]`, `contact`, `order.reference{orderId, airlineBookings[]{airlineCode, airlineName, airlinePnr}}`.

**Example request:**
```json
{
  "offerId": "h6NwaWTZJDAx...",
  "usePaymentSdk": true,
  "payment": { "descriptorSuffix": "FLIGHT" },
  "contact": { "email": "j.doe@example.com", "firstName": "John", "lastName": "Doe", "phoneCountryCode": "33", "phoneNumber": "670-355-3640" },
  "passengers": [
    { "birthday": "1996-04-28", "documentExpiry": "2030-04-28", "documentIssueCountry": "US",
      "documentNumber": "123456789", "documentType": "passport", "firstName": "John",
      "gender": "M", "lastName": "Doe", "nationality": "US", "passengerType": 0 }
  ]
}
```

**Example response (load-bearing fields):**
```json
{ "data": [ {
  "prebookId": "019d0674-834d-7db7-9c8b-93fe8e46e7b8",
  "price": 740.27, "currency": "USD",
  "transactionId": "tr_cts_WaTMwICRB0h_dOfyvLvvN",
  "secretKey": "pi_3TChNEA4FXPoRk9Y1hbH6uVq_secret_CRhCAan4jSlXcwLs8N3r1qN6D",
  "publishableKey": null,
  "paymentTypes": ["TRANSACTION_ID","CREDIT","WALLET"],
  "servicesAttachable": { "expiresAt": "2026-03-19T14:31:42.493Z", "groups": [ /* seat + baggage */ ] },
  "booking": { "bookingId": "BK123456", "bookingRef": "FH-260319-ABC123", "status": "PENDING_CONFIRMATION", "paymentStatus": "pending", "order": { "reference": { "orderId": "OQTYXX", "airlineBookings": [ { "airlineCode": "DE", "airlinePnr": "DCDE" } ] } } }
} ] }
```

Source: `reference/post_flights-prebooks`.

---

## 8. Attach services (seats/bags) - `POST /flights/prebooks/{prebookId}/services`

**Path:** `prebookId`. **Body:** `selectedServices[]` (each `{ passengerIndex (0-based, matches passengers[]), serviceId (from servicesAttachable), quantity }`), optional `voucherCode`.

**Response:** same envelope as prebook, but **returns a NEW `transactionId` and NEW `secretKey`** reflecting the new total (old ones superseded). Also `price` (new total), `sellingPriceToUser` (price minus voucher), `voucherTotalAmount`, refreshed `servicesAttachable`, and `booking.selectedServices[]`.

> **Migration-critical rule (the #1 bug risk):** after attach-services you MUST use the **most recent** `transactionId`/`secretKey`. "The API does not validate which one you send - using a stale one causes a payment mismatch." Version them (v1 = prebook, v2 = services) and always confirm Stripe + book with the latest. Store server-side; never trust a client-supplied value.

Source: `reference/post_flights-prebooks-prebookid-services`.

---

## 9. Payment linkage (flight-side; full Stripe detail in `PAYMENTS.md`)

- `usePaymentSdk: true` → Nuitee creates a **Stripe PaymentIntent server-side**, returns `transactionId` (keep server-side) + `secretKey` (client secret for the frontend) [+ `publishableKey`, often null].
- **"The API does not charge the card - Stripe does."** The frontend confirms the PaymentIntent with the Stripe SDK using `secretKey`. Observe `paymentIntent.status === 'succeeded'` **before** calling `/flights/bookings`.
- **Book consumes payment** via `payment.method: "TRANSACTION_ID"` + `payment.transactionId: <latest>`. Alternative `payment.method: "CREDIT"` (credit line, no transactionId; needs an enabled credit line). ⚠️ **Do NOT trust `paymentTypes[]` from prebook** - it advertises `ACC_CREDIT_CARD`/`WALLET`, but the book endpoint rejects both for flights (`45009`). The real accepted set is `CREDIT | TRANSACTION_ID | THIRD_PARTY` (verified 2026-07-17, see §10).

---

## 10. Book / finalize - `POST /flights/bookings` (the resolved endpoint)

> `post_flights-book.md` = 404. The real path is `POST /flights/bookings` on `api.liteapi.travel/v3.0`.

**Request body:**
- `prebookId` (required - the UUID from prebook).
- `payment` (required): `method` (**`CREDIT` | `TRANSACTION_ID` | `THIRD_PARTY`**), `transactionId` (**required when method = TRANSACTION_ID** - the latest Stripe id).

> **VERIFIED 2026-07-17 (live sandbox).** The flight book endpoint accepts **only** `CREDIT | TRANSACTION_ID | THIRD_PARTY`. It **rejects `ACC_CREDIT_CARD` and `WALLET`** with `45009 "payment method must be CREDIT, TRANSACTION_ID, or THIRD_PARTY"` - even though the prebook response's own `paymentTypes[]` advertises `ACC_CREDIT_CARD`/`WALLET` (see §7). **This differs from hotels**, which DO accept `ACC_CREDIT_CARD`. Do not assume the two products share a payment enum.
> Also verified: in **sandbox**, `TRANSACTION_ID` settles automatically with no Stripe confirm (`paymentStatus: succeeded -> completed`) and returns a real airline PNR. Production will still need the Stripe step. `CREDIT` returns `45009 "no credit line available for this account"` unless a credit line is configured.

**Examples:**
```json
{ "prebookId": "019d0674-834d-7db7-9c8b-93fe8e46e7b8",
  "payment": { "method": "TRANSACTION_ID", "transactionId": "tr_cts_t0LaZePPxdCM_Kyskafml" } }
```
```json
{ "prebookId": "019d0674-834d-7db7-9c8b-93fe8e46e7b8", "payment": { "method": "CREDIT" } }
```

**Response (HTTP 201)** `{ "data": [ { booking, message } ] }`. `booking`:
- `bookingId`, **`bookingRef`** (Nuitee ref `FH-YYM-XXXXXXXX`), `status` (`PENDING_CONFIRMATION`|`CONFIRMED`|`CANCELLED`|`PENDING`|`TICKETED`), `timestamp`, `offerId`, `paymentStatus` (`pending`|`completed`|`failed`|`not_required`), `providerEnvironment`, `ticketLimitTime`, `lastRefreshedAt`.
- `journey{journeyKey, segments[], price{base,taxes,fees,total,currency}, terms{changeable,refundable,changeFee,refundFee}}`.
- `passengers[]{type (ADT|CHD|INF), firstName, lastName, dateOfBirth, gender, nationality, documentType, documentNumber, documentExpiry}`.
- `contact{firstName, lastName, email, phoneCountryCode, phoneNumber}`.
- **`order.reference`**: `{orderId, provider{code, pnr}, airlineBookings[]{airlineCode, airlineName, airlinePnr, pnr}}`; plus `order.currency/status/price/ticketLimitTime/timestamp`.
- **`airlineLocators[]{airlineCode, airlinePnr}`** (convenience copy of the airline PNR).
- `pricing{subtotal, servicesAmount, seatsAmount, baggageAmount, totalAmount, currency}`.
- `payment{amount, currency}`, `paymentInput{method, transactionId}`.
- `bookedServices[]{serviceId, name, category (seat|baggage), passengerIndex, segmentKey, quantity, pricing{display{amount,currency}}}`.
- `ticketData{confirmationId, ticketedAt, provider}`.

> The real **airline PNR** the passenger uses at the airport is `order.reference.airlineBookings[].airlinePnr` (and `airlineLocators[]`), distinct from Nuitee's `bookingRef`.

**Idempotency:** idempotent on `prebookId` - retrying returns the existing booking, no double-charge.

Source: `reference/post_flights-bookings`.

---

## 11. Retrieve booking - `GET /flights/bookings/{bookingId}`

- Path `bookingId`. Auth `X-API-Key`. Response = the same `booking{}` object (in `data[]`).
- **Poll status** after booking (`PENDING_CONFIRMATION → CONFIRMED → TICKETED`), using `ticketLimitTime` / `lastRefreshedAt` / `status`.
- **No cancellation endpoint documented** (`DELETE /flights/bookings/{id}` 404s). Post-booking flight management is retrieval-only in current docs - a gap to raise with Nuitee before go-live (see §14).

Source: `reference/get_flights-bookings-bookingid`.

---

## 12. Idempotency, errors, retries, streaming

- **Idempotency:** `/flights/bookings` idempotent on `prebookId`; safe to retry up to 3× with `prebookId` as the key. Enforce server-side.
- **Retryable (backoff):** 502, 503, 504. **Non-retryable:** 400, 401, 403, 404, 409.
  - **404 = offer/prebook expired → return user to SEARCHING.**
  - **409 = conflict → do NOT retry; check for an existing confirmed booking first.**
- **Per-endpoint:** `/verify` 502 → retry once after 1s; `/prebooks` 502 → do NOT auto-retry (check for existing prebook, then retry once), 503 → backoff 1s/2s/4s max 3; `/bookings` 502/503 → backoff up to 3×.
- **SSE:** search `Accept: text/event-stream` or `/flights/rates/stream` → `rates-chunk`/`rates-complete`/`error`/`message`. Matrix → `matrix-start`/`matrix-chunk`/`matrix-complete`.
- **Recommended session persistence** (architecture doc): a TTL-driven `booking_sessions(session_id, user_id, offer_id, prebook_id, transaction_id, verified_price, currency, status, expires_at, ...)` table (status enum matches §3), plus a permanent `bookings(...)` table with `provider_payload JSONB` + `stripe_payment_id`.

Source: `docs/flight-booking-architecture`, `docs/booking-flow-recipes`.

---

## 13. Reference data

**Airports - `GET /data/flights/airports`**
- Query: `q` (required, min 2 chars, e.g. `JFK` or `New York`).
- Response `{ "data": [ { airports[], count } ] }`; `airports[]` = `{ city, country, iata, icao, lat, lon, name, state, tz }`.

**Airlines - `GET /data/flights/airlines`**
- Query: `q` (optional), `limit` (default 20), `activeOnly` (default true), `alliance` (`star_alliance`|`oneworld`|`skyteam`|`vanilla_alliance`).
- Response `{ "data": [ { airlines[] } ] }`; `airlines[]` = `{ active, alliance, callsign, country, iata, icao, logo, name }` (`logo` is a relative path).

Source: `reference/get_data-flights-airports`, `reference/get_data-flights-airlines`.

---

## 14. Duffel → LiteAPI migration flags

| Concern | Duffel | LiteAPI | Action |
|---|---|---|---|
| Search | `POST /air/offer_requests` then read `offers` | `POST /flights/rates` returns `journeys[].offers[]` directly | collapse the two-call pattern |
| Offer id | `off_...` UUID | opaque msgpack/base64 blob | never parse `offerId` |
| Price lock | confirmed at order creation | explicit `POST /flights/verify` with a `changes{}` diff | wire verify + "price changed, confirm?" UX |
| Payment | `payments` on order create | prebook mints Stripe PaymentIntent (`transactionId`/`secretKey`); confirm Stripe, then book with `payment.method=TRANSACTION_ID` | see `PAYMENTS.md` |
| Ancillaries | `seat_maps` + `available_services` | `servicesAttachable.groups[]` in prebook; attach via `selectedServices[]` (regenerates the payment intent) | handle re-minted `transactionId` |
| PNR | `booking_reference` | TWO: `bookingRef` (`FH-YYM-…`) + real airline PNR at `order.reference.airlineBookings[].airlinePnr` / `airlineLocators[]` | persist both |
| Passenger type | string `adult`/`child`/`infant_without_seat` | integer `passengerType` 0/1/2 in request; string `ADT`/`CHD`/`INF` in response | map both directions |
| Post-book events | order webhooks | no flight webhooks documented | poll `GET /flights/bookings/{id}` via `status`/`ticketLimitTime` |
| Cancellation | `order_cancellations` | **no documented flight cancel/refund endpoint** | raise with Nuitee before go-live |
| Currency/FX | - | `pricing.converted` (bool) + `display` block | respect `converted` in UI |

---

## 15. WhereTo integration notes (what to build)

Today flights go through `activeFlightProvider` in [src/providers/flights/index.ts](src/providers/flights/index.ts) → `DuffelProvider`. The migration keeps that seam:

1. **New provider `src/providers/flights/LiteApiFlightProvider.ts`** implementing `FlightProvider` ([types.ts](src/providers/flights/types.ts)). Swap the export in `index.ts` to flip Duffel → LiteAPI (mirrors how hotels swap in `src/providers/hotels/index.ts`).
2. **`FlightProvider` contract will need extension.** Current `searchFlights(params, maxResults)` returns `FlightItinerary[]` shaped for Duffel - notably `passengerIds: string[]` (a Duffel concept with no LiteAPI equivalent; drop or repurpose it). LiteAPI needs verify → prebook → (services) → book primitives, so the contract should grow booking methods like the hotel provider has (`quoteHotel`/`prebook`/`book`). Model it on `HotelProvider`.
3. **New edge function `supabase/functions/liteapi-flight-book`** (mirror `liteapi-book`) with actions `verify` / `prebook` / `attachServices` / `book`, calling **`api.liteapi.travel/v3.0`** (NOT `book.`). Reuse the existing `LITEAPI_KEY` secret. Keep the key server-side; the search path can also route through an edge function.
4. **Search can stay client-side initially** (like current hotel search) but the money path (prebook/book) must be server-side from day one, same as `liteapi-book`.
5. **Persistence:** flights currently write the Duffel-shaped `orders` table via `duffel-checkout`. Decide whether LiteAPI flights reuse `orders` (with a `provider` discriminator) or get a `flight_orders` table like hotels got `hotel_orders`. The architecture doc's suggested `booking_sessions` + `bookings` split is a good reference for the TTL-driven session state.
6. **Payment:** identical Model A (Nuitee = merchant of record via Stripe) as hotels - see `PAYMENTS.md`. The re-minted `transactionId` after attach-services is the sharp edge to get right.
7. **Prereq:** request flight API access (sandbox) in the dashboard now - it is off by default and gates all testing.

---

_Cross-refs: `PAYMENTS.md` (Stripe SDK, merchant of record), `PLATFORM.md` (auth, rate limits, webhooks), `HOTELS.md` (parallel hotel flow), `README.md` (transition plan)._
