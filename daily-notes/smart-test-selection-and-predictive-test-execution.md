# Smart Test Selection (STS) & Test Impact Analysis in CI/CD Pipelines

## 1. The Bottleneck of Monolithic Test Suites in CI/CD

As software repositories grow, automated test suites (unit, integration, component, and end-to-end browser tests) multiply into tens of thousands of tests.
- **The Problem**: Running the full test suite on every single pull request can take 45–90 minutes.
- **The Impact**: Slower developer velocity, delayed PR merges, clogged CI runner queues, and high cloud compute costs.

In modern engineering teams, running *every test on every commit* is an anti-pattern. Instead, QA engineers configure **Smart Test Selection (STS)** and **Test Impact Analysis (TIA)** to identify and run **only the tests affected by the changed code**.

```
┌────────────────────────────────────────────────────────────────────────┐
│                     Git Commit / Pull Request                          │
│                     Changed Files: `src/auth/jwt_validator.ts`         │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   Test Impact Analysis (TIA) Engine                    │
│  - Static AST Dependency Graph Inspection                              │
│  - Code Coverage Mapping (File -> Tests)                               │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │
                     Selective Test Execution
                                   │
               ┌───────────────────┴───────────────────┐
               ▼                                       ▼
    ┌──────────────────────┐               ┌──────────────────────┐
    │     Tests to RUN     │               │    Tests to SKIP     │
    │  - auth_service.test │               │  - checkout.test     │
    │  - jwt_expiry.test   │               │  - payment.test      │
    │  (Total: 42 tests)   │               │  - inventory.test    │
    │  ⏱️ Duration: 45s    │               │  (Total: 3,450 tests)│
    └──────────────────────┘               └──────────────────────┘
```

---

## 2. Core Methodologies for Test Impact Analysis

### A. Static Dependency Graph Mapping (AST Parsing)
Tools like **Jest**, **Vitest**, or **Nx** parse code into an Abstract Syntax Tree (AST) to track `import` and `require` statements.
- If `auth.ts` changes, the test runner queries the graph for any test file that directly or transitively imports `auth.ts`.
- Command in Jest:
  ```bash
  jest --onlyChanged --passWithNoTests
  # Or compare directly against the target branch
  jest --changedSince=origin/main
  ```

### B. Dynamic Code Coverage Mapping
During the nightly full test run, code coverage tools record which lines of source code are exercised by each specific test case.
- If a PR edits lines `45–52` of `discount_calculator.py`, the CI runner queries the coverage database to identify only the 12 tests that touched those exact lines.

---

## 3. Implementing Git-Diff Based Test Selection for Playwright

While unit test runners have native `--onlyChanged` flags, End-to-End (E2E) browser test suites often require custom mapping between changed UI components and relevant spec files.

Below is a Node.js script used in CI pipelines to select relevant Playwright specs based on Git diffs:

```javascript
const { execSync } = require('child_process');
const path = require('path');

// Component-to-Test Mapping Configuration
const COMPONENT_TEST_MAP = {
  'src/components/checkout': 'tests/e2e/checkout.spec.ts',
  'src/components/cart': 'tests/e2e/cart.spec.ts',
  'src/components/auth': 'tests/e2e/auth.spec.ts',
  'src/components/products': 'tests/e2e/product_catalog.spec.ts',
};

function getChangedFiles(baseBranch = 'origin/main') {
  try {
    const diff = execSync(`git diff --name-only ${baseBranch}...HEAD`, { encoding: 'utf-8' });
    return diff.split('\n').filter(Boolean);
  } catch (err) {
    console.error('Failed to calculate git diff, falling back to full suite:', err.message);
    return null;
  }
}

function selectImpactedTests() {
  const changedFiles = getChangedFiles();
  if (!changedFiles) return null; // Run all

  const matchedSpecs = new Set();
  let runFullSuite = false;

  for (const file of changedFiles) {
    // If core configuration or shared dependencies change, run full suite
    if (file.includes('package.json') || file.includes('playwright.config') || file.includes('src/core/')) {
      runFullSuite = true;
      break;
    }

    // Match modified components to specific spec files
    for (const [componentPath, specFile] of Object.entries(COMPONENT_TEST_MAP)) {
      if (file.startsWith(componentPath)) {
        matchedSpecs.add(specFile);
      }
    }
  }

  if (runFullSuite || matchedSpecs.size === 0) {
    console.log('Core changes detected or no direct mapping found. Running full test suite.');
    return 'npx playwright test';
  }

  const specList = Array.from(matchedSpecs).join(' ');
  console.log(`Smart Test Selection active! Executing ${matchedSpecs.size} impacted spec(s): ${specList}`);
  return `npx playwright test ${specList}`;
}

const command = selectImpactedTests();
execSync(command, { stdio: 'inherit' });
```

---

## 4. Safety Nets: The "Two-Tier" CI Strategy

To guarantee that bugs in unmapped code paths never escape to production, QA engineering teams employ a **Two-Tier Quality Gate**:

1. **Tier 1 (Fast PR Gate - < 5 minutes)**:
   - Uses Smart Test Selection (TIA) on every PR commit.
   - Developers receive rapid feedback on their immediate changes.
2. **Tier 2 (Nightly / Post-Merge Full Run - 60 minutes)**:
   - Runs 100% of all unit, integration, and E2E tests against the merged `main` branch.
   - Regenerates the test impact coverage matrix and updates golden visual baselines.

---

## 5. QA Verification Checklist

- [ ] **Dependency Graph Accuracy**: Verify that changes to shared utility modules (`utils/date_formatter.ts`) correctly trigger all downstream consumer tests.
- [ ] **Fail-Safe Fallbacks**: If Git history cannot be computed (e.g., shallow clone), ensure CI automatically defaults to running the full test suite.
- [ ] **Feedback Loop Metrics**: Track CI duration metrics before and after TIA adoption (target: > 70% reduction in PR test execution time).
- [ ] **Nightly Safety Runs**: Never rely on Smart Test Selection exclusively without a scheduled 100% full regression pass.
