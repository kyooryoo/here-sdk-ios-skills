# HERE SDK for iOS — AI Agent Skills

A **skill set that teaches AI coding agents to build iOS apps with the HERE
SDK for iOS (4.x)** — from Xcode project bootstrap to turn-by-turn
navigation, offline maps, cost/performance hardening, and official-doc
verification.

The skills encode the official HERE SDK iOS Developer Guide (v4.28.0.0),
collected by walking the guide's machine-readable indexes (`llms.txt`), plus
the non-obvious contracts that generated code usually gets wrong:
main-thread-only calls, XOR error/result closures, license gating (Explore vs
Navigate), the manual `heresdk.xcframework` integration, engine lifecycle and
re-init cascades, and the warning-listener gate.

---

## 1. Description

| | |
|---|---|
| **Purpose** | Enable an AI agent to help developers create HERE SDK iOS apps correctly, following official HERE guides and best practices, without re-deriving the SDK's quirks on every task. |
| **Target SDK** | HERE SDK for iOS **4.28.0.0** (Explore & Navigate licenses) |
| **Format** | Agent skills: one `SKILL.md` per topic (router frontmatter + instructions + verify checklist) with a shared correctness core in `shared/` |
| **Coverage** | Xcode integration & credentials · maps/camera/gestures/items/styles · search & geocoding · routing (car/truck/EV) · traffic · positioning · turn-by-turn navigation (voice, warners, lanes, deviation, e-horizon) · offline maps · performance & data-cost hardening · troubleshooting · official examples lookup · live-docs verification |
| **Design principle** | *Skills teach patterns; the official examples supply known-good code; `here-ios-docs-navigation` verifies signatures.* Every skill ends with a **verify checklist** derived from documented failure modes. |

## 2. How it was created

### Phase 1 — Research

1. Started from `https://docs.here.com/here-sdk/docs/ios-readme` and
   followed the docs' agent indexes (`llms.txt` at project level, then the
   iOS guide index) to map ~70 guide pages.
2. Fanned out **6 parallel research agents**, each fetching every `.md` page
   in a documentation cluster:
   - setup/licensing/integration (requirements, editions, credentials,
     `heresdk.xcframework`, SwiftUI `UIViewRepresentable`, CarPlay)
   - maps & customization (schemes, camera, gestures, items, picking,
     features, styles, custom layers, indoor)
   - search / routing / traffic (all pages incl. EV, isolines, TrafficEngine)
   - positioning & navigation (warners, lanes, e-horizon, truck, optimization)
   - offline & examples (region API, updates, offline search/routing,
     prefetch, example apps, tutorials)
   - best practices / troubleshooting / billing / legal (key-concepts,
     engines, supplements, optimization, changelog policy, FAQ, support)
3. Notes were cross-checked for **naming traps** (e.g. the offline API is
   region-based `MapDownloader`/`MapUpdater`, *not* a pack manager; there is
   no `LanesList` type — iOS uses `ManeuverViewLaneAssistance`/
   `JunctionViewLaneAssistance`; no `TextToSpeech` class — iOS docs show
   `AVSpeechSynthesizer` via a `VoiceAssistant` helper; no logger API beyond
   `LogControl`/`LogAppender` with `hsdk-` prefixes).

### Phase 2 — Build

Skills were written to the conventions of the sibling skill sets in this
workspace (`here-maps-sdk-android`, `here-maps-sdk-flutter`,
`here-maps-sdk-js`): router-ready YAML frontmatter, task-shaped boundaries, a
shared correctness core, and a verify-before-delivering checklist on every
skill. Facts that could not be confirmed on a fetched page are marked "verify
via `here-ios-docs-navigation`" instead of being guessed. Known docs bugs
found during research (e.g. duplicate `addMapPolyline` in a snippet, a
non-compiling callback label) are flagged in-file rather than silently
"fixed".

### Phase 3 — Validation

- Frontmatter (`name`, `description`) validated on all 12 skills.
- Every API name used in a skill appears in the fetched research notes from
  the cited official page; no invented APIs.
- Cross-references between skills and `shared/*` resolved.

## 3. What's inside

```
here-maps-sdk-ios/
├── README.md                          ← this file
├── shared/                            ← correctness core, referenced by all skills
│   ├── credentials-and-licensing.md   OAuth keys, Explore vs Navigate matrix, XCFramework, legal
│   └── threading-and-lifecycle.md     main-thread law, init order, engine lifecycle, SwiftUI teardown
└── skills/
    ├── here-ios-bootstrap/            [Tier 0] Xcode/xcframework, init, MapView, SwiftUI, scene, CarPlay
    ├── here-ios-maps/                 [Tier 1] schemes, camera, gestures, items, picking, features, indoor
    ├── here-ios-search/               [Tier 1] text/suggest/geocode/category, Place results, EV, W3W
    ├── here-ios-routing/              [Tier 1] routes, modes, options, avoid, truck/EV, isolines, advanced
    ├── here-ios-traffic/              [Tier 1] flow/incident layers, TrafficEngine, traffic-on-route, costs
    ├── here-ios-positioning/          [Tier 1] LocationEngine, permissions, accuracy, puck, map matching
    ├── here-ios-navigation/           [Tier 1] VisualNavigator/Navigator, voice, deviation, warners, lanes
    ├── here-ios-offline/              [Tier 1] regions, updates, offline search/routing, prefetch
    ├── here-ios-performance/          [Tier 2] engines, memory, frame rate, LayerConfiguration, pkg size
    ├── here-ios-troubleshooting/      [Tier 2] symptom→fix table, LogControl, device logs, symbolication
    ├── here-ios-examples/             [Tier 3] feature → official example-app lookup table
    └── here-ios-docs-navigation/      [Tier 3] llms.txt/.md page fetching, license/version checks
```

