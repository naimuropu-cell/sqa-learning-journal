# Micro-Frontend Testing Strategies & Module Federation

## Introduction

Just as backend microservices decoupled monolithic backend codebases into independently deployable services, **Micro-Frontends (MFEs)** decompose monolithic frontend single-page applications into smaller, semi-independent web apps.

In a micro-frontend architecture, an enterprise web portal (e.g., an online banking or e-commerce platform) is composed of independent micro-apps—such as a `Navigation Shell`, `Product Catalog MFE`, `Shopping Cart MFE`, and `User Profile MFE`—often managed by separate feature teams and assembled at runtime using **Webpack Module Federation**, **single-spa**, or **Web Components**.

Testing micro-frontends requires a well-architected QA strategy to verify both isolated micro-apps and their collective integration inside the host container.

---

## The Micro-Frontend Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                 Root Shell / Container App                  │
│       (Global Navigation, Auth Session, Event Bus)          │
├──────────────────────────────┬──────────────────────────────┤
│  ┌────────────────────────┐  │  ┌────────────────────────┐  │
│  │   Catalog Micro-App    │  │  │   Checkout Micro-App   │  │
│  │   (React 18 / Team A)  │  │  │   (Vue 3 / Team B)     │  │
│  └────────────────────────┘  │  └────────────────────────┘  │
│  ┌────────────────────────┐  │  ┌────────────────────────┐  │
│  │   Profile Micro-App    │  │  │   Support Micro-App    │  │
│  │   (Svelte / Team C)    │  │  │   (Vanilla JS / Team D)│  │
│  └────────────────────────┘  │  └────────────────────────┘  │
└──────────────────────────────┴──────────────────────────────┘
```

---

## The Micro-Frontend Testing Pyramid

```
                ┌─────────────────────────┐
                │   Full E2E Container    │ ◄── Shell + all MFEs running
                │   Integration Tests     │     (Slow, smoke critical paths)
                └────────────┬────────────┘
                             │
            ┌────────────────┴────────────────┐
            │   Cross-MFE Contract Tests      │ ◄── Verify CustomEvents,
            │   (Event Bus / Shared State)    │     shared props & routing
            └────────────────┬────────────────┘
                             │
       ┌─────────────────────┴─────────────────────┐
       │     Isolated Component & Mocked Shell     │ ◄── Test micro-app in
       │     Integration Testing (Fast & Cheap)    │     standalone harness
       └───────────────────────────────────────────┘
```

1. **Isolated Micro-App Testing**: Test the individual micro-frontend in standalone mode with a mock shell provider (mocking global auth tokens and user profile state).
2. **Contract Testing across MFEs**: Test the communication boundaries between micro-frontends (e.g., verifying that when the Catalog MFE dispatches `CART_ITEM_ADDED`, the Cart MFE correctly processes the payload schema).
3. **Container End-to-End Testing**: Deploy the root container with all federated micro-apps in an integrated staging environment and execute end-to-end user journeys.

---

## Critical Testing Areas & Edge Cases in MFEs

### 1. CSS Bleeding and Style Collisions
Because multiple micro-apps live in the same browser DOM:
* **The Bug**: Global CSS rules in Team A's micro-app (e.g., `button { background: red; }` or un-namespaced utility classes) inadvertently alter the styles of Team B's micro-app.
* **QA Test**: Perform visual regression tests to verify that styles are encapsulated using CSS Modules, Shadow DOM, or styled-components.

### 2. Cross-MFE Communication & Event Bus Drift
* Micro-apps typically communicate via `CustomEvent` dispatched on `window`:
  ```javascript
  window.dispatchEvent(new CustomEvent('ITEM_ADDED', { detail: { productId: 101, qty: 1 } }));
  ```
* **QA Test**: Verify that payload schemas remain strictly backward-compatible. If Team A alters the event structure without coordinating, Team B's dependent component will break silently.

### 3. Route Synchronization & Deep Linking
* **QA Test**: Verify that browser back/forward buttons, URL query parameters, and direct bookmark links navigate accurately across micro-apps without causing page reloads or broken state.

### 4. Shared Dependency Conflicts
* Webpack Module Federation shares dependencies (e.g., single instances of `react` or `lodash`).
* **QA Test**: Verify that if one micro-app upgrades a library, it does not break peer micro-apps expecting an older version.

---

## Practical Test Automation: Testing MFE Communication in Playwright

```typescript
import { test, expect } from '@playwright/test';

test.describe('Micro-Frontend Shell & Cart Integration', () => {
  test('Adding product in Catalog MFE updates item counter in Shell Navigation MFE', async ({ page }) => {
    await page.goto('https://shop.example.com');

    // Wait for Module Federation to mount remote micro-apps
    const catalogFrame = page.locator('#mfe-catalog-container');
    const headerCartBadge = page.locator('#shell-nav-cart-badge');

    await expect(catalogFrame).toBeVisible();
    await expect(headerCartBadge).toHaveText('0');

    // Click 'Add to Cart' inside the Catalog MFE
    const addToCartButton = catalogFrame.locator('button.add-to-cart').first();
    await addToCartButton.click();

    // Verify Shell Header MFE catches the CustomEvent and increments count
    await expect(headerCartBadge).toHaveText('1');
  });
});
```

---

## SQA Interview Questions & Answers

### Q: What is the biggest challenge when testing Micro-Frontends compared to monolithic SPAs?
**Answer:**
Independent deployment risk and integration failure. In an MFE architecture, Team B can deploy their micro-app to production at any time without Team A re-deploying the shell. Testing must ensure that individual micro-apps can run and be deployed independently while strictly adhering to shared contracts (event buses, global states, routing protocols) so that deployments never break peer applications.

### Q: How do you isolate an MFE for automated component testing without running the entire platform?
**Answer:**
By utilizing a **Mock Shell Test Harness**. The test harness wraps the micro-app with mock providers that supply the necessary global context (e.g., authentication tokens, theme providers, navigation hooks) without needing the real shell application or peer micro-apps to be running.

---

## Key Takeaways

* Micro-frontends allow independent team deployments but introduce integration risks like CSS bleeding and event drift.
* Test micro-apps in isolation with mock harnesses, and validate cross-app communication via contract tests.
* Verify CSS encapsulation and seamless browser history navigation across micro-apps.

---

## Conclusion

Micro-frontend architectures offer incredible development autonomy for large organizations. By establishing robust contract verification, isolated testing harnesses, and targeted container integration tests, QA engineers ensure that independent frontend deployments combine seamlessly into a unified user experience.
