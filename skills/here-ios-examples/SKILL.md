---
name: here-ios-examples
description: >
  Find official HERE SDK for iOS example apps for a feature before writing
  code from scratch: maps, camera, gestures, map items, POI picking, styles,
  traffic, search, EV routing, positioning, navigation, warners, rerouting,
  truck guidance, offline maps, indoor maps, CarPlay, unit testing. Use when
  the user asks how HERE itself implements a feature, or when you need a
  known-good Swift reference to adapt. Prereq: here-ios-bootstrap; see also
  here-ios-maps, here-ios-search, here-ios-offline.
---

# HERE SDK iOS — Example lookup

Prereqs: `here-ios-bootstrap` (credentials work in all examples for the
matching license). When adapting example callbacks into your app, apply
`shared/threading-and-lifecycle.md` (main-thread calls, XOR error contract).

**Two access paths** `[ios-examples]`:

1. Bundled in the SDK package downloaded from <https://platform.here.com/sdk>.
2. GitHub: **<https://github.com/heremaps/here-sdk-examples>**
   - Explore: `.../examples/latest/explore/ios`
   - Navigate: `.../examples/latest/navigate/ios`

**Path structure:** `examples/latest/{explore|navigate}/ios/<AppName>/<AppName>/<AppName>Example.swift`
— e.g. `OfflineMaps/OfflineMaps/OfflineMapsExample.swift`,
`RoutingHybrid/RoutingHybrid/RoutingHybridExample.swift`. **Language:** Swift
("mostly pure Swift code"; iOS dependencies kept out; no Objective-C).

**Naming (verbatim):** "Most example apps are written in a consistent way
where a 'XYExample.swift' file contains the relevant code. UI elements are
mostly left out or kept minimal to not shadow the important parts of the
code. Note that the HERE SDK itself does offer most APIs headless without UI."
Style: function call expressions are preferred over closures to show full
callback type names; forced unwrapping avoided "except when the HERE SDK
guarantees by design that it is safe to do so."

**Run in Xcode:** (1) copy the `heresdk.xcframework` folder to the app root
(contains `Info.plist`, `ios-arm64`, `ios-arm64_x86_64-simulator`); (2) add
`AccessKeyId` and `AccessKeySecret` to `[APP_NAME]App.swift` — the `App` class
is the iOS entry point. "Unless otherwise noted, the same app can be executed
with Explore or Navigate credentials."

## Which example to open

Paths below are `examples/latest/navigate/ios/…`; **Navigate only** = guide
labels it as such, otherwise it runs with either license.

| Feature task | Example app |
|---|---|
| Show a map view | `HelloMap` |
| Show a map view using Storyboard | `HelloMapWithStoryboard` |
| Create CarPlay app with the HERE SDK | `HelloMapCarPlay` |
| Use the map camera | `Camera` |
| Create custom camera animations | `CameraKeyframeTracks` |
| Show various map schemes & map layers | `MapFeatures` *(Navigate only)* |
| Visualize real-time traffic on a map view | `Traffic` |
| Interact with the map view using gestures | `Gestures` |
| Add map items (circles, polygons, polylines, native views, 2D/3D markers) | `MapItems` |
| Pick embedded POIs | `CartoPOIPicking` |
| Style your own map schemes (HERE Style Editor) | `CustomMapStyles` |
| Add custom raster layers | `CustomRasterLayers` |
| Add custom tile sources (point layers, clustering) | `CustomTileSource` |
| Generate map images offscreen | `OffscreenMap` |
| Use the HERE SDK without a map view | `StandAloneEngine` |
| Find places, geocoding, reverse geocoding online | `Search` |
| …online & offline | `SearchHybrid` *(Navigate only)* |
| Calculate and visualize routes online | `Routing` |
| …online & offline | `RoutingHybrid` *(Navigate only)* |
| Calculate EV routes (EV, isoline, search along route) | `EVRouting` |
| Optimize routes with avoidance options (+ `SegmentDataLoader`) | `RoutingWithAvoidanceOptions` *(Navigate only)* |
| Calculate routes for public transport | `PublicTransit` |
| Integrate HERE Positioning | `Positioning` *(Navigate only)* |
| Track and record outdoor trips (GPX traces) | `HikingDiary` *(Navigate only)* |
| Get started with navigation | `NavigationQuickStart` *(Navigate only)* |
| Turn-by-turn navigation with tracking & voice | `Navigation` *(Navigate only)* |
| Customize turn-by-turn navigation | `NavigationCustom` *(Navigate only)* |
| Navigation events (speed limit, lane assist) | `NavigationWarners` *(Navigate only)* |
| Spatial audio navigation | `SpatialAudioNavigation` *(Navigate only)* |
| Handle rerouting during navigation | `Rerouting` *(Navigate only)* |
| Use navigation for trucks | `TruckGuidance` *(Navigate only)* |
| Headless navigation without a map view | `NavigationHeadless` *(Navigate only)* |
| Use offline maps | `OfflineMaps` *(Navigate only)* |
| Integrate indoor maps (private venues) | `IndoorMap` *(Navigate only)* |
| Write unit tests (mocking SDK classes) | `UnitTesting` |

