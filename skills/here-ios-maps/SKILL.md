---
name: here-ios-maps
description: >
  Work with the HERE SDK for iOS map view: switch map schemes (day/night,
  satellite, lite, logistics, topo), animate and configure the camera,
  handle gestures and taps, pick map items/POIs, add
  markers/polylines/polygons/arrows/clusters/3D models/pins, toggle map
  features (buildings, traffic, terrain), apply Style Editor styles and
  custom layers, change map language, and show indoor venues. Use for any
  display, annotation, camera, gesture, or style task on a HERE iOS map.
---

# HERE SDK iOS — Maps, camera & map items

Prereqs: `here-ios-bootstrap`. All SDK calls on the main thread; callbacks
arrive on the main thread (`shared/threading-and-lifecycle.md`).

## Map scenes & schemes

```swift
mapView.mapScene.loadScene(mapScheme: MapScheme.normalDay, completion: onLoadScene)
// custom local style: mapView.mapScene.loadScene(fromFile: path, completion: onLoadScene)
```

| `MapScheme` values |
|---|
| `.normalDay` / `.normalNight` |
| `.hybridDay` / `.hybridNight` |
| `.liteDay` / `.liteNight` / `.liteHybridDay` / `.liteHybridNight` |
| `.satellite` (raster, no labels) |
| `.logisticsDay` / `.logisticsNight` / `.logisticsHybridDay` / `.logisticsHybridNight` (truck POI icons) |
| `.roadNetworkDay` / `.roadNetworkNight` (roads only — mini-maps) |
| `.topoDay` / `.topoNight` (**Navigate only**) |

- **The SDK does NOT auto-switch day/night** — the app decides (clock, user
  setting, tunnel events).
- Reloading a scene **resets all `MapFeatures` to defaults** — re-enable them
  after every `loadScene`; toggling features *while* a scene loads may throw.
- Language: `MapView.primaryLanguage = "ja"` (or `nil` = region default) and
  `MapView.secondaryLanguage` — global for all `MapView`s. Disputed
  territories: `SDKOptions.politicalView` (beta) — set **once before init**
  (changing it needs cache clear + re-init).
