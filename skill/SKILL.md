---
name: adapty-migration
description: Migrate apps from RevenueCat to Adapty (current 3.x baseline, verified per-migration against the live Adapty docs) AND answer questions about RevenueCat → Adapty API equivalents. Supports TWO platforms — iOS Swift and Flutter (Dart) — with strict per-platform isolation. The skill ALWAYS performs platform detection first (Step 0) and routes to exactly one platform path; iOS and Flutter content is never mixed. CASE 1 — Full project migration. Triggered by phrasings like "migrate this to Adapty", "replace RevenueCat with Adapty", "swap RevenueCat for Adapty", "rewrite this with Adapty", or any project that has `import RevenueCat` / `import RevenueCatUI` (iOS) or a `pubspec.yaml` containing `purchases_flutter` / `purchases_ui_flutter` (Flutter) with a request to integrate Adapty. Trigger this skill even when the user only says "add Adapty" or "integrate Adapty" on a project that already uses RevenueCat — the migration is the implied task. On iOS the skill detects whether each paywall-touching file uses the Paywall Builder flow (RevenueCatUI → AdaptyUI + `.paywall()` modifier) or a custom paywall flow (custom UI → direct `Adapty.getPaywall` + `Adapty.getPaywallProducts` + `Adapty.makePurchase`) and migrates each appropriately. On Flutter the skill detects whether the app uses `purchases_ui_flutter` (Paywall Builder flow → `AdaptyUI().createPaywallView` + `view.present()`) or only `purchases_flutter` (custom flow → `Adapty().getPaywall` + `Adapty().makePurchase`). Produces migrated source files in the matching language only (Swift OR Dart, never both), a path-aware RevenueCat → Adapty mapping table, and a mandatory Adapty Dashboard setup checklist. CASE 2 — Equivalent / Q&A lookup. Triggered by phrasings like "what's the Adapty equivalent of X", "how do I do X in Adapty (we had it in RevenueCat)", "Adapty version of `Purchases.shared.logIn`" / "Adapty version of `Purchases.logIn` for Flutter", "RevenueCat → Adapty for this snippet", or any standalone RevenueCat code fragment with a request to convert or explain it. Trigger this skill even for one-line lookup questions — the same canonical mapping is the source of truth for both modes. Do NOT use for migrations from other SDKs (Qonversion, Glassfy, Purchasely, native StoreKit) or for Android (native Kotlin/Java), React Native, Unity, or any non-iOS / non-Flutter platform — those are out of scope.
---

# Adapty Migration Skill (RevenueCat → Adapty)

This skill operates in **two modes** AND across **two platforms** (iOS Swift, Flutter/Dart). Before doing anything else, the skill must determine BOTH (a) which mode applies and (b) which platform applies, and then load ONLY the reference files for that platform.

> **🚨 Hard isolation rule — read first.** iOS and Flutter migrations are completely separate paths with separate reference files. The skill MUST NOT mix them. Once Step 0 classifies the project as `ios` or `flutter`, the skill reads ONLY files under `references/<platform>/...` plus the shared files (`platform-detection.md`, `dashboard-checklist.md`, `output-format.md`). Reading the wrong-platform reference files, or emitting wrong-platform code (e.g., Swift code into a Flutter project, or `pubspec.yaml` changes for an iOS project), is the dominant failure mode this skill exists to prevent.

---

## Step 0 — Platform detection (MANDATORY, runs before everything else)

Before reading any reference file, deciding any mode, or producing any output, the skill MUST classify the project's platform. The full detection algorithm and decision rules live in `references/platform-detection.md` — read that file now (it is short).

The output of Step 0 is exactly ONE of these classifications:

- **`ios`** — iOS Swift project using RevenueCat. Proceed with the iOS path; ONLY read files under `references/ios/`.
- **`flutter`** — Flutter project using `purchases_flutter` (and optionally `purchases_ui_flutter`). Proceed with the Flutter path; ONLY read files under `references/flutter/`.
- **`mixed`** — both an iOS-native RevenueCat surface AND a Flutter `purchases_flutter` surface are present in the same repo (rare; e.g., a Flutter app whose iOS module contains additional native RevenueCat code). Ask the user once which one to migrate. Default offer: migrate the Flutter side only and leave any non-Flutter native RevenueCat code in place with a Manual review note.
- **`other` / `refuse`** — Android-native (Kotlin/Java), React Native, Unity, KMP, web, or any other platform. Politely refuse with a one-line explanation and stop. Do NOT attempt a partial migration.

**After Step 0, the skill is committed to exactly one platform path for the remainder of the conversation.** Switching paths mid-migration is forbidden; if new information later reveals the wrong path was chosen, restart Step 0 from scratch, do not silently merge.

### Hard isolation rules enforced after Step 0

These rules apply for the rest of the conversation, both in Mode A (Q&A) and Mode B (full migration):

1. **iOS path:** read ONLY `references/ios/*.md` + the shared files (`platform-detection.md`, `dashboard-checklist.md`, `output-format.md`). NEVER read `references/flutter/*.md`. NEVER emit `.dart` files, `pubspec.yaml` changes, or any Flutter API call. The output language for code is Swift.
2. **Flutter path:** read ONLY `references/flutter/*.md` + the shared files (`platform-detection.md`, `dashboard-checklist.md`, `output-format.md`). NEVER read `references/ios/*.md`. NEVER emit `.swift` files, `Podfile` / `Package.swift` / `*.pbxproj` edits (except the one narrow case below), or any Swift-only Adapty API call. The output language for code is Dart.
3. **Flutter narrow exception:** if the Flutter project's `ios/` host folder contains app-owned custom native Swift code that directly calls `Purchases.shared.*` (rare — typically platform channel bridges), the Flutter path may touch those Swift files. In that case, the patterns are still constrained — DO NOT load `references/ios/*.md`; instead, surface the Swift files as a Manual review item and ask the user whether they want the iOS-path skill run separately for those files. Default: leave them with a `// TODO: ADAPTY MANUAL` and do not migrate.
4. **No "smart" reuse across platforms.** Even when a concept (e.g., "entitlement → access level") appears identical on both platforms, the skill MUST use the platform-specific reference file. The Dart and Swift APIs differ in signature, async style, naming, and error semantics; assuming parity is the exact hallucination this isolation prevents.

---

## Mode selection (after platform is fixed)

Once Step 0 has classified the project as `ios` or `flutter`, identify whether this is a Q&A lookup or a full migration.

### Mode A — Quick equivalent / Q&A lookup

Use this mode when the user is asking a question about a specific RevenueCat → Adapty mapping rather than asking for a full migration. Signals:

- Snippet-level conversion: *"What's the Adapty equivalent of `Purchases.shared.logIn`?"* (iOS), *"How do I do `Purchases.logIn` in adapty_flutter?"* (Flutter), *"Convert this 10-line snippet"*
- No project folder attached; no `.zip`; the user pastes a short fragment or just names a RevenueCat API
- The user explicitly says they're researching / planning, not yet migrating

**What to do in Mode A (still platform-scoped):**

1. Read `references/<platform>/revenuecat-to-adapty-map.md` first (the primary lookup table for that platform)
2. Find the matching entry. The map entry tells you which path file is relevant (if any) and whether the signature is canonical or verify-required.
3. If the entry references `references/<platform>/adapty-reference.md`, `references/<platform>/paywall-builder-path.md`, or `references/<platform>/custom-paywall-path.md` — read only the one that applies (and only from the matching `<platform>/` directory), then answer with the before/after code and cite the file + section.
4. If the entry is marked "verify signature" or the user's snippet hints at a less-common surface (delegate / stream listener, observer mode, attribution sequencing) — consult the current Adapty docs (start from `https://adapty.io/docs/`, search the concept, prefer the Flutter or iOS docs page matching the platform), confirm the Adapty 3.x signature, then answer with the before/after code and cite the source URL plus today's date. If the runtime cannot reach the docs, say so and answer from the reference files, flagging the answer as unverified.
5. If the entry isn't in the map at all — say so explicitly, consult the current Adapty docs (platform-specific page) for the concept, and answer with the URL. Do not guess.

Mode A output is a **chat answer only** — no files written. Keep it short: the snippet, the Adapty replacement, one line on why, the source (file:section or URL+date), and any ordering caveat ("requires `await Adapty.activate()` to have completed first" / "requires the paywall to have `hasViewConfiguration == true`" (iOS) / "requires the paywall to have a non-empty `hasViewConfiguration` (Flutter)" ).

**Platform-ambiguous Q&A:** if the user pastes a one-line snippet without context (e.g., `Purchases.logIn("u1")` — could be Flutter or older iOS API), ask once: *"Is this from an iOS Swift project or a Flutter project? The Adapty equivalent uses a different SDK package on each."* Then proceed with that platform's mapping file only.

### Mode B — Full project migration

Use this mode when the user provides a project folder (multi-file) and asks for the full migration. Signals:

- Project files or a folder/zip is attached
- The user uses phrasings like "migrate this project", "rewrite my whole integration", "replace RevenueCat in this app"

**What to do in Mode B:** follow the 6-step workflow below. Each step branches by platform; both branches share the same scaffolding (the report shape, the dashboard checklist, the verification posture), but the actual reference files loaded and the code generated are platform-specific. Mode B always produces a `MIGRATION_REPORT.md` plus migrated source files (in `.swift` for iOS OR `.dart` for Flutter, never both).

### When in doubt

If the request is ambiguous (e.g., a couple of files pasted but no clear instruction), ask the user once: *"Do you want a full migration with file output, or just the Adapty equivalents for these snippets?"* — then proceed in the matching mode.

---

## Hard rules (NEVER violate — apply on every platform, both modes)

These rules exist because hallucinated APIs, version mismatches, and wrong call ordering are the dominant failure modes in SDK migrations. A migration that compiles but silently breaks paywalls or skips initialization costs real revenue. The rules below apply in spirit to both platforms; the specific API names differ — always use the per-platform reference file for the exact call.

1. **Step 0 platform detection happens before everything else.** No SDK code is emitted, no reference file is loaded, and no migration claim is made until the platform has been classified per `references/platform-detection.md`. Skipping Step 0 is the root cause of cross-platform hallucination.

2. **Use the current Adapty 3.x baseline, never a hard-pinned version.** At the start of every migration (Mode B) and every verify-required Q&A (Mode A), consult the current Adapty installation docs for the matching platform (`https://adapty.io/docs/` → iOS SDK installation page OR Flutter SDK installation page) to confirm the current supported Adapty 3.x SDK version. Use that version in dependency declarations. Never write a cached version number from the reference files into output without re-confirming first. If the runtime cannot reach the docs, leave the version as `<VERIFY_FROM_DOCS>` and surface a Manual review item. Adapty 2.x and 3.x have incompatible APIs — never mix.

3. **Await every SDK lifecycle call, in strict order.** The Adapty 3.x API is async (Swift: async/throws; Dart: `Future`) and the SDK has documented ordering requirements:
   - `Adapty.activate(...)` (iOS) / `Adapty().activate(configuration: ...)` (Flutter) MUST complete (`await`-ed) before any other Adapty call. Synchronous-looking activate without `await` is wrong on either platform.
   - `Adapty.identify(...)` (iOS) / `Adapty().identify(...)` (Flutter) MUST complete before any paywall, profile, purchase, or restore call when the project uses authenticated users.
   - Never fire paywall / profile / purchase / restore calls concurrently with — or before — activate/identify. Sequence them with `await`.
   - **Flutter-specific errors to surface when ordering fails:** `#2002 notActivated`, `#3006 profileWasChanged` (see `references/flutter/adapty-reference.md`).

4. **Detect the paywall path before migrating; do not assume Paywall Builder.** Both platforms have two flavors that migrate to incompatible Adapty patterns:
   - **iOS — Paywall Builder path:** source uses `import RevenueCatUI`, `PaywallView`, `paywallFooter`, `presentPaywallIfNeeded`, or any RevenueCatUI surface. Migrates to `AdaptyUI` + the SwiftUI `.paywall()` modifier. Read `references/ios/paywall-builder-path.md`.
   - **iOS — Custom paywall path:** source uses its own UIKit/SwiftUI views combined with direct `Purchases.shared.purchase(package:)` / `restorePurchases` calls. Migrates to `Adapty.getPaywall` + `Adapty.getPaywallProducts` + `Adapty.makePurchase` / `restorePurchases`. Read `references/ios/custom-paywall-path.md`.
   - **Flutter — Paywall Builder path:** source `pubspec.yaml` includes `purchases_ui_flutter` and/or code uses `RevenueCatUI.presentPaywall(...)`, `RevenueCatUI.presentPaywallIfNeeded(...)`, `PaywallView(...)`. Migrates to the Adapty Flutter paywall-builder flow: `AdaptyConfiguration(...)..withActivateUI(true)` → `Adapty().getPaywall(placementId:)` → `paywall.hasViewConfiguration` guard → `AdaptyUI().createPaywallView(paywall:)` → `view.present()`, plus `AdaptyUI().setPaywallsEventsObserver(...)` for action handling. Read `references/flutter/paywall-builder-path.md`.
   - **Flutter — Custom paywall path:** source only depends on `purchases_flutter` and uses its own Flutter widgets + `Purchases.purchasePackage(...)` / `Purchases.purchase(PurchaseParams(...))`. Migrates to `Adapty().getPaywall(placementId:)` + product list from the paywall + `Adapty().makePurchase(product:)` + `Adapty().restorePurchases()` with the project's existing widgets preserved. Read `references/flutter/custom-paywall-path.md`.
   - A project may use BOTH paths on the same platform in different screens — handle them per-screen, not globally.

