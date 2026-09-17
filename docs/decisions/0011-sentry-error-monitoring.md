# ADR-0011: Sentry for crash and error monitoring

- **Status:** Accepted
- **Date:** 2026-09-14
- **Deciders:** thinvin
- **Supersedes:** the error monitoring half of [ADR-0007](0007-posthog-sentry-deferred.md)

## Context

ADR-0007 deferred error monitoring. On 2026-08-10 PostHog's exception autocapture was
switched on to stand in for it, but it only ever caught JavaScript errors nothing
handled. It could not see native crashes, iOS app hangs or Android ANRs, errors the app
catches and carries on from, or anything inside an Edge Function. PostHog also has an
open bug where a fatal JavaScript crash can be lost before its report is saved.

The gap cost us: on 2026-09-09 a native crash in expo-notifications boot-looped the iOS
dev build for two days, and finding it took a hand-built boot beacon writing to a
Supabase table.

## Decision

Adopt Sentry on both sides, ahead of the next beta round.

- **App** (Sentry project `whereto-app`): `@sentry/react-native` ~7.11.0, the version
  Expo SDK 56 pins. It starts from a custom entry (`index.js`) before
  `expo-router/entry`, so a crash during the launch itself is reported. Native crashes,
  uncaught JS errors, unhandled rejections, iOS app hangs and Android ANRs report on
  their own. Checkout issues shown to a traveller as an error, and missing payment
  sessions, report explicitly. A production `warn()` becomes a breadcrumb. Source maps
  and debug symbols upload during EAS Build through the Expo config plugin.
- **Edge Functions** (Sentry project `whereto-edge`): `npm:@sentry/deno` 8.55.2 behind
  `supabase/functions/_shared/sentry.ts`, used by the five `liteapi-*` functions. It
  reports unhandled throws, refused bookings, confirmed bookings whose order row did
  not save, provider outages, and supplier-side failure webhooks.
- PostHog's exception autocapture is removed from the SDK config. PostHog stays for
  analytics.

## Consequences

- Crashes, hangs and failed bookings reach us without waiting for a traveller to report
  them.
- Traveller details are scrubbed before a report leaves the phone or the function: keys
  like passport, birth date, email, phone and names are redacted, and emails and long
  numbers are masked in text. The Privacy Statement already covers crash logs (§2) and
  analytics service providers (§4), so no legal change was needed.
- Every EAS build profile needs `SENTRY_AUTH_TOKEN` on the EAS environment it reads
  (development, preview, production). Without it an iOS release build fails at the
  upload step.
- Free Developer plan: 5,000 errors a month and one login. A second person means the
  Team plan.
- Catching a crash before any JavaScript runs (`useNativeInit`) needs Sentry React
  Native 8.x, which Expo SDK 56 does not pin. Revisit when the Expo SDK moves.
- Performance tracing and session replay are off for now. Edge Functions other than
  `liteapi-*` are not instrumented yet.
