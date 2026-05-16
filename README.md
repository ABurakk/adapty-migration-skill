# adapty-migration-skill

A Claude skill that migrates the **code-side** of **iOS Swift** and **Flutter (Dart)** apps from **RevenueCat** to **Adapty** — handling both the Paywall Builder (AdaptyUI) wiring and the custom-paywall direct-API flow, with a mandatory Adapty Dashboard checklist generated for every migration.

The skill **always runs platform detection first (Step 0)** and routes the migration down exactly one path — iOS Swift OR Flutter Dart, never both. Each platform has its own isolated set of reference files; the skill never mixes Swift and Dart Adapty patterns in the same migration, which prevents the most common cross-platform hallucination failures.

> **Note on Paywall Builder migrations (both platforms):** the skill rewrites your paywall *code* (swaps RevenueCatUI's `PaywallView` / `presentPaywallIfNeeded` for AdaptyUI's `.paywall()` modifier on iOS, or `AdaptyUI().createPaywallView` + `view.present()` on Flutter, adds the `hasViewConfiguration` guard, etc.) — but you will need to **rebuild your paywall design from scratch in Adapty's visual builder. RevenueCat paywall layouts do not transfer.** Use a template as a starting point, or recreate your existing design.

> Migrates the common RevenueCat integration patterns to Adapty on iOS Swift AND Flutter: path detection, SDK swap, AdaptyUI wiring, a per-file change report, and a dashboard checklist customized for the project's stores (App Store Connect on iOS / Apple-store Flutter, Google Play on Android-store Flutter). Conservative by design — flags products-by-ID, observer mode, Virtual Currencies, unsupported attribution providers, and Android subscription-replacement params as migration blockers / manual-review items rather than guessing.

---

## What it does

- **Detects the platform first.** Step 0 classifies the project as `ios` / `flutter` / `mixed` / `other` based on signals like `pubspec.yaml` + `purchases_flutter`, `.xcodeproj` + `import RevenueCat`, etc. The skill commits to one platform for the entire run and only reads that platform's reference files — no cross-platform code mixing.
- **Detects the paywall path per file (within the chosen platform).**
  - **iOS:** files using `RevenueCatUI` (`PaywallView`, `paywallFooter`, `presentPaywallIfNeeded`) migrate to AdaptyUI + the `.paywall()` modifier. Files using custom UI + direct `Purchases.shared.purchase(package:)` migrate to `Adapty.getPaywall` + `Adapty.getPaywallProducts` + `Adapty.makePurchase`.
  - **Flutter:** projects with `purchases_ui_flutter` migrate to the Adapty Flutter paywall builder (`..withActivateUI(true)` + `AdaptyUI().createPaywallView` + `view.present()` + `AdaptyUIPaywallsEventsObserver`). Projects with only `purchases_flutter` migrate to `Adapty().getPaywall` + product list + `Adapty().makePurchase` + `Adapty().restorePurchases`.
  - Mixed projects are handled per-file (within the same platform).
- **Enforces Adapty's call-ordering invariants on both platforms.** `Adapty.activate(...)` (iOS) / `Adapty().activate(configuration: ...)` (Flutter) must complete before any other SDK call; `Adapty.identify(...)` / `Adapty().identify(...)` must complete before paywall/profile/purchase/restore when auth is involved. The Flutter path additionally surfaces the SDK's `#2002 notActivated` and `#3006 profileWasChanged` error codes as the ordering failure signals.
- **Verifies signatures against current docs.** No version is hard-pinned. Every migration consults the current Adapty SDK installation docs for the matched platform (iOS or Flutter) before emitting dependency code. Reference patterns describe shape; live docs confirm exact signatures. If the runtime cannot reach the docs, the skill emits a `<VERIFY_FROM_DOCS>` placeholder and surfaces a Manual review item.
- **Validates `hasViewConfiguration` for Paywall Builder migrations on both platforms.** The generated code includes the guard required for the dashboard's "Show on device" toggle. Without it, paywalls render nothing at runtime — a silent failure mode this skill prevents.
- **Generates a mandatory dashboard checklist.** Every migration produces an Adapty Dashboard checklist covering access levels, placements, paywalls, products, "Show on device", plus per-store sections: **App Store Server Notifications (production + sandbox URLs)** for iOS / Apple-store Flutter, and **Google Play service account + Real-Time Developer Notifications (RTDN)** for Android-store Flutter; attribution integrations; webhooks/raw events forwarding.
- **Flags offering↔placement mappings for manual review.** RevenueCat offerings and Adapty placements are related but not 1:1; the skill carries names over and always asks the user to confirm semantic equivalence.
- **Carries over Android subscription-replacement params on Flutter.** Google Play upgrade/downgrade logic that used RevenueCat's `PurchaseParams.googleProductChangeInfo` is migrated to Adapty's `subscriptionUpdateParams` on `makePurchase` — not silently dropped.

## What it does NOT do

