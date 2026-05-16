# Adapty Flutter Reference — Common Base Patterns

This file contains the patterns shared by **every** RevenueCat → Adapty Flutter migration regardless of paywall flavor: dependency switch, activation, identification, the `AdaptyManager` equivalent, the strict call-ordering invariants, and the Flutter-specific error codes the migration must surface. Path-specific patterns live in two separate files:

- `paywall-builder-path.md` (sibling file) — for projects that use `purchases_ui_flutter` (`RevenueCatUI.presentPaywall`, `presentPaywallIfNeeded`, `PaywallView`) → migrate to the Adapty Flutter Paywall Builder flow (`..withActivateUI(true)` + `AdaptyUI().createPaywallView` + `view.present()` + observer)
- `custom-paywall-path.md` (sibling file) — for projects with custom Flutter widgets and direct `Purchases.purchasePackage(...)` / `Purchases.purchase(PurchaseParams(...))` calls → migrate to `Adapty().getPaywall` + product list + `Adapty().makePurchase` + `Adapty().restorePurchases`

> **Isolation reminder.** This file is read ONLY when Step 0 (see `../platform-detection.md`) classified the project as `flutter`. iOS Swift content is in `../ios/` and MUST NOT be cross-cited here, even when concepts overlap. The Swift and Dart Adapty APIs differ in signature, async style, and naming — do not silently translate one to the other.

**Target platform:** Flutter (Dart 3.x), iOS and/or Android targets. The `adapty_flutter` package supports both stores from a single Dart codebase.
**Language:** Dart, `async` / `await` throughout.
**SDK version:** the current supported `adapty_flutter` version, confirmed at migration time against the live Adapty Flutter SDK installation docs (`https://adapty.io/docs/sdk-installation-flutter` or the current equivalent). Do not bake a version number into output without re-confirming. If the runtime cannot reach the docs, emit `^<VERIFY_FROM_DOCS>` as the version placeholder and surface a Manual review item.

---

## 1. Installation — `pubspec.yaml`

**Before reading this section, verify the current Adapty Flutter version** from `https://adapty.io/docs/sdk-installation-flutter`. The version below is a placeholder; substitute the value confirmed from docs at migration time.

### Remove

```yaml
dependencies:
  purchases_flutter: ^<old-version>          # REMOVE
  purchases_ui_flutter: ^<old-version>       # REMOVE (only if present)
```

### Add

```yaml
dependencies:
  adapty_flutter: ^<VERSION>                 # version from current docs
```

After updating `pubspec.yaml`, the user must run `flutter pub get` to fetch the new package. The skill notes this in Manual review; it does not run the command itself.

**Native host-folder housekeeping:** the `adapty_flutter` plugin pulls in the iOS and Android Adapty SDKs transitively. The migration does NOT manually edit `ios/Podfile`, `android/build.gradle`, or `android/app/build.gradle` for Adapty's native deps — the Flutter plugin handles those via its own platform-side build configuration. The only exception is if the project's existing `Podfile` or Gradle file contains a *manual* RevenueCat dependency entry (very rare in Flutter — usually `purchases_flutter` brings RC in transitively); in that case, remove the manual entry and add a Manual review note.

## 2. Imports

Replace `purchases_flutter` (and `purchases_ui_flutter`, if present) imports throughout the Dart codebase:

### Before

```dart
import 'package:purchases_flutter/purchases_flutter.dart';
import 'package:purchases_ui_flutter/purchases_ui_flutter.dart'; // only on Paywall Builder path
```

### After

```dart
import 'package:adapty_flutter/adapty_flutter.dart';
```

`adapty_flutter` exposes both the core Adapty surface (`Adapty()`, `AdaptyConfiguration`, `AdaptyProfile`, `AdaptyPaywall`, `AdaptyPaywallProduct`, `AdaptyPurchaseResult*`) and — when `..withActivateUI(true)` is set — the `AdaptyUI()` surface used on the Paywall Builder path. There is no separate UI package import on Flutter (unlike iOS, where `import AdaptyUI` is distinct from `import Adapty`).

## 3. Activation — must be awaited before anything else

`Adapty().activate(...)` is async. The Adapty SDK enforces that **no other Adapty call may execute before activate completes**. Calls that race ahead fail with error **`#2002 notActivated`** (see §6).

### Canonical pattern in `main()`

```dart
import 'package:flutter/material.dart';
import 'package:adapty_flutter/adapty_flutter.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  try {
    final configuration = AdaptyConfiguration(apiKey: 'YOUR_PUBLIC_SDK_KEY');
    // Paywall Builder migrations add:
    // ..withActivateUI(true)
    // If the user is already authenticated at app launch, also chain:
    // ..withCustomerUserId('USER_ID_FROM_YOUR_AUTH_SYSTEM')
    await Adapty().activate(configuration: configuration);
  } on AdaptyError catch (e) {
    debugPrint('❌ Adapty activation failed: code=${e.code}, message=${e.message}');
  } catch (e) {
    debugPrint('❌ Adapty activation failed: $e');
  }

  runApp(const MyApp());
}
```

