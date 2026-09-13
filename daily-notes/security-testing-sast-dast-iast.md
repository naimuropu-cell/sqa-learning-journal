# DevSecOps for QA: SAST vs. DAST vs. IAST Security Testing

## Introduction

Traditionally, security testing was treated as an isolated audit performed by external penetration testers right before production release. This outdated model led to severe deployment bottlenecks and expensive late-stage architectural fixes.

In modern **DevSecOps**, security is shifted left and integrated directly into continuous QA pipelines. QA engineers collaborate with security teams to automate security gates using **SAST**, **DAST**, **IAST**, and **SCA**.

---

## The Application Security Testing Landscape

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DevSecOps Testing Pipeline                         │
├───────────────────┬─────────────────────────┬───────────────────────────┤
│    Code Commit    │      Build & Stage      │    Test & Runtime (QA)    │
├───────────────────┼─────────────────────────┼───────────────────────────┤
│  • SAST           │  • SCA                  │  • DAST                   │
│    (SonarQube,    │    (Dependabot, Snyk)   │    (OWASP ZAP, Burp)      │
│     Semgrep)      │    Checks 3rd party     │    Black-box live scan    │
│    White-box code │    libraries/CVEs       │  • IAST (Contrast Sec)    │
│    analysis       │                         │    Hybrid agent sensor    │
└───────────────────┴─────────────────────────┴───────────────────────────┘
```

---

## Detailed Comparison: SAST vs. DAST vs. IAST vs. SCA

| Feature | SAST (Static) | DAST (Dynamic) | IAST (Interactive) | SCA (Composition) |
| :--- | :--- | :--- | :--- | :--- |
| **Testing Type** | White-box (Source code) | Black-box (Running app) | Hybrid (Agent inside app) | Dependency scanner |
| **Stage in SDLC** | Early coding / PR | Staging / Test execution | Automated QA test runs | Build / Package |
| **Application State**| Code at rest (No execution)| Running live application | Running live application | Manifests (`package.json`)|
| **What It Finds** | Hardcoded secrets, SQL injection patterns, buffer overflows | Cross-Site Scripting (XSS), missing security headers, auth bypass | Real-time code execution vulnerabilities, data flow leaks | Known CVEs in open-source dependencies |
| **False Positives**| High (flags theoretical issues) | Low (validates actual exploits)| Very low (monitors execution) | Very low (matches CVE DB) |
| **Tools** | SonarQube, Semgrep, Checkmarx | OWASP ZAP, Burp Suite, Nuclei | Contrast Security, Invicti | Snyk, Dependabot, npm audit |

---

## Automating DAST with OWASP ZAP in CI/CD

**OWASP ZAP (Zed Attack Proxy)** is an open-source dynamic web scanner. QA engineers can run containerized baseline scans against staging environments inside GitHub Actions:

```yaml
name: Automated DAST Security Scan

on:
  schedule:
    - cron: '0 3 * * 1' # Every Monday at 3:00 AM UTC
  workflow_dispatch:

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Source Code
        uses: actions/checkout@v4

      - name: Run OWASP ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.12.0
        with:
          target: 'https://staging.example.com'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a' # Include alpha passive scan rules
          fail_action: true # Fail pipeline if High/Medium vulnerabilities are detected

      - name: Upload Security Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: zap-security-report
          path: report_html.html
```

---

## Common Security Vulnerabilities Checked by QA

1. **Missing HTTP Security Headers**:
   * `Strict-Transport-Security (HSTS)`
   * `Content-Security-Policy (CSP)`
   * `X-Frame-Options: DENY` (prevents Clickjacking)
   * `X-Content-Type-Options: nosniff`
2. **Insecure Cookie Flags**:
   * Ensuring all session cookies include `Secure`, `HttpOnly`, and `SameSite=Strict`.
3. **Cross-Site Scripting (XSS)**:
   * Injecting payload scripts (`<script>alert(1)</script>`) into input fields to verify sanitization.
4. **Broken Access Control & IDOR**:
   * Verifying that changing a resource ID in the URL (`/api/invoices/1001` to `/api/invoices/1002`) blocks unauthorized users.

---

## SQA Interview Questions & Answers

### Q: Why can't SAST replace DAST (or vice versa)?
**Answer:**
SAST inspects static source code and finds syntax-level vulnerabilities (e.g., hardcoded API keys, unparameterized SQL queries) before deployment, but cannot detect server runtime issues like weak TLS configurations, authentication cookie flaws, or microservice routing vulnerabilities. DAST tests the compiled, live running application over HTTP, detecting real exploitability. Both are complementary and required for defense-in-depth.

### Q: What is Software Composition Analysis (SCA) and why is it critical?
**Answer:**
Modern software applications are built with 70–90% open-source libraries (npm packages, Maven JARs, Python pip modules). SCA tools scan project dependency trees against known Common Vulnerabilities and Exposures (CVE) databases (e.g., the National Vulnerability Database). They alert QA and developers when an imported library contains known exploits (e.g., the Log4j vulnerability).

---

## Key Takeaways

* Modern QA teams actively participate in DevSecOps by automating security guardrails in CI/CD.
* SAST checks static source code, SCA audits open-source dependencies, and DAST scans live running web endpoints.
* OWASP ZAP integrates seamlessly into CI pipelines to catch common vulnerabilities before production release.

---

## Conclusion

Security is a primary attribute of software quality. By incorporating SAST, DAST, and dependency analysis into the testing pipeline, QA engineers ensure that systems are not only functionally correct and fast, but robustly fortified against malicious attacks.
