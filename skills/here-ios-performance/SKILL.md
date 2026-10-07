---
name: here-ios-performance
description: >
  Optimize and harden a HERE SDK for iOS app: engine lifecycle and reuse,
  memory management and low-memory mode, frame-rate and rendering
  optimization, map cache tuning, data/OTA cost control with
  LayerConfiguration, package-size reduction, credential protection,
  privacy manifest, and usage-stats accounting. Use for performance reviews,
  cost audits, release hardening, or "app uses too much data/battery/binary
  is too big" tasks in a HERE iOS app.
---

# HERE SDK iOS — Performance, cost & release hardening

Sources: `ios-key-concepts`, `ios-tips-engines`, `ios-optimization`,
`ios-tips-supplements`, `ios-billing`. Shared rules:
`shared/threading-and-lifecycle.md`.

## Engine discipline

- **Initialize the SDK once per app lifetime**
  (`SDKNativeEngine.makeSharedInstance(options:)`); create feature engines
  lazily after init and keep them — "the creation of the `SDKNativeEngine`
  and other engines happens synchronously in a negligible amount of time",
  but when you re-init, **every engine created with the parameterless
  constructor must be recreated**, and each `MapView` must get `onCreate(..)`
  again (`mapView.isValid()` to check).
- Multiple parallel instances of *one* engine type are allowed; prefer
  reusing one. All engines except `SDKNativeEngine` operate independently of
  each other and of `MapView` (headless apps supported).
- ⚠ Multiple `SDKNativeEngine` instances **cannot share one access key ID**
  (cache path is unique per key — duplicate throws `InstantiationException`):
  "when you need multiple instances, you also need a distinct set of
  credentials for each."
- Cancel work you don't need: "most asynchronous methods provide a
  `TaskHandle` as immediate return value" (status + abort).
- Rate limits: engine returns `requestLimitReached` — "If you expect very
  high app usage, please talk to your HERE representative upfront."
- Conditional init: "initialize the HERE SDK only when it is necessary."
- Usage accounting: `SDKNativeEngine.sharedInstance?.enableUsageStats(enabled:
  true)` **before init each session** (not persisted) → `getSdkUsageStats()`
  = per-feature `NetworkStats` (`sentBytes`, `receivedBytes`,
  `requestCounter`, `methodCall`); retained **only for the current app
  session**. Account-level metering: platform usage dashboard
  <https://platform.here.com/management/usage> (`ios-billing`).

## Memory

```swift
NotificationCenter.default.addObserver(forName: UIApplication.didReceiveMemoryWarningNotification,
                                       object: nil, queue: .main) { _ in
    self.handleLowMemory()
}

func handleLowMemory() {
    SDKNativeEngine.sharedInstance!.purgeMemoryCaches(
        strategy: SDKNativeEngine.PurgeMemoryStrategy.full)
    var options = SDKOptions(authenticationMode: authenticationMode)
    options.lowMemoryMode = true
    try? SDKNativeEngine.makeSharedInstance(options: options)  // engines must be recreated after!
}
```

- Only `PurgeMemoryStrategy.full` is supported.
- `lowMemoryMode = true` = smaller caches at reduced performance — enable
  **only after an OS low-memory warning**, then restore.
- Free resources: `SDKNativeEngine.sharedInstance = nil`, nil all engine
  references; a disposed `SDKNativeEngine` invalidates every `MapView`.

## Rendering & CPU

| Lever | Setting |
|---|---|
| Frame rate | default **60 FPS**; `mapView.frameRate = 30` on low-end devices; `0` **deactivates automatic render cycles** (render on demand only); negative values have no effect; per-`MapView` |
| Custom styles | fewer style rules / lower-detail base style = less rendering load |
| Map features | disable every `MapFeature` you don't show — each costs rendering; see `here-ios-maps` |
| Map items | cluster large marker sets; batch additions; see `here-ios-maps` |
| Cache | even a headless app uses the map cache; tune `SDKOptions.cacheSizeInBytes` |
| Preload | offline regions (Navigate) avoid online fetch hitches (`here-ios-offline`) |
| Engine scoping | narrow `SearchOptions`/routing options so engines do less (`here-ios-search`) |

## Map cache (`ios-tips-engines`)

| Cache | Persistence | Eviction |
|---|---|---|
| Map tile cache (OCM: routing/search/display) | disk; `SDKOptions.cachePath` + `cacheSizeInBytes` | **LRU when full (Navigate); after one day (Explore)** |
| Satellite/raster tiles | disk; not configurable | after one day |
| Custom raster layer | disk; per-source `RasterDataSourceConfiguration.Cache` | per config |
| Offline regions (Navigate) | disk; `SDKOptions.persistentMapStoragePath` | manual deletion only |
| Decoded tiles/textures | memory | `purgeMemoryCaches` |

- Recommended cache size ≥ 256 MB (default), **512 MB–1 GB recommended**;
  Navigate "requires" a valid cache path (turn-by-turn needs zoom-14 tiles ≈
  2 km circle).
- `SDKOptions` cache expiry `0` = never store ⇒ no offline after connectivity
  loss. Setting shared instance to `nil` **does not** clear the cache; clear
  explicitly:

  ```swift
  SDKCache.fromEngine(SDKNativeEngine.sharedInstance!).clearCache { error in … }
  ```

