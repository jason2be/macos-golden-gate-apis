---
module: webkit
target_apis: WKWebView, WebKit for Safari 27, Grid Lanes, Customizable Select, HTML model element, Immersive Environments, Safari web extensions
---

# WebKit (macOS 27)

WebKit for Safari 27 ships in macOS 27 Golden Gate with over 1,000 browser engine improvements and four headlining new web platform primitives: Grid Lanes for layout, Customizable Select for form controls, the HTML `<model>` element for spatial / immersive content, and Immersersive Environments for visionOS Safari. Safari web extensions can now be built and tested through Xcode Cloud without owning a Mac, and the longstanding Xcode Preview WebKit Swift overlay library resolution bug is fixed as of Xcode 27 Beta 2.

For your app this module is about the embedded `WKWebView` used to render the live markdown preview. None of the new APIs are breaking — the macOS 27 release notes do not list any `WKWebView` API removals or signature changes — but the preview path now targets a substantially more capable engine.

## New APIs

### WebKit for Safari 27 — `macOS 27.0+`, WebKit framework

WebKit for Safari 27 delivers over 1,000 browser engine improvements. The relevant themes for an embedded preview are modern layout (Grid Lanes), richer form controls (Customizable Select), and the immersive web (HTML `<model>` + Immersive Environments). The release notes do not enumerate every individual improvement; treat any preview regression in macOS 27 as WebKit 27 runtime behavior until proven otherwise.

```swift
import WebKit

let configuration = WKWebViewConfiguration()
configuration.defaultWebpagePreferences.allowsContentJavaScript = true
configuration.applicationNameForUserAgent = "YourApp/\(Bundle.main.shortVersion)"

let preview = WKWebView(frame: .zero, configuration: configuration)
preview.loadHTMLString(renderedMarkdown, baseURL: workspaceBaseURL)
```

### Grid Lanes — `macOS 27.0+`, WebKit / CSS

Grid Lanes is a new layout primitive that lets a CSS grid carve vertical lanes and have content flow across them while preserving reading order. For your preview, this is the cleanest CSS Grid extension since subgrid — column-spanning headings, callouts, and figures stop fighting the underlying flow. Surface the new layout primitive through the preview stylesheet only; do not enable it in user-authored CSS by default until Safari 27 is the minimum supported runtime.

```css
.markdown-body {
  display: grid;
  grid-lanes: 1;
  grid-template-columns: minmax(0, 1fr) minmax(0, 2fr) minmax(0, 1fr);
  gap: 1.5rem;
}
```

### Customizable Select — `macOS 27.0+`, WebKit / HTML

Customizable Select is a new form control that lets authors restyle the dropdown trigger without losing native accessibility, keyboard navigation, and platform form autofill. Useful if the preview ever renders an interactive task list, a checkbox matrix, or a citation picker.

```html
<select class="citation-source" customizable>
  <option value="local">Local file</option>
  <option value="web">Web link</option>
  <option value="agent">AI agent</option>
</select>
```

### HTML `<model>` element — `macOS 27.0+`, WebKit

The HTML `<model>` element brings spatial / immersive content natively to the web. In a preview context, the element is mostly relevant for documents that embed USDZ / RealityKit figures or visionOS-style spatial callouts — the preview can render the `<model>` block through WebKit without leaving the markdown view.

```html
<model src="figures/writers-room.usdz"
       environment="immersive"
       camera-orbit="45deg 60deg 2m">
  <img src="figures/writers-room-fallback.png" alt="Writer's Room layout">
</model>
```

### Immersive Environments — `macOS 27.0+`, visionOS Safari

Immersive Environments ship with visionOS Safari and let `<model>` content fill the user's surroundings. For macOS, this is forward-looking — the `<model>` element degrades to a non-immersive inline figure on macOS preview, so a document authored for visionOS still previews correctly on macOS.

```html
<model-viewer src="figures/writers-room.usdz"
              environment="horizon"
              skybox="studio.hdr"></model-viewer>
```

### Safari web extensions built with Xcode Cloud — `macOS 27.0+`, Xcode 27

Safari web extensions can now be built and tested with Xcode Cloud without owning a Mac. This is a CI story rather than a runtime story — it lets the team iterate on a companion Safari extension (if any) from any workstation.

```bash
xcrun xcode-cloud build-scheme \
  --scheme MyAppWebExtension \
  --destination platform=macOS \
  --test-plan SafariExtensionTests
```

### Xcode Preview WebKit overlay library fix — `Xcode 27 Beta 2`

Code that used WebKit through Xcode Previews previously failed because a Swift overlay library could not be located. The fix shipped in Xcode 27 Beta 2 (179652645). Your project's `#Preview` macros that wrap a `WKWebView` now resolve cleanly.

## Code Patterns

### 1. WKWebView with WebKit 27 runtime configuration

