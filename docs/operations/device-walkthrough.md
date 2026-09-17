# Driving the app on a real device over adb

Written from an end-to-end pass on 2026-09-17: dashboard to a paid sandbox
booking of both legs (hotel `GSypVXZE3`, flight `FH-269-FCQHWOBS`), on a Pixel
over USB with Metro running. It exists so the next pass does not rediscover the
same coordinates, the same shell traps and the same sandbox behaviour.

This is a debugging tool, not a test suite. It reads the live screen, so it
tells you what the app actually did, which Sentry and tsc both miss.

## The driver

`scratchpad/nav.sh` (session scratch, recreate it when it is gone):

| Command | What it does |
|---|---|
| `nav.sh tap X Y [_ SEC]` | `input tap`, then sleeps `SEC` (default 3) |
| `nav.sh swipe X1 Y1 X2 Y2 [MS]` | `input swipe` |
| `nav.sh back [SEC]` | keyevent 4 |
| `nav.sh link URL [SEC]` | `am start -a VIEW -d URL` |
| `nav.sh dump LABEL` | `uiautomator dump`, pulls it, prints the texts and every clickable node's bounds |
| `nav.sh shot LABEL` | screencap, scaled to 260px wide via ffmpeg so it is cheap to read |

`dump` is the workhorse: text tells you where you are, bounds tell you where to
tap. Take a `shot` only when the question is visual (an overlay, a colour, a
spinner).

### Shell traps, both of which cost real time

- **`export MSYS_NO_PATHCONV=1`.** Git Bash rewrites `/sdcard/u.xml` into
  `C:/Files/Git/sdcard/u.xml` and the pull silently produces nothing. `nav.sh`
  sets it; any inline adb loop you write must set it too.
- **Bound every wait loop.** `for i in $(seq 1 12)` with a `break`, never
  `while true`. A poll on a screen that never changes otherwise hangs the turn.
- Backticks inside a double-quoted `node -e` are command-substituted by bash.
  Write the script to a file instead (same rule as the heredoc one).

## Coordinates that held all session (1080x2340, 3-button nav)

| Target | Point |
|---|---|
| Footer tabs | y 2135, x 143 / 424 / 713 / 937 |
| Hamburger | 970, 176 |
| Sticky CTA, single button | 820, 2086 |
| Sticky CTA, price left + action right | action at 816, 2103 |
| Dashboard tiles | Discover 290,975 · Explore 800,975 · Together 290,1665 · Direct 800,1665 |

Sticky bars sit over the scroll content, so a `dump` can report a tile's bounds
underneath the bar. Tap the CTA by its own bounds, not by where the list ends.

Deep links work and are faster than walking the UI: `whereto://bookings`,
`whereto://profile?section=friends` (see `DEEP_LINKS` in `app/profile.tsx`).

## The flows, as the screens actually appear

**Discover** — Step 1 location types (10 tiles, multi-select, Continue at the
bottom) → Step 2 trip basics (origin, Domestic/International/Both, travellers,
dates, budget) → Step 3 vibe shelves, each opening a picker modal with a
`Select Vibes` button → search → results.

**Results** — a card per destination with Vibe Score, package estimate and the
flight/hotel split, and three actions: `View why <city> scored...`,
`Discover <city>`, `Package Details`. Backfill keeps adding destinations after
the first paint (11 → 14 in one pass), so a dump early and a dump late disagree
and both are right.

**Package Details** → the wizard: Hotels (WhereTo Package Hotel above Other
Hotel Options, sorted Best Budget Fit) → hotel detail → `Select Hotel` →
Flights (same two-section shape) → outbound fare → return fare → staging.
A trip already staged interrupts with REPLACE / CONTINUE CURRENT TRIP / CANCEL.

**Staging** (`app/bookings/passengers.tsx`) — the one booking hub. Package
breakdown, a flight card and a hotel card, `Continue booking` which takes the
flight first. Tapping the hotel card opens the hotel checkout directly, which
is how you book the legs in the other order.

**Hotel checkout** — Your stay → Guests → Review → Book & Pay → LiteAPI
payment widget → confirmation.
**Flight checkout** — Travelers → Bags → Seats → Review → Book & Pay → same.

Both confirmations end in the next-step prompt: `Continue to <other leg>` in
amber, `Return Home` stroked in amber.

## Sandbox reality, which shapes every walkthrough

- **Staged trips go dead fast.** Anything staged more than a few minutes ago
  will not book. Flights come back `42004 offer expired`, and a refreshed offer
  can still fail `52099 failed to verify`. Hotels come back
  `That room is no longer available for these dates`.
- So: **search fresh, stage, and pay within the same few minutes.** Do not try
  to resurrect yesterday's staged trip; replace it.
- Sandbox flight data is not realistic (AUS→TLL in 6h 3m). Judge the flow, not
  the itinerary.
- The hotel leg is the reliable one. Book it first when you only need to reach
  a payment step.

## Reading what happened

Metro's client logs are the ground truth, at `.expo/dev/logs/start.log`:

```bash
tail -c 60000 .expo/dev/logs/start.log | grep -a client_log | tail -20
```

**That file also contains every `.env` value**, so never print it unfiltered
into a transcript. Filter, and keep the filter narrow.

The lines worth knowing:

| Line | Means |
|---|---|
| `[hotelBooking] prepare — <hotel> \| <in>→<out>` | checkout started |
| `[hotelBooking] quote+prebook attempt N/2 failed:` | the quote/prebook race, or the room is gone |
| `[hotelBooking] prebooked <id> \| <amount>` | rate locked, payment ready |
| `[payment] widget redirected to return URL — paid (sandbox)` | the card went through |
| `[hotelBooking] confirmed — <bookingId>` | booked |
| `[flightBooking] prepare / confirmed — <ref>` | same, flight side |
| `[liteapi-flights] verify FAILED <code>` | the offer is stale or the provider refused |
| `[checkout] <ctx> issue (<kind>):` | what the traveller was actually shown |

## Payment is always the traveller's own hands

At the payment step, stop and hand the phone back. Card details are never typed
by the agent, sandbox test cards included. Post the ask as a
🔴 **DO THIS**, wait, then pick the walkthrough back up at the confirmation
screen.

## What this pass found

- The confirmation modal printed dates a day early on both ends
  (`new Date('2026-10-18')` is UTC midnight, so Austin renders the 17th).
  Fixed in 0.3.940 with `tripDate()` in `src/utils/defaultDates.ts`.
- "This rate expired" dead-ended: OK re-quoted, the room was gone, the same
  dialog came back with no way out. Fixed in 0.3.941 — a fresh quote missing
  the room class is now `sold_out`, which opens the room picker.
- A flight confirmation showed no dates at all. Cause not yet established; the
  sheet reads `flight` live from staging, so anything that clears the row while
  the confirmation is on screen empties its recap. Open.
