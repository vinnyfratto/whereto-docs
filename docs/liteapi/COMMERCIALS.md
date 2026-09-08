# LiteAPI (Nuitee Connect) - Commercials Reference

**Scope:** how the API is priced, how WhereTo earns money (net/retail/commission/markup), Nuitee-managed dynamic pricing, perks & promotions, the price-index endpoints, and the loyalty program (the "better pricing for logged-in users" lever the whole transition is partly about).
**Last researched:** 2026-07-14.

> All numbers, percentages, and field names below are reproduced from the docs. Two facts cross-validated across independent pages (price-index = $0.05/request appears on both the pricing page and the endpoint page). Items the docs did not fully resolve are listed in §8.

---

## 1. API pricing & usage costs

**Cost model:** LiteAPI is "an open API and free to use." The **core booking workflow (Rates → Prebook → Book) is free** when used per the Terms of Service. Nuitee makes money on the **room margin/commission** (see §2), not on core API calls. A few premium data endpoints and dashboard add-ons carry explicit charges.

**Paid per-request endpoints:**

| Service | Cost |
|---|---|
| Price index endpoints (`/prices/city`, `/prices/hotels`) | **$0.05 / request** |
| Places endpoints (`/data/places`) | **$0.01 / request** |

**Dashboard add-ons (monthly):**

| Add-on | Cost |
|---|---|
| Admin seat | $4.99 / seat / month |
| Agent seat | $1.99 / seat / month |
| AI Insights | $4.99 / month |
| Advanced Logs | $4.99 / month |
| Reports & Analytics | $4.99 / month |

**Free:** everything else - the full booking flow and all hotel content (`/data/*`) endpoints.

**Not documented:** no monthly minimum, no per-search volume tiers, no sandbox call caps were surfaced. The only rate limits found are the global 500 RPS (see `PLATFORM.md`) and the price-index ~10 req/min limit (§5).

Source: `reference/api-pricing-usage-costs`.

> WhereTo impact: search, rates, prebook, and book are all free. Budget only for `/data/places` autocomplete ($0.01/call - cache aggressively or debounce) and price-index calls ($0.05/call - call sparingly, cache per city/hotel/day).

---

## 2. Revenue management & commission - how WhereTo earns

You buy at a **net rate** and sell at a **retail rate**; the difference is your commission/margin.

**Terms:**
- **Net rate** - wholesale price you pay. Achieved by setting markup to `0` (there is no separate net-rate field; `priceType` stays `"commission"` even at 0).
- **Retail rate** - what the customer pays (net + your markup). Lives in `retailRate.total` on each rate.
- **Suggested Selling Price (SSP)** - the hotel's recommended **minimum public** selling price, in `suggestedSellingPrice` (with a `source`, e.g. `booking.com`). Selling **at or above SSP** on public channels is always allowed.
- **Markup / margin** - your commission, a percentage (decimals allowed).

**Setting your markup - three ways:**
1. **Dashboard default** - a default markup % in the dashboard commission section. `0` = net.
2. **Per-request `margin`** - **overrides** the dashboard default for that request. `{ "margin": 15 }`.
3. **Per-request `additionalMarkup`** - **adds to** the default (does not replace). Default 10% + `additionalMarkup: 15` = 25% total. A $100 net becomes $125.

**Worked example (net = $100, SSP = $115):**

| Setting | Sell price | Commission | Public-sellable? |
|---|---|---|---|
| `margin: 0` (net) | $100 | $0 | - |
| `margin: 5` | $105 | $5 | below SSP → CUG only |
| `margin: 15` | $115 | $15 | yes (= SSP) |

**Closed User Groups (CUG) - the member-pricing mechanism.** Selling **below SSP on a public channel** is a "rate violation." To legally sell below SSP you must use a **non-public channel**: a **Closed User Group** such as a login-restricted app, logged-in-user discounts, or bundled travel. **A logged-in WhereTo user is a legitimate CUG**, so authenticated users can see prices below SSP without a violation. This is the cleanest "members save X%" lever.