## Guide pages that walk through examples

- Offline download/repair flow → `OfflineMapsExample.swift` (`ios-offline-maps-installation`)
- Offline routing switch → `RoutingHybridExample.swift` (`ios-offline-maps-routing`)
- Offline/online search switch → `SearchHybrid` (`ios-offline-maps-search`)
- Indoor routing/rendering → `IndoorMap` (`ios-venues-use-cases`)
- First steps: new project → `ios-integrate-here-sdk`; show a map →
  `ios-maps#show-a-map`; CarPlay → `ios-carplay`

## Tutorials (guide + blog pages) `[ios-tutorials]`

| Tutorial | Summary | Where |
|---|---|---|
| How to get started with the HERE platform | HERE SDK product family overview | `here.com/learn/blog/how-to-get-started-with-here-platform` |
| HERE Data SDK & HERE SDK on the HERE platform | SDK combined with the Data SDK | `here.com/learn/blog/here-data-sdk-here-sdk-on-the-here-platform` |
| Get started | Set up environment, first app | `ios-integrate-here-sdk` |
| Displaying maps | Integrate & customize maps | `ios-maps#show-a-map` |
| Integrate CarPlay into your app | `MapView` in the car's display | `ios-carplay` |
| Build a GPX recording app *(Navigate)* | Diary via HERE Positioning | `ios-hiking-app-tutorial` |
| Build a navigation app *(Navigate)* | `VisualNavigator` turn-by-turn | `ios-navigation-app-tutorial` |

**Icons/assets:** maneuver and map-marker icons come from the HERE Icon
Library: <https://github.com/heremaps/here-icons>. **Other apps:** HERE WeGo
(<https://www.here.com/products/wego>) uses the same tech stack; the official
reference app is **Flutter only**
(<https://github.com/heremaps/here-sdk-ref-app-flutter>, Navigate) — not an
iOS/Swift reference.

## Rules

1. **Look here first** — adapt official code instead of re-deriving APIs;
   then verify signatures against the guide via `here-ios-docs-navigation`.
2. Match the example's license to the user's credentials — Explore
   credentials fail on Navigate-only examples with a feature error, not at
   startup (`shared/credentials-and-licensing.md` §3).
3. Don't copy the examples' placeholder credentials or their minimal error
   handling verbatim into production code — harden both.
4. The function-call-expression style is deliberate (full callback type
   names on display) — keep types when converting to closures.

## Verify before delivering

- [ ] Example name and path checked against the repo table above
      (`examples/latest/{explore|navigate}/ios/…`)
- [ ] Example license matches the user's credentials (Navigate-only flagged)
- [ ] `heresdk.xcframework` embedded and credentials set in `[APP_NAME]App.swift`
- [ ] Adapted code hardened beyond the examples' minimal error handling
- [ ] Icons/assets sourced from `heremaps/here-icons`, not re-drawn
- [ ] Cross-links to `here-ios-offline` / `here-ios-search` / `here-ios-maps`
- [ ] Any signature beyond this page verified via `here-ios-docs-navigation`
