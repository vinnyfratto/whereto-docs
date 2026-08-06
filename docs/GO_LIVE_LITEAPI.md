# LiteAPI Booking — What Works, What Is Blocked, What Production Needs

**Written:** 2026-07-29, at the end of the session that wired the payment step.
**Scope:** the state of flight + hotel booking against LiteAPI (Nuitee Connect),
what was proven in sandbox, what is blocked and on whom, and the exact list of
things that must change to go live.

This is the handover document. `docs/liteapi/FLIGHTS.md` and
`docs/liteapi/PAYMENTS.md` remain the API reference; this is the state of play.

---

## 1. TL;DR

| | Flights | Hotels |
|---|---|---|
| Search / quote | ✅ | ✅ |
| Verify / re-quote | ✅ | ✅ |
| Prebook | ✅ | ✅ |
| Seats + bags (attach) | ✅ | n/a |
| **Traveller payment** | ✅ | ✅ |
| Book | ✅ **synthetic carrier only** | ✅ |
| Real inventory | ❌ **blocked at Nuitee** | ✅ |

**Nothing in our code is known to be blocking.** The one open item is an account
entitlement: the sandbox key cannot ticket real airlines. Hotels book real
properties today.

---

## 2. What was proven end to end

All in sandbox, all multi-passenger, all with the hosted payment widget in front
of book.

**Flights** — `FH-267-LAPCCKWP`, 2 passengers, AUS→MRS return:
```
search → verify → prebook ($2,650.54, 244 attachable services)
       → attach 2 bags + 2 seats ($2,794.39)
       → transaction re-minted  tr_cts_83B…U9iN → tr_cts_PrJ…vXrw
       → hosted payment widget → returnUrl
       → book TRANSACTION_ID with the RE-MINTED id → CONFIRMED
```

**Hotels** — `-ntuge3a_`, The Reykjavik EDITION, 2 guests, 7 nights, $10,481:
```
re-quote → prebook (txn + secret) → hosted payment widget → returnUrl
        → book TRANSACTION_ID → CONFIRMED
```
Three hotel bookings, three Confirmed, verified on Nuitee's Bookings page.

**`TRANSACTION_ID` had never succeeded on hotels before this session.** It
returned `2014 "payment not completed"` in July when tried without a real
payment. Hotels had only ever booked via `ACC_CREDIT_CARD`, which charges *our*
account card and asks the traveller for nothing. That is not a business model,
and it is now the sandbox fallback only.

---

## 3. The one blocker: real carriers

`55099 "Failed to create booking"` on every real airline. Settled by the
operating-carrier column on Nuitee's **Flight Reservations** page, which no API
response exposes:

| Carrier | Result |
|---|---|
| Nuitée Air (synthetic) | CONFIRMED, every time |
| United Airlines | FAILED, Payment Refunded |
| Air Canada | FAILED, Payment Refunded |

