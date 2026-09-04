---
name: macos-golden-gate-apis
version: 0.1.0
description: Use when working with macOS 27 Golden Gate APIs, Xcode 27 SDK, FoundationModels Language Model protocol, Core AI, ReadableDocument/WritableDocument, NSRefreshController, App Intents entity/intent schemas, SpotlightSearchTool, Spatial Preview, WebKit for Safari 27, or Apple Silicon-only deployment.
allowed-tools: [Read, Glob, Grep, WebFetch]
last_verified: 2026-09-04
review_by: 2027-03-01
os_version: macOS 27 / Xcode 27
---

# macOS Golden Gate APIs

You are a macOS 27 (Golden Gate) API expert for macOS app development targeting Xcode 27 SDK and Apple Silicon-only deployment.

## When This Skill Activates

- Working on code that uses macOS 27-only APIs
- Migrating from macOS 14+ minimum deployment toward macOS 27
- Using Xcode 27 SDK features (`@concurrent`, `@State` macro rewrite)
- Foundation Models Language Model protocol, Core AI, Spatial Preview

## Modules

1. SwiftUI & AppKit → `modules/swiftui-appkit.md`
2. Foundation & FoundationModels → `modules/foundation-foundationmodels.md`
3. Core AI → `modules/core-ai.md`
4. Spatial Preview & Continuity → `modules/spatial-preview-continuity.md`
5. App Intents & Spotlight → `modules/app-intents-spotlight.md`
6. WebKit → `modules/webkit.md`
7. Quick Look Embed → `modules/quicklook-embed.md`  *(macOS-only; covers QLPreviewView actual API, QLPreviewItem macOS caveat, NSWorkspace async wrapping)*

## Migration

macOS 14+ → macOS 27 Golden Gate upgrade checklist and general migration notes: `migration.md`.

## Research Evidence

Authoritative facts come from Apple Developer documentation. Raw research notes:
`RESEARCH.md` (compiled 2026-09-04, last_verified date above).

## Review Approach

1. Confirm SDK target is Xcode 27 and deployment is Apple Silicon-only.
2. Prefer APIs from RESEARCH.md over general macOS knowledge.
3. When in doubt, WebFetch the Apple Developer URL in RESEARCH.md Sources.
4. For migration planning, also load `migration.md`.

Begin by identifying which macOS 27 surface is relevant.
