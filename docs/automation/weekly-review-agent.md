# Weekly Review Agent

This is the spec the weekly agent follows. It runs every Friday. Its job is to keep the
engineering record current with almost no founder effort.
The agent proposes; the founder approves the ADRs and the STATE delta.

## Inputs

1. Run the deterministic collector first:
   ```
   node scripts/weekly-report-data.mjs
   ```
   It emits a markdown "data packet": commits by type, version delta, dependency changes,
   schema and Edge Function changes, and ADR/regression signals. Use `--days N` or
   `--since YYYY-MM-DD` to change the window.
2. The current living docs in `docs/` (to update them).

## Steps and outputs

### 1. Write the week-in-review
Create `week-in-review/YYYY-Www.md` from
[templates/weekly-review-template.md](../templates/weekly-review-template.md). Fill:
- **Development:** features shipped (from `feat` commits), notable changes (`refactor`,
  `chore`), and the by-the-numbers line (commits, files, version delta).
- **Bug fixes:** each `fix` commit as a line. Where the root cause is knowable from the
  diff or commit, add a one-line "cause:" note. This section is the raw material for
  incident history.
- **Regressions, debt added, rolled back:** anything reverted, rolled back, or shipped as
  a known shortcut. Never skip this section. If nothing, write "none". Honest regress
  tracking is the point.

### 2. Append the changelog
Add the week's user-facing changes to `CHANGELOG.md` under the current version, using
Keep a Changelog sections (Added / Changed / Fixed / Removed). At a version bump, close
the version heading and start a new `[Unreleased]`.

### 3. Update the tech stack if dependencies changed
If the data packet shows added or changed dependencies, update
[../tech-stack.md](../tech-stack.md). For any new dependency that is copyleft (GPL/LGPL/
AGPL), Creative Commons, or carries an attribution clause, add a row to the license
inventory. Run `npx license-checker --production --json` when in doubt.

### 4. Refresh STATE.md
Update [../STATE.md](../STATE.md): version, "what works today", and any number that
moved. This is the one-page system snapshot; keep it honest and current.

### 5. Update the risk register
Add any new debt to [../risks.md](../risks.md). Mark items resolved with the date rather
than deleting them.

### 6. Draft ADR stubs for real decisions
When the data packet flags a possible ADR trigger (a new dependency, a new or changed
Edge Function, a schema change, or a security-relevant change), draft a stub in
`docs/decisions/NNNN-title.md` from the template, with Status "Proposed". Do not invent a
rationale; fill Context and Decision from the diff and leave a note for the founder to
confirm Consequences. The founder promotes it to "Accepted".

## Rules

- Plain language. No marketing tone. No em-dashes.
- Do not fabricate. If the cause of a fix or the reason for a change is not evident, say
  "cause unknown, confirm" rather than guessing.
- The agent edits docs only. It does not change app code or run builds.
- If the agent itself makes a committable change (these docs), the versioning rule
  applies: bump `version` and `versionCode` in `app.json`.

## Cadence and scheduling

Target: every Friday. Options to run it (pick one):
- **Local:** run `/weekly-review` in Claude Code, or schedule
  `node scripts/weekly-report-data.mjs` via Windows Task Scheduler and review with Claude.
- **Cloud routine:** schedule a Claude Code routine so it runs even when the machine is
  off. See the project's scheduling skill.

The founder still reads and approves each week's output. This is a documentation
assistant, not an unattended writer of the external-facing narrative.
