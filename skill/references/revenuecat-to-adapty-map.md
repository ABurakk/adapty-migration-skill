# RevenueCat → Adapty API Mapping

This file is the lookup table the skill consults for each RevenueCat construct found in the project. Each entry tells the skill:

- The before (RevenueCat) and after (Adapty) code
- Which reference file the Adapty equivalent lives in (`adapty-reference.md`, `paywall-builder-path.md`, or `custom-paywall-path.md`)
- Whether the signature should be verified against the current Adapty docs for the current SDK version
- Notes about gotchas the skill must surface in Manual review

**Important:** the skill never bakes a specific Adapty SDK version into output without first verifying it against the current Adapty iOS SDK installation docs (start from `https://adapty.io/docs/`) at migration time. The patterns below describe API *shape* (which methods, in what order); exact *signatures* (parameter names, return types) MUST be verified against current docs. If the runtime cannot reach the docs, flag every emitted signature in Manual review as unverified and emit `<VERIFY_FROM_DOCS>` where a version number would go.

**Legend**

- ✅ **Reference-canonical** — pattern is in one of the reference files; use it verbatim. Cite the file + section.
- 🌐 **Verify signature** — the pattern shape is in the reference, but verify exact parameter names / return types against current Adapty docs at migration time. Cite the verified URL + today's date.
- 🌐🌐 **Verify first** — pattern is NOT in any reference; consult current Adapty docs for the concept before writing code.
- ⚠️ **Manual review item** — every verified mapping AND every offering↔placement mapping is automatically added to Manual review.
- 🅿️🅱️ **Paywall Builder path only** — applies only to files migrating via `paywall-builder-path.md`
- 🅒🅟 **Custom paywall path only** — applies only to files migrating via `custom-paywall-path.md`

---

## 1. Dependencies (Podfile / Package.swift / Cartfile)

### CocoaPods (`Podfile`)

**Before:**
```ruby
pod 'RevenueCat'
pod 'RevenueCatUI' # if present
```

**After:** Remove the RevenueCat pods. Switch the project to SPM for Adapty per `adapty-reference.md` Installation, OR if the project must stay on CocoaPods, replace with Adapty's pods. CocoaPods support for Adapty exists but pod names should be confirmed against current Adapty iOS SDK installation docs (`https://adapty.io/docs/`). Default: prefer SPM and add a Manual review note: "Removed RevenueCat pods. The Adapty integration uses SPM (see Xcode project). If you must stay on CocoaPods, consult `https://adapty.io/docs/` for current Adapty pod names."

### Swift Package Manager (`Package.swift` or `.xcodeproj`)

**Before:** Dependency entry pointing at `https://github.com/RevenueCat/purchases-ios`

**After:** First consult current Adapty iOS SDK installation docs (`https://adapty.io/docs/`) to get the current supported 3.x version. Then replace with (substituting `<VERSION>` with the value from docs, or `<VERIFY_FROM_DOCS>` if docs are unreachable):
```swift
.package(url: "https://github.com/adaptyteam/AdaptySDK-iOS", exact: "<VERSION>")
```
Add products to the app target:
- `"Adapty"` — always
- `"AdaptyUI"` — **only** if at least one file in the project migrates via the Paywall Builder path. For custom-paywall-only migrations, do NOT add AdaptyUI.

In `project.pbxproj`, the skill updates the package references in `XCRemoteSwiftPackageReference`, `XCSwiftPackageProductDependency`, and the target's `packageProductDependencies` build phase entries. If unsure about pbxproj edits, the skill leaves the existing file alone and adds a Manual review entry instructing the user to remove the RevenueCat package in Xcode (File → Packages) and add Adapty manually with the version from docs.

### Carthage (`Cartfile`)

Adapty does not officially distribute via Carthage in 3.x. If the project uses Carthage, flag in Manual review: "RevenueCat was integrated via Carthage. Adapty 3.x officially supports SPM (recommended) and CocoaPods. Migrate the project to SPM for Adapty — consult `https://adapty.io/docs/` for current installation guidance."

---

## 2. Imports — path-aware ✅

**Before:**
```swift
import RevenueCat
import RevenueCatUI  // only in some files
```

**After — depends on the file's paywall path:**

For files migrating via the Paywall Builder path (🅿️🅱️):
```swift
import Adapty
import AdaptyUI
```

For files migrating via the custom paywall path (🅒🅟), AND any file using `Purchases` without paywall UI:
```swift
import Adapty
```

**Critical:** `import RevenueCatUI` → `import AdaptyUI` is NOT a universal find/replace. Files that used RevenueCat APIs without RevenueCatUI must NOT gain an `import AdaptyUI` line. Doing so signals incorrect path detection and bloats the binary with an unused module.

---

## 3. SDK Initialization ✅

**Before (closure or sync style):**
```swift
Purchases.logLevel = .debug
Purchases.configure(withAPIKey: "rc_xxxxxxxxxxxx")
// or builder form:
Purchases.configure(with: Configuration.Builder(withAPIKey: "rc_xxxxxxxxxxxx").build())
```

**After (`adapty-reference.md` §3) — awaited inside a Task:**
```swift
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
```

**Rules:**
- `Adapty.activate` is async/throws — wrap in `Task` because `@main`'s `init()` is synchronous.
- **No other Adapty call may execute before `try await Adapty.activate(...)` completes.** This is a hard SDK requirement; calls fired in parallel will fail.
- Include `try await AdaptyUI.activate()` ONLY if the project has at least one Paywall Builder path file. Otherwise omit it.
- The API key literal stays `"YOUR_ADAPTY_PUBLIC_KEY"` — never carry the RevenueCat key value across (different formats; silent failure).
- For verbose logging or custom configuration, use the builder form. 🌐 Consult current Adapty docs (search `https://adapty.io/docs/` for "configuring" or "AdaptyConfiguration") for the exact `AdaptyConfiguration.builder(...)` method names — these can shift between minor SDK versions:
  ```swift
  let config = AdaptyConfiguration
      .builder(withAPIKey: "YOUR_ADAPTY_PUBLIC_KEY")
      .with(logLevel: .verbose)
      .build()
  try await Adapty.activate(with: config)
  ```

---

## 4. Customer info / profile fetch ✅

**Before (closure):**
```swift
Purchases.shared.getCustomerInfo { customerInfo, error in
    guard let info = customerInfo else { return }
    // use info
}
```

**Before (async):**
```swift
let customerInfo = try await Purchases.shared.customerInfo()
```

**After (`adapty-reference.md` §5):**
```swift
let profile = try await Adapty.getProfile()
```

If the original closure-based code was called from a synchronous context, wrap the Adapty call in a `Task { ... }`. Confirm `Adapty.activate()` has completed first — see `adapty-reference.md` §6 ordering invariants.

---

## 5. Entitlement / access-level check ✅

**Before:**
```swift
let isPro = customerInfo.entitlements["pro"]?.isActive == true
// or
let isPro = customerInfo.entitlements.active["pro"] != nil
// or
if let entitlement = customerInfo.entitlements["pro"], entitlement.isActive { ... }
```

**After (`adapty-reference.md` §5):**
```swift
let isPro = profile.accessLevels["pro"]?.isActive ?? false
```

**Rule:** carry over the entitlement name (`"pro"`, `"premium"`, `"vip"`, whatever the project uses) unchanged. RevenueCat entitlements and Adapty access levels are semantically very similar (both are server-side capability flags), so a 1:1 rename is *usually* safe — but still flag in the Dashboard checklist: "Create an access level named `pro` in Adapty Dashboard → Access Levels (case-sensitive, must match the entitlement name from your old RevenueCat setup)."

---

## 6. Offerings → Paywall fetch — path-aware ⚠️

**Before:**
```swift
Purchases.shared.getOfferings { offerings, error in
    if let offering = offerings?.current {
        // use offering.availablePackages
    }
}
// or async:
let offerings = try await Purchases.shared.offerings()
```

**After — depends on the file's paywall path:**

🅿️🅱️ For Paywall Builder path files, this fetch is wrapped inside `loadPaywallAndShow()` in `paywall-builder-path.md` §2. The fetch + render is one operation.

🅒🅟 For custom paywall path files, the fetch is a two-call sequence in a view model — see `custom-paywall-path.md` §2:
```swift
let paywall = try await Adapty.getPaywall(
    placementId: "premium",  // ← see warning below
    locale: Locale.current.identifier
)
let products = try await Adapty.getPaywallProducts(paywall: paywall)
```

**Type mapping:**
- RevenueCat `Offering` → Adapty `AdaptyPaywall`
- RevenueCat `Offerings` (the collection) → no direct equivalent; Adapty fetches per-placement
- RevenueCat `offering.availablePackages` (array of `Package`) → Adapty `[AdaptyPaywallProduct]` returned from `getPaywallProducts`
- RevenueCat `Package` → Adapty `AdaptyPaywallProduct`

**⚠️ Offering ↔ Placement is NOT a guaranteed 1:1 rename — Manual review item:**

RevenueCat's `Offering` and Adapty's `Placement` are *related* concepts but they have different semantic surfaces:

- A RevenueCat **Offering** bundles a set of *packages* (products) under a name. The app picks a specific offering and renders all its packages on one paywall.
- An Adapty **Placement** is an "ad slot" or display location. A placement has **one** paywall assigned to it, and that paywall in turn has its own product set assigned in the dashboard. Different placements can serve different paywalls (used for A/B testing, segmentation, contextual paywalls).

So:
- `RevenueCat offering "premium" → Adapty placement "premium"` is a reasonable starting point IF the project only has one paywall surface
- But if the project has multiple offerings used for different surfaces (e.g., onboarding vs settings), each needs its own placement AND its own paywall in Adapty Dashboard
- And if the project has one offering shown in many places, that may map to one placement OR multiple placements pointing at the same paywall

The skill carries the name over verbatim AND adds a Manual review entry: "RevenueCat offering `<name>` was carried over as Adapty placement `<name>`. These concepts are related but not guaranteed 1:1. In the Adapty Dashboard, create a placement with this ID, attach a paywall to it, and confirm the products map to the same set your old RevenueCat offering exposed. If the project uses multiple offerings in different contexts, you may need multiple Adapty placements — review per call site."

---

## 7. Paywall UI presentation ✅

## 7. Paywall UI presentation — strictly path-aware

