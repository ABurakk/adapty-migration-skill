# Flutter Paywall Builder Path — `AdaptyUI()` + `createPaywallView` + `view.present()`

Read this file when the migrating file uses `purchases_ui_flutter` (`RevenueCatUI.presentPaywall`, `RevenueCatUI.presentPaywallIfNeeded`, `PaywallView` widget) or otherwise indicates the project relies on RevenueCat's Paywall Builder for paywall rendering. The Adapty equivalent is the AdaptyUI Flutter paywall builder, rendered via `AdaptyUI().createPaywallView(paywall:)` + `view.present()`, plus an `AdaptyUIPaywallsEventsObserver` for action handling.

Migrations of files NOT in this category go to `custom-paywall-path.md` (sibling file). Do not mix patterns across files.

> **Isolation reminder.** This file is the Flutter Paywall Builder reference. The iOS equivalent is `../ios/paywall-builder-path.md` (SwiftUI `.paywall()` modifier — completely different API). Do not cite or copy from the iOS file when working on a Flutter migration.

**Prerequisite:** the migrating project's `main()` uses the activation sequence from `adapty-reference.md` §3, AND the `AdaptyConfiguration` includes **`..withActivateUI(true)`** (the AdaptyUI subsystem MUST be activated on Paywall Builder migrations).

---

## 0. Scope of this migration — what does NOT transfer

The skill migrates the **code** that fetches and presents the paywall. It does **not** migrate the visual design of your paywalls.

**You will need to rebuild your paywall design from scratch in Adapty's visual builder — RevenueCat paywall layouts do not transfer. Use a template as a starting point, or recreate your existing design.**

This applies to every Flutter Paywall Builder migration. The code emitted here renders whatever exists in Adapty Dashboard for the given placement — so until a paywall design exists there, the `paywall.hasViewConfiguration` guard will fall through and nothing visible will happen at runtime.

**Mandatory Manual review entry the skill always surfaces on this path:**

> "RevenueCat paywall designs do NOT transfer to Adapty. For each placement migrated here, recreate the paywall layout in Adapty Dashboard → Paywalls → visual builder. Plan time for design rebuild, not just code review."

---

## 1. AdaptyUI activation (required only on this path)

In `main()`, set `..withActivateUI(true)` on the `AdaptyConfiguration`:

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  try {
    final configuration = AdaptyConfiguration(apiKey: 'YOUR_PUBLIC_SDK_KEY')
      ..withActivateUI(true);
      // ..withCustomerUserId('USER_ID')   // if known at launch
    await Adapty().activate(configuration: configuration);

    // Register the paywall events observer once, after activation.
    AdaptyUI().setPaywallsEventsObserver(MyPaywallEventsObserver());

    await AdaptyManager.instance.initialize();
  } on AdaptyError catch (e) {
    debugPrint('❌ Adapty activation failed: code=${e.code}, message=${e.message}');
  }

  runApp(const MyApp());
}
```

`..withActivateUI(true)` is what activates the AdaptyUI surface. Without it, `AdaptyUI().createPaywallView(...)` will fail at runtime. Register the events observer (see §4) once at startup so action callbacks reach the app from any paywall presented later.

---

## 2. Paywall presentation — canonical Flutter pattern

The pattern below replaces `RevenueCatUI.presentPaywall(...)` / `RevenueCatUI.presentPaywallIfNeeded(...)` / `PaywallView(...)`. Adapt to the user's app structure, but keep the four-step sequence AND the `paywall.hasViewConfiguration` guard (§3).

```dart
import 'package:adapty_flutter/adapty_flutter.dart';
import 'package:flutter/foundation.dart';

