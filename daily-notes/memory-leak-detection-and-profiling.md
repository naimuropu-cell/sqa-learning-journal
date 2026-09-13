# Memory Leak Detection & Performance Profiling Guide for QA

## Introduction

In modern Single Page Applications (SPAs) like React, Angular, and Vue, users may keep an application open in their browser tabs for hours or days without refreshing. If an application fails to release allocated memory that is no longer needed, memory consumption steadily climbs.

Over time, this results in a **Memory Leak**, leading to sluggish UI performance, stuttering frame rates, unresponsive buttons, and eventual browser tab crashes ("Out of Memory" or `STATUS_BREAKPOINT`).

For QA engineers, functional testing alone will not reveal memory leaks. Profiling memory allocation, inspecting garbage collection behavior, and writing automated memory threshold tests are essential to maintaining long-session application health.

---

## How Garbage Collection (GC) Works

Modern browser JavaScript engines (V8 in Chrome/Edge/Node.js) utilize a **Mark-and-Sweep** Garbage Collection algorithm:

```
┌─────────────────────────────────────────────────────────────┐
│                 Root Object (Window / Global)               │
└──────────────────────────────┬──────────────────────────────┘
                               │ Reachable references
                               ▼
┌──────────────────────────────┴──────────────────────────────┐
│                  Reachable Objects (Kept)                   │
│   - Active component state, active event handlers           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                Unreachable Objects (Swept)                  │
│   - Orphaned data with no paths from root -> Destroyed ✅   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│            Detached / Leaked Objects (VULNERABILITY ❌)      │
│   - Removed from DOM, but a lingering closure or listener   │
│     retains a reference -> Cannot be garbage collected!     │
└─────────────────────────────────────────────────────────────┘
```

---

## The Four Most Common Frontend Memory Leaks

1. **Detached DOM Nodes**: A DOM node has been removed from the visual document, but a JavaScript variable or array still holds a reference to it in memory.
2. **Uncleared Intervals and Timers**: Using `setInterval()` or `setTimeout()` inside a component without clearing it on unmount (`clearInterval()`).
3. **Forgotten Event Listeners**: Calling `window.addEventListener('resize', handler)` or `document.addEventListener('scroll', handler)` without calling `removeEventListener()`.
4. **Accidental Global Variables**: Declaring variables without `const`, `let`, or `var`, attaching them to the root `window` object indefinitely.

---

## Detecting Leaks using Chrome DevTools

### 1. The Performance Panel "Sawtooth" Pattern
1. Open **Chrome DevTools** ➔ **Performance** tab.
2. Check the **Memory** checkbox.
3. Record 60 seconds of interacting with the page (e.g., repeatedly opening and closing a modal dialog 20 times).
4. Click the garbage can icon to force Garbage Collection.
5. **Healthy Pattern**: Memory rises during interactions and falls back down to baseline after GC (**Sawtooth curve**).
6. **Leaking Pattern**: Memory baseline stair-steps steadily upward without ever returning to the original baseline.

```
Healthy Memory Sawtooth:        Leaking Memory Stair-Step:
      /\    /\    /\                     /|    /|    /|
     /  \  /  \  /  \                   / |   / |   / |
____/    \/    \/    \___          ____/  |__/  |__/  |____
(Returns to Baseline)              (Baseline continuously rises!)
```

### 2. Heap Snapshot Comparison
1. Go to the **Memory** tab ➔ select **Heap snapshot** ➔ take **Snapshot 1** (Baseline).
2. Perform the user action 10 times (e.g., open and close a data table).
3. Force Garbage Collection and take **Snapshot 2**.
4. In the dropdown, switch from **Summary** to **Comparison** against Snapshot 1.
5. Filter by `Detached`: Look for `Detached HTMLDivElement` or `Detached HTMLInputElement`. If hundreds of detached elements persist, a memory leak has been identified.

---

## Automated Memory Profiling with Playwright

QA engineers can assert against memory usage in automated test suites:

```typescript
import { test, expect } from '@playwright/test';

test('Opening and closing modal 20 times does not leak heap memory', async ({ page }) => {
  await page.goto('https://example.com/dashboard');
  
  // 1. Establish initial heap size
  const initialMetrics = await page.evaluate(() => (performance as any).memory?.usedJSHeapSize);

  // 2. Stress action: Open and close modal 20 times
  for (let i = 0; i < 20; i++) {
    await page.click('#open-dialog-btn');
    await page.waitForSelector('#modal-container');
    await page.click('#close-dialog-btn');
    await page.waitForSelector('#modal-container', { state: 'hidden' });
  }

  // 3. Measure final heap size
  const finalMetrics = await page.evaluate(() => (performance as any).memory?.usedJSHeapSize);

  // Heap increase should not exceed 10 MB after garbage collection
  const heapDeltaMB = (finalMetrics - initialMetrics) / (1024 * 1024);
  console.log(`Heap growth after 20 cycles: ${heapDeltaMB.toFixed(2)} MB`);

  expect(heapDeltaMB).toBeLessThan(10);
});
```

---

## SQA Interview Questions & Answers

### Q: What is a "Detached DOM Node" and how does it cause memory leaks?
**Answer:**
A detached DOM node is a DOM element that was removed from the live page tree via JavaScript (e.g., `element.remove()` or React component unmount), but is still retained in memory because a variable, callback function, or event listener continues to hold a reference to it. Because the engine cannot confirm the node is completely unreachable, it cannot garbage-collect the node or any of its child elements.

### Q: How do you verify whether a memory increase is a genuine memory leak or simply normal application caching?
**Answer:**
By forcing Garbage Collection (using the DevTools GC icon or running Node with `--expose-gc`) and taking repeated Heap Snapshots across multiple cycles of the same user flow. If memory increases with each cycle and never resets even after forced GC, it is a genuine memory leak rather than legitimate static cache.

---

## Key Takeaways

* Single Page Applications are highly susceptible to memory leaks during long-running sessions.
* Use Chrome DevTools Performance and Memory tabs to compare heap snapshots and detect detached DOM nodes.
* Automate heap delta thresholds in Playwright to catch memory bloat in CI regression suites.

---

## Conclusion

Memory leak testing protects users from frustrating performance degradations and unexpected application crashes. By integrating memory profiling and heap snapshot comparisons into QA testing, teams ensure their web applications deliver fast, reliable, and smooth experiences even over prolonged usage.
