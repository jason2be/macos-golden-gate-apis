---
module: core-ai
target_apis: Core AI, CoreAI, AOT compilation, on-device specialization cache, com.apple.developer.background-tasks.continued-processing.inference, Neural Engine background access, Foundation Models Instrument, large model loading, Metal tensors
---

# Core AI (macOS 27)

> **Note:** The Tahoe-era MLX Framework module has no direct successor in macOS 27. Apple positions **Core AI** as the on-device ML story for Apple Silicon. This module documents Core AI; refer to `foundation-foundationmodels.md` for Foundation Models integration and language model session patterns.

## What Is Core AI

Core AI is a brand-new, system-level framework shipped with macOS 27 ("Golden Gate") and built directly into the OS. It is purpose-built for Apple Silicon and provides a modern, memory-safe Swift API to **load, specialize, and run AI models entirely on-device**. It replaces the Tahoe-era MLX guidance with an Apple-supported path: models are automatically specialized for the hardware they run on via ahead-of-time (AOT) compilation, with fine-grained control over inference memory, zero-copy data paths, and stateful execution. Core AI covers everything from compact vision models to large-scale generative AI, runs on the Neural Engine (ANE), CPU, and GPU, and integrates with the Metal tensor stack for custom ML operations.

Core AI is **not** a drop-in replacement for FoundationModels. FoundationModels is the higher-level framework for orchestrating language-model sessions, callable tools, and structured generation. Core AI sits one layer below: it is the framework you reach for when you want to **bundle a custom model with your app** and run it on the user's machine. The two compose: a Core AI model can be exposed to a `LanguageModelSession` through the new `LanguageModel` protocol.

## New APIs

### `Core AI` Swift API surface — `macOS 27.0+`, Xcode 27 SDK

The headline addition. A Swift-first API for loading, specializing, and running AI models on-device. Models are described as bundles (a directory containing the model weights plus a manifest) and run through a `Model` handle that exposes a typed `predict` / `generate` surface.

```swift
import CoreAI

// Load a model bundle from the app's Resources directory.
let model = try await CoreAI.Model.load(
    url: Bundle.main.url(forResource: "summarizer", withExtension: "aibundle")!,
    configuration: .init(preferredCompute: .neuralEngine)
)

let summary: String = try await model.generate(
    prompt: "Summarize the following notes in three bullets.",
    context: notes
)
```

### AOT compilation via Xcode 27 Beta 2+

Core AI ships models that have been ahead-of-time compiled for the target hardware. The build pipeline (Xcode 27 Beta 2+) emits a specialized bundle the first time a model is loaded on a device; subsequent loads reuse the cached specialization. No runtime graph compilation. (181264112)

```swift
import CoreAI

// Build configuration tells Core AI to prefer AOT-specialized kernels
// when present, and to fall back to JIT specialization otherwise.
let config = CoreAI.Configuration(
    preferredCompute: .neuralEngine,
    allowRuntimeCompilation: false   // AOT-only — strict prod path
)

let model = try await CoreAI.Model.load(url: bundleURL, configuration: config)
```

### On-device specialization cache — `macOS 27.0+`, Xcode 27 SDK

Each Apple Silicon device compiles a model to its specific ANE/GPU configuration. Core AI caches the specialization in a per-app, per-model directory so subsequent launches avoid the compile pass. The cache is invalidated automatically when the model bundle's hash changes. The cache policy can be inspected and cleared. (169746264)

```swift
import CoreAI

let cache = CoreAI.SpecializationCache.default
let stats = await cache.statistics(for: model.identifier)
print("hits=\(stats.hits) misses=\(stats.misses) sizeBytes=\(stats.sizeBytes)")

// Force a re-specialization on the next load (useful for QA).
await cache.purge(modelIdentifier: model.identifier)
```

### Neural Engine background access — restricted entitlement

Neural Engine access in the background is restricted in the same way as GPU access has been since macOS 14. Background inference requires the new entitlement:

```
com.apple.developer.background-tasks.continued-processing.inference
```

Without this entitlement, any attempt to keep an ANE-backed model running after the app moves to the background is terminated by the system. Foreground inference is unaffected. (179282606)

```xml
<!-- MyApp.entitlements -->
<key>com.apple.developer.background-tasks.continued-processing.inference</key>
<true/>
```

```swift
import CoreAI

func canRunInBackground() -> Bool {
    // CoreAI exposes the runtime gate so you can branch on it.
    CoreAI.BackgroundEntitlement.isInferenceAllowed
}

if canRunInBackground() {
    // Register a BGProcessingTaskRequest for continued inference.
} else {
    // Gracefully finish the in-flight token and release ANE resources.
}
```

### Foundation Models Instrument in Instruments — `macOS 27.0+`, Xcode 27 SDK

A new template in Instruments that traces Foundation Models usage end-to-end: instructions, prompts, responses, token usage, and inference performance. The instrument hooks into both `FoundationModels` sessions and `Core AI` model handles, so custom Core AI models are visible in the same timeline. (164223804, Xcode 27 release notes)

```bash
# Open Instruments with the Foundation Models template against an app.
xcrun xctrace record \
    --template "Foundation Models" \
    --attach "MyApp" \
    --output ~/Desktop/foundation-models.trace
```

```swift
import CoreAI
import Foundation

// In code: tag the region of interest so the instrument can group samples.
await CoreAI.Instrumentation.beginInterval("outline-generation")
let outline = try await model.generate(prompt: prompt)
await CoreAI.Instrumentation.endInterval("outline-generation")
```

### Large model loading (> 1 GB) optimizations — `macOS 27.0+`, Xcode 27 SDK

Loading a multi-gigabyte generative model used to require a multi-second pause on cold start. Core AI on macOS 27 introduces lazy, mmap-backed weight loading with priority prefetch. The first token of a generation is delivered while subsequent layers are still being paged in. Performance is improved on the Neural Engine specifically, with the same lazy-load path on GPU and CPU. Memory attributed to the loaded weights now appears under the app process in the Allocations instrument. (174796039)

```swift
import CoreAI

// Streaming variant — yields partial outputs while layers are still loading.
let stream = model.generateStream(prompt: longPrompt, maxTokens: 2048)
for try await token in stream {
    print(token, terminator: "")
}
```

## Code Patterns

### 1. Loading a model with Core AI

```swift
import CoreAI

@MainActor
final class LocalSummarizer {
    private let model: CoreAI.Model

    init() async throws {
        guard let url = Bundle.main.url(forResource: "summarizer-v1",
                                         withExtension: "aibundle") else {
            throw CoreAI.Error.modelBundleMissing
        }
        self.model = try await CoreAI.Model.load(
            url: url,
            configuration: .init(preferredCompute: .neuralEngine)
        )
    }

    func summarize(_ text: String) async throws -> String {
        try await model.generate(
            prompt: "Summarize the following in three short bullets:\n\n\(text)"
        )
    }
}
```

### 2. Setting up AOT compilation in the build

AOT specialization happens on-device on first launch by default. To pre-specialize for QA devices, use the `coreai` CLI that ships with Xcode 27 Beta 2+.

```bash
# Pre-specialize a model for a target device class.
xcrun coreai specialize \
    --bundle ./Models/summarizer-v1.aibundle \
    --target mac-arm64 \
    --output ./Models/summarizer-v1.macos.aibundle
```

```swift
import CoreAI

// In code, point Core AI at the pre-specialized bundle on supported devices.
let url = Bundle.main.url(forResource: "summarizer-v1.macos",
                          withExtension: "aibundle")!
let model = try await CoreAI.Model.load(
    url: url,
    configuration: .init(preferredCompute: .neuralEngine,
                         allowRuntimeCompilation: false)
)
```

### 3. Handling Neural Engine background access with entitlement check

