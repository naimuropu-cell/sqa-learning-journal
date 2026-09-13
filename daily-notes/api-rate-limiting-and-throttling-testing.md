# API Rate Limiting and Throttling Testing Guide

## Introduction

In modern public and microservice APIs, resources are finite. A single misconfigured client or malicious actor could fire millions of automated requests per second, overwhelming backend servers, crashing databases, or driving up cloud bills.

**Rate Limiting and Throttling** are traffic control mechanisms designed to protect APIs from Denial-of-Service (DoS) attacks, brute-force credential stuffing, and the "noisy neighbor" problem by restricting the number of requests a client can make within a specified timeframe.

Testing rate limits is a crucial QA responsibility to ensure that abusive traffic is blocked while legitimate customers receive smooth, uninterrupted service.

---

## Rate Limiting Algorithms Overview

```
┌─────────────────────────────────────────────────────────────┐
│                   Rate Limiting Algorithms                  │
├───────────────────────┬─────────────────────────────────────┤
│ Algorithm             │ Behavior & Characteristics          │
├───────────────────────┼─────────────────────────────────────┤
│ 1. Token Bucket       │ Tokens accumulate at steady rate.   │
│                       │ Allows brief traffic bursts.        │
│ 2. Leaky Bucket       │ Requests process at a fixed constant│
│                       │ rate, smoothing sudden spikes.      │
│ 3. Fixed Window       │ Resets count at fixed intervals     │
│                       │ (e.g., 100 req/min). Prone to burst │
│                       │ at window boundary edges.           │
│ 4. Sliding Window     │ Computes request rate dynamically   │
│                       │ across rolling time windows.        │
└───────────────────────┴─────────────────────────────────────┘
```

---

## Standard Rate Limiting HTTP Headers

Well-designed APIs communicate quota status via standard HTTP headers:

* `RateLimit-Limit` (or `X-RateLimit-Limit`): Maximum allowed requests in the current window (e.g., `100`).
* `RateLimit-Remaining` (or `X-RateLimit-Remaining`): Remaining allowed calls before throttling occurs (e.g., `42`).
* `RateLimit-Reset` (or `X-RateLimit-Reset`): Unix epoch timestamp when the window resets.
* `Retry-After`: Returned with **HTTP 429 Too Many Requests**, indicating the number of seconds the client must wait before retrying.

```
Request 99  ──► HTTP 200 OK  (X-RateLimit-Remaining: 1)
Request 100 ──► HTTP 200 OK  (X-RateLimit-Remaining: 0)
Request 101 ──► HTTP 429 Too Many Requests
                Headers:
                Retry-After: 30
                Body:
                {
                  "error": "RATE_LIMIT_EXCEEDED",
                  "message": "Quota exceeded. Try again in 30 seconds."
                }
```

---

## Core QA Test Scenarios for Rate Limiting

### 1. The Exact Boundary Transition ($N$ vs. $N+1$)
* Send requests up to the allowed limit ($N$). Verify each returns `HTTP 200 OK` and `X-RateLimit-Remaining` decrements accurately.
* Send request $N+1$. Verify immediate response with **HTTP 429 Too Many Requests**.

### 2. Window Reset & Recovery
* After receiving an HTTP 429, wait for the number of seconds specified in `Retry-After`.
* Send a new request. Verify the API allows the call (`HTTP 200 OK`) and resets `X-RateLimit-Remaining` to its maximum capacity.

### 3. Tier-Based Rate Limits (Free vs. Premium)
* Validate that free tier API keys are capped at their tier limit (e.g., 60 req/min) while premium API keys successfully process higher thresholds (e.g., 1,000 req/min).

### 4. IP-Based vs. User-Based Limiting
* Verify that rate limiting applied to an IP address does not penalize distinct users behind a corporate NAT or shared office Wi-Fi if the application supports user-based token limiting.

---

## Automated Rate Limit Testing Script (Node.js / Axios)

```javascript
const axios = require('axios');

async function testRateLimit() {
  const ENDPOINT = 'https://api.example.com/v1/data';
  const API_KEY = 'test_free_tier_key';
  const MAX_ALLOWED = 5;

  console.log(`Firing ${MAX_ALLOWED + 1} rapid requests to test rate limiting...`);

  for (let i = 1; i <= MAX_ALLOWED + 1; i++) {
    try {
      const res = await axios.get(ENDPOINT, {
        headers: { 'Authorization': `Bearer ${API_KEY}` },
      });
      console.log(`Request #${i}: Status ${res.status} | Remaining: ${res.headers['x-ratelimit-remaining']}`);
    } catch (err) {
      if (err.response) {
        console.log(`Request #${i}: Caught Expected Status ${err.response.status}`);
        console.log(`Retry-After Header: ${err.response.headers['retry-after']} seconds`);
        
        // Assertions
        if (i === MAX_ALLOWED + 1) {
          if (err.response.status === 429) {
            console.log('✅ PASS: Rate limit successfully triggered at boundary N+1.');
          } else {
            console.error(`❌ FAIL: Expected 429, received ${err.response.status}`);
          }
        }
      }
    }
  }
}

testRateLimit();
```

---

## SQA Interview Questions & Answers

### Q: What HTTP status code should an API return when a rate limit is exceeded?
**Answer:**
The API must return **HTTP 429 Too Many Requests**. Returning HTTP 400 or HTTP 500 is incorrect because the request syntax was valid and the server did not experience an internal crash; rather, the client sent too many requests within a given time window.

### Q: What is the difference between Rate Limiting and Throttling?
**Answer:**
* **Rate Limiting** is an absolute constraint on the total number of requests permitted within a specific time window (e.g., 100 requests per minute). Requests exceeding the limit are rejected with HTTP 429.
* **Throttling** is the regulation of request processing speed or bandwidth (e.g., delaying incoming requests, degrading response fidelity, or queueing calls) to prevent infrastructure overload without outright dropping requests.

---

## Key Takeaways

* Rate limiting protects backend systems from DDoS attacks, brute-force attempts, and runaway client scripts.
* Validate boundaries ($N$ vs. $N+1$), `HTTP 429` status codes, and `Retry-After` recovery.
* Verify quota isolation across IP addresses, user accounts, and pricing subscription tiers.

---

## Conclusion

Rate limiting and throttling are vital for API availability, security, and fair resource sharing. By methodically verifying quota boundaries, error payloads, and retry intervals, QA engineers safeguard systems against traffic surges and malicious abuse.
