# Output Format: `MIGRATION_REPORT.md`

The skill always writes a single `MIGRATION_REPORT.md` alongside the `migrated/` directory (the runtime's outputs path if one is provided, otherwise the user's working directory). This file is the deliverable's table of contents — the user reads it first, then opens individual migrated `.swift` files.

The report has five sections in this exact order. Use the headings below verbatim so the structure is consistent across migrations.

---

## Template

```markdown
# RevenueCat → Adapty Migration Report

**Generated:** <ISO date>
**Source SDK:** RevenueCat
**Target SDK:** Adapty (3.x — version `<VERSION>` confirmed from current Adapty iOS SDK installation docs on `<date>`, or `<VERIFY_FROM_DOCS>` if docs were unreachable)
**Paywall path(s) detected:** <Paywall Builder | Custom | Mixed — both>
**Project:** <project name or root folder name>

## 1. Summary

<One paragraph, 3–5 sentences:>
- What was migrated (number of files touched, primary subsystems, paywall path per file)
- Confidence level (e.g., "All patterns matched canonical references" vs "3 patterns required docs fetch — see Manual review")
- What the user must do before the migrated code works (always: configure Adapty Dashboard per section 4, replace the API key placeholder, switch App Store Server Notifications URLs, enable "Show on device" for Paywall Builder paywalls, run sandbox tests)

## 2. API Mapping Table

| RevenueCat | Adapty | Source |
|---|---|---|
| `Purchases.configure(withAPIKey: ...)` | `try await Adapty.activate(...)` (+ optional `AdaptyUI.activate()` on Paywall Builder migrations) | adapty-reference.md §3 |
| `customerInfo.entitlements["pro"]?.isActive` | `profile.accessLevels["pro"]?.isActive ?? false` | adapty-reference.md §5 |
| `Purchases.shared.logIn(...)` | `try await Adapty.identify(...)` | adapty-reference.md §4 |
| `PaywallView(offering:)` | `.paywall()` modifier + `hasViewConfiguration` guard | paywall-builder-path.md §2 |
| `Purchases.shared.purchase(package:)` (custom UI) | `try await Adapty.makePurchase(product:)` | custom-paywall-path.md §3 |
| ... | ... | ... |

<Include only rows for APIs actually encountered in this project. Source column cites the reference file + section, OR the verified docs URL + date. Do not list APIs that weren't in the source project.>

## 3. File-by-File Changes

<For each modified file, one entry like:>

### `Sources/App/MyApp.swift`
- Replaced `import RevenueCat` with `import Adapty` + `import AdaptyUI`
- Replaced `Purchases.configure(...)` with the three-line activation sequence
- API key set to placeholder `"YOUR_ADAPTY_PUBLIC_KEY"` — must be replaced before running

### `Sources/App/Services/SubscriptionService.swift`
- Renamed `RevenueCatService` → `AdaptyManager` per the canonical singleton pattern
- Switched `getCustomerInfo` closure to `Adapty.getProfile()` async/await
- Entitlement `"pro"` carried over as access-level key `"pro"` (verify in dashboard)

### `Sources/App/Views/PaywallView.swift` (Paywall Builder path)
- Removed RevenueCatUI's `PaywallView(offering:)` usage
- Added `import AdaptyUI`
- Added the `.paywall(...)` modifier with all seven callbacks (didPerformAction, didFinishPurchase, didFailPurchase, didFinishRestore, didFailRestore, didFailRendering) + loadPaywallAndShow helper + `hasViewConfiguration` guard
- Placement ID set to `"premium"` (carried over from RevenueCat offering ID — manual review flagged)

### `Sources/App/Views/CustomPaywallView.swift` (custom paywall path)
- Kept the project's existing custom SwiftUI view structure
- Did NOT add `import AdaptyUI`
- Introduced `PaywallViewModel` using `Adapty.getPaywall` + `Adapty.getPaywallProducts` for product loading
- Replaced `Purchases.shared.purchase(package:)` with `try await Adapty.makePurchase(product:)`
- Replaced restore call with `try await Adapty.restorePurchases()`

<...>

## 4. Adapty Dashboard Checklist

<Copy the full contents of references/dashboard-checklist.md, with the placeholders substituted:>
- <ACCESS_LEVEL_NAMES>  → the actual entitlement names found (comma-separated list)
- <PLACEMENT_NAMES>     → the actual offering names found
- <PRODUCT_IDS>         → the actual product IDs found in the project (if any were hardcoded)
- <APP_BUNDLE_ID>       → the bundle ID from Info.plist or pbxproj (if available)

## 5. Manual Review

These items require user attention. Each entry has a file path, a line reference, a description of what was migrated, and (if applicable) the docs URL that was fetched to confirm the API + the date of fetch.

- **`Sources/App/MyApp.swift`** — API key placeholder. Replace `"YOUR_ADAPTY_PUBLIC_KEY"` with the Public SDK Key from Adapty Dashboard → App settings → API Keys. **Never reuse the RevenueCat key value here** — different format, silent failure.
- **App Store Server Notifications** — switch production AND sandbox ASSN URLs in App Store Connect from RevenueCat's to Adapty's. See Dashboard checklist §4. Invisible from app code; can only be verified in App Store Connect.
- **`Sources/App/Views/PaywallView.swift` (Paywall Builder)** — `hasViewConfiguration` guard: enable "Show on device" toggle in Adapty Dashboard for placement `"premium"`. Without it the migrated paywall renders nothing at runtime. See Dashboard checklist §9.
- **Paywall design rebuild (Paywall Builder migrations only)** — RevenueCat paywall designs do NOT transfer to Adapty. For each placement migrated on the Paywall Builder path (`<list placement IDs>`), recreate the paywall layout in Adapty Dashboard → Paywalls → visual builder. Plan time for design rebuild, not just code review.
- **Placement naming** — RevenueCat offering `current` was carried over as Adapty placement `current`. RevenueCat offerings and Adapty placements are related but NOT guaranteed 1:1 equivalents (one offering bundles packages; one placement hosts one paywall). Confirm the semantic match before relying on it.
- **`Sources/App/AuthService.swift:42`** — User identification call (`try await Adapty.identify`). Ordering: ensure `Adapty.activate` completes before this call, and that subsequent paywall/profile/purchase calls await this completion. See adapty-reference.md §4.
- **`Sources/App/Analytics.swift:88`** — Attribution call. Migrated based on current Adapty Adjust integration docs (consulted at `<verified URL>` on `<date>`). (a) Configure the Adjust integration in Adapty Dashboard → Integrations. (b) Verify `Adapty.updateAttribution` is called from Adjust's attribution callback, NOT directly after `Adapty.activate`. See revenuecat-to-adapty-map.md §12.
- **`Sources/App/Legacy/UIKitPaywall.swift`** — `// TODO: ADAPTY MANUAL — UIKit Paywall Builder hosting`. SwiftUI patterns covered in references; UIKit hosting consulted from current Adapty docs at `<verified URL>` on `<date>`.
- **Access-level naming** — your RevenueCat entitlements were `pro` and `family`. Create access levels with those exact names in Adapty Dashboard (case-sensitive). Entitlement ↔ access level is the closest 1:1 mapping in the migration, but the code matches by exact string.
- **Webhooks / Raw events** — RevenueCat webhook events do NOT carry over automatically. If your team consumes webhooks for analytics, CRM, or server-side state, re-configure each pipeline in Adapty Dashboard → Integrations. See Dashboard checklist §12.

