---
module: app-intents-spotlight
target_apis: App Intents, Entity schemas, Intent schemas, View Annotations API, App Intents Testing, Core Spotlight, SpotlightSearchTool, notes.createNote, notes.updateNote, calendar.deleteEvent, AppEntity
---

# App Intents & Spotlight (macOS 27)

App Intents and Spotlight together form the system-level integration surface for macOS 27: App Intents lets users invoke app actions from Siri, Shortcuts, Spotlight, and the system intelligence layer; Core Spotlight indexes app content into a semantic search graph that Siri and the on-device language model can query. macOS 27 makes both surfaces AI-aware — entities contribute automatically to the Spotlight semantic index, the new `SpotlightSearchTool` lets a `LanguageModelSession` query Core Spotlight with natural language, and the App Intents Testing framework gives the integration a real test harness.

For your app, this surface is the bridge between "documents on disk" and "conversational workspace": a workspace item becomes an `AppEntity`, surfaces in Spotlight, becomes referenceable by Siri, and is queryable from a FoundationModels agent via `SpotlightSearchTool`.

## New APIs

### Entity schemas — `macOS 27.0+`, App Intents framework

Entity schemas declare the shape of an app's content as it appears in the system graph. An entity contributed through the schema surface is automatically indexed into Spotlight's semantic index, and Siri can surface the entity with attribution back to the contributing app. This is the recommended path for making workspace content (notes, sections, projects) discoverable across macOS without writing a custom Spotlight importer.

```swift
import AppIntents
import CoreSpotlight

struct NoteEntity: AppEntity {
    static var typeDisplayRepresentation: TypeDisplayRepresentation = "Note"

    var id: UUID
    var title: String
    var body: String
    var tags: [String]

    static var defaultQuery = NoteQuery()

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(title)", subtitle: "\(tags.joined(separator: ", "))")
    }

    @MainActor
    func load(from index: NoteIndex) async throws -> NoteEntity {
        self
    }
}

struct NoteQuery: EntityQuery {
    func entities(for identifiers: [UUID]) async throws -> [NoteEntity] { [] }
    func suggestedEntities() async throws -> [NoteEntity] { [] }
}
```

### Intent schemas — `macOS 27.0+`, App Intents framework

Intent schemas let users trigger app actions with natural language without developers defining fixed trigger phrases. The system expands and routes the utterance to the right intent based on the schema description, parameter types, and result declarations. This is the path for your app actions like "open my latest draft" or "summarize this section" without authoring a list of pre-canned phrases.

```swift
import AppIntents

struct OpenNote: AppIntent {
    static var title: LocalizedStringResource = "Open Note"
    static var description = IntentDescription("Opens a note by title.")

    @Parameter(title: "Title")
    var title: String

    static var parameterSummary: some ParameterSummary {
        Summary("Open \(\.$title)")
    }

    @MainActor
    func perform() async throws -> some IntentResult & ProvidesDialog {
        try await NoteService.shared.open(matching: title)
        return .result(dialog: "Opened \(title).")
    }
}
```

### View Annotations API — `macOS 27.0+`, SwiftUI / App Intents

The View Annotations API maps a SwiftUI view to an `AppEntity` so users can refer to what is on screen conversationally — "send this to Jason", "summarize this section", "what is this?". Annotations are declarative; the system tracks the active selection and routes the intent to the entity behind the focused view.

```swift
import SwiftUI
import AppIntents

struct OutlineRow: View {
    let section: SectionEntity

    var body: some View {
        HStack {
            Image(systemName: "doc.text")
            Text(section.title)
        }
        .appEntityAnnotation(section)
    }
}
```

### App Intents Testing framework — `macOS 27.0+`, Xcode 27 SDK

Introduced in WWDC26 295, the App Intents Testing framework validates an App Intents integration through real system pathways — entity resolution, intent dispatch, dialog generation, Spotlight indexing — without UI automation. The framework runs in `XCTest` and `swift-testing` and reports per-intent pass/fail with diagnostics.

```swift
import XCTest
import AppIntentsTesting

final class NoteIntentTests: XCTestCase {
    func testOpenNoteByTitle() async throws {
        let session = IntentTestSession()
        let result = try await session.perform(
            OpenNote(title: "Meeting notes")
        )
        XCTAssertTrue(result.didOpenNote)
    }
}
```

### LLM search via Core Spotlight — `macOS 27.0+`, Core Spotlight

