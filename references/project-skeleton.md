# Project Skeleton

This is the minimum viable file layout for a URL-wrapper Mac app. Pure Swift, no Xcode project, builds with a shell script straight into `/Applications`. The user can edit any file in any editor and rebuild in seconds.

## Why no Xcode project?

Xcode projects are large, mostly-machine-generated, and a nightmare to diff or edit by hand. A pure-Swift package with a 100-line `build.sh` is faster to iterate on, trivially shareable, and avoids the entire class of "project file got out of sync" bugs. The trade-off: no Interface Builder. Everything is programmatic AppKit/SwiftUI. For a WKWebView wrapper that's a feature, not a limitation.

## Directory layout

```
<AppName>/
├── build.sh                         # Build + sign + install
├── Package.swift                    # Swift Package Manager manifest
├── README.md
├── Resources/
│   ├── Info.plist                   # Bundle metadata + usage descriptions
│   ├── AppIcon.icns                 # 1024x1024 master, converted to .icns
│   └── <AppName>.entitlements       # Camera/mic/network entitlements
└── Sources/
    ├── main.swift                   # NSApplication bootstrap (7 lines)
    ├── AppDelegate.swift            # Menus, key monitors, window registry
    ├── WebViewController.swift      # Per-tab WKWebView + delegates
    ├── TabManager.swift             # Per-window tab bar + tab lifecycle
    ├── SessionManager.swift         # Save/restore windows + tabs (JSON)
    ├── SettingsWindowController.swift   # AppKit NSWindowController + NSTabView
    └── ThemeManager.swift           # CSS theme registry + injection
```

Only the files needed for the chosen feature set need to exist. A minimum-viable app is just `main.swift`, `AppDelegate.swift`, `WebViewController.swift`, `Info.plist`, `build.sh`, and `Package.swift`. Add the rest as you wire features.

## `Package.swift`

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "<AppName>",
    platforms: [.macOS(.v13)],
    targets: [
        .executableTarget(
            name: "<AppName>",
            path: "Sources"
        )
    ]
)
```

Why platform .v13: WKWebView APIs improved meaningfully through macOS 12 → 13 (media capture permission delegate, `inspectable` property, etc.). Below 13, you'll need conditional code.

## `Resources/Info.plist`

Minimum keys. Replace `<>` placeholders before writing.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleName</key>
    <string><App Display Name></string>
    <key>CFBundleDisplayName</key>
    <string><App Display Name></string>
    <key>CFBundleIdentifier</key>
    <string><com.you.appname></string>
    <key>CFBundleExecutable</key>
    <string><AppName></string>
    <key>CFBundleIconFile</key>
    <string>AppIcon</string>
    <key>CFBundleShortVersionString</key>
    <string>0.0.1</string>
    <key>CFBundleVersion</key>
    <string>1</string>
    <key>CFBundlePackageType</key>
    <string>APPL</string>
    <key>LSMinimumSystemVersion</key>
    <string>13.0</string>
    <key>NSHighResolutionCapable</key>
    <true/>
    <key>NSPrincipalClass</key>
    <string>NSApplication</string>

    <!-- Required if the wrapped site uses non-HTTPS resources or arbitrary CDNs -->
    <key>NSAppTransportSecurity</key>
    <dict>
        <key>NSAllowsArbitraryLoads</key>
        <true/>
    </dict>

    <!-- Required for camera/mic permission prompts to appear -->
    <key>NSCameraUsageDescription</key>
    <string><AppName> uses your camera for video calls in <site>.</string>
    <key>NSMicrophoneUsageDescription</key>
    <string><AppName> uses your microphone for voice and video calls in <site>.</string>
</dict>
</plist>
```

`CFBundleShortVersionString` is the release version you bump by hand. `CFBundleVersion` will be overwritten by `build.sh` with a date-stamped value at build time — the `1` is just a placeholder so the plist is valid before the first build.

