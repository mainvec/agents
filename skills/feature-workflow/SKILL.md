---
name: feature-workflow
description: "Use when starting a feature, bug fix, performance change, or chore; creating a GitHub issue; creating an issue-numbered branch; writing a technical plan for substantial changes; maintaining task progress; committing; opening a PR; or archiving a completed plan."
---

# Feature Workflow

End-to-end process for feature, bug fix, performance, and chore work. Requires `gh` CLI and `git`.

## Workflow Overview

```
GitHub issue  →  branch  →  [plan if substantial]  →  implement and update tasks  →  PR  →  [archive plan]
```

---

## Step 1 — Create GitHub Issue

Always first. Never skip.

```bash
# Feature
gh issue create --title "Short title" --label "feature" --body "Problem statement and goal."

# Bug
gh issue create --title "Short title" --label "bug" --body "Steps to reproduce and expected vs actual."

# Chore / internal
gh issue create --title "Short title" --label "chore" --body "What and why."

# Performance
gh issue create --title "Short title" --label "performance" --body "What is slow, the target, and how it will be measured."
```

Capture the issue number from the output URL (e.g. `https://github.com/org/repo/issues/42` → `#42`).

**Multi-module repos:** also add a `module:` label to scope the issue to a specific module (e.g. `module:iulink-ui`). Add the label first if it doesn't exist:
```bash
gh label create module:iulink-ui --color "#bfd4f2" --description "iulink-ui module"
gh issue create --title "Short title" --label "feature" --label "module:iulink-ui" --body "..."
```

If labels don't exist yet in the repo, create them first:
```bash
gh label create feature --color "#0075ca" --description "New feature or request"
gh label create bug --color "#d73a4a" --description "Something is broken"
gh label create performance --color "#fbca04" --description "Performance improvement"
gh label create chore --color "#e4e669" --description "Internal work, no user-facing change"
```

The issue owns overall lifecycle, assignee, priority, dependencies, discussion, acceptance, and open/closed state. Do not duplicate the plan's detailed task checklist in the issue. GitHub Projects are optional views over issues.

---

## Step 2 — Create Branch

```bash
git checkout -b feat/42-slug    # for features
git checkout -b fix/42-slug     # for bugs
git checkout -b perf/42-slug    # for performance
git checkout -b chore/42-slug   # for chores
```

---

## Step 3 — Create a Plan for Substantial Work

A plan is required for new behavior, features, exported APIs, commands, drivers, UI screens, protocol or configuration changes, migrations, security-sensitive work, architecture changes, significant refactors, and cross-module work.

Narrow bug fixes, documentation, test-only work, dependency updates, formatting, and routine chores may skip this step. Bug fixes still need regression tests.

Path: `plans/NNN-slug.md` (zero-pad the issue number to 3 digits, e.g. `042`). Use [./assets/plan-template.md](./assets/plan-template.md).

- Add one `## Progress` checkbox per implementation task (`T1`, `T2`, ...).
- Add one matching `## Tasks` block per Progress item.
- Give each task an Outcome, focused Verification, and Notes field.
- Do not include workflow checkboxes or a separate task Status field.
- Review the plan before substantial implementation. Use a planning-only PR only for unusually risky or cross-team designs.

---

## Step 4 — Implement and Maintain Task Progress

- For **Go code**: follow `test-first` instructions — write a failing test before any production code.
- For **mvepapi / spec changes**: follow `mvep-codegen` skill — edit spec, regenerate, then implement `runXxxCmd`.
- For all other code: work in the branch, keep commits focused.

For a substantial plan:
1. Work on one logical task at a time when practical.
2. Implement the task and run its focused Verification check.
3. Check the matching Progress item only after that check succeeds.
4. Record technical discoveries, blockers, changed assumptions, and evidence in Notes.
5. Update Proposed Design, Risks, Rollout, or Decision Log when understanding changes.

---

## Step 5 — Commit

Each commit that implements work must reference the issue:

```bash
git commit -m "feat: add thing (#42)"
git commit -m "fix: handle edge case in thing (#42)"
git commit -m "perf: reduce relay allocations (#42)"
git commit -m "chore: update deps (#42)"
```

**Multi-module repos:** use the Conventional Commits scope to identify the module:
```bash
git commit -m "feat(iulink-ui): add dark mode (#42)"
git commit -m "fix(core): handle nil peer conn (#42)"
```

---

## Step 6 — Open PR

Before opening the PR, complete all intended Progress items or move unfinished scope to a follow-up issue and plan.

```bash
gh pr create \
  --title "feat: add thing (#42)" \
  --body "$(printf 'Closes #42\n\n## Summary\n<what changed and why>')"
```

Rules:
- Title: same convention as commits — include scope if module-scoped: `feat(iulink-ui): add thing (#42)`
- Body: `Closes #NNN` must be on its own line (GitHub auto-closes on merge)

---

## Step 7 — After Merge

- GitHub issue auto-closes (via `Closes #NNN`)
- For substantial work, move the plan to `plans/archived/YYYY-MM-DD-NNN-slug.md` as non-blocking housekeeping. Keep its completed task checklist and notes as history.
- Delete the branch (optional):
  ```bash
  git branch -d feat/42-slug
  gh pr view 42 --json headRefName | xargs -I{} git push origin --delete {}
  ```

---

## Quick Reference

| Item | Pattern |
|---|---|
| Branch | `feat/NNN-slug`, `fix/NNN-slug`, `perf/NNN-slug`, `chore/NNN-slug` |
| Commit | `feat: desc (#NNN)`, `fix: desc (#NNN)`, `perf: desc (#NNN)`, `chore: desc (#NNN)` |
| PR title | same as commit |
| PR body | must include `Closes #NNN` |
| Plan file | `plans/NNN-slug.md` for substantial work |
| Task progress | `## Progress` task checkboxes, matched by detailed `## Tasks` blocks |
| Archived plan | `plans/archived/YYYY-MM-DD-NNN-slug.md` |

## Parallel Work

- Use one issue, plan, and branch or worktree per independently shippable change.
- Split large efforts into parent and child issues so contributors own separate plans.
- Avoid multiple contributors editing one plan concurrently.
- Represent dependencies with linked GitHub issues rather than a shared repository registry.
