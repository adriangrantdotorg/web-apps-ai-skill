# WKWebView Configuration, Navigation Policy, Permissions

This file covers the per-tab `WebViewController` in depth: `WKWebViewConfiguration` setup, custom user agent, zoom persistence, the navigation allowlist (with the auth-vs-content carve-out) that keeps external links opening in the user's default browser, `WKUIDelegate` for OAuth and `window.open` (returning a real `WKWebView` so sign-in popups actually complete), the four WKUIDelegate panel methods (file upload via `NSOpenPanel` + alert/confirm/prompt — all silent no-ops if unimplemented), recovery from web-content-process termination (the "app went blank after sleep/long idle" bug), camera/mic permissions, SPA-aware navigation interception, and the JS injection plumbing the other features rely on.

If you only read one reference file, read this one — it's where 80% of the "feels like a real app" comes from.

## Complete `WebViewController` shape

```swift
import AppKit
import WebKit

final class WebViewController: NSViewController, WKNavigationDelegate, WKUIDelegate, WKScriptMessageHandler {
    let id = UUID()
    private(set) var webView: WKWebView!
    private var titleObservation: NSKeyValueObservation?
    private var urlObservation: NSKeyValueObservation?
    private var loadingObservation: NSKeyValueObservation?

    var onTitleChanged: ((String) -> Void)?
    var onURLChanged: ((URL?) -> Void)?
    var onOpenInNewTab: ((URL, _ background: Bool) -> Void)?
    var onLoadFinished: (() -> Void)?
    // Returns a REAL new WKWebView (a fresh foreground tab) for window.open popups
    // — the host builds the tab and hands back its webView. See "OAuth popups".
    var onCreatePopupTab: ((WKWebViewConfiguration) -> WKWebView?)?
    // The popup called window.close() — host closes the tab hosting this webView.
    var onClosePopupTab: ((WKWebView) -> Void)?

    // Recovery state for content-process termination (see that section below).
    private var lastLoadedURL: URL?
    private(set) var contentProcessDidTerminate = false

    // Configurable per app:
    private let primaryDomains: [String] = ["<site>.com"]       // Always allow in-app.
    private let authDomains: [String] = [
        "accounts.google.com", "appleid.apple.com", "github.com",
        "login.microsoftonline.com", "challenges.cloudflare.com",
    ]
    // Hosts that match an authDomains wildcard (e.g. *.google.com is kept allowed
    // for SSO) but are actually CONTENT the user expects in their real browser.
    // Checked BEFORE the allowlist on user clicks / popups. Tune per wrapped site.
    private let contentHosts: [String] = [
        "docs.google.com", "drive.google.com", "sheets.google.com",
        "slides.google.com", "forms.google.com", "sites.google.com",
        "calendar.google.com", "meet.google.com", "mail.google.com",
        "keep.google.com", "photos.google.com",
        // Google AI products are content too (otherwise clicked Gemini
        // chat links get hijacked into an in-app tab).
        "gemini.google.com", "notebooklm.google.com", "aistudio.google.com",
    ]
    private let startPageURL: URL

    init(startPageURL: URL) {
        self.startPageURL = startPageURL
        super.init(nibName: nil, bundle: nil)
    }
    required init?(coder: NSCoder) { fatalError() }

    override func loadView() {
        let config = WKWebViewConfiguration()
        // NOTE: WKProcessPool is deprecated as of macOS 12 — cookies/sessions are
        // auto-shared via the modern WKWebsiteDataStore.default() and the property
        // "no longer has any effect." New apps should omit it entirely; existing
        // apps that still set it can remove the line to silence the warning.
        config.websiteDataStore = .default()
        config.preferences.javaScriptCanOpenWindowsAutomatically = true
        config.defaultWebpagePreferences.allowsContentJavaScript = true
        config.mediaTypesRequiringUserActionForPlayback = []
        config.allowsAirPlayForMediaPlayback = true

        let contentController = WKUserContentController()
        config.userContentController = contentController
        contentController.add(self, name: "openInNewTab")
        contentController.add(self, name: "openInBrowser")

        // Inject the SPA new-tab interceptor at document-start.
        contentController.addUserScript(WKUserScript(
            source: Self.middleClickInterceptorJS,
            injectionTime: .atDocumentStart,
            forMainFrameOnly: false
        ))

        // Default in every wrapper: kill the macOS autocorrect/autocapitalize suggestion pill in
        // web text fields. Body in references/settings.md → "Disable the macOS text-suggestion pill".
        contentController.addUserScript(WKUserScript(
            source: Self.inputHardeningJS,
            injectionTime: .atDocumentStart,
            forMainFrameOnly: false
        ))

        webView = WKWebView(frame: .zero, configuration: config)
        webView.customUserAgent = Self.desktopSafariUA
        webView.allowsBackForwardNavigationGestures = true
        webView.uiDelegate = self
        webView.navigationDelegate = self
        if #available(macOS 13.3, *) {
            webView.isInspectable = true
        }

        observeKVO()
        restoreZoom()

        self.view = webView
    }

    func load(url: URL) { lastLoadedURL = url; webView.load(URLRequest(url: url)) }

    // Return the MATCHED entry (not a Bool) so call sites can apply the
    // same-site guard: a tab already ON that product keeps its own links
    // in-app. Bounce only when the click's matched host differs from the
    // current page's matched host:
    //   if let target = contentHost(matching: url.host),
    //      contentHost(matching: webView.url?.host) != target { openExternally }
    private func contentHost(matching host: String?) -> String? {
        guard let host else { return nil }
        return contentHosts.first(where: { host == $0 || host.hasSuffix("." + $0) })
    }

    private func observeKVO() {
        titleObservation = webView.observe(\.title, options: [.new]) { [weak self] _, change in
            if let title = change.newValue ?? nil { self?.onTitleChanged?(title) }
        }
        urlObservation = webView.observe(\.url, options: [.new]) { [weak self] _, change in
            self?.onURLChanged?(change.newValue ?? nil)
            self?.restoreZoom()  // SPA navigations don't fire didFinish.
        }
    }

    // MARK: - WKNavigationDelegate

    func webView(_ webView: WKWebView,
                 decidePolicyFor navigationAction: WKNavigationAction,
                 decisionHandler: @escaping (WKNavigationActionPolicy) -> Void) {
        guard let url = navigationAction.request.url else { decisionHandler(.cancel); return }

        // Always allow subframes (iframes, embedded media, OAuth iframes).
        if navigationAction.targetFrame?.isMainFrame != true {
            decisionHandler(.allow); return
        }

        let host = url.host?.lowercased() ?? ""

        // URL-redirector services (Google, LinkedIn, Slack, etc.) wrap external links
        // in their own host. The redirector page itself is allowlisted (it's on the
        // wrapped site's domain), then JS-redirects to the real target. Pre-intercept
        // the wrapper here so the user never sees the "Redirecting you to…"
        // interstitial — extract the `q=` (or equivalent) and route to the system
        // browser directly when the target host isn't allowed.
        if (host == "www.google.com" || host == "google.com"), url.path == "/url",
           let comps = URLComponents(url: url, resolvingAgainstBaseURL: false),
           let target = comps.queryItems?.first(where: { $0.name == "q" })?.value,
           let targetURL = URL(string: target),
           let scheme = targetURL.scheme?.lowercased(), ["http", "https"].contains(scheme),
           !isAllowed(host: targetURL.host?.lowercased() ?? "") {
            NSWorkspace.shared.open(targetURL)
            decisionHandler(.cancel)
            return
        }

        // Auth-allowlisted CONTENT (e.g. a clicked Google Doc under *.google.com,
        // which is allowed for SSO) → default browser, not an in-app tab. Gate on
        // .linkActivated so programmatic OAuth redirects (navigationType == .other)
        // are unaffected. Checked BEFORE the allowlist because the host matches it.
        // Same-site guard: a tab already ON that product keeps its own links in-app.
        if navigationAction.navigationType == .linkActivated,
           let target = contentHost(matching: url.host),
           contentHost(matching: webView.url?.host) != target {
            NSWorkspace.shared.open(url)
            decisionHandler(.cancel)
            return
        }

        if isAllowed(host: host) {
            decisionHandler(.allow); return
        }

        // Main-frame navigation to an outside host — open in default browser.
        // This catches both user link clicks AND programmatic redirects (JS /
        // meta-refresh). Earlier versions of this skill only opened externally
        // for .linkActivated and silently cancelled everything else, which strands
        // the user on URL-redirector pages whenever the q= extraction above misses
        // (different redirector format, different host, etc.). Cancelling without
        // opening externally is almost always wrong — if it's a real attack vector
        // the redirect would still happen in the user's browser anyway.
        if let scheme = url.scheme?.lowercased(), ["http", "https"].contains(scheme) {
            NSWorkspace.shared.open(url)
        }
        decisionHandler(.cancel)
    }

    func webView(_ webView: WKWebView, didCommit navigation: WKNavigation!) {
        // A navigation committed — the content process is alive and serving this
        // page. Remember the URL (for termination recovery) and clear the dead flag.
        if let url = webView.url { lastLoadedURL = url }
        contentProcessDidTerminate = false
    }

    func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
        onLoadFinished?()
        restoreZoom()
        // Re-assert the combined theme CSS after load (SPA renders can strip the style tag).
        if ThemeManager.shared.activeCSS() != nil { injectActiveTheme() }
    }

    // MARK: - Content-process termination recovery

    // The web content process backing this tab was terminated by the system
    // (memory pressure / GPU recycling after long use, or jettisoned during
    // sleep). WebKit blanks the view to its background color and does NOT reload
    // on its own — recover here. If the view is off-screen the reload may no-op,
    // so recoverIfTerminated() (called on wake / activate / heartbeat) retries.
    func webViewWebContentProcessDidTerminate(_ webView: WKWebView) {
        contentProcessDidTerminate = true
        reloadAfterTermination()
    }

    private func reloadAfterTermination() {
        // Always a fresh load, never reload(): after a process kill the
        // back-forward list may have died with the process, and reload() then
        // silently no-ops — the flag stays set and every sweep retries the
        // same dead end while the tab stays blank.
        if let url = lastLoadedURL ?? webView.url {
            webView.load(URLRequest(url: url))
        } else {
            webView.reload()
        }
    }

    // Called by the host on system/display wake, app reactivation, and the
    // ~30s heartbeat. No-ops on healthy tabs, so live pages are never
    // reloaded out from under the user.
    func recoverIfTerminated() {
        if contentProcessDidTerminate {
            reloadAfterTermination()
            return
        }
        // Wedged-load watchdog: a provisional load stuck longer than any real
        // page load (process suspended mid-load during sleep / hidden window)
        // will neither commit nor fail on its own. provisionalLoadStartedAt is
        // set in didStartProvisionalNavigation, cleared on commit and failure.
        if let started = provisionalLoadStartedAt,
           -started.timeIntervalSinceNow > 30,
           let url = lastLoadedURL ?? webView.url {
            provisionalLoadStartedAt = nil
            webView.stopLoading()
            webView.load(URLRequest(url: url))
        }
    }

    // MARK: - WKUIDelegate (window.open, OAuth popups, target=_blank)

    func webView(_ webView: WKWebView,
                 createWebViewWith configuration: WKWebViewConfiguration,
                 for navigationAction: WKNavigationAction,
                 windowFeatures: WKWindowFeatures) -> WKWebView? {
        // Only handle requests for a NEW window (target="_blank" / window.open).
        guard navigationAction.targetFrame?.isMainFrame != true else { return nil }

        let url = navigationAction.request.url
        let host = url?.host?.lowercased() ?? ""

        // Auth-allowlisted CONTENT opened via window.open (e.g. a Google Doc under
        // *.google.com) → default browser, not an in-app tab. Checked first.
        // Same-site guard here too: that product's own tab may window.open itself.
        if let url, let target = contentHost(matching: host),
           contentHost(matching: webView.url?.host) != target {
            NSWorkspace.shared.open(url)
            return nil
        }

        // A popup to a host outside both allowlists → default browser. (A nil/blank
        // URL is an in-app popup shell and falls through to the real-webView path.)
        if let url, !host.isEmpty, !isAllowed(host: host) {
            NSWorkspace.shared.open(url)
            return nil
        }

        // Everything else is a genuine popup: the wrapped site's own popups, blank
        // popup shells, and OAuth provider windows (Google/Apple "Continue with…").
        // Return a REAL, freshly-created WKWebView so window.open() yields a valid
        // window reference and the popup's window.opener points back at this tab —
        // BOTH are required for the OAuth callback to postMessage its result home
        // before calling window.close() (handled in webViewDidClose). Loading into
        // the same view, or returning nil, gives a black/blank sign-in window.
        return onCreatePopupTab?(configuration)
    }

    // The popup page called window.close() (e.g. the OAuth callback page after it
    // handed the result to its opener) — close the tab hosting this web view.
    func webViewDidClose(_ webView: WKWebView) {
        onClosePopupTab?(webView)
    }

    // Camera/mic.
    func webView(_ webView: WKWebView,
                 requestMediaCapturePermissionFor origin: WKSecurityOrigin,
                 initiatedByFrame frame: WKFrameInfo,
                 type: WKMediaCaptureType,
                 decisionHandler: @escaping (WKPermissionDecision) -> Void) {
        let host = origin.host.lowercased()
        if isAllowed(host: host) {
            decisionHandler(.grant)
        } else {
            decisionHandler(.prompt)
        }
    }

    // MARK: - WKUIDelegate panels (file upload + JS dialogs) — ALL FOUR, from day one
    // WKWebView has no default UI for any of these; an unimplemented method is a
    // silent no-op. Missing runOpenPanelWith makes every <input type="file"> click
    // dead ("Choose file does nothing").

    func webView(_ webView: WKWebView,
                 runOpenPanelWith parameters: WKOpenPanelParameters,
                 initiatedByFrame frame: WKFrameInfo,
                 completionHandler: @escaping ([URL]?) -> Void) {
        let panel = NSOpenPanel()
        panel.allowsMultipleSelection = parameters.allowsMultipleSelection
        panel.canChooseDirectories = parameters.allowsDirectories
        panel.canChooseFiles = true
        if let window = webView.window {
            panel.beginSheetModal(for: window) { completionHandler($0 == .OK ? panel.urls : nil) }
        } else {
            completionHandler(panel.runModal() == .OK ? panel.urls : nil)
        }
    }

    func webView(_ webView: WKWebView, runJavaScriptAlertPanelWithMessage message: String,
                 initiatedByFrame frame: WKFrameInfo, completionHandler: @escaping () -> Void) {
        let alert = NSAlert(); alert.messageText = message; alert.addButton(withTitle: "OK")
        if let window = webView.window {
            alert.beginSheetModal(for: window) { _ in completionHandler() }
        } else { alert.runModal(); completionHandler() }
    }

    func webView(_ webView: WKWebView, runJavaScriptConfirmPanelWithMessage message: String,
                 initiatedByFrame frame: WKFrameInfo, completionHandler: @escaping (Bool) -> Void) {
        let alert = NSAlert(); alert.messageText = message
        alert.addButton(withTitle: "OK"); alert.addButton(withTitle: "Cancel")
        if let window = webView.window {
            alert.beginSheetModal(for: window) { completionHandler($0 == .alertFirstButtonReturn) }
        } else { completionHandler(alert.runModal() == .alertFirstButtonReturn) }
    }

    func webView(_ webView: WKWebView, runJavaScriptTextInputPanelWithPrompt prompt: String,
                 defaultText: String?, initiatedByFrame frame: WKFrameInfo,
                 completionHandler: @escaping (String?) -> Void) {
        let alert = NSAlert(); alert.messageText = prompt
        alert.addButton(withTitle: "OK"); alert.addButton(withTitle: "Cancel")
        let field = NSTextField(frame: NSRect(x: 0, y: 0, width: 260, height: 24))
        field.stringValue = defaultText ?? ""
        alert.accessoryView = field
        if let window = webView.window {
            alert.beginSheetModal(for: window) { completionHandler($0 == .alertFirstButtonReturn ? field.stringValue : nil) }
        } else { completionHandler(alert.runModal() == .alertFirstButtonReturn ? field.stringValue : nil) }
    }

    // MARK: - WKScriptMessageHandler

    func userContentController(_ uc: WKUserContentController, didReceive message: WKScriptMessage) {
        switch message.name {
        case "openInNewTab":
            if let dict = message.body as? [String: Any], let s = dict["url"] as? String, let url = URL(string: s) {
                onOpenInNewTab?(url, false)
            }
        case "openInBrowser":
            if let s = message.body as? String, let url = URL(string: s) {
                NSWorkspace.shared.open(url)
            }
        default: break
        }
    }

    // MARK: - Allowlist

    private func isPrimary(host: String) -> Bool {
        primaryDomains.contains(where: { host == $0 || host.hasSuffix("." + $0) })
    }
    private func isAllowed(host: String) -> Bool {
        isPrimary(host: host) || authDomains.contains(where: { host == $0 || host.hasSuffix("." + $0) })
    }
}

```

Cookie/session sharing across tabs and windows is now handled by `WKWebsiteDataStore.default()`, which is process-wide automatically. The old `SharedWebKit.processPool = WKProcessPool()` trick (a singleton process pool wired into every WKWebView's config) is no longer needed on macOS 12+ — `WKProcessPool` has been formally deprecated and "no longer has any effect" per Apple's docs. If you find this pattern in older code built from this kit, delete the enum and the `config.processPool = …` line; nothing else needs to change.

## JavaScript clipboard access

WKWebView **disables page-JavaScript clipboard access by default.** So the moment a wrapped site offers its own "Copy" affordance — Notion's copy-link and code-block buttons, GitHub's "copy file contents", a docs "copy as Markdown", a "copy share link" button — the underlying `document.execCommand('copy')` / `navigator.clipboard.writeText()` call is **silently denied**. The visible failure mode varies by site: some buttons just do nothing; Notion falls back to a manual "right-click and copy" prompt. Users report "the copy button is broken."

The fix is two private-but-stable WebKit preference keys, set on the `WKWebViewConfiguration` *before* the web view is created (same place and same idiom as `developerExtrasEnabled`):

```swift
// Let page JavaScript read/write the system clipboard. WKWebView disables this
// by default, so the site's in-page "Copy" buttons silently fail until enabled.
config.preferences.setValue(true, forKey: "javaScriptCanAccessClipboard")
config.preferences.setValue(true, forKey: "DOMPasteAllowed")
```

- `javaScriptCanAccessClipboard` covers the *write* path (`execCommand('copy')`, `navigator.clipboard.writeText`).
- `DOMPasteAllowed` covers the *read/paste* path so programmatic paste handlers also work.
- These are read at web-view-creation time, like the other `preferences` keys — set them once on the shared config, not per-navigation.
- This is orthogonal to macOS clipboard *entitlements*; you do not need an entitlement for a desktop app's own web view to use the clipboard once these are on.

## Custom user agent

```swift
static let desktopSafariUA = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Safari/605.1.15"
```

Why: WKWebView's default UA identifies as a non-Safari WebKit build, and many sites either degrade or refuse based on the UA string. Forcing the Safari UA gets the desktop-class experience.

Update the version periodically — check Apple's `Safari/Settings/Advanced/Develop` menu for the current production UA.

## Zoom persistence

WKWebView's `pageZoom` is per-view. Persisting it globally:

```swift
private static let zoomDefaultsKey = "webViewZoom"

func restoreZoom() {
    let zoom = UserDefaults.standard.double(forKey: Self.zoomDefaultsKey)
    if zoom > 0 { webView.pageZoom = zoom } else { webView.pageZoom = 1.0 }
}

@objc func zoomIn()   { setZoom(webView.pageZoom + 0.1) }
@objc func zoomOut()  { setZoom(webView.pageZoom - 0.1) }
@objc func zoomReset(){ setZoom(1.0) }

private func setZoom(_ z: CGFloat) {
    let clamped = max(0.5, min(3.0, z))
    webView.pageZoom = clamped
    UserDefaults.standard.set(clamped, forKey: Self.zoomDefaultsKey)
}
```

Wire to View menu with `Cmd+=`, `Cmd+-`, `Cmd+0`. Apply on every `didFinish` so SPA navigations don't reset. Apply the SAME level to every `WKWebView` the app owns — the Settings window included, never a per-view exception (SKILL.md item 37 clause 8). These three View-menu items are a standing default in EVERY Mac app, not only wrappers — full rule (order, level row, rebindable combos, no toast/notification on zoom): SKILL.md item 37.

## Navigation allowlist — design notes

Two layers of allowlist:

1. **`primaryDomains`** — the wrapped site itself and its CDNs. Always allowed in-app. These should be apex domains plus subdomain wildcards (`hasSuffix`-style).
2. **`authDomains`** — third-party identity providers and CAPTCHA. Allowed in-app *during navigation*, so OAuth callbacks complete and Cloudflare challenges render. Don't include analytics/tracking domains — they only run via subframes and are covered by the "always allow subframes" rule.
3. **`contentHosts`** — a *carve-out* from the allowlist, not a third allow layer. **Auth allowlist ≠ content allowlist.** You allowlist `google.com` so SSO works, but that wildcard also captures `docs.google.com`, `drive.google.com`, `calendar.google.com`, `mail.google.com`, … — *content* the user expects to open in their real browser. A clicked Google Doc inside a Notion/Linear wrapper getting hijacked into an in-app tab is the classic symptom. **Include Google's AI products too**: `gemini.google.com`, `notebooklm.google.com`, `aistudio.google.com` — a clicked Gemini chat link is content exactly like a Doc. List these subdomains explicitly and check them **before** the allowlist, in *both* `decidePolicyFor` and `createWebViewWith`. Gate the `decidePolicyFor` check on `navigationType == .linkActivated` so programmatic OAuth redirects (which are `.other`) still complete in-app.
4. **Same-site guard on the `contentHosts` carve-out.** Multi-tab wrappers may host one of those products as a tab of its own (e.g. a "Google Gemini" tab alongside Notion). A blanket carve-out then bounces that tab's *own* links to the external browser. Have the matcher return the matched entry (not a Bool) and bounce only when the clicked URL's matched host differs from the current page's: `contentHost(matching: url.host) != contentHost(matching: webView.url?.host)`. In-product clicks stay in-app; cross-site clicks (a Gemini link inside a Notion page) go external.

What gets opened in the default browser:

- Any main-frame navigation to a host outside both lists — **including programmatic redirects** (JS / meta-refresh, not just `.linkActivated`). The "silently cancel programmatic non-allowed nav" pattern from earlier versions of this skill is wrong: it strands the user on URL-redirector pages (Google's `google.com/url?q=…`, LinkedIn's `lnkd.in`, Slack's `slack-redir.net`, etc.). The redirector itself is on an allowlisted host, so it loads; then it JS-redirects to the target on a non-allowlisted host with `navigationType == .other` — and the cancel-without-opening path leaves the user staring at "Redirecting you to…". Always open externally for disallowed http(s) main-frame navigation.
- A user click (`.linkActivated`) on a `contentHosts` URL — even though its host is allowlisted for auth.
- `window.open` / `target="_blank"` to a non-primary, non-auth host (or any `contentHosts` URL).

URL redirectors — extra credit:

- Pre-intercept known redirectors before the page loads at all. Google wraps external links in `https://www.google.com/url?q=<target>`; parse the `q=` parameter, and if the target host isn't allowlisted, `NSWorkspace.open` the target directly and cancel — the user never sees the redirector page flash. The fallback (open-externally-on-any-disallowed-nav) covers the case where the redirector format isn't one you recognized.

What's always allowed:

- Subframes (iframes). Critical for embeds, OAuth iframes, analytics.

Diagnosing "the link opened in the wrong app":

- `NSWorkspace.shared.open(url)` hands the URL to the LaunchServices default https handler — which may be a link-router app rather than a browser. That chain is the user's choice; don't "fix" it by hardcoding a browser.
- Before blaming the router or LaunchServices, **reproduce the exact call in a scratch swiftc binary** and print `NSWorkspace.frontmostApplication` after a few seconds. If the repro opens the right app, the misroute is in the *source* app's own navigation handling (usually this allowlist hijack), not in the open call.
- Universal-link hijack is a separate class: a plain `open(url)` can route to a native app claiming the domain. Bypass while still honoring the user's chosen handler chain: resolve `urlForApplication(toOpen: URL(string: "https://")!)` and open with `open(_:withApplicationAt:configuration:)` — web schemes only; `mailto:` etc. keep their own handlers.

## OAuth popups — the tricky case

OAuth flows typically follow this dance:

1. User clicks "Continue with Google/Apple" in the wrapped site.
2. Site calls `const w = window.open('https://accounts.google.com/…', '_blank')` and keeps the returned handle.
3. The provider handles auth and redirects the popup to a callback URL.
4. The callback page reads `window.opener`, `postMessage`s the result back to the opener (the main tab), then calls `window.close()`.

Both `window.open()`'s **return value** and the popup's **`window.opener`** must be live for steps 2–4 to work. That only happens if your `createWebViewWith` returns a **real, freshly-created `WKWebView`**. This is the single biggest gotcha and earlier versions of this skill got it wrong.

What does NOT work (and how it fails):

- **Loading the request into the *same* web view and returning `nil`** (the old advice). `window.open()` returns `null`, so the site has no popup handle and `window.opener` is never set. Result: the **"Continue with Google" window is black/blank** and sign-in never completes. Do not do this.
- **Returning `nil` outright.** The popup simply never opens.
- **Creating a tab but not returning its `WKWebView`.** Same broken-opener problem as same-view.

The right move:

- For genuine popups — OAuth provider windows (`authDomains`), the wrapped site's own popups (`primaryDomains`), and blank/`about:blank` popup shells — build a real new foreground tab and **return its `webView`** (`onCreatePopupTab` in the shape above). The opener relationship is wired up automatically by WebKit because you returned the view it asked for.
- Implement **`webViewDidClose`** so when the callback page calls `window.close()`, you close the tab hosting that web view and return the user to the (now signed-in) opener tab. Without this the spent OAuth tab lingers as a dead blank pane.
- For popup hosts outside both allowlists (and auth-allowlisted *content* hosts like Google Docs), open in the default browser and return `nil`.

Edge case: if the popup is somehow the *only* tab when it tries to close itself, don't honor `window.close()` blindly (you'd leave an empty window) — point that tab back at the start page instead.

## Cross-domain SSO cookies (Intelligent Tracking Prevention)

**Symptom:** a site that signs in against a *different* domain than it runs on hangs forever on its loading splash, or the sign-in bounces and never completes. One real example: the app runs on `app.clockify.me` but its SSO is brokered by `cake.com`; WebKit's Intelligent Tracking Prevention (ITP) classifies the auth domain's cookies as third-party tracking and partitions them away, so the SSO cookie the site looks for is never visible. Clockify surfaces this as `NO_SSO_COOKIE_FOUND (40009)` and never finishes loading. There's no error a user would understand — the app just looks broken on first launch.

**Cause:** ITP is on by default in WKWebView and is built for a general-purpose browser visiting many sites. In a single-site wrapper the "third-party" domain is the site's *own* identity provider, so the privacy heuristic is actively wrong here.

**Fix:** disable resource-load statistics (ITP) on the app's website data store at launch, before any web view loads. The toggle is a private `WKWebsiteDataStore` selector; guard it with `responds(to:)` so a future WebKit change degrades to "ITP stays on" rather than crashing.

```swift
/// Disable Intelligent Tracking Prevention so cross-domain SSO cookies are visible.
/// Without this, sites whose auth lives on a different domain (e.g. CAKE ↔ Clockify) can't
/// find their SSO cookie and hang on the loading splash.
static func disableTrackingPrevention() {
    let store = WKWebsiteDataStore.default()
    let sel = NSSelectorFromString("_setResourceLoadStatisticsEnabled:")
    guard store.responds(to: sel), let imp = store.method(for: sel) else { return }
    typealias Fn = @convention(c) (NSObject, Selector, ObjCBool) -> Void
    unsafeBitCast(imp, to: Fn.self)(store, sel, ObjCBool(false))
}
```

Call it once from `applicationDidFinishLaunching`, before opening the first window.

**Scope it honestly:** only reach for this when auth crosses domains. A single-domain site (login and app on the same registrable domain) never hits the partition and doesn't need the SPI. It's defensible *because* this is a dedicated wrapper for one service — don't carry the pattern into anything that browses arbitrary sites.

## Content-process termination recovery

**Symptom:** the app is left open for hours, or the Mac sleeps and wakes, and the window is now a **solid blank panel** — just the `NSWindow.backgroundColor`, no content. ⌘R fixes it, but the user shouldn't have to.

**Cause:** each `WKWebView` runs its page in a separate *web content process*. macOS terminates that process under two everyday conditions:

- **Memory pressure / GPU-process recycling** after long uptime (the "left it open overnight" case).
- **Sleep** — the OS jettisons background web content processes to reclaim memory; on wake the view is dead.

When the process dies WebKit does **not** reload — it just shows the host view's background color. There is exactly one callback for this, and the default implementation is a no-op, so wrappers that don't implement it appear to "blank out." This is not theme- or site-specific; every WKWebView wrapper has it until fixed.

**Fix — two coordinated halves.** The per-tab half lives in `WebViewController` (already shown in the shape above): `webViewWebContentProcessDidTerminate(_:)` sets a flag and recovers with a **fresh `load(URLRequest(url: lastCommittedURL))`** (tracked in `didCommit`), *never* a bare `reload()`. The reload() trap: after a process kill the back-forward list can die with the process, `reload()` then silently no-ops, the flag stays set, and every later sweep retries the same dead end forever — blank until app restart, despite "recovery" running constantly. The flag is cleared in `didCommit`.

The host half handles the timing problem: a load issued while the screen is still off (or the app is in the background) can wedge or no-op, so the termination callback alone isn't enough. Observe wake/activate **and run a slow heartbeat**, gated on the per-tab checks so healthy tabs are never disturbed:

```swift
// AppDelegate.applicationDidFinishLaunching(_:)
let ws = NSWorkspace.shared.notificationCenter
ws.addObserver(self, selector: #selector(recoverWebViews),
               name: NSWorkspace.didWakeNotification, object: nil)         // system sleep → wake
ws.addObserver(self, selector: #selector(recoverWebViews),
               name: NSWorkspace.screensDidWakeNotification, object: nil)  // display sleep → wake
NotificationCenter.default.addObserver(self, selector: #selector(recoverWebViews),
               name: NSApplication.didBecomeActiveNotification, object: nil) // refocus after long idle

// Heartbeat: wake/activate observers never fire when the app is ALREADY
// frontmost as a tab blanks — the user then stares at a black window with
// nothing to heal it (the "restart the app" case). ~30s, tolerance 5;
// zero work when everything is healthy (flag/timestamp checks only).
recoveryHeartbeat = Timer.scheduledTimer(withTimeInterval: 30, repeats: true) { [weak self] _ in
    self?.recoverWebViews()
}

@objc private func recoverWebViews() {
    for entry in allWindows {
        // Skip hidden windows (a closed-but-hiding main window): reloading
        // pages nobody can see fights the OS's memory reclaim. They heal on
        // reshow — reopening activates the app, which sweeps again.
        guard entry.window.isVisible || entry.window.isMiniaturized else { continue }
        entry.tabManager.recoverTerminatedWebViews()
    }
}
```

**Also add the wedged-load watchdog and a visible-tab retry** (both in the per-tab shape above): (a) track `provisionalLoadStartedAt` (`didStartProvisionalNavigation` sets it; commit/fail clear it) and have `recoverIfTerminated()` stop-and-restart any load stuck >30s — a content process suspended mid-load leaves `isLoading == true` forever with no callback, which no flag-based recovery ever catches; (b) in `didFailProvisionalNavigation` (non-cancelled), if the tab is user-visible and `webView.url == nil` (nothing ever committed — this gate keeps policy-cancels and healthy pages out), retry `lastLoadedURL` after a backoff (3s doubling to 60s, counter reset on commit) instead of just printing.

`TabManager` keeps its tab list private, so add the sweep there rather than exposing the controllers:

```swift
// TabManager
func recoverTerminatedWebViews() {
    for tab in tabs { tab.controller.recoverIfTerminated() }
}
```

Why both halves and why gated:

- The **callback** handles active-use termination (memory pressure) immediately — content was already gone, so the reload isn't disruptive.
- The **wake/activate retry** handles the sleep case, where the callback may fire while the view is off-screen and its reload no-ops. Retrying once visible recovers it.
- **Gating on the flag** (set in the callback, cleared on `didCommit`) means a normal app-switch or wake never reloads a healthy tab — no lost scroll position, no flash. Don't reload-on-wake unconditionally; that's the lazy version and users feel it.

One caveat worth stating to users: recovery is a reload, so an in-page edit not yet persisted by the site is lost. For autosaving apps (Notion, Linear, Google Docs) this is a non-issue in practice — the save interval is seconds and the crash would have to land in that exact gap.

## SPA new-tab interception

Single-page apps don't always trigger `decidePolicyFor`. They mutate `history` directly. To make middle-click and Cmd-click open in new tabs, you need JS at document-start that watches for the user's intent and intercepts the navigation:

```swift
static let middleClickInterceptorJS = """
(function() {
    let lastMiddleClick = 0;
    let lastCmdClick = 0;
    document.addEventListener('mousedown', (e) => {
        if (e.button === 1) lastMiddleClick = Date.now();
        if (e.button === 0 && e.metaKey) lastCmdClick = Date.now();
    }, true);

    // Catch links explicitly (cheap and reliable).
    document.addEventListener('click', (e) => {
        const a = e.target.closest && e.target.closest('a[href]');
        if (!a) return;
        const wantsNewTab = e.button === 1 || (e.button === 0 && (e.metaKey || a.target === '_blank'));
        if (!wantsNewTab) return;
        const href = a.href;
        if (!href || href.startsWith('javascript:')) return;
        e.preventDefault();
        e.stopPropagation();
        window.webkit.messageHandlers.openInNewTab.postMessage({ url: href });
    }, true);

    // SPA navigation interception: pushState within 100ms of a middle/cmd click goes to a new tab.
    const origPushState = history.pushState;
    history.pushState = function(state, title, url) {
        const recentClick = (Date.now() - Math.max(lastMiddleClick, lastCmdClick)) < 100;
        if (recentClick && url) {
            const abs = new URL(url, location.href).href;
            window.webkit.messageHandlers.openInNewTab.postMessage({ url: abs });
            return;
        }
        return origPushState.apply(this, arguments);
    };
})();
"""
```

The 100ms window is tight enough to avoid false positives (random pushStates from background work) but wide enough to catch the user's actual click-to-pushState gap.

## Camera/microphone permissions

Auto-grant for the primary domain; prompt for others. Granting unconditionally is a bad idea — if the wrapped site embeds a third-party meeting iframe, you don't want it to gain mic access silently.

```swift
func webView(_ webView: WKWebView,
             requestMediaCapturePermissionFor origin: WKSecurityOrigin,
             initiatedByFrame frame: WKFrameInfo,
             type: WKMediaCaptureType,
             decisionHandler: @escaping (WKPermissionDecision) -> Void) {
    let host = origin.host.lowercased()
    if isPrimary(host: host) {
        decisionHandler(.grant)
    } else {
        decisionHandler(.prompt)  // System dialog.
    }
}
```

Required Info.plist keys (won't prompt without these — silent failure):

```
NSCameraUsageDescription
NSMicrophoneUsageDescription
```

Required entitlements:

```
com.apple.security.device.camera
com.apple.security.device.microphone
```

### Meeting transcription hears only the user's side — mic access is not enough

Granting the mic makes recording *work*, but a wrapped meeting-transcription feature (Notion AI Meeting Notes, etc.) then transcribes **only the user's own voice**. The other participants come out of the Mac's speakers — that's *system audio*, and a web page has no API to hear it. The site's official Electron app captures system audio natively and mixes it in; a WKWebView wrapper must do the same or the feature ships half-broken. The mic/camera grant alone is easily mistaken for a full fix.

The working architecture (fits in one `SystemAudioCapture.swift`, ~450 lines):

1. **Native capture — Core Audio process tap (macOS 14.4+, gate with `@available`).** `CATapDescription(stereoGlobalTapButExcludeProcesses: [])` (set `isPrivate = true`, `muteBehavior = .unmuted`) → `AudioHardwareCreateProcessTap` → wrap in a **private aggregate device** (`kAudioAggregateDeviceIsPrivateKey: true`, default *system* output device as main sub-device + the tap under `kAudioAggregateDeviceTapListKey` with drift compensation, `kAudioAggregateDeviceTapAutoStartKey: true`) → `AudioDeviceCreateIOProcIDWithBlock` + `AudioDeviceStart`. No kexts, no virtual-device drivers, no Screen Recording permission.
2. **Chunk + ship into the page.** In the IOProc, downmix whatever arrives (interleaved or per-channel buffers) to mono Float32 → Int16, accumulate ~200 ms chunks, base64, and `evaluateJavaScript("window.__sysAudio && window.__sysAudio.pushAll('<b64>', <rate>)")` on the main thread (~5 calls/sec — negligible). Don't resample natively; pass the tap's native rate and let Web Audio resample.
3. **JS side — monkey-patch `MediaDevices.prototype.getUserMedia` in a `.atDocumentStart` user script** (page world, main frame). When the site requests audio: resolve the real mic stream, build an `AudioContext` → `createMediaStreamDestination()`, connect the mic source *and* a gain node fed by the pushed chunks (schedule each as an `AudioBufferSourceNode` against a running `next` timestamp with a ~150 ms jitter buffer), and hand back `new MediaStream(videoTracks + [mixedTrack])`. The site transcribes both sides without knowing anything changed.
4. **Stop paths — two, both required.** (a) JS: the site stops the *mixed* track it was handed, so patch `mixed.stop` to also stop the **real mic tracks** (else the orange mic dot stays on forever), post a stop message, close the context. (b) Native backstop: block-KVO `webView.microphoneCaptureState` — when it returns to `.none` (navigation, tab close, crashed page), tear down any sessions for that web view and stop the tap. Refcount sessions; stop the engine when none remain.
5. **Identity spoofing.** The destination node's track is labeled "MediaStreamAudioDestinationNode" — override `label`/`getSettings`/`getCapabilities` to mirror the mic track so device pickers don't glitch.

Details that bite:

- **Info.plist needs `NSAudioCaptureUsageDescription`.** First tap use triggers macOS's one-time **"System Audio Recording"** TCC prompt (its own class — much lighter than Screen Recording, no periodic re-nag). It is genuinely unavoidable; say so to the user up front, and never trigger it yourself during testing.
- **Run tap setup off the main thread** (a serial control queue): tap creation can stall behind the TCC prompt.
- **Read device UID CFString properties via `Unmanaged<CFString>?` + `withUnsafeMutablePointer`** — passing `&someCFString` to `AudioObjectGetPropertyData` draws a legitimate object-reference warning.
- **Route the start/stop messages through the same stateless script-message router as everything else**, and make capture a Settings toggle (default ON — parity with the official app is the point). User scripts bake at web-view creation; note "applies to new tabs" in the settings hint.
- The tap captures *everything* the Mac plays (including the wrapper's own web audio — WKWebView audio renders in the separate GPU process, so excluding your own PID does **not** exclude it). That matches official desktop-app behavior; don't fight it.

## Downloads

`WKDownloadDelegate` is the official API. Minimum implementation that just delegates to the user's chosen save location:

```swift
@available(macOS 11.3, *)
extension WebViewController: WKDownloadDelegate {
    func webView(_ webView: WKWebView, navigationAction: WKNavigationAction, didBecome download: WKDownload) {
        download.delegate = self
    }
    func download(_ download: WKDownload,
                  decideDestinationUsing response: URLResponse,
                  suggestedFilename: String,
                  completionHandler: @escaping (URL?) -> Void) {
        let panel = NSSavePanel()
        panel.nameFieldStringValue = suggestedFilename
        panel.canCreateDirectories = true
        if panel.runModal() == .OK { completionHandler(panel.url) }
        else { completionHandler(nil) }
    }
    func downloadDidFinish(_ download: WKDownload) {
        // Optionally: NSWorkspace.shared.activateFileViewerSelecting([url])
    }
}
```

Most wrapped sites delegate downloads to the browser by triggering an `<a download>`; the navigation delegate will catch this and you can return `.download` for the policy decision (macOS 11.3+).

## Dock badge

If the wrapped site has unread counts in the document title (a very common pattern: `(3) Inbox — Linear`), parse the title and update the dock:

```swift
self.onTitleChanged = { title in
    let count = Self.extractUnreadCount(from: title)
    DispatchQueue.main.async {
        NSApp.dockTile.badgeLabel = count > 0 ? "\(count)" : nil
    }
}

private static func extractUnreadCount(from title: String) -> Int {
    // Common patterns: "(3) Inbox", "[3] Foo", "3 unread — Foo"
    let patterns = [#"^\((\d+)\)"#, #"^\[(\d+)\]"#, #"(\d+)\s+unread"#]
    for p in patterns {
        if let regex = try? NSRegularExpression(pattern: p),
           let m = regex.firstMatch(in: title, range: NSRange(title.startIndex..., in: title)),
           let r = Range(m.range(at: 1), in: title),
           let n = Int(title[r]) {
            return n
        }
    }
    return 0
}
```

If you support multiple windows, sum counts across all tabs in all windows. Update on every title change.

## Inspect Element (Cmd+Opt+I)

Every wrapper exposes the Web Inspector via Cmd+Opt+I, matching Safari/Chrome. **On macOS Sonoma+ (and especially Tahoe / macOS 26), getting this to actually work requires three pieces of glue, all of which must be in place. Setting `isInspectable = true` alone is no longer sufficient and silently fails — no error, no context-menu item, the keyboard shortcut does nothing.**

### Piece 1 — entitlement

The inspector connects to the inspected process via task-port debugging. Ad-hoc-signed apps (everything built via `build.sh` in this skill) need the `get-task-allow` entitlement or the connection is refused with no diagnostics. Add to `Resources/<AppName>.entitlements`:

```xml
<key>com.apple.security.get-task-allow</key>
<true/>
```

After rebuilding, verify it's actually signed in:

```bash
codesign -d --entitlements - "/Applications/<AppName>.app" 2>&1 | grep get-task-allow
```

If grep returns nothing, the entitlements file isn't being passed to `codesign --entitlements` in `build.sh` — fix that first.

### Piece 2 — developer extras preference

WKPreferences has a private-but-stable `developerExtrasEnabled` key that controls whether the context-menu "Inspect Element" item is shown. Without it, right-click on the page shows the standard nav menu *without* an Inspect option. Set on the `WKWebViewConfiguration` **before** creating the WKWebView:

```swift
let config = WKWebViewConfiguration()
// … other config setup …
config.preferences.setValue(true, forKey: "developerExtrasEnabled")

webView = WKWebView(frame: .zero, configuration: config)
```

This is private API but has been stable since WebKit's first macOS port; Safari uses the same toggle internally.

### Piece 3 — isInspectable (macOS 13.3+ public API)

Set immediately after WKWebView construction, before anything else touches the web view:

```swift
webView = WKWebView(frame: .zero, configuration: config)
if #available(macOS 13.3, *) {
    webView.isInspectable = true
}
// … rest of setup (userAgent, delegates, etc.) …
```

Setting this *after* other configuration has been observed to silently no-op in some WebKit builds. Do it first.

### Piece 4 — menu item + action

Append to the bottom of the View menu in `AppDelegate.buildMainMenu()`:

```swift
viewMenu.addItem(NSMenuItem.separator())
let inspectItem = NSMenuItem(title: "Inspect Element",
                             action: #selector(inspectElement), keyEquivalent: "i")
inspectItem.keyEquivalentModifierMask = [.command, .option]
viewMenu.addItem(inspectItem)
```

And the action — uses WKWebView's private `_inspector` ivar (no public API for programmatically opening the inspector exists, but `_inspector` has been stable across every Safari/WebKit version since macOS 10.10). **Toggle visibility** by reading `isVisible` and dispatching to `show` or `hide`, not just calling `show` blindly — otherwise the keyboard shortcut feels broken because it only opens, never closes:

```swift
@objc func inspectElement() {
    // Single-window app: reach into the one WebViewController directly.
    // Tabbed app: use tabManager.activeWebViewController?.webView instead.
    guard let webView = webViewController.webView,
          let inspector = webView.value(forKey: "_inspector") as? NSObject
    else { return }
    let visible = (inspector.value(forKey: "isVisible") as? Bool) ?? false
    inspector.perform(NSSelectorFromString(visible ? "hide" : "show"))
}
```

Both `show` and `hide` keep the inspector *instance* alive — only its window is toggled — so state (selected element, console history, breakpoints) survives across toggles. Don't use `close`; that destroys the instance and the next `show` opens a fresh one with all state lost. The toggle also covers the right-click → Inspect Element flow: after the user opens via right-click, pressing Cmd+Opt+I hides it cleanly because `isVisible` is already true.

### Why all four are needed

| Piece | Skipping it produces… |
|---|---|
| `get-task-allow` entitlement | Inspector connection refused at debugger-attach time. No menu, no shortcut, no error. Most common failure mode on Tahoe. |
| `developerExtrasEnabled` preference | Right-click context menu lacks "Inspect Element" entirely. |
| `isInspectable = true` | macOS 13.3+ refuses to bind the inspector. |
| Menu item + private `_inspector` action | Right-click works, but the keyboard shortcut and View menu have no entry point. |

### Other notes

- **Don't conditionalize on debug builds.** Ship the inspector in production too — the wrappers in this skill are power-user tools.
- **The key combo is `Cmd+Opt+I`, not `Cmd+Shift+I`.** Cmd+Shift+I conflicts with several macOS-wide shortcuts. Cmd+Opt+I matches Chrome's devtools and Safari's "Show Web Inspector."
- **After adding the entitlement, force-refresh LaunchServices** (`lsregister -f "/Applications/<AppName>.app"`) and relaunch — otherwise the OS keeps using the cached entitlement set.
- **If it still doesn't work**, in this order: (1) verify the entitlement is in the signed bundle with `codesign -d --entitlements -`, (2) print `webView.value(forKey: "_inspector")` inside `inspectElement` to confirm it's non-nil, (3) try right-clicking — if Inspect appears in the context menu but the shortcut doesn't open it, the `_inspector` selector name has changed (rare, but check WebKit's `_WKInspector.h`).

## Traffic-light clearance (tab-less wrappers only)

When the window uses `[.fullSizeContentView]` + `titlebarAppearsTransparent = true` (the recommended setup — see `tabs-and-windows.md`), the WKWebView's content extends underneath the title bar. If the app has a tab bar, the tab bar's leading edge starts at ~78px to reserve space for the traffic-light buttons and everything is fine.

**Single-window, tab-less wrappers don't have that buffer.** The web content sits directly under the traffic lights and the site's leftmost top-bar UI (logo, hamburger menu, nav buttons) collides with the red/yellow/green circles. There are two CSS-injection fixes — pick whichever fits the site:

### Option A — shrink the site's leftmost element (lighter touch, preferred when it works)

If the site's leftmost element is a big logo (Google Calendar, Gmail, etc.), shrinking it is the least invasive fix — nothing moves, the logo just gets smaller, and the traffic lights end up sitting in the new whitespace next to it.

```css
/* Example: Google Calendar logo */
header[role="banner"] a[aria-label*="Calendar"] img,
header[role="banner"] img[alt*="Google Calendar"],
header[role="banner"] img[alt="Calendar"] {
    width: 24px !important;
    height: 24px !important;
}
```

This works when the rest of the top bar already starts far enough right that *only* the logo collides. Confirm by clicking each traffic-light button — if you can reach all three without hitting the site's hamburger or nav, you're done.

### Option B — pad the entire top header right (heavier, always works)

If shrinking one element isn't enough (e.g., the hamburger menu is also under the traffic lights), push the whole header right:

```swift
private static let trafficLightClearanceCSS = """
/* Push the site's leftmost top-bar content right of the macOS traffic lights. */
header[role="banner"] {
    padding-left: 80px !important;
}
"""
```

### Option C — push the whole top bar *down* by a few pixels (cheapest, often enough)

The traffic-light buttons are vertically centered in the title-bar area. If the site's top header has any tolerance for being shifted down a touch, a small `margin-top` on the banner moves the entire header out of the title-bar overlap zone — no need to push horizontally:

```swift
private static let trafficLightClearanceCSS = """
/* Push the entire top bar down so it clears the macOS title-bar / traffic-light buttons. */
header[role="banner"] {
    margin-top: 10px !important;
}
"""
```

This is the lightest-touch option visually — the site's UI layout isn't reshuffled, just shifted. 10px is the sweet spot; <5px isn't enough breathing room, >12px starts to feel like the page is "floating." Often you can combine this with Option A (shrink the logo) for a clean result on sites with a particularly bulky leftmost element.

This is the one to try **first** for most wrappers — it's a single rule, sites tend to absorb a few pixels of header offset gracefully, and you avoid touching the site's actual element sizes.

### Common machinery (all three options)

Either way, inject the CSS through the same MutationObserver pattern as theme injection so SPA re-renders don't drop it:

```swift
func injectChromeCSS() {
    let escaped = Self.trafficLightClearanceCSS
        .replacingOccurrences(of: "\\", with: "\\\\")
        .replacingOccurrences(of: "`", with: "\\`")
        .replacingOccurrences(of: "$", with: "\\$")
    let js = """
    (function() {
        const STYLE_ID = '__appname_chrome';
        const CSS = `\(escaped)`;
        function ensure() {
            let el = document.getElementById(STYLE_ID);
            if (!el) {
                el = document.createElement('style');
                el.id = STYLE_ID;
                el.textContent = CSS;
                (document.head || document.documentElement).appendChild(el);
            }
        }
        ensure();
        if (!window.__appname_chromeObserver) {
            window.__appname_chromeObserver = new MutationObserver(() => {
                if (!document.getElementById(STYLE_ID)) ensure();
            });
            window.__appname_chromeObserver.observe(document.documentElement, {
                childList: true, subtree: true
            });
        }
    })();
    """
    webView.evaluateJavaScript(js, completionHandler: nil)
}
```

Trigger it from the same hooks as theme injection — `didFinish` and the `webView.url` KVO observer — so SPA route changes don't lose the rule. Keep the chrome style ID (`__appname_chrome`) distinct from the theme style ID (`__appname_theme`) so they don't fight.

**Picking the selector.** The CSS is necessarily site-specific. Defaults to try in order:

1. `header[role="banner"]` — standard ARIA pattern, works for almost all Google products (Calendar, Gmail, Drive, etc.) and most modern web apps.
2. `[role="banner"]` — same role, broader. Use if the banner is on a `div`, not a `header`.
3. Site-specific class or id — last resort. Document it in the project's CLAUDE.md so future you knows the selector might rot if the site changes their HTML.

**Width.** 80px is enough for the three traffic-light buttons plus comfortable breathing room. Don't go below 78px — that's the de-facto minimum tab-bar offset across the macOS wrapper ecosystem.

**Don't try to fix this at the AppKit level** by adding a toolbar accessory view or shrinking the web view's frame. The accessory view sits on top of the title bar, not pushing content underneath; shrinking the frame creates an ugly empty band above the page where the window background shows through. CSS injection is the right layer.

## Window dragging from the title-bar zone (tab-less wrappers only)

Same setup, paired bug: with `[.fullSizeContentView] + titlebarAppearsTransparent = true`, the WKWebView extends under the title bar AND **intercepts every mouse-down in that zone**. The user clicks the empty dark band at the top expecting to drag the window — nothing happens. Tabbed apps don't have this problem because the tab bar is an NSView (not WKWebView) sitting in the title-bar zone, and AppKit drags it natively.

The fix is a transparent overlay view sized to the title-bar zone, layered on top of the WKWebView. **Don't** use `mouseDownCanMoveWindow = true` — it's the documented API, but it's unreliable next to layer-backed siblings (WKWebView always is): the window-move handoff happens at a layer level and the click ends up routed into the web content instead of triggering a drag. Use the explicit `NSWindow.performDrag(with:)` API in `mouseDown(with:)` instead — that's the canonical pattern and it works regardless of layer-backing.

### The drag-overlay view

```swift
private final class TitleBarDragView: NSView {
    override func mouseDown(with event: NSEvent) {
        window?.performDrag(with: event)
    }
}
```

That's the whole class — three lines. Default `hitTest(_:)` returns `self` when the point is in bounds, which is what we want (every click in the overlay's bounds becomes a drag).

### Wiring it into the window

The webView can't be the window's contentView directly anymore — wrap it in a plain container so the drag overlay can be a sibling on top:

```swift
private static let titleBarHeight: CGFloat = 28
private var titleBarDragView: TitleBarDragView!

// In buildWindow(), after creating webViewController:
let container = NSView()
let webView = webViewController.view
webView.translatesAutoresizingMaskIntoConstraints = false
container.addSubview(webView)

titleBarDragView = TitleBarDragView()
titleBarDragView.translatesAutoresizingMaskIntoConstraints = false
container.addSubview(titleBarDragView)  // added last → on top of the webView

NSLayoutConstraint.activate([
    webView.leadingAnchor.constraint(equalTo: container.leadingAnchor),
    webView.trailingAnchor.constraint(equalTo: container.trailingAnchor),
    webView.topAnchor.constraint(equalTo: container.topAnchor),
    webView.bottomAnchor.constraint(equalTo: container.bottomAnchor),

    titleBarDragView.leadingAnchor.constraint(equalTo: container.leadingAnchor),
    titleBarDragView.trailingAnchor.constraint(equalTo: container.trailingAnchor),
    titleBarDragView.topAnchor.constraint(equalTo: container.topAnchor),
    titleBarDragView.heightAnchor.constraint(equalToConstant: Self.titleBarHeight),
])

window.contentView = container
```

The traffic-light buttons are drawn by the window frame on top of the contentView, so they remain clickable through the overlay — no special handling needed for the leftmost ~70pt.

### Pair with chrome CSS that pushes the header down

A 28pt drag overlay covers the top 28pt of whatever the page is rendering. If the site's header sits at y=0 (most do), its click targets are now eaten by the drag overlay. Push the header down:

```css
header[role="banner"] {
    margin-top: 28px !important;
}
```

This is the same `header[role="banner"]` selector used for traffic-light clearance — combine the two rules in one chrome CSS block. The 10px-margin-top option from the traffic-light section is no longer sufficient on its own: it clears the buttons visually but still leaves the header's top 18px under the drag overlay (un-clickable). 28px is the canonical macOS title-bar height; matching it gives a clean separation.

### Fullscreen behavior

In fullscreen there's no title bar to clear, so the drag overlay would just eat the top 28pt of the page for no reason. Hide it and reclaim the header margin on the fullscreen transitions:

```swift
func windowWillEnterFullScreen(_ notification: Notification) {
    titleBarDragView?.isHidden = true
    webViewController?.setFullScreenAttribute(true)
}

func windowWillExitFullScreen(_ notification: Notification) {
    titleBarDragView?.isHidden = false
    webViewController?.setFullScreenAttribute(false)
}
```

Where `setFullScreenAttribute(_:)` flips a `data-` attribute on `<html>` that a CSS rule keys off:

```swift
func setFullScreenAttribute(_ on: Bool) {
    let js = "document.documentElement.setAttribute('data-yourapp-fullscreen', '\(on ? "true" : "false")');"
    webView.evaluateJavaScript(js, completionHandler: nil)
}
```

```css
html[data-yourapp-fullscreen="true"] header[role="banner"] {
    margin-top: 0 !important;
}
```

Use `windowWillEnterFullScreen` / `windowWillExitFullScreen` (not `windowDidEnter…`/`windowDidExit…`) so the page reflows during the animation instead of after.

### Why not just turn off `fullSizeContentView`?

This is a real fork in the road, not a trap to avoid — pick deliberately:

**Keep `fullSizeContentView` (seamless look).** The web content flows under a transparent title bar, so the wrapper feels native to the *site* rather than like a generic Safari window — the look every polished single-view wrapper uses (Linear, Notion, ChatGPT). The price: you take on the drag overlay (~30 lines), traffic-light-clearance CSS, header-margin CSS, and the fullscreen transitions — **all of it DOM-dependent**, keyed off the site's `header[role="banner"]` and friends. When the site reshuffles its markup, that chrome CSS silently breaks and needs maintenance. Use `titleVisibility = .hidden` here (the title would otherwise paint over the content).

**Turn it off (standard title bar).** Use a plain titled window — `[.titled, .closable, .miniaturizable, .resizable]`, `titlebarAppearsTransparent = true`, `titleVisibility = .visible`, **no** `fullSizeContentView`. The web view now sits *below* a thin (~28pt) native title bar instead of under it. That one change removes the entire fragile-CSS surface: **no drag overlay** (the native title bar drags itself), **no traffic-light clearance**, **no header-margin CSS**, **nothing keyed off the site's DOM** — so a site redesign can't break your chrome. The transparent title bar still blends into the window background for a tidy look, and `titleVisibility = .visible` gives you the page title in the bar for free (handy when there's no tab bar to host it). The only cost is that ~28pt strip isn't part of the web page.

**Rule of thumb:** if the wrapped site's header is complex or changes often, or you simply don't want to babysit per-site chrome CSS, **turn `fullSizeContentView` off**. If the seamless under-chrome aesthetic is the priority and you're willing to maintain the CSS, keep it on and spend the overlay code.

## Anti-patterns

- **Don't add `WKWebsiteDataStore.nonPersistent()`.** Users get logged out on every launch. Use `.default()`.
- **Don't override `WKNavigationDelegate.decidePolicyFor` and forget the subframe case.** You'll break OAuth iframes, embedded videos, payment widgets, and CAPTCHA. Always allow subframes.
- **Don't load the OAuth request into the *same* web view (or return `nil`) from `createWebViewWith`.** This is the load-in-same-view trap — `window.open()` returns `null` and `window.opener` is never wired, so "Continue with Google/Apple" opens a **black/blank window** that never completes sign-in. Return a real, freshly-created `WKWebView` (a new foreground tab) so the opener relationship is live, and implement `webViewDidClose` to clean up the spent popup tab. (The same-view approach is a common mistake; see *OAuth popups*.)
- **Don't forget `webViewWebContentProcessDidTerminate(_:)`.** Without it the app silently blanks to the window background color after long uptime or sleep/wake and never recovers. See *Content-process termination recovery*.
- **Don't reload every tab unconditionally on wake.** Gate recovery on the per-tab "terminated" flag — a blanket reload-on-wake throws away scroll position and in-progress state on healthy tabs and is immediately noticeable.
- **Don't put zoom in `WKPreferences`.** It doesn't exist there. `webView.pageZoom` is the property.
- **Don't ship a tab-less wrapper without the traffic-light clearance CSS.** It looks fine in screenshots until the user notices the hamburger menu is under the red circle. Always inject the padding.
- **Don't open `NSWorkspace.shared.open(url)` for every link the user clicks.** Only when the host is outside the allowlist. Otherwise every internal navigation kicks the user out of the app.
- **Don't disable JavaScript.** The whole point is to wrap a JS-driven web app. `allowsContentJavaScript` must be `true`.
- **Don't put `customUserAgent` in `WKWebViewConfiguration`.** It's a property on `WKWebView` itself. Setting it on the config does nothing.
