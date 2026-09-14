# CORS Misconfiguration Security Testing Guide for QA

## Introduction

The **Same-Origin Policy (SOP)** is the foundational security mechanism of the World Wide Web. It prevents scripts executing on one origin (e.g., `https://attacker.com`) from reading sensitive data or accessing DOM elements on another origin (e.g., `https://mybank.com`).

However, modern distributed web applications frequently need to share resources across different domains or subdomains (e.g., frontend on `https://app.example.com` calling an API on `https://api.example.com`).

**Cross-Origin Resource Sharing (CORS)** is an HTTP-header-based mechanism that allows a server to explicitly declare which foreign origins are permitted to access its data.

When developers misconfigure CORS headers, they inadvertently punch a massive hole in the browser's Same-Origin Policy, enabling malicious websites to steal customer profile data, private banking records, and authentication tokens with a single victim click.

---

## How CORS Works: Preflight and Headers

```
┌──────────────┐                                    ┌──────────────┐
│   Browser    │ ── 1. OPTIONS /api/v1/user ──────► │  API Server  │
│              │    Origin: https://attacker.com    │              │
│              │                                    │              │
│              │ ◄─ 2. CORS Response Headers ──────┤              │
│              │    Access-Control-Allow-Origin     │              │
│              │    Access-Control-Allow-Credentials│              │
└──────────────┘                                    └──────────────┘
```

* `Origin`: Sent by browser indicating the domain executing the JavaScript.
* `Access-Control-Allow-Origin (ACAO)`: Declares which origin is allowed access.
* `Access-Control-Allow-Credentials (ACAC)`: If `true`, permits the browser to expose responses when authentication cookies or authorization headers are included.

---

## The Top 4 High-Risk CORS Misconfigurations

```
┌─────────────────────────────────────────────────────────────┐
│                 Dangerous CORS Misconfigurations            │
├─────────────────────┬───────────────────────────────────────┤
│ Misconfiguration    │ The Vulnerability & Exploit Mechanism │
├─────────────────────┼───────────────────────────────────────┤
│ 1. Origin Reflection│ Server blindly echoes back whatever   │
│    + Credentials    │ Origin header is sent, coupled with   │
│    (Critical 🚨)    │ Access-Control-Allow-Credentials: true│
├─────────────────────┼───────────────────────────────────────┤
│ 2. Trusting 'null'  │ Origin: null is trusted (exploitable  │
│    Origin           │ via local HTML files or sandboxed     │
│                     │ <iframe sandbox> tags)                │
├─────────────────────┼───────────────────────────────────────┤
│ 3. Weak Regex       │ Matches "example.com" naively:        │
│    Subdomain Bypass │ Allows "attacker-example.com" or      │
│                     │ "example.com.attacker.com"            │
├─────────────────────┼───────────────────────────────────────┤
│ 4. Intranet Exposure│ Public web API allows requests from   │
│                     │ localhost or internal corporate IPs   │
└─────────────────────┴───────────────────────────────────────┘
```

---

## Testing for Dangerous Origin Reflection (The Exploit)

### The Vulnerability:
A developer writes a quick backend fix for cross-origin issues:
```javascript
// INSECURE BACKEND CODE:
app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", req.headers.origin); // Blind reflection!
  res.header("Access-Control-Allow-Credentials", "true");        // Critical flaw!
  next();
});
```

### The QA Test:
Send a request supplying an arbitrary attacker origin using `curl`:

```bash
curl -i -H "Origin: https://malicious-attacker.com" \
  -H "Cookie: session_token=secret_123" \
  https://api.example.com/v1/user/private-profile
```

### Dangerous Vulnerable Response:
```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://malicious-attacker.com
Access-Control-Allow-Credentials: true
Content-Type: application/json

{"email": "victim@bank.com", "balance": 95000, "ssn": "XXX-XX-1234"}
```
> [!CRITICAL]
> **Vulnerability Confirmed**: Because the server reflected `https://malicious-attacker.com` alongside `Allow-Credentials: true`, an attacker hosting a website can use `fetch()` to steal the victim's complete profile and balance silently in the background!

---

## Testing for `null` Origin Exploitation

```bash
curl -i -H "Origin: null" https://api.example.com/v1/user/data
```
If the server responds with:
`Access-Control-Allow-Origin: null` and `Access-Control-Allow-Credentials: true`
* **Vulnerability Confirmed**: Attackers can execute cross-origin requests using sandboxed iframes (`<iframe sandbox="allow-scripts allow-top-navigation">`), which generate an `Origin: null` header in all browsers.

---

## SQA Interview Questions & Answers

### Q: Can you set `Access-Control-Allow-Origin: *` together with `Access-Control-Allow-Credentials: true`?
**Answer:**
No. Modern web browsers strictly reject this combination by default according to W3C CORS specifications. If a server attempts to respond with wildcard origin `*` and credentials `true`, the browser will throw a CORS error in the console and refuse to expose the response data to JavaScript. Attackers bypass this restriction when lazy developers implement origin reflection instead of a strict allowlist.

### Q: How do developers properly secure CORS configurations?
**Answer:**
1. **Explicit Whitelists**: Maintain a strict allowlist of approved production domains (e.g., `['https://app.example.com', 'https://admin.example.com']`).
2. **Avoid Wildcard Origins on Authenticated Endpoints**: Only use `*` on completely public, unauthenticated APIs (like public weather feeds or public image CDNs).
3. **Never Trust `null` Origin**: Reject requests where `Origin === 'null'`.

---

## Key Takeaways

* Misconfigured CORS headers completely bypass browser Same-Origin Policy protections.
* Test APIs by sending arbitrary attacker origins and verifying they are rejected.
* Never allow origin reflection paired with `Access-Control-Allow-Credentials: true`.

---

## Conclusion

CORS security testing is a fundamental aspect of API quality assurance. By methodically fuzzing origin headers, validating credentials flags, and verifying strict whitelist enforcement, QA engineers protect customer accounts and sensitive business data from cross-origin theft.
