# Mobile Deep Linking & Universal Links Testing Guide for QA

## Introduction

In mobile marketing and customer communications (email campaigns, SMS marketing, social media advertisements), sending a customer to the generic home screen of a mobile app creates friction and causes high drop-off rates.

**Deep Linking** is the technology that enables a link to navigate a user directly to a specific in-app screen or resource—such as a specific product page, a pre-filled promo discount code, or an order tracking screen.

Testing deep links requires validating operating system routing, web association manifests (`apple-app-site-association` and `assetlinks.json`), cold launch state handling, and **Deferred Deep Linking**.

---

## Deep Linking Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                 Mobile Deep Link Technologies               │
├─────────────────────┬───────────────────┬───────────────────┤
│ Type                │ Platform          │ Mechanism         │
├─────────────────────┼───────────────────┼───────────────────┤
│ 1. Custom URL Scheme│ iOS & Android     │ myapp://product/45│
│                     │ (Legacy / Simple) │ Fallback prone ⚠️ │
├─────────────────────┼───────────────────┼───────────────────┤
│ 2. Universal Links  │ iOS (Apple)       │ https://shop.com/ │
│                     │                   │ Verified HTTPS    │
├─────────────────────┼───────────────────┼───────────────────┤
│ 3. App Links        │ Android (Google)  │ https://shop.com/ │
│                     │                   │ SHA-256 Verified  │
├─────────────────────┼───────────────────┼───────────────────┤
│ 4. Deferred Deep    │ iOS & Android     │ Passes target link│
│    Links            │ (Branch / Adjust) │ THROUGH App Store │
└─────────────────────┴───────────────────┴───────────────────┘
```

---

## What is Deferred Deep Linking?

Standard deep links only work if the app is already installed on the phone. **Deferred Deep Linking** handles the first-time user journey:

```
User clicks link on Instagram: [ https://shop.com/item/101 ]
                      │
                      ▼
        Is App currently installed?
           ├── YES ──► Opens App directly to Item #101 ✅
           └── NO
                 │
                 ▼
         Redirects to Apple App Store / Google Play
                 │
                 ▼
         User installs and opens app for the first time
                 │
                 ▼
         SDK detects deferred link ──► Routes to Item #101! ✅
```

---

## Testing Deep Links via Command Line (CLI)

QA engineers can automate deep link testing on emulators and real devices without manually clicking links in third-party email apps:

### 1. Android ADB Intent Launch:
```bash
# Test Custom Scheme on connected Android device/emulator
adb shell am start -W -a android.intent.action.VIEW \
  -d "myapp://products/sneakers-402?promo=SAVE20" com.mycompany.app

# Test Android App Link (HTTPS)
adb shell am start -W -a android.intent.action.VIEW \
  -d "https://mycompany.com/products/sneakers-402" com.mycompany.app
```

### 2. iOS Simctl URL Launch:
```bash
# Test Universal Link or Scheme on booted iOS Simulator
xcrun simctl openurl booted "https://mycompany.com/products/sneakers-402"
```

---

## Key Testing Scenarios for QA

### 1. Cold Launch vs. Warm Launch
* **Warm Launch**: App is already running in background RAM. Tapping the link brings the app to the foreground and navigates directly to the target view controller.
* **Cold Launch**: App is completely terminated (`SIGKILL`). Tapping the link must initialize the entire app bootstrap, bypass or restore authentication, and navigate to the target screen without losing URL parameters.

### 2. Authentication Gates
* What happens when an unauthenticated user clicks a deep link requiring login (e.g., `myapp://account/billing`)?
* **QA Test**: The app should prompt for login, and *after successful authentication*, it must complete navigation to the originally requested billing screen rather than dumping the user on the home screen.

### 3. Malformed and Expired Link Handling
* Test links with invalid IDs (`myapp://products/invalid_9999`) or expired promo tokens.
* **Expected Result**: App displays a friendly error banner ("Product no longer available") and falls back gracefully to the catalog without crashing.

---

## SQA Interview Questions & Answers

### Q: What is the primary difference between a Custom URL Scheme and an iOS Universal Link?
**Answer:**
* **Custom URL Schemes** (e.g., `myapp://`) have no centralized domain ownership verification. If two apps on a phone register the same scheme (`shop://`), iOS displays an ambiguous prompt or randomly routes the intent to the wrong app.
* **Universal Links** use standard HTTPS URLs (e.g., `https://shop.com/product/10`). Apple verifies domain ownership via an `apple-app-site-association` (AASA) JSON file hosted on the server's root domain. If the app is installed, the link opens the app securely without launching Safari; if not installed, it opens the web page seamlessly.

### Q: How do you verify domain association files in QA?
**Answer:**
Inspect the domain's hosted manifest via browser or curl:
* iOS: `curl -I https://domain.com/.well-known/apple-app-site-association`
* Android: `curl -I https://domain.com/.well-known/assetlinks.json`
Verify that the response returns `HTTP 200`, `Content-Type: application/json`, and contains the accurate App ID, Team ID, and SHA-256 certificate fingerprints.

---

## Key Takeaways

* Deep links navigate users directly to specific in-app views, maximizing marketing conversions.
* Use `adb` and `xcrun simctl` to test deep links deterministically via CLI.
* Verify both Cold Launch (app terminated) and Warm Launch (app in background) states.

---

## Conclusion

Deep linking and universal links are essential for creating cohesive, friction-free mobile user experiences. By validating link parameters, domain association manifests, and cold launch routing across iOS and Android, QA engineers guarantee flawless navigation journeys for all mobile users.
