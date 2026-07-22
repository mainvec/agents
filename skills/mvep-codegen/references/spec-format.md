# MVEP Spec Format Reference

Specs live under `mvepapi/spec/*.json` (or `*.jsonc`) and are validated against:

```
https://spec.mainvec.com/mvepspec/0.2/schema/2026-01-15
```

Specs pinned to earlier schema URLs continue to validate for backward compatibility.

## Top-Level Shape

```jsonc
{
  "$id": "myservice",
  "$schema": "https://spec.mainvec.com/mvepspec/0.2/schema/2026-01-15",
  "name": "myservice",
  "namespace": "myservicens",
  "title": "My Service API",
  "version": "v0.1",

  "gen_options": {
    "go_package":     "github.com/acme/myservice/mvepapi/go;myservice",
    "go_api_package": "github.com/acme/myservice/mvepapi/go/api;api",
    "format": "plain"
  },

  "commands":    { /* … */ },
  "recordsDefs": { /* … */ }
}
```

### Required Top-Level Fields

| Field | Notes |
|-------|-------|
| `$id` | Unique service id |
| `name` | File-name basis for generated files |
| `namespace` | Used for protobuf package and JS namespace |

### `gen_options`

| Option | Values | Notes |
|--------|--------|-------|
| `format` | `plain` \| `pb3` | `plain` = Go structs + JSON tags, no protobuf dep. `pb3` = full protobuf. |
| `go_package` | `<module/path>;<alias>` | Must match the directory's actual Go package |
| `go_api_package` | `<module/path>;<alias>` | Generated `api/` sub-package path |
| `edition` | `2023` | Only for `pb3` |
| `go_default_api_level` | `API_OPAQUE` | Only for `pb3` |

## Commands

Each command is an entry under `commands`. Convention: PascalCase + `Cmd` suffix.

```jsonc
"UserRegisterCmd": {
  "title": "Register a user",
  "alias": "register",            // CLI subcommand name (snake_case)
  "desc":  "Creates a new user account",
  "fields": {
    "email":    { "fnum": 1, "type": "string", "tags": ["required"] },
    "password": { "fnum": 2, "type": "string", "tags": ["required"] },
    "name":     { "fnum": 3, "type": "string" }
  },
  "resultFields": {
    "userID": { "fnum": 1, "type": "string" },
    "token":  { "fnum": 2, "type": "string" }
  }
}
```

`fields` and `resultFields` each have their own independent `fnum` numbering.

## Records (`recordsDefs`)

Shared structures referenced from command fields:

```jsonc
"recordsDefs": {
  "User": {
    "name": "User",
    "title": "User record",
    "fields": {
      "id":        { "fnum": 1, "type": "string" },
      "email":     { "fnum": 2, "type": "string" },
      "active":    { "fnum": 3, "type": "boolean" },
      "createdAt": { "fnum": 4, "type": "timestamp" },
      "metadata":  { "fnum": 5, "type": "map", "valueType": "string" }
    }
  }
}
```

Use from a command field:

```jsonc
"user": { "fnum": 1, "type": "recRef", "$ref": "#/recordsDefs/User" }
```

## Field Types

| Type        | Go                                | JS/TS         | Notes |
|-------------|-----------------------------------|---------------|-------|
| `string`    | `string`                          | `string`      | UTF-8 |
| `boolean`   | `bool`                            | `boolean`     | |
| `int32`     | `int32`                           | `number`      | |
| `int64`     | `int64`                           | `number`      | |
| `uint32`    | `uint32`                          | `number`      | |
| `sint32`    | `int32`                           | `number`      | ZigZag |
| `float`     | `float32`                         | `number`      | |
| `double`    | `float64`                         | `number`      | |
| `bytes`     | `[]byte`                          | `Uint8Array`  | |
| `timestamp` | `*timestamppb.Timestamp` / `time.Time` | `Date`   | |
| `duration`  | `*durationpb.Duration`            | `number`      | |
| `uuid`      | `string`                          | `string`      | |
| `recRef`    | pointer to record struct          | object        | requires `$ref` |
| `map`       | `map[string]T`                    | `Object`      | requires `valueType` |
| `recDef`    | inline struct                     | object        | inline definition |
| `oneOf`     | interface                         | union         | discriminated union |

## Field Properties

| Prop        | Required | Notes |
|-------------|----------|-------|
| `fnum`      | ✅ | Stable, unique-within-scope. Maps to protobuf field number. **Never reuse.** |
| `type`      | ✅ | One of the types above. |
| `repeated`  |   | `true` for `[]T` / arrays. |
| `tags`      |   | e.g. `["required"]`. |
| `alias`     |   | CLI flag alias for the field. |
| `$ref`      |   | Required for `recRef`: `"#/recordsDefs/Name"`. |
| `valueType` |   | Required for `map`: e.g. `"string"`, `"int32"`. |
| `title`     |   | Short label (used in CLI help). |
| `desc`      |   | Long description. |

## Field Number Discipline

- Every field needs a `fnum`.
- Unique **within a scope**. Each of the following is its own scope: a command's `fields`, a command's `resultFields`, each record's `fields`.
- **Never reuse** a previously-used `fnum`, even after deleting the field. Always increment past the highest historical number.
- Field numbers are wire-format identity — changing one is a breaking change.

## Naming Conventions

- Command names: PascalCase + `Cmd` suffix → `UserRegisterCmd`, `OrderCreateCmd`.
- Record names: PascalCase → `User`, `OrderItem`.
- CLI aliases: snake_case → `"alias": "register"`, `"alias": "create_order"`.

## Format Choice

- **`plain`** — Simpler. Go structs with JSON tags. No protobuf dependency. Good default for REST/JSON APIs.
- **`pb3`** — Full protobuf. Use when you need binary serialization, gRPC, or strict schema-evolution guarantees.