Future<void> loadPaywallAndShow({String placementId = 'premium'}) async {
  // Gate on activation: prevents #2002 notActivated when called from a screen
  // that may render before main() finishes awaiting activate().
  if (!AdaptyManager.instance.isInitialized) {
    debugPrint('⚠️ Adapty not yet initialized — aborting paywall load');
    return;
  }

  try {
    // 1) Fetch the paywall metadata for this placement.
    final paywall = await Adapty().getPaywall(
      placementId: placementId,
      locale: 'en',   // ← prefer the device locale; see adapty-reference.md §6
    );

    // 2) Guard hasViewConfiguration. The Paywall Builder paywall only renders
    //    when the dashboard's "Show on device" toggle is enabled for this
    //    placement. Without a view configuration, AdaptyUI cannot render —
    //    fall back gracefully.
    if (!paywall.hasViewConfiguration) {
      debugPrint(
        "⚠️ Paywall '$placementId' has no view configuration. "
        "Enable 'Show on device' in Adapty Dashboard for this placement, "
        "or use the custom paywall path for this screen.",
      );
      // Optional: navigate to a fallback UI here.
      return;
    }

    // 3) Create the AdaptyUI view from the paywall.
    final view = await AdaptyUI().createPaywallView(paywall: paywall);

    // 4) Present the view. User interactions (purchase, restore, close)
    //    flow through the AdaptyUIPaywallsEventsObserver registered at startup.
    await view.present();
  } on AdaptyError catch (e) {
    debugPrint('❌ Failed to load/present paywall: code=${e.code}, message=${e.message}');
  }
}
```

### Customization rules

- Substitute `'premium'` in `placementId: 'premium'` with the user's RevenueCat offering / placement name carried over (and flag it in Manual review as offering↔placement is not guaranteed semantic equivalence).
- KEEP `locale: 'en'` only as a fallback. Prefer the device locale: `Localizations.localeOf(context).toLanguageTag()`, or the project's existing locale source. Hardcoding a single language disables Adapty's localization layer.
- KEEP the `paywall.hasViewConfiguration` guard. Non-negotiable; see §3.
- KEEP the `AdaptyManager.instance.isInitialized` check. It prevents the most common timing crash (`#2002 notActivated`) when a UI screen triggers `loadPaywallAndShow` before activation completes.

### Where post-purchase state lives

The migrated code does NOT call `Adapty().makePurchase(...)` directly on this path — the AdaptyUI-presented view handles purchases and restores internally. Post-purchase state arrives via two channels:

1. **`Adapty().didUpdateProfileStream`** — already wired inside `AdaptyManager.initialize()` per `adapty-reference.md` §5. When the user buys or restores, the stream fires with the refreshed profile and `AdaptyManager.instance.isPremium` flips automatically. The rest of the app, if observing the manager, sees the change without any extra code.
2. **The events observer** (see §4) — called for navigation actions (close, system back, link tap). Use it to dismiss your route stack after purchase / close, not to recompute access state (that's the stream's job).

If the original code had `onPurchaseCompleted` / `onRestoreCompleted` callbacks on `PaywallView`, the migration's equivalent is:
- The stream listener picks up the access-level change.
- The events observer's `paywallViewDidPerformAction` `CloseAction` case (or your own logic after `view.present()` resolves) handles route dismissal.

---

## 3. The `paywall.hasViewConfiguration` guard — non-negotiable

`paywall.hasViewConfiguration` is truthy only when the placement's paywall in Adapty Dashboard has the **"Show on device"** toggle enabled. This is the Paywall Builder rendering flag; without it, `AdaptyUI().createPaywallView(...)` cannot render the paywall (it either throws or returns an unpresentable view).

### Why the guard is non-optional

- The migration cannot detect from the project alone whether the user will enable "Show on device" in the dashboard — that's a manual dashboard step.
- A paywall that loads server-side but has `hasViewConfiguration == false` produces silent UI failure: the user taps "Get Premium", nothing happens, no error visible, no purchase ever attempted.
- The guard converts that silent failure into a logged warning, giving the user a debuggable signal.

### What goes in the fallback branch

Three reasonable options, in increasing complexity:

1. **Just log + return.** Acceptable for the migration's first pass; the user notices when their paywall doesn't show and fixes the dashboard.
2. **Show a generic "Premium Features" sheet with static product info.** Useful if the project must always be able to monetize; usually overkill for migration v1.
3. **Fall back to the custom paywall path for this screen.** Most robust — but requires the file to also be migrated as custom paywall. Flag in Manual review if this hybrid is needed.

The skill's default is option 1 (log + return). It surfaces a Manual review item: *"File `<path>:<line>` falls back to no-op if `paywall.hasViewConfiguration` is false. Enable 'Show on device' in Adapty Dashboard for placement `<id>`."*

---

## 4. Paywall action handling — `AdaptyUIPaywallsEventsObserver`

`purchases_ui_flutter`'s `PaywallView` exposes per-callback parameters (`onPurchaseCompleted`, `onRestoreCompleted`, `onDismiss`, etc.). The Adapty Flutter equivalent is a single observer registered once at startup that receives **action events** for things the user does on the paywall (close it, follow a link, hit the Android system back button). Purchase / restore success is observed via the profile stream (§2), not via observer callbacks — keep these responsibilities separate.

### Implementation

