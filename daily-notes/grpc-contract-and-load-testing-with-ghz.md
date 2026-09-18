# gRPC Contract Validation & High-Throughput Load Testing with Ghz

## 1. Overview of gRPC & Protocol Buffers in Modern Architectures

As microservices architectures evolve beyond traditional REST APIs over HTTP/1.1, engineering organizations increasingly adopt **gRPC** (Google Remote Procedure Call) for internal service-to-service communication. Built on top of **HTTP/2** and utilizing **Protocol Buffers (Protobuf)** as its Interface Definition Language (IDL) and binary serialization format, gRPC delivers substantial performance, efficiency, and type-safety advantages.

```
┌───────────────────────────────┐                  ┌───────────────────────────────┐
│         gRPC Client           │                  │         gRPC Server           │
│  (Generated Client Stubs)     │                  │  (Service Implementation)     │
├───────────────────────────────┤                  ├───────────────────────────────┤
│ Protobuf Binary Serialization │                  │ Protobuf Binary Serialization │
├───────────────────────────────┤  HTTP/2 Frames   ├───────────────────────────────┤
│ HTTP/2 Transport (Multiplexed)│ ◄──────────────► │ HTTP/2 Transport (Multiplexed)│
│  - HPACK Header Compression   │  Single TCP Conn │  - Streaming (Server/Client)  │
│  - Stream Multiplexing        │                  │  - Flow Control & Pings       │
└───────────────────────────────┘                  └───────────────────────────────┘
```

### Key Differences: REST vs. gRPC for QA Engineers

| Architecture Dimension | REST (JSON over HTTP/1.1) | gRPC (Protobuf over HTTP/2) | QA Impact / Testing Consideration |
| :--- | :--- | :--- | :--- |
| **Data Serialization** | Text-based JSON (human-readable) | Compact binary format | Payloads cannot be read raw in Wireshark/cURL without schema reflection |
| **Transport Layer** | HTTP/1.1 (head-of-line blocking) | HTTP/2 multiplexed streams | Single TCP connection handles hundreds of parallel streams |
| **API Contract** | OpenAPI/Swagger (often optional) | Strict `.proto` definition | Breaking changes can be caught at compile and CI lint time |
| **Streaming Support** | SSE or WebSockets required | Native unary, client, server, bi-di | Complex async test assertions and long-lived stream testing |
| **Performance Overhead**| Higher CPU parsing overhead | Up to 7-10x faster serialization | Significantly higher load thresholds needed to stress services |

---

## 2. Protocol Buffer Contract & Schema Testing

In gRPC, the contract is governed by `.proto` files. Backward and forward compatibility are critical; any uncoordinated breaking change can cascade through dozens of microservices.

### Contract Rule Violations to Test

1. **Tag Number Mutation**: Changing the numerical field tag (e.g., changing `string user_id = 1;` to `string user_id = 2;`) breaks binary wire compatibility.
2. **Field Removal**: Removing a field without reserving its tag (`reserved 1;`) allows new fields to reuse the tag, causing silent deserialization corruption.
3. **Type Incompatibility**: Converting `int32` to `string` or altering enum values.

### Automated Schema Validation with `buf`

The modern standard for validating Protobuf contracts is the **Buf CLI**. QA automation pipelines incorporate Buf linting and breaking-change detection as early quality gates:

```bash
# Lint protobuf definitions for API design style rules
buf lint

# Compare current schema against the main branch / staging registry
buf breaking --against "https://github.com/organization/api-protos.git#branch=main"
```

Sample configuration (`buf.yaml`):

```yaml
version: v1
lint:
  use:
    - DEFAULT
  except:
    - PACKAGE_VERSION_SUFFIX
breaking:
  use:
    - FILE
    - WIRE_JSON
```

---

## 3. High-Throughput Load Testing with `ghz`

Standard HTTP load testing tools like ApacheBench or basic Locust scripts cannot natively handle HTTP/2 multiplexing, gRPC frame headers, and Protobuf binary serialization. **`ghz`** is the de-facto benchmarking and load testing CLI tool for gRPC services.

### Installation & Basic Execution

