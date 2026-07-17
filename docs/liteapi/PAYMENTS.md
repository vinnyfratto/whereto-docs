# LiteAPI (Nuitee Connect) - Payments, Stripe & Merchant of Record

**Scope:** the four payment methods, the User Payment (Stripe SDK) path we will use with Nuitee as merchant of record, the React Native / Expo integration, PCI scope, refunds/settlement, and sandbox testing.
**Last researched:** 2026-07-14. Applies to **both** flights and hotels (same payment model).

---

## 0. Headline facts - read first

1. **`payment.method` for the Stripe path is `TRANSACTION_ID`, NOT `TRANSACTION`.** Every current LiteAPI source (the book endpoint `anyOf` schema, the User Payment page, the flight prebook `paymentTypes`) uses the exact string `TRANSACTION_ID`. **Our existing hotel code and the older project notes use `'TRANSACTION'` - that is a latent bug and must be aligned/verified before the first live booking.** See §11.
2. **Nuitee (Nuitée Travel Limited) IS the merchant of record** on the User Payment path. Confirmed by the SDK hardcoding `business.name: 'Nuitee Connect'` and by Nuitee handling refunds/chargebacks/disputes. The other three methods make our app the merchant of record. This matches the chosen Model A.
3. **There is a native React Native package: `liteapi-react-native-payment-wrapper` (v1.0.6)**, built on `@stripe/stripe-react-native`. It is **native Stripe, not a WebView.** So Model A in Expo does **not** require a WebView (correcting the older assumption in `HOTEL_BOOKING_REFERENCE.md`). Caveats about its age/pinned deps in §4.2/§11.
4. **`secretKey` = a Stripe PaymentIntent client secret** (`pi_..._secret_...`). **`transactionId` = a LiteAPI transaction reference** (`tr_ct_...` hotels, `tr_cts_...` flights) that you pass back to book. Two different things.
5. **The four method enum values:** `TRANSACTION_ID`, `ACC_CREDIT_CARD`, `WALLET`, `CREDIT`.

---

## 1. The four payment methods

| Method | `payment.method` | What it is | Merchant of record | Use when |
|---|---|---|---|---|
| **User Payment** | `TRANSACTION_ID` (+ `transactionId`) | Use Nuitee's payment SDK to accept the traveler's card. "The most common option, allowing Nuitee Connect to handle all the payment processing." | **Nuitee** | Default. LiteAPI is MoR, handles PCI, refunds, disputes. **Our path.** |
| **Account Credit Card** | `ACC_CREDIT_CARD` | Pay from the account's card on file. Easiest for sandbox. | **Our app** | When we want to be MoR / bundle services and charge the traveler ourselves. |
| **Account Wallet** | `WALLET` | Prepay a balance, draw down for bookings; overflow auto-charges the account card. | **Our app** | Prepaid balance model. |
| **Credit Line** | `CREDIT` | Book against an invoiced credit line. | **Our app** | Enterprise; requires a contract + credit check. Not in sandbox. |

### Exact `payment` objects sent to book (`anyOf`)
```jsonc
// Shape A - no transactionId:
{ "method": "ACC_CREDIT_CARD" }   // or "WALLET" or "CREDIT"

// Shape B - User Payment / Stripe SDK path (ours):
{ "method": "TRANSACTION_ID", "transactionId": "tr_ct_ELlwT7nqeZCKsxZq1Vr" }
```

Method notes:
- **`ACC_CREDIT_CARD`:** each sandbox key has a hidden testing card, so this is the easiest end-to-end sandbox smoke test even without a card attached. The real account card is only on the production key.
- **`WALLET`:** must prepay funds first; overflow above balance auto-charges the account card.
- **`CREDIT`:** fails if no credit line is configured; not available in sandbox; test only refundable rates then cancel.

Source: `docs/implementing-payment`, `reference/post_rates-book`.

---

## 2. User Payment (Stripe SDK) - end-to-end sequence

1. **Prebook with `usePaymentSdk: true`** (hotels `POST /rates/prebook`, flights `POST /flights/prebooks`).
2. **Receive credentials** in the prebook response:
   - **`secretKey`** = Stripe PaymentIntent client secret (`pi_..._secret_...`). "The `secretKey` must be the same `secretKey` returned from the prebook API call."
   - **`transactionId`** = LiteAPI transaction reference (`tr_ct_...` / `tr_cts_...`) to pass to book. "Each call to prebook generates a new `transactionId` and `prebookId`, so it is important that you save the `transactionId` sent to the payment SDK."
   - Flights also return **`publishableKey`** (Stripe publishable key; often `null`). The hotel example did not include it (see §7).
   - The PaymentIntent lives in **Nuitee's Stripe account** → Nuitee is MoR.
