---
module: swiftui-appkit
target_apis: ReadableDocument, WritableDocument, Document, URLDocumentConfiguration, TextInputBorderShape, textInputBorderShape, TabsPickerStyle, concentricCornerRadii, fileExporter, FileWrapperDocumentWriter, FileWrapperDocumentReader, AsyncImage, View.asyncImageURLSession, TextRenderer, textRenderer, @Entry, @State macro, NSCrashOnStuckGestureTimeout, NSRefreshController, NSScrollView.refreshController, NSToolbarItemGroup.role, NSToolbarItemGroupRole, NSSegmentedControl.role, NSSegmentedControlRole, NSTextSelectionManager, NSApplication.presentationOptions, NSGestureRecognizer.cancellableByScrollGesture, NSView.exclusiveGestureBehavior, NSGestureRecognizerSuppressesMainMenuActions, NSTitlebarAccessoryViewController, NSTextView.menuForEvent, NSMenuItem.preferredImageVisibility, NSMenu, QLPreviewView, QLPreviewViewStyle, QLPreviewItem, FileWrapperDocumentReader
---

# SwiftUI & AppKit (macOS 27)

## New APIs

### SwiftUI — document model

#### `ReadableDocument` and `WritableDocument` — `macOS 27.0+`, Xcode 27 SDK

Protocols that replace `FileDocument` for documents with async I/O, progress reporting, and direct URL access. The two protocols combine into `Document`. `FileDocument` is deprecated; `ReferenceFileDocument` is superseded by `Document` for read-and-write cases. (158441552, 177458781, 178776840)

**CRITICAL: The new protocols have a fundamentally different shape from `FileDocument`.** The actual API surface (verified against Apple Developer Documentation 2026-09-04) is:

```swift
// ReadableDocument — read-only document
protocol ReadableDocument: AnyObject {  // MUST be a class (AnyObject)
    static var readableContentTypes: [UTType] { get }
    func reader(configuration: sending ReadConfiguration) -> sending FileWrapperDocumentReader<Snapshot>
    @MainActor func apply(snapshot: sending Snapshot, previous: sending Snapshot?) async throws
}

// WritableDocument — adds write capability
protocol WritableDocument: AnyObject {
    static var writableContentTypes: [UTType] { get }
    func writer(configuration: sending WriteConfiguration) -> sending FileWrapperDocumentWriter<Snapshot>
    @MainActor func snapshot(contentType: UTType) async throws -> sending Snapshot
}

// Document — combines both
protocol Document: ReadableDocument, WritableDocument {}
```

**The `Snapshot` type is whatever the conforming type chooses** (`String`, `Data`, `MarkdownRenderedDocument`, etc.). The lifecycle is:
1. SwiftUI calls `reader(configuration:)` to get a `FileWrapperDocumentReader<Snapshot>`.
2. The reader's closure transforms a `FileWrapper` to a `Snapshot` (synchronously, off the main actor if the closure is marked `@concurrent`).
3. SwiftUI then calls `@MainActor apply(snapshot:previous:)` on the document to install the snapshot.
4. On save, SwiftUI calls `@MainActor snapshot(contentType:)` to get the current state, then `writer(configuration:)` to get a `FileWrapperDocumentWriter<Snapshot>` whose closure produces a `FileWrapper`.

**There is no `init(configuration:)` on these protocols.** The legacy `FileDocument.init(configuration:)` pattern is **completely gone**.

#### `URLDocumentConfiguration` — `macOS 27.0+`, Xcode 27 SDK

`@MainActor`-isolated `@Observable` reference type that replaces `ReferenceFileDocumentConfiguration`. No longer conforms to `Sendable`; closures that previously captured it in `Sendable` contexts must drop the constraint. (180302075)

```swift
import SwiftUI

@Observable
@MainActor
final class WorkspaceDocumentModel {
    var draft: String = ""
    var lastSavedAt: Date?

    func configure(_ configuration: URLDocumentConfiguration) {
        configuration.didUndoChange = { [weak self] _ in
            self?.refreshFromDisk()
        }
    }
}
```

#### `TextInputBorderShape` + `textInputBorderShape(_:)` — `macOS 27.0+`, Xcode 27 SDK

New shape type plus modifier for input fields. `.squareBorder` and `.roundedBorder` are soft-deprecated in favor of `.bordered`. (173362083)

