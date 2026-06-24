---
name: superappanalytics
description: Install, repair, or verify SuperAppAnalytics in a StoreKit iOS app. Use when an agent needs to connect an Xcode project to SuperAppAnalytics, add the Swift package and StoreKit 2 tracking, associate purchases with signed-in customers, configure App Store Server Notifications V2, or diagnose an existing SuperAppAnalytics integration.
---

# SuperAppAnalytics

Integrate StoreKit analytics through the deterministic installer, then adapt customer identity to the app's existing architecture and verify the result in Xcode.

## Workflow

1. Inspect the repository before editing.
   - Locate the `.xcodeproj`, application target, `@main` entry point, authentication/session model, and StoreKit purchase code.
   - Read `git status` and preserve unrelated changes.
   - Require macOS with Xcode and Node.js 20 or later.

2. Run from the iOS project root:

   ```sh
   npx superappanalytics install
   ```

   Let the user complete browser login when requested. Do not ask for app IDs, client tokens, organizations, or bundle IDs unless automatic detection explicitly fails.

3. Review the installer result.
   - Confirm the detected app name, target, and bundle ID are correct.
   - Confirm `.superappanalytics/SuperAppAnalyticsSDK`, `SuperAppAnalyticsBootstrap.swift`, the Xcode package dependency, and the startup call were added.
   - Treat `.superappanalytics/project.pbxproj.backup` as recovery data. Never overwrite user work with it automatically.

4. Connect customer identity when the app has accounts.
   - Use the app's stable backend user ID, never a display name or mutable email address.
   - Replace the generated `CurrentCustomerProvider` placeholder when the existing session API makes the ID unambiguous.
   - Call `identify(customerId:attributes:)` after successful login or signup.
   - Add `appAccountToken(SuperAppAnalytics.appAccountToken(for: customerId))` to existing StoreKit purchase options so server notifications map back to the same customer.
   - Preserve the app's entitlement delivery and transaction-finishing behavior. Do not finish a transaction twice or replace a working purchase manager unnecessarily.
   - Leave anonymous tracking enabled when the app has no account system.

5. Verify the integration.
   - Resolve Swift packages and build the application target for an iOS Simulator.
   - Fix integration errors without changing unrelated application behavior.
   - Run the installer a second time when practical and confirm it makes no duplicate project or startup changes.
   - Explain that StoreKit Testing in Xcode and Apple Sandbox are recorded as `sandbox`; App Store purchases are recorded as `production` automatically.

6. Report the result.
   - List the files changed and the successful build command.
   - Return the generated App Store Server Notifications V2 URL and tell the user to configure it for both Production and Sandbox in App Store Connect.
   - Surface any customer-ID or purchase-flow step that could not be completed safely.

## Guardrails

- Use the installer as the source of truth instead of manually recreating package-project mutations.
- Do not embed Firebase administrator credentials, App Store Connect private keys, or dashboard user credentials in the client app.
- Do not classify an environment from a user-facing toggle. The SDK and verified Apple payload determine it; the dashboard selector only changes which data is viewed.
- Do not claim server-authoritative renewals, cancellations, billing issues, or refunds are complete until App Store Server Notifications V2 is configured.
