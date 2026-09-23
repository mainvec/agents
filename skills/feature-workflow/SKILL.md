---
name: feature-workflow
description: "Use when starting a feature, bug fix, performance change, or chore; creating a GitHub issue, discussion, issue-numbered branch, Git worktree, or parallel agent workspace; writing a technical plan; maintaining task progress; committing; opening a draft or ready PR; archiving a completed plan; or icebox/design labels."
---

# Feature Workflow

End-to-end process for feature, bug fix, performance, and chore work. Requires `gh` CLI and `git`.

## Workflow Overview

```
[discussion]  →  GitHub issue  →  branch  →  [plan if substantial, label design]  →  draft PR  →  implement and update tasks  →  [archive plan in PR]  →  merge
```

---

## Step 1 — Create GitHub Issue

Always first, except for trivial changes (typos, formatting, dependency bumps, CI tweaks with no behavior change), which go straight to a PR with no `(#NNN)`.

In **discussion-first** projects (issue templates disable blank issues and link to Discussions), ideas and bug triage start as a Discussion; a maintainer opens the issue once the work is accepted and links the Discussion from it.

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
gh label create design --color "#d4c5f9" --description "Plan being designed or reviewed"
gh label create icebox --color "#c5def5" --description "Parked: not on the current roadmap"
```

Large efforts: open an umbrella issue that lists child issues (`gh issue create` per child, then link them as sub-issues). Each child gets its own branch, PR, and plan; the umbrella plan records scope and decisions only.

The issue owns overall lifecycle, assignee, priority, dependencies, discussion, acceptance, and open/closed state. Do not duplicate the plan's detailed task checklist in the issue. GitHub Projects are optional views over issues.

---

## Step 2 — Create Branch

Use a normal branch checkout by default:

```bash
git checkout -b feat/42-slug    # for features
git checkout -b fix/42-slug     # for bugs
git checkout -b perf/42-slug    # for performance
git checkout -b chore/42-slug   # for chores
```

### Optional: Parallel Agents with Worktrees

Use this mode only when the developer explicitly requests parallel or multi-agent work. A coordinator creates one worktree per issue from an explicit base ref and opens each in a separate VS Code window:

```bash
git fetch origin
git worktree add -b feat/42-slug ../repo-042 origin/main
code --new-window ../repo-042
```

Replace `repo-042` and `origin/main` with the appropriate worktree directory and integration base. Never omit the base ref: otherwise Git branches from the current checkout, which may be another feature branch.

Worktree rules:
- One issue, plan, branch, worktree, and agent per independently shippable change.
- Keep the ordinary single-checkout workflow as the default.
- Do not use `git worktree add --force`, share a branch between worktrees, or run destructive operations against another agent's refs.
- Avoid `git stash` for agent handoff because the stash ref is shared by the repository; commit coherent work to the issue branch instead.
- Existing uncommitted and untracked files do not appear in a new worktree. Commit required inputs first or create them in the assigned worktree.
- Worktrees isolate files, indexes, and build output, but refs, remotes, hooks, repository config, object storage, and merge conflicts remain shared concerns.
- Coordinate plans that overlap generated files, schemas, lockfiles, changelogs, or external working-tree dependencies.
- Assign unique ports, sockets, and state directories to concurrent services unless sharing a service is intentional and safe.

---

## Step 3 — Create a Plan for Substantial Work

A plan is required for new behavior, features, exported APIs, commands, drivers, UI screens, protocol or configuration changes, migrations, security-sensitive work, architecture changes, significant refactors, and cross-module work.

Narrow bug fixes, documentation, test-only work, dependency updates, formatting, and routine chores may skip this step. Bug fixes still need regression tests.

Path: `plans/NNN-slug.md`, where `NNN` is **the Step 1 issue number** zero-padded to 3 digits (`#42` → `plans/042-slug.md`, `#1234` → `plans/1234-slug.md`). Use [./assets/plan-template.md](./assets/plan-template.md).

- The plan number must equal the issue number. Never use the next free plan number, and never create a plan before its issue exists. Parked ideas get an issue (e.g. `icebox` label) first.
- The header `Issue:` link, the file-name number, and the branch number must agree. Check with `ls plans/*.md` before creating a plan.

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

Before opening the PR, complete all intended Progress items or move unfinished scope to a follow-up issue and plan. Open it as a **draft** as soon as implementation starts if others need visibility, and mark it ready when done.

```bash
gh pr create --draft \
  --title "feat: add thing (#42)" \
  --body "$(printf 'Closes #42\n\n## Summary\n<what changed and why>')"
gh issue edit 42 --remove-label design
```

Rules:
- Title: same convention as commits — include scope if module-scoped: `feat(iulink-ui): add thing (#42)`
- Body: `Closes #NNN` must be on its own line (GitHub auto-closes on merge)
- Rebase on `main` before merging; prefer squash merge.
- Last commit before merge archives the plan:
  ```bash
  git mv plans/042-slug.md plans/archived/$(date +%F)-042-slug.md
  git commit -m "chore: archive plan 042 (#42)"
  ```

---

## Step 7 — After Merge

- GitHub issue auto-closes (via `Closes #NNN`)
- The plan was already archived by the PR's last commit; do not add an archive commit on `main`.
- Delete the branch (optional):
  ```bash
  git branch -d feat/42-slug
  gh pr view 42 --json headRefName | xargs -I{} git push origin --delete {}
  ```
- If a worktree was used, preserve or commit any required changes and remove it before deleting its local branch:
  ```bash
  git worktree remove ../repo-042
  git worktree prune
  git branch -d feat/42-slug
  ```

---

## Quick Reference

| Item | Pattern |
|---|---|
| Branch | `feat/NNN-slug`, `fix/NNN-slug`, `perf/NNN-slug`, `chore/NNN-slug` |
| Commit | `feat: desc (#NNN)`, `fix: desc (#NNN)`, `perf: desc (#NNN)`, `chore: desc (#NNN)` |
| PR title | same as commit |
| PR body | must include `Closes #NNN` |
| Plan file | `plans/NNN-slug.md` for substantial work, `NNN` = issue number |
| Task progress | `## Progress` task checkboxes, matched by detailed `## Tasks` blocks |
| Archived plan | `plans/archived/YYYY-MM-DD-NNN-slug.md`, moved in the PR's last commit |
| Parked work | `icebox` label; plan in `plans/icebox/NNN-slug.md` |
| Plan in design | `design` label on the issue |

## Parallel Work

- Keep normal branch checkout as the default; use worktrees only when the developer requests parallel or multi-agent work.
- Use one issue, plan, branch, worktree, and VS Code window per independently shippable parallel change.
- Split large efforts into parent and child issues so contributors own separate plans.
- Avoid multiple contributors editing one plan concurrently.
- Represent dependencies with linked GitHub issues rather than a shared repository registry.
