# Tabs, Multi-Window, and Keyboard Focus

The architecture: **one `TabManager` per window**, each `TabManager` owns N `WebViewController` instances. The `AppDelegate` keeps a registry of every open window so menu actions (Cmd+R, save session, Window menu) find the right one. The `TabManager` is stored as an associated object on its `NSWindow`, so given any window you can fetch its tabs without a separate map.

This file covers: TabManager skeleton, multi-window registry, dynamic Window menu, the `makeFirstResponder` discipline that cures "typing doesn't work" bugs, Cmd+T new tab, Cmd+W close, tab duplication, and the SPA-aware new-tab flow.

## Why one TabManager per window?

The naive design is a single global tab list. That's wrong: macOS users expect each window to be independent. Closing window B should not affect window A's tabs. Cmd+R should reload the *front* window's active tab. Session save needs to know which tabs belong to which window. One TabManager per window models this correctly.

## TabManager skeleton

```swift
import AppKit
import WebKit

final class TabManager: NSObject {
    private weak var containerView: NSView?
    private var tabs: [Tab] = []
    private var activeTabIndex: Int = 0
    private var tabBar: NSStackView!
    private let tabBarHeight: CGFloat = 36

    struct Tab {
        let id: UUID
        let controller: WebViewController
        weak var tabButton: NSButton?
    }

    init(containerView: NSView) {
        self.containerView = containerView
        super.init()
        buildTabBar()
    }

    // MARK: - Public API

    @discardableResult
    func addTab(url: URL, inBackground: Bool = false) -> Tab {
        let controller = WebViewController()
        controller.onTitleChanged = { [weak self] title in self?.updateTabTitle(controllerID: controller.id, title: title) }
        controller.onOpenInNewTab = { [weak self] newURL, background in self?.addTab(url: newURL, inBackground: background) }
        controller.load(url: url)

        let tab = Tab(id: UUID(), controller: controller, tabButton: nil)
        tabs.append(tab)
        rebuildTabBar()
        if !inBackground {
            selectTab(at: tabs.count - 1)
        }
        notifySessionDirty()
        return tab
    }

    func closeTab(at index: Int) {
        guard tabs.indices.contains(index) else { return }
        let removed = tabs.remove(at: index)
        removed.controller.view.removeFromSuperview()
        if tabs.isEmpty {
            containerView?.window?.close()
            return
        }
        if activeTabIndex >= tabs.count { activeTabIndex = tabs.count - 1 }
        rebuildTabBar()
        selectTab(at: activeTabIndex)
        notifySessionDirty()
    }

    func selectTab(at index: Int) {
        guard tabs.indices.contains(index) else { return }
        for (i, tab) in tabs.enumerated() {
            tab.controller.view.isHidden = (i != index)
        }
        activeTabIndex = index
        layoutActiveWebView()

        // Critical: keystrokes go to the web view immediately. Without this,
        // the user has to click in the page before typing works.
        if let activeWebView = tabs[index].controller.webView,
           let window = containerView?.window {
            window.makeFirstResponder(activeWebView)
        }
        rebuildTabBar()
    }

    func duplicateTab(at index: Int) {
        guard tabs.indices.contains(index),
              let currentURL = tabs[index].controller.webView?.url else { return }
        // Insert immediately to the right of the source tab — matches Safari/Chrome behavior.
        let newTab = makeTab(url: currentURL)
        tabs.insert(newTab, at: index + 1)
        rebuildTabBar()
        selectTab(at: index + 1)
    }

    func reloadActiveTab() {
        tabs[safe: activeTabIndex]?.controller.webView?.reload()
    }

    var activeWebView: WKWebView? { tabs[safe: activeTabIndex]?.controller.webView }
    var allURLs: [URL] { tabs.compactMap { $0.controller.webView?.url } }

    // MARK: - Internal

    private func makeTab(url: URL) -> Tab {
        let controller = WebViewController()
        controller.load(url: url)
        return Tab(id: UUID(), controller: controller, tabButton: nil)
    }

    private func layoutActiveWebView() {
        guard let container = containerView else { return }
        let bounds = container.bounds
        let webFrame = NSRect(x: 0, y: 0, width: bounds.width, height: bounds.height - tabBarHeight)
        tabs[safe: activeTabIndex]?.controller.view.frame = webFrame
        if let view = tabs[safe: activeTabIndex]?.controller.view, view.superview != container {
            container.addSubview(view)
        }
    }

    private func buildTabBar() { /* … glassmorphic NSStackView at top with per-tab buttons … */ }
    private func rebuildTabBar() { /* … */ }
    private func updateTabTitle(controllerID: UUID, title: String) { /* … */ }
    private func notifySessionDirty() {
        NotificationCenter.default.post(name: .sessionStateChanged, object: nil)
    }
}

private extension Array {
    subscript(safe index: Int) -> Element? { indices.contains(index) ? self[index] : nil }
}

extension Notification.Name {
    static let sessionStateChanged = Notification.Name("sessionStateChanged")
}
```

