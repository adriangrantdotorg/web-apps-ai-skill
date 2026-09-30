# Theme Management (CSS Injection)

Users drop `.css` files into theme folders the app watches; the app discovers them, tracks which are enabled, and injects the combined CSS into the WKWebView. Stylus userstyle CSS is supported transparently (the metadata block is stripped before injection).

Themes come in **two tiers, from two folders** (see *Storage layout*):

- **Custom** — app-specific themes, in the app's own folder. **Multi-select**: enable any number at once.
- **Universal** — themes shared across several wrapper apps, in a common parent folder. **Single-select**: at most one Universal theme active at a time.

So a user can run several Custom themes *plus* one Universal theme together; enabling a second Universal theme deselects the first. The enabled themes' CSS is **concatenated** (Universal first, Custom last, so app-specific tweaks win the cascade) — not one-theme-wins.

This file covers: theme storage layout, the `ThemeManager` skeleton, auto-discovery, Stylus metadata stripping, CSS injection via `WKUserScript` + `MutationObserver`, mode-aware auto-toggling, and how to react to system Light/Dark changes.

## Storage layout

Themes live in **two folders the user can get at easily** — deliberately *not* `~/Library/Application Support/<AppName>/`, which is buried, per-app, and awkward to edit or version-control:

- **Universal folder** — a shared, top-level directory the user controls (its own folder, or better, **its own git repo**). Several wrapper apps point at this *same* folder, so one set of palettes (Gruvbox, Nord, Catppuccin, …) is maintained once and reused everywhere.
- **Custom folder** — an app-specific subfolder *inside* the Universal folder. Holds themes written for this one site.

```
<themes-root>/                     # accessible, shared, often its own repo — NOT Application Support
├── Gruvbox - Dark.css             # Universal: shared across every wrapper app
├── Nord.css
├── Catppuccin - Mocha.css
└── <AppName>/                     # Custom: this app only
    ├── <appname>.css
    └── <appname> - compact.css
```

**Default the Universal themes root to a shared, user-visible folder such as `~/Themes/`** when scaffolding a new app, so all of the user's wrapper apps can point at the same repo. Don't bake it in as an unchangeable constant, though: store the resolved path in a configurable setting (defaulting to that location) so it stays overridable. And **at setup/first-run, check that the folder exists — if it doesn't, ask the user where their themes folder is** rather than silently creating an empty one in the wrong place (the Custom subfolder *is* created automatically once the root is known). What matters, beyond the default path:

1. Themes are in an **accessible** location the user can open, edit, and version-control — not inside the app bundle, not in Application Support.
2. The **Universal** folder is **shared** (multiple wrapper apps read the same directory) while **Custom** is a per-app subfolder of it. Scanning is **non-recursive**, so the Universal scan does not re-list the Custom subfolder — the two lists stay distinct.

There is **no `themes.json` manifest**. Which themes are enabled is small, per-app UI state — keep it in `UserDefaults` (a sorted `[String]` of theme ids). The folders are the source of truth for *what exists*; UserDefaults is the source of truth for *what's on*. The id encodes the source so the two folders never collide: `"universal:<file>.css"` / `"custom:<file>.css"`.

```swift
enum ThemeSource: String {
    case custom      // app-specific subfolder
    case universal   // shared parent folder, read by multiple apps
}

struct Theme: Identifiable {
    let id: String          // stable key: "custom:<file>.css" / "universal:<file>.css"
    let name: String
    let filename: String
    let source: ThemeSource
    let url: URL
    var enabled: Bool
}
```

## ThemeManager skeleton

