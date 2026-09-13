# API Gateway and Reverse Proxy Testing Guide

## Introduction

In distributed microservices architectures, client applications (web, mobile, IoT) rarely communicate directly with backend microservices. Instead, all external network traffic enters through an **API Gateway or Reverse Proxy** (such as Kong, NGINX, AWS API Gateway, Traefik, or Envoy).

The API Gateway acts as the single point of entry, providing centralized traffic routing, SSL termination, authentication offloading, rate limiting, and CORS enforcement.

Testing the API Gateway layer ensures that requests are properly routed to upstream services, security policies are uniformly enforced, and client requests do not leak unauthorized headers.

---

## Core Architecture & Gateway Responsibilities

```
┌─────────────────────────────────────────────────────────────┐
│                 Client (Web / Mobile / CLI)                 │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTPS (Public Internet)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                  API Gateway / Reverse Proxy                │
│  • SSL/TLS Termination       • Auth Token Verification      │
│  • CORS Header Injection     • Rate Limiting & Throttling   │
│  • Request Routing (Path)    • Header Stripping / Enriched  │
└───────┬──────────────────────┴──────────────────────┬───────┘
        │                                             │
        ▼ (Private Network)                           ▼ (Private Network)
┌─────────────────────────────┐             ┌─────────────────────────────┐
│   User Service (Microservice│             │  Order Service (Microservice│
│   Port: 8081                │             │  Port: 8082                 │
└─────────────────────────────┘             └─────────────────────────────┘
```

---

## Key Testing Areas for API Gateways

### 1. Path-Based and Host-Based Routing Rules
* **QA Test**: Verify that requests sent to `/api/v1/users` route exclusively to the User Service, while `/api/v1/orders` routes to the Order Service.
* **Negative Test**: Send requests to unmapped endpoints (e.g., `/api/v1/invalid-route`). Verify the Gateway returns **HTTP 404 Not Found** without leaking internal server details.

### 2. CORS (Cross-Origin Resource Sharing) Policy Validation
When web applications call APIs hosted on different domains, the browser fires an HTTP `OPTIONS` preflight request:
* **Preflight Headers to Assert**:
  * `Access-Control-Allow-Origin`: Must strictly match permitted white-listed domains (never blindly `*` if credentials/cookies are accepted).
  * `Access-Control-Allow-Methods`: `GET, POST, PUT, DELETE, OPTIONS`.
  * `Access-Control-Allow-Headers`: `Authorization, Content-Type`.
  * `Access-Control-Allow-Credentials`: `true`.

### 3. Header Stripping and Injection
* **Header Stripping**: A malicious external client might inject `X-User-Role: admin` in their request. The Gateway must strip this untrusted header before forwarding the request upstream.
* **Header Injection**: After validating a JWT, the Gateway must inject trusted internal headers (e.g., `X-Internal-User-Id: 4021`) to inform downstream microservices of the user's identity.

### 4. Upstream Timeout & Circuit Breaking
* If an upstream microservice hangs or crashes:
  * Verify the Gateway returns **HTTP 504 Gateway Timeout** within the configured threshold (e.g., 5 seconds) rather than keeping the client waiting indefinitely.
  * Verify the Gateway fails over to secondary replicas if available.

---

## Practical Test Script: CORS & Gateway Route Verification

Here is an automated test verifying Gateway routing and CORS policies using Postman / JavaScript:

```javascript
// Test 1: Verify Preflight CORS Options Request
pm.test("CORS Preflight returns 204 or 200 with allowed origins", function () {
    pm.expect([200, 204]).to.include(pm.response.code);
    
    // Assert Allowed Origin Header
    pm.expect(pm.response.headers.get("Access-Control-Allow-Origin")).to.eql("https://app.example.com");
    
    // Assert Allowed Methods
    var methods = pm.response.headers.get("Access-Control-Allow-Methods");
    pm.expect(methods).to.include("GET");
    pm.expect(methods).to.include("POST");
});

// Test 2: Verify Gateway does not leak internal server signatures
pm.test("Server header does not expose internal microservice framework", function () {
    var serverHeader = pm.response.headers.get("Server");
    // Ensure sensitive versions are omitted (e.g., Kestrel, Express version)
    pm.expect(serverHeader).to.not.contain("Express");
    pm.expect(serverHeader).to.not.contain("Kestrel");
});
```

---

## SQA Interview Questions & Answers

### Q: Why is an API Gateway crucial in microservice architectures?
**Answer:**
Without an API Gateway, client applications would need to track dozens of individual microservice URLs and ports, negotiate CORS on every service independently, and implement redundant authentication logic across all services. The Gateway provides a unified entry point, offloads authentication and rate limiting, terminates SSL, and shields the internal service topology from the public internet.

### Q: What is the risk of setting `Access-Control-Allow-Origin: *`?
**Answer:**
Setting the allowed origin to wildcard `*` allows any third-party malicious website to execute cross-origin requests to your API. If your API relies on session cookies or standard credentials, browsers will reject the request if credentials mode is enabled; however, for token-based APIs, wildcards can expose sensitive internal endpoints to malicious cross-site scripting attacks.

---

## Key Takeaways

* The API Gateway is the frontline guard for microservices, handling routing, SSL, auth, and CORS.
* Test CORS preflight requests and assert that allowed origins are restricted to trusted domains.
* Verify upstream timeouts (HTTP 504) and ensure untrusted incoming headers are stripped.

---

## Conclusion

Testing API Gateways and reverse proxies ensures that the front door of your software architecture remains secure, performant, and resilient. By validating routing policies, CORS headers, and timeout behaviors, QA engineers safeguard downstream microservices from external turbulence.
