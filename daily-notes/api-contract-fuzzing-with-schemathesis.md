# Automated API Contract Fuzzing & Property-Based Testing with Schemathesis

## 1. Beyond Example-Based API Testing

Traditional API automated test cases are almost always **example-based**:
- A QA engineer writes specific assertions using hardcoded values: `POST /users with { "name": "John", "age": 25 } -> expect 201 Created`.
- Negative test cases test a handful of known edge cases: `POST /users with { "age": -1 } -> expect 400 Bad Request`.

While example-based tests verify known "happy paths" and anticipated error states, they rarely uncover unknown boundary defects, unhandled null pointers, or memory faults hidden deep within input combinations.

**Schemathesis** is a modern, property-based testing and contract fuzzing tool built on Python's **Hypothesis** engine. It reads an application's **OpenAPI (Swagger)** or **GraphQL** specification and automatically generates thousands of pseudo-random, edge-case, and malformed payloads to test server compliance against its own contract.

```
┌────────────────────────────────────────────────────────┐
│               OpenAPI / Swagger Spec                   │
│   - Paths, Query Params, Headers, Request Schemas      │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│                  Schemathesis Engine                   │
│  - Hypothesis-driven Property Generator                │
│  - Boundary Values (MAX_INT, Unicode, Null bytes, NaN) │
│  - Type Violations, Structural Array Overflow          │
└──────────────────────────┬─────────────────────────────┘
                           │
                 Thousands of Generated Calls
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│                   Target Web API                       │
│  - Assert: No Unhandled 5xx Server Errors              │
│  - Assert: Response Matches OpenAPI Schema             │
│  - Assert: Proper Content-Type & Status Code           │
└────────────────────────────────────────────────────────┘
```

---

## 2. Defects Caught by Schemathesis That QA Often Misses

1. **Unhandled Internal Server Errors (HTTP 500)**: Any request payload—no matter how bizarre or malformed—must return a clean `4xx Client Error` if rejected, never an unhandled `500 Internal Server Error`.
2. **Schema Conformance Violations**: The API returns fields not declared in the OpenAPI schema, or returns data types that contradict the contract (e.g., returning `null` when a field is marked `nullable: false`).
3. **Missing Header Validation**: Server crashes when receiving missing or invalid `Content-Type` or `Accept` headers.
4. **Regex DoS (ReDoS) & Payload Explosions**: Large string inputs or deeply nested arrays that trigger excessive memory or CPU consumption.

---

## 3. Running Schemathesis via CLI

Schemathesis can run directly against a live staging server or local Docker container without writing boilerplate test scripts:

```bash
# Basic fuzzing run with built-in checks
st run https://api.staging.example.com/openapi.json \
  --base-url=https://api.staging.example.com \
  --checks=all \
  --workers=4 \
  --max-response-time=1500 \
  --header="Authorization: Bearer eyJhbGciOi..."
```

### Built-in Check Options (`--checks=all`):
- `not_a_server_error`: Ensures no request produces an HTTP `5xx` response code.
- `status_code_conformance`: Ensures returned status codes match status codes documented in the schema.
- `content_type_conformance`: Ensures returned `Content-Type` matches the specification.
- `response_schema_conformance`: Validates that the response JSON strictly matches the schema definition.

---

## 4. Pytest Integration with Custom Stateful Workflows

QA engineers can integrate Schemathesis into **Pytest** to define custom authentication hooks, data fixtures, and stateful sequences:

```python
import pytest
import schemathesis
from schemathesis.checks import not_a_server_error, response_schema_conformance

# Load OpenAPI specification
schema = schemathesis.from_uri("http://localhost:8000/api/v1/openapi.json")

@schema.parametrize()
def test_api_contract_conformance(case):
    # Dynamically inject QA test auth headers
    case.headers = {
        "Authorization": "Bearer qa-automation-jwt-token",
        "X-QA-Run": "schemathesis-fuzz-suite"
    }

    # Dispatch generated test case
    response = case.call()

    # Assert standard property checks
    case.validate_response(
        response,
        checks=(
            not_a_server_error,
            response_schema_conformance,
        )
    )

    # Custom Assertion: Response time must remain under 800ms even under boundary payloads
    assert response.elapsed.total_seconds() < 0.8, (
        f"Latency violation! {case.method} {case.formatted_path} took {response.elapsed.total_seconds()}s"
    )
```

---

## 5. Automated CI/CD Quality Gate (GitHub Actions)

Incorporate Schemathesis as a nightly or pre-release pull request quality gate:

```yaml
name: API Contract Fuzzing Quality Gate

on:
  schedule:
    - cron: '0 2 * * *' # Run nightly at 2 AM
  workflow_dispatch:

jobs:
  schemathesis-fuzz:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Schemathesis
        run: pip install schemathesis

      - name: Execute API Contract Fuzzing
        run: |
          st run ${{ secrets.STAGING_OPENAPI_URL }} \
            --checks=all \
            --hypothesis-max-examples=100 \
            --exitfirst \
            --report=junit \
            --junit-xml=schemathesis-results.xml
```

---

## 6. QA Verification Checklist for API Fuzzing

- [ ] **Contract Coverage**: Ensure the OpenAPI specification accurately documents all valid endpoints, query parameters, and request bodies.
- [ ] **Zero 500 Errors**: Strict requirement that malformed data, emojis, unicode characters, and boundary numbers return `4xx` responses, never `500`.
- [ ] **Negative Boundary Generation**: Verify fuzz tests test edge numbers (e.g., `-2^31`, `0`, `2^31-1`, floating-point values in integer fields).
- [ ] **Sanitization of Error Messages**: Confirm that `4xx` and `5xx` responses never leak sensitive database stack traces or SQL syntax snippets to consumers.
