---
name: here-ios-routing
description: >
  Calculate and display routes with the HERE SDK for iOS: car, truck,
  pedestrian, scooter, bicycle, EV and transit routing, route options and
  avoid rules, waypoints, alternatives, tolls, MapPolyline drawing, maneuver
  lists, ETA reading, isolines, route refresh/return-to-route, and offline
  routing. Use for any directions, routing, or route-display task in a HERE
  iOS app.
---

# HERE SDK iOS — Routing

Prereqs: `here-ios-bootstrap`. Online `RoutingEngine` = **Explore or Navigate**; `OfflineRoutingEngine`,
`DynamicRoutingEngine`, `returnToRoute`, `optimizeWaypointsOrder` and (per feature list) isolines =
**Navigate only** → `shared/credentials-and-licensing.md` §3. Main-thread calls + main-thread callbacks →
`shared/threading-and-lifecycle.md`; closures follow the **XOR contract**. Sources: `ios-routing`,
`ios-routing-options`, `ios-routing-ev`, `ios-routing-advanced`, `ios-routing-ui`.

## Core A→B

```swift
do { try routingEngine = RoutingEngine() }   // parameterless, no credentials arg; HERE Routing API v8
catch let e { fatalError("Failed to initialize routing engine. Cause: \(e)") }
routingEngine.calculateRoute(with: [Waypoint(coordinates: startGeoCoordinates!),
                                    Waypoint(coordinates: destinationGeoCoordinates!)],
                             options: RoutingOptions()) { (routingError, routes) in
    if let error = routingError { print("Error: \(error)"); return }
    // XOR: error nil ⇒ routes non-nil; exactly 1 route unless alternatives were requested.
    let route = routes!.first!
    showRouteDetails(route: route); showRouteOnMap(route: route); logRouteViolations(route: route)
}
```

- Parallel `calculateRoute` calls are allowed; public transit → `TransitRoutingEngine` + `TransitRouteOptions`.
  Hints: `waypoint.sideOfStreetHint`; `startingWaypoint?.headingInDegrees = location.bearingInDegrees`
  (avoids start U-turns/street crossings).
- Waypoints: `stopover` (default; **first/last must be stopover**; guaranteed passed, splits sections) or
  `passThrough` (hint only); list order = pass order. From a `Place` use `place.accessPoints.first!` +
  `sideOfStreetHint`. ⚠ Online engine: **~200 waypoints at most**; `OfflineRoutingEngine` unlimited.

## Transport modes & options

```swift
let options = RoutingOptions()                                    // unified class for all modes
options.transportSpecification.transportMode = TransportMode.car  // .car (default), .truck, .pedestrian, .scooter, .bicycle, .taxi, .bus, .privateBus
options.routeOptions.optimizationMode = OptimizationMode.fastest  // default
options.routeOptions.alternatives = 2                             // int, default 0 = none
options.routeOptions.enableTolls = true                           // default false → read section.tolls
```

- **Traffic-aware routing is ON by default** (`TrafficOptimizationMode.timeDependent` "is set by default";
  `arrivalTime = nil` → device time). ⚠ "disabling `TrafficOptimizationMode` will also disable any traffic
  information from the `Route` and the ETA will not include potential delays."
- EV (`ios-routing-ev`): `routingOptions.evOptions = ElectricVehicleOptions()`. ⚠ `batterySpecifications`
  needs **all** of `totalCapacityInKilowattHours`, `initialChargeInKilowattHours`, `chargingCurve` else
  **`INVALID_PARAM`**, plus `empiricalConsumptionModel` (min: `ascentConsumptionInWattHoursPerMeter`,
  `descentRecoveryInWattHoursPerMeter`, `freeFlowSpeedTable`). `ensureReachability = true` ⇒ **also** empty
  `AvoidanceOptions`, `speedCapInMetersPerSecond = nil`, `.fastest`, `alternatives = 0`; plan stops via
  `waypoint.chargingStop = ChargingStop(powerInKilowatts:…, supplyType: .dc, …)`; read `section.postActions`
  (`.chargingSetup`/`.charging`/`.wait`) + `departurePlace.chargingStation` (arrival charge `< 0.0` =
  unreachable). ⚠ Billing: "Routing EV" **only** with `ensureReachability` true, else "billed as normal car
  route."
- Truck: `TransportMode.truck` + `VehicleSpecification` via `TransportSpecification.TruckBuilder`
  (dimensions/weight; guide shows no field names — verify via `here-ios-docs-navigation`). ⚠ **No multimodal
  route in one request** — compose separate requests.
- Actual mode: `route.requestedTransportMode` vs `route.getSectionTransportMode()` → `SectionTransportMode`
  (adds `FERRY`, `CAR_SHUTTLE_TRAIN`); a mode change splits a new `Section`.
- Scooter: only `OptimizationMode.fastest` supported, avoids roads > 60 km/h. ⚠ `ensureReachability` is
  **not supported** by the `OfflineRoutingEngine`.

## Avoid rules & violations

```swift
let avoidanceOptions = AvoidanceOptions()
// RoadFeatures (e.g. controlledAccessHighway), ZoneCategory (LEZ), GeoBox,
// CountryCode, ferries, tolls, harsh turns — and specific segments via
// SegmentReference (segmentId + tilePartitionId from Route spans) in .segments
routingOptions.avoidanceOptions = avoidanceOptions
```

⚠ **"It is recommended to always check a calculated `Route` for possible violations"** (empty list = none):
walk `route.sections[].spans[].noticeIndexes` → `section.sectionNotices[Int(index)].code` (e.g.
`"violatedVehicleRestriction"`). `SegmentReference` also via `SegmentDataLoader` (Navigate only); example app:
**RoutingWithAvoidanceOptions**; imported routes surface violations the same way (`Section.sectionNotices`).

## Reading the route

| Need | API |
|---|---|
| Length / duration / traffic delay | `route.lengthInMeters`, `route.duration` (includes traffic delay), `route.trafficDelay` (predictive) |
| ETA | last section's `section.arrivalLocationTime` (`LocationTime`) — ⚠ **section durations/times are cumulative** |
| Maneuvers | `route.sections[].maneuvers[]` → `.action: ManeuverAction`, `.text`, `.coordinates` |
| Per-span detail | `dynamicSpeedInfo`, `streetAttributes`, `carAttributes`/`truckAttributes`/`scooterAttributes`/`walkAttributes`, `streetNames`, `routeNumbers`, `speedLimitInMetersPerSecond`, `consumptionInKilowattHours`, `functionalRoadClass`, `geometry` |
| Traffic incidents | `section.getTrafficIncidents()` → `TrafficIncidentOnRoute` (accident, congestion, construction, roadClosure, weather, …) |
| Tolls / misc | `section.tolls` → `toll.tollSystems`, `toll.countryCode`, `toll.fares[].price/.currency/.paymentMethods`; `route.railwayCrossings[]`; `Route.serialize(...)`/`Route.deserialize(...)` |

⚠ Maneuver text trap: `maneuver.text` "is empty during navigation when it is taken from `Navigator` or
`VisualNavigator`. It only contains localized instructions when taken from a `Route` instance" — conversely
`roadTexts`, `nextRoadTexts`, `exitSignTexts` are non-empty **only** from the navigator.

## Draw the route

```swift
let line = try MapPolyline(geometry: route.geometry,   // MapPolyline(...) throws — wrap in do/catch
    representation: MapPolyline.SolidRepresentation(
        lineWidth: MapMeasureDependentRenderSize(sizeUnit: RenderSize.Unit.pixels, size: 20.0),
        color: UIColor(red: 0, green: 0.56, blue: 0.54, alpha: 0.63),
        capShape: LineCap.round))
mapView.mapScene.addMapPolyline(line)   // add ONCE (guide snippet adds it twice — doc bug)
```

