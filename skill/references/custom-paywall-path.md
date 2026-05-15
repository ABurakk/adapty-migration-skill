# Custom Paywall Path — direct `Adapty.makePurchase` (no AdaptyUI)

Read this file when the migrating file uses **its own custom paywall UI** (SwiftUI or UIKit views the project owns) plus direct purchase calls like `Purchases.shared.purchase(package:)`. The Adapty equivalent is to call `Adapty.getPaywall` + `Adapty.getPaywallProducts` for product loading, and `Adapty.makePurchase` / `Adapty.restorePurchases` for the transaction — keeping the project's existing UI views intact.

Migrations of files that use RevenueCatUI go to `paywall-builder-path.md`. Do not mix patterns across files.

**Critical:** custom paywall migrations do NOT add `import AdaptyUI`. They do NOT call `AdaptyUI.activate()`. They do NOT use the `.paywall()` modifier. The whole UI layer stays the project's.

---

## 1. No AdaptyUI in this path

In the `@main` struct's activation Task (per `adapty-reference.md` §3):

```swift
try await Adapty.activate("YOUR_ADAPTY_PUBLIC_KEY")
// NO AdaptyUI.activate() here — this is a custom-UI migration
await AdaptyManager.shared.initialize()
```

If the project has **mixed paths** (some screens Paywall Builder, others custom), then `AdaptyUI.activate()` IS still required globally — but in custom-paywall files specifically, do not `import AdaptyUI` and do not call `.paywall()`.

## 2. Loading products for the paywall

Replace `Purchases.shared.getOfferings { ... }` (and the `offering.availablePackages` usage) with a two-call async sequence: `Adapty.getPaywall` (the placement metadata) then `Adapty.getPaywallProducts` (the buyable products array).

**Canonical pattern in a view model:**

```swift
import Foundation
import Adapty

@MainActor
final class PaywallViewModel: ObservableObject {
    @Published private(set) var products: [AdaptyPaywallProduct] = []
    @Published private(set) var isLoading = false
    @Published private(set) var errorMessage: String?

    private var paywall: AdaptyPaywall?

    /// Call from .onAppear or when the paywall screen is presented.
    /// Returns silently on failure — UI binds to `products` and `errorMessage`.
    func load(placementId: String) async {
        guard AdaptyManager.shared.isAdaptyInitialized else {
            errorMessage = "Adapty not ready yet"
            return
        }

        isLoading = true
        defer { isLoading = false }

        do {
            let paywall = try await Adapty.getPaywall(
                placementId: placementId,
                locale: Locale.current.identifier
            )
            self.paywall = paywall
            self.products = try await Adapty.getPaywallProducts(paywall: paywall)
            self.errorMessage = nil
        } catch {
            self.errorMessage = error.localizedDescription
            print("❌ Failed to load custom paywall \(placementId): \(error)")
        }
    }
}
```

**Notes:**

- The `paywall` reference is kept around as a property because Adapty uses it to attribute purchase events back to the correct placement / paywall variant. Discarding it after `getPaywallProducts` loses this attribution.
- `hasViewConfiguration` is NOT checked here. That flag matters for AdaptyUI rendering only; custom paywalls don't read it.
- The project's existing custom view (e.g., `MyPaywallScreen`) binds to `viewModel.products` and renders price/title/period from the `AdaptyPaywallProduct` fields:
  - `product.localizedTitle`
  - `product.localizedPrice`
  - `product.subscriptionPeriod`
  - `product.vendorProductId` (matches the App Store Connect product ID)

## 3. Performing a purchase — direct `Adapty.makePurchase`

Replace `Purchases.shared.purchase(package: pkg) { transaction, customerInfo, error, userCancelled in ... }` with:

```swift
func buy(_ product: AdaptyPaywallProduct) async {
    do {
        let result = try await Adapty.makePurchase(product: product)

        switch result {
        case .userCancelled:
            // User cancelled — silent, no error UI
            return

        case .pending:
            // Purchase is in a pending state (e.g., StoreKit "Ask to Buy",
            // SCA challenge, billing issue). The final state will arrive
            // asynchronously via AdaptyDelegate.didLoadLatestProfile.
            // Show a "pending" UI; do NOT treat this as failure.
            return

        case let .success(profile, _):
            let isPremium = profile.accessLevels["premium"]?.isActive ?? false
            if isPremium {
                await AdaptyManager.shared.refresh()
                // dismiss paywall / route the user
            }
        }
    } catch {
        print("❌ Purchase failed: \(error)")
        // surface a user-facing error message
    }
}
```

**Key differences from RevenueCat:**

