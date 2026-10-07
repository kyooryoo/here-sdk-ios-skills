# HERE SDK for iOS — Threading, callbacks & lifecycle (shared core)

The rules below cause most generated-code bugs. They apply to **every**
`here-ios-*` skill.

## 1. Threading rules (verbatim-derived)

> "The HERE SDK is not guaranteed to be thread safe and it is required to
> make calls to the SDK from the main thread. Internally, the HERE SDK will
> offload most of its work to a background thread, but callbacks to your code
> will always occur on the main thread."

- **Call all SDK APIs from the main thread.** Do not touch engines, `MapView`,
  or map items from worker threads.
- **All callbacks/delegates arrive on the main thread** (`onLocationUpdated`,
  scene-load completion, search/route completions) — safe to update UI
  directly; no `DispatchQueue.main` hop needed (but it also does no harm).
- Thread safety of shared objects is the **caller's** responsibility; never
  reuse an engine across threads without synchronization.
- **Exception (Navigate):** `mapdata` loaders — `SegmentDataLoader`,
  `AdministrativeRulesLoader`, `SegmentReferenceConverter` — are synchronous
  and block the calling thread: "If called from the main thread, it will
  block the main thread." Run them on a **background thread**; each instance
  is single-thread only.
- **Exception 2:** `MapDownloader`/`MapUpdater` must be created via
  `fromEngineAsync(...)` (async) so UI isn't blocked; SVG rasterization for
  realistic views should be off the UI thread.

Source: <https://docs.here.com/here-sdk/docs/ios-tips-supplements.md>

## 2. Initialization order

1. `initializeHERESDK()` (creates the shared `SDKNativeEngine`) — synchronous,
   ~100 ms, **on the main thread**, once per app lifetime. In SwiftUI, do it
   in `App.init()`; nothing renders before it.
2. Create the `MapView` (`UIViewRepresentable.makeUIView`, storyboard, or
   `MapView(frame:)` in code).
3. `mapView.mapScene.loadScene(mapScheme:completion:)` — until a scene loads,
   the map is empty and the log shows
   `No map content will be displayed until valid config is provided.` The
   completion handler `(MapError?) -> Void` runs on the main thread; do
   camera setup inside it (or after `mapError == nil`).
4. Feature engines (`SearchEngine`, `RoutingEngine`, `LocationEngine`, …) are
   created **after** the shared engine exists — parameterless
   `try Engine()` constructors throw if SDK init failed beforehand.

There is **no `mapView.onCreate(...)` forwarding requirement on iOS**: "The
MapView listens for lifecycle events on its own, so usually, an application
does not need to do any cleanup or lifecycle handling on its own." Do forward
`didReceiveMemoryWarning` → `mapView.handleLowMemory()` (see §6).

## 3. App lifecycle & disposal (SwiftUI first)

```swift
@main
struct HelloMapApp: App {
    init() {
        observeAppLifecycle()      // willTerminate → dispose
        initializeHERESDK()        // once per app lifetime
    }
    private func observeAppLifecycle() {
        NotificationCenter.default.addObserver(forName: UIApplication.willTerminateNotification,
                                               object: nil, queue: nil) { _ in
            disposeHERESDK()
        }
    }
    private func disposeHERESDK() {
        // After this call the HERE SDK is no longer usable unless re-initialized.
        SDKNativeEngine.sharedInstance = nil
    }
}
```

- Disposal = `SDKNativeEngine.sharedInstance = nil`. It "doesn't do harm and
  is considered good practice" at app termination — but **do not dispose when
  only one screen goes away** while the rest of the app still uses the SDK.
  Init/dispose per screen is allowed if the whole screen owns the SDK.
- ⚠ **After disposal, every engine created with the parameterless
  constructor is dead** and must be recreated; `MapView` state becomes
  invalid (recreate the view; the engines page references an `onCreate(..)`
  re-init step and `mapView.isValid()` — verify current behavior via
  `here-ios-docs-navigation`).
- ⚠ **`SDKOptions` are not persisted** — re-supply scope, catalog
  configurations, layer configuration, cache path on every re-init.
- Navigation: `visualNavigator.stopRendering()` **before** the `MapView` is
  deinitialized; stop `LocationSimulator`/`DynamicRoutingEngine`; a paused
  (backgrounded) `MapView` pauses/resumes rendering automatically.

## 4. Engine lifecycle rules

- Create feature engines with their **parameterless throwing constructor** in
  `do/catch` (they attach to the shared `SDKNativeEngine`):
  `try SearchEngine()`, `try RoutingEngine()`, `try LocationEngine()`,
  `try VisualNavigator()`, `try TrafficEngine()`, …
- **One shared instance per app is the recommendation:**
  `SDKNativeEngine.makeSharedInstance(options:)` (disposes any previous
  shared instance internally). Multiple instances need **distinct access key
  IDs** (cache path is unique per key; duplicate key throws
  `InstantiationException`).
- Engines are independent of each other and of `MapView` (headless apps
  supported); multiple parallel instances of one engine type are allowed.
- Most async calls return a **`TaskHandle`** — status + cancel; ignore it if
  unneeded.
- Credential/scope/catalog settings are applied at init only; changing them
  means a new `SDKNativeEngine`.

## 5. Callback vs delegate conventions

- **Completion handler** = one-shot result (`searchByText`, `calculateRoute`,
  scene load). **Delegate/protocol** = recurring events (gestures, location,
  progress); multiple delegates use `addXDelegate(...)`/`removeXDelegate(...)`,
  single ones are properties assignable `nil`.
- By convention the SDK uses **protocols for recurring events and completion
  handlers for one-time events**.
- **Error/result XOR:** `error` and the result list "can never be nil at the
  same time - or non-nil at the same time." Guard the error first, then the
  list is guaranteed non-nil (safe to unwrap after checking).
- **Wrap every callback/delegate body in defensive handling** — an exception
  thrown app-side inside a callback is your crash, not the SDK's.
- Most delegate properties are `weak`; add/remove pairs still must be balanced
  for multi-delegate lists (`addLocationDelegate`/`removeLocationDelegate`).

## 6. Memory

- Observe `UIApplication.didReceiveMemoryWarningNotification`:

  ```swift
  SDKNativeEngine.sharedInstance!.purgeMemoryCaches(
      strategy: SDKNativeEngine.PurgeMemoryStrategy.full)   // only .full exists
  mapView.handleLowMemory()
  ```

- `SDKOptions.lowMemoryMode = true` (smaller caches, reduced performance) —
  set only **after** an OS low-memory warning, then re-init.
- `SDKNativeEngine.sharedInstance = nil` does **not** clear the disk cache;
  clear explicitly with
  `SDKCache.fromEngine(engine).clearCache { ... }`.

## 7. Cleanup checklist (teardown order)

1. Stop location sources (`locationEngine.stop()`, simulator `.stop()`).
2. `visualNavigator.route = nil` / nil single delegates; `remove*Delegate`
   for add/remove pairs; `warnerEngine.finalizeGivenWarnings()` when guidance
   stops.
3. Cancel outstanding `TaskHandle`s.
4. `visualNavigator.stopRendering()` (before MapView teardown).
5. Optionally dispose the SDK at app termination (`sharedInstance = nil`).