5. **AdaptyUI is conditional, not a universal replacement for RevenueCatUI.**
   - **iOS:** only add `import AdaptyUI` and the `AdaptyUI.activate()` call when the migration is genuinely going down the Paywall Builder path. Custom paywall migrations stay on `import Adapty` alone.
   - **Flutter:** only set `..withActivateUI(true)` on the `AdaptyConfiguration` and use `AdaptyUI()` APIs when the migration is genuinely going down the Paywall Builder path. Custom paywall migrations omit `withActivateUI(true)` and never call `AdaptyUI()`.
   - Adding AdaptyUI to a custom-UI project is a bug, not a safe default.

6. **Validate `hasViewConfiguration` in every Paywall Builder migration, and flag the design rebuild.** Adapty paywalls only render via the visual builder when the dashboard's "Show on device" toggle is enabled and a Paywall Builder design exists. The migrated code MUST guard against `paywall.hasViewConfiguration` being false/empty with a graceful fallback (log + skip presentation), and the Dashboard checklist MUST explicitly call out enabling "Show on device" for every placement. Additionally, **every Paywall Builder migration (iOS OR Flutter) MUST surface a Manual review entry stating that RevenueCat paywall designs do NOT transfer to Adapty and must be rebuilt in Adapty's visual builder.** The skill migrates code, not paywall design — never imply otherwise in the report.

7. **The reference is canonical, but version-current.** Patterns in `references/ios/*.md` and `references/flutter/*.md` are the source of truth for *shape* (which APIs to call, in what order, with what arguments). Always verify *signatures* (exact parameter names, async style, return types) against the current Adapty docs (`https://adapty.io/docs/`) for the matching platform at migration time — the references may be out of date relative to the latest SDK minor version. If the runtime cannot reach the docs, flag every emitted signature in Manual review as unverified.

8. **Unknown features must be verified, not guessed.** If RevenueCat code uses a feature NOT covered in the platform's references (examples: observer mode, custom attributes, server-side webhooks, delegate callbacks / event streams, Android subscription update params on Flutter), the skill MUST consult the current Adapty docs for the matching platform and confirm the exact 3.x signature before writing any migration code. If the docs do not give a clear 3.x answer (or the docs are unreachable), leave a `// TODO: ADAPTY MANUAL — <description>` comment in the code and surface it in the "Manual review" section. Never hallucinate a method name or signature.

9. **Output language is English** — explanations, comments, mapping tables, checklists, code. Everything.

10. **The Dashboard checklist is mandatory and complete.** Even if the user only asks for code, ALWAYS produce the Dashboard checklist from `references/dashboard-checklist.md` (the dashboard itself is platform-neutral — same Adapty Dashboard regardless of SDK). On Flutter migrations, add Google Play–specific items (Google Play service account JSON, RTDN — real-time developer notifications, Play Billing API access) in addition to the App Store items, and skip any iOS-only items if the project is Flutter-Android-only. Omitting checklist items silently breaks the migration at runtime.

11. **RevenueCat → Adapty name mapping requires manual review, not silent rename.** Entitlement names map reasonably cleanly to access levels (both are server-side capability flags). But RevenueCat **offerings** and Adapty **placements** are related-but-not-equivalent concepts: an offering bundles packages, while a placement is an "ad slot" with one paywall attached. The skill carries names over verbatim where it can but ALWAYS flags offering↔placement mappings in Manual review with a note like "Confirm the semantic match — this is a rename, not a guaranteed equivalence." Do not assume 1:1 correctness.