```dart
import 'package:adapty_flutter/adapty_flutter.dart';
import 'package:flutter/foundation.dart';
import 'package:url_launcher/url_launcher.dart';  // optional, for OpenUrlAction

class MyPaywallEventsObserver implements AdaptyUIPaywallsEventsObserver {
  @override
  void paywallViewDidPerformAction(
    AdaptyUIPaywallView view,
    AdaptyUIAction action,
  ) {
    switch (action) {
      case CloseAction():
        view.dismiss();
        break;
      case AndroidSystemBackAction():
        view.dismiss();
        break;
      case OpenUrlAction(url: final url):
        // Use the project's existing link-opening mechanism (url_launcher,
        // a custom in-app browser, the platform default — whatever the app
        // already does for external URLs).
        launchUrl(Uri.parse(url));
        break;
      default:
        // 🌐 Verify the full action set against current Adapty Flutter docs
        // (search https://adapty.io/docs/ for "Flutter paywall events" or
        // "AdaptyUIAction"). New action types may appear between SDK
        // versions; treat unknowns as no-ops rather than crashing.
        break;
    }
  }
}
```

Register the observer once (typically in `main()`, right after `Adapty().activate(...)`):

```dart
AdaptyUI().setPaywallsEventsObserver(MyPaywallEventsObserver());
```

### 🌐 Verify the observer interface against current docs

The exact method name (`paywallViewDidPerformAction` vs `didPerformAction`), the parameter order, and the full set of action subclasses (`CloseAction`, `AndroidSystemBackAction`, `OpenUrlAction`, plus any others introduced in newer SDK releases) should be confirmed against `https://adapty.io/docs/flutter-quickstart-paywalls` at migration time. Some older releases had a different observer shape (`AdaptyUIObserver` with broader event coverage); the current Flutter API focuses observer callbacks on action events specifically.

⚠️ Always Manual review when migrating from `PaywallView` callbacks: *"Replaced `PaywallView` per-callback parameters (`onPurchaseCompleted`, `onRestoreCompleted`, `onDismiss`) with: (a) the `Adapty().didUpdateProfileStream` listener inside `AdaptyManager.initialize()` for purchase/restore state, and (b) `AdaptyUIPaywallsEventsObserver.paywallViewDidPerformAction` for action events. If your `onPurchaseCompleted` did app-specific work beyond state refresh (e.g., logged a custom analytics event, navigated to a thank-you screen), audit that work and move it into the matching code path — the stream listener for state, an action observer case for navigation, or a `await view.present()` continuation for screen-flow completion."*

---

## 5. Embedded paywall (RevenueCat `paywallFooter` / inline paywall equivalent)

RevenueCat's `paywallFooter` (and the `PaywallView` widget rendered inline within a parent widget tree) embed the paywall inside the app's own layout instead of taking over the screen. Adapty Flutter's paywall builder supports embedded layouts — but the embedded configuration is set in the **Adapty Dashboard visual builder** (paywall → Layout → Embedded), not in code.

Code-side, the migration still uses `createPaywallView` + `view.present()` for fullscreen paywalls. For embedded layouts:

- 🌐🌐 **Verify against current Adapty Flutter docs** whether inline-embed support is available via a dedicated widget (e.g., `AdaptyPaywallWidget`) or only via dashboard-configured `..present()` with an embedded layout. The API surface for inline embedding has been less stable than the fullscreen path; do not invent a widget class without confirmation. If the docs do not describe a Flutter widget for inline embedding at migration time, flag this as Manual review: *"File `<path>` used RevenueCatUI's inline / embedded paywall presentation. Adapty Flutter's inline embed support depends on SDK version — verify against `https://adapty.io/docs/flutter-quickstart-paywalls` and either (a) use the current embed API if available, or (b) migrate this screen to the custom paywall path for full layout control."*

---

## 6. Failure modes specific to this path

When this path goes wrong, it usually goes wrong silently. The most common causes:

1. **"Show on device" disabled in Adapty Dashboard** — `paywall.hasViewConfiguration` is false; the guard logs but UI does nothing. (Most common silent failure.)
2. **`..withActivateUI(true)` omitted** — `AdaptyUI().createPaywallView(...)` fails or no-ops. Verify the `AdaptyConfiguration` in `main()`.
3. **Placement ID mismatch** — code says `'premium'`, dashboard placement is `'Premium'`. Case-sensitive.
4. **Paywall in draft, not published** — `getPaywall` returns a paywall without a view config. Publish in the dashboard.
5. **Products not assigned to placement / paywall** — the AdaptyUI view renders but shows no products. Add products in the dashboard.
6. **Events observer not registered** — `Close` / `OpenUrl` actions are silently dropped, so the user gets stuck on the paywall. Register the observer once at startup, before the first `view.present()`.
7. **Profile stream subscription dropped** — if `AdaptyManager` is disposed by a short-lived widget, the `didUpdateProfileStream` listener stops firing and post-purchase state refreshes never reach the rest of the app. Keep `AdaptyManager.instance` long-lived.
8. **Locale not supported** — paywall content shows in a fallback language; not a crash but a UX bug. Add localizations in Adapty Dashboard for the locales the project ships in.

The skill surfaces #1, #2, #3, #4, #5 in the Dashboard checklist for the user to verify before testing.
