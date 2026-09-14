# LiteAPI (Nuitee Connect) Integration - WhereTo

**Decision (2026-07-14):** WhereTo is standardizing on **LiteAPI (Nuitee Connect) for BOTH flights and hotels**, and **retiring Duffel**. **Nuitee is the merchant of record**, capturing payment through its Stripe integration (Model A). This folder is the engineering reference for that integration, compiled from a deep read of the full LiteAPI documentation.

> Research-only deliverable. No app code was changed. Action items for the actual migration are consolidated in §6.

---

## 1. Documents in this folder

| Doc | Covers |
|---|---|
| [PLATFORM.md](PLATFORM.md) | Hosts/environments, auth (`X-API-Key` + HMAC), keys, endpoint map, webhooks, rate limits (500 RPS), SDK, security/compliance |
| [FLIGHTS.md](FLIGHTS.md) | Flight search/verify/prebook/services/book/retrieve, matrix, reference data, error/retry, **Duffel→LiteAPI migration map** |
| [HOTELS.md](HOTELS.md) | Hotel static data, rates (+ min-rates, streaming), rate JSON model, prebook/book/retrieve/list/cancel/amend/rebook |
| [PAYMENTS.md](PAYMENTS.md) | The four payment methods, User Payment (Stripe SDK), **native RN Stripe**, PCI, refunds/settlement, sandbox |
| [COMMERCIALS.md](COMMERCIALS.md) | API pricing/costs, commission/markup, dynamic pricing, perks/promotions, price index, **loyalty (member pricing)** |

Related existing docs: [../HOTEL_BOOKING_REFERENCE.md](../HOTEL_BOOKING_REFERENCE.md) (app-level hotel flow), and the provider architecture ADR at `docs/decisions/0002-provider-abstraction.md`.

---

## 2. Where WhereTo stands today

| Product | Provider today | Server-side path | Status |
|---|---|---|---|
| **Hotels** | **LiteAPI** already | `liteapi-stays` (search), `liteapi-book` (quote/prebook/book) | Built; booking key server-side; payment UI deferred |
| **Flights** | **Duffel** | `duffel-*` edge functions, `orders` table | To be migrated to LiteAPI |

Provider seams (the swap points, both already abstracted):
- Flights: `activeFlightProvider` in `src/providers/flights/index.ts` (currently `DuffelProvider`).
- Hotels: `src/providers/hotels/index.ts` → `LiteApiProvider`.

So this transition = **build a LiteAPI flight provider + flight booking edge function, then flip the flight seam and remove Duffel**, plus finish the deferred hotel payment UI. The hotel work already proves the pattern end to end.

---

## 3. Architecture at a glance

- **Two hosts, one key.** `api.liteapi.travel/v3.0` for data/search/loyalty **and all flight endpoints (including flight booking)**; `book.liteapi.travel/v3.0` for **hotel** booking (prebook/book/cancel/amend). One `LITEAPI_KEY` (Supabase secret) serves everything.
  - ⚠️ **Asymmetry:** hotel booking is on `book.`, but **flight booking is on `api.`** (`POST /flights/bookings`). Do not send flight booking to `book.`.
- **Environment = key prefix.** `sand_` = sandbox, `prod_` = production. Same hosts. Our `liteapi-book` already detects this (`!key.startsWith('sand')`).
- **Auth:** `X-API-Key` header. HMAC (SHA-512) available as optional hardening.
- **Money path is server-side only.** The LiteAPI key never ships in the client; the app only ever holds a Stripe **client secret** during payment.
- **Payment (Model A):** prebook (`usePaymentSdk:true`) mints a Stripe PaymentIntent in **Nuitee's** account → app confirms it with native `@stripe/stripe-react-native` → book with `payment:{ method:"TRANSACTION_ID", transactionId }`.

---

## 4. Booking flow (both products are the same shape)

```
HOTELS:  /hotels/rates ──► /rates/prebook ──► [Stripe pay] ──► /rates/book ──► /bookings/{id}
                          (book.liteapi.travel)                (book.liteapi.travel)

FLIGHTS: /flights/rates ─► /flights/verify ─► /flights/prebooks ─► [+services] ─► [Stripe pay] ─► /flights/bookings ─► /flights/bookings/{id}
                          (all on api.liteapi.travel)
```
Both: prebook returns `transactionId` + `secretKey`; charge via Stripe; finalize with `TRANSACTION_ID`. Idempotency via `clientReference` (hotels) / `prebookId` (flights). Full field-level detail in the per-product docs.

---

## 5. Consolidated critical findings

