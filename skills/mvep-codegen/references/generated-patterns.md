# Generated Patterns Reference

What `mvep generate -lang go -format=plain` emits, and how to wire it up at runtime.

## Output Layout

```
<service>/go/
├── <service>_impl.go        # NOMVEP — runXxxCmd functions (you write these)
├── <service>_commands.go    # generated — GetCommandRunner() factory
├── api/
│   ├── <service>.plain.go   # generated — command/result/record structs
│   └── <service>_package.go # generated — handlers, PkgCommandRunner, dispatch
└── cmd/<service>/
    └── <service>_main_cmd.go # generated (runtime mode) or NOMVEP (custom) — CLI main
```

With `--format=pb3`:

```
api/
├── <service>.proto          # generated — protobuf definition
├── <service>.pb.go          # generated — compiled protobuf
└── <service>_package.go     # generated — handlers, dispatch
```

## Handler Types (in `api/*_package.go`)

For every command, the generator emits a typed handler signature:

```go
type UserRegisterCmdHandler   func(context.Context, *UserRegisterCmd)   (*UserRegisterCmdResult,   error)
type UserGetProfileCmdHandler func(context.Context, *UserGetProfileCmd) (*UserGetProfileCmdResult, error)
```

## `PkgCommandRunner` (in `api/*_package.go`)

Holds one handler per command; implements `mvep.CommandRunner`:

```go
type PkgCommandRunner struct {
    RunUserRegisterCmd   UserRegisterCmdHandler
    RunUserGetProfileCmd UserGetProfileCmdHandler
}

func (r *PkgCommandRunner) RunCmd(ctx context.Context, cmd any) (any, error) {
    switch c := cmd.(type) {
    case *UserRegisterCmd:   return r.RunUserRegisterCmd(ctx, c)
    case *UserGetProfileCmd: return r.RunUserGetProfileCmd(ctx, c)
    }
    return nil, fmt.Errorf("unknown command type: %T", cmd)
}
```

## `Package` Interface (in `api/*_package.go`)

The package implements `mvep.Package` via `mvep.NewPackageFromDesc`, which derives
`InstanceOf`/`NameOf`/`CommandNames` from a `PackageDesc` literal rather than
emitting hand-written switch statements:

```go
var pkgDesc = mvep.PackageDesc{
    Name:        "myservice",
    // ...
    Commands: []mvep.CommandDesc{ /* ... */ },
    Records:  []mvep.RecordDesc{ /* ... */ },
}

var pkg = mvep.NewPackageFromDesc(&pkgDesc)

func NewPackage() mvep.Package { return pkg }
func Describe() *mvep.PackageDesc { return &pkgDesc }

func InstanceOf(compName string) (any, bool) { return pkg.InstanceOf(compName) }
func NameOf(comp any) string                  { return pkg.NameOf(comp) }
```

`GetName()` still returns `Name + "Package"` (e.g. `"myservicePackage"`) to
preserve HTTP routing. The descriptor is emitted by codegen from the same run
that produces the command structs, so they cannot disagree.
`FieldDesc.Ptr` closes over a real struct field — a codegen mistake is a
compile error, not a silent runtime drop.

## CLI Entry Point (in `cmd/<service>/*_main_cmd.go`)

The CLI mode is controlled by `gen_options.cli`:

| Mode | Behaviour |
|------|-----------|
| `runtime` (default) | Emits a descriptor-driven `cli.New(api.Describe(), ...)` main |
| `legacy` | Emits the old hand-wired `prepareXxxCmd` pattern |
| `none` | No CLI main generated |

`skipCmd=true` forces `none` regardless of the genopt. The generated `runtime`
main is:

```go
func main() {
    app := cli.New(api.Describe(), &cli.LocalExecutor{Runner: runner})
    app.Root().Version = resolveVersion()
    err := app.Run(context.Background())
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(cli.ExitCode(err))
    }
}
```

Add `// NOMVEP` as the first line to protect a hand-customized entry point.
The `mvep/cli` library provides flag binding (all `FieldType`s via `Ptr`),
required-flag enforcement, pre/post hooks, custom renderers, and exit-code
classification. See `runtime/go/mvep/cli/README.md` for the full guide.

## `GetCommandRunner` Factory (in `*_commands.go`)

Generated — wires impl functions to the runner:

```go
func GetCommandRunner() *api.PkgCommandRunner {
    return &api.PkgCommandRunner{
        RunUserRegisterCmd:   runUserRegisterCmd,
        RunUserGetProfileCmd: runUserGetProfileCmd,
    }
}
```

If a `runXxxCmd` is referenced here but missing from `*_impl.go`, the build fails — that's the signal to add the impl.

## Implementation Stubs (in `*_impl.go`, NOMVEP)