The `Tab` struct keeps a weak ref to its tab-bar button so updates (title change, active state) are cheap. The `activeWebView` and `allURLs` accessors are what the session manager and AppDelegate use.

## Multi-window registry in AppDelegate

```swift
final class AppDelegate: NSObject, NSApplicationDelegate {
    var allWindows: [(window: NSWindow, tabManager: TabManager)] = []

    func openInNewWindow(url: URL) {
        let win = NSWindow(
            contentRect: NSRect(x: 0, y: 0, width: 1200, height: 800),
            styleMask: [.titled, .closable, .miniaturizable, .resizable, .fullSizeContentView],
            backing: .buffered,
            defer: false
        )
        win.title = "<App Display Name>"
        win.titlebarAppearsTransparent = true
        win.titleVisibility = .hidden
        win.backgroundColor = NSColor(red: 0.07, green: 0.07, blue: 0.09, alpha: 1.0)
        win.minSize = NSSize(width: 800, height: 500)
        win.collectionBehavior = [.fullScreenPrimary]

        let container = NSView(frame: NSRect(origin: .zero, size: win.frame.size))
        win.contentView = container

        let tabManager = TabManager(containerView: container)
        tabManager.addTab(url: url)

        // Pin TabManager lifetime to the window via associated object.
        objc_setAssociatedObject(win, &TabManagerKey.key, tabManager, .OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        allWindows.append((win, tabManager))

        NotificationCenter.default.addObserver(forName: NSWindow.willCloseNotification, object: win, queue: .main) { [weak self] _ in
            self?.allWindows.removeAll { $0.window === win }
        }

        win.center()
        win.makeKeyAndOrderFront(nil)
        if let webView = tabManager.activeWebView {
            win.makeFirstResponder(webView)
        }
    }
}

private struct TabManagerKey { static var key = 0 }
```

### Why the associated object?

The TabManager isn't part of the `NSWindow`'s view hierarchy — it's a controller object. If you only keep it in `allWindows`, that's fine, but you lose the ability to fetch "given this window, what's its TabManager?" without searching the array. The associated object makes it O(1) and survives the window passing through delegate callbacks that only have an `NSWindow` reference.

## Routing menu actions to the right window

The trap: `@IBAction` style methods on AppDelegate operate on `mainWindow`, but the user expects them to operate on whichever window is *frontmost*. Always route through `NSApp.keyWindow` (or `mainWindow` as fallback).

**Centralize the resolution in ONE shared property and route every action through it — the bug re-enters per-action, not per-app.** In one wrapper, reload and spell-check had hand-rolled keyWindow resolution while ⌘T, ⌘1-9, ⌘[/⌘], ⌘W, zoom, and Inspect Element all still hardcoded the main window's manager — and because a hidden-on-close main window is a first-class state in this architecture, each of those read as a *dead key* in secondary windows rather than as acting on the wrong window. Local `NSEvent` key-monitor branches are exactly as affected as menu items. The sweep that finds the class: `grep -n '[^.a-zA-Z]tabManager\.' AppDelegate.swift` — anything in an `@objc` action or key-monitor branch that isn't launch/restore/session wiring is a latent wrong-window bug.

