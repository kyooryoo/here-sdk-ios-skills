---
name: here-ios-docs-navigation
description: >
  Verify HERE SDK for iOS APIs, versions, and license gating against the live
  official documentation. Use whenever a class/method/option shown in another
  here-ios-* skill has not been seen before, the SDK version may have moved
  past 4.28, a changelog/known-issues check is needed, or official example
  code is required. Also use when asked what a HERE SDK for iOS feature is
  called or whether it exists.
---

# HERE SDK iOS — Docs navigation

The guide publishes **agent-friendly plain-markdown pages** (`.md` URLs) and
`llms.txt` indexes. Prefer them over scraping HTML. When any other
`here-ios-*` skill says "verify via `here-ios-docs-navigation`", do it —
never invent an API.

## Indexes (start here)

| Index | URL |
|---|---|
| Site index | <https://docs.here.com/llms.txt> |
| HERE SDK project index | <https://docs.here.com/here-sdk/llms.txt> |
| iOS guide index (every page) | <https://docs.here.com/here-sdk/docs/here-sdk-for-ios-developer-guide/llms.txt> |
| Changelogs index | <https://docs.here.com/here-sdk/docs/changelogs/llms.txt> |

## Fetching a page

Take the slug from the iOS guide index and append `.md`:

```
https://docs.here.com/here-sdk/docs/<slug>.md
```

Examples: `ios-routing.md`, `ios-key-concepts.md`,
`ios-navigation-lane-assistance.md`. The frontmatter carries `updatedAt` —
use it to judge staleness.

## What to verify, and where

| Question | Page (slug) |
|---|---|
| Does this API exist / exact Swift signature? | iOS guide page for the feature, then the **API Reference** (Swift interface ships in the framework — `heresdk.xcframework`; also browsable on the docs site) |
| Explore or Navigate license? | `ios-introduction-feature-list` — **Navigate-only pages carry an explicit banner**; if a page shows a "…only available with the Navigate license" callout, flag it to the user |
| Is it new/beta/deprecated? | Changelog (`here-sdk-for-ios-navigate-changelog` / `here-sdk-for-ios-explore-changelog`; keyword-searchable back to 4.8.2.0) |
| What changed when upgrading? | Changelog for the target version (guide pages carousel every 4.28-era page) |
| Known bug vs our bug? | Changelog "known issues" + `ios-tips-troubleshooting` |
| Canonical working example | <https://github.com/heremaps/here-sdk-examples> — `examples/latest/explore/ios/…` or `examples/latest/navigate/ios/…`; Swift entry point is `<AppName>Example.swift` (see `here-ios-examples`) |
| Class list (superset of both licenses) | API Reference — includes Navigate+Explore; license gating is per-page, not per-reference |
| Current map version / OCM catalog | `getCurrentMapVersion()` / `MapVersionHandle` (`ios-tips-engines`) |

## Rules

1. **Pin the documented version.** The current guide documents **4.28.0.0**;
   other framework versions "may vary and you may need to adapt the source
   code." Confirm what the app actually links in the Xcode project, and match
   the guide/changelog to it.
2. **Beta APIs have no deprecation process** — call that out when a skill
   uses a beta API (custom raster layers, electronic horizon, indoor maps,
   offline search index, map matching…). Deprecated APIs are kept for ~two
   major releases; iOS-only note: **Objective-C is not supported** (Swift
   only).
3. **Never mix generations or editions**: 3.x credentials don't work with
   4.x; Explore credentials don't work with Navigate APIs.
4. If the fetched page contradicts a skill in this repo, **trust the live
   page and report the drift** — then the skill should be updated.
5. Report the exact URL used when citing, so the user can spot-check.
6. iOS pages use Swift (`do/catch` instantiation, closure completions with
   XOR error/result, callbacks on the main thread) — don't transcribe Android
   Kotlin where the Swift spelling differs.

## Example code sources

- Examples repo: <https://github.com/heremaps/here-sdk-examples>
- Icons (maneuver, lane, UI): <https://github.com/heremaps/here-icons>
- Ref app (Flutter only): `github.com/heremaps/here-sdk-ref-app-flutter`
- Support/bugs: <https://www.here.com/contact> (include SDK version,
  platform, license, logs, repro steps); GitHub issues only for
  docs/example problems.