---
name: macos-golden-gate-apis
version: 0.1.3
description: Use when working with macOS 27 Golden Gate APIs, Xcode 27 SDK, FoundationModels Language Model protocol, Core AI, ReadableDocument/WritableDocument, NSRefreshController, App Intents entity/intent schemas, SpotlightSearchTool, Spatial Preview, WebKit for Safari 27, or Apple Silicon-only deployment.
allowed-tools: [Read, Glob, Grep, Bash, WebFetch]
last_verified: 2026-09-04
review_by: 2026-10-31
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

- **GM 触发条款**：macOS 27 / Xcode 27 GM 发布后 14 天内，全量复核一次——重点：RESEARCH.md 全部 API 条目对照 GM release notes 与本机 GM swiftinterface（Xcode 27 GM SDK）、Sources 区 URL 逐条核对、modules 抽样。
- 复核完成前，GM 文档/SDK 与本技能冲突处以 GM 为准。
- `review_by: 2026-10-31` 仅为兜底日期（最晚 GM + 14 天复核窗口取整）；GM 触发优先于日期；GM 复核完成后 `review_by` 重置为常规节奏（复核日 + 约 6 个月）。

## Review Approach

1. Confirm SDK target is Xcode 27 and deployment is Apple Silicon-only.
2. Prefer APIs from RESEARCH.md over general macOS knowledge.
3. When in doubt, WebFetch the Apple Developer URL in RESEARCH.md Sources.
4. For migration planning, also load `migration.md`.
5. **SwiftUI 查询按此清单路由到 `modules/swiftui-appkit.md` 的对应小节**（不新建专项技能）：

| SwiftUI 主题 | 去处（swiftui-appkit.md） |
|---|---|
| Document / ReadableDocument / WritableDocument、ReferenceFileDocument 迁移、URLDocumentConfiguration | New APIs → SwiftUI — document model；Pattern 1 |
| AsyncImage、selection API | New APIs → SwiftUI — AsyncImage and selection |
| @State 宏重写、@concurrent | New APIs → SwiftUI — concurrency and macros；Mistake 1；Pattern 4 |
| 菜单图标 / preferredImageVisibility / Label 行为 | New APIs → SwiftUI — menu / label / icon behavior；Pattern 5 |
| NSRefreshController、NSToolbarItemGroup.role、窗口与菜单 | New APIs → AppKit — scroll, refresh, toolbar / windows and menus |
| 手势系统（cancellableByScrollGesture、exclusiveGestureBehavior、卡死超时、touch 约束） | New APIs → AppKit — gesture system |

索引未列的主题（NSTextSelectionManager、TextInputBorderShape、TabsPickerStyle、concentricCornerRadii、fileExporter、FileWrapperDocumentReader/Writer、@Entry、LabeledContent 等）先按符号名 grep `modules/swiftui-appkit.md` 定位小节再读。

6. **通用 SwiftUI 的 macOS 差异**（Table 多列、`.inspector`、窗口密度等平台行为差异）属于既有惯例积累而非 27 增量：若本机安装了 axiom 系技能，路由到 **axiom-macos（skills/swiftui-differences.md）** 与 axiom-swiftui；axiom-macos 未安装时按一般 macOS 经验处理，并回本技能查 27 增量。

## Local Evidence（开发者本机证据源）

除 Apple 网页外，做过相关开发的机器上大概率有下列**同位置**离线证据，在获得权限时可作依据之一——注意时效：

1. **Xcode SDK swiftinterface**（Swift/SwiftUI overlay API 的最终权威）：`/Applications/Xcode<version>.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX<ver>.sdk/System/Library/Frameworks/<Framework>.framework/Modules/<Framework>.swiftmodule/arm64e-apple-macos.swiftinterface`。**注意范围**：SwiftUI/SwiftUICore/Foundation 等 Swift overlay 框架的 27 增量在 interface 里；**AppKit 等纯 ObjC 框架的 27 增量大多不在 `AppKit.swiftinterface` 里，以 `AppKit.framework/Headers/*.h` 为准**（如 `NSRefreshController`、`NSMenuItem.preferredImageVisibility` 只在 Headers）。**时效规则**：结论前先 `xcode-select -p` / 查 SDK 目录确认机器上实际装的版本；本技能所有断言钉死在 **macOS 27 Beta 8 (26A5425a) + Xcode 27 Beta 6**，机器上的 SDK 若更新，以其标注为准并对照 GM 条款。
2. **本地开发者文档资产**（Deprecated/讨论/示例）：`/System/Library/AssetsV2/com_apple_MobileAsset_AppleDeveloperDocumentation/<hash>.asset/AssetData/documentation-db/index.sql`（`<hash>` 因机器而异，先 `ls` 该目录定位；SQLite 只读 `sqlite3 "file:...?mode=ro"`）。**时效与待验**：资产自带 OSVersion，用前核对；库表结构实测自同族 iOS 侧资产（documents/attributes/observations），macOS 资产使用前先复核 schema；`documents.document` BLOB 编码未解码，仅作辅助。
3. 诚实性守则（沿用本技能制作约定）：只收录有 Apple 来源支撑的条目；新结论先经上述两源在本机复核，复核不过的不写。

Begin by identifying which macOS 27 surface is relevant.