```swift
/// Non-content key windows (Settings, Sessions) carry no tabManager and fall
/// back to the main window. Return the WINDOW too — close semantics need it.
var frontmostWindowEntry: (window: NSWindow, tabManager: TabManager) {
    let keyWindow = NSApp.keyWindow ?? mainWindow!
    if keyWindow !== mainWindow,
       let manager = objc_getAssociatedObject(keyWindow as Any, "tabManager") as? TabManager {
        return (keyWindow, manager)
    }
    return (mainWindow, tabManager)
}
var frontmostTabManager: TabManager { frontmostWindowEntry.tabManager }

@objc func reloadActiveTab() { frontmostTabManager.activeWebViewController?.reload() }

@objc func closeTab() {   // the ONE action needing the window, not just the manager
    let (window, manager) = frontmostWindowEntry
    if manager.tabCount > 1 {
        manager.closeCurrentTab()
    } else if window === mainWindow {
        mainWindow.orderOut(nil)      // main window hides on close — match it
    } else {
        window.performClose(nil)      // secondaries really close (willClose teardown)
    }
}
```

## Dynamic Window menu

`NSMenuDelegate` lets you rebuild the Window submenu on open so it shows all currently-open windows:

```swift
extension AppDelegate: NSMenuDelegate {
    func menuNeedsUpdate(_ menu: NSMenu) {
        guard menu.identifier?.rawValue == "WindowMenu" else { return }
        // Keep the first few static items (Minimize, Zoom, …), then list windows.
        while menu.items.count > staticWindowMenuItemCount {
            menu.removeItem(at: menu.items.count - 1)
        }
        for (i, entry) in allWindows.enumerated() {
            let title = entry.window.title.isEmpty ? "Window \(i + 1)" : entry.window.title
            let item = NSMenuItem(title: title, action: #selector(focusWindowFromMenu(_:)), keyEquivalent: "")
            item.target = self
            item.representedObject = entry.window
            menu.addItem(item)
        }
    }

    @objc func focusWindowFromMenu(_ sender: NSMenuItem) {
        guard let win = sender.representedObject as? NSWindow else { return }
        win.makeKeyAndOrderFront(nil)
    }
}
```

Wire it up: assign `windowMenu.identifier = NSUserInterfaceItemIdentifier("WindowMenu")` and `windowMenu.delegate = self` when building the menu bar.

## The `makeFirstResponder` discipline

This is the single most-impactful pattern in the entire skill. Every time **any** of these happens, explicitly call `window.makeFirstResponder(activeWebView)`:

1. A new tab is created and becomes active.
2. An existing tab is selected (clicked or via Cmd+1..9).
3. A new window is opened.
4. A session is restored.
5. The app comes to the foreground after being hidden.

```swift
func applicationDidBecomeActive(_ notification: Notification) {
    for entry in allWindows {
        if entry.window.isKeyWindow, let wv = entry.tabManager.activeWebView {
            entry.window.makeFirstResponder(wv)
        }
    }
}
```

Without this, the WKWebView is a passive view — the window has focus but keystrokes go to the empty first responder chain and silently vanish. Users will report "the app feels broken on launch" without being able to articulate why.

## Cmd+T, Cmd+W, Cmd+Shift+T, Cmd+1..9

Map these in the menu bar with `keyEquivalent`s. Don't use a global key event monitor — menu shortcuts are more discoverable, work consistently with macOS conventions, and respect "Disable shortcut" preferences.

