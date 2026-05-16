# RevenueCat (Flutter) → Adapty (Flutter) API Mapping

This file is the lookup table the skill consults for each RevenueCat construct found in a Flutter project. Each entry tells the skill:

- The before (`purchases_flutter` / `purchases_ui_flutter`) and after (`adapty_flutter`) Dart code
- Which reference file the Adapty equivalent lives in (`adapty-reference.md`, `paywall-builder-path.md`, or `custom-paywall-path.md` — all siblings of this file)
- Whether the signature should be verified against the current Adapty Flutter docs for the current SDK version
- Notes about gotchas the skill must surface in Manual review

> **Isolation reminder.** This file is the Flutter mapping. The iOS mapping lives in `../ios/revenuecat-to-adapty-map.md` and MUST NOT be cross-cited. The two SDKs share concepts but not signatures — translating between them silently is the failure mode this skill exists to prevent.

**Important:** the skill never bakes a specific Adapty Flutter version into output without first verifying it against the current Adapty Flutter SDK installation docs (`https://adapty.io/docs/sdk-installation-flutter`) at migration time. The patterns below describe API *shape* (which methods, in what order); exact *signatures* (parameter names, named-vs-positional, return types) MUST be verified against current docs. If the runtime cannot reach the docs, flag every emitted signature in Manual review as unverified and emit `^<VERIFY_FROM_DOCS>` where a version number would go.

**Legend**

- ✅ **Reference-canonical** — pattern is in one of the reference files; use it verbatim. Cite the file + section.
- 🌐 **Verify signature** — the pattern shape is in the reference, but verify exact parameter names / return types against current Adapty Flutter docs at migration time. Cite the verified URL + today's date.
- 🌐🌐 **Verify first** — pattern is NOT in any reference; consult current Adapty Flutter docs for the concept before writing code.
- ⚠️ **Manual review item** — every verified mapping AND every offering↔placement mapping is automatically added to Manual review.
- 🅿️🅱️ **Paywall Builder path only** — applies only to files migrating via `paywall-builder-path.md`
- 🅒🅟 **Custom paywall path only** — applies only to files migrating via `custom-paywall-path.md`

---

## 1. Dependencies — `pubspec.yaml` ✅

### Before

```yaml
dependencies:
  purchases_flutter: ^<old-version>
  purchases_ui_flutter: ^<old-version>   # only if Paywall Builder path
```

### After

First consult `https://adapty.io/docs/sdk-installation-flutter` for the current supported `adapty_flutter` version. Then:

```yaml
dependencies:
  adapty_flutter: ^<VERSION>             # version confirmed from docs
```

There is no separate `adapty_ui_flutter` package — the AdaptyUI surface is part of `adapty_flutter` and is activated via `..withActivateUI(true)` on `AdaptyConfiguration` (see §3, Paywall Builder path only).

After updating, the user must run `flutter pub get`. The skill does NOT run this command; it surfaces it as a Manual review item: *"Run `flutter pub get` after applying the migrated `pubspec.yaml`."*

**Native dep housekeeping:** in the vast majority of Flutter projects, RevenueCat's native iOS/Android SDKs are pulled in transitively by `purchases_flutter` and require no manual `Podfile` / `build.gradle` edit. The skill does NOT auto-edit `ios/Podfile` or `android/build.gradle` for Adapty — `adapty_flutter` brings its native deps the same way. Only if the existing project has a manually-declared RC entry in `Podfile` / `build.gradle` should the skill remove it and add a Manual review note.

---

## 2. Imports ✅

### Before

```dart
import 'package:purchases_flutter/purchases_flutter.dart';
import 'package:purchases_ui_flutter/purchases_ui_flutter.dart';   // only on Paywall Builder path
```

### After

```dart
import 'package:adapty_flutter/adapty_flutter.dart';
```

One import covers both the core Adapty surface and the `AdaptyUI` surface. Do not synthesize a `package:adapty_ui_flutter/...` import — no such package exists.

---

## 3. SDK Initialization ✅

### Before — `Purchases.configure(PurchasesConfiguration(...))`

```dart
await Purchases.setLogLevel(LogLevel.debug);
await Purchases.configure(PurchasesConfiguration('rc_xxxxxxxxxxxx'));
// or, with user ID at launch:
final config = PurchasesConfiguration('rc_xxxxxxxxxxxx')..appUserID = 'user_123';
await Purchases.configure(config);
```

### After — `Adapty().activate(...)` awaited in `main()`