```swift
import SwiftUI

struct QuickFindBar: View {
    @State private var query: String = ""

    var body: some View {
        TextField("Find in document", text: $query)
            .textFieldStyle(.roundedBorder)
            .textInputBorderShape(.bordered)
            .padding(.horizontal)
    }
}
```

#### `TabsPickerStyle` — `macOS 27.0+`, Xcode 27 SDK

`Picker` style for tab-based navigation; VoiceOver reads each option as "tab". (173211711)

```swift
import SwiftUI

struct EditorModePicker: View {
    @State private var mode: EditorMode = .write

    var body: some View {
        Picker("Mode", selection: $mode) {
            Label("Write", systemImage: "pencil").tag(EditorMode.write)
            Label("Review", systemImage: "eye").tag(EditorMode.review)
            Label("Diff", systemImage: "rectangle.split.2x1").tag(EditorMode.diff)
        }
        .pickerStyle(.tabs)
    }

    enum EditorMode { case write, review, diff }
}
```

#### `concentricCornerRadii` / `concentricCornerRadii(in:)` — `macOS 27.0+`, Xcode 27 SDK

Properties on `GeometryProxy` that return the corner radii of a `ConcentricRectangle` aligned to the same frame. Drives custom drawing that matches the system material shape. (177185166)

```swift
import SwiftUI

struct RoundedBackdrop: View {
    var body: some View {
        GeometryReader { proxy in
            Canvas { context, _ in
                let radii = proxy.concentricCornerRadii
                let path = Path(roundedRect: proxy.frame(in: .local),
                                cornerRadii: RectangleCornerRadii(topLeading: radii.topLeading,
                                                                   bottomLeading: radii.bottomLeading,
                                                                   bottomTrailing: radii.bottomTrailing,
                                                                   topTrailing: radii.topTrailing))
                context.fill(path, with: .color(.background))
            }
        }
    }
}
```

#### `fileExporter(isPresented:documents:contentTypes:onCompletion:onCancellation:)` — `macOS 27.0+`, Xcode 27 SDK

New batch-export modifier that accepts any `[WritableDocument]`. Companion to the single-document `fileExporter` that already exists. (180301165)

```swift
import SwiftUI

struct ExportPanel: View {
    @State private var showExporter = false
    @State private var snapshots: [ExportableSnapshot] = []

    var body: some View {
        Button("Export All…") { showExporter = true }
            .fileExporter(isPresented: $showExporter,
                          documents: snapshots,
                          contentTypes: [.pdf]) { result in
                if case .success(let urls) = result {
                    print("Exported \(urls.count) files")
                }
            } onCancellation: {
                snapshots.removeAll()
            }
    }
}
```

#### `FileWrapperDocumentWriter.makeFileWrapper(previous:)` — `macOS 27.0+`, Xcode 27 SDK

The closure now receives a `previous: FileWrapper?` parameter so package-format documents can mutate bundles in place rather than always replacing. (180301399)

```swift
struct NotePackage: WritableDocument {
    static let writableContentTypes: [UTType] = [.package]

    var markdown: String
    var attachments: [URL]

    func makeFileWrapper(configuration: WriteConfiguration) throws -> FileWrapper {
        var bundle = configuration.existingFile ?? FileWrapper(directoryWithFileWrappers: [:])
        if let existing = configuration.existingFile {
            bundle = try Self.mutating(existing, with: self)
        } else {
            bundle = try Self.building(with: self)
        }
        return bundle
    }
}
```

#### `\.newDocument` environment value — `macOS 27.0+`, Xcode 27 SDK

Accepts an autoclosured in-memory `ReadableDocument` so "New from Template" flows do not require disk I/O up front. (180300890)

```swift
@main
struct YourApp: App {
    var body: some Scene {
        DocumentGroup(newDocument: { EmptyNote() }) { config in
            EditorView(configuration: config)
                .environment(\.newDocument) { /* template factory */ EmptyNote() }
        }
    }
}
```

### SwiftUI — AsyncImage and selection

#### `AsyncImage` HTTP cache + custom `URLSession` — `macOS 27.0+`, Xcode 27 SDK

`AsyncImage` now respects HTTP caching headers by default. New initializers accept `URLRequest` with a custom `cachePolicy`. New `View.asyncImageURLSession(_:)` environment API sets a custom `URLSession` for every descendant `AsyncImage`. (78212597)