3. **Mount the payment UI client-side** using `secretKey` (native RN sheet - §4).
4. **User pays** inside the Stripe-hosted UI (the app never sees the card number).
5. **Confirm.** Native wrapper fires `onPaymentSucceeded(transactionId)`; web SDK redirects to `returnUrl`.
6. **Book** with `payment: { method: "TRANSACTION_ID", transactionId }` + `prebookId` + `holder` + `guests`.

> Operational caution: unfinalized payment holds persist 1–2 business days. Always persist both `transactionId` and `prebookId` from each prebook (regenerated every call). For flights, remember the `transactionId` is **re-minted** if you attach seats/bags after prebook - always use the latest (see `FLIGHTS.md` §8).

Source: `docs/user-payment`.

---

## 3. Prebook is where the PaymentIntent is created

- **Hotel prebook** (`POST https://book.liteapi.travel/v3.0/rates/prebook`) with `usePaymentSdk: true` returns `prebookId`, `transactionId` (`tr_ct_...`), `secretKey` (`pi_..._secret_...`), plus price/commission/cancellation fields. See `HOTELS.md` §8.
- **Flight prebook** (`POST https://api.liteapi.travel/v3.0/flights/prebooks`) with `usePaymentSdk: true` returns `prebookId`, `transactionId` (`tr_cts_...`), `secretKey`, **`publishableKey`**, and `paymentTypes[]` (e.g. `["TRANSACTION_ID","CREDIT","WALLET"]`). It also supports `payment.descriptorSuffix` to customize the Stripe statement descriptor. See `FLIGHTS.md` §7.

Both `secretKey`s are denominated in the prebook `currency`.

---

## 4. React Native / Expo integration

### 4.1 Recommended shape
Because prebook already returns a Stripe **PaymentIntent client secret** (`secretKey`) and (for flights) a **publishableKey**, the cleanest native integration is:

> Prebook server-side (our edge function) → return `secretKey` (+ `publishableKey`) to the app → confirm the PaymentIntent with **`@stripe/stripe-react-native`** natively → observe `succeeded` → call our book edge function with `{ method: "TRANSACTION_ID", transactionId }`.

This keeps the LiteAPI key server-side (the app only ever holds the Stripe client secret, which is safe to expose to the client by design) and uses the maintained Stripe RN SDK directly.

### 4.2 The official wrapper: `liteapi-react-native-payment-wrapper` (v1.0.6)
A first-party wrapper exists and is **native Stripe** (depends on `@stripe/stripe-react-native ^0.38.4`, not a WebView).

```
npm install liteapi-react-native-payment-wrapper
```
```tsx
import { PayButton } from 'liteapi-react-native-payment-wrapper';
import LiteAPIPayment from 'liteapi-react-native-payment-wrapper/dist/LiteAPIPayment';

<LiteAPIPayment sandbox={false}>
  <PayButton
    offerId="[OFFER-ID]"
    apiKey="[YOUR-API-KEY]"
    onPaymentSucceeded={(transactionId: string) => { /* call book */ }}
    onPaymentFailed={() => { /* handle failure */ }}
    buttonColor="blue" textColor="white" borderRadius={10}
    buttonWidth={200} buttonHeight={50} fontWeight="bold" buttonTitle="Book now"
  />
</LiteAPIPayment>
```
- `<LiteAPIPayment sandbox>` - boolean env toggle.
- `<PayButton>` props: `offerId`, `apiKey`, `onPaymentSucceeded(transactionId)`, `onPaymentFailed`, plus styling (`buttonColor`, `textColor`, `borderRadius`, `buttonWidth`, `buttonHeight`, `fontWeight`, `buttonTitle`).
- **The wrapper does the prebook itself** (it takes raw `offerId` + `apiKey`, prebooks internally, mounts the Stripe sheet, returns `transactionId`).

**Why we likely should NOT use the wrapper as-is:**
- **It puts `apiKey` on the device** (it prebooks client-side). That conflicts with our server-side-key posture. If used, only a sandbox/appropriately-scoped key should ship.
- **It is stale (~2 years old)** and pins `react-native ^0.74.5`, `react ^18.2.0`, `@stripe/stripe-react-native ^0.38.4`, `axios ^0.21.1` (axios 0.21 has known CVEs). Under Expo SDK 56 (newer RN, React 19) these ranges likely clash.
- **Recommendation:** drive `@stripe/stripe-react-native` directly with the prebook `secretKey`/`publishableKey` from our edge function (§4.1), rather than adopt the wrapper. Use the wrapper only as a reference implementation.

