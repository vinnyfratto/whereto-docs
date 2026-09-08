# LiteAPI (Nuitee Connect) - Payments, Stripe & Merchant of Record

**Scope:** the four payment methods, the User Payment (Stripe SDK) path we will use with Nuitee as merchant of record, the React Native / Expo integration, PCI scope, refunds/settlement, and sandbox testing.
**Last researched:** 2026-07-14. **Materially corrected 2026-07-29 — see §0.0.** Applies to **both** flights and hotels (same payment model).

---

## 0.0 ⚠️ CORRECTION (2026-07-29) — read before §4

The 2026-07-14 research recommended **native `@stripe/stripe-react-native`** driven by the prebook's
`secretKey` + `publishableKey`. **That was built, it failed, and it was fully reverted (v0.3.124 →
v0.3.125).** The rest of this document was never updated, so §0 item 3, §4.1, §10, §11 item 2 and
§12 item 2 all still point at the abandoned approach. They are corrected in place below; this
section is the summary.

**1. There is no Stripe publishable key, and there never was one to wait for.**
The hosted SDK's `publicKey` field is **not** a Stripe key — it takes the literal string
`"sandbox"` or `"live"`. Nuitee support confirmed separately that they do not issue a Stripe
publishable key for this flow at all, because the flow was never designed for a native SDK. The
"blocked on a key from Nuitee" belief traces entirely to §4.1's recommendation, which needed one.

**2. The real config is four values we already hold:**

| Field | Value |
|---|---|
| `publicKey` | `"sandbox"` or `"live"` — an environment string, NOT a key |
| `secretKey` | the prebook's Stripe client secret (the **latest** one — see §2 note) |
| `targetElement` | a DOM id in our own HTML shell |
| `returnUrl` | a URL we intercept to detect success |

**3. Model A in Expo DOES require a WebView.** `HOTEL_BOOKING_REFERENCE.md` §11 item 2 was right
all along and §0 item 3 / §11 item 2 below were wrong. The payment step is browser JavaScript that
renders into a DOM node; there is no native equivalent Nuitee will support.

**4. Importing `@stripe/stripe-react-native` at all crashes any build lacking the compiled native
module** — the crash fires at JS module-load time in `app/_layout.tsx`, not at hook-call time, so a
half-wired "off by default" flag does not avoid it. Do not reintroduce the package speculatively.

**5. Flights have no fallback.** Hotels can become their own MoR via `ACC_CREDIT_CARD`; **flights
reject `ACC_CREDIT_CARD` with `45009`** (`FLIGHTS.md` §10) and `CREDIT` needs a credit line this
account does not have. For flights, `TRANSACTION_ID` via the hosted widget is the only route.

---

## 0. Headline facts - read first

1. ~~**`payment.method` for the Stripe path is `TRANSACTION_ID`, NOT `TRANSACTION`.**~~ **Resolved and shipped** — verified against live sandbox bookings for both products; the enum is `TRANSACTION_ID` and the code was aligned in v0.3.103 (hotels) / v0.3.143 (flights).
2. **Nuitee (Nuitée Travel Limited) IS the merchant of record** on the User Payment path. Confirmed by the SDK hardcoding `business.name: 'Nuitee Connect'` and by Nuitee handling refunds/chargebacks/disputes. The other three methods make our app the merchant of record. This matches the chosen Model A.
3. ~~**There is a native React Native package... So Model A in Expo does not require a WebView.**~~ **WRONG — see §0.0.** The wrapper exists but is stale and unusable here, the native path was reverted, and Model A **does** require a WebView.
4. **`secretKey` = a Stripe PaymentIntent client secret** (`pi_..._secret_...`). **`transactionId` = a LiteAPI transaction reference** (`tr_ct_...` hotels, `tr_cts_...` flights) that you pass back to book. Two different things.
5. **The four method enum values:** `TRANSACTION_ID`, `ACC_CREDIT_CARD`, `WALLET`, `CREDIT`. Flights additionally accept `THIRD_PARTY` and reject `ACC_CREDIT_CARD`/`WALLET` — see `FLIGHTS.md` §10.

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
   - ~~Flights also return **`publishableKey`**~~ — **they do not.** A live flight prebook captured 2026-07-29 has no `publishableKey` field at all (not null — absent). See `FLIGHTS.md` §7a.
   - The PaymentIntent lives in **Nuitee's Stripe account** → Nuitee is MoR.
3. **Mount the payment UI client-side** using `secretKey`, in a WebView (§4).
4. **User pays** inside the Stripe-hosted UI (the app never sees the card number).
5. **Confirm.** The SDK redirects to `returnUrl?tid=...&pid=...`; there is no success callback, so the host intercepts the navigation.
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

### 4.1 Recommended shape — WebView over the hosted widget

> Prebook server-side (our edge function) → return `secretKey` to the app → render LiteAPI's hosted
> payment script inside a `react-native-webview` → the widget redirects to our sentinel `returnUrl`
> → intercept that navigation → call our book edge function with `{ method: "TRANSACTION_ID", transactionId }`.

