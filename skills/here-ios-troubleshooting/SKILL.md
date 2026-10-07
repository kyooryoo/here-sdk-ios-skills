---
name: here-ios-troubleshooting
description: >
  Debug HERE SDK for iOS problems: blank/white maps, authentication and
  credential failures, module/integration errors, simulator and archive
  build issues, offline and positioning failures, logging with LogControl,
  collecting device logs, crash symbolication with dSYMs, and filing useful
  bug reports. Use whenever generated HERE map code misbehaves at runtime or
  the user reports a HERE SDK error in an iOS app.
---

# HERE SDK iOS — Debugging & troubleshooting

Sources: `ios-tips-troubleshooting`, `ios-tips-engines`, `ios-tips-support`,
`ios-use-cases`. Diagnose top-down; each row is an official fix.

## Logging first

```swift
// MUST be called before initializing the HERE SDK:
LogControl.enableLoggingToConsole(LogLevel.logLevelInfo)   // INFO < WARNING < ERROR < FATAL; logLevelOff silences
// LogControl.disableLoggingToConsole()                      // to silence entirely
```

- Works "even for release builds"; all SDK log lines are prefixed
  **`hsdk-`**.
- Built-in minimum level cannot change after init; a custom `LogAppender`
  can filter/steer at runtime but does **not** disable console output — call
  `disableLoggingToConsole()` too.
- Capture on device: Xcode → **Window → Devices and Simulators** (newer:
  Open Recent Logs / Open Console) → **View Device Logs**. Simulator without
  Xcode:

  ```
  xcrun simctl spawn booted log stream --level debug
  ```

## Symptom → fix table

| Symptom | Fix |
|---|---|
| Blank white map | Check credentials, internet, slow-connection tile delay, and **device clock** — "wrong device time can cause auth issues" |
| Log: `No map content will be displayed until valid config is provided.` | You never called `mapView.mapScene.loadScene(...)` |
| Xcode: `MapView` not known | Framework not integrated per `ios-integrate-here-sdk` |
| Xcode won't load `heresdk` module (SourceKit) | Known `rdar://42087654`; integrate per guide and ensure **no space symbols in the project/framework path** |
| Simulator build fails on Apple Silicon | SDK < 4.9.2.0 — upgrade, or exclude ARM64 for "Any iOS Simulator SDK" (Build Settings → Architectures → Excluded Architectures → Debug, TARGETS) i.e. Rosetta |
| `These credentials do not authorize access` | Invalid/incompatible credentials — 3.x keys don't work on 4.x; **Explore keys don't work with Navigate APIs** |
| Init "succeeds" but features fail | Normal: "The HERE SDK does not verify credentials at instantiation time" — first real feature call fails instead (e.g. `SearchError.authenticationFailed`, log `Failed to get the authentication token: Authentication failed, error code: 1`) |
| Release build fails; log shows `-embed-bitcode` | Bitcode deprecated — `ENABLE_BITCODE = NO;` in debug/release build settings |
| Archive option grayed out | Active scheme targets the simulator → pick a physical device or **Generic iOS Device**; or check the workspace Build tab "Archive" checkbox |
| `InstantiationException` creating 2nd `SDKNativeEngine` | Two instances share one access key ID (cache path unique per key) — need a distinct credential set per instance |
| `MapView` dead after re-init | State invalid once `SDKNativeEngine` is disposed — call `mapView.onCreate(..)` again; verify with `mapView.isValid()` |
| Engines return stale/no results after re-init | Parameterless-constructor engines must be **recreated** after every re-init; `SDKOptions` are not persisted — re-supply them |
| Location updates never arrive | `ConsentEngine` consent not given / permission missing / engine not started (`here-ios-positioning`) |
| No warnings delivered | `WarnerEngine.setEnabledWarnings(...)` not called (`here-ios-navigation`) |
| Offline search/route fails (`SearchError.layersNotDownloaded`, `.mapNotReady`) | Layer configuration changed without re-running `updateCatalog()`/`performMapUpdate()`, or region downloaded under a different `LayerConfiguration` (`here-ios-offline`) |
| Offline suddenly "empty" after rotating credentials | Cache path is keyed to access key ID — old cache reappears when old credentials return |
| Large storage spikes updating offline maps | Query installed `Region`s with `MapDownloader`, compare sizes, avoid automatic updates |
| Blank map after `loadScene` on satellite/vector scheme | Satellite & other raster schemes are **always online**; offline mode needs vector schemes (`here-ios-offline`) |

Always check first: minimum requirements (`ios-introduction-requirements`),
supported devices, and **known issues in the changelog**
(`here-sdk-for-ios-navigate-changelog` / `here-sdk-for-ios-explore-changelog`).

## Behavioral traps (not errors, but look like bugs)

- Day/night never switches automatically — app's job.
- `MapFeatures` reset on every `loadScene`.
- Online engines (`SearchEngine`, `RoutingEngine`) ignore offline data by
  design; only `OfflineSearchEngine`/`OfflineRoutingEngine` read it.
- XOR contract: exactly one of `(error, result)` is non-nil in every
  completion — guard it (`shared/threading-and-lifecycle.md`).
- All callbacks arrive **on the main thread** — never block them.
- Suggestions completions arrive out of order.
- Credential change at runtime invalidates the cache path (see table).
- Deviation events during navigation fire continuously → debounce
  (`here-ios-navigation`).

## Crash symbolication

- Since **HERE SDK 4.13.3**, dSYMs ship inside `heresdk.xcframework` (per
  device architecture); native C++ crashes otherwise show only addresses.
- For App Store/TestFlight builds: in Apple's Connect Portal click **Open in
  Xcode** (instead of downloading dSYMs) to see symbolicated stacks — also
  works with Firebase Crashlytics. dSYMs are not on user devices; open Xcode
  on the machine with your project.

## Filing a HERE bug report (<https://www.here.com/contact>)

Include (verbatim checklist from `ios-tips-support`): issue type (crash log
+ stack trace, or expected vs actual) · regression? since when? · detailed
repro steps (reproducible in example apps? isolated code?) · which APIs +
all parameters/options · reproducibility rate · environment (OS, IDE,
device/simulator OS + name) · map data version / catalog · other frameworks
+ versions · screenshots/video · for search/routing: coordinates or a GPX
trace. **Always: HERE SDK version, platform (Explore/Navigate), license and
logs.**

- Docs/example-app issues → <https://github.com/heremaps/here-sdk-examples/issues/new>
  ("the most effective way to ensure they are addressed promptly").
- Community: HERE developer Slack (`heredev.slack.com`); releases are monthly
  (6 minor / 4 major / 2 LTS per year; LTS supported 6 months; beta APIs
  have **no** deprecation process).
- Keyword-search the changelog back to 4.8.2.0 for features/known issues.

## Verify before delivering a fix

- [ ] `LogControl` enabled **before init**; `hsdk-` lines captured
      (device logs or `simctl … log stream`)
- [ ] Blank-map triage walked in order (creds/device time → `loadScene` →
      integration/path → simulator arch → credentials class)
- [ ] Credential generation confirmed (3.x/4.x, Explore/Navigate)
- [ ] XOR contracts guarded; callback bodies exception-safe; nothing blocks
      the main thread
- [ ] Re-init cascades handled (engines recreated, `mapView.onCreate`,
      options re-supplied)
- [ ] Offline failures traced to layer config / catalog update, not "offline
      is broken"
- [ ] If it might be an SDK bug: symbolicated log + full bug-checklist items
      → HERE contact
- [ ] Anything beyond this page verified via `here-ios-docs-navigation`
