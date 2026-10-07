---
name: here-ios-search
description: >
  Search places and addresses with the HERE SDK for iOS: text/one-box
  search, autosuggest, forward and reverse geocoding, category search, Place
  details, search along a route, EV charging data, what3words, and offline
  search with OfflineSearchEngine. Use when the user wants to find
  addresses/POIs, autocomplete input, geocode, or drop search results on the
  map.
---

# HERE SDK iOS — Search & geocoding

Prereqs: `here-ios-bootstrap`. Online search needs connectivity and works
with **Explore or Navigate**; `OfflineSearchEngine` and `W3WSearchEngine`
are **Navigate only**. All callbacks on the main thread.

## Engine

```swift
do { try searchEngine = SearchEngine() }
catch let engineInstantiationError {
    fatalError("Failed to initialize SearchEngine. Cause: \(engineInstantiationError)")
}
```

- Parameterless `try SearchEngine()` — **no credentials argument**; throws
  only if SDK init failed beforehand.
- Covers: discover, suggest, geocode, reverse geocode, HERE Place ID lookup,
  search along a route, category-along-route (beta), plus (Navigate)
  `OfflineSearchEngine`, `W3WSearchEngine`, My places.
- Every request is async and returns a **`TaskHandle`** (status/cancel;
  ignore if unused).

## Queries & options

```swift
let searchOptions = SearchOptions(languageCode: LanguageCode.enUs, maxItems: 30)

// Text (one-box) — area scoping:
let queryArea = TextQuery.Area(inBox: getMapViewGeoBox())     // also .areaCenter:, .inCorridor:, countries
let textQuery = TextQuery("Pizza Chausseestraße", area: queryArea)

_ = searchEngine.searchByText(textQuery: textQuery,
                              options: searchOptions,
                              completion: onSearchCompleted)

// XOR contract: error and items are never both nil / both non-nil.
func onSearchCompleted(error: SearchError?, items: [Place]?) {
    if let searchError = error { print("Error: \(searchError)"); return }
    for result in items! { /* safe to unwrap */ }
}
```

| Task | Call |
|---|---|
| One-box text search | `searchByText(TextQuery, options:completion:)` |
| Autosuggest (type-ahead) | `suggestByText(TextQuery, options:completion:)` → `[Suggestion]?` (`.place` optional — generic terms have only `.title`) |
| Reverse geocode | `searchByCoordinates(GeoCoordinates, options:completion:)` → `items!.first!.address.addressText` (defaults EN_US) |
| Forward geocode | `searchByAddress(AddressQuery(queryString, near: geo), options:completion:)` — `near` biases ranking |
| Category search | `searchByCategory(CategoryQuery([PlaceCategory(id: PlaceCategory.eatAndDrink)], area: CategoryQuery.Area(areaCenter: geo)), options:completion:)` — 700+ categories; multiple categories = at least one must match |
| Place ID lookup | `search(ids:)`-style APIs — verify signature via `here-ios-docs-navigation` |

- **Empty query string is an error** (search fails). Commas optional.
- `SearchOptions.languageCode` sets result language; `maxItems` caps count.
- Area ranking: out-of-area results rank lower (except globally relevant
  matches). "Manhattan" in Berlin still finds Manhattan.

## Results: `Place`

- Fields: `geoCoordinates` (**always set — except nil for suggestions only**),
  `title`, `address.addressText`, `id`, `details`, `accessPoints`.
- Attach to map markers via metadata:

  ```swift
  let metadata = Metadata()
  metadata.setCustomValue(key: "key_search_result", value: SearchResultMetadata(result))
  // class SearchResultMetadata: CustomMetadataValue { var searchResult: Place; func getTag() -> String {…} }
  // retrieve in pick handler:
  marker.metadata?.getCustomValue(key: "key_search_result") as? SearchResultMetadata
  ```

- Zoom to results:
  `camera.lookAt(area: GeoBox.containing(geoCoordinates: coordsList),
  orientation: GeoOrientationUpdate(bearing: nil, tilt: nil))`.