Pattern lives in `adapty-reference.md` §3. Verbatim:

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  try {
    final configuration = AdaptyConfiguration(apiKey: 'YOUR_PUBLIC_SDK_KEY')
      // ..withActivateUI(true)              // ← only on Paywall Builder migrations
      // ..withCustomerUserId('user_123')    // ← only if appUserID was set at configure time
      ;
    await Adapty().activate(configuration: configuration);
  } on AdaptyError catch (e) {
    debugPrint('❌ Adapty activation failed: code=${e.code}, message=${e.message}');
  }

  runApp(const MyApp());
}
```

### Rules

- `Adapty().activate(...)` is async — `await` it. Skipping `await` causes any subsequent SDK call to fail with `#2002 notActivated`.
- **No other Adapty call may execute before activate completes.** This is a hard SDK requirement.
- Include `..withActivateUI(true)` ONLY if the project has at least one Paywall Builder path file (i.e., uses `purchases_ui_flutter` or RevenueCatUI Dart APIs). Otherwise omit it.
- If `PurchasesConfiguration` set `appUserID` at configure time, chain `..withCustomerUserId('<id>')` on the `AdaptyConfiguration`. If the ID is only known after login, use `Adapty().identify(...)` instead — see §10.
- The API key literal stays `'YOUR_PUBLIC_SDK_KEY'` — never carry the RevenueCat key value across (different formats; silent failure).
- For verbose logging or custom configuration, use the fluent setters on `AdaptyConfiguration`. 🌐 Consult current Adapty Flutter docs (`https://adapty.io/docs/sdk-installation-flutter`) for the exact method names — these can shift between SDK minor versions.

---

## 4. Customer info / profile fetch ✅

### Before

```dart
final customerInfo = await Purchases.getCustomerInfo();
// use customerInfo.entitlements.active['pro'] etc.
```

### After (`adapty-reference.md` §5)

```dart
final profile = await Adapty().getProfile();
```

Confirm `Adapty().activate(...)` has completed first — see `adapty-reference.md` §6 ordering invariants. If called from a screen that may render before activation finishes, gate on `AdaptyManager.instance.isInitialized`.

---

## 5. Entitlement → access-level check ✅

### Before

```dart
final isPro = customerInfo.entitlements.active['pro']?.isActive ?? false;
// or
final isPro = customerInfo.entitlements.all['pro']?.isActive == true;
// or
final ent = customerInfo.entitlements.active['pro'];
if (ent != null && ent.isActive) { /* unlock */ }
```

### After (`adapty-reference.md` §5)

```dart
final isPro = profile.accessLevels['pro']?.isActive ?? false;
```

**Rule:** carry over the entitlement name (`'pro'`, `'premium'`, `'vip'`, whatever the project uses) unchanged. RevenueCat entitlements and Adapty access levels are semantically very similar (both are server-side capability flags), so a 1:1 rename is *usually* safe — but still flag in the Dashboard checklist: "Create an access level named `pro` in Adapty Dashboard → Access Levels (case-sensitive, must match the entitlement name from your old RevenueCat setup)."

---

## 6. Offerings → Paywall fetch — path-aware ⚠️

### Before

```dart
final offerings = await Purchases.getOfferings();
final offering = offerings.current;
final packages = offering?.availablePackages ?? [];

// or, with placement IDs:
final offering = await Purchases.getCurrentOfferingForPlacement('paywall_screen');
```

### After — depends on the file's paywall path

🅿️🅱️ For Paywall Builder path files, the fetch is wrapped inside the load + present helper from `paywall-builder-path.md` §2 (fetch the paywall → guard `hasViewConfiguration` → create the view → present).

🅒🅟 For custom paywall path files, the fetch is a single call in a view model — see `custom-paywall-path.md` §2:

```dart
final paywall = await Adapty().getPaywall(
  placementId: 'premium',   // ← see warning below
  locale: 'en',             // ← prefer device locale; see §6 of adapty-reference.md
);
// The product list comes from the paywall. 🌐 Verify the current Adapty Flutter
// docs for the exact accessor (`paywall.products`, a separate `getPaywallProducts`
// call, or similar) — the shape has shifted between SDK minor versions.
final products = await _productsForPaywall(paywall);  // see custom-paywall-path.md §2
```

### Type mapping

| RevenueCat (Flutter) | Adapty (Flutter) |
|---|---|
| `Offering` | `AdaptyPaywall` |
| `Offerings` (the collection) | (no direct equivalent) — Adapty fetches per-placement |
| `offering.availablePackages` | the products attached to the `AdaptyPaywall` (see `custom-paywall-path.md` §2) |
| `Package` | `AdaptyPaywallProduct` |
| `StoreProduct` | `AdaptyPaywallProduct` (it carries the store-product info) |

