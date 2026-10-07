# HERE SDK for iOS — Credentials, licensing, and legal (shared core)

Referenced by every `here-ios-*` skill. Facts sourced from the official
guide pages cited inline; verify against live docs when licensing changes
(`here-ios-docs-navigation`).

## 1. Getting credentials

1. Sign up for the HERE Base Plan → <https://www.here.com/get-started/pricing>.
2. Sign in to the HERE platform portal → <https://platform.here.com/portal>.
3. **Access Manager → Apps tab → Register new app** → Register (creates a
   unique App ID).
4. **Credentials tab → OAuth 2.0 → Create credentials → Download** → saves
   `credentials.properties` (user ID, `here.client.id` = app ID, token
   endpoint URL, `Access key ID`, `Access key Secret`).
   ⚠ The access key ID/secret are shown **only once** — after clicking Close
   they cannot be retrieved. Credentials are tied to the **Application ID**
   (not to your app's name or bundle ID).

Navigate: before the credentials work, the **quote must be accepted in the
Sales portal using that Application ID** — until then the credentials only
work for Explore. Explore works right after Base Plan signup.

Docs: <https://docs.here.com/here-sdk/docs/ios-get-started.md>

## 2. Use the credentials in the app

```swift
import heresdk

private func initializeHERESDK() {
    let accessKeyID = "YOUR_ACCESS_KEY_ID"
    let accessKeySecret = "YOUR_ACCESS_KEY_SECRET"
    let authenticationMode = AuthenticationMode.withKeySecret(accessKeyId: accessKeyID,
                                                              accessKeySecret: accessKeySecret)
    let options = SDKOptions(authenticationMode: authenticationMode)
    do {
        try SDKNativeEngine.makeSharedInstance(options: options)
    } catch let engineInstantiationError {
        fatalError("Failed to initialize the HERE SDK. Cause: \(engineInstantiationError)")
    }
}
```

- **Initialization succeeds even with invalid credentials.** Verify
  credentials by calling a feature engine (e.g. `SearchEngine().searchByText`)
  and checking for an auth error enum (e.g. `SearchError.authenticationFailed`;
  log text: `Failed to get the authentication token: Authentication failed,
  error code: 1`).
- Init is synchronous, on the main thread, ~100 ms. The SDK is **not**
  auto-initialized — you choose when it starts.
- Credentials are **not tied to the bundle ID** — only to the account.
- Do not hardcode real secrets in committed code; keep them in local config /
  CI secrets (Keychain / fetched from your backend over TLS for release
  builds) and rotate if leaked.

## 3. Explore vs Navigate licenses

| Capability | Explore | Navigate |
|---|---|---|
| Map rendering, schemes, gestures, markers, camera, custom styles | ✅ | ✅ |
| Online search/geocoding/autosuggest/categories, search-along-route | ✅ | ✅ |
| Online routing (car/truck/pedestrian/EV/toll costs, alternatives, maneuvers, isolines*) | ✅ | ✅ |
| Traffic flow/incident overlays, traffic-aware routes | ✅ | ✅ |
| Turn-by-turn navigation (`Navigator`/`VisualNavigator`), voice guidance text | ❌ | ✅ |
| HERE Positioning (`LocationEngine`), background updates, map matching | ❌ | ✅ |
| Offline maps / `OfflineSearchEngine` / `OfflineRoutingEngine` | ❌ | ✅ |
| Warners, lane assistance, Electronic Horizon, truck guidance, milestones | ❌ | ✅ |
| Custom map catalogs (BYOD), indoor/venue maps, textured 3D landmarks, topo schemes | ❌ | ✅ |
| `DynamicRoutingEngine`, `returnToRoute` (online route handle flows), route prefetching, `SegmentDataLoader` | ❌ | ✅ |

\* Isoline routing: **Navigate-only** per the feature list (online).
Waypoint-order optimization (`optimizeWaypointsOrder`) is offline/Navigate only.

- Explore credentials used with a Navigate-only API (e.g.
  `LocationEngine`, `OfflineSearchEngine`, `VisualNavigator`) fail with a
  dedicated error enum — **not** at SDK init.
- The Explore binary is a subset of the Navigate binary: Navigate builds run
  Explore example apps, not vice versa. Changelog/API Reference are tailored
  per license.
- Sources: <https://docs.here.com/here-sdk/docs/ios-introduction-editions.md>,
  <https://docs.here.com/here-sdk/docs/ios-introduction-feature-list.md>

**Rule for agents:** before writing code that uses navigation, positioning,
offline, or warner APIs, confirm the user has a Navigate license; otherwise
offer the Explore alternative (online engines, app-side handling).

## 4. Multiple apps / scopes

One credential set shared by several apps → set the scope **before** init:

```swift
var sdkOptions = SDKOptions(authenticationMode: authenticationMode)
sdkOptions.scope = "hrn:here:authorization::olp-here-sdk-navigate:project/abcd123"
```

- Projects come from the platform **Projects Manager** (Launcher → Manage);
  each project has an immutable HRN.
- Wrong/unknown scope → **all** authentication fails (visible only in logs).
- When a scope is set, the project must link all needed services
  (HERE Routing, Search-Autosuggest/Browse/Forward/One Box/Places ID/Reverse,
  Vector Tile, Vector Tile - Traffic, Isoline Routing, Network Positioning,
  Raster Tile, Routing - Transit…) **plus**, for Navigate, the catalogs
  `Optimized Client Map` (`hrn:here:data::olp-here:ocm`), `ocm-patch` and
  optionally `ocm-japan` — otherwise requests fail with
  `403: Not authorized to access the resources`.

Docs: <https://docs.here.com/here-sdk/docs/ios-multiple-apps.md>

## 5. Integration (no package manager)

There is **no CocoaPods, no Swift Package Manager, no Carthage route** — the
SDK is a manually integrated XCFramework:

1. Download the package from <https://platform.here.com/portal/sdk>.
2. Copy `heresdk.xcframework` into the app root.
3. Xcode → Project → Target → **General → Frameworks, Libraries, and
   Embedded Content → + → Add other → Add files…** → add
   `heresdk.xcframework` (embed).
4. `import heresdk` (lowercase module name).

Package also contains the mandatory **`HERE_NOTICE` file**, guide, API
Reference and example apps. Slides: `ios-arm64` + `ios-arm64_x86_64-simulator`
slices inside the XCFramework.

Docs: <https://docs.here.com/here-sdk/docs/ios-integrate-here-sdk.md>

## 6. Legal requirements (must be in every generated app)

- **`HERE_NOTICE` file** ships in the SDK package and its copyright/OSS
  notices must be embedded in the app or hosted and linked — required.
- **Privacy disclosure (exact text required)** in the app's privacy policy:

  > "Note: This application uses the HERE SDK. Technical, non-personal
  > telemetry data related to map rendering and data usage may be processed by
  > HERE to improve the HERE SDK and map performance."

- **Privacy manifest**: ships inside the framework
  (`heresdk.xcframework/ios-arm64/heresdk.framework/PrivacyInfo.xcprivacy`);
  required for App Store releases since May 2024. Navigate's manifest also
  declares coarse + precise location collection (App Functionality, not
  tracking). For SDKs < 4.18.2.0, add it manually.
- **HERE logo/watermark must stay visible** on the map view; reposition with
  `mapView.setWatermarkLocation(...)` only when the UI requires it; removal
  requires a special agreement with HERE. It auto-hides when both edges are
  < 250 dp.
- SDK usage stats (optional, session-only, **not persisted** — re-enable each
  init): `SDKNativeEngine.sharedInstance?.enableUsageStats(enabled: true)` →
  `getSdkUsageStats()`; the platform Usage Manager dashboard
  (<https://platform.here.com/management/usage>) meters account usage.

Docs: <https://docs.here.com/here-sdk/docs/ios-about.md>,
<https://docs.here.com/here-sdk/docs/ios-billing.md>