12. **No cross-platform contamination.** This bears repeating because it is THE failure mode this skill exists to prevent:
    - Flutter migrations MUST NOT cite, copy from, or call APIs documented in `references/ios/*.md` — the Swift API surface differs (e.g., iOS uses `Adapty.activate("KEY")`, Flutter uses `Adapty().activate(configuration: AdaptyConfiguration(apiKey: 'KEY'))`).
    - iOS migrations MUST NOT cite, copy from, or call APIs documented in `references/flutter/*.md` — the Dart API surface differs.
    - If a Mode A user pastes a Swift snippet but is on a Flutter project (or vice versa), say so explicitly: "This is a Swift snippet but your project is Flutter — the Flutter equivalent uses different APIs. Here is the Flutter version." Don't silently translate.

## Mode B workflow (full project migration)

The 6 steps below apply only to Mode B. For Mode A (Q&A), follow the much shorter steps in the Mode selection section above and stop. Every step is split into common scaffolding + per-platform specifics; always read from the matching `<platform>/` directory.

### Step 1: Detect platform, then read platform-specific references and confirm current SDK version

1. **Run Step 0** per `references/platform-detection.md`. Classify as `ios` / `flutter` / `mixed` / `other`. If `other`, refuse and stop. If `mixed`, ask the user once which to migrate (default: Flutter side).
2. Consult the current Adapty SDK installation docs for the matched platform:
   - iOS → `https://adapty.io/docs/` → iOS SDK installation page
   - Flutter → `https://adapty.io/docs/` → Flutter SDK installation page (`sdk-installation-flutter`)
   Note the current supported 3.x SDK version. If the runtime has no web access, record this fact and proceed with `<VERIFY_FROM_DOCS>` as the version placeholder; surface a Manual review item.
3. Read the platform's reference files (and ONLY those):
   - **iOS path:** `references/ios/adapty-reference.md`, `references/ios/revenuecat-to-adapty-map.md`
   - **Flutter path:** `references/flutter/adapty-reference.md`, `references/flutter/revenuecat-to-adapty-map.md`
4. Read the shared files: `references/dashboard-checklist.md`, `references/output-format.md`.
5. Defer reading the platform's `paywall-builder-path.md` / `custom-paywall-path.md` until Step 2 determines which path applies.

### Step 2: Inventory RevenueCat usage AND detect paywall path (platform-specific)

The user provides a project folder. List its contents recursively and scan based on the platform.

**iOS path — for each `.swift` file scan for:**

- `import RevenueCat`, `import RevenueCatUI`
- `Purchases.configure(...)` — initialization (closure/async/builder form). Watch for `purchasesAreCompletedBy: .myApp` → observer mode (see `references/ios/revenuecat-to-adapty-map.md` §14c).
- `Purchases.shared.*` — any method call
- `Purchases.logLevel`, `Purchases.canMakePayments()`
- Types: `CustomerInfo`, `EntitlementInfo`, `Offering`, `Offerings`, `Package`, `StoreProduct`, `StoreTransaction`, `PurchasesDelegate`
- `PaywallView`, `presentPaywallIfNeeded`, `paywallFooter` — RevenueCatUI APIs
- `attribution.set*` — attribution calls (most map to `Adapty.setIntegrationIdentifier(key:value:)` — see `references/ios/revenuecat-to-adapty-map.md` §12)
- Login/logout: `Purchases.shared.logIn`, `Purchases.shared.logOut`
- **Migration blockers and partial-coverage surfaces** as enumerated in `references/ios/revenuecat-to-adapty-map.md` (products-by-ID §14b, observer mode §14c, virtual currencies §14d, unsupported attribution providers §12, web checkout / redeem, customer center, offline entitlements, etc.)

Also scan project-level dependency files: `Podfile`, `Package.swift`, `*.xcodeproj/project.pbxproj`, `Cartfile`.

**Flutter path — scan for:**

