# Email System — Master Reference

Everything about how WhereTo sends email: architecture, current inventory, the
bounce/complaint pipeline, conventions for building the next one, and the known
backlog. Last updated 2026-07-30. **Paste this whole doc into a fresh thread to pick
up email work with zero re-derivation** — it's written to be self-contained.

---

## 1. Two separate systems — don't confuse them

1. **Resend Templates** — our own Resend account, our own edge functions call the
   Resend API directly. Full design control, `{{{VARIABLE}}}` placeholder syntax,
   dashboard-managed at [resend.com](https://resend.com) → Templates (alias-based —
   the alias string is what the edge function references, e.g. `"template": {"id":
   "welcome-email", "variables": {...}}`). This is what `send-notification` and
   `welcome-email` use.
2. **Supabase Auth email templates** — a *completely different* system. Supabase's
   own Auth server sends these automatically whenever `supabase.auth.signUp()`,
   `.resetPasswordForEmail()`, OAuth linking, etc. run. Resend and our API key are
   never in that path at all. Managed at **Supabase Dashboard → Authentication →
   Emails** (two tabs: "Templates" for the classic auth flows, "Security" for the
   newer account-change notifications). Uses Go template syntax
   (`{{ .ConfirmationURL }}`, `{{ .Email }}`, `{{ .Token }}`, `{{ .NewEmail }}`),
   **not** `{{{...}}}`.

Both render as edge-to-edge HTML tables — same visual design system, entirely
different plumbing to get content into them.

## 2. File organization convention

All template source HTML lives in [`email-templates/`](../email-templates/), organized
by wiring status — **keep using this for anything new**:

- `email-templates/` (root) — new/not-yet-wired-up templates land here first
- `email-templates/_Emails Added/_Resend/` — confirmed created + **published** in
  the Resend dashboard
- `email-templates/_Emails Added/_Supabase/` — confirmed pasted into the Supabase
  Dashboard + saved

Move a file from root into the matching subfolder only after it's actually live in
the respective dashboard, not when the code referencing it is written.

## 3. Current inventory

### Resend Templates

| Template | Alias | File | Status |
|---|---|---|---|
| Booking confirmation (flight only) | `booking-confirmation-flight` | `_Emails Added/_Resend/booking-confirmation-flight.html` | ✅ Live |
| Booking confirmation (hotel only) | `booking-confirmation-hotel` | `_Emails Added/_Resend/booking-confirmation-hotel.html` | ✅ Live |
| Booking confirmation (both) | `booking-confirmation-both` | `_Emails Added/_Resend/booking-confirmation-both.html` | ✅ Live — verified against a real sandbox booking end to end |
| Welcome / account created | `welcome-email` | `_Emails Added/_Resend/welcome.html` | ✅ Live |
| Share a destination | `share-destination` | `_Emails Added/_Resend/share-destination.html` | ✅ Published in Resend — **not yet end-to-end tested** (see below) |
| Share a hotel | `share-hotel` | `_Emails Added/_Resend/share-hotel.html` | ✅ Published in Resend — **not yet end-to-end tested** (see below) |

Both share templates include the required signup CTA ("New to WhereTo?" card +
Sign Up Free button) and Google Play / App Store badges — Google Play links out
for real, Apple has no `<a>` wrapper at all (no click action) since the app isn't
published there and `app.json` has no `bundleIdentifier` configured yet. Both
badges are hand-built with CSS/table markup rather than the official Apple/Google
artwork, since there's no hosted copy of those images on our own domain to
reference reliably — swap in the real ones once available.

**Send path (built 2026-07-30):** `supabase/functions/share-email` — accepts
`{type: 'destination'|'hotel', to, message?, destination?, hotel?}`, requires a
real signed-in user JWT (rejects anonymous calls — verified: anon-key-only
request correctly 401s with "Sign in to share by email"), resolves the sender's
name server-side from `profiles`, picks the alias, sends via Resend. Both
templates are published in Resend as of 2026-07-30 (aliases confirmed matching
the code: `share-destination` / `share-hotel`), but the actual signed-in send
path is **still untested** — I can't simulate a real user JWT from here without
handling credentials, so this needs an actual in-app test: sign in, tap ✉️ on a
destination or hotel, send to yourself, confirm it lands.

Wired into all 4 in-app "Share" locations found on 2026-07-30 (added a ✉️
button/link next to each existing native-OS-share button — the native share is
untouched, this is additive):
- `src/components/TravelDetailSheet.tsx` — generic destination hero sheet
  (shared by Hotel/Flight/Car results panels)
