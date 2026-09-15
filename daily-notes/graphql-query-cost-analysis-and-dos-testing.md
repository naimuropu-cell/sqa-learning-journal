# GraphQL Query Cost Analysis & DoS Security Testing

## 1. Overview: The Vulnerability of GraphQL Flexibility

Unlike REST APIs where endpoints expose fixed, pre-calculated resource models (e.g., `GET /users/123`), GraphQL grants clients complete flexibility to request arbitrary fields, nested relationships, and multiple root operations in a single HTTP request.

While this flexibility prevents over-fetching and under-fetching on mobile and web clients, it introduces significant **Denial of Service (DoS)** and resource exhaustion vulnerabilities:
- **Deeply Nested Queries (Cyclic Graph Traversal)**: Traversal between circular relationships (e.g., `User -> Friends -> Friends -> Friends`) causes exponential database queries.
- **Batch Query Flooding**: Packing hundreds of expensive queries into a single JSON array payload.
- **Query Complexity Explosion**: Requesting high-cardinality pagination fields with large limit multipliers (e.g., `authors(limit: 1000) { books(limit: 1000) { reviews(limit: 1000) } }`).

QA and Application Security Engineers must verify defensive protections including **Max Query Depth**, **Static Query Cost Analysis**, **Field Rate Limiting**, and **Persisted Queries**.

```
┌────────────────────────────────────────────────────────┐
│            Incoming GraphQL Query Request              │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│                GraphQL Validation Phase                │
│  1. Depth Limiter Check (e.g., Max Depth = 5)         │
│  2. Cost Analyzer (e.g., Calculated Cost <= 1000)      │
└──────────┬─────────────────────────────────┬───────────┘
           │                                 │
     Cost <= Threshold                 Cost > Threshold
           │                                 │
           ▼                                 ▼
┌─────────────────────────┐       ┌──────────────────────┐
│  Execute Query Engine   │       │ Reject with 400 Error│
│  Resolve DB Resolvers   │       │ "Query too complex"  │
└─────────────────────────┘       └──────────────────────┘
```

---

## 2. Denial-of-Service Attack Vectors in GraphQL

| Attack Pattern | Example Payload | Server Impact | Defensive Control |
| :--- | :--- | :--- | :--- |
| **Circular / Recursive Query** | `query { user { friends { user { friends { user } } } } }` | Memory exhaustion & database deadlocks | **Max Depth Limiting** (e.g., limit depth to 5 levels) |
| **Field Multiplying / Pagination Bomb** | `query { orders(first: 10000) { items(first: 1000) { details } } }` | Fetches $10^7$ records in a single query | **Slicing Limits & Cost Multipliers** |
| **Alias Query Abuse** | `query { a1: expensiveOp, a2: expensiveOp, ... a1000: expensiveOp }` | Bypasses standard rate limiters by bundling 1000 ops into 1 HTTP POST | **Operation Complexity Summation** |
| **Directives Exhaustion** | Recursive `@include` or custom heavy directives | CPU spikes during AST parse phase | **AST Parsing Depth Limits** |

---

## 3. Query Complexity Calculation Algorithms

Security-conscious GraphQL engines calculate the static complexity of an Abstract Syntax Tree (AST) before running resolvers:

$$\text{Total Cost} = \sum (\text{Base Field Cost}) + \sum (\text{Multiplier} \times \text{Child Cost})$$

### Cost Example:
- Simple scalar field (`id`, `name`): **1 point**
- Database relation field (`orders`): **10 points**
- Paginated connection with multiplier (`items(first: 20)`): $20 \times \text{itemCost}$

```graphql
query ComplexOrderAudit {
  user(id: "123") {            # Cost = 1
    name                       # Cost = 1
    orders(first: 10) {        # Cost = 10 + (10 * children)
      orderId                  # Cost = 1
      items(first: 5) {        # Cost = 5 + (5 * 2) = 15
        sku                    # Cost = 1
        price                  # Cost = 1
      }
    }
  }
}
# Total Calculated Cost: 1 + 1 + 10 + (10 * (1 + 15)) = 172 points.
# If max server complexity budget is 150 points, this query is rejected immediately!
```

---

## 4. Automated Security Testing Suite with Python