```swift
import SwiftUI

struct AvatarView: View {
    let request: URLRequest

    var body: some View {
        AsyncImage(url: request.url!, transaction: Transaction(animation: .spring)) { phase in
            switch phase {
            case .empty: ProgressView()
            case .success(let image): image.resizable().scaledToFit()
            case .failure: Image(systemName: "person.crop.circle")
            @unknown default: EmptyView()
            }
        }
    }
}

struct ReaderContainer: View {
    var body: some View {
        AvatarView(request: URLRequest(url: avatarURL))
            .asyncImageURLSession(.shared)
    }
}
```

#### Selectable `Text` + `TextRenderer` — `macOS 27.0+`, Xcode 27 SDK

`TextRenderer` content now renders on `Text` views that also have `.textSelection(.enabled)`. (158160386, 151015350)

```swift
import SwiftUI

struct FadingText: View {
    var body: some View {
        Text("Important heading")
            .font(.title)
            .textSelection(.enabled)
            .textRenderer(OpacityRenderer())
    }
}

struct OpacityRenderer: TextRenderer {
    func draw(layout: Text.Layout, in context: inout TextRenderer.Context) {
        for line in layout {
            context.opacity = line.opacity
            context.draw(line)
        }
    }
}
```

### SwiftUI — concurrency and macros

#### `@concurrent` async document methods — `macOS 27.0+`, Xcode 27 SDK

The `FileWrapperDocumentReader` and `FileWrapperDocumentWriter` closures can be marked `@concurrent` to defeat approachable-concurrency's main-actor inference. Custom conforming types should follow the same pattern:

```swift
func reader(configuration: sending ReadConfiguration) -> sending FileWrapperDocumentReader<MySnapshot> {
    FileWrapperDocumentReader(configuration) { @concurrent fileWrapper in
        guard let data = fileWrapper.regularFileContents else {
            throw CocoaError(.fileReadCorruptFile)
        }
        return try MySnapshot.parse(data)
    }
}
```

**NOTE:** There is no `DocumentReader` / `DocumentWriter` protocol in Apple's SwiftUI framework. The actual types are `FileWrapperDocumentReader<Snapshot>` and `FileWrapperDocumentWriter<Snapshot>` — concrete generic types wrapping a closure. If your code references a `DocumentReader` / `DocumentWriter` protocol, that is a project-local abstraction, not Apple API. (180302015)

#### `@MainActor` `DocumentGroup` factory closures — `macOS 27.0+`, Xcode 27 SDK

`makeDocument:` and `makeReadableDocument:` closures on `DocumentGroup` initializers are `@MainActor`-isolated, allowing direct access to `URLDocumentConfiguration` without isolation hops. (180302065)

```swift
@main
struct WorkspaceApp: App {
    var body: some Scene {
        DocumentGroup { @MainActor in
            WorkspaceDocument(initialDraft: "")
        } editor: { config in
            EditorView(configuration: config)
        }
    }
}
```

#### Macro-based `@State` — `macOS 27.0+` / iOS 17 back-deploy

Xcode 27 introduces a macro-based `@State` that does not re-evaluate the initial expression on every view re-instantiation. Two patterns no longer compile:

- Assigning in `init` while also providing a default at the declaration.
- Synthesized private memberwise `init` via an extension when all stored members are private and any is `@State`.

`@State` cannot compose with other property wrappers or macros. (105893279)

```swift
struct CounterView: View {
    @State private var count: Int = 0
    @State private var draft: String

    init(initial: String) {
        _draft = State(initialValue: initial)   // OK: no default at declaration
    }

    var body: some View {
        Stepper("Count: \(count)", value: $count)
    }
}
```

#### `@Entry` macro warning — `macOS 27.0+`

`@Entry` warns if you store default class instances or closures in the environment. Store plain values or use a factory that constructs instances on access. (175902616)

```swift
extension EnvironmentValues {
    @Entry var documentMetrics: DocumentMetrics = DocumentMetrics()
}

@Observable
final class DocumentMetrics {
    var charCount: Int = 0
    var wordCount: Int = 0
}
```

### SwiftUI — menu / label / icon behavior

#### Menu bar images hidden by default — `macOS 27.0+`, Xcode 27 SDK

Menu bar and context menus show no item images by default (reversing Tahoe). Use `labelStyle(.titleAndIcon)` to keep icons visible. (170480710, 170477566, 179374305)

```swift
Menu {
    Button("New", systemImage: "doc.badge.plus") { /* … */ }
    Button("Open…", systemImage: "folder") { /* … */ }
} label: {
    Label("File", systemImage: "doc")
}
.labelStyle(.titleAndIcon)
```

#### `LabeledContent` as menu subtitle — `macOS 27.0+`, Xcode 27 SDK

