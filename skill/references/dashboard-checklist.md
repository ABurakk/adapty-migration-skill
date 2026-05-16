# Adapty Dashboard Manual Setup Checklist

The migration produces working code, but the code will not function at runtime until the Adapty Dashboard is configured. This checklist is mandatory — it gets copied (and customized with project-specific names) into the "Adapty Dashboard checklist" section of every `MIGRATION_REPORT.md`, regardless of what the user asked for.

This file is **shared across platforms** (iOS and Flutter migrations both copy it into the report). The Adapty Dashboard is the same dashboard regardless of SDK. The skill applies the per-platform store sections based on what the project targets:

- **iOS-only migration** → include the App Store Connect sections (§2–§4), skip the Google Play sections (§4b).
- **Flutter migration targeting iOS only** → same as iOS-only above.
- **Flutter migration targeting Android only** → skip §2–§4 (App Store Connect / ASSN), include §4b (Google Play service account + Real-Time Developer Notifications).
- **Flutter migration targeting both iOS and Android** → include §2–§4 AND §4b.

The skill customizes the placeholders in **bold** with values discovered while scanning the project:

- **`<ACCESS_LEVEL_NAMES>`** — the entitlement names from RevenueCat (e.g., `pro`, `premium`)
- **`<PLACEMENT_NAMES>`** — the offering IDs from RevenueCat, used as placement IDs in Adapty
- **`<PRODUCT_IDS>`** — App Store Connect / Google Play product identifiers referenced in the RevenueCat code
- **`<APP_BUNDLE_ID>`** — bundle identifier from the project (iOS)
- **`<ANDROID_PACKAGE_NAME>`** — Android package name (Flutter Android target)
- **`<ATTRIBUTION_PROVIDERS>`** — Adjust / AppsFlyer / Branch / AppMetrica, if any were detected in code

**Items marked 🔴 MANDATORY** are non-negotiable — skipping them silently breaks the migration at runtime.

---

## Pre-flight

- [ ] You have an Adapty account at https://app.adapty.io/ (sign up if you don't)
- [ ] Your App Store Connect account has the products configured (the same products you previously used with RevenueCat)
- [ ] You have App Store Connect API access — Adapty needs an App Store Connect API Key to read product info

## 1. Create the app in Adapty 🔴 MANDATORY

- [ ] Log in to https://app.adapty.io/
- [ ] Create a new app (or select an existing one)
- [ ] Set the bundle ID to **`<APP_BUNDLE_ID>`** — must match the app's actual bundle ID exactly

## 2. Connect App Store Connect API 🔴 MANDATORY

Without this, Adapty can't read products, validate receipts, or update subscription state.

- [ ] Go to **App settings → iOS SDK → App Store integration** in Adapty Dashboard
- [ ] In App Store Connect → Users and Access → Keys → App Store Connect API → Generate Key. Use the minimum role currently documented by Adapty (search current docs at `https://adapty.io/docs/` for "App Store Connect" / "ASC integration" — required roles have changed over time).
- [ ] Paste the Key ID, Issuer ID, and the `.p8` private key file into Adapty's App Store integration page
- [ ] Verify the integration page shows green / connected before proceeding

## 3. App-Specific Shared Secret 🔴 MANDATORY

Required for server-side receipt validation. Without it, subscription state can't be updated after the initial purchase.

- [ ] In App Store Connect → My Apps → your app → App Information → App-Specific Shared Secret, generate a shared secret if you don't already have one
- [ ] In Adapty App settings → App Store integration, paste the shared secret
- [ ] Save and verify it shows as configured

## 4. App Store Server Notifications 🔴 MANDATORY — most-missed step

App Store Server Notifications (ASSN) are how Apple tells Adapty about subscription state changes: renewals, cancellations, refunds, billing retries. Without ASSN, Adapty's view of subscription state diverges from reality over time — your code will eventually return `isPremium == true` for a user who cancelled days ago, or vice versa.

This step is the single most common missed item in RevenueCat → Adapty migrations. RevenueCat had its own notification URLs; those must be replaced with Adapty's.

Both production AND sandbox URLs must be configured. They are different URLs.

