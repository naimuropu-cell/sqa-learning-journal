# Mocking gRPC Services for Isolated Integration Testing with GripMock

## 1. The Need for gRPC Service Virtualization in QA

When executing automated integration tests against microservices architectures, dependent gRPC services are often:
- **Unstable or in active development** (leading to flaky upstream test failures).
- **Expensive or rate-limited** (e.g., third-party fraud detection, credit bureaus, or payment orchestrators).
- **Difficult to manipulate** into returning edge cases (e.g., specific gRPC error codes like `RESOURCE_EXHAUSTED`, `DEADLINE_EXCEEDED`, or corrupted metadata).

Traditional HTTP mock servers (like standard WireMock or MockServer) do not natively understand Protocol Buffers (Protobuf) or HTTP/2 framing. **GripMock** is a lightweight, mock gRPC server that uses `.proto` schema definitions to parse and respond to incoming gRPC requests with configurable JSON stubs.

```
┌────────────────────────────────────────────────────────┐
│             Test Runner / Target Service               │
│               (gRPC Client Under Test)                 │
└──────────────────────────┬─────────────────────────────┘
                           │
                 gRPC Call (HTTP/2 + Proto)
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│                   GripMock Server                      │
│  - Port 50051: gRPC Server (Serving Stubs)             │
│  - Port 4770:  HTTP Admin API (Setting Dynamic Stubs)  │
└──────────────────────────┬─────────────────────────────┘
                           │
                 Configured via JSON
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│                Dynamic In-Memory Stubs                 │
│  Match Request JSON ──► Return Predetermined Response  │
└────────────────────────────────────────────────────────┘
```

---

## 2. Setting Up GripMock with Docker

GripMock runs seamlessly via Docker by mounting the directory containing `.proto` files:

```bash
docker run -d --name gripmock-server \
  -p 4770:4770 \
  -p 50051:50051 \
  -v $(pwd)/proto:/proto \
  tkpd/gripmock /proto/order_service.proto /proto/payment_service.proto
```

* **Port 50051**: The gRPC mock endpoint your service connects to instead of the real backend.
* **Port 4770**: The REST management API used by QA automation scripts to add, inspect, and clear stubs on the fly.

---

## 3. Dynamic Stubbing via Admin HTTP API

During test execution, QA automation scripts configure GripMock's behavior via HTTP POST requests to `http://localhost:4770/add`:

### Example 1: Mocking a Successful Response
```json
{
  "service": "PaymentService",
  "method": "ProcessPayment",
  "input": {
    "equals": {
      "account_id": "acc_qa_99182",
      "amount_cents": 2500
    }
  },
  "output": {
    "data": {
      "transaction_id": "tx_mock_992813",
      "status": "TRANSACTION_CONFIRMED"
    }
  }
}
```

### Example 2: Simulating gRPC Server Failures & Error Codes
Testing how the client handles upstream timeouts and server errors:
```json
{
  "service": "PaymentService",
  "method": "ProcessPayment",
  "input": {
    "contains": {
      "account_id": "acc_fail_user"
    }
  },
  "output": {
    "error": "Payment gateway processing timeout",
    "code": 4
  }
}
```
*(Code `4` corresponds to canonical gRPC status code `DEADLINE_EXCEEDED`)*.

---

## 4. Automated Integration Test Script (Node.js)

Below is an automated test suite that dynamically sets up GripMock stubs before each test and verifies client behavior:

```javascript
const axios = require('axios');
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const { expect } = require('chai');

const GRIPMOCK_ADMIN_URL = 'http://localhost:4770';
const GRPC_MOCK_TARGET = 'localhost:50051';

describe('Order Processing with GripMock Dependency Suite', () => {
  let paymentClient;

  before(async () => {
    // Clear all existing stubs in GripMock
    await axios.get(`${GRIPMOCK_ADMIN_URL}/clear`);

    // Initialize gRPC Client pointing to GripMock
    const packageDef = protoLoader.loadSync('./proto/payment_service.proto');
    const proto = grpc.loadPackageDefinition(packageDef);
    paymentClient = new proto.payment.PaymentService(
      GRPC_MOCK_TARGET,
      grpc.credentials.createInsecure()
    );
  });

  it('should process payment when GripMock returns confirmed status', async () => {
    // 1. Arrange: Register stub in GripMock
    await axios.post(`${GRIPMOCK_ADMIN_URL}/add`, {
      service: 'PaymentService',
      method: 'ProcessPayment',
      input: { equals: { account_id: 'acc_test_1' } },
      output: { data: { transaction_id: 'tx_stub_123', status: 'SUCCESS' } },
    });

    // 2. Act: Invoke client
    const response = await new Promise((resolve, reject) => {
      paymentClient.ProcessPayment({ account_id: 'acc_test_1' }, (err, res) => {
        if (err) return reject(err);
        resolve(res);
      });
    });

    // 3. Assert: Validate response
    expect(response.status).to.equal('SUCCESS');
    expect(response.transaction_id).to.equal('tx_stub_123');
  });

  it('should gracefully handle upstream DEADLINE_EXCEEDED errors', async () => {
    // Arrange: Stub gRPC error
    await axios.post(`${GRIPMOCK_ADMIN_URL}/add`, {
      service: 'PaymentService',
      method: 'ProcessPayment',
      input: { equals: { account_id: 'acc_timeout_1' } },
      output: { error: 'Service Unavailable', code: 4 }, // DEADLINE_EXCEEDED
    });

    // Act & Assert
    try {
      await new Promise((resolve, reject) => {
        paymentClient.ProcessPayment({ account_id: 'acc_timeout_1' }, (err, res) => {
          if (err) return reject(err);
          resolve(res);
        });
      });
      expect.fail('Expected gRPC call to throw error');
    } catch (err) {
      expect(err.code).to.equal(grpc.status.DEADLINE_EXCEEDED);
    }
  });
});
```

---

## 5. QA Verification Checklist

- [ ] **Proto Schema Parity**: Ensure GripMock uses identical `.proto` definitions as the production service registry.
- [ ] **Stub Isolation**: Clear all registered stubs (`/clear`) before and after each test suite to avoid state bleed.
- [ ] **Error Code Coverage**: Systematically test client-side resilience against all gRPC status codes (`UNAVAILABLE`, `DEADLINE_EXCEEDED`, `RESOURCE_EXHAUSTED`).
- [ ] **Metadata Validation**: Verify that stubs correctly evaluate and return custom gRPC headers and trailers.
