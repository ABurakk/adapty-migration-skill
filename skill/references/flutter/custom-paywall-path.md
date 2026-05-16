# Flutter Custom Paywall Path — direct `Adapty().makePurchase` (no AdaptyUI)

Read this file when the migrating file uses **its own custom Flutter paywall widgets** (the project's own `StatelessWidget` / `StatefulWidget` UI) plus direct purchase calls like `Purchases.purchasePackage(...)` or `Purchases.purchase(PurchaseParams(...))`. The Adapty equivalent is to call `Adapty().getPaywall` for the paywall metadata and product list, and `Adapty().makePurchase` / `Adapty().restorePurchases` for the transaction — keeping the project's existing widgets intact.

Migrations of files that use `purchases_ui_flutter` (or RevenueCatUI Dart APIs) go to `paywall-builder-path.md` (sibling file). Do not mix patterns across files.

> **Isolation reminder.** This file is the Flutter custom-paywall reference. The iOS equivalent is `../ios/custom-paywall-path.md` — different APIs, different async style. Do not cite or copy from the iOS file when working on a Flutter migration.

**Critical:** Flutter custom paywall migrations do NOT set `..withActivateUI(true)` on `AdaptyConfiguration`. They do NOT use `AdaptyUI()` APIs. They do NOT call `view.present()`. The whole UI layer stays the project's.

---

## 1. No AdaptyUI in this path

In `main()` (per `adapty-reference.md` §3):

```dart
final configuration = AdaptyConfiguration(apiKey: 'YOUR_PUBLIC_SDK_KEY');
// NO ..withActivateUI(true) here — this is a custom-UI migration
await Adapty().activate(configuration: configuration);
```

If the project has **mixed paths** (some screens custom, others Paywall Builder), then `..withActivateUI(true)` IS still required globally on the single `AdaptyConfiguration` — but in custom-paywall files specifically, do not call any `AdaptyUI()` API and do not present an AdaptyUI view.

---

## 2. Loading products for the paywall

Replace `await Purchases.getOfferings()` / `await Purchases.getCurrentOfferingForPlacement(...)` (and the `offering.availablePackages` usage) with a single async call: `Adapty().getPaywall(placementId:)`. The product list comes from the returned `AdaptyPaywall`.

### Canonical pattern in a `ChangeNotifier`-based view model

```dart
import 'package:adapty_flutter/adapty_flutter.dart';
import 'package:flutter/foundation.dart';

class PaywallViewModel extends ChangeNotifier {
  bool _isLoading = false;
  bool get isLoading => _isLoading;

  String? _errorMessage;
  String? get errorMessage => _errorMessage;

  List<AdaptyPaywallProduct> _products = const [];
  List<AdaptyPaywallProduct> get products => _products;

  AdaptyPaywall? _paywall;

  /// Call from `initState` (wrapped in a microtask) or when the paywall screen
  /// is opened. Surfaces failure via `errorMessage`.
  Future<void> load(String placementId) async {
    if (!AdaptyManager.instance.isInitialized) {
      _errorMessage = 'Adapty not ready yet';
      notifyListeners();
      return;
    }

    _isLoading = true;
    _errorMessage = null;
    notifyListeners();

    try {
      final paywall = await Adapty().getPaywall(
        placementId: placementId,
        locale: 'en',   // ← prefer the device locale; see adapty-reference.md §6
      );
      _paywall = paywall;

      // 🌐 Verify against current Adapty Flutter docs: in current releases the
      // product list is exposed on the paywall object (e.g. `paywall.products`)
      // OR fetched via a follow-up call. Confirm the exact accessor at migration
      // time and update this line accordingly.
      _products = await _productsForPaywall(paywall);
    } on AdaptyError catch (e) {
      _errorMessage = e.message;
      debugPrint('❌ Failed to load paywall $placementId: code=${e.code}, message=${e.message}');
    } finally {
      _isLoading = false;
      notifyListeners();
    }
  }

  Future<List<AdaptyPaywallProduct>> _productsForPaywall(AdaptyPaywall paywall) async {
    // 🌐 IMPLEMENTATION DEPENDS ON CURRENT SDK SHAPE. Verify against
    // `https://adapty.io/docs/flutter-quickstart-paywalls` at migration time
    // and replace this body with the correct accessor:
    //
    //   - If products are a synchronous property: `return paywall.products;`
    //   - If products require a separate call: `return await Adapty().getPaywallProducts(paywall: paywall);`
    //
    // Do not invent an accessor — use whatever the live docs document.
    throw UnimplementedError(
      'TODO: ADAPTY MANUAL — replace with the current Adapty Flutter product accessor for AdaptyPaywall.',
    );
  }
}
```

### Notes

- The `_paywall` reference is kept around as a field because Adapty uses it to attribute purchase events back to the correct placement / paywall variant. Discarding it after fetching products loses this attribution.
- `paywall.hasViewConfiguration` is NOT checked here. That flag matters for AdaptyUI rendering only; custom paywalls don't read it.
- The project's existing custom widget (e.g., `MyPaywallScreen`) binds to `viewModel.products` and renders price/title/period from the `AdaptyPaywallProduct` fields. Common Dart-side fields (🌐 verify exact names against current docs — they shift between SDK minor versions):
  - `product.localizedTitle`
  - `product.localizedPrice` (formatted string)
  - `product.localizedSubscriptionPeriod` / `product.subscriptionPeriod`
  - `product.vendorProductId` (matches the App Store Connect / Google Play product ID)
- Use whichever state-management style the project already uses (`ChangeNotifier`, `Stream`, `Riverpod`, `bloc`, `GetX`). The shape above is `ChangeNotifier`-based for readability — porting to the project's existing pattern is part of the migration, not a separate refactor.

---

## 3. Performing a purchase — direct `Adapty().makePurchase`

Replace `await Purchases.purchasePackage(package)` (and the `PurchaseParams` form) with:

```dart
Future<void> buy(AdaptyPaywallProduct product) async {
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
        // Purchase is in a pending state (e.g., StoreKit "Ask to Buy",
        // SCA challenge, Google Play deferred). The final state will arrive
        // asynchronously via Adapty().didUpdateProfileStream — DO NOT treat
        // this as failure.
        // Show a "pending" UI; the listener in AdaptyManager will refresh
        // isPremium when the purchase resolves.
        break;

      case AdaptyPurchaseResultUserCancelled():
        // User cancelled — silent, no error UI.
        break;
    }
  } on AdaptyError catch (e) {
    debugPrint('❌ Purchase failed: code=${e.code}, message=${e.message}');
    // surface a user-facing error message
  }
}
```

### Key differences from RevenueCat (Flutter)

- `Adapty().makePurchase(product: ...)` takes the `AdaptyPaywallProduct` directly. There is no separate "Package" concept on Adapty — products are paywall-scoped.
- The result is a sealed type with three cases: `AdaptyPurchaseResultSuccess(profile:)`, `AdaptyPurchaseResultPending()`, `AdaptyPurchaseResultUserCancelled()`. **User cancellation is NOT a thrown error** — it is the `AdaptyPurchaseResultUserCancelled` case.
- The `AdaptyPurchaseResultPending` case represents legitimate intermediate states (App Store Ask-to-Buy / SCA / Google Play deferred). **Treating `Pending` as failure silently drops valid purchases.** Show a pending UI and rely on `Adapty().didUpdateProfileStream` (wired in `AdaptyManager` per `adapty-reference.md` §5) to deliver the final profile when the purchase resolves.
- After a successful purchase, call `AdaptyManager.instance.refresh()` so other parts of the app observing the manager see the change immediately. The stream-driven listener will also fire, but the explicit refresh removes any race for the screen that triggered the purchase.

🌐 Verify case names and the sealed-class pattern shape against `https://adapty.io/docs/flutter-making-purchases` at migration time — some older SDK releases used an enum + accessor properties instead of Dart 3 sealed classes. If docs are unreachable, write the pattern as shown and flag the signature as unverified in Manual review.