## `Resources/<AppName>.entitlements`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.device.camera</key>
    <true/>
    <key>com.apple.security.device.microphone</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
    <!-- Web Inspector wiring (3/3): required on Sonoma+ for the inspector to bind
         to the ad-hoc-signed process. Without it, isInspectable + developerExtrasEnabled
         silently no-op. Always include this — every wrapper in this skill ships Cmd+Opt+I. -->
    <key>com.apple.security.get-task-allow</key>
    <true/>
</dict>
</plist>
```

Only include the *camera* and *microphone* entitlements if the app actually uses them. `network.client` and `get-task-allow` are always required.

## `Sources/main.swift`

```swift
import AppKit

let app = NSApplication.shared
let delegate = AppDelegate()
app.delegate = delegate
app.setActivationPolicy(.regular)
app.run()
```

That's it. Everything else lives in `AppDelegate`.

## `Sources/AppDelegate.swift` — minimal viable

This is the smallest AppDelegate that produces a working URL wrapper. Expand with the tab/window/menu patterns from `tabs-and-windows.md`.

```swift
import AppKit
import WebKit

final class AppDelegate: NSObject, NSApplicationDelegate {
    private var window: NSWindow!
    private var webViewController: WebViewController!

    private let startPageURL: URL = {
        let stored = UserDefaults.standard.string(forKey: "startPageURL")
        return URL(string: stored ?? "https://<DEFAULT-URL>")!
    }()

    func applicationDidFinishLaunching(_ notification: Notification) {
        // Backstop for the macOS "Currently…" suggestion pill in web fields. NOTE: these
        // defaults DON'T suffice on their own — the real fix is per-field autocorrect/
        // autocapitalize/spellcheck attributes injected as a WKUserScript (see
        // references/settings.md → "Disable the macOS text-suggestion pill"). Set these too.
        UserDefaults.standard.set(false, forKey: "NSAutomaticTextCompletionEnabled")
        UserDefaults.standard.set(false, forKey: "WebAutomaticTextCompletionEnabled")
        UserDefaults.standard.set(false, forKey: "NSAutomaticInlinePredictionEnabled")

        buildMainMenu()
        buildWindow()
        webViewController.load(url: startPageURL)
        // build.sh --relaunch opens with `open -g` + this env var so dev rebuilds
        // restart the app in the background without stealing focus. Skipping
        // activate alone isn't enough — the
        // ordered-front window would still visually cover the active app.
        if ProcessInfo.processInfo.environment["<APPNAME>_BACKGROUND_LAUNCH"] != nil {
            for window in NSApp.windows { window.orderBack(nil) }
        } else {
            NSApp.activate(ignoringOtherApps: true)
        }
    }

    private func buildWindow() {
        window = NSWindow(
            contentRect: NSRect(x: 0, y: 0, width: 1200, height: 800),
            styleMask: [.titled, .closable, .miniaturizable, .resizable, .fullSizeContentView],
            backing: .buffered,
            defer: false
        )
        window.title = "<App Display Name>"
        window.titlebarAppearsTransparent = true
        window.titleVisibility = .hidden
        window.backgroundColor = NSColor(red: 0.07, green: 0.07, blue: 0.09, alpha: 1.0)
        window.minSize = NSSize(width: 800, height: 500)
        window.collectionBehavior = [.fullScreenPrimary]
        window.center()

        webViewController = WebViewController()
        window.contentView = webViewController.view
        window.makeKeyAndOrderFront(nil)
        window.makeFirstResponder(webViewController.webView)  // Critical for typing on launch.
    }