**Two merchant-of-record models:**
- **Nuitee as Merchant of Record** (our chosen Model A) - set commission via `margin`; Nuitee handles the payment; `margin` raises the customer's final price. See `PAYMENTS.md`.
- **You as Merchant of Record** - work at net (`margin: 0`) and add your own commission via service fees/bundling.

**Payouts:** a booking is "confirmed" when the guest **checks out**. **Commissions pay out weekly** after confirmation. No hidden fees; commission is the margin.

**Field caveat:** the exact response field for the earned commission amount is `commission[] { amount, currency }` on each rate (confirmed in the rates schema - see `HOTELS.md`). Effective commission ≈ `retailRate.total − net`.

Source: `docs/revenue-management-and-commission`.

---

## 3. Nuitee-managed (dynamic) pricing

An automated markup optimizer that uses the SSP as an anchor and adjusts markup between a floor and a ceiling.

**Mechanism (3 steps):**
1. **Start with SSP** (baseline reference).
2. **Apply a threshold** - a percentage adjustment to SSP: `-30%` (30% below SSP), `0%` (match SSP), `+15%` (15% above SSP).
3. **Clamp between floor and ceiling** - safety rails based on the **net rate** (not SSP): Floor = min markup above net (e.g. Net +5%), Ceiling = max markup above net (e.g. Net +20%).

Guiding principle (quoted): "The threshold determines the target selling price, while the floor and ceiling ensure that target stays within acceptable commercial limits." The doc's worked scenarios produce final prices of $120, $157.50, and $180 under ceiling-hit / neutral / floor conditions.

**Trade-off:** automatic per-property optimization (never below floor, never above ceiling) in exchange for giving up fixed per-booking margin predictability. The exact enable toggle and the threshold/floor/ceiling field names were not surfaced by the docs - verify in the dashboard before relying on it (§8).

Source: `docs/nuitee-managed-dynamic-pricing`.

---

## 4. Perks & promotions

Two mechanisms, both surfaced on the rate object.

**Promotions / discounts** - compare two fields:
- `retailRate.total` - final **discounted** price.
- `retailRate.initialPrice` - standard price (always ≥ total).

When `initialPrice > total`, a discount is active, described by a **`promotions[]`** array:

| Field | Purpose |
|---|---|
| `name` | promo title (e.g. "Early booker Deal") |
| `from` / `to` | date range (YYYY-MM-DD) |
| `discount` | numeric value |
| `discountType` | e.g. `percentage` |
| `promotionCode` | optional required code |
| `currency` | denomination |

At the offer level the equivalents are `offerInitialPrice` and `offerRetailRate`.

**Perks (in-stay benefits)** - breakfast, upgrades, credits, etc., in a **`perks[]`** array at `data.roomTypes[].rates[].perks`:
```json
{ "perkId": 10, "name": "Daily breakfast for 2", "amount": 40, "currency": "USD", "level": "HOTEL" }
```
- `level` scopes the perk: `HOTEL` (property-wide), `RATE` (specific rate), `ROOM` (specific room type).
- `guestLevel` on the rate/perk indicates the **loyalty tier** that unlocks it - a direct lever for giving members better perks. Perks appear across Search, Prebook, Book, and retrieval.

Source: `docs/perks-and-promotions`.

---

## 5. Price-index endpoints (for "is this a good deal?" UI)

Compare a live nightly quote against the historical average. **Both cost $0.05/request** and are **rate-limited (~10 req/min; 429 with `Retry-After` on the hotels variant).** Both on `api.liteapi.travel/v3.0`, header `X-API-Key`.

### 5a. `GET /prices/city`
Query: `countryCode` (req, ISO-2), `cityName` (req), `fromDate` (opt, default today), `toDate` (opt, default +1yr).
```json
{ "countryCode": "US", "cityName": "New York", "prices": [ { "day": "2026-01-18", "avgPriceUsd": 107 } ] }
```