### Android subscription change / upgrade

If the original RevenueCat code used `PurchaseParams` with `googleProductChangeInfo`, carry that over via `subscriptionUpdateParams` on `makePurchase`:

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

See `revenuecat-to-adapty-map.md` §14 for the full mapping. 🌐 Verify the parameter name and enum case names against current docs — both Adapty's wrapper and Google's underlying API have shifted as Google deprecated older proration constants. ⚠️ Always Manual review: "Project performs Google Play subscription replacement at `<file:line>`. Verify `oldSubVendorProductId` and `replacementMode` against the existing Google Play product setup and test in a Google Play sandbox account."

---

## 4. Restoring purchases — direct `Adapty().restorePurchases`

Replace `await Purchases.restorePurchases()` with:

```dart
Future<void> restore() async {
  try {
    final profile = await Adapty().restorePurchases();
    final isPremium = profile.accessLevels['premium']?.isActive ?? false;
    await AdaptyManager.instance.refresh();

    if (isPremium) {
      // confirm restore — show success UI
    } else {
      // tell user "nothing to restore"
    }
  } on AdaptyError catch (e) {
    debugPrint('❌ Restore failed: code=${e.code}, message=${e.message}');
    // surface a user-facing error message
  }
}
```

The single async call replaces the previous one. `restorePurchases()` returns the refreshed profile directly. 🌐 Verify return type against `https://adapty.io/docs/flutter-restore-purchase` at migration time.