### 🅿️🅱️ From RevenueCatUI's `PaywallView` / `presentPaywallIfNeeded` — Paywall Builder path

**Before:**
```swift
.sheet(isPresented: $showingPaywall) {
    PaywallView(offering: currentOffering)
}
// or
.presentPaywallIfNeeded(requiredEntitlementIdentifier: "pro")
```

**After:** Use the SwiftUI `.paywall()` modifier with all seven callbacks AND the `hasViewConfiguration` guard. See the full template in `paywall-builder-path.md` §2. Mandatory elements:

- `try await AdaptyUI.activate()` in the app's activation sequence
- `paywall.hasViewConfiguration` guard before calling `AdaptyUI.getPaywallConfiguration` (silent UI failure if missing — see `paywall-builder-path.md` §3)
- All seven callbacks present: `didPerformAction`, `didFinishPurchase`, `didFailPurchase`, `didFinishRestore`, `didFailRestore`, `didFailRendering`
- The `loadPaywallAndShow()` helper that performs the fetch + render

⚠️ Add to Manual review: "Enable 'Show on device' in Adapty Dashboard → Paywalls for placement `<id>`. Without this, `hasViewConfiguration` returns false and the migrated paywall renders nothing at runtime."

### 🅿️🅱️ From RevenueCatUI's `paywallFooter` — embedded Paywall Builder

**Before:**
```swift
.paywallFooter(offering: currentOffering)
```

**After:** Same `.paywall()` modifier from `paywall-builder-path.md` §2; the embedded layout is configured in the Adapty Dashboard visual builder (paywall → Layout → Embedded), not in code.

⚠️ Manual review item: "Configure paywall for placement `<id>` as embedded in Adapty Dashboard (visual builder → Layout → Embedded)."

### 🅒🅟 From a custom UI + `Purchases.shared.purchase(package:)` — custom paywall path

**Before:**
```swift
// Custom SwiftUI or UIKit paywall view
struct MyPaywallView: View {
    var body: some View {
        Button("Subscribe") {
            Purchases.shared.purchase(package: package) { ... }
        }
    }
}
```

**After:** Keep the custom view. Replace the data source and purchase call with patterns from `custom-paywall-path.md`:
- `Adapty.getPaywall` + `Adapty.getPaywallProducts` for product loading (§2)
- `Adapty.makePurchase(product:)` for the buy action (§3)
- `Adapty.restorePurchases()` for restore (§4)
- Do NOT `import AdaptyUI` in custom paywall files
- `hasViewConfiguration` is NOT checked on this path

### 🌐🌐 From UIKit hosting

If the project hosts the paywall in UIKit (UIViewController/UINavigationController), behavior depends on path:

- 🅿️🅱️ **UIKit + Paywall Builder:** consult current Adapty docs (search `https://adapty.io/docs/` for "Paywall Builder" or "AdaptyUI") for the current UIKit hosting controller API (`AdaptyPaywallController` or similar — exact name varies by SDK minor version). Leave a `// TODO: ADAPTY MANUAL — UIKit Paywall Builder hosting: see <URL>` if signatures don't match the canonical SwiftUI flow.
- 🅒🅟 **UIKit + custom paywall:** the migration is mostly mechanical — view model uses `custom-paywall-path.md` patterns, UIKit view controller binds to it the same way it bound to RevenueCat data. No special UIKit-specific Adapty API needed.

---

## 8. Direct purchase

**Before:**
```swift
Purchases.shared.purchase(package: pkg) { transaction, customerInfo, error, userCancelled in
    if customerInfo?.entitlements["pro"]?.isActive == true {
        // unlock
    }
}
// or async:
let result = try await Purchases.shared.purchase(package: pkg)
```

**After — path-dependent:**

🅿️🅱️ For Paywall Builder migrations, the purchase happens inside the `.paywall()` modifier — the `didFinishPurchase` callback is the entry point. Migrate the post-purchase logic into that callback. See `paywall-builder-path.md` §2.

🅒🅟 For custom paywall migrations, use `Adapty.makePurchase(product:)` directly. See `custom-paywall-path.md` §3 for the full pattern. The return type is **`AdaptyPurchaseResult`, an enum with three cases**: `.userCancelled`, `.pending`, `.success(profile:transaction:)`. User cancellation is NOT a thrown error — it is the `.userCancelled` case. The `.pending` case represents legitimate intermediate StoreKit states (Ask to Buy, SCA, deferred) and must NOT be treated as failure. 🌐 Verify case names against current Adapty docs (search `https://adapty.io/docs/` for "makePurchase" / "AdaptyPurchaseResult") at migration time.

```swift
let result = try await Adapty.makePurchase(product: product) // product: AdaptyPaywallProduct

switch result {
case .userCancelled:
    return
case .pending:
    // show a pending UI; AdaptyDelegate.didLoadLatestProfile will fire when resolved
    return
case let .success(profile, _):
    let isPremium = profile.accessLevels["premium"]?.isActive ?? false
    // ...
}
```

⚠️ Always add to Manual review: "Direct purchase call at `<file:line>` — `AdaptyPurchaseResult` enum cases verified against current Adapty docs on <date> (cite the verified URL). Confirm `.userCancelled` / `.pending` / `.success` against your installed Adapty SDK version. After successful purchase, ensure `AdaptyManager.shared.refresh()` is called so global state updates. **Do NOT treat `.pending` as failure** — that silently drops valid Ask-to-Buy / SCA purchases."