A `LabeledContent` placed inside a `Menu` maps its value to the platform menu item's subtitle. (175594929)

```swift
Menu("Recent") {
    LabeledContent("Today", value: "scratchpad.md")
    LabeledContent("Yesterday", value: "review.md")
    Divider()
    Button("Show All…") { /* … */ }
}
```

#### `Slider` no longer uses `NSSlider` — `macOS 27.0+`, Xcode 27 SDK

SwiftUI `Slider` implementation switches off `NSSlider`. Customizations that relied on AppKit subclassing will not apply. (173990195)

```swift
struct HueSlider: View {
    @State private var hue: Double = 0.5

    var body: some View {
        Slider(value: $hue, in: 0...1) {
            Text("Hue")
        } minimumValueLabel: {
            Image(systemName: "drop.fill")
        } maximumValueLabel: {
            Image(systemName: "drop.degreesign")
        }
    }
}
```

#### Disabled checkbox toggles no longer tinted — `macOS 27.0+`, Xcode 27 SDK

`Toggle` with `.checkbox` style no longer paints a tint when disabled — enabled vs disabled state is now visually distinct. (172689844)

#### `TabView` appearance in inspectors — `macOS 27.0+`, Xcode 27 SDK

`TabView` shown inside inspectors matches sidebar styling and automatically uses the new `.tabs` picker style. (170678002)

### AppKit — scroll, refresh, toolbar

#### `NSRefreshController` — `macOS 27.0+`, Xcode 27 SDK

Pull-to-refresh controller for `NSScrollView`. Set via `NSScrollView.refreshController`. Has `beginRefreshing()` and `endRefreshing()`. Replaces any custom pull-to-refresh implementation. (160867808)

```swift
let scrollView = NSScrollView()
let controller = NSRefreshController { [weak scrollView] in
    Task {
        await WorkspaceModel.shared.refresh()
        await MainActor.run {
            scrollView?.refreshController?.endRefreshing()
        }
    }
}
scrollView.refreshController = controller
```

#### `NSToolbarItemGroup.role` + `NSSegmentedControl.role` — `macOS 27.0+`, Xcode 27 SDK

Both gain a `role` property with a strongly-typed enum. `NSSegmentedControlRole` includes `.tabs` — VoiceOver reads tab-style segmented controls as "tabs". (162577742)

```swift
let segmented = NSSegmentedControl()
segmented.segmentStyle = .separated
segmented.role = .tabs
segmented.segmentCount = 3
segmented.setLabel("Write", forSegment: 0)
segmented.setLabel("Preview", forSegment: 1)
segmented.setLabel("Diff", forSegment: 2)
```

### AppKit — text view selection

#### `NSTextSelectionManager` — `macOS 27.0+`, Xcode 27 SDK

Selection gestures now route through an `NSTextSelectionManager` (gesture-recognizer-based) rather than `NSEvent` mouse overrides. `NSTextView` uses it internally. Existing `mouseDown:` overrides keep working through a binary-compatible fallback. (163365571)

```swift
import AppKit

final class MarkdownTextView: NSTextView {
    private lazy var linkTap: NSClickGestureRecognizer = {
        let recognizer = NSClickGestureRecognizer(target: self, action: #selector(handleLinkTap(_:)))
        recognizer.numberOfClicksRequired = 1
        return recognizer
    }()

    override func viewDidMoveToWindow() {
        super.viewDidMoveToWindow()
        addGestureRecognizer(linkTap)
    }

    @objc private func handleLinkTap(_ recognizer: NSClickGestureRecognizer) {
        let location = recognizer.location(in: self)
        guard let candidate = candidateListForSelection(at: location) else { return }
        openURL(candidate)
    }
}
```

### AppKit — gesture system

#### `NSGestureRecognizer.cancellableByScrollGesture` — `macOS 27.0+`, Xcode 27 SDK

When `true`, the gesture automatically cancels when the enclosing scroll view starts panning. Useful for click-and-hold / context gestures that should defer to scroll. (165650612)

```swift
let hold = NSPressGestureRecognizer(target: self, action: #selector(handleHold(_:)))
hold.cancellableByScrollGesture = true
view.addGestureRecognizer(hold)
```

#### Exclusive gesture behavior — `macOS 27.0+`, Xcode 27 SDK

By default only the initial hit-tested view hierarchy activates gestures until all terminate. Escape hatches:

