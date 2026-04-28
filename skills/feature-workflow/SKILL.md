---
name: feature-workflow
description: "Use when starting a new feature, bug fix, or chore; creating or updating a GitHub issue; writing a plan file (plans/NNN-slug.md); updating plans/registry.json; opening a PR; naming a branch; writing commit messages. Complete end-to-end workflow: gh issue create → plan doc → registry entry → branch → implement → PR. Use when the user says 'add a feature', 'fix a bug', 'start a task', 'create an issue', 'open a PR', or asks about the feature workflow."
---

# Feature Workflow

End-to-end process for every feature, bug fix, or chore. Requires `gh` CLI and `git`.

## Workflow Overview

```
gh issue create  →  plans/NNN-slug.md  →  registry.json  →  branch  →  implement  →  PR
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
gh label create chore --color "#e4e669" --description "Internal work, no user-facing change"
```

---

## Step 2 — Create Plan File

Path: `plans/NNN-slug.md` (zero-pad issue number to 3 digits, e.g. `042`).

Use the template at [./assets/plan-template.md](./assets/plan-template.md). Substitute:
- `NNN` → issue number (zero-padded)
- `slug` → lowercase-hyphenated summary (max 5 words)
- Fill in Type, GitHub URL, Branch, Problem/Goal, Approach.
- Break the work into tasks (`T1`, `T2`, …) — add one line per task to the `## Progress` checklist and one `### TN` block to `## Tasks`.
- Start each task block with `**Status**: todo` and an empty `**Notes**:` field.

**As work proceeds, keep the plan file current:**
- Check off items in `## Progress` when done.
- Update `**Status**` in each task block: `todo → in-progress → done`.
- Append to `**Notes**` — decisions made, blockers, alternatives rejected. This is the context log.
- Only one task should be `in-progress` at a time.

---

## Step 3 — Update plans/registry.json

If `plans/registry.json` does not exist, create it:
```json
{
  "issues": []
}
```

Add an entry to the `"issues"` array:
```json
{
  "id": 42,
  "type": "feature",
  "title": "Short title matching the GitHub issue",
  "module": "iulink-ui",
  "status": "planned",
  "plan": "plans/042-slug.md",
  "branch": "feat/42-slug",
  "github_issue": "https://github.com/org/repo/issues/42",
  "created": "YYYY-MM-DD"
}
```

Valid `type` values: `feature`, `bug`, `chore`
Valid `status` values: `planned`, `in-progress`, `review`, `done`

`module` is optional. Omit it for root-level work. Use it when the change is scoped to a specific sub-module or sub-directory (e.g. `iulink-ui`, `iulink-tui`, `core`).

Update `status` as work progresses.

---

## Step 4 — Create Branch

```bash
git checkout -b feat/42-slug    # for features
git checkout -b fix/42-slug     # for bugs
git checkout -b chore/42-slug   # for chores
```

---

## Step 5 — Implement

- For **Go code**: follow `test-first` instructions — write a failing test before any production code.
- For **mvpapi / spec changes**: follow `mvp-codegen` skill — edit spec, regenerate, then implement `runXxxCmd`.
- For all other code: work in the branch, keep commits focused.

---

## Step 6 — Commit

Each commit that implements work must reference the issue:

```bash
git commit -m "feat: add thing (#42)"
git commit -m "fix: handle edge case in thing (#42)"
git commit -m "chore: update deps (#42)"
```

**Multi-module repos:** use the Conventional Commits scope to identify the module:
```bash
git commit -m "feat(iulink-ui): add dark mode (#42)"
git commit -m "fix(core): handle nil peer conn (#42)"
```

---

## Step 7 — Open PR

```bash
gh pr create \
  --title "feat: add thing (#42)" \
  --body "$(printf 'Closes #42\n\n## Summary\n<what changed and why>')"
```

Rules:
- Title: same convention as commits — include scope if module-scoped: `feat(iulink-ui): add thing (#42)`
- Body: `Closes #NNN` must be on its own line (GitHub auto-closes on merge)
- Update `status` in `plans/registry.json` to `"review"`

---

## Step 8 — After Merge

- GitHub issue auto-closes (via `Closes #NNN`)
- Update `status` in `plans/registry.json` to `"done"`
- Update `**Status**` in `plans/NNN-slug.md` to `done`
- Delete the branch (optional):
  ```bash
  git branch -d feat/42-slug
  gh pr view 42 --json headRefName | xargs -I{} git push origin --delete {}
  ```

---

## Quick Reference

| Item | Pattern |
|---|---|
| Branch | `feat/NNN-slug`, `fix/NNN-slug`, `chore/NNN-slug` |
| Commit | `feat: desc (#NNN)`, `fix: desc (#NNN)` |
| PR title | same as commit |
| PR body | must include `Closes #NNN` |
| Plan file | `plans/NNN-slug.md` |
| Registry | `plans/registry.json` |
