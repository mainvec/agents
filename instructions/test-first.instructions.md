---
description: "Use when adding a new feature, function, command, driver, or behavior change. Enforces test-first development: write a failing test before any production code."
applyTo: "**/*.go"
---

# Test-First Development

For **any new feature or behavior change**, a test case MUST be written and committed (or staged) **before** the production code that implements it.

## Hard Rules

- **No production code without a failing test first.** If asked to add a feature, the first edit must be a new `*_test.go` test (or a new case in an existing one) that fails for the right reason.
- **Run the test and confirm it fails** before writing the implementation. Cite the failure output briefly.
- **Then implement** the minimum production code to make the test pass.
- **Re-run the test** to confirm it passes. Run the package's full test suite (`go test ./<pkg>/`) to confirm no regressions.

## What Counts as a "New Feature"

Any of the following requires a test first:
- A new exported function, method, type, or interface
- A new CLI command (in `mvpapi/iulink/go/*_impl.go` or `mvpapi/iulinkd/go/*_impl.go`)
- A new broker driver, transport driver, or registry entry
- A new code path, branch, or option flag in existing code
- A bug fix (write a regression test that reproduces the bug first)

Pure refactors with no behavior change are exempt, but existing tests must still pass.

## Test Placement

- Unit tests live next to the code: `core/foo.go` → `core/foo_test.go`.
- Use the project's existing test patterns — check neighboring `*_test.go` files first (e.g., `core/broker_test.go`, `core/netlet_e2e_test.go`) for setup helpers and conventions.
- Prefer table-driven tests for multiple cases.
- Use `t.Parallel()` where the test does not share global state.

## Workflow Per Feature

1. Restate the feature in one sentence and identify the observable behavior to assert.
2. Write the test (`TestXxx`) — it should fail to compile or fail at runtime.
3. Run `go test -run TestXxx ./<pkg>/` and report the failure.
4. Implement the minimum code to pass.
5. Run `go test -run TestXxx ./<pkg>/`, then `go test ./<pkg>/` for the package.
6. If the change touches generated code, update specs in `mvpapi/spec/` and re-run `mvpapi/generate_api.sh` — never hand-edit generated files.

## When to Push Back

If the user asks for a feature without a test, briefly state that a test will be written first, then proceed. Do not skip the test step even if the change seems trivial.