- `app/search/vibe-explore.tsx` — destination vibe blog hero
- `app/search/destination-explore.tsx` — destination blog hero (2 buttons: header + bottom CTA)
- `src/components/HotelDetailViewV2.tsx` — hotel detail hero (2 buttons: sticky bar + hero actions)

All 4 use the shared `src/components/ShareEmailSheet.tsx` component (email +
optional personal-note input, prompts sign-in if not authenticated, calls
`share-email`). None currently pass a real destination/hotel deep link (`url`
field) — no public per-destination or per-hotel web page exists yet, so it
falls back server-side to `APP_URL` (`https://wheretotrips.com`). Add a real
link once such a page exists.

Sent from `supabase/functions/send-notification` (bookings — picks flight/hotel/both
alias based on which data is present in the order) and `supabase/functions/welcome-email`.
Both read `RESEND_API_KEY` / `FROM_EMAIL` from Supabase secrets — already configured,
`wheretotrips.com` domain verified (DKIM/SPF).

### Supabase Auth templates — Authentication tab

All in `_Emails Added/_Supabase/`, all pasted in + saved.

| Template | File | App flow status |
|---|---|---|
| Confirm signup | `auth-confirm-signup.html` | ✅ Live (`signUp()`, email confirmation on) |
| Reset password | `auth-reset-password.html` | ✅ Live (`resetPasswordForEmail()`) |
| Invite user | `auth-invite-user.html` | ⚪ No app flow exists |
| Magic link or OTP | `auth-magic-link-otp.html` | ⚪ No app flow exists (email/password only) |
| Change email address | `auth-change-email.html` | ⚪ No email-change UI in profile.tsx |
| Reauthentication | `auth-reauthentication.html` | ⚪ Not enabled |

### Supabase Auth templates — Security tab

All in `_Emails Added/_Supabase/`, all pasted in, all toggles switched ON.

| Template | File | App flow status |
|---|---|---|
| Password changed | `auth-password-changed.html` | ✅ Live (`updateUser({password})` in authStore.ts) |
| Sign-in method linked | `auth-signin-method-linked.html` | ✅ Live (Google/Facebook OAuth linking in profile.tsx) |
| Email address changed | `auth-email-changed.html` | ⚪ No email-change UI exists |
| Phone number changed | `auth-phone-changed.html` | ⚪ No phone auth exists |
| Sign-in method removed | `auth-signin-method-removed.html` | ⚪ No unlink-identity UI exists |
| MFA method added | `auth-mfa-added.html` | ⚪ No MFA enrollment exists |
| MFA method removed | `auth-mfa-removed.html` | ⚪ No MFA enrollment exists |

"App flow status" is a separate axis from dashboard status — it tells you whether
anything in the app actually *triggers* that email today, independent of whether the
template itself is wired up. Several are wired but currently inert, waiting on a
feature (email change, MFA, etc.) that doesn't exist yet — that's intentional
future-proofing, not a bug.

## 4. Bounce / complaint pipeline (done 2026-07-30)

`supabase/functions/resend-webhook` — receives Resend's `email.bounced` and
`email.complained` webhook events, verifies the Svix signature
(`RESEND_WEBHOOK_SECRET` secret), logs every event to `public.email_delivery_events`
(RLS-locked, service-role only — schema in `supabase/email_delivery_events.sql`),
and fires a best-effort alert email to `ALERT_EMAILS` (defaults to
`vfratto@vcinnovationsgroup.com,ccupero@vcinnovationsgroup.com`, override via
Supabase secret, comma-separated, no redeploy needed).

Verified end to end twice against Resend's `bounced@resend.dev` /
`complained@resend.dev` test addresses — DB rows and alert emails both confirmed.

**Not yet built**: nothing reads `email_delivery_events` to auto-flag a user's
address as bad (e.g. on `profiles`) or suppress future sends to it. Pure
observability today. Natural next layer if it's ever wanted.

## 5. Known backlog — other emails still on the old unbranded design

Found while auditing 2026-07-30, not yet touched. These still use the pre-reskin
`#1C3649`/`#D44C35` inline HTML (same state `send-notification` and `welcome-email`
were in before this work started) — same migration playbook as those two applies:

| Function | Sends | Notes |
|---|---|---|
| `together-notify` | Group booking invite / confirmed / failed | 3 templates in one file |
| `friend-notify` | Friend request sent / accepted | 2 templates in one file |
| `nudge-participant` | Organizer nudges a non-committed participant | 1 template |
| `send-curation-email` | Internal admin tool notification (image curation) | Lower priority — not user-facing |

