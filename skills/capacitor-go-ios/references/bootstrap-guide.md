# Capacitor 7/8 + Vue 3 + Go CGo iOS App — Bootstrap Guide

Complete step-by-step guide for creating an iOS app that embeds a Go daemon via CGo, with a Capacitor + Vue 3 frontend and optional NetworkExtension VPN tunnel.

---

## Prerequisites

- macOS with Xcode 26+ (or Xcode 16+ for Capacitor 7)
- Go 1.24+ with CGo support
- Node.js 22+
- Apple Developer account (for device builds and Network Extension entitlement)

---

## Phase 1: Vue + Capacitor Project Setup

### 1.1 Initialize the Project

```bash
mkdir my-app-mobile && cd my-app-mobile
npm init -y
npm install vue@^3.5 vue-router@^5.0
npm install -D vite@^8.0 @vitejs/plugin-vue@^6.0 vue-tsc@^3.2 typescript@^6.0
npm install -D @tailwindcss/vite@^4.2 tailwindcss@^4.2
npm install @capacitor/core@^8.3 @capacitor/ios@^8.3
npm install -D @capacitor/cli@^8.3
```

### 1.2 TypeScript Configuration

Use the **Vue project references pattern** — this is important for proper type checking:

**tsconfig.json** (root — references only):
```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.node.json" },
    { "path": "./tsconfig.app.json" }
  ]
}
```

**tsconfig.app.json** (app sources):
```json
{
  "extends": "@vue/tsconfig/tsconfig.dom.json",
  "include": ["env.d.ts", "src/**/*", "src/**/*.vue"],
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

**tsconfig.node.json** (build tools):
```json
{
  "extends": "@tsconfig/node22/tsconfig.json",
  "include": ["vite.config.*"],
  "compilerOptions": {
    "noEmit": true,
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.node.tsbuildinfo",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "types": ["node"]
  }
}
```

Install the tsconfig presets:
```bash
npm install -D @vue/tsconfig @tsconfig/node22
```

### 1.3 Vite Configuration

**vite.config.ts:**
```typescript
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [vue(), tailwindcss()],
  resolve: {
    alias: {
      "@": new URL("./src", import.meta.url).pathname,
    },
  },
});
```

### 1.4 Create Source Files

**index.html:**
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover" />
    <title>My App</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

**src/style.css:**
```css
@import "tailwindcss";
```

**src/main.ts:**
```typescript
import { createApp } from "vue";
import App from "./App.vue";
import "./style.css";
createApp(App).mount("#app");
```

**src/App.vue** — create your Vue app component.

### 1.5 Add npm Scripts

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "preview": "vite preview"
  }
}
```

### 1.6 Capacitor Configuration

**capacitor.config.ts:**
```typescript
import type { CapacitorConfig } from "@capacitor/cli";

const config: CapacitorConfig = {
  appId: "com.yourcompany.yourapp",
  appName: "YourApp",
  webDir: "dist",
  ios: {
    preferredContentMode: "mobile",
    scrollEnabled: false,
  },
  // Uncomment for hot-reload during development:
  // server: {
  //   url: 'http://YOUR_MAC_IP:5173',
  //   cleartext: true,
  // },
};

export default config;
```

> **IMPORTANT**: The `appId` MUST match the bundle identifier in Xcode. If you change it in Xcode, update it here and run `npx cap sync ios`.

### 1.7 Initialize Capacitor iOS

```bash
npm run build
npx cap add ios
npx cap sync ios
```

### 1.8 Verify Web Build Works

```bash
npm run dev
# Open http://localhost:5173 — should show your UI
```

> **Warning**: If using `npx vite` from a parent workspace directory, `npx` may pick up a cached different Vite version. Always run from the project directory, or use `node_modules/.bin/vite` directly.

---

## Phase 2: Go CGo Static Library

### 2.1 Create Go Module

