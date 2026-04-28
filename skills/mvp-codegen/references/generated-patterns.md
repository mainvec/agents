# Generated Patterns Reference

What `mvp generate -lang go -format=plain` emits, and how to wire it up at runtime.

## Output Layout

```
<service>/go/
├── <service>_impl.go        # NOMVGEN — runXxxCmd functions (you write these)
├── <service>_commands.go    # generated — GetCommandRunner() factory
├── api/
│   ├── <service>.plain.go   # generated — command/result/record structs
│   └── <service>_package.go # generated — handlers, PkgCommandRunner, dispatch
└── cmd/<service>/
    └── <service>_main_cmd.go # generated — CLI main
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

Holds one handler per command; implements `mvp.CommandRunner`:

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

Implements `mvp.Package`:

```go
func (p *myservicePackage) GetName() string { return "myservicePackage" }

func (p *myservicePackage) InstanceOf(name string) (any, bool) {
    switch name {
    case "UserRegisterCmd":       return &UserRegisterCmd{}, true
    case "UserRegisterCmdResult": return &UserRegisterCmdResult{}, true
    }
    return nil, false
}

func (p *myservicePackage) NameOf(comp any) string {
    switch comp.(type) {
    case *UserRegisterCmd: return "UserRegisterCmd"
    }
    return ""
}
```

Use via `api.NewPackage()`.

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

## Implementation Stubs (in `*_impl.go`, NOMVGEN)

```go
// NOMVGEN
package myservice

import (
    "context"
    "github.com/acme/myservice/mvpapi/go/api"
)

func runUserRegisterCmd(ctx context.Context, cmd *api.UserRegisterCmd) (*api.UserRegisterCmdResult, error) {
    // your business logic
    return &api.UserRegisterCmdResult{UserID: "...", Token: "..."}, nil
}
```

Signature is fixed: `func(ctx, *api.XxxCmd) (*api.XxxCmdResult, error)`.

## Runtime: `mvpgo`

`github.com/mainvec/mvp/mvpgo` provides the runtime infrastructure.

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

HTTP transport prefixes header keys with `x-mvp-`.

### `PackageHandler`

Bridges package + runner + transport:

```go
handler := mvp.NewPackageHandler(pkg, transporter, runner, interceptor)
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
chain := mvp.Chain(
    mvp.RecoveryInterceptor(),
    mvp.LoggingInterceptor(),
    mvp.AuthInterceptor(validator),
)

// Skip auth on public commands
auth := mvp.SkipCommands(mvp.AuthInterceptor(v), "UserRegisterCmd", "UserLoginCmd")

// Apply only to specific commands
admin := mvp.OnlyCommands(adminCheckInterceptor, "AdminDeleteUserCmd")
```

### Server

```go
import (
    mvpapi "github.com/acme/myservice/mvpapi/go"
    "github.com/acme/myservice/mvpapi/go/api"
    "github.com/mainvec/mvp/mvpgo/mvp"
    "github.com/mainvec/mvp/mvpgo/mvp/server"
)

func main() {
    pkg     := api.NewPackage()
    runner  := mvpapi.GetCommandRunner()
    handler := mvp.NewPackageHandler(pkg, nil, runner,
        mvp.Chain(
            mvp.RecoveryInterceptor(),
            mvp.LoggingInterceptor(),
            mvp.RequestIDInterceptor(nil),
        ),
    )

    srv := server.NewServer(server.ServerConfig{
        Addr:         ":8080",
        BasePath:     "/api",
        EnableHealth: true,
        EnableCORS:   true,
    }, handler)
    srv.Start()
}
```

### Unix Socket Server

```go
srv := server.NewServer(server.ServerConfig{
    Addr: "unix:///tmp/myservice.sock",
}, handler)
```