Fit: `camera.lookAt(area: route.boundingBox, orientation: GeoOrientationUpdate(bearing: nil, tilt: nil))`;
animated with padding: `MapCameraUpdateFactory.lookAt(area:orientation:viewRectangle:)` +
`MapCameraAnimationFactory.createAnimation(from:duration:easing:)` → `mapView.camera.startAnimation(...)`
(see `here-ios-maps`).

## Maneuver list UI

Build the list from `route.sections[].maneuvers[]` (see Reading):

```swift
for section in route.sections { for maneuver in section.maneuvers {
    print("\(maneuver.text), Action: \(maneuver.action), Location: \(maneuver.coordinates)") } }
```

- ⚠ "the HERE SDK itself does not ship with maneuver icons" — use the open-source **HERE Icon Library**
  (`github.com/heremaps/here-icons`, `guidance-icons/manoeuvers`). Road shields:
  `iconProvider.createRoadShieldIcon(...)` works **offline** from `Route` data. Full `ManeuverAction` enum in
  `ios-routing-ui`.
- No maneuver panel SDK class: `ManeuverView`/`ManeuverModel` are example-app code (Rerouting example,
  Navigate); "More reusable UI building blocks are planned for the future."

## Advanced

| Need | API (license) |
|---|---|
| Reachability area | `try IsolineRoutingEngine()` → `calculateIsoline(center: Waypoint, isolineOptions: IsolineOptions(calculationOptions: IsolineOptions.Calculation(rangeType: .consumptionInWattHours, rangeValues: [400]), routingOptions:))` { error, isolines } → `isoline.polygons` → `MapPolygon`; ⚠ one center point per isoline, isoline count == range-value count (Navigate) |
| Fresh traffic + ETA on previewed route | `routingEngine.calculateTrafficOnRoute(route:lastTraveledSectionIndex:taveledDistanceOnLastSectionInMeters:)` → `TrafficOnRoute` (`trafficSections[].trafficSpans[].duration/.trafficDelay`); pass `0, 0` for the whole route; **does not change distance or geometry** |
| Update an existing `Route` | `refreshRoute(refreshRouteParameters: RefreshRouteParameters(routeHandle:startingPoint:), routingOptions:)` — new start "must be on or very close to the original route" |
| Rejoin after a detour (Navigate) | `returnToRoute()` — for when the starting point is far from the route; stopovers guaranteed passed, pass-through not |
| Hunt better routes while driving (Navigate) | `DynamicRoutingEngine` — online; `dynamicRoutingEngineOptions.pollInterval` |
| Import recorded track / backend handle | `importRoute(with: [Location], options:)` — dense fixes (~1 Hz), online, `RouteStop` preserves stops; or `importRoute(routeHandle: RouteHandle(handle:), options:)` — REST string "only valid for a couple of hours" |
| Offline (Navigate) | `OfflineRoutingEngine` — same call shapes, no waypoint limit; ⚠ "Refreshing routes is not supported" |

⚠ `RouteOptions.enableRouteHandle = true` **before** the original calculation is required for
`route.routeHandle` — prerequisite of `refreshRoute` **and** `calculateTrafficOnRoute`; handles expire after
a few hours (persist via `Route.serialize(...)` instead).

## Verify before delivering

- [ ] `try RoutingEngine()` after SDK init; XOR closure guarded (error first); `sectionNotices` checked for
      violations; traffic optimization untouched unless asked
- [ ] Transport mode + vehicle/EV options match the use case (EV: `ensureReachability` strict requirements);
      `enableTolls` / `enableRouteHandle` set before calculation where needed
- [ ] Route drawn once via `MapPolyline.SolidRepresentation` (throws); camera fit to `route.boundingBox`;
      maneuver icons from `here-icons`
- [ ] Navigate-only features (offline, dynamic routing, returnToRoute, isolines, waypoint optimization)
      license-checked — `shared/credentials-and-licensing.md` §3
- [ ] Anything beyond this page verified via `here-ios-docs-navigation`
