---
name: here-ios-traffic
description: >
  Show and consume traffic with the HERE SDK for iOS: traffic flow and
  incident map layers, picking incidents, TrafficEngine area queries, jam
  factor semantics, traffic on routes and during navigation, and controlling
  traffic data costs. Use for any traffic overlay, congestion, incident-feed,
  or traffic-aware ETA task in a HERE iOS app.
---

# HERE SDK iOS — Traffic

Prereqs: `here-ios-bootstrap`. Sources: `ios-traffic`, `ios-traffic-features`, `ios-traffic-engine`. Online
traffic (map layers, `TrafficEngine`, `TrafficOnRoute`) works with **Explore or Navigate** (Base Plan); radio
`TrafficBroadcast` and `VisualNavigator` traffic rendering are **Navigate only** →
`shared/credentials-and-licensing.md` §3. Main-thread calls, main-thread callbacks →
`shared/threading-and-lifecycle.md`; every completion follows the **XOR contract**.

## Traffic map layers

```swift
do { try MapContentSettings.setTrafficRefreshPeriod(5 * 60.0) }   // allowed range 1–5 min; throws
catch let error as MapContentSettings.TrafficRefreshPeriodError { print("TrafficRefreshPeriodError: \(error)") }
catch { print("An unexpected error occurred: \(error)") }

mapView.mapScene.enableFeatures([MapFeatures.trafficFlow: MapFeatureModes.trafficFlowWithFreeFlow])
mapView.mapScene.enableFeatures([MapFeatures.trafficIncidents: MapFeatureModes.defaultMode])
// mapView.mapScene.disableFeatures([MapFeatures.trafficFlow, MapFeatures.trafficIncidents])
```

- **No `TrafficFlowLayer`/`TrafficIncidentLayer` classes** — layers are map *features* toggled on `mapScene`
  (see `here-ios-maps`); "If `MapFeatures.trafficFlow` is disabled, no requests are made." Feature toggles
  reset on `loadScene` — re-enable after every scene load.
- Flow colors: Green = normal, Amber = high, Red = very high, Black = blocking.
- ⚠ **Cost:** "each feature request is counted as an individual transaction… if a user requests both traffic
  flow and traffic incident data, this will be considered as two separate transactions." New requests fire on
  tile load, **every pan/zoom** (immediately) or when backend validity expires; `setTrafficRefreshPeriod`
  overrides it to **1–5 minutes** — "Depending on your contract, each call may be charged separately. It is
  the application's responsibility to decide how often this code should be executed."
- ⚠ "During turn-by-turn navigation, viewport changes can happen multiple times per second and may result in
  a high number of Traffic Vector Tile requests… consider using TrafficOnRoute to update only the traffic
  visualization along the route itself."
- Real-time vs predictive: the flow/incident layers are real-time (Traffic Vector Tile API);
  `RoutingEngine`/`TrafficOnRoute` are predictive (Traffic API v7) — discrepancies are expected.

## Pick an incident

```swift
func pickTrafficIncident(touchPointInPixels: Point2D) {
    let rectangle = Rectangle2D(origin: touchPointInPixels, size: Size2D(width: 1, height: 1))
    mapView.pickMapContent(inside: rectangle, completion: onPickMapContent)
}

func onPickMapContent(mapContentResult: PickMapContentResult?) {
    if mapContentResult == nil { return }   // pick error
    let incidents = mapContentResult!.trafficIncidents
    if incidents.count == 0 { print("No traffic incident found at picked location"); return }
    showDialog(title: "Traffic incident picked:", message: "Type: \(incidents.first!.type.rawValue)")
    findIncidentByID(incidents.first!.originalId)   // → trafficEngine.lookupIncident(with:originalId,
}                                                   //    lookupOptions: TrafficIncidentLookupOptions(), completion:)
// func onTrafficIncidentCompletion(trafficQueryError: TrafficQueryError?, trafficIncident: TrafficIncident?)
```

⚠ The guide's `onTrafficIncidentCompletion` signature as printed declares two
parameters with the **same external label** — it won't compile; use one label.

## TrafficEngine queries

```swift
do { try trafficEngine = TrafficEngine() }   // parameterless; HERE Traffic API v7
catch let engineInstantiationError {
    fatalError("Failed to initialize TrafficEngine. Cause: \(engineInstantiationError)")
}

// Areas: GeoCircle / GeoBox / GeoCorridor. XOR: "If error is nil, it is
// guaranteed that the list will not be nil."  Result: (TrafficQueryError?, [TrafficIncident]?)
trafficEngine.queryForIncidents(inside: GeoCircle(center: centerCoords, radiusInMeters: 1000),
                                queryOptions: TrafficIncidentsQueryOptions(),
                                completion: onTrafficIncidentsFound)

let geoCorridor = GeoCorridor(polyline: route.geometry.vertices, halfWidthInMeters: Int32(500))
trafficEngine.queryForFlow(inside: geoCorridor,
                           queryOptions: TrafficFlowQueryOptions()) { (trafficQueryError, trafficFlowList) in
    for trafficFlow in trafficFlowList ?? [] {
        guard let confidence = trafficFlow.confidence, confidence > 0.5 else {
            continue   // "Exclude speed-limit data and include only real-time and historical flow information."
        }
        addTrafficPolylines(jamFactor: trafficFlow.jamFactor, geoPolyline: trafficFlow.location.polyline)
    }
}
```

