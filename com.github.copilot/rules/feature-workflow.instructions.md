---
description: "Use when starting or working on a feature, bug fix, performance change, chore, parallel agent task, or Git worktree; creating issues, discussions, plans, branches, commits, or PRs; resolving merge conflicts in plans, changelogs, or generated code. Enforces issue-first tracking, plan number = issue number, issue-numbered branches, optional worktree isolation, technical plans, task progress, conventional commits, and PR conventions."
---

# Feature & Bug Workflow Rules

These rules apply to **all projects** and **all languages**, for team and open-source work alike.

## Work Lifecycle

| Stage | Where it lives |
|---|---|
| Idea, not yet accepted | A short issue, or a Discussion in discussion-first projects. No plan file. |
| Accepted | An issue with a milestone. Discussion-first projects open or convert the issue from the discussion and link it. |
| Plan being designed or reviewed | `design` label on the issue; `plans/NNN-slug.md` on the issue branch, not on `main` |
| Implementation in progress | Draft PR from the issue branch; remove the `design` label |
| Parked | `icebox` label (or leave it as a Discussion); plan, if any, in `plans/icebox/NNN-slug.md` |
| Done | PR merged with `Closes #NNN`; the plan was archived inside that PR |

Never encode state in file names (`DRAFT-`, `WIP-`). State lives on the issue and PR.

## Discussion-First Projects

If a project routes ideas and bug triage through GitHub Discussions (issue templates disable blank issues and link to Discussions):
- Do not open issues for unrefined ideas; open or reply to a Discussion.
- A plan starts only after a maintainer opens the issue. The issue and the plan header both link the Discussion.
- Design talk belongs in the issue or Discussion, not in a WIP pull request.

## Trivial Changes

Typo and wording fixes, formatting, dependency bumps, and CI tweaks with no behavior change need no issue or plan. Use a Conventional Commit PR title without `(#NNN)` and describe the change in the PR body.

## Before Starting Any Work

1. Check GitHub for an existing issue for the work (and Discussions in discussion-first projects).
2. If no issue exists, create one first — see the `feature-workflow` skill for the exact `gh` commands. Skip this for trivial changes (see above).
3. Note the issue number `#NNN`; it anchors the branch, plan when required, commits, and PR.
4. Create `feat/NNN-slug`, `fix/NNN-slug`, `perf/NNN-slug`, or `chore/NNN-slug` before editing repository files.

Use a normal branch checkout by default. Only use a Git worktree when the developer explicitly requests parallel or multi-agent work. For that mode:
- Assign one issue, branch, worktree, and VS Code window to each agent.
- Create each worktree from an explicit base ref such as `origin/main`; never rely on the current checkout's `HEAD`.
- Keep agents out of each other's worktrees and branches. Do not use `--force`, shared stashes, or destructive operations on another agent's refs.
- Treat worktrees as filesystem isolation, not merge-conflict prevention. Coordinate overlapping generated files, lockfiles, changelogs, schemas, and external working-tree dependencies.
- Give concurrently run services separate ports, sockets, and state directories unless sharing one service is intentional and safe.

## Sources of Truth

- The GitHub issue owns overall lifecycle, assignee, priority, dependencies, discussion, acceptance, and open/closed state.
- A substantial change uses `plans/NNN-slug.md` for technical design, detailed tasks, task progress, notes, decisions, and verification evidence.
- GitHub Projects are optional views over issues. Do not manually synchronize Project state with plan files.
- Do not create or update `plans/registry.json`.

## When a Plan Is Required

A plan is required for:
- New behavior, features, exported APIs, commands, drivers, or UI screens
- Protocol, configuration, storage, or other contract changes
- Migrations or security-sensitive work
- Architecture changes, significant refactors, or cross-module work

An issue and branch are sufficient for narrow bug fixes, documentation, test-only work, dependency updates, formatting, and routine chores. Bug fixes still require regression tests.

## Plan File Structure

