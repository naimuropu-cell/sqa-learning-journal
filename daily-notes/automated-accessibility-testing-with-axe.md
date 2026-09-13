# Automated Accessibility Testing with Axe-core and Playwright

## Introduction

Digital accessibility (a11y) ensures that websites and applications can be used by everyone, including people with visual, auditory, motor, or cognitive disabilities. Beyond social responsibility and inclusive design, accessibility is a legal mandate in many jurisdictions (ADA in the United States, European Accessibility Act in the EU), where non-compliant businesses face costly lawsuits.

While manual accessibility testing with screen readers (NVDA, JAWS, VoiceOver) is vital, **automated accessibility testing** catches 30% to 50% of common WCAG (Web Content Accessibility Guidelines) violations instantly on every pull request.

**Axe-core** (developed by Deque Systems) is the industry standard automated accessibility testing engine. Integrating it into **Playwright** creates a continuous accessibility gate in your CI/CD pipeline.

---

## The Four Principles of Accessibility (POUR)

```
┌─────────────────────────────────────────────────────────────┐
│                 WCAG Foundational Principles                │
├─────────────────┬─────────────────┬─────────────────────────┤
│ Principle       │ Meaning         │ Examples Checked by Axe │
├─────────────────┼─────────────────┼─────────────────────────┤
│ 1. Perceivable  │ Information must│ • Missing image alt text│
│                 │ be perceptible  │ • Low color contrast    │
│ 2. Operable     │ Interface must  │ • Missing form labels   │
│                 │ be navigable    │ • Missing focus rings   │
│ 3. Understand-  │ Content must be │ • Missing HTML lang tag │
│    able         │ clear & logical │ • Inconsistent menus    │
│ 4. Robust       │ Must work across│ • Invalid ARIA roles    │
│                 │ screen readers  │ • Duplicate element IDs │
└─────────────────┴─────────────────┴─────────────────────────┘
```

---

## Integrating `@axe-core/playwright` into Automated Tests

### Installation
```bash
npm install -D @axe-core/playwright
```

### Complete Automated Playwright Test Spec (`tests/accessibility.spec.ts`):

```typescript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Automated Accessibility (WCAG 2.1 AA) Audits', () => {
  test('Homepage meets WCAG 2.1 Level AA standards', async ({ page }) => {
    await page.goto('https://example.com');
    await page.waitForLoadState('networkidle');

    // Run Axe accessibility scan with WCAG 2.1 AA rules
    const accessibilityScanResults = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21aa'])
      .exclude('#third-party-chat-widget') // Exclude unowned external widgets
      .analyze();

    // Assert zero violations
    expect(accessibilityScanResults.violations).toEqual([]);
  });

  test('Checkout modal component has valid ARIA labels and color contrast', async ({ page }) => {
    await page.goto('https://example.com/cart');
    await page.click('#open-checkout-modal');

    // Target scan specifically to the active modal dialog locator
    const modalResults = await new AxeBuilder({ page })
      .include('#checkout-dialog')
      .analyze();

    // Filter only Critical and Serious violations
    const criticalViolations = modalResults.violations.filter(
      (v) => v.impact === 'critical' || v.impact === 'serious'
    );

    expect(criticalViolations).toHaveLength(0);
  });
});
```

---

## Formatting and Logging Readable Violation Reports

When an accessibility assertion fails, raw JSON objects can be difficult to read. Here is a helper utility to format violations clearly in test output:

```typescript
function formatAxeViolations(violations: any[]) {
  return violations.map((violation) => ({
    ruleId: violation.id,
    impact: violation.impact,
    description: violation.description,
    helpUrl: violation.helpUrl,
    nodes: violation.nodes.map((node: any) => ({
      html: node.html,
      failureSummary: node.failureSummary,
    })),
  }));
}
```

### Example Terminal Output:
```text
Error: Expected 0 accessibility violations, but found 1:
Rule: "color-contrast" (serious)
Description: Elements must have sufficient color contrast (minimum 4.5:1)
HTML: <button class="btn-subtle" style="color: #999; background: #fff;">Cancel</button>
Fix: Increase contrast ratio from 2.8:1 to at least 4.5:1
Help: https://dequeuniversity.com/rules/axe/4.9/color-contrast
```

---

## What Automation Can and Cannot Catch

| Accessible Property | Can Automated Axe Catch It? | Requires Manual QA Verification |
| :--- | :---: | :---: |
| **Missing image `alt` attributes** | ✅ Yes | Does the alt text accurately describe the image context? |
| **Color contrast below 4.5:1** | ✅ Yes | Does the page remain legible in high contrast OS modes? |
| **Missing `<form>` input labels** | ✅ Yes | Does the label make intuitive sense to a blind user? |
| **Logical Tab order navigation** | ❌ No | Can a keyboard user navigate without getting stuck in a trap? |
| **Screen reader audio clarity** | ❌ No | Does VoiceOver/NVDA pronounce dynamic alerts naturally? |

---

## SQA Interview Questions & Answers

### Q: Why isn't 100% automated accessibility testing sufficient?
**Answer:**
Automated tools like Axe-core analyze the static DOM syntax, checking for structural compliance (e.g., presence of `alt` attributes, valid ARIA tags, color contrast math). However, they cannot assess human experience: an image may have `alt="image123"`, which passes automated checks but is completely useless to a visually impaired user. Full accessibility compliance requires pairing automated CI scans with manual keyboard navigation and screen reader exploratory audits.

### Q: What is the difference between WCAG Level A, AA, and AAA?
**Answer:**
* **Level A**: Minimum accessibility baseline (basic accessibility features; applications that fail Level A are virtually unusable for assistive technology).
* **Level AA**: The global legal and enterprise standard (addresses the most common barriers for disabled users, including 4.5:1 contrast ratios, keyboard accessibility, and clear captions).
* **Level AAA**: The highest level of accessibility (includes strict requirements like 7:1 contrast ratios and sign language interpretation for all prerecorded audio), typically reserved for specialized government portals.

---

## Key Takeaways

* Automated accessibility testing with Axe-core integrates seamlessly into Playwright to catch common WCAG violations in CI/CD.
* Target WCAG 2.1 Level AA as the primary enterprise standard.
* Combine automated scans for syntactic violations with manual keyboard and screen reader testing for genuine usability.

---

## Conclusion

Accessibility is a fundamental pillar of modern software engineering. By embedding Axe-core into automated Playwright regression suites, QA engineers ensure that digital products are inclusive, legally compliant, and usable by everyone from day one.