### Rules

- `WidgetsFlutterBinding.ensureInitialized()` MUST run before any Adapty call when activation is in `main()` before `runApp`.
- `await` the activation. A bare `Adapty().activate(...)` without `await` will race subsequent calls and trigger `#2002 notActivated`.
- The API-key literal stays a placeholder. **Never copy the RevenueCat key value over** — RevenueCat keys and Adapty keys are different formats; reusing one silently breaks the integration.
- `..withCustomerUserId(...)` is the configuration-time identification shortcut. Use it ONLY when the customer user ID is known at launch (e.g., the user is already signed in with a stored token). If the ID becomes available later (login flow), use `Adapty().identify(...)` instead — see §4.
- `..withActivateUI(true)` is required ONLY on Paywall Builder migrations. Custom-paywall migrations omit it. Adding it unnecessarily activates AdaptyUI machinery that won't be used; not a runtime crash, but pointless overhead and a signal of incorrect path detection.
- Verbose logging or custom configuration: consult `https://adapty.io/docs/sdk-installation-flutter` for the current `AdaptyConfiguration` builder methods (the API is fluent: chain `..withLogLevel(...)` etc.). Verify each option against current docs before emitting.

## 4. Identification — required when the project uses authenticated users

If the source `purchases_flutter` project calls `Purchases.logIn(...)` anywhere, the Adapty migration MUST insert `Adapty().identify(...)` at every matching call site. The same ordering rule applies: **no profile, paywall, purchase, or restore call may execute before identify completes** when the user is logged in.

### Canonical login pattern

```dart
Future<void> handleUserLogin(String customerUserId) async {
  try {
    await Adapty().identify(customerUserId);
    // Only after identify resolves, refresh access state:
    await AdaptyManager.instance.refresh();
  } on AdaptyError catch (e) {
    debugPrint('❌ Adapty.identify failed: code=${e.code}, message=${e.message}');
  }
}
```

### Canonical logout pattern

```dart
Future<void> handleUserLogout() async {
  try {
    await Adapty().logout();
    await AdaptyManager.instance.refresh();
  } on AdaptyError catch (e) {
    debugPrint('❌ Adapty.logout failed: code=${e.code}, message=${e.message}');
  }
}
```

### Important Adapty behavior

- `Adapty().logout()` **creates a new anonymous profile.** Any access-level state for the previous user is cleared on the device — this propagates to UI via `AdaptyManager.instance.refresh()`.
- If identifying after app launch, always `await Adapty().identify(customerUserId)` before any other SDK call. Calls that race ahead can trigger **`#3006 profileWasChanged`** (the profile changed mid-call; see §6).
- If the source project never calls `Purchases.logIn` (anonymous-only), skip this entire section — Adapty's anonymous profile handling takes over automatically.

### Profile semantics differ from RevenueCat

Same caveat as the iOS path applies on Flutter:

- **RevenueCat:** `logIn("A")` → `logIn("B")` creates a new profile for `"B"`. Switching back returns to the original. RC also merges some profile data across these transitions.
- **Adapty:** `identify("B")` on an anonymous profile **links that customer ID to the current profile** rather than creating a fresh one. If `"B"` already exists, the SDK switches to the existing profile. **Adapty does NOT merge profile data across these transitions.**

If the migrating code relies on `logIn("A")` → `logIn("B")` creating distinct profiles, insert `await Adapty().logout()` between them. The skill MUST detect consecutive `logIn` calls without an intervening `logOut` and flag this as a behavior change.

## 5. `AdaptyManager` singleton — common pattern (both paths)

Create or replace any RevenueCat-related manager (`PurchasesManager`, `BillingService`, `SubscriptionService`, etc.) with this class. Port the original manager's public API surface onto it, but keep the internal structure.

```dart
import 'dart:async';
import 'package:adapty_flutter/adapty_flutter.dart';
import 'package:flutter/foundation.dart';

class AdaptyManager extends ChangeNotifier {
  AdaptyManager._();
  static final AdaptyManager instance = AdaptyManager._();

  bool _isPremium = false;
  bool get isPremium => _isPremium;

  bool _isInitialized = false;
  bool get isInitialized => _isInitialized;

  StreamSubscription<AdaptyProfile>? _profileSub;

  /// Call once after `Adapty().activate(...)` resolves in main().
  Future<void> initialize() async {
    _isInitialized = true;

    // React to profile updates from the SDK (background renewals,
    // restores from another device, attribution-driven changes, etc.).
    _profileSub = Adapty().didUpdateProfileStream.listen(_applyProfile);

    await refresh();
  }

  /// Re-fetches the profile and updates access state. Call after
  /// purchase, restore, identify, or logout.
  Future<void> refresh() async {
    try {
      final profile = await Adapty().getProfile();
      _applyProfile(profile);
    } on AdaptyError catch (e) {
      debugPrint('❌ Failed to fetch Adapty profile: code=${e.code}, message=${e.message}');
    }
  }

  void _applyProfile(AdaptyProfile profile) {
    final hasAccess = profile.accessLevels['premium']?.isActive ?? false;
    if (hasAccess != _isPremium) {
      _isPremium = hasAccess;
      notifyListeners();
    }
  }

  @override
  void dispose() {
    _profileSub?.cancel();
    super.dispose();
  }
}
```

