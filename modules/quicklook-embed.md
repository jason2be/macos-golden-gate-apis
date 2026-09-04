---
description: macOS 27 Quick Look embed via QLPreviewView. Use when embedding QLPreviewView in an NSView hierarchy (e.g. SwiftUI NSViewRepresentable), exposing non-Apple file types to QLPreviewItem, or wrapping NSWorkspace completion-handler APIs as async/await.
target_apis: QLPreviewView, QLPreviewViewStyle, QLPreviewItem, NSWorkspace.OpenConfiguration, NSWorkspace.open(_:withApplicationAt:configuration:completionHandler:)
os_version: macOS 27 / Xcode 27
last_verified: 2026-09-04
---

# Quick Look Embed (macOS 27)

> **Scope:** This module covers embedding `QLPreviewView` (QuickLookUI framework) into an `NSView` hierarchy on macOS 27. It documents the actual public API of `QLPreviewView` (verified against Apple Developer Documentation on 2026-09-04), the cross-platform caveats of `QLPreviewItem`, and the standard pattern for wrapping `NSWorkspace.open(...)` completion handlers as `async throws`.

## QLPreviewView — actual public API

`QLPreviewView` is a `NSView` subclass in the **QuickLookUI** framework (not `QuickLook`). Its actual public surface (verified 2026-09-04):

```swift
class QLPreviewView: NSView {
    // Initialization
    init(frame: CGRect, style: QLPreviewViewStyle)   // style: .normal or .compact
    init(frame: CGRect)                              // defaults to .normal

    // Item management
    var previewItem: QLPreviewItem?                  // assign to set; nil to clear
    func refreshPreviewItem()                        // re-render the current item

    // Behavior
    var autostarts: Bool                             // default true; auto-render on set
    var shouldCloseWithWindow: Bool                  // default true

    // Lifecycle
    func close()                                     // release current item; view reusable
}
```

**There is NO `start(_:)`, NO `stop()`, NO `style` property.** Style is set in `init` and immutable.

### Correct SwiftUI NSViewRepresentable wrapper

```swift
import SwiftUI
import QuickLookUI

@MainActor
struct QuickLookView: NSViewRepresentable {
    let fileURL: URL
    let isCompact: Bool

    func makeNSView(context: Context) -> QLPreviewView {
        let view = QLPreviewView(
            frame: .zero,
            style: isCompact ? .compact : .normal
        )
        view.shouldCloseWithWindow = false
        view.autostarts = true
        view.previewItem = QuickLookPreviewItem(url: fileURL, title: fileURL.lastPathComponent)
        return view
    }

    func updateNSView(_ nsView: QLPreviewView, context: Context) {
        // Only refresh if URL actually changed
        guard (nsView.previewItem as? QuickLookPreviewItem)?.previewItemURL != fileURL else { return }
        nsView.previewItem = QuickLookPreviewItem(url: fileURL, title: fileURL.lastPathComponent)
        nsView.refreshPreviewItem()
    }

    static func dismantleNSView(_ nsView: QLPreviewView, coordinator: Coordinator) {
        nsView.previewItem = nil
        nsView.close()
    }
}
```

## QLPreviewItem — macOS availability caveat

`QLPreviewItem` protocol's Apple Developer page lists availability as **iOS, iPadOS, visionOS — no macOS** (as of 2026-09-04). However, `QLPreviewView` on macOS does read the `previewItemURL` property via ObjC-runtime KVC, so a plain `NSObject` subclass with `@objc` properties works:

```swift
import Foundation

@MainActor
final class QuickLookPreviewItem: NSObject {
    @objc let previewItemURL: URL?
    @objc let previewItemTitle: String?

    init(url: URL, title: String?) {
        self.previewItemURL = url
        self.previewItemTitle = title
        super.init()
    }
}
```

**Do NOT explicitly conform to `QLPreviewItem`** — it may not have `macOS` availability in the type system, and the runtime path is ObjC KVC anyway.

## QLPreviewController — iOS only

`QLPreviewController` (with `canPreviewItem:`) is **iOS only**. On macOS, there is no equivalent "can this file be Quick-Look-previewed?" API. Use one of these heuristics:

1. `NSWorkspace.shared.urlsForApplications(toOpen: url)` — non-empty means some app claims the type (approximate).
2. `UTType(filenameExtension: ext).conforms(to: .image)` or similar UTI conformance checks.
3. A static extension allowlist maintained by the app.

## NSWorkspace.open — async pattern

`NSWorkspace.open(_:withApplicationAt:configuration:completionHandler:)` is **completion-handler only** (no `async throws` overload). Standard wrapping pattern:

```swift
func openInDefaultApp(_ fileURL: URL) async throws {
    guard let appURL = NSWorkspace.shared.urlForApplication(toOpen: fileURL) else {
        throw MyError.noApplication
    }
    let config = NSWorkspace.OpenConfiguration()
    config.activates = true

    try await withCheckedThrowingContinuation { (cont: CheckedContinuation<Void, Error>) in
        NSWorkspace.shared.open(
            [fileURL],
            withApplicationAt: appURL,
            configuration: config
        ) { _, error in
            if let error {
                cont.resume(throwing: error)
            } else {
                cont.resume(returning: ())
            }
        }
    }
}
```

## Sources

- [QLPreviewView — Apple Developer](https://developer.apple.com/documentation/quicklookui/qlpreviewview)
- [QLPreviewViewStyle — Apple Developer](https://developer.apple.com/documentation/quicklookui/qlpreviewviewstyle)
- [QLPreviewItem — Apple Developer](https://developer.apple.com/documentation/quicklook/qlpreviewitem)
- [NSWorkspace — Apple Developer](https://developer.apple.com/documentation/appkit/nsworkspace)
- [Document — Apple Developer](https://developer.apple.com/documentation/swiftui/document)
- [ReadableDocument — Apple Developer](https://developer.apple.com/documentation/swiftui/readabledocument)
- Verified 2026-09-04.
