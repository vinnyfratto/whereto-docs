# Servicing: cancellations, changes and support after booking

**Scope:** what happens to a flight or hotel booking after it is confirmed. Who does what between Nuitée and WhereTo, what the API and webhooks actually give us, and what the app does with it.

**Sources, read 2026-09-16:** `docs/flights-support-billing-model` (updated 2026-08-12), `docs/canceling-a-booking`, `docs/using-liteapi-webhooks`, `docs/faq`, `docs/whitelabel-booking-site`, and the API reference pages for flight cancel quote / cancel / services / extra charges and hotel cancel / amend / alternative-prebooks / rebook. Plus our signed Terms of Service §2 to §4 and real payloads captured in `liteapi_webhook_events`.

---

## 1. Who does what

### Flights

Nuitée's own words: "We execute, not just notify." Their 24/7 desk does the airline-side work and hands back a settled booking.

| Situation | Nuitée | WhereTo |
|---|---|---|
| Airline changes or cancels | Gets the airline's message (GDS queue, NDC order change, or its own polling for low-cost carriers), rebooks or reissues or refunds, makes at least three contact attempts, then accepts the airline's protected itinerary so the traveller stays ticketed. Sends the updated booking as a webhook plus an email to our ops inbox. | Keep our record true, tell the traveller what changed, and (Terms §4) help them get a refund if they decline a significant change. |
| Traveller cancels | Cancels with the airline and refunds as merchant of record. | Show the quote first, call cancel, handle "pending" (202), confirm to the traveller. |
| Traveller wants a different flight, date or name | Their desk does it. **There is no API for changing a flight.** | Take the request and relay it with the identifiers Nuitée needs. |
| Talking to the traveller | Only under **First Line** (Nuitée's desk is the traveller's contact). | Everything under **B2B-Relayed**, which is what our Terms §4 already describe (support@wheretotrips.com is the first stop). |
| Money for changes | Fare difference and change fee at cost. $25 flat per voluntary case, waived on involuntary changes. | Under B2B-Relayed this bills **our credit line**, which we do not have, and we would have to collect from the traveller, which Terms §2-3 say we never do. Nuitée can charge the traveller directly for servicing, "agreed at onboarding". |
| Final 24-48 hours | The airline takes operational control; Nuitée's emergency desk and hotline stay with the traveller. | Point the traveller to the right contact. |

Nuitée flight desks, by departure proximity:

| Situation | Address |
|---|---|
| 2+ days from departure | `flights@nuitee.com` |
| Same-day departure | `priorityflights@nuitee.com` |
| Live airport emergency | `flights.emergency@nuitee.com` plus hotline |
| In trip | `flights.inflight@nuitee.com` |

### Hotels

There is no hotel version of the flights support doc for API partners. What is documented:

| Situation | Nuitée / LiteAPI | WhereTo |
|---|---|---|
| Guest cancels | `PUT /bookings/{id}` applies the rate's policy on its own and refunds the original card. | Show the policy and deadline first, call cancel, show fee and refund. |
| Name or email | `PUT /bookings/{id}/amend` creates a **pending** request, lead guest only. | Submit, show pending. |
| Dates or guests | `POST /bookings/{id}/alternative-prebooks` then `POST /rates/rebook`; the original cancels itself. Non-refundable originals go to Nuitée's team by hand (202). **"Any price delta is settled out of band"**, and nothing says by whom. | Treated as a help request until Nuitée answers the price-difference question. |
| Hotel-side problems | Relocation, refund, compensation and check-in instructions arrive as webhooks. | Update our record, tell the guest. |
| Guest support | Documented only for Nuitée's hosted booking site ("Nuitée provides customer support for your guests by default"). | Us, per Terms §4, until Nuitée says otherwise. |

The hotel booking record has `cancelledBy`, `cancelledAt`, `amountRefunded`, `refundedAt`, `refundType`, `rebookFrom`. The flight booking record has none of these.

---

## 2. What we receive

**Webhooks** (all on one endpoint, `liteapi-webhook`). Real payloads, not just the docs:

