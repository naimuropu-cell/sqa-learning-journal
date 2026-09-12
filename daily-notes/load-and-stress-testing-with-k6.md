# Modern Load and Stress Testing with k6

## Introduction

An application may function flawlessly with one user during manual testing or automated functional suites, but crumble when thousands of users browse products, add items to cart, or process payments simultaneously.

Performance testing ensures an application's speed, scalability, stability, and responsiveness under simulated concurrent user traffic. While Apache JMeter has historically dominated performance testing, **Grafana k6** has emerged as the modern standard because it treats performance tests as code (JavaScript/TypeScript), integrates natively into Git, and is built for developer and QA CI/CD pipelines.

---

## Types of Performance Tests

```
   Traffic
      ▲
      │                 ┌───────────────┐
      │                 │ Spike Testing │
      │                 └───────┬───────┘
      │                         │  /\
      │                         │ /  \
      │          ┌──────────────┼/────\────────┐
      │          │ Stress Test  │      \       │
      │          └──────┬───────┘       \      │
      │                 │                \     │
      │    ┌────────────┼──────────┐      \    │
      │    │ Load Test  │          │       \   │
      │    └────────────┘          └───────────┘
      └──────────────────────────────────────────────► Time
```

1. **Smoke Test**: Minimal load (1-2 VUs) to verify that performance test scripts work without errors.
2. **Load Test**: Assesses system behavior under anticipated standard and peak production traffic.
3. **Stress Test**: Increases load beyond normal capacity to determine the system's breaking point and verify graceful degradation.
4. **Spike Test**: Injects sudden, extreme bursts of traffic to observe how quickly the system handles and recovers from surges.
5. **Soak / Endurance Test**: Sustains moderate load over prolonged periods (hours/days) to identify memory leaks and database connection pool exhaustion.

---

## Core Performance Metrics QA Must Track

* **Virtual Users (VUs)**: Simulated concurrent users executing test iterations in parallel.
* **Throughput (Requests Per Second - RPS)**: Total number of requests processed by the server each second.
* **Response Time Percentiles**:
  * **Average**: Often misleading because a few fast requests can mask major lag for many users.
  * **p95 (95th Percentile)**: 95% of users experienced response times at or below this value.
  * **p99 (99th Percentile)**: 99% of requests completed within this threshold (identifies worst-case tail latencies).
* **Error Rate**: Percentage of requests returning HTTP 4xx, 5xx, or network timeouts.

---

## Practical k6 Load Testing Script

Here is an end-to-end k6 performance script (`tests/load-test.js`) testing an e-commerce catalog API with realistic ramp-up stages and strict performance thresholds:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

// Test configuration: Stages and Quality Gate Thresholds
export const options = {
  stages: [
    { duration: '30s', target: 20 },  // Ramp-up to 20 users over 30 seconds
    { duration: '1m',  target: 50 },  // Ramp-up to 50 users (Peak Load)
    { duration: '30s', target: 0 },   // Ramp-down to 0 users
  ],
  thresholds: {
    // 95% of requests must finish within 300ms
    http_req_duration: ['p(95)<300'],
    // Error rate must remain below 1%
    http_req_failed: ['rate<0.01'],
  },
};

const BASE_URL = 'https://dummyjson.com';

export default function () {
  // 1. Fetch Product Listing
  const listRes = http.get(`${BASE_URL}/products?limit=10`);
  check(listRes, {
    'status is 200': (r) => r.status === 200,
    'response has products': (r) => JSON.parse(r.body).products.length > 0,
  });

  // Simulate user think time (reading product details)
  sleep(1);

  // 2. Search Specific Product
  const searchRes = http.get(`${BASE_URL}/products/search?q=phone`);
  check(searchRes, {
    'search status is 200': (r) => r.status === 200,
    'search duration < 250ms': (r) => r.timings.duration < 250,
  });

  sleep(2);
}
```

---

## Running and Interpreting k6 Output

To execute the test:
```bash
k6 run tests/load-test.js
```

### Example Terminal Summary:
```text
✓ status is 200
✓ response has products
✓ search status is 200
✓ search duration < 250ms

checks.........................: 100.00% ✓ 4800   ✗ 0
http_req_duration..............: avg=112ms min=45ms med=98ms max=412ms p(90)=180ms p(95)=220ms
http_req_failed................: 0.00%   ✓ 0      ✗ 4800
vus............................: 1       min=1    max=50
```

Because `p(95)=220ms` is under the 300ms threshold and failures are 0%, k6 exits with code `0`, confirming the release passes the performance quality gate.

---

## Best Practices for Performance Testing

* **Isolate Test Environments**: Never run heavy stress tests against shared staging environments where developers are actively testing or deploying.
* **Include Realistic Think Time**: Real users do not click buttons every millisecond. Use `sleep()` to model realistic human behavior.
* **Clean Up Test Artifacts**: When load testing creates database records (e.g., placing test orders), ensure cleanup routines run to prevent database bloat.
* **Monitor Infrastructure Alongside Application**: Observe CPU utilization, memory consumption, and database connection pools during load runs.

---

## SQA Interview Questions & Answers

### Q: Why is p95 or p99 preferred over Average Response Time?
**Answer:**
Average response time hides outliers. If 90 requests take 50ms and 10 requests take 10 seconds, the average might appear acceptable (around 1 second), even though 10% of customers experienced intolerable slowdowns. Percentiles like p95 and p99 accurately expose tail latencies affecting real users.

### Q: What is the difference between Load Testing and Stress Testing?
**Answer:**
* **Load Testing** measures performance under expected normal and peak user loads to verify that Service Level Objectives (SLOs) are satisfied.
* **Stress Testing** pushes the system beyond maximum anticipated capacity until it breaks, identifying hardware limits, bottlenecks, and whether the system recovers gracefully once traffic normalizes.

---

## Key Takeaways

* Performance testing safeguards business revenue during high-traffic events (e.g., Black Friday, flash sales).
* Modern tools like k6 enable version-controlled performance tests written in JavaScript and integrated into CI/CD.
* Rely on percentiles (p95, p99) and error rates to establish meaningful pass/fail quality gates.

---

## Conclusion

Load and stress testing are vital components of modern SQA engineering. By incorporating k6 into automated test pipelines, QA teams can catch performance regressions before they impact end-users in production.