    private func buildMainMenu() {
        let menubar = NSMenu()
        let appMenuItem = NSMenuItem()
        menubar.addItem(appMenuItem)
        let appMenu = NSMenu()
        appMenu.addItem(withTitle: "About <AppName>", action: #selector(showAboutPanel), keyEquivalent: "")
        appMenu.addItem(NSMenuItem.separator())
        appMenu.addItem(withTitle: "Hide <AppName>", action: #selector(NSApplication.hide(_:)), keyEquivalent: "h")
        appMenu.addItem(withTitle: "Quit <AppName>", action: #selector(NSApplication.terminate(_:)), keyEquivalent: "q")
        appMenuItem.submenu = appMenu
        NSApp.mainMenu = menubar
    }

    @objc func showAboutPanel() {
        let info = Bundle.main.infoDictionary ?? [:]
        let shortVersion = info["CFBundleShortVersionString"] as? String ?? ""
        let displayVersion = (info["<AppName>VersionDisplay"] as? String)
            ?? (info["CFBundleVersion"] as? String ?? "")
        NSApp.orderFrontStandardAboutPanel(options: [
            .applicationVersion: shortVersion,
            .version: NSAttributedString(string: displayVersion)
        ])
    }
}
```

## `Sources/WebViewController.swift` — minimal viable

```swift
import AppKit
import WebKit

final class WebViewController: NSViewController {
    private(set) var webView: WKWebView!

    override func loadView() {
        let config = WKWebViewConfiguration()
        // NOTE: WKProcessPool is deprecated on macOS 12+ and "no longer has any effect" —
        // cookies/sessions are shared automatically via WKWebsiteDataStore.default().
        // No process pool line needed.
        config.preferences.javaScriptCanOpenWindowsAutomatically = true
        config.defaultWebpagePreferences.allowsContentJavaScript = true
        config.mediaTypesRequiringUserActionForPlayback = []
        // Web Inspector wiring (1/3): private-but-stable WebKit preference, must be set
        // BEFORE WKWebView construction. Without it, right-click context menu has no
        // "Inspect Element" item. See webview-config.md → "Inspect Element" for the
        // other two pieces (get-task-allow entitlement + isInspectable).
        config.preferences.setValue(true, forKey: "developerExtrasEnabled")

        webView = WKWebView(frame: .zero, configuration: config)
        if #available(macOS 13.3, *) {
            webView.isInspectable = true  // Web Inspector wiring (2/3): must be set immediately after construction.
        }
        webView.customUserAgent = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Safari/605.1.15"
        webView.allowsBackForwardNavigationGestures = true
        webView.uiDelegate = self
        webView.navigationDelegate = self
        self.view = webView
    }

    func load(url: URL) {
        webView.load(URLRequest(url: url))
    }
}

extension WebViewController: WKUIDelegate, WKNavigationDelegate {
    // Camera/mic — auto-grant for the wrapped site.
    func webView(_ webView: WKWebView,
                 requestMediaCapturePermissionFor origin: WKSecurityOrigin,
                 initiatedByFrame frame: WKFrameInfo,
                 type: WKMediaCaptureType,
                 decisionHandler: @escaping (WKPermissionDecision) -> Void) {
        decisionHandler(.grant)
    }
}
```

Expand the `WebViewController` with patterns from `webview-config.md` — navigation allowlist, OAuth handling, zoom persistence, theme injection hooks.

## `build.sh`

This is the linchpin. It does seven things:

1. Compiles the Swift sources — **FIRST, before touching the installed bundle**, so a compile error never leaves the user app-less (an `rm -rf`-first ordering deletes the bundle and then fails the build).
2. Assembles a real `.app` bundle structure.
3. Copies `Info.plist`, the icon, and the binary into place.
4. Stamps `CFBundleVersion` with a date+commit value (plus an About-panel display key).
5. Strips quarantine xattrs.
6. Ad-hoc-signs with entitlements.
7. **Ships a `--relaunch` flag from day one.** Every new build.sh embeds the self-verifying quit → open → PID-verify cycle behind `--relaunch`; don't scaffold without it and "add it later" — it gets missed, and every rebuild then needs a hand-composed quit/sleep/open line. SIGTERM only (no AppleScript → no macOS Automation/TCC prompt), quit only AFTER a successful compile, absolute binary paths (`/usr/bin/pgrep`, `/usr/bin/pkill`, `/bin/sleep`, `/usr/bin/open`) so the flag works from launchers and automation tools with a stripped PATH, and a non-zero exit when the PID didn't change. **The relaunch is a BACKGROUND launch:** `open -g` + a `<APPNAME>_BACKGROUND_LAUNCH=1` env var that the AppDelegate gates on to skip `NSApp.activate` and `orderBack` its windows — dev rebuilds must never steal focus from the app you are working in (Dock/Finder launches activate normally).

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_NAME="<AppName>"
BUNDLE_ID="<com.you.appname>"
EXEC_NAME="<AppName>"   # CFBundleExecutable — what pgrep/pkill match on
SOURCE_DIR="$(cd "$(dirname "$0")" && pwd)"

# Build into /Applications so macOS Gatekeeper doesn't refuse to execute
# (it refuses cloud-synced paths like ~/Dropbox, ~/Google Drive, ~/iCloud).
APP_PATH="/Applications/${APP_NAME}.app"

# --relaunch: after a successful build, quit any running instance (SIGTERM — no
# AppleScript, so no macOS Automation/TCC prompt), reopen the fresh bundle, and
# verify the PID actually changed (exit non-zero if not). The quit happens only
# AFTER the compile succeeds, so a broken build leaves the old app running.
RELAUNCH=0
for arg in "$@"; do
    case "$arg" in
        --relaunch) RELAUNCH=1 ;;
        *) echo "Unknown option: $arg (supported: --relaunch)"; exit 2 ;;
    esac
done

echo "Building ${APP_NAME}…"

# 1. Compile FIRST — the installed bundle isn't touched until this succeeds,
#    so a compile error never leaves the user app-less.
cd "${SOURCE_DIR}"
swift build -c release --product "${APP_NAME}" 2>&1
BIN_PATH=".build/release/${APP_NAME}"

# 2. Clean prior bundle and assemble.
rm -rf "${APP_PATH}"
mkdir -p "${APP_PATH}/Contents/MacOS"
mkdir -p "${APP_PATH}/Contents/Resources"
cp "${BIN_PATH}" "${APP_PATH}/Contents/MacOS/${APP_NAME}"

# 3. Stamp build metadata into Info.plist.
PLIST_SRC="Resources/Info.plist"
PLIST_DST="${APP_PATH}/Contents/Info.plist"
cp "${PLIST_SRC}" "${PLIST_DST}"

# Date-stamped CFBundleVersion: YYYYMMDD.HHMM from latest commit timestamp.
if git rev-parse --git-dir > /dev/null 2>&1; then
    COMMIT_TS=$(git log -1 --format=%cI origin/main 2>/dev/null || git log -1 --format=%cI HEAD)
    BUILD_NUMBER=$(date -j -f "%Y-%m-%dT%H:%M:%S%z" "${COMMIT_TS}" "+%Y%m%d.%H%M" 2>/dev/null || date "+%Y%m%d.%H%M")
    SHORT_SHA=$(git rev-parse --short HEAD)
    PRETTY_DATE=$(date "+%m-%d-%y %-I:%M %p")
    DISPLAY="${PRETTY_DATE} · ${SHORT_SHA}"
else
    BUILD_NUMBER=$(date "+%Y%m%d.%H%M")
    DISPLAY="$(date "+%m-%d-%y %-I:%M %p") · dev"
fi

/usr/libexec/PlistBuddy -c "Set :CFBundleVersion ${BUILD_NUMBER}" "${PLIST_DST}"
/usr/libexec/PlistBuddy -c "Add :${APP_NAME}VersionDisplay string ${DISPLAY}" "${PLIST_DST}" 2>/dev/null \
    || /usr/libexec/PlistBuddy -c "Set :${APP_NAME}VersionDisplay ${DISPLAY}" "${PLIST_DST}"

# 4. Copy icon and any other resources.
if [ -f "Resources/AppIcon.icns" ]; then
    cp "Resources/AppIcon.icns" "${APP_PATH}/Contents/Resources/AppIcon.icns"
fi

# 5. Strip quarantine xattrs so the app launches without Gatekeeper warnings.
xattr -cr "${APP_PATH}"

# 6. Ad-hoc sign with entitlements.
ENTITLEMENTS="Resources/${APP_NAME}.entitlements"
if [ -f "${ENTITLEMENTS}" ]; then
    codesign --force --deep --sign - \
        --entitlements "${ENTITLEMENTS}" \
        "${APP_PATH}"
else
    codesign --force --deep --sign - "${APP_PATH}"
fi

echo "✓ Built ${APP_PATH}"
echo "  Version: $(/usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' "${PLIST_DST}") (${DISPLAY})"

# 7. Optional quit → reopen → PID-verify cycle. Absolute binary paths so this
#    works from stripped-PATH environments (launchers, automation tools).
if [ "${RELAUNCH}" -eq 1 ]; then
    OLD_PID=$(/usr/bin/pgrep -x "${EXEC_NAME}" 2>/dev/null || true)
    if [ -n "${OLD_PID}" ]; then
        kill -TERM ${OLD_PID} 2>/dev/null || true
        for _ in 1 2 3; do
            /bin/sleep 1
            /usr/bin/pgrep -x "${EXEC_NAME}" > /dev/null 2>&1 || break
        done
        if /usr/bin/pgrep -x "${EXEC_NAME}" > /dev/null 2>&1; then
            /usr/bin/pkill -9 -x "${EXEC_NAME}" 2>/dev/null || true
            /bin/sleep 1
        fi
    fi
    # Background relaunch: -g keeps the app
    # BEHIND the frontmost app, and the env var makes the app skip its own
    # NSApp.activate and order its windows back — dev rebuilds never steal focus.
    /usr/bin/open -g --env <APPNAME>_BACKGROUND_LAUNCH=1 "${APP_PATH}"
    /bin/sleep 3
    NEW_PID=$(/usr/bin/pgrep -x "${EXEC_NAME}" 2>/dev/null || true)
    if [ -z "${NEW_PID}" ]; then
        echo "✗ Relaunch FAILED: ${EXEC_NAME} is not running after open" >&2
        exit 1
    elif [ -n "${OLD_PID}" ] && [ "${NEW_PID}" = "${OLD_PID}" ]; then
        echo "✗ Relaunch FAILED: same PID (${NEW_PID}) — old instance never quit" >&2
        exit 1
    fi
    echo "✓ Relaunched: ${EXEC_NAME} PID ${NEW_PID} (was ${OLD_PID:-none})"
fi
```

Make it executable: `chmod +x build.sh`.

**SIGTERM caveat:** the quit path skips `applicationWillTerminate`, so the app must not save critical state ONLY there — use debounced/periodic autosave (the sessions pattern in `references/sessions.md` already does).

### Why /Applications and not the source tree?

macOS Gatekeeper enforces that executables in cloud-synced locations (Dropbox, Google Drive, iCloud, OneDrive) are quarantined and require user re-approval on every launch — or refuses to launch entirely. If the user's repo lives in any synced folder, building in place is broken from day one. `/Applications` is the only universally-safe target. The build script overwriting the running app is fine; macOS lets you replace `.app` bundles even while they're running.

### Why `xattr -cr`?

Files copied or downloaded inherit a `com.apple.quarantine` extended attribute. On launch, macOS will pop "are you sure you want to run this?" warnings. Stripping the attribute makes the app behave like one the user built themselves — which they did.

## After scaffolding

1. Run `./build.sh` to verify it produces `/Applications/<AppName>.app`.
2. Open `/Applications/<AppName>.app` to verify the URL loads.
3. Bump `CFBundleShortVersionString` for each release; the About panel shows it beside the build stamp.
4. Use `./build.sh --relaunch` for the edit-build-relaunch cycle from here on.

If the user wants tabs, sessions, themes, or settings, read the matching reference file and integrate.