### 4.3 Expo specifics
- `@stripe/stripe-react-native` uses native modules → requires an **Expo dev client / prebuild (config plugin)** and `pod install` on iOS. **It will not run in Expo Go.** This is a build-setup item to plan for (we are already SDK 56 with custom native config).

### 4.4 Web SDK (fallback, web-only)
`liteAPIPayment.js` is a browser SDK loaded by `<script>`, configured with `publicKey: 'live'|'sandbox'`, `secretKey` (the prebook client secret), `targetElement`, and `returnUrl`; invoked via `new LiteAPIPayment(config).handlePayment()`. It is **redirect-based** (confirmation returns to `returnUrl?tid=...&pid=...`, no success callback). For RN it would require an HTML page in `react-native-webview` and intercepting the return URL. Prefer the native path.

Source: `docs/user-payment`; npm registry for the wrapper.

---

## 5. PCI scope

On the User Payment path the app **never touches raw card data** - card entry is inside the Stripe-hosted UI (native Stripe sheets on RN, Stripe Elements on web). This keeps us at **PCI SAQ A** scope. (Follows from the hosted-form architecture; corroborated by web sources, not stated verbatim in the LiteAPI page.)

---

## 6. Add-ons on payment

Add-ons are attached **at prebook** (`addons[]`) so their cost is folded into the PaymentIntent the SDK charges. Supported: **Uber vouchers** ($10 increments, $10–$100, USD, non-refundable) and **eSIM** (via eSimply, USD, non-refundable, with `addonDetails{package_id,destination_code,start_date,end_date}`; discover packages via `/v3.0/addons/esimply/packages/{countryCode}`). USD only - convert in the UI for other currencies. The book response returns `addons[]` with `addonVoucherCode` (Uber) and `qrCode` (eSIM). See `HOTELS.md` §12.

Source: `docs/attaching-add-ons-to-user-payment`.

---

## 7. 3-D Secure / SCA, card networks, currencies, Apple/Google Pay

**Not documented by LiteAPI.** Reasoned from the Stripe dependency (verify with LiteAPI support before relying on any of these):
- **3-D Secure / SCA:** handled automatically by Stripe (native 3DS via `@stripe/stripe-react-native`; Stripe.js on web).
- **Apple Pay / Google Pay:** supported by `@stripe/stripe-react-native` in principle, but the documented `PayButton` exposes only a card button. Treat as "supported by the SDK, unconfirmed in the LiteAPI wrapper" - likely needs driving Stripe RN directly.
- **Card networks:** not enumerated; inherited from Nuitee's Stripe account (typically Visa/MC/Amex).
- **Currencies:** the prebook `currency` drives the PaymentIntent denomination; examples are USD. Add-ons are USD-only. No published settlement-currency list.

---

## 8. Refunds, chargebacks, payouts, settlement

- **Funds (User Payment):** held by Nuitee (MoR) via its Stripe account. The traveler pays the full selling price (net + margin + add-ons) to Nuitee.
- **Commission settlement:** a booking is "confirmed" when the guest **checks out**; commission is then locked in and paid in the **next weekly payout**. (So commission is realized at checkout, not at booking.)
- **Refunds & chargebacks:** Nuitee, as MoR, handles refunds/chargebacks/disputes; refunds flow back through Nuitee per the rate's `cancellationPolicies`/`refundableTag`. The booking record exposes `refundedAt`, `cancelledAt`, `cancelledBy`, `goodwillPayment`.
- **Other three methods:** our app is MoR - we collect from the traveler and own our refunds/chargebacks; LiteAPI charges our card/wallet/credit line the net + markup.

See `COMMERCIALS.md` for net/retail/margin math.

---

## 9. Test cards / sandbox

- **Sandbox card:** `4242 4242 4242 4242`, any future expiry, any CVC (standard Stripe test card).
- **Env matching:** web SDK `publicKey: 'sandbox'` (+ sandbox API key); RN wrapper `<LiteAPIPayment sandbox={true}>`.
- **`ACC_CREDIT_CARD`** is the easiest full end-to-end sandbox test (hidden test card on each sandbox key).
- Book responses include `"sandbox": true` - assert environment in tests.

---

## 10. Recommended WhereTo payment architecture (Model A, both flights + hotels)