`ghz` supports test execution using raw `.proto` definitions or via **gRPC Server Reflection**:

```bash
ghz --insecure \
  --proto=./proto/user_service.proto \
  --call=user.UserService.GetUserProfile \
  --data='{"user_id": "usr_998124"}' \
  --metadata='{"authorization": "Bearer eyJhbGciOi..."}' \
  -c 50 \
  -n 10000 \
  -q 1000 \
  10.0.4.12:50051
```

### Key Parameters:
* `-c 50`: 50 concurrent worker connections.
* `-n 10000`: Total requests to dispatch across workers.
* `-q 1000`: Rate limiting at 1,000 QPS (Queries Per Second).
* `--call`: Fully-qualified `package.Service.Method`.
* `--metadata`: Custom gRPC metadata headers (used for authentication, tracing IDs, and tenant routing).

### Automated CI Performance Assertion via JSON Output

`ghz` can export detailed test summaries as JSON, enabling automated QA assertions on latency percentiles:

```bash
ghz --insecure \
  --proto=./proto/order_service.proto \
  --call=order.OrderService.CreateOrder \
  --data-file=./test-data/order_payload.json \
  -c 20 \
  -z 60s \
  --format=json \
  --output=./reports/ghz-results.json \
  localhost:50051
```

---

## 4. Node.js Automated gRPC Integration Test Suite

Below is an automated functional test script using `@grpc/grpc-js` and `@grpc/proto-loader` verifying error status codes, metadata trailers, and response payloads:

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');
const { expect } = require('chai');

const PROTO_PATH = path.resolve(__dirname, '../proto/payment_service.proto');
const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const protoDescriptor = grpc.loadPackageDefinition(packageDefinition);
const paymentService = protoDescriptor.payment.PaymentService;

describe('PaymentService gRPC Contract & Functional Verification', () => {
  let client;

  before(() => {
    client = new paymentService(
      'localhost:50051',
      grpc.credentials.createInsecure()
    );
  });

  after(() => {
    client.close();
  });

  it('should successfully process payment and return TRANSACTION_CONFIRMED', (done) => {
    const request = {
      account_id: 'acc_77182',
      amount_cents: 4500,
      currency: 'USD',
      idempotency_key: 'idem_4410298132',
    };

    const metadata = new grpc.Metadata();
    metadata.add('x-client-id', 'qa-automation-suite');

    client.ProcessPayment(request, metadata, (err, response) => {
      expect(err).to.be.null;
      expect(response).to.have.property('status', 'TRANSACTION_CONFIRMED');
      expect(response).to.have.property('transaction_id').that.is.a('string');
      expect(response.amount_cents).to.equal('4500');
      done();
    });
  });

  it('should reject requests with INVALID_ARGUMENT when currency is unsupported', (done) => {
    const invalidRequest = {
      account_id: 'acc_77182',
      amount_cents: 1000,
      currency: 'INVALID_CURRENCY',
      idempotency_key: 'idem_4410298133',
    };

    client.ProcessPayment(invalidRequest, (err, response) => {
      expect(err).to.not.be.null;
      expect(err.code).to.equal(grpc.status.INVALID_ARGUMENT);
      expect(err.details).to.include('Unsupported ISO currency code');
      done();
    });
  });
});
```

---

## 5. QA Verification Checklist for gRPC Services

- [ ] **Protobuf Backwards Compatibility**: Run `buf breaking` against base release tags in CI before merging PRs.
- [ ] **Error Code Standardization**: Ensure developers return standard canonical gRPC status codes (`NOT_FOUND`, `ALREADY_EXISTS`, `PERMISSION_DENIED`, `UNAUTHENTICATED`) rather than generic `INTERNAL` errors.
- [ ] **Metadata Propagation**: Verify that distributed tracing headers (`traceparent`, `x-request-id`) are correctly extracted and passed across downstream services.
- [ ] **Deadlines & Timeouts**: Test client cancellation when server-side processing exceeds `grpc-timeout` values.
- [ ] **Load Capacity & Concurrency**: Validate maximum concurrent HTTP/2 streams per connection (`MAX_CONCURRENT_STREAMS`) under stress with `ghz`.