- `Adapty.makePurchase` takes the `AdaptyPaywallProduct` directly. There is no separate "Package" concept — products are paywall-scoped.
- The result is an `AdaptyPurchaseResult` enum with three cases: `.userCancelled`, `.pending`, and `.success(profile:transaction:)`. **User cancellation is NOT a thrown error** — it is the `.userCancelled` case. Do not write `catch let error as AdaptyError where error.adaptyErrorCode == .paymentCancelled`; that pattern is wrong for `makePurchase`. The `throw` path is reserved for actual failures (network, configuration, etc.).
- The `.pending` case represents legitimate intermediate states from StoreKit (Ask to Buy, SCA, deferred). Treating `.pending` as failure silently drops valid purchases. Show a pending UI and rely on `AdaptyDelegate.didLoadLatestProfile` to deliver the final profile when the purchase resolves.
- On `.success`, destructure as `case let .success(profile, _)` — the success case carries the refreshed `AdaptyProfile` (and the transaction). `result.profile` also exists as a convenience computed property on the enum, but the switch form makes the three-state handling explicit and is the recommended pattern.
- After a successful purchase, call `AdaptyManager.shared.refresh()` to update the global `isPremium` state. Without this, other parts of the app observing `AdaptyManager.shared.$isPremium` won't see the change immediately.

Verify the exact case names and associated values against current Adapty docs (search `https://adapty.io/docs/` for "makePurchase" / "AdaptyPurchaseResult") at migration time — they have shifted between minor releases. If docs are unreachable, write the pattern as shown and flag the signature as unverified in Manual review.

## 4. Restoring purchases — direct `Adapty.restorePurchases`

Replace `Purchases.shared.restorePurchases { customerInfo, error in ... }` with:

```swift
func restore() async {
    do {
        let profile = try await Adapty.restorePurchases()
        let isPremium = profile.accessLevels["premium"]?.isActive ?? false
        await AdaptyManager.shared.refresh()

        if isPremium {
            // confirm restore
        } else {
            // tell user "nothing to restore"
        }
    } catch {
        print("❌ Restore failed: \(error)")
        // surface a user-facing error message
    }
}
```

The single async call replaces the closure pattern. `restorePurchases` returns the refreshed profile directly.

## 5. Wiring the custom view

The project's existing custom paywall view binds to the view model exactly like it did before — only the underlying data source changed from `Purchases.shared.*` to the view model. Concrete examples (translate to the project's existing structure):

**SwiftUI:**
```swift
struct MyPaywallScreen: View {
    @StateObject private var viewModel = PaywallViewModel()

    var body: some View {
        VStack {
            ForEach(viewModel.products, id: \.vendorProductId) { product in
                Button {
                    Task { await viewModel.buy(product) }
                } label: {
                    HStack {
                        Text(product.localizedTitle)
                        Spacer()
                        Text(product.localizedPrice ?? "")
                    }
                }
            }
            Button("Restore Purchases") {
                Task { await viewModel.restore() }
            }
        }
        .task { await viewModel.load(placementId: "premium") }
    }
}
```

**UIKit:** the project's `UIViewController` keeps its existing layout. The view model is owned by the VC, `viewModel.products` is read after `await load(...)` resolves, and table/collection view data sources reload accordingly. The skill does NOT rewrite the project's UIKit layout code — only the data source layer.

## 6. The `AdaptyManager.shared.refresh()` pattern

Every successful purchase, restore, identify, or logout MUST call `AdaptyManager.shared.refresh()` so the global access-level state stays in sync with the server profile. Without this:

- The view that triggered the purchase sees the local result (good)
- Other views observing `AdaptyManager.shared.$isPremium` see stale state (bad)
- The user appears to be "non-premium" elsewhere in the app until next launch

The pattern is consistent across both paywall paths. It's worth flagging in Manual review for the user to verify: "Confirm `AdaptyManager.shared.refresh()` is called after each purchase/restore/identify/logout."

## 7. Failure modes specific to this path

1. **Placement returns no products** — `getPaywallProducts` returns `[]`. Cause: products not attached to the placement in dashboard. Check Dashboard checklist §6.
2. **`makePurchase` succeeds but `isPremium` is false** — the product is not assigned to the access level in dashboard. Check Dashboard checklist §5.
3. **App Store sandbox state stuck** — old subscriptions from previous tests interfering. Create a fresh sandbox tester.
4. **`paywall` reference discarded** — if the project saves only `products` and not the `AdaptyPaywall` object, Adapty's attribution to that paywall/placement breaks. Keep both.
5. **Async/await ordering** — calling `makePurchase` before `activate` completes throws. The `AdaptyManager.shared.isAdaptyInitialized` guard prevents this.
