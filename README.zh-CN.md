[English](README.md) | 中文

# macOS Golden Gate APIs

> 面向 macOS 27 Golden Gate 的 AI Agent 技能：API 参考与迁移指南。

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## 这是什么

一份面向 macOS 27 "Golden Gate" / Xcode 27 SDK 开发的、带源码引用的 API 参考手册，专为 AI 编程 Agent（Claude Code、Copilot CLI、Gemini CLI 等）设计，可作为技能直接加载。

每条 API 声明均附有 Apple Developer 文档链接。研究素材来自 Apple 发布说明、WWDC26 技术讲座和框架文档。

## 目录结构

```
├── SKILL.md                              # 技能入口（Agent 加载此文件）
├── RESEARCH.md                           # 370+ 行经 Apple 文档验证的 API 研究
├── migration.md                          # macOS 14+ → macOS 27 升级清单
├── modules/
│   ├── swiftui-appkit.md                 # Document 协议、@State 宏、菜单、手势
│   ├── foundation-foundationmodels.md    # LanguageModel 协议、会话、多模态
│   ├── core-ai.md                        # 端侧 ML、AOT 编译、Neural Engine
│   ├── webkit.md                         # WebKit for Safari 27、Grid Lanes、<model>
│   ├── quicklook-embed.md                # QLPreviewView 实际 API、QLPreviewItem 注意事项、NSWorkspace
│   ├── app-intents-spotlight.md          # Entity/Intent schema、View Annotations、SpotlightSearchTool
│   └── spatial-preview-continuity.md     # Spatial Preview、SharePlay、visionOS
├── VERIFICATION.md                       # 技能检索与应用测试记录
└── README.md
```

## 模块总览

| 模块 | 内容 |
|------|------|
| **SwiftUI & AppKit** | `ReadableDocument` / `WritableDocument`、`@State` 宏重写、菜单图标隐藏、独占手势行为、`NSRefreshController` |
| **Foundation & FoundationModels** | `LanguageModel` 协议（provider 无关）、`LanguageModelSession`、多模态提示、Dynamic Profiles、Evaluations 框架、`fm` CLI |
| **Core AI** | 端侧 ML 框架、AOT 编译、Neural Engine 后台权限、Foundation Models Instrument |
| **WebKit** | Grid Lanes CSS、Customizable Select、HTML `<model>` 元素、Immersive Environments |
| **Quick Look** | `QLPreviewView` 真实公开 API（与 Apple 文档不同）、`QLPreviewItem` macOS 注意事项、`NSWorkspace.open` 异步封装 |
| **App Intents & Spotlight** | Entity schema、Intent schema、View Annotations API、`SpotlightSearchTool` `.focused()` 配置 |
| **Spatial Preview** | Mac → Vision Pro 预览框架、SharePlay 协作 |

## 适用人群

- **AI 编程 Agent** — 加载 `SKILL.md` 即可获得 macOS 27 API 专业知识
- **macOS 开发者** — 从 macOS 14+ 迁移到 macOS 27 Golden Gate
- **Swift 6.4 采用者** — 理解 `@concurrent`、`@State` 宏、Approachable Concurrency

## 验证基准

| 组件 | 版本 |
|------|------|
| macOS | 27 Golden Gate Beta 8 (26A5425a) |
| Xcode | 27 Beta 6 |
| Swift | 6.4 |
| 最后验证日期 | 2026-09-04 |

## 作为技能使用

将此目录复制到 Agent 的技能文件夹：

```bash
# Claude Code / opencode
cp -r macos-golden-gate-apis ~/.agents/skills/

# 或针对特定项目
cp -r macos-golden-gate-apis /path/to/your/project/.agents/skills/
```

Agent 会在相关场景自动发现并加载 `SKILL.md`。

## 作为参考手册使用

阅读 `RESEARCH.md` 了解完整 API 全景，或跳转到 `modules/*.md` 深入特定框架。

## 本机证据源

除 Apple 网页外，做相关开发的机器在相同位置几乎都有同样的离线证据——经许可后可用，注意以本机实际版本为准（时效规则在 `SKILL.md`）：

- **Xcode SDK swiftinterface**（SwiftUI/Foundation 等 Swift overlay 框架的最终权威；AppKit 等 ObjC 框架的 27 增量不在 interface 里，以 `AppKit.framework/Headers/*.h` 为准）：当前 Xcode 下 `MacOSX27.0.sdk/System/Library/Frameworks/<Framework>.framework/Modules/<Framework>.swiftmodule/arm64e-apple-macos.swiftinterface`。
- **本地开发者文档资产**（Deprecated/讨论细节）：`/System/Library/AssetsV2/com_apple_MobileAsset_AppleDeveloperDocumentation/<hash>.asset/AssetData/documentation-db/index.sql`——只读 SQLite；`<hash>` 目录因机器而异。

SwiftUI 查询由 `SKILL.md` 的主题索引路由；Tahoe 世代的 SwiftUI 平台差异（Table、`.inspector`、窗口工具栏）在本机装有 **axiom-macos** 技能时交由其 `skills/swiftui-differences.md`——未安装时按一般 macOS 经验处理，并回本技能查 27 增量。

## Beta 文档声明

> **本技能基于 Apple Beta 文档编写**（macOS 27 Beta 8、Xcode 27 Beta 6）。
> Apple 可能在正式发布前更改 API 签名、废弃功能或新增能力。当 Apple 发布
> macOS 27 正式版文档后，本技能将同步更新。
> macOS 27 GM 发布后 14 天内将做一次全量复核（见 SKILL.md 与 RESEARCH.md 的 GM 触发条款）；
> frontmatter 的 review_by 日期仅作兜底。
>
> 如果发现本技能与 Apple 正式文档存在差异，请提交 Issue。

## 致谢

本技能站在 **macos-tahoe-apis** 技能的肩膀上。该技能覆盖了 macOS 26 Tahoe API，
是 [superpowers](https://github.com/obra/superpowers) 技能集合的一部分。
tahoe-apis 技能确立了模块结构、研究方法论和面向 Agent 的文档模式，
本 Golden Gate 版本在其基础上扩展。

特别感谢 superpowers 社区创建和维护了使本项目成为可能的技能基础设施。

## 许可证

MIT 许可证 — 详见 [LICENSE](LICENSE)。

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