Every substantial `plans/NNN-slug.md` must contain:
1. A linked GitHub issue and sections for Problem/Goal, Goals, Non-goals, Proposed Design, Affected Modules, Risks and Compatibility, Verification, Rollout, and Decision Log.
2. A `## Progress` checklist containing only implementation tasks (`T1`, `T2`, ...). Do not include workflow steps such as issue creation, branch creation, tests passing, or PR opening.
3. A matching `## Tasks` block for each Progress item. Each task records its Outcome, focused Verification, and running technical Notes.

The Progress checkbox is the only task completion state; do not add a separate task Status field. Check a task only after its implementation and focused verification succeed. Keep task notes, design, risks, and decisions current as technical understanding changes. Do not copy assignee, review, PR, or merge state into the plan.

## Naming Conventions

- **Branches**: `feat/NNN-slug`, `fix/NNN-slug`, `perf/NNN-slug`, `chore/NNN-slug`
- **Commits**: `feat: description (#NNN)`, `fix: description (#NNN)`, `perf: description (#NNN)`, `chore: description (#NNN)`
- **Multi-module commits**: use Conventional Commits scope — `feat(module-name): description (#NNN)`
- **Plan files**: `plans/NNN-slug.md` where `NNN` is **the GitHub issue number** the plan belongs to, zero-padded to 3 digits (`#42` → `042`, `#1234` → `1234`)
- **Archived plans**: `plans/archived/YYYY-MM-DD-NNN-slug.md` (same issue number)
- **Slugs**: lowercase, hyphen-separated, max 5 words

## Plan Numbering Rules

- The plan number **is** the issue number. Never pick the next free sequence number, and never number a plan independently of its issue.
- Create the issue before the plan file. A plan with no issue must not exist; a parked idea gets an issue (e.g. labelled `icebox`) and its plan is named by that issue.
- The plan header links the same issue (the template's `**GitHub Issue**: [#NNN](...)` line, or `Issue: [#NNN](...)`). The file-name number, the header link, and the branch number must all agree.
- One issue owns at most one plan. Split large work into child issues, each with its own plan.
- Cross-reference other plans by their issue number (`plan 135`, or `#135`).
- When an existing plan's number does not match its issue, rename it (`git mv`) and update every cross-reference in the same change. Archived plans keep their historical names.

## Commit & PR Rules

- Every implementation commit must reference the issue: `(#NNN)`
- PR title follows the same convention: `feat: description (#NNN)` or `feat(module): description (#NNN)` for module-scoped work
- PR body must include `Closes #NNN` on its own line — this auto-closes the issue on merge
- For multi-module repos, use `module:` labels on the issue when useful
- Before merging a substantial plan's PR, make its last commit move the plan to `plans/archived/YYYY-MM-DD-NNN-slug.md` (date = merge day). Never archive with a follow-up commit on `main`.

## Avoiding Merge Conflicts

- **Short-lived branches.** Merge small PRs into `main` often; rebase on `main` before merging; prefer squash merges.
- **No long-lived umbrella branches.** Large efforts are an umbrella issue with child issues. Each child has its own branch, PR, and plan; the umbrella plan holds scope and decisions only.
- **No shared index files.** Do not maintain roadmap, registry, or plan-index files that every PR edits; milestones and labels are the index.
- **Changelog.** If the repo generates its changelog from PR titles at release time, do not edit `CHANGELOG.md` in feature PRs. Otherwise add exactly one line under `Unreleased`.
- **Generated code.** Never hand-merge generated files. On conflict, take either side, rerun the generator, and commit the result. CI should verify generated output is current.
- **Lockfiles.** Resolve by regenerating (`go mod tidy`, `npm install`), not by hand-editing.

## After Merge

- The PR closes the issue through `Closes #NNN`.
- The plan is already archived by the PR's last commit, with its completed task checklist, notes, decisions, and verification evidence preserved as implementation history.
- If the branch has a worktree, remove it with `git worktree remove <path>` after preserving any required changes, then run `git worktree prune` when stale metadata needs cleanup.

## Relationship to Other Instructions

- For **Go code**: also follow `test-first.instructions.md` — write a failing test before production code.
- For **mvepapi changes**: also follow `mvep-codegen` skill — edit spec, re-run generator, implement `runXxxCmd`.
