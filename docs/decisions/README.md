# Architecture Decision Records

An ADR captures **one** significant decision: the context, what we chose, and the
consequences we accepted. ADRs are the highest-value thing a small team can keep,
because anyone who maintains the system needs to know *why* it is the way it is, and that
reasoning is unrecoverable once it leaves the founder's head.

## Rules

- One decision per file. Number them in order: `NNNN-short-title.md`.
- Never rewrite a decided ADR. If a decision changes, write a new ADR and set the old
  one's status to "Superseded by ADR-XXXX".
- Keep them short. Context, Decision, Consequences. Half a page is plenty.
- Use [_template.md](_template.md).

## Index

| # | Decision | Status |
| --- | --- | --- |
| [0001](0001-supabase-backend.md) | Supabase as the single backend | Accepted |
| [0002](0002-provider-abstraction.md) | Provider abstraction for flights and hotels | Accepted |
| [0003](0003-server-side-travel-apis.md) | Third-party travel APIs run server-side in Edge Functions | Accepted |
| [0004](0004-rls-disabled-together.md) | RLS intentionally disabled on Wander Together tables | Accepted |
| [0005](0005-versioning.md) | Manual SemVer + versionCode on every change | Accepted |
| [0006](0006-skins-design-tokens.md) | Skins design-token system, no inline styles | Accepted |
| [0007](0007-posthog-sentry-deferred.md) | PostHog for analytics; error monitoring deferred | Accepted |
| [0008](0008-icon-codegen.md) | Icons via Solar/Iconify codegen | Accepted |

_These eight were backfilled on 2026-07-05 from git history and project memory. Going
forward, the weekly agent drafts a new ADR stub whenever it detects an architectural,
dependency, or security-relevant change._