Core Spotlight now exposes a search surface that the on-device system language model can query directly. `SpotlightSearchTool` is the FoundationModels-side adapter; when attached to a `LanguageModelSession`, the agent can answer natural-language questions over indexed content (notes, files, calendar events, photos) with results returned as structured entities.

```swift
import FoundationModels
import CoreSpotlight

let tool = SpotlightSearchTool(
    configuration: SpotlightSearchTool.Configuration(
        sources: [.coreSpotlight],
        guide: .focused(.documents)
    )
)

let session = LanguageModelSession(tools: [tool])
let answer = try await session.respond(
    to: "Find the draft about the Golden Gate release notes."
)
```

### `SpotlightSearchTool` `.focused()` guide — required configuration

The required configuration when using `SpotlightSearchTool` with `LanguageModelSession` backed by the on-device system model. Default configuration exceeds the on-device context window and the call fails. Available focused guides: `.communications`, `.calendar`, `.documents`, `.visualMedia`, `.audio`. (183770678)

```swift
let tool = SpotlightSearchTool(
    configuration: .init(sources: [.coreSpotlight], guide: .focused(.documents))
)
```

### `notes.createNote` / `notes.updateNote` accept `name: AttributedString`

The system-provided `notes` intents now accept `name` of type `AttributedString`. Use this when you want the note title to carry rich formatting — bold, links, inline images — that the system surfaces back to the user in confirmation dialogs and Spotlight. (173431080)

### `calendar.deleteEvents` → `calendar.deleteEvent` rename

The Calendar intent schema has been renamed from `calendar.deleteEvents` (plural) to `calendar.deleteEvent` (singular) to reflect that the intent deletes a single event at a time. If you have a Shortcuts workflow or App Intents invocation that calls the old name, update it. (176751155)

### `AppEntity` cumulative size limit: 10 MB

Every `AppEntity` instance, including all child properties and values, must serialize to under 10 MB. Entities that exceed the cap are rejected at runtime. For your app this matters if a single entity embeds the full body of a long note or a large embedded image — chunk the content into referenced sub-entities, or move large payloads behind a query. (181763422)

## Code Patterns

### 1. EntitySchema conformance

```swift
import AppIntents

struct NoteEntity: AppEntity {
    static var typeDisplayRepresentation: TypeDisplayRepresentation = "Note"

    @EntityProperty
    var id: UUID

    @EntityProperty
    var title: String

    @EntityProperty
    var workspaceName: String

    static var defaultQuery = NoteQuery()

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(title)", subtitle: "\(workspaceName)")
    }
}

struct NoteQuery: EntityQuery {
    func entities(for identifiers: [UUID]) async throws -> [NoteEntity] {
        try await NoteIndex.shared.notes(with: identifiers)
    }

    func suggestedEntities() async throws -> [NoteEntity] {
        try await NoteIndex.shared.recentNotes(limit: 10)
    }
}
```

### 2. IntentSchema with natural language

```swift
import AppIntents

struct SummarizeNote: AppIntent {
    static var title: LocalizedStringResource = "Summarize Note"
    static var description = IntentDescription(
        "Produces a three-bullet summary of a note using the on-device model."
    )

    @Parameter(title: "Note")
    var note: NoteEntity

    static var parameterSummary: some ParameterSummary {
        Summary("Summarize \(\.$note)")
    }

    @MainActor
    func perform() async throws -> some IntentResult & ReturnsValue<String> & ProvidesDialog {
        let summary = try await NoteSummarizer.shared.summarize(note)
        return .result(value: summary, dialog: "Summary ready.")
    }
}
```

### 3. View Annotations API usage

```swift
import SwiftUI
import AppIntents

struct OutlineView: View {
    @State private var sections: [SectionEntity] = []

    var body: some View {
        List(sections) { section in
            HStack {
                Image(systemName: section.systemImage)
                Text(section.title)
            }
            .appEntityAnnotation(section)
            .entityShortcut(.init(systemImageName: "doc.text", intent: OpenNote(title: section.title)))
        }
    }
}
```

### 4. SpotlightSearchTool with `.focused()` guide

```swift
import FoundationModels
import CoreSpotlight

@MainActor
func searchNotes(query: String) async throws -> String {
    let tool = SpotlightSearchTool(
        configuration: .init(sources: [.coreSpotlight], guide: .focused(.documents))
    )
    let session = LanguageModelSession(tools: [tool])
    let response = try await session.respond(to: query)
    return response.content
}
```

### 5. App Intents Testing example