```swift
import CoreAI
import BackgroundTasks

func registerBackgroundInference() {
    BGTaskScheduler.shared.register(
        forTaskWithIdentifier: "com.example.app.inference",
        using: nil
    ) { task in
        guard CoreAI.BackgroundEntitlement.isInferenceAllowed else {
            task.setTaskCompleted(success: false)
            return
        }
        handleInference(task: task as! BGProcessingTask)
    }
}

private func handleInference(task: BGProcessingTask) {
    let operation = Task {
        let result = try await ModelRunner.shared.runPending()
        task.setTaskCompleted(success: true)
        _ = result
    }
    task.expirationHandler = { operation.cancel() }
}
```

```xml
<!-- MyApp.entitlements -->
<key>com.apple.developer.background-tasks.continued-processing.inference</key>
<true/>
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
    <string>com.example.app.inference</string>
</array>
```

### 4. Using the Foundation Models Instrument from code

```swift
import CoreAI
import Foundation

func traceGeneration(_ prompt: String) async throws -> String {
    await CoreAI.Instrumentation.beginInterval("summary")
    defer { Task { await CoreAI.Instrumentation.endInterval("summary") } }

    let model = try await ModelRegistry.shared.model(for: "summarizer-v1")
    return try await model.generate(prompt: prompt)
}
```

```bash
# Capture a trace from the command line.
xcrun xctrace record \
    --template "Foundation Models" \
    --launch "MyApp.app" \
    --output ~/Desktop/fm.trace
```

### 5. Streaming generation for a large model

```swift
import CoreAI

func streamSummary(_ text: String) -> AsyncThrowingStream<String, Error> {
    AsyncThrowingStream { continuation in
        Task {
            do {
                let model = try await CoreAI.Model.load(
                    url: Bundle.main.url(forResource: "summarizer-v1",
                                         withExtension: "aibundle")!,
                    configuration: .init(preferredCompute: .neuralEngine)
                )
                let stream = model.generateStream(
                    prompt: "Summarize:\n\(text)",
                    maxTokens: 512
                )
                for try await token in stream {
                    continuation.yield(token)
                }
                continuation.finish()
            } catch {
                continuation.finish(throwing: error)
            }
        }
    }
}
```

## Migration from Tahoe

The macOS 26 guidance referenced MLX as the third-party array framework for on-device ML. The macOS 27 release notes do not contain a separate "MLX" section; the **Core AI** framework is positioned as Apple's official on-device ML story. Migration items drawn from RESEARCH.md §3:

- **MLX Framework community path remains available**, but the Apple-official path is now Core AI. If a future project adopts MLX, evaluate a Core AI equivalent for tighter Apple Silicon integration and AOT specialization. Treat MLX as a portability / cross-platform option; treat Core AI as the production Apple-supported path.
- **Large model loading performance improved** — multi-GB generative models now stream weights into the Neural Engine and deliver the first token while remaining layers are still being paged in. Cold-start latency for > 1 GB models is significantly reduced compared to Tahoe-era patterns.
- **Neural Engine memory attributed to the app process** — visible in the Allocations instrument. This makes diagnosing ANE-driven memory pressure possible from Instruments rather than requiring external sampling. (174796039)
- **AOT compilation requires Xcode 27 Beta 2+** — earlier Xcode 27 betas compile but do not emit the optimized bundle; confirm the build host runs Beta 2 or later. (181264112)
- **On-device specialization cache** — Core AI caches specializations per device. The cache is invalidated automatically when the bundle hash changes, so shipping a new model version always triggers a re-specialization pass on first launch. Plan for that warm-up time in release notes and CI device farms. (169746264)
- **App-group support for shared model bundles** — fixed in macOS 27. If a future feature in your app shares a model bundle across an app group, this works out of the box on macOS 27 (Tahoe required manual symlink wiring). (179732320)
- **Metal tensor kernel load failures** — custom Metal kernels used to fail to load under certain dynamic-shape conditions; fixed. WWDC26 330 covers authoring Metal-backed custom ML ops for Core AI. (178056451, 177991751)
- **Foundation Models Instrument** — a new Instruments template is the recommended way to trace both FoundationModels sessions and Core AI model invocations. The Foundation Models Instrument is the single source of truth for prompt-level telemetry across both frameworks. (164223804)

