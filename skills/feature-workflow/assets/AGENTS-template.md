# AGENTS.md

Instructions for AI coding agents and human contributors working in this repo.

## Project

<!-- One paragraph: what this repo is, its modules, and the main build/test commands. -->

## Workflow

- Accepted work starts from a GitHub issue `#NNN`. Trivial changes (typos, formatting, dependency bumps) need no issue.
- Branch: `feat/NNN-slug`, `fix/NNN-slug`, `perf/NNN-slug`, or `chore/NNN-slug`, from `main`. Keep it short-lived; rebase on `main` before merge.
- Substantial work has a plan at `plans/NNN-slug.md`, where `NNN` is the issue number. Never create a plan without an issue, and never encode state (`DRAFT`, `WIP`) in file names.
- Plans keep a `## Progress` checklist of implementation tasks; check a task only after its verification passes.
- Commits and PR titles: Conventional Commits with the issue, e.g. `feat(module): add thing (#NNN)`. PR body has `Closes #NNN` on its own line.
- The PR's last commit moves the plan to `plans/archived/YYYY-MM-DD-NNN-slug.md`.
- Never hand-merge generated code or lockfiles: regenerate them.
- Changelog: <!-- either "generated from PR titles at release; do not edit CHANGELOG.md" or "add one line under Unreleased" -->

## Project rules

<!-- Repo-specific rules: codegen commands, test-first, security boundaries, platforms. -->
