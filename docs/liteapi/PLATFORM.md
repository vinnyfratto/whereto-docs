# LiteAPI (Nuitee Connect) - Platform Reference

**Scope:** environments, base URLs, authentication, API keys, the full endpoint map, webhooks, rate limits, the Node SDK, the MCP server, and security/compliance.
**Audience:** engineers wiring LiteAPI into WhereTo's Supabase edge functions.
**Last researched:** 2026-07-14 against the live docs at https://docs.liteapi.travel .

> Branding note: the docs now call the platform **"Nuitee Connect,"** but every hostname is still `*.liteapi.travel`. Nuitee is the company; LiteAPI is the API. In WhereTo code and secrets we keep the `liteapi` / `LITEAPI_KEY` naming.

---

## 1. Environments & base URLs

| Category | Base URL | Notes |
|---|---|---|
| Hotel data (static content) | `https://api.liteapi.travel/v3.0` | `/data/*` endpoints |
| Search (rates / availability) | `https://api.liteapi.travel/v3.0` | `POST /hotels/rates` |
| Loyalty & guests | `https://api.liteapi.travel/v3.0` | `/guests`, `/loyalties` |
| **Booking (money path)** | `https://book.liteapi.travel/v3.0` | `/rates/prebook`, `/rates/book`, `/rates/rebook`, `/prebooks/{id}`, `/bookings` |
| MCP server (AI agents) | `https://mcp.liteapi.travel/api/mcp?apiKey=…` | dev convenience only |

- **Version segment is `v3.0` on both hosts.** Confirmed verbatim from each OpenAPI `servers` block.
- **The split-host architecture is real and required.** All read/data/search/loyalty calls go to `api.liteapi.travel`. All reservation/money calls go to `book.liteapi.travel`. Do not send booking calls to the `api.` host. WhereTo already follows this: `liteapi-stays`/search use `api.`, `liteapi-book` prebook/book use `book.`.
- **Sandbox vs production is NOT a different host.** Both environments use the **same base URLs**; the environment is chosen entirely by **which API key you send**. Confirmed three ways: (a) the access-control page states each API key "uniquely identifies a customer and environment"; (b) the Node cookbook just swaps `SAND_API_KEY`/`PROD_API_KEY` against the same endpoints; (c) webhook payloads carry a boolean `sandbox` field. So "switching environments" == swapping the key value, nothing else.
- **Dashboard host** is referred to only as the "Nuitee Connect platform / developer dashboard." No exact dashboard hostname is printed in the docs, so do not hardcode a guess - use the login you already have (`https://dashboard.liteapi.travel` per our `.env.example`, unconfirmed by docs).

> Doc quirk: the HMAC page's raw HTTP example shows `Host: api.liteapi.com` and `/v1/...`. That is stale (`.com`, `v1`). The authoritative host/version is `api.liteapi.travel/v3.0` per every OpenAPI spec and other page.

Sources: `reference/getting-started-1`, `reference/authentication`, `reference/openapi-specifications`, `reference/mcp-server`.

---

## 2. API keys

- **Where:** create a Nuitee Connect account, then **Developer page (sidebar) → API Keys tab**.
- **Formats / prefixes (verbatim doc examples):**
  - Sandbox: `sand_` + UUID, e.g. `sand_c0155ab8-c683-4f26-8f94-b5e92c5797b9`.
  - Production: `prod_` + token, e.g. `prod_9xxxxxxxxxx`.
- **Sandbox is free, no credit card.** To get a **production** key you must **attach a credit card** and **set up a payout method** (to collect earnings).
- **Rotation:** the platform supports key rotation without downtime, immediate revocation if compromised, and multiple keys for different apps. Rotate periodically and after any suspected incident.
- **Server-side only:** docs are explicit - never expose the key in client code; use env vars / secret managers. For WhereTo this means the key lives **only in Supabase edge-function secrets** (`LITEAPI_KEY`). See §11 for our current client-key exposure gap.

Our code already keys environment detection off the prefix:
```ts
// supabase/functions/liteapi-book/index.ts
const LIVE_MODE = !!LITEAPI_KEY && !LITEAPI_KEY.startsWith('sand');
```

Sources: `docs/getting-a-sandbox-key`, `reference/authentication`, `docs/authentication-access-control`.

---

## 3. Authentication (simple - the default)

- **Header:** `X-API-Key: <key>` on every request. This is the primary auth for **all** endpoints including booking.
```shell
curl -X GET "https://api.liteapi.travel/v3.0/data/hotels?countryCode=IT&cityName=Rome" \
  -H "X-API-Key: YOUR_API_KEY"
```
Source: `reference/authentication`.

---

## 4. HMAC / secure authorization (optional hardening)

- **Purpose:** signature auth so the private key "never leaves your system."
- **Algorithm:** **HMAC SHA-512** (note: 512, not 256).
- **Signed string:** `${privateApiKey}${publicApiKey}${timestamp}` where `timestamp` is **UNIX epoch seconds**. Output hex.
- **Header:** `Authorization: PublicKey=<pub>,Signature=<sig>,Timestamp=<ts>`.
- **Required for booking?** No. The docs present HMAC as an enhanced option; simple `X-API-Key` remains the documented default for all endpoints. Treat HMAC as optional unless LiteAPI tells our account otherwise.

