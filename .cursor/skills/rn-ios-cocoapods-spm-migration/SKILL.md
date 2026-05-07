---
name: rn-ios-cocoapods-spm-migration
description: >-
  Plans or debugs migrating iOS native dependencies from CocoaPods to Swift Package
  Manager (SPM) in React Native apps: hybrid Pods+SPM, autolinking vs direct pods,
  duplicate-symbol avoidance, transitive deps, modular headers / static linkage,
  SPM target linking, CI and Package.resolved, and verification. Use when the user
  touches ios/Podfile, Podfile.lock, Xcode Package Dependencies, duplicate symbols,
  “could not build module”, SPM resolution failures, or any library’s Pod-to-SPM move.
disable-model-invocation: false
---

# React Native iOS: CocoaPods ↔ Swift Package Manager

## What this skill is for

- **Migrating** a specific native library from a **Pod** to an **SPM** product (when the vendor supports both).
- **Diagnosing** build and link failures after Pod / SPM / Xcode changes—without assuming a particular vendor (analytics, DB, UI, networking, etc.).
- **React Native context:** most apps still use **`ios/*.xcworkspace`** + CocoaPods for the core stack and autolinked modules; SPM is usually added **on top** for chosen packages.

## Long-form narrative

Repo guide (phases, obstacles, examples): [reference.md](reference.md).

## Layers (where to look)

| Concern | Where |
|--------|--------|
| JS package version | `package.json` |
| Native module wiring | npm package **podspec**, autolinking, `ios/Podfile` / `Podfile.lock` |
| SPM | Xcode **Package Dependencies**, `project.pbxproj`, `**/swiftpm/Package.resolved` |

Changing only npm does **not** migrate native iOS if Pods or Xcode packages still pull the old binary.

## Critical rules (any library)

1. **One supplier per native SDK.** Do not link the **same** compiled library twice (e.g. once via Pods and again via SPM) unless upstream documents that split. Symptom: **duplicate symbol** at link time or unstable runtime.
2. **Use the workspace.** With CocoaPods, open **`ios/*.xcworkspace`**, not the bare `.xcodeproj`.
3. **Know direct vs transitive.** If `Podfile.lock` shows the pod only as a **dependency of another pod**, you cannot drop it by deleting your own `pod` line—you must upgrade/replace the **parent** or wait for maintainers.
4. **Map Pod names to SPM products.** Subspecs and SPM **product** names often differ; read the library’s SPM README.

## Audit before changing anything

- **A1:** Grep `Podfile` and `Podfile.lock` for the library; note **direct** vs **transitive** owners.
- **A2:** List exact **SPM products** and version rules from the vendor; match to previous Pod subspecs.
- **A3:** Compare package **minimum iOS** (and Swift tools) to the app target and `platform :ios` in the Podfile.
- **A4:** CI must use a comparable **Xcode** version; commit **`Package.resolved`** when the team locks SPM versions; ensure CI runs package resolution (`xcodebuild -resolvePackageDependencies` or a full build).

## Manual flow: move one library Pod → SPM

1. Remove the **`pod '…'`** line you control (if any); run `cd ios && bundle exec pod install`; build from the **workspace** as a baseline.
2. Xcode: **Project → Package Dependencies → +** → vendor Git URL → rule → attach **products** to the **app** target (and app extensions that need them).
3. Fix imports (`import ModuleName` in Swift; follow vendor for Obj-C / `.mm`).
4. **Clean Build Folder**; if `No such module`, re-check target membership for the SPM product.
5. Commit `project.pbxproj` and `Package.resolved` paths Xcode updates.

## Symptom → likely cause → what to try

| Symptom | Likely cause | What to try |
|--------|----------------|------------|
| Duplicate symbol `_OBJC_CLASS_$_…` / same vendor prefix twice | Same static code linked from **Pod + SPM** (or two Pods) | Remove one integration path; `pod install`; inspect **Link Binary** / build log for duplicate `.a` / frameworks. |
| Swift / “does not define modules” / static library | Obj-C pod lacks module map; RN often uses **static** linkage | In `Podfile`, `:modular_headers => true` on the **pod named in the error** (often a low-level util pod). Avoid blanket `use_modular_headers!` unless the team accepts side effects. |
| `No such module 'X'` (SPM) | Product not linked to target | Target → Frameworks / SPM product list; re-add package product to correct target. |
| Package resolve fails | URL, auth, or version rule | Fix URL and token/SSH for private repos; relax or pin version; align Xcode with local. |
| Build OK locally, fails on CI | Missing resolve / wrong Xcode | Commit `Package.resolved`; pin Xcode; run `xcodebuild -resolvePackageDependencies` in CI. |
| Opens `.xcodeproj` and Pods missing | Wrong entry point | Always **`*.xcworkspace`**. |
| Errors after toggling New Architecture | Native module / Swift interop | Treat as **integration** issue: align RN, native modules, and vendor pods; check issue trackers for the **specific** modules in the stack. |

## Ecosystem-specific (only when relevant)

- **Wrappers (e.g. RN bindings)** often still ship **podspecs**; the app author migrates **underlying** Apple SDKs only when the **wrapper’s** docs say so. Do not add SPM for an SDK that the wrapper still pulls via CocoaPods unless documented—see **duplicate symbol** row.
- **High-churn SDKs** (e.g. some Google / Firebase Apple SDKs) may publish **timeline** or install-method docs; use the **vendor’s** official migration page for that product, not generic guesses.

## Verification

- [ ] `bundle exec pod install` succeeds.
- [ ] **Debug** and **Release** builds; **Archive** if you ship to the store.
- [ ] Link log free of **duplicate symbol** for the migrated library.
- [ ] Smoke-test code paths that call into the native library.

## Reference links (general)

- [Swift Package Manager (Apple)](https://www.swift.org/package-manager/)
- [Adding package dependencies to an app (Xcode)](https://developer.apple.com/documentation/xcode/adding-package-dependencies-to-your-app) (concept; UI may vary by Xcode version)
- [CocoaPods guides](https://guides.cocoapods.org/using/getting-started.html)
