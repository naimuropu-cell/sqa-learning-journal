# JWT Security Vulnerabilities & Penetration Testing Guide for QA

## Introduction

JSON Web Tokens (JWT - RFC 7519) are the dominant standard for stateless authentication and authorization across modern web and mobile APIs. Because JWTs are cryptographically signed and self-contained, backend microservices can verify user identities without performing a database lookup on every incoming request.

However, improper implementation of JWT libraries, weak secret keys, and flawed signature validation logic introduce severe security vulnerabilities.

QA engineers must possess hands-on understanding of **JWT attack vectors** to ensure that authentication and authorization mechanisms cannot be bypassed.

---

## JWT Anatomy & The Signature Contract

```
┌─────────────────────────────────────────────────────────────┐
│ eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9                        │ ◄── HEADER: Algorithm & Token Type
├─────────────────────────────────────────────────────────────┤
│ .eyJzdWIiOiIxMDEiLCJuYW1lIjoiQXB1Iiwicm9sZSI6InVzZXIifQ     │ ◄── PAYLOAD: Claims (User ID, Role)
├─────────────────────────────────────────────────────────────┤
│ .SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c                │ ◄── SIGNATURE: Cryptographic Hash
└─────────────────────────────────────────────────────────────┘
```

The signature guarantees **integrity**: if an attacker modifies a single character in the header or payload (e.g., changing `"role": "user"` to `"role": "admin"`), the cryptographic signature becomes invalid and the server rejects the token.

---

## The Top 5 JWT Security Vulnerabilities

```
┌─────────────────────────────────────────────────────────────┐
│                     Top JWT Vulnerabilities                 │
├─────────────────────┬───────────────────────────────────────┤
│ Vulnerability       │ Attack Mechanism                      │
├─────────────────────┼───────────────────────────────────────┤
│ 1. The "alg: none"  │ Setting algorithm to "none" and       │
│    Bypass           │ stripping signature completely        │
├─────────────────────┼───────────────────────────────────────┤
│ 2. Key Confusion    │ Tricking server into verifying an     │
│    (HMAC vs. RSA)   │ asymmetric RSA token with HMAC secret │
├─────────────────────┼───────────────────────────────────────┤
│ 3. Weak Secret Keys │ Brute-forcing weak HS256 secret       │
│                     │ passwords (e.g., "secret123")         │
├─────────────────────┼───────────────────────────────────────┤
│ 4. Expired Token    │ Server decodes token without checking │
│    Acceptance       │ the `exp` (expiration) timestamp claim│
├─────────────────────┼───────────────────────────────────────┤
│ 5. Missing Signature│ Server parses JSON payload without    │
│    Verification     │ validating the signature hash         │
└─────────────────────┴───────────────────────────────────────┘
```

---

## 1. The "alg: none" Signature Bypass Attack

The original JWT specification allowed tokens to be unsigned by declaring the algorithm as `none` (intended for debugging).
* **The Exploit**:
  1. Take a valid JWT.
  2. Base64-decode the header: `{"alg": "HS256", "typ": "JWT"}`.
  3. Change the algorithm to: `{"alg": "none", "typ": "JWT"}`.
  4. Base64-decode the payload: change `"role": "user"` to `"role": "admin"`.
  5. Delete the signature portion (keeping the trailing period: `header.payload.`).
* **QA Test**: Replay the request with this unsigned token in Postman.
* **Expected Result**: Server must return **HTTP 401 Unauthorized**. Flawed libraries accept `alg: none` and grant administrative access!

---

## 2. Algorithm Confusion (HMAC vs. RSA)

* **Asymmetric Tokens (RS256)**: Signed using a **Private Key** (kept secret on the auth server) and verified using a **Public Key** (publicly downloadable by everyone).
* **Symmetric Tokens (HS256)**: Both signed and verified using the same shared **Secret Key**.
* **The Exploit**: An attacker takes the server's publicly available RSA Public Key (which is public knowledge), changes the token header to `"alg": "HS256"`, and signs the tampered token using the Public Key as the HMAC secret! Flawed backends expecting HMAC will verify the token using the public key and accept the forged token!
* **QA Defense**: Verify that the backend explicitly restricts allowed algorithms: `algorithms: ['RS256']`.

---

## 3. Testing Expired Tokens (`exp` claim)

JWTs must include an expiration timestamp claim (`exp`).
* **QA Test**:
  1. Capture a valid JWT from the browser DevTools.
  2. Wait until the token expires (or craft a token whose `exp` claim is in the past: `Date.now() - 3600`).
  3. Send an API request using the expired token.
  4. **Expected Result**: Server must return **HTTP 401 Unauthorized** with error: `"jwt expired"`.

---

## SQA Interview Questions & Answers

### Q: Why is it dangerous to store sensitive data (like passwords or credit card numbers) inside a JWT payload?
**Answer:**
Standard JWT payloads are **Base64URL-encoded, NOT encrypted**. Anyone who intercepts or possesses the token can paste it into `jwt.io` or run `atob()` in browser DevTools to read all claims in plain text within a fraction of a second. Only non-sensitive public identifiers (such as user ID, public username, and role permissions) should ever be included in a JWT payload.

### Q: How should an application revoke a compromised JWT before its expiration date?
**Answer:**
Because JWTs are stateless, the server does not check a database session table on each request. To revoke a token, the backend must maintain a **Token Blacklist (Revocation List)** in a high-speed memory cache (like Redis) storing revoked token IDs (`jti` claim). When a user logs out or changes passwords, the token ID is blacklisted until its natural expiration timestamp elapses.

---

## Key Takeaways

* Never trust user-supplied tokens; verify that backends strictly enforce signatures.
* Test against `alg: none` and algorithm confusion (RS256 vs HS256) attacks.
* Verify that expired tokens (`exp`) are rejected immediately with HTTP 401.

---

## Conclusion

JWT security testing is essential for protecting modern API infrastructure. By testing algorithm restrictions, signature enforcement, and token lifecycles, QA engineers ensure that stateless authentication remains resilient against spoofing and privilege escalation exploits.