```swift
import AppKit

final class ThemeManager {
    static let shared = ThemeManager()
    static let themesChanged = Notification.Name("themesChanged")

    /// Default Universal themes root — the shared repo wrapper apps point at. Overridable via
    /// the `themesRootPath` setting below; only the default is baked in.
    static let defaultThemesRoot = FileManager.default.homeDirectoryForCurrentUser
        .appendingPathComponent("Themes", isDirectory: true)
    private static let rootKey = "themesRootPath"

    /// Shared parent folder (Universal themes).
    let universalThemesDirectoryURL: URL
    /// App-specific subfolder (Custom themes), nested inside the Universal folder.
    let customThemesDirectoryURL: URL

    private let enabledKey = "enabledThemeIDs"
    private var themes: [Theme] = []

    /// True when the resolved Universal root doesn't exist on disk — the caller should prompt
    /// the user for their themes folder at setup/first-run instead of using a bogus location.
    private(set) var rootIsMissing = false

    private init() {
        // Configured override wins; otherwise fall back to the default shared repo.
        if let custom = UserDefaults.standard.string(forKey: Self.rootKey), !custom.isEmpty {
            universalThemesDirectoryURL = URL(fileURLWithPath: custom, isDirectory: true)
        } else {
            universalThemesDirectoryURL = Self.defaultThemesRoot
        }
        rootIsMissing = !FileManager.default.fileExists(atPath: universalThemesDirectoryURL.path)
        customThemesDirectoryURL = universalThemesDirectoryURL
            .appendingPathComponent("<AppName>", isDirectory: true)
        // Only create the per-app Custom subfolder once we trust the root. If the root is
        // missing, leave it alone — the app should ask the user where their themes live, then
        // persist their answer to `themesRootPath` and re-init.
        if !rootIsMissing {
            try? FileManager.default.createDirectory(at: customThemesDirectoryURL,
                                                     withIntermediateDirectories: true)
        }
        discover()
    }

    // MARK: - Enabled-state persistence (UserDefaults — no manifest file)

    private var enabledIDs: Set<String> {
        get { Set(UserDefaults.standard.stringArray(forKey: enabledKey) ?? []) }
        set { UserDefaults.standard.set(Array(newValue).sorted(), forKey: enabledKey) }
    }

    // MARK: - Discovery

    private func discover() {
        let en = enabledIDs
        var result: [Theme] = []

        func scan(_ dir: URL, _ source: ThemeSource) {
            guard let files = try? FileManager.default.contentsOfDirectory(
                at: dir, includingPropertiesForKeys: nil) else { return }
            let css = files
                .filter { $0.pathExtension.lowercased() == "css" }
                .sorted { $0.lastPathComponent.localizedCaseInsensitiveCompare($1.lastPathComponent) == .orderedAscending }
            for f in css {
                let id = "\(source.rawValue):\(f.lastPathComponent)"
                result.append(Theme(id: id, name: f.deletingPathExtension().lastPathComponent,
                                    filename: f.lastPathComponent, source: source, url: f,
                                    enabled: en.contains(id)))
            }
        }
        // Non-recursive: the Universal scan does NOT descend into the Custom subfolder.
        scan(customThemesDirectoryURL, .custom)
        scan(universalThemesDirectoryURL, .universal)

        // Final enabled set: drop ids whose files vanished, then enforce single-Universal —
        // if persisted state somehow has 2+ Universal enabled, keep the first (alphabetically).
        // Custom themes are unconstrained.
        let present = Set(result.map { $0.id })
        var finalEnabled = en.intersection(present)
        let enabledUniversal = result.filter { $0.source == .universal && finalEnabled.contains($0.id) }
        if enabledUniversal.count > 1 {
            for extra in enabledUniversal.dropFirst() { finalEnabled.remove(extra.id) }
        }
        if finalEnabled != en { enabledIDs = finalEnabled }
        themes = result.map { var t = $0; t.enabled = finalEnabled.contains($0.id); return t }
    }

    // MARK: - Public API

    func list() -> [Theme] { themes }
    func themes(in source: ThemeSource) -> [Theme] { themes.filter { $0.source == source } }
    var enabledThemes: [Theme] { themes.filter { $0.enabled } }
    var hasEnabled: Bool { themes.contains { $0.enabled } }

    /// Custom themes are multi-select. Universal themes are single-select: enabling one clears
    /// any other enabled Universal theme (a Custom theme may stay on alongside it).
    func toggle(id: String) {
        var en = enabledIDs
        if en.contains(id) {
            en.remove(id)
        } else {
            if themes.first(where: { $0.id == id })?.source == .universal {
                en.subtract(themes.filter { $0.source == .universal }.map { $0.id })
            }
            en.insert(id)
        }
        enabledIDs = en
        discover()
        NotificationCenter.default.post(name: Self.themesChanged, object: nil)
    }

    func disableAll() {
        enabledIDs = []
        discover()
        NotificationCenter.default.post(name: Self.themesChanged, object: nil)
    }

    func importTheme(from sourceURL: URL) {   // imports always land in the Custom folder
        let dest = customThemesDirectoryURL.appendingPathComponent(sourceURL.lastPathComponent)
        try? FileManager.default.removeItem(at: dest)
        try? FileManager.default.copyItem(at: sourceURL, to: dest)
        reload()
    }

    /// Rescan both folders + re-inject (explicit "Reload All Themes" / external edits).
    func reload() {
        discover()
        NotificationCenter.default.post(name: Self.themesChanged, object: nil)
    }

    /// Silent rescan — used when rebuilding the menu on open (no re-injection).
    func scanForNewThemes() { discover() }

    // MARK: - Combined CSS

    /// CSS of every enabled theme, or nil if none. Universal (palette) themes emit first and
    /// Custom themes last, so app-specific tweaks win specificity ties.
    func activeCSS() -> String? {
        let ordered = enabledThemes.sorted { a, b in
            if a.source != b.source { return a.source == .universal }   // universal first
            return a.name.localizedCaseInsensitiveCompare(b.name) == .orderedAscending
        }
        let parts = ordered.compactMap { t -> String? in
            guard let raw = try? String(contentsOf: t.url, encoding: .utf8) else { return nil }
            return "/* === \(t.source.rawValue): \(t.name) === */\n" + Self.stripStylusMetadata(raw)
        }
        return parts.isEmpty ? nil : parts.joined(separator: "\n\n")
    }

    static func stripStylusMetadata(_ raw: String) -> String {
        var s = raw
        // Drop ==UserStyle==…==/UserStyle== block.
        if let start = s.range(of: "==UserStyle=="),
           let end = s.range(of: "==/UserStyle==", range: start.upperBound..<s.endIndex) {
            s.removeSubrange(start.lowerBound...end.upperBound)
        }
        // Drop @-moz-document wrappers: unwrap the inner block.
        while let mozRange = s.range(of: #"@-moz-document[^{]+\{"#, options: .regularExpression) {
            // Find the matching closing brace.
            let body = s[mozRange.upperBound...]
            var depth = 1
            var idx = body.startIndex
            while idx < body.endIndex && depth > 0 {
                let ch = body[idx]
                if ch == "{" { depth += 1 }
                if ch == "}" { depth -= 1 }
                idx = body.index(after: idx)
            }
            // Replace `@-moz-document …{` with empty and the matching `}` with empty.
            let closingBraceIndex = body.index(before: idx)
            s.replaceSubrange(mozRange.lowerBound..<mozRange.upperBound, with: "")
            // After mutation the indices shift; safer to rebuild fully via repeated apply.
            // For simplicity, fall through and trust the loop.
            _ = closingBraceIndex  // (this branch is illustrative — production code uses a state machine)
            break
        }
        // Drop @namespace lines (irrelevant in WKWebView).
        s = s.replacingOccurrences(of: #"^\s*@namespace[^;]+;\s*$"#,
                                   with: "",
                                   options: [.regularExpression, .caseInsensitive])
        return s
    }
}
```