```bash
mkdir -p gomobile/mylib
cd gomobile/mylib
go mod init mylib
```

### 2.2 Write Exported Functions

**gomobile/mylib/mylib.go:**
```go
//go:build !ios || cgo
package main

import "C"
import "unsafe"

// Global state
var myState *SomeState

//export MyLibStart
func MyLibStart(configDir *C.char) C.int {
    dir := C.GoString(configDir)
    // ... start your service ...
    return C.int(port)
}

//export MyLibStop
func MyLibStop() {
    // ... stop your service ...
}

//export MyLibFreeString
func MyLibFreeString(s *C.char) {
    C.free(unsafe.Pointer(s))
}

func main() {} // Required for c-archive
```

### 2.3 Build Script

**gomobile/build_ios_cgo.sh:**
```bash
#!/bin/bash
set -e

LIBNAME="mylib"
GOPACKAGE="./mylib"
OUTPUT_DIR="."
FRAMEWORK_NAME="MyLib"

# Detect SDK paths
IPHONEOS_SDK=$(xcrun --sdk iphoneos --show-sdk-path)
IPHONESIMULATOR_SDK=$(xcrun --sdk iphonesimulator --show-sdk-path)
IPHONEOS_SDK_VERSION=$(xcrun --sdk iphoneos --show-sdk-version)
IPHONESIMULATOR_SDK_VERSION=$(xcrun --sdk iphonesimulator --show-sdk-version)

WORK_DIR=$(mktemp -d)
trap "rm -rf $WORK_DIR" EXIT

echo "=== Building iOS device (arm64) ==="
CGO_ENABLED=1 \
GOOS=ios \
GOARCH=arm64 \
SDK=iphoneos \
CGO_CFLAGS="-fembed-bitcode -isysroot $IPHONEOS_SDK -mios-version-min=15.0 -arch arm64" \
CGO_LDFLAGS="-isysroot $IPHONEOS_SDK -mios-version-min=15.0 -arch arm64" \
CC=$(xcrun --sdk iphoneos -f clang) \
  go build -buildmode=c-archive -trimpath \
  -o "$WORK_DIR/device-arm64/lib${LIBNAME}.a" \
  $GOPACKAGE

echo "=== Building iOS simulator (arm64) ==="
CGO_ENABLED=1 \
GOOS=ios \
GOARCH=arm64 \
SDK=iphonesimulator \
CGO_CFLAGS="-isysroot $IPHONESIMULATOR_SDK -mios-simulator-version-min=15.0 -arch arm64 -target arm64-apple-ios15.0-simulator" \
CGO_LDFLAGS="-isysroot $IPHONESIMULATOR_SDK -mios-simulator-version-min=15.0 -arch arm64" \
CC=$(xcrun --sdk iphonesimulator -f clang) \
  go build -buildmode=c-archive -trimpath \
  -o "$WORK_DIR/sim-arm64/lib${LIBNAME}.a" \
  $GOPACKAGE

echo "=== Building iOS simulator (amd64) ==="
CGO_ENABLED=1 \
GOOS=ios \
GOARCH=amd64 \
SDK=iphonesimulator \
CGO_CFLAGS="-isysroot $IPHONESIMULATOR_SDK -mios-simulator-version-min=15.0 -arch x86_64" \
CGO_LDFLAGS="-isysroot $IPHONESIMULATOR_SDK -mios-simulator-version-min=15.0 -arch x86_64" \
CC=$(xcrun --sdk iphonesimulator -f clang) \
  go build -buildmode=c-archive -trimpath \
  -o "$WORK_DIR/sim-amd64/lib${LIBNAME}.a" \
  $GOPACKAGE

echo "=== Creating universal simulator binary ==="
mkdir -p "$WORK_DIR/sim-universal"
lipo -create \
  "$WORK_DIR/sim-arm64/lib${LIBNAME}.a" \
  "$WORK_DIR/sim-amd64/lib${LIBNAME}.a" \
  -output "$WORK_DIR/sim-universal/lib${LIBNAME}.a"
cp "$WORK_DIR/sim-arm64/lib${LIBNAME}.h" "$WORK_DIR/sim-universal/"

echo "=== Creating XCFramework ==="
rm -rf "$OUTPUT_DIR/${FRAMEWORK_NAME}.xcframework"
xcodebuild -create-xcframework \
  -library "$WORK_DIR/device-arm64/lib${LIBNAME}.a" \
  -headers "$WORK_DIR/device-arm64/" \
  -library "$WORK_DIR/sim-universal/lib${LIBNAME}.a" \
  -headers "$WORK_DIR/sim-universal/" \
  -output "$OUTPUT_DIR/${FRAMEWORK_NAME}.xcframework"

echo "=== Done: ${FRAMEWORK_NAME}.xcframework ==="
```

