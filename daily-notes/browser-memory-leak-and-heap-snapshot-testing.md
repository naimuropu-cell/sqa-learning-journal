# Client-Side Memory Leak Detection & Heap Snapshot Analysis for Web QA

## 1. The QA Impact of Front-End Memory Leaks

In Single Page Applications (SPAs) built with modern frameworks like React, Vue, or Angular, pages do not reload across route transitions. The JavaScript runtime environment persists across user interactions. Consequently, uncollected memory accumulates over time, resulting in:
- **Degraded Frame Rates (Jank)**: Frequent, lengthy Garbage Collection (GC) pauses freeze the browser UI thread (exceeding the 16.6ms per frame budget for 60fps).
- **Tab Crashes (OOM)**: Memory usage climbing steadily into gigabytes causes browser tabs to display `Aw, Snap!` or `STATUS_BREAKPOINT`.
- **Sluggish Input Responsiveness**: Virtual DOM diffing and event delegation slow down due to thousands of unreleased detached DOM nodes.

```
┌────────────────────────────────────────────────────────┐
│               Chrome V8 Heap Memory                   │
├───────────────────────────┬────────────────────────────┤
│        Live Objects       │     Leaked / Zombie Obj    │
│  (Active UI Components,   │  - Detached DOM Elements   │
│   Current Route State,    │  - Uncleaned Event Listeners│
│   Active WebSockets)      │  - Lingering Timer Clbs    │
└───────────────────────────┴─────────────┬──────────────┘
                                          │
                               Retained via Closure
                                          ▼
                             ┌─────────────────────────┐
                             │  GC Root (Window / DOM) │
                             │  Unable to be Collected │
                             └─────────────────────────┘
```

---

## 2. Common Causes of Front-End Memory Leaks

### A. Detached DOM Nodes
A DOM node is removed from the visible document tree (e.g., `parentNode.removeChild(el)`), but a JavaScript variable or event handler retains a reference to it. The garbage collector cannot free the element or any of its child nodes.

### B. Uncleared Intervals and Timers
```javascript
// BUG: Component unmounts, but interval keeps running and references component scope!
useEffect(() => {
  const timer = setInterval(() => {
    fetchLiveMetrics();
  }, 1000);
  // Missing cleanup return: () => clearInterval(timer);
}, []);
```

### C. Unremoved Global Event Listeners
Attaching listeners to `window`, `document`, or global message brokers without removing them when components dismount.

---

## 3. Investigating Leaks with Chrome DevTools Heap Snapshots

QA engineers use Chrome DevTools **Memory Profiler** to identify leak retainers:

### The 3-Snapshot Testing Protocol:
1. **Baseline Snapshot (Snapshot 1)**: Navigate to target application, stabilize, and take initial heap snapshot.
2. **Action Iteration**: Perform the user journey 10 times (e.g., Open Modal $\rightarrow$ Close Modal, or Navigate to Orders $\rightarrow$ Navigate to Dashboard).
3. **Post-Action Snapshot (Snapshot 2)**: Force manual Garbage Collection (trash can icon in DevTools), then take snapshot.
4. **Analysis with "Objects allocated between Snapshot 1 and 2"**:
   - Filter by **Detached**: Look for `Detached HTMLDivElement`, `Detached EventTarget`.
   - Inspect the **Retainer Tree**: Trace back from the leaked element to the GC Root to determine which closure or array holds the reference.

| Metric Term | Definition | QA Interpretation |
| :--- | :--- | :--- |
| **Shallow Size** | Memory held by the object itself (its own properties/primitives) | Usually small for DOM elements (~few bytes) |
| **Retained Size** | Total memory freed if the object and its dependencies are deleted | High retained size indicates a major leak anchor holding massive sub-trees |
| **Distance** | Shortest path of references from the GC Root | Distance to GC root helps pinpoint accidental global scope attachments |

---

## 4. Automated Memory Leak Testing with Playwright & CDP

Using Playwright's Chrome DevTools Protocol (CDP) session, QA engineers can automate memory leak detection in CI pipelines by tracking heap metrics across repeated navigation cycles.

```typescript
import { test, expect } from '@playwright/test';

test.describe('Automated Frontend Memory Leak Verification', () => {
  test('User dashboard navigation does not leak detached DOM elements', async ({ page }) => {
    // Navigate and stabilize
    await page.goto('https://app.staging.example.com/login');
    await page.fill('#username', 'qa_perf_user');
    await page.fill('#password', 'TestingSecurePass123!');
    await page.click('button[type="submit"]');
    await page.waitForURL('**/dashboard');

    // Create CDP Session
    const cdp = await page.context().newCDPSession(page);

    // Force Garbage Collection before baseline
    await cdp.send('HeapProfiler.collectGarbage');
    
    // Get initial heap usage
    const baselineMemory = await page.evaluate(() => {
      return (performance as any).memory ? (performance as any).memory.usedJSHeapSize : null;
    });

    console.log(`Baseline JS Heap: ${(baselineMemory / (1024 * 1024)).toFixed(2)} MB`);

    // Perform action cycle 25 times
    for (let i = 0; i < 25; i++) {
      await page.click('nav >> text=Analytics');
      await page.waitForSelector('.chart-container');
      await page.click('nav >> text=Overview');
      await page.waitForSelector('.metrics-summary');
    }

    // Force Garbage Collection post-iterations
    await cdp.send('HeapProfiler.collectGarbage');

    // Get final heap usage
    const postTestMemory = await page.evaluate(() => {
      return (performance as any).memory ? (performance as any).memory.usedJSHeapSize : null;
    });

    console.log(`Post-Test JS Heap: ${(postTestMemory / (1024 * 1024)).toFixed(2)} MB`);

    const growthMB = (postTestMemory - baselineMemory) / (1024 * 1024);
    console.log(`Heap Growth: ${growthMB.toFixed(2)} MB`);

    // Query detached DOM nodes via CDP
    const nodeCount = await page.evaluate(() => {
      return document.querySelectorAll('*').length;
    });

    // Assert heap growth does not exceed 15MB threshold for 25 cycles
    expect(growthMB).toBeLessThan(15.0);
  });
});
```

---

## 5. QA Defect Reporting Guide for Memory Leaks

When filing a memory leak ticket in Jira, include:
1. **Reproduction Loop**: Exact steps repeated $N$ times (e.g., 20 cycles of switching tabs).
2. **Memory Delta**: Starting Heap size vs. Final Heap size after forced GC.
3. **Retainer Path Screenshot**: DevTools retainer tree showing the exact variable or listener preventing collection.
4. **Chrome/Browser Version**: Exact browser build used during testing.