When picking one of these up: same pattern as `send-notification` — build the
Resend Template HTML (edge-to-edge, design system below), get the alias created +
published in Resend, then swap the function's inline `fetch()` call to
`template: {id, variables}` instead of `html:`. See
`supabase/functions/send-notification/index.ts`'s `sendTemplateEmail()` for the
reference implementation of that swap.

## 6. Two things blocking full production-readiness (see also `GO_LIVE_LITEAPI.md` §4)

### Free-tier template lock
Supabase blocks Auth email template edits on **free-tier projects created after
2026-06-03**, unless custom SMTP is configured. Unknown whether this project falls
before or after that date.

### Still on Supabase's built-in email sender
Authentication → Emails shows a permanent warning: *"You're using the built-in
email service. This service has rate limits and is not meant to be used for
production apps."* Will start silently dropping/delaying signup and password-reset
emails once real user volume hits it.

**Fix for both, in one move:** point Supabase's Auth SMTP settings at Resend's SMTP
relay (`smtp.resend.com`) — Resend is already fully set up (verified domain,
working API key), so this is mostly just entering Resend's SMTP credentials into
Supabase Dashboard → Authentication → Emails → SMTP Settings. Not done as of
2026-07-30 — single highest-leverage remaining item on this whole list.

## 7. Design system reference — use this for every new template

- **Colors** (from `src/Skins/WanderSkin.ts`): charcoal `#313131` (header/footer),
  azure `#209CE0` (accent/CTA/links), cream `#F2F5F7`, success green `#3DA96E` /
  `#E6F5EC`, error red `#DE4C4C` / `#FBE9E9`, muted text `#8A9098`, divider
  `#E2E7EB`
- **Fonts**: `Georgia, 'Times New Roman', serif` for headlines (approximates the
  app's RCL Morland serif — real web fonts don't render reliably in email clients),
  `Helvetica, Arial, sans-serif` for body copy
- **Layout**: edge-to-edge, no card/box (`background:#FFFFFF` all the way, not a
  cream page background with a boxed white card — that was tried and rejected),
  `max-width:600px`, table-based markup (`<table role="presentation">`, required
  for Outlook/Gmail rendering), white logo
  `https://wheretotrips.com/assets/whereto-logo-white.svg` in a charcoal header bar
- **Structure every template follows**: hidden preheader `<div>` (preview text),
  icon badge in a soft-tint circle, serif headline with an italic azure accent on
  the last word/period, body copy in `#5C616A`, charcoal footer with the
  wheretotrips.com link and a one-line "why you're receiving this"
- **CTA buttons**: azure `#209CE0` pill (`border-radius:999px`), white bold text,
  `14px 36px` padding
- **Detail cards** (booking specifics, code displays): white background,
  `1px solid #E2E7EB` border, `12px` radius, rows separated by top-borders not
  full dividers

## 8. Quick recipe for the next Resend Template

1. Copy the closest existing file in `_Emails Added/_Resend/` as a starting point
2. Swap in `{{{VARIABLE}}}` placeholders, write the top-comment block (alias to set,
   suggested subject, variable list) — see any existing file for the format
3. Save the new file in `email-templates/` (root, not yet in `_Emails Added/`)
4. Write/update the edge function to call it via `template: {id: "<alias>",
   variables: {...}}` instead of `html:` — pass `subject` explicitly in the send
   call too (Resend requires it if the template has no default subject set)
5. Tell the user: import the file into Resend (Templates → Create → Import), set
   the alias exactly, set a subject, define the variables, **publish**
6. Once they confirm it's published, move the file into `_Emails Added/_Resend/`
   and update this doc's inventory table

## 9. Quick recipe for the next Supabase Auth template

1. Copy the closest existing file in `_Emails Added/_Supabase/`
2. Use only Supabase's own variables (`{{ .Email }}`, `{{ .ConfirmationURL }}`,
   `{{ .Token }}`, `{{ .NewEmail }}`, `{{ .SiteURL }}`) — never `{{{...}}}`
3. Save in `email-templates/` root, note whether a matching app flow exists yet
4. Tell the user: Supabase Dashboard → Authentication → Emails → find the row
   (toggle ON first if it's a Security-tab row) → paste HTML → set subject → Save
5. Once confirmed, move the file into `_Emails Added/_Supabase/` and update this
   doc's inventory table
