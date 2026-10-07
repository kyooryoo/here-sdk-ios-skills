---
name: here-ios-positioning
description: >
  Get device locations with HERE Positioning in an app built with the HERE SDK
  for iOS: LocationEngine setup and start/stop, location and status delegates,
  Info.plist location permissions and background modes, accuracy/power modes,
  showing the blue dot with LocationIndicator, map matching with MapMatcher,
  location filtering, and route/GPX simulation. Use for any GPS, location
  tracking, blue-dot, location-updates, or positioning task with the HERE SDK
  for iOS (here-ios-positioning), typically feeding here-ios-navigation.
---

# HERE SDK iOS — Positioning

⚠ **Navigate license only.** Every positioning page carries this banner
(`ios-get-locations`, `ios-get-locations-optimization`,
`ios-hiking-app-tutorial`); Explore credentials fail with a dedicated error
enum at engine use — see `shared/credentials-and-licensing.md` §3.
Prereqs: `here-ios-bootstrap` (SDK init first). All calls and callbacks are
**main-thread** (`shared/threading-and-lifecycle.md`). Map display:
`here-ios-maps`; consuming these locations in guidance: `here-ios-navigation`.
Sources: `ios-positioning-overview`, `ios-get-locations`,
`ios-get-locations-optimization`, `ios-hiking-app-tutorial`,
`ios-navigation-optimization`.

## Permissions (Info.plist)

```xml
<key>UIRequiredDeviceCapabilities</key>
<array>
   <string>location-services</string>
   <string>gps</string>
</array>
<key>NSLocationWhenInUseUsageDescription</key>
   <string>This app needs to access your current location to display it on the map.</string>
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
   <string>This app needs to access your current location to display it on the map.</string>
<key>NSMotionUsageDescription</key>
   <string>Motion detection is needed to determine more accurate locations, when no GPS signal is found or used.</string>
```

- `NSLocationAlwaysAndWhenInUseUsageDescription` needed **only** for
  background updates; `NSMotionUsageDescription` for sensor-assisted fixes.
- Runtime check via `CLLocationManager.authorizationStatus()`: `.denied` /
  `.restricted` ⇒ prompt won't show; `.notDetermined` ⇒
  `locationManager.requestWhenInUseAuthorization()`.
- ⚠ Even recent iPads can lack the `gps` capability: keeping it in
  `Info.plist` blocks install on such devices; removing it installs but
  locations fall back to network positioning with a larger accuracy radius.
- Privacy: "As of now, on iOS, the HERE SDK will not collect any data" — no
  HERE Privacy Notice reference needed for `LocationEngine`; Network
  Positioning uses Network Positioning API v2.

## Engine lifecycle

```swift
do {
    try locationEngine = LocationEngine()          // throws if SDK init failed
} catch let engineInstantiationError {
    fatalError("Failed to initialize LocationEngine. Cause: \(engineInstantiationError)")
}

private func startLocating() {
    locationEngine.addLocationStatusDelegate(locationStatusDelegate: self)
    locationEngine.addLocationDelegate(locationDelegate: self)
    if locationEngine.start(locationAccuracy: .bestAvailable) == .missingPermissions {
        requestLocationAuthorization()
    }
}

public func stopLocating() {
    locationEngine.removeLocationDelegate(locationDelegate: self)
    locationEngine.removeLocationStatusDelegate(locationStatusDelegate: self)
    locationEngine.stop()
}
```

- `start(locationAccuracy:)` returns a `LocationStatus` (e.g.
  `.missingPermissions`).
- `LocationStatusDelegate.onStatusChanged(locationEngineStatus:)` receives
  `.engineStarted` after a successful start, `.engineStopped` after stop.
- ⚠ **Only one engine can be started at a time** — restart without `stop()`
  or a second engine ⇒ `.alreadyStarted`. Query with `isStarted()`.
- "In general, it is recommended to stop the `LocationEngine` when an app
  gets disposed."

## Receiving locations

```swift
class PositioningExample: LocationDelegate {
    func onLocationUpdated(location: Location) {
        print("Location updated: \(location.coordinates)")
    }
}
// LocationStatusDelegate also offers:
func onFeaturesNotAvailable(features: [LocationFeature]) { … }   // "pending feature"
```

- ⚠ "The callback `onLocationUpdated()` is received on the main thread -
  same as for all other callbacks." — no queue hop needed for UI.
- Multiple delegates allowed: repeated `addLocationDelegate` /
  `addLocationStatusDelegate` (balance with removes).
- Only `coordinates` + `timestamp` are guaranteed on `Location`; `bearing`,
  `speed`, etc. may be `nil`. Downstream map matching **requires**
  `Location.time` — preserve it (plus bearing/speed) for navigation.
- `lastKnownLocation`: available **without** starting the engine or
  listeners, as long as the engine ran once in a previous session and ≥1
  valid event was received; persists across app sessions.

## Accuracy vs. battery (`LocationAccuracy` → `CLLocationAccuracy`)

