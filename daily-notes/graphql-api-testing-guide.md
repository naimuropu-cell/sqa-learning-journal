# GraphQL API Testing Guide for QA Engineers

## Introduction

In traditional REST APIs, servers define rigid data shapes across dozens of distinct endpoints (`GET /users`, `GET /users/1/orders`, `POST /products`). Clients frequently encounter two major issues:
* **Over-fetching**: Downloading more data than needed (e.g., retrieving a 40-field user object just to display a username).
* **Under-fetching**: A single screen needing data from three different REST endpoints, causing sequential roundtrips.

Developed by Meta, **GraphQL** is a query language for APIs that provides a complete, understandable description of the data in your API, giving clients the power to ask for exactly what they need and nothing more via a single endpoint (typically `POST /graphql`).

---

## REST vs. GraphQL Comparison

| Dimension | REST API | GraphQL API |
| :--- | :--- | :--- |
| **Endpoints** | Multiple (`/users`, `/posts`, `/comments`) | Single endpoint (`/graphql`) |
| **HTTP Methods** | `GET`, `POST`, `PUT`, `DELETE`, `PATCH` | Almost exclusively `POST` |
| **Payload Control** | Fixed response structure dictated by server | Dynamic response structure requested by client |
| **HTTP Status Codes**| Standard status codes (`200`, `400`, `404`, `500`) | Often returns `200 OK` with an `errors` array |
| **Versioning** | URI versioning (`/api/v1/`, `/api/v2/`) | Single evolving schema with `@deprecated` tags |

---

## Core GraphQL Operations

```
┌─────────────────────────────────────────────────────────────┐
│                      GraphQL Operations                     │
└─────────────────────────────────────────────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         ▼                     ▼                     ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│     Query        │  │     Mutation     │  │   Subscription   │
│  (Read-only data │  │ (Create, update, │  │ (Real-time live  │
│   retrieval)     │  │  or delete data) │  │  event updates)  │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

### 1. Query Example (Read Operation)
```graphql
query GetUserOrders($userId: ID!) {
  user(id: $userId) {
    id
    username
    orders(status: COMPLETED) {
      id
      totalAmount
      currency
    }
  }
}
```

### 2. Mutation Example (Write Operation)
```graphql
mutation CreateNewProduct($input: ProductInput!) {
  createProduct(input: $input) {
    id
    title
    price
    createdAt
  }
}
```

---

## The Critical Gotcha: "HTTP 200 OK with Errors"

In REST APIs, an error returns HTTP 4xx or 5xx. In GraphQL, the server frequently responds with **HTTP 200 OK**, even when an operation completely fails! The errors are returned inside an `errors` array:

```json
{
  "errors": [
    {
      "message": "Cannot query field 'nonExistentField' on type 'User'.",
      "locations": [{ "line": 2, "column": 3 }],
      "extensions": {
        "code": "GRAPHQL_VALIDATION_FAILED"
      }
    }
  ],
  "data": null
}
```

> [!CAUTION]
> **QA Assertion Rule**: Never rely solely on `pm.response.to.have.status(200)` in GraphQL tests. Always assert that `response.errors` is `undefined` or empty for positive test cases!

---

## QA Testing Checklist for GraphQL

1. **Schema Introspection**: Test if introspection queries (`__schema`) are disabled in production to prevent attackers from discovering hidden queries or internal types.
2. **Query Complexity & Depth Limiting**: Send nested queries (e.g., user -> friends -> friends -> friends) to verify the server limits depth and prevents Denial of Service (DoS).
3. **Field-Level Authorization**: Verify that unauthenticated or standard users cannot query sensitive fields like `user.passwordHash` or `user.ssn`.
4. **Negative Mutation Tests**: Verify validation errors when required non-null fields (`!`) are omitted.

---

## Practical Test Automation: GraphQL with Playwright / JavaScript

```typescript
import { test, expect } from '@playwright/test';

test.describe('GraphQL Product Catalog API', () => {
  const GRAPHQL_ENDPOINT = 'https://api.example.com/graphql';

  test('Query product by ID returns valid data without errors', async ({ request }) => {
    const query = `
      query GetProduct($id: ID!) {
        product(id: $id) {
          id
          title
          price
        }
      }
    `;

    const response = await request.post(GRAPHQL_ENDPOINT, {
      data: {
        query,
        variables: { id: "prod_101" },
      },
      headers: {
        'Content-Type': 'application/json',
      },
    });

    expect(response.status()).toBe(200);

    const body = await response.json();
    
    // Crucial: Assert no errors array exists
    expect(body.errors).toBeUndefined();
    
    // Assert response structure
    expect(body.data.product).toBeDefined();
    expect(body.data.product.id).toBe("prod_101");
    expect(typeof body.data.product.price).toBe('number');
  });

  test('Requesting non-existent entity returns proper GraphQL error', async ({ request }) => {
    const query = `
      query GetProduct($id: ID!) {
        product(id: $id) {
          id
        }
      }
    `;

    const response = await request.post(GRAPHQL_ENDPOINT, {
      data: {
        query,
        variables: { id: "invalid_9999" },
      },
    });

    const body = await response.json();
    expect(body.errors).toBeDefined();
    expect(body.errors[0].message).toContain('Product not found');
  });
});
```

---

## SQA Interview Questions & Answers

### Q: Why can an automated test pass in GraphQL even when the query failed?
**Answer:**
Because GraphQL processes queries at the application level over a single HTTP POST endpoint. Even if the query syntax is invalid or a resolver throws an exception, the HTTP transport layer successfully delivered the request, resulting in an HTTP 200 status code. The failure details are enclosed inside the response body's `errors` array. QA tests must parse the body and assert that `errors` does not exist.

### Q: What is Schema Introspection and why is it important for testing?
**Answer:**
Introspection is GraphQL's ability to query its own schema (types, fields, queries, and mutations). QA uses introspection to automatically generate test cases and validate documentation. However, QA must also verify that introspection is strictly disabled in production environments to prevent reconnaissance attacks.

---

## Key Takeaways

* GraphQL provides dynamic querying capabilities over a single POST endpoint.
* Always assert on `data` and ensure `errors` is empty—never trust an HTTP 200 status code alone.
* Verify security protections like depth limiting, query cost analysis, and introspection suppression in production.

---

## Conclusion

Testing GraphQL APIs requires shifting from HTTP status code verification to payload structure and schema validation. By mastering queries, mutations, error handling, and query complexity checks, QA engineers ensure high-performing and secure GraphQL services.
