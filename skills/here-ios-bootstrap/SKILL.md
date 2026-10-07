---
name: here-ios-bootstrap
description: >
  Create, set up, or configure a HERE SDK for iOS (4.x) project: manual
  heresdk.xcframework integration (no SPM/CocoaPods), credentials and
  SDKNativeEngine initialization, adding a MapView in SwiftUI or UIKit,
  loading the first map scene, CarPlay, and fixing a blank map. Always load
  first — every other here-ios-* skill builds on this initialization.
---

# HERE SDK iOS — Bootstrap

Emit this initialization **exactly once** per app. Shared rules:
`shared/credentials-and-licensing.md` (keys, Explore vs Navigate, legal),
`shared/threading-and-lifecycle.md` (main-thread + lifecycle law).

## 1. Project prerequisites

- **iOS/iPadOS 15.2+**; **Xcode 16.2 minimum** (examples built with 16.4);
  simulator iOS 15.2+ supported.
- Devices: arm64 (iPhone 5s and newer); simulators: x86_64 **and**
  simulator-arm64 (M1+ works).
- **Swift only** — the SDK cannot be called from Objective-C (Swift files
  inside an Obj-C project are fine; no bridging headers, no Xamarin, no
  React Native support).
- Disk: ~60 MB app + **256 MB** map-data cache recommended (≥ 512 MB–1 GB
  preferred). Real device recommended over simulator (map rendering is tuned
  for mobile GPUs); headless engines need no GPU.

### Dependency — manual XCFramework (the only route)

```text
no CocoaPods · no Swift Package Manager · no Carthage
```

1. Download from <https://platform.here.com/portal/sdk>.
2. Copy `heresdk.xcframework` into the app root.
3. Xcode → Target → **General → Frameworks, Libraries, and Embedded Content
   → + → Add other → Add files…** → `heresdk.xcframework`.
4. `import heresdk` — **lowercase module name**.

Package contents: `heresdk.xcframework` (slices `ios-arm64`,
`ios-arm64_x86_64-simulator`, plus dSYMs ≈ 80 % of package size — keep them,
they symbolicate native crashes), documentation, examples, and the mandatory
**`HERE_NOTICE`** file (must ship with the app).

## 2. Initialize the SDK (main thread, in `App.init()`)

```swift
import heresdk
import SwiftUI

@main
struct HelloMapApp: App {
    init() {
        observeAppLifecycle()
        initializeHERESDK()      // once per app lifetime, synchronous (~100 ms)
    }

    var body: some Scene { WindowGroup { ContentView() } }

    private func observeAppLifecycle() {
        NotificationCenter.default.addObserver(forName: UIApplication.willTerminateNotification,
                                               object: nil, queue: nil) { _ in
            disposeHERESDK()
        }
    }

    private func initializeHERESDK() {
        let accessKeyID = "YOUR_ACCESS_KEY_ID"
        let accessKeySecret = "YOUR_ACCESS_KEY_SECRET"
        let authenticationMode = AuthenticationMode.withKeySecret(accessKeyId: accessKeyID,
                                                                  accessKeySecret: accessKeySecret)
        var options = SDKOptions(authenticationMode: authenticationMode)
        // options.scope = "hrn:here:authorization::…:project/xxxx"  // only for multi-app credential sharing
        do {
            try SDKNativeEngine.makeSharedInstance(options: options)
        } catch let e as InstantiationErrorCode {
            fatalError("Initialization of HERE SDK failed: \(e.localizedDescription)")
        }
    }

    private func disposeHERESDK() {
        // Only at app termination — engines/views die with this instance.
        SDKNativeEngine.sharedInstance = nil
    }
}
```

- **Init succeeds even with invalid credentials** — verify by running a
  feature engine and checking its error enum
  (`shared/credentials-and-licensing.md` §2, `shared/threading-and-lifecycle.md` §2).
- Initialize once per app lifetime; dispose only when the whole app goes
  away. Never dispose because one screen disappeared.

## 3. Add the map view

### SwiftUI (`MapView` is a `UIView` → wrap it)

