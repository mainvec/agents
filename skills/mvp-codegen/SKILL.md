---
name: mvp-codegen
description: "Use when adding, modifying, or implementing MVP (Mainvec Platform) commands, fields, records, or APIs in any project that uses the mvp code generator; editing files under mvpapi/spec/ or mvpapi/<service>/go/; running `mvp generate` or `generate_api.sh`; writing typed Go (mvp/client) or TypeScript (@mainvec/mvpjs) clients on top of generated code; debugging 'undefined: runXxxCmd' build errors after spec changes; understanding generated-vs-handwritten file safety (NOMVGEN/NOWOGEN); wiring mvp.PackageHandler / interceptors. Spec-driven, command-based API framework: edit JSON spec → mvp generate → implement runXxxCmd in *_impl.go."
---

# MVP Codegen Skill

The Mainvec Platform (MVP) is a **spec-driven, command-based API framework**. You write a declarative JSON spec; the `mvp` CLI generates type-safe server, client, and CLI code in Go and JS/TS. This skill covers the spec format, the spec→generate→implement workflow, and how to safely edit projects that use it.

## The Golden Rule

**Edit specs, regenerate, then implement.** Never hand-edit a generated file unless its first line is `// NOMVGEN` (newer projects) or `// NOWOGEN` (legacy projects). Both markers tell the generator to skip that file.

## Identifying an MVP Project

A project uses MVP if it has:
- An `mvpapi/` (or similarly-named) directory with `spec/*.json`
- A `generate_api.sh` script (or equivalent) calling `mvp generate`
- Per-service subdirectories with `<service>_impl.go` (handwritten) and `<service>_commands.go` + `api/` (generated)

## Typical Project Layout

```
mvpapi/
├── generate_api.sh                  # runs mvp generate for each spec & lang
├── spec/
│   └── <service>-spec.json          # source of truth — you edit this
└── <service>/
    ├── go/
    │   ├── <service>_impl.go        # ✏️ NOMVGEN/NOWOGEN — runXxxCmd functions
    │   ├── <service>_commands.go    # ⛔ generated — wires impls to runner
    │   ├── api/                     # ⛔ generated — types, dispatch, package
    │   └── cmd/<service>/           # ⛔ generated — CLI main
    └── js/
        ├── api/                     # ⛔ generated — JS classes + .d.ts
        └── api/client/              # ✏️ handwritten — typed TS client wrapper
```

## File Edit Safety (memorize)

| Path pattern | Edit? | Why |
|--------------|-------|-----|
| `mvpapi/spec/*.json` | ✅ | Source of truth |
| `**/*_impl.go` (NOMVGEN/NOWOGEN header) | ✅ | Your business logic |
| `**/api/client/*` (JS/TS) | ✅ | Handwritten typed client |
| `**/*_commands.go` | ❌ | Regenerated — runner factory |
| `**/api/*.plain.go`, `*.pb.go`, `*.proto` | ❌ | Regenerated structs |
| `**/api/*_package.go` | ❌ | Regenerated — `Package`, dispatch |
| `**/cmd/<service>/*` | ❌ | Regenerated CLI main |
| `**/api/<service>.js`, `*.d.ts`, `*_package.js` | ❌ | Regenerated |

If a file starts with `// Code generated` and **not** `// NOMVGEN` / `// NOWOGEN`, the change belongs in the spec or in a NOMVGEN-marked file.

## Standard Workflows

### Add a new command

1. Open the spec (`mvpapi/spec/<service>-spec.json`).
2. Find the highest `fnum` used in any command. Each `fields` / `resultFields` block has its own independent `fnum` numbering starting at 1.
3. Add the command (see [spec-format.md](./references/spec-format.md) for type details):
   ```jsonc
   "MyNewCmd": {
     "title": "Short label",
     "alias": "my_new",
     "fields":       { "foo": { "fnum": 1, "type": "string", "tags": ["required"] } },
     "resultFields": { "ok":  { "fnum": 1, "type": "boolean" } }
   }
   ```
4. Validate: `mvp validate --in mvpapi/spec/<service>-spec.json`
5. Regenerate: `cd mvpapi && bash generate_api.sh`
6. Build: expect `undefined: runMyNewCmd` in the generated `*_commands.go`.
7. Add `runMyNewCmd(ctx context.Context, cmd *api.MyNewCmd) (*api.MyNewCmdResult, error)` to the `*_impl.go` file. Signature is fixed.

