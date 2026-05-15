---
name: adapty-ios-migration
description: Migrate iOS Swift apps from RevenueCat to Adapty SDK (current 3.x baseline, verified per-migration against the live Adapty docs) AND answer questions about RevenueCat → Adapty API equivalents. Use this skill in two cases. CASE 1 — Full project migration. Triggered by phrasings like "migrate this to Adapty", "replace RevenueCat with Adapty", "swap RevenueCat for Adapty", "rewrite this with Adapty", or any project that has `import RevenueCat` / `import RevenueCatUI` with a request to integrate Adapty. Trigger this skill even when the user only says "add Adapty" or "integrate Adapty" on a project that already uses RevenueCat — the migration is the implied task. The skill detects whether each paywall-touching file uses the Paywall Builder flow (RevenueCatUI → AdaptyUI + `.paywall()` modifier) or a custom paywall flow (custom UI → direct `Adapty.getPaywall` + `Adapty.getPaywallProducts` + `Adapty.makePurchase`) and migrates each appropriately. Produces migrated Swift source files, a path-aware RevenueCat → Adapty mapping table, and a mandatory Adapty Dashboard setup checklist that covers access levels, placements, "Show on device", App Store Server Notifications, and attribution integrations. CASE 2 — Equivalent / Q&A lookup. Triggered by phrasings like "what's the Adapty equivalent of X", "how do I do X in Adapty (we had it in RevenueCat)", "Adapty version of `Purchases.shared.logIn`", "RevenueCat → Adapty for this snippet", or any standalone RevenueCat code fragment with a request to convert or explain it. Trigger this skill even for one-line lookup questions — the same canonical mapping is the source of truth for both modes. Do NOT use for migrations from other SDKs (Qonversion, Glassfy, Purchasely, native StoreKit) or for Android, React Native, Flutter, Unity, or any non-iOS platform — this skill is RevenueCat-to-Adapty on iOS Swift only.
---

# Adapty iOS Migration Skill (RevenueCat → Adapty)

This skill operates in **two modes**. Identify which mode applies before doing anything else.

## Mode selection

### Mode A — Quick equivalent / Q&A lookup

Use this mode when the user is asking a question about a specific RevenueCat → Adapty mapping rather than asking for a full migration. Signals:

- Snippet-level conversion: *"What's the Adapty equivalent of `Purchases.shared.logIn`?"*, *"How do I check entitlements in Adapty?"*, *"Convert this 10-line RevenueCat snippet to Adapty"*
- No project folder attached; no `.zip`; the user pastes a short fragment or just names a RevenueCat API
- The user explicitly says they're researching / planning, not yet migrating

**What to do in Mode A:**

1. Read `references/revenuecat-to-adapty-map.md` first (the primary lookup table)
2. Find the matching entry. The map entry tells you which path file is relevant (if any) and whether the signature is canonical or verify-required.
3. If the entry references `references/adapty-reference.md`, `references/paywall-builder-path.md`, or `references/custom-paywall-path.md` — read only the one that applies, then answer with the before/after code and cite the file + section.
4. If the entry is marked "verify signature" or the user's snippet hints at a less-common surface (delegate, observer mode, attribution sequencing) — consult the current Adapty docs (start from `https://adapty.io/docs/`, search the concept), confirm the Adapty 3.x signature, then answer with the before/after code and cite the source URL plus today's date (because the answer is only as fresh as the lookup). If the runtime cannot reach the docs, say so and answer from the reference files, flagging the answer as unverified.
5. If the entry isn't in the map at all — say so explicitly, consult the current Adapty docs for the concept, and answer with the URL. Do not guess.

Mode A output is a **chat answer only** — no files written. Keep it short: the snippet, the Adapty replacement, one line on why, the source (file:section or URL+date), and any ordering caveat ("requires `await Adapty.activate()` to have completed first" / "requires the paywall to have `hasViewConfiguration == true`").

### Mode B — Full project migration

