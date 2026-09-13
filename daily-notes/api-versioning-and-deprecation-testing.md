# API Versioning and Deprecation Testing Strategies

## Introduction

In software systems, APIs are public contracts. Once an API is published and consumed by mobile applications, third-party partners, or internal frontend clients, making breaking changes without version control can immediately crash thousands of client applications.

Because mobile apps in the Apple App Store or Google Play Store cannot be forced to update simultaneously on every user's device, backend systems must support multiple API versions concurrently.

**API Versioning and Deprecation Testing** verifies that newer API versions introduce capabilities smoothly while older versions continue to function reliably until their announced retirement date.

---

## The Four Primary API Versioning Strategies

```
┌─────────────────────────────────────────────────────────────┐
│                   API Versioning Strategies                 │
├───────────────────────┬─────────────────────────────────────┤
│ Strategy              │ Example                             │
├───────────────────────┼─────────────────────────────────────┤
│ 1. URI Path           │ GET /api/v1/users                   │
│    (Most Popular ⭐)  │ GET /api/v2/users                   │
├───────────────────────┼─────────────────────────────────────┤
│ 2. Query Parameter    │ GET /api/users?version=2            │
├───────────────────────┼─────────────────────────────────────┤
│ 3. Custom Header      │ GET /api/users                      │
│                       │ Header: X-API-Version: 2            │
├───────────────────────┼─────────────────────────────────────┤
│ 4. Accept Header      │ GET /api/users                      │
│    (Content Neg.)     │ Accept: application/vnd.app.v2+json │
└───────────────────────┴─────────────────────────────────────┘
```

---

## Breaking vs. Non-Breaking Changes

QA engineers must evaluate API Pull Requests against strict backward-compatibility rules:

| Category | API Modification | Impact | Versioning Required? |
| :--- | :--- | :--- | :---: |
| **Non-Breaking** | Adding a new optional request parameter | Existing clients unaware of parameter work normally | No (Minor update) |
| **Non-Breaking** | Adding a new field to JSON response body | Tolerant JSON parsers ignore extra fields | No (Minor update) |
| **Breaking 🚨** | Renaming an existing field (`userId` ➔ `id`) | Existing mobile apps crash parsing `userId` | **YES (New Major Version)** |
| **Breaking 🚨** | Changing data type (string to integer) | Client type-casting exceptions occur | **YES (New Major Version)** |
| **Breaking 🚨** | Making an optional parameter required | Existing requests fail with HTTP 400 | **YES (New Major Version)** |
| **Breaking 🚨** | Removing an existing endpoint entirely | Clients receive unexpected HTTP 404 | **YES (New Major Version)** |

---

## Deprecation Protocols & RFC Standard Headers

When phasing out an older API version, standards mandate communicating deprecation schedules via HTTP response headers:

* `Deprecation: @1735689600` (or `true`): Informs clients that the endpoint is deprecated.
* `Sunset: Tue, 31 Dec 2026 23:59:59 GMT` (RFC 8594): The exact date and time when the endpoint will be permanently decommissioned and return `HTTP 410 Gone`.
* `Link: <https://api.example.com/docs/v2-migration>; rel="deprecation"`: Directs developers to migration instructions.

---

## Practical Test Automation: Verifying API Sunset in Postman

```javascript
// Postman Test Script: Asserting Sunset Header on Deprecated Endpoint
pm.test("Status code is 200 OK for deprecated v1 endpoint", function () {
    pm.response.to.have.status(200);
});

pm.test("Response includes RFC 8594 Sunset header with valid future date", function () {
    pm.response.to.have.header("Sunset");
    var sunsetDate = new Date(pm.response.headers.get("Sunset"));
    var now = new Date();
    
    // Sunset date must be in the future
    pm.expect(sunsetDate.getTime()).to.be.above(now.getTime());
});

pm.test("Response includes link to v2 migration guide", function () {
    var linkHeader = pm.response.headers.get("Link");
    pm.expect(linkHeader).to.include("rel=\"deprecation\"");
});
```

---

## SQA Interview Questions & Answers

### Q: Why is URI path versioning (`/api/v1/`) the most widely adopted versioning strategy?
**Answer:**
URI path versioning is explicit, transparent, and effortlessly testable. It allows testers and developers to inspect and bookmark endpoints directly in browser address bars, curl, or Postman without configuring custom headers. It also simplifies caching rules on CDNs and reverse proxies (NGINX/Cloudflare), which can route and cache distinct versions by path without inspecting request headers.

### Q: What HTTP status code should an API return when a deprecated endpoint is permanently removed?
**Answer:**
The server should return **HTTP 410 Gone** rather than a generic HTTP 404 Not Found. While HTTP 404 indicates that a resource is missing (which might be temporary or a typo), HTTP 410 explicitly signals that the requested resource once existed but has been permanently and deliberately removed, instructing client systems and caches to delete references to the endpoint.

---

## Key Takeaways

* APIs are contracts; breaking changes require major version increments.
* Non-breaking changes (new optional fields) should not increment major versions.
* Use RFC standard `Sunset` and `Deprecation` headers to communicate retirement schedules before permanently disabling endpoints.

---

## Conclusion

API versioning and deprecation governance ensure that software platforms evolve dynamically without stranding existing consumers. By testing parallel version coexistence, deprecation headers, and data compatibility, QA engineers safeguard the stability of multi-client ecosystems.
