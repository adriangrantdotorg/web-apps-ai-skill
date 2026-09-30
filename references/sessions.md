# Session Persistence

Sessions = saved state of all open windows + their tabs + their frames. Users expect that closing the app and reopening it restores everything *exactly* as it was. They also expect to be able to *name* sessions (e.g., "Work setup", "Research dump") and switch between them.

This file covers: the `SessionManager` JSON shape, where to store it, save triggers, restore-before-UI launch order, and a named-sessions UI.

## Storage location

```
~/Library/Application Support/<AppName>/sessions/sessions.json
```

Use the standard support directory — *not* `UserDefaults`. UserDefaults is fine for primitive settings but choked by a hundred-tab session blob. Filesystem JSON is human-readable, easy to back up, and easy to debug.

```swift
private static var sessionsDirectory: URL {
    let fm = FileManager.default
    let appSupport = fm.urls(for: .applicationSupportDirectory, in: .userDomainMask).first!
    let dir = appSupport.appendingPathComponent("<AppName>/sessions", isDirectory: true)
    try? fm.createDirectory(at: dir, withIntermediateDirectories: true)
    return dir
}
```

## Data shape

```swift
struct SessionTab: Codable {
    let title: String
    let url: String
}

struct SessionWindow: Codable {
    let tabs: [SessionTab]
    let activeTabIndex: Int
    var frameString: String?   // NSStringFromRect(window.frame)
}

struct Session: Codable, Identifiable {
    var id: String              // UUID
    var name: String            // Display name (user-set or auto-generated)
    var windows: [SessionWindow]
    var savedAt: Date
    var appearance: SessionAppearance?   // global look at save time (see below)
}

/// The global visual state in effect when the session was saved, so reopening
/// it restores the same look — not just the same tabs.
struct SessionAppearance: Codable {
    var tabBarTokens: [String: String]?  // raw tab-bar color tokens + sizes,
                                         // captured directly so a customized-
                                         // but-unsaved look is preserved too
    var activeThemeId: String?           // CSS theme that was active
}

struct SessionStore: Codable {
    var lastSession: Session    // The auto-saved "where you left off" state.
    var named: [Session]        // User-named saved sessions.
}
```

The split between `lastSession` and `named` matters:

- `lastSession` is overwritten on every relevant event. It's not a "save" — it's a continuous mirror of "what's on screen right now."
- `named` are explicit snapshots the user created via "Save Session…". They never change unless the user re-saves with the same name.

This means accidental close → reopen always restores. And the user can keep their working state without polluting their named-sessions list.

## SessionManager skeleton

```swift
import AppKit

final class SessionManager {
    static let shared = SessionManager()

    private let fileURL: URL
    private let encoder = JSONEncoder()
    private let decoder = JSONDecoder()
    private var saveDebounceWork: DispatchWorkItem?

    private init() {
        let fm = FileManager.default
        let appSupport = fm.urls(for: .applicationSupportDirectory, in: .userDomainMask).first!
        let dir = appSupport.appendingPathComponent("<AppName>/sessions", isDirectory: true)
        try? fm.createDirectory(at: dir, withIntermediateDirectories: true)
        self.fileURL = dir.appendingPathComponent("sessions.json")
        encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
        encoder.dateEncodingStrategy = .iso8601
        decoder.dateDecodingStrategy = .iso8601
    }

    // MARK: - Persistence

    private func load() -> SessionStore? {
        guard let data = try? Data(contentsOf: fileURL),
              let store = try? decoder.decode(SessionStore.self, from: data) else { return nil }
        return store
    }

    private func write(_ store: SessionStore) {
        do {
            let data = try encoder.encode(store)
            try data.write(to: fileURL, options: .atomic)
        } catch {
            NSLog("Session save failed: \(error)")
        }
    }

    // MARK: - Public API

    func loadLastSession() -> Session? { load()?.lastSession }
    func loadNamedSessions() -> [Session] { load()?.named ?? [] }

    func saveLastSession(windows: [SessionWindow]) {
        let session = Session(
            id: "last",
            name: "Last Session",
            windows: windows,
            savedAt: Date()
        )
        var store = load() ?? SessionStore(lastSession: session, named: [])
        store.lastSession = session
        write(store)
    }

    func saveNamedSession(_ name: String, windows: [SessionWindow]) {
        var store = load() ?? SessionStore(
            lastSession: Session(id: "last", name: "Last Session", windows: [], savedAt: Date()),
            named: []
        )
        let trimmed = name.trimmingCharacters(in: .whitespacesAndNewlines)
        let finalName = trimmed.isEmpty ? autoSessionName() : trimmed
        let session = Session(id: UUID().uuidString, name: finalName, windows: windows, savedAt: Date())
        store.named.removeAll { $0.name == finalName }   // Overwrite same-name.
        store.named.append(session)
        store.named.sort { $0.savedAt > $1.savedAt }
        write(store)
    }

    func deleteNamedSession(id: String) {
        guard var store = load() else { return }
        store.named.removeAll { $0.id == id }
        write(store)
    }

    // MARK: - Debounced auto-save

    func scheduleAutoSave(_ collect: @escaping () -> [SessionWindow]) {
        saveDebounceWork?.cancel()
        let work = DispatchWorkItem { [weak self] in
            self?.saveLastSession(windows: collect())
        }
        saveDebounceWork = work
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.5, execute: work)
    }

    // MARK: - Helpers

    private func autoSessionName() -> String {
        let formatter = DateFormatter()
        formatter.dateFormat = "EEE, MMM d  h:mm:ssa"
        formatter.amSymbol = "am"; formatter.pmSymbol = "pm"
        return formatter.string(from: Date())
    }
}
```