## Common Mistakes

### 1. Forgetting the background-inference entitlement

Any code path that calls `BGTaskScheduler.shared.submit(...)` against a Core AI inference task without the `com.apple.developer.background-tasks.continued-processing.inference` entitlement will be terminated by the system the moment the app moves to the background. Declare the entitlement in `MyApp.entitlements` and verify it survives the App Store re-sign step. (179282606)

### 2. Mixing MLX and Core AI patterns in the same app

MLX (third-party array framework) and Core AI are not designed to interoperate. Pick one per model. If a future release of your app bundles a model with the app, choose Core AI for Apple-supported AOT specialization, entitlement-gated background inference, and first-class Instruments support. Reserve MLX for cross-platform code paths that need to run on non-Apple hardware.

### 3. Loading > 1 GB models without checking available memory

Core AI streams weights lazily, but the working set during inference still occupies substantial RAM. Check `ProcessInfo.processInfo.physicalMemory` (or Core AI's `availableMemoryBudget`) before loading a multi-GB model and degrade gracefully — fall back to a smaller model, or fail with a clear message rather than letting the jetsam kill the app. The Allocations instrument (where ANE memory now appears under your process) is the source of truth for setting a sensible budget.

### 4. Assuming Core AI runs on Intel hardware

Core AI is Apple-Silicon-only. macOS 27 itself only runs on Apple silicon, but verify any third-party CI hosts and developer machines are on Apple silicon before assuming the build pipeline works. The `xcrun coreai specialize --target` flag rejects non-arm64 targets explicitly. Your app targets Apple silicon and is unaffected, but vendored packages must be audited.

### 5. Re-running AOT specialization on every launch

The on-device specialization cache exists for a reason. Treat AOT compilation as a first-launch cost, not a per-launch cost. Inspect the cache statistics before profiling; if you see `misses` increasing across launches, something is invalidating the cache (typically a bundle hash change or an entitlement reset). Use `CoreAI.SpecializationCache.default.purge(...)` only in deliberate QA flows.

### 6. Shipping a Core AI bundle without on-demand resources

For large generative models, tag the bundle as an on-demand resource (`Tags: "core-ai-model"`) and download it via `NSBundleResourceRequest`. Otherwise the user pays the full bundle size at install time, and your App Store install conversion takes a hit. Pair with Background Assets (163944365) when the model is locale-specific.

## References

- [What's new in macOS 27 — Core AI](https://developer.apple.com/macos/whats-new/)
- [macOS 27 release notes — Core AI](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes)
- [WWDC26 324 — Meet Core AI](https://developer.apple.com/videos/play/wwdc2026/324/)
- [WWDC26 325 — Dive into Core AI model authoring and optimization](https://developer.apple.com/videos/play/wwdc2026/325/)
- [WWDC26 326 — Integrate on-device AI models into your app using Core AI](https://developer.apple.com/videos/play/wwdc2026/326/)
- [WWDC26 330 — Optimize custom machine learning operations with Metal tensors](https://developer.apple.com/videos/play/wwdc2026/330/)
- [Xcode 27 release notes — Foundation Models Instrument](https://developer.apple.com/documentation/Xcode-Release-Notes/xcode-27-release-notes)
- [WWDC26 243 — Debug and profile agentic app experiences with Instruments](https://developer.apple.com/videos/play/wwdc2026/243/)
- [WWDC26 241 — What's new in the Foundation Models framework](https://developer.apple.com/videos/play/wwdc2026/241/)
- [Apple Silicon porting guide](https://developer.apple.com/documentation/apple-silicon/porting-your-macos-apps-to-apple-silicon)
