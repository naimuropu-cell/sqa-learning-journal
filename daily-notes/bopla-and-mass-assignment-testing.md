# BOPLA & Mass Assignment API Security Testing Guide

## Introduction

In the **OWASP API Security Top 10 (2023 edition)**, two previously distinct vulnerabilities—*Mass Assignment* and *Excessive Data Exposure*—were merged into a single high-risk category: **Broken Object Property Level Authorization (BOPLA / API3:2023)**.

BOPLA occurs when an API endpoint allows users to read or write specific object properties that they should not have authorization to access.

Modern web frameworks (such as Ruby on Rails, Spring Boot, ASP.NET Core, and Express/Mongoose) feature automatic object-binding: they conveniently take incoming JSON request bodies and automatically map all key-value pairs directly to database models. If developers do not strictly filter these properties, attackers can inject unauthorized properties—such as granting themselves administrative roles or altering account balances.

---

## How Mass Assignment Exploitation Operates

```
NORMAL USER REQUEST (Updating Profile Name)
POST /api/v1/users/profile
{
  "name": "Jane Doe"
}

ATTACKER'S TAMPERED PAYLOAD (Injecting Admin Privileges)
POST /api/v1/users/profile
{
  "name": "Jane Doe",
  "isAdmin": true,           ◄── INJECTED PROPERTY
  "role": "SUPERADMIN",      ◄── INJECTED PROPERTY
  "accountBalance": 1000000  ◄── INJECTED PROPERTY
}

UNPROTECTED BACKEND ORM:
User.update(req.body); // Automatically binds ALL keys to DB table!
// Result: Attacker is now an Administrator! 🚨
```

---

## The Flip Side: Excessive Data Exposure

BOPLA also encompasses unauthorized **reading** of object properties:
* **The Glitch**: A developer queries the database: `SELECT * FROM users WHERE id = 101` and returns the raw database object directly to the client:
  ```json
  {
    "id": 101,
    "username": "janedoe",
    "passwordHash": "$2b$12$e8...", ◄── EXPOSED SENSITIVE PROPERTY!
    "failedLoginAttempts": 0,
    "internalNotes": "High churn risk user"
  }
  ```
* The developer assumes: *"The React UI only displays the username, so it's fine."*
* **The Reality**: An attacker opens browser DevTools or uses Postman to inspect the raw JSON payload, immediately harvesting password hashes and internal company notes!

---

## How QA Engineers Test for BOPLA

```
┌─────────────────────────────────────────────────────────────┐
│                     BOPLA Testing Strategy                  │
├─────────────────────┬───────────────────────────────────────┤
│ 1. Property Fuzzing │ Injecting privileged keys (role,      │
│                     │ isAdmin, balance, planTier) in PATCH  │
├─────────────────────┼───────────────────────────────────────┤
│ 2. Response Audit   │ Inspecting API responses for hidden   │
│                     │ PII, internal IDs, password hashes    │
├─────────────────────┼───────────────────────────────────────┤
│ 3. DTO Validation   │ Verifying backend uses Data Transfer  │
│                     │ Objects (DTOs) with strict allowlists │
└─────────────────────┴───────────────────────────────────────┘
```

---

## Practical Test Automation: BOPLA Fuzzing in Postman

Here is a Postman script testing whether an e-commerce checkout endpoint permits mass assignment price manipulation:

```javascript
// Postman Pre-Request Script: Injecting tampered price property
const tamperedPayload = {
    productId: "prod_macbook_pro_16",
    quantity: 1,
    unitPrice: 1.00 // Real price is $2499.00! Attempting mass assignment!
};
pm.request.body.raw = JSON.stringify(tamperedPayload);

// Postman Test Tab: Verifying backend ignores or rejects client-side price
pm.test("Status code is 201 Created", function () {
    pm.response.to.have.status(201);
});

pm.test("Backend rejects or ignores client-supplied unitPrice", function () {
    var responseJson = pm.response.json();
    
    // The server MUST calculate price from its own database, not client input!
    pm.expect(responseJson.totalAmount).to.be.above(2000.00);
    pm.expect(responseJson.totalAmount).to.not.eql(1.00);
});
```

---

## Remediation Best Practices QA Must Verify

1. **Strict Data Transfer Objects (DTOs)**: Ensure backend endpoints bind incoming requests to strict input DTO classes containing *only* the fields the user is permitted to edit.
2. **Explicit Allowlists**: Never rely on blocklists (e.g., "ignore `isAdmin`"). Always define an explicit allowlist (e.g., "only allow `name` and `phoneNumber`").
3. **Response Serialization**: Ensure responses are mapped through output DTOs or serializers (e.g., Class Transformers) that explicitly exclude internal flags and password hashes.

---

## SQA Interview Questions & Answers

### Q: What is the difference between BOLA and BOPLA in the OWASP API Top 10?
**Answer:**
* **BOLA (Broken Object Level Authorization - API1:2023)** deals with accessing entirely different object records by swapping IDs (e.g., User A accessing User B's invoice at `/invoices/1002`).
* **BOPLA (Broken Object Property Level Authorization - API3:2023)** deals with reading or altering specific sensitive *properties* within an object (e.g., User A editing their own profile and injecting `"role": "admin"`, or reading another user's public profile and receiving their private `ssn` field in the JSON response).

### Q: Why is relying on frontend UI validation insufficient to prevent Mass Assignment?
**Answer:**
Frontend UI validation only controls what is submitted via the browser form. Anyone can bypass the frontend using API clients (Postman, curl, Burp Suite) and append arbitrary JSON properties directly into the HTTP request body. The backend must enforce property authorization independently of the frontend.

---

## Key Takeaways

* BOPLA combines Mass Assignment and Excessive Data Exposure into one critical API security risk.
* Fuzz POST, PUT, and PATCH payloads with privileged properties (`isAdmin`, `role`, `status`, `price`).
* Audit all API JSON responses to verify that internal database columns and password hashes are never serialized to the client.

---

## Conclusion

Securing object properties is essential for API defense. By systematically testing for unauthorized property injection and auditing responses for sensitive data leaks, QA engineers protect applications from privilege escalation and data theft.
