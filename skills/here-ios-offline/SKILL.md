---
name: here-ios-offline
description: >
  Ship offline capability with the HERE SDK for iOS: downloading and updating
  map regions with MapDownloader/MapUpdater, prefetching and custom-area
  downloads, OfflineSearchEngine and OfflineRoutingEngine, the global offline
  mode, cache vs persistent storage, and storage repair. Use for any offline
  maps, region download, or offline search/routing task in a HERE iOS app.
  Prereq: here-ios-bootstrap; see also here-ios-search, here-ios-maps,
  here-ios-examples.
---

# HERE SDK iOS — Offline maps & data

Prereqs: `here-ios-bootstrap`. ⚠ **"Offline maps are only available with the Navigate license."** — banner on every offline page; Explore credentials fail with a feature error, not at init (`shared/credentials-and-licensing.md` §3). All calls and callbacks on the main thread (`shared/threading-and-lifecycle.md`) — **except** `MapDownloader`/`MapUpdater`, created async.

**Naming (region-based API):** `MapDownloader`, `MapUpdater`, `MapDownloaderTask`, `Region`/`RegionId`/`InstalledRegion`, `DownloadRegionsStatusListener`, `MapLoaderError`, `PersistentMapStatus` — there is no `OfflineMapDownloader` and no pack manager; a "pack" is a `Region`. Sources: `ios-offline-maps`, `ios-offline-maps-installation`, `ios-offline-maps-update`, `ios-offline-maps-options`, `ios-offline-maps-search`, `ios-offline-maps-routing`, `ios-maps-byod`.

## Storage: cache vs persistent regions

| Store | Contents | Behavior |
|---|---|---|
| **Map cache** (`SDKOptions.cachePath`) | viewport tiles while online | auto-filled, size-limited, LRU eviction; `RoutePrefetcher` fills it |
| **Persistent map storage** (`SDKOptions.persistentMapStoragePath`) | installed `Region`s | never auto-deleted; "a second map storage and fully separated from the map cache"; default ⇒ internal app storage |

- Resolution order: cache → persistent storage → (offline: less/no detail; online: fill cache, LRU when full). Berlin installed ⇒ panning Berlin while online downloads nothing into the cache.
- Read via `sdkNativeEngine.options.cachePath` / `.persistentMapStoragePath`; changing storage ⇒ **new `SDKNativeEngine`** ("the storage path is unique for your credentials key"). App-specific dir only — "Using shared directories such as Documents is not recommended."
- ⚠ Vector only: "Offline maps work for all map schemes that are vector based. Satellite based map schemes are not part of the downloaded map data."

## Global offline switch

```swift
SDKNativeEngine.sharedInstance?.isOfflineMode = true   // default false
// before init instead: SDKOptions.offlineMode = true ("alone enough to operate fully offline")
var passThroughFeaturesSet: Set<PassThroughFeature> = [.trafficTilesFlow, .trafficTilesIncidents]
sdkNativeEngine.passThroughFeatures = passThroughFeaturesSet   // allowed while offline
```

Blocks all SDK online connections "except when `PassThroughFeature` items are set" and stops ongoing/queued requests.
**Engine split:** ⚠ "The online engine will not return results when it is executed offline - even if offline data is available." Online `SearchEngine`/`RoutingEngine` never read local data; only `OfflineSearchEngine`/`OfflineRoutingEngine` do — and those never download.
`Navigator`/`VisualNavigator` have **no** split (cache → persistent → else auto-download unless offline mode).

## Download regions (`MapDownloader`)

```swift
// Create in background to not block the UI thread. One MapDownloader per SDKNativeEngine.
MapDownloader.fromEngineAsync(sdkNativeEngine, { mapDownloader in self.mapDownloader = mapDownloader })

// 1. list — continents → countries hierarchy, localized names, returns a TaskHandle
_ = mapDownloader.getDownloadableRegions(languageCode: LanguageCode.deDe, completion: onDownloadableRegionsCompleted)
// onDownloadableRegionsCompleted(error: MapLoaderError?, regions: [Region]?) — XOR: "If error is nil,
// it is guaranteed that the list will not be nil." Region: .name, .childRegions, .sizeOnDiskInBytes, .regionId.id
// 2. download — returns a MapDownloaderTask immediately
let task = mapDownloader.downloadRegions(regions: [region.regionId], statusListener: self)
mapDownloaderTasks.append(task)
```

