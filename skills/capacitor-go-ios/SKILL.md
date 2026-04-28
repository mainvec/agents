---
name: capacitor-go-ios
description: "Bootstrap a Capacitor 7/8 + Vue 3 + Go CGo iOS app with NetworkExtension VPN support. USE FOR: creating mobile apps that embed Go static libraries; setting up iOS projects with Capacitor and Go XCFrameworks; fixing blank screen issues with Go frameworks in Capacitor; configuring NEPacketTunnelProvider extensions; setting up two-process iOS architecture (app + extension); upgrading Capacitor and frontend dependencies. DO NOT USE FOR: pure web apps without native Go code; Android-only projects; desktop apps."
argument-hint: "Describe the mobile app you want to create (e.g., 'VPN app with Go daemon')"
---

# Capacitor 7/8 + Go CGo iOS App Bootstrap

## When to Use

- Creating a mobile iOS app that embeds a Go daemon/library via CGo
- Setting up Capacitor with a Go XCFramework (c-archive)
- Adding a NetworkExtension (VPN/packet tunnel) to an iOS app
- Debugging blank screens in Capacitor apps that link Go static libraries
- Upgrading Capacitor and frontend dependencies
- Two-process architecture: main app + NetworkExtension tunnel

## Tested Dependency Versions (April 2026)

| Package | Version | Notes |
|---------|---------|-------|
| Capacitor (core/ios/cli) | 8.3.0 | SPM via `capacitor-swift-pm` |
| Vue | 3.5.32 | |
| Vue Router | 5.0.4 | |
| Vite | 8.0.3 | |
| @vitejs/plugin-vue | 6.0.5 | |
| Tailwind CSS | 4.2.2 | |
| @tailwindcss/vite | 4.2.2 | |
| TypeScript | 6.0.2 | |
| vue-tsc | 3.2.6 | |
| Xcode | 26.0.1 | iOS SDK 26.0 |
| Go | 1.24.3+ | CGo with c-archive |

## Critical Knowledge

### Go + Capacitor Blank Screen Fix

