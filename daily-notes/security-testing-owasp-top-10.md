# Security Testing Essentials and OWASP Top 10 for QA Engineers

## Introduction

In modern software delivery, security is no longer an afterthought handled exclusively by dedicated penetration testers right before release. With the shift toward **DevSecOps**, Quality Assurance (QA) engineers play a vital role in identifying security flaws, data leaks, and authentication vulnerabilities during everyday functional and API testing.

Security testing verifies that an application protects confidential data, maintains data integrity, and adheres to strict authorization boundaries against malicious attacks.

---

## What is OWASP?

The **Open Web Application Security Project (OWASP)** is a globally recognized non-profit organization dedicated to improving software security. The **OWASP Top 10** represents the most critical web application security risks compiled by security experts worldwide.

---

## The OWASP Top 10 Vulnerabilities & QA Test Scenarios

### 1. Broken Access Control (IDOR / Privilege Escalation)
* **What it is**: Users can act outside of their intended permissions (e.g., a standard customer accessing another customer's profile or an admin dashboard).
* **QA Test Check**:
  * **Vertical Escalation**: Attempt accessing `/admin/users` while logged in as a normal user.
  * **Horizontal Escalation (IDOR)**: Change URL query or API parameters (e.g., change `GET /api/orders/105` to `GET /api/orders/106`) to view another user's invoice.

### 2. Cryptographic Failures (Sensitive Data Exposure)
* **What it is**: Inadequate encryption of sensitive data in transit or at rest.
* **QA Test Check**:
  * Verify HTTPS is enforced and HTTP requests automatically redirect to HTTPS.
  * Ensure passwords, credit cards, or tokens are never displayed in plaintext in network logs, browser storage, or database records.

### 3. Injection Flaws (SQL Injection, Command Injection)
* **What it is**: Untrusted user input is executed as a command or database query.
* **QA Test Check**:
  * Input payloads like `' OR 1=1 --` into login form inputs and search fields to test if authentication is bypassed.
  * Verify that input fields use parameterized queries or strict sanitization.

### 4. Insecure Design
* **What it is**: Missing security design patterns, such as lack of rate-limiting or unprotected password reset flows.
* **QA Test Check**:
  * Attempt submitting a login form 50 times in rapid succession to ensure rate limiting or CAPTCHA triggers.
  * Verify password recovery questions cannot be easily guessed or bypassed.

### 5. Security Misconfiguration
* **What it is**: Unhardened servers, default administrative passwords enabled, or verbose error messages.
* **QA Test Check**:
  * Trigger 500 Internal Server errors by sending malformed payloads and confirm that raw database stack traces are suppressed in user-facing responses.

### 6. Vulnerable and Outdated Components
* **What it is**: Using third-party libraries, NPM packages, or plugins that contain known security CVEs.
* **QA Test Check**:
  * Run automated dependency scans such as `npm audit` or Snyk in the repository.

### 7. Identification and Authentication Failures
* **What it is**: Weak credential handling, missing session timeouts, or credential stuffing susceptibility.
* **QA Test Check**:
  * Verify session tokens are invalidated immediately after clicking **Logout**.
  * Confirm brute-force protection locks or delays accounts after repeated failed attempts.

### 8. Software and Data Integrity Failures
* **What it is**: Code or plugins sourced from untrusted CDNs or repositories without integrity checksum verification.

### 9. Security Logging and Monitoring Failures
* **What it is**: Failure to log critical security events like failed logins, privilege changes, and payment transactions.
* **QA Test Check**:
  * Trigger failed login attempts and inspect audit logs to verify timestamps and IP addresses are recorded.

### 10. Server-Side Request Forgery (SSRF)
* **What it is**: The web server fetches a remote resource without validating the user-supplied URL, allowing attackers to access internal network services.

---

## Daily Security Testing Checklist for QA

- [ ] **Input Sanitization**: Test form fields for HTML, JavaScript tags (`<script>alert(1)</script>`), and special characters.
- [ ] **Session Management**: Ensure session cookies have `Secure`, `HttpOnly`, and `SameSite` flags configured.
- [ ] **Data Masking**: Check that sensitive numbers (SSN, credit cards) are masked (e.g., `**** **** **** 1234`).
- [ ] **File Upload Validation**: Ensure users cannot upload executable scripts (`.php`, `.exe`, `.sh`) disguised as image attachments.
- [ ] **URL Tampering**: Verify that changing query string parameters does not grant unauthorized data access.

---

## Security Testing Tools for QA

| Tool | Category | Primary Use Case |
|---|---|---|
| **OWASP ZAP** | Dynamic Application Security Testing (DAST) | Intercepting HTTP traffic, automated vulnerability spidering |
| **Burp Suite** | Security Proxy / Pentesting | Modifying headers, inspecting raw requests, security fuzzing |
| **Postman** | API Security Verification | Testing authentication endpoints, header validation, rate limiting |
| **Snyk / npm audit** | Software Composition Analysis (SCA) | Scanning open-source libraries for known vulnerabilities |

---

## Interview Questions & Answers

### Q: What is an IDOR vulnerability, and how do you test for it?
**Answer:** 
IDOR (Insecure Direct Object Reference) occurs when an application exposes a reference to an internal object (like an ID in the URL or payload) without validating authorization. A QA engineer tests for IDOR by creating two separate test user accounts (User A and User B). With User A's session, the tester captures an API request referencing User A's resource ID, replaces that ID with User B's resource ID, and sends the request. If User A can read or modify User B's data, an IDOR vulnerability exists.

### Q: What is the difference between Authentication and Authorization?
**Answer:** 
* **Authentication (AuthN)** validates *who* the user is (e.g., verifying username and password).
* **Authorization (AuthZ)** determines *what permissions* the authenticated user has (e.g., verifying if a normal user can access admin functions or delete resources).

---

## Key Takeaways

* Security is part of overall quality; finding vulnerabilities early protects users and businesses.
* The OWASP Top 10 provides a clear blueprint for common web application attack vectors.
* Even manual QA testers can discover critical security bugs like IDOR, missing session invalidation, and input sanitization gaps during regular testing.

---

## Conclusion

Developing a security-conscious testing mindset empowers QA engineers to act as the first line of defense against vulnerabilities, ensuring applications are both functionally robust and secure.