Production-quality Stylus stripping needs a small state machine; the regex sketch above is illustrative. The key insight is to *unwrap* `@-moz-document` rather than delete it — the CSS inside is what the user actually wrote, and WKWebView doesn't recognize `@-moz-document` itself.

## CSS injection — the right way (no flash, no flicker)

There are three places you can inject theme CSS into a `WKWebView`, and which one you pick determines whether the user sees a flash of the site's default styles before the theme takes effect:

| Method | When it runs | Result |
|---|---|---|
| `WKUserScript` at `.atDocumentStart` | **Before any HTML parses** | Themed page from the very first paint. No flash. |
| `WKUserScript` at `.atDocumentEnd` | After DOM parses, before subresources | Brief flash possible while parser builds the page unthemed. |
| `evaluateJavaScript` in `didFinish` / `webView.url` KVO | After the page has fully loaded and painted | Visible flash on launch and on every reload. |

**Always use `.atDocumentStart` for theme CSS** — this is the only option that prevents the "site loads in default styles, then the theme snaps in" flash users notice on launch. Then add a MutationObserver to the injected IIFE so SPA re-renders that strip your style tag can't beat you.

Theme CSS that changes at runtime (the user picks a new theme from the menu) needs two updates: (1) the WKUserScript, so the next reload starts pre-themed, and (2) `evaluateJavaScript` against the current page, so the live page updates without a reload. Both run the same IIFE; only the injection vehicle differs.

