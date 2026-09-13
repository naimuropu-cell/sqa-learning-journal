# CSRF Prevention and Secure Session Management Testing Guide

## Introduction

Web applications authenticate users and preserve state across multiple HTTP requests through **Session Management** (session cookies or tokens).

However, because web browsers automatically attach stored cookies to every outgoing request destined for a domain—even when that request is triggered from a malicious external website—applications are vulnerable to **Cross-Site Request Forgery (CSRF)**.

QA engineers must systematically verify that applications enforce robust session lifecycles, validate CSRF tokens, configure strict cookie attributes, and properly invalidate sessions upon logout.

---

## How Cross-Site Request Forgery (CSRF) Operates

```
1. Victim logs in to online banking (Session cookie stored in browser)
2. Victim visits malicious attacker site in another tab
3. Attacker page contains hidden auto-submitting form:
   <form action="https://bank.com/transfer" method="POST">
     <input type="hidden" name="toAccount" value="attacker_id" />
     <input type="hidden" name="amount" value="5000" />
   </form>
   <script>document.forms[0].submit();</script>
4. Browser sends request to bank.com AND AUTOMATICALLY ATTACHES THE SESSION COOKIE!
5. Bank server processes the transfer because the session cookie is valid!
```

---

## Defense Mechanisms Tested by QA

```
┌─────────────────────────────────────────────────────────────┐
│                    CSRF Defense Verification                │
├─────────────────────┬───────────────────────────────────────┤
│ Defense Technique   │ What QA Tests                         │
├─────────────────────┼───────────────────────────────────────┤
│ 1. Anti-CSRF Token  │ Request fails with HTTP 403 when      │
│                     │ token is missing, expired, or altered │
│ 2. SameSite Cookies │ Cookie header contains `SameSite=Lax` │
│                     │ or `SameSite=Strict`                  │
│ 3. Custom Headers   │ Request must contain custom header    │
│                     │ (e.g., `X-Requested-With`)            │
└─────────────────────┴───────────────────────────────────────┘
```

### The `SameSite` Cookie Attribute
* `SameSite=Strict`: The cookie is NEVER sent on cross-site requests (highest security, best for banking).
* `SameSite=Lax`: Cookie is sent only when following top-level GET navigation links, not on cross-origin POST/PUT requests.
* `SameSite=None`: Cookie is sent on all cross-site requests (requires the `Secure` flag).

---

## Secure Session Lifecycle Testing Checklist

### 1. Cookie Security Flags
Inspect the `Set-Cookie` response header in browser DevTools:
* `HttpOnly`: Prevents client-side JavaScript (`document.cookie`) from reading the cookie, mitigating XSS token theft.
* `Secure`: Ensures the cookie is strictly transmitted over encrypted HTTPS connections.
* `SameSite=Strict` or `Lax`: Blocks cross-origin CSRF attacks.

### 2. Session Invalidation on Logout
* **The Vulnerability**: Clicking "Logout" clears the cookie in the browser, but the session identifier remains active on the server.
* **QA Test**:
  1. Log in and copy the session cookie value from DevTools.
  2. Click "Logout" in the UI.
  3. Using Postman or curl, replay an authenticated API request manually passing the copied cookie.
  4. **Expected Result**: Server returns **HTTP 401 Unauthorized** (confirms server-side session revocation).

### 3. Session Fixation Testing
* **The Vulnerability**: The server assigns a session identifier to an anonymous visitor and fails to regenerate a brand-new ID when that user logs in.
* **QA Test**: Check session ID before and after logging in. The session ID **must change** immediately upon successful authentication.

### 4. Idle Session Timeout
* **QA Test**: Log in and leave the tab idle without activity for the configured timeout duration (e.g., 15 minutes). Verify that subsequent actions redirect to the login screen and invalidate the session.

---

## Practical Security Test Cases

| Test Case ID | Test Description | Action / Payload | Expected Result |
| :--- | :--- | :--- | :--- |
| **SEC-CSRF-01** | Verify form submission without CSRF token | Strip `_csrf` token parameter from POST request | Server responds with **HTTP 403 Forbidden** |
| **SEC-CSRF-02** | Verify tampered CSRF token | Alter last 4 characters of `_csrf` token | Server responds with **HTTP 403 Forbidden** |
| **SEC-SESS-01** | Verify cookie `HttpOnly` flag | Execute `console.log(document.cookie)` in DevTools | Auth session cookie is invisible to JavaScript |
| **SEC-SESS-02** | Replay session token post-logout | Send authenticated GET request with old session ID | Server returns **HTTP 401 Unauthorized** |

---

## SQA Interview Questions & Answers

### Q: Why does CSRF affect cookie-based authentication but not Authorization Header (Bearer JWT) authentication?
**Answer:**
Browsers automatically attach cookies to cross-origin requests directed to the target domain without client script intervention. In contrast, `Authorization: Bearer <token>` headers must be manually attached by JavaScript code. Because malicious external websites cannot read local tokens stored in another origin's memory or storage (due to the Same-Origin Policy), they cannot attach the Bearer token to forge requests.

### Q: What is Session Fixation and how do you test for it?
**Answer:**
Session fixation is an attack where a malicious actor induces a victim to authenticate using a pre-known session ID. QA tests for this by recording the session cookie value while browsing anonymously, logging in with valid credentials, and comparing the post-login session cookie. If the session cookie remains identical, the application is vulnerable to session fixation.

---

## Key Takeaways

* CSRF exploits the browser's automatic inclusion of cookies on cross-origin requests.
* Defend against CSRF using Anti-CSRF synchronizer tokens and `SameSite=Strict/Lax` cookie flags.
* Always verify server-side session invalidation upon logout and test idle timeout boundaries.

---

## Conclusion

Session management and CSRF prevention are fundamental to web application security. By rigorously testing cookie flags, session lifecycles, and anti-CSRF token enforcement, QA engineers ensure that user identities and account privileges remain strictly guarded against malicious exploitation.
