# Migration: macOS 14+ → macOS 27 Golden Gate

> For module-specific API changes, see `modules/*.md`.
> This file is the generic upgrade checklist for macOS 27 Golden Gate migration.

Use this checklist when bumping your app's deployment target to macOS 27 "Golden Gate" and adopting the Xcode 27 / Swift 6.4 toolchain.

This checklist is the **upgrade procedure**, not the **API reference**. Every row in the table below points back to `RESEARCH.md` for the underlying API rationale and to the matching `modules/*.md` file for the broader SwiftUI / AppKit / FoundationModels / WebKit story.

The breaking-changes list is intentionally compact: it only enumerates items that compile-time break your code on the Xcode 27 / Swift 6.4 toolchain. Visual regressions (menu icon hiding, Liquid Glass refinements) are captured in the table because they manifest as runtime / visual diffs, not as compile errors.

## How to use this file

1. Walk the **Pre-flight** section once, in order, before any code change. Each item is a hard prerequisite — failing any of them invalidates every subsequent step.
2. Audit the **API-Specific** table, then map each "Action" to a work item before touching code.
3. Apply the **Tahoe → Golden Gate Breaking Changes** in dependency order: SwiftUI document/`@State` first, AppKit menu/gesture changes second, concurrency refinements third.
4. Run the **Verification** commands in the listed order.
5. Cross-link the resulting PR / commit to this file so the next migration can diff against a known-good baseline.

## Pre-flight

- [ ] Confirm target Xcode is **27+**, Swift **6.4+** (`xcodebuild -version`, `swift --version`)
- [ ] Confirm deployment drops Intel — Golden Gate is **Apple Silicon only** (`uname -m` on dev machine is `arm64`)
- [ ] Audit third-party frameworks for Intel-only or universal binaries — Apple has pre-announced that all Intel-based software (excluding legacy games) will be incompatible with macOS 28.0
- [ ] Audit `Darwin.stat` / custom `FilePath.stat` extensions against new Swift System APIs (SYS-0006, SYS-0008) to avoid resolution collisions

## Tahoe (26) → Golden Gate (27) Breaking Changes

1. `FileDocument` deprecated → use `Document` / `ReadableDocument` / `WritableDocument`. `ReferenceFileDocument` is **deprecated as well** (same interface marking, message "Use Document protocol instead."). **IMPORTANT: The new protocols have a fundamentally different shape from `FileDocument` — they use reader/writer closures returning `FileWrapperDocumentReader<Snapshot>` / `FileWrapperDocumentWriter<Snapshot>`, not synchronous `init(configuration:)` / `fileWrapper(snapshot:configuration:)` methods. See `modules/swiftui-appkit.md` § "SwiftUI — document model" for the actual API.** (177458781, 178776840)
2. `FileWrapperDocumentWriter.makeFileWrapper` closure signature gained `previous: FileWrapper?` so package documents can mutate in place. (180301399)
3. `@State` macro rewrite (Xcode 27, back-deploys to iOS 17-aligned OSes) — 2 compile-time breaks:
   - Assigning in `init` while also providing a default at declaration no longer compiles.
   - Synthesized private memberwise init via an extension is disabled when any stored member is private and uses `@State`.
4. `URLDocumentConfiguration` is `@MainActor`-isolated `@Observable` — don't capture or use it off the main actor (isolation checks reject it; this is not a `Sendable` change). (180302075)
5. `NSTextView` selection now uses `NSTextSelectionManager` (gesture-recognizer-based) rather than `NSEvent` mouse overrides. Existing `mouseDown:` overrides keep working via a binary-compatible fallback. (163365571)
6. `NSMenu` hides menu item images by default for apps linked on macOS 27 SDK — both symbol and non-symbol. Use `NSMenuItem.preferredImageVisibility` to opt back in; SwiftUI `Menu`s should use `labelStyle(.titleAndIcon)` for icons. (170477566, 179374305)
7. Exclusive gesture behavior is now the default — only the initial hit-tested hierarchy activates gestures until all terminate. Opt-out via `NSView.exclusiveGestureBehavior`, app-wide `Info.plist` key `NSViewGestureRecognizerIsExclusive`, or `NSGestureRecognizerSuppressesMainMenuActions`. (173551081)
8. SE-0508 source break: computed property with `init` accessor + array/dictionary literal initial — swap the order (declare the `init` accessor before the getter). (180969028)
9. `PreviewProvider` deprecated in Xcode 27 Beta 3 → use the `#Preview` macro.
10. `ImageCreator` discontinued on macOS 27 / iOS 27 / iPadOS 27 / visionOS 27 or later — Apple recommends Image Playground.
11. `DVDPlayback` framework removed from the macOS 27 SDK. (179136240)
12. AFP removed — Time Machine backups to AirPort Time Capsule no longer supported; SMB / network drive to APFS is the replacement. Boot Camp also removed.

### Ordering notes

- Items 1–4 are SwiftUI / document-stack compile-time breaks and must land before any test run.
- Items 5–7 are AppKit runtime behavior changes — code still compiles on the new SDK, but user-visible behavior shifts in selection, menus, and gestures.
- Item 8 (SE-0508) is a Swift 6.4 source-compatibility break that the Xcode 27 compiler surfaces as a diagnostic. Sweep the codebase before declaring the build green.
- Items 9–12 are deprecations and removals that compile cleanly but fail at link time (item 11) or at runtime (items 10, 12). Run a smoke test after the toolchain bump to catch them.
- The `@concurrent` attribute change is the largest off-main async correctness fix in this release; treat it as a tier-1 migration item, not a follow-up.

## API-Specific Migration Table