**Totals:** 12 skills · 2 shared docs · ~2,500 lines.

### Skill inventory

| Skill | Use it when the user wants to… |
|---|---|
| `here-ios-bootstrap` | create/init a HERE iOS project, integrate the framework, add a map, fix a blank map at startup (always first) |
| `here-ios-maps` | switch theme/scheme, animate camera, handle gestures/taps, draw markers/shapes, style the map |
| `here-ios-search` | search addresses/POIs, autocomplete, geocode, search along a route |
| `here-ios-routing` | calculate/display routes, truck/EV routing, waypoints, avoid options, ETA |
| `here-ios-traffic` | show traffic, query incidents/flow, refresh route ETA |
| `here-ios-positioning` | get GPS locations, show the blue dot/puck, position accuracy, simulated tracks |
| `here-ios-navigation` | turn-by-turn guidance, voice, rerouting, warners, lane guidance |
| `here-ios-offline` | download maps, work offline, manage storage, prefetch routes |
| `here-ios-performance` | audit performance/data costs/binary size, harden for release |
| `here-ios-troubleshooting` | debug an error, collect logs, file a HERE bug report |
| `here-ios-examples` | find the official example app for a feature |
| `here-ios-docs-navigation` | verify an API, check license gating, check changelog/version |

## 4. How to use

### Prerequisites (for generated apps)

- HERE account + OAuth credentials → `shared/credentials-and-licensing.md`
- License choice: **Explore** (online maps/search/routing) vs **Navigate**
  (adds navigation, positioning, offline, traffic engine extras,
  warners…). Every skill that uses Navigate-only APIs says so.
- iOS 15.2+/Xcode 16.2; manual `heresdk.xcframework` copy — **no CocoaPods
  and no SPM** (`shared/credentials-and-licensing.md`).

### Loading the skills in an agent

- **Claude Code / similar**: copy `skills/` into the project's skills
  directory (e.g. `.claude/skills/`), plus `shared/`.
- **OpenCode**: add the paths as skill sources in config; skills load on
  description match.
- **Custom agents**: parse frontmatter — `description` is the router key;
  load the full `SKILL.md` on match; resolve `shared/*` references on demand.

Recommended loading order:

1. `here-ios-bootstrap` — always (defines the shared init).
2. Domain skill(s) matching the task (maps · search · routing · traffic ·
   positioning · navigation · offline).
3. `here-ios-examples` before writing feature code — adapt official example
   logic when one exists.
4. `here-ios-docs-navigation` whenever a skill says "verify", the SDK
   version differs from 4.28, or license gating matters.
5. `here-ios-performance` + `here-ios-troubleshooting` for review passes and
   runtime issues.

### Example prompts the skills are tuned for

- "Create an iOS app showing a HERE map in SwiftUI."
- "Search nearby restaurants and drop markers for the results."
- "Draw a truck route and show the maneuvers list."
- "Add turn-by-turn navigation with voice prompts and speed warnings."
- "Let users download Europe for offline use, then navigate offline."
- "Our app uses too much data — audit it." / "Blank map, how do I debug?"

## 5. Suggestions & future work

- **Golden-prompt evaluation set** (~20 tasks × expected checklist outcomes)
  run against an agent with/without the skills; fold misses into checklist
  lines.
- **Automated staleness check**: a small job comparing each fetched page's
  `updatedAt` frontmatter against the build date to flag skills needing
  review (the `.md` pages expose it — ideal for CI).
- **Harvested recipe catalog**: like `here-maps-web`'s `here-task-recipes`,
  a script could index the `here-sdk-examples` repo (tree API → per-example
  API-symbol index) for code-first lookups; the current
  `here-ios-examples` table is hand-curated.
- **Version-scoped skills** if HERE ships a breaking release; pin via a
  version banner in frontmatter and let `here-ios-docs-navigation` detect
  drift.
- **Split navigation skill** (`voice`/`warners`/`lanes`/`positioning`
  sub-pages) if usage data shows the page grows unwieldy; it's the largest
  skill today.
- Regional/vertical deep-dives (Japan data, truck amenities, indoor maps,
  venues) are cross-referenced only — promote if demand shows.

---

*Built from the official HERE SDK documentation (subject to the
[HERE Documentation License](https://legal.here.com/en-gb/terms/documentation-license));
example links point to the official `heremaps` GitHub repositories.*