# Progressive Web App (PWA) Offline Mode, Service Worker & CacheStorage Testing

## 1. Overview of PWA Testing for QA Engineers

Progressive Web Applications (PWAs) deliver app-like user experiences on mobile and desktop web browsers. Unlike traditional web applications that completely fail when an internet connection drops, PWAs leverage **Service Workers** (background scripts acting as programmable network proxies) and the **CacheStorage API** to provide resilient offline access.

QA engineers must validate:
- **Service Worker Lifecycle Transitions**: Registration $\rightarrow$ Installation $\rightarrow$ Activation $\rightarrow$ Updates.
- **Offline Caching Strategies**: Stale-While-Revalidate, Cache-First, and Network-First.
- **Background Sync**: Queuing mutations (e.g., submitting an offline form) and verifying automatic synchronization upon network restoration.
- **Web App Manifest**: Compliance with PWA installability criteria (`manifest.json`).

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Service Worker Proxy Architecture               │
│                                                                        │
│   Web Page (DOM Window)                                                │
│         │                                                              │
│         │ fetch('/api/user-profile')                                   │
│         ▼                                                              │
│   Service Worker (Background Worker Thread)                            │
│         │                                                              │
│         ├───► Strategy: Cache-First?                                   │
│         │        ├─► HIT: Return from CacheStorage (Instant, 0ms)      │
│         │        └─► MISS: Fetch from Network                          │
│         │                                                              │
│         └───► Strategy: Network-First?                                 │
│                  ├─► Online: Fetch from Cloud Server                   │
│                  └─► Offline: Fallback to Offline HTML / CacheStorage  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Common Caching Strategies & QA Assertions

| Strategy Name | How It Works | Target Use Case | Failure Risk to Test |
| :--- | :--- | :--- | :--- |
| **Cache First** | Checks cache; falls back to network only on cache miss | Static assets (JS bundles, CSS, fonts, logos) | Users stuck on stale code if cache busting/hashes fail |
| **Network First** | Tries network; falls back to cache if offline/timeout | User profiles, recent messages, dashboards | Stale cached data displayed without offline warning banner |
| **Stale While Revalidate** | Returns cached version immediately, fetches update in background | News feeds, product catalogs | Discrepancies between visible UI and synced backend |
| **Network Only** | Never caches responses | Financial transactions, checkout, authentication | App crashes or shows generic browser error when offline |

---

## 3. Automated PWA Testing with Playwright

Using Playwright's `browserContext.setOffline()` and Chrome DevTools Protocol (CDP), QA engineers can automate offline transition assertions:

```typescript
import { test, expect } from '@playwright/test';

test.describe('PWA Offline & Service Worker Verification Suite', () => {
  test('Service Worker registers and caches static assets for offline navigation', async ({ context, page }) => {
    // 1. Navigate online and allow Service Worker to activate
    await page.goto('https://pwa.staging.example.com');
    await page.waitForLoadState('networkidle');

    // Verify Service Worker is active and controlling the page
    const isControlling = await page.evaluate(async () => {
      if (!('serviceWorker' in navigator)) return false;
      const reg = await navigator.serviceWorker.ready;
      return reg.active !== null && navigator.serviceWorker.controller !== null;
    });
    expect(isControlling).toBe(true);

    // Verify CacheStorage contains critical shell assets
    const cachedKeys = await page.evaluate(async () => {
      const cacheNames = await caches.keys();
      if (cacheNames.length === 0) return [];
      const cache = await caches.open(cacheNames[0]);
      const requests = await cache.keys();
      return requests.map((r) => r.url);
    });

    expect(cachedKeys.some((url) => url.includes('main.js'))).toBe(true);
    expect(cachedKeys.some((url) => url.includes('styles.css'))).toBe(true);

    // 2. Simulate complete network disconnection
    await context.setOffline(true);

    // Reload page while offline
    await page.reload();

    // Verify application shell renders successfully from cache
    await expect(page.locator('header.app-header')).toBeVisible();
    await expect(page.locator('.offline-status-banner')).toContainText('You are currently offline');

    // 3. Re-enable network
    await context.setOffline(false);
    await page.reload();
    await expect(page.locator('.offline-status-banner')).not.toBeVisible();
  });
});
```

---

## 4. Testing Service Worker Updates & Cache Invalidation

One of the most frequent defects in PWA releases is the **"Stuck Service Worker"**:
- When a new version of the app is deployed, the old service worker continues to serve legacy cached assets until all existing tabs are closed or `skipWaiting()` is executed.

### QA Verification Steps:
1. Load version `v1.0.0` of the application.
2. Deploy version `v2.0.0` to the test environment.
3. Refresh the page: Ensure an **"Update Available - Click to Refresh"** notification banner appears.
4. Click the update button: Verify that the new service worker activates (`skipWaiting()` / `clients.claim()`) and the page reloads with `v2.0.0` assets without manual cache clearing.

---

## 5. QA Verification Checklist

- [ ] **Offline Resilience**: Complete critical viewing journeys (e.g., view saved orders) with `context.setOffline(true)`.
- [ ] **Offline Mutation Guard**: Ensure destructive actions (submitting orders, making payments) display clear disabled states when offline.
- [ ] **Storage Quota Checks**: Verify that CacheStorage does not exceed storage limits (`navigator.storage.estimate()`).
- [ ] **Manifest & Lighthouse PWA Audit**: Run automated Google Lighthouse audits in CI to assert 100% PWA installability scores.