### ⚠️ Offering ↔ Placement is NOT a guaranteed 1:1 rename — Manual review item

Same caveat as the iOS path (the concepts are SDK-agnostic; the warning text is the same):

- A RevenueCat **Offering** bundles a set of *packages* (products) under a name. The app picks a specific offering and renders all its packages on one paywall.
- An Adapty **Placement** is an "ad slot" or display location. A placement has **one** paywall assigned to it, and that paywall in turn has its own product set assigned in the dashboard.

So:
- `RevenueCat offering "premium"` → `Adapty placement "premium"` is a reasonable starting point IF the project only has one paywall surface.
- If the project has multiple offerings used for different surfaces (e.g., onboarding vs settings), each needs its own placement AND its own paywall in Adapty Dashboard.
- If the project uses `Purchases.getCurrentOfferingForPlacement('paywall_screen')`, that placement ID maps cleanly to an Adapty placement ID with the same name — still flag in Manual review for semantic confirmation.

The skill carries the name over verbatim AND adds a Manual review entry: "RevenueCat offering / placement `<name>` was carried over as Adapty placement `<name>`. These concepts are related but not guaranteed 1:1. In the Adapty Dashboard, create a placement with this ID, attach a paywall to it, and confirm the products match the set your old RevenueCat offering exposed. If the project uses multiple offerings in different contexts, you may need multiple Adapty placements — review per call site."

---

## 7. Paywall UI presentation — strictly path-aware

### 🅿️🅱️ From `purchases_ui_flutter`'s `RevenueCatUI.presentPaywall` / `presentPaywallIfNeeded` / `PaywallView`

#### Before

```dart
// Programmatic presentation:
final result = await RevenueCatUI.presentPaywall();
// or with a specific offering:
final result = await RevenueCatUI.presentPaywall(offering: offering);
// or "show only if the user lacks an entitlement":
final result = await RevenueCatUI.presentPaywallIfNeeded('pro');

// Widget-style embed:
PaywallView(
  offering: offering,
  onPurchaseCompleted: ...,
  onRestoreCompleted: ...,
)
```

#### After

Adapty Flutter's paywall builder presentation is a four-step sequence — see the full pattern in `paywall-builder-path.md` §2. Mandatory elements:

- `..withActivateUI(true)` on the `AdaptyConfiguration` passed to `Adapty().activate(...)`
- `final paywall = await Adapty().getPaywall(placementId: '<id>')`
- `if (!paywall.hasViewConfiguration) return;` (graceful fallback; see `paywall-builder-path.md` §3)
- `final view = await AdaptyUI().createPaywallView(paywall: paywall);`
- `await view.present();`
- `AdaptyUI().setPaywallsEventsObserver(observer);` registered once at app startup, where `observer` implements `AdaptyUIPaywallsEventsObserver` and handles `paywallViewDidPerformAction(view, action)` with `CloseAction()`, `AndroidSystemBackAction()`, `OpenUrlAction(url:)` cases.

⚠️ Add to Manual review: "Enable 'Show on device' in Adapty Dashboard → Paywalls for placement `<id>`. Without this, `paywall.hasViewConfiguration` is empty/false and the migrated paywall renders nothing at runtime."

⚠️ Always Manual review for Paywall Builder migrations: **"RevenueCat paywall designs do NOT transfer to Adapty. For each placement migrated here, recreate the paywall layout in Adapty Dashboard → Paywalls → visual builder. Plan time for design rebuild, not just code review."**

#### Behavior differences vs RevenueCat

- RevenueCat's `presentPaywallIfNeeded('entitlement')` couples "fetch the paywall" with "only show if entitlement is missing." Adapty's flow separates these: the migration must fetch the paywall AND explicitly gate on `profile.accessLevels['<name>']?.isActive` in app code. The migrated code uses `AdaptyManager.instance.isPremium` (from `adapty-reference.md` §5) as the gate. **Do not invent an Adapty `presentIfNeeded` helper — there isn't one.**
- RevenueCat's `onPurchaseCompleted` / `onRestoreCompleted` `PaywallView` widget callbacks are replaced by automatic Paywall Builder flow (purchases/restores happen inside the AdaptyUI view) AND the `AdaptyUIPaywallsEventsObserver` for action events. Migration logic that relied on `onPurchaseCompleted` to refresh state should move into the observer's relevant callbacks OR rely on `Adapty().didUpdateProfileStream.listen(...)` (see §13).

### 🅒🅟 From a custom widget + `Purchases.purchasePackage(...)` — custom paywall path

#### Before