---

## 9. Restore purchases

**Before:**
```swift
Purchases.shared.restorePurchases { customerInfo, error in ... }
// or async:
let customerInfo = try await Purchases.shared.restorePurchases()
```

**After — path-dependent:**

🅿️🅱️ For Paywall Builder migrations, the restore happens inside the `.paywall()` modifier — the `didFinishRestore` callback is the entry point. Migrate any post-restore logic into that callback. See `paywall-builder-path.md` §2.

🅒🅟 For custom paywall migrations (and any standalone "Restore Purchases" button outside a paywall):

```swift
let profile = try await Adapty.restorePurchases()
let isPremium = profile.accessLevels["premium"]?.isActive ?? false
await AdaptyManager.shared.refresh()
```

🌐 Consult current Adapty docs (search `https://adapty.io/docs/` for "making purchases" or "restorePurchases") for the exact return type and error semantics; the return type may be wrapped in a result type depending on SDK version.

⚠️ Manual review: "Restore call at `<file:line>` — signature verified against current Adapty docs on <date> (cite the verified URL). Confirm against your installed SDK."

---

## 10. User identification ✅ (with ordering invariant)

**Before:**
```swift
Purchases.shared.logIn("user_id_123") { customerInfo, created, error in ... }
// or async:
let result = try await Purchases.shared.logIn("user_id_123")
```

**After (`adapty-reference.md` §4):**
```swift
try await Adapty.identify("user_id_123")
// After identify resolves:
await AdaptyManager.shared.refresh()
```

**Critical ordering rules:**

1. `Adapty.activate()` MUST complete before `Adapty.identify()`. Both go through the activation Task in `@main` or the auth flow — never fire identify before activate is awaited.
2. `Adapty.identify()` MUST complete before any subsequent `Adapty.getPaywall`, `getProfile`, `makePurchase`, or `restorePurchases` call. Race conditions here attribute purchases to the anonymous profile, which is a silent data-quality bug, not a crash.
3. If the project has a screen that may present a paywall while login is in progress, the screen must await identify completion (or check a "user ready" flag) before fetching the paywall.

If the project never calls logIn (anonymous-only), skip this section entirely — Adapty's anonymous profile handling takes over.

**Profile semantics differ from RevenueCat — critical for migration correctness:**

RevenueCat and Adapty handle repeated identify and account switching differently. Code that depends on RC's exact semantics may behave unexpectedly on Adapty:

- **RevenueCat:** `logIn("A")` → `logIn("B")` creates a new profile for `"B"`. Switching back to `"A"` with `logIn("A")` returns to the original profile. RevenueCat also merges some profile data across these transitions.
- **Adapty:** the SDK has a separate concept of `profile_id` (an internal Adapty ID) and `customer_user_id` (your app's user ID). Calling `Adapty.identify("B")` on an anonymous profile **links that customer ID to the current profile** rather than creating a fresh one. If `"B"` already exists on Adapty's side, the SDK switches to the existing profile. **Adapty does NOT merge profile data across these transitions.**

**Practical implications the skill MUST flag:**

- If the original RC code relies on `logIn("A")` → `logIn("B")` creating two distinct profiles (e.g., for multi-account apps, family-sharing-style switching), the equivalent Adapty migration should call `Adapty.logout()` between the two identify calls. The skill MUST detect any consecutive `logIn` calls without an intervening `logOut` and flag this as a behavior change.
- If your app re-sends user attributes (email, phone, custom attributes) on every login, you should continue doing so after migration — Adapty does not carry these across profile transitions. (See the warning in current Adapty docs about resubmitting significant user data.)

⚠️ Always Manual review:

- "User identification migrated based on `adapty-reference.md` §4. Verify the customer ID format your backend uses matches what Adapty stores (Adapty preserves the string verbatim; some backends pass UUIDs, some pass internal IDs)."
- "**Profile semantics differ from RevenueCat:** Adapty does not merge profile data across identify/logout transitions, and repeated `identify` on the same profile does not create a new profile. If your RC code relied on `logIn(A)` → `logIn(B)` creating distinct profiles, insert an `Adapty.logout()` between them. Re-send user attributes (email, phone, custom attributes) after every identify if your app relied on RC's data-merging behavior."

---

## 11. Logout ✅

**Before:**
```swift
Purchases.shared.logOut { customerInfo, error in ... }
// or async:
try await Purchases.shared.logOut()
```

**After (`adapty-reference.md` §4):**
```swift
try await Adapty.logout()
await AdaptyManager.shared.refresh()
```

After logout, the SDK reverts to an anonymous profile. Any access-level state for the previous user is cleared — the migration's `AdaptyManager.shared.refresh()` call propagates this to UI.

**Migration caveat — opt-out apps:** if the original RC project deliberately avoided `logOut` to prevent creating an anonymous profile (some apps prefer to keep the user signed-in identifier permanently and never go anonymous), preserve that behavior on the Adapty side — do not insert `Adapty.logout()` calls that weren't in the original code. Adapty also creates a fresh anonymous profile on logout; calling it unnecessarily is a behavior change.

⚠️ Manual review: "Logout migrated. Adapty's logout reverts to a fresh anonymous profile, same as RC's. If your app deliberately avoids logout to keep a persistent identity, the migration preserved that behavior — verify no unintended `Adapty.logout()` calls were introduced."

---

## 12. Attribution / integration identifiers 🌐 ⚠️

RevenueCat exposes one method per MMP / analytics integration (`setAdjustID`, `setAppsflyerID`, etc.). Adapty unifies these behind a single API: **`Adapty.setIntegrationIdentifier(key:value:)`**. This is the correct migration for the *ID setter* family. The separate `Adapty.updateAttribution(...)` API also exists but is reserved for uploading a full attribution payload from a provider's callback — it is NOT what `setAdjustID` and friends map to.

**Before (RevenueCat ID setters):**
```swift
Purchases.shared.attribution.setAdjustID("<ADJUST_ID>")
Purchases.shared.attribution.setAppsflyerID("<APPSFLYER_ID>")
Purchases.shared.attribution.setFBAnonymousID("<FB_ANONYMOUS_ID>")
Purchases.shared.attribution.setOnesignalID("<ONESIGNAL_ID>")
Purchases.shared.attribution.setMixpanelDistinctID("<MIXPANEL_ID>")
Purchases.shared.attribution.setFirebaseAppInstanceID("<FIREBASE_ID>")
Purchases.shared.attribution.setTenjinAnalyticsInstallationID("<TENJIN_ID>")
Purchases.shared.attribution.setPostHogUserID("<POSTHOG_USER_ID>")
```

**After (Adapty integration identifiers) — confirm key strings against current docs:**
```swift
try await Adapty.setIntegrationIdentifier(key: "adjust_device_id", value: "<ADJUST_ID>")
try await Adapty.setIntegrationIdentifier(key: "appsflyer_id", value: "<APPSFLYER_ID>")
try await Adapty.setIntegrationIdentifier(key: "facebook_anonymous_id", value: "<FB_ANONYMOUS_ID>")
try await Adapty.setIntegrationIdentifier(key: "one_signal_subscription_id", value: "<ONESIGNAL_ID>")
try await Adapty.setIntegrationIdentifier(key: "mixpanel_user_id", value: "<MIXPANEL_ID>")
try await Adapty.setIntegrationIdentifier(key: "firebase_app_instance_id", value: "<FIREBASE_ID>")
try await Adapty.setIntegrationIdentifier(key: "tenjin_analytics_installation_id", value: "<TENJIN_ID>")
try await Adapty.setIntegrationIdentifier(key: "posthog_distinct_user_id", value: "<POSTHOG_USER_ID>")
```

**Ordering:** call `setIntegrationIdentifier` AFTER `Adapty.activate()` has completed, and before any user-action methods (paywall fetch, purchase). Per Adapty's call-order recommendations, integration identifiers should be set as part of the post-activation setup.

**Providers Adapty does NOT yet support (as of current Adapty docs — re-verify):**

The skill MUST flag these as unsupported and leave the original RC call as a `// FIXME: ADAPTY MIGRATION BLOCKED — Adapty does not yet support <provider>` if the project uses any of them:

- mParticle (`Purchases.shared.attribution.setMparticleID`)
- Airship (`Purchases.shared.attribution.setAirshipChannelID`)
- CleverTap (`Purchases.shared.attribution.setCleverTapID`)
- Kochava (`Purchases.shared.attribution.setKochavaDeviceID`)

For these, the migration cannot proceed — surface a top-level Manual review entry: "Project uses `<provider>`, which Adapty does not yet integrate with. The original RevenueCat call was left in place with a FIXME comment. Track Adapty's integrations roadmap for support."

**UTM tags (mediaSource / campaign / adGroup / ad / keyword / creative):**

RevenueCat's `setMediaSource` / `setCampaign` / `setAdGroup` etc. have no direct equivalent in Adapty's integration-identifier API. Flag in Manual review: "Project sets UTM-style attribution tags via RevenueCat's `setMediaSource` / `setCampaign` / etc. Adapty does not expose a direct equivalent setter; these values are typically delivered through provider integrations (Adjust, AppsFlyer) rather than set manually. Review whether these tags are still needed after the MMP integration is wired up."

**APNS push tokens:**

RevenueCat's `setPushToken` / `setPushTokenString` have no direct Adapty equivalent. Flag in Manual review.

### 12b. Full attribution payload uploads — `Adapty.updateAttribution(...)`

This is a SEPARATE API from `setIntegrationIdentifier`. It is used when a provider (typically Adjust or AppsFlyer) delivers a complete attribution payload via its own callback, and you want to forward that payload to Adapty.

🌐 Always consult the provider-specific Adapty docs (search `https://adapty.io/docs/` for the provider name) for the exact callback to hook into, the `Adapty.updateAttribution` overload to call, and the `source` enum value. These shapes shift between SDK versions.

**General shape (verify against current docs):**

```swift
// In your Adjust/AppsFlyer attribution callback:
func attributionCallback(payload: [AnyHashable: Any]) {
    Task {
        do {
            try await Adapty.updateAttribution(
                payload,
                source: .adjust  // or .appsflyer — match current docs
            )
        } catch {
            print("❌ Adapty.updateAttribution failed: \(error)")
        }
    }
}
```

Do not call `updateAttribution` directly after `Adapty.activate` with an empty payload — it produces silent attribution gaps.

**Manual review entries (always added when attribution code is detected):**

- "Attribution ID setters migrated to `Adapty.setIntegrationIdentifier(key:value:)`. The key strings (`adjust_device_id`, `appsflyer_id`, etc.) were chosen to match current Adapty docs as of `<date>` — re-verify against the current docs if your SDK version is newer."
- "Configure each `<provider>` integration in Adapty Dashboard → Integrations — without dashboard configuration, integration identifiers and attribution payloads are dropped server-side."
- "If the project also forwards a full attribution payload via `updateAttribution(...)`, verify the source enum value and payload shape against current Adapty provider-specific docs."
- "If using authenticated users with attribution, ensure `Adapty.identify(...)` completes before the provider attribution callback fires, so the attribution data is bound to the right Adapty profile."

---

## 13. Custom user attributes 🌐

**Before:**
```swift
Purchases.shared.attribution.setAttributes(["email": "user@example.com"])
// or
Purchases.shared.attribution.setEmail("user@example.com")
```

**After:** consult current Adapty docs (search `https://adapty.io/docs/` for "user attributes" or "updateProfile"). Expected shape uses `AdaptyProfileParameters.Builder`:
```swift
let builder = AdaptyProfileParameters.Builder()
    .with(email: "user@example.com")
    // .with(customAttributes: [...])
let params = builder.build()
try await Adapty.updateProfile(params: params)
```
Confirm exact builder method names against the fetched docs. Add to Manual review.

---

## 14. PurchasesDelegate → AdaptyDelegate 🌐

**Before:**
```swift
Purchases.shared.delegate = self
extension MyClass: PurchasesDelegate {
    func purchases(_ purchases: Purchases, receivedUpdated customerInfo: CustomerInfo) {
        // react to subscription changes
    }
}
```

**After:** Adapty exposes `AdaptyDelegate` for profile-change events. Verify exact callback signatures against current Adapty docs (search `https://adapty.io/docs/` for "AdaptyDelegate" / "listen events"). Expected shape:

```swift
final class SubscriptionManager: AdaptyDelegate {
    nonisolated func didLoadLatestProfile(_ profile: AdaptyProfile) {
        // react to profile changes
        // NOTE: this callback is `nonisolated` — hop to a main-actor
        // context before touching @MainActor state.
    }
}

// Hold the delegate in a long-lived owner — do NOT assign a throwaway
// instance. The SDK does not retain the delegate; if no one else owns
// it, it will be deallocated and the callback will silently stop firing.
let subscriptionManager = SubscriptionManager()
Adapty.delegate = subscriptionManager
```

**Migration rules:**

- `AdaptyDelegate` callbacks are declared `nonisolated` in the current SDK. Mark your implementation `nonisolated` to match the protocol and avoid actor-isolation warnings under Swift 6.
- Hold the delegate instance somewhere with a lifetime that matches the app (e.g., an `AdaptyManager` singleton property, or a `let` on `AppDelegate`/`@main` struct). Do not write `Adapty.delegate = SubscriptionManager()` — the instance has no owner and will be deallocated.
- If the original RC `PurchasesDelegate` was implemented on a view controller or other short-lived object, migrate the implementation onto the long-lived `AdaptyManager` (or equivalent service class) instead of preserving the original ownership.

⚠️ Manual review: "AdaptyDelegate implementation at `<file:line>` migrated from `PurchasesDelegate`. (a) Verify the callback signature against current Adapty docs. (b) Confirm the delegate instance is held by a long-lived owner — the SDK does not retain it. (c) Confirm `nonisolated` matches the current protocol declaration in your installed SDK version."

---

## 14a. Promotional & win-back offers ⚠️

RevenueCat lets you explicitly select a promotional or win-back offer at purchase time:

```swift
// RevenueCat — explicit promo offer
let promoOffers = await package.storeProduct.eligiblePromotionalOffers()
guard let promoOffer = promoOffers.first else { return }
try await Purchases.shared.purchase(package: package, promotionalOffer: promoOffer)

// RevenueCat — explicit win-back offer
let winBackOffers = try await Purchases.shared.eligibleWinBackOffers(forPackage: package)
let params = PurchaseParams.Builder(package: package)
    .with(winBackOffer: winBackOffers.first!)
    .build()
try await Purchases.shared.purchase(params)
```

**Adapty applies eligible offers automatically** — there is no API to pass a specific promo or win-back offer to `Adapty.makePurchase`. Eligible offers are attached to products in **Adapty Dashboard → Products → (product) → Offers** (or the equivalent paywall configuration), and the SDK applies them when the user qualifies.

**Migration action:**

1. **Remove the offer-selection code.** Any `eligiblePromotionalOffers()` / `eligibleWinBackOffers(...)` call and the `.with(promotionalOffer:)` / `.with(winBackOffer:)` builder chain becomes dead code.
2. **Replace the purchase call** with the standard `Adapty.makePurchase(product:)` pattern (see §8). The offer will be applied automatically if the user is eligible.
3. **Surface the dashboard step in Manual review.** Without dashboard configuration, no offer is applied.

**To detect eligibility for UI hints** (e.g., "Trial available" badge), check `product.subscriptionOffer` on the `AdaptyPaywallProduct`. If non-nil, an offer is attached and will be applied at purchase time. RevenueCat's `checkTrialOrIntroDiscountEligibility(product:)` maps to this property check.

⚠️ Manual review (always added when promo / win-back code is detected):

- "Removed RevenueCat's explicit promotional-offer selection at `<file:line>`. Adapty applies eligible offers automatically based on dashboard configuration. **Action required:** in Adapty Dashboard → Products, attach the equivalent promotional offer(s) to each product. Without this, no offer will be applied at purchase."
- "Removed RevenueCat's explicit win-back-offer selection at `<file:line>`. Same applies — configure win-back offers per product in Adapty Dashboard."
- "Note: Adapty does NOT support Apple's StoreKit Messages API for win-back redemption (as of current docs). If your RC integration relied on the StoreKit message presentation flow, the equivalent UX needs a custom paywall trigger."

---

## 14b. Hard-coded product IDs / fallback products ❗️ BLOCKER

RevenueCat allows fetching products directly by App Store Connect product ID, without an offering or placement:

```swift
let products = try await Purchases.shared.products(["com.app.product.weekly", "com.app.product.yearly"])
try await Purchases.shared.purchase(product: products[0])
```

This is commonly used for **fallback paywalls** — a hard-coded set of products the app can fall back to if the network or dashboard is unreachable.

**Adapty has no equivalent API.** Products in Adapty 3.x are paywall-scoped: you fetch them via `Adapty.getPaywall(placementId:)` + `Adapty.getPaywallProducts(paywall:)`. There is no "give me these product IDs without a paywall" call.

**Migration action — this is a MIGRATION BLOCKER, not a silent rewrite:**

1. **Leave the original RC code in place.**
2. **Add a `// FIXME: ADAPTY MIGRATION BLOCKED — Purchases.shared.products([...]) has no Adapty equivalent` comment above the call.**
3. **Surface a top-level Manual review entry**, not buried in a list:
   > "**Migration blocker at `<file:line>`:** the project uses `Purchases.shared.products([...])` to fetch products by ID without a paywall. Adapty 3.x does not support this — products must be fetched via a placement + paywall. Migration options: (a) create a dedicated Adapty placement for this set of products and migrate the code to `getPaywall` / `getPaywallProducts`; (b) if this was used as a fallback, rebuild the fallback as an Adapty placement with the same product set; (c) if the products are unrelated to a specific paywall (e.g., one-off non-consumable unlocks), still create a placement to host them. The migration cannot proceed automatically — review the surrounding business logic before choosing an option."
4. **Mention the blocker in the report Summary** so the user knows the migration is partial.

---

## 14c. Observer mode — `reportTransaction` ⚠️

If the original RC project uses observer mode (RC SDK does not finish transactions; the app does), the migration MUST switch to Adapty's observer mode AND wire up `reportTransaction` calls. Otherwise Adapty will not see the purchases and access levels will never update.

**Before (RevenueCat observer mode):**

```swift
// configure the SDK
Purchases.configure(
    with: .init(withAPIKey: <API_KEY>)
        .with(purchasesAreCompletedBy: .myApp, storeKitVersion: .storeKit2)
)

// then report transactions (RC requires this only for SK2 + macOS)
try await Purchases.shared.recordPurchase(<PURCHASE_RESULT>)
```

**After (Adapty observer mode):**

```swift
// configure the SDK with observer mode
try await Adapty.activate(
    with: .builder(withAPIKey: "YOUR_ADAPTY_PUBLIC_KEY")
        .with(observerMode: true)
        .build()
)

// then report EVERY transaction (regardless of platform or StoreKit version)
try await Adapty.reportTransaction(transaction, withVariationId: "<PAYWALL_VARIATION_ID>")
```

**Critical behavior difference vs RevenueCat:**

- RevenueCat only requires `recordPurchase` for SK2 on macOS. **Adapty requires `reportTransaction` for EVERY purchase**, on every platform, regardless of StoreKit version.
- That means migrating an iOS-only RC observer-mode project still requires inserting `reportTransaction` calls at every successful purchase site — even though RC's iOS observer-mode flow didn't need `recordPurchase`. **This is the most likely silent failure mode in observer-mode migrations.**
- The `withVariationId:` parameter ties the transaction to a specific paywall variation for attribution. If the original RC code didn't track which paywall produced which purchase, you need to add that bookkeeping during migration.

🌐 Verify the exact signature against current Adapty docs (search `https://adapty.io/docs/` for "observer mode" / "reportTransaction").

⚠️ Always Manual review when observer mode is detected:

- "Project uses observer mode. Migrated to `Adapty.activate(with: .builder(...).with(observerMode: true).build())`. Verify against current Adapty docs."
- "**Every** purchase site needs a `try await Adapty.reportTransaction(transaction, withVariationId: ...)` call after the StoreKit transaction succeeds — unlike RevenueCat, Adapty requires this for every platform and StoreKit version, not just SK2/macOS. Audit every `SKPaymentQueue` / `Transaction.updates` listener in the project and confirm the `reportTransaction` call is present."
- "The `withVariationId` parameter ties the transaction to a paywall variation. If the original RC code didn't track this, decide whether to (a) use a single fixed variation ID (acceptable for non-A/B-tested projects) or (b) plumb the variation ID through the purchase flow."

---

## 14d. Consumables, non-consumables, virtual currencies

### Non-consumables (lifetime unlocks)

Both RC and Adapty recommend modeling lifetime/forever unlocks as **non-consumable products attached to an access level** — not as consumables. The migration is straightforward: the non-consumable product carries over as an Adapty product, attached to the corresponding access level in Adapty Dashboard. The code-side migration uses the same pattern as subscriptions (`makePurchase` returns a result; check `profile.accessLevels[...]?.isActive`).

✅ Manual review note: "Project includes non-consumable product(s). Attach them to the matching access level in Adapty Dashboard → Access Levels. Verify the access level returns `isActive == true` after purchase in sandbox."

### Consumables — Virtual Currencies ❗️ NOT YET SUPPORTED

**Adapty does not currently support a Virtual Currencies feature** equivalent to RevenueCat's. RevenueCat tracks consumable balances server-side (so the app doesn't need to maintain its own credit/coin counter); Adapty does not yet offer this.

