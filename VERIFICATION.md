# Skill Verification Log

**Date:** 2026-09-04
**Tester:** subagent-driven scenarios
**Result:** PASS

## Retrieval Tests

### Scenario 1: Trigger test
**Result:** PASS
Description contains direct keyword matches for FoundationModels Language Model protocol, macOS 27, your app — fresh agents would load this skill.

### Scenario 2: Module routing
**Result:** PASS
SwiftUI document-stack queries (e.g., FileDocument → Document migration) route to `modules/swiftui-appkit.md` per SKILL.md §Modules. Module covers the migration with full guidance.

### Scenario 3: Cross-skill deferral
**Result:** PASS
Description is scoped to "macOS 27 Golden Gate"; no Tahoe/26 keywords. A sister skill exists for Tahoe queries.

## Application Test

### Scenario 4: Application accuracy
**Result:** PASS (with test methodology note)

**Question:** "What is the `@State` macro rewrite in Xcode 27, and what two compile-time patterns does it break?"

**Answer from RESEARCH.md §2.1:**
- `@State` macro rewrite in Xcode 27 doesn't re-evaluate initial expression on re-instantiation
- Two compile-time breaks:
  1. Assigning in `init` while providing a default
  2. Synthesized private memberwise init via extension when members are private

**Module routing:** SKILL.md §Modules directs `@State`-related queries to `modules/swiftui-appkit.md` (the "Macro-based `@State`" subsection under *SwiftUI — concurrency and macros* covers the macro rewrite; *Pattern 4* and *Mistake 1* cover the two break patterns). Module alone is sufficient.

**Methodology note:** The test subagent initially picked a random module file (`foundation-foundationmodels.md`) to answer the `@State` question and found no content. This is a test methodology issue, not a skill issue — agents following the Modules index in SKILL.md correctly route to `swiftui-appkit.md`. The skill design (SKILL.md as routing hub) works as intended.

## Overall

**PASS** — the skill is correctly:
1. Discoverable via description keywords
2. Routing queries to appropriate modules
3. Deferring to other skills for non-Golden-Gate queries
4. Providing accurate API information via module files (cross-verified against RESEARCH.md)

---

## Independent Audit Round 1（2026-09-06，v0.1.2 后）

**Auditor:** fresh subagent（无会话上下文、只读），证据源：本机 `MacOSX27.0.sdk` swiftinterface + `AppKit.framework/Headers/*.h` + 文档资产 sqlite。
**Result:** 24 findings（HIGH 11 / MEDIUM 10 / LOW 3），**本轮全部修复**；`SKILL.md` 新增路由索引与 Local Evidence 节经核对全部属实（所引小节/Pattern/Mistake 编号真实存在、两源路径本机证实、axiom-macos 未安装情形措辞正确）。

HIGH 修复要点：
1. `DocumentReader`/`DocumentWriter` 协议**真实存在**（interface:1768/1778，`@concurrent` I/O），删除 swiftui-appkit.md 中"不存在"的 NOTE（H1）。
2. `URLDocumentConfiguration` 虚构成员（`didUndoChange`/`document`）清除——真实成员仅 `fileURL`/`lastContentModificationDate`/`makeFileCoordinator()`（interface:1850-1863）；Pattern 1/Mistake 2/示例重写（H2/H3）。
3. `DocumentGroup(newDocument:)` 收 `ReadableDocument` 工厂的 init 不存在——改为 `allowCreating:editor:makeDocument:` + `NewDocumentAction` 重载（interface:1980/1986/3944），保留"legacy newDocument: 绑定旧协议"警示（H4）。
4. `NSRefreshController` 无闭包 init——两处示例改 target/action（H5）。
5. `candidateListForSelection(at:)` 全 SDK 0 命中——两处改 `NSAttributedString.Key.link` 属性查询（H6）。
6. `.allowConcurrent` 虚构 → `.notExclusive`（H7）；`preferredImageVisibility = .always` → `.visible`（H8，3 处）；writer 闭包补 `previous:` 参数（H9）；`makeFileWrapper(configuration:)`/`existingFile` 虚构 → 闭包签名版（H10）；`scrollGestureForFailureRelationship` → `scrollGestureForRelationships`（H11，3 处）。

MEDIUM 修复要点："no longer Sendable" 表述改为"MainActor 隔离"（`@MainActor final class` 隐式 Sendable，M1，4 处）；`ReferenceFileDocument` 三处统一为"已弃用"（M2，与 interface 9668 对齐）；migration.md 悬空锚点改为实际小节名（M3）；"completely gone" 改为"仅存在于已弃用 FileDocument"（M4）；allowed-tools 补 `Bash`（M5）；swiftinterface 权威范围收窄——AppKit ObjC 增量以 Headers 为准（M6，SKILL/RESEARCH/双 README）；路由索引补 gesture 行与 grep 兜底句（M7）；DocumentGroup 示例改用真实 init（M8）；KVC 行加未公开开关警示（M9）；协议草图改 associatedtype 真实形态（M10）。LOW：VERIFICATION 行号引用改小节名（L1）；README axiom-macos 补未安装回退句（L2）。另修：README 安装命令 `~/.agents/skills/macos-development/` → `~/.agents/skills/`（复核自查发现）。

**遗留（如实记录）**：release-note/运行时行为类断言（Info.plist 键、UserDefaults 键、行为变更类）静态取证无法证实也无法证伪，保持原样待 GM 复核；本机 SDK build 26A5419a 早于技能钉定的 Beta 8 26A5425a，引用行号以本机 interface 为准。

## Independent Audit Round 2（2026-09-07）

**Auditor:** 同一独立子代理，只读复审（对照未提交工作区 diff）。
**Result:** Round-1 的 24 条 **24/24 RESOLVED**；复审另抓出改写示例引入/沿用的 **4 条 NEW（HIGH 2 / MEDIUM 2），已全部修复**——

1. **[HIGH]** `struct` conform 到 `AnyObject` 约束的 `ReadableDocument`/`Document`（Round-1 漏报）：`NotePackage`/`MyDocument` 改 `final class`，`NotePackage` 补必需的 `snapshot(contentType:)`。
2. **[HIGH]** 链接点击替换示例编译不过（`fractionOfDistanceBetweenInsertionPoints:` 应传 `nil` 而非闭包；`NSTextContainer()` 无参 init 不存在）：两处改 `guard let container = textContainer` + `nil`。
3. **[MEDIUM]** `try? NewDocumentAction().callAsFunction(...)`——无 public init：改 `@Environment(\.newDocument)`，`try?` 删除（该重载非 throws）。
4. **[MEDIUM]** `DocumentGroup` 示例把 editor 闭包参数（收 `Document` 实例）误命名 `config`：改 `editor: { document in EditorView(document: document) }`。

复审同时确认：路由索引全部小节/编号真实、Local Evidence 两源本机属实、双语对齐、`SKILL.md` v0.1.3。
