# Playwright Network Mocking, Interception, and Route Routing Guide

## Introduction

End-to-End (E2E) tests often suffer from high execution latency and flakiness because they rely on live backend servers, external third-party payment gateways, and third-party tracking scripts (Google Analytics, Hotjar, Facebook Pixel).

When a backend staging database experiences a maintenance restart or a third-party analytics script takes 4 seconds to load, your UI tests fail or crawl to a halt.

**Playwright's Network Routing & Mocking API (`page.route`)** allows QA engineers to intercept, inspect, modify, and mock HTTP requests directly at the network transport layer without altering a single line of application source code.

---

## The Network Interception Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                       Browser Engine                        │
│                 (Application makes API call)                │
└──────────────────────────────┬──────────────────────────────┘
                               │ fetch('/api/v1/cart')
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 Playwright page.route() Interceptor         │
├──────────────────────────────┬──────────────────────────────┤
│ Option A: Fulfill Mock       │ Option B: Abort Request      │
│ route.fulfill({ json: ... }) │ route.abort('failed')        │
│ (Instant Mock Response ✅)   │ (Simulate Network Drop ❌)   │
├──────────────────────────────┼──────────────────────────────┤
│ Option C: Modify & Continue  │ Option D: Block Heavy Ads    │
│ route.continue({ headers })  │ route.abort() on tracking    │
└──────────────────────────────┴──────────────────────────────┘
```

---

## 1. Mocking Dynamic API Responses (`route.fulfill`)

Instead of waiting for a real backend database, stub the response instantly:

```typescript
import { test, expect } from '@playwright/test';

test('Render empty state message when cart API returns zero items', async ({ page }) => {
  // Intercept the cart API request and return a mock empty array
  await page.route('**/api/v1/cart', async (route) => {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({ items: [], total: 0.00 }),
    });
  });

  await page.goto('/cart');

  // Verify UI displays empty state illustration
  await expect(page.locator('.empty-cart-message')).toHaveText('Your cart is empty');
  await expect(page.locator('#checkout-btn')).toBeDisabled();
});
```

---

## 2. Simulating Server Crashes & Timeouts (`route.abort`)

Testing how the UI handles backend crashes or network timeouts:

```typescript
test('Display error notification when catalog service crashes with 500 error', async ({ page }) => {
  // 1. Simulate 500 Internal Server Error
  await page.route('**/api/v1/products', async (route) => {
    await route.fulfill({
      status: 500,
      contentType: 'application/json',
      body: JSON.stringify({ error: 'DATABASE_OFFLINE' }),
    });
  });

  await page.goto('/products');
  await expect(page.locator('.toast-error')).toContainText('Unable to load products. Please try again.');
});

test('Simulate complete network disconnection', async ({ page }) => {
  // 2. Abort network request to simulate network connection drop
  await page.route('**/api/v1/user/profile', async (route) => {
    await route.abort('failed'); // Simulates net::ERR_FAILED
  });

  await page.goto('/profile');
  await expect(page.locator('.network-warning')).toBeVisible();
});
```

---

## 3. Accelerating Tests by 300%: Blocking Third-Party Trackers

Marketing scripts (Google Analytics, Sentry, Intercom, Hotjar) introduce significant network latency and memory overhead in CI test runs:

```typescript
test.beforeEach(async ({ context }) => {
  // Block all non-essential third-party analytics across all pages in the context
  await context.route(/google-analytics|hotjar|intercom|segment/, (route) => route.abort());
});
```

Blocking tracking beacons accelerates test suite execution by up to **300%** while saving cloud bandwidth costs.

---

## 4. HAR (HTTP Archive) Recording and Replaying

Playwright can record all live network traffic into a `.har` file during a single execution run and replay it deterministically offline during CI:

```typescript
// Replay network from previously recorded HAR file (Fast & 100% offline!)
test('Checkout flow using recorded HAR fixtures', async ({ page }) => {
  await page.routeFromHAR('fixtures/checkout-network.har', {
    url: '**/api/**',
    update: false, // Replay mode
  });

  await page.goto('/checkout');
  await expect(page.locator('#order-confirmation')).toBeVisible();
});
```

---

## SQA Interview Questions & Answers

### Q: Why is network mocking at the browser transport layer (`page.route`) superior to mocking inside application code?
**Answer:**
Mocking inside application source code (e.g., using conditional environment flags like `if (process.env.TEST) return mockData`) pollutes production code with test logic and creates the risk that mock code accidentally leaks into production. Playwright's `page.route` operates *outside* the application at the browser network transport level, allowing the real, unmodified production bundle to execute while intercepting HTTP traffic transparently.

### Q: What is the risk of over-relying on network mocking in E2E tests?
**Answer:**
Over-mocking creates a false sense of security. If every backend API call is mocked with synthetic JSON, the test suite never verifies that the live frontend and live backend communicate accurately in real life (missing database migration failures, CORS misconfigurations, or API schema drift). Best practice is to use network mocking for negative and edge-case testing, while running a core set of unmocked smoke tests against staging.

---

## Key Takeaways

* Use `page.route()` and `route.fulfill()` to test edge cases (empty states, errors, large datasets) in milliseconds.
* Test error recovery by simulating network drops with `route.abort('failed')`.
* Block third-party tracking scripts to make CI test execution significantly faster and more stable.

---

## Conclusion

Playwright's network routing capabilities transform brittle UI testing into fast, deterministic verification. By intercepting HTTP traffic, simulating adverse server states, and filtering third-party network bloat, QA engineers create test suites that run with unprecedented speed and resilience.
