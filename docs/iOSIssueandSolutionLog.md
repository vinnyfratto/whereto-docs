# iOS Issue and Solution Log

A running log of iOS-specific bugs in this app: what the symptom looked like, what
the actual cause turned out to be, and what fixed it. Kept separate from the general
bug history because these are platform quirks that will look like a mystery again
the next time they show up in a different screen, unless someone remembers this file
exists.

Add a new entry every time an iOS-only bug (something that does not reproduce on
Android with the same code) gets root-caused. Put the newest entry at the top.

---

## 2026-08-05 — Second native Modal silently fails to present on iOS

**Symptom:** Tapping a Hotel or Flight card in Hotel Results / Flight Results, or
using the Package Details wizard's cross-navigation ("I don't want a hotel" etc.),
did nothing on a real iPhone. No error, no crash. Full native touch feedback and all
JS logic ran correctly (search, detail fetch, room-join, everything logged as
expected) but the destination screen never appeared. Sometimes it looked like a
freeze, sometimes like the whole flow silently closed and dropped back to Destination
Results. Identical code worked correctly on Android.

**Root cause:** React Native's `<Modal>` component can silently refuse to present a
*second* native modal while a first one is still active or mid-dismiss on iOS.
Android's modal stacking is more forgiving and does not have this problem, which is
why the bug never showed up there.

This app nests modals in two places that hit this:
- `TravelDetailSheet` (the Hotel/Flight Results panel) wraps a `<Modal>`. Tapping a
  card opens `HotelDetailViewV2` (or a flight fare sheet), which *also* wraps its own
  separate `<Modal>`, rendered as a React sibling while the first modal is still open.
- The Package wizard's Hotel <-> Flight cross-nav closes one `TravelDetailSheet`
  modal and opens another in the same synchronous handler.

Confirmed definitively (not just suspected) via `Modal`'s own `onShow`/`onDismiss`
callbacks: `onShow` never fired for the second modal, across multiple attempts,
while the JS side kept reporting a normal, successful render. That is the tell for
this bug: everything upstream of the native presentation looks fine, because React
Native keeps a `Modal`'s children mounted and rendered even while `visible={false}` —
so any logging inside that content proves React built it, never that iOS displayed it.

**What did NOT fix it, and why it's worth remembering:**
- Staggering the two `visible` props with a `setTimeout` delay (close one, wait
  ~350ms, open the other) was not enough on its own. Toggling `visible={false}` on a
  `Modal` component that stays mounted does not reliably release iOS's native
  presentation state.
- Removing the second `<Modal>` entirely on iOS (rendering `HotelDetailViewV2` as a
  plain absolutely-positioned `View` instead) got the content to actually paint for
  the first time, but introduced a *new*, worse bug: an infinite render loop
  (renders every ~30-45ms, indefinitely) that eventually crashed the screen. Suspect
  cause: something about safe-area-inset or layout recalculation behaving
  differently for a plain `View` versus a `Modal`-wrapped view, though the exact
  mechanism was never isolated. Reverted.

**What fixed it:** Keep `<Modal>` on both platforms (do not remove it), but fully
**unmount** the first modal's component from the React tree, not just toggle its
`visible` prop, before mounting/showing the second one. Concretely, in
`HotelDetailSheet.tsx`:
- A `hotelListMounted` boolean wraps `<TravelDetailSheet>` in a conditional render
  (`{hotelListMounted && <TravelDetailSheet ... />}`), separate from its `visible`
  prop.
- Opening a hotel: `visible` flips false immediately (plays the normal ~320ms close
  animation), then ~420ms later `hotelListMounted` flips false (fully unmounts the
  `Modal` element) and the detail view's `visible` flips true in the same tick.
- Closing: symmetric in reverse. The detail view starts closing immediately;
  `TravelDetailSheet` remounts right away but stays hidden (its own `visible` prop is
  still false at that point) and only plays its entrance animation once the handoff
  delay clears.
- The same pattern (close one, wait, open the other — never both `visible` at once)
  was applied to the Hotel <-> Flight cross-nav handlers in `app/search/results.tsx`.

The ~420ms delay is not arbitrary: `TravelDetailSheet`'s own exit animation takes
~320ms (see its `setTimeout(() => setMounted(false), 320)`), so the handoff waits
slightly longer than that to make sure the outgoing modal is actually gone.

**Files touched:** `src/components/HotelDetailSheet.tsx`,
`src/components/TravelDetailSheet.tsx`, `src/components/HotelDetailViewV2.tsx`,
`app/search/results.tsx`. Landed in v0.3.354 (the crash-causing "remove Modal on
iOS" attempt was v0.3.353, reverted the same day).

**Update, same day:** Flight Results had the identical bug for the identical reason —
`FlightDetailSheet.tsx` chains three independently `<Modal>`-wrapped components
(`TravelDetailSheet` -> `FareSelectSheet` -> `FlightDetailView`), each closing one and
opening the next in the same tick. Applied the exact same fix (conditional mount +
~420ms staggered handoff) to both the List<->Fare and Fare<->Itinerary transitions.
Landed in v0.3.356. If a third screen turns up with this shape, it's worth just
grepping the codebase for every `<Modal` usage and checking each one's siblings up
front instead of waiting for it to get reported.

**How to recognize this bug again:** Any screen that opens *another* full-screen
`<Modal>`-based component from within an already-open `<Modal>`-based component,
where the JS logs all look correct but nothing shows up (or it silently drops back
to whatever's behind both), on iOS only. Check for two components each independently
wrapping `<Modal>` and being told `visible=true` close together in time. Add
`onShow`/`onDismiss` to the `Modal` in question first, before adding any other
logging — it is the only direct signal for whether iOS actually presented it.

**General rule going forward:** never let two `<Modal>`-wrapped components both be
`visible={true}` at the same instant. When navigating from one to another, fully
unmount the first (not just `visible=false`) and give its exit animation time to
finish before mounting/showing the second.
