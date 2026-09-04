---
module: foundation-foundationmodels
target_apis: Stat, FilePath.stat, FileDescriptor.stat, FileType, FileMode, FileFlags, UserID, GroupID, DeviceID, Inode, URL.URLWithString, LanguageModel, LanguageModelSession, @Generable, Evaluations framework, fm CLI
---

# Foundation & FoundationModels (macOS 27)

This module covers two related surfaces in macOS 27 (Golden Gate). The first is a small but sharp Swift modernization in Foundation: the new `Stat` family that brings `stat(2)`/`lstat(2)`/`fstat(2)`/`fstatat(2)` into type-safe Swift on `FilePath` and `FileDescriptor`. The second is a substantial expansion of Apple's FoundationModels framework — the on-device AI framework introduced in macOS 26 (Tahoe) and broadened in macOS 27 to support any model provider, multimodal inputs, and callable Vision tools.

Both surfaces may touch your project directly: `Stat` may collide with existing `FilePath.stat` extensions on subprocess paths, and FoundationModels is the substrate for document structuring and OCR-driven text extraction.

## New APIs

### Foundation — `Stat` family

#### `Stat` — `macOS 27.0+`, Xcode 27 SDK

New value type produced by the modernized Swift `stat` family. `Stat` aggregates the C `stat` struct fields into strongly-typed Swift properties: `FileType`, `FileMode`, `FileFlags`, `UserID`, `GroupID`, `DeviceID`, `Inode`, plus size, timestamps, and link count. Inits accept a `FilePath`, a `FileDescriptor`, or a C string (`init(path:)`); `init(path:)` follows symbolic links by default while `lstat` semantics are exposed via `init(lstat:)`. Backed by [swift-system SYS-0006](https://github.com/apple/swift-system/blob/main/Proposals/0006-system-stat.md). (160612181)

```swift
import System

let path: FilePath = "/Users/me/Documents/notes.md"
let info = try Stat(path: path)
print(info.fileType)        // .regular
print(info.fileMode.permissions) // [.ownerRead, .ownerWrite, ...]
print(info.size)            // UInt64
print(info.inode)           // Inode
```

#### `FilePath.stat()` / `FileDescriptor.stat()` — `macOS 27.0+`, Xcode 27 SDK

Instance methods on `FilePath` and `FileDescriptor` returning `Stat`. Replace the older pattern of calling `Darwin.stat` with an in-out pointer. These are the canonical replacements — unqualified `Darwin.stat` calls in extensions on `FilePath`/`FileDescriptor` are deprecated in favor of these instance methods. Backed by [SYS-0008](https://github.com/apple/swift-system/blob/main/Proposals/0008-backdeploy-cinterop-stat.md). (177911316)

```swift
import System

let path: FilePath = "/etc/hosts"
let st = try path.stat()
if st.fileType == .symbolicLink {
    let target = try path.lstat()  // lstat does not follow links
    _ = target
}
```

#### `FileType` — `macOS 27.0+`, Xcode 27 SDK

Enum describing the inode file kind: `.regular`, `.directory`, `.symbolicLink`, `.characterSpecial`, `.blockSpecial`, `.socket`, `.fifo`, `.whiteout`. Replaces raw C `S_IF*` constants.

```swift
let path: FilePath = "/tmp"
if try path.stat().fileType == .directory {
    print("is a directory")
}
```

#### `FileMode` — `macOS 27.0+`, Xcode 27 SDK

Wraps the C mode bitmask. Exposes `permissions` as an `OptionSet` (`FilePermissions`) and helpers like `isReadable(by: .owner)`, `isWritable(by: .group)`. Replaces raw `S_IRUSR`/`S_IWGRP`/etc.

```swift
let info = try FilePath("/tmp/report.md").stat()
if info.fileMode.permissions.contains(.ownerWrite) {
    // writable by owner
}
```

#### `FileFlags` — `macOS 27.0+`, Xcode 27 SDK

Option set for BSD file flags (`UF_HIDDEN`, `UF_NODUMP`, `SF_IMMUTABLE`, etc.) previously accessed via `chflags(2)`/`fchflags(2)`.

```swift
let st = try path.stat()
if st.flags.contains(.hidden) {
    // file has the UF_HIDDEN flag
}
```

#### `UserID`, `GroupID`, `DeviceID`, `Inode` — `macOS 27.0+`, Xcode 27 SDK

Strongly-typed wrappers over `uid_t`, `gid_t`, `dev_t`, and `ino_t`. Compare with `==` directly; no raw integer arithmetic required.

```swift
let st = try FilePath("/dev/disk0s1").stat()
let diskOwner: UserID = st.owner
print(st.deviceID, st.inode)
```

### FoundationModels framework

#### `LanguageModel` protocol — `macOS 27.0+`, Xcode 27 SDK

Any language model — Apple Foundation Models, Private Cloud Compute, Claude, Gemini, or a self-hosted provider — conforms to the new `LanguageModel` protocol. `LanguageModelSession` is generic over the protocol, so you can swap providers without rewriting call sites. WWDC26 241 / 339 cover provider conformance.

```swift
import FoundationModels

let onDevice: any LanguageModel = SystemLanguageModel()
let session = LanguageModelSession(model: onDevice)
let answer = try await session.respond(to: "Summarize this note.")
```

#### `LanguageModelSession` (continued from Tahoe) — `macOS 26.0+`, expanded in `27.0+`

The session API from Tahoe 26 carries forward with new capabilities in 27: multimodal prompts, dynamic profiles, Vision tools, and any-provider dispatch. WWDC26 242 covers agentic patterns.

#### `@Generable` macro — `macOS 26.0+`, expanded in `27.0+`

Macros a Swift type (struct, enum, or class) into a guided-generation schema. Used both for structured output and to bind tool inputs to model-callable functions. Note: in macOS 27 SDK, `@Generable` on an `enum` produces an unavoidable deprecation warning about `GenerationError` (177899620) — silence is not possible; only the new SDK removes it.

```swift
import FoundationModels

@Generable
struct Outline {
    var title: String
    var sections: [Section]
    @Generable struct Section { var heading: String; var bullets: [String] }
}
```

#### Multimodal prompts — `macOS 27.0+`, Xcode 27 SDK

`LanguageModelSession.respond(to:)` now accepts prompts that bundle images and text. The on-device system model reasons about visual content alongside prose; no separate vision pipeline is required.

```swift
import FoundationModels
import CoreGraphics

let image = CGImage(...) // loaded from disk
let prompt = Prompt(text: "Describe this diagram.", image: image)
let reply = try await session.respond(to: prompt)
```

#### Vision framework tools (OCR, barcode) — `macOS 27.0+`, Xcode 27 SDK

Vision's text and barcode recognizers are exposed as on-device callable tools that a `LanguageModelSession` can invoke during a turn. The session orchestrates the call; you register the tool once.

```swift
import FoundationModels
import VisionTooling // vision tools exposed via FoundationModels

let session = LanguageModelSession(
    model: SystemLanguageModel(),
    tools: [VisionOCRTool(), VisionBarcodeTool()]
)
```

#### Dynamic Profiles — `macOS 27.0+`, Xcode 27 SDK

A `Profile` bundles a model + tools + instructions. Sessions can swap profiles mid-flight without losing transcript, enabling a "deep work" vs "quick summary" toggle inside one continuous session.

```swift
import FoundationModels

let deepWork = Profile(name: "deep",
                       model: SystemLanguageModel(),
                       instructions: "Be thorough.",
                       tools: [VisionOCRTool()])
let summary = Profile(name: "summary",
                      model: SystemLanguageModel(),
                      instructions: "One sentence.")
session.profile = deepWork  // swap mid-session
```

#### Evaluations framework — `macOS 27.0+`, Xcode 27 SDK

New framework for verifying AI features across dynamic conditions. Use it to regression-test prompts, model swaps, and tool wiring. WWDC26 298 (introduction) and WWDC26 299 (agentic apps) / 335 (hill-climbing prompts) cover the API.

```swift
import Evaluations

let suite = EvaluationSuite(
    name: "outline-quality",
    cases: TestCorpus.outlines
)
let report = try await suite.run(against: SystemLanguageModel())
```

#### `fm` CLI + Python SDK — `macOS 27.0+`, Xcode 27 SDK

Command-line tool (`fm`) and Python SDK for AI-powered scripts. Lets non-Swift pipelines (build scripts, Python tools) call into the same FoundationModels surface. WWDC26 334 covers installation and use.

```bash
$ fm prompt --model on-device "Summarize notes.md"
```

#### Small Business Program benefit — `macOS 27.0+`, Xcode 27 SDK

Apps enrolled in the Small Business Program with fewer than 2 million first-time downloads can access next-generation Apple Foundation Models on Private Cloud Compute at no cloud API cost. WWDC26 319 covers the entitlement flow.

## Code Patterns

### 1. Stat usage via `FilePath`

```swift
import System

func describe(_ path: FilePath) throws -> String {
    let st = try path.stat()
    return """
    path=\(path.description)
    type=\(st.fileType)
    size=\(st.size) bytes
    owner=\(st.owner):\(st.group)
    mode=\(st.fileMode.permissions.rawValue, radix: 8)
    modified=\(st.modificationDate)
    """
}

do {
    let info = try describe(FilePath("/Users/me/notes.md"))
    print(info)
} catch {
    print("stat failed: \(error)")
}
```

### 2. Stat usage via `FileDescriptor`

```swift
import System
import Foundation

func describe(fd: FileDescriptor) throws -> String {
    let st = try fd.stat()
    return "fd=\(fd.rawValue) type=\(st.fileType) size=\(st.size)"
}

let url = URL(fileURLWithPath: "/etc/hosts")
let fd = try FileDescriptor.open(url.path, .readOnly)
defer { try? fd.close() }
print(try describe(fd: fd))
```

### 3. Language Model protocol conformance

```swift
import FoundationModels

struct ClaudeLanguageModel: LanguageModel {
    let endpoint: URL
    let apiKey: String

    func generate(prompt: Prompt,
                  options: GenerationOptions) async throws -> Response {
        var request = URLRequest(url: endpoint)
        request.httpMethod = "POST"
        request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
        // Serialize `prompt` to the provider's wire format and decode
        // the response into FoundationModels.Response.
        return Response(text: "")
    }
}

let session = LanguageModelSession(model: ClaudeLanguageModel(...))
let _ = try await session.respond(to: "Hello.")
```

### 4. Multimodal prompt example

```swift
import FoundationModels
import CoreGraphics
import ImageIO

func describe(imageURL: URL) async throws -> String {
    guard let src = CGImageSourceCreateWithURL(imageURL as CFURL, nil),
          let image = CGImageSourceCreateImageAtIndex(src, 0, nil) else {
        throw CocoaError(.fileReadUnknown)
    }
    let session = LanguageModelSession(model: SystemLanguageModel())
    let prompt = Prompt(text: "What text appears in this image?",
                        image: image)
    let reply = try await session.respond(to: prompt)
    return reply.text
}
```

### 5. Calling Vision OCR as a FoundationModels tool

```swift
import FoundationModels

@Generable
struct ExtractedText {
    var lines: [String]
    var confidence: Double
}

let session = LanguageModelSession(
    model: SystemLanguageModel(),
    tools: [VisionOCRTool()]
)

let prompt = Prompt(text: """
    Read the handwritten note in this image and return the lines as
    structured text.
    """, image: loadedImage)

let response = try await session.respond(
    to: prompt,
    generating: ExtractedText.self
)
print(response.content.lines)
```

## Migration from Tahoe

Items from RESEARCH.md §3 and §4 that affect this module:

- **System Swift APIs for `stat` family** — unqualified `Darwin.stat` calls may collide with new `FilePath.stat()` / `FileDescriptor.stat()` instance methods if your code uses unqualified `Darwin.stat` calls in extensions. Review and rename. See [SYS-0008](https://github.com/apple/swift-system/blob/main/Proposals/0008-backdeploy-cinterop-stat.md). (177911316)
- **`@Generable` enum deprecation warning about `GenerationError`** — the warning cannot be silenced; only the new SDK behavior removes it. (177899620)
- **`PrivateCloudComputeLanguageModel` always used greedy decoding** — fixed in macOS 27; sampling parameters are now respected. (178181782)
- **`onPrompt` not called when applied to a `Profile` without instructions** — fixed. Transcript truncation via `onPrompt` no longer causes unexpected runtime errors. (177901494, 177902488)
- **Foundation URL fix** — `+[NSURL URLWithString:]` no longer double-encodes `%` of valid percent-escape sequences. If you have workarounds in place, drop them. (161588649)

## Common Mistakes

1. **Keeping `Darwin.stat` calls in extensions on `FilePath` or `FileDescriptor`.** In macOS 27 SDK the unqualified C call collides with the new instance methods and is deprecated. Either move to `try path.stat()` / `try fd.stat()` or qualify the call as `Darwin.stat(...)`. (177911316)
2. **Comparing `uid_t`/`gid_t`/`ino_t` with raw integer arithmetic.** The new `UserID`/`GroupID`/`Inode`/`DeviceID` types are `Equatable` directly — use `==`. Don't `.rawValue == otherRawValue`.
3. **Trying to silence the `@Generable` enum `GenerationError` deprecation warning.** The warning cannot be silenced; only the macOS 27 SDK removes it. Don't waste time on `#suppress` or build-flag workarounds. (177899620)
4. **Using `SpotlightSearchTool` with `LanguageModelSession` backed by the on-device system model without a `.focused()` guide.** Default config exceeds the on-device context window and the call fails. Always pass `SpotlightSearchTool.Configuration(sources: [.coreSpotlight], guide: .focused(.documents))` (or another focused guide). (183770678)
5. **Hardcoding `SystemLanguageModel` as the only path.** Treat model choice as configuration: conform third-party providers to `LanguageModel`, surface a provider picker, and write the call site as `LanguageModelSession(model: any LanguageModel)`.
6. **Assuming Vision OCR has to be called before the model.** Register `VisionOCRTool` and let the session decide when to call it; do not pre-OCR every image and then hand the model text only.

## References

- [What's new in the Foundation Models framework (WWDC26 241)](https://developer.apple.com/videos/play/wwdc2026/241/)
- [Build agentic app experiences with the Foundation Models framework (WWDC26 242)](https://developer.apple.com/videos/play/wwdc2026/242/)
- [Debug and profile agentic app experiences with Instruments (WWDC26 243)](https://developer.apple.com/videos/play/wwdc2026/243/)
- [Meet the Evaluations framework (WWDC26 298)](https://developer.apple.com/videos/play/wwdc2026/298/)
- [Create robust evaluations for agentic apps (WWDC26 299)](https://developer.apple.com/videos/play/wwdc2026/299/)
- [Improve your prompts by hill-climbing with Evaluations (WWDC26 335)](https://developer.apple.com/videos/play/wwdc2026/335/)
- [Build with the new Apple Foundation Model on Private Cloud Compute (WWDC26 319)](https://developer.apple.com/videos/play/wwdc2026/319/)
- [Bring an LLM provider to the Foundation Models framework (WWDC26 339)](https://developer.apple.com/videos/play/wwdc2026/339/)
- [Build AI powered scripts with the fm CLI and Python SDK (WWDC26 334)](https://developer.apple.com/videos/play/wwdc2026/334/)
- [Foundation Models framework documentation](https://developer.apple.com/documentation/FoundationModels)
- [What's new in macOS 27](https://developer.apple.com/macos/whats-new/)
- [macOS 27 release notes](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes)
- [swift-system SYS-0006 (Stat APIs)](https://github.com/apple/swift-system/blob/main/Proposals/0006-system-stat.md)
- [swift-system SYS-0008 (backdeploy / cinterop stat)](https://github.com/apple/swift-system/blob/main/Proposals/0008-backdeploy-cinterop-stat.md)