- **It does not migrate paywall designs.** Layouts, copy, images, fonts, and colors from your RevenueCat paywalls do not transfer. On the Paywall Builder path, you rebuild each paywall in Adapty Dashboard's visual builder. The skill migrates the code that *renders* the paywall; the visual design is manual work in either dashboard.
- It does not mix platforms. A Flutter project gets a Dart-only migration; an iOS project gets a Swift-only migration. The skill refuses to emit Swift into a Flutter project (or vice versa), even for shared concepts.
- It does not access your Adapty Dashboard. You configure it manually using the generated checklist.
- It does not paste your API key. The migrated code uses `"YOUR_ADAPTY_PUBLIC_KEY"` / `'YOUR_PUBLIC_SDK_KEY'` as a placeholder; you replace it.
- It does not run sandbox tests. Sandbox / test purchase verification (App Store sandbox tester or Google Play license tester) is on you, before shipping.
- It does not migrate from other SDKs (Qonversion, Glassfy, Purchasely, native StoreKit) or other platforms (Android-native Kotlin/Java, React Native, Unity, KMP, web). **iOS Swift OR Flutter Dart, RevenueCat → Adapty only.**

## Two modes

**Mode A — Q&A lookup.** Ask Claude "what's the Adapty equivalent of `Purchases.shared.logIn`?" (iOS) or "what's the Adapty Flutter equivalent of `Purchases.logIn`?" (Flutter) and get a chat answer with the canonical replacement, the source reference file, and any ordering caveat. No files written. If the snippet is platform-ambiguous, Claude asks once which platform before answering.

**Mode B — Full project migration.** Hand Claude a project folder and say "migrate this to Adapty." After Step 0 detects the platform, Claude produces:
- Migrated source files in the matching language only (Swift OR Dart, never both), preserving the original folder structure
- Updated dependency files (`Package.swift` / `Podfile` for iOS; `pubspec.yaml` for Flutter)
- A `MIGRATION_REPORT.md` with:
  1. Summary (platform detected, file count, paywall paths detected, SDK version confirmed)
  2. API mapping table (per-API, with reference citations or fetched URLs + dates)
  3. File-by-file changes
  4. The Adapty Dashboard checklist, customized with your project's access-level names, placement IDs, product IDs, and per-store items (App Store Connect / Google Play, as applicable)
  5. Manual review items (API key replacement, ASSN URL switch (iOS) / Google Play RTDN switch (Android Flutter), "Show on device" toggle, offering↔placement confirmations, paywall design rebuild reminder for Paywall Builder migrations, etc.)

## Install

### Option 1 — Download the latest release (recommended)

Download `adapty-migration-skill.zip` from the [latest release](../../releases/latest), then upload it to Claude:

- **Claude.ai web/desktop:** Settings → Capabilities → Skills → Upload skill → select the zip
- **Claude Code:** unzip into `~/.claude/skills/adapty-migration-skill/` and restart your terminal

### Option 2 — Clone and zip yourself

```bash
git clone https://github.com/ABurakk/adapty-migration-skill.git
cd adapty-migration-skill/skill
zip -r ../adapty-migration-skill.zip .
cd ..
```

This matches the layout produced by the release workflow: the zip root contains `SKILL.md` and the `references/` directory directly — not a wrapping `skill/` folder. Then upload `adapty-migration-skill.zip` the same way.

## How to use

Once installed, just talk to Claude normally about your RevenueCat-to-Adapty migration:

```
> I want to migrate this iOS project from RevenueCat to Adapty.
  [attach your project folder or zip]
```

```
> Migrate this Flutter app from purchases_flutter to adapty_flutter.
  [attach your Flutter project folder]
```

Or for a quick lookup:

```
> What's the Adapty equivalent of Purchases.shared.getOfferings?
```

```
> Adapty Flutter equivalent of Purchases.purchasePackage(...)?
```

Claude detects when the skill applies, runs platform detection (Step 0), and loads the matching reference files automatically — no special invocation needed.

## Skill structure

```
skill/
├── SKILL.md                            # Workflow + hard rules + Step 0 routing
└── references/
    ├── platform-detection.md           # Step 0: classify ios / flutter / mixed / other
    ├── dashboard-checklist.md          # Adapty Dashboard setup (shared; iOS + Google Play sections)
    ├── output-format.md                # MIGRATION_REPORT.md template (shared)
    │
    ├── ios/                            # iOS Swift content — loaded ONLY on iOS path
    │   ├── adapty-reference.md             # Common patterns: install, activate, identify, AdaptyManager
    │   ├── paywall-builder-path.md         # AdaptyUI + .paywall() + hasViewConfiguration
    │   ├── custom-paywall-path.md          # Adapty.getPaywall + getPaywallProducts + makePurchase
    │   └── revenuecat-to-adapty-map.md     # iOS per-API mapping, path-aware
    │
    └── flutter/                        # Flutter Dart content — loaded ONLY on Flutter path
        ├── adapty-reference.md             # Common patterns: pubspec, activate, identify, ordering, error codes
        ├── paywall-builder-path.md         # withActivateUI + createPaywallView + present + observer
        ├── custom-paywall-path.md          # Adapty().getPaywall + makePurchase + restorePurchases
        └── revenuecat-to-adapty-map.md     # Flutter per-API mapping, path-aware
```

## License

[MIT](./LICENSE)
