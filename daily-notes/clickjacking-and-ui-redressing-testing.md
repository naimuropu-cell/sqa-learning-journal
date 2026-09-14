# Clickjacking & UI Redressing Security Testing Guide for QA

## Introduction

Web security testing often focuses on data manipulation (SQL injection) or script execution (XSS). However, an attacker does not always need to steal data directly if they can trick a legitimate, authenticated user into performing unintended actions on their behalf.

**Clickjacking** (also known as a **UI Redressing Attack**) is an interface-based attack where a malicious website loads a vulnerable web application inside a transparent, invisible `<iframe>` positioned precisely over a deceptive button (such as "Play Video" or "Click to Claim Prize"). When the victim clicks the visible decoy button, they unwittingly click an invisible button inside the target application—authorizing a bank transfer, changing account settings, or deleting their account.

QA engineers must systematically verify that applications enforce strict framing protections across all pages.

---

## The Mechanics of a Clickjacking Attack

```
┌─────────────────────────────────────────────────────────────┐
│                 Attacker's Malicious Website                │
│                                                             │
│   [ Decoy Button: "Click Here to Claim $1,000 Prize!" ]    │
│                                                             │
│   ┌ - - - - - - - - - - - - - - - - - - - - - - - - - - ┐   │
│   │ INVISIBLE IFRAME (opacity: 0.001, z-index: 10)      │   │
│   │ https://mybank.com/transfer                         │   │
│   │                                                     │   │
│   │   [ Invisible Button: "Confirm Wire Transfer" ]     │   │
│   │   (Positioned directly over the decoy button!)      │   │
│   └ - - - - - - - - - - - - - - - - - - - - - - - - - - ┘   │
└─────────────────────────────────────────────────────────────┘
  When the victim clicks the decoy, the click registers on the
  hidden bank button with the victim's active session cookie! 🚨
```

---

## Defense Mechanisms Tested by QA

Modern web browsers enforce two primary HTTP response headers that control whether a web page is permitted to be embedded inside an `<iframe>`:

```
┌─────────────────────────────────────────────────────────────┐
│                 Clickjacking Defenses Tested                │
├─────────────────────┬───────────────────────────────────────┤
│ Header              │ Values & Rules                        │
├─────────────────────┼───────────────────────────────────────┤
│ 1. Content-Security-│ • frame-ancestors 'none'; (Disallows  │
│    Policy (CSP)     │   all framing entirely ⭐)            │
│    (Modern Standard)│ • frame-ancestors 'self'; (Only allows│
│                     │   framing by the same domain)         │
├─────────────────────┼───────────────────────────────────────┤
│ 2. X-Frame-Options  │ • DENY (Disallows all framing)        │
│    (Legacy Standard)│ • SAMEORIGIN (Only same origin frames)│
└─────────────────────┴───────────────────────────────────────┘
```

> [!IMPORTANT]
> **CSP Takes Precedence**: If both `Content-Security-Policy: frame-ancestors` and `X-Frame-Options` are present, modern browsers prioritize the CSP rule. QA must verify that both headers are aligned to protect both modern and legacy browsers.

---

## Step-by-Step QA Testing: The PoC Iframe Test

The most definitive way to verify Clickjacking defense is to create a lightweight **Proof of Concept (PoC) HTML file**:

### 1. Create `test-clickjacking.html`:
```html
<!DOCTYPE html>
<html>
<head>
  <title>Clickjacking PoC Verification Test</title>
</head>
<body>
  <h1>Testing Framing Protection on Target Application</h1>
  <!-- Attempt to frame the target application -->
  <iframe src="https://staging.example.com/account/settings" width="800" height="600"></iframe>
</body>
</html>
```

### 2. Execution & Expected Assertions:
1. Open `test-clickjacking.html` in a web browser.
2. **Vulnerable Behavior (Defect ❌)**: The target application successfully renders inside the iframe. (File high-severity security bug!).
3. **Secure Behavior (Pass ✅)**:
   * The iframe remains completely blank.
   * The browser console displays the security error:
     ```text
     Refused to display 'https://staging.example.com' in a frame because it set
     'X-Frame-Options' to 'SAMEORIGIN' (or 'frame-ancestors' to 'none').
     ```

---

## Automated Clickjacking Header Assertion in Playwright

```typescript
import { test, expect } from '@playwright/test';

test.describe('Clickjacking Header Protection Audit', () => {
  const sensitiveEndpoints = [
    '/login',
    '/account/settings',
    '/checkout',
    '/admin/dashboard',
  ];

  for (const endpoint of sensitiveEndpoints) {
    test(`Verify framing protection headers on ${endpoint}`, async ({ request }) => {
      const response = await request.get(endpoint);
      const headers = response.headers();

      // Verify either CSP frame-ancestors or X-Frame-Options is strictly enforced
      const cspHeader = headers['content-security-policy'] || '';
      const xFrameOptions = headers['x-frame-options'] || '';

      const hasCspProtection = cspHeader.includes("frame-ancestors 'none'") || cspHeader.includes("frame-ancestors 'self'");
      const hasXFrameProtection = xFrameOptions.toUpperCase() === 'DENY' || xFrameOptions.toUpperCase() === 'SAMEORIGIN';

      expect(
        hasCspProtection || hasXFrameProtection,
        `Endpoint ${endpoint} lacks Clickjacking protection headers!`
      ).toBe(true);
    });
  }
});
```

---

## SQA Interview Questions & Answers

### Q: Why are legacy JavaScript "Framebusting" scripts considered obsolete and insecure?
**Answer:**
Early developers attempted to prevent clickjacking using client-side JavaScript (e.g., `if (top !== self) top.location = self.location;`). These framebusting scripts are notoriously easy to neutralize: an attacker can neutralize the script using HTML5 iframe sandboxing (`<iframe sandbox="allow-forms allow-scripts">`), which disables the framed page from altering the parent window's location. True framing protection must be enforced by the browser via HTTP response headers (`X-Frame-Options` and CSP `frame-ancestors`).

### Q: When is it permissible to allow framing (`SAMEORIGIN` or specific origins)?
**Answer:**
Framing is permissible when an application legitimately embeds its own components across subdomains (e.g., embedding a dashboard widget inside a parent portal). In such cases, `Content-Security-Policy: frame-ancestors 'self' https://trusted-partner.com` allows only authenticated, whitelisted domains to frame the application, preventing arbitrary malicious websites from executing overlay attacks.

---

## Key Takeaways

* Clickjacking tricks authenticated users into clicking invisible buttons inside an embedded iframe.
* Enforce `Content-Security-Policy: frame-ancestors 'none'` and `X-Frame-Options: DENY` on all sensitive views.
* Validate protection using both automated header assertions and live PoC iframe rendering tests.

---

## Conclusion

Clickjacking security testing guarantees that user interface integrity cannot be compromised by third-party adversaries. By strictly asserting framing policies and verifying that applications reject untrusted embedding, QA engineers protect user accounts from deceptive visual exploitation.
