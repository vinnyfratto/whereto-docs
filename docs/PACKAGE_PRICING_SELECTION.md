# How WhereTo Picks the Flight and Hotel for a Package Price

_Living doc. Last updated: 2026-09-08._

This describes the rules the app uses to pick **one flight** and **one hotel** out of
a destination's full search results, whenever it needs to show a single "package"
price. This is the same logic behind the destination card's "Explore Italy for
around $4,759" estimate, the WhereTo Pick badge in hotel/flight lists, and the
Package Details wizard's running total. The rules never change based on where they
run; only the pool of flights/hotels they run against can differ, which is what
makes the numbers match or diverge (see [Keeping the numbers consistent](#keeping-the-numbers-consistent)
below).

The picker always runs in this order: **flight first, then hotel.** The flight's
price becomes the budget anchor the hotel is picked against.

---

## 1. Flight selection

Code: `pickFlight()` in `src/lib/vibeMatch.ts`.

**The rule, in order of priority:**

1. Start at nonstop (0 stops). Check higher stop counts (1, 2, ...) only if nothing
   is available at the current one.
2. At the current stop count, if the traveller has preferred airlines set, take the
   **cheapest preferred-airline flight** at that stop count, if one exists. It wins
   immediately, full stop.
3. If no preferred-airline flight exists at that stop count, take the **cheapest
   flight from any airline** at that stop count instead.
4. If nothing exists at that stop count at all, move up to the next stop count and
   repeat from step 2.

**The one rule that overrides everything else: fewer stops always wins.** Price and
airline preference only ever compete against other options at the *same* stop
count. A cheaper flight with more stops never beats a pricier flight with fewer
stops, and a preferred airline never jumps a flight ahead of a lower stop count.

### Examples

Preferred airline: **Delta**.

| Flight | Stops | Airline | Price |
|---|---|---|---|
| A | 0 | Delta | $620 |
| B | 0 | JetBlue | $540 |
| C | 1 | Delta | $410 |
| D | 1 | American | $380 |
| E | 2 | Delta | $290 |

**Picked: A ($620 Delta nonstop).** Even though JetBlue is a cheaper nonstop and
Delta's own 1-stop is far cheaper, step 2 finds a preferred-airline flight at the
nonstop tier and takes it immediately. Price never gets a chance to compare across
stop counts.

---

Preferred airline: **Delta**. This time Delta has no nonstop.

| Flight | Stops | Airline | Price |
|---|---|---|---|
| A | 0 | American | $500 |
| B | 1 | Delta | $350 |
| C | 1 | United | $310 |

**Picked: A ($500 American nonstop).** At the nonstop tier, no Delta option exists,
so step 3 falls back to the cheapest flight from any airline at that tier — American.
The 1-stop tier, including the cheaper Delta option, is never even considered: step 4
only advances to the next tier when a tier has **no candidate at all**, preferred or
otherwise.

---

Preferred airline: **Delta**. No nonstop flights exist for this route at all.

| Flight | Stops | Airline | Price |
|---|---|---|---|
| A | 1 | Delta | $410 |
| B | 1 | American | $380 |

**Picked: A ($410 Delta, 1 stop).** The nonstop tier is empty, so the walk advances
to 1 stop. There, Delta is available, so the preferred-airline rule applies again and
wins over the $30-cheaper American option.

---

## 2. Hotel selection

Code: `pickHotel()` in `src/lib/vibeMatch.ts`. Runs **after** the flight is picked,
using:

```
remaining budget = (budget-high x (1 + flex%)) - flight price
```

`flex%` is the traveller's own Profile price-flexibility setting (0%, 2%, 5%, or
10%) — this is the same number that makes the "in budget!" pill on a card agree with
what the traveller actually configured.

**The rule:**

1. Drop any hotel whose total is **more than the remaining budget**. Those are not
   eligible for the main pick.
2. Score every hotel that's left with a 50/50 blend of two things:
   - **Budget fit** — how close the price is to 80% of the remaining budget. A hotel
     priced at exactly 80% of what's left scores 1.0 on this half; a cheaper hotel
     scores less than 1.0 in proportion (it's "leaving money on the table" relative
     to the target); nothing scores above 1.0.
   - **Rating** — guest rating out of 10 (or star rating x2 if no guest rating is
     available), normalized to 0–1.
3. The hotel with the highest combined score wins.
4. **Fallback:** if no hotel fits under the remaining budget at all, the cheapest
   hotel overall is picked anyway, so the traveller sees a real number instead of
   "no hotel available." (This fallback can be turned off for a caller that needs a
   strict yes/no "does anything actually fit" signal — no current caller does this
   today.)

This is deliberately not "always pick the cheapest" and not "always pick the
nicest." It's "get close to using the budget well, then let rating break ties."

### Example

Remaining budget after the flight: **$1,200**. Target (80%) = **$960**.

| Hotel | Price | Rating | Budget-fit score | Rating score | Combined |
|---|---|---|---|---|---|
| A | $200 | 7.0 | 200 / 960 = 0.21 | 0.70 | 0.5×0.21 + 0.5×0.70 = **0.45** |
| B | $950 | 8.0 | 950 / 960 = 0.99 | 0.80 | 0.5×0.99 + 0.5×0.80 = **0.90** |
| C | $600 | 9.5 | 600 / 960 = 0.63 | 0.95 | 0.5×0.63 + 0.5×0.95 = **0.79** |
| D | $1,300 | 9.0 | — over the $1,200 remaining, dropped in step 1 | | |

**Picked: B ($950, rated 8.0).** It isn't the cheapest (A) or the best-rated (C or
D) — it's the one that uses the budget well *and* has a solid rating. C comes close
on the strength of its rating, but B's near-perfect budget fit edges it out.

### Fallback example

Remaining budget after the flight: **$150**. Every hotel in the pool costs $220 or
more.

**Picked: the cheapest one available ($220-ish), flagged as over budget**, rather
than showing no hotel at all. The card's "in budget?" flag is computed separately
from this pick, so a fallback pick correctly still shows as over budget even though
a hotel is displayed.

---

## Keeping the numbers consistent

Both rules above are deterministic: the *same* flight/hotel candidate pool run
through the *same* remaining-budget number always produces the *same* pick. Every
mismatch we've seen between two screens showing a "package price" for the same
destination has come from feeding the picker **different inputs**, not from the
rules themselves changing:

- **A different candidate pool.** The destination card's original estimate always
  searches hotels with no budget filter applied to the search itself (the picker's
  own step 1 is what applies the budget). A screen that instead asks the hotel
  search API to pre-filter by budget can silently exclude the exact hotel the
  estimate picked, if that hotel used the over-budget fallback.
- **A different remaining-budget number.** If one screen forgets the flex%
  adjustment (`budget-high x (1 + flex%)`) and another includes it, the two remaining
  amounts differ, which can flip which hotel scores highest even from the identical
  pool.
- **A stale vs. fresh flight price.** Live inventory changes between two separate
  flight searches. If a screen re-searches instead of reusing the flight already
  picked for the estimate, `flight price` in the remaining-budget formula can shift
  even though the picking rule itself is unchanged.
- **Reading the estimate out of the wrong field.** One pricing pass writes its
  totals to `dest.scoring` for a vibe-matched destination (Explore, Discover) and
  to `dest.priceBreakdown` for one that skipped vibe scoring (WhereTo: Direct).
  The totals are identical in shape and come from the same `pickFlight`/`pickHotel`
  run — but a screen that reads only `.scoring` sees *nothing* on Direct, which is
  worse than seeing something stale: `flight price` silently becomes 0, so the
  hotel gets the entire budget as its remaining share. Read both, via
  `packageEstimate(dest)` (`src/utils/packageEstimate.ts`).

**2026-08-21 fix:** the Package Details wizard (`PackageWizardSheet.tsx`) was doing
the first two of these — passing the budget straight into its hotel search (hard
pre-filter) and computing remaining budget without the flex% adjustment — which made
its sticky-tray total diverge from the destination card's estimate. Fixed by
searching hotels unfiltered (matching the estimate), matching the flex-adjusted
ceiling formula, and preferring the exact hotel the estimate used whenever it's
still present in the fresh results.

**2026-09-08 fix:** all three of those repairs read `dest.scoring`, so none of them
ever applied on **WhereTo: Direct**, whose cards carry `dest.priceBreakdown`
instead. On Direct the wizard therefore had no flight total to subtract (remaining
budget = the whole ceiling → a pricier hotel than the card quoted), no hotel id to
prefer, and no flight leg in its running total. The Suggested Trip popup's "Book
Now" had the same blind spot from the other end: it displayed the estimate's hotel
but staged only the flight. Fixed by routing every estimate read through
`packageEstimate(dest)`, which returns whichever field this destination has —
`app/search/results.tsx`, `PackageWizardSheet.tsx`, `FlightDetailSheet.tsx`, and
the `scoring` prop `HotelDetailSheet.tsx` receives. The vibe-only fields
(`display`/`components`/`coverage`/`vibes`) still read `dest.scoring` directly;
they are genuinely absent on Direct.

**2026-09-08, second pass:** `HotelDetailSheet.tsx` — the non-wizard "Change
Hotel"/"View Hotels" browse panel — was the last surface still doing all three of
the divergences listed above, on every workflow. Its search passed
`budgetHigh`/`budgetFlex` (hard pre-filter, at the WHOLE trip's ceiling applied to
the hotel alone), its remaining-budget used plain `budgetHigh` with no flex
adjustment, and it deliberately re-derived its own pick rather than honouring the
one the card already quoted. All three now match the pricing pass and the wizard,
including the same refundable guard on honouring the card's hotel. Its
`[PackagePrice]` trace was also mislabelled `WIZARD HOTEL STEP` — the wizard
stopped opening this sheet when `PackageWizardSheet` landed — and is now
`HOTEL BROWSE`.

---

## Tunable constants

These live in `src/lib/vibeMatch.ts` and can be adjusted without changing the rule
structure above:

| Constant | Value | Meaning |
|---|---|---|
| `HOTEL_BUDGET_FIT_WEIGHT` | 0.5 | Weight given to budget fit in the hotel score |
| `HOTEL_RATING_WEIGHT` | 0.5 | Weight given to rating in the hotel score |
| `HOTEL_BUDGET_FIT_TARGET` | 0.8 | The fraction of remaining budget that scores a full 1.0 on budget fit |

---

## Key files

| File | Role |
|---|---|
| `src/lib/vibeMatch.ts` | `pickFlight()` and `pickHotel()` — the rules themselves |
| `src/store/appStore.ts` | Runs the scoring pass that produces each destination card's estimate |
| `src/components/PackageWizardSheet.tsx` | Package Details wizard; re-derives the same pick for its sticky-tray total |
| `src/components/HotelDetailSheet.tsx` | Non-wizard hotel browsing; uses the same `pickHotel()` for its "WhereTo Pick" badge |
| `src/providers/hotels/LiteApiProvider.ts` | Hotel search provider; owns the budget hard-filter that must stay off for any screen feeding `pickHotel()` |
| `src/utils/packageEstimate.ts` | `packageEstimate()` — reads the card's totals from `scoring` or `priceBreakdown`, whichever the destination has |