`DownloadRegionsStatusListener` has **exactly four events**: `onDownloadRegionsComplete(error: MapLoaderError?, regions: [RegionId]?)`, `onProgress(region: RegionId, percentage: Int32)`, `onPause(error:)`, `onResume()`.

- ⚠ "Downloading multiple regions simultaneously can consume significant amounts of RAM… may cause the operating system to terminate your app" ⇒ "Consider sequential downloads on such devices to lower the RAM pressure."
- SDK auto-retries network failures; too many retries ⇒ error in `onPause()`, only the user resumes (`pause()`/`resume()`; `downloadRegions()` for a paused region ≡ `resume()`). `cancel()` is final — start a fresh request. Failed downloads leave the SDK "fully operable — Just try to download again".
- ⚠ No background downloading: "users should be notified to `pause()` or `cancel()` ongoing downloads… consider providing a `resume()` option." Show `region.sizeOnDiskInBytes` (post-decompression) before downloading.

## List, delete, repair

```swift
let regions = try? downloader.getInstalledRegions()   // InstalledRegion: .regionId, .sizeOnDiskInBytes, .status
let regionIds: [RegionId] = installedRegionList.map { $0.regionId }
mapDownloader?.deleteRegions(regions: regionIds) { error, deletedRegions in /* XOR */ }
// Crash/corruption check — "will not change during the lifetime of an app":
if mapDownloader.getInitialPersistentMapStatus() != PersistentMapStatus.ok {
    mapDownloader.repairPersistentMap(completion: onMapRepairCompleted)   // (PersistentMapRepairError?)
}
```

Deletion during a pending download ⇒ `MapLoaderError.internalError`, during an ongoing download ⇒ `.notReady` — unpause/cancel first. Status `pendingUpdate` cannot be repaired — run an update instead. Worst case: `deleteRegions()` + `SDKCache` `clearCache()` + restart.

## Update map data (`MapUpdater`)

```swift
MapUpdater.fromEngineAsync(sdkNativeEngine, { mapUpdater in self.mapUpdater = mapUpdater })
let versionInfo = try mapUpdaterInstance.getCurrentMapVersion().stringRepresentation(separator: ",")   // e.g. "47.47,47.47"
_ = mapUpdater.retrieveCatalogsUpdateInfo(callback: onCatalogUpdateCompleted)
// (mapLoaderError: MapLoaderError?, catalogList: [CatalogUpdateInfo]?) — empty list = up to date
_ = mapUpdater.updateCatalog(catalogInfo: catalogUpdateInfo, completion: catalogUpdateListenerImpl)
```

`CatalogUpdateProgressListener` = `onPause(error:)`, `onProgress(region:percentage:)`, `onComplete(error:)`, `onResume()`.

- Weekly OCM updates; **incremental (patch) updates are the default** — smaller downloads for **~8 OCM versions**, then a bigger pull; `CatalogUpdateInfo` shows size beforehand. Compatibility: ⚠ "Backward compatibility of installed offline maps is supported for one year."
- ⚠ **After every `updateCatalog()` completion, call `getDownloadableRegions()`** at least once before any new download/update/delete (refreshes internal catalog data).
- ⚠ "It is not possible to enforce a map update by uninstalling all downloaded regions via `deleteRegions()` and then downloading the desired regions again. In that case, the same map version would be downloaded." No installed regions ⇒ `updateCatalog()` "will only clear the cache"; cache version is never older than the offline maps version.

## Prefetch & custom areas (cache-level alternatives)

```swift
let geoCircle = GeoCircle(center: currentGeoCoordinates, radiusInMeters: 10000.0)
polygonPrefetcher.prefetch(geoPolygon: GeoPolygon(geoCircle: geoCircle), callback: …)
routePrefetcher.prefetchAroundRouteOnIntervals(navigator)   // call ONCE after navigation starts
_ = mapDownloader?.downloadArea(area: GeoPolygon(geoBox: getMapViewGeoBox()), statusListener: impl)
```

