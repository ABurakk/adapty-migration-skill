# Adapty Reference — Common Base Patterns

This file contains the patterns shared by **every** RevenueCat → Adapty migration regardless of paywall flavor: installation, activation, identification, the `AdaptyManager` singleton, and the strict call-ordering invariants. Path-specific patterns live in two separate files:

- `references/paywall-builder-path.md` — for projects that use RevenueCatUI (`PaywallView`, `paywallFooter`, `presentPaywallIfNeeded`) → migrate to AdaptyUI + `.paywall()` modifier
- `references/custom-paywall-path.md` — for projects with custom paywall UI and direct purchase calls → migrate to `Adapty.getPaywall` + `Adapty.getPaywallProducts` + `Adapty.makePurchase`

**Target platform:** iOS 13+ (Adapty 3.x minimum)
**Language:** Swift 5.7+ (async/await required)
**SDK version:** the current supported Adapty 3.x version, confirmed at migration time against the live Adapty iOS SDK installation docs (start from `https://adapty.io/docs/` and navigate to the iOS SDK installation page). Do not bake a version number into output without re-confirming from docs. If the runtime cannot reach the docs, emit `<VERIFY_FROM_DOCS>` as the version placeholder and surface a Manual review item.

---

## 1. Installation (Swift Package Manager)

**Step before writing any dependency code:** consult the current Adapty iOS SDK installation docs and note the version Adapty currently documents as the supported 3.x release. Use that exact version in the SPM declaration.

Then update the project's SPM configuration (`Package.swift` or `.xcodeproj`):

- **Repository:** `https://github.com/adaptyteam/AdaptySDK-iOS`
- **Version:** the version confirmed from docs above (use exact-version pinning to make the migration reproducible)
- **Products to add to the app target:**
  - `Adapty` — always
  - `AdaptyUI` — **only** if the migration includes the Paywall Builder path (see `paywall-builder-path.md`). For custom-paywall-only migrations, do NOT add AdaptyUI.

If the project currently uses CocoaPods for RevenueCat, the migration removes the RevenueCat pods. Default to switching to SPM for Adapty (recommended). If the project must stay on CocoaPods, the skill consults the current Adapty iOS SDK installation docs for the current Adapty pod names and version, then writes the equivalent `Podfile` entries — but flags in Manual review that SPM is the documented primary integration path.

## 2. Imports

Replace RevenueCat imports based on the file's paywall path:

**Paywall Builder path files:**
```swift
import Adapty
import AdaptyUI
```

**Custom paywall path files (and all non-paywall files using `Purchases`):**
```swift
import Adapty
```

`import AdaptyUI` is never a universal replacement for `import RevenueCatUI`. Adding it where it isn't needed bloats the binary and signals incorrect path detection.

## 3. Activation — must be awaited before anything else

The Adapty 3.x `activate` function is async and throws. Synchronous-looking `Adapty.activate("KEY")` (without `await`) is wrong, even though some third-party guides show it that way. The SDK enforces that **no other Adapty call may execute before activate completes**.

**Canonical pattern in `@main` struct:**

```swift
import SwiftUI
import Adapty
// import AdaptyUI  ← only if Paywall Builder path is used anywhere in the project

@main
struct YourApp: App {
    init() {
        Task {
            do {
                try await Adapty.activate("YOUR_ADAPTY_PUBLIC_KEY")
                // try await AdaptyUI.activate()   ← only on Paywall Builder migrations
                await AdaptyManager.shared.initialize()
            } catch {
                print("❌ Adapty activation failed: \(error)")
            }
        }
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

**Rules:**
- Wrap the activation sequence in `Task { ... }` because `init()` is synchronous.
- The order inside the `Task` is fixed: `Adapty.activate` → (optional) `AdaptyUI.activate` → `AdaptyManager.shared.initialize`.
- The API-key literal stays a placeholder. Never copy the RevenueCat key value over — RevenueCat keys and Adapty keys are different formats and silently reusing one breaks the integration.
- If the project needs custom configuration (log level, observer mode, custom backend URL), use the builder form. Consult the current Adapty iOS SDK configuration docs (search `https://adapty.io/docs/` for "configuring" or "AdaptyConfiguration") for the exact builder method names and pass them through:
  ```swift
  let config = AdaptyConfiguration
      .builder(withAPIKey: "YOUR_ADAPTY_PUBLIC_KEY")
      .with(logLevel: .verbose)
      // .with(observerMode: false)  // confirm exact method name against current Adapty docs
      .build()
  try await Adapty.activate(with: config)
  ```
- Any code that touches Adapty BEFORE activate completes will fail. UI that may try to show a paywall on first launch must check `AdaptyManager.shared.isAdaptyInitialized` first (see §5).

## 4. Identification — required when the project uses authenticated users

If the source RevenueCat project calls `Purchases.shared.logIn(...)` anywhere, the Adapty migration MUST insert `try await Adapty.identify(...)` at every matching call site. The same ordering rule applies: **no profile, paywall, purchase, or restore call may execute before identify completes** when the user is logged in.

**Canonical login pattern:**

