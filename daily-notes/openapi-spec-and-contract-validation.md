# OpenAPI / Swagger Specification & Contract Validation Testing Guide

## Introduction

In modern API-first development, the **OpenAPI Specification (OAS)** (formerly known as **Swagger**) serves as the single source of truth defining an API's endpoints, request parameters, response schemas, and authentication models.

However, in fast-moving development teams, **Documentation Drift** is a rampant issue: developers update backend endpoints, rename response properties, or alter data types without updating the OpenAPI document. When client teams (mobile and web) build against the outdated specification, integration breaks.

**Contract Validation Testing** is the automated practice of validating that live, running API responses strictly adhere to the published OpenAPI specification, preventing contract drift before code reaches production.

---

## The Contract Validation Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                 OpenAPI Specification (OAS)                 │
│         (swagger.yaml / openapi.json Contract)              │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│               Automated Contract Validation Tool            │
│                 (Dredd / Prism / Ajv Engine)                │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 Live API Endpoint (Staging)                 │
│                 GET /api/v1/users/101                       │
└──────────────────────────────┬──────────────────────────────┘
                               │
            ┌──────────────────┴──────────────────┐
            ▼                                     ▼
┌──────────────────────┐               ┌──────────────────────┐
│ Schema Matches Spec  │               │ Schema Drift Detected│
│ Test PASSES ✅       │               │ Missing 'email' or   │
│                      │               │ type mismatch ❌     │
└──────────────────────┘               └──────────────────────┘
```

---

## The Core Elements of an OpenAPI Contract

A valid OpenAPI 3.0 document defines exact parameter types and required response fields:

```yaml
paths:
  /users/{id}:
    get:
      summary: Fetch user by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                required:
                  - id
                  - username
                  - email
                  - isActive
                properties:
                  id:
                    type: integer
                  username:
                    type: string
                  email:
                    type: string
                    format: email
                  isActive:
                    type: boolean
```

---

## Practical Tooling for Contract Validation

1. **Dredd**: A command-line tool that parses your `openapi.yaml` file, automatically generates HTTP requests for every declared endpoint, fires them against your backend server, and validates that the responses match the schemas.
   ```bash
   dredd ./openapi.yaml https://staging-api.example.com
   ```
2. **Prism**: An open-source HTTP proxy that inspects incoming requests and outgoing responses against the specification in real-time. If an API returns an extra unapproved field or violates a type, Prism logs a contract violation.
3. **Spectral**: An API linter that enforces style rules on the OpenAPI file itself (e.g., ensuring all operations have tags, descriptions, and standard 4xx error responses).

---

## Automated Schema Assertion in JavaScript with `ajv`

QA engineers can validate live API responses against JSON Schemas extracted from OpenAPI files using **Ajv (Another JSON Schema Validator)**:

```typescript
import { test, expect } from '@playwright/test';
import Ajv from 'ajv';
import addFormats from 'ajv-formats';

const ajv = new Ajv({ allErrors: true });
addFormats(ajv); // Supports 'email', 'date-time', 'uuid' formats

// Schema extracted from OpenAPI spec
const userSchema = {
  type: 'object',
  required: ['id', 'username', 'email', 'isActive'],
  properties: {
    id: { type: 'integer' },
    username: { type: 'string', minLength: 3 },
    email: { type: 'string', format: 'email' },
    isActive: { type: 'boolean' },
  },
  additionalProperties: false, // Disallow undocumented rogue properties
};

test('Validate GET /users/101 strictly satisfies OpenAPI contract', async ({ request }) => {
  const response = await request.get('https://api.example.com/v1/users/101');
  expect(response.status()).toBe(200);

  const responseBody = await response.json();

  // Validate live response against contract
  const validate = ajv.compile(userSchema);
  const isValid = validate(responseBody);

  if (!isValid) {
    console.error('Contract Schema Violations:', validate.errors);
  }

  expect(isValid).toBe(true);
});
```

---

## SQA Interview Questions & Answers

### Q: What is the difference between Schema Validation and Business Logic Testing in APIs?
**Answer:**
* **Schema Validation** verifies structural compliance with the contract: it asserts that data types are correct, required properties are present, and formatting rules (e.g., email or ISO-8601 dates) are respected.
* **Business Logic Testing** verifies data accuracy and operational behavior: it asserts that a user's calculated cart total matches applicable tax rates, or that deleting an account marks the record as inactive in the database. Both are essential for complete API quality.

### Q: Why should an API contract enforce `additionalProperties: false` during testing?
**Answer:**
Setting `additionalProperties: false` ensures that the API cannot return undocumented or accidental properties that are not declared in the specification. This catches unintended data leaks (such as internal database columns, debugging flags, or password hashes) and forces developers to keep the documentation synchronized with the code.

---

## Key Takeaways

* OpenAPI/Swagger specifications are living contracts between API producers and consumers.
* Contract testing prevents documentation drift by automatically validating live API payloads against the schema.
* Tools like Dredd, Prism, and Ajv automate schema assertions inside CI/CD pipelines.

---

## Conclusion

Automating OpenAPI contract validation guarantees that API documentation remains an authoritative, trustworthy reference. By treating the API specification as an executable test suite, QA engineers eliminate communication friction and prevent unexpected breaking changes across modern software architectures.
