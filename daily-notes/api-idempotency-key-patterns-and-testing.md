# REST API Idempotency-Key Architecture, Replay & Race Condition Testing

## 1. Overview of HTTP Idempotency in Payment & Order APIs

In RESTful APIs, HTTP methods have defined semantics according to RFC 7231:
- `GET`, `PUT`, `DELETE`: **Idempotent by specification** (calling them multiple times yields the same end state).
- `POST`: **Non-idempotent by specification** (each invocation creates a new resource or triggers an action).

In financial transactions, e-commerce orders, and subscription billing, duplicate `POST` requests caused by network drops, browser button double-clicks, or aggressive mobile client retries can lead to catastrophic double-charges.

To solve this, APIs implement the **`Idempotency-Key`** header (standardized by Stripe, IETF, and major payment processors).

```
Client                              API Gateway / App Server                    Redis / Database
  │                                            │                                       │
  │ 1. POST /v1/charges                        │                                       │
  │    Header: Idempotency-Key: "idemp_abc123" │                                       │
  ├───────────────────────────────────────────►│                                       │
  │                                            │ 2. Check Key & Acquire Lock           │
  │                                            ├──────────────────────────────────────►│
  │                                            │    (Lock Acquired: Status = PENDING)  │
  │                                            │                                       │
  │                                            │ 3. Execute Charge Against Bank        │
  │                                            │                                       │
  │                                            │ 4. Store Result & Release Lock        │
  │                                            ├──────────────────────────────────────►│
  │ 5. HTTP 201 Created (Body: {id: "ch_99"})  │                                       │
  │◄───────────────────────────────────────────┤                                       │
  │                                            │                                       │
  │ 6. Duplicate POST /v1/charges              │                                       │
  │    Header: Idempotency-Key: "idemp_abc123" │                                       │
  ├───────────────────────────────────────────►│                                       │
  │                                            │ 7. Key Found! Return Stored Result    │
  │                                            ├──────────────────────────────────────►│
  │ 8. HTTP 201 Created (Cached Body Returned) │                                       │
  │◄───────────────────────────────────────────┤                                       │
```

---

## 2. Idempotency Edge Cases Every QA Engineer Must Test

1. **Exact Duplicate Replay**: Resending the exact same request with the same idempotency key must return the cached HTTP status code, headers, and response payload without reprocessing.
2. **Payload Mismatch Detection**: If a client sends the same `Idempotency-Key` with an altered body (e.g., changing `$50.00` to `$150.00`), the server must reject it with **`422 Unprocessable Entity`** or **`400 Bad Request`** with error code `idempotency_key_payload_mismatch`.
3. **In-Flight Concurrent Requests (Race Condition)**: If two identical requests arrive simultaneously before the first finishes processing, the server must either:
   - Queue the second request until the first finishes, or
   - Return **`409 Conflict`** with header `Retry-After: 2`.
4. **Key Expiration & TTL**: Idempotency keys must expire after a defined retention period (typically 24 to 48 hours).

---

## 3. Automated Concurrent Idempotency Test Suite (JavaScript)

Below is an automated test suite verifying duplicate handling, concurrent collision locking, and payload mutation detection:

```javascript
const axios = require('axios');
const { expect } = require('chai');
const crypto = require('crypto');

const BASE_URL = 'https://api.staging.example.com/v1';

describe('Payment API Idempotency Verification Suite', () => {
  function generateKey() {
    return `idem_${crypto.randomBytes(16).toString('hex')}`;
  }

  it('should return identical response payload on duplicate request replay', async () => {
    const idempotencyKey = generateKey();
    const payload = { amount: 2500, currency: 'usd', customer: 'cus_1001' };

    // Initial Request
    const res1 = await axios.post(`${BASE_URL}/charges`, payload, {
      headers: { 'Idempotency-Key': idempotencyKey },
    });

    expect(res1.status).to.equal(201);
    const chargeId = res1.data.id;
    expect(chargeId).to.be.a('string');

    // Duplicate Request with identical key and payload
    const res2 = await axios.post(`${BASE_URL}/charges`, payload, {
      headers: { 'Idempotency-Key': idempotencyKey },
    });

    // Verify same status and exact same charge ID without creating a new record
    expect(res2.status).to.equal(201);
    expect(res2.data.id).to.equal(chargeId);
    expect(res2.headers).to.have.property('idempotent-replayed', 'true');
  });

  it('should reject request when payload is mutated with an existing idempotency key', async () => {
    const idempotencyKey = generateKey();

    // 1. Send initial charge for $25.00
    await axios.post(
      `${BASE_URL}/charges`,
      { amount: 2500, currency: 'usd' },
      { headers: { 'Idempotency-Key': idempotencyKey } }
    );

    // 2. Resend with same key but mutated amount ($50.00)
    try {
      await axios.post(
        `${BASE_URL}/charges`,
        { amount: 5000, currency: 'usd' }, // Mutated!
        { headers: { 'Idempotency-Key': idempotencyKey } }
      );
      expect.fail('Server should have rejected mutated idempotency payload');
    } catch (err) {
      expect(err.response.status).to.be.oneOf([400, 422]);
      expect(err.response.data.error).to.equal('idempotency_key_mismatch');
    }
  });

  it('should safely handle 10 concurrent requests with the same key without double charging', async () => {
    const idempotencyKey = generateKey();
    const payload = { amount: 1000, currency: 'usd', customer: 'cus_race_user' };

    // Fire 10 simultaneous requests
    const promises = Array.from({ length: 10 }, () =>
      axios.post(`${BASE_URL}/charges`, payload, {
        headers: { 'Idempotency-Key': idempotencyKey },
        validateStatus: () => true, // capture all status codes
      })
    );

    const responses = await Promise.all(promises);

    // Filter successful responses
    const successfulResponses = responses.filter((r) => r.status === 201);
    const conflictResponses = responses.filter((r) => r.status === 409);

    // Exactly all success responses must refer to the ONE unique charge ID
    const uniqueChargeIds = new Set(successfulResponses.map((r) => r.data.id));
    expect(uniqueChargeIds.size).to.equal(1);

    // Total successful + 409 conflicts must account for all 10 requests
    expect(successfulResponses.length + conflictResponses.length).to.equal(10);
  });
});
```

---

## 4. QA Verification Checklist

- [ ] **Locking Mechanism**: Ensure the server implements distributed mutex locking (e.g., Redis `SET NX`) during in-flight processing.
- [ ] **Hash Verification**: Verify that the server computes and stores an SHA-256 hash of the request payload to detect payload mutations.
- [ ] **Header Caching**: Validate that security headers and content-type headers are faithfully restored upon replaying responses.
- [ ] **TTL Validation**: Confirm that keys expire after 24–48 hours, allowing subsequent reuse without unexpected side effects.
