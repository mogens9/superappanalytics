---
name: superappanalytics
description: Install, repair, or verify SuperAppAnalytics in a StoreKit iOS app. Use when an agent needs to connect an Xcode project to SuperAppAnalytics, add the Swift package and StoreKit 2 tracking, associate purchases with signed-in customers, configure App Store Server Notifications V2, or diagnose an existing SuperAppAnalytics integration.
---

# SuperAppAnalytics

Integrate StoreKit analytics through the deterministic installer, then adapt customer identity to the app's existing architecture and verify the result in Xcode.

## Workflow

1. Inspect the repository before editing.
   - Locate the `.xcodeproj`, application target, `@main` entry point, authentication/session model, and StoreKit purchase code.
   - Map the app's main product journey from source: onboarding, authentication, paywall, purchase triggers, and the important actions between install and paid conversion.
   - Map onboarding/profile data collection from source: form fields, view models, reducers, persistence models, API request bodies, and profile update calls.
   - Read `git status` and preserve unrelated changes.
   - Require macOS with Xcode and Node.js 20 or later.

2. Create a custom funnel event plan for this specific app.
   - Do not use a fixed generic list. Choose events from the app's actual screens, actions, services, reducers, view models, and product language.
   - Prefer high-signal events that explain conversion: onboarding step reached/completed, account created, paywall viewed, product-specific input created/uploaded, primary CTA pressed, generation/job started, result shown, export/share/save completed, and equivalent domain-specific milestones.
   - Use stable snake_case names such as `photo_uploaded`, `generate_pressed`, `generation_completed`, `workout_started`, or `journal_entry_saved`.
   - Keep the first pass focused: normally 5-12 events. Avoid noisy low-value taps, purely decorative navigation, or events that can fire repeatedly during rendering.
   - Create `.superappanalytics/` if needed, then write `.superappanalytics/funnel-events.json` before running the installer:

     ```json
     {
       "events": [
         {
           "name": "photo_uploaded",
           "reason": "User selected an input photo before generation"
         },
         {
           "name": "generate_pressed",
           "reason": "Primary AI generation CTA"
         }
       ]
     }
     ```

3. Run from the iOS project root:

   ```sh
   npx superappanalytics install --funnel-events .superappanalytics/funnel-events.json
   ```

   Let the user complete browser login when requested. Do not ask for app IDs, client tokens, organizations, or bundle IDs unless automatic detection explicitly fails.

4. Review the installer result.
   - Confirm the detected app name, target, and bundle ID are correct.
   - Confirm `.superappanalytics/SuperAppAnalyticsSDK`, `SuperAppAnalyticsBootstrap.swift`, the Xcode package dependency, and the startup call were added.
   - Confirm the generated `SuperAppAnalyticsFunnel` helpers match the custom funnel plan.
   - Treat `.superappanalytics/project.pbxproj.backup` as recovery data. Never overwrite user work with it automatically.

5. Connect customer identity when the app has accounts.
   - Use the app's stable backend user ID, never a display name or mutable email address.
   - Replace the generated `CurrentCustomerProvider` placeholder when the existing session API makes the ID unambiguous.
   - Call `identify(customerId:attributes:)` after successful login or signup, including safe onboarding/profile attributes when available.
   - Add `appAccountToken(SuperAppAnalytics.appAccountToken(for: customerId))` to existing StoreKit purchase options so server notifications map back to the same customer.
   - Preserve the app's entitlement delivery and transaction-finishing behavior. Do not finish a transaction twice or replace a working purchase manager unnecessarily.
   - Leave anonymous tracking enabled when the app has no account system.

6. Add customer attributes from onboarding/profile data.
   - Choose attributes by analyzing the actual app. Good candidates are structured, low-sensitivity fields that explain conversion and segmentation, such as `age_range`, `country`, `language`, `goal`, `use_case`, `fitness_level`, `selected_plan`, `onboarding_variant`, or app-specific preference buckets.
   - Normalize attributes before sending:
     - Use stable snake_case keys.
     - Convert values to strings.
     - Prefer buckets/enums over raw values, e.g. `18_24` instead of exact age when possible.
     - Keep the set small and useful, normally 3-10 attributes.
   - Instrument at the source-of-truth point where onboarding/profile data is saved or submitted, not in view rendering.
   - If the customer id is known, call:

     ```swift
     try? await SuperAppAnalytics.shared.identify(
       customerId: customerId,
       attributes: [
         "age_range": ageRange,
         "country": countryCode,
         "goal": selectedGoal
       ]
     )
     ```

   - If the app has no account system yet, attach the attributes to the current anonymous customer by using the SDK's current anonymous identity through normal event tracking only when safe, or defer until a stable customer id exists.
   - Never send sensitive personal data or free-form user content as attributes: no names, emails, phone numbers, precise birth dates, addresses, health details, prompts, uploaded content, photos, secrets, tokens, or raw text answers. If a field is sensitive but useful analytically, send a coarse non-identifying bucket only.

7. Instrument the custom funnel events.
   - Add calls at the real source-of-truth event locations, not just in UI rendering code.
   - Use generated helpers when available:

     ```swift
     Task {
       await SuperAppAnalyticsFunnel.generatePressed(customerId: currentUserId)
     }
     ```

   - If a helper was not generated, use the generic API:

     ```swift
     Task {
       try? await SuperAppAnalytics.shared.trackFunnelStep("domain_specific_event", customerId: currentUserId)
     }
     ```

   - For async workflows, distinguish intent and result when both matter: e.g. `generate_pressed` when the user starts generation, `generation_completed` only after a successful result.
   - Include small string payloads only when they materially help analysis, such as `screen`, `template`, `source`, or `model`. Do not send private user content, images, prompts, free-form text, API keys, or secrets.
   - Keep customer identity consistent with the purchase mapping.

8. Verify the integration.
   - Resolve Swift packages and build the application target for an iOS Simulator.
   - Fix integration errors without changing unrelated application behavior.
   - Run the installer a second time when practical and confirm it makes no duplicate project or startup changes.
   - Explain that StoreKit Testing in Xcode and Apple Sandbox are recorded as `sandbox`; App Store purchases are recorded as `production` automatically.

9. Report the result.
   - List the files changed and the successful build command.
   - List the custom funnel events added and where they were instrumented.
   - List the customer attributes added, their source fields, and any attributes intentionally skipped as too sensitive or free-form.
   - Return the generated App Store Server Notifications V2 URL and tell the user to configure it for both Production and Sandbox in App Store Connect.
   - Surface any customer-ID or purchase-flow step that could not be completed safely.

## Guardrails

- Use the installer as the source of truth instead of manually recreating package-project mutations.
- Do not embed Firebase administrator credentials, App Store Connect private keys, or dashboard user credentials in the client app.
- Do not classify an environment from a user-facing toggle. The SDK and verified Apple payload determine it; the dashboard selector only changes which data is viewed.
- Do not claim server-authoritative renewals, cancellations, billing issues, or refunds are complete until App Store Server Notifications V2 is configured.
- Do not claim full-funnel tracking is complete unless the custom event callsites were actually inserted and built successfully.
- Do not claim customer attribute tracking is complete unless the attribute callsite was actually inserted, uses safe non-sensitive values, and builds successfully.
