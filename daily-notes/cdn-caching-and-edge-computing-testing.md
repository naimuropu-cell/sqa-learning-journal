# CDN Caching & Edge Computing Testing Guide for QA

## Introduction

In modern web performance engineering, an origin web server located in Northern Virginia cannot deliver sub-100 millisecond response times to users accessing an application from Tokyo, London, or Sydney without encountering speed-of-light physical latency.

To eliminate latency, applications route traffic through a **Content Delivery Network (CDN)** and **Edge Computing platform** (such as Cloudflare, Fastly, AWS CloudFront, or Akamai). CDNs maintain hundreds of Points of Presence (PoPs) globally, caching static assets and executing lightweight serverless functions directly at the network edge closest to the user.

However, improper CDN configuration is a frequent source of catastrophic defects: caching sensitive customer data publicly, serving stale JavaScript bundles after a new release, or incurring massive origin database loads due to cache misses.

---

## How CDN Caching Operates

```
[ User in Tokyo ]
        │
        ▼ 1. Request GET /app.js
┌─────────────────────────────────────────────────────────────┐
│                 Tokyo CDN Edge Node (PoP)                   │
├─────────────────────────────────────────────────────────────┤
│ • Cache Check: Is /app.js in local SSD memory?              │
│   ├── YES (Cache HIT! ✅) ──► Returns file in 15ms!         │
│   └── NO  (Cache MISS! ⚠️) ──► Forwards to Origin Server    │
└──────────────────────────────┬──────────────────────────────┘
                               │ 2. Fetch from Origin (Slow ~250ms)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 Origin Server (Virginia, USA)               │
│                 Returns /app.js with Cache-Control headers  │
└─────────────────────────────────────────────────────────────┘
```

---

## The Core `Cache-Control` Directives QA Must Audit

The HTTP `Cache-Control` response header dictates how browsers and CDN edge nodes handle data:

```
┌─────────────────────────────────────────────────────────────┐
│                 Cache-Control Directive Matrix              │
├───────────────────────┬─────────────────────────────────────┤
│ Directive             │ Behavior & Meaning                  │
├───────────────────────┼─────────────────────────────────────┤
│ 1. public             │ Can be cached by browsers AND public│
│                       │ CDN edge servers                    │
├───────────────────────┼─────────────────────────────────────┤
│ 2. private            │ Can ONLY be cached by the user's    │
│                       │ private browser; CDN MUST NOT cache!│
├───────────────────────┼─────────────────────────────────────┤
│ 3. no-store           │ DO NOT CACHE ANYWHERE! (Mandatory   │
│                       │ for banking, medical, and PII data) │
├───────────────────────┼─────────────────────────────────────┤
│ 4. max-age=N          │ Time-to-live (seconds) in browser   │
├───────────────────────┼─────────────────────────────────────┤
│ 5. s-maxage=N         │ Time-to-live (seconds) on CDN edge  │
│                       │ (Overrides max-age for CDNs)        │
├───────────────────────┼─────────────────────────────────────┤
│ 6. stale-while-       │ Serves cached version instantly     │
│    revalidate=N       │ while fetching fresh asset in backgr│
└───────────────────────┴─────────────────────────────────────┘
```

---

## Critical QA Test Scenarios

### 1. Static Asset Cache Verification
* **Test**: Request a CSS bundle or image twice via curl:
  ```bash
  curl -I https://example.com/assets/main.css
  ```
* **Assertion**: On the second request, inspect CDN response headers:
  * Cloudflare: `cf-cache-status: HIT`
  * AWS CloudFront: `x-cache: Hit from cloudfront`
  * Fastly: `x-cache: HIT`

### 2. Cache Poisoning & Data Leakage Prevention (Critical Security Audit!)
* **The Disaster**: An API endpoint `/api/v1/user/account-balance` is accidentally configured with `Cache-Control: public, s-maxage=3600`.
* **The Consequence**: User A logs in and checks their balance. The CDN caches the response. User B logs in 5 seconds later and is served User A's cached balance and profile!
* **QA Test**: Verify that **ALL authenticated endpoints** strictly return:
  `Cache-Control: no-store, no-cache, private`

### 3. Cache Purging / Invalidation on Deployments
* **The Scenario**: Developers deploy a critical CSS/JS hotfix.
* **QA Test**: Deploy the build, trigger an automated CDN Cache Purge in CI/CD, and assert that requesting the root HTML page immediately loads the new asset hashes rather than serving stale, cached stylesheets.

---

## SQA Interview Questions & Answers

### Q: What is the difference between `no-cache` and `no-store`?
**Answer:**
* **`no-store`**: Complete caching prohibition. The response must never be written to temporary disk or memory caches anywhere (browser or CDN). Every request must fetch the full resource from the origin server.
* **`no-cache`**: The response *can* be stored in cache, but the browser or CDN **must revalidate** with the origin server before serving it (using conditional requests like `If-None-Match` with `ETag`). If unchanged, the server returns `304 Not Modified` with zero body transfer.

### Q: What is Cache Invalidation and why is it challenging?
**Answer:**
Cache invalidation is the process of purging or evicting cached objects from global CDN edge servers before their configured TTL expires when new code or content is published. It is challenging in distributed systems because purging must propagate across hundreds of globally distributed edge data centers simultaneously; incomplete invalidations result in users in Tokyo seeing new code while users in London see broken, cached legacy assets.

---

## Key Takeaways

* CDNs reduce physical latency by caching static assets at global Edge PoPs.
* Never allow authenticated endpoints or PII to return `Cache-Control: public`.
* Verify that CI/CD deployment pipelines automatically trigger CDN cache purges to prevent stale asset bugs.

---

## Conclusion

CDN and Edge computing testing ensures that software achieves global speed without sacrificing data security or content freshness. By auditing Cache-Control headers, verifying cache hit ratios, and testing cache purge automation, QA engineers guarantee exceptional web performance worldwide.
