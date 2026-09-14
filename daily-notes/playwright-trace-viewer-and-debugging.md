# Playwright Trace Viewer & Time-Travel Debugging

## 1. Introduction to Playwright Trace Viewer

Debugging end-to-end (E2E) UI test failures in headless continuous integration (CI) environments has historically been painful. When a test fails in GitHub Actions or Jenkins, developers and QA engineers often only have access to a static screenshot or a short stack trace stating `TimeoutError: element not found`.

**Playwright Trace Viewer** is a GUI tool that captures comprehensive runtime execution metadata during test runs. It provides full **time-travel debugging**, capturing:
- Step-by-step action execution and timings.
- Full DOM snapshots before, during, and after each browser interaction (click, type, navigate).
- Complete network waterfall (HTTP/WS requests, request/response headers, status codes, payload bodies).
- Browser console logs and telemetry errors.
- Visual filmstrip with timeline scrubbing.
- Source code location mapped directly to the executing test line.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Playwright Trace Viewer                         │
│  [ Timeline Filmstrip: 0.0s ─── 1.2s ─── 2.4s ─── 3.8s (Failed) ]      │
├───────────────────┬─────────────────────────────────┬──────────────────┤
│ Actions List      │ Interactive DOM Snapshot        │ Metadata Panel   │
│ 1. page.goto      │ (Live inspectable DOM tree)     │ ─ Console Logs   │
│ 2. locator.fill   │                                 │ ─ Network Trace  │
│ 3. locator.click  │ [ Username ] [ Password ]       │ ─ Source Code    │
│ 4. expect(error)  │ [ Submit (Red Click Point) ]    │ ─ Call Stacks    │
└───────────────────┴─────────────────────────────────┴──────────────────┘
```

---

## 2. Configuring Automated Traces in CI/CD

To balance storage overhead with forensic utility, the recommended industry practice is to capture traces **only on test failure** or on retry.

### `playwright.config.ts` Configuration

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0, // Auto-retry in CI
  workers: process.env.CI ? 2 : undefined,
  reporter: [
    ['html', { open: 'never' }],
    ['list']
  ],
  use: {
    baseURL: 'https://staging.example.com',
    headless: true,
    
    // Trace options: 'off' | 'on' | 'retain-on-failure' | 'on-first-retry'
    trace: 'retain-on-failure',
    
    // Screenshots & Videos
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
  ],
});
```

---

## 3. GitHub Actions Workflow for Trace Artifact Publishing

```yaml
name: Playwright E2E Regression Suite

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: 20
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Install Playwright Browsers
      run: npx playwright install --with-deps

    - name: Run Playwright tests
      run: npx playwright test

    # Publish HTML report and traces on failure
    - name: Upload Test Results & Traces
      uses: actions/upload-artifact@v4
      if: failure()
      with:
        name: playwright-failure-artifacts
        path: |
          playwright-report/
          test-results/
        retention-days: 14
```

---

## 4. Analyzing and Debugging Traces

Once the `trace.zip` is downloaded from CI artifacts, QA engineers can inspect it locally or directly in the browser:

```bash
# View trace file via CLI
npx playwright show-trace ./test-results/login-test-chromium/trace.zip

# Or inspect online without installing Node:
# Navigate to https://trace.playwright.dev and drag-and-drop the trace.zip!
```

### Key Diagnostic Panels:
1. **Action Timeline**: Hover over each action (`click`, `fill`, `hover`) to observe the red circular dot indicating the exact $(x, y)$ coordinate clicked.
2. **Before / After DOM**: Compare the DOM before the click vs. after to verify state updates without relying on guesswork.
3. **Network Tab**: Filter by `XHR / Fetch` to inspect backend responses; verify whether test timeouts resulted from slow or failing 500 API responses.
4. **Console Panel**: Check for unhandled JavaScript exceptions, hydration errors, or CSP violations occurring inside the target page.

---

## 5. Trace Viewer vs. Traditional Video/Screenshot

| Feature | Screenshots | Video Recording | Playwright Trace Viewer |
| :--- | :--- | :--- | :--- |
| **Inspectable DOM** | No (flat image) | No (pixel video) | **Yes (live interactive DOM tree)** |
| **Network Requests & Bodies** | No | No | **Yes (complete headers, payload, status)** |
| **Console Errors & Warnings**| No | Only if rendered in UI | **Yes (direct browser engine log stream)** |
| **Time Travel Scrubbing** | No | Yes (video scrub) | **Yes (frame-by-frame action scrub)** |
| **Storage Overhead** | Minimal (< 100 KB) | High (2–10 MB per test) | Moderate (compressed zip, 500 KB–2 MB) |
| **CI Failure Triage Speed** | Slow | Moderate | **Instant** |

---

## 6. SQA Interview Questions & Answers

### Q1: What makes Playwright Trace Viewer superior to recording video for CI test failure investigation?
> **Answer**:
> While video only captures visual pixels, Trace Viewer captures a full programmatic record of the browser session. QA engineers can interact with the live DOM tree at any millisecond of the test execution, inspect CSS selectors and computed styles, review full network request/response headers and JSON payloads, and see exact mouse coordinates and console errors.

### Q2: Why is the `'retain-on-failure'` configuration recommended over `'on'`?
> **Answer**:
> Setting `trace: 'on'` records full DOM trees, screenshots, and network logs for every single test case, which consumes significant disk space and memory and slows down CI execution times. 
> Setting `trace: 'retain-on-failure'` records trace data in memory and only persists it to disk when a test assertion fails, maximizing CI performance and minimizing storage while providing complete diagnostics when failures occur.

---

## 7. Key Takeaways & Best Practices

- Always configure `trace: 'retain-on-failure'` in CI to capture rich debugging traces with zero overhead on passing suites.
- Use `https://trace.playwright.dev` to inspect traces instantly in any modern browser without local dependencies.
- Use the network waterfall panel within the trace to differentiate between genuine frontend UI defects and backend API timeouts.