- Per-view: `NSView.exclusiveGestureBehavior`.
- App-wide: `Info.plist` key `NSViewGestureRecognizerIsExclusive`.
- `NSGestureRecognizerSuppressesMainMenuActions` allows menu actions to fire during gestures. (173551081)

```swift
inspectorView.exclusiveGestureBehavior = .allowConcurrent

NSClickGestureRecognizer(target: self, action: #selector(handleClick(_:)))
    .setValue(true, forKey: "suppressesMainMenuActions")
```

#### Stuck-gesture timeout — `macOS 27.0+`, Xcode 27 SDK

Auto-cancels gestures that remain active past the timeout. Set `NSCrashOnStuckGestureTimeout` to `true` in user defaults to crash on stuck gestures for debugging. `NSGestureRecognizerCrashOnMissingOverrides` lets you crash when a gesture is missing required overrides. (175705302, 176396492)

```swift
UserDefaults.standard.set(true, forKey: "NSCrashOnStuckGestureTimeout")
UserDefaults.standard.set(true, forKey: "NSGestureRecognizerCrashOnMissingOverrides")
```

#### `NSScrollView` touch constraints — `macOS 27.0+`, Xcode 27 SDK

New properties on `NSScrollView` for constraining the touches-needed-to-scroll threshold plus `scrollGestureForFailureRelationship`. (164924201)

### AppKit — windows and menus

#### `NSApplication.presentationOptions` — `.disableScreenCornerInteractions` — `macOS 27.0+`, Xcode 27 SDK

New option that disables Hot Corners while a window is presented with that option set. (168692527)

```swift
NSApp.presentationOptions = [.disableScreenCornerInteractions, .hideDock]
```

#### `NSTitlebarAccessoryViewController` overdraw — `macOS 27.0+`, Xcode 27 SDK

Title-bar accessory controllers may now draw outside their bounds by default — enabling shadows, glows, and interactive glass. Clipping applies only during reveal animations or when the accessory is `hidden`. (180962967)

```swift
final class StatusPillAccessory: NSTitlebarAccessoryViewController {
    override func loadView() {
        let host = NSView(frame: NSRect(x: 0, y: 0, width: 80, height: 24))
        host.wantsLayer = true
        host.layer?.cornerRadius = 12
        host.layer?.backgroundColor = NSColor.systemBlue.cgColor
        view = host
    }
}
```

#### `NSTextView.menuForEvent:` Layout Orientation — `macOS 27.0+`, Xcode 27 SDK

The Layout Orientation menu item moves into the Font submenu of the context menu for apps linking the macOS 27 SDK. (177605020)

#### `NSMenu` symbol and non-symbol images hidden by default — `macOS 27.0+`, Xcode 27 SDK

For apps linked on the macOS 27 SDK, both symbol and non-symbol menu item images are hidden by default. Use `NSMenuItem.preferredImageVisibility` to keep specific items visible. (170477566, 179374305, 179936632)

```swift
let item = NSMenuItem(title: "Refresh", action: #selector(refresh), keyEquivalent: "r")
item.image = NSImage(systemSymbolName: "arrow.clockwise", accessibilityDescription: "Refresh")
item.preferredImageVisibility = .always
menu.addItem(item)
```

## Code Patterns

### Pattern 1 — Migrating `ReferenceFileDocument` to `Document` + `URLDocumentConfiguration`

Your app's document type currently uses `ReferenceFileDocument`. Under macOS 27 the recommended shape is `Document` with a separate `@Observable` `@MainActor` view-model holding the configuration. The pattern below shows how to keep undo/redo working.

```swift
import SwiftUI
import UniformTypeIdentifiers

struct MarkdownDocument: Document {
    static let readableContentTypes: [UTType] = [.markdown, .plainText]
    static let writableContentTypes: [UTType] = [.markdown]

    var text: String

    init(configuration: ReadConfiguration) throws {
        guard let data = configuration.file.regularFileContents else {
            throw CocoaError(.fileReadCorruptFile)
        }
        self.text = String(decoding: data, as: UTF8.self)
    }

    func fileWrapper(configuration: WriteConfiguration) throws -> FileWrapper {
        let data = Data(text.utf8)
        return FileWrapper(regularFileWithContents: data)
    }
}

@Observable
@MainActor
final class MarkdownViewModel {
    var text: String
    private weak var configuration: URLDocumentConfiguration<MarkdownDocument>?

    init(text: String) {
        self.text = text
    }

    func bind(_ configuration: URLDocumentConfiguration<MarkdownDocument>) {
        self.configuration = configuration
        configuration.didUndoChange = { [weak self] undoManager in
            self?.text = configuration.document.text
            undoManager?.registerUndo(withTarget: self) { vm in
                configuration.didUndoChange = { _ in /* restore */ }
            }
        }
    }
}
```

