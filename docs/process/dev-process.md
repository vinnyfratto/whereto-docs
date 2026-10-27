# Development Process

_Living doc. How work actually gets done on WhereTo. Last updated: 2026-07-05._

## Team and workflow

- **Single maintainer** (`thinvin` / vinnyfratto). This is the biggest structural risk
  (bus factor of one; see GAP-REPORT G-14). The purpose of this whole `docs/`
  folder is to make the system legible to a second engineer.
- **Branching:** trunk-based on `master`. Commits go straight to `master`. No pull
  requests, no feature branches today.
- **AI-assisted:** development is driven with Claude Code, with a Supabase MCP connected
  (`.mcp.json`) and project rules in `AGENTS.md` / `CLAUDE.md`.

## Commit conventions

Commits are free-form imperative subjects (for example "Fix airport search regression",
"Add US Southwest sub-region"). There is no strict Conventional Commits prefix. The
weekly agent classifies commits by verb and content into features, fixes, and chores, so
keep the first word meaningful (Add / Fix / Update / Improve / Remove / Refactor).

## Versioning

Hard rule: increment `version` and `versionCode` in `app.json` on **every** change.
Scheme: `0.0.x` dev, `0.x.0` beta, `1.0.0` go-live. See
[ADR-0005](../decisions/0005-versioning.md).

## Definition of done

A change is done when:

1. It works on a physical device (the user tests on device; do not rely on the web
   preview server).
2. `version` and `versionCode` are bumped.
3. Any DB change ships with the exact runnable SQL block.
4. Style values come from `src/Skins/`, never inline.
5. Scrollable tab screens clear the footer (see `AGENTS.md`).

## Coding standards

- TypeScript throughout. `expo lint` (ESLint) is available (`npm run lint`) but not
  enforced by CI.
- Design tokens only, via Skins ([ADR-0006](../decisions/0006-skins-design-tokens.md)).
- Read the exact versioned Expo SDK 56 docs before writing framework code (`AGENTS.md`).

## Testing

No automated tests today (GAP-REPORT G-03). Verification is manual on device. First test
targets when a suite is started: checkout, auth, and the provider layer, because those
carry the most money and risk.

## Documentation cadence

- **Per change:** changelog entry, version bump, ADR if the change is architectural.
- **Weekly:** the weekly agent produces the week-in-review and refreshes living docs (see
  [../automation/weekly-review-agent.md](../automation/weekly-review-agent.md)).
- **Quarterly:** re-run the [GAP-REPORT.md](../GAP-REPORT.md) audit.
