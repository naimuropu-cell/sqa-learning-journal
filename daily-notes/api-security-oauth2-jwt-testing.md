# API Security Testing: OAuth 2.0, JWT, and OWASP API Vulnerabilities

## Introduction

APIs are the primary conduits through which data moves across web, mobile, and cloud services. Because APIs expose underlying application logic and sensitive customer databases directly to the internet, they are the number one target for cyberattacks.

According to the **OWASP API Security Top 10**, vulnerabilities like Broken Object Level Authorization (BOLA) and Broken Authentication account for over 60% of enterprise security breaches. QA engineers must actively test security controls around **OAuth 2.0** and **JSON Web Tokens (JWT)** rather than treating security as a developer-only concern.

---

## JSON Web Token (JWT) Anatomy

A JWT consists of three Base64URL-encoded parts separated by periods (`.`):

```
┌─────────────────────────────────────────────────────────────┐
│ eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9                        │ ◄─── 1. Header (Algorithm & Token Type)
├─────────────────────────────────────────────────────────────┤
│ .eyJzdWIiOiIxMjM0NTYiLCJuYW1lIjoiQXB1Iiwicm9sZSI6InVzZXIifQ│ ◄─── 2. Payload (Claims & Identity)
├─────────────────────────────────────────────────────────────┤
│ .SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c                │ ◄─── 3. Signature (Cryptographic Verification)
└─────────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> **Key Security Reality**: The payload in a standard JWT is **encoded**, NOT encrypted. Anyone can decode and read the contents of a JWT payload. Never store sensitive secrets (passwords, social security numbers) inside a JWT!

---

## Critical API Security Test Scenarios

```
                               ┌─────────────────────────┐
                               │   API Security Tests    │
                               └────────────┬────────────┘
                                            │
         ┌──────────────────┬───────────────┴───────────────┬──────────────────┐
         ▼                  ▼                               ▼                  ▼
┌──────────────────┐ ┌──────────────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│   BOLA / IDOR    │ │      JWT Tampering       │ │      BFLA        │ │  Rate Limiting   │
│ (Accessing other │ │ (Alg: none, expired exp, │ │ (Standard user   │ │ (Brute forcing   │
│  users' data)    │ │  tampered user_id)       │ │  calling admin)  │ │  login/endpoints)│
└──────────────────┘ └──────────────────────────┘ └──────────────────┘ └──────────────────┘
```

### 1. Broken Object Level Authorization (BOLA / IDOR)
The most common and severe API defect:
* **Attack Vector**: User A logs in and accesses their invoice at `/api/v1/invoices/1045`. User A then manually alters the request URI to `/api/v1/invoices/1046` (belonging to User B).
* **Expected QA Assertion**: The server must return **HTTP 403 Forbidden** or **HTTP 404 Not Found**, never the invoice details of User B!

### 2. Broken Function Level Authorization (BFLA)
* **Attack Vector**: A standard customer sends an HTTP request to an administrative endpoint (e.g., `DELETE /api/v1/users/52` or `POST /api/v1/admin/discount-codes`).
* **Expected QA Assertion**: Server returns **HTTP 403 Forbidden**.

### 3. JWT "None" Algorithm & Tampering Attack
* **Attack Vector**: An attacker decodes the JWT header, changes `"alg": "HS256"` to `"alg": "none"`, strips the signature, and alters their role from `"role": "user"` to `"role": "admin"`.
* **Expected QA Assertion**: Server rejects unsigned tokens with **HTTP 401 Unauthorized**.

### 4. Expired Token Enforcement
* **Attack Vector**: Sending an API request using an access token whose expiration timestamp (`exp`) has elapsed.
* **Expected QA Assertion**: The API must reject the token with **HTTP 401 Unauthorized** and require token renewal via the refresh token endpoint.

---

## Automated Security Assertions in Postman

Here is a Postman test script verifying authorization enforcement on a resource endpoint:

```javascript
// Postman Test Tab: Validating BOLA Prevention
pm.test("Status code is 403 Forbidden when accessing another user's resource", function () {
    pm.response.to.have.status(403);
});

pm.test("Response body contains authorization error message", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.error).to.eql("ACCESS_DENIED");
    pm.expect(jsonData.message).to.contain("You are not authorized to view this record");
});

pm.test("Sensitive user information is not leaked", function () {
    var responseText = pm.response.text();
    pm.expect(responseText).to.not.include("creditCard");
    pm.expect(responseText).to.not.include("passwordHash");
});
```

---

## OAuth 2.0 Token Refresh Testing

When testing OAuth 2.0 implementations, verify the complete token refresh lifecycle:

```
[ Expired Access Token ] ──► (API returns HTTP 401)
         │
         ▼
[ POST /oauth/v2/token ] ──► (Payload: refresh_token, client_id, client_secret)
         │
         ▼
[ New Access Token Generated ]
         │
         ▼
[ Subsequent API Request Succeeds ]
```

* **Test Case**: Attempting to reuse an old refresh token after a new one was issued must immediately invalidate all sessions (Refresh Token Rotation).

---

## SQA Interview Questions & Answers

### Q: What is BOLA (Broken Object Level Authorization) and why is it so prevalent in APIs?
**Answer:**
BOLA (formerly known as IDOR - Insecure Direct Object Reference) occurs when an endpoint accepts an object identifier (e.g., `/accounts/{accountId}`) from the client and retrieves the object without verifying that the authenticated user actually owns that object. It is prevalent because developers often rely solely on generic authentication gates (`isAuthenticated()`) without implementing object-level permission checks (`account.ownerId === currentUser.id`).

### Q: Why should tokens have a short expiration time (TTL)?
**Answer:**
Because access tokens are stateless, revoking a compromised JWT before it expires is difficult without complex token blacklisting. Giving access tokens a short lifespan (e.g., 5 to 15 minutes) minimizes the window of opportunity for an attacker if a token is intercepted.

---

## Key Takeaways

* Never trust user input or assume authentication equals authorization.
* Actively test for BOLA by swapping user IDs in API endpoints.
* Verify that expired, tampered, or algorithm-manipulated JWTs are strictly rejected by the server.

---

## Conclusion

API security testing is a fundamental responsibility of modern QA engineers. By incorporating BOLA checks, token tampering tests, and authorization audits into regular test suites, QA teams protect user privacy and fortify business infrastructure against costly security incidents.
