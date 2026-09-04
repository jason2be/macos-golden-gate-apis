# macOS Golden Gate APIs

A comprehensive API reference and migration guide for macOS 27 "Golden Gate" development with Xcode 27 SDK.

## What's Inside

- **RESEARCH.md** — Authoritative API research compiled from Apple Developer documentation, release notes, and WWDC26 sessions. Covers SwiftUI, AppKit, Foundation, FoundationModels, Core AI, WebKit, and more.
- **migration.md** — Generic macOS 14+ → macOS 27 upgrade checklist with breaking changes and verification steps.
- **modules/** — Deep-dive guides for specific API surfaces:
  - `swiftui-appkit.md` — Document protocols (`ReadableDocument`/`WritableDocument`), `@State` macro, menu icons, gesture behavior
  - `foundation-foundationmodels.md` — `LanguageModel` protocol, `LanguageModelSession`, multimodal prompts, Dynamic Profiles
  - `core-ai.md` — On-device ML framework, AOT compilation, Neural Engine
  - `webkit.md` — WebKit for Safari 27, Grid Lanes, `<model>` element
  - `quicklook-embed.md` — `QLPreviewView` actual API, `QLPreviewItem` macOS caveat, `NSWorkspace` async wrapping
  - `app-intents-spotlight.md` — Entity schemas, Intent schemas, View Annotations, `SpotlightSearchTool`
  - `spatial-preview-continuity.md` — Spatial Preview framework, SharePlay, visionOS integration

## Who This Is For

macOS developers targeting macOS 27 Golden Gate with Xcode 27 SDK, especially those:
- Migrating from macOS 14+ minimum deployment
- Adopting Swift 6.4 and `@concurrent` concurrency patterns
- Using Foundation Models, Core AI, or WebKit for on-device AI features
- Building document-based apps with the new `Document` protocol

## Verified Against

- macOS 27 Golden Gate Beta 8 (26A5425a)
- Xcode 27 Beta 6
- All API claims backed by Apple Developer documentation URLs

## License

MIT