```go
// NOMVEP
package myservice

import (
    "context"
    "github.com/acme/myservice/mvepapi/go/api"
)

func runUserRegisterCmd(ctx context.Context, cmd *api.UserRegisterCmd) (*api.UserRegisterCmdResult, error) {
    // your business logic
    return &api.UserRegisterCmdResult{UserID: "...", Token: "..."}, nil
}
```

Signature is fixed: `func(ctx, *api.XxxCmd) (*api.XxxCmdResult, error)`.

## Runtime: `mvep`

`github.com/mainvec/mvep/runtime/go/mvep` provides the runtime infrastructure.

### Core Interfaces

```go
type Package interface {
    GetName() string
    InstanceOf(name string) (any, bool)
    NameOf(comp any) string
}

type CommandRunner interface {
    RunCmd(ctx context.Context, cmd any) (any, error)
}
```

### Envelope

```go
type CmdReq  struct { Cmd any; Headers map[string]string; Payload []byte }
type CmdResp struct { Headers map[string]string; Payload []byte; Error *ErrorInfo }
```

HTTP transport prefixes header keys with `x-mvep-`.

### `PackageHandler`

Bridges package + runner + transport:

```go
handler := mvep.NewPackageHandler(pkg, transporter, runner, interceptor)
handler.ServeCmdReq(ctx, req)  // server side
handler.SendCmdReq(ctx, req)   // client side
```

### Interceptors / Middleware

```go
type CmdHandler     func(ctx, *CmdReq) *CmdResp
type CmdInterceptor func(ctx, *CmdReq, next CmdHandler) *CmdResp
```

Built-ins: `LoggingInterceptor`, `AuthInterceptor(validator)`, `RecoveryInterceptor`, `RequestIDInterceptor`.

Composition:

```go
chain := mvep.Chain(
    mvep.RecoveryInterceptor(),
    mvep.LoggingInterceptor(),
    mvep.AuthInterceptor(validator),
)

// Skip auth on public commands
auth := mvep.SkipCommands(mvep.AuthInterceptor(v), "UserRegisterCmd", "UserLoginCmd")

// Apply only to specific commands
admin := mvep.OnlyCommands(adminCheckInterceptor, "AdminDeleteUserCmd")
```

### Server

```go
import (
    "context"
    "log"
    "os/signal"
    "syscall"
    "time"

    mvepapi "github.com/acme/myservice/mvepapi/go"
    "github.com/acme/myservice/mvepapi/go/api"
    "github.com/mainvec/mvep/runtime/go/mvep"
    "github.com/mainvec/mvep/runtime/go/mvep/server"
)

func main() {
    srv, err := server.NewServer(&server.ServerConfig{
        Listeners:         []server.ListenerConfig{{Address: ":8080"}},
        BasePath:          "/api",
        EnableHealthCheck: true,
        EnableCORS:        true,
        AllowedOrigins:    []string{"https://app.example.com"}, // required for CORS headers to be emitted
        MaxRequestBytes:   4 << 20,                             // default 4 MiB; oversized bodies get 413
        VerboseErrors:     false,                                // true only for local dev
        Interceptor: mvep.Chain(
            mvep.RecoveryInterceptor(),
            mvep.LoggingInterceptor(),
            mvep.RequestIDInterceptor(nil),
        ),
    })
    if err != nil {
        log.Fatal(err)
    }

    if err := srv.RegisterPackage(api.NewPackage(), mvepapi.GetCommandRunner()); err != nil {
        log.Fatal(err)
    }

    // StartAsync binds all listeners and returns once they're serving.
    if err := srv.StartAsync(); err != nil {
        log.Fatal(err)
    }

    // The server installs no signal handlers — the app owns shutdown.
    signalCtx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()
    select {
    case <-signalCtx.Done():
        ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
        defer cancel()
        _ = srv.ShutdownContext(ctx) // drains active requests
    case <-srv.Done():
        log.Printf("server stopped: %v", srv.Err())
    }
}
```

`NewServer` copies `ServerConfig` and returns `(*Server, error)`. A `Server` is
single-use — construct a new one instead of restarting. `ServerConfig.OnShutdown`
was removed (runtime v0.8.0); sequence cleanup explicitly around `ShutdownContext`.

**v0.9.0 HTTP hardening:** non-`POST` requests get `405`; unregistered
`Content-Type`s get `415`; error responses are redacted by default (raw detail
only server-side logged, unless `VerboseErrors: true`); `EnableCORS` with empty
`AllowedOrigins` emits no CORS headers (fail closed, not `*`); `LocalTrustMiddleware`
verifies the peer (Unix socket or loopback TCP) before trusting a request enough
to skip `AuthInterceptor`.

### Unix Socket Server

```go
srv, err := server.NewServer(&server.ServerConfig{
    Listeners: []server.ListenerConfig{{Address: "unix:///tmp/myservice.sock"}},
})
```