| `LocationAccuracy` | Maps to | Use |
|---|---|---|
| `.bestAvailable` | `kCLLocationAccuracyBest` | default |
| `.navigation` | `kCLLocationAccuracyBestForNavigation` | **feed guidance** (~1 Hz) |
| `.tensOfMeters` | `kCLLocationAccuracyNearestTenMeters` | low power |
| `.hundredsOfMeters` | `kCLLocationAccuracyHundredMeters` | low power |
| `.kilometers` | `kCLLocationAccuracyThreeKilometers` | coarse |

- Sub-meter accuracy is **not supported on iOS**; **HD GNSS is not available
  even with Navigate** (Android/Flutter only).
- `horizontalAccuracyInMeters` = uncertainty radius at 68 % (CEP68);
  CEP50 = 0.78×, CEP90 = 1.42×, CEP95 = 1.62×, CEP99 = 2.01× CEP68.
  Altitude is MSL; same 68 % rule for vertical/bearing/speed accuracy.
- ⚠ By default the engine **auto-pauses updates while stationary** —
  control with `LocationEngine.setPauseLocationUpdatesAutomatically()`;
  pass `false` to keep updates running when not moving.

## Show the blue dot

```swift
locationIndicator = LocationIndicator()
locationIndicator.isAccuracyVisualized = true       // MapCircle = CEP68 radius
locationIndicator.locationIndicatorStyle = .pedestrian   // or .navigation
locationIndicator.updateLocation(myLocation)
locationIndicator.enable(for: mapView)
mapCamera.lookAt(point: myLocation.coordinates, zoom: MapMeasure(kind: .distanceInMeters, value: 250))
```

- Style/enable/disable details: `here-ios-maps`. `VisualNavigator
  .startRendering` drives its own indicator during guidance — don't fight it
  with a second one.

## Map matching (positioning side)

- Standalone: `MapMatcher(sdkEngine:useRenderingLayers:)` →
  `match(location:)` → `MapMatchedLocation`? — **beta**, matches within a
  **50 m radius**; returns `nil` when off-road, `Location.time` is missing,
  or tiles aren't cached (missing tiles trigger a silent background download,
  `nil` returned immediately).
- During guidance/tracking the `VisualNavigator`/`Navigator` does matching
  internally ("All events are given based on a map-matched location") — you
  don't call `MapMatcher` yourself (`here-ios-navigation`).

## Background location

```xml
<key>UIBackgroundModes</key>
<array>
    <string>location</string>
    <string>processing</string>   <!-- iOS 13.0+ -->
</array>
<!-- iOS 13.0+ also: -->
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
    <string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
</array>
```

- Background updates are **off by default**: `locationEngine
  .setBackgroundLocationAllowed(allowed: true)` +
  `.setBackgroundLocationIndicatorVisible(...)` — both return
  `.notAllowed` if the capabilities are missing, else `.ok`.
- Alternative: enable in Xcode **Signing & Capabilities → Background Modes →
  Location updates + Background processing**.
- Combine with `setPauseLocationUpdatesAutomatically(false)` so stationary
  fixes keep flowing in the background.

## Filtering, simulation & GPX

- ⚠ GPS noise while stationary shows "traveling miles while sitting at home"
  — filter fixes: implement `LocationFilterStrategy`
  (`checkIfLocationCanBeUsed(_:)`); example `DistanceAccuracyLocationFilter`
  uses `accuracyRadiusThresholdInMeters = 10.0`,
  `distanceThresholdInMeters = 15.0`; bypass with `DefaultLocationFilter`
  (returns `true`). Full class in `ios-hiking-app-tutorial`.
- Simulation: `LocationSimulator(route:options:)` replays a `Route` **or GPX
  track** (`LocationSimulatorOptions(speedFactor:notificationInterval:)`,
  defaults 1.0 / 1 s); details and playback-switching recipes:
  `here-ios-navigation`.
- GPX **recording** uses `GPXTrackWriter` + `GPXManager(gpxDocumentFileName:)`
  + `GPXDocument` — from GitHub example code, **not core SDK API**.

## Verify before delivering

- [ ] Navigate license confirmed (`shared/credentials-and-licensing.md` §3)
- [ ] Info.plist keys present: `NSLocationWhenInUseUsageDescription` (+ Always
      only if background), `NSMotionUsageDescription`; `gps` capability
      decision documented
- [ ] Runtime authorization requested before `start()`; `.missingPermissions`
      branch handled
- [ ] Exactly one started engine; `remove*Delegate` + `stop()` on teardown;
      no engine creation before SDK init
- [ ] Accuracy mode fits the use case (`.navigation` for guidance,
      low-power modes for background; no sub-meter/HD expectations on iOS)
- [ ] Stationary pausing understood (`setPauseLocationUpdatesAutomatically`)
- [ ] `Location.time` (+ bearing/speed) preserved for any navigation/map-match
      consumer; nil optional fields guarded
- [ ] Background usage: `setBackgroundLocationAllowed` returns `.ok` and
      `UIBackgroundModes` present, else gracefully degraded
- [ ] Any signature beyond this page verified via `here-ios-docs-navigation`