```bash
chmod +x gomobile/build_ios_cgo.sh
cd gomobile && bash build_ios_cgo.sh
```

---

## Phase 3: Xcode Project Configuration

### 3.1 Critical iOS Files

After `npx cap add ios`, you MUST create/modify these files before the app will work with a Go framework:

#### GoCompatibleRouter + Custom ViewController

**ios/App/App/AppViewController.swift:**
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

> **WHY**: The Go static library interferes with Capacitor's default file routing in WKWebView. Without this, every page request returns empty/404, causing a blank white screen.

#### SceneDelegate

**ios/App/App/SceneDelegate.swift:**
```swift
import UIKit
import Capacitor

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
        guard let _ = (scene as? UIWindowScene) else { return }
    }
}
```

#### AppDelegate (Scene Lifecycle)

**ios/App/App/AppDelegate.swift** — must include these methods:
```swift
func application(_ application: UIApplication, configurationForConnecting connectingSceneSession: UISceneSession, options: UIScene.ConnectionOptions) -> UISceneConfiguration {
    return UISceneConfiguration(name: "Default Configuration", sessionRole: connectingSceneSession.role)
}

func application(_ application: UIApplication, didDiscardSceneSessions sceneSessions: Set<UISceneSession>) {
}
```

Remove `var window: UIWindow?` from AppDelegate (it goes in SceneDelegate).

#### Main.storyboard

Change the view controller class from `CAPBridgeViewController`/`Capacitor` to your custom class:
```xml
<viewController id="BYZ-38-t0r" customClass="AppViewController" customModule="App" sceneMemberID="viewController"/>
```

#### Info.plist — Scene Manifest

Add `UIApplicationSceneManifest` inside the root `<dict>`:
```xml
<key>UIApplicationSceneManifest</key>
<dict>
    <key>UIApplicationSupportsMultipleScenes</key>
    <false/>
    <key>UISceneConfigurations</key>
    <dict>
        <key>UIWindowSceneSessionRoleApplication</key>
        <array>
            <dict>
                <key>UISceneConfigurationName</key>
                <string>Default Configuration</string>
                <key>UISceneDelegateClassName</key>
                <string>$(PRODUCT_MODULE_NAME).SceneDelegate</string>
                <key>UISceneStoryboardFile</key>
                <string>Main</string>
            </dict>
        </array>
    </dict>
</dict>
```

### 3.2 Link the XCFramework in Xcode