```swift
struct MapViewUIRepresentable: UIViewRepresentable {
    func makeUIView(context: Context) -> MapView {
        return MapView()
    }
    func updateUIView(_ mapView: MapView, context: Context) {
        // Load a map scene here — nothing renders without it.
        mapView.mapScene.loadScene(mapScheme: MapScheme.normalDay, completion: onLoadScene)

        func onLoadScene(mapError: MapError?) {
            guard mapError == nil else {
                print("Error: Map scene not loaded, \(String(describing: mapError))")
                return
            }
            let camera = mapView.camera
            let distanceInMeters = MapMeasure(kind: .distanceInMeters, value: 1000)
            camera.lookAt(point: GeoCoordinates(latitude: 52.517543, longitude: 13.408991),
                          zoom: distanceInMeters)
        }
    }
}
```

### UIKit / storyboard

`MapView(frame: view.bounds)` + `view.addSubview(mapView)` (see the
`HelloMapWithStoryboard` example); CarPlay uses exactly this code path.

- The scene-load completion runs on the **main thread**; any non-nil
  `MapError` means the load failed.
- The camera can be set as soon as the view exists, but changes become
  visible only after the scene-load completion fires.
- Multiple `MapView` instances on one screen are supported.
- "The MapView listens for lifecycle events on its own" — no
  view-lifecycle forwarding needed on iOS; do forward
  `didReceiveMemoryWarning` → `mapView.handleLowMemory()`.

## 4. First scene + features + language

```swift
mapView.mapScene.enableFeatures([MapFeatures.landmarks: MapFeatureModes.landmarksTextured])
MapView.primaryLanguage = "ja"          // all MapView instances; nil = region default
```

- Reloading a scene **resets all `MapFeatures` to defaults** — re-enable
  them after every `loadScene`.
- Until a scene loads the map is blank (triage in
  `here-ios-troubleshooting`).
- Some layers are **Navigate-only** (`landmarks`, `terrain`, `contours`,
  `vehicleRestrictions`, `publicTransit`, `safetyCameras`, …) — see
  `here-ios-maps`.

## 5. Multiple apps using one credential set

```swift
sdkOptions.scope = "YOUR_PROJECT_HRN"   // BEFORE makeSharedInstance
```

Wrong scope ⇒ all auth fails; missing service/catalog linkage ⇒
`403: Not authorized to access the resources`. See
`shared/credentials-and-licensing.md` §4.

## 6. CarPlay (quick facts)

- No HERE-specific setup — follow Apple's CarPlay docs; HERE compatibility
  "is not guaranteed" (dependency not managed by the HERE SDK team).
- Entitlement `com.apple.developer.carplay-maps` in `Entitlements.plist`
  (Build Settings → Code Signing Entitlements) + Apple's entitlement review;
  until approved, test in the **CarPlay simulator** (Simulator menu →
  I/O → External Displays → CarPlay).
- `Info.plist`: `UIApplicationSceneManifest` with a
  `CPTemplateApplicationSceneSessionRoleApplication` entry pointing at your
  `CarPlaySceneDelegate` (`CPTemplateApplicationSceneDelegate`), plus
  `UILaunchScreen` (else compatibility mode).
- Create the head-unit map in code:
  `mapView = MapView(frame: view.bounds)`; **no map gestures on CarPlay** —
  interact via `CPBarButton`s (`zoomBy(2, around: camera.principalPoint)`).
- Low memory: `mapView.handleLowMemory()` in `didReceiveMemoryWarning`.
- Example: `examples/latest/navigate/ios/HelloMapCarPlay`.

## Verify before delivering

- [ ] `heresdk.xcframework` embedded (Frameworks/Libraries/Embedded Content)
      and `import heresdk` compiles; no SPM/CocoaPods added for HERE
- [ ] `initializeHERESDK()` on the main thread, once, in `App.init()` (or
      equivalent); credentials not committed
- [ ] `loadScene(mapScheme:completion:)` called and its `MapError` branch
      handled; camera set inside/after the success callback
- [ ] `MapFeatures` re-enabled after any scene reload
- [ ] `SDKNativeEngine.sharedInstance = nil` only on `willTerminate`;
      `SDKOptions` re-supplied if the SDK is ever re-initialized
- [ ] Explore/Navigate license checked for the features the app uses
      (`shared/credentials-and-licensing.md` §3)
- [ ] `HERE_NOTICE` shipped, required privacy text in the privacy policy,
      HERE watermark left visible
- [ ] Blank-map triage done: credentials/internet/**device time** →
      `loadScene` called? → module/framework integrated? → see
      `here-ios-troubleshooting`
- [ ] Anything beyond this page verified via `here-ios-docs-navigation`