- ⚠ Cache path is tied to the access key ID — rotating credentials makes the
  old cache look gone (it is not auto-cleared; it returns with the old key).

## Data & OTA cost (Navigate; `ios-optimization`)

Banner: "Managing data and OTA costs are only available with the Navigate
license." `SDKOptions.layerConfiguration` controls two lists:

- `enabledFeatures` — layers included in **region downloads**; "Only the
  features in this list will be enabled; all others will be disabled."
- `implicitlyPrefetchedFeatures` — layers fetched into the **cache while
  online panning**. **Default is exactly three: `detailRendering`,
  `navigation`, `rendering`.** `[]` = minimize network while panning.

```swift
var features: [LayerConfiguration.Feature] = [.detailRendering, .rendering, .offlineSearch]
options.layerConfiguration = LayerConfiguration(enabledFeatures: features)
options.layerConfiguration.implicitlyPrefetchedFeatures = features
```

- Off by default and opt-in only if used: `offlineSearchGlobal` (+11–16%
  size; drop `offlineSearch` alongside), `truck`, `junctionView3x4`/`16x9`
  (**each image up to 15 MB**), `junctionSign*` (≤300 KB each), `rdsTraffic`,
  `terrain`, `fuelStationAttributes`, `truckServiceAttributes`,
  `offlineBusRouting`. `rendering` is the base layer and **cannot be
  disabled**.
- Online-only apps: "`SearchEngine` or the `RoutingEngine` will never make
  use of cached or downloaded map data" — safe to disable `offlineSearch`/
  `offlineRouting`.
- ⚠ "Only the `LayerConfiguration` that was set globally when a `Region` was
  first downloaded or cached is used." Changing it later:
  `mapUpdater.retrieveCatalogsUpdateInfo` → `updateCatalog(...)` (or delete +
  re-download), and with a `MapView` a **full SDK re-init + MapView
  recreation** (`resetMapView()`), engines recreated, `MapUpdater` re-fetched
  via `MapUpdater.fromEngineAsync`.
- Other cost levers: traffic refresh period (`here-ios-traffic`),
  `SearchOptions.maxItems`, `RoutePrefetcher` (10×5 km corridor) instead of
  ad-hoc downloads, `isOfflineMode`/`PassThroughFeature` for radio-silent
  mode (`here-ios-offline`).

## Package size (iOS)

`heresdk.xcframework` = device (arm64) + simulator (x86_64) frameworks +
device dSYMs — Xcode strips unused slices at deploy.

- Remove unused **voice packages** from
  `heresdk.framework/ios-arm64/voice_assets` (Navigate; e.g.
  `voice_package_ar-SA`).
- Remove unused **fonts** in `assets/geoviz/fonts/` (`DroidSansFallback.ttf`,
  `NotoSansJP-Regular.otf`, …; max ≈ 11 MB) — ⚠ **keep at least one font** or
  map labels break; a missing-font error is logged **per character**.
- Remove unused **truck restriction icons**
  (`assets/geoviz/assets/oslo/truck_restriction/{night,day}/ui/`, ≈ 264 KB)
  if `IconProvider.createVehicleRestrictionIcon()` and
  `MapFeatures.vehicleRestriction` are unused.
- dSYMs ≈ 80% of XCFW size but "symbolized crash stacks are treated with
  higher priority" — keep them.

## Credentials, privacy & release safety

- Never commit key secrets; no plaintext — Keychain/Secure Enclave, TLS
  fetch from backend, anti-tampering/anti-debugging if abuse is a concern.
- Access tokens: regenerated on each init; refresh via
  `Authentication.authenticate(SDKNativeEngine.sharedInstance, callback)` →
  `authenticationData.token` for external HERE REST APIs.
- **Objective-C is NOT supported** (no bridging headers); Swift only.
- Privacy manifest ships in the framework since 4.18.2.0
  (`PrivacyInfo.xcprivacy`); Navigate's declares collected **coarse + precise
  location** (Linked=false, Tracking=false, AppFunctionality).
- Ship `HERE_NOTICE` attribution + required privacy telemetry text + visible
  watermark (`shared/credentials-and-licensing.md` §6).
- Unit tests: "It is not necessary to initialize the HERE SDK when writing
  unit tests" (pure value types work standalone — see `UnitTesting`
  example).
- IPA: Release scheme → Clean Build Folder → Archive → Distribute.

## Verify before delivering (release pass)

- [ ] SDK initialized once; engines reused; no delegate/handle leaks
- [ ] `didReceiveMemoryWarningNotification` wired; low-memory strategy decided
- [ ] `mapView.frameRate` tuned for device tier; unused `MapFeature`s off
- [ ] `LayerConfiguration` audited (both lists); offline features match the
      product's offline story
- [ ] Cache size/path/expiry chosen deliberately (≥ 512 MB for Navigate apps)
- [ ] Unused voice/font/truck-icon assets excluded; fonts not fully stripped
- [ ] Secrets out of VCS; NOTICE + privacy text + watermark + privacy manifest
      present
- [ ] `enableUsageStats` reviewed in a dev build to find network hot spots
- [ ] Anything beyond this page verified via `here-ios-docs-navigation`
