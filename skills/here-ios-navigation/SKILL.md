---
name: here-ios-navigation
description: >
  Build turn-by-turn navigation with the HERE SDK for iOS: Navigator and
  VisualNavigator setup, route progress and maneuver events, voice
  guidance/TTS, map matching, route deviation and returnToRoute rerouting,
  warners (speed, safety camera, truck restrictions), lane assistance,
  Electronic Horizon, truck guidance, and performance tuning. Use for any
  turn-by-turn, guidance, in-car HUD, or tracking-mode task with the HERE
  SDK for iOS (here-ios-navigation).
---

# HERE SDK iOS — Navigation

⚠ **Navigate license only** — banner on every navigation page
(`shared/credentials-and-licensing.md` §3). Sources: `ios-navigation`,
`ios-navigation-concepts`, `-voice-guidance`, `-deviation`, `-warners`,
`-lane-guidance`, `-truck`, `-optimization`, `ios-electronic-horizon`,
`ios-navigation-app-tutorial`. Prereqs: `here-ios-bootstrap`,
`here-ios-routing` (need a `Route`), `here-ios-positioning` (location source),
map = `here-ios-maps`. All calls and **all callbacks on the main thread**
(`shared/threading-and-lifecycle.md`); completions follow the **XOR error/result
contract** — guard `error` first, then unwrap the result.

## Pick the entry point