### Customization rules

- If the project's RevenueCat entitlement is called something other than `'premium'` (e.g., `'pro'`, `'unlimited_access'`, `'vip'`), replace the literal `'premium'` in the `accessLevels[...]` lookup AND surface a Manual review note: "Create an access level named `<name>` in Adapty Dashboard → Access Levels (case-sensitive)."
- If the project tracks multiple entitlements (e.g., `'basic'` and `'pro'`), expand `isPremium` into separate booleans per entitlement, and update each in `_applyProfile`. Flag in Manual review for user confirmation.
- `ChangeNotifier` + `notifyListeners()` is shown above for projects that consume Adapty state via `Provider` / `ChangeNotifierProvider` / `ListenableBuilder`. If the project uses `Stream` / `Riverpod` / `bloc` / `GetX` instead, port the same shape into the project's existing state-management idiom — do NOT inject `ChangeNotifier` into a codebase that doesn't already use it.
- Hold `AdaptyManager.instance` as a long-lived singleton so the `didUpdateProfileStream` subscription stays alive for the app's lifetime. Disposing it early silently drops background profile updates.

## 6. Strict ordering invariants (apply globally)

These invariants exist because the Adapty SDK enforces them; violating them produces specific error codes (below) that the migrating code must surface, not swallow.

1. **`Adapty().activate(...)` MUST complete before any other Adapty call.** Code that depends on Adapty must check `AdaptyManager.instance.isInitialized` (or otherwise await the activation `Future`) before issuing calls.
2. **`Adapty().identify(...)` MUST complete before paywall / profile / purchase / restore calls when auth is in play.** The migrated login flow awaits identify before any subsequent SDK call.
3. **Paywall fetch is async and may fail.** Treat `Adapty().getPaywall(...)`, the product list from the paywall, `Adapty().makePurchase(...)`, `Adapty().restorePurchases()`, and (on Paywall Builder path) `AdaptyUI().createPaywallView(paywall: ...)` as all-or-nothing — wrap in `try / on AdaptyError catch` blocks and log failures.
4. **Paywall Builder paywalls require `paywall.hasViewConfiguration` to be non-empty / truthy.** See `paywall-builder-path.md` §3. Custom paywalls do not have this requirement.
5. **The access-level check is always `profile.accessLevels['<name>']?.isActive ?? false`.** Any other expression is wrong.
6. **Locale for paywall fetch** is typically the device locale: `locale: 'en'` is a fallback. Prefer reading the current locale from `Localizations.localeOf(context).toLanguageTag()` or the project's existing locale source.

### Adapty Flutter error codes the migration MUST surface

The Flutter SDK reports errors via `AdaptyError` with numeric `code` and string `message`. The migration always catches and logs both. The two most common ordering-failure codes are worth calling out explicitly in inline comments AND in Manual review:

- **`#2002 notActivated`** — a method was called before `Adapty().activate(...)` resolved. Migration cause: a UI screen tried to fetch a paywall while `main()` was still in its activation `await`. Fix: gate the call on `AdaptyManager.instance.isInitialized`, or move the call later in the lifecycle.
- **`#3006 profileWasChanged`** — a method was in flight when the profile changed underneath it (typical cause: `identify`/`logout` raced against a concurrent paywall/purchase call). Fix: serialize identify/logout against other SDK calls; do not fire them concurrently.

Other error codes exist for network / config / store issues — the migrating code's `catch` block should preserve the code and message in its log so the user can match them against current Adapty docs (`https://adapty.io/docs/` → error reference for the Flutter SDK).

## 7. What this file does NOT cover

Paywall presentation and purchase mechanics depend on which path applies — load the matching file:

- Project uses `purchases_ui_flutter` (or `RevenueCatUI` Dart APIs) → `paywall-builder-path.md` (sibling)
- Project uses custom Flutter widgets + direct purchase → `custom-paywall-path.md` (sibling)

Other topics not covered here (consult the current Adapty Flutter docs at `https://adapty.io/docs/` when encountered):

- Attribution integration with MMP-specific sequencing (Adjust, AppsFlyer, Branch, AppMetrica) — see `revenuecat-to-adapty-map.md` §12 for the Flutter path
- Custom user attributes (`Adapty().updateProfile`)
- Android subscription update / replacement params on `makePurchase` — see `revenuecat-to-adapty-map.md` §14 (Flutter)
- Webhooks / server-side notifications (App Store Server Notifications for iOS; Google Play Real-Time Developer Notifications for Android) — see `../dashboard-checklist.md`
