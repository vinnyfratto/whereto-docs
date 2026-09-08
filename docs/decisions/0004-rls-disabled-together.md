# ADR-0004: RLS intentionally disabled on Wander Together tables

- **Status:** Accepted
- **Date:** 2026-07-05 (backfilled)
- **Deciders:** thinvin

## Context

Row-Level Security is enabled across most tables. On the Wander Together tables
(`trip_participants` and related), enabling RLS through the Supabase dashboard toggle
causes **infinite recursion** in the policy evaluation, because the participant policy
needs to read the participant table to decide access. This made the group features
unusable with RLS on.

## Decision

Leave RLS **disabled** on the Wander Together tables for now, as a deliberate, documented
choice. Access to group data is mediated through Edge Functions that perform the
membership checks in application code (`group-checkout`, `validate-invite`,
notification functions), rather than relying on database policies.

## Consequences

- Group features work today.
- The compensating control is application-level: any code path that reads or writes group
  data must enforce membership itself. This is a real obligation and a review focus.
- This must be presented to any auditor or reviewer as a **decision with mitigations**, not
  discovered as an open table, or it reads as a breach risk.
- Tracked in [risks.md](../risks.md) R-07: the proper fix is a non-recursive policy
  (for example a `SECURITY DEFINER` helper function) so RLS can be re-enabled.
