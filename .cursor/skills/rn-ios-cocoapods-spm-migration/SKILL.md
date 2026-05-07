---
name: rn-ios-cocoapods-spm-migration
description: >-
  Migrates or plans migration of React Native iOS native dependencies from CocoaPods
  to Swift Package Manager (SPM), including hybrid Pods+SPM setups, Firebase Apple SDK
  timeline context, @react-native-firebase and react-native-screens notes, duplicate-symbol
  avoidance, Podfile modular headers for Firebase Swift pods, CI Package.resolved, and
  verification. Use when the user works on iOS Podfile, Xcode package dependencies,
  RNFirebase, Firebase CocoaPods deprecation, SPM coexistence, or manual Pod-to-SPM steps.
disable-model-invocation: false
---

# React Native iOS: CocoaPods ↔ Swift Package Manager

## Scope

- **React Native** apps using **`ios/*.xcworkspace`** + CocoaPods (autolinking).
- **Coexistence:** CocoaPods for RN and most native modules; SPM for selected Swift packages on the app target.
- **Firebase:** treat [official CocoaPods migration doc](https://firebase.google.com/docs/ios/cocoapods-deprecation) and **React Native Firebase** release notes / [issue #9010](https://github.com/invertase/react-native-firebase/issues/9010) as source of truth for timeline and supported layouts.

## Long-form reference

Full step-by-step guide, obstacle catalog, and sample-repo explanation: [reference.md](reference.md).

## Core concepts

| Layer | Typical location |
|-------|------------------|
| JS | `package.json`, Metro |
| Native iOS | `ios/Podfile`, Xcode **Package Dependencies**, `project.pbxproj`, `Package.resolved` |

`npm update` alone does not migrate native iOS; **Podfile / Xcode** must match.

## Critical rules

1. **Never link the same native SDK twice** (e.g. Firebase via **both** RNFB’s CocoaPods **and** SPM Firebase products) unless upstream docs explicitly support it. Expect **duplicate symbol** / runtime issues otherwise.
2. **Always open `ios/*.xcworkspace`**, not the bare `.xcodeproj`, when using CocoaPods.
3. **Firebase via RNFB today:** usually **CocoaPods** transitive pods (`Firebase/CoreOnly`, etc.). A **neutral** SPM library (e.g. Kingfisher for image loading) may be used only to **prove** SPM + Pods coexist—it is **not** part of Firebase migration.
4. **`react-native-screens`:** consume via **podspec / autolinking**; app authors typically do **not** “migrate screens to SPM” unless maintainers document it.

## Pre-migration audit (do first)

- **A1:** `Podfile` + `Podfile.lock` — is the dependency **direct** or **transitive**? If only transitive, you cannot drop it without changing the parent pod / upgrading the library.
- **A2:** Map CocoaPods **subspecs** to SPM **product** names (they often differ).
- **A3:** SPM minimum **iOS** vs app deployment target.
- **A4:** CI: same **Xcode** major as local; commit **`Package.resolved`**; run **`xcodebuild -resolvePackageDependencies`** or full build on CI.

## Manual flow: move one library from Pod → SPM

1. Remove `pod 'Lib', …` from `Podfile` (if you own it); `cd ios && bundle exec pod install`; confirm build from **workspace**.
2. Xcode: **Project → Package Dependencies → +** → Git URL from library README → version rule → add **product** to **app** (and extensions if needed).
3. `import Lib` in Swift; **Clean Build Folder**; fix “No such module” by ensuring the SPM product is on the target.
4. Commit `project.pbxproj` and `**/swiftpm/Package.resolved`.

## Firebase-specific (when upstream documents SPM)

1. Read RNFB changelog + Firebase install docs for **your** major versions.
2. Align **npm** `@react-native-firebase/*` versions; `pod install`.
3. Apply **only** documented native changes (Pods removal vs SPM add).
4. Verify **no duplicate** Firebase in the link step.

## Common obstacles (symptom → fix)

| Symptom | Likely fix |
|--------|------------|
| Swift pod + static libs / “does not define modules” / `GoogleUtilities` | In `Podfile` target, before `use_react_native!`: `pod 'GoogleUtilities', :modular_headers => true` (or the pod named in the error). Avoid global `use_modular_headers!` unless team accepts blast radius. |
| Duplicate `FIR*` / Firebase symbols | Remove duplicate source: one of SPM vs Pods for Firebase. |
| `No such module` for SPM | Link SPM product to correct target; clean / DerivedData. |
| CI resolves Pods but not SPM | Add `xcodebuild -resolvePackageDependencies` or full build. |
| Wrong project opened | Use `.xcworkspace`. |

## Verification checklist

- [ ] `bundle exec pod install` **0**
- [ ] Debug + **Release** build; **Archive** if shipping
- [ ] No duplicate symbols in link log
- [ ] Smoke-test flows using affected native code
- [ ] Production Firebase: **`GoogleService-Info.plist`** on app target, `firebase.json` if using RNFB config

## Official links

- [Firebase: Migrate from CocoaPods](https://firebase.google.com/docs/ios/cocoapods-deprecation)
- [Firebase: Installation methods (SPM)](https://firebase.google.com/docs/ios/installation-methods)
- [RNFirebase SPM / CocoaPods tracking](https://github.com/invertase/react-native-firebase/issues/9010)