### Pattern 2 — `NSRefreshController` in a document list

The most common place to use pull-to-refresh is a workspace document list. The controller is created lazily, stored on the scroll view, and torn down with it.

```swift
import AppKit

final class RefreshableDocumentListController: NSViewController {
    private let scrollView = NSScrollView()
    private let tableView = NSTableView()

    override func loadView() {
        scrollView.documentView = tableView
        scrollView.hasVerticalScroller = true
        scrollView.refreshController = NSRefreshController { [weak self] in
            self?.refresh()
        }
        view = scrollView
    }

    private func refresh() {
        Task {
            await WorkspaceModel.shared.reloadFromDisk()
            await MainActor.run { [weak scrollView] in
                scrollView?.refreshController?.endRefreshing()
                tableView.reloadData()
            }
        }
    }
}
```

### Pattern 3 — `NSTextView` + gesture recognizer for clickable links

The project currently overrides `mouseDown:` to detect link clicks. Under macOS 27 the recommendation is to use `NSClickGestureRecognizer` and let `NSTextSelectionManager` handle selection.

```swift
import AppKit

final class ClickableLinkTextView: NSTextView {
    private lazy var linkTap: NSClickGestureRecognizer = {
        let r = NSClickGestureRecognizer(target: self, action: #selector(handleLinkTap(_:)))
        r.numberOfClicksRequired = 1
        r.cancellableByScrollGesture = true
        return r
    }()

    override func viewDidMoveToWindow() {
        super.viewDidMoveToWindow()
        if linkTap.view === nil {
            addGestureRecognizer(linkTap)
        }
    }

    @objc private func handleLinkTap(_ recognizer: NSClickGestureRecognizer) {
        let point = recognizer.location(in: self)
        guard let candidates = candidateListForSelection(at: point) else { return }
        for candidate in candidates {
            if let url = candidate.url {
                NSWorkspace.shared.open(url)
                return
            }
        }
    }
}
```

### Pattern 4 — `@State` macro migration

The new macro-based `@State` does not re-run the initial expression on every view re-instantiation. Existing code that assigned `@State` inside `init` while providing a default value must be rewritten.

```swift
struct DraftEditor: View {
    @State private var draft: String
    @State private var selection: NSRange

    init(initialDraft: String) {
        _draft = State(initialValue: initialDraft)
        _selection = State(initialValue: NSRange(location: 0, length: 0))
    }

    var body: some View {
        TextEditor(text: $draft)
            .onChange(of: draft) { _, newValue in
                selection = NSRange(location: newValue.count, length: 0)
            }
    }
}
```

### Pattern 5 — Menu customization with `preferredImageVisibility`

When the migration requires specific items to keep icons (e.g. toolbar-style entries), use `NSMenuItem.preferredImageVisibility` instead of trying to override the SDK default.

```swift
import AppKit

enum AppMenuFactory {
    static func buildFileMenu() -> NSMenu {
        let menu = NSMenu(title: "File")
        let newItem = NSMenuItem(title: "New", action: #selector(FileActions.new),
                                 keyEquivalent: "n")
        newItem.image = NSImage(systemSymbolName: "doc.badge.plus",
                                accessibilityDescription: "New document")
        newItem.preferredImageVisibility = .always
        menu.addItem(newItem)
        menu.addItem(.separator())
        return menu
    }
}
```

## Migration from Tahoe

### Tahoe (26) → Golden Gate (27) SwiftUI deltas