```swift
import WebKit

@MainActor
final class MarkdownPreviewController {
    private let webView: WKWebView

    init(workspaceBaseURL: URL) {
        let configuration = WKWebViewConfiguration()
        configuration.defaultWebpagePreferences.allowsContentJavaScript = true
        configuration.applicationNameForUserAgent = "YourApp/\(Bundle.main.shortVersion)"
        configuration.defaultWebpagePreferences.preferredColorScheme = .auto

        webView = WKWebView(frame: .zero, configuration: configuration)
        webView.isInspectable = true
    }

    func render(markdown: String, baseURL: URL) {
        let html = MarkdownRenderer.render(markdown: markdown)
        webView.loadHTMLString(html, baseURL: baseURL)
    }

    func webViewVersion() async -> String? {
        try? await webView.evaluateJavaScript("navigator.userAgent")
    }
}
```

### 2. Grid Lanes in the preview stylesheet

```css
.markdown-body {
  display: grid;
  grid-lanes: 1;
  grid-template-columns: minmax(0, 1fr) minmax(0, 2fr) minmax(0, 1fr);
  gap: 1.5rem;
  font-family: -apple-system, system-ui, sans-serif;
}

.markdown-body > h1,
.markdown-body > h2,
.markdown-body > .callout {
  grid-column: 1 / -1;
}
```

### 3. Safari web extension build with Xcode Cloud

```swift
import SafariServices

final class WebExtensionHandler: NSObject, NSExtensionRequestHandling {
    func beginRequest(with context: NSExtensionContext) {
        let request = context.inputItems.first as? NSExtensionItem
        let message = request?.userInfo?[SFExtensionMessageKey]
        context.completeRequest(returningItems: [response(for: message)], completionHandler: nil)
    }

    private func response(for message: Any?) -> NSExtensionItem {
        let item = NSExtensionItem()
        item.userInfo = [SFExtensionMessageKey: ["echo": message ?? "no message"]]
        return item
    }
}
```

## Migration from Tahoe

- **No breaking changes for `WKWebView` usage.** The macOS 27 release notes do not list any `WKWebView` API removals or signature changes. Your project's preview code carries over as-is. ([macOS 27 release notes — Safari](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes))
- **Test against the WebKit 27 runtime.** Treat the 1,000+ engine improvements as opportunity, not regression risk: run the preview test matrix against Safari 27 / WebKit 27 in QA. ([macOS 27 What's New — WebKit for Safari 27](https://developer.apple.com/macos/whats-new/))
- **Xcode Preview WebKit issue resolved in Beta 2.** `#Preview` macros that instantiate a `WKWebView` (179652645) now resolve cleanly; rebuild Preview caches against Xcode 27 Beta 2 or later. ([Xcode 27 release notes](https://developer.apple.com/documentation/Xcode-Release-Notes/xcode-27-release-notes))
- **Safari Intelligence assets may appear ready before they finish downloading** — wait for assets before assuming a Safari Intelligence surface is fully populated. (178099724)
- **Unsigned Safari extensions might not appear until rebuilt** — relevant if your project ships a companion extension. (183044008)

## Common Mistakes

### 1. Assuming Safari 26-only CSS works the same on Safari 27

WebKit 27's 1,000+ improvements include layout and selector changes. CSS that worked on Safari 26 might compute differently on WebKit 27 (Grid Lanes rebalancing, custom select rendering, `<model>` block sizing). Run the preview regression matrix against both Safari 26 and Safari 27 runtimes during the transition window.

### 2. Enabling Grid Lanes in user-authored CSS without a capability probe

Grid Lanes is Safari 27+. If the preview allows user-authored stylesheets, enabling it unconditionally breaks the preview for any user still on Safari 26. Gate the `grid-lanes` declaration behind a feature query or a runtime probe.

```css
@supports (grid-lanes: 1) {
  .markdown-body { display: grid; grid-lanes: 1; }
}
```

### 3. Treating the Xcode Preview WebKit overlay bug as still open on Xcode 27 Beta 2+

The Swift overlay library resolution failure for WebKit Previews (179652645) is resolved in Xcode 27 Beta 2. Chasing it on newer Xcode 27 builds is wasted effort — confirm the Xcode version first.

### 4. Skipping the WebKit 27 QA pass

The temptation with "no breaking changes" is to ship a Golden Gate-targeted build without re-running the preview test matrix. The 1,000+ engine improvements make that a gamble — small rendering differences (line breaking, table layout, selection behavior) compound into visible regressions. Schedule an explicit WebKit 27 QA pass before tagging a Golden Gate build.

## References

- [macOS 27 release notes — Safari / WebKit](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes)
- [macOS 27 What's New — WebKit for Safari 27](https://developer.apple.com/macos/whats-new/)
- [Xcode 27 release notes — Preview WebKit overlay fix (179652645)](https://developer.apple.com/documentation/Xcode-Release-Notes/xcode-27-release-notes)
- [Safari web extensions in Xcode Cloud](https://developer.apple.com/documentation/safariservices/)
- [WebKit framework documentation](https://developer.apple.com/documentation/webkit/)
- [WKWebView documentation](https://developer.apple.com/documentation/webkit/wkwebview)