```swift
func handleUserLogin(userId: String) async {
    do {
        try await Adapty.identify(userId)
        // Only after identify resolves, refresh access state:
        await AdaptyManager.shared.refresh()
    } catch {
        print("❌ Adapty.identify failed: \(error)")
    }
}
```

**Canonical logout pattern:**

```swift
func handleUserLogout() async {
    do {
        try await Adapty.logout()
        await AdaptyManager.shared.refresh()
    } catch {
        print("❌ Adapty.logout failed: \(error)")
    }
}
```

**Ordering invariant:** if a screen presenting a paywall may be reached while a user is logging in, the screen MUST await identify (or check a "user ready" flag) before calling `Adapty.getPaywall`. Showing a paywall to a not-yet-identified user attributes the eventual purchase to the wrong (anonymous) profile.

If the source project never calls logIn (anonymous-only), this entire section is skipped — Adapty's anonymous profile handling takes over automatically.

## 5. `AdaptyManager` singleton — common pattern (both paths)

Create or replace any RevenueCat-related manager (`PurchasesManager`, `RevenueCatService`, `SubscriptionService`, etc.) with this class. Port the original manager's public API surface onto it, but keep the internal structure:

```swift
import Foundation
import Adapty

@MainActor
final class AdaptyManager: ObservableObject {
    static let shared = AdaptyManager()

    @Published private(set) var isPremium: Bool = false
    @Published private(set) var isAdaptyInitialized: Bool = false

    private init() {}

    /// Called once from @main App.init's activation Task, after Adapty.activate() resolves.
    func initialize() async {
        isAdaptyInitialized = true
        await refresh()
    }

    /// Re-fetches the profile and updates access state. Call after purchase, restore, identify, or logout.
    func refresh() async {
        await checkPremiumStatus()
    }

    /// Updates `isPremium` based on the current Adapty profile.
    /// - Parameter purchasedProduct: optional vendor product ID, passed by paywall callbacks for logging.
    func checkPremiumStatus(purchasedProduct: String? = nil) async {
        do {
            let profile = try await Adapty.getProfile()
            let hasAccess = profile.accessLevels["premium"]?.isActive ?? false
            self.isPremium = hasAccess
            print("✅ Premium status: \(hasAccess)\(purchasedProduct.map { " (just purchased: \($0))" } ?? "")")
        } catch {
            print("❌ Failed to fetch Adapty profile: \(error)")
        }
    }
}
```

**Customization rules:**

- If the project's RevenueCat entitlement is called something other than `"premium"` (e.g., `"pro"`, `"unlimited_access"`, `"vip"`), replace the literal `"premium"` with that name in the `accessLevels[...]` lookup AND surface a Manual review note: "Create an access level named `<name>` in Adapty Dashboard → Access Levels (case-sensitive)."
- If the project tracks multiple entitlements (e.g., `"basic"` and `"pro"`), expand `isPremium` into separate `@Published` properties per entitlement, and update each in `checkPremiumStatus`. Flag in Manual review for user confirmation.
- The `@MainActor` annotation removes the need for `await MainActor.run { ... }` blocks inside the class — all property writes are already on main.
- Keep `print` debug lines; they're a convenient sandbox-debugging convention.

## 6. Strict ordering invariants (apply globally)

These invariants are why the SDK uses async/throws and why this skill audits every call site:

1. **`Adapty.activate` must complete before anything else.** Code that depends on Adapty must check `AdaptyManager.shared.isAdaptyInitialized` (or otherwise await the activation Task) before issuing calls.
2. **`Adapty.identify` must complete before paywall/profile/purchase/restore when auth is in play.** The migrated login flow awaits identify before any subsequent SDK call.
3. **Paywall fetch is async and may fail.** Treat `Adapty.getPaywall`, `Adapty.getPaywallProducts`, `Adapty.makePurchase`, `Adapty.restorePurchases`, and `AdaptyUI.getPaywallConfiguration` as all-or-nothing — surround them with proper `do/catch` and log failures.
4. **Paywall Builder paywalls require `hasViewConfiguration == true`.** See `paywall-builder-path.md` §3. Custom paywalls do not have this requirement.
5. **The entitlement check is always `profile.accessLevels["<name>"]?.isActive ?? false`.** Any other expression is wrong.
6. **Locale for paywall fetch is `Locale.current.identifier`.** Hardcoding "en" disables Adapty's localization layer.

## 7. What this file does NOT cover

Paywall presentation and purchase mechanics depend on which path applies — load the matching file:

- Project uses RevenueCatUI → `references/paywall-builder-path.md`
- Project uses custom paywall UI + direct purchase → `references/custom-paywall-path.md`

Other topics not covered here (consult the current Adapty docs at `https://adapty.io/docs/` when encountered):

- Attribution integration with MMP-specific sequencing (Adjust, AppsFlyer, Branch, AppMetrica) — see `revenuecat-to-adapty-map.md` §12 for the path
- Custom user attributes (`Adapty.updateProfile`)
- `AdaptyDelegate` for profile-change push events
- Observer mode (rare; the skill flags this and asks for confirmation)
- Server-side receipt validation / App Store Server Notifications — see `dashboard-checklist.md`
- Verbose logging configuration — see the builder form in §3
