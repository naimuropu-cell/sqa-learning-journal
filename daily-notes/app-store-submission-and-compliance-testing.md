# Mobile App Store Submission & Compliance Testing Checklist

## Introduction

Building a feature-complete, bug-free mobile application is only half the battle. Before an iOS or Android application can reach end users, it must pass the rigorous manual and automated review processes of the **Apple App Store Review Guidelines** and the **Google Play Policy Center**.

Having a release rejected by Apple or Google review teams delays product launches by days or weeks, disrupts marketing campaigns, and requires emergency engineering refactors.

QA engineers must execute a dedicated **App Store Pre-Submission Compliance Audit** to guarantee that applications satisfy all legal, privacy, monetization, and platform guidelines before binary submission.

---

## The Top Reasons for App Store Rejections

```
┌─────────────────────────────────────────────────────────────┐
│                 Top App Store Rejection Causes              │
├─────────────────────┬───────────────────────────────────────┤
│ Rejection Reason    │ Apple / Google Policy Rule            │
├─────────────────────┼───────────────────────────────────────┤
│ 1. Missing Account  │ Apple Guideline 5.1.1(v): Apps allowing│
│    Deletion Flow    │ account creation MUST allow users to  │
│                     │ initiate account deletion in-app      │
├─────────────────────┼───────────────────────────────────────┤
│ 2. Missing Privacy  │ iOS Info.plist usage descriptions     │
│    Strings          │ (Camera, Location, Microphone) cannot │
│                     │ be blank or generic ("For app usage") │
├─────────────────────┼───────────────────────────────────────┤
│ 3. In-App Purchase  │ Digital goods/subscriptions MUST use  │
│    Violations       │ Apple/Google IAP (Stripe is prohibited│
│                     │ for digital unlocks!)                 │
├─────────────────────┼───────────────────────────────────────┤
│ 4. Crashes on Launch│ Any crash during reviewer testing     │
│                     │ triggers an immediate hard rejection  │
├─────────────────────┼───────────────────────────────────────┤
│ 5. Invalid Reviewer │ Providing expired test credentials or │
│    Credentials      │ geo-blocked login prevents review     │
└─────────────────────┴───────────────────────────────────────┘
```

---

## The Pre-Submission Compliance Checklist

### 1. Mandatory In-App Account Deletion Flow
* **The Rule**: If your application allows users to create an account, it must provide a clear, easily discoverable option within account settings to delete their account and personal data entirely.
* **QA Test**: Verify the deletion workflow:
  * User taps "Delete Account" ➔ Confirmation modal appears with clear warnings.
  * Account data is purged or scheduled for deletion.
  * Session is revoked and user is returned to the login screen.

### 2. Permissions & Privacy Description Strings (iOS `Info.plist`)
When an app requests access to device hardware, iOS displays a permission prompt displaying a custom explanation string configured in `Info.plist`:
* ❌ **Rejection**: `NSCameraUsageDescription: "Camera access"` (Generic string = immediate rejection).
* ✅ **Compliant**: `NSCameraUsageDescription: "Camera access is needed to scan QR codes and capture profile photos."`
* **QA Test**: Trigger every permission dialog (Camera, Microphone, Photos, Location, Bluetooth) and verify that the explanatory text clearly informs the user why the data is needed.

### 3. In-App Purchases (IAP) vs. Third-Party Payments
* **Digital Content (E-books, subscriptions, game credits)**: MUST exclusively use Apple In-App Purchase / Google Play Billing. Using external payment processors (Stripe, PayPal) for digital features violates store terms.
* **Physical Goods & Real-World Services (Clothing, food delivery, Uber rides)**: MUST use third-party payment gateways (Stripe, Braintree). Apple IAP is prohibited for physical goods.

### 4. Reviewer Demo Account & Environmental Preparation
* Reviewers test applications from Apple (Cupertino, California) and Google (Mountain View, California).
* **QA Checklist**:
  * Verify that test credentials (`reviewer@company.com`) work without requiring SMS two-factor authentication (provide a static bypass OTP for reviewers).
  * Verify that backend servers do not geo-block US IP addresses.
  * Ensure the test account has pre-seeded data (e.g., active orders, products) so reviewers see a live, populated application.

---

## Pre-Release Testing Tracks (TestFlight & Internal Testing)

Before submitting for public review, QA must test the production release build (AOT-compiled, minified, signed with production certificates):

* **Apple TestFlight**: Distribute the exact App Store `.ipa` build to internal team members and external beta testers to verify push notifications, in-app purchases (in sandbox mode), and release performance.
* **Google Play Internal Testing Track**: Distribute the `.aab` (Android App Bundle) to verify Google Play integrity, installation speed, and dynamic feature delivery.

---

## SQA Interview Questions & Answers

### Q: What is the App Tracking Transparency (ATT) framework on iOS, and what must QA test?
**Answer:**
Apple's ATT framework mandates that iOS apps request explicit user permission before tracking their activity across other companies' apps and websites for advertising purposes. QA must verify that:
1. The ATT prompt (`requestTrackingAuthorization`) appears *before* any third-party tracking SDK (Facebook SDK, Adjust, AppsFlyer) initializes or transmits IDFA advertising identifiers.
2. If the user selects "Ask App not to Track," all tracking beacons are strictly suppressed.

### Q: What is Apple's Guideline 4.2 regarding "Minimum Functionality"?
**Answer:**
Guideline 4.2 states that an app must provide compelling, differentiated functionality beyond what could be achieved with a standard mobile website. Apps that are simply "wrapped websites" (a WebView pointing to a responsive web page with no native hardware integration, offline support, or native navigation) will be rejected for lack of minimum functionality.

---

## Key Takeaways

* Account deletion must be self-serve and accessible inside the app.
* Provide clear, descriptive justification strings for all hardware permissions in `Info.plist`.
* Supply functional, pre-seeded reviewer accounts without SMS 2FA hurdles to prevent review delays.

---

## Conclusion

App Store compliance testing is a vital quality gate that protects product launch schedules. By methodically auditing privacy strings, account deletion workflows, and monetization guidelines on TestFlight and Google Play testing tracks, QA engineers ensure smooth, first-time approval for public app releases.