---

## 5. Wiring the custom widget

The project's existing custom paywall widget binds to the view model exactly like it did before — only the underlying data source changed from `Purchases.*` to the view model. Concrete example (translate to the project's existing structure):

```dart
class MyPaywallScreen extends StatefulWidget {
  const MyPaywallScreen({super.key});
  @override
  State<MyPaywallScreen> createState() => _MyPaywallScreenState();
}

class _MyPaywallScreenState extends State<MyPaywallScreen> {
  final viewModel = PaywallViewModel();

  @override
  void initState() {
    super.initState();
    // Use a post-frame callback or microtask to start the async load.
    WidgetsBinding.instance.addPostFrameCallback((_) {
      viewModel.load('premium');
    });
  }

  @override
  void dispose() {
    viewModel.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: viewModel,
      builder: (context, _) {
        if (viewModel.isLoading) {
          return const Center(child: CircularProgressIndicator());
        }
        if (viewModel.errorMessage != null) {
          return Center(child: Text(viewModel.errorMessage!));
        }
        return Column(
          children: [
            for (final product in viewModel.products)
              ListTile(
                title: Text(product.localizedTitle),
                trailing: Text(product.localizedPrice ?? ''),
                onTap: () => viewModel.buy(product),
              ),
            TextButton(
              onPressed: viewModel.restore,
              child: const Text('Restore Purchases'),
            ),
          ],
        );
      },
    );
  }
}
```

For projects using `Provider` / `Riverpod` / `bloc` / `GetX` / `mobx`, port the shape into the project's existing pattern — do not inject `ChangeNotifier` into a codebase that doesn't already use it. The view model's *responsibilities* (load → expose products → buy → restore) carry over unchanged; only the binding mechanism differs.

---

## 6. The `AdaptyManager.instance.refresh()` pattern

Every successful purchase, restore, identify, or logout MUST call `AdaptyManager.instance.refresh()` so the global access-level state stays in sync with the server profile. Without this:

- The view that triggered the purchase sees the local result (good)
- Other views observing `AdaptyManager.instance` see stale state until the next `didUpdateProfileStream` event fires (bad)
- The user may appear to be "non-premium" elsewhere in the app for a few seconds after a successful purchase

The pattern is consistent across both paywall paths. Flag in Manual review for the user to verify: "Confirm `AdaptyManager.instance.refresh()` is called after each purchase/restore/identify/logout."

---

## 7. Failure modes specific to this path

1. **Placement returns no products** — the product list from `getPaywall` is empty. Cause: products not attached to the placement/paywall in Adapty Dashboard. Check Dashboard checklist (placements + products sections).
2. **`makePurchase` succeeds but `isPremium` is false** — the product is not assigned to the access level in the dashboard. Check Dashboard checklist (access levels section).
3. **App Store / Google Play sandbox state stuck** — old subscriptions from previous tests interfering. Create a fresh sandbox tester (App Store Connect) or use a Google Play licensed-testing account with a cleared subscription history.
4. **`paywall` reference discarded** — if the view model saves only `products` and not the `AdaptyPaywall` object, Adapty's attribution to that paywall/placement breaks. Keep both.
5. **Async/await ordering** — calling `makePurchase` before `activate` completes fails with `#2002 notActivated`. The `AdaptyManager.instance.isInitialized` guard prevents this.
6. **Concurrent identify + purchase** — firing `Adapty().identify(...)` at the same time as a paywall fetch or purchase can produce `#3006 profileWasChanged`. Serialize identify against subsequent SDK calls (see `adapty-reference.md` §6).
7. **`Pending` swallowed as failure** — the most common silent failure on this path. The `switch` shown in §3 MUST handle the `AdaptyPurchaseResultPending` case with a pending UI, not a generic error toast.