When a Go static library (c-archive XCFramework) is linked into a Capacitor iOS app, the default `CAPBridgeViewController` fails to route web assets correctly, causing a **blank white screen**. This is a known issue (https://github.com/ionic-team/capacitor/issues/7844).

**Required fix**: Create a custom ViewController with a `GoCompatibleRouter`:

```swift
import UIKit
import Capacitor

struct GoCompatibleRouter: Router {
    var basePath: String = ""
    func route(for path: String) -> String {
        let safePath = path.isEmpty ? "/" : path
        let pathUrl = URL(fileURLWithPath: safePath)
        if pathUrl.pathExtension.isEmpty {
            return basePath + "/index.html"
        }
        return basePath + path
    }
}

class AppViewController: CAPBridgeViewController {
    override func router() -> any Router {
        GoCompatibleRouter()
    }
}
```

Then update `Main.storyboard` to use `AppViewController` with `customModule="App"` instead of `CAPBridgeViewController`/`Capacitor`.

### Scene Lifecycle (Required for Capacitor 7/8)

Capacitor 7/8 expects the UIScene-based app lifecycle. Projects must have:

1. **SceneDelegate.swift** with `UIWindowSceneDelegate`
2. **AppDelegate.swift** with `configurationForConnecting` and `didDiscardSceneSessions`
3. **Info.plist** with `UIApplicationSceneManifest` pointing to SceneDelegate
4. No `var window: UIWindow?` in AppDelegate (it belongs in SceneDelegate)

### Go CGo XCFramework Build

Build a universal iOS framework using `CGO_ENABLED=1` and `-buildmode=c-archive`:

- **Device**: `GOOS=ios GOARCH=arm64` with iPhoneOS SDK
- **Simulator**: `GOOS=ios GOARCH=arm64` + `GOARCH=amd64` with iPhoneSimulator SDK, then `lipo` to create universal binary
- Use `xcodebuild -create-xcframework` to bundle both into `.xcframework`
- Link with `-lresolv` (Go's DNS resolver needs `res_9_*` symbols from libresolv)

### NetworkExtension Setup

- Tunnel extension is a **separate process** — it needs its own copy of shared Swift files and its own bridging header
- Both App and Extension targets need the **Network Extensions** capability with Packet Tunnel checked
- Both targets need the same **App Group** for IPC via `UserDefaults(suiteName:)`
- The extension bundle ID must be a child of the app's (e.g., `com.example.app.tunnel`)

## Procedure

Follow the [bootstrap guide](./references/bootstrap-guide.md) for the complete step-by-step process.

### Quick Reference — File Structure

```
my-app-mobile/
├── package.json
├── tsconfig.json              # Vue project references pattern
├── tsconfig.app.json          # Extends @vue/tsconfig/tsconfig.dom.json
├── tsconfig.node.json         # Extends @tsconfig/node22/tsconfig.json
├── vite.config.ts             # Vue + Tailwind CSS 4 plugins
├── capacitor.config.ts        # appId must match Xcode bundle ID
├── index.html
├── src/
│   ├── main.ts
│   ├── App.vue
│   ├── style.css              # @import "tailwindcss"
│   └── plugins/
│       └── my-plugin.ts       # registerPlugin<T>("PluginName")
├── gomobile/
│   ├── build_ios_cgo.sh       # XCFramework build script
│   ├── mylib/
│   │   ├── go.mod
│   │   └── mylib.go           # //export FunctionName with package main
│   └── MyLib.xcframework/     # Built output
└── ios/
    └── App/
        ├── App.xcodeproj/
        └── App/
            ├── AppDelegate.swift          # Scene lifecycle
            ├── SceneDelegate.swift         # UIWindowSceneDelegate
            ├── AppViewController.swift     # GoCompatibleRouter
            ├── MyBridge.swift              # Swift ↔ C FFI wrapper
            ├── MyPlugin.swift              # CAPPlugin + CAPBridgedPlugin
            ├── Info.plist                  # UIApplicationSceneManifest
            ├── Base.lproj/
            │   └── Main.storyboard        # Points to AppViewController
            └── public/                    # Capacitor web assets (synced)
```

### Quick Reference — Common Pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| Blank white screen | Go framework breaks Capacitor path routing | Add GoCompatibleRouter custom ViewController |
| Blank white screen | Missing scene lifecycle | Add SceneDelegate + UIApplicationSceneManifest |
| `res_9_*` undefined symbols | Go DNS resolver needs libresolv | Add `-lresolv` to OTHER_LDFLAGS |
| Extension can't find bridge | Extension is separate process | Add shared Swift files to extension target too |
| `npx vite` serves 404 | npx picks up wrong Vite version from cache | Use `node_modules/.bin/vite` or ensure correct cwd |
| `missing required module 'MobileCoreServices'` | Stale SPM build cache after Capacitor upgrade on Xcode 26 | Clean build folder (Cmd+Shift+K, then Cmd+B) or `xcodebuild clean` |
| "Plugin" not implemented on ios | Auto-discovery doesn't work with custom ViewController | Register plugin in `capacitorDidLoad()` via `bridge?.registerPluginInstance()` |
| App Group mismatch | Different IDs in entitlements vs Swift code | Grep for appGroupId, ensure all match |
| Bundle ID unavailable | Globally taken on App Store | Try without dots in suffix (e.g., `iulinkmobile` not `iulink.mobile`) |

### Quick Reference — Upgrading Dependencies

All dependencies can be upgraded independently. Tested upgrade path:

```bash
# 1. Upgrade Capacitor
npm install @capacitor/core@latest @capacitor/ios@latest @capacitor/cli@latest

# 2. Upgrade Vue ecosystem  
npm install vue@latest vue-router@latest

# 3. Upgrade build tools
npm install -D vite@latest @vitejs/plugin-vue@latest tailwindcss@latest \
  @tailwindcss/vite@latest typescript@latest vue-tsc@latest

# 4. Rebuild and sync
npm run build && npx cap sync ios

# 5. Clean build in Xcode (important after Capacitor major upgrade)
# Cmd+Shift+K to clean, then Cmd+B to build
```

**Capacitor 7 → 8 notes:**
- Minimal iOS breaking changes; SPM `Package.swift` updates automatically via `cap sync`
- Requires Xcode 26+ and iOS 15.0+ deployment target
- After upgrading, always do a **clean build** in Xcode — stale SPM cache causes `missing required module 'MobileCoreServices'` errors
- Plugin pattern (`CAPBridgedPlugin` protocol) is unchanged between Cap 7 and 8

### Quick Reference — Capacitor Plugin Pattern (Capacitor 7/8)

```swift
import Capacitor

@objc(MyPlugin)
public class MyPlugin: CAPPlugin, CAPBridgedPlugin {
    public let identifier = "MyPlugin"
    public let jsName = "MyPlugin"
    public let pluginMethods: [CAPPluginMethod] = [
        CAPPluginMethod(name: "doSomething", returnType: CAPPluginReturnPromise),
    ]

    @objc func doSomething(_ call: CAPPluginCall) {
        call.resolve(["result": "ok"])
    }
}
```

No manual registration needed — Capacitor 7/8 auto-discovers plugins via `CAPBridgedPlugin` protocol **only when using the default `CAPBridgeViewController`**. With a custom ViewController (required for GoCompatibleRouter), you must explicitly register plugins:

```swift
class AppViewController: CAPBridgeViewController {
    override func router() -> any Router {
        GoCompatibleRouter()
    }

    override func capacitorDidLoad() {
        bridge?.registerPluginInstance(MyPlugin())
    }
}
```