- Rich images: `place.details.images` (`WebImage`) needs TripAdvisor
  rich-content credentials + `searchEngine.setCustomOption(name:
  "discover.show", value: "tripadvisor")`; empty list = missing license.

## Traps

- **Suggestions arrive out of order**: results are ranked, but completion
  events are not ordered (`'piz'` can beat `'pi'`) — debounce/dedupe in UI.
- **No search UI is provided** — build the autocomplete UI app-side.
- Every `search()` call is a charged transaction; scope with `maxItems` and
  areas (`ios-billing`).
- Engine errors surface as `SearchError` (e.g. `.authenticationFailed`,
  `.forbidden`, offline causes). Online engines **never** read offline data.

## EV charging details (extra licenses)

```swift
// Needs credentials with the EV entitlement, else SearchError.forbidden:
if let error = searchEngine.setCustomOption(name: "browse.show", value: "ev") { … }
// read: place.details.evChargingPool?.chargingStations[] → supplierName,
// connectorCount, availableConnectorCount, occupiedConnectorCount, lastUpdated
```

- `EVSearchEngine` (EV Charge Points API v3) needs a **separate EV license on
  top of Navigate**: `evSearchEngine = try EVSearchEngine()` +
  `searchEngine.setEVInterface(evSearchEngine)`;
  `evSearchEngine.setOptions(EVSearchOptions())`, `.search(ids:)`. Ships with
  the `EVRouting` example.
- Category shortcut: `PlaceCategory.businessAndServicesEVChargingStation`.

## Search along a route (correct way)

```swift
let routeCorridor = GeoCorridor(polyline: route.geometry.vertices,
                                halfWidthInMeters: Int32(200))
let queryArea = CategoryQuery.Area(inCorridor: routeCorridor,
                                   near: mapView.camera.state.targetCoordinates)
let categoryQuery = CategoryQuery(PlaceCategory(id: PlaceCategory.eatAndDrink),
                                  area: queryArea)
searchEngine.searchByCategory(categoryQuery, options: searchOptions, completion: onSearchCompleted)
```

Using `TextQuery("restaurants", in: route.boundingBox)` works but returns
results far off route. Larger `halfWidthInMeters` = less precise but cheaper
on long routes. Full code: `EVRouting` example.

## Offline search (Navigate only)

```swift
try offlineSearchEngine = OfflineSearchEngine()   // same interface shape as SearchEngine
// hybrid switching:
if isDeviceConnected {
    _ = searchEngine.searchByText(textQuery, options: searchOptions, completion: onSearchCompleted)
} else {
    _ = offlineSearchEngine.searchByText(textQuery, options: searchOptions, completion: onSearchCompleted)
}
```

- Searches **only** downloaded `Region` data (cache fallback is unreliable —
  "recommended to not rely on cached map data"); reach ≈ 62.5 km radius,
  capitals globally; offline place IDs differ from online ones.
- **what3words**: `try W3WSearchEngine()` →
  `search(words:language:completion:)` / `search(coordinates:language:completion:)`
  (ISO 639-1 code, e.g. `"en"`).
- My places: `GeoPlace.makeMyPlace(title:coordinates:)` →
  `myPlaces.addPlaces(...)` + `offlineSearchEngine.attach(dataSource: myPlaces)`.
- `distributedResults = true` + corridor area = parallelized offline search;
  `OfflineSearchIndex` (beta, +~5 % storage) finds installed-region places
  away from the search center.
- Details: `here-ios-offline`.

## Verify before delivering

- [ ] `try SearchEngine()` after SDK init; XOR contract guarded (error first)
- [ ] Query areas and `maxItems` set — results scoped, costs bounded
- [ ] Empty query strings rejected before calling the SDK
- [ ] Suggestion UI handles out-of-order completions and nil `.place`
- [ ] Result coordinates treated as non-nil except for suggestions; metadata
      attached via `CustomMetadataValue`, not force-casts of `Metadata`
- [ ] EV/what3words/offline features checked against licenses
      (`shared/credentials-and-licensing.md` §3)
- [ ] Charged operations (per-request transactions) called out to the user
- [ ] Anything beyond this page verified via `here-ios-docs-navigation`
