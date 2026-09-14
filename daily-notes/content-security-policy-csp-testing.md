# Content Security Policy (CSP) Verification & Bypass Testing

## 1. Overview of Content Security Policy (CSP)

**Content Security Policy (CSP)** is an HTTP response header that allows site operators to restrict the resources (such as JavaScript, CSS, Images, Frames, and Fonts) that the browser is allowed to load for a given page. It serves as a critical defense-in-depth layer against:
- **Cross-Site Scripting (XSS)** (Stored, Reflected, and DOM-based)
- **Clickjacking** (`frame-ancestors`)
- **Data Injection & Exfiltration** (`connect-src`)
- **Mixed Content Attacks** (`upgrade-insecure-requests`)

As QA and Security Engineers, testing CSP involves verifying header presence, analyzing directive coverage, evaluating nonces and hashes, testing reporting endpoints, and probing for common configuration bypasses.

```
       [ HTTP Response Header: Content-Security-Policy ]
                           │
       ┌───────────────────┼───────────────────┐
       ▼                   ▼                   ▼
[script-src 'self'] [style-src 'self'] [frame-ancestors 'none']
       │                   │                   │
  Blocks inline       Restricts CSS       Prevents Clickjacking
  scripts & 3rd party  injections         in iframe containers
```

---

## 2. Key Directives & Security Implications

| Directive | Purpose | Insecure Setting | Secure Best Practice |
| :--- | :--- | :--- | :--- |
| `default-src` | Fallback for other fetch directives | `'*' / None` | `'self'` or `'none'` |
| `script-src` | Controls allowed JavaScript execution | `'unsafe-inline' 'unsafe-eval'` | Nonce-based (`'nonce-rAnd0m'`) or strict hash |
| `object-src` | Restricts Flash, Java applets, plugins | `'*'` | `'none'` |
| `base-uri` | Controls the `<base>` tag target URL | Missing / `'*'` | `'self'` or `'none'` (prevents base hijacking) |
| `frame-ancestors` | Restricts who can embed the page in frames | Missing / `'*'` | `'none'` or `'self'` (modern replacement for `X-Frame-Options`) |
| `connect-src` | Restricts endpoints for `fetch`, `XHR`, `WebSocket` | `'*'` | Specific backend API origins |
| `upgrade-insecure-requests` | Automatically upgrades HTTP to HTTPS | Missing on mixed sites | Included in all modern web applications |

---

## 3. Automated Playwright CSP Violation Detection

Modern QA automation frameworks can intercept and validate CSP headers and catch CSP violation console events directly.

```typescript
import { test, expect } from '@playwright/test';

test.describe('Content Security Policy (CSP) Automated Verification', () => {

    test('Verify strong CSP response headers on production endpoints', async ({ page }) => {
        const response = await page.goto('/dashboard');
        expect(response).not.toBeNull();

        const headers = response!.headers();
        const csp = headers['content-security-policy'];

        // Assert CSP is present
        expect(csp, 'CSP header should be present').toBeDefined();

        // Directives assertion
        expect(csp).toContain("object-src 'none'");
        expect(csp).toContain("base-uri 'self'");
        expect(csp).toContain("frame-ancestors 'none'");

        // Disallow dangerous legacy wildcards in production
        expect(csp).not.toContain("'unsafe-eval'");
        expect(csp).not.toContain("'unsafe-inline'");
    });

    test('Detect CSP violation reports via console listeners', async ({ page }) => {
        const cspViolations: string[] = [];

        // Listen to console events indicating CSP blocks
        page.on('console', msg => {
            if (msg.type() === 'error' && msg.text().includes('Content Security Policy')) {
                cspViolations.push(msg.text());
            }
        });

        await page.goto('/dashboard');

        // Deliberately attempt an inline script injection
        await page.evaluate(() => {
            const script = document.createElement('script');
            script.textContent = "window.__malicious_exec = true;";
            document.body.appendChild(script);
        });

        // Verify that inline execution failed
        const executed = await page.evaluate(() => (window as any).__malicious_exec);
        expect(executed).toBeUndefined();

        // Verify CSP violation was raised in the browser engine
        expect(cspViolations.length).toBeGreaterThan(0);
    });
});
```

---

## 4. Common CSP Misconfigurations & QA Bypasses

```
   [ CSP Misconfiguration Patterns ]
              │
   ┌──────────┴──────────┬──────────────────────┐
   ▼                     ▼                      ▼
['unsafe-inline']    [CDN Wildcard]      [Missing base-uri]
Inline script tags   Allowed cdnjs/      Attacker injects
execute freely       Angular gadgets     <base href="..."> to redirect
                                         relative script imports
```

### 1. The CDN Wildcard Flaw
- **Bad Config**: `script-src 'self' https://cdnjs.cloudflare.com;`
- **Vulnerability**: Attackers can load old vulnerable versions of libraries (such as AngularJS 1.5.x) or JSONP endpoints hosted on the CDN to bypass CSP and execute arbitrary JS.
- **QA Verification**: Check if third-party domains in `script-src` provide unauthenticated script upload, JSONP APIs, or client-side template engines.

### 2. Missing `base-uri`
- If `base-uri` is missing, an attacker who can inject HTML can inject `<base href="https://attacker.com/">`.
- Any relative script import `<script src="/app.js"></script>` will resolve to `https://attacker.com/app.js` and execute.

---

## 5. CSP Evaluation with Google CSP Evaluator

QA teams should integrate automated audits against CSP policy strings using the Google CSP Evaluator CLI or API:

```bash
# Analyze deployed CSP policy
curl -sI https://example.com | grep -i "content-security-policy"
```

| Check Item | Pass Criteria | Severity if Failed |
| :--- | :--- | :--- |
| **Strict Dynamic** | Nonce + `'strict-dynamic'` present | High |
| **Object-src** | Explicitly `'none'` | High |
| **Base-uri** | Explicitly `'none'` or `'self'` | High |
| **Report-Only Mode** | Disabled in production (`Content-Security-Policy-Report-Only` not sole defense) | Medium |

---

## 6. SQA Interview Questions & Answers

### Q1: What is the difference between `Content-Security-Policy` and `Content-Security-Policy-Report-Only`?
> **Answer**:
> `Content-Security-Policy` actively enforces policy rules: violating scripts and assets are blocked immediately in the browser, and violation logs are sent to the report URI if specified.
> `Content-Security-Policy-Report-Only` monitors and logs violations without blocking any resources or breaking user functionality. It is used during staging or rollout to discover breaking scripts prior to strict enforcement.

### Q2: Why is `'unsafe-inline'` dangerous, and how does a cryptographic nonce solve it?
> **Answer**:
> `'unsafe-inline'` allows any inline `<script>` tags or inline event handlers (`onload`, `onclick`) to execute, rendering CSP ineffective against Cross-Site Scripting (XSS).
> A **cryptographic nonce** (`nonce-randomToken123`) requires the server to generate a cryptographically strong, unique token per HTTP request and include it both in the header (`script-src 'nonce-randomToken123'`) and on valid inline scripts (`<script nonce="randomToken123">`). Attackers cannot predict the nonce, preventing injected inline scripts from running.

---

## 7. Key Takeaways & Best Practices

- Always specify `base-uri 'none'` or `base-uri 'self'` to prevent base hijacking attacks.
- Avoid broad domain wildcards (e.g., `https://*` or `*.googleapis.com`); use strict nonces with `'strict-dynamic'` for modern single-page applications.
- Run automated Playwright tests in CI to verify that no CSP violations are triggered during normal user workflows.