This keeps the LiteAPI key server-side (the app only ever holds the Stripe client secret, which is
safe to expose to the client by design). `react-native-webview` is already a dependency (13.16.1)
and already used elsewhere in the app, so this adds no native build work and nothing that can break
Expo Go behaviour further than it already is.

Implemented as `src/components/booking/LiteApiPaymentView.tsx`.

> ~~**PREVIOUS RECOMMENDATION (do not follow):** confirm the PaymentIntent with
> `@stripe/stripe-react-native` natively using `secretKey` + `publishableKey`.~~
> Built and reverted in v0.3.124 → v0.3.125. It needs a Stripe publishable key for Nuitee's account,
> which Nuitee does not issue (§0.0). Reintroducing the package also crashes any build without the
> compiled native module, at module-load time.

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

### 4.4 The hosted web SDK — THIS IS THE PATH (was labelled "fallback, web-only")

`liteAPIPayment.js` is a browser SDK loaded by `<script>`:

```html
<script src="https://payment-wrapper.liteapi.travel/dist/liteAPIPayment.js?v=a1"></script>
<div id="targetElement"></div>
<script>
  var liteAPIPayment = new LiteAPIPayment({
    publicKey:     'sandbox',        // literally 'sandbox' | 'live' — NOT a Stripe key
    secretKey:     '<prebook secretKey>',
    targetElement: '#targetElement',
    returnUrl:     'https://whereto.payment/return',
  });
  liteAPIPayment.handlePayment();
</script>
```

It is **redirect-based**: on success it navigates to `returnUrl?tid=...&pid=...` with no success
callback, so the RN host detects completion by intercepting the navigation
(`onShouldStartLoadWithRequest`). The `returnUrl` never has to resolve — it only has to be
recognisable, so a sentinel origin we own is enough.

`publicKey` being an environment string rather than a key is the single fact that unblocks this
whole path; see §0.0.

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
[edge fn] prebook (usePaymentSdk:true)  ──►  { prebookId, transactionId, secretKey, liveMode }
   │                                            (LiteAPI key stays server-side)
   ▼
[app] (flights only) attach seats/bags  ──►  RE-MINTED transactionId + secretKey — use these
   │
   ▼
[app] WebView → payment-wrapper.liteapi.travel with { publicKey:'sandbox'|'live', secretKey }
   │      user pays inside Stripe's hosted form
   │      widget redirects to returnUrl?tid=…&pid=…  ──►  host intercepts = paid
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

1. ~~**`TRANSACTION` → `TRANSACTION_ID`.**~~ **Done** — aligned v0.3.103 (hotels) / v0.3.143 (flights) and confirmed by live sandbox bookings on both products.
2. ~~**Model A does not need a WebView.**~~ **This correction was itself wrong.** `HOTEL_BOOKING_REFERENCE.md` §11 item 2 ("the payment UI needs a WebView that loads the SDK") is **correct** — leave that plan alone. See §0.0.
3. **Payment UI: flights are wired, hotels are not.** `LiteApiPaymentView` + the flight checkout call it before book. `hotelBooking.ts` still books without a payment step and needs the same treatment.

---

## 12. Open questions / risks

1. ~~**Confirm `TRANSACTION_ID`**~~ — done, see §11 item 1.
2. ~~**Wrapper vs direct Stripe RN**~~ — **neither.** Both need a publishable key Nuitee does not issue. The hosted widget in a WebView is the path (§4.1, §4.4).
3. ~~**API key on device**~~ — moot: we never adopted the wrapper, and prebook stays behind our edge function. The device only ever receives the Stripe client secret.
4. ~~**Expo:** native Stripe requires a dev client / prebuild~~ — moot; no native Stripe module. `react-native-webview` is already in the build.
5. **Apple Pay / Google Pay:** unknown, and now a question about the *hosted widget* rather than about a native SDK. Whether Stripe's hosted form offers the payment-request button inside a WebView is unverified — assume card-only until tested.
6. ~~**`publishableKey`**~~ — **fully resolved 2026-07-29:** there is nothing to obtain. Hotel prebook has no such field; flight prebook has none either (a live capture shows it absent, not null). The hosted SDK's `publicKey` is the string `'sandbox'`/`'live'`. Do not open a support ticket for a Stripe publishable key.

### Answered by test (2026-07-29)

- ~~**Does the hosted widget work for FLIGHT prebooks?**~~ **Yes.** Verified twice in sandbox, including a 2-passenger real-carrier booking with seats and bags attached (`FH-267-LAPCCKWP`). The widget accepts a flight prebook's `secretKey` even though `docs/user-payment` only documents hotels.
- ~~**Real GDS carriers still fail at book with `55099`.**~~ **They book once payment is settled.** The payment step is what cleared it — see `FLIGHTS.md` §10 for the three runs. This is the opposite of what §12 said a few hours earlier; the intermediate reading was drawn from a synthetic carrier that does not enforce payment.

