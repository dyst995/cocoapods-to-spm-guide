# Migrating iOS native dependencies from CocoaPods to Swift Package Manager (React Native)

This guide is for teams that maintain React Native apps on iOS and need a **manual, step-by-step** path when an Apple library (or the Firebase Apple SDK) moves from **CocoaPods** to **Swift Package Manager (SPM)**. It complements official docs from [Firebase](https://firebase.google.com/docs/ios/cocoapods-deprecation) and library maintainers (for example [Invertase / React Native Firebase](https://github.com/invertase/react-native-firebase)).

### Sample repository: Firebase works as intended; Kingfisher is only an SPM example

In the sample project that accompanies this guide, **Firebase is wired the way React Native Firebase expects today**: `@react-native-firebase/app` brings in the Firebase Apple SDK through **CocoaPods** (for example `Firebase`, `FirebaseCore`, and related pods). That **CocoaPods-based Firebase integration is intentional and works as intended** in the sample—it validates autolinking, the Podfile adjustments sometimes required for Firebase Swift pods under React Native’s static linking, and a successful iOS build **without** moving Firebase to SPM.

**Kingfisher is unrelated to Firebase** and was added **only as a teaching example**. [Kingfisher](https://github.com/onevcat/Kingfisher) is a **Swift library for downloading, caching, and displaying images** (similar in role to other iOS image loaders). It is distributed via both [CocoaPods](https://cocoapods.org/pods/Kingfisher) and **SPM**; the sample adds it **through SPM alone** so you can prove that **SPM and CocoaPods can live in the same Xcode workspace** next to React Native and RNFB. If you do not need image loading in your app, you may remove Kingfisher and use any other neutral library that supports both Pod and SPM for the same kind of demonstration.

---

## Table of contents

1. [Sample repository: Firebase works as intended; Kingfisher is only an SPM example](#sample-repository-firebase-works-as-intended-kingfisher-is-only-an-spm-example)
2. [Why this matters (Firebase and October 2026)](#why-this-matters-firebase-and-october-2026)
3. [Concepts: two layers in a React Native iOS app](#concepts-two-layers-in-a-react-native-ios-app)
4. [Can CocoaPods and SPM coexist?](#can-cocoapods-and-spm-coexist)
5. [Before you migrate: audit checklist](#before-you-migrate-audit-checklist)
6. [Manual migration: generic library (Pod → SPM)](#manual-migration-generic-library-pod--spm)
7. [Manual migration: Firebase Apple SDK (future / library-driven)](#manual-migration-firebase-apple-sdk-future--library-driven)
8. [Library-specific notes](#library-specific-notes)
9. [Obstacles you might face (symptoms → causes → fixes)](#obstacles-you-might-face-symptoms--causes--fixes)
10. [Verification after migration](#verification-after-migration)
11. [Verified setup in this repository](#verified-setup-in-this-repository)
12. [Further reading](#further-reading)

---

## Why this matters (Firebase and October 2026)

Google has documented a move away from publishing **new** Firebase Apple SDK versions via CocoaPods, with SPM as a recommended install path. See [Migrate from CocoaPods](https://firebase.google.com/docs/ios/cocoapods-deprecation) for the authoritative timeline and FAQ.

**What changes for you:** you may need to change **how the native Firebase binaries are attached in Xcode** (Pods vs SPM), while your **JavaScript** packages (`@react-native-firebase/*`) may still come from npm. The exact steps depend on the version of React Native Firebase and what that version officially supports—always read the changelog for your major version and [tracking issues](https://github.com/invertase/react-native-firebase/issues/9010).

---

## Concepts: two layers in a React Native iOS app

| Layer | What it is | Typical tool today |
|--------|------------|-------------------|
| **JS / npm** | `react-native-screens`, `@react-native-firebase/app`, etc. | `package.json`, Metro |
| **Native iOS** | Compiled code Xcode links into the app | CocoaPods (autolinking) + sometimes SPM |

**Important:** migrating “the library” on iOS usually means changing the **native** integration (Podfile, Xcode package dependencies, build settings). Running `npm update` alone is not enough if the native side still pulls an old Pod.

Your migration is only “done” when:

- There is **no duplicate** of the same native SDK (see [Critical rule](#critical-rule-avoid-linking-the-same-sdk-twice)).
- **Debug and Release** (and ideally **Archive**) builds succeed.
- Your **New Architecture** setting (on/off) matches what your dependencies support.

---

## Can CocoaPods and SPM coexist?

Yes. Xcode supports **both** in the same project:

- **CocoaPods** continues to manage React Native and most community native modules through the **`.xcworkspace`** and the `Pods` project.
- **SPM** packages are attached to your **app target** (or extensions) via **File → Add Package Dependencies…** in Xcode, which updates `project.pbxproj` and usually creates or updates **`Package.resolved`**.

This repository demonstrates that combination: React Native, **@react-native-firebase/app** (and thus Firebase via **CocoaPods**), and **react-native-screens** install through CocoaPods, while **Kingfisher** is added only through SPM as a **standalone example**—not as part of Firebase migration (see [Sample repository](#sample-repository-firebase-works-as-intended-kingfisher-is-only-an-spm-example)). Kingfisher ships via [CocoaPods](https://cocoapods.org/pods/Kingfisher) and [SPM](https://github.com/onevcat/Kingfisher); the sample uses SPM to prove the hybrid layout without duplicating Firebase.

### Critical rule: avoid linking the same SDK twice

If a native dependency is already brought in by CocoaPods (for example `Firebase/CoreOnly` via `@react-native-firebase/app`), **do not** add the same Firebase products again from SPM unless the library’s documentation explicitly describes that split.

**Typical failure modes:** duplicate symbol errors at link time, crashes, or two different versions of Firebase loaded at once.

When Firebase’s CocoaPods distribution winds down, the sustainable approach is **one** integration path for the Apple SDK, wired the way React Native Firebase (or your stack) expects.

---

## Before you migrate: audit checklist

Do this **before** deleting Pods or adding SPM packages.

### Step A1 — Inventory CocoaPods usage

1. Open `ios/Podfile` and note every direct `pod '…'` you added yourself.
2. Open `ios/Podfile.lock` and search for the library you plan to move (for example `Kingfisher`, `Alamofire`, `Firebase`).
3. Identify **who** pulls it:
   - **Direct:** your own line in the Podfile.
   - **Transitive:** pulled by React Native, `RNFBApp`, another pod, etc.

**Obstacle:** If the dependency is **only transitive**, you cannot “remove Kingfisher” from the Podfile until you change the parent (for example upgrade a pod that dropped that dependency). You may need to wait for a maintainer or use a fork.

### Step A2 — Map Pod subspecs to SPM products

SPM **product names** and CocoaPods **subspecs** are not always identical.

1. Open the library’s README for SPM and list the **products** you must add (for example `Kingfisher`, `KingfisherSwiftUI`).
2. Match each product to what you used in CocoaPods (for example `pod 'Kingfisher', '~> 8.0'`).

**Obstacle:** Some libraries expose **extra modules** only on CocoaPods or only on SPM. If you need a rare subspec, confirm SPM exposes it before migrating.

### Step A3 — Confirm Xcode and iOS deployment target

1. Note your **minimum iOS version** in Xcode (app target → General → Minimum Deployments) and in `Podfile` (`platform :ios, …`).
2. Confirm the Swift package supports that minimum OS version.

**Obstacle:** SPM may require a **newer iOS** than your app currently supports. You then either raise the deployment target or stay on CocoaPods until you can.

### Step A4 — Decide CI behavior

1. Ensure CI uses the **same Xcode major version** as local developers.
2. Plan to **commit** `Package.resolved` when your team agrees on lockfile policy (highly recommended for reproducible builds).

**Obstacle:** CI runs `pod install` but never opens Xcode, so **package resolution** might not happen until someone runs a build. Use `xcodebuild -resolvePackageDependencies` or a full build step in CI (see [Verification](#verification-after-migration)).

---

## Manual migration: generic library (Pod → SPM)

Use this when **you** (or a direct Podfile entry) control the dependency, and the library officially supports SPM.

### Phase 1 — Remove the CocoaPods integration

**Step 1.1** — In `ios/Podfile`, remove the `pod 'YourLibrary', …` line(s) for that library. If you never added it directly, stop here and resolve the transitive case (audit Step A1).

**Step 1.2** — Search the repo for other references:

- `grep -r "YourLibrary" ios/` (and your app’s native folders).
- Check **`.podspec` local path pods** or `podspec` URLs if you vendor code.

**Step 1.3** — From the `ios` directory, refresh Pods:

```bash
cd ios
bundle exec pod install
```

**Step 1.4** — Open **`ios/*.xcworkspace`** in Xcode (not the bare `.xcodeproj`). Confirm the project still builds **before** adding SPM, so you know any later failure is from the new integration.

**Obstacle:** `pod install` fails because another pod still depends on the library you removed. You must upgrade that parent pod or keep the child on CocoaPods until the ecosystem catches up.

---

### Phase 2 — Add the Swift package in Xcode (by hand)

**Step 2.1** — In Xcode, with the **workspace** open, select the **project** (blue icon) in the navigator.

**Step 2.2** — Select the **app target** (for example `podLibraryMigrationGuide`) → **General** → scroll to **Frameworks, Libraries, and Embedded Content** if you want to confirm what is linked (optional sanity check).

**Step 2.3** — Select the **project** again → **Package Dependencies** tab.

**Step 2.4** — Click **+** (Add).

**Step 2.5** — Paste the **Git URL** from the library’s documentation (example for Kingfisher: `https://github.com/onevcat/Kingfisher.git`).

**Step 2.6** — Choose a **dependency rule**:

- **Up to Next Major** is common for apps.
- **Exact Version** or **Commit** is common for strict reproducibility.

**Step 2.7** — Add the package. When Xcode prompts for **targets**, check your **application target** (and extensions that need the code).

**Step 2.8** — Wait for **Resolve Package** to finish. If it fails, read the error in the Report navigator; see [Package resolution failures](#package-resolution-failures).

**Obstacle:** You added the package to the **project** but not the **target**. Symptom: `No such module 'YourLibrary'` in Swift. Fix: target → **General** → **Frameworks…** → **+** → add the SPM product, or re-run the package add flow and tick the target.

---

### Phase 3 — Wire code and build settings

**Step 3.1** — In Swift files that use the API, add:

```swift
import YourLibrary
```

**Step 3.2** — If you used the library from **Objective-C++** (`.mm`) or bridging headers, check the library’s docs for Obj-C API or module name. SPM-first libraries are often Swift-first.

**Step 3.3** — **Product → Clean Build Folder**, then build.

**Step 3.4** — If you use **precompiled binaries** or **resource bundles** from the old Pod, confirm SPM exposes them (some packages use `.process()` resources; behavior can differ from CocoaPods resource scripts).

**Obstacle:** **Clang / Swift explicit modules** or **New Architecture** exposes missing module maps. See [Module map and “could not build module”](#module-map-and-could-not-build-module) and [New Architecture and Firebase-related pods](#new-architecture-and-firebase-related-pods).

---

### Phase 4 — Source control and team handoff

**Step 4.1** — In git status, expect changes to:

- `ios/**/project.pbxproj`
- `ios/**/xcworkspace/xcshareddata/swiftpm/Package.resolved` (path may vary slightly by Xcode version)

**Step 4.2** — Commit `Package.resolved` unless your org has a different policy (not recommended for apps).

**Step 4.3** — Document the **SPM URL and version rule** in your internal wiki or PR description so the next upgrade is intentional.

**Obstacle:** Teammate opens **`.xcodeproj`** and sees missing packages or wrong schemes. Fix: always use **`.xcworkspace`** for CocoaPods-based React Native.

---

## Manual migration: Firebase Apple SDK (future / library-driven)

You should **not** invent a one-off Firebase SPM layout that fights `@react-native-firebase/app` unless you are following official RNFB docs for your version. When maintainers document SPM, the manual steps will resemble the following **pattern** (exact flags and env vars will differ by release):

### Step F1 — Read the release notes

1. Read [Firebase: Migrate from CocoaPods](https://firebase.google.com/docs/ios/cocoapods-deprecation).
2. Read React Native Firebase [changelog / docs](https://github.com/invertase/react-native-firebase) for your major version and any **SPM** or **CocoaPods sunset** instructions.

### Step F2 — Align npm and native

1. Upgrade `@react-native-firebase/*` packages together (avoid mixing very old and very new native pod expectations).
2. Run `npm install`, then `cd ios && bundle exec pod install`.

### Step F3 — Apply library-specific native changes

This may include (examples only; follow upstream, not this list blindly):

- Removing Firebase pods from the CocoaPods graph **only if** RNFB tells you to.
- Adding Firebase via **Package Dependencies** with the **official** Firebase product list from [installation methods](https://firebase.google.com/docs/ios/installation-methods).
- Setting documented **environment variables** or Podfile hooks if the wrapper needs to disable CocoaPods Firebase.

### Step F4 — Validate no duplicate Firebase

1. Build and search the link command / build log for multiple `FirebaseCore` copies.
2. If you see **duplicate symbol** errors involving `FIR*` symbols, assume two Firebase sources are linked—rollback and fix before shipping.

---

## Library-specific notes

### `react-native-screens`

- **What you do:** keep installing from npm; iOS uses the **`RNScreens`** pod via autolinking.
- **What you usually do not do:** manually “migrate react-native-screens to SPM” as an app developer unless the maintainers publish and document SPM as the primary path.
- **Obstacle:** version skew between JS and native. Fix: align `react-native` and `react-native-screens` using the compatibility notes in the screens repo.

### `@react-native-firebase/app`

- **What you do today:** rely on the **podspec** (`Firebase/CoreOnly`, etc.) and a correct **`GoogleService-Info.plist`**.
- **Podfile tip (common with RN + static linking):** Swift-heavy Firebase pods sometimes need modular headers for Obj-C dependencies:

```ruby
pod 'GoogleUtilities', :modular_headers => true
```

- **Obstacle:** “Swift pod `FirebaseCoreInternal` depends upon `GoogleUtilities`, which does not define modules…” — see the same modular-headers approach and [Obstacles](#obstacles-you-might-face-symptoms--causes--fixes).

---

## Obstacles you might face (symptoms → causes → fixes)

### Duplicate symbols at link time

- **Symptom:** `duplicate symbol` in `Firebase*`, `nanopb`, `GoogleUtilities`, or the library you migrated.
- **Cause:** The same static archive or object code is linked twice (Pod + SPM, or two Pods).
- **Fix:** Remove one source of truth. Use `Podfile.lock` and Xcode’s **Frameworks, Libraries, and Embedded Content** to ensure only one chain supplies that SDK. For Firebase, never add SPM Firebase while RNFB still pulls CocoaPods Firebase unless documented.

---

### “Swift pod cannot yet be integrated as static libraries” / modular headers

- **Symptom:** CocoaPods error mentioning **Swift pods**, **static libraries**, and **module maps** / **GoogleUtilities**.
- **Cause:** React Native often builds dependencies as **static libraries**; some Firebase-related pods need **module maps** for Obj-C dependencies used from Swift.
- **Fix (common):** In `ios/Podfile`, before `use_react_native!`, add:

```ruby
pod 'GoogleUtilities', :modular_headers => true
```

If another pod names a different Obj-C dependency in the error, apply `:modular_headers => true` to **that** pod (or discuss with your team before `use_modular_headers!` globally, which has wider blast radius).

---

### `use_frameworks!` and React Native

- **Symptom:** Exotic link errors, different Hermes / RN behavior, or CocoaPods warnings after enabling `use_frameworks!`.
- **Cause:** Framework linkage changes how symbols are exported; not all React Native stacks test every combination.
- **Fix:** Treat `use_frameworks!` as a **last resort** unless a dependency mandates it. Prefer targeted modular headers or the library’s recommended RN snippet.

---

### Package resolution failures

- **Symptom:** Xcode cannot resolve a package; SSL or git errors; “package graph” unsatisfiable.
- **Causes:** Wrong URL, private repo without credentials, version rule incompatible with your Swift tools version, or transient registry issues.
- **Fixes:**
  - Verify the URL matches the README **exactly** (HTTPS git URL).
  - For private packages, configure **SSH keys** or **netrc** on CI and locally.
  - Align **Xcode** versions across the team.

---

### Wrong project opened (`.xcodeproj` vs `.xcworkspace`)

- **Symptom:** Undefined symbols for React / Pods, or builds that “worked yesterday” fail after Pod changes.
- **Cause:** CocoaPods integrates through the **workspace**. Opening the project file bypasses the `Pods` project.
- **Fix:** Close Xcode, open `ios/YourApp.xcworkspace`, clean build.

---

### `No such module 'PackageName'`

- **Symptom:** Swift compiler error on `import`.
- **Causes:** Product not linked to the target; typo in module name; package only builds for a platform you did not select; stale derived data.
- **Fixes:** Re-add the SPM product to the **app target**; **Clean Build Folder**; delete **DerivedData** for the project if needed.

---

### Module map and “could not build module”

- **Symptom:** Clang errors building umbrella headers, non-modular include inside framework module, etc.
- **Cause:** Mixed C / Obj-C / Swift boundaries, especially with **precompiled headers** or aggressive **non-modular includes**.
- **Fixes:** Prefer the library’s RN + iOS instructions; adjust **Header Search Paths** only when you know why; avoid duplicating Firebase headers from two package managers.

---

### New Architecture and Firebase-related pods

- **Symptom:** Build failures mentioning **React Native New Architecture**, **Fabric**, **TurboModules**, or Swift pods under **static** linkage.
- **Cause:** Interaction between RN’s C++ codegen surface and Swift pods (Firebase has had multiple community reports over time).
- **Fix:** Upgrade **RN**, **RNFB**, and **Firebase** together to versions your maintainers tested; read RNFB issues for your combination; temporarily toggling New Architecture is a **diagnostic** step, not always a product decision.

---

### CI builds but local fails (or the reverse)

- **Symptom:** Drift between machines.
- **Causes:** Different Xcode, missing `Package.resolved`, CI not resolving SPM, cached Pods.
- **Fixes:** Commit `Package.resolved`; pin Xcode on CI; run `pod install` from a clean state; add an explicit `xcodebuild -resolvePackageDependencies` step.

---

### Runtime crash after migration

- **Symptom:** App launches then crashes inside `+[FIRApp configure]` or on first use of a native API.
- **Cause:** Missing **`GoogleService-Info.plist`**, wrong **bundle ID**, or mismatched Firebase product set (for example Analytics not linked but called).
- **Fix:** Treat this as **configuration**, not package manager: verify plist target membership, bundle identifier, and that JS calls match installed native products.

---

## Verification after migration

Run through this list after any Pod or SPM change.

| # | Check |
|---|--------|
| 1 | `cd ios && bundle exec pod install` exits **0**. |
| 2 | Xcode resolves packages with **no** warnings you do not understand. |
| 3 | **Debug** build on a simulator succeeds. |
| 4 | **Release** build succeeds (different optimizations). |
| 5 | **Archive** (or CI export) succeeds if you ship to TestFlight. |
| 6 | No **duplicate symbol** errors in the full link log. |
| 7 | Critical user flows touching native code (push, analytics, screens) are smoke-tested. |

**Command-line package resolution (useful for CI):**

```bash
cd ios
xcodebuild -workspace YourApp.xcworkspace \
  -scheme YourApp \
  -resolvePackageDependencies
```

---

## Verified setup in this repository

The following was validated with `xcodebuild` (Debug, iOS Simulator, `CODE_SIGNING_ALLOWED=NO`):

- **Firebase / React Native Firebase (CocoaPods only):** `@react-native-firebase/app` is integrated **as intended** through CocoaPods and transitive Firebase pods (for example Firebase 12.x for RNFB). This confirms the **current** RNFB + CocoaPods path **works** in a New Architecture React Native app when the Podfile accounts for Firebase’s Swift/Obj-C module needs (see `GoogleUtilities` modular headers in this repo’s `Podfile`). No Firebase products are added via SPM here, so there is **no duplicate Firebase** from mixing package managers.
- **CocoaPods (autolinking, remainder):** React Native 0.85.x, `react-native-screens`, `react-native-safe-area-context`, and other React Native pods.
- **SPM (example only):** [Kingfisher](https://github.com/onevcat/Kingfisher) (resolved at 8.9.x), linked to the app target; `AppDelegate.swift` references `KingfisherManager.shared` **only** to exercise the SPM link step. Kingfisher is an **image caching** library and is **not** required for Firebase; substitute or remove it if you prefer another neutral SPM dependency.

To reproduce locally:

```bash
npm install
cd ios && bundle install && bundle exec pod install
# Then build from Xcode or use xcodebuild with your workspace and scheme.
```

### Firebase configuration you still need for a real app

For production use of React Native Firebase, add **`GoogleService-Info.plist`** from the Firebase console to the Xcode app target and optionally a root **`firebase.json`** for RNFB behavior. This sample focuses on native packaging (Pods + SPM), not Firebase project wiring.

---

## Further reading

- [Firebase: Migrate from CocoaPods](https://firebase.google.com/docs/ios/cocoapods-deprecation)
- [Firebase: Options to install Firebase (SPM)](https://firebase.google.com/docs/ios/installation-methods)
- [React Native Firebase issue tracker (SPM / CocoaPods timeline)](https://github.com/invertase/react-native-firebase/issues/9010)
- [Kingfisher installation (CocoaPods vs SPM)](https://github.com/onevcat/Kingfisher)
- **Cursor:** condensed Agent skill in this repo at [`.cursor/skills/rn-ios-cocoapods-spm-migration/SKILL.md`](../.cursor/skills/rn-ios-cocoapods-spm-migration/SKILL.md)