- [ ] In Adapty Dashboard, go to **App settings → iOS SDK → App Store Server Notifications** (or the page that currently documents ASSN — search `https://adapty.io/docs/` for "App Store Server Notifications" if the dashboard navigation has changed)
- [ ] Copy Adapty's **Production** ASSN URL
- [ ] Copy Adapty's **Sandbox** ASSN URL
- [ ] In App Store Connect → My Apps → your app → App Information → App Store Server Notifications:
  - [ ] Set the **Production Server URL** to Adapty's production URL
  - [ ] Set the **Sandbox Server URL** to Adapty's sandbox URL
  - [ ] Use the version that matches Adapty's current documentation (Apple supports v1 and v2; check what Adapty currently recommends — typically v2)
- [ ] If you had RevenueCat URLs configured previously, you are REPLACING them with Adapty's. Do not leave RevenueCat URLs configured — only one set of URLs can be active per environment.
- [ ] After saving, verify in Adapty Dashboard that ASSN test events are arriving (Adapty's dashboard has a test/verification flow for this)

⚠️ **Manual review item the skill always surfaces:** "App Store Server Notifications must be switched from RevenueCat's URLs to Adapty's URLs in App Store Connect → App Information. Both production and sandbox. This is the most-missed migration step and is invisible from the iOS app code — it can only be verified in App Store Connect + Adapty Dashboard."

## 4b. Google Play setup 🔴 MANDATORY for Flutter migrations targeting Android

This section applies only when the project targets Android (typically Flutter migrations that ship on Google Play, or Flutter projects with the Android module configured). Skip this section entirely on iOS-only migrations.

Google Play's equivalents of App Store Connect API + ASSN are: a **Google Play Developer API service account** (for product / subscription data) and **Real-Time Developer Notifications (RTDN)** via a Pub/Sub topic (for subscription state changes). Both are required for Adapty to keep server-side subscription state in sync.

- [ ] In Adapty Dashboard, go to **App settings → Android SDK → Google Play integration** (or the page that currently documents Google Play integration — search `https://adapty.io/docs/` for "Google Play integration" / "Android SDK installation" if the dashboard navigation has changed)
- [ ] Set the Android package name to **`<ANDROID_PACKAGE_NAME>`** — must match the app's actual `applicationId` in `android/app/build.gradle` exactly
- [ ] In Google Play Console → Setup → API access → Service accounts, create a service account with the Google Play Developer API permissions Adapty currently requires (verify the exact role set against current docs — Google has changed the required roles over time)
- [ ] Download the service account JSON key file and paste / upload it into Adapty's Google Play integration page
- [ ] Verify the integration page shows green / connected
- [ ] In Google Play Console → Monetization setup → Real-Time Developer Notifications, configure RTDN:
  - [ ] Copy Adapty's **Pub/Sub topic name** from Adapty Dashboard's Google Play integration page (verify the current location — search `https://adapty.io/docs/` for "RTDN" / "Real-Time Developer Notifications" if needed)
  - [ ] Paste the topic name into Google Play Console's RTDN settings for the app
  - [ ] If you had RevenueCat's RTDN topic configured previously, you are REPLACING it with Adapty's. Only one RTDN topic can be active per app.
  - [ ] Send a test notification (Google Play Console has a "Send test notification" button) and verify it arrives in Adapty Dashboard
- [ ] If the project uses license testing, add your tester Google accounts in Google Play Console → Setup → License testing

⚠️ **Manual review item the skill always surfaces (Flutter Android migrations):** "Google Play service account JSON and Real-Time Developer Notifications (RTDN) topic must be configured in Google Play Console. Switch the RTDN topic from RevenueCat's to Adapty's. Without RTDN, subscription state diverges from reality the same way ASSN does on iOS — silently, over days. This step is invisible from app code and can only be verified in Google Play Console + Adapty Dashboard."

## 5. Import products 🔴 MANDATORY

- [ ] Go to **Products** in Adapty Dashboard
- [ ] For each product the project uses (**`<PRODUCT_IDS>`**), either:
  - Import from App Store Connect / Google Play Console (preferred — auto-syncs metadata and prices), OR
  - Create the product manually with the matching store product ID (case-sensitive)
- [ ] Verify each product shows the correct localized title, price, and subscription duration
- [ ] If a product doesn't appear on iOS, App Store Connect integration (§2) is likely incomplete or the product isn't "Ready to Submit" / "Approved" in App Store Connect. On Android, check that Google Play integration (§4b) is connected and the product is "Active" in Google Play Console
- [ ] **Flutter projects that ship on both stores:** each product must exist in BOTH App Store Connect AND Google Play Console with the same product ID where possible. Adapty represents these as a single product if the IDs match exactly across stores; otherwise you'll have two separate Adapty products

## 6. Create access levels 🔴 MANDATORY

For each entitlement name from the RevenueCat code, create a matching Adapty access level. This is the closest 1:1 mapping in the migration — entitlement names map cleanly to access level names — but the code matches by exact string, so case mismatches silently deny access.

- [ ] Go to **Access Levels** in Adapty Dashboard
- [ ] For each name in **`<ACCESS_LEVEL_NAMES>`**, click "Add access level" and use the same string. Names are case-sensitive.
- [ ] For each access level, assign the products that grant it (same mapping as your RevenueCat entitlement-product mapping)
- [ ] If the project tracks multiple entitlements (e.g., `basic` + `pro`), each needs its own access level with its own product assignments

🔴 **Critical name-match warning:** the migrated Swift code references access levels by exact string (`profile.accessLevels["premium"]?.isActive`). If your Adapty access level is `Premium` but your code says `"premium"`, the check returns nil and silently denies access. Match case exactly.

## 7. Create placements ⚠️ MANUAL REVIEW REQUIRED

For each RevenueCat offering ID used in the code, create a matching Adapty placement. **BUT** — RevenueCat offerings and Adapty placements are not guaranteed 1:1 equivalents (see `revenuecat-to-adapty-map.md` §6). Review semantically before assuming the names map cleanly.

- [ ] Go to **Placements** in Adapty Dashboard
- [ ] For each name in **`<PLACEMENT_NAMES>`**, click "Add placement" with the same string ID
- [ ] **Manual review:** confirm each placement matches the semantic role of its source offering. RevenueCat offerings bundle packages; Adapty placements are display locations with one paywall each. If your project uses one offering across multiple contexts (onboarding + settings + paywall trigger), each context may need its own placement.
- [ ] Assign products to each placement (the products that should appear on that placement's paywall)

⚠️ **Same name-match warning as access levels:** the code says `Adapty.getPaywall(placementId: "premium")` — the placement ID in the dashboard must be exactly `"premium"`.

## 8. Build the paywall(s) 🔴 MANDATORY

For each placement, create the paywall:

- [ ] Go to **Paywalls** in Adapty Dashboard
- [ ] Click "Add paywall" and select the placement created in step 7
- [ ] **For Paywall Builder migrations** (`paywall-builder-path.md`): you will need to rebuild your paywall design from scratch in Adapty's visual builder — RevenueCat paywall layouts do not transfer. Use a template as a starting point, or recreate your existing design. The migration's `.paywall()` modifier renders whatever the visual builder produces.
- [ ] **For custom paywall migrations** (`custom-paywall-path.md`): you don't need the visual builder — your code renders the UI. Just configure the paywall metadata + assign products.
- [ ] Add the products from step 5 to the paywall
- [ ] If your original RevenueCat setup had a `paywallFooter` embedded layout: in the visual builder, set Layout → Embedded for this paywall
- [ ] **Important:** publish the paywall. Draft paywalls won't be returned by `getPaywallConfiguration` (silent failure).

## 9. "Show on device" toggle 🔴 MANDATORY for Paywall Builder migrations

This step is specific to migrations that include the Paywall Builder path. Skip it for migrations that are entirely custom-paywall.

The "Show on device" toggle controls whether a paywall's view configuration is sent to the SDK. Without it, `paywall.hasViewConfiguration` returns `false` and `AdaptyUI.getPaywallConfiguration` cannot render the paywall — the migrated code logs a warning and shows nothing. This is the single most common reason "the migration compiled but my paywall is blank."

- [ ] In Adapty Dashboard → Paywalls, open each Paywall Builder paywall created in §8
- [ ] Find the **"Show on device"** toggle (typically near the paywall name/header — the exact UI may shift between dashboard versions; search `https://adapty.io/docs/` for "Paywall Builder" / "Show on device" if unsure)
- [ ] Enable "Show on device" for each Paywall Builder paywall
- [ ] Save and verify the toggle stays on
- [ ] If a paywall is intentionally not yet ready to ship (still being designed), leave the toggle off — but expect the migrated code's `hasViewConfiguration` guard to log and skip rendering until you enable it

🔴 **Skill always surfaces this in Manual review:** "Paywall Builder paywall for placement `<id>` requires 'Show on device' enabled in Adapty Dashboard. Without it, the migrated `hasViewConfiguration` guard skips rendering and your paywall is invisible at runtime."

## 10. Get the Public SDK Key 🔴 MANDATORY

- [ ] Go to **App settings → API Keys** in Adapty Dashboard
- [ ] Copy the **Public SDK Key** (it starts with `public_live_...` for production or `public_test_...` for test)
- [ ] Open the migrated app's `@main` struct
- [ ] Replace `"YOUR_ADAPTY_PUBLIC_KEY"` in `try await Adapty.activate("YOUR_ADAPTY_PUBLIC_KEY")` with the actual key

🔴 **Never paste an SDK Secret Key here.** The activate call expects the Public SDK Key. Secret keys are server-side only and exposing them in client code is a security incident.

🔴 **Never reuse the old RevenueCat key value here.** RevenueCat keys and Adapty keys have different formats. Pasting one into the other compiles fine and silently breaks the integration with no error message — Adapty rejects the malformed key server-side.

## 11. Attribution integrations (only if project uses MMP) ⚠️ MANDATORY IF DETECTED

Only required if the project integrates an attribution provider (**`<ATTRIBUTION_PROVIDERS>`** — typically Adjust, AppsFlyer, Branch, or AppMetrica). If no MMP code was detected, skip this section.

Adapty's attribution integration requires both dashboard configuration AND specific code-level sequencing — see `revenuecat-to-adapty-map.md` §12.

- [ ] Go to **App settings → Integrations** in Adapty Dashboard
- [ ] For each provider the project uses, configure the matching Adapty integration:
  - Adjust: provide Adjust App Token + Environment
  - AppsFlyer: provide AppsFlyer Dev Key + App ID
  - Branch: provide Branch Key
  - AppMetrica: provide AppMetrica API Key
  - Other: search current Adapty docs at `https://adapty.io/docs/` for "attribution" for the current provider list
- [ ] Verify the integration shows green / connected
- [ ] In the migrated code, confirm `Adapty.updateAttribution(..., source: .<provider>)` is called from inside the provider's attribution callback (NOT directly after `Adapty.activate`)
- [ ] Confirm the provider's customer user ID (if used) is consistent with the Adapty customer ID — see `revenuecat-to-adapty-map.md` §12 ordering notes

⚠️ **Without dashboard configuration**, the code-side `Adapty.updateAttribution` calls succeed locally but the attribution data is dropped server-side. There is no error. The data is just missing in your attribution reports.

## 12. Webhooks / Raw events forwarding (optional but recommended) ⚠️ REVIEW

If your team currently consumes RevenueCat webhooks (for analytics, CRM enrichment, server-side feature flags, or any downstream system), the equivalent in Adapty is **Integrations + Webhooks**.

This is where projects most often drop data during migration: code keeps working but the downstream pipeline goes silent because nothing is forwarding events anymore.

- [ ] Go to **App settings → Integrations** in Adapty Dashboard
- [ ] If you had RevenueCat webhooks pointing to your backend, configure Adapty's webhook integration with the same URL:
  - Custom Webhook → enter your endpoint URL
  - Map the event types you previously consumed from RevenueCat to Adapty's event types (search current Adapty docs at `https://adapty.io/docs/` for "webhook" / "events" for the current taxonomy — names differ from RevenueCat's)
- [ ] If you forward events to analytics platforms (Amplitude, Mixpanel, Branch, AppsFlyer, Firebase, etc.), Adapty has built-in integrations for many of these in the same Integrations page. Configure each one your team currently uses.
- [ ] If you use Adapty's **raw events** export (for data warehousing — BigQuery, Snowflake, S3), enable that integration too. RevenueCat has a different raw-data export setup; you're switching pipelines.
- [ ] Test by triggering a sandbox purchase and verifying the event arrives at each downstream destination

⚠️ **Manual review item the skill always surfaces if it sees signs of downstream usage** (webhook URLs in environment configs, analytics SDK initialization, server-side entitlement checks): "Confirm webhook / event forwarding is configured in Adapty Dashboard → Integrations. RevenueCat's downstream event pipeline does NOT carry over automatically — every integration must be re-configured on the Adapty side. Run a sandbox purchase and verify the event arrives at each downstream system before going live."

## 13. Test on sandbox 🔴 MANDATORY before going live

- [ ] Build and run the migrated app on a real device (sandbox / test purchases don't work reliably on simulators or emulators)
- [ ] **iOS / Apple-store Flutter:** sign in to App Store with a fresh sandbox tester account (old testers may have stuck subscription state from previous tests)
- [ ] **Android / Google-Play Flutter:** use a tester account added to Google Play Console → License testing. Old subscription state on the account can interfere — cancel and refund any test subscriptions before re-testing
- [ ] Trigger the paywall — it should load and display products
- [ ] **Paywall Builder migrations:** the paywall UI should render via AdaptyUI. If it doesn't, check `hasViewConfiguration` log line (§9 — "Show on device") and Xcode console for `AdaptyUI` errors.
- [ ] **Custom paywall migrations:** the product list should populate in your custom UI. If empty, check products are assigned to the placement (§7) and to the paywall (§8).
- [ ] Make a sandbox purchase — `didFinishPurchase` (Paywall Builder) or `Adapty.makePurchase` return value (custom) should fire, and `isPremium` should flip to `true`
- [ ] Force-quit and relaunch the app — `isPremium` should still be `true` (verifies server-side state via §3 shared secret and §4 ASSN)
- [ ] Test restore — `didFinishRestore` (Paywall Builder) or `Adapty.restorePurchases` (custom) should fire and state should be reconciled
- [ ] If anything is off, check Xcode console for the `✅` / `❌` / `⚠️` log lines from `AdaptyManager` and the paywall callbacks — they indicate which step is failing

## 14. RevenueCat shutdown (only after Adapty is verified)

Only after Adapty is verified working in production:

- [ ] Remove RevenueCat from billing in your RevenueCat dashboard
- [ ] Remove RevenueCat ASSN URLs from App Store Connect (already overwritten in §4, but verify) — iOS / Apple-store Flutter only
- [ ] Remove RevenueCat's RTDN topic from Google Play Console (already overwritten in §4b, but verify) — Android / Google-Play Flutter only
- [ ] Archive or delete the RevenueCat project
- [ ] Remove any RevenueCat-related backend webhooks and SDK API keys from your secrets manager
- [ ] Document the cut-over date for your team

🔴 Do NOT do step 14 before confirming Adapty works in production. Running both SDKs in parallel for at least one release cycle is the safe migration path. Some teams run them in parallel for a full subscription cycle (1 week / 1 month) to ensure all server-side state — renewals, refunds, grace periods — flows correctly through Adapty.

---

## Common dashboard-related runtime failures, in order of frequency

These are the most common reasons a "correctly migrated" Adapty integration fails at runtime:

1. **"Show on device" disabled** — Paywall Builder paywall returns `hasViewConfiguration` false/empty; UI renders nothing. (§9)
2. **App Store Server Notifications not switched** (iOS / Apple-store Flutter) — subscription state diverges from reality over days/weeks. (§4)
3. **Google Play RTDN topic not switched** (Android / Google-Play Flutter) — same silent divergence on Android. (§4b)
4. **Placement not published** — paywall exists in dashboard but is in draft state → paywall fetch returns no view configuration. (§8)
5. **Access level name mismatch** — code says `"premium"`, dashboard has `"Premium"` → silent denial of access. (§6)
6. **Wrong API key** — using the SDK Secret Key instead of the Public SDK Key, or pasting the RevenueCat key, or using the test key in production. (§10)
7. **Products not imported** — products exist in App Store Connect / Google Play Console but haven't been pulled into Adapty. (§5)
8. **Placement-product assignment missing** — paywall has no products to show because none were attached to the placement. (§7)
9. **Shared secret missing** (iOS) — receipt validation silently fails server-side. (§3)
10. **Google Play service account JSON missing or expired** (Android) — product data can't be read; products show as unavailable. (§4b)
11. **Attribution integration code-only** — attribution call succeeds but dashboard integration not configured → data dropped. (§11)
12. **Webhooks not re-configured** — downstream analytics / CRM goes silent without warning. (§12)
13. **Sandbox / test account stuck** — old subscriptions from previous tests interfering; create a fresh sandbox tester (iOS) or clear license-testing subscription state (Android). (§13)
