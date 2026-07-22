---
description: "Use when starting or working on a feature, bug fix, performance change, or chore. Enforces issue-first tracking, issue-numbered branches, technical plans for substantial changes, task progress in plans, conventional commits, and PR conventions."
---

# Feature & Bug Workflow Rules

These rules apply to **all projects** and **all languages**. No exceptions.

## Before Starting Any Work

1. Check GitHub for an existing issue for the work.
2. If no issue exists, create one first — see the `feature-workflow` skill for the exact `gh` commands.
3. Note the issue number `#NNN`; it anchors the branch, plan when required, commits, and PR.
4. Create `feat/NNN-slug`, `fix/NNN-slug`, `perf/NNN-slug`, or `chore/NNN-slug` before editing repository files.

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
- **Plan files**: `plans/NNN-slug.md` — zero-pad to 3 digits (e.g. `042`)
- **Archived plans**: `plans/archived/YYYY-MM-DD-NNN-slug.md`
- **Slugs**: lowercase, hyphen-separated, max 5 words

## Commit & PR Rules

- Every implementation commit must reference the issue: `(#NNN)`
- PR title follows the same convention: `feat: description (#NNN)` or `feat(module): description (#NNN)` for module-scoped work
- PR body must include `Closes #NNN` on its own line — this auto-closes the issue on merge
- For multi-module repos, use `module:` labels on the issue when useful

## After Merge

- The PR closes the issue through `Closes #NNN`.
- Move a substantial plan to `plans/archived/YYYY-MM-DD-NNN-slug.md` as non-blocking housekeeping.
- Preserve its completed task checklist, notes, decisions, and verification evidence as implementation history.

## Relationship to Other Instructions

- For **Go code**: also follow `test-first.instructions.md` — write a failing test before production code.
- For **mvepapi changes**: also follow `mvep-codegen` skill — edit spec, re-run generator, implement `runXxxCmd`.
