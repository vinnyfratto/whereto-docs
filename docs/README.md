# WhereTo Engineering Documentation

This folder is the living record of what WhereTo is built on, how we build it, and
what changes week to week. It is the single source of truth for the system: a new
engineer, or an outside technical reviewer, should be able to understand the whole
codebase from these files in a day, instead of extracting it from the founder's head
over weeks.

There are two layers.

## 1. Living docs (always current)

| File | What it answers |
| --- | --- |
| [tech-stack.md](tech-stack.md) | What is this built on, with versions |
| [architecture/overview.md](architecture/overview.md) | How the pieces fit together |
| [architecture/integrations.md](architecture/integrations.md) | Every external service and its blast radius |
| [architecture/data-model.md](architecture/data-model.md) | The Supabase schema and data |
| [security-and-data.md](security-and-data.md) | Secrets, auth, PII, PCI, legal posture |
| [operations/ci-cd-and-releases.md](operations/ci-cd-and-releases.md) | How code ships |
| [process/dev-process.md](process/dev-process.md) | How work gets done |
| [risks.md](risks.md) | Known debt and risk register |
| [STATE.md](STATE.md) | Rolling "what is this system right now" snapshot |
| [PRELAUNCH_TESTING.md](PRELAUNCH_TESTING.md) | The plan to verify core workflows before launch |

## 2. Accreting record (append-only history)

| Location | What it captures |
| --- | --- |
| [decisions/](decisions/) | Architecture Decision Records: **why** it is built this way |
| [../CHANGELOG.md](../CHANGELOG.md) | What shipped, per version |
| [../week-in-review/](../week-in-review/) | Weekly dev + bug reviews, including regressions |
| [decisions/](decisions/) postmortems | Blameless incident write-ups |

## How the weekly update works

A weekly agent (see [automation/weekly-review-agent.md](automation/weekly-review-agent.md))
runs every Friday. It reads the week's git history, dependency changes, and schema
changes, then:

1. Writes `week-in-review/YYYY-Www.md` (dev summary + bug summary + regressions).
2. Appends `CHANGELOG.md`.
3. Updates `tech-stack.md` if dependencies changed, and flags new licenses.
4. Refreshes `STATE.md`.
5. Adds new items to `risks.md`.
6. **Drafts an ADR stub** whenever it detects an architectural, dependency, or
   security-relevant change, for the founder to confirm.

The agent proposes. A human approves the ADRs and the STATE delta, because those are
the external-facing technical narrative.

## Important: this repo is currently PUBLIC

`github.com/vinnyfratto/Wander_App` is a public repository, so **everything in this
folder is world-readable**, as is all source code. Two consequences:

- These docs deliberately contain **no secrets and no exploit-level detail**. Security
  content here describes posture and controls only.
- For a commercial product, a public app repo is itself a review flag. See
  [GAP-REPORT.md](GAP-REPORT.md) item G-01. If the repo is made private, the sensitive
  docs can carry more detail.

## Maintenance rules

- Plain language. No marketing tone. No em-dashes.
- One decision per ADR. Never edit a decided ADR; supersede it with a new one.
- Every dependency added gets a line in `tech-stack.md` with its license.
- Update the "Last updated" line at the top of any living doc you touch.

_Last updated: 2026-07-08_