```dart
// Custom Flutter widgets the project owns:
class MyPaywall extends StatelessWidget {
  final Package package;
  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: () async {
        final result = await Purchases.purchasePackage(package);
        // unlock if entitlement is active
      },
      child: const Text('Subscribe'),
    );
  }
}
```

#### After

Keep the custom widget. Replace the data source and purchase call with patterns from `custom-paywall-path.md`:

- `Adapty().getPaywall(placementId: ...)` + product list from the paywall for product loading (§2)
- `Adapty().makePurchase(product: product)` for the buy action (§3)
- `Adapty().restorePurchases()` for restore (§4)
- Do NOT set `..withActivateUI(true)` in `AdaptyConfiguration` for a custom-only project
- `paywall.hasViewConfiguration` is NOT checked on this path

---

## 8. Direct purchase 🅒🅟 + 🌐

### Before

```dart
// Package-based purchase:
final purchaserInfo = await Purchases.purchasePackage(package);
final isPro = purchaserInfo.entitlements.active['pro']?.isActive ?? false;

// Or, with the newer PurchaseParams builder (also Android upgrade/replace):
final result = await Purchases.purchase(
  PurchaseParams.subscriptionOption(package, optionId: '...'),
);
```

### After — custom paywall path (`custom-paywall-path.md` §3)

```dart
try {
  final result = await Adapty().makePurchase(product: product);

  switch (result) {
    case AdaptyPurchaseResultSuccess(profile: final profile):
      final isPremium = profile.accessLevels['premium']?.isActive ?? false;
      if (isPremium) {
        await AdaptyManager.instance.refresh();
        // dismiss paywall / route the user
      }
      break;
    case AdaptyPurchaseResultPending():
      // StoreKit Ask-to-Buy / SCA / Google Play deferred — show pending UI.
      // Final state arrives via Adapty().didUpdateProfileStream (see §13).
      break;
    case AdaptyPurchaseResultUserCancelled():
      // No UI surfacing — user chose to back out.
      break;
  }
} on AdaptyError catch (e) {
  debugPrint('❌ Purchase failed: code=${e.code}, message=${e.message}');
  // surface a user-facing error message
}
```

🌐 **Verify against current docs.** The exact case names (`AdaptyPurchaseResultSuccess` / `AdaptyPurchaseResultPending` / `AdaptyPurchaseResultUserCancelled`) and their associated values are taken from the Adapty Flutter making-purchases docs. Confirm against `https://adapty.io/docs/flutter-making-purchases` at migration time — the sealed-class shape may have shifted between SDK minor versions (some older releases used an enum + accessor properties instead of pattern matching).

### Key differences from RevenueCat (Flutter)

- `Adapty().makePurchase(...)` takes the `AdaptyPaywallProduct` directly. There is no separate "Package" concept on Adapty — products are paywall-scoped.
- The result is a sealed type with three cases: `AdaptyPurchaseResultSuccess(profile:)`, `AdaptyPurchaseResultPending()`, `AdaptyPurchaseResultUserCancelled()`. **User cancellation is NOT a thrown error** — it's the `AdaptyPurchaseResultUserCancelled` case. Do not write a `catch` that treats user cancellation as failure.
- The `AdaptyPurchaseResultPending` case represents legitimate intermediate states (App Store Ask-to-Buy, SCA, Google Play deferred). **Treating it as failure silently drops valid purchases.** Show a pending UI and rely on `Adapty().didUpdateProfileStream` (§13) to deliver the final profile when the purchase resolves.
- After a successful purchase, call `AdaptyManager.instance.refresh()` so the global `isPremium` state propagates to all listeners. Without this, other parts of the app observing `AdaptyManager.instance` will see stale state.

⚠️ Always add to Manual review: "Direct purchase call at `<file:line>` — `AdaptyPurchaseResult*` types verified against `https://adapty.io/docs/flutter-making-purchases` on `<date>`. Confirm pattern-match shape against your installed `adapty_flutter` version. **Do NOT treat `AdaptyPurchaseResultPending` as failure** — that silently drops valid Ask-to-Buy / SCA / deferred-purchase flows."

🅿️🅱️ For Paywall Builder migrations, the purchase happens inside the AdaptyUI-presented view automatically — no direct `makePurchase` call in app code. Post-purchase state refresh comes from `Adapty().didUpdateProfileStream`.

---

## 9. Restore purchases ✅ + 🌐

### Before

```dart
final purchaserInfo = await Purchases.restorePurchases();
final isPro = purchaserInfo.entitlements.active['pro']?.isActive ?? false;
```

### After — custom paywall path (`custom-paywall-path.md` §4)