1. Open `ios/App/App.xcodeproj` in Xcode
2. Select the **App** target → **General** → **Frameworks, Libraries, and Embedded Content**
3. Click **+** → **Add Other** → **Add Files** → select `gomobile/MyLib.xcframework`
4. Set the framework to **Do Not Embed** (it's a static library)
5. Add `$(PROJECT_DIR)/../../gomobile` to **FRAMEWORK_SEARCH_PATHS** in Build Settings

### 3.3 Linker Flags

For both App and Extension targets, add to **OTHER_LDFLAGS**:
```
-lresolv
```

Go's DNS resolver uses `res_9_ninit`, `res_9_nsearch`, `res_9_nclose` from libresolv. Without this flag, you'll get undefined symbol linker errors.

### 3.4 Swift ↔ C Bridge

Create a bridging header or use the Xcode-managed one. Include the Go-generated header:

**App-Bridging-Header.h:**
```c
#include "libmylib.h"
```

Create a Swift wrapper singleton:

**MyBridge.swift:**
```swift
import Foundation

class MyBridge {
    static let shared = MyBridge()
    private init() {}

    func start(configDir: String) -> Int {
        let cDir = configDir.withCString { ptr in
            return Int(MyLibStart(UnsafeMutablePointer(mutating: ptr)))
        }
        return cDir
    }

    func stop() {
        MyLibStop()
    }
}
```

### 3.5 Capacitor Plugin

**MyPlugin.swift:**
```swift
import Capacitor

@objc(MyPlugin)
public class MyPlugin: CAPPlugin, CAPBridgedPlugin {
    public let identifier = "MyPlugin"
    public let jsName = "MyPlugin"
    public let pluginMethods: [CAPPluginMethod] = [
        CAPPluginMethod(name: "start", returnType: CAPPluginReturnPromise),
        CAPPluginMethod(name: "stop", returnType: CAPPluginReturnPromise),
    ]

    @objc func start(_ call: CAPPluginCall) {
        let result = MyBridge.shared.start(configDir: getDataDir())
        call.resolve(["port": result])
    }

    @objc func stop(_ call: CAPPluginCall) {
        MyBridge.shared.stop()
        call.resolve(["status": "stopped"])
    }
}
```

No manual plugin registration needed — Capacitor 7/8 auto-discovers via `CAPBridgedPlugin`.

**TypeScript definition (src/plugins/my-plugin.ts):**
```typescript
import { registerPlugin } from "@capacitor/core";

export interface MyPluginInterface {
  start(): Promise<{ port: number }>;
  stop(): Promise<{ status: string }>;
}

const MyPlugin = registerPlugin<MyPluginInterface>("MyPlugin");
export default MyPlugin;
```

---

## Phase 4: NetworkExtension (Optional — VPN/Tunnel)

### 4.1 Add Extension Target

In Xcode: File → New → Target → **Network Extension** → configure:
- Product Name: `MyTunnel`
- Bundle ID: `com.yourcompany.yourapp.tunnel` (child of app bundle ID)
- Team: same as app

### 4.2 Extension Configuration

Both targets need:
- **App Groups** capability with same group ID (e.g., `group.com.yourcompany.yourapp`)
- **Network Extensions** capability with **Packet Tunnel** checked
- The XCFramework linked (as a static library, Do Not Embed)
- `-lresolv` in OTHER_LDFLAGS
- A bridging header including the Go-generated `.h` file

### 4.3 Shared Code

The extension runs in a separate process. It needs its own copies of:
- The Swift bridge file (e.g., `MyBridge.swift`) — add to extension target's Compile Sources
- Its own bridging header (e.g., `MyTunnel-Bridging-Header.h`)

### 4.4 IPC via App Groups

```swift
// Write from extension
let defaults = UserDefaults(suiteName: "group.com.yourcompany.yourapp")
defaults?.set(port, forKey: "api_port")

// Read from app
let defaults = UserDefaults(suiteName: "group.com.yourcompany.yourapp")
let port = defaults?.integer(forKey: "api_port") ?? 0
```

### 4.5 Apple Entitlement

The Network Extension entitlement may need approval from Apple:
1. Go to developer.apple.com → Certificates, Identifiers & Profiles
2. Select your App ID → enable **Network Extensions**
3. If it's greyed out, submit a request at https://developer.apple.com/contact/request/network-extension/

---

## Phase 5: Build & Deploy Workflow

### Development Cycle

```bash
# 1. Build Go framework (only when Go code changes)
cd gomobile && bash build_ios_cgo.sh

# 2. Build Vue app
npm run build

# 3. Sync to Xcode
npx cap sync ios

# 4. Build & run in Xcode (Cmd+R)
```

### Live Reload (Development Only)

In `capacitor.config.ts`, uncomment the server block with your Mac's local IP:
```typescript
server: {
  url: 'http://192.168.x.x:5173',
  cleartext: true,
},
```

Then run `npm run dev`, `npx cap sync ios`, and rebuild in Xcode. The app loads from Vite's dev server with hot reload.

**Remember to comment this out before production builds.**

### Bundle ID Tips

- Bundle IDs are globally unique across all Apple developers
- If `com.company.my.app` is taken, try `com.company.myapp` (no dots in suffix)
- Extension bundle ID must be a child: `com.company.myapp.tunnel`
- App Group ID format: `group.com.company.myapp`
- Keep all IDs consistent across: `capacitor.config.ts`, Xcode targets, entitlements, Swift code

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Blank white screen on iOS | Add GoCompatibleRouter (see Phase 3.1). This is the #1 issue. |
| Blank screen, no Go framework | Check scene lifecycle: SceneDelegate + Info.plist manifest |
| `res_9_nclose` undefined | Add `-lresolv` to OTHER_LDFLAGS |
| Extension can't find Swift types | Add shared .swift files to extension target's Compile Sources |
| `npx vite` returns 404 | Run from project directory, not parent. Use `node_modules/.bin/vite` |
| Capacitor plugin not found | Verify `@objc(Name)`, `CAPBridgedPlugin` protocol, `jsName` matches `registerPlugin` |
| Web assets stale after build | Run `npx cap sync ios` after `npm run build` |
| Xcode signing fails | Check bundle ID availability in Apple Developer portal |
| PLA Update error | Accept Program License Agreement at developer.apple.com |
| `missing required module 'MobileCoreServices'` | Clean build folder in Xcode (Cmd+Shift+K, then Cmd+B). Stale SPM cache after Capacitor upgrade on Xcode 26. |

---

## Upgrading Capacitor (e.g., 7 → 8)

### Step-by-step

```bash
# 1. Update npm packages
npm install @capacitor/core@latest @capacitor/ios@latest @capacitor/cli@latest

# 2. Rebuild and sync (updates SPM Package.swift automatically)
npm run build && npx cap sync ios

# 3. Clean build in Xcode
# Cmd+Shift+K (clean), then Cmd+B (build)
# Or from CLI:
cd ios/App && xcodebuild clean -scheme App && cd ../..
```

### Capacitor 7 → 8 specifics

- **iOS changes are minimal**: SPM package version updates automatically via `cap sync`
- **Requires**: Xcode 26+, Node.js 22+, iOS 15.0+ deployment target
- **Plugin pattern unchanged**: `CAPBridgedPlugin` protocol auto-discovery works the same
- **Clean build required**: After major Capacitor upgrade, always clean the Xcode build folder. Stale SPM artifacts cause `missing required module 'MobileCoreServices'` on Xcode 26
- **No Swift code changes needed**: AppDelegate, SceneDelegate, ViewController, plugins all work as-is

### Frontend dependency upgrade

All frontend deps can be upgraded independently alongside Capacitor:

```bash
npm install vue@latest vue-router@latest
npm install -D vite@latest @vitejs/plugin-vue@latest tailwindcss@latest \
  @tailwindcss/vite@latest typescript@latest vue-tsc@latest
npm run build  # verify type checking + build pass
npx cap sync ios
```

Tested working combination (April 2026): Vue 3.5.32, Vue Router 5.0.4, Vite 8.0.3, @vitejs/plugin-vue 6.0.5, Tailwind CSS 4.2.2, TypeScript 6.0.2, vue-tsc 3.2.6.