- **`FileDocument` deprecated** → migrate to `Document` (combined) or `ReadableDocument` + `WritableDocument`. The project currently uses `ReferenceFileDocument` in the document type and app entry point; both should plan for `Document` + a separate `@MainActor` `@Observable` configuration model. (178776840, 180302075)
- **`FileWrapperDocumentWriter.makeFileWrapper` gains `previous:` parameter** — the closure signature changes for any package-format document. Add the new parameter and adopt in-place mutation for bundle packages. (180301399)
- **Macro-based `@State`** — audit for the two broken patterns: (a) `init` assignment plus declaration default, (b) extension-memberwise-`init` synthesis when all stored members are private and any is `@State`. Back-deploys to iOS 17-aligned OSes but compiles against the new Xcode 27 SDK. (105893279)
- **Menu bar images hidden by default** — symbols no longer appear in menu bars / context menus by default. Use `labelStyle(.titleAndIcon)` in SwiftUI or `NSMenuItem.preferredImageVisibility = .always` in AppKit to preserve icons for items that need them. (170480710, 170477566)
- **Selectable `Text` + `TextRenderer`** — previously `TextRenderer` had no effect on selectable text; this is now supported on macOS 27 SDK. (158160386)
- **`Slider` no longer wraps `NSSlider`** — any custom `NSSlider` subclass your app relies on will not apply. Audit custom slider styling. (173990195)
- **`TabView` in inspectors uses `.tabs` picker style automatically** — explicit `.pickerStyle(.tabs)` may no longer be needed. (170678002)
- **`TextInputBorderShape`** — `.squareBorder` / `.roundedBorder` soft-deprecated. Adopt `.bordered`. (173362083)
- **`URLDocumentConfiguration` no longer `Sendable`** — drop `Sendable` constraints from closures or types that captured it. (180302075)
- **`@concurrent` replaces `nonisolated`** on `DocumentReader.read` / `DocumentWriter.write` async methods — same fix should be applied to custom conforming types. (180302015)
- **`@Entry` warning** — if you store default class instances or closures in the environment, refactor to factories or plain values. (175902616)
- **`\.newDocument` environment value** — adopt for "New from Template" flows that should not hit disk up front. (180300890)
- **`TabsPickerStyle`** — adopt for tab-style pickers that should be announced as "tabs" by VoiceOver. (173211711)
- **Data-item / error-object `alert` / `confirmationDialog`** back-deploys to iOS 15 / macOS 12 — adopt on existing deployment targets to get a more flexible signature. (179388848)

### Tahoe (26) → Golden Gate (27) AppKit deltas

- **`NSTextView` selection uses `NSTextSelectionManager`** internally. Existing `mouseDown:` overrides keep working through a binary-compatible fallback but new code should use `NSClickGestureRecognizer` / `NSPressGestureRecognizer` etc. The project has three call sites to audit. (163365571)
- **`NSRefreshController`** — adopt for any custom pull-to-refresh. (160867808)
- **`NSToolbarItemGroup.role` / `NSSegmentedControl.role`** — set `.tabs` on segmented controls that should be announced as tabs by VoiceOver. (162577742)
- **`NSApplication.presentationOptions.disableScreenCornerInteractions`** — opt out of Hot Corners while presenting a window that requires them disabled. (168692527)
- **Exclusive gesture behavior** — opt-out via `NSView.exclusiveGestureBehavior`, `Info.plist` `NSViewGestureRecognizerIsExclusive`, or `NSGestureRecognizerSuppressesMainMenuActions`. (173551081)
- **Stuck-gesture timeout** — set `NSCrashOnStuckGestureTimeout` user default during development. (175705302)
- **`NSGestureRecognizerCrashOnMissingOverrides`** — diagnostic user default for missing-required-overrides crashes. (176396492)
- **`NSTitlebarAccessoryViewController` overdraw** — by default, accessories may draw outside their bounds. (180962967)
- **`NSTextView.menuForEvent:` Layout Orientation** — Layout Orientation menu item moves into Font submenu for apps linking the macOS 27 SDK. (177605020)
- **`NSMenu` images hidden by default** — both symbol and non-symbol images hidden by default; use `NSMenuItem.preferredImageVisibility` to keep specific items visible. (170477566, 179374305, 179936632)
- **`NSScrollView` touch constraints** — new properties for constraining touches-needed-to-scroll + `scrollGestureForFailureRelationship`. (164924201)

## Common Mistakes

### Mistake 1 — Continuing to assign `@State` in `init` while declaring a default

The new macro-based `@State` does not re-evaluate the initial expression, and assigning in `init` while also providing a default at the declaration no longer compiles.

```swift
// Wrong — fails to compile under Xcode 27.
struct Counter: View {
    @State private var count: Int = 0

    init(initial: Int) {
        count = initial   // assignment in init + default at declaration
    }
}

// Right.
struct Counter: View {
    @State private var count: Int

    init(initial: Int) {
        _count = State(initialValue: initial)
    }
}
```

### Mistake 2 — Capturing `URLDocumentConfiguration` across actor boundaries

`URLDocumentConfiguration` is `@MainActor` and no longer `Sendable`. Treating it as `Sendable` and crossing an actor boundary with it will not compile under strict concurrency.

