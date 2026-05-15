# adapty-migration-skill

A Claude skill that migrates the **code-side** of iOS Swift apps from **RevenueCat** to **Adapty** — handling both the Paywall Builder (AdaptyUI) wiring and the custom-paywall direct-API flow, with a mandatory Adapty Dashboard checklist generated for every migration.

> **Note on Paywall Builder migrations:** the skill rewrites your paywall *code* (swaps RevenueCatUI's `PaywallView` / `presentPaywallIfNeeded` for AdaptyUI's `.paywall()` modifier, adds the `hasViewConfiguration` guard, etc.) — but you will need to **rebuild your paywall design from scratch in Adapty's visual builder. RevenueCat paywall layouts do not transfer.** Use a template as a starting point, or recreate your existing design.

> Migrates the common RevenueCat iOS integration patterns to Adapty: path detection, SDK swap, AdaptyUI wiring, a per-file change report, and a 14-step dashboard checklist. Conservative by design — flags products-by-ID, observer mode, Virtual Currencies, and unsupported attribution providers as migration blockers rather than guessing.

---

## What it does

- **Detects the paywall path per file.** Files using `RevenueCatUI` (`PaywallView`, `paywallFooter`, `presentPaywallIfNeeded`) migrate to AdaptyUI + the `.paywall()` modifier. Files using custom UI + direct `Purchases.shared.purchase(package:)` migrate to `Adapty.getPaywall` + `Adapty.getPaywallProducts` + `Adapty.makePurchase`. Mixed projects are handled per-file.
- **Enforces Adapty's call-ordering invariants.** `try await Adapty.activate(...)` must complete before any other SDK call; `try await Adapty.identify(...)` must complete before paywall/profile/purchase/restore when auth is involved. The skill encodes these as hard rules.
- **Verifies signatures against current docs.** No version is hard-pinned in the skill — every migration consults the current Adapty iOS SDK installation docs first to confirm the current Adapty 3.x SDK version. Reference patterns describe shape; live docs confirm exact signatures. If the runtime cannot reach the docs, the skill emits a `<VERIFY_FROM_DOCS>` placeholder and surfaces a Manual review item.
- **Validates `hasViewConfiguration` for Paywall Builder migrations.** The generated code includes the guard required for the dashboard's "Show on device" toggle. Without it, paywalls render nothing at runtime — a silent failure mode this skill prevents.
- **Generates a mandatory dashboard checklist.** Every migration produces a 14-step Adapty Dashboard checklist covering access levels, placements, paywalls, products, "Show on device", **App Store Server Notifications (production + sandbox URLs)**, attribution integrations, and webhooks/raw events forwarding.
- **Flags offering↔placement mappings for manual review.** RevenueCat offerings and Adapty placements are related but not 1:1; the skill carries names over and always asks the user to confirm semantic equivalence.

## What it does NOT do

- **It does not migrate paywall designs.** Layouts, copy, images, fonts, and colors from your RevenueCat paywalls do not transfer. On the Paywall Builder path, you rebuild each paywall in Adapty Dashboard's visual builder. The skill migrates the code that *renders* the paywall; the visual design is manual work in either dashboard.
- It does not access your Adapty Dashboard. You configure it manually using the generated checklist.
- It does not paste your API key. The migrated code uses `"YOUR_ADAPTY_PUBLIC_KEY"` as a placeholder; you replace it.
- It does not run sandbox tests. Sandbox purchase verification is on you, before shipping.
- It does not migrate from other SDKs (Qonversion, Glassfy, Purchasely, native StoreKit) or other platforms (Android, React Native, Flutter, Unity). **iOS Swift, RevenueCat → Adapty only.**

## Two modes

**Mode A — Q&A lookup.** Ask Claude "what's the Adapty equivalent of `Purchases.shared.logIn`?" and get a chat answer with the canonical replacement, the source reference file, and any ordering caveat. No files written.

**Mode B — Full project migration.** Hand Claude a project folder (or a few `.swift` files) and say "migrate this to Adapty." Claude produces:
- Migrated `.swift` files preserving the original folder structure
- Updated `Package.swift` / `Podfile`
- A `MIGRATION_REPORT.md` with:
  1. Summary (file count, paywall paths detected, SDK version confirmed)
  2. API mapping table (per-API, with reference citations or fetched URLs + dates)
  3. File-by-file changes
  4. The 14-step Adapty Dashboard checklist, customized with your project's access-level names, placement IDs, and product IDs
  5. Manual review items (API key replacement, ASSN URL switch, "Show on device" toggle, offering↔placement confirmations, etc.)

## Install

### Option 1 — Download the latest release (recommended)

Download `adapty-migration-skill.zip` from the [latest release](../../releases/latest), then upload it to Claude:

- **Claude.ai web/desktop:** Settings → Capabilities → Skills → Upload skill → select the zip
- **Claude Code:** unzip into `~/.claude/skills/adapty-migration-skill/` and restart your terminal

### Option 2 — Clone and zip yourself

```bash
git clone https://github.com/ABurakk/adapty-migration-skill.git
cd adapty-migration-skill/skill
zip -r ../adapty-migration-skill.zip .
cd ..
```

This matches the layout produced by the release workflow: the zip root contains `SKILL.md` and the `references/` directory directly — not a wrapping `skill/` folder. Then upload `adapty-migration-skill.zip` the same way.

## How to use

Once installed, just talk to Claude normally about your RevenueCat-to-Adapty migration:

```
> I want to migrate this iOS project from RevenueCat to Adapty.
  [attach your project folder or zip]
```

Or for a quick lookup:

```
> What's the Adapty equivalent of Purchases.shared.getOfferings?
```

Claude detects when the skill applies and loads it automatically — no special invocation needed.

## Skill structure

```
skill/
├── SKILL.md                            # Workflow + hard rules
└── references/
    ├── adapty-reference.md             # Common patterns: install, activate, identify, AdaptyManager
    ├── paywall-builder-path.md         # AdaptyUI + .paywall() + hasViewConfiguration
    ├── custom-paywall-path.md          # Adapty.getPaywall + getPaywallProducts + makePurchase
    ├── revenuecat-to-adapty-map.md     # Per-API mapping, path-aware
    ├── dashboard-checklist.md          # 14-step mandatory setup
    └── output-format.md                # MIGRATION_REPORT.md template
```

## License

[MIT](./LICENSE)