Use this mode when the user provides a project folder (multi-file) and asks for the full migration. Signals:

- Project files or a folder/zip is attached
- The user uses phrasings like "migrate this project", "rewrite my whole integration", "replace RevenueCat in this app"

**What to do in Mode B:** follow the full 6-step workflow below. Mode B always produces a `MIGRATION_REPORT.md` plus migrated `.swift` files. Write them to whatever location is conventional for the current runtime (a sandboxed outputs directory if one is provided, otherwise the user's working directory). Return the report first, then the migrated files.

### When in doubt

If the request is ambiguous (e.g., a couple of files pasted but no clear instruction), ask the user once: *"Do you want a full migration with file output, or just the Adapty equivalents for these snippets?"* — then proceed in the matching mode.

---

## Hard rules (NEVER violate, both modes)

These rules exist because hallucinated APIs, version mismatches, and wrong call ordering are the dominant failure modes in SDK migrations. A migration that compiles but silently breaks paywalls or skips initialization costs real revenue.

1. **Use the current Adapty 3.x baseline, never a hard-pinned version.** At the start of every migration (Mode B) and every verify-required Q&A (Mode A), consult the current Adapty installation docs (start from `https://adapty.io/docs/` and navigate to the iOS SDK installation page) to confirm the current supported Adapty 3.x SDK version. Use that version in dependency declarations. Never write a cached version number from the reference files into output without re-confirming first. If the runtime cannot reach the docs, leave the version as `<VERIFY_FROM_DOCS>` and surface a Manual review item asking the user to pin the version themselves. Adapty 2.x and 3.x have incompatible APIs — never mix.

2. **Await every SDK lifecycle call, in strict order.** The Adapty 3.x API is async/throws and the SDK has documented ordering requirements:
   - `try await Adapty.activate(...)` MUST complete before any other Adapty call. Synchronous-looking activate without `await` is wrong, even if older Adapty docs or third-party guides show it that way.
   - `try await Adapty.identify(...)` MUST complete before any paywall, profile, purchase, or restore call when the project uses authenticated users.
   - Never fire `Adapty.getPaywall`, `getProfile`, `makePurchase`, `restorePurchases`, or `getPaywallConfiguration` concurrently with — or before — activate/identify. They must be sequenced inside `Task { ... }` blocks with `try await` on the lifecycle calls first.

3. **Detect the paywall path before migrating; do not assume Paywall Builder.** RevenueCat projects come in two flavors that migrate to incompatible Adapty patterns:
   - **Paywall Builder path** — source uses `import RevenueCatUI`, `PaywallView`, `paywallFooter`, `presentPaywallIfNeeded`, or any RevenueCatUI surface. Migrates to `AdaptyUI` + the SwiftUI `.paywall()` modifier. Read `references/paywall-builder-path.md` for the full pattern.
   - **Custom paywall path** — source uses its own UIKit/SwiftUI views combined with direct `Purchases.shared.purchase(package:)` / `restorePurchases` calls. Migrates to `Adapty.getPaywall` + `Adapty.getPaywallProducts` + `Adapty.makePurchase` / `restorePurchases`, with the project's existing UI preserved. Does NOT import `AdaptyUI`. Read `references/custom-paywall-path.md` for the full pattern.
   - A project may use BOTH paths in different screens — handle them per-screen, not globally.

4. **AdaptyUI is conditional, not a universal replacement for RevenueCatUI.** Only add `import AdaptyUI` and the `AdaptyUI.activate()` call when the migration is genuinely going down the Paywall Builder path. Custom paywall migrations stay on `import Adapty` alone — adding AdaptyUI to a custom-UI project is a bug, not a safe default.

5. **Validate `hasViewConfiguration` in every Paywall Builder migration, and flag the design rebuild.** Adapty paywalls only have a remote view configuration (and therefore only render via `AdaptyUI.getPaywallConfiguration`) when the dashboard's "Show on device" toggle is enabled and a Paywall Builder design exists. The migrated code MUST guard against `paywall.hasViewConfiguration == false` with a graceful fallback (log + skip presentation), and the Dashboard checklist MUST explicitly call out enabling "Show on device" for every placement. Additionally, **every Paywall Builder migration MUST surface a Manual review entry stating that RevenueCat paywall designs do NOT transfer to Adapty and must be rebuilt in Adapty's visual builder** (see `references/paywall-builder-path.md` §0). The skill migrates code, not paywall design — never imply otherwise in the report.

6. **The reference is canonical, but version-current.** Patterns in `references/adapty-reference.md`, `references/paywall-builder-path.md`, and `references/custom-paywall-path.md` are the source of truth for *shape* (which APIs to call, in what order, with what arguments). Always verify *signatures* (exact parameter names, async/throws, return types) against the current Adapty docs (`https://adapty.io/docs/`) at migration time — the references may be out of date relative to the latest SDK minor version. If the runtime cannot reach the docs, flag every emitted signature in Manual review as unverified.

7. **Unknown features must be verified, not guessed.** If RevenueCat code uses a feature NOT covered in the references (examples: observer mode, custom attributes, server-side webhooks, delegate callbacks), the skill MUST consult the current Adapty docs (`https://adapty.io/docs/`) and confirm the exact 3.x signature before writing any migration code. If the docs do not give a clear 3.x answer (or the docs are unreachable), leave a `// TODO: ADAPTY MANUAL — <description>` comment in the code and surface it in the "Manual review" section. Never hallucinate a method name or signature.

8. **Output language is English** — explanations, comments, mapping tables, checklists, code. Everything.

9. **The Dashboard checklist is mandatory and complete.** Even if the user only asks for code, ALWAYS produce the Dashboard checklist. It must include: access levels, placements, paywalls, products, "Show on device" toggle, App Store Server Notifications (production + sandbox URLs), App Store Connect integration, shared secret, and any attribution integrations the project uses. Omitting any of these silently breaks the migration at runtime.

10. **RevenueCat → Adapty name mapping requires manual review, not silent rename.** Entitlement names map reasonably cleanly to access levels (both are server-side capability flags). But RevenueCat **offerings** and Adapty **placements** are related-but-not-equivalent concepts: an offering bundles packages, while a placement is an "ad slot" with one paywall attached. The skill carries names over verbatim where it can but ALWAYS flags offering↔placement mappings in Manual review with a note like "Confirm the semantic match — this is a rename, not a guaranteed equivalence." Do not assume 1:1 correctness.

## Mode B workflow (full project migration)

The 6 steps below apply only to Mode B. For Mode A (Q&A), follow the much shorter steps in the Mode selection section above and stop.

### Step 1: Read references and confirm current SDK version

Before opening any of the user's project files:

1. Consult the current Adapty iOS SDK installation docs (`https://adapty.io/docs/` → iOS SDK installation page) and note the current supported 3.x SDK version. If the runtime has no web access, record this fact and proceed with `<VERIFY_FROM_DOCS>` as the version placeholder; surface a Manual review item.
2. Read `references/adapty-reference.md` — common base patterns (init, identify, AdaptyManager, ordering)
3. Read `references/revenuecat-to-adapty-map.md` — per-API mapping table
4. Read `references/dashboard-checklist.md` — Adapty Dashboard manual setup steps
5. Read `references/output-format.md` — the structure of the final `MIGRATION_REPORT.md`

Defer reading `paywall-builder-path.md` / `custom-paywall-path.md` until Step 2 determines which path applies — they're loaded based on inventory results, not upfront.

### Step 2: Inventory RevenueCat usage AND detect paywall path

The user provides a project folder. List its contents recursively and for each `.swift` file scan for:

- `import RevenueCat`, `import RevenueCatUI`
- `Purchases.configure(...)` — initialization (closure/async/builder form). Watch for `purchasesAreCompletedBy: .myApp` → observer mode (see `revenuecat-to-adapty-map.md` §14c).
- `Purchases.shared.*` — any method call
- `Purchases.logLevel` — logging configuration
- `Purchases.canMakePayments()` — capability check
- Types: `CustomerInfo`, `EntitlementInfo`, `Offering`, `Offerings`, `Package`, `StoreProduct`, `StoreTransaction`, `PurchasesDelegate`
- `PaywallView`, `presentPaywallIfNeeded`, `paywallFooter` — RevenueCatUI APIs
- `attribution.set*` — attribution calls (note: most map to `Adapty.setIntegrationIdentifier(key:value:)`, NOT `updateAttribution` — see `revenuecat-to-adapty-map.md` §12)
- Login/logout: `Purchases.shared.logIn`, `Purchases.shared.logOut`
- **Migration blockers and partial-coverage surfaces** (each requires a Manual review entry, sometimes a `// FIXME` blocker):
  - `Purchases.shared.products([...])` — fetching products by ID without a paywall (§14b — BLOCKER, no Adapty equivalent)
  - `Purchases.shared.recordPurchase(...)` — observer-mode transaction recording (§14c — migrates to `Adapty.reportTransaction`, required for EVERY platform on Adapty)
  - `eligiblePromotionalOffers()`, `eligibleWinBackOffers(...)`, `.with(promotionalOffer:)`, `.with(winBackOffer:)` — explicit offer selection (§14a — code is removed; Adapty applies offers automatically based on dashboard config)
  - `checkTrialOrIntroDiscountEligibility(...)` — replaced with `product.subscriptionOffer != nil` check (§14a)
  - RevenueCat Virtual Currencies API surface (consumables with server-tracked balances) — Adapty does not yet support (§14d — BLOCKER)
  - Consumable purchases with app-side balance counters — preserved as-is (§14d)
  - Non-consumable products mapped to entitlements — straightforward migration (§14d)
  - `attribution.setMparticleID`, `setAirshipChannelID`, `setCleverTapID`, `setKochavaDeviceID` — providers NOT yet supported by Adapty (§12 — BLOCKER per-provider)
  - `attribution.setPushToken` / `setPushTokenString` / `setMediaSource` / `setCampaign` / `setAdGroup` / `setAd` / `setKeyword` / `setCreative` — no direct Adapty equivalent (§12 — Manual review only)
  - `webCheckoutUrl` on a package — has an Adapty web-paywall equivalent; verify the current API name against Adapty docs before writing the migration (do not assume a specific function name)
  - `redeemWebPurchase(...)` — Adapty does NOT support RevenueCat redemption links (Manual review BLOCKER)
  - `offerings.current` / `offerings.default` — RC's "default offering" concepts; no direct equivalent in Adapty's placement model (Manual review — see §6)
  - `customerInfoStream` — RC's Swift Concurrency stream API; Adapty uses delegate callbacks instead (§14 delegate pattern)
  - `Customer Center`, `showManageSubscriptions(...)`, `beginRefundRequest(...)` — Adapty does NOT yet support these (Manual review BLOCKER per call)
  - `Offline Entitlements` usage in RC — Adapty does NOT yet support (Manual review)

Also scan project-level dependency files: `Podfile`, `Package.swift`, `*.xcodeproj/project.pbxproj`, `Cartfile`.

**Paywall path detection (mandatory):**

- If the project imports `RevenueCatUI` OR uses `PaywallView`, `paywallFooter`, or `presentPaywallIfNeeded` → that screen/file is on the **Paywall Builder path**. Read `references/paywall-builder-path.md` now.
- If the project performs `Purchases.shared.purchase(package:)` directly from its own custom UI (not via `PaywallView`) → that screen/file is on the **custom paywall path**. Read `references/custom-paywall-path.md` now.
- Mixed projects exist — annotate each paywall-touching file with its path independently.
- If ambiguous (e.g., the project has both `PaywallView` and direct `purchase(package:)` calls in different places), ask the user once: "I see both Paywall Builder usage and direct purchase calls — should I migrate everything to AdaptyUI Paywall Builder, or keep the custom paywall flows custom (recommended if you have non-standard paywall UI)?"

**Auth detection (also mandatory):**

- If the project calls `Purchases.shared.logIn` / `Purchases.shared.logOut` → the migration MUST insert `try await Adapty.identify(...)` / `Adapty.logout()` at matching call sites and ensure all profile/paywall/purchase calls happen *after* identify completes
- If the project never calls logIn → anonymous-only; skip identify entirely

Build a categorized inventory: initialization, entitlement check, offerings/paywall fetch (per path), purchase (per path), restore (per path), identification, attribution, delegate, dependencies, other.

### Step 3: Plan each replacement, path-aware

For each item in the inventory, look it up in `references/revenuecat-to-adapty-map.md` and classify:

- **Mapped, single path** → use the path-specific pattern verbatim (substituting the user's variable, class, and access-level names)
- **Mapped, both paths** (e.g., entitlement check) → both paths use the same pattern; just apply it
- **Mapped but flagged "verify signature"** → consult the current Adapty docs for the relevant concept (start from `https://adapty.io/docs/`), confirm the exact 3.x parameter names and return types, then write the migration. Note the verified URL + date in Manual review. If docs are unreachable, write the pattern from the reference file and flag the signature as unverified.
- **Not in the map** → search the current Adapty docs for the concept. If still unclear (or docs unreachable), leave `// TODO: ADAPTY MANUAL — <description>` and add to Manual review.

For offering → placement specifically: carry the name over BUT add to Manual review: "Offering `<name>` carried over as placement `<name>`. RevenueCat offerings and Adapty placements are related but not 1:1 equivalent; confirm the semantic match before relying on it."

Never write Adapty code from memory for anything not in the reference. Memory is the failure mode this skill exists to prevent.

### Step 4: Write the migrated Swift files

Write each modified `.swift` file to a `migrated/` directory under the runtime's normal output location, preserving the original project's relative paths. (In a sandboxed runtime with a dedicated outputs path, use that; otherwise use a `migrated/` directory under the user's working directory.) Rules:

**Always:**
- Replace `import RevenueCat` with `import Adapty`
- Replace the `Purchases.configure(...)` call with the awaited activation sequence from `references/adapty-reference.md` (note: `try await Adapty.activate(...)` inside a `Task` in `init()`, NOT a synchronous call)
- Create or update `AdaptyManager.swift` matching `references/adapty-reference.md`
- Convert closure-based RevenueCat callbacks to `async/await` Swift Concurrency throughout
- Wherever the project used `Purchases.shared.logIn` → insert `try await Adapty.identify(...)` and ensure subsequent profile/paywall/purchase calls await identify completion
- Replace `Purchases.shared.getCustomerInfo` calls with `try await Adapty.getProfile()`
- Preserve all comments, business logic, and unrelated code unchanged
- Update `Podfile`/`Package.swift`/`.pbxproj` to remove RevenueCat and add Adapty using the version confirmed in Step 1

**Paywall Builder path only (when applicable, per-file):**
- Replace `import RevenueCatUI` with `import AdaptyUI`
- Add `try await AdaptyUI.activate()` after `Adapty.activate()` in the entry point
- Replace `PaywallView(offering:)` / `presentPaywallIfNeeded` with the `.paywall()` modifier from `references/paywall-builder-path.md`
- Include all callbacks per the reference (didPerformAction, didFinishPurchase, didFailPurchase, didFinishRestore, didFailRestore, didFailRendering)
- Guard `paywall.hasViewConfiguration` before calling `AdaptyUI.getPaywallConfiguration` — if false, log a warning and skip presentation. This is non-optional.

**Custom paywall path only (when applicable, per-file):**
- Do NOT add `import AdaptyUI`
- Replace `Purchases.shared.getOfferings` with `try await Adapty.getPaywall(placementId:, locale:)` + `try await Adapty.getPaywallProducts(paywall:)` per `references/custom-paywall-path.md`
- Replace `Purchases.shared.purchase(package:)` with `try await Adapty.makePurchase(product:)`
- Replace `Purchases.shared.restorePurchases` with `try await Adapty.restorePurchases()`
- Keep the project's existing custom UI views — they bind to the new product list, no UI rewrite
- A `paywall.hasViewConfiguration` check is NOT needed on this path (the project isn't using AdaptyUI rendering)

Files in the project that contain no RevenueCat references are NOT copied to `migrated/` — only changed files appear there. This keeps the diff readable.

### Step 5: Assemble `MIGRATION_REPORT.md`

Write `MIGRATION_REPORT.md` (alongside the `migrated/` directory from Step 4) following `references/output-format.md` exactly. Five sections:

1. **Summary** — what was migrated, file count, paywall path(s) detected, SDK version confirmed from docs (or noted as unverified), next steps in one paragraph
2. **Mapping table** — every RevenueCat API touched in this migration, with three columns: RevenueCat | Adapty | Source (reference file + section, or verified URL + date)
3. **File-by-file changes** — one short paragraph per modified file, noting which paywall path applied
4. **Adapty Dashboard checklist** — copied from `references/dashboard-checklist.md` and customized with this project's specific access-level names, placement IDs, product IDs, and attribution providers
5. **Manual review** — every `// TODO: ADAPTY MANUAL`, every verified API the user should double-check, every offering↔placement semantic-match warning, every entitlement↔access-level naming reminder, the API key replacement, and the App Store Server Notifications URL switch

### Step 6: Present the deliverables

Return `MIGRATION_REPORT.md` first (highest user value), then every file under `migrated/`. Use whatever file-presentation mechanism the runtime offers (some runtimes auto-render written files; others expect an explicit listing in the chat reply). Keep the post-amble short — the report itself explains everything.

## Triggering examples

**Mode A (Q&A / equivalent lookup) — these should trigger and produce a chat answer:**

- "What's the Adapty equivalent of `Purchases.shared.logIn`?"
- "How do I check entitlements in Adapty? We use `customerInfo.entitlements["pro"].isActive` in RevenueCat."
- "Convert this RevenueCat code to Adapty:" + a short snippet
- "Adapty version of attribution setup?" (after mentioning RevenueCat context)
- "Is `Adapty.makePurchase` async/await in 3.x?"

**Mode B (full migration) — these should trigger and produce files:**

- "Migrate this iOS app from RevenueCat to Adapty" + project folder attached
- "Replace RevenueCat with Adapty in this Swift project" + project folder
- "I want to switch from RevenueCat to Adapty" + project folder
- "Adapty integration" on a project that already imports RevenueCat across multiple files

**These should NOT trigger the skill:**

- "Set up Adapty in a new project" (no RevenueCat present — use general Adapty knowledge)
- "Migrate from Qonversion / Glassfy / Purchasely to Adapty" (out of scope)
- "Migrate Android app from RevenueCat to Adapty" (not iOS)
- "How does Adapty compare to RevenueCat?" (comparison question, not a migration or equivalent lookup)

## File layout

```
adapty-ios-migration/
├── SKILL.md (this file)
└── references/
    ├── adapty-reference.md            # Common base: install, activate, identify, AdaptyManager, ordering invariants
    ├── paywall-builder-path.md        # AdaptyUI + .paywall() modifier + hasViewConfiguration validation
    ├── custom-paywall-path.md         # Adapty.getPaywall + getPaywallProducts + makePurchase + restorePurchases
    ├── revenuecat-to-adapty-map.md    # Per-API mapping table, path-aware
    ├── dashboard-checklist.md         # Mandatory Adapty Dashboard setup (access levels, placements, Show on device, ASSN, integrations)
    └── output-format.md               # Exact structure of MIGRATION_REPORT.md
```
