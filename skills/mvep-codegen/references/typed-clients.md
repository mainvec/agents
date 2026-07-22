# Typed Client Patterns

Generated MVEP code gives you raw types and a `Package`. To consume them ergonomically, wrap the runtime client in a typed struct.

## Go Client (`mvep/client`)

### Plain Use

```go
import (
    "github.com/mainvec/mvep/runtime/go/mvep/client"
    "github.com/acme/myservice/mvepapi/go/api"
)

c, err := client.NewClient(client.ClientConfig{
    BaseURL:  "http://localhost:8080",          // or "unix:///path/to/sock"
    BasePath: "/api",
    Timeout:  30 * time.Second,
})
if err != nil { return err }
defer c.Close()

pkg := api.NewPackage()
pc, err := c.RegisterPackage(pkg)
if err != nil { return err }

// Without headers
result, err := pc.SendCmd(ctx, &api.UserRegisterCmd{
    Email: "alice@example.com", Password: "s3cret", Name: "Alice",
})
reg := result.(*api.UserRegisterCmdResult)

// With headers (returns typed result + envelope)
result, resp, err := pc.SendCmdReq(ctx, &api.UserGetProfileCmd{
    UserID: reg.UserID,
}, map[string]string{"auth": reg.Token})
```

`BaseURL` auto-detects the `unix://` scheme and configures HTTP transport accordingly.

### `ClientConfig`

```go
type ClientConfig struct {
    BaseURL     string                 // Required
    BasePath    string                 // URL path prefix
    Encoder     string                 // default "application/json"
    Timeout     time.Duration          // default 30s
    HTTPClient  *http.Client           // optional
    Interceptor mvep.ClientInterceptor  // optional
}
```

### Built-in Client Interceptors

| Interceptor | Description |
|-------------|-------------|
| `AuthHeaderInterceptor(tokenProvider)` | Dynamic auth header. `TokenProvider = func(ctx) (string, error)` |
| `StaticAuthHeaderInterceptor(token)` | Fixed `auth` header on every request |
| `HeaderInterceptor(headers)` | Custom static headers |
| `ClientLoggingInterceptor()` | Logs request/response timing via `slog` |
| `RetryInterceptor(maxRetries, delay)` | Retries on transport errors |
| `ClientRequestIDInterceptor(generator)` | Adds `request-id` header |
| `SkipCommandsClient(interceptor, cmds...)` | Skip interceptor for listed commands |

### Typed Wrapper Pattern

```go
type MyClient struct {
    raw       *client.Client
    pc        *client.PackageClient
    mu        sync.RWMutex
    authToken string
}

func New(cfg Config) (*MyClient, error) {
    mc := &MyClient{}
    tp := func(ctx context.Context) (string, error) {
        mc.mu.RLock(); defer mc.mu.RUnlock()
        return mc.authToken, nil
    }
    c, err := client.NewClient(client.ClientConfig{
        BaseURL: cfg.BaseURL,
        Timeout: cfg.Timeout,
        Interceptor: mvep.ChainClient(
            mvep.ClientLoggingInterceptor(),
            mvep.AuthHeaderInterceptor(tp),
            mvep.RetryInterceptor(3, time.Second),
        ),
    })
    if err != nil { return nil, err }
    pc, err := c.RegisterPackage(api.NewPackage())
    if err != nil { c.Close(); return nil, err }
    mc.raw, mc.pc = c, pc
    return mc, nil
}

func (c *MyClient) Close() error { return c.raw.Close() }

func (c *MyClient) SetAuthToken(t string) {
    c.mu.Lock(); defer c.mu.Unlock(); c.authToken = t
}

// ── Typed command methods ──

func (c *MyClient) RegisterUser(ctx context.Context, email, password, name string) (*api.UserRegisterCmdResult, error) {
    r, _, err := c.pc.SendCmdReq(ctx, &api.UserRegisterCmd{
        Email: email, Password: password, Name: name,
    }, nil)
    if err != nil { return nil, err }
    res := r.(*api.UserRegisterCmdResult)
    if res.Token != "" { c.SetAuthToken(res.Token) }
    return res, nil
}

func (c *MyClient) GetUserProfile(ctx context.Context, userID string) (*api.UserGetProfileCmdResult, error) {
    r, _, err := c.pc.SendCmdReq(ctx, &api.UserGetProfileCmd{UserID: userID}, nil)
    if err != nil { return nil, err }
    return r.(*api.UserGetProfileCmdResult), nil
}
```

## TypeScript Client (`@mainvec/mvep`)

The generated `js/api/` contains:

```
js/api/
├── <service>.js         # JS classes (constructors, verify, fromObject, toObject, toJSON)
├── <service>.d.ts       # TS interfaces + classes
└── <service>_package.js # PACKAGE_NAME, instanceOf(name), nameOf(cmd)
```

Hand-written client code lives in `js/api/client/` (not regenerated).

### Generated JS Class Shape

```js
ns.UserRegisterCmd = class UserRegisterCmd {
  static _typeName = 'UserRegisterCmd';
  constructor(data = {}) {
    this.email    = data.email    ?? '';
    this.password = data.password ?? '';
    this.name     = data.name     ?? '';
  }
  static verify(message)    { /* validates fields */ }
  static fromObject(obj)    { /* creates instance from plain object */ }
  static toObject(message)  { /* converts to plain object */ }
  toJSON()                  { /* serializes to JSON string */ }
};
```

### Package Adapter

```ts
// js/api/client/myservice_package.ts
import type { Package } from '@mainvec/mvep';
import * as pkg from '../myservice_package.js';

export class MyServicePackage implements Package {
    getName() { return pkg.PACKAGE_NAME; }
    instanceOf(name: string) { return pkg.instanceOf(name) ?? undefined; }
    nameOf(cmd: unknown) { return pkg.nameOf(cmd); }
}
```

### Typed Client with Auth & Storage

```ts
import {
    newClient, chainClient,
    type Client, type PackageClient, type ClientInterceptor,
} from '@mainvec/mvep';
import { MyServicePackage } from './myservice_package';
import { myservicens } from '../myservice';
import type { myservicens as types } from '../myservice';

function authHeader(getToken: () => string): ClientInterceptor {
    return async (ctx, req, next) => {
        const t = getToken();
        if (t) (req.headers ??= {})['auth'] = t;  // sent as x-mvep-auth
        return next(ctx, req);
    };
}

export interface MyClientConfig {
    baseUrl: string;
    basePath?: string;
    storageType?: 'localStorage' | 'sessionStorage' | 'none';
    timeout?: number;
}

export class MyClient {
    private constructor(
        private pc: PackageClient,
        private storageType: NonNullable<MyClientConfig['storageType']>,
        private authToken: string,
    ) {}

    static async create(cfg: MyClientConfig): Promise<MyClient> {
        const storageType = cfg.storageType ?? 'localStorage';
        let token = '';
        if (storageType !== 'none' && typeof window !== 'undefined') {
            const s = storageType === 'sessionStorage' ? sessionStorage : localStorage;
            token = s.getItem('myservice_auth') ?? '';
        }
        const c = newClient({
            baseUrl: cfg.baseUrl,
            basePath: cfg.basePath,
            timeout: cfg.timeout,
            interceptor: chainClient(authHeader(() => token)),
        });
        const pc = c.registerPackage(new MyServicePackage());
        const mc = new MyClient(pc, storageType, token);
        // capture token mutations
        const orig = mc.setAuthToken.bind(mc);
        mc.setAuthToken = (t: string) => { orig(t); token = t; };
        return mc;
    }

    setAuthToken(t: string) {
        this.authToken = t;
        if (this.storageType !== 'none' && typeof window !== 'undefined') {
            const s = this.storageType === 'sessionStorage' ? sessionStorage : localStorage;
            s.setItem('myservice_auth', t);
        }
    }

    isAuthenticated() { return this.authToken !== ''; }

    // ── Typed command methods ──

    async registerUser(email: string, password: string, name: string): Promise<types.UserRegisterCmdResult> {
        const cmd = new myservicens.UserRegisterCmd({ email, password, name });
        const res = await this.pc.sendCmd<types.UserRegisterCmdResult>(cmd);
        if (res.token) this.setAuthToken(res.token);
        return res;
    }

    async getUserProfile(userID: string): Promise<types.UserGetProfileCmdResult> {
        const cmd = new myservicens.UserGetProfileCmd({ userID });
        return this.pc.sendCmd<types.UserGetProfileCmdResult>(cmd);
    }
}
```

### Recommended Client Directory Layout

```
js/api/
├── <service>.js                  # ⛔ generated
├── <service>.d.ts                # ⛔ generated
├── <service>_package.js          # ⛔ generated
└── client/                       # ✏️ hand-written
    ├── <service>_client.ts       # typed client with auth & methods
    ├── <service>_package.ts      # mvep Package adapter
    └── index.ts                  # barrel exports
```

### Header / Auth Convention

- Headers added by the client are prefixed with `x-mvep-` over HTTP.
- The `auth` header is the convention for bearer tokens; server-side `AuthInterceptor` reads it via a `TokenValidator` implementation.
