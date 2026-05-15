# Paywall Builder Path — AdaptyUI + `.paywall()` modifier

Read this file when the migrating file uses RevenueCatUI APIs (`PaywallView`, `paywallFooter`, `presentPaywallIfNeeded`) or otherwise indicates the project relies on RevenueCat's Paywall Builder for paywall rendering. The Adapty equivalent is the Paywall Builder feature, rendered via `AdaptyUI.getPaywallConfiguration` and the SwiftUI `.paywall()` modifier.

Migrations of files NOT in this category go to `custom-paywall-path.md`. Do not mix patterns across files.

**Prerequisite:** the migrating project's `@main` entry point uses the activation sequence from `adapty-reference.md` §3, AND that sequence includes `try await AdaptyUI.activate()` (the AdaptyUI subsystem MUST be activated on Paywall Builder migrations).

---

## 0. Scope of this migration — what does NOT transfer

The skill migrates the **code** that fetches and renders the paywall. It does **not** migrate the visual design of your paywalls.

**You will need to rebuild your paywall design from scratch in Adapty's visual builder — RevenueCat paywall layouts do not transfer. Use a template as a starting point, or recreate your existing design.**

This applies to every Paywall Builder migration. The `.paywall()` modifier this file installs renders whatever exists in Adapty Dashboard for the given placement — so until a paywall design exists there, the modifier will hit the `hasViewConfiguration == false` fallback at runtime.

**Mandatory Manual review entry the skill always surfaces on this path:**

> "RevenueCat paywall designs do NOT transfer to Adapty. For each placement migrated here, recreate the paywall layout in Adapty Dashboard → Paywalls → visual builder. Plan time for design rebuild, not just code review."

---

## 1. AdaptyUI activation (required only on this path)

Inside the activation `Task` in the `@main` struct:

```swift
try await Adapty.activate("YOUR_ADAPTY_PUBLIC_KEY")
try await AdaptyUI.activate()
await AdaptyManager.shared.initialize()
```

`AdaptyUI.activate()` must come AFTER `Adapty.activate()` and BEFORE any `AdaptyUI.getPaywallConfiguration` call. Without it the paywall configuration fetch errors out.

## 2. Paywall presentation — canonical SwiftUI pattern

The pattern below replaces `PaywallView(offering:)` / `presentPaywallIfNeeded` / `paywallFooter`. Adapt to the user's view structure, but keep the `.paywall()` modifier, the seven callbacks, the `loadPaywallAndShow()` function, AND the `hasViewConfiguration` guard inside the loader (§3).

```swift
import SwiftUI
import Adapty
import AdaptyUI

struct YourView: View {
    @State private var showingPaywall = false
    @State private var paywallConfiguration: AdaptyUI.PaywallConfiguration?
    @ObservedObject private var adaptyManager = AdaptyManager.shared

    var body: some View {
        VStack {
            if adaptyManager.isPremium {
                Text("Premium Features Unlocked! ✨")
            } else {
                Button("Get Premium") {
                    Task { await loadPaywallAndShow() }
                }
            }
        }
        .paywall(
            isPresented: $showingPaywall,
            paywallConfiguration: paywallConfiguration,
            didPerformAction: { action in
                switch action {
                case .close:
                    showingPaywall = false
                default:
                    break
                }
            },
            didFinishPurchase: { product, profile in
                print("✅ Purchase completed: \(product.vendorProductId)")
                Task { await adaptyManager.checkPremiumStatus(purchasedProduct: product.vendorProductId) }
                showingPaywall = false
            },
            didFailPurchase: { product, error in
                print("❌ Purchase failed: \(error.localizedDescription)")
            },
            didFinishRestore: { profile in
                print("✅ Restore completed")
                Task { await adaptyManager.checkPremiumStatus() }
                showingPaywall = false
            },
            didFailRestore: { error in
                print("❌ Restore failed: \(error.localizedDescription)")
            },
            didFailRendering: { error in
                print("❌ Paywall rendering failed: \(error.localizedDescription)")
                showingPaywall = false
            }
        )
    }

    private func loadPaywallAndShow() async {
        guard adaptyManager.isAdaptyInitialized else {
            print("⚠️ Adapty not yet initialized — aborting paywall load")
            return
        }

        do {
            let paywall = try await Adapty.getPaywall(
                placementId: "premium",
                locale: Locale.current.identifier
            )

            // CRITICAL: Paywall Builder paywalls only render when the dashboard's
            // "Show on device" toggle is enabled for this placement. Without a
            // view configuration, AdaptyUI cannot render — fall back gracefully.
            guard paywall.hasViewConfiguration else {
                print("⚠️ Paywall '\(paywall.placement.id)' has no view configuration. " +
                      "Enable 'Show on device' in Adapty Dashboard for this placement, " +
                      "or use the custom paywall path for this screen.")
                // Optional: surface a fallback UI / sheet here.
                return
            }

            let products = try await Adapty.getPaywallProducts(paywall: paywall)

            let configuration = try await AdaptyUI.getPaywallConfiguration(
                forPaywall: paywall,
                loadTimeout: 10.0,
                products: products
            )

            self.paywallConfiguration = configuration
            self.showingPaywall = true
        } catch {
            print("❌ Failed to load paywall: \(error)")
        }
    }
}
```

