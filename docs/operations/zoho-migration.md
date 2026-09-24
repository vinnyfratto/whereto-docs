# Zoho One migration map

Status: **planning**. Nothing is built. Written 2026-09-20, before the Zoho One
subscription exists, so every Zoho-side detail here is to be confirmed at signup.

The decision is made: where a Zoho app and a WhereTo feature do the same job, the
WhereTo feature retires. This document says *which* features, *what has to stay
regardless*, and the order to do it in.

---

## How we talk to Zoho at all

There is no Zoho connector for our tooling and no Zoho MCP server. Every
integration is the same shape:

- A Supabase edge function calls a Zoho REST API with an OAuth 2.0 bearer token.
- Zoho calls us back via a webhook, handled the way `liteapi-webhook` and
  `resend-webhook` already are.

Auth is a **Self Client** (server-to-server, no runtime user consent) created in
the Zoho API Console. Access tokens live one hour; we store the refresh token as
a Supabase secret and mint access tokens on demand, cached in `_shared/`.

Two things that bite people:

- **The accounts server is per-datacenter.** A client registered in the `.com`
  datacenter cannot be refreshed against `.eu`, and the error is unhelpful. Note
  which datacenter the Zoho One org is created in and pin it in config.
- **API calls are metered as credits on a rolling 24-hour window**, allocated per
  org by user count, not per app. A chatty integration in one Zoho app can starve
  another. Budget the sync, do not poll.

Vinny creates the Self Client and hands over client ID, secret and refresh token
as Supabase secrets. No credentials get typed by Claude.

---

## Per-feature verdicts

### Zoho Forms → replaces web3forms. Clean retire.

Today: website forms POST to web3forms for the notification email, with
`capture-submission` as a fire-and-forget side channel that writes a durable row
and sends a branded Resend notification.

After: forms submit to Zoho Forms. web3forms retires completely.

One decision: `capture-submission` also exists to give the admin pages something
to read. Either keep writing our row (Zoho Forms webhook → `capture-submission`,
which stops sending its own email) or let the admin pages read Zoho. Prefer the
former — it keeps the `intent` taxonomy the site already sets per section, and
admin tooling never gates an app change.

### Zoho Desk → replaces the support inbox. Retire the email, **keep the function**.

This is the one place where "retire the feature" is the wrong instinct, for a
reason that is not about overlap.

`supabase/functions/booking-help/index.ts` does not just send an email. It
proves the caller owns the booking and reads every Nuitée identifier **from the
database row, never from the request**, then works out which Nuitée desk the case
belongs to (for flights, by hours to departure). A traveller filling in a Zoho
Desk web form can assert any booking ID they like and none of that holds.

So: the app keeps calling `booking-help`. Inside it, the Resend support email is
replaced by a Zoho Desk ticket created over the API, with the identifiers and the
desk routing already resolved server-side. A Desk webhook writes status back to
`booking_help_requests`, and My Bookings can finally show the traveller where
their case stands — which is a real gain over an email thread.

Terms §4 names support@wheretotrips.com as the first stop for any booking
problem. That address must **route into Desk**, not be replaced by a different
one, or the signed Terms stop being true.

### Zoho CRM → the partner CRM. Do this one last, and deliberately.

`partner-crm` is 1321 lines against a written spec, and its header is a list of
hard rules: no TIN, SSN, EIN, bank account or routing number is accepted, stored,
logged or returned anywhere; the database rejects number-shaped strings in the
compliance notes as a second line of defence; no tracking code is ever minted
here; nothing is hard-deleted.

Those are not features that overlap with Zoho CRM. They are a compliance posture
that happens to be written in TypeScript and SQL. Zoho CRM will cheerfully store
a TIN in a custom field, and "nobody will add that field" is not the same
guarantee as "the database rejects it".

Moving is still possible — Zoho validation rules, Blueprint for the stage gates,
field-level permissions for the rest — but it is a re-implementation of the spec
in someone else's system, not a migration. It also does not solve the thing that
actually blocked this work, which was attribution.

Recommendation: migrate Forms and Desk first, live with them for a while, then
decide on CRM with the spec open.

### Zoho Campaigns → marketing email only. **Resend stays.**

Campaigns is for newsletters and marketing sends. It is explicitly not for
transactional mail. Zoho's transactional product is **ZeptoMail**, which is
pay-as-you-go and **not part of a Zoho One subscription** — so "we already pay
for it" does not apply here.

We have roughly 25 `send-*` functions on Resend, all transactional, all sharing
`_shared/`, and a change to that shared layer means redeploying every importer.
Swapping them to ZeptoMail costs that whole redeploy and buys nothing.

Verdict: Campaigns takes marketing lists. Resend keeps transactional. Do not
touch the `send-*` family.

---

## Legal

The signed Privacy Policy §4 lists processor **categories**, not named vendors,
so adding Zoho does not require re-signing it. Confirm the categories actually
cover helpdesk and form-capture before the first ticket is created, and record
Zoho in whatever vendor list we keep outside the policy.

---

## Order of work

1. Zoho One org created; note the datacenter. Self Client created; secrets set.
2. `_shared/zoho.ts` — token mint + cache + datacenter-pinned base URLs. One
   place, so the refresh logic never gets a second copy.
3. Zoho Forms live on the website; web3forms retired; `capture-submission`
   rewired to the Zoho webhook.
4. Zoho Desk: `booking-help` creates tickets instead of emailing; support@ routed
   into Desk; Desk webhook writes status back; My Bookings shows it.
5. CRM: separate decision, spec open, after 3 and 4 have run for a while.

Steps 3 and 4 are independent and can go in either order.