Fourteen bookings, no exceptions. Payment state is irrelevant — real carriers
fail with and without a completed payment, Nuitée Air succeeds either way. The
request body is byte-identical on success and failure (compared side by side in
Nuitee's request logs).

**Nuitee's own error assistant** (wrench icon on a log row) concluded:

> "If you are already using the correct schema and it succeeds for Nuitée Air,
> the request body likely does not need changes. The correction is
> **environment / account enablement**, not JSON."

A support ticket is open. Its questions: is real GDS/NDC inventory bookable on a
sandbox key at all; what enables live flight booking and ticketing; which
providers are enabled; are there carrier restrictions; is anything else needed
for the payment flow.

**Real-airline booking cannot be validated before production.** Plan the
go-live schedule around that, because the lead time is Nuitee's, not ours.

---

## 4. Production checklist

### Must do

1. **Get flight booking enabled on a production key.** The open ticket. Longest
   lead time; start everything else in parallel.
2. **Re-test the payment widget against a live key.** Sandbox settles without a
   payment, so a green sandbox run proves the UI works, not that the charge is
   required. `liveMode` flows from the edge functions and flips the widget's
   `publicKey` to `'live'` automatically — no code change, but verify it.
3. **Test 3-D Secure on a real card.** Sandbox test cards never raise a real
   challenge. `setSupportMultipleWindows={false}` is in place for it
   (`LiteApiPaymentView.tsx`) and is untested against an actual issuer.
4. **Turn off `FORCE_PAYMENT_IN_SANDBOX`** — or rather, confirm it is irrelevant.
   It lives in `LiteApiPaymentView.tsx`. In live mode payment is mandatory and
   the sandbox escape hatches are unreachable, but read the two call sites in
   `FlightPrebookView` / `HotelPrebookView` and satisfy yourself.
5. **Statement descriptor.** Currently falls back to Nuitee's default, so
   travellers see a Nuitée line on their card, not WhereTo. Chargeback risk.
   The API route does not work (§5); it has to come from Nuitee.
6. **Decide what happens to `ACC_CREDIT_CARD` on hotels.** It is the sandbox
   fallback when the widget is dismissed. Unreachable in live mode, but it is a
   path that charges our own card and it should probably be deleted rather than
   left dormant.

### Should do

7. **Persist the provisional booking.** Prebook creates a provider-side booking
   with a real airline PNR *before* payment. The edge function maps it to
   `provisional` and the client drops it. After a failed book there may be a
   record at Nuitee we have no trace of. Not an orphan risk today (failures show
   as `FAILED`/`Refunded`), but we are flying blind on it.
8. **Move `transactionId` server-side.** The client currently holds it between
   prebook and book. LiteAPI's own architecture guidance is a TTL-keyed
   `booking_sessions` row, keyed on `prebookId`, never trusting a client value.
9. **Retry policy on prebook.** `fetchLite` retries 429 only. LiteAPI's guidance
   for `/prebooks` is 503 → backoff 1s/2s/4s, 502 → no auto-retry.
10. **`filters.flightNumbers` does nothing.** The targeted re-search returns 0
    while the plain search returns the same journey. Either fix the filter
    format or drop the pass; it is a wasted round trip on every offer refresh.
11. **Wire up the remaining Supabase Auth security notifications** (email
    changed, phone changed, sign-in method removed, MFA added/removed) once
    their matching profile features exist. Full inventory + copy-paste-ready
    templates in `docs/EMAIL_TEMPLATES_SETUP.md`.
12. **Move off Supabase's built-in email service before real signups scale.**
    It's rate-limited and explicitly "not meant for production apps" per
    Supabase's own dashboard warning. Resend is already fully configured —
    pointing Supabase Auth's SMTP settings at Resend's relay fixes this and
    also removes the free-tier template-editing lock. See
    `docs/EMAIL_TEMPLATES_SETUP.md`.
13. ~~**Add a Resend webhook for `email.bounced` / `email.complained`.**~~
    **Done 2026-07-30** — `supabase/functions/resend-webhook` verifies the
    Svix signature, logs both event types to `public.email_delivery_events`
    (RLS-locked, service-role only), and fires a best-effort alert email to
    `vfratto@vcinnovationsgroup.com` + `ccupero@vcinnovationsgroup.com` (the
    `ALERT_EMAILS` Supabase secret, comma-separated, overrides the default
    without a redeploy). Verified end to end twice against Resend's
    `bounced@resend.dev` / `complained@resend.dev` test addresses — DB rows
    and alert emails both confirmed landing correctly. Still no auto-flagging
    of bad addresses on `profiles` — that'd be the next layer if it's wanted.

---

## 5. Hard-won facts — do not re-derive these

Each of these cost real debugging time. They are counter-intuitive and several
contradict LiteAPI's own documentation.

**The payment widget needs no Stripe key.** Its `publicKey` field takes the
literal string `'sandbox'` or `'live'`. Nuitee does not issue a Stripe
publishable key, because the flow was never designed for a native SDK. The
native `@stripe/stripe-react-native` path was built and reverted (v0.3.124 →
v0.3.125). **Model A requires a WebView.** Do not try native Stripe again.

**The prebook `payment` block breaks BOTH products.** Sending it returns `53099`
on flights and `5000` on hotels — on fresh offers, every time. `descriptorSuffix`
*is* on the flight prebook reference page; being documented was not enough. It is
opt-in and ships off. Never give it a default value: doing so shipped it to
production the moment the function deployed, on both products, with nobody
having asked for it.

**Attach re-mints BOTH the transactionId and the secretKey.** Charging the
superseded secret bills the fare without the extras and then books a total
nobody paid. Both are carried; the confirm log labels which was used.

**LiteAPI returns seats it will not sell,** marked `available: false` and priced
at `0`. Anything keying "free seat" off price alone renders exactly the
unsellable seats as the most attractive ones.

**Aircraft type is only on the offer's `segmentAmenities`,** never on
`journey.segments`. Every itinerary reported an unknown plane while the data sat
one level up.

**`hotelConfirmationCode` is `"test"` in sandbox.** It was first in the
fallback chain, so travellers were shown "test" as their confirmation number.
Prefer the supplier booking id, which is what the property keys off.

**`priceDifferencePercent` is unreliable in both directions** — 0% during real
drift, and -1% on a 25-cent increase. Gate on our own computed delta.

**Passenger type has two spellings.** The reference documents
`passengerType` (int); LiteAPI's own testbed sends `type: "ADT"`. The response
echoes neither. We send both.

---

## 6. Method notes

Two process lessons, both expensive.

**Go to the provider's own records before theorising.** `55099` was explained
three different ways in one day, each confidently, each from a single run with
one variable changed and no repeat — and twice from a data point that turned out
to be mislabelled (a "real carrier" booking that was actually Nuitée Air). The
Flight Reservations page settled it in minutes because it showed the one column
the API never returns.

**Nuitee's error assistant is a lead, not an authority.** It correctly diagnosed
the `53099` payment block first time. On `55099` it confidently claimed we were
posting the hotel payload to the flights endpoint and told us to rebuild a
request that had returned `201` twice that day; it only corrected when handed
the contradicting booking refs.

---

## 7. Where things live

| Concern | File |
|---|---|
| Flight workflow | `src/utils/flightBooking.ts` |
| Hotel workflow | `src/utils/hotelBooking.ts` |
| Payment step (both) | `src/components/booking/LiteApiPaymentView.tsx` |
| Error classification | `src/utils/checkoutErrors.ts` |
| Flight checkout UI | `src/components/booking/FlightPrebookView.tsx` |
| Hotel checkout UI | `src/components/booking/HotelPrebookView.tsx` |
| Seat map | `src/components/booking/SeatMap.tsx` |
| Flight edge function | `supabase/functions/liteapi-flights/index.ts` |
| Hotel edge function | `supabase/functions/liteapi-book/index.ts` |
| Staging area | `app/bookings/passengers.tsx` |

Secrets, all optional and all currently unset: `PAYMENT_DESCRIPTOR_SUFFIX`,
`PAYMENT_DESCRIPTOR_SUFFIX_HOTEL`, `LITEAPI_PMC_ID`,
`LITEAPI_HOTEL_PAYMENT_OPTS`. Setting any of the first three re-enables the
prebook `payment` block — see §5 before you do.

Nuitee dashboard, the two pages worth knowing:
`connect.nuitee.com/flights/` (carrier + status per booking) and
`connect.nuitee.com/request-logs/` (raw request/response, plus the wrench).
