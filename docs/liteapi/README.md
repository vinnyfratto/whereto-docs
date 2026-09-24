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
| [SERVICING.md](SERVICING.md) | After booking: **who does what (Nuitée vs WhereTo)**, cancellations, schedule changes, webhooks as they really arrive, support desks, what the app does |

Related existing docs: [../HOTEL_BOOKING_REFERENCE.md](../HOTEL_BOOKING_REFERENCE.md) (app-level hotel flow), and the provider architecture ADR at `docs/decisions/0002-provider-abstraction.md`.

---

## 2. Where WhereTo stands today

*Updated 2026-09-17: the migration this folder was written for is done.*

| Product | Provider today | Server-side path | Status |
|---|---|---|---|
| **Hotels** | **LiteAPI** | `liteapi-stays` (search), `liteapi-book` (quote/prebook/book), `hotel_orders` table | Booking and traveller payment built end to end |
| **Flights** | **LiteAPI** | `liteapi-flights` (search/verify/prebook/book), `flight_orders` table | Booking and traveller payment built end to end |

What is proven against a production key, and what is still waiting on Nuitee, is tracked in [../GO_LIVE_LITEAPI.md](../GO_LIVE_LITEAPI.md) rather than here.

Provider seams (the swap points, both abstracted):
- Flights: `activeFlightProvider` in `src/providers/flights/index.ts` → `LiteApiFlightProvider`.
- Hotels: `src/providers/hotels/index.ts` → `LiteApiProvider`.

Duffel is gone: provider, edge functions, key and the `orders` table were all deleted on
2026-09-17. What remains open is commercial and payment polish, not a provider swap.

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

**Flights (new build, mirrors hotels) — all done, kept for the record**
- [x] `src/providers/flights/LiteApiFlightProvider.ts` implementing an extended `FlightProvider` (add verify/prebook/services/book like `HotelProvider`; drop the old provider's `passengerIds`).
- [x] Flight booking edge function hitting **`api.liteapi.travel/v3.0`**. It shipped as `liteapi-flights` rather than the `liteapi-flight-book` name planned here, covering search and the money path in one function.
- [x] Persistence: `flight_orders`, mirroring `hotel_orders`, rather than a `provider` discriminator on the old shared table.
- [x] Flip `activeFlightProvider` to LiteAPI; remove Duffel. Finished 2026-09-17 — the `duffel-*` functions, `DUFFEL_API_KEY` and the `orders` table are all deleted.

**Payments (shared, finish the deferred fork)**
- [ ] Native `@stripe/stripe-react-native` payment step: confirm PaymentIntent from prebook `secretKey`, then book with `TRANSACTION_ID`.
- [ ] **Fix the `TRANSACTION` → `TRANSACTION_ID` enum** in `hotels/types.ts` / `hotelBooking.ts` / `liteapi-book` after sandbox verification.
- [ ] Add Expo dev-client/prebuild config for Stripe native module.

**Platform / hygiene**
- [ ] Route hotel search through `liteapi-stays` and drop the client key `EXPO_PUBLIC_LITEAPI_KEY`.
- [x] Add a LiteAPI **webhook** edge function (verify via shared `authorization` token, `JSON.parse` the `request`/`response` fields, dedupe on `event_id`) - acting on booking events since 2026-09-16, see [SERVICING.md](SERVICING.md).
- [ ] `send-notification` `hotel_confirmation` branch; "My Bookings" `hotel_orders` union; wire hotel `cancel()`.

**Commercial config**
- [ ] Decide member-pricing strategy: per-request `margin` for logged-in (CUG) vs anonymous; whether to enable the cashback loyalty program + Guests mapping.
- [ ] Confirm with Nuitee: PCI level, flight cancellation support, production rate limits, DPA.

---

## 7. Notes

- All facts are cited to the source `.md` pages inside each doc. Items the docs left ambiguous are flagged "verify against sandbox" - treat those as open until confirmed on a live sandbox key.
- Version/`versionCode` were **not** bumped: this change is documentation only and ships nothing in the app bundle. The version bump belongs with the actual migration code.