1. **`payment.method` = `TRANSACTION_ID`, not `TRANSACTION`.** Our hotel code and old notes use `TRANSACTION`. Latent bug - verify against sandbox and align. ([PAYMENTS.md](PAYMENTS.md) §11)
2. **Flights need explicit access enablement** (off by default, even sandbox) - request in the dashboard; sandbox integration required before production. Lead-time item. ([FLIGHTS.md](FLIGHTS.md) §2)
3. **Flight booking is on `api.liteapi.travel`, not `book.`**; the endpoint is `POST /flights/bookings` (not `/flights/book`). ([FLIGHTS.md](FLIGHTS.md) §10)
4. **Model A does not need a WebView** - native `@stripe/stripe-react-native` works (needs an Expo dev client/prebuild, not Expo Go). ([PAYMENTS.md](PAYMENTS.md) §4)
5. **No documented flight cancellation/refund endpoint** (hotels have one). Raise with Nuitee before go-live if we support flight cancellations. ([FLIGHTS.md](FLIGHTS.md) §11)
6. **Flight attach-services re-mints `transactionId`/`secretKey`** - always use the latest, or payment mismatches. The #1 flight migration bug risk. ([FLIGHTS.md](FLIGHTS.md) §8)
7. **Member pricing is legitimate via Closed User Groups + per-request `margin`** - a logged-in app is a valid non-public channel, so authenticated users can see below-SSP prices. Plus cashback points/vouchers via the Guests API. ([COMMERCIALS.md](COMMERCIALS.md) §2, §6–7)
8. **Core API is free** (Rates→Prebook→Book); charges apply only to price-index ($0.05) and places ($0.01) calls and some dashboard add-ons. Nuitee earns on room margin. ([COMMERCIALS.md](COMMERCIALS.md) §1)
9. **Two `GET /bookings` shapes** and mixed camel/snake casing + a misspelled `voucherTransationId` - parse defensively. ([HOTELS.md](HOTELS.md) §11, §16)

---

## 6. Migration action items (for when we build)

**Prerequisites / lead time**
- [ ] Request **flight API access** (sandbox first) in the LiteAPI dashboard - gates all flight testing.
- [ ] Confirm `LITEAPI_KEY` secret is set and `liteapi-book` deployed; run the `hotel_orders` migration.

**Flights (new build, mirrors hotels)**
- [ ] `src/providers/flights/LiteApiFlightProvider.ts` implementing an extended `FlightProvider` (add verify/prebook/services/book like `HotelProvider`; drop Duffel-only `passengerIds`).
- [ ] `supabase/functions/liteapi-flight-book` (actions verify/prebook/attachServices/book) hitting **`api.liteapi.travel/v3.0`**.
- [ ] Persistence: reuse `orders` with a `provider` discriminator, or add `flight_orders` (mirror `hotel_orders`). Consider the `booking_sessions` TTL table from the flight architecture doc.
- [ ] Flip `activeFlightProvider` to LiteAPI; remove Duffel (`duffel-*` functions, `DUFFEL_API_KEY`, `.env.example` note) once verified.

**Payments (shared, finish the deferred fork)**
- [ ] Native `@stripe/stripe-react-native` payment step: confirm PaymentIntent from prebook `secretKey`, then book with `TRANSACTION_ID`.
- [ ] **Fix the `TRANSACTION` → `TRANSACTION_ID` enum** in `hotels/types.ts` / `hotelBooking.ts` / `liteapi-book` after sandbox verification.
- [ ] Add Expo dev-client/prebuild config for Stripe native module.

**Platform / hygiene**
- [ ] Route hotel search through `liteapi-stays` and drop the client key `EXPO_PUBLIC_LITEAPI_KEY`.
- [ ] Add a LiteAPI **webhook** edge function (verify via shared `authorization` token, `JSON.parse` the `request`/`response` fields, dedupe on `event_id`) - needed for async flight ticketing and supplier-side cancellations.
- [ ] `send-notification` `hotel_confirmation` branch; "My Bookings" `hotel_orders` union; wire hotel `cancel()`.

**Commercial config**
- [ ] Decide member-pricing strategy: per-request `margin` for logged-in (CUG) vs anonymous; whether to enable the cashback loyalty program + Guests mapping.
- [ ] Confirm with Nuitee: PCI level, flight cancellation support, production rate limits, DPA.

---

## 7. Notes

- All facts are cited to the source `.md` pages inside each doc. Items the docs left ambiguous are flagged "verify against sandbox" - treat those as open until confirmed on a live sandbox key.
- Version/`versionCode` were **not** bumped: this change is documentation only and ships nothing in the app bundle. The version bump belongs with the actual migration code.