The debounce matters: `windowDidEndLiveResize` fires once per resize, but multiple resize events in quick succession (e.g., user dragging) should collapse into one save. 500ms is the sweet spot.

## Save triggers

Auto-save on every event that changes the visible state:

| Event | Where to wire it |
|---|---|
| Window moved | `NSWindowDelegate.windowDidMove` |
| Window resized (end of drag) | `NSWindowDelegate.windowDidEndLiveResize` |
| Window enters/exits fullscreen | `windowDidEnterFullScreen`, `windowDidExitFullScreen` |
| Tab added | `TabManager.addTab` → `notifySessionDirty()` |
| Tab closed | `TabManager.closeTab` → `notifySessionDirty()` |
| Active tab changed | `TabManager.selectTab` → `notifySessionDirty()` |
| Navigation completed | `WKNavigationDelegate.didFinish` (the URL changed) |
| App will terminate | `applicationWillTerminate` (synchronous, not debounced) |

The `TabManager` posts `.sessionStateChanged`; the AppDelegate observes it once and calls `SessionManager.shared.scheduleAutoSave { … }`:

```swift
NotificationCenter.default.addObserver(forName: .sessionStateChanged, object: nil, queue: .main) { [weak self] _ in
    guard let self = self else { return }
    SessionManager.shared.scheduleAutoSave { self.collectCurrentWindows() }
}

private func collectCurrentWindows() -> [SessionWindow] {
    return allWindows.map { entry in
        let tabs = entry.tabManager.allURLs.map { url in
            SessionTab(title: entry.tabManager.title(for: url) ?? "", url: url.absoluteString)
        }
        return SessionWindow(
            tabs: tabs,
            activeTabIndex: entry.tabManager.activeIndex,
            frameString: NSStringFromRect(entry.window.frame)
        )
    }
}
```

For `applicationWillTerminate`, do a synchronous final save — debounce won't fire after the run loop ends:

```swift
func applicationWillTerminate(_ notification: Notification) {
    SessionManager.shared.saveLastSession(windows: collectCurrentWindows())
}
```

**Never rely on `applicationWillTerminate` alone — add a periodic timer (~60 s).** Crashes obviously skip it, but so does the everyday rebuild-and-relaunch cycle: a dev rebuild script that quits the app via `SIGTERM`/`SIGKILL` (to avoid the osascript Automation TCC prompt), and signals never deliver AppKit's termination callbacks. Any state saved only at quit silently evaporates on every rebuild. A `Timer.scheduledTimer(withTimeInterval: 60, repeats: true)` with `tolerance = 10` writing a few KB of JSON makes the snapshot lose at most a minute under any exit path.

## The last-viewed-tabs snapshot (start-page restore)

Distinct from *named* sessions: an invisible, automatically-maintained snapshot backing the "Open Last Viewed Tabs" start-page mode (`references/settings.md`). Learnings from the shipped implementation:

- **Separate file, not a hidden named session** (`last-viewed-tabs.json` next to the sessions dir). It must never appear in the sessions UI, and it captures one thing named sessions didn't: **`activeTabIndex` per window**, so launch puts the user back on the exact tab.
- **Write it on the quit hook *and* the 60 s timer above; write unconditionally** regardless of the current start-page mode — it's tiny, and it means flipping the mode on always has yesterday's tabs to restore.
- **Snapshot ONLY windows with `isVisible || isMiniaturized` — the zombie-window loop.** Wrappers hide (not close) the main window on its close button, so a "closed" main window keeps its live tabs in `allWindows`. An unfiltered snapshot re-saves that hidden window every minute; launch restore resurrects it; the user closes it again; it re-saves — an immortal stale window set that overrides whatever the user actually had open, every single launch. Filter at the save site, and keep the writer's existing guard against empty input (`guard !windows.isEmpty`) so an all-hidden moment (⌘H when the timer fires) preserves the last good snapshot instead of wiping it.
- **Skip tabs with unresolvable URLs when saving** (hibernated tabs should resolve through their remembered URL; genuinely blank tabs would restore as dead views) — and re-clamp `activeTabIndex` after filtering.
- **Restore at launch replaces the initial `addTab(startURL)` call**, mirroring named-session restore minus appearance and minus the close-everything preamble: first snapshot window fills the main window, remaining windows via a create-empty-window path (refactor the open-URL-in-new-window helper so the window shell is reusable without an eager first tab). Re-key the **main** window at the end; the last-created window otherwise steals key.
- **Restore tabs pre-hibernated with their saved titles — never eager-load N tabs at launch.** Eagerly loading every restored tab fires N simultaneous site loads: launch crawls, some loads wedge (stuck `isLoading` for 8+ min), and a single-tab secondary window sits black long enough to look broken. Instead create each restored tab directly in the hibernated state (`lastLoadedURL` + `hibernatedURL` set, `isHibernated = true`, **no load issued**) and set the tab's title from the snapshot; then one `selectTab` per window on the saved index wakes only the visible tab through the normal hibernation-wake path. Persisting the title matters beyond cosmetics: eager-restored background tabs used to be re-saved before finishing their load, so titles decayed to "New Tab" across restarts — with saved titles round-tripping, the snapshot stays stable.
- **Fall back gracefully**: snapshot missing/empty → last-viewed *page* → configured start page.

## Restore order on launch — critical

The naive flow is: show the main window with start URL, *then* restore. The user sees a flash of empty state.

The correct flow is:

1. In `applicationWillFinishLaunching` or at the top of `applicationDidFinishLaunching`, before any `makeKeyAndOrderFront`, load the last session.
2. If there are windows to restore, create them with their frames + tabs in place.
3. Only then call `makeKeyAndOrderFront` and `makeFirstResponder`.

```swift
func applicationDidFinishLaunching(_ notification: Notification) {
    buildMainMenu()

    if let last = SessionManager.shared.loadLastSession(), !last.windows.isEmpty {
        restore(last)
    } else {
        buildDefaultWindow()
    }
    // Background-relaunch gate (see project-skeleton.md — dev rebuilds never steal focus).
    if ProcessInfo.processInfo.environment["<APPNAME>_BACKGROUND_LAUNCH"] != nil {
        for window in NSApp.windows { window.orderBack(nil) }
    } else {
        NSApp.activate(ignoringOtherApps: true)
    }
}

private func restore(_ session: Session) {
    for (i, sessionWindow) in session.windows.enumerated() {
        let win = makeWindow()
        if let frameStr = sessionWindow.frameString {
            win.setFrame(NSRectFromString(frameStr), display: false)
        }
        let tm = TabManager(containerView: win.contentView!)
        objc_setAssociatedObject(win, &TabManagerKey.key, tm, .OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        for tab in sessionWindow.tabs {
            if let url = URL(string: tab.url) {
                tm.addTab(url: url, inBackground: true)
            }
        }
        tm.selectTab(at: min(sessionWindow.activeTabIndex, sessionWindow.tabs.count - 1))
        allWindows.append((win, tm))
        win.makeKeyAndOrderFront(nil)
        if let wv = tm.activeWebView, i == 0 {
            win.makeFirstResponder(wv)
        }
    }
}
```

`addTab(inBackground: true)` is important: during restore, you don't want each tab to steal focus as it loads. Set the active tab once at the end.