```swift
// Wrong.
@MainActor
func bind(_ configuration: URLDocumentConfiguration<Doc>) {
    Task.detached {
        configuration.didUndoChange = { _ in /* … */ }   // crossing actor
    }
}

// Right.
@MainActor
func bind(_ configuration: URLDocumentConfiguration<Doc>) {
    configuration.didUndoChange = { _ in /* … */ }   // already on main actor
}
```

### Mistake 3 — Forgetting `previous: FileWrapper?` in `makeFileWrapper`

Package-format documents must mutate the previous file wrapper in place; ignoring the new parameter produces bundle churn on every save and breaks macOS document browsers' "edited by" tracking.

```swift
// Wrong.
func makeFileWrapper(configuration: WriteConfiguration) throws -> FileWrapper {
    let bundle = FileWrapper(directoryWithFileWrappers: [:])
    try Self.populate(bundle, with: self)
    return bundle
}

// Right.
func makeFileWrapper(configuration: WriteConfiguration) throws -> FileWrapper {
    let bundle = configuration.existingFile ?? FileWrapper(directoryWithFileWrappers: [:])
    if let previous = configuration.existingFile {
        try Self.mutate(previous, with: self)
    } else {
        try Self.populate(bundle, with: self)
    }
    return bundle
}
```

### Mistake 4 — Overriding `mouseDown:` for new selection behaviors

Under macOS 27, `NSTextView` selection is gesture-recognizer-based. New selection behaviors should not override `mouseDown:` — they should install an `NSClickGestureRecognizer` (or `NSPressGestureRecognizer`) on the text view. `mouseDown:` remains valid for legacy behavior via the binary-compat fallback, but new code will not compose with `NSTextSelectionManager`.

```swift
// Wrong.
final class CustomTextView: NSTextView {
    override func mouseDown(with event: NSEvent) {
        super.mouseDown(with: event)
        // Custom selection behavior here will not compose with NSTextSelectionManager.
    }
}

// Right.
final class CustomTextView: NSTextView {
    private lazy var recognizer: NSClickGestureRecognizer = {
        NSClickGestureRecognizer(target: self, action: #selector(handleClick(_:)))
    }()

    override func viewDidMoveToWindow() {
        super.viewDidMoveToWindow()
        addGestureRecognizer(recognizer)
    }
}
```

### Mistake 5 — Leaving menu item images visible when targeting both Tahoe and Golden Gate

Menu images are visible by default under Tahoe and hidden by default under Golden Gate. Using a single `Menu { Button("Open", systemImage: "folder") { } }` declaration will render with the icon on Tahoe and without on Golden Gate. If the icon matters, force visibility with `.labelStyle(.titleAndIcon)`.

```swift
// Inconsistent — image disappears under Golden Gate.
Menu("File") {
    Button("Open…", systemImage: "folder") { /* … */ }
}

// Right — explicit icon policy.
Menu("File") {
    Button("Open…", systemImage: "folder") { /* … */ }
        .labelStyle(.titleAndIcon)
}
```

## References

- [macOS 27 release notes — SwiftUI](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes)
- [macOS 27 release notes — AppKit](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes)
- [What's new in macOS 27](https://developer.apple.com/macos/whats-new/)
- [SwiftUI framework documentation](https://developer.apple.com/documentation/swiftui)
- [Apple Design Resources — macOS 27](https://developer.apple.com/design/resources/)
- [WWDC26 269 — What's new in SwiftUI](https://developer.apple.com/videos/play/wwdc2026/269/)
- [WWDC26 272 — Use SwiftUI with AppKit and UIKit](https://developer.apple.com/videos/play/wwdc2026/272/)
- [WWDC26 321 — Dive into lazy stacks and scrolling with SwiftUI](https://developer.apple.com/videos/play/wwdc2026/321/)
- [WWDC26 322 — Compose advanced graphics effects with SwiftUI](https://developer.apple.com/videos/play/wwdc2026/322/)
- [Xcode 27 release notes](https://developer.apple.com/documentation/Xcode-Release-Notes/xcode-27-release-notes)
- [Rosetta deprecation announcement](https://developer.apple.com/news/?id=w5ngl9k2)
- [swift-system SYS-0006 — stat APIs](https://github.com/apple/swift-system/blob/main/Proposals/0006-system-stat.md)
- [swift-system SYS-0008 — backdeploy / cinterop stat](https://github.com/apple/swift-system/blob/main/Proposals/0008-backdeploy-cinterop-stat.md)