### Genuinely open

- **The missing control:** a real carrier with payment skipped, on the current build, to confirm the causal link rather than infer it from one changed variable. One-minute test via `FORCE_PAYMENT_IN_SANDBOX`.
- **Apple Pay / Google Pay** inside the hosted form in a WebView — unverified, assume card-only.
- ~~**Hotels still book with no payment step.**~~ **Done 2026-07-29** — `HotelPrebookView` runs `LiteApiPaymentView` and books with `TRANSACTION_ID`. `ACC_CREDIT_CARD` survives only as the sandbox fallback when the widget is dismissed, and is unreachable in live mode. `HotelWizard.tsx` is dormant and still on the old path; see its header.
- ⚠️ **The prebook `payment` block breaks BOTH products. It is opt-in and ships OFF.**

  | Product | Sending `payment` on prebook |
  |---|---|
  | Hotels | `500 {"code":5000,"description":"payment create failed, please try again"}` |
  | Flights | `500 {"code":53099,"description":"Failed to create prebook"}` |

  Both on fresh offers that verified seconds earlier, across multiple routes and
  properties. `payment.descriptorSuffix` **is** on the flight prebook reference
  page, so "it's documented" was not sufficient evidence that it works.

  The design mistake worth not repeating: `descriptorSuffix` was given a *default
  value*, which meant the block shipped to production the instant the function
  deployed, with nobody having opted in. Nothing is sent now unless a secret is
  explicitly set, so an unconfigured deploy is byte-identical to the last
  known-good build. `LITEAPI_PMC_ID` is additionally shape-checked against
  `/^pmc_[A-Za-z0-9]+$/` and logged-and-ignored otherwise, because a literal
  `pmc_...` placeholder would otherwise fail every prebook with an error that
  names none of this. Hotels carry a second gate, `LITEAPI_HOTEL_PAYMENT_OPTS=1`.

  **Confirmed by Nuitee's own error assistant** (the wrench icon on a log row at
  `connect.nuitee.com/request-logs/`), run against one of the 53099 flight
  prebooks. It identified both fields as the cause:
  - `paymentMethodConfiguration: "pmc_..."` — the literal placeholder was
    genuinely in the request, which is why the shape check now exists.
  - `descriptorSuffix` — *"may not be accepted for flight prebooks (it's not
    shown as a documented field for this endpoint)"*.

  Note that last point **contradicts the flight prebook reference page**, which
  does list `payment.descriptorSuffix`. Empirically the assistant is right and
  the reference is wrong. Its sanctioned shape is PMC-only:
  ```json
  "payment": { "paymentMethodConfiguration": "pmc_REAL_VALUE_FROM_YOUR_DASHBOARD" }
  ```
  So if a real `pmc_` ever arrives: set `LITEAPI_PMC_ID` and leave
  `PAYMENT_DESCRIPTOR_SUFFIX` unset. Do not send both.

  **Consequence: statement descriptors fall back to Nuitee's default on both
  products.** Travellers will see a Nuitée line rather than a WhereTo one — a
  chargeback risk worth raising with Nuitee alongside the `pmc_` request, since
  the API route to setting it does not currently work.

  > The wrench is a genuinely useful diagnostic and worth reaching for before
  > theorising. Treat its output as a suggestion, not instruction — it is an LLM
  > reading the same request and response, and it was wrong about the reference
  > page even while being right about the fix.
- **`LITEAPI_PMC_ID`** (optional, both functions) sets `payment.paymentMethodConfiguration`, which controls which methods the hosted form offers — the lever for dropping Link and its SMS code. The `pmc_...` must exist on **Nuitee's** Stripe account, so it has to be requested from them; unset means their default.
- **3-D Secure on a live card.** Sandbox test cards do not exercise a real challenge; `setSupportMultipleWindows={false}` is in place for it but untested.

---

## 13. Sources

Official: `docs/implementing-payment`, `docs/user-payment`, `docs/account-credit-card`, `docs/account-wallet`, `docs/credit-line`, `docs/revenue-management-and-commission`, `docs/attaching-add-ons-to-user-payment`, `reference/post_rates-prebook`, `reference/post_flights-prebooks`, `reference/post_rates-book`.
Package registry / web (secondary): `registry.npmjs.org/liteapi-react-native-payment-wrapper` (README + deps), `github.com/liteapi-travel/react-native-payment-wrapper`, `npmjs.com/package/liteapi-node-sdk`; Stripe docs for 3DS/SCA/Apple Pay/Google Pay; merchant-of-record corroboration via Stripe hospitality pages and third-party privacy policies naming Nuitée as MoR.

---

_Cross-refs: `FLIGHTS.md` (flight prebook/book, re-minted transactionId), `HOTELS.md` (hotel prebook/book), `COMMERCIALS.md` (commission/settlement), `PLATFORM.md` (security/PCI), `README.md` (transition plan)._