| Need | Use |
|---|---|
| Full HUD: route line, camera, interpolated location, 3D arrow | **`VisualNavigator`** (recommended) |
| Headless guidance (your own rendering) | `Navigator` (subset of `VisualNavigator`'s interfaces) |
| Tracking without a route ("driver's assistance mode") | either, `route = nil` |

Both conform to `LocationDelegate`; there is **no `startNavigation(...)`** —
"Starting the location source after setting the route starts guidance."
Guidance and tracking are unsupported for `PUBLIC_TRANSIT` (all other modes OK).

## Start guidance

```swift
private func startGuidance(route: Route) {
    do { try visualNavigator = VisualNavigator() } catch let e { fatalError("…: \(e)") }
    visualNavigator!.startRendering(mapView: mapView)   // route drawing, camera, interpolation
    visualNavigator!.routeProgressDelegate = self       // optional delegates
    visualNavigator!.eventTextDelegate = self           // voice/maneuver text
    visualNavigator!.route = route                      // ← leaves tracking; guidance starts…
    herePositioningProvider.startLocating(locationDelegate: visualNavigator!,
                                          accuracy: .navigation)   // …when locations flow
}
```

- ⚠ Locations need `Location.time` (**required** — else map matching ignores them)
  plus `bearing` and `speed`; "The `Navigator` API reference recommends at least
  one update per second" → `LocationAccuracy.navigation` (~1 Hz). Omit the route
  (or set it later) for tracking mode.
- `stopRendering()` **before** deinitializing `MapView`; also stop
  `LocationSimulator`/`DynamicRoutingEngine`. A paused `MapView` pauses/resumes
  rendering automatically. After `stopRendering()` the camera keeps its last state
  but `RouteProgress` keeps firing until delegates are nil'd — shutdown is
  app-owned: location source → delegates → engines → `stopRendering()` → `MapView`.

## Route progress, maneuvers & milestones

```swift
func onRouteProgressUpdated(_ routeProgress: RouteProgress) {
    let last = routeProgress.sectionProgress.last!   // guaranteed non-empty; distance + trafficDelay
    guard let next = routeProgress.maneuverProgress.first else { return }
    let maneuver = visualNavigator.getManeuver(index: next.maneuverIndex)  // roadTexts, exitSignTexts…
}
```

- ⚠ "As long as you navigate, do not take the `Maneuver` data from the `Route`
  object directly" — navigator maneuvers are synced to your current location.
- ⚠ Source split: `roadTexts`/`nextRoadTexts`/`exitSignTexts` are **empty on a
  `Route`**, non-empty from the navigator; `maneuver.text` is non-empty from both.
  Localize via `names.defaultValue()` / `names.preferredValue(for: [locale])`;
  `RoadTextsDelegate` gives the current road in tracking mode.
- ⚠ `sectionProgress.trafficDelay` is computed at calculation time and **not
  refreshed during guidance** — refresh ETA every 5–15 min:
  `routingEngine.calculateTrafficOnRoute(currentRoute:lastTraveledSectionIndex:
  traveledDistanceOnLastSectionInMeters:)` → `visualNavigator.trafficOnRoute`
  (applies to the next `RouteProgress`; may be charged).
- Destination/milestones: `DestinationReachedDelegate.onDestinationReached()`;
  `MilestoneStatusDelegate.onMilestoneStatusUpdated(milestone:status:)` — `Milestone`
  = stopovers + destination + SDK-added waypoints (e.g. ferry), start excluded,
  `waypointIndex == nil` ⇒ system waypoint; status = `reached`/`missed`. ⚠
  Overlapping start sections fire spurious `missed` — recalculate on `missed`. At
  destination turn-by-turn stops but **warners keep firing** — set a new route,
  switch to tracking, or shut down.

## Tracking, map matching & stopping

- Stop guidance but keep map-matched position + warnings: `visualNavigator
  .route = nil`; restart with a new `Route`. Only route-independent delegates fire
  in tracking (`NavigableLocationDelegate` → map-matched location, speed,
  `isDrivingInTheWrongWay`; `SpeedWarningDelegate`; warners). Recommended:
  `visualNavigator.cameraBehavior = SpeedBasedCameraBehavior()`. Stop completely:
  nil each delegate individually, `stopRendering()`, stop the location provider.
- Map matching is automatic via the navigator's internal `MapMatcher` — "All
  events are given based on a map-matched location." Standalone (beta):
  `MapMatcher(sdkEngine:useRenderingLayers:)` → `match(location:)` →
  `MapMatchedLocation`? within 50 m (`here-ios-positioning`).

## Deviation & rerouting

```swift
func onRouteDeviation(_ routeDeviation: RouteDeviation) {
    guard let route = visualNavigator.route else { return }
    // currentLocation.mapMatchedLocation may be nil → fall back to originalLocation.coordinates
    routingEngine.returnToRoute(originalRoute, startingPoint: newStartingPoint,
        lastTraveledSectionIndex: routeDeviation.lastTraveledSectionIndex,
        traveledDistanceOnLastSectionInMeters: routeDeviation.traveledDistanceOnLastSectionInMeters) { (routingError, routes) in
        if routingError == nil { let newRoute = routes?.first /* XOR: safe */ } else { /* handle error */ }
    }
}
```

- ⚠ **"The HERE SDK does not recalculate routes automatically, it only notifies on
  the deviation distance."** `RouteDeviation` fires **per location update** —
  "await around 3 events before deciding on possible actions."
- Options: (1) full re-plan with `RoutingEngine` (set bearing on the first
  `Waypoint` from current location), (2) **`returnToRoute(...)`** — general
  recommendation; online engine needs a `RouteHandle` on the original route else
  `NO_ROUTE_HANDLE`; returns one route, preserves `OptimizationMode`, (3)
  `refreshRoute()` + `RouteHandle` — same path only, **not sufficient alone for
  deviation**, (4) `DynamicRoutingEngine` — better-route suggestions, not the
  deviation tool. Optional: `newStartingPoint.headingInDegrees =
  currentMapMatchedLocation.bearingInDegrees`. ⚠ Stale old-route events may
  arrive once after swapping routes — attach new delegates after setting the route.

## Voice guidance (bring your own TTS)

⚠ **"the HERE SDK does not include pre-recorded voice skins… you need to integrate
a TTS engine for playback"** (example helper wraps Apple's `AVSpeechSynthesizer`).

```swift
visualNavigator.eventTextDelegate = self
func onEventTextUpdated(_ eventText: heresdk.EventText) {
    voiceAssistant.speak(message: eventText.text)   // .maneuver type → maneuverNotificationDetails?.maneuver
}
```

- Set `maneuverNotificationOptions` **before a route is set** (else defaults
  `en-US`/metric): `language` (`LanguageCode`), `unitSystem`,
  `includedNaturalGuidanceTypes` (e.g. `.trafficLight`, **off by default**),
  `includedNotificationTypes`, `enablePhoneme`, `notificationFormatOption = .ssml`.
- 37 languages via `VisualNavigator.availableLanguagesForManeuverNotifications()`;
  voice assets ship in framework folder `voice_assets` (unused `voice_package_*`
  deletable to shrink the package). Timing order `range → reminder → distance →
  action` via `get/setManeuverNotificationTimingOptions(transportMode:timingProfile:)`;
  default first car-highway (`range`) notice **1300 m** ahead. Phoneme SSML may not
  be supported by all TTS engines.
- Lane hints in TTS: `maneuverNotificationOptions.enableLaneRecommendation = true`
  — `distance` type only, threshold default **18 s**. Audio focus/ducking: **no SDK
  API — the app decides**. Spatial audio: `EventTextOptions.enableSpatialAudio` +
  `SpatialNotificationDetails` (example: SpatialAudioNavigation).

## Warners (unified `WarnerEngine` since 4.26)

```swift
let warnerEngine = visualNavigator.warnerEngine
let enabledWarnings: [WarningType] = [.safetyCamera, .truckRestriction, .roadSign,
    .schoolZone, .realisticView, .borderCrossing, .dangerZone, .railwayCrossing,
    .lowSpeedZone, .trafficMerge, .tollStop, .laneDecrease]
warnerEngine.setEnabledWarnings(warningTypes: enabledWarnings)     // ← REQUIRED gate
warnerEngine.addWarningListener(warningListener: myWarnerHandler)  // WarningDelegate
// onWarnings(warnings:warningsRegistry:) → switch warning.warningType →
//   registry.getSafetyCameraWarning(warning:) … (typed lookup)
```

- ⚠ **Without `setEnabledWarnings(...)` no warnings are ever delivered**,
  regardless of listeners. Warners work **without a route** (tracking) and
  continue after leaving the route/destination. Also `.environmentalZone`,
  `.custom`; positional warners go `ahead → approaching → reached → passed`.
- Distances (default slow 500 / regular 750 / fast 1000 m):
  `get/setWarningNotificationDistances(warningType:warningNotificationDistances:)`.
- `SpeedWarningDelegate` + `SpeedLimitDelegate` live **on the navigator, NOT in
  `WarnerEngine`**: `.speedLimitExceeded`/`.speedLimitRestored` once per
  transition (example plays `AudioServicesPlaySystemSound(SystemSoundID(1016))`);
  offsets via `visualNavigator.speedWarningOptions = SpeedWarningOptions(speedLimitOffset: SpeedLimitOffset(...))`.
- Safety-camera TTS needs `SafetyCameraWarningOptions.enableTextNotification = true`
  + `EventTextListener`; coordinates only in non-tracking mode (coverage excludes
  FR/DE). Custom warners: `CustomWarningProvider` via `addCustomWarningProvider(...)`
  **before** `setEnabledWarnings`, include `.custom`. `warnerEngine.finalizeGivenWarnings()`
  on guidance stop. Realistic view (SVG junctions) = beta; needs third-party SVG
  renderer + `SignpostFonts.zip` fonts (`UIAppFonts`); rasterize **off the UI thread**.

## Lane assistance (visual only — no TTS)

⚠ **"When no `Route` is set, no lane assistance is provided"** — except
`CurrentSituationLaneView` (current-road lanes, works in tracking mode).

| Delegate | Fires |
|---|---|
| `ManeuverViewLaneAssistance` | lanes for the next maneuver's junction |
| `JunctionViewLaneAssistance` | **complex junctions only**; empty list = junction passed |
| `CurrentSituationLaneAssistanceViewDelegate` | current road lanes (tracking OK) |

- Events 300 m ahead (non-highway) / 1300 m (highway). `Lane`: `directions`,
  `directionsOnRoute`, `recommendationState` (`.recommended`/`.highlyRecommended`/
  `.notRecommended`), `directionCategory`, `type`, `access`, `laneMarkings`. ⚠
  Ordered **leftmost (index 0) → rightmost in both driving sides**; current driving
  lane not highlighted; hide stale info when `previousManeuverIndex != nextManeuverIndex`.

## Truck & Electronic Horizon

- Truck: route with `TransportMode.truck` + `VehicleSpecification` +
  `HazardousMaterial`; `setEnabledWarnings([.truckRestriction])` →
  `registry.getTruckRestrictionWarning(...)` (`weightRestriction` /
  `dimensionRestriction` / both nil = "no trucks allowed", `timeRule`). ⚠ Stopping
  guidance converts pending `ahead` restrictions to `passed`; off-route > 15 m
  re-enables them; warnings arrive regardless of transport mode — filter app-side.
  Tracking: `navigator.trackingTransportSpecification`; CVR limits need map v32+.
- Electronic Horizon (**Navigate, beta** — APIs may change without deprecation; no
  ADASIS): `ElectronicHorizonEngine(sdkEngine:options:transportMode:route:)` +
  `ElectronicHorizonOptions(lookAheadDistancesInMeters:trailingDistanceInMeters:)`
  (values > 0, not clamped) → `update(mapMatchedLocation:)` — ⚠ **input must be
  map-matched** (unlike `VisualNavigator`, which accepts raw `Location`s) →
  `onElectronicHorizonUpdated(error:update:)` (XOR) →
  `ElectronicHorizonDataLoader(SegmentDataLoaderOptions…)` (status per path level,
  0 = MPP; cache ≈ 1000 or "not found in cache" errors) → `SegmentData`. Emits no
  warnings itself — feeds custom warners; remove both listeners on cleanup.

## Performance & UX tuning

- `startRendering(mapView:)` = smooth interpolation, tilt, heading, 3D arrow,
  `LocationIndicator`; camera tracking **on by default** — release with
  `visualNavigator.cameraBehavior = nil`, re-enable `FixedCameraBehavior()` /
  `DynamicCameraBehavior()`. Without interpolation the map "jumps"; alternative:
  `InterpolatedLocationListener`.
- ⚠ MapView renders 60 fps but turn-by-turn drops to **30 fps** — restore with
  `visualNavigator.guidanceFrameRate = 60`. Route styling:
  `isRouteProgressVisible = false`, `VisualNavigatorColors.dayColors()`,
  `RouteProgressColors(...)`, `maneuverArrowColor`. Event frequency ≈ location
  frequency (~1 Hz at `.navigation`); deviation/road-text/milestone events are
  exception-driven.
- `DynamicRoutingEngine` (online + route handle): `DynamicRoutingEngineOptions(minTimeDifferencePercentage:minTimeDifference:pollInterval:)`
  → `start(route:delegate:)` → `onBetterRouteFound(newRoute:etaDifferenceInSeconds:distanceDifferenceInMeters:)`;
  must call `updateCurrentLocation(mapMatchedLocation:sectionIndex:)` or nothing
  fires; **not auto-stopped at destination** — sequence new routes as stop →
  `navigator.route = newRoute` → start, outside `onBetterRouteFound()`; polling may
  be charged.
- Off-road: `OffRoadProgressDelegate` + `isOffRoadDestinationVisible = true` — ⚠
  once off-road events start there is **no way back** to `RouteProgress` without a
  new route. Tunnels: extrapolation on by default; in-tunnel estimated locations
  have `time` and `speed` **nil on purpose**, with no API to detect them.
- Dev/test: `LocationSimulator(route:options:)` replays `Route`/GPX —
  `notificationInterval = 1000 ms / speedFactor` (inversely coupled); ⚠
  `VisualNavigator` **skips animations when consecutive locations are > 100 m
  apart**; always stop one source before starting another. Examples:
  NavigationQuickStart, Navigation, NavigationCustom, NavigationHeadless,
  NavigationWarners, Rerouting, TruckGuidance, SpatialAudioNavigation.

## Verify before delivering

- [ ] Navigate license confirmed (`shared/credentials-and-licensing.md` §3)
- [ ] Location source ~1 Hz at `LocationAccuracy.navigation` with `time`/bearing/speed
- [ ] `startRendering` before route; `stopRendering` before MapView teardown; engines stopped; delegates nil'd
- [ ] `maneuverNotificationOptions` set **before** the route; TTS + audio focus app-side
- [ ] Maneuvers read from the navigator, not the `Route`; XOR completions guarded error-first
- [ ] `setEnabledWarnings(...)` called (else zero warnings); `finalizeGivenWarnings()` on stop
- [ ] Deviation handler debounced (≥3 events) with re-plan/`returnToRoute` fallback (`NO_ROUTE_HANDLE`)
- [ ] Destination reached → explicit next step; overlapping-section milestones handled
- [ ] Beta/charged features called out (EH, realistic view, `calculateTrafficOnRoute`, DynamicRoutingEngine)
- [ ] Any signature beyond this page verified via `here-ios-docs-navigation`