- `pubspec.yaml` entries: `purchases_flutter:` (mandatory signal), `purchases_ui_flutter:` (Paywall Builder signal)
- Dart imports: `import 'package:purchases_flutter/purchases_flutter.dart';`, `import 'package:purchases_ui_flutter/purchases_ui_flutter.dart';`
- `Purchases.configure(...)` / `Purchases.setup(...)` — initialization
- `Purchases.logIn(...)`, `Purchases.logOut()`
- `Purchases.getOfferings()`, `Purchases.getCurrentOfferingForPlacement(...)`
- `Purchases.getCustomerInfo()`, `Purchases.addCustomerInfoUpdateListener(...)`
- `Purchases.restorePurchases()`
- `Purchases.purchasePackage(...)`, `Purchases.purchase(PurchaseParams(...))` (including Google Play upgrade/replacement params)
- `RevenueCatUI.presentPaywall(...)`, `RevenueCatUI.presentPaywallIfNeeded(...)`, `PaywallView(...)` — RevenueCatUI Flutter APIs
- App-owned platform-channel bridges in `ios/` or `android/` that call native RevenueCat directly (rare — flag as Manual review per the §0 narrow exception above; do NOT auto-migrate)

Also scan for Android subscription change/upgrade logic in purchase calls (Google `subscriptionReplacementMode` / `oldProductIdentifier` / proration) — these must carry over via Adapty's `subscriptionUpdateParams`. See `references/flutter/revenuecat-to-adapty-map.md`.

**Paywall path detection (mandatory, per-platform):**

- **iOS:** if the project imports `RevenueCatUI` OR uses `PaywallView` / `paywallFooter` / `presentPaywallIfNeeded` → that screen/file is on the Paywall Builder path → read `references/ios/paywall-builder-path.md`. Otherwise → custom paywall path → read `references/ios/custom-paywall-path.md`.
- **Flutter:** if `pubspec.yaml` includes `purchases_ui_flutter` OR any Dart file uses `RevenueCatUI.presentPaywall` / `presentPaywallIfNeeded` / `PaywallView` → Paywall Builder path → read `references/flutter/paywall-builder-path.md`. Otherwise → custom paywall path → read `references/flutter/custom-paywall-path.md`.
- Mixed projects exist on either platform — annotate each paywall-touching file with its path independently.
- If ambiguous (e.g., both surfaces present in different screens), ask the user once: "I see both Paywall Builder usage and custom purchase calls — should I migrate everything to the Adapty Paywall Builder flow, or keep the custom paywall flows custom (recommended if you have non-standard paywall UI)?"

**Auth detection (also mandatory, both platforms):**

- If the project calls `Purchases.shared.logIn` / `Purchases.shared.logOut` (iOS) OR `Purchases.logIn` / `Purchases.logOut` (Flutter) → the migration MUST insert the matching Adapty identify/logout calls at matching call sites and ensure all profile/paywall/purchase calls happen *after* identify completes.
- If the project never calls logIn → anonymous-only; skip identify entirely.

Build a categorized inventory: initialization, entitlement/access-level check, offerings/paywall fetch (per path), purchase (per path), restore (per path), identification, attribution, delegate/stream listener, dependencies, other.

### Step 3: Plan each replacement, path-aware AND platform-aware

For each item in the inventory, look it up in the platform's `references/<platform>/revenuecat-to-adapty-map.md` and classify:

