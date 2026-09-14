# Network Chaos Testing & Resilience Guide with Toxiproxy

## Introduction

One of the most dangerous fallacies in software engineering is the assumption that:
> *"The network is reliable, latency is zero, and bandwidth is infinite."*

In real-world cloud architectures, networks experience transient packet loss, cross-region latency spikes, database connection resets, and third-party API degradation. If an application is only tested under ideal local network conditions, it will collapse in production when a downstream microservice or database slows down.

**Toxiproxy** is an open-source TCP proxy framework built by Shopify specifically for simulating network chaos and system failure conditions directly inside automated test suites.

---

## How Toxiproxy Works

Toxiproxy sits as an intermediary between your application under test and its downstream dependencies (databases, caches, third-party HTTP endpoints):

```
┌─────────────────────────────────────────────────────────────┐
│                 Application Under Test                      │
└──────────────────────────────┬──────────────────────────────┘
                               │ Normal TCP Connection
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                   Toxiproxy (Port: 22222)                   │
│   ┌──────────────────────────────────────────────────────┐  │
│   │ "Toxic" Injected: 3,000ms Latency + 20% Packet Drop  │  │
│   └──────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────┘
                               │ Impaired / Degraded Network
                               ▼
┌─────────────────────────────────────────────────────────────┐
│         Downstream Service (Redis / Postgres / API)         │
└─────────────────────────────────────────────────────────────┘
```

By controlling Toxiproxy's HTTP REST API, automated test scripts can dynamically inject, modify, and remove network faults ("toxics") during test execution.

---

## The Core "Toxics" Supported by Toxiproxy

```
┌─────────────────────────────────────────────────────────────┐
│                 Toxiproxy Network Impairments               │
├─────────────────────┬───────────────────────────────────────┤
│ Toxic Type          │ Simulated Failure Condition           │
├─────────────────────┼───────────────────────────────────────┤
│ 1. Latency & Jitter │ Injects fixed delays (e.g., 2000ms)   │
│                     │ with random jitter variance           │
├─────────────────────┼───────────────────────────────────────┤
│ 2. Bandwidth Limit  │ Throttles throughput (e.g., 50 KB/s)  │
├─────────────────────┼───────────────────────────────────────┤
│ 3. Slow Close       │ Delays closing TCP connections        │
├─────────────────────┼───────────────────────────────────────┤
│ 4. Timeout          │ Stops sending data and drops socket   │
│                     │ after specified duration              │
├─────────────────────┼───────────────────────────────────────┤
│ 5. Reset Peer       │ Injects TCP RST packet, terminating   │
│                     │ socket connection abruptly            │
├─────────────────────┼───────────────────────────────────────┤
│ 6. Slicer           │ Slices TCP packets into tiny fragments│
│                     │ to stress buffer reassembly logic     │
└─────────────────────┴───────────────────────────────────────┘
```

---

## Practical Test Automation with Toxiproxy (Node.js)

Here is a practical test verifying that an application's circuit breaker opens and falls back to a cached response when the Redis cache slows down:

```typescript
import { test, expect } from '@playwright/test';
import axios from 'axios';

// Toxiproxy management API URL
const TOXIPROXY_API = 'http://localhost:8474';

test.describe('Resilience Testing with Toxiproxy', () => {
  test('Application circuit breaker trips when database experiences 4000ms latency', async () => {
    // 1. Inject a Latency Toxic into the database proxy via Toxiproxy REST API
    await axios.post(`${TOXIPROXY_API}/proxies/postgres_proxy/toxics`, {
      name: 'high_db_latency',
      type: 'latency',
      stream: 'downstream',
      toxicity: 1.0, // 100% of packets affected
      attributes: {
        latency: 4000, // 4 seconds delay!
        jitter: 500,
      },
    });

    try {
      // 2. Fire application request
      const start = Date.now();
      const response = await axios.get('http://localhost:3000/api/v1/products/featured', {
        timeout: 5000,
      });
      const duration = Date.now() - start;

      // 3. Assertions:
      // Did the app timeout gracefully or invoke fallback rather than waiting 4 seconds?
      expect(response.status).toBe(200);
      expect(response.data.isFallback).toBe(true);
      expect(duration).toBeLessThan(1500); // Circuit breaker failed fast in < 1.5s!

    } finally {
      // 4. Teardown: Remove the toxic to restore normal network conditions
      await axios.delete(`${TOXIPROXY_API}/proxies/postgres_proxy/toxics/high_db_latency`);
    }
  });
});
```

---

## SQA Interview Questions & Answers

### Q: Why is Toxiproxy preferred over Linux `tc` (Traffic Control) or iptables for automated testing?
**Answer:**
Linux `tc` and `iptables` require root/sudo access, affect the entire virtual machine or container globally, and are difficult to control within unit or integration test code. Toxiproxy operates as an unprivileged user space TCP proxy that exposes a clean HTTP REST API, allowing automated test suites (Jest, Playwright, PyTest) to programmatically inject and remove network faults on a per-test basis without impacting other network traffic on the host.

### Q: What is the difference between a Connection Timeout and a Read Timeout?
**Answer:**
* **Connection Timeout**: The duration the application waits to establish the initial TCP handshake with the server. If the server is offline or unreachable, connection timeout occurs.
* **Read (Socket) Timeout**: The TCP connection was successfully established, but the application waited longer than its timeout threshold to receive the data packets from the server (e.g., due to an unindexed database query or slow third-party API).

---

## Key Takeaways

* Never assume networks are reliable; design test suites that actively inject latency and connection drops.
* Toxiproxy allows programmatic injection of network latency, bandwidth limits, and TCP resets via REST.
* Test circuit breakers, retry backoffs, and fallback caches under simulated network degradation.

---

## Conclusion

Resilience is a fundamental attribute of modern cloud architectures. By incorporating Toxiproxy into continuous integration test suites, QA engineers prove that applications degrade gracefully, fail fast, and heal automatically under severe real-world network turbulence.