```swift
let fileMenu = NSMenu(title: "File")
fileMenu.addItem(withTitle: "New Tab", action: #selector(newTab), keyEquivalent: "t")
fileMenu.addItem(withTitle: "New Window", action: #selector(newWindow), keyEquivalent: "n")
fileMenu.addItem(NSMenuItem.separator())
fileMenu.addItem(withTitle: "Close Tab", action: #selector(closeTab), keyEquivalent: "w")
fileMenu.addItem(withTitle: "Close Window", action: #selector(closeWindow), keyEquivalent: "W")  // Cmd+Shift+W
fileMenu.addItem(withTitle: "Reopen Closed Tab", action: #selector(reopenClosedTab), keyEquivalent: "T")  // Cmd+Shift+T

let viewMenu = NSMenu(title: "View")
viewMenu.addItem(withTitle: "Reload", action: #selector(reloadActiveTab), keyEquivalent: "r")
// Standing default (SKILL.md item 37): the View menu ends with the zoom block.
viewMenu.addItem(NSMenuItem.separator())
let zoomLevel = NSMenuItem(title: "Zoom: 100%", action: nil, keyEquivalent: "")
zoomLevel.isEnabled = false  // rewritten on every zoom change
viewMenu.addItem(zoomLevel)
viewMenu.addItem(withTitle: "Zoom In", action: #selector(zoomIn), keyEquivalent: "=")
viewMenu.addItem(withTitle: "Zoom Out", action: #selector(zoomOut), keyEquivalent: "-")
viewMenu.addItem(withTitle: "Actual Size", action: #selector(zoomReset), keyEquivalent: "0")
```

(Key equivalents shown as the defaults — in the real app they come from the shortcut registry, like every other combo. Zoom is silent: no toast or notification when these fire.)

For Cmd+1..9 (jump to tab N), add nine menu items in a "Window" or hidden submenu with `keyEquivalent: "1"` … `"9"`, each calling `selectTab(at:)` with a static N.

## Reopen-Closed-Tab stack

Keep a stack of recently-closed tab URLs per window. On close, push. On Cmd+Shift+T, pop and reopen at the same position.

```swift
private var closedTabStack: [(url: URL, index: Int)] = []

func closeTab(at index: Int) {
    if let url = tabs[index].controller.webView?.url {
        closedTabStack.append((url, index))
        if closedTabStack.count > 50 { closedTabStack.removeFirst() }
    }
    // … rest of close logic
}

func reopenClosedTab() {
    guard let last = closedTabStack.popLast() else { return }
    let restoreIndex = min(last.index, tabs.count)
    let tab = makeTab(url: last.url)
    tabs.insert(tab, at: restoreIndex)
    rebuildTabBar()
    selectTab(at: restoreIndex)
}
```

## SPA-aware new-tab flow

Single-page apps use `history.pushState` to navigate without firing `decidePolicyFor`. If the user middle-clicks a row in a data table, the SPA might call `pushState` on the current frame instead of opening a new tab. To make middle-click-opens-in-new-tab work uniformly, intercept in JS:

```javascript
// Inject at document-start
(function() {
    let lastMiddleClickAt = 0;
    document.addEventListener('mousedown', (e) => {
        if (e.button === 1) lastMiddleClickAt = Date.now();
    }, true);

    const origPushState = history.pushState;
    history.pushState = function(state, title, url) {
        const wasMiddleClick = (Date.now() - lastMiddleClickAt) < 100;
        if (wasMiddleClick && url) {
            const abs = new URL(url, location.href).href;
            window.webkit.messageHandlers.openInNewTab.postMessage({ url: abs });
            return;  // Don't actually navigate.
        }
        return origPushState.apply(this, arguments);
    };
})();
```

On the Swift side, register the message handler and route to `addTab(url:inBackground:)`. The 100ms window catches middle-click → pushState cases without false-positives.

## Cmd+T search (focus the URL bar, optional)

Many wrappers add a Cmd+T behavior that opens a "search this site" overlay or focuses an in-page search box rather than opening a blank new tab. If the target site has a global search hotkey (e.g., `/`), you can route Cmd+T to inject that keystroke into a new tab so the user lands ready-to-type. Trade-off: it's per-site and brittle. Default to plain new-tab unless the user asks for it.

## When the last tab closes