**Restore appearance *before* the tabs.** If a session carries a `SessionAppearance` (tab-bar tokens + active theme), apply it at the very top of `restore(_:)` — before any `TabManager` is built or any tab is added — so the tabs are created with the right tab-bar colors and the web views load already pre-themed (theme CSS is injected at `.atDocumentStart`; see `references/themes.md`). Applying appearance *after* tabs are added means a visible flash as colors/theme snap in on already-loaded pages. Capturing the *raw* tab-bar tokens (not just the active preset id) means a customized-but-unsaved look is preserved across the session round-trip, exactly like an unsaved theme.

## Multi-monitor & frame restoration

`NSStringFromRect`/`NSRectFromString` is a string-serializable rect format. macOS keeps coordinates in global screen space, so a window saved on a secondary display restores there too — as long as that display is still attached. If it isn't, the window will land off-screen.

Defensive fix: after `setFrame`, clamp to the visible-screens bounding box:

```swift
if let visible = NSScreen.screens.map({ $0.visibleFrame }).reduce(NSRect.null, { $0.union($1) }),
   !visible.intersects(win.frame) {
    win.center()
}
```

## Named sessions UI

Two menu items in the File menu:

- "Save Session…" → prompt for a name, call `saveNamedSession`.
- "Open Session" → submenu listing saved sessions; clicking restores.

```swift
@objc func saveSessionMenuAction() {
    let alert = NSAlert()
    alert.messageText = "Save Session"
    alert.informativeText = "Name this session:"
    let input = NSTextField(frame: NSRect(x: 0, y: 0, width: 240, height: 24))
    input.stringValue = SessionManager.shared.defaultAutoName()
    alert.accessoryView = input
    alert.addButton(withTitle: "Save")
    alert.addButton(withTitle: "Cancel")
    if alert.runModal() == .alertFirstButtonReturn {
        SessionManager.shared.saveNamedSession(input.stringValue, windows: collectCurrentWindows())
    }
}
```

When opening a saved session, close existing non-empty windows first or restore into them — the latter is gentler:

```swift
@objc func openSession(_ sender: NSMenuItem) {
    guard let session = sender.representedObject as? Session else { return }
    // Close all but the main window; reuse the main window for the first restored window.
    for entry in allWindows.dropFirst().reversed() {
        entry.window.close()
    }
    // … then restore as in `restore(_:)`.
}
```

## When the user explicitly opens a new window

Don't auto-save *before* the new window has tabs — the debounce would write an empty window. Pattern: when creating a new empty window, defer the session-dirty notification until after the first tab has loaded its URL.

## Edge cases worth handling

- **The site URL changed shape** (e.g., the user switched accounts or workspaces). The saved URLs may 404. Don't crash — let the page load show its own error.
- **The saved tab URL is no longer in the allowlist** (rare, but if you tightened the allowlist between releases). The navigation delegate will redirect to the default browser. Detect this: if a tab's URL is rejected by the policy and there's no fallback URL, replace with the start page.
- **A stranded mid-auth URL was saved.** If the app was quit (or crashed, or the content process died) mid-sign-in, the last-committed URL can be an auth/redirect hop — `/login`, `/sso`, `/auth`, a token-exchange path, an identity-provider bounce. Restoring it just re-triggers the sign-in dance and can relaunch to a blank page. Guard **both** ends: when persisting a "last viewed URL," only store URLs on the wrapped site's *own* hosts; when restoring, treat a saved URL as restorable only if it's a normal page on those hosts **and** its path isn't one of the transient auth routes (`/login`, `/sign-in`, `/sso`, `/auth`, `/logout`, `/token-exchange`, …) — otherwise fall back to the start page. Keep small `isSiteHost(_:)` + `isRestorableURL(_:)` helpers for exactly this, and only write `lastViewedURL` when the current page passes the host check.
- **Hundreds of tabs**. Don't load them all eagerly. Use `addTab(inBackground:)` but defer the actual `webView.load(url:)` until the tab is first selected. Lazy session restore can take a 200-tab restore from 30 seconds to instant.

Lazy-load skeleton:

```swift
final class WebViewController {
    private var deferredURL: URL?

    func loadDeferred(url: URL) {
        deferredURL = url
        // Don't call webView.load yet.
    }

    func materialize() {
        guard let url = deferredURL else { return }
        deferredURL = nil
        webView.load(URLRequest(url: url))
    }
}

// In TabManager.selectTab(at:):
tabs[index].controller.materialize()
```

The tab bar shows the saved title (from the session JSON) until the user clicks the tab and it actually loads.