**Customization rules:**

- Substitute `YourView` with the actual SwiftUI view name from the user's project.
- Substitute `"premium"` in `placementId: "premium"` with the user's RevenueCat offering name carried over (and flag it in Manual review as offering↔placement is not guaranteed semantic equivalence).
- Substitute the placeholder UI strings (`"Premium Features Unlocked! ✨"`, `"Get Premium"`) with whatever the original RevenueCat view had — preserve the project's copy.
- KEEP `loadTimeout: 10.0`. The reviewer's projects converge on this; deviating without reason invites flaky paywalls on slow networks.
- KEEP `locale: Locale.current.identifier`. Hardcoding a language disables Adapty's localization for paywall content.
- KEEP all seven callbacks even if some are no-ops in the user's original code. The full callback surface is the canonical convention; missing callbacks mean missing failure logging.

## 3. The `hasViewConfiguration` guard — non-negotiable

`paywall.hasViewConfiguration` is `true` only when the placement's paywall in Adapty Dashboard has the **"Show on device"** toggle enabled. This is the Paywall Builder rendering flag; without it, `AdaptyUI.getPaywallConfiguration` throws or returns an unrenderable configuration.

**Why the guard is non-optional:**

- The migration cannot detect from the project alone whether the user will enable "Show on device" in the dashboard — that's a manual dashboard step
- A paywall that loads server-side but has `hasViewConfiguration == false` produces silent UI failure: the user taps "Get Premium", nothing happens, no error visible, no purchase ever attempted
- The guard converts that silent failure into a logged warning, giving the user a debuggable signal

**What goes in the fallback branch:**

Three reasonable options, in increasing complexity:

1. **Just log + return.** Acceptable for the migration's first pass; the user notices when their paywall doesn't show and fixes the dashboard.
2. **Show a generic "Premium Features" sheet with static product info.** Useful if the project must always be able to monetize; usually overkill for migration v1.
3. **Fall back to the custom paywall path for this screen.** Most robust — but requires the file to be migrated as custom-paywall too. Flag in Manual review if this hybrid is needed.

The skill's default is option 1 (log + return). It surfaces a Manual review item: "File `<path>:<line>` falls back to no-op if `hasViewConfiguration` is false. Enable 'Show on device' in Adapty Dashboard for placement `<id>`."

## 4. Embedded paywall (`paywallFooter` equivalent)

RevenueCat's `paywallFooter` embeds the paywall inline within a parent view. Adapty supports this via the same `.paywall()` modifier — but the paywall in Adapty Dashboard's visual builder must be configured as **embedded** (not full-screen). Code-side, no change.

Flag in Manual review: "File `<path>` used `paywallFooter`. Configure the paywall in Adapty Dashboard as embedded for placement `<id>` (visual builder → Layout → Embedded). The same `.paywall()` modifier handles both layouts."

## 5. Failure modes specific to this path

When this path goes wrong, it usually goes wrong silently. The most common causes:

1. **"Show on device" disabled** — `hasViewConfiguration` is false; the guard logs but UI does nothing.
2. **`AdaptyUI.activate()` skipped** — `getPaywallConfiguration` fails. Verify the activation sequence.
3. **Placement ID mismatch** — code says `"premium"`, dashboard placement is `"Premium"`. Case-sensitive.
4. **Paywall draft, not published** — `getPaywall` returns no view config. Publish in dashboard.
5. **Products not assigned to placement** — `getPaywallProducts` returns empty. Add products in dashboard.
6. **Locale not supported** — paywall content shows in a fallback language; not a crash but a UX bug. Add localizations in Adapty Dashboard for the locales the project ships in.

The skill surfaces #1, #3, #4, #5 in the Dashboard checklist for the user to verify before testing.