If the migrating project uses RevenueCat's Virtual Currencies API (consumable purchases that grant in-app currency, server-tracked balances, balance-source-of-truth via RC):

1. **Leave the original RC consumable + Virtual Currencies code in place.**
2. **Add `// FIXME: ADAPTY MIGRATION BLOCKED — Adapty does not yet support Virtual Currencies` comments at every Virtual Currencies API call.**
3. **Top-level Manual review entry:**
   > "**Partial migration blocker:** the project uses RevenueCat's Virtual Currencies feature (consumable purchases with server-tracked balances). Adapty does not yet offer an equivalent (tracked on Adapty's roadmap). Options: (a) keep RevenueCat running in parallel for consumables only, migrating subscriptions/non-consumables to Adapty; (b) build a custom server-side balance tracker; (c) defer migration until Adapty ships the feature. The migration is partial — Adapty will handle subscriptions and access levels; consumable balance tracking is on you."
4. **Mention this in the Summary.**

If the project uses **plain consumables WITHOUT Virtual Currencies** (the app maintains its own balance counter), the migration is simpler: the consumable purchase still goes through `Adapty.makePurchase`, and the app's existing balance-tracking code is preserved unchanged. Flag in Manual review: "Project uses plain consumables. Adapty will record the purchase but does not track consumable balances server-side. Your app's existing balance code is preserved."