- `RoutePrefetcher(engine)` / `PolygonPrefetcher` **(beta — APIs may change without deprecation)** feed the **map cache** (evictable), not regions. Corridor defaults **10 km × 5 km**; next prefetch starts 1 km before corridor end (first at 9 km traveled); "increases network traffic continuously during guidance."
- `prefetchAroundLocationWithRadius()` works outside navigation; `prefetchGeoCorridor()` + `PrefetchStatusListener` (`onProgress`/`onComplete`) covers a whole route — parts inside installed `Region`s are reused, not re-downloaded; insufficient cache space ⇒ `MapLoaderError`; `PolygonPrefetcher` can estimate `MapDataSize` first.
- `downloadArea(...)` takes a polygon or `GeoBox`; storage deduplicated ("saving map data only once"); completion yields **one unique `RegionId`** — store a human-readable name for later `deleteRegions(...)`.

## Offline search & routing (Navigate)

```swift
do { try offlineSearchEngine = OfflineSearchEngine() } catch { … }
do { try offlineRoutingEngine = OfflineRoutingEngine() } catch { … }
routingEngine = isDeviceConnected ? onlineRoutingEngine : offlineRoutingEngine   // both adopt RoutingProtocol
```

- Hybrid switching (shared completion handlers) and query patterns: `here-ios-search`. ⚠ "You can only search on already cached or preloaded offline maps data"… "recommended to not rely on cached map data." Reach ≈ 62.5 km (capitals globally); offline place IDs differ from online IDs.
- **Indexing (beta):** `OfflineSearchIndex.Options().enabled = true` + static `OfflineSearchEngine.setIndexOptions(sdkEngine:options:listener:)`; `OfflineSearchIndexListener` = `onStarted(operation:)`, `onProgress(percentage:)`, `onComplete(error:)`; **+~5% storage**; index rebuilds after `downloadRegions`/`deleteRegions`/`clearPersistentMapStorage`/`repairPersistentMap`/`performMapUpdate`/`updateCatalog`.
- **`OFFLINE_SEARCH_GLOBAL` (beta)** via `SDKOptions.layerConfiguration` (`LayerConfiguration(enabledFeatures:)` with `.offlineSearchGlobal`, omitting `.offlineSearch`): **+~11–16%** downloaded size, **+~40–140%** cache size via `implicitlyPrefetchedFeatures`. After switching, run `performMapUpdate()`/`updateCatalog()` or hit `SearchError.layersNotDownloaded`/`.mapNotReady`.
- BYO places: `GeoPlace.makeMyPlace(title:coordinates:)` → `myPlaces.addPlaces(places:)` → `offlineSearchEngine.attach(dataSource:)`; corridor `distributedResults = true` is `OfflineSearchEngine`-only.
- ⚠ **Empty maneuvers offline:** "the resulting routes contain only an empty list of maneuvers… get them directly from the `Navigator` or `VisualNavigator` via the provided `nextManeuverIndex`."
- ⚠ **Not supported offline:** "Transit routes (via `TransitRoutingEngine`) and EV routes (via `ElectricVehicleOptions`…) are not yet supported and only work online. The `OfflineRouteEngine` will fail with `RoutingError.invalidParameter` in case of unsupported transport modes."
- Offline routing "is intended to be used with offline maps and not solely with cached map data" — missing data anywhere ⇒ route calculation fails. Try one engine, fall back to the other. Example: `RoutingHybridExample.swift`.

## Limitations & extras

- Pre-install packages: "Upon request, it is possible to manually create such packages… there is no API available for this and you need to contact us." BYOD catalogs are Navigate-only + extra HERE agreement; pass `catalogConfigurations`/`customEngineOptions` on **every** init (`ios-maps-byod`).
- Some `MapFeatures` layers are not offline by default — `terrain` "needs to be enabled with a 'FeatureConfiguration' to be part of a downloadable `Region`"; cached layers work offline if panned beforehand.

## Verify before delivering

- [ ] Navigate license confirmed (`shared/credentials-and-licensing.md` §3); Explore alternative offered if not
- [ ] `fromEngineAsync` used for `MapDownloader`/`MapUpdater`; everything else on the main thread (`shared/threading-and-lifecycle.md`)
- [ ] XOR error/result contract guarded in every `MapLoader*` completion
- [ ] Downloads sequential (RAM trap), progress UI + pause/resume/cancel affordances, no background downloading, size shown upfront
- [ ] `getDownloadableRegions()` re-run after every `updateCatalog()`
- [ ] Engine choice matches connectivity; offline maneuver/EV/transit limits surfaced in UI expectations
- [ ] Beta flags called out (RoutePrefetcher, OfflineSearchIndex, OFFLINE_SEARCH_GLOBAL)
- [ ] Any signature beyond this page verified via `here-ios-docs-navigation`
