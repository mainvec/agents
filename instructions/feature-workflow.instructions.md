---
description: "Use when starting or working on a feature, bug fix, or chore in any project. Enforces GitHub Issues tracking, local plan registry, conventional commits, and PR conventions. Use when: creating a new feature, fixing a bug, starting a task, creating a GitHub issue, writing a plan file, updating registry.json, naming a branch, writing a commit message, opening a PR."
---

# Feature & Bug Workflow Rules

These rules apply to **all projects** and **all languages**. No exceptions.

## Before Starting Any Work

1. Check `plans/registry.json` in the repo for an existing entry for this task.
2. If no GitHub issue exists, create one first — see the `feature-workflow` skill for the exact `gh` commands.
3. Note the issue number `#NNN` — it anchors everything that follows.

## Required Artefacts (create in order)

| # | Artefact | Path |
|---|---|---|
| 1 | GitHub Issue | `gh issue create ...` → gets `#NNN` |
| 2 | Plan file | `plans/NNN-slug.md` (from template in `feature-workflow` skill) |
| 3 | Registry entry | add/update entry in `plans/registry.json` |
| 4 | Branch | `feat/NNN-slug`, `fix/NNN-slug`, or `chore/NNN-slug` |

## Plan File Structure (mandatory)

Every `plans/NNN-slug.md` must have:
1. **`## Progress` checklist at the top** — one line per workflow step plus one line per task (`T1`, `T2`, …). Check off as work completes.
2. **`## Problem / Goal`** and **`## Approach`** sections.
3. **`## Tasks` section** — one `### TN: title` block per task, each with:
   - `**Status**`: `todo`, `in-progress`, or `done`
   - `**Notes**`: running context — decisions made, blockers encountered, alternatives rejected.

Update task status and notes as work progresses. This is the agent's working log — keep it current.

## Naming Conventions

- **Branches**: `feat/NNN-slug`, `fix/NNN-slug`, `chore/NNN-slug`
- **Commits**: `feat: description (#NNN)`, `fix: description (#NNN)`, `chore: description (#NNN)`
- **Multi-module commits**: use Conventional Commits scope — `feat(module-name): description (#NNN)`
- **Plan files**: `plans/NNN-slug.md` — zero-pad to 3 digits (e.g. `042`)
- **Slugs**: lowercase, hyphen-separated, max 5 words

## Commit & PR Rules

- Every commit that implements feature/fix work must reference the issue: `(#NNN)`
- PR title follows the same convention: `feat: description (#NNN)` or `feat(module): description (#NNN)` for module-scoped work
- PR body must include `Closes #NNN` on its own line — this auto-closes the issue on merge
- Update `status` in `plans/registry.json` as work progresses: `planned → in-progress → review → done`
- For multi-module repos: add a `module:` GitHub label to the issue and a `"module"` field to the registry entry

## What Counts as a "New Feature or Bug Fix"

Any of the following requires following this workflow:
- New exported function, method, type, command, or driver
- New CLI or daemon command
- A bug fix (with regression test)
- Any new code path, option, or configuration field
- A UI feature or screen

Pure refactors with no behaviour change are exempt but must still pass tests.

## Relationship to Other Instructions

- For **Go code**: also follow `test-first.instructions.md` — write a failing test before production code.
- For **mvpapi changes**: also follow `mvp-codegen` skill — edit spec, re-run generator, implement `runXxxCmd`.