```
[app] select room/flight
   │
   ▼
[edge fn] prebook (usePaymentSdk:true)  ──►  { prebookId, transactionId, secretKey, publishableKey? }
   │                                            (LiteAPI key stays server-side)
   ▼
[app] @stripe/stripe-react-native confirm PaymentIntent(secretKey)  ──►  status: succeeded
   │
   ▼
[edge fn] book  { prebookId, holder, guests, payment:{ method:"TRANSACTION_ID", transactionId }, clientReference }
   │
   ▼
persist order (hotel_orders / flight order), fire confirmation notification
```

- One `LITEAPI_KEY` secret serves both. Hotels book on `book.liteapi.travel`, flights book on `api.liteapi.travel` (see `FLIGHTS.md`).
- The app only ever holds the Stripe **client secret**, never the LiteAPI key.

---

## 11. Corrections to our existing code / notes

1. **`TRANSACTION` → `TRANSACTION_ID`.** `src/providers/hotels/types.ts` (`HotelPayment`), `src/utils/hotelBooking.ts`, and the `liteapi-book` book call all assume `method: 'TRANSACTION'`. Current docs use `TRANSACTION_ID` everywhere. **Verify against the live sandbox book call, then align the enum.** (The hotel booking reference example shows `TRANSACTION_ID`; some older LiteAPI material used `TRANSACTION` - the sandbox is the tiebreaker, but plan for `TRANSACTION_ID`.)
2. **Model A does not need a WebView.** `HOTEL_BOOKING_REFERENCE.md` §11 item 2 says the payment UI needs "a WebView that loads the SDK." The native `@stripe/stripe-react-native` path (§4) is better. Update that plan.
3. **Payment UI is still the deferred fork** - this doc is the spec for building it (native Stripe RN, confirm PaymentIntent from prebook `secretKey`, book with `TRANSACTION_ID`).

---

## 12. Open questions / risks

1. **Confirm `TRANSACTION_ID`** (vs `TRANSACTION`) against a live sandbox booking before shipping - load-bearing for a successful charge.
2. **Wrapper vs direct Stripe RN:** prefer driving `@stripe/stripe-react-native` directly (maintained, keeps our key server-side) over the stale `liteapi-react-native-payment-wrapper@1.0.6`.
3. **API key on device:** if the wrapper is used, confirm which key scope is safe for production; otherwise front prebook through our edge function and hand only the Stripe secret to the client.
4. **Expo:** native Stripe requires a dev client / prebuild config plugin + `pod install` (no Expo Go).
5. **Apple Pay / Google Pay:** confirm availability (likely needs direct Stripe RN, not the wrapper).
6. ~~**`publishableKey`:** flights return it; confirm whether hotels prebook returns one for native init, or whether we must configure the Stripe publishable key another way.~~ **Resolved 2026-07-15:** neither reliably does — hotel prebook's `usePaymentSdk:true` response has no publishable-key field at all (confirmed via live sandbox debug log: full key list is `prebookId, offerId, hotelId, currency, termsAndConditions, roomTypes, suggestedSellingPrice, isPackageRate, commission, price, priceType, priceDifferencePercent, cancellationChanged, boardChanged, supplier, supplierId, transactionId, secretKey, paymentTypes, checkin, checkout, sellingPriceToUser`), and flights' own documented example returns `publishableKey: null`. It's account-level config (identifies Nuitee's Stripe account), not a per-transaction value — get it directly from Nuitee/LiteAPI support, not the API.

---

## 13. Sources

Official: `docs/implementing-payment`, `docs/user-payment`, `docs/account-credit-card`, `docs/account-wallet`, `docs/credit-line`, `docs/revenue-management-and-commission`, `docs/attaching-add-ons-to-user-payment`, `reference/post_rates-prebook`, `reference/post_flights-prebooks`, `reference/post_rates-book`.
Package registry / web (secondary): `registry.npmjs.org/liteapi-react-native-payment-wrapper` (README + deps), `github.com/liteapi-travel/react-native-payment-wrapper`, `npmjs.com/package/liteapi-node-sdk`; Stripe docs for 3DS/SCA/Apple Pay/Google Pay; merchant-of-record corroboration via Stripe hospitality pages and third-party privacy policies naming Nuitée as MoR.

---

_Cross-refs: `FLIGHTS.md` (flight prebook/book, re-minted transactionId), `HOTELS.md` (hotel prebook/book), `COMMERCIALS.md` (commission/settlement), `PLATFORM.md` (security/PCI), `README.md` (transition plan)._