```dart
try {
  final profile = await Adapty().restorePurchases();
  final isPremium = profile.accessLevels['premium']?.isActive ?? false;
  await AdaptyManager.instance.refresh();
} on AdaptyError catch (e) {
  debugPrint('❌ Restore failed: code=${e.code}, message=${e.message}');
}
```

🌐 Verify return type against `https://adapty.io/docs/flutter-restore-purchase` — in current Adapty Flutter, `restorePurchases()` returns the refreshed `AdaptyProfile` directly; older releases may have wrapped it.

⚠️ Manual review: "Restore call at `<file:line>` — signature verified against `https://adapty.io/docs/flutter-restore-purchase` on `<date>`. Confirm against your installed SDK."

🅿️🅱️ For Paywall Builder migrations, restores happen inside the AdaptyUI-presented view automatically. A standalone "Restore Purchases" button OUTSIDE the paywall still uses `Adapty().restorePurchases()` as above.

---

## 10. User identification ✅ (with ordering invariant)

### Before

```dart
final result = await Purchases.logIn('user_id_123');
// result is a LogInResult — contains customerInfo and a `created` boolean
```

### After (`adapty-reference.md` §4)

```dart
await Adapty().identify('user_id_123');
// After identify resolves:
await AdaptyManager.instance.refresh();
```

### Critical ordering rules

1. `Adapty().activate(...)` MUST complete before `Adapty().identify(...)`. Both go through the activation sequence in `main()` or the auth flow — never fire identify before activate is awaited.
2. `Adapty().identify(...)` MUST complete before any subsequent `Adapty().getPaywall`, `Adapty().getProfile`, `Adapty().makePurchase`, or `Adapty().restorePurchases` call. Race conditions here can produce `#3006 profileWasChanged` AND attribute purchases to the anonymous profile (silent data-quality bug, not a crash).
3. If the project has a screen that may present a paywall while login is in progress, the screen must await identify completion (or check an "auth ready" flag) before fetching the paywall.

If the project never calls `Purchases.logIn` (anonymous-only), skip this section entirely — Adapty's anonymous profile handling takes over.

### Profile semantics differ from RevenueCat — critical for migration correctness

Same as the iOS path; reproduced here so the Flutter file is self-contained:

- **RevenueCat:** `logIn("A")` → `logIn("B")` creates a new profile for `"B"`. Switching back returns to the original. RC also merges some profile data across these transitions.
- **Adapty:** the SDK has a separate concept of `profile_id` (internal) and `customer_user_id` (your app's user ID). Calling `Adapty().identify("B")` on an anonymous profile **links that customer ID to the current profile**. If `"B"` already exists, the SDK switches to the existing profile. **Adapty does NOT merge profile data across these transitions.**

### Practical implications the skill MUST flag

- If the original RC code relies on `logIn("A")` → `logIn("B")` creating two distinct profiles, the equivalent Adapty migration should call `Adapty().logout()` between the two identify calls. The skill MUST detect any consecutive `logIn` calls without an intervening `logOut` and flag this as a behavior change.
- If the app re-sends user attributes (email, phone, custom attributes) on every login, continue doing so after migration — Adapty does not carry these across profile transitions.

⚠️ Always Manual review:

- "User identification migrated based on `adapty-reference.md` §4. Verify the customer ID format your backend uses matches what Adapty stores (Adapty preserves the string verbatim)."
- "**Profile semantics differ from RevenueCat:** Adapty does not merge profile data across identify/logout transitions, and repeated `identify` on the same profile does not create a new profile. If your RC code relied on `logIn(A)` → `logIn(B)` creating distinct profiles, insert an `Adapty().logout()` between them. Re-send user attributes (email, phone, custom attributes) after every identify if your app relied on RC's data-merging behavior."

---

## 11. Logout ✅

### Before

```dart
await Purchases.logOut();
```

### After (`adapty-reference.md` §4)

```dart
await Adapty().logout();
await AdaptyManager.instance.refresh();
```

`Adapty().logout()` reverts to a fresh anonymous profile — same semantics as `Purchases.logOut()`. Any access-level state for the previous user is cleared on the device; `AdaptyManager.instance.refresh()` propagates the change to UI listeners.

**Migration caveat — opt-out apps:** if the original RC project deliberately avoided `logOut` (some apps prefer to keep the user signed-in identifier permanently), preserve that behavior on the Adapty side — do not insert `Adapty().logout()` calls that weren't in the original code.

⚠️ Manual review: "Logout migrated. Adapty's `logout()` reverts to a fresh anonymous profile, same as RC's. If your app deliberately avoided logout, the migration preserved that behavior — verify no unintended `Adapty().logout()` calls were introduced."

---

## 12. Attribution / integration identifiers 🌐 ⚠️

`purchases_flutter` exposes a per-MMP setter family on `Purchases.setAttributes`, `Purchases.setAdjustID`, `Purchases.setAppsflyerID`, etc. Adapty's Flutter SDK unifies these behind a single API: **`Adapty().setIntegrationIdentifier(key: ..., value: ...)`** (verify exact method signature against current Flutter docs — search `https://adapty.io/docs/` for "Flutter attribution" or "integration identifier").

### Before (RevenueCat Flutter ID setters)

```dart
await Purchases.setAdjustID('<ADJUST_ID>');
await Purchases.setAppsflyerID('<APPSFLYER_ID>');
await Purchases.setFBAnonymousID('<FB_ANONYMOUS_ID>');
await Purchases.setOnesignalID('<ONESIGNAL_ID>');
await Purchases.setMixpanelDistinctID('<MIXPANEL_ID>');
await Purchases.setFirebaseAppInstanceID('<FIREBASE_ID>');
```

### After (Adapty Flutter integration identifiers) — 🌐 verify key strings + method signature against current docs

```dart
await Adapty().setIntegrationIdentifier(key: 'adjust_device_id', value: '<ADJUST_ID>');
await Adapty().setIntegrationIdentifier(key: 'appsflyer_id', value: '<APPSFLYER_ID>');
await Adapty().setIntegrationIdentifier(key: 'facebook_anonymous_id', value: '<FB_ANONYMOUS_ID>');
await Adapty().setIntegrationIdentifier(key: 'one_signal_subscription_id', value: '<ONESIGNAL_ID>');
await Adapty().setIntegrationIdentifier(key: 'mixpanel_user_id', value: '<MIXPANEL_ID>');
await Adapty().setIntegrationIdentifier(key: 'firebase_app_instance_id', value: '<FIREBASE_ID>');
```

**Ordering:** call `setIntegrationIdentifier` AFTER `Adapty().activate(...)` has completed, and before any user-action methods (paywall fetch, purchase). Per Adapty's call-order recommendations, integration identifiers belong in post-activation setup.

### Providers Adapty does NOT yet support (re-verify against current docs)

The skill MUST flag these as unsupported and leave the original RC call as a `// FIXME: ADAPTY MIGRATION BLOCKED — Adapty does not yet support <provider>` if the project uses any of them:

- mParticle, Airship, CleverTap, Kochava — confirm the current support list against `https://adapty.io/docs/` (the list shifts as Adapty ships new integrations)

For these, surface a top-level Manual review entry: "Project uses `<provider>`, which Adapty does not yet integrate with. The original RevenueCat call was left in place with a FIXME comment. Track Adapty's integrations roadmap for support."

### Full attribution payload uploads — separate API 🌐🌐

If the project forwards a full attribution payload from a provider callback (typical for Adjust / AppsFlyer), the Adapty Flutter equivalent is a separate `Adapty().updateAttribution(...)` call (verify method name + signature against current docs — search `https://adapty.io/docs/` for "Flutter Adjust" / "Flutter AppsFlyer"). Do not call this directly after `activate` with an empty payload — it produces silent attribution gaps. Hook into the provider's attribution callback and forward the payload from there.

⚠️ Manual review entries (always added when attribution code is detected):

- "Attribution ID setters migrated to `Adapty().setIntegrationIdentifier(...)`. The key strings (`adjust_device_id`, etc.) were chosen to match the Adapty docs as of `<date>` — re-verify against current docs if your SDK version is newer."
- "Configure each `<provider>` integration in Adapty Dashboard → Integrations — without dashboard configuration, integration identifiers and attribution payloads are dropped server-side."
- "If the project also forwards a full attribution payload via `updateAttribution(...)`, verify the source enum value and payload shape against current Adapty provider-specific Flutter docs."
- "If using authenticated users with attribution, ensure `Adapty().identify(...)` completes before the provider attribution callback fires, so the attribution data is bound to the right Adapty profile."

---

## 13. Customer info update listener → profile update stream ✅

### Before

```dart
Purchases.addCustomerInfoUpdateListener((customerInfo) {
  final isPro = customerInfo.entitlements.active['pro']?.isActive ?? false;
  // react
});
```

### After (`adapty-reference.md` §5 — wired inside `AdaptyManager`)

```dart
final subscription = Adapty().didUpdateProfileStream.listen((profile) {
  final isPro = profile.accessLevels['pro']?.isActive ?? false;
  // react
});
// Cancel on disposal: subscription.cancel();
```

The stream fires whenever the profile changes for any reason: a successful purchase, a restore, a logout, a server-side renewal/cancellation arriving via App Store Server Notifications or Google Play RTDN, an attribution-driven access-level update, etc. The migration wires the listener once inside `AdaptyManager.initialize()` (see `adapty-reference.md` §5) and lets the manager's `notifyListeners()` / `Stream` propagation update the rest of the app.

⚠️ Manual review: "Replaced `Purchases.addCustomerInfoUpdateListener` at `<file:line>` with `Adapty().didUpdateProfileStream.listen(...)` inside `AdaptyManager.initialize()`. Confirm the manager singleton is long-lived (not disposed by short-lived widgets) so the listener keeps firing for the app's lifetime."

---

## 14. Android subscription change / upgrade params 🌐 ⚠️

If the project performs Google Play subscription replacement (upgrade / downgrade / cross-grade) via the `PurchaseParams` builder's `googleProductChangeInfo` / `replacementMode`, that logic must carry over to Adapty explicitly via `subscriptionUpdateParams` on `makePurchase`.

### Before

```dart
final result = await Purchases.purchase(
  PurchaseParams.subscriptionOption(
    package,
    googleProductChangeInfo: GoogleProductChangeInfo(
      oldProductIdentifier: 'old_sub_id',
      googleProrationMode: GoogleProrationMode.immediateWithTimeProration,
    ),
  ),
);
```

### After — Adapty Flutter `subscriptionUpdateParams`

```dart
final updateParams = AdaptySubscriptionUpdateParameters(
  oldSubVendorProductId: 'old_sub_id',
  replacementMode: AdaptyAndroidSubscriptionUpdateReplacementMode.withTimeProration,
);

final result = await Adapty().makePurchase(
  product: product,
  subscriptionUpdateParams: updateParams,
);
```

🌐 Verify both the parameter name (`subscriptionUpdateParams` vs `subscriptionUpdateParameters`) AND the enum case names (`withTimeProration`, `chargeProratedPrice`, etc.) against current Adapty Flutter docs (`https://adapty.io/docs/flutter-making-purchases` and the Android-specific page). The proration / replacement-mode naming shifted when Google deprecated the older constants — both Adapty's wrapper and Google's underlying API have moved.

⚠️ Manual review: "Project performs Google Play subscription replacement at `<file:line>`. Carried over to Adapty via `subscriptionUpdateParams`. Verify (a) the `oldSubVendorProductId` matches the existing Google Play product ID exactly, (b) the `replacementMode` enum case corresponds to the original RC proration mode, (c) the migrated flow has been tested in a Google Play sandbox account with an active subscription on the old product. **Do NOT silently drop this parameter** — purchases will succeed but the user will be billed twice or not get the expected proration."

---

## 15. Hard-coded product IDs / fallback products ❗️ BLOCKER

RevenueCat Flutter exposes:

```dart
final products = await Purchases.getProducts(['com.app.product.weekly', 'com.app.product.yearly']);
final purchaserInfo = await Purchases.purchaseStoreProduct(products[0]);
```

This is commonly used for fallback paywalls — a hard-coded set of products the app can fall back to if the dashboard is unreachable.

**Adapty has no equivalent API on Flutter.** Products in Adapty 3.x are paywall-scoped: you fetch them via `Adapty().getPaywall(placementId:)` and use the products attached to the returned `AdaptyPaywall`. There is no "give me these product IDs without a paywall" call.

**Migration action — this is a MIGRATION BLOCKER, not a silent rewrite:**

1. **Leave the original RC code in place.**
2. Add a `// FIXME: ADAPTY MIGRATION BLOCKED — Purchases.getProducts([...]) has no Adapty Flutter equivalent` comment above the call.
3. Surface a top-level Manual review entry:
   > "**Migration blocker at `<file:line>`:** the project uses `Purchases.getProducts([...])` to fetch products by ID without a paywall. Adapty 3.x does not support this on Flutter — products must be fetched via a placement + paywall. Migration options: (a) create a dedicated Adapty placement for this set of products and migrate the code to `getPaywall` + product list; (b) if this was used as a fallback, rebuild the fallback as an Adapty placement with the same product set. The migration cannot proceed automatically — review the surrounding business logic before choosing an option."
4. Mention the blocker in the report Summary so the user knows the migration is partial.

---

## 16. "Present if entitlement missing" gating — no direct equivalent ⚠️

`purchases_ui_flutter` exposes `RevenueCatUI.presentPaywallIfNeeded('pro')` — it combines "check entitlement" + "show paywall if missing" in one call. **Adapty has no direct equivalent.** The Flutter migration must implement the gating in app code:

```dart
Future<void> showPaywallIfNotPremium() async {
  // Fast path: rely on cached state from AdaptyManager.
  if (AdaptyManager.instance.isPremium) {
    return;
  }

  // Optional: re-fetch the profile to be absolutely sure (e.g., right after launch).
  try {
    final profile = await Adapty().getProfile();
    final isPro = profile.accessLevels['pro']?.isActive ?? false;
    if (isPro) {
      await AdaptyManager.instance.refresh();
      return;
    }
  } on AdaptyError catch (_) {
    // Fall through and show the paywall — soft-fail rather than block monetization.
  }

  await loadPaywallAndShow();   // see paywall-builder-path.md §2 (Paywall Builder)
                                // OR navigate to your custom paywall screen
}
```

⚠️ Manual review: "Replaced `RevenueCatUI.presentPaywallIfNeeded('<entitlement>')` at `<file:line>` with an explicit `AdaptyManager.instance.isPremium` + optional `Adapty().getProfile()` gate, followed by the platform's paywall presentation. Adapty does not expose a single 'present if entitlement missing' helper on Flutter."

---

## 17. Custom user attributes 🌐

### Before

```dart
await Purchases.setAttributes({'email': 'user@example.com'});
// or:
await Purchases.setEmail('user@example.com');
```

### After

🌐 Consult current Adapty Flutter docs (search `https://adapty.io/docs/` for "Flutter update profile" / "user attributes" / `AdaptyProfileParameters`). Expected shape — verify exact builder method names:

```dart
final params = AdaptyProfileParameters.builder()
    .setEmail('user@example.com')
    // .setCustomAttributes(...)
    .build();

await Adapty().updateProfile(params);
```

Confirm exact method names against the fetched docs. Add to Manual review.

---

## 18. Capability check

### Before

```dart
final canMake = await Purchases.canMakePayments();
```

### After

🌐🌐 Adapty Flutter does not wrap StoreKit's `canMakePayments` / Google Play's billing-availability check. Use the platform-native equivalents via existing Flutter plugins (e.g., `in_app_purchase` for capability checks) OR drop the check if the original codebase only used it defensively. Add to Manual review: "Removed `Purchases.canMakePayments()` at `<file:line>` — Adapty Flutter does not wrap this. If your app needs to gate UI on store availability, use a separate plugin (`in_app_purchase`'s `InAppPurchase.instance.isAvailable()`) or rely on `Adapty().getPaywall(...)` errors as the signal."

---

## 19. Type cross-reference

| RevenueCat (Flutter) type | Adapty Flutter equivalent | Notes |
|---|---|---|
| `Purchases` (the class with static methods) | `Adapty()` (instance accessor) | `adapty_flutter` uses instance methods on `Adapty()`, not static methods |
| `CustomerInfo` | `AdaptyProfile` | Returned by `Adapty().getProfile()` |
| `EntitlementInfo` | `AdaptyAccessLevel` (the value type in `profile.accessLevels`) | Accessed via `profile.accessLevels['<name>']` |
| `Offerings` | (no direct equivalent) | Use `Adapty().getPaywall(placementId:)` per call |
| `Offering` | `AdaptyPaywall` | One per placement |
| `Package` | `AdaptyPaywallProduct` | Provided by the paywall |
| `StoreProduct` | `AdaptyPaywallProduct` | The Flutter SDK does not expose a separate "store product" type — the paywall product carries store metadata |
| `LogInResult` | `void` (return of `Adapty().identify(...)`) | Adapty's identify does not return a `created` flag; query the profile afterward if you need it |
| `PurchasesConfiguration` | `AdaptyConfiguration` | Builder-style with fluent setters (`..withCustomerUserId(...)`, `..withActivateUI(true)`, etc.) |
| `Purchases.addCustomerInfoUpdateListener` | `Adapty().didUpdateProfileStream.listen(...)` | See §13 |
| `PurchaseParams.GoogleProductChangeInfo` | `AdaptySubscriptionUpdateParameters` | See §14 — verify exact shape |

---

## 20. Unmapped patterns

If the skill encounters a `purchases_flutter` / `purchases_ui_flutter` construct not listed above:

1. Search current Adapty Flutter docs at `https://adapty.io/docs/` for the concept (try the RevenueCat keyword + the closest Adapty term — e.g., "entitlement" → "access level", "offering" → "placement", "package" → "product")
2. If the docs make the equivalent unambiguous, write the code and add to Manual review with the source URL + date
3. If still unclear, or if the runtime cannot reach the docs, leave `// TODO: ADAPTY MANUAL — <one-line description of what was being done>` at the call site and surface it in Manual review

Do not guess. Do not fall back to "what RevenueCat does is probably what Adapty does." Migration silence is more expensive than asking the user to review one line.