### 5b. `GET /prices/hotels`
Query: `hotelIds` (req, comma-separated, **max 50**), `fromDate` (opt), `toDate` (opt).
```json
{ "data": [ { "hotelId": "lp19d9e", "prices": [ { "day": "2026-01-18", "avgPriceUsd": 107 } ] } ] }
```
`avgPriceUsd` = average nightly price in USD, rounded to nearest integer. Errors `{ "message", "code" }`; 429 adds a `Retry-After` header (seconds).

Sources: `reference/getpriceindexcity`, `reference/getpriceindexhotels`.

---

## 6. Loyalty - there are TWO separate systems

This is the most important thing to get right, and the two are easy to conflate:

1. **WhiteLabel Loyalty** - a feature of LiteAPI's **hosted white-label booking site**. It surfaces **your own external** loyalty program (your existing points/miles) inside the funnel. It only *displays* messaging and optionally *captures* a membership number for you to process later. It does **not** award or hold points inside LiteAPI.
2. **API-native Loyalty (Cashback + Guests + Points + Vouchers)** - a LiteAPI-**hosted** program you drive by API. Enable a cashback %, register **guests** (`guestId`), guests **earn points**, and points **redeem into vouchers** (**10 points = $1 USD**) that discount future bookings.

**For WhereTo (custom React Native flow, not the hosted site), system #2 is the relevant one**, combined with CUG + `margin` for member-only pricing.