- `TrafficIncident`: `description.text`, `location.polyline` (+ gaps via `location.additionalPolylines`),
  `vehicleRestrictions` (dict keyed by `TrafficIncident.RestrictedVehicleCategory` → `VehicleRestriction`:
  `isRestrictedAlways`, `isThroughTrafficRestricted`, `isTrailerRestricted`,
  `restrictedIfGrossWeightMoreThanInKilograms`, `restrictedIfHigherThanInCentimeters`).
- `TrafficFlow`: `jamFactor`, `confidence`, `location.polyline`. Render incidents yourself with `MapPolyline`
  (or your own `MapMarker` icons).
- ⚠ Results are **not sorted by distance** — sort app-side. Language defaults to the country language when
  unsupported.

## Traffic on routes (the right tool during guidance)

| Scenario | Call |
|---|---|
| Refresh ETA on a previewed route | `routingEngine.calculateTrafficOnRoute(route:lastTraveledSectionIndex:taveledDistanceOnLastSectionInMeters:)` → `TrafficOnRoute` — pass `0, 0` for the whole route; **does not change distance or geometry**; ⚠ needs `RouteOptions.enableRouteHandle = true` at calculation |
| Update an existing `Route` instance | `refreshRoute(...)` — see `here-ios-routing` |
| Push into guidance | `visualNavigator.trafficOnRoute = …` (**Navigate**) |
| Actively hunt better routes | `DynamicRoutingEngine` (**Navigate**, online) |
| Route drawn next to live flow / custom jam overlay | adjacent or per-span `MapPolyline`s (below) |

```swift
for section in trafficOnRoute.trafficSections {
    var updatedETAInSeconds = 0.0; section.trafficSpans.forEach { updatedETAInSeconds += Double($0.duration) }
    var updatedTrafficDelayInSeconds = 0.0; section.trafficSpans.forEach { updatedTrafficDelayInSeconds += Double($0.trafficDelay) }
}
```

- Adjacent rendering: `MapMeasureDependentRenderSize(measureKind: MapMeasure.Kind.zoomLevel, sizeUnit: .pixels,
  sizes: widthTable)` (widths × `mapView.pixelScale`) with `outlineWidth = 1.23 * mapView.pixelScale`;
  Navigate default widths from `VisualNavigator.defaultRouteManeuverArrowMeasureDependentWidths()`.
- Custom jam overlay: one `MapPolyline` per span, colored by `span.dynamicSpeedInfo.calculateJamFactor()`
  (0–10): `0–<4` none/light (skip) · `4–<8` moderate (yellow) · `8–<10` severe (red) · `10` blocked (black).
  ⚠ Route jam factor "does not necessarily match exactly the traffic flow visualization on the map view."
- ⚠ Perf guard printed in the docs: `if route.lengthInMeters / 1000 > 5000 { print("Skip showing
  traffic-on-route for longer routes."); return }` (odd arithmetic, but that's what the page says).

## Special sources (Navigate)

- `TrafficBroadcast`/`TrafficDataProvider` (RDS-TMC radio traffic, **beta**, special hardware):
  `trafficBroadcast.activate()`; works offline; `OfflineRoutingEngine` can consume it when the `rdsTraffic`
  layer is enabled via `LayerConfiguration`.
- Per-route incidents with categories: `section.getTrafficIncidents()` → `TrafficIncidentOnRoute` (accident,
  congestion, construction, disabledVehicle, laneRestriction, plannedEvent, roadClosure, roadHazard,
  weather, …) — see `here-ios-routing`.

## Cost control

- During guidance disable the `trafficFlow` layer; use `TrafficOnRoute` (avoids viewport request storms, see
  above). Only enable `trafficIncidents` when the UI actually shows them.
- Longer refresh period (max 5 min) = fewer transactions; each `TrafficEngine` flow/incident request and each
  `TrafficOnRoute` request is a billed transaction (Traffic API v7, Base Plan pricing).

## Verify before delivering

- [ ] Refresh period within 1–5 min; both features re-enabled after every `loadScene`; charged per-tile /
      per-request operations called out
- [ ] XOR error/result guarded on every callback; incidents sorted app-side (results are not distance-sorted)
- [ ] Guidance mode uses `TrafficOnRoute`, not the viewport flow layer (cost + correctness)
- [ ] `RouteOptions.enableRouteHandle = true` set before `calculateTrafficOnRoute`/`refreshRoute`
- [ ] Jam thresholds match the UI legend; `confidence > 0.5` filter applied where probe quality matters
- [ ] Real-time vs predictive discrepancy explained to the user if relevant
- [ ] RDS/traffic-broadcast and navigator traffic rendering flagged as Navigate-only —
      `shared/credentials-and-licensing.md` §3
- [ ] Anything beyond this page verified via `here-ios-docs-navigation`
