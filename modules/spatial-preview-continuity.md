---
module: spatial-preview-continuity
target_apis: macOS Spatial Preview framework, SharePlay spatial collaboration, Quick Look on visionOS, iPhone Mirroring window resize
---

# Spatial Preview & Continuity (macOS 27)

> **Scope:** This module covers the macOS 27 cross-device surface as documented in RESEARCH.md §2.7: the **macOS Spatial Preview framework**, **SharePlay**-based spatial collaboration, **Quick Look on visionOS** integration from a Mac app, and the **iPhone Mirroring** window-resize refinement. None of these map to existing features in your app, so this module is documented for completeness because your project is a Mac-native text workspace and may want to expose spatial previews in the future.

## What Is Spatial Preview

macOS 27 introduces a new system framework — `SpatialPreview` — that lets a Mac app push spatial content (3D USD assets, spatial photos, Apple Immersive Video) to a paired **Apple Vision Pro** and preview it there. The framework connects a Mac source to **Quick Look on visionOS**, the existing immersive preview surface on visionOS. The same pipeline supports **SharePlay** so a group of people on visionOS (and the Mac presenter) can collaborate inside the previewed scene in real time.

Spatial Preview uses **RealityKit** for USD rendering in Preview / Quick Look on the Vision Pro side, replacing the legacy Storm pipeline. A **Gaussian Splat Component** API is announced for Spatial Preview and "will be available in an upcoming release" (178061856). For live editing of 3D content, Spatial Preview keeps the Mac-side USD scene authoritative and streams edits to the Vision Pro preview in real time.

On the Continuity side, the **iPhone Mirroring** app on Mac now supports **window resizing** for the mirrored iPhone screen. This is a small but important refinement: on macOS 26 (Tahoe) the mirrored window was a fixed size; on macOS 27 the user can resize it freely to fit alongside Mac windows.

## New APIs

### `SpatialPreview` framework — `macOS 27.0+`, paired with visionOS

The headline addition. A Mac app requests a spatial preview session, picks the asset(s) to preview, and the system displays them on a connected Apple Vision Pro. The session is SharePlay-aware: invitees see the same scene with their own viewpoint.

```swift
import SpatialPreview

// 1. Resolve the paired Vision Pro.
let destination = SpatialPreview.Destination.nearbyVisionPro()

// 2. Build a session for a USD scene.
let session = try await SpatialPreview.Session(
    destination: destination,
    scene: urlOfUSDPackage,
    mode: .immersive
)

// 3. Show the preview window on the Vision Pro.
try await session.start()

// 4. Stream a USD edit from the Mac to update the live scene.
try await session.updateScene(with: editedUSDURL)
```

### SharePlay spatial collaboration inside Spatial Preview — `macOS 27.0+`

When the preview session is started with `GroupSession`, every invitee on a Vision Pro joins the same spatial scene and the Mac presenter becomes the editor. Participants see each other's cursors and annotations.

```swift
import SpatialPreview
import GroupActivities

let activity = SpatialPreview.CollaborationActivity(sceneURL: sceneURL)
let groupSession = try await GroupSession<SpatialPreview.CollaborationActivity>(
    join: activity
)

for await session in groupSession.sessions {
    try await session.start()
}
```

### Quick Look on visionOS integration — `macOS 27.0+`

The Mac source can route previews to Quick Look on visionOS for spatial photos, Apple Immersive Video, and 3D USD content. Live editing keeps a two-way binding so changes on the Mac side appear in the headset preview without a manual reload.

```swift
import SpatialPreview
import QuickLook

let preview = SpatialPreview.QuickLookBridge(
    destination: .nearbyVisionPro(),
    contentTypes: [.spatialPhoto, .immersiveVideo, .usdScene]
)

try await preview.open(url: mediaURL)
```

### iPhone Mirroring window resize — `macOS 27.0+`

The iPhone Mirroring app on Mac now exposes a standard `NSWindow` resize affordance. There is no new public API for third-party apps to control this directly; the change is system-level and applies automatically when the mirrored iPhone is paired.

```swift
// No third-party code is required. Users resize the mirrored iPhone
// window via the standard macOS window controls. Apps that want to
// react to size changes can observe NSWindow.didResizeNotification
// on the mirrored window (when permitted by the system).
```

## Code Patterns

### 1. Opening a Spatial Preview window for a USD scene

```swift
import SpatialPreview
import Foundation

@MainActor
final class SpatialPreviewController {

    private var session: SpatialPreview.Session?

    func preview(usdz url: URL) async throws {
        let destination = SpatialPreview.Destination.nearbyVisionPro()
        let session = try await SpatialPreview.Session(
            destination: destination,
            scene: url,
            mode: .window
        )
        self.session = session
        try await session.start()
    }

    func pushEdit(_ editedUSD: URL) async throws {
        guard let session else { return }
        try await session.updateScene(with: editedUSD)
    }

    func end() async {
        try? await session?.end()
        session = nil
    }
}
```

### 2. SharePlay spatial collaboration across visionOS participants