Decide policy up front:

- **Close the window**: simple, matches Safari. The TabManager skeleton above does this.
- **Open a blank/start page in the same tab**: keeps the window alive. Add `if tabs.isEmpty { addTab(url: startPageURL); return }` instead of `window.close()`.

The first is usually right.

## Single-tab auto-hide + "Always Show Tabs"

A wrapper that's usually used with one tab looks cleanest when the tab bar
*hides* until there's a second tab — the window then reads like a chromeless
web app, and a top-edge hover reveals the bar. But some users want the bar
always visible (it's where the active-page title and tab affordances live), so
expose an **Always Show Tabs** toggle that overrides the auto-hide. Keep both: a single global `alwaysShowTabs` default, applied live to every open window.

```swift
private var alwaysShowTabs: Bool { UserDefaults.standard.bool(forKey: "alwaysShowTabs") }

/// Bar is visible when there's more than one tab OR the user pinned it on.
private func updateTabBarVisibility() {
    if tabs.count > 1 || alwaysShowTabs { showTabBar(animated: false) }
    else { hideTabBar(animated: false) }
}

/// Called from the menu/Settings toggle so the change lands in every window now.
func refreshTabBarVisibility() { updateTabBarVisibility() }
```

Two subtleties that bite:

- **The hover-reveal timer must respect the toggle.** The mouse-exit auto-hide should only fire when `tabs.count == 1 && !alwaysShowTabs`. Guard `hideTabBar(animated:)` on `!alwaysShowTabs` so a stray hover-out doesn't collapse a pinned bar.
- **Keep a separate "force" path for typing-hide.** If you also hide the bar while the user types (a Zen-mode behavior), that's a *forced* hide that should bypass the `alwaysShowTabs` guard — otherwise Always Show Tabs would defeat typing-hide. Two code paths: the count-driven auto-hide (respects the toggle) and the explicit `hideTabBarForTyping()` (forces it regardless).
- **Apply live to all windows.** When the toggle flips, loop `allWindows` and call `refreshTabBarVisibility()` on each TabManager — don't wait for the next launch.

## Tab bar appearance presets (parallels Themes)

Once the tab bar is themeable (colors, active/inactive tints, close-button
size), users want to *save* a look and switch between saved looks — exactly
like CSS themes. Mirror the `ThemeManager` design with a `TabBarPresetManager`:

- Persist named snapshots of the tab-bar color tokens + close-button size to a
  JSON manifest in `~/Library/Application Support/<AppName>/`.
- Surface them through a top-level **Tabs** menu (list presets with a checkmark
  on the active one → apply / save current as… / reset to default / Manage…)
  *and* a presets section in the Settings "Tab Bar" tab — same "menu is the
  primary surface, Settings is secondary" rule as Themes (non-negotiable #13).
- **Track the active preset honestly.** Store the active preset id, but the
  moment the user hand-edits any token away from the preset's saved value,
  *clear the marker* (the look is now "Custom"/unsaved). Post a change
  notification so an open Settings list updates its checkmark live.

This is the generalizable lesson: **any multi-token visual customization in the
wrapper (tab bar, accent colors, future chrome) should follow the Themes shape**
— Application-Support JSON, a menu-bar list that rebuilds on open, an
active-marker that hand-edits invalidate, and capture into sessions (below).

## Common bugs

- **Tab title flashes**: when a new tab is created and immediately navigated to the same URL as another tab, both tabs may briefly show the same title. Fix by debouncing `onTitleChanged` to ignore titles that match the URL string verbatim — those are placeholder titles before the page loads.
- **Cmd+R reloads wrong window**: route through `NSApp.keyWindow`, not `mainWindow`.
- **Multi-window Cmd+W closes wrong tab**: same issue, same fix.
- **Tab duplication ends up at the end of the bar**: insert at `sourceIndex + 1`, not `tabs.count`.
- **Background tabs lose state when made active**: don't recreate the WKWebView when switching tabs; just `isHidden = false` on the existing view.
