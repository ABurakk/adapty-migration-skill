# Step 0 — Platform Detection (run before everything else)

This file is read FIRST on every invocation of the skill, in both Mode A (Q&A) and Mode B (full migration). Its single job: classify the project (or, in Mode A, the snippet's context) as exactly ONE of `ios` / `flutter` / `mixed` / `other`, and commit the rest of the conversation to the matching platform path.

Skipping Step 0 — or running it lazily — is the dominant failure mode this skill exists to prevent. The Swift Adapty API and the Dart Adapty API are not interchangeable: same concepts, different signatures, different async style, different module structure. Mixing them silently produces code that does not compile or, worse, compiles and fails at runtime in ways that look like Adapty bugs.

---

## 1. Detection signals (strongest → weakest)

Scan the project root and one or two levels deep. The signals below are ranked: a single strong signal is conclusive; weak signals only matter in the absence of strong ones.

### 1a. Strong Flutter signals (any one of these → Flutter)

- A `pubspec.yaml` file at the project root.
- A `pubspec.yaml` listing **`purchases_flutter:`** under `dependencies:` (with or without a version pin).
- A `pubspec.yaml` listing **`purchases_ui_flutter:`** under `dependencies:` (additionally signals the Paywall Builder path).
- Any `.dart` file with `import 'package:purchases_flutter/purchases_flutter.dart';`
- Any `.dart` file with `import 'package:purchases_ui_flutter/purchases_ui_flutter.dart';`
- A `lib/` directory containing `main.dart` and other `.dart` files.

### 1b. Strong iOS signals (any one of these → iOS)

- An `.xcodeproj` or `.xcworkspace` directory at the project root.
- A `Package.swift` declaring a dependency on `https://github.com/RevenueCat/purchases-ios`.
- A `Podfile` listing `pod 'RevenueCat'` or `pod 'RevenueCatUI'`.
- A `Cartfile` referencing `RevenueCat/purchases-ios`.
- Any `.swift` file with `import RevenueCat` or `import RevenueCatUI`.
- An `Info.plist` + Swift `App` / `AppDelegate` + no `pubspec.yaml` anywhere in the tree.

### 1c. Out-of-scope signals (any of these → `other` / refuse)

- `build.gradle` / `build.gradle.kts` + no `pubspec.yaml`, with `com.revenuecat.purchases:purchases:` in dependencies → **Android-native (Kotlin/Java) — out of scope.**
- `package.json` with `react-native` and `react-native-purchases` (or similar RN RevenueCat package) → **React Native — out of scope.**
- A Unity project layout (`Assets/`, `ProjectSettings/`) with RevenueCat Unity plugin → **Unity — out of scope.**
- A KMP / Kotlin Multiplatform project structure → **out of scope.**
- A web project (`package.json` + browser bundler config) with any RC web wrapper → **out of scope.**

For all `other` cases, refuse with a one-line explanation citing the detected stack, and stop. Do NOT attempt a partial migration even if "some files look migrable."

### 1d. Weak / ambiguous signals (only used if no strong signal fires)

- Project name or folder name containing "flutter" / "ios" / "swift" — informational only, not conclusive.
- The user's prose ("my Flutter app", "this Swift project") — informational, can override only weak signals.
- A bare `.swift` snippet pasted in Mode A → likely iOS, but a Flutter project's `ios/Runner/` folder also contains `.swift` files. If the file path is `ios/Runner/...` or `ios/Classes/...` inside a Flutter project tree, treat as Flutter (the iOS files in a Flutter project are not the migration target unless they contain custom RevenueCat code — see §3 below).

If no strong signal fires after a recursive scan, do not guess: ask the user *"Is this an iOS Swift project or a Flutter project?"* and proceed only after they answer.

---

## 2. The four classifications

After scanning, decide on exactly ONE classification:

### `ios`

- Strong iOS signals fire. No strong Flutter signals fire.
- All migration work proceeds against Swift sources under `references/ios/`.
- Output language: Swift only. No `.dart` files emitted. No `pubspec.yaml` written.

### `flutter`

- Strong Flutter signals fire. No standalone iOS-native RevenueCat surface that lives outside Flutter's expected `ios/` host folder (i.e., the iOS host folder contains only standard Flutter Runner code, no app-owned `import RevenueCat`).
- All migration work proceeds against Dart sources (`lib/**/*.dart`) plus `pubspec.yaml`, under `references/flutter/`.
- Output language: Dart only. No `.swift` files emitted (except the narrow bridge exception below). No `Podfile` / `Package.swift` / `.pbxproj` edits.

### `mixed`

- Both a Flutter `purchases_flutter` surface AND an iOS-native RevenueCat surface are present in the same repo.
- Most common shape: a Flutter app whose `ios/Runner/` folder contains additional app-owned Swift code that calls `Purchases.shared.*` directly (e.g., a custom platform channel bridge that pre-dates the Flutter SDK adoption).
- **Action:** ask the user once, before doing anything else: *"This repo has both a Flutter `purchases_flutter` integration and additional iOS-native RevenueCat code in `ios/Runner/`. I can migrate one of them in this run — which would you like? Default: Flutter side only (the iOS Swift code will be flagged as Manual review)."*
- Default if the user does not answer / is unclear: migrate the Flutter side only, leave the iOS Swift RevenueCat code in place with `// TODO: ADAPTY MANUAL — file detected as iOS-native RevenueCat in a Flutter project; rerun this skill against the iOS bridge separately if you want it migrated.`
- Never migrate both sides in a single run — even if the user asks. Each path requires loading its own reference files, and the isolation rule (SKILL.md §0) is non-negotiable. Offer to run the skill twice instead.

### `other` / `refuse`

- Android-native, React Native, Unity, KMP, web, or any other platform.
- Respond with a one-line refusal naming the detected stack, suggest the user check the Adapty docs for the matching SDK (`https://adapty.io/docs/`), and stop.
- Example refusal: *"This is a React Native project (`package.json` includes `react-native-purchases`). This skill only supports iOS Swift and Flutter migrations. For React Native, see Adapty's React Native SDK docs at `https://adapty.io/docs/`."*

---

## 3. The Flutter narrow Swift-bridge exception

A Flutter app may contain app-owned native Swift code in `ios/Runner/` (or `ios/Classes/`) that calls `Purchases.shared.*` directly. This is rare — typically the result of an early platform channel bridge written before the Flutter RC SDK was available, or a piece of custom StoreKit/RC interop. It is NOT the same as Flutter's standard generated `ios/Runner/AppDelegate.swift`, which contains no RC code.

If this code is detected during a `flutter` classification:

1. Do NOT silently migrate it. The Flutter path's references contain no Swift patterns.
2. Do NOT load `references/ios/*.md` to migrate it. Doing so would violate the isolation rule.
3. Flag it as a Manual review item with a clear instruction: *"`ios/Runner/<File>.swift` contains custom RevenueCat Swift code. This file is outside the Flutter migration's scope. Either (a) port the logic into Dart via the Flutter Adapty SDK, or (b) rerun this skill against just the iOS Swift bridge as a separate migration."*
4. Leave the original Swift file untouched in the output.

This rule preserves isolation: a single run still produces only Dart output, never mixed Dart+Swift Adapty code.

---

## 4. Recording the classification

Once the classification is decided, state it explicitly to the user in the very first response that involves any reference reading or code emission. Example openings:

- *"Detected: iOS Swift project (found `Package.swift` declaring `purchases-ios`, and `import RevenueCat` in `Sources/App/MyApp.swift`). Proceeding with the iOS migration path."*
- *"Detected: Flutter project (found `pubspec.yaml` with `purchases_flutter: ^7.0.0` and `purchases_ui_flutter`, plus `import 'package:purchases_flutter/purchases_flutter.dart';` in `lib/services/billing.dart`). Proceeding with the Flutter migration path; Paywall Builder flow because `purchases_ui_flutter` is present."*
- *"Detected: mixed (Flutter `purchases_flutter` + a custom Swift `Purchases.shared.logIn` call in `ios/Runner/Bridge.swift`). I'll migrate the Flutter side only and flag the iOS Swift file for separate review — say "iOS instead" if you want to migrate the Swift bridge in this run instead of the Flutter side."*
- *"This looks like an Android-native (Kotlin) project (`build.gradle` with `com.revenuecat.purchases:purchases:7.x`). This skill only handles iOS Swift and Flutter migrations — for Android-native, see Adapty's Android SDK docs at `https://adapty.io/docs/`."*

Stating the classification has two purposes: it gives the user a chance to correct a misclassification immediately, and it documents the path choice for the rest of the conversation.

---

## 5. What to do if the user disputes the classification

If the user says "no, this is a <different platform>," do not silently switch. Re-run Step 0 with the new information weighted in, and either:

- Confirm the new classification and explicitly state which references will be loaded, OR
- Ask one clarifying question if the signals still conflict (e.g., "I see both `pubspec.yaml` and a standalone iOS-only RevenueCat file — which side did you mean?").

Never proceed with a mixed code output. Always commit to one platform per run.

---

## 6. Mode A specifics — when there's no project to scan

In Mode A (Q&A lookup), there is often no project folder, just a pasted snippet or a code question. The same classification logic applies, with the signals coming from the snippet itself:

- Swift syntax (`import RevenueCat`, `Purchases.shared.*`, `try await`, `let`, `func ... async throws`) → `ios`.
- Dart syntax (`import 'package:...';`, `Purchases.logIn(...)`, `await`, `Future`, `class ... extends`, no `let`, `final` everywhere) → `flutter`.
- A bare API name like `Purchases.logIn("u")` is ambiguous (RevenueCat has it on both platforms with slightly different shapes) — ask once: *"Is this from an iOS Swift project or a Flutter project? The Adapty equivalent uses a different SDK package on each."*

Once the platform is fixed, load ONLY the matching `references/<platform>/revenuecat-to-adapty-map.md` and proceed.
