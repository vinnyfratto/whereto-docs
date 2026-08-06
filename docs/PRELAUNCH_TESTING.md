# Pre-Launch Testing Plan

_Last updated: 2026-07-08. Closes GAP-REPORT item G-03 (no automated tests)._

## Goal

Verify the core user workflows work before the first public release. Cover the
highest-risk paths first (booking/checkout and auth), then the rest. This is the
concrete plan behind GAP-REPORT G-03.

## Approach: two layers

We build two testing layers that fit how this app is actually developed: a native app,
tested on an iPhone, from a Windows dev machine, with no web preview server in the loop.

### Layer 1: Logic harness (tsx scripts)

Small runnable scripts that call the real workflow code with mock inputs and assert the
outputs. Runs headless with `npx tsx`, no device or server needed, so it can run on
every change and feed into CI later. `tsx` is already a dev dependency.

Covers the logic-heavy workflows:

- Booking wizard: offer to passenger payload to order request shape, plus the 24h hold TTL.
- Hotel booking: `src/utils/hotelBooking.ts` request build and normalization for all surfaces.
- Vibe scoring and match score: inputs to ranked output.
- Provider adapters: flights and hotels normalize vendor responses correctly.

Who runs it: Claude, headless. Fast to extend, cheap to run.

### Layer 2: In-app QA screen

A hidden Dev/QA screen that runs each workflow with seed data and shows pass or fail plus
logs on the device. Extends the existing temporary "Test Walkthrough" button pattern from
onboarding (v0.3.53).

Covers the UI and native flows that Layer 1 cannot reach (navigation, SecureStore session,
push, image picker):

- Onboarding walkthrough (5-step).
- Auth: Google sign-in plus SecureStore session save (the 2048-byte chunking path).
- Discover walkthrough and Direct search, end to end.
- Booking wizard and hotel booking screens.
- Saved, Wander Together (lobby, invite, roles, vibes, voting), notifications.

Who runs it: founder, on the iPhone. Results are read back from the screen, or logged to
console or Supabase for Claude to read.

## Deferred approaches

Kept in mind, not built now:

- **Expo Web plus browser automation.** Could drive the real screens headlessly, but
  native-only modules (SecureStore, push, image picker) break on web, and running the web
  dev server clogs the on-phone workflow. Revisit only if a web build becomes a real target.
- **Maestro end-to-end.** Free and open source (only the optional hosted cloud costs
  money), but the Windows dev machine has no iOS simulator, so it would only exercise an
  Android build, and it needs a heavy Android emulator. Revisit when iOS is built on a Mac,
  or an Android emulator is set up.

## Workflow coverage

| Workflow | Primary layer | Notes |
| --- | --- | --- |
| Booking wizard, order, 24h TTL | Logic + QA screen | Highest risk. Do first. |
| Auth: Google plus SecureStore session | QA screen | Highest risk. Do first. |
| Hotel search and booking (LiteAPI) | Logic + QA screen | `hotelBooking.ts` |
| Flight search (Duffel) | Logic | Provider normalization |
| Vibe scoring and match score | Logic | |
| Onboarding walkthrough (5-step) | QA screen | |
| Discover walkthrough | QA screen + logic | |
| Direct search | QA screen + logic | |
| Saved destinations | QA screen | Persistence |
| Wander Together (groups) | QA screen + logic | Roles, per-member vibes, voting |
| Notifications and email | Logic | send-notification edge fn |
| Affiliate attribution | Logic | |

## Sequencing

1. Stand up the Layer 1 tsx harness and cover booking, hotel booking, TTL, and vibe scoring.
2. Build the Layer 2 QA screen and cover auth and the UI flows, checkout first.
3. Exit criteria for launch: every workflow above has at least one green pass at its primary
   layer, with booking/checkout and auth passing first.

_Not started. Tracked in the go-live checklist._