> Two bugs in the doc's own snippet if you copy it: it interpolates an undefined `publicKey` (declared `publicApiKey`), and it uses **CryptoJS** syntax (`crypto.SHA512(...).toString(crypto.enc.Hex)`), not Node's `crypto`. In a Supabase edge function (Deno) implement it with Web Crypto `subtle.sign('HMAC', …)` and SHA-512, not that literal snippet.

Source: `reference/secure-authorization`.

---

## 5. Endpoint map

All data/search/loyalty on `api.liteapi.travel/v3.0`; all booking on `book.liteapi.travel/v3.0`. Version `v3.0` everywhere. See `FLIGHTS.md`, `HOTELS.md`, `COMMERCIALS.md` for full request/response schemas per endpoint.

**Hotel Data API** - `api.liteapi.travel/v3.0`
- `GET /data/hotels` - list hotels (by city/country/coords)
- `GET /data/hotel` - one hotel's detail
- `GET /data/reviews` - reviews (+ AI sentiment)
- `GET /data/cities` · `GET /data/countries` · `GET /data/currencies`
- `GET /data/iataCodes` · `GET /data/facilities` · `GET /data/hotelTypes` · `GET /data/chains`
- `GET /data/places` · `GET /data/places/{placeId}` (geosearch)

**Search API** - `api.liteapi.travel/v3.0`
- `POST /hotels/rates` - full room rates for a list of hotel IDs
- `POST /hotels/min-rates` - cheapest rate per hotel (lighter)

**Booking API** - `book.liteapi.travel/v3.0`
- `POST /rates/prebook` - open checkout session, returns `prebookId` + `transactionId` + `secretKey`
- `GET /prebooks/{prebookId}` - retrieve a prebook
- `POST /rates/book` - confirm booking with the transaction
- `POST /rates/rebook` - hard amendment (rebook)
- `GET /bookings` - list · `GET /bookings/{bookingId}` - detail · `PUT /bookings/{bookingId}` - cancel
- `PUT /bookings/{bookingId}/amend` - amend guest name · `POST /bookings/{bookingId}/alternative-prebooks` - amend dates/occupancy

**Loyalty API** - `api.liteapi.travel/v3.0`
- `GET /guests` · `GET /guests/{guestId}` · `GET /guests/{guestId}/bookings` · `GET /guests/{guestId}/vouchers`
- `GET /loyalties` · `PUT /loyalties` · `POST /loyalties`
- `GET /guests/{guestId}/loyalty/points` · `POST /guests/{guestId}/loyalty/points/redeem`

**Flights API** - see `FLIGHTS.md` for the confirmed host and full endpoint list (`/flights-rates`, `/flights/verify`, `/flights-prebooks`, `/flights-prebooks/{id}/services`, plus `/data/flights/*` reference data).

**Other API groups** (machine-readable specs at `https://docs.liteapi.travel/openapi/`): `api-search.json`, `api-booking.json`, `api-hotel-data.json`, `api-loyalty.json`, `api-vouchers.json`, `api-analytics.json`, `api-price-index.json`. So there are also **Vouchers**, **Analytics**, and **Price Index** APIs (covered in `COMMERCIALS.md`).

Sources: `reference/api-endpoints-overview`, `reference/openapi-specifications`.

---

## 6. Webhooks

Register at **dashboard → Developer tools → Webhooks**.

**Config fields:** Webhook URL (must be HTTPS + publicly reachable), Description, **Authentication Token** (optional shared secret), Retry config (max retries + initial wait seconds, exponential backoff), event subscriptions, optional custom headers.

**Event names (exact):**

Hotel lifecycle: `booking.prebook`, `booking.book`, `booking.cancel`, `booking.book.hotelConfirmationNumber`, `booking.checkinInstruction`
Hotel errors: `booking.prebook_error`, `booking.book_error`, `booking.cancel_error`
Amendment/mgmt: `booking.rebook.rfn`, `booking.rebook.nrfn`, `booking.amendment`, `booking.amendment.relocation`, `booking.refund`, `booking.compensation`
Flights (where enabled): `flight.prebook`, `flight.attachServices`, `flight.book.created`, `flight.book.pending.confirmation`, `flight.book.confirmed`, `flight.book.cancelled`, `flight.book.failed`, `flight.book.expired`

**Payload - 5 top-level fields:** `event_id`, `event_name`, `request`, `response`, `sandbox`.
```json
{
  "event_id": "…",
  "event_name": "booking.prebook",
  "request": "{ \"offerId\": \"…\", \"clientReference\": \"…\" }",
  "response": "{ \"prebookId\": \"…\", \"hotelId\": \"…\", \"roomTypes\": [ … ] }",
  "sandbox": false
}
```