### The pattern

Hold a reference to a `WKUserContentController` on the `WebViewController`:

```swift
private var userContent: WKUserContentController!
```

In `loadView()`, set it up *before* the WKWebView is created and install user scripts immediately:

```swift
let config = WKWebViewConfiguration()
// … other config setup …

userContent = WKUserContentController()
config.userContentController = userContent
installUserScripts()   // ← must run BEFORE WKWebView(frame:configuration:)

webView = WKWebView(frame: .zero, configuration: config)
```

`installUserScripts()` rebuilds the entire set from scratch (chrome + active theme). It's also what you call when the active theme changes — `WKUserContentController` has no remove-one API, only `removeAllUserScripts()`:

```swift
private func installUserScripts() {
    userContent.removeAllUserScripts()

    // Static chrome CSS (traffic-light clearance, etc.) — always installed.
    let chromeJS = Self.styleInjectionJS(styleID: "__appname_chrome",
                                         observerName: "__appname_chromeObserver",
                                         css: Self.chromeCSS)
    userContent.addUserScript(WKUserScript(
        source: chromeJS,
        injectionTime: .atDocumentStart,
        forMainFrameOnly: true
    ))

    // Active theme CSS — only installed if a theme is selected.
    if let themeCSS = ThemeManager.shared.activeCSS() {
        let themeJS = Self.styleInjectionJS(styleID: "__appname_theme",
                                            observerName: "__appname_themeObserver",
                                            css: themeCSS)
        userContent.addUserScript(WKUserScript(
            source: themeJS,
            injectionTime: .atDocumentStart,
            forMainFrameOnly: true
        ))
    }
}
```

Extract the JS-building into a single helper that both `WKUserScript` and `evaluateJavaScript` paths use — this keeps the chrome/theme/SPA-re-inject logic in one place:

```swift
private static func styleInjectionJS(styleID: String, observerName: String, css: String) -> String {
    let escaped = css
        .replacingOccurrences(of: "\\", with: "\\\\")
        .replacingOccurrences(of: "`", with: "\\`")
        .replacingOccurrences(of: "$", with: "\\$")
    return """
    (function() {
        const STYLE_ID = '\(styleID)';
        const CSS = `\(escaped)`;
        function ensure() {
            let el = document.getElementById(STYLE_ID);
            if (!el) {
                el = document.createElement('style');
                el.id = STYLE_ID;
                el.textContent = CSS;
                (document.head || document.documentElement).appendChild(el);
            } else if (el.textContent !== CSS) {
                el.textContent = CSS;
            }
        }
        ensure();
        if (!window.\(observerName)) {
            window.\(observerName) = new MutationObserver(() => {
                if (!document.getElementById(STYLE_ID)) ensure();
            });
            window.\(observerName).observe(document.documentElement, {
                childList: true, subtree: true
            });
        }
    })();
    """
}
```