### 6a. WhiteLabel Loyalty (system #1) - brief
- Displays points on hotel cards + checkout, collects a membership ID, configurable earn rules, tier differentiation, booking tracking, data export.
- Points formula: **(Booking Amount ÷ Value) × Reward × Tier Multiplier.** Example: $200 booking, Value = $1, Reward = 10 pts, Gold tier ×2 = 4,000 pts.
- **Two validation modes:**
  - *No Validation Required* - generic messaging only, zero integration, no tracking/collection.
  - *Membership Number Validation* - optional membership-number field at checkout, stored as-entered (no real-time validation; you reconcile later), added to Back Office columns (Membership #, Points Earned, Program). Accrual "After Check-out" or "At Booking".
- Tiers/multipliers/badges live in the **hosted-site config**, not the API. If WhereTo wants tiers, model them app-side.

Sources: `docs/loyalty-program-1`, `docs/loyalty-program-no-validation-required`, `docs/loyalty-program-membership-number-validation`.

### 6b. API-native Loyalty (system #2) - the strategic core

**Program settings**
- `GET /loyalties` → `{ "data": [ { "cashbackRate": 0.1, "cashbackCurrency": "USD", "status": "disabled" } ] }`. `cashbackRate` is a decimal (0.1 = 10%).
- `PUT /loyalties` - body `{ "status": "enabled", "cashbackRate": 0.1 }` (both required). Enables/updates the program.

> Reconciliation: the API program is a **single flat cashback rate + currency + on/off** - NOT the tiers/multipliers/badges from the WhiteLabel docs. Tiers must be modeled app-side.

**Guests model** - `GET /guests` returns:
```json
{
  "id": 5, "email": "joe@nuitee.com", "firstName": "Joe", "lastName": "Doe",
  "phoneNumber": "+330000000", "points": 3172, "upcomingPoints": 11,
  "bookings": ["MKDW0OjCT", "H1VRQCX5o"],
  "createdAt": "2026-07-08T13:50:55+02:00", "updatedAt": "...", "deletedAt": null
}
```
A logged-in WhereTo user maps to a LiteAPI **guest** (`id` → `guestId`). Bookings attach via the `bookings` array. `points` = redeemable now; `upcomingPoints` = pending from confirmed-but-not-checked-out stays.

**Points balance** - `GET /guests/{guestId}/loyalty-points` → `{ "data": { "currentPoints": 1540, "upcomingPoints": 900 } }` (note: guest object calls it `points`, this endpoint calls it `currentPoints`).

**Redeem points → voucher** - `POST /guests/{guestId}/loyalty-points/redeem`, body `{ "points": 100, "currency": "USD" }` (**10 points = $1 USD**). Returns a fixed-amount voucher:
```json
{
  "data": {
    "voucherCode": "0193f986", "guestID": 1, "discountType": "fixed_amount",
    "discountValue": 90.95, "minimumSpend": 0, "maximumDiscountAmount": 9,
    "currency": "USD", "validityStart": "2026-07-24", "validityEnd": "2026-12-24",
    "usages": 0, "usagesLimit": 1, "status": "active", "category": "redemption"
  }
}
```
The voucher is a discount code the guest applies to a future booking (via `voucherCode` on prebook - see `HOTELS.md`).

**Guest vouchers** - `GET /guests/{guestId}/vouchers` returns an array. **Casing gotcha:** the redeem response is **camelCase** (`voucherCode`, `discountValue`, `guestID`), but `GET vouchers` is **snake_case** (`voucher_code`, `discount_value`, `guest_id`). Normalize per-endpoint in code.

Sources: `reference/get_loyalties`, `reference/put_loyalties`, `reference/get_guests`, `reference/get_guests-guestid-loyalty-points`, `reference/post_guests-guestid-loyalty-points-redeem`, `reference/get_guests-guestid-vouchers`.

---

## 7. Synthesis - giving logged-in WhereTo users better pricing/rewards

Four independent levers, usable together:

1. **Member-only rates via CUG + `margin`.** A logged-in app is a legitimate non-public channel, so send a **lower `margin`** (even `0`, pure net) for authenticated users and a higher `margin` (at/above SSP) for anonymous ones. Instant, no extra state, stays compliant. **Recommended primary mechanic.**
2. **Cashback points.** Enable `PUT /loyalties` (`status: enabled`, e.g. `cashbackRate: 0.1`). Map each user to a `guestId`, attach bookings, accrue `points`/`upcomingPoints`, redeem into vouchers (10 pts = $1). A true earn-and-burn wallet for members. Requires storing `guestId` alongside each app user and binding it at book time.
3. **Vouchers as targeted perks.** Redemption mints vouchers, but vouchers (fixed/percent, min spend, max discount, validity, usage limit) are a general promo primitive for member rewards, win-backs, etc.
4. **Tier-gated perks.** `guestLevel` on perks lets higher-tier members see better in-stay benefits (breakfast, upgrades, credits). Tiers are app-side; the API exposes only a flat `cashbackRate`.

**Recommended for WhereTo:** drive member pricing with **CUG + per-request `margin`** (immediate), and layer **cashback points + redemption vouchers** via the Guests API for retention.

---

## 8. Open items to verify (not resolved by docs)

- The exact response field for **earned commission amount** - the rates schema shows `commission[] { amount, currency }` (in `HOTELS.md`); confirm it also appears on the Book response.
- **Dynamic pricing:** the concrete dashboard **enable toggle**, the exact **threshold/floor/ceiling field names**, and precedence vs a manual `margin`.
- **Create-guest** endpoint (`POST /guests`?) and the exact **`guestId` binding at book time** - the fetched pages covered fetch/points/redeem/vouchers but not guest creation or booking attachment. Verify against the Guests API section / a live book request.
- Whether any **monthly minimum / search-volume tiers / sandbox caps** exist beyond the per-request price-index/places charges.
- Points behavior when redeeming in a **non-USD currency** (the 10 pts = $1 rule is stated in USD).

---

_Cross-refs: `HOTELS.md` (rate/commission fields, `voucherCode` on prebook), `PAYMENTS.md` (merchant-of-record models), `PLATFORM.md` (loyalty endpoint hosts), `README.md`._