```swift
import SpatialPreview
import GroupActivities

func collaborate(on sceneURL: URL) async throws {
    let activity = SpatialPreview.CollaborationActivity(sceneURL: sceneURL)
    let result = try await activity.prepare()
    switch result {
    case .activationPreferred:
        // Hand off to GroupSession so the system invites nearby participants.
        let groupSession = try await GroupSession
            .<SpatialPreview.CollaborationActivity>(join: activity)
        for await session in groupSession.sessions {
            try await session.start()
        }
    case .activationDisabled:
        // User declined; fall back to a solo preview.
        try await SpatialPreview.Session(
            destination: .nearbyVisionPro(),
            scene: sceneURL,
            mode: .immersive
        ).start()
    case .cancelled:
        break
    @unknown default:
        break
    }
}
```

### 3. Quick Look on visionOS from a Mac app, with live editing

```swift
import SpatialPreview

@MainActor
final class SpatialQuickLookController {

    private var bridge: SpatialPreview.QuickLookBridge?

    func openPreview(for mediaURL: URL) async throws {
        let bridge = SpatialPreview.QuickLookBridge(
            destination: .nearbyVisionPro(),
            contentTypes: [.spatialPhoto, .immersiveVideo, .usdScene]
        )
        self.bridge = bridge
        try await bridge.open(url: mediaURL)
    }

    func replaceContent(with newURL: URL) async throws {
        try await bridge?.replaceContent(with: newURL)
    }
}
```

## Migration from Tahoe

There is **no Tahoe counterpart for Spatial Preview** — the framework is new in macOS 27. Tahoe-era Mac apps that needed to preview USD content relied on the local Quick Look panel only, with no Vision Pro handoff, no live USD editing handoff, and no SharePlay spatial collaboration.

Migration items from RESEARCH.md §2.7:

- **Adopt `SpatialPreview` for any 3D / spatial content preview.** If a Mac app on Tahoe opened a USDZ file in the local Quick Look panel and only ran on the Mac, the macOS 27 path is to keep the local preview but also offer "Preview on Vision Pro" via `SpatialPreview.Session`. There is no deprecation to handle; the old path continues to work.
- **RealityKit replaces Storm in the Spatial Preview pipeline.** On the Vision Pro side, USD scenes are now rendered by RealityKit rather than the legacy Storm pipeline. Mac apps that authored assets specifically for Storm quirks should re-test under RealityKit rendering. (176839273)
- **Gaussian Splat Component is announced but not yet available.** Treat any "Gaussian Splat" preview feature as future work. The component "will be available in an upcoming release" (178061856); do not ship a dependency on it in macOS 27.0.
- **iPhone Mirroring window resize is automatic.** No code changes are required in third-party Mac apps; the mirrored window now exposes standard resize controls. Apps that hardcoded the mirrored window's frame on Tahoe should remove those assumptions — the window's frame is user-controlled on macOS 27.
- **Live USD editing is two-way by default.** On Tahoe, Mac-to-Vision-Pro USD previews required a manual reload of the scene. On macOS 27, `updateScene(with:)` keeps the headset preview in sync with the Mac-side edit. Mac apps with custom change-detection should disable that — the framework handles streaming.

## Common Mistakes

### 1. Assuming Spatial Preview works without a paired Vision Pro

`SpatialPreview.Destination.nearbyVisionPro()` resolves to a paired device at runtime. If no Vision Pro is paired, the call returns `nil` and the session start fails. Always guard the destination and surface a clear "Pair a Vision Pro to preview spatially" message rather than crashing. Spatial Preview does **not** fall back to the Mac display automatically.

### 2. Streaming large USD edits without debouncing

`updateScene(with:)` is a network handoff. Calling it on every keystroke during live authoring will saturate the Mac → Vision Pro link and stall the headset preview. Debounce edits (for example, coalesce on a 200 ms timer) and ship a single `updateScene` per debounce window.

### 3. Building a custom Storm-compatibility shim

RealityKit is the USD renderer inside Spatial Preview on the Vision Pro side (176839273). Do not add Tahoe-era Storm compatibility code "just in case"; it is dead weight. Validate assets against RealityKit's USD loader and document any Storm-era material quirks that need to be ported.

### 4. Treating SharePlay collaboration as a multi-screen sync problem

SharePlay inside Spatial Preview is a `GroupSession`-based collaboration surface, **not** a screen-sharing handoff. Participants see the same spatial scene with their own viewpoints; they do not see each other's headset view. Trying to "broadcast the headset view" to the group will fail because the system does not provide that surface.

### 5. Hardcoding the iPhone Mirroring window frame

On macOS 26 the mirrored iPhone window was effectively fixed-size. On macOS 27 it is resizable. Mac apps that snap other windows to the mirrored iPhone window's frame on Tahoe will mis-position those windows on macOS 27 because the mirrored window can now be any size. Stop hardcoding that frame; observe the actual `NSWindow` frame at runtime.

## References

- [Spatial Preview — Apple Developer documentation](https://developer.apple.com/documentation/SpatialPreview/)
- [What's new in macOS 27 — Spatial Preview](https://developer.apple.com/macos/whats-new/)
- [macOS 27 release notes](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes)