### Add a field to an existing command

1. Find the command's highest `fnum` within the relevant scope (`fields` and `resultFields` are separate scopes).
2. Add the field with `fnum` = highest + 1. **Never reuse** an old `fnum` — even one belonging to a deleted field. This is a wire-format invariant (especially under `pb3`).
3. Regenerate. Existing `runXxxCmd` keeps compiling — populate the new field where appropriate.

### Add a shared record

1. Add under `recordsDefs` at the spec root.
2. Reference from a field: `{ "fnum": N, "type": "recRef", "$ref": "#/recordsDefs/MyRecord" }`.
3. Regenerate.

### Rename / remove

- Removing a field is fine; **do not** later reuse its `fnum` for a different field.
- Renaming a command renames its `runXxxCmd` — update the matching function in `*_impl.go`.

## Diagnosing Common Errors

| Symptom | Cause | Fix |
|---------|-------|-----|
| `undefined: runXxxCmd` after build | Spec added a command; no impl yet | Add `runXxxCmd` in `*_impl.go` |
| `cannot use ... as XxxCmdHandler` | Impl signature drifted from spec | Re-check field types; signature: `func(ctx, *api.XxxCmd) (*api.XxxCmdResult, error)` |
| Edits to `*_commands.go` / `api/*.plain.go` vanished | File is generated | Move logic into a NOMVGEN file or into the spec |
| `unknown command type: *api.XxxCmd` at runtime | Forgot to regen after spec change | `bash generate_api.sh` |
| Validation: duplicate `fnum` | Two fields share a number in same scope | Bump one to next available |
| Validation: missing `$ref` | `recRef` field without `$ref` | Add `"$ref": "#/recordsDefs/Name"` |
| Wrong `go_package` import path | `gen_options.go_package` doesn't match module | Fix `go_package` in spec to `<module-path>;<alias>` |

## Build & Test

```bash
# Validate first — clearer errors than the generator
mvp validate --in mvpapi/spec/<service>-spec.json

# Regenerate
cd mvpapi && bash generate_api.sh && cd ..

# Build
go build ./...
```

## Protect a customized generated file

If a project requires hand-editing what was originally a generated file, add the marker as the **first line**:

```go
// NOMVGEN
package myservice
```

Use `// NOWOGEN` only if the project predates the rename (the generator accepts both).

## When You Need More Detail

Load only when relevant:

- [Spec format reference](./references/spec-format.md) — every type, all field properties, `gen_options`, complete example, schema URLs.
- [Generated patterns](./references/generated-patterns.md) — what `*_package.go`, `*_commands.go`, and `*_impl.go` look like; `mvp.Package` / `mvp.CommandRunner` interfaces; `PackageHandler`; interceptor chains; server setup.
- [Typed clients](./references/typed-clients.md) — building Go (`mvp/client`) and TypeScript (`@mainvec/mvpjs`) typed clients with auth-token interceptors and storage.

## CLI Quick Reference

```bash
mvp init     --name <service> --ns <namespace>
mvp validate --in <spec.json>
mvp generate --in <spec.json> --lang go      --outdir ./go --format=plain
mvp generate --in <spec.json> --lang js      --outdir ./js --format=plain
mvp generate --in <spec.json> --lang go,js   --outdir ./out
mvp generate --in <spec.json> --lang go --format=pb3   # protobuf mode
```

`format=plain` → Go structs with JSON tags, no protobuf dependency. `format=pb3` → full protobuf with `*.proto` and `*.pb.go`.

## Anti-patterns

- ❌ Editing `*_commands.go` to "wire up" a new command — it's regenerated. Add `runXxxCmd` to `*_impl.go`.
- ❌ Adding fields without `fnum`, or duplicating one within a scope.
- ❌ Reusing a deleted field's `fnum` for something new.
- ❌ Skipping `mvp validate` and going straight to `generate_api.sh`.
- ❌ Hand-writing structs in `api/` — they will be overwritten.
- ❌ Forgetting to regenerate after spec changes — silent runtime dispatch failures.
- ❌ Putting the `// NOMVGEN` marker on line 2 — it MUST be line 1.
