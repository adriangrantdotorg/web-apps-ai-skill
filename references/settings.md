# Settings / Preferences Window

A native macOS Preferences window with category tabs at the top, AppKit-backed. Not SwiftUI Settings — AppKit gives finer control over WKWebView-aware toggles and avoids the SwiftUI lifecycle quirks that bite when settings need to mutate live web views.

This file covers: the `SettingsWindowController` shape, three standard category tabs (General, Editor, Themes), the WKWebView-aware toggles that almost every wrapper needs, a `SettingsDelegate` protocol for change notification, and how to apply changes to already-open web views without a full reload.

> **Alternative layout for larger settings surfaces:** once the app has many sections, a left icon nav (icon + name per section, selected row highlighted) scales better than top tabs; the controller/delegate + WKWebView-aware toggle wiring below is unchanged. Useful touches: optional shortcuts that open Settings straight to a section; a content pane of a section title plus bordered cards, each row a bold name + muted description + right-aligned control; **visual choices rendered as mini previews** (icon packs, display styles, themes) rather than radio labels; and every folder/file-location row with **Browse… + Reveal in Finder + Apply** next to a monospace path field (directly applicable to the Themes-folder and start-page settings in this file).

## Why AppKit, not SwiftUI Settings?

SwiftUI's `Settings { … }` scene is convenient but:

- Categories are styled differently across macOS versions in ways you can't override.
- Two-way binding to `@AppStorage` doesn't always fire when the underlying `UserDefaults` is changed externally (e.g., via `defaults write`).
- Custom controls (color pickers for theme tokens, font selectors, file pickers) need extra wrappers.
- The Settings scene has no good way to programmatically focus a specific tab.

An AppKit `NSWindowController` with an `NSTabView` is 200 lines, fully under control, and renders identically on macOS 13–15.

## Categories to include

The exact set depends on the app, but most URL wrappers want:

1. **General** — Start page URL, app icon picker, zen mode toggle, dock badge toggle.
2. **Editor** — WKWebView text-input behavior toggles (the WKWebView-aware ones below).
3. **Themes** — Theme import/select/delete. See `themes.md`.
4. **Tab Bar** *(optional)* — Custom colors / sizes for the tab bar UI.
5. **Advanced** *(optional)* — Reset all settings, clear web data, export sessions.

Drop anything that doesn't apply to the target site. Don't add categories with one toggle each — fold them into General.

## SettingsWindowController skeleton

```swift
import AppKit
import WebKit

protocol SettingsDelegate: AnyObject {
    func settingsDidChangeStartPage(_ url: URL)
    func settingsDidChangeEditor()         // autocorrect, smart quotes, etc.
    func settingsDidChangeThemes()
    func settingsDidChangeTabBarAppearance()
}

final class SettingsWindowController: NSWindowController {
    weak var settingsDelegate: SettingsDelegate?
    private var tabView: NSTabView!

    init() {
        let win = NSWindow(
            contentRect: NSRect(x: 0, y: 0, width: 560, height: 460),
            styleMask: [.titled, .closable, .miniaturizable],
            backing: .buffered,
            defer: false
        )
        win.title = "Settings"
        win.center()
        super.init(window: win)
        buildUI()
    }
    required init?(coder: NSCoder) { fatalError() }

    private func buildUI() {
        guard let contentView = window?.contentView else { return }
        tabView = NSTabView(frame: contentView.bounds)
        tabView.autoresizingMask = [.width, .height]
        contentView.addSubview(tabView)

        tabView.addTabViewItem(makeGeneralTab())
        tabView.addTabViewItem(makeEditorTab())
        tabView.addTabViewItem(makeThemesTab())
    }

    func focus(category: String) {
        if let idx = tabView.tabViewItems.firstIndex(where: { $0.label == category }) {
            tabView.selectTabViewItem(at: idx)
        }
        showWindow(nil)
        window?.makeKeyAndOrderFront(nil)
    }
}
```

Each `make…Tab()` returns an `NSTabViewItem` whose `view` contains the controls for that category. Build them with `NSStackView` for vertical layout; it handles spacing and alignment without manual frames.

## WKWebView-aware toggles — the important ones

These are the toggles every URL-wrapper user wants. They affect how WKWebView handles text input, and they're not configurable through any single SwiftUI modifier:

| Toggle | UserDefaults key | What it actually does |
|---|---|---|
| Spelling autocorrect | `NSAutomaticSpellingCorrectionEnabled` (+ `WebAutomaticSpellingCorrectionEnabled`) | macOS autocorrects misspelled words as you type. Persistent UserDefaults that WKWebView respects on creation. |
| Smart quotes | `NSAutomaticQuoteSubstitutionEnabled` | Converts `"foo"` to `"foo"`. Annoys programmers; loved by writers. |
| Smart dashes | `NSAutomaticDashSubstitutionEnabled` | `--` becomes em-dash. |
| Text replacement | `NSAutomaticTextReplacementEnabled` | macOS-wide replacements (e.g., `omw` → `on my way`). |
| Smart link detection | `NSAutomaticLinkDetectionEnabled` | URLs typed into editables become clickable links. |
| Text-completion bubble | `NSAutomaticTextCompletionEnabled` (+ legacy `WebAutomaticTextCompletionEnabled`) | The "Currently…" suggestion pill macOS pops up under the cursor as you type in a web field. A web app usually has its own autocomplete, so the OS bubble just overlaps it and gets in the way. |

**Disable the macOS text-suggestion pill in every wrapper.** This is the "Currently…" / capitalized-word bubble macOS pops under the cursor as you type in a web field (autocorrect + autocapitalize + inline prediction). **The UserDefaults toggles alone do *not* reliably suppress it in WKWebView** — the thing that does is setting WebKit's own per-field attributes off. Inject this as a `WKUserScript` at `.atDocumentStart` on every web view, re-applied via a MutationObserver (SPAs render inputs on demand, so a one-time pass misses the field the user actually clicks into) plus a focusin guard right before they type:

```js
(function() {
    function harden(el) {
        if (!el || el.nodeType !== 1) return;
        if (el.tagName !== 'INPUT' && el.tagName !== 'TEXTAREA' && !el.isContentEditable) return;
        if (el.getAttribute('autocorrect')    !== 'off')   el.setAttribute('autocorrect', 'off');
        if (el.getAttribute('autocapitalize') !== 'off')   el.setAttribute('autocapitalize', 'off');
        if (el.getAttribute('spellcheck')     !== 'false') el.setAttribute('spellcheck', 'false');
    }
    function sweep(root) {
        try { harden(root); root.querySelectorAll && root.querySelectorAll('input, textarea, [contenteditable]').forEach(harden); } catch (e) {}
    }
    sweep(document);
    document.addEventListener('DOMContentLoaded', () => sweep(document));
    if (!window.__app_inputObserver) {
        window.__app_inputObserver = new MutationObserver(ms => { for (const m of ms) for (const n of m.addedNodes) sweep(n); });
        window.__app_inputObserver.observe(document.documentElement, { childList: true, subtree: true });
    }
    document.addEventListener('focusin', e => harden(e.target), true);
})();
```

`autocorrect="off"` is the one that kills the pill — WebKit ties both autocorrection *and* inline predictions to it; `autocapitalize="off"` stops the capitalized-word suggestion; `spellcheck="false"` removes the squiggles. As a **backstop only**, also set these app-level defaults at launch (they don't suffice alone, but disable the separate completion-*list* feature): `NSAutomaticTextCompletionEnabled`, `WebAutomaticTextCompletionEnabled`, `NSAutomaticInlinePredictionEnabled` → `false`. Default this on in every wrapper; it isn't worth a settings checkbox.

```swift
private func makeEditorTab() -> NSTabViewItem {
    let item = NSTabViewItem(identifier: "editor")
    item.label = "Editor"

    let stack = NSStackView()
    stack.orientation = .vertical
    stack.alignment = .leading
    stack.spacing = 12
    stack.edgeInsets = NSEdgeInsets(top: 24, left: 24, bottom: 24, right: 24)

    func checkbox(_ title: String, key: String, onChange: @escaping () -> Void) -> NSButton {
        let cb = NSButton(checkboxWithTitle: title, target: self, action: #selector(checkboxToggled(_:)))
        cb.state = UserDefaults.standard.bool(forKey: key) ? .on : .off
        cb.identifier = NSUserInterfaceItemIdentifier(key)
        objc_setAssociatedObject(cb, &CheckboxCallback.key, onChange, .OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        return cb
    }

    stack.addArrangedSubview(checkbox("Correct spelling automatically",
                                      key: "NSAutomaticSpellingCorrectionEnabled") { [weak self] in
        self?.settingsDelegate?.settingsDidChangeEditor()
    })
    stack.addArrangedSubview(checkbox("Use smart quotes",
                                      key: "NSAutomaticQuoteSubstitutionEnabled") { [weak self] in
        self?.settingsDelegate?.settingsDidChangeEditor()
    })
    stack.addArrangedSubview(checkbox("Use smart dashes",
                                      key: "NSAutomaticDashSubstitutionEnabled") { [weak self] in
        self?.settingsDelegate?.settingsDidChangeEditor()
    })
    stack.addArrangedSubview(checkbox("Use text replacement",
                                      key: "NSAutomaticTextReplacementEnabled") { [weak self] in
        self?.settingsDelegate?.settingsDidChangeEditor()
    })

    item.view = stack
    return item
}

@objc private func checkboxToggled(_ sender: NSButton) {
    guard let key = sender.identifier?.rawValue else { return }
    UserDefaults.standard.set(sender.state == .on, forKey: key)
    if let cb = objc_getAssociatedObject(sender, &CheckboxCallback.key) as? () -> Void {
        cb()
    }
}

private struct CheckboxCallback { static var key = 0 }
```

## Applying changes to live WKWebViews

The hard truth: most of these settings are read by WKWebView at *creation* time. Changing them post-hoc requires either rebuilding the web view (heavy — loses scroll position and JS state) or injecting JS to enforce the new behavior on every editable element.

The right pattern: persist to UserDefaults, then notify open web views via the delegate, which loops through tabs and runs JS:

```swift
extension AppDelegate: SettingsDelegate {
    func settingsDidChangeEditor() {
        let autocorrectOff = !UserDefaults.standard.bool(forKey: "NSAutomaticSpellingCorrectionEnabled")
        let js: String
        if autocorrectOff {
            // Forcibly disable autocorrect/autocapitalize on every editable on focus.
            js = """
            (function() {
                document.addEventListener('focusin', (e) => {
                    const t = e.target;
                    if (!t) return;
                    if (t.isContentEditable || ['INPUT','TEXTAREA'].includes(t.tagName)) {
                        t.setAttribute('autocorrect','off');
                        t.setAttribute('autocomplete','off');
                        t.setAttribute('autocapitalize','off');
                        // Only touch autocorrect/autocomplete/autocapitalize here —
                        // leave t.spellcheck alone. Autocorrect (inline rewriting) and
                        // the browser's spell-check underline are independent features;
                        // disabling the underline because the user turned off autocorrect
                        // is a classic bug.
                    }
                }, true);
            })();
            """
        } else {
            js = """
            (function() {
                // Re-enable defaults.
                document.querySelectorAll('[autocorrect="off"]').forEach(el => el.removeAttribute('autocorrect'));
            })();
            """
        }
        for entry in allWindows {
            for url in entry.tabManager.allURLs {
                entry.tabManager.webView(for: url)?.evaluateJavaScript(js, completionHandler: nil)
            }
        }
    }
}
```

For toggles that *can't* be applied live (smart quotes substitution, for example, happens deep in the WebKit text engine), persist the new value and apply it on next tab creation. Tell the user that "Smart quotes" requires a relaunch via the checkbox subtitle.

## Start page — three modes, not just a URL field

The single most important setting, and it deserves more than a bare text field. Offer a **dropdown with three modes** (users of tabbed wrappers eventually want all three, so scaffold them from day one):

1. **Open Last Viewed Page** — at launch, restore the single page that was active when the app last quit.
2. **Open Last Viewed Tabs** — restore *every* window and tab from quit time, including window frames and which tab was active. See `references/sessions.md` → *the last-viewed-tabs snapshot* for the persistence half.
3. **Custom Start Page** — a fixed URL the user types, **host-restricted** to the wrapped site.

Model the mode as a string-backed enum, not a bool — and migrate the old bool so nobody's setting silently resets on upgrade:

```swift
enum StartPageMode: String {
    case lastViewedPage, lastViewedTabs, custom

    static var current: StartPageMode {
        let d = UserDefaults.standard
        if let raw = d.string(forKey: "appStartPageMode"),
           let mode = StartPageMode(rawValue: raw) { return mode }
        // Legacy bool from the two-mode era.
        return d.bool(forKey: "appRestoreLastViewedPage") ? .lastViewedPage : .custom
    }
}
```

```swift
let modePopup = NSPopUpButton()
modePopup.addItems(withTitles: ["Open Last Viewed Page", "Open Last Viewed Tabs", "Custom Start Page"])
modePopup.target = self
modePopup.action = #selector(startPageModeChanged(_:))
switch StartPageMode.current {
case .lastViewedPage: modePopup.selectItem(at: 0)
case .lastViewedTabs: modePopup.selectItem(at: 1)
case .custom:         modePopup.selectItem(at: 2)
}

// The URL field only EXISTS in Custom Start Page mode — don't just disable it,
// omit it, so the card collapses with no dead space (both last-viewed modes collapse).
if StartPageMode.current == .custom {
    // …add the "Start Page URL" label + text field here…
}

@objc private func startPageModeChanged(_ sender: NSPopUpButton) {
    let mode: StartPageMode = [.lastViewedPage, .lastViewedTabs, .custom][min(sender.indexOfSelectedItem, 2)]
    UserDefaults.standard.set(mode.rawValue, forKey: "appStartPageMode")
    // Keep the legacy bool coherent for anything still reading it.
    UserDefaults.standard.set(mode == .lastViewedPage, forKey: "appRestoreLastViewedPage")
    // Rebuild the tab so the card collapses/expands around the URL field.
    if let container = generalTabContainer {
        container.subviews.forEach { $0.removeFromSuperview() }
        buildGeneralTab(in: container)
    }
}
```

Give each mode its own hint line under the card ("Reopens the page you were viewing…", "Reopens every window and tab…", "Press Return to save — <host> URLs only").

Three things that make this feel finished:

- **Collapse, don't disable.** When "Open Last Viewed Page" is selected, *remove* the URL field and rebuild the card so it shrinks to just the dropdown — a greyed-out field leaves an ugly dead rectangle.
- **Restrict the custom URL to the wrapped host.** A start page pointing at `google.com` would just bounce straight back out through the navigation allowlist. Validate on save; beep and show an inline error otherwise. The wrapper apex(es) are the same ones in your nav allowlist:

  ```swift
  private func isAllowedStartPageURL(_ s: String) -> Bool {
      guard let url = URL(string: s),
            let scheme = url.scheme?.lowercased(), scheme == "https" || scheme == "http",
            let host = url.host?.lowercased() else { return false }
      return host == "notion.so"  || host.hasSuffix(".notion.so")
          || host == "notion.com" || host.hasSuffix(".notion.com")
  }
  ```

- **Wire "last viewed" on the AppDelegate side.** Persist the active tab's URL to `appLastViewedPageURL` on quit (and on navigation), then compute the first-tab URL at launch:

  ```swift
  private var initialPageURL: String {
      if UserDefaults.standard.bool(forKey: "appRestoreLastViewedPage"),
         let last = UserDefaults.standard.string(forKey: "appLastViewedPageURL"), !last.isEmpty {
          return last
      }
      return startPageURL   // the Custom Start Page value, or the built-in default
  }
  ```

The delegate reaction to a *custom-URL* change is the usual "don't reload existing tabs, just use the new URL for the next launch / Cmd+T." Note the start-page modes are distinct from **named sessions** (`references/sessions.md`): named sessions are user-managed workspaces; "Open Last Viewed Tabs" is invisible bookkeeping that restores the quit-time state automatically, and "last viewed page" is the single-tab version of the same convenience.

Also make the settings window **wide enough that card borders clear the scroll bar** — a card whose right edge tucks under the scroller looks like a rendering bug. Bump the content width a few points rather than insetting every card.

## Inline help: info popovers, not tooltips

The default `NSView.toolTip` has a ~1.5s hover delay that feels broken for help text, and it can't be clicked to pin. Replace every help tooltip (and inline "hint" lines under controls) with a small **ⓘ icon button** that opens an `NSPopover` *instantly* on click and after a brief (~0.2s) hover. A hover-opened popover closes on mouse-out; a click-opened one pins until the next click. This is the right pattern any time a setting needs a sentence of explanation:

```swift
final class InfoPopoverButton: NSButton, NSPopoverDelegate {
    private let popover = NSPopover()
    private let textLabel = NSTextField(wrappingLabelWithString: "")
    private var hoverTimer: Timer?
    private var pinnedByClick = false

    // commonInit: isBordered = false, imageOnly, target/action = handleClick,
    // popover.behavior = .transient, popover.delegate = self, label in a VC view.

    func configure(symbolName: String, tint: NSColor, help: String) {
        image = NSImage(systemSymbolName: symbolName, accessibilityDescription: "More info")
        contentTintColor = tint
        textLabel.stringValue = help
        // size the popover to the measured wrapped-text height (+a few px slack)
    }

    override func mouseEntered(with e: NSEvent) {
        hoverTimer = Timer.scheduledTimer(withTimeInterval: 0.2, repeats: false) { [weak self] _ in
            self?.showPopover()
        }
    }
    override func mouseExited(with e: NSEvent) {
        hoverTimer?.invalidate(); hoverTimer = nil
        if !pinnedByClick { popover.performClose(nil) }   // hover-opened → close on exit
    }
    @objc private func handleClick() {
        hoverTimer?.invalidate()
        if popover.isShown { pinnedByClick ? popover.performClose(nil) : (pinnedByClick = true) }
        else { pinnedByClick = true; showPopover() }       // click → pin open
    }
    func popoverDidClose(_ n: Notification) { pinnedByClick = false }
}
```

Use a tinted SF Symbol (`info.circle`) so it reads as help, not a control. Add an `NSTrackingArea` in `updateTrackingAreas()` for the hover events. Give the popover `appearance = NSAppearance(named: .darkAqua)` if your settings window is dark.

## Opening the Settings window

Wire to Cmd+, on the app menu:

```swift
appMenu.addItem(withTitle: "Settings…", action: #selector(openSettings), keyEquivalent: ",")

@objc func openSettings() {
    if settingsController == nil {
        settingsController = SettingsWindowController()
        settingsController?.settingsDelegate = self
    }
    settingsController?.showWindow(nil)
    settingsController?.window?.makeKeyAndOrderFront(nil)
}
```

`Cmd+,` is the macOS standard. Don't use anything else.

## App icon picker (optional but loved)

Lots of users like swapping the dock icon. To support this:

1. Bundle a few `.icns` variants in `Resources/Icons/`.
2. In Settings, show a grid of icon options.
3. On selection, overwrite `<App>.app/Contents/Resources/AppIcon.icns` with the chosen file, then `touch` the bundle to invalidate macOS's icon cache.

```swift
@objc private func selectIcon(_ sender: NSButton) {
    guard let bundleIconsDir = Bundle.main.resourceURL?.appendingPathComponent("Icons"),
          let chosenName = sender.identifier?.rawValue else { return }
    let source = bundleIconsDir.appendingPathComponent("\(chosenName).icns")
    let dest = Bundle.main.resourceURL!.appendingPathComponent("AppIcon.icns")
    try? FileManager.default.removeItem(at: dest)
    try? FileManager.default.copyItem(at: source, to: dest)

    // Invalidate icon cache.
    let task = Process()
    task.launchPath = "/usr/bin/touch"
    task.arguments = [Bundle.main.bundlePath]
    try? task.run()
    task.waitUntilExit()

    // Visually nudge macOS to refresh the dock tile.
    NSApp.applicationIconImage = NSImage(contentsOf: dest)
}
```

This is a slightly cheeky pattern — overwriting bundle resources at runtime — but macOS doesn't block it for ad-hoc-signed apps, and users love being able to pick an icon.

## Anti-patterns

- **Don't store complex types in UserDefaults via `Codable` encoded as Data.** Quick to write, hard to debug. Use the filesystem (Application Support).
- **Don't reload the entire web view when a toggle changes.** Inject JS that re-applies the setting on focus, or queue the change for next tab creation. Reloading destroys scroll position, JS state, in-flight uploads, etc.
- **Don't put a "Quit on close" toggle.** macOS apps don't quit on close. Users who want this preference can use `Cmd+Q`.
- **Don't show "Apply" buttons.** macOS settings save immediately. The user expects toggles to take effect now.
- **Don't add a search field unless the settings list is >30 items.** Smaller than that and it's noise.