<If there are no manual review items, write: "No items beyond the API key placeholder, App Store Server Notifications URL switch, 'Show on device' toggle (Paywall Builder migrations), and the Dashboard checklist. The migration matched the reference patterns end-to-end.">
```

---

## Conventions for filling the template

### Tone
Direct, technical, English. No marketing. No emoji except the ✅/❌/⚠️ already present in the canonical AdaptyManager (those carry over because they're part of the reference pattern, not added by the skill).

### Mapping table rules
- One row per distinct RevenueCat API actually encountered. Do not list mappings that didn't appear in the project.
- The "Source" column is either:
  - `adapty-reference.md §N` / `paywall-builder-path.md §N` / `custom-paywall-path.md §N` (when reference-canonical), OR
  - `verified against <URL> on <YYYY-MM-DD>` (when consulted against current Adapty docs at migration time)
- Group rows roughly in order: dependencies, imports, initialization, customer info, entitlement check, offerings/paywall, purchase, restore, identification, attribution, other.
- When a single RevenueCat API has different Adapty equivalents per paywall path, list both rows, each tagged with the path (Paywall Builder / Custom).

### File-by-file rules
- List only files that were modified. Unchanged files are not mentioned.
- Use the file's path relative to the project root.
- Bullet points should describe *what changed*, not the full new content. The actual code is in the file itself.

### Manual review rules
- One bullet per item.
- File path + line number if applicable.
- For every API verified against current Adapty docs, the bullet must include the source URL — this lets the user verify the skill's claim against the live docs.
- Always include the API key replacement bullet.
- Always include any access-level / placement name-mismatch warnings.

### When the migration is trivial
Even for a one-file migration with no fetched docs, all five sections appear in the report. Section 5 ("Manual review") can be short ("Replace `YOUR_ADAPTY_PUBLIC_KEY` and complete the Dashboard checklist"), but section 4 (Dashboard checklist) is never abbreviated — it's the full content from `dashboard-checklist.md` with the placeholders substituted.

### When something is too uncertain to migrate
If the skill encounters code it cannot confidently migrate even after fetching docs:
- Leave the original RevenueCat code in place
- Add a `// FIXME: ADAPTY MIGRATION BLOCKED — <reason and URL consulted>` comment above it
- Add a top-level entry in Manual review: "Section X of `<file>` was not migrated. Reason: <reason>. Consulted: <URL>. Recommend manual review with Adapty support."
- Mention this in Summary so the user knows the migration is partial

A partial-but-honest migration is much more useful than a complete-but-wrong one.
