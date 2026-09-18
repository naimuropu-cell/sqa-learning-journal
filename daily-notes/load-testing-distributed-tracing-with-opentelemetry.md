# Correlating Distributed Tracing with Load Testing Using OpenTelemetry & Jaeger

## 1. Why Aggregate Load Metrics Are Insufficient

When executing load tests using tools like **k6**, **Gatling**, or **JMeter**, performance reports provide macro-level metrics:
- Overall RPS (Requests Per Second)
- Error rate percentages
- Aggregated latency percentiles (p50, p90, p99)

While a report showing *`p99 = 2,450ms`* proves that latency degraded under stress, it **cannot answer why**:
- Which specific microservice in the downstream dependency chain stalled?
- Was latency caused by a slow database query, Redis connection pool starvation, an unindexed table scan, or third-party API throttling?

By integrating **OpenTelemetry (OTel)** distributed tracing into load tests, QA engineers can inspect the exact execution span graph for degraded requests.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Distributed Trace Span Tree                     │
│                                                                        │
│ [POST /api/checkout] (Total Duration: 2,450ms)                         │
│ ├── [AuthService: ValidateToken] ─────────────► (15ms)                 │
│ ├── [InventoryService: ReserveStock] ─────────► (35ms)                 │
│ └── [PaymentService: AuthorizePayment] ───────► (2,400ms) ⚠️ BOTTLENECK│
│     ├── [Redis: CheckIdempotencyKey] ─────────► (2ms)                  │
│     └── [Postgres: UPDATE accounts FOR UPDATE]► (2,398ms) 💥 ROW LOCK  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Injecting W3C Trace Context into Load Tests

OpenTelemetry uses the standard **W3C Trace Context** specification consisting of the `traceparent` HTTP header:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
              │  └───────────────┬──────────────┘ └───────┬────────┘ └─ Sampled Flag
              │               Trace ID                 Parent Span ID
            Version
```

### Passing Trace Context in a k6 Load Script:

```javascript
import http from 'k6/http';
import { check } from 'k6';
import crypto from 'k6/crypto';

export const options = {
  vus: 50,
  duration: '1m',
  thresholds: {
    http_req_duration: ['p(95)<500'],
  },
};

// Generate W3C compliant traceparent
function generateTraceparent() {
  const version = '00';
  const traceId = crypto.hex(crypto.randomBytes(16));
  const spanId = crypto.hex(crypto.randomBytes(8));
  const flags = '01'; // Sampled: Force recording in Jaeger/Tempo
  return `${version}-${traceId}-${spanId}-${flags}`;
}

export default function () {
  const traceparent = generateTraceparent();

  const params = {
    headers: {
      'Content-Type': 'application/json',
      'traceparent': traceparent,
      'x-test-run-id': 'load_test_phase_3',
    },
  };

  const payload = JSON.stringify({
    user_id: 'usr_load_test',
    cart_id: 'cart_99182',
  });

  const res = http.post('https://api.staging.example.com/api/checkout', payload, params);

  check(res, {
    'status is 200': (r) => r.status === 200,
  });

  // Log Trace ID for degraded requests (p99 outliers)
  if (res.timings.duration > 2000) {
    console.warn(`Slow Request Detected! Duration: ${res.timings.duration}ms | Trace: ${traceparent}`);
  }
}
```

---

## 3. Investigating Traces in Jaeger / Grafana Tempo

When the test runner flags a trace ID (e.g., `4bf92f3577b34da6a3ce929d0e0e4736`), the QA engineer directly queries **Jaeger UI**:

1. Paste the Trace ID into the search field.
2. Expand the flame graph hierarchy.
3. Identify the longest horizontal span.
4. Inspect the span tags and database attributes:
   - `db.statement`: `SELECT * FROM orders WHERE status = 'PENDING' FOR UPDATE`
   - `db.system`: `postgresql`
   - `error`: `lock_wait_timeout`

---

## 4. Automated Trace Assertion with OpenTelemetry Collector

QA pipelines can assert distributed trace quality by configuring OTel trace validation assertions:

```javascript
const axios = require('axios');
const { expect } = require('chai');

describe('Distributed Trace Propagation Quality Gate', () => {
  it('should propagate traceparent across all 4 microservice hops', async () => {
    const traceId = 'a1b2c3d4e5f60718293a4b5c6d7e8f90';
    const traceparent = `00-${traceId}-0000000000000001-01`;

    await axios.post('http://gateway.local/api/orders', { item: 'book' }, {
      headers: { traceparent: traceparent },
    });

    // Wait for OpenTelemetry collector ingestion
    await new Promise((res) => setTimeout(res, 2000));

    // Query Jaeger API
    const jaegerRes = await axios.get(`http://jaeger:16686/api/traces/${traceId}`);
    const spans = jaegerRes.data.data[0].spans;

    // Verify all microservices participated in the trace
    const participatingServices = new Set(spans.map((s) => s.process.serviceName));
    expect(participatingServices).to.include('api-gateway');
    expect(participatingServices).to.include('order-service');
    expect(participatingServices).to.include('payment-service');
    expect(participatingServices).to.include('notification-service');
  });
});
```

---

## 5. QA Verification Checklist

- [ ] **Sampling Configuration**: Set trace sampling rate to 100% (`sampled=01`) during load testing to capture all edge cases.
- [ ] **Context Propagation**: Ensure asynchronous background workers (Kafka consumers, BullMQ jobs) extract and re-inject trace context into downstream spans.
- [ ] **Database Span Instrumentation**: Verify SQL statements are logged without exposing sensitive PII (passwords, credit card numbers).
- [ ] **Root-Cause Attribution**: Cross-reference load test p99 latency spikes directly with corresponding Jaeger traces.
