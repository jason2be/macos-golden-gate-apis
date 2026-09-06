English | [中文](README.zh-CN.md)

# macOS Golden Gate APIs

> An AI-agent skill for macOS 27 Golden Gate API reference and migration guidance.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## What This Is

A comprehensive, source-cited API reference for macOS 27 "Golden Gate" development with Xcode 27 SDK, designed to be loaded by AI coding agents (Claude Code, Copilot CLI, Gemini CLI, etc.) as a skill.

Every API claim is backed by an Apple Developer documentation URL. The research was compiled from Apple's release notes, WWDC26 sessions, and framework documentation.

## What's Inside

```
├── SKILL.md                              # Skill entry point (agent loads this)
├── RESEARCH.md                           # 370+ lines of Apple-verified API research
├── migration.md                          # macOS 14+ → macOS 27 upgrade checklist
├── modules/
│   ├── swiftui-appkit.md                 # Document protocols, @State macro, menus, gestures
│   ├── foundation-foundationmodels.md    # LanguageModel protocol, sessions, multimodal
│   ├── core-ai.md                        # On-device ML, AOT compilation, Neural Engine
│   ├── webkit.md                         # WebKit for Safari 27, Grid Lanes, <model>
│   ├── quicklook-embed.md                # QLPreviewView API, QLPreviewItem caveat, NSWorkspace
│   ├── app-intents-spotlight.md          # Entity/Intent schemas, View Annotations, SpotlightSearchTool
│   └── spatial-preview-continuity.md     # Spatial Preview, SharePlay, visionOS
├── VERIFICATION.md                       # Skill retrieval and application test log
└── README.md
```

## Modules

| Module | Covers |
|--------|--------|
| **SwiftUI & AppKit** | `ReadableDocument` / `WritableDocument`, `@State` macro rewrite, menu icon hiding, exclusive gesture behavior, `NSRefreshController` |
| **Foundation & FoundationModels** | `LanguageModel` protocol (provider-agnostic), `LanguageModelSession`, multimodal prompts, Dynamic Profiles, Evaluations framework, `fm` CLI |
| **Core AI** | On-device ML framework, AOT compilation, Neural Engine background entitlement, Foundation Models Instrument |
| **WebKit** | Grid Lanes CSS, Customizable Select, HTML `<model>` element, Immersive Environments |
| **Quick Look** | `QLPreviewView` actual public API (not what Apple's docs say), `QLPreviewItem` macOS caveat, `NSWorkspace.open` async wrapping |
| **App Intents & Spotlight** | Entity schemas, Intent schemas, View Annotations API, `SpotlightSearchTool` with `.focused()` guide |
| **Spatial Preview** | Mac → Vision Pro preview framework, SharePlay collaboration |

## Who This Is For

- **AI coding agents** — load `SKILL.md` as a skill to get instant macOS 27 API expertise
- **macOS developers** — migrating from macOS 14+ to macOS 27 Golden Gate
- **Swift 6.4 adopters** — understanding `@concurrent`, `@State` macro, approachable concurrency

## Verified Against

| Component | Version |
|-----------|---------|
| macOS | 27 Golden Gate Beta 8 (26A5425a) |
| Xcode | 27 Beta 6 |
| Swift | 6.4 |
| Last verified | 2026-09-04 |

## How to Use as a Skill

Copy this directory to your agent's skills folder:

```bash
# For Claude Code / opencode
cp -r macos-golden-gate-apis ~/.agents/skills/macos-development/

# Or for a specific project
cp -r macos-golden-gate-apis /path/to/your/project/.agents/skills/
```

The agent will automatically discover and load `SKILL.md` when relevant.

## How to Use as a Reference

Read `RESEARCH.md` for the full API landscape, or jump to a specific `modules/*.md` for deep dives on a particular framework.

## Beta Documentation Notice

> **This skill is based on Apple's beta documentation** (macOS 27 Beta 8, Xcode 27 Beta 6).
> Apple may change API signatures, deprecate features, or add new capabilities before the
> final release. When Apple ships the official macOS 27 GM documentation, this skill will
> be updated to reflect any changes.
> A full re-verification is due within 14 days of the macOS 27 GM release (GM trigger
> clause in `SKILL.md` and `RESEARCH.md`); the `review_by` date is only a backstop.
>
> If you find a discrepancy between this skill and the released Apple documentation,
> please open an issue.

## Acknowledgments

This skill builds on the foundation laid by the **macos-tahoe-apis** skill, which covered macOS 26 Tahoe APIs and was part of the [superpowers](https://github.com/obra/superpowers) skill collection. The tahoe-apis skill established the module structure, research methodology, and agent-facing documentation patterns that this Golden Gate edition extends.

Special thanks to the superpowers community for creating and maintaining the skill infrastructure that makes this possible.

## License

MIT License — see [LICENSE](LICENSE) for details.

```
MIT License

Copyright (c) 2026 MarkdownQLite Contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
