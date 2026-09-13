# Progressive Web App (PWA) & Offline Mode Testing Guide

## Introduction

Historically, web applications required an active internet connection. If a user lost connectivity, the browser immediately displayed the browser's dinosaur offline screen, losing all in-flight form data.

**Progressive Web Apps (PWAs)** bridge the capability gap between native mobile apps and responsive web applications. Built using **Service Workers**, the **Cache Storage API**, and **Web App Manifests**, PWAs are installable onto mobile and desktop home screens, support background push notifications, and—most importantly—function reliably in poor network conditions or completely **Offline**.

Testing PWAs requires specialized methodologies to evaluate caching strategies, offline background synchronization, and service worker update lifecycles.

---

## The Three Core Pillars of a PWA

```
┌─────────────────────────────────────────────────────────────┐
│                       The PWA Triad                         │
├─────────────────────┬───────────────────────────────────────┤
│ Pillar              │ Functionality Tested by QA            │
├─────────────────────┼───────────────────────────────────────┤
│ 1. Web App Manifest │ JSON file controlling home screen icon│
│    (manifest.json)  │ splash screen, theme color, display   │
├─────────────────────┼───────────────────────────────────────┤
│ 2. Service Worker   │ JavaScript worker acting as a client- │
│    (sw.js)          │ side network proxy in the background  │
├─────────────────────┼───────────────────────────────────────┤
│ 3. Storage API      │ Cache Storage (assets) and IndexedDB  │
│    (Cache/IndexedDB)│ (structured offline application state)│
└─────────────────────┴───────────────────────────────────────┘
```

---

## Service Worker Caching Strategies Tested by QA

A Service Worker intercepts every HTTP request departing the browser. QA must verify that each asset type uses the appropriate caching strategy:

```
1. Cache-First (Static Assets: Images, Fonts, CSS)
   Browser ──► Service Worker ──(Found in Cache?)──► Return Cache ✅
                                      │ (Not in cache)
                                      ▼
                                Fetch from Network

2. Network-First (Dynamic API Data: Stock prices, messages)
   Browser ──► Service Worker ──► Fetch Network ──(Success?)──► Return Data & Update Cache
                                        │ (Offline / 500 error)
                                        ▼
                                 Return Stored Cache

3. Stale-While-Revalidate (Dashboards, Feeds)
   Browser ◄── Return Stored Cache IMMEDIATELY (Instant UI render!)
   Service Worker ──► Fetch fresh network data in background ──► Update Cache & UI
```

---

## Key Offline Mode Test Scenarios

### 1. Offline Form Submission & Background Sync
* **The Scenario**: A field technician submits an inspection report or an e-commerce user adds an item to cart while driving through a tunnel without cellular signal.
* **QA Test**:
  1. Toggle **Offline** mode in browser DevTools.
  2. Fill out and submit the inspection form.
  3. **Assertion 1**: UI must not crash or display an HTTP error; it should display an informative toast: *"Saved offline. Will synchronize once online."*
  4. Inspect DevTools **IndexedDB**: Verify the form payload is stored in the local outbox queue.
  5. Toggle network back to **Online**.
  6. **Assertion 2**: Service Worker triggers `sync` event, posts the queued payload to the backend server, and updates the UI indicator to *"Synchronized ✅"*.

### 2. Service Worker Lifecycle & Updates (Cache Invalidation)
* **The Danger**: If a service worker is aggressively cached, users will be permanently stuck on an obsolete version of the web app even after developers deploy bug fixes.
* **QA Test**: Deploy a new version. Verify that:
  * The browser detects the updated `sw.js` file.
  * A banner appears: *"A new version is available. Click to refresh."*
  * Clicking refresh activates the new worker and purges obsolete cache buckets.

---

## Automated PWA Testing in Playwright

Playwright allows simulating offline state and validating service worker caching:

```typescript
import { test, expect } from '@playwright/test';

test.describe('PWA Offline Verification Suite', () => {
  test('App loads cached catalog and navigates when completely offline', async ({ context, page }) => {
    // 1. Visit app online to prime the Service Worker cache
    await page.goto('https://pwa.example.com');
    await page.waitForLoadState('networkidle');

    // Wait for Service Worker to activate
    await page.waitForFunction(() => !!navigator.serviceWorker.controller);

    // 2. Simulate complete network disconnection
    await context.setOffline(true);

    // 3. Reload page while offline
    await page.reload();

    // 4. Assert UI still renders successfully from Cache Storage
    await expect(page.locator('#product-catalog')).toBeVisible();
    await expect(page.locator('.offline-status-banner')).toHaveText('Operating in Offline Mode');

    // 5. Reconnect network
    await context.setOffline(false);
    await expect(page.locator('.offline-status-banner')).toBeHidden();
  });
});
```

---

## SQA Interview Questions & Answers

### Q: What is the purpose of the Web App Manifest in a PWA?
**Answer:**
The Web App Manifest (`manifest.json`) is a JSON metadata file that informs the browser how the web app should behave when installed on a user's mobile or desktop device. It defines the app's official name, home screen app icons, theme colors, orientation, start URL, and display mode (e.g., `standalone`, which removes the browser URL address bar to give the app a native look and feel).

### Q: How do you verify that an application satisfies PWA standards?
**Answer:**
By running **Google Lighthouse Audits** in Chrome DevTools or via CI/CD. The Lighthouse PWA audit verifies that the application is served over HTTPS, registers a functional Service Worker with an offline `fetch` handler, provides a valid Web App Manifest with proper icons, and responds with `HTTP 200` when offline.

---

## Key Takeaways

* PWAs provide native app-like capabilities: home screen installation, offline support, and push alerts.
* Test that Service Workers implement appropriate caching strategies (Cache-First vs. Network-First).
* Verify offline queues and background synchronization when toggling between offline and online states.

---

## Conclusion

PWA and offline testing guarantee that modern web applications remain resilient regardless of network connectivity. By validating service worker lifecycles, caching mechanics, and background synchronization, QA engineers ensure that users stay productive anywhere, anytime.