---

## 15. Capability check

**Before:**
```swift
if Purchases.canMakePayments() { ... }
```

**After:** Native StoreKit check (Adapty does not wrap this):
```swift
import StoreKit
if SKPaymentQueue.canMakePayments() { ... }
```

No fetch required — this is a standard StoreKit API.

---

## 16. Cache invalidation

**Before:**
```swift
Purchases.shared.invalidateCustomerInfoCache()
```

**After:** Adapty refetches the profile each `Adapty.getProfile()` call within its own caching policy. The skill removes the `invalidateCustomerInfoCache` call and replaces it with a direct `try await Adapty.getProfile()` if the caller needed fresh data. Note in Manual review: "RevenueCat's customer info cache invalidation was removed at <file:line>. Adapty manages profile caching internally; if you need to force a refresh and `Adapty.getProfile()` is returning stale data, consult current Adapty docs (search `https://adapty.io/docs/` for 'getProfile' or 'profile caching')."

---

## 17. Type cross-reference

| RevenueCat type | Adapty 3.x equivalent | Notes |
|---|---|---|
| `Purchases` | `Adapty` | Static methods, no shared instance access |
| `CustomerInfo` | `AdaptyProfile` | Returned by `Adapty.getProfile()` |
| `EntitlementInfo` | `AdaptyProfile.AccessLevel` | Accessed via `profile.accessLevels[...]` |
| `EntitlementInfos` | `[String: AdaptyProfile.AccessLevel]` | Dictionary on `profile.accessLevels` |
| `Offerings` | (no direct equivalent) | Use `Adapty.getPaywall(placementId:)` per call |
| `Offering` | `AdaptyPaywall` | One per placement |
| `Package` | `AdaptyPaywallProduct` | Returned by `getPaywallProducts` |
| `StoreProduct` | `AdaptyPaywallProduct.skProduct` (or vendor info) | Access underlying SKProduct if needed |
| `StoreTransaction` | (handled inside `.paywall()` callback) | The `didFinishPurchase` callback delivers the product |
| `PurchasesDelegate` | `AdaptyDelegate` | See section 14 |
| `LogLevel` | (Adapty config) | See section 3, fetch docs |

---

## 18. Unmapped patterns

If the skill encounters a RevenueCat construct not listed above:

1. Search current Adapty docs at `https://adapty.io/docs/` for the concept (try the RevenueCat keyword + the closest Adapty term — e.g., "entitlement" → "access level", "offering" → "placement")
2. If the docs make the equivalent unambiguous, write the code and add to Manual review with the source URL + date
3. If still unclear, or if the runtime cannot reach the docs, leave `// TODO: ADAPTY MANUAL — <one-line description of what was being done>` at the call site and surface it in Manual review

Do not guess. Do not fall back to "what RevenueCat does is probably what Adapty does." Migration silence is more expensive than asking the user to review one line.
