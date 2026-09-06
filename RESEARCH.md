# macOS 27 "Golden Gate" APIs — Research Findings

**Compiled:** 2026-09-04
**Sources:** Apple Developer release notes, Apple Newsroom, WWDC26 pages, Wikipedia (for high-level context only). Every API claim below is backed by an Apple source URL listed in the Sources section.
**Build under study:** macOS 27 Golden Gate Beta 8 (26A5425a, August 31, 2026), Xcode 27 Beta 6.

> **GM 复核条款（2026-09-06 订立）**：macOS 27 / Xcode 27 GM 发布后 14 天内，完成一次全量复核：
> ① §2 全部 API 条目对照 GM release notes 与本机 GM swiftinterface；② §4 Sources 逐条核对；③ modules/*.md 抽样。
> 本文件所有断言仅代表 Beta 8 / Beta 6 时点状态，GM 后以 GM 材料为准。与 SKILL.md「GM 触发条款」对应；
> frontmatter `review_by: 2026-10-31` 仅为兜底。

---

## 1. Release metadata

| Field | Value | Source |
|---|---|---|
| Marketing name | macOS 27 Golden Gate | [macOS 27 release notes](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes) |
| Internal build (beta 8) | 26A5425a | [macOS 27 release notes](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes) |
| Announced | June 8, 2026 (WWDC26) | [Apple Developer news](https://developer.apple.com/news/?id=8rgqj83s) |
| First developer beta | June 8, 2026 | [Wikipedia: macOS Golden Gate](https://en.wikipedia.org/wiki/MacOS_Golden_Gate) |
| Most recent beta as of report | Beta 8 — Aug 31, 2026 | [macOS 27 release notes](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes) |
| Public beta 4 | Aug 17, 2026 | [Wikipedia: macOS Golden Gate](https://en.wikipedia.org/wiki/MacOS_Golden_Gate) |
| Expected release | "Fall 2026" / late 2026 | [Rosetta deprecation news](https://developer.apple.com/news/?id=w5ngl9k2) |
| Required SDK / Xcode | Xcode 27 (Swift 6.4) | [Xcode 27 release notes](https://developer.apple.com/documentation/Xcode-Release-Notes/xcode-27-release-notes) |
| Architectures supported | Apple silicon (ARM64) only | [Rosetta deprecation news](https://developer.apple.com/news/?id=w5ngl9k2) |
| Last release to support Rosetta 2 | Yes — full Rosetta functionality | [Rosetta deprecation news](https://developer.apple.com/news/?id=w5ngl9k2) |
| Will run Rosetta Intel binaries | Yes, on Apple silicon Macs | [Rosetta deprecation news](https://developer.apple.com/news/?id=w5ngl9k2) |
| Will natively run Intel binaries | No — Intel-only apps no longer run on Apple silicon | [Rosetta deprecation news](https://developer.apple.com/news/?id=w5ngl9k2) |

**Deprecation summary (one-line):** Apple drops Intel support on the OS itself, makes Golden Gate the last version with Rosetta 2, and pre-announces that all Intel-based software (excluding legacy games) will be incompatible with macOS 28.0.

---

## 2. New APIs grouped by category

### 2.1 SwiftUI

Source: [macOS 27 release notes — SwiftUI](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes)

**New views / protocols / patterns**
- `ReadableDocument` and `WritableDocument` protocols supporting async read/write, progress reporting, and direct URL access — preferred over `ReferenceFileDocument`. (158441552)
- `Document` protocol combines `ReadableDocument` + `WritableDocument`; `FileDocument` is now deprecated, `ReferenceFileDocument` is superseded. (177458781, 178776840)
- `URLDocumentConfiguration` is `@MainActor`-isolated `@Observable` reference type. (180302075)
- `TextInputBorderShape` type + `textInputBorderShape(_:)` modifier; `.squareBorder` / `.roundedBorder` are soft-deprecated in favor of `.bordered`. (173362083)
- `TabsPickerStyle` for pickers representing tab-based navigation; VoiceOver reads as "tabs". (173211711)
- `concentricCornerRadii` / `concentricCornerRadii(in:)` on `GeometryProxy`. (177185166)
- New `fileExporter(isPresented:documents:contentTypes:onCompletion:onCancellation:)` for batch-exporting `WritableDocument` values. (180301165)
- `FileWrapperDocumentWriter.makeFileWrapper` now receives a `previous: FileWrapper?` parameter so package documents can mutate in place. (180301399)
- Data-item / error-object based `alert` and `confirmationDialog` modifiers back-deploy to iOS 15 / macOS 12. (179388848)
- `\.newDocument` environment value accepts an autoclosured in-memory `ReadableDocument` for "New from Template" flows. (180300890)

**AsyncImage changes**
- `AsyncImage` now respects HTTP caching headers by default. New initializers accept `URLRequest` with custom `cachePolicy`. New `View.asyncImageURLSession(_:)` API to set a custom `URLSession` for all child `AsyncImage` views. (78212597)

**Selectable text + TextRenderer**
- Selectable `Text` views now support `TextRenderer` in apps built with macOS 27 SDK. (158160386)
- Custom `TextRenderer` applied via `.textRenderer(_:)` now takes effect on a `Text` view that also has `.textSelection(.enabled)`. (151015350)

**Menu / label / icon behavior**
- Menu bar and context menus present a reduced set of menu item images by default (reversing Tahoe's behavior). Use `labelStyle(.titleAndIcon)` to keep icons visible. (170480710)
- `LabeledContent` inside a `Menu` maps its value to the platform menu item's subtitle. (175594929)
- Bordered `Menu` and `Picker` buttons no longer use `NSPopUpButton` internally; better label customization. (68559433)
- Disabled `Toggle` using `.checkbox` style is no longer tinted, for clearer enabled/disabled distinction. (172689844)
- `Slider` implementation no longer uses `NSSlider`. (173990195)
- `TabView` appearance in inspectors matches sidebars; both contexts automatically use the new `.tabs` picker style. (170678002)

**@State macro (behavior change, back-deploys to iOS 17)**
- Xcode 27 introduces a macro-based `@State` that does not re-evaluate the initial expression on every view re-instantiation. (105893279)
- Breaking: assigning in an `init` while also providing a default at declaration no longer compiles.
- Breaking: the synthesized private `init` for all-private-stored-members structs is disabled when `@State` is used.
- Cannot compose `@State` with other property wrappers or macros.

**@Entry / environment**
- `@Entry` macro now warns of potential issues if you store default class instances or closures in the environment. (175902616)

**Concurrency refinements (SwiftUI-side)**
- `DocumentReader.read(from:progress:)` and `DocumentWriter.write(snapshot:to:previous:progress:)` are declared with `@concurrent` instead of `nonisolated` (off-main correctness fix for approachable-concurrency). (180302015)
- `makeDocument:` and `makeReadableDocument:` closures on `DocumentGroup` initializers are `@MainActor`-isolated. (180302065)
- `@ContentBuilder` type-checking performance is further improved vs. Beta 1. (177526032)

### 2.2 AppKit

Source: [macOS 27 release notes — AppKit](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes)

- **`NSRefreshController`** — new pull-to-refresh API for `NSScrollView`. Set via `NSScrollView.refreshController`. Has `beginRefreshing()` / `endRefreshing()`. (160867808)
- **`NSToolbarItemGroup.role`** + `NSToolbarItemGroupRole` enum. **`NSSegmentedControl.role`** + `NSSegmentedControlRole` enum (including `.tabs`). VoiceOver reads tab-style segmented controls as "tabs". (162577742)
- **`NSTextSelectionManager`** — text-selection gesture handling (`NSGestureRecognizer`-based) rather than `NSEvent` mouse overrides. `NSTextView` now uses this internally. `NSTextView` subclasses that override `mouseDown:` continue to work via a binary-compatible fallback. (163365571)
- **`NSApplication.presentationOptions`** gains `.disableScreenCornerInteractions` (disables Hot Corners). (168692527)
- **`NSGestureRecognizer.cancellableByScrollGesture`** — gesture auto-cancels when enclosing scroll view is panned. (165650612)
- **Exclusive gesture behavior** — only the initial hit-tested view hierarchy activates gestures until all terminate. Opt-out via `NSView.exclusiveGestureBehavior`, app-wide `Info.plist` key `NSViewGestureRecognizerIsExclusive`, and `NSGestureRecognizerSuppressesMainMenuActions` to allow menu actions during gestures. (173551081)
- **Stuck-gesture timeout** — auto-cancels stuck gestures; `NSCrashOnStuckGestureTimeout` user default lets you crash to debug. (175705302)
- **Diagnostic user default** `NSGestureRecognizerCrashOnMissingOverrides`. (176396492)
- **`NSScrollView`** new properties for constraining touches-needed-to-scroll + `scrollGestureForFailureRelationship`. (164924201)
- **`NSTitlebarAccessoryViewController`** now allowed to draw outside bounds by default (shadows, interactive glass). Clipped only during reveal animations or when `hidden`. (180962967)
- **`NSTextView.menuForEvent:`** — Layout Orientation menu item moves into the Font submenu of the context menu (applies only when linking on macOS 27 SDK). (177605020)
- **`NSMenu`** — both symbol and non-symbol menu item images are now hidden by default for apps linked on macOS 27 SDK. `NSMenuItem.preferredImageVisibility` lets you keep specific items visible. (179374305, 170477566, 179936632)
- Open/save panel Recents list accessible via ⌘⇧F; Move & Resize / Full Screen Tile menu items apply to non-sheet panels. (120442314, 150791154)

### 2.3 Foundation / FoundationModels

Source: [macOS 27 release notes — Foundation](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes), [macOS 27 What's New](https://developer.apple.com/macos/whats-new/)

- **System Swift APIs for `stat`/`lstat`/`fstat`/`fstatat`** — new `Stat` type with `init` from `FilePath`, `FileDescriptor`, or C string; `FilePath.stat()` / `FileDescriptor.stat()` instance methods; supporting types `FileType`, `FileMode`, `FileFlags`, `UserID`, `GroupID`, `DeviceID`, `Inode`. Backed by [swift-system SYS-0006](https://github.com/apple/swift-system/blob/main/Proposals/0006-system-stat.md). (160612181)
- **Foundation URL fix** — `+[NSURL URLWithString:]` no longer double-encodes `%` of valid percent-escape sequences. (161588649)

**FoundationModels framework** ([WWDC26 241 What's new in the Foundation Models framework](https://developer.apple.com/videos/play/wwdc2026/241/))
- Foundation Models framework can now work with **any language model** — Apple Foundation Models, cloud providers (Claude, Gemini), or any provider that conforms to the new **`Language Model` protocol**.
- **Multimodal prompts** — pass images alongside text; the model reasons about visual content on-device.
- **Vision framework tools** — OCR and barcode readers are available to the model as callable tools, all on-device.
- **Dynamic Profiles** — swap models, tools, and instructions within a continuous session.
- Small Business Program apps with < 2M first-time downloads can access next-gen Apple Foundation Models on Private Cloud Compute at no cloud API cost.
- New **Evaluations framework** for verifying AI features across dynamic conditions ([WWDC26 298 Meet the Evaluations framework](https://developer.apple.com/videos/play/wwdc2026/298/)).
- `fm` CLI and Python SDK for AI-powered scripts ([WWDC26 334](https://developer.apple.com/videos/play/wwdc2026/334/)).
- Fixed: `@Generable` on an `enum` produces a deprecation warning about `GenerationError` that cannot be silenced. (177899620)
- Fixed: `PrivateCloudComputeLanguageModel` always uses greedy decoding. (178181782)
- Fixed: `onPrompt` may not be called when applied to a `Profile` without instructions; transcript truncation via `onPrompt` can cause unexpected runtime errors. (177901494, 177902488)

### 2.4 Concurrency / Swift 6

Source: [macOS 27 release notes](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes), [Xcode 27 release notes](https://developer.apple.com/documentation/Xcode-Release-Notes/xcode-27-release-notes)

- **Swift 6.4** ships with Xcode 27. (Xcode 27 release notes)
- **SE-0508 source break** — computed property with both an `init` accessor and an array/dictionary literal initial value no longer compiles if the getter is declared before the `init` accessor. Workaround: swap order. (180969028)
- New **`@concurrent` attribute for off-main async methods** — `DocumentReader`/`DocumentWriter` async methods use `@concurrent` instead of `nonisolated` to defeat approachable-concurrency's main-actor inference. (180302015)
- **`DocumentGroup` factory closures** are `@MainActor`-isolated; closures can access `URLDocumentConfiguration` without isolation hops. (180302065)
- `URLDocumentConfiguration` is `@MainActor`-isolated `@Observable` reference type and no longer conforms to `Sendable`. (180302075)
- **LLDB** can now inspect data types with `~Copyable` fields in standard library and system frameworks (Xcode 27 Beta 4). (176282041)
- **LLDB** ships with an MCP server (`lldb-mcp`) (Xcode 27 Beta 2). (176901842)
- **LLDB** `language swift task tree` command for inspecting Swift tasks (Xcode 27 Beta 1). (169471480)
- **Swift Concurrency Tasks / Executors Instruments** now capture `TaskExecutor` and `SerialExecutor` types on iOS 27 / macOS 27 and later. (171189428)
- **SwiftData** — fixed deadlock for `@Query` when saving a `ModelContext` on a background actor while scheduling new async tasks for a `ModelActor`. (178113288)

### 2.5 Liquid Glass / Design system (refinements over Tahoe)

Sources: [macOS 27 release notes](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes), [macOS 27 What's New](https://developer.apple.com/macos/whats-new/), [Apple Design Resources for macOS 27](https://developer.apple.com/design/resources/)

- Liquid Glass design language is **refined, not replaced** — Apple design kits for macOS 27 include "Updates to Liquid Glass". ([Apple Developer news e2lxw9l1](https://developer.apple.com/news/?id=e2lxw9l1))
- **Dark Mode for macOS** is added to Liquid Glass design kits.
- **Menu bar icons**: revert to the pre-Tahoe behavior of showing no icons by default. Menu item images (symbol and non-symbol) are hidden by default for apps linked on macOS 27 SDK. Use `NSMenuItem.preferredImageVisibility` to override. (170477566, 179374305)
- **Window borders / traffic lights**: Apple redesigned traffic-light window controls and unified window border shapes (Apple design kits page). Wikipedia sources this to 9to5Mac beta 6 coverage but no developer-facing API is named.
- **Design kit naming changes** better align with code.
- **Borders accessibility setting** ("Show borders") now shows borders on section backgrounds in `.grouped` form style. (155906280)
- **`concentricCornerRadii`** on `GeometryProxy` lets you drive custom drawing that matches a `ConcentricRectangle`'s corner radii. (177185166)
- **Icon Composer 2.0** supports a sharper rendering mode for upcoming 2027 OS releases with refractivity, outside specular, and deeper shadows. (172404678)

### 2.6 MLX / Machine Learning — Core AI

Source: [macOS 27 What's New — Core AI](https://developer.apple.com/macos/whats-new/), [macOS 27 release notes — Core AI](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes)

The headline addition is a brand-new framework called **Core AI** (built directly into the OS, purpose-built for Apple Silicon):

- Modern, memory-safe Swift API to **load, specialize, and run AI models entirely on-device**.
- Models are **automatically specialized for the hardware they run on** with ahead-of-time compilation for quick load times.
- **Fine-grained control over inference memory**, zero-copy data paths, and **stateful execution**.
- Runs everything from compact vision models to large-scale generative AI.
- WWDC26 sessions: [Meet Core AI](https://developer.apple.com/videos/play/wwdc2026/324/), [Core AI model authoring and optimization](https://developer.apple.com/videos/play/wwdc2026/325/), [Integrate on-device AI models](https://developer.apple.com/videos/play/wwdc2026/326/), [Optimize custom ML operations with Metal tensors](https://developer.apple.com/videos/play/wwdc2026/330/).

**Core AI operational changes**
- **Neural Engine background access restricted** — same restriction model as GPU. Access in the background requires new entitlement: `com.apple.developer.background-tasks.continued-processing.inference`. (179282606)
- Large model loading (> 1 GB) performance is improved on the Neural Engine.
- Neural Engine memory usage now attributed to your app process (appears in Allocations instrument). (174796039)
- Many fixed model-loading/execution issues: app-group support (179732320), AOT with Xcode 27 Beta 2 (181264112), on-device specialization cache policy (169746264), inference encode blocking (175789258), control flow on dynamic-shape tensors (177354777), Metal API Validation failures (177991751), custom Metal kernels failing to load (178056451).
- The Foundation Models Instrument in Instruments helps trace and debug Foundation Models usage — instructions, prompts, responses, token usage, inference performance. (164223804, Xcode 27 release notes)

> **Note on MLX:** The macOS 26 guidance referenced MLX as a third-party array framework; the macOS 27 What's New page describes **Core AI** as the on-device ML story, with WWDC26 session 330 covering Metal tensor optimizations. The release notes do not contain a separate "MLX" section. Treat Core AI as the successor story for on-device model integration. MLX (Apple's open-source array framework) may continue as a third-party / open-source option — that is outside Apple's release notes and is not cited here.

### 2.7 Continuity / cross-device

Source: [macOS 27 What's New — Spatial Preview](https://developer.apple.com/macos/whats-new/), [macOS 27 release notes](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes)

- **macOS Spatial Preview framework** — preview spatial content from a Mac directly on Apple Vision Pro, collaborate via SharePlay.
- Connects your Mac app to Quick Look on visionOS for spatial photos, Apple Immersive Video, 3D content with live USD editing.
- iPhone Mirroring app supports window resizing. (Wikipedia; Apple design kit reference)

### 2.8 App Intents / Spotlight

Source: [macOS 27 release notes — App Intents, Core Spotlight](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes), [macOS 27 What's New — App Intents](https://developer.apple.com/macos/whats-new/)

- **Entity schemas** contribute app content to the Spotlight semantic index; Siri surfaces content with attribution back to your app.
- **Intent schemas** let people take action via natural language without defined phrases.
- **View Annotations API** — map views to entities so users can reference and act on what's on screen conversationally.
- **App Intents Testing framework** — validate the entire integration through real system pathways without UI automation ([WWDC26 295](https://developer.apple.com/videos/play/wwdc2026/295/)).
- **LLM search via Core Spotlight** ([WWDC26 246](https://developer.apple.com/videos/play/wwdc2026/246/)).
- `SpotlightSearchTool.Configuration(sources: [.coreSpotlight], guide: .focused())` is the **required** form when used with `LanguageModelSession` backed by the on-device system model (default config exceeds on-device context window). Available guides: `.focused(.communications)`, `.focused(.calendar)`, `.focused(.documents)`, `.focused(.visualMedia)`, `.focused(.audio)`. (183770678)
- `notes.createNote` and `notes.updateNote` now accept `name` of type `AttributedString`. (173431080)
- `calendar.deleteEvents` schema renamed to `calendar.deleteEvent`. (176751155)
- `AppEntity` cumulative size limit of **10 MB** including all child properties and values. (181763422)
- Existing entities conforming to `@AppEntity(schema: .photos.asset)` may fail to compile because of new properties added to the schema in this release. Workaround: gate behind availability check. (181800016)

### 2.9 WebKit

Source: [macOS 27 What's New — WebKit for Safari 27](https://developer.apple.com/macos/whats-new/), [macOS 27 release notes — Safari](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes)

- WebKit for Safari 27: **over 1,000 browser engine improvements**.
- **Grid Lanes** and **Customizable Select** for layout and form controls.
- **HTML `<model>` element** and **Immersive Environments** bring spatial / immersive content natively to the web.
- Safari web extensions can now be built and tested with Xcode Cloud, no Mac required.
- Previews fix: code using WebKit previously failed because a Swift overlay library couldn't be found (resolved). (179652645)
- Safari Intelligence features might appear available before assets finish downloading — wait for assets. (178099724)
- Safari Extensions: unsigned extensions might not appear until rebuilt. (183044008)

### 2.10 Process / subprocess

The macOS 27 release notes do **not** have a dedicated "Process" or "subprocess" section. There are no first-party Apple changes to `Process`, `NSTask`, or the Swift `Subprocess` package announced in the macOS 27 release notes.

However, the related ecosystem changes worth flagging:
- **System `stat`/`lstat`/`fstat`/`fstatat` Swift APIs** (160612181) — useful for subprocess operations that inspect output files.
- **`NSCrashOnStuckGestureTimeout`** user default in AppKit (175705302) — not directly related, but indicates macOS 27 ships additional debug-friendly user defaults.
- **`launchd`** no longer supports loading property list files with the quarantine extended attribute. (166415497)

### 2.11 Vision / OCR

No dedicated "Vision" framework section appears in the macOS 27 release notes. The noteworthy adjacent item is:

- **Foundation Models' multimodal prompts + Vision framework tools** — OCR and barcode readers are exposed as on-device tools that Foundation Models can call directly. ([macOS 27 What's New](https://developer.apple.com/macos/whats-new/))

There is no Apple-published Vision framework release-note change list that could be fetched — the macOS release notes do not include a Vision-specific section.

### 2.12 Other frameworks

- **Background Assets** — localized asset packs delivered based on user's preferred languages. (163944365)
- **DiskImageKit** — new Swift API framework for creating/managing standalone and stacked disk images in ASIF and raw formats; integrates with Virtualization framework's `VZDiskImageStorageDeviceAttachment`. (177868758)
- **RealityKit / USDKit** — Spatial Preview uses RealityKit for USD rendering in Preview/Quick Look (replacing Storm). (176839273) Gaussian Splat Component API "will be available in an upcoming release". (178061856)
- **CryptoKit / Swift Charts** — Swift Charts: `_ConditionalContent` inside a `Chart` closure produced a warning on older minimum deployment targets; fixed. (174168981)
- **StoreKit** — Advanced Commerce API gains `partnerName` and `partnerId`; subscription Bundles and Suites via `Product.ProductType`; offer code redemption returns `VerificationResult`. (167808780, 160501742, 141012819, 156749517)
- **Network Security** — select system processes enforce stricter TLS requirements starting in 27.0 (MDM, DDM, ADE, configuration profile installation, app installation, software updates). Servers must support TLS 1.2 minimum with ATS-compliant ciphers/certs. (176055825)
- **Automatic Assessment Configuration** — flexible / granular config with controls for Dock, Menu Bar, accessibility settings, plus system pre-checks and app launch restrictions. (158132137)

---

## 3. Migration highlights from Tahoe (26) → Golden Gate (27)

1. **Intel / Rosetta transition** — final macOS release with full Rosetta 2; subsequent releases drop it. Audit any third-party frameworks still building Intel-only or as universal binaries. ([Rosetta deprecation news](https://developer.apple.com/news/?id=w5ngl9k2))
2. **`@State` macro rewrite** — Xcode 27's macro-based `@State` back-deploys to iOS 17-aligned OSes, but breaks two patterns: (a) assigning `@State` in `init` while also providing a default value at declaration; (b) using the synthesized private memberwise init via an extension when all stored members are private and any of them is `@State`. ([SwiftUI](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes))
3. **`FileDocument` is deprecated**, prefer `Document` / `ReadableDocument` / `WritableDocument`. `ReferenceFileDocument` remains available. ([SwiftUI](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes))
4. **`FileWrapperDocumentWriter.makeFileWrapper`** now receives `previous: FileWrapper?` — closure signature change. ([SwiftUI](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes))
5. **Menu bar / context menu images hidden by default** — SwiftUI hides symbol menu item images by default; AppKit hides both symbol and non-symbol by default for macOS 27 SDK links. Use `NSMenuItem.preferredImageVisibility` or `labelStyle(.titleAndIcon)` to opt back in. ([SwiftUI](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes), [AppKit](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes))
6. **`NSTextSelectionManager` in NSTextView** — `NSTextView` now uses gesture-recognizer-based selection internally. Existing `mouseDown:` overrides keep working via binary-compatible fallback, but new code should use `NSGestureRecognizer` subclasses. ([AppKit](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes))
7. **NSRefreshController** — new pull-to-refresh in `NSScrollView`. If you have custom pull-to-refresh, evaluate replacement. ([AppKit](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes))
8. **App Intents / `@AppEntity(schema: .photos.asset)`** conformance may break due to new schema properties — gate behind `if #available` checks. ([App Intents](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes))
9. **`AppEntity` 10 MB cumulative size limit** (181763422) — relevant if you index many workspace entities.
10. **`@Generable` enum deprecation warning about `GenerationError`** cannot be silenced — only fixed by the new SDK behavior. (177899620)
11. **System Swift APIs for `stat` family** — unqualified `stat()` calls may collide with new `FilePath.stat()` / `FileDescriptor.stat()` instance methods if your code uses unqualified `Darwin.stat` calls. See [SYS-0008](https://github.com/apple/swift-system/blob/main/Proposals/0008-backdeploy-cinterop-stat.md). (177911316)
12. **`@concurrent` vs `nonisolated`** — `DocumentReader`/`DocumentWriter` async methods changed from `nonisolated` to `@concurrent`. Same fix should be applied to your own conforming types. (180302015)
13. **Liquid Glass refinements** — design kits now include Dark Mode for macOS, refreshed materials, refined typography. Existing Liquid Glass components continue to work but the visual treatment may shift. ([Apple design resources](https://developer.apple.com/design/resources/))
14. **`@concurrent` attribute availability** — check that custom concurrency helpers in the codebase use `@concurrent` instead of `nonisolated` when off-main work is intended, given Swift 6.4 + approachable-concurrency defaults inferring `MainActor`. (180302015)
15. **Exclusive gesture behavior default** — by default only the initial hit-tested hierarchy activates gestures until all terminate. New escape hatches via `NSView.exclusiveGestureBehavior`, `Info.plist` keys. ([AppKit](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes))

---

## 4. Breaking changes / deprecations

### Removed frameworks / features

- **`DVDPlayback` framework removed from macOS 27 SDK**. (179136240)
- **`ImageCreator` class discontinued** — no longer works in iOS 27, iPadOS 27, macOS 27, visionOS 27 or later. ([Apple Developer news](https://developer.apple.com/news/?id=dz9wvq0r))
- **AFP (Apple Filing Protocol)** removed — Time Machine backups to AirPort Time Capsule no longer supported. ([Wikipedia: macOS Golden Gate](https://en.wikipedia.org/wiki/MacOS_Golden_Gate))
- **Boot Camp removed** — Golden Gate no longer supports Intel CPUs. ([Wikipedia: macOS Golden Gate](https://en.wikipedia.org/wiki/MacOS_Golden_Gate))
- **AirPort Utility no longer included with new installs** — still downloadable; not guaranteed to work. ([macOS 27 release notes](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes))
- **TCC direct database access** — apps can no longer access the local TCC database directly. (90775556)
- **`PHAssetResource.originalFilename`** incorrectly marked non-nullable; **deprecated** in favor of new nullable `filename`. (175412725)
- **`fileImporter` / `document`-based Scenes** — `FileDocument` deprecated in favor of `Document`. (178776840)
- **UIKit apps without scene-based lifecycle fail to launch** when linked against latest SDK. (141837548)
- **`PreviewProvider`** deprecated in Xcode 27 Beta 3. (144168701, Xcode 27 release notes)

### Future-looking deprecations

- **Intel-based applications** that will no longer run in macOS 28.0 now display a treatment in Get Info. (169548657)
- **All Intel-based software** will no longer be compatible with macOS 28.0, excluding legacy games. (176042635)
- **Rosetta 2** ends with macOS 27 for general use; legacy games continue. ([Rosetta deprecation news](https://developer.apple.com/news/?id=w5ngl9k2))
- **`ImageCreator`** — already removed. ([Apple Developer news](https://developer.apple.com/news/?id=dz9wvq0r))
- **Encrypted HFS+ (CoreStorage)** deprecated; transition to encrypted APFS. (175892420)
- **System / `stat` Swift APIs** deprecate unqualified `Darwin.stat` calls in extensions on `FilePath` / `FileDescriptor`. ([SYS-0008](https://github.com/apple/swift-system/blob/main/Proposals/0008-backdeploy-cinterop-stat.md))

### What replaces it (mapping)

| Deprecated | Replacement |
|---|---|
| `FileDocument` | `Document` (combined) or `ReadableDocument` + `WritableDocument` (158441552, 177458781, 178776840) |
| `ReferenceFileDocument` | `Document` for common read-and-write cases |
| `NSSlider` (used by SwiftUI `Slider`) | New `Slider` implementation that no longer uses `NSSlider` (173990195) |
| `NSMenu` auto-showing symbol/non-symbol images | `NSMenuItem.preferredImageVisibility` (170477566) |
| Pull-to-refresh DIY | `NSRefreshController` on `NSScrollView` (160867808) |
| `originalFilename` on `PHAssetResource` | `filename` (nullable) (175412725) |
| `previewingContext` / `PreviewProvider` | `#Preview` macro (Xcode 27 Beta 3) |
| Image generation via `ImageCreator` | Image Playground framework |
| Time Machine to AFP / Time Capsule | SMB / network drive to APFS volume |
| Direct TCC DB access | Public TCC APIs |

---

## Sources

All claims above are backed by one or more of the following Apple Developer URLs (or Apple's public Apple Newsroom/News posts):

**Primary Apple sources**
- macOS 27 Beta 8 release notes: https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes
- macOS 27 Beta 8 release notes (Markdown): https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes.md
- What's new in macOS 27: https://developer.apple.com/macos/whats-new/
- Xcode 27 Beta 6 release notes: https://developer.apple.com/documentation/Xcode-Release-Notes/xcode-27-release-notes
- Xcode 27 Beta 6 release notes (Markdown): https://developer.apple.com/documentation/Xcode-Release-Notes/xcode-27-release-notes.md
- Rosetta deprecation announcement (Apple News, Sep 1, 2026): https://developer.apple.com/news/?id=w5ngl9k2
- WWDC26 announcement (Jun 8, 2026): https://developer.apple.com/news/?id=8rgqj83s
- ImageCreator deprecation: https://developer.apple.com/news/?id=dz9wvq0r
- Design kits for iOS, iPadOS, macOS 27: https://developer.apple.com/news/?id=e2lxw9l1
- Apple Design Resources (downloads): https://developer.apple.com/design/resources/

**WWDC26 session pages** (for Foundation Models, App Intents, Core AI, WebKit, Evaluations, Music, NowPlaying, SwiftUI)
- 204 — What's new in WebKit for Safari 27: https://developer.apple.com/videos/play/wwdc2026/204/
- 240 — Build intelligent Siri experiences with App Intents: https://developer.apple.com/videos/play/wwdc2026/240
- 241 — What's new in the Foundation Models framework: https://developer.apple.com/videos/play/wwdc2026/241/
- 242 — Build agentic app experiences with the Foundation Models framework: https://developer.apple.com/videos/play/wwdc2026/242/
- 243 — Debug and profile agentic app experiences with Instruments: https://developer.apple.com/videos/play/wwdc2026/243/
- 246 — LLM search using Core Spotlight: https://developer.apple.com/videos/play/wwdc2026/246/
- 250 — Principles of great design: https://developer.apple.com/videos/play/wwdc2026/250/
- 251 — Communicate your brand identity on iOS: https://developer.apple.com/videos/play/wwdc2026/251/
- 253 — Meet the Music Understanding framework: https://developer.apple.com/videos/play/wwdc2026/253/
- 254 — Integrate MusicKit into your app: https://developer.apple.com/videos/play/wwdc2026/254/
- 256 — Discover generated subtitles and subtitle styles: https://developer.apple.com/videos/play/wwdc2026/256/
- 269 — What's new in SwiftUI: https://developer.apple.com/videos/play/wwdc2026/269/
- 272 — Use SwiftUI with AppKit and UIKit: https://developer.apple.com/videos/play/wwdc2026/272/
- 277 — WidgetKit foundations: https://developer.apple.com/videos/play/wwdc2026/277/
- 278 — Modernize your UIKit app: https://developer.apple.com/videos/play/wwdc2026/278/
- 290 — Craft clear names for features and labels in your app: https://developer.apple.com/videos/play/wwdc2026/290/
- 292 — Design intuitive search experiences: https://developer.apple.com/videos/play/wwdc2026/292/
- 295 — Meet the App Intents Testing framework: https://developer.apple.com/videos/play/wwdc2026/295/
- 298 — Meet the Evaluations framework: https://developer.apple.com/videos/play/wwdc2026/298/
- 299 — Create robust evaluations for agentic apps: https://developer.apple.com/videos/play/wwdc2026/299/
- 303 — Build a responsive camera app that launches quickly: https://developer.apple.com/videos/play/wwdc2026/303/
- 304 — Implement high resolution photo capture: https://developer.apple.com/videos/play/wwdc2026/304/
- 305 — Enhance RAW image processing with Core Image: https://developer.apple.com/videos/play/wwdc2026/305/
- 312 — Meet the NowPlaying framework: https://developer.apple.com/videos/play/wwdc2026/312/
- 319 — Build with the new Apple Foundation Model on Private Cloud Compute: https://developer.apple.com/videos/play/wwdc2026/319/
- 321 — Dive into lazy stacks and scrolling with SwiftUI: https://developer.apple.com/videos/play/wwdc2026/321/
- 322 — Compose advanced graphics effects with SwiftUI: https://developer.apple.com/videos/play/wwdc2026/322/
- 324 — Meet Core AI: https://developer.apple.com/videos/play/wwdc2026/324/
- 325 — Dive into Core AI model authoring and optimization: https://developer.apple.com/videos/play/wwdc2026/325/
- 326 — Integrate on-device AI models into your app using Core AI: https://developer.apple.com/videos/play/wwdc2026/326/
- 330 — Optimize custom machine learning operations with Metal tensors: https://developer.apple.com/videos/play/wwdc2026/330/
- 334 — Build AI powered scripts with the fm CLI and Python SDK: https://developer.apple.com/videos/play/wwdc2026/334/
- 335 — Improve your prompts by hill-climbing with Evaluations: https://developer.apple.com/videos/play/wwdc2026/335/
- 339 — Bring an LLM provider to the Foundation Models framework: https://developer.apple.com/videos/play/wwdc2026/339/
- 341 — Support the Center Stage front camera in your iOS app: https://developer.apple.com/videos/play/wwdc2026/341/
- 343 — Advanced App Intent schemas for Siri: https://developer.apple.com/videos/play/wwdc2026/343/
- 344 — Code-along: Make your app available to Siri: https://developer.apple.com/videos/play/wwdc2026/344/
- 345 — Discover new capabilities in the App Intents framework: https://developer.apple.com/videos/play/wwdc2026/345/
- 357 — Speedrun your game port with agentic coding: https://developer.apple.com/videos/play/wwdc2026/357/
- 358 — Make your game great with touch: https://developer.apple.com/videos/play/wwdc2026/358/
- 359 — Build real-time neural rendering pipelines with Metal: https://developer.apple.com/videos/play/wwdc2026/359/
- 388 — Find and fix performance issues in your Metal games: https://developer.apple.com/videos/play/wwdc2026/388/

**Apple framework documentation**
- Foundation Models: https://developer.apple.com/documentation/FoundationModels
- App Intents: https://developer.apple.com/documentation/appintents/
- Spatial Preview: https://developer.apple.com/documentation/SpatialPreview/
- WebKit: https://developer.apple.com/documentation/WebKit
- SwiftUI: https://developer.apple.com/documentation/swiftui
- Apple Silicon porting guide: https://developer.apple.com/documentation/apple-silicon/porting-your-macos-apps-to-apple-silicon
- Apple Silicon: https://developer.apple.com/documentation/apple-silicon

**swift-system proposals**
- SYS-0006 (stat APIs): https://github.com/apple/swift-system/blob/main/Proposals/0006-system-stat.md
- SYS-0008 (backdeploy / cinterop stat): https://github.com/apple/swift-system/blob/main/Proposals/0008-backdeploy-cinterop-stat.md

**Wikipedia (high-level context only — not cited for API claims)**
- macOS: https://en.wikipedia.org/wiki/MacOS
- macOS Golden Gate: https://en.wikipedia.org/wiki/MacOS_Golden_Gate