```swift
import XCTest
import AppIntentsTesting
@testable import YourApp

final class NoteIntentsTests: XCTestCase {
    func testSummarizeRoutesThroughFoundationModels() async throws {
        let session = IntentTestSession()
        let note = try await NoteIndex.shared.note(with: UUID())

        let result = try await session.perform(SummarizeNote(note: note))

        XCTAssertFalse(result.value.isEmpty)
        XCTAssertGreaterThan(result.value.count, 40)
    }
}
```

## Migration from Tahoe

- **`@AppEntity(schema: .photos.asset)` may fail to compile.** macOS 27 adds new properties to the `.photos.asset` schema; existing conformances can no-link because the new required members are missing. Gate every conformance behind `if #available(macOS 27, *)` and provide the new properties in the gated branch. (181800016)
- **10 MB cumulative size limit on `AppEntity`.** Any entity that aggregates its children inline can blow past the cap once notes grow. For your app this means: never embed the full body of a long note directly on the entity; expose a separate query or a chunked sub-entity that the system loads on demand. (181763422)
- **`SpotlightSearchTool` without a `.focused()` guide fails with the on-device system LM.** The default configuration pulls in too much context for the system model to fit. Always pass a focused guide that matches the content domain. (183770678)
- **Adopt `App Intents Testing` for every new intent.** Replacing ad-hoc UI automation or manual smoke tests with the new framework keeps intent regressions out of the shipped build. ([WWDC26 295](https://developer.apple.com/videos/play/wwdc2026/295/))
- **Update any Shortcuts that called `calendar.deleteEvents`.** The schema renamed to `calendar.deleteEvent`. (176751155)
- **Use `AttributedString` for `notes.createNote` / `notes.updateNote` names** if the title needs formatting; the system preserves the attributes through confirmation dialogs. (173431080)
- **Index workspace content through `EntityQuery.suggestedEntities()`**, not through a separate Spotlight importer — schema-driven indexing now piggybacks on the same path.

## Common Mistakes

### 1. Using `SpotlightSearchTool` without a `.focused()` guide

The default `SpotlightSearchTool` configuration pulls in a corpus that exceeds the on-device system model's context window. The call returns an error and the agent's response is empty. Always pass `SpotlightSearchTool.Configuration(sources: [.coreSpotlight], guide: .focused(.documents))` (or another focused guide that matches the content domain). (183770678)

### 2. Conforming to `@AppEntity(schema: .photos.asset)` without an `if #available` gate

macOS 27 adds new properties to the `.photos.asset` schema. Existing conformances that do not supply those properties fail to compile on the macOS 27 SDK. Wrap the conformance in an availability check and provide the new properties in the gated branch. (181800016)

### 3. Embedding the full body in a workspace entity

A `NoteEntity` that serializes the entire note body, all embedded images, and the full revision history will exceed the 10 MB `AppEntity` cap once the note grows. Keep the entity lightweight and expose the body through an `EntityQuery` that loads it on demand.

### 4. Defining natural-language intents without an `IntentDescription`

`IntentSchema` routing depends on the `description`, parameter types, and result declarations. An intent with a one-line title and no description will not be routable by the system. Always provide `IntentDescription(...)` that describes what the intent does and what the parameters mean.

### 5. Skipping `App Intents Testing` and shipping intent regressions

Every intent and entity query should have an `IntentTestSession` test. Skipping the test framework and relying on manual Shortcuts smoke tests lets subtle regressions in entity resolution or dialog generation slip into release builds.

## References

- [macOS 27 release notes — App Intents, Core Spotlight](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes)
- [What's new in macOS 27 — App Intents](https://developer.apple.com/macos/whats-new/)
- [WWDC26 240 — Build intelligent Siri experiences with App Intents](https://developer.apple.com/videos/play/wwdc2026/240)
- [WWDC26 246 — LLM search using Core Spotlight](https://developer.apple.com/videos/play/wwdc2026/246/)
- [WWDC26 295 — Meet the App Intents Testing framework](https://developer.apple.com/videos/play/wwdc2026/295/)
- [WWDC26 343 — Advanced App Intent schemas for Siri](https://developer.apple.com/videos/play/wwdc2026/343/)
- [WWDC26 344 — Code-along: Make your app available to Siri](https://developer.apple.com/videos/play/wwdc2026/344/)
- [WWDC26 345 — Discover new capabilities in the App Intents framework](https://developer.apple.com/videos/play/wwdc2026/345/)
- [App Intents framework documentation](https://developer.apple.com/documentation/appintents/)