**Critical handling rules for our webhook edge function:**
1. `request` and `response` are **stringified JSON** (escaped strings), not nested objects. `JSON.parse()` them.
2. **Verification:** there is no HMAC/signature scheme for inbound webhooks. Verification is the optional **Authentication Token**, sent as the `authorization` header value. Check `req.headers.authorization === OUR_SHARED_SECRET`.
3. **Dedupe on `event_id`** (at-least-once delivery + exponential-backoff retries). Store processed IDs.
4. Honor the `sandbox` boolean to route/label test events.

Source: `docs/using-liteapi-webhooks`.

> WhereTo status: we have **no LiteAPI webhook function yet.** `liteapi-book` persists the booking synchronously after `POST /rates/book` returns, so it works without webhooks. A webhook function becomes valuable for async flight ticketing (`flight.book.pending.confirmation` → `flight.book.confirmed`), supplier-side cancellations, and check-in instructions.

---

## 7. Rate limiting & performance

- **Limit: 500 requests/second per customer.** Only documented limit; same 500 RPS on the MCP server. No separate sandbox figure, no documented concurrency cap.
- **Over-limit:** `429 Too Many Requests`. Use **exponential backoff** (our `liteapi-stays` already retries once on 429 after 1.2s).
- **Their SLOs:** avg response < 3s for rate search and prebook; 5xx ratio < 0.05%.
- **Not documented:** rate-limit response headers (no named `Retry-After`/`X-RateLimit-*`), client timeout recommendations, or cache TTLs. Cache `/data/*` static content on our side regardless.
- Higher limits available on request via Nuitee support.

Source: `docs/performance-reliability-rate-limiting`.

---

## 8. Node SDK & cookbook

- The cookbook centers on a **clonable Express example** (git clone → `npm install`), not a one-line SDK install. It uses a `.env` with `PROD_API_KEY` / `SAND_API_KEY`.
- **SDK methods shown:** `sdk.getHotels(countryCode, city)`, `sdk.getFullRates(checkin, checkout, currency, country, hotelIds, adults)`, plus `preBook` / `book`.
- **Sandbox test card:** `4242 4242 4242 4242`, any future expiry, any CVC.
- **Package name caveat:** the page did not print an explicit `npm install <package>`. The commonly published package is `liteapi-node-sdk`, but confirm on npm before depending on it. It is a **Node (`require`) SDK**; Supabase edge functions run **Deno**, so we call the REST endpoints directly with `X-API-Key` (as we already do) rather than importing the Node SDK.

Source: `docs/nodejs-cookbook`.

---

## 9. MCP server

- Endpoint: `https://mcp.liteapi.travel/api/mcp?apiKey=YOUR_API_KEY` (key as query param).
```json
{ "mcpServers": { "liteapi": { "url": "https://mcp.liteapi.travel/api/mcp?apiKey=YOUR_API_KEY" } } }
```
- Tools: hotel search, price & availability, place search, hotel details, prebooking, booking. Same 500 RPS. Dev/agent convenience - not for the production booking path.

Source: `reference/mcp-server`.

---

## 10. Security & compliance (summary)

- **PCI DSS:** in scope of their program, but the **level is not published** (ask support). Upside: LiteAPI can be merchant of record / capture cards on their side, so WhereTo can avoid touching raw card data. See `PAYMENTS.md`.
- **GDPR roles:** **we are the Data Controller; Nuitee is the Data Processor.** They don't use data for ads/profiling/resale.
- **DPA:** exists, available on request.
- **Data residency:** may process outside our country including in the EU, under Standard Contractual Clauses.
- **Retention:** kept only as long as necessary, then deleted/anonymized.
- **Encryption:** described generically (strong, secrets not logged); specific algorithms not published.
- **What it means for us:** we're the controller for guest PII in `hotel_orders` / booking tables - normal controller duties (lawful basis, retention, deletion on request). Supabase Postgres encrypts at rest; keep our RLS posture. No LiteAPI restriction forbids persisting booking records; just keep the key server-side and verify/dedupe webhooks.

Sources: `docs/security-privacy-compliance-overview`, `docs/authentication-access-control`, `docs/data-protection-privacy`.

---

## 11. What this means for WhereTo (platform layer)

- **Base URLs and header are already correct** in `liteapi-stays` and `liteapi-book` (`api.` for search, `book.` for booking, `X-API-Key`). No change needed for the hotel platform layer.
- **Flights will reuse the same key and, per `FLIGHTS.md`, the same hosting pattern.** One `LITEAPI_KEY` secret serves flights + hotels + loyalty.
- **Open gaps (platform):**
  1. Search still uses the client key `EXPO_PUBLIC_LITEAPI_KEY` in `LiteApiProvider.ts`. The server proxy `liteapi-stays` exists; route all search through it and drop the client key to remove key exposure.
  2. No webhook function yet - add one before flights go live (async ticketing) and to catch supplier-side cancellations. Verify via the shared `authorization` token, `JSON.parse` the `request`/`response` fields, dedupe on `event_id`.
  3. Consider HMAC auth for the booking function as hardening (optional).

---

_Cross-refs: `FLIGHTS.md`, `HOTELS.md`, `PAYMENTS.md`, `COMMERCIALS.md`, `README.md` (transition plan)._
