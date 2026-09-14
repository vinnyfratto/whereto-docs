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
| Book | ✅ synthetic carrier always, real carrier **intermittently** | ✅ |
| Real inventory | ⚠️ **succeeds ~1 in 5** in sandbox (§3, corrected 2026-08-09) | ✅ |

**Nothing in our code is known to be blocking.** The open item on Nuitee's side
was believed to be a hard account entitlement gap (sandbox cannot ticket real
airlines at all); direct evidence now shows real-airline bookings DO succeed
in sandbox, just unreliably — see §3 for the corrected finding. Hotels book
real properties today.

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

## 3. The one blocker: real carriers, and it's intermittent, not absolute

`55099 "Failed to create booking"` on real airlines — but not every time.
Originally written up here as "no exceptions" from a 14-booking sample where
every real-carrier attempt happened to fail. **Corrected 2026-08-09**: four
United Airlines bookings on the identical ATL→EWR→NAP flight, same session,
within seven minutes — one hit PENDING_CONFIRMATION/Paid, three hit `55099`.
Fare and seat selection differed only because each retry re-quoted a fresh
offer; the succeeding attempt and one of the failures shared the exact same
fare. Full evidence table in `docs/liteapi/FLIGHTS.md` §10.

| Carrier | Result |
|---|---|
| Nuitée Air (synthetic) | CONFIRMED, every time |
| United Airlines | Mostly FAILED/Refunded, occasionally PENDING_CONFIRMATION/Paid |
| Air Canada | FAILED, Payment Refunded (no success observed yet) |

Payment state is irrelevant — failures happen with and without a completed
payment, Nuitée Air succeeds either way. The request body is byte-identical on
success and failure (compared side by side in Nuitee's request logs).

**Nuitee's own error assistant** (wrench icon on a log row) concluded, on the
original sample:

> "If you are already using the correct schema and it succeeds for Nuitée Air,
> the request body likely does not need changes. The correction is
> **environment / account enablement**, not JSON."

Treat that as a lead, not a verdict — the same assistant, asked about the
2026-08-09 failure specifically, invented a trailing-slash endpoint mismatch
and a wrong payment-method enum, both checked against source and both false.
It has no visibility into our actual request; only the dashboard's
reservation records do.

A support ticket is open. Its questions now: why does real-carrier ticketing
succeed intermittently rather than never — is this rate-limited, capacity-
limited, or a genuine per-attempt GDS flake — and does that rate improve on a
production key.

**Real-airline booking now HAS been observed to succeed in sandbox**, just not
reliably enough to validate a flow against. `classifyCheckoutError` already
treats `55099` as "provider couldn't complete it, get a fresh price and
retry" (not a dead end), which turns out to be the right response for the
right reason after all. Plan the go-live schedule around the success RATE
improving in production, not around it being entirely unavailable pre-launch.

---

## 4. Production checklist

> ### 2026-09-08 — production key verified; flights still blocked
>
> Ran `npm run check:liteapi-key` against the **Production – Private API Key**
> (`prod_05…e3dc`). Result:
>
> | Probe | Result |
> |---|---|
> | `GET /data/countries`, `/data/currencies` | 200 |
> | `POST /hotels/rates` | **200, real rates** |
> | `POST /flights/rates` | **403 `code=40301`** "user does not have enough permissions" |
> | `POST /rates/prebook` (book host) | 400 `4002` — auth passed, invalid offer as designed |
>
> **The key is fine.** The earlier "production key is 401 on everything" reading
> (ticket LAS-2081) was the `prod_public_` HMAC key being sent as `X-API-Key` —
> see `docs/liteapi/PLATFORM.md` §2. That ticket is resolved and is *not* the
> flights blocker.
>
> **Flights production entitlement is unchanged since 2026-08-22** and needs a
> new ticket. Only Nuitee can clear it.
>
> **Do not split the secret to ship hotels-only without also disabling flights
> in the app.** All four edge functions read one `LITEAPI_KEY`, and `LIVE_MODE`
> (`!key.startsWith('sand')`) drives the payment widget's `publicKey` per
> function (`LiteApiPaymentView.tsx:72`). A prod-hotels / sandbox-flights split
> is technically a one-line change, but a **package is two independent
> transactions** — `flight_status` and `hotel_status` settle separately through
> their own prebook + payment views (`app/bookings/passengers.tsx:318,407`)
> while the traveller is shown one combined `packageTotal` (`:501`). The split
> therefore sells a package that charges real money for the hotel leg and
> settles the flight leg against sandbox. Shipping hotels alone is a product
> decision (hide flights + packages behind a flag), not a config flip.
>
> **Decision 2026-09-08: stay fully sandbox, including through the 2026-09-21
> soft launch.** `LITEAPI_KEY` is untouched (last written 2026-07-14, sandbox)
> and **must not be pointed at production until flights `40301` clears.** Both
> products keep working end to end and book nothing real. Revisit when Nuitee
> answers the flights ticket — the production key itself is already proven, so
> that day is a secret swap plus a redeploy of the four `liteapi-*` functions,
> not an investigation.

### Must do

1. **Get flight booking enabled on a production key.** The open ticket. Longest
   lead time; start everything else in parallel. **Re-opened 2026-09-08** — see
   the box above; LAS-2081 was a different problem and closing it changed
   nothing here.
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