- Custom style file: export from the web Style Editor
  (<https://platform.here.com/style-editor/>) → **File → Export Map Style**.
  Editor ≥ v1.13.0 exports `.tar.gz`: extract and **repackage as ZIP** in the
  Xcode bundle, then `loadScene(fromFile: path, type implied)` with
  `"zip"`; older exports are a single `.json`. After editing files on disk
  call `mapScene.reloadScene()` (bypasses stale cache).
- Watermark/HERE logo must remain visible (`mapView.setWatermarkLocation(...)`
  to reposition; removal needs a HERE agreement; auto-hidden < 250 dp edges).
- Embedded (Carto) POIs: pickable; per-category visibility via
  `MapContentSettings.setPoiCategoriesVisibility` (**Navigate only**).

Sources: `ios-maps`, `ios-map-schemes`, `ios-map-style-editor`.

## Camera (`mapView.camera` → `MapCamera`)

| Need | Call |
|---|---|
| Jump to place | `camera.lookAt(point: GeoCoordinates(lat, lng), zoom: MapMeasure(kind: .distanceInMeters, value: meters))` |
| Fit area | `camera.lookAt(area: geoBox, orientation: GeoOrientationUpdate(bearing: nil, tilt: nil))` |
| Rotate/tilt | `camera.lookAt(point:…, orientation: GeoOrientationUpdate(bearing: 90, tilt: 45), zoom:…)` |
| Zoom | `camera.zoomTo(level)` — levels **[0, 22]**; `camera.zoomBy(factor, around: point)` (1 = no change) |
| Read state | `camera.state.targetCoordinates`, `camera.boundingBox`, current zoom/distance |
| Shift the center (nav HUD) | `camera.principalPoint = Point2D(x: width/2, y: height * 0.75)` (px, device-scaled) |
| Animate | `camera.startAnimation(MapCameraAnimationFactory.flyTo(target: GeoCoordinatesUpdate(dest), bowFactor: 1, duration: 3))` |

Gotchas: bearing default 0 (north-up, clockwise from north); **tilt is
absolute, not a delta** — out-of-range values clamped; bearing changes shift
the target on a tilted camera. Don't mix zoom level and meters (both `Double`).
During guidance **don't set `principalPoint` on the MapView** — the navigator
owns it; customize via
`visualNavigator.cameraBehavior = FixedCameraBehavior()` +
`normalizedPrincipalPoint = Anchor2D(horizontal: 0.5, vertical: 0.5)`.

## Gestures & taps

```swift
// Conform to TapDelegate
func onTap(origin: Point2D) {
    let geo = mapView.viewToGeoCoordinates(viewCoordinates: origin)  // view px → geo
}
mapView.gestures.tapDelegate = self          // nil to clear
```

- Delegate types: `tapDelegate`, `doubleTapDelegate`, `longPressDelegate`,
  `panDelegate`, `twoFingerPanDelegate`, `twoFingerTapDelegate`,
  `pinchRotateDelegate` — **one delegate per gesture**; setting it does
  **not** disable default map behavior.
- Toggle defaults independently:
  `mapView.gestures.disableDefaultAction(forGesture: .doubleTap)` /
  `enableDefaultAction(forGesture:)` — you still receive the event.
- Continuous gestures carry `GestureState` (`begin/update/end/cancel`):
  `func onLongPress(state: GestureState, origin: Point2D)`; `.cancel` fires
  when the view loses focus (modal, background).
- Defaults: single tap & long press have no action; double tap zoom in;
  one-finger pan moves (momentum); two-finger pan tilts; two-finger tap zooms
  out; pinch zooms + rotates.
- Idle detection: `MapIdleListener` on `mapView.hereMap`.

## Pick POIs / map items

```swift
let rectangle = Rectangle2D(origin: touchPoint, size: Size2D(width: 1, height: 1))
var contentTypes: [MapScene.MapPickFilter.ContentType] = [.mapItems]   // + .mapContent, .customLayerData
let filter = MapScene.MapPickFilter(filter: contentTypes)
mapView.pick(filter: filter, inside: rectangle, completion: onMapItemsPicked)

func onMapItemsPicked(mapPickResults: MapPickResult?) {
    let topmost = mapPickResults?.mapItems.markers.first          // === to your instance
    let places = mapPickResults?.mapContent.pickedPlaces          // embedded Carto POIs
}
```

Pickable: markers, polygons, polylines, clusters (`.clusteredMarkers` →
`[MapMarkerCluster.Grouping]`), embedded POIs, traffic incidents
(`mapView.pickMapContent(inside:completion:)`). **Not pickable:** 3D markers,
`LocationIndicator` (pins use their native view's tap). A ~1–2 px rectangle
is usually enough.

## Map items (add to `mapView.mapScene`)

```swift
// Marker
guard let image = UIImage(named: "poi"), let data = image.pngData() else { return }
let marker = MapMarker(at: geo,
    image: MapImage(pixelData: data, imageFormat: .png),
    anchor: Anchor2D(horizontal: 0.5, vertical: 1))   // default (0.5, 0.5)
marker.text = "Hello"; marker.fadeDuration = 3
mapView.mapScene.addMapMarker(marker)                 // batch: addMapMarker([…])

// Polyline (throws; ≥ 2 vertices via GeoPolyline(vertices:))
let line = try MapPolyline(geometry: geoPolyline,
    representation: MapPolyline.SolidRepresentation(
        lineWidth: MapMeasureDependentRenderSize(sizeUnit: .pixels, size: 20),
        color: UIColor(red: 0, green: 0.56, blue: 0.54, alpha: 0.63),
        capShape: .round))
mapView.mapScene.addMapPolyline(line)

// Polygon / circle (circle = polygon over GeoCircle; ≥ 3 points, CLOCKWISE)
let poly = MapPolygon(geometry: try GeoPolygon(geoCircle: GeoCircle(center: geo, radiusInMeters: 300)),
                      color: UIColor(red: 1, green: 0, blue: 0, alpha: 0.2))
mapView.mapScene.addMapPolygon(poly)
```

| Want | API |
|---|---|
| Arrow along a path | `MapArrow(geometry: geoPolyline, widthInPixels: 30, color:)` — visible above zoom 13 |
| Cluster many markers | `MapMarkerCluster(imageStyle: ImageStyle(image:), counterStyle: CounterStyle())` → `.addMapMarker(marker:)` |
| 3D model | `MapMarker3DModel(geometryFilePath: "x.obj", textureFilePath:)` + `MapMarker3D(at:model:)` (`.obj` only, no `.mtl`; `isDepthCheckEnabled = true` to allow occlusion) |
| Flat marker (rotates with map) | `MapMarker3D(at: image:scale:unit: .densityIndependentPixels)` — add with `addMapMarker3d` (lowercase d) |
| Puck / current location | `LocationIndicator` — `enable(for: mapView)`, `updateLocation(_:)`, `locationIndicatorStyle = .navigation|.pedestrian`, `isAccuracyVisualized = true`, `disable()` |
| SwiftUI label pinned to geo | `mapView.pinView(view: UIHostingController(rootView: …).view!, to: geo)` — lags slightly on pan, less performant than markers |
| Attach data to an item | `Metadata()` + `setString`/`setCustomValue(key:value:)` → retrieve in pick handlers |
| Custom label fonts | `AssetsManager(mapView.mapContext).registerFont(fontName: "SimpleSans", fontPath: "SimpleSans.ttf")` |
| Lifecycle hooks on a custom object | `mapView.addLifecycleDelegate(obj)` → `onAttach/onPause/onResume/onDetach/onDestroy` |

Rules: re-adding an attached item silently no-ops; markers are screen-fixed,
polylines/polygons/circles/arrows scale with zoom; `drawOrder` orders only
within the same item class; `visibilityRanges` accept `.zoomLevel` only (half-
open ranges); polygons need clockwise order and self-intersect badly; markers
always draw on top; keep your own arrays of items — one scene, remove items
individually (`removeMapMarker(s)`, `removeMapPolyline`, `removeMapPolygon`,
`removeMapMarkerCluster`).

## Map features (layers toggled per scene)

```swift
mapView.mapScene.enableFeatures([
    MapFeatures.buildingFootprints: MapFeatureModes.buildingFootprintsAll,
    MapFeatures.trafficFlow: MapFeatureModes.defaultMode])
mapView.mapScene.disableFeatures([MapFeatures.terrain])
```

Key constants: `buildingFootprints` & `extrudedBuildings` (on by default),
`trafficFlow`, `trafficIncidents`, `landmarks`*, `terrain`*, `contours`*,
`publicTransit`*, `safetyCameras`*, `vehicleRestrictions`*,
`roadExitLabels`*, `lowSpeedZones`, `environmentalZones`, `congestionZones`,
`trafficLights`, `shadows` (+ `MapView.shadowQuality`), `ambientOcclusion`.
(∗ = Navigate only — verify current matrix via `here-ios-docs-navigation`.)
**Enabling features costs rendering and — for traffic — per-tile network
transactions**; re-enable after every scene load.

## Custom layers (beta) & indoor maps

- Raster/vector overlays: `RasterDataSource`/`Line|Polygon|PointDataSource`
  → `MapLayerBuilder().forMap(mapView.hereMap).withName(x + "Layer")
  .withDataSource(named: contentType:)…build()` with `Style` from
  `JsonStyleFactory.createFromString(json)`; tile URL template `{z}/{x}/{y}`
  via `TileUrlProviderFactory.fromXyzUrlTemplate`. Runtime tweaks to
  predefined styles: `mapView.hereMap.style.update(...)` (e.g.
  `General.Labels.Scale.Factor`). **Beta — APIs may change without
  deprecation**; keep strong refs to data sources/layers; third-party tile
  attribution required (e.g. OSM/Thunderforest).
- Indoor/venue maps (**Navigate only, beta**): `VenueEngine` (create **after
  the scene loads**) → `venueEngine.venueMap` / `.venueService` →
  `addServiceDelegate`/`addVenueDelegate`/`addVenueSelectionDelegate`,
  `venueService.loadTopologies()`, `venueEngine.start(callback:)`; disable
  `extrudedBuildings` while showing a venue; `selectVenueAsync` /
  `addVenueAsync`. Full guide + indoor routing: `ios-venues`,
  `ios-venues-use-cases`.

## Verify before delivering

- [ ] Scene loaded via completion; `MapError` branch handled; features
      re-enabled after `loadScene`
- [ ] Day/night switching is app-driven where required
- [ ] All camera/gesture/map-item calls on the main thread
- [ ] Pick logic uses only pickable types; cluster picks read
      `.clusteredMarkers`
- [ ] Marker icons from `UIImage`/bundle; polygons clockwise with ≥ 3 points;
      polylines ≥ 2 points; textures ≥ 1 px
- [ ] HERE watermark visible; performance-affecting/transaction-costing
      features justified
- [ ] Beta flags called out (custom layers, indoor maps, political view)
- [ ] Signatures beyond this page verified via `here-ios-docs-navigation`