- `flight.book.created` / `pending.confirmation` / `confirmed` / `failed` / `expired`: the whole booking object in `response`.
- `flight.book.cancelled`: `request: {bookingId}`, `response: {status, currency, bookingId, refund_type, refund_amount, cancellation_fee}`. **Nothing says who cancelled.** The first one we ever received (2026-09-06, sandbox) was for a booking nobody in the app cancelled.
- No event exists for a flight schedule change. Nuitée says the updated booking arrives "as a webhook event" without naming it. Experiences has `experience.book.updated.datetime`; flights have no equivalent.
- Hotel cancel webhooks only fire in production.

**Polling:** `GET /flights/bookings/{id}` and `GET /bookings/{id}` return current state only. There is no before/after.

---

## 3. What the app does

### Records (built 2026-09-16)

One path, `supabase/functions/_shared/bookingSync.ts`, used by all three ways news arrives:

| Trigger | Where |
|---|---|
| A booking event | `liteapi-webhook` re-reads the booking, then syncs. Nothing depends on which event name carries an update. |
| The traveller opens a booking | `liteapi-flights` / `liteapi-book` `retrieve` |
| Backstop | `trip-reminder-sweep` (every 15 min) re-checks what `due_booking_syncs()` picks: never-checked rows first, then daily more than 30 days out, 6-hourly within 30 days, 2-hourly within a week, every 30 minutes inside 48 hours. Stops a day after the trip ends. Runs before the reminders, so a cancelled flight never gets a "see you tomorrow". It answers pg_net at once (202) and works after the response, because pg_net gives up after 5 seconds; results are in the function logs. It stops starting checks after 60 seconds, and a booking that fails to read is still marked checked so it cannot block the queue. |

What a sync does:

- **Status** always follows LiteAPI. A final cancellation result wins over a GET that has not caught up.
- **Flight schedule change:** compares the current segments with `flight_orders.itinerary_snapshot` (the itinerary the traveller was last told about) using `_shared/bookingChanges.mjs`. A first sync only records the snapshot and never alerts, because an old row has no trustworthy "before". Retimes under 15 minutes are not emailed, and the snapshot is not rolled forward, so small shifts add up and get sent together. "Significant" (which turns on the refund-entitlement sentence) is only set for a move of 6 hours or more, 3 hours when every airport is in the US, a different departure or arrival airport, or an added connection. Departure and return columns follow the new times so reminders do too.
- **Hotel change:** a different `hotelId` or dates versus the row's own columns (written from the book response, so trustworthy from day one). A moved hotel clears the stored photo and address rather than keep the wrong ones.
- **Cancellation:** `cancel_requested_at` is stamped before the app asks LiteAPI to cancel. A cancellation without it came from the airline, the hotel or Nuitée, and gets the "you didn't cancel this" email.
- **Emails are claimed before sending** (a conditional update only one caller can win) and released if sending fails, so a webhook retry racing an on-open refresh cannot double-send.
- Before `20260916_booking_servicing.sql` runs, sync falls back to status only with no emails.

Tests: `npm run test:servicing`. A schedule change cannot be triggered in sandbox, so those tests are the only place detection runs before a real airline moves a real flight.

### Emails

`_shared/booking-emails.ts`, sent by `send-notification` with reply-to `support@wheretotrips.com` and a copy to ops.

| Type | When |
|---|---|
| `airline_change` | Material schedule change. Shows "Was" against the new departure and arrival. Never tells the traveller to call the airline (they cannot act on our `FH-` reference, and Terms §4 make us the first stop). Says what happens if they do nothing: the airline's new schedule stays. |
| `stay_change` | Moved hotel or new dates. |
| `cancellation` (`supplier`) | Cancelled by someone other than the traveller. Flights state the refund entitlement. |
| `cancellation` (`requested`) | The traveller cancelled: fee, refund, and airline credit details when the refund is a voucher. A `pending` variant for a flight cancel the airline has not confirmed. |

### Help with a booking (built 2026-09-16)

Every flight and hotel booking page has a **Manage booking** section: **Get help with this booking**, and **Change this flight** / **Change dates or guests**. Change opens the same help request with the topic already picked, because a flight change has no API and a hotel date change settles its price difference "out of band" with nobody named.

`app/bookings/help.tsx` sends `{kind, orderId, topic, message}` to the `booking-help` edge function, which:

1. checks the caller owns the booking and reads every identifier from the row, never the request;
2. builds the facts Nuitée needs (`_shared/helpContext.ts`): booking reference, airline PNR, LiteAPI bookingId, passengers, ticket numbers, each flight, total paid including extras, hours until departure; for hotels the hotel confirmation code, dates, rooms and rate;
3. picks the Nuitée desk by hours to departure (`flightRelayDesk`: more than 48h `flights@`, inside 48h `priorityflights@`, mid-trip `flights.inflight@`). Nuitée publishes no hotel desk, so hotel cases say to use the dashboard's Request Assistance;
4. saves the request to `booking_help_requests` (skipped, not fatal, until the migration runs);
5. emails **support@wheretotrips.com** (cc `BOOKING_HELP_CC`, reply-to the traveller) with the traveller's message, the facts, and a message ready to forward to that desk plus a one-tap mailto; and emails the traveller that we have it (reply-to support).

Limited to 6 requests per traveller per hour. The emails never include passport numbers or birthdays; Nuitée's desk already has those on its own record.

### Self-serve cancel (built 2026-09-16)

**Cancel this flight** / **Cancel this stay** on the booking page (hidden once the trip has started or the booking is no longer active) opens `app/bookings/cancel.tsx`, which shows the numbers before anything can be cancelled:

- **Flights:** `liteapi-flights` `cancelQuote` returns the airline's quote. The refund reads "Refund, up to" unless the airline marked the quote `confirmed`, says whether money comes back to the card or as airline credit (with the credit code and expiry), flags the void window, and says plainly when a ticket returns no money. Asked once per visit: every quote call reaches the airline.
- **Hotels:** there is no quote endpoint, so `liteapi-book` `cancelQuote` re-reads the booking and applies its cancellation policies (`hotelCancellationPreview`: the most recently passed policy's amount applies; nothing passed on a refundable rate is free). When the policies cannot settle it, the screen says so instead of guessing.
- On a flight + hotel trip it says which half stays booked.

One confirm, then `cancel`:

1. stamps `cancel_requested_at` first (the only way to tell a traveller's cancel from an airline's later);
2. calls LiteAPI (`POST .../cancellations` for flights, `PUT /bookings/{id}` for hotels);
3. **final** (200): syncs the row with the result, and the sync sends the confirmation email with the fee, refund and any airline credit (claimed, so the webhook that follows cannot send it again);
4. **pending** (flights, 202): records it, emails "we've asked the airline" once, and the booking page shows "Cancellation requested" until the final result arrives as a webhook or the sweep finds it;
5. **refused:** takes the stamp back, nothing changed, and offers help.

After a cancel, the booking page re-reads its live status (`liveRefreshToken` in the bookings store). The temporary `-prod` test twins refuse `cancel`.

---

## 4. Build status

| Piece | Status |
|---|---|
| Webhooks act on booking events | Built 2026-09-16 |
| Schedule change / stay change / cancellation emails | Built 2026-09-16 |
| Backstop sync in `trip-reminder-sweep` | Built 2026-09-16 |
| `trip-reminder-sweep` deployed with JWT verification off (cron was refused at the gateway, so no reminder had ever sent) | Fixed 2026-09-16 |
| "Get help with this booking" (and the change requests that ride on it) | Built 2026-09-16 |
| Self-serve cancel, flights and hotels | Built 2026-09-16. Not yet run end to end: it needs a signed-in traveller, so the first run is on a phone against a sandbox booking |
| Hotel name/email amend | Not started |
| Hotel date change | Waiting on Nuitée (who pays the price difference) |
| Flight change | No API. Always a help request. |

---

## 5. Open questions for Nuitée

1. Which webhook event carries an updated booking after a schedule change or reissue?
2. Which support model is our account on, and can Nuitée charge the traveller directly for servicing costs under B2B-Relayed?
3. What operations inbox do you have on file for the mirrored emails?
4. Does `flight.book.cancelled` fire for both partner and airline cancellations, and is there any field that tells them apart?
5. Does a cancellation through the API count as a $25 voluntary servicing case?
6. On a hotel rebook, who pays or receives the price difference?
7. Who handles guest support for hotels booked through the API?
8. Can a schedule change or involuntary cancellation be simulated in sandbox?