Below is an automated probe testing whether a GraphQL endpoint properly enforces depth and complexity limits:

```python
import requests

GRAPHQL_URL = "https://staging.example.com/graphql"

def generate_deeply_nested_query(depth: int) -> str:
    """Generates a recursive circular query of specified depth."""
    query = "id\n"
    for _ in range(depth):
        query = f"author {{\n  id\n  books {{\n    {query}  }}\n}}"
    return f"query MaliciousDepthTest {{\n  {query}\n}}"

def test_graphql_max_depth_enforcement():
    # Construct a malicious query with depth = 15
    nested_query = generate_deeply_nested_query(depth=15)

    response = requests.post(
        GRAPHQL_URL,
        json={"query": nested_query},
        headers={"Content-Type": "application/json"},
        timeout=10
    )

    data = response.json()

    # Assert query was rejected and didn't crash the server
    assert response.status_code in [200, 400], f"Unexpected HTTP status: {response.status_code}"
    assert "errors" in data, "Query should have been rejected with validation errors!"

    error_message = data["errors"][0]["message"].lower()
    print(f"[*] Server Response: {error_message}")

    # Check for depth or complexity violation messages
    assert any(term in error_message for term in ["depth", "complexity", "exceeds maximum allowed"]), \
        f"Server failed to enforce depth limit! Error received: {error_message}"

def test_graphql_field_multiplier_cost_limit():
    # Request unreasonable page sizes designed to overwhelm memory
    multiplier_query = """
    query PaginationBomb {
      users(first: 100000) {
        id
        posts(first: 10000) {
          id
          comments(first: 1000) {
            id
            body
          }
        }
      }
    }
    """

    response = requests.post(GRAPHQL_URL, json={"query": multiplier_query}, timeout=10)
    data = response.json()

    assert "errors" in data, "Pagination bomb query was not rejected!"
    assert any(term in data["errors"][0]["message"].lower() for term in ["cost", "complexity", "limit", "first"]), \
        "Failed to enforce pagination multiplier budget!"
```

---

## 5. Defense-in-Depth Checklist for GraphQL Gateways

| Defensive Layer | Implementation Mechanism | Recommended Threshold |
| :--- | :--- | :--- |
| **Max Depth Limiting** | `graphql-depth-limit` | Maximum 5–7 levels deep |
| **Query Cost Analysis** | `graphql-cost-analysis` / Apollo Armor | Max cost 500–1000 points per request |
| **Persisted Queries (APQ)** | Client sends cryptographic SHA-256 hash instead of arbitrary query strings | Whitelist known queries in production; disable arbitrary strings entirely |
| **Introspection Disabled** | `introspection: false` in production | Prevent attackers from mapping schema surface |
| **Timeout Guard** | Server-side execution timeout | Abort resolver execution if taking $> 3000\text{ms}$ |

---

## 6. SQA Interview Questions & Answers

### Q1: Why is disabling GraphQL Introspection in production considered security through obscurity, and what real defenses should accompany it?
> **Answer**:
> Disabling introspection prevents tools like GraphQL Voyager or Altair from automatically mapping the entire schema, but it does not stop attackers from discovering endpoints through mobile app reverse engineering, guessing common field names, or intercepting frontend client traffic. 
> Real defense requires **hard constraints**: depth limiting, query cost analysis, resolver execution timeouts, and using **Persisted Queries** (where only pre-approved queries are executed).

### Q2: What are Automatic Persisted Queries (APQ), and how do they eliminate GraphQL query DoS attacks?
> **Answer**:
> With Persisted Queries, clients register approved GraphQL queries at build time, and the server assigns each query a cryptographic hash (e.g., SHA-256). In production, clients only send the hash and variables (`{"id": "<hash>", "variables": {...}}`). 
> The server looks up the approved query from a secure cache and executes it. Untrusted clients cannot send custom, deeply nested, or crafted DoS queries because arbitrary query text is completely rejected by the gateway.

---

## 7. Key Takeaways & Best Practices

- Integrate automated depth and complexity probes into your security test pipelines.
- Enforce strict upper bounds on all list pagination inputs (`first`, `limit`) at the schema level.
- Adopt Persisted Queries in production to eliminate arbitrary query injection attacks entirely.