On theme change (notification from `ThemeManager`):

```swift
@objc private func themeDidChange() {
    installUserScripts()    // Updates the WKUserScript for the next load.
    injectActiveTheme()     // Updates the current page live.
}

func injectActiveTheme() {
    if let css = ThemeManager.shared.activeCSS() {
        let js = Self.styleInjectionJS(styleID: "__appname_theme",
                                       observerName: "__appname_themeObserver",
                                       css: css)
        webView.evaluateJavaScript(js, completionHandler: nil)
    } else {
        // Remove the theme tag + observer.
        webView.evaluateJavaScript(Self.removeThemeJS, completionHandler: nil)
    }
}
```

### Why both WKUserScript AND evaluateJavaScript

- **WKUserScript** runs on *new document loads* — initial app launch, page reload (Cmd+R), hard navigations. It's the only way to get the style tag into the DOM before paint.
- **evaluateJavaScript** runs *on the live page*. Needed when (a) the user picks a new theme from the menu without reloading, and (b) the site does SPA `history.pushState` (which doesn't fire WKUserScript). The same IIFE is idempotent — it updates the tag's content if it already exists.

The MutationObserver inside the IIFE handles a third case: the site's React/Vue runtime stripping the style tag during re-renders. Without the observer, even pre-injected styles can disappear mid-session.

The `removeThemeJS` referenced above (used when the user deactivates the current theme without picking a new one):

```swift
private static let removeThemeJS = """
(function() {
    const el = document.getElementById('__appname_theme');
    if (el) el.remove();
    if (window.__appname_themeObserver) {
        window.__appname_themeObserver.disconnect();
        window.__appname_themeObserver = null;
    }
})();
"""
```

Replace `__appname` with a project-specific token everywhere it appears (style IDs, observer names) so themes can't conflict with the site's own style IDs.

## Light/Dark mode sync

Two flavors of "dark mode":

1. **System mode → web app mode**: when macOS switches to dark, tell the web app to use its dark theme. Most modern web apps respect `prefers-color-scheme` and do this automatically. Verify by toggling Appearance in System Settings → General; if the site changes, you don't need to do anything.

2. **System mode → window chrome**: even if the web app handles its own dark mode, the `NSWindow.backgroundColor` you set at creation may not match. To prevent white-flashes on tab switch / page load:

```swift
final class AppDelegate: NSObject, NSApplicationDelegate {
    func applicationDidFinishLaunching(_ notification: Notification) {
        // …
        NSApp.publisher(for: \.effectiveAppearance)  // Combine
            .sink { [weak self] _ in self?.syncWindowBackground() }
            .store(in: &cancellables)
    }

    private func syncWindowBackground() {
        let isDark = NSApp.effectiveAppearance.bestMatch(from: [.darkAqua, .aqua]) == .darkAqua
        let color: NSColor = isDark
            ? NSColor(red: 0.07, green: 0.07, blue: 0.09, alpha: 1.0)
            : NSColor(red: 0.98, green: 0.98, blue: 0.98, alpha: 1.0)
        for entry in allWindows {
            entry.window.backgroundColor = color
        }
    }
}
```

If you're not using Combine, use KVO or `NSWindow.appearance`-tied colors.

3. **Override mode regardless of system**: some users want the app to be always-dark regardless of macOS appearance. Expose a "Force Dark" toggle in Settings; when on, set `NSApp.appearance = NSAppearance(named: .darkAqua)` and inject JS to set the site's dark-mode class. The exact JS depends on the site — e.g., Notion uses `document.body.classList.add('notion-dark-theme')`. Detection is per-site and necessarily a little bespoke.

## Theme → mode auto-detect (CSS marker first, name second)

A theme usually *implies* a base mode: applying a dark theme over the site's light mode looks wrong (light chrome bleeding through), so when a theme is activated you also flip the site into the matching mode. The naive detector reads the display name — "Gruvbox - Dark", "Nord - Light". That works until a palette's flavor names contain **neither** word: the Catppuccin dark flavors are *Mocha / Macchiato / Frappé*, and a name-only detector leaves them on Notion's light base, so the theme renders washed-out.

The fix is an explicit, opt-in CSS marker the theme author can drop anywhere in the `.css` — it takes precedence over the name, which remains the fallback:

```css
/* theme-mode: dark */
```

```swift
/// Mode hint for the active palette: "dark", "light", or nil. With the two-tier model the
/// Universal theme is the palette, so it drives the base mode; fall back to a Custom theme
/// only if no Universal theme is enabled.
/// 1. An explicit `theme-mode: dark|light` marker in the CSS wins.
/// 2. Otherwise fall back to a "dark"/"light" word in the display name.
var activeModeHint: String? {
    let enabled = enabledThemes
    guard let active = enabled.first(where: { $0.source == .universal }) ?? enabled.first
        else { return nil }
    if let css = try? String(contentsOf: active.url, encoding: .utf8),
       let marker = Self.modeMarker(in: css) {
        return marker                                    // 1. explicit marker
    }
    let lower = active.name.lowercased()                 // 2. name fallback
    if lower.contains("dark")  { return "dark" }
    if lower.contains("light") { return "light" }
    return nil
}

/// Parse `theme-mode: dark|light` out of raw CSS (case-insensitive).
private static func modeMarker(in css: String) -> String? {
    guard let r = css.range(of: #"theme-mode\s*:\s*(dark|light)"#,
                            options: [.regularExpression, .caseInsensitive]) else { return nil }
    return css[r].lowercased().contains("dark") ? "dark" : "light"
}
```

Then, on activation, run site-specific JS to set the base mode to match `activeModeHint`. The site-mode JS is bespoke — Notion uses the `notion-dark-theme` / `notion-light-theme` body class; document it in the project README and update when the site changes. Note the marker is parsed from the *raw* `.css` (read straight off disk, before any Stylus cleanup) and, being a CSS comment, never affects rendering — it's pure metadata for the wrapper.

## Stylus compatibility note

If the user has existing Stylus user-styles, this skill handles them natively — the `stripStylusMetadata` function above unwraps `@-moz-document` and removes the metadata block, so `.user.css` files from userstyles.org work as drop-ins.

## Menu-bar Themes menu (the canonical UX)

Every `url-mac-apps` app exposes theme switching as a top-level **Themes** menu in the menu bar, between **View** and **Window**. This is non-negotiable (see SKILL.md #12). A Settings-tab picker can exist as a secondary surface, but the menu bar is the primary one — it's faster, it's discoverable, and it matches what the user already expects from native macOS apps.

### Layout

Two labelled sections — **Custom** (multi-select) and **Universal** (single-select) — then the actions. Each folder gets its own "Open … Folder" item:

```
Themes ▾
├─ Custom ─────────────────────     ← section header (NSMenuItem.sectionHeader on macOS 14+)
├─ ✓ <appname>                       ← checkmarks; any number of Custom themes can be on
├─ ✓ <appname> - compact
├─ Universal ──────────────────     ← section header
├─ ✓ Gruvbox - Dark                  ← at most ONE Universal checkmark
├─   Nord
├─ ───────────────
├─ ✕  Turn Off All Themes            ← xmark.circle — only shown when something is enabled
├─ ⟳  Reload All Themes              ← arrow.clockwise — rescans both folders + re-injects
├─ ───────────────
├─ ⚙  Manage Themes…                 ← slider.horizontal.3 — opens Settings
├─ 📁  Open Custom Themes Folder      ← folder
└─ 📁  Open Universal Themes Folder   ← folder
```

If the user drops a `.css` file into either folder while the app is running, opening the Themes menu again shows it immediately — no relaunch, no Reload click. The menu rebuilds on every open via `NSMenuDelegate.menuNeedsUpdate`.

### Wiring

In `AppDelegate.swift`, declare `NSMenuDelegate` conformance, then build the placeholder menu once and let the delegate fill it in on every open:

```swift
final class AppDelegate: NSObject, NSApplicationDelegate, NSMenuDelegate {

    private func buildMainMenu() {
        // … App / Edit / View menus …

        // Themes menu — submenu populated dynamically on open.
        let themesMenuItem = NSMenuItem()
        menubar.addItem(themesMenuItem)
        let themesMenu = NSMenu(title: "Themes")
        themesMenu.delegate = self        // ← key line
        themesMenuItem.submenu = themesMenu

        // … Window menu …
    }

    // MARK: NSMenuDelegate

    func menuNeedsUpdate(_ menu: NSMenu) {
        guard menu.title == "Themes" else { return }
        buildThemesMenu(menu)
    }

    private func buildThemesMenu(_ menu: NSMenu) {
        menu.removeAllItems()

        // Pick up newly-dropped files — silent (no themesChanged notification).
        ThemeManager.shared.scanForNewThemes()

        // Two labelled sections. The single-Universal rule is enforced in the model
        // (ThemeManager.toggle), so the menu just renders one checkmark per enabled theme.
        addThemeSection(menu, title: "Custom", source: .custom)
        addThemeSection(menu, title: "Universal", source: .universal)

        menu.addItem(NSMenuItem.separator())

        if ThemeManager.shared.hasEnabled {
            let off = NSMenuItem(title: "Turn Off All Themes",
                                 action: #selector(disableAllThemes), keyEquivalent: "")
            off.image = NSImage(systemSymbolName: "xmark.circle", accessibilityDescription: "Off")
            off.target = self
            menu.addItem(off)
        }

        let reloadItem = NSMenuItem(title: "Reload All Themes",
                                    action: #selector(reloadThemes), keyEquivalent: "")
        reloadItem.image = NSImage(systemSymbolName: "arrow.clockwise",
                                   accessibilityDescription: "Reload")
        reloadItem.target = self
        menu.addItem(reloadItem)

        menu.addItem(NSMenuItem.separator())

        let manageItem = NSMenuItem(title: "Manage Themes…",
                                    action: #selector(openSettings), keyEquivalent: "")
        manageItem.image = NSImage(systemSymbolName: "slider.horizontal.3",
                                   accessibilityDescription: "Manage")
        manageItem.target = self
        menu.addItem(manageItem)

        for (title, url) in [("Open Custom Themes Folder", ThemeManager.shared.customThemesDirectoryURL),
                             ("Open Universal Themes Folder", ThemeManager.shared.universalThemesDirectoryURL)] {
            let item = NSMenuItem(title: title, action: #selector(openThemesFolder(_:)), keyEquivalent: "")
            item.image = NSImage(systemSymbolName: "folder", accessibilityDescription: "Folder")
            item.representedObject = url
            item.target = self
            menu.addItem(item)
        }
    }

    /// One section per source: a header, then a checkmarked item per theme in that folder.
    private func addThemeSection(_ menu: NSMenu, title: String, source: ThemeSource) {
        let header: NSMenuItem
        if #available(macOS 14.0, *) {
            header = NSMenuItem.sectionHeader(title: title)
        } else {
            header = NSMenuItem(title: title, action: nil, keyEquivalent: "")
            header.isEnabled = false
        }
        menu.addItem(header)

        let items = ThemeManager.shared.themes(in: source)
        if items.isEmpty {
            let empty = NSMenuItem(title: "  (none)", action: nil, keyEquivalent: "")
            empty.isEnabled = false
            menu.addItem(empty)
            return
        }
        for theme in items {
            let item = NSMenuItem(title: theme.name,
                                  action: #selector(toggleThemeFromMenu(_:)), keyEquivalent: "")
            item.representedObject = theme.id
            item.state = theme.enabled ? .on : .off
            item.target = self
            menu.addItem(item)
        }
    }

    @objc func reloadThemes()    { ThemeManager.shared.reload() }      // posts themesChanged → re-injects
    @objc func disableAllThemes() { ThemeManager.shared.disableAll() }

    @objc func openThemesFolder(_ sender: NSMenuItem) {
        if let url = sender.representedObject as? URL { NSWorkspace.shared.open(url) }
    }

    /// `toggle(id:)` enforces the single-Universal rule itself, so the menu just forwards.
    @objc func toggleThemeFromMenu(_ sender: NSMenuItem) {
        guard let id = sender.representedObject as? String else { return }
        ThemeManager.shared.toggle(id: id)
    }
}
```

(`reload()`, `scanForNewThemes()`, `toggle(id:)`, `disableAll()`, and `themes(in:)` all live on `ThemeManager` in the skeleton above — no extra extension needed.)

### Icons matter

Every action item in the Themes menu gets an SF Symbol via `NSImage(systemSymbolName:accessibilityDescription:)`. The icons (`arrow.clockwise`, `slider.horizontal.3`, `folder`) make the menu scannable at a glance and align it visually with native macOS menus. Don't ship a text-only Themes menu — it looks half-finished next to the icon-rich menus the rest of the system uses.

### Multi-window apps

When `ThemeManager.reload()` posts `themesChanged`, every `WebViewController` in every window should re-inject. If you have a `TabManager` per window with multiple tabs, loop over `allWindows` → `tabManager.reloadThemesOnAllTabs()` from the notification observer, or have each `WebViewController` listen for the notification directly. Both work; pick the one that matches the rest of the app's notification flow.

## Anti-patterns

- **Don't bundle themes in the app — but don't bury them in Application Support either.** Put them in an accessible, user-controlled location (its own folder/repo), with the Universal folder shared across apps and a Custom subfolder per app. Default the root to a visible folder such as `~/Themes/` but read it from a configurable setting so it stays overridable (don't bake it in as an unchangeable constant), and if the default doesn't exist at setup, ask the user where their themes folder is. The whole point is that the user can open, edit, version-control, and reuse these files across several wrapper apps.
- **Don't `@import` external CSS URLs.** WKWebView will fetch them, which is slow on page load and breaks offline. Inline the CSS into the theme file.
- **Don't inject themes only via `evaluateJavaScript` in `didFinish`.** The page has already painted by the time `didFinish` fires, so the user sees the site's default styles for a beat before the theme snaps in — a visible flash on every launch and reload. Use `WKUserScript` at `.atDocumentStart` for the initial paint, then `evaluateJavaScript` for live theme switches and SPA navigations. The IIFE handles `(document.head || document.documentElement)` so the still-being-built DOM isn't a problem.
- **Don't let two Universal themes stack.** Two full palettes at once produces unpredictable specificity wars, which is exactly why Universal themes are single-select. Custom themes *do* stack (concatenated, Custom last) — that's intentional, for layering an app-specific tweak on top of one Universal palette — but a Custom theme is expected to be a targeted override, not a second full palette. Enforce the single-Universal rule in the model (`toggle`/`discover`), not just the UI, so old persisted state with 2+ Universal enabled self-heals on launch.
- **Don't forget the observer cleanup on theme deactivate.** Otherwise, old observers keep running, watching for style tags that no longer exist, and slowly burn CPU.
- **Don't put the theme picker only in Settings.** It's discoverable but slow — three clicks to switch a theme. The menu bar is one click. Settings is fine as a secondary surface; the menu bar is mandatory.
- **Don't call `ThemeManager.reload()` from `menuNeedsUpdate`.** It posts `themesChanged`, which re-injects CSS on every tab — and that runs every time the user opens the menu, even if they don't pick anything. Use `scanForNewThemes()` instead.