- **Mapped, single path** → use the path-specific pattern verbatim (substituting the user's variable, class/widget, and access-level names)
- **Mapped, both paths** (e.g., entitlement / access-level check) → both paths use the same pattern; just apply it
- **Mapped but flagged "verify signature"** → consult the current Adapty docs for the relevant concept (start from `https://adapty.io/docs/`, prefer the page matching the platform), confirm the exact 3.x parameter names and return types, then write the migration. Note the verified URL + date in Manual review. If docs are unreachable, write the pattern from the reference file and flag the signature as unverified.
- **Not in the map** → search the current Adapty docs for the concept on the matching platform's pages. If still unclear (or docs unreachable), leave a TODO comment (`// TODO: ADAPTY MANUAL — ...` in Swift; `// TODO: ADAPTY MANUAL — ...` in Dart) and add to Manual review.

For offering → placement specifically: carry the name over BUT add to Manual review: "Offering `<name>` carried over as placement `<name>`. RevenueCat offerings and Adapty placements are related but not 1:1 equivalent; confirm the semantic match before relying on it."

Never write Adapty code from memory for anything not in the platform's reference. Memory is the failure mode this skill exists to prevent.

### Step 4: Write the migrated source files (platform-specific)

Write each modified source file to a `migrated/` directory under the runtime's normal output location, preserving the original project's relative paths. (In a sandboxed runtime with a dedicated outputs path, use that; otherwise use a `migrated/` directory under the user's working directory.)

**iOS path (Swift output only — see `references/ios/` for full details):**

Always:
- Replace `import RevenueCat` with `import Adapty`
- Replace `Purchases.configure(...)` with the awaited activation sequence from `references/ios/adapty-reference.md`
- Create or update `AdaptyManager.swift` matching `references/ios/adapty-reference.md`
- Convert closure-based RevenueCat callbacks to `async/await` Swift Concurrency throughout
- Insert `try await Adapty.identify(...)` and ensure subsequent calls await identify completion (when auth is in play)
- Replace `Purchases.shared.getCustomerInfo` with `try await Adapty.getProfile()`
- Preserve all comments, business logic, and unrelated code unchanged
- Update `Podfile` / `Package.swift` / `.pbxproj` to remove RevenueCat and add Adapty using the version confirmed in Step 1

Paywall Builder path only: add `import AdaptyUI`, the `try await AdaptyUI.activate()` call, the `.paywall()` modifier, and the `hasViewConfiguration` guard per `references/ios/paywall-builder-path.md`.

Custom paywall path only: do NOT add `import AdaptyUI`; use `Adapty.getPaywall` + `Adapty.getPaywallProducts` + `Adapty.makePurchase` + `Adapty.restorePurchases` per `references/ios/custom-paywall-path.md`.

**Flutter path (Dart output only — see `references/flutter/` for full details):**

Always:
- Update `pubspec.yaml`: remove `purchases_flutter:` (and `purchases_ui_flutter:` if present), add `adapty_flutter: ^<VERSION>` (using the version confirmed in Step 1)
- Replace Dart imports: `import 'package:purchases_flutter/purchases_flutter.dart';` → `import 'package:adapty_flutter/adapty_flutter.dart';`
- Replace `Purchases.configure(PurchasesConfiguration(...))` with `await Adapty().activate(configuration: AdaptyConfiguration(apiKey: 'YOUR_PUBLIC_SDK_KEY'))` (Paywall Builder path adds `..withActivateUI(true)`; if a `customerUserId` was passed at configure time, add `..withCustomerUserId(...)`)
- Insert `await Adapty().identify(customerUserId)` and `await Adapty().logout()` at matching call sites; ensure subsequent calls await identify completion (when auth is in play)
- Replace `Purchases.getCustomerInfo()` with `await Adapty().getProfile()`; replace `Purchases.addCustomerInfoUpdateListener(...)` with `Adapty().didUpdateProfileStream.listen(...)`
- Convert `CustomerInfo.entitlements["pro"]?.isActive` to `profile.accessLevels['pro']?.isActive ?? false`
- Preserve all comments, business logic, and unrelated Dart code unchanged
- Do NOT modify the iOS/Android host folders unless the narrow Swift-bridge exception applies (see Hard isolation rules §3)

Paywall Builder path only (Flutter): keep `..withActivateUI(true)`, use `AdaptyUI().createPaywallView(paywall: ...)` + `view.present()`, register `AdaptyUI().setPaywallsEventsObserver(...)` for `AdaptyUIAction` handling (`CloseAction`, `AndroidSystemBackAction`, `OpenUrlAction`), guard `paywall.hasViewConfiguration` — see `references/flutter/paywall-builder-path.md`.

Custom paywall path only (Flutter): omit `withActivateUI(true)`; load products with `Adapty().getPaywall(placementId:)` then the products from that paywall; buy with `await Adapty().makePurchase(product: product)`, switching on `AdaptyPurchaseResultSuccess` / `AdaptyPurchaseResultPending` / `AdaptyPurchaseResultUserCancelled`; restore with `await Adapty().restorePurchases()`. Keep the project's existing widgets — see `references/flutter/custom-paywall-path.md`. For Android subscription replacement, carry the existing parameters over via `subscriptionUpdateParams`.

Files in the project that contain no RevenueCat references are NOT copied to `migrated/` — only changed files appear there. This keeps the diff readable.

### Step 5: Assemble `MIGRATION_REPORT.md`

Write `MIGRATION_REPORT.md` (alongside the `migrated/` directory from Step 4) following `references/output-format.md`. Five sections:

1. **Summary** — what was migrated, file count, **platform detected (iOS / Flutter)**, paywall path(s) detected, SDK version confirmed from docs (or noted as unverified), next steps in one paragraph
2. **Mapping table** — every RevenueCat API touched in this migration, with three columns: RevenueCat | Adapty | Source (reference file + section, or verified URL + date). All rows MUST come from the matching platform's mapping file.
3. **File-by-file changes** — one short paragraph per modified file, noting which paywall path applied
4. **Adapty Dashboard checklist** — copied from `references/dashboard-checklist.md` and customized with this project's specific access-level names, placement IDs, product IDs, attribution providers, AND store-specific items (App Store Connect items for iOS / Apple-store Flutter apps; Google Play items for Android-store Flutter apps; both if the Flutter app ships on both stores)
5. **Manual review** — every TODO, every verified API the user should double-check, every offering↔placement semantic-match warning, every entitlement↔access-level naming reminder, the API key replacement, the App Store Server Notifications URL switch (iOS / Apple-store Flutter), the Google Play RTDN URL switch (Android-store Flutter), the paywall design rebuild reminder (Paywall Builder migrations only)

### Step 6: Present the deliverables

Return `MIGRATION_REPORT.md` first (highest user value), then every file under `migrated/`. Use whatever file-presentation mechanism the runtime offers. Keep the post-amble short — the report itself explains everything.

## Triggering examples

**Mode A (Q&A / equivalent lookup) — these should trigger and produce a chat answer:**

- *(iOS)* "What's the Adapty equivalent of `Purchases.shared.logIn`?"
- *(iOS)* "How do I check entitlements in Adapty? We use `customerInfo.entitlements['pro'].isActive` in RevenueCat."
- *(Flutter)* "What's the Adapty equivalent of `Purchases.logIn` in Flutter?"
- *(Flutter)* "How do I show a paywall in adapty_flutter? We used `RevenueCatUI.presentPaywall(offering: ...)`."
- *(Flutter)* "Convert this Dart snippet to adapty_flutter:" + snippet
- *(Platform-ambiguous)* "Adapty version of `Purchases.restorePurchases()`?" → ask once which platform, then answer.

**Mode B (full migration) — these should trigger and produce files:**

- "Migrate this iOS app from RevenueCat to Adapty" + Swift project folder
- "Migrate this Flutter app from RevenueCat to Adapty" + Flutter project folder
- "Replace RevenueCat with Adapty in this Swift project" + project folder
- "Replace purchases_flutter with adapty_flutter in this app" + project folder
- "I want to switch from RevenueCat to Adapty" + project folder (Step 0 detects which platform)
- "Adapty integration" on a project that already imports RevenueCat across multiple files

**These should NOT trigger the skill:**

- "Set up Adapty in a new project" (no RevenueCat present — use general Adapty knowledge)
- "Migrate from Qonversion / Glassfy / Purchasely to Adapty" (out of scope)
- "Migrate an Android-native (Kotlin) app from RevenueCat to Adapty" (out of scope — not iOS-Swift, not Flutter)
- "Migrate a React Native app from RevenueCat to Adapty" (out of scope)
- "Migrate a Unity / KMP / web app from RevenueCat to Adapty" (out of scope)
- "How does Adapty compare to RevenueCat?" (comparison question, not a migration or equivalent lookup)

## File layout

```
adapty-migration/
├── SKILL.md (this file)
└── references/
    ├── platform-detection.md         # Step 0: how to classify ios / flutter / mixed / other
    ├── dashboard-checklist.md        # Mandatory Adapty Dashboard setup (shared across platforms)
    ├── output-format.md              # Exact structure of MIGRATION_REPORT.md (shared)
    │
    ├── ios/                          # iOS Swift migration content — read ONLY on iOS path
    │   ├── adapty-reference.md            # Common base: install, activate, identify, AdaptyManager
    │   ├── paywall-builder-path.md        # AdaptyUI + .paywall() modifier + hasViewConfiguration
    │   ├── custom-paywall-path.md         # Adapty.getPaywall + getPaywallProducts + makePurchase
    │   └── revenuecat-to-adapty-map.md    # iOS per-API mapping table, path-aware
    │
    └── flutter/                      # Flutter Dart migration content — read ONLY on Flutter path
        ├── adapty-reference.md            # Common base: pubspec, activate, identify, ordering, error codes
        ├── paywall-builder-path.md        # withActivateUI + createPaywallView + present + observer
        ├── custom-paywall-path.md         # Adapty().getPaywall + makePurchase + restorePurchases
        └── revenuecat-to-adapty-map.md    # Flutter per-API mapping table, path-aware
```