| Surface | Golden Gate implication | Action |
|---|---|---|
| Swift 6 strict concurrency | `@concurrent` replaces bare `nonisolated` for off-main async | Audit `DocumentReader` / `DocumentWriter` and any custom concurrency helpers; replace `nonisolated` with `@concurrent` where off-main work is intended |
| WKWebView preview | WebKit for Safari 27 — 1,000+ improvements, no breaking changes to `WKWebView` | Test against WebKit 27 runtime |
| Foundation Models | Foundation Models now work with **any** provider via the `Language Model` protocol; multimodal prompts; Dynamic Profiles; Evaluations framework; `fm` CLI + Python SDK | Refactor to provider-agnostic protocol; expose `Dynamic Profiles` for mode toggle; consider Evaluations harness for AI regression tests |
| Apple Vision OCR | Vision OCR is now exposed as a callable tool for Foundation Models | Decide per-document: keep direct `VNRecognizeTextRequest` for structured pipeline, or move OCR invocation into an FM tool |
| App Intents / Spotlight | `@AppEntity(schema: .photos.asset)` may fail to compile due to new schema properties (181800016); `AppEntity` 10 MB cumulative size limit (181763422); `SpotlightSearchTool` requires `.focused()` guide with on-device system LM | Conditionalize `.photos.asset` conformance with `if #available`; chunk large entities or move children to queries; configure `SpotlightSearchTool.Configuration(sources: [.coreSpotlight], guide: .focused(.documents))` |
| `@Observable` / `@Entry` / MVVM | `URLDocumentConfiguration` is now `@MainActor`-isolated `@Observable`; `@Entry` warns if you store default class instances or closures in the environment | Audit document-state models; add explicit `@MainActor` if needed; review `@Entry` defaults |
| SwiftUI `Menu` / `labelStyle` | Menus hide symbol/non-symbol images by default for macOS 27 SDK links — **applies to `Menu` items, `ToolbarItem` items, AND `.contextMenu` items**. `Toggle` and `Picker` are unaffected. | Add `labelStyle(.titleAndIcon)` on `Menu` instances where icons were relied upon; audit `.contextMenu { }` closures too |
| NSTextView / TextKit 2 | `NSTextView` now uses `NSTextSelectionManager` (gesture-based) internally | Audit any `mouseDown:` overrides; migrate custom selection gestures to `NSGestureRecognizer` subclasses |
| Exclusive gesture behavior | Exclusive is the new default | If your app relies on non-exclusive gestures, opt out per-view via `NSView.exclusiveGestureBehavior` or app-wide via `Info.plist` |
| swift-system / stat | System Swift APIs for `stat`/`lstat`/`fstat`/`fstatat`; unqualified `Darwin.stat` calls in extensions may collide | Audit custom `stat` extensions; qualify calls with `Darwin.stat(...)` explicitly, or migrate to the new `FilePath.stat()` instance methods |
| Quick Look embed (QLPreviewView) | `QLPreviewView` is `macOS 10.6+` in QuickLookUI framework — actual API is `init(frame:style:)`, `previewItem` property, `refreshPreviewItem()`, `close()`. `QLPreviewItem` protocol has no explicit `macOS` availability; rely on `@objc dynamic previewItemURL` on a plain `NSObject` subclass. `QLPreviewController` is **iOS-only**. | Adopt `NSViewRepresentable` wrapper for `QLPreviewView` using `init(frame:style:)`; wrap `NSWorkspace.open(...)` with `withCheckedThrowingContinuation`. See `modules/quicklook-embed.md`. |
| Liquid Glass refinements | Refined (not replaced) Liquid Glass: Dark Mode for macOS added, refreshed materials, refined typography | Refresh design assets; review visual regressions |
| Core AI | New framework for on-device ML on Apple Silicon; AOT compilation, Neural Engine background access (requires entitlement) | Evaluate for custom on-device models; declare `com.apple.developer.background-tasks.continued-processing.inference` entitlement if needed |

## Verification

Run, in order:

- [ ] `xcodebuild -version` confirms Xcode 27+
- [ ] `swift --version` confirms Swift 6.4+
- [ ] `uname -m` on dev machine is `arm64`
- [ ] Full build succeeds with no errors
- [ ] Full test suite passes
- [ ] UI smoke test in an isolated runner (do not use foreground-UI automation on the operator's desktop)
- [ ] Visual regression check: menu icons, Liquid Glass materials, gesture behavior

## References

- macOS 27 release notes: https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes
- macOS 27 What's New: https://developer.apple.com/macos/whats-new/
- Xcode 27 release notes: https://developer.apple.com/documentation/Xcode-Release-Notes/xcode-27-release-notes
- Rosetta deprecation announcement: https://developer.apple.com/news/?id=w5ngl9k2
- WWDC26 announcement: https://developer.apple.com/news/?id=8rgqj83s
- Design kits for iOS, iPadOS, macOS 27: https://developer.apple.com/news/?id=e2lxw9l1
- ImageCreator deprecation: https://developer.apple.com/news/?id=dz9wvq0r
- WWDC26 241 — What's new in the Foundation Models framework: https://developer.apple.com/videos/play/wwdc2026/241/
- WWDC26 295 — Meet the App Intents Testing framework: https://developer.apple.com/videos/play/wwdc2026/295/
- WWDC26 298 — Meet the Evaluations framework: https://developer.apple.com/videos/play/wwdc2026/298/
- WWDC26 334 — Build AI powered scripts with the fm CLI and Python SDK: https://developer.apple.com/videos/play/wwdc2026/334/
- swift-system SYS-0006 (stat APIs): https://github.com/apple/swift-system/blob/main/Proposals/0006-system-stat.md
- swift-system SYS-0008 (backdeploy / cinterop stat): https://github.com/apple/swift-system/blob/main/Proposals/0008-backdeploy-cinterop-stat.md
- RESEARCH.md (this skill): `RESEARCH.md`
