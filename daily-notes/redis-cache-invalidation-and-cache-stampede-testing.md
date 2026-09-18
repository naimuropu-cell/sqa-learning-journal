# Redis Cache Invalidation, Penetration & Cache Stampede Testing

## 1. The Critical Role of Caching in Scalable Architectures

In high-throughput web applications, **Redis** is deployed in front of relational databases (PostgreSQL/MySQL) to serve frequent queries with sub-millisecond latencies using the **Cache-Aside (Lazy Loading)** pattern.

However, caching introduces distributed state synchronization challenges that QA engineers must rigorously test:
- **Stale Data / Invalidation Failure**: Updates made to the primary database fail to evict or update the corresponding Redis keys.
- **Cache Penetration**: Requests for non-existent data bypass the cache entirely, repeatedly hitting and overwhelming the primary database.
- **Cache Stampede (Thundering Herd)**: When a heavily queried key expires, thousands of concurrent requests miss simultaneously and flood the database to recompute the same value.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Cache Stampede Scenario                         │
│                                                                        │
│   Concurrent User Requests (10,000 req/sec)                            │
│   ═════════════════════════════════════════════════════════════════►   │
│                                                                        │
│                    ┌────────────────────────────┐                      │
│                    │   Redis Cache (Key Expired)│                      │
│                    └─────────────┬──────────────┘                      │
│                                  │                                     │
│                     Simultaneous Cache MISSES                          │
│                                  │                                     │
│                                  ▼                                     │
│                    ┌────────────────────────────┐                      │
│                    │     Primary Database       │                      │
│                    │    (CPU & IOPS 100% Sat)   │                      │
│                    │   💥 Connection Exhaustion 💥                      │
│                    └────────────────────────────┘                      │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Testing Strategies for Cache Edge Cases

### A. Cache Penetration Testing
- **Vulnerability**: Attackers request malicious or non-existent resource IDs (`/api/products?id=-9999` or randomly generated UUIDs).
- **QA Verification**: Ensure the system caches empty results (`null` with short TTL, e.g., 60 seconds) or utilizes a **Bloom Filter** to reject invalid IDs before querying the database.

### B. Cache Stampede Mitigation Verification
To verify resilience against stampedes, QA tests validate the implementation of:
1. **Mutex / Distributed Locking**: Only one worker acquires a lock (`SET key value NX PX 5000`) to query the database and populate the cache; all other requests wait or return stale data.
2. **Probabilistic Early Expiration (XFetch Algorithm)**: The cache entry computes expiration probability based on read frequency and background worker pre-warms the key before actual TTL expiry.

---

## 3. Automated Cache Invalidation & Stampede Test Suite

Below is a Mocha/Chai test script utilizing `ioredis` and a mock database to test cache invalidation timing and concurrent stampede lock protection:

```javascript
const Redis = require('ioredis');
const { expect } = require('chai');

const redis = new Redis('redis://localhost:6379');

// Mock Data Access Layer with query counter
let dbQueryCount = 0;
async function fetchProductFromDatabase(productId) {
  dbQueryCount++;
  // Simulate slow DB latency
  await new Promise((res) => setTimeout(res, 100));
  return { id: productId, name: 'Wireless Headphones', price: 89.99 };
}

// Resilient Cache-Aside Fetch with Distributed Mutex Lock
async function getProductWithMutex(productId) {
  const cacheKey = `product:${productId}`;
  const lockKey = `lock:product:${productId}`;

  // 1. Try reading from cache
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // 2. Try acquiring single mutex lock (valid for 3 seconds)
  const acquiredLock = await redis.set(lockKey, 'locked', 'NX', 'PX', 3000);

  if (acquiredLock === 'OK') {
    try {
      // Re-fetch database
      const product = await fetchProductFromDatabase(productId);
      // Cache with 10-second TTL
      await redis.set(cacheKey, JSON.stringify(product), 'EX', 10);
      return product;
    } finally {
      // Release lock
      await redis.del(lockKey);
    }
  } else {
    // Wait and retry reading from cache
    await new Promise((res) => setTimeout(res, 50));
    return getProductWithMutex(productId);
  }
}

describe('Redis Cache Invalidation & Stampede Resilience Suite', () => {
  beforeEach(async () => {
    dbQueryCount = 0;
    await redis.flushall();
  });

  after(async () => {
    await redis.quit();
  });

  it('should only query database once when 100 concurrent requests hit an expired key', async () => {
    const productId = 'prod_1001';

    // Dispatch 100 parallel requests simultaneously
    const requests = Array.from({ length: 100 }, () => getProductWithMutex(productId));
    const results = await Promise.all(requests);

    // All requests must receive accurate data
    expect(results).to.have.lengthOf(100);
    results.forEach((res) => expect(res.name).to.equal('Wireless Headphones'));

    // Crucial Stampede Verification: Database must ONLY have been hit 1 time!
    expect(dbQueryCount).to.equal(1);
  });

  it('should immediately evict cache when entity is updated', async () => {
    const productId = 'prod_1002';
    const cacheKey = `product:${productId}`;

    // Seed cache
    await getProductWithMutex(productId);
    expect(dbQueryCount).to.equal(1);

    // Simulate entity update: Service must invalidate cache
    await redis.del(cacheKey);

    // Subsequent request must re-query database
    await getProductWithMutex(productId);
    expect(dbQueryCount).to.equal(2);
  });
});
```

---

## 4. QA Monitoring & Debugging Commands

During load testing, QA engineers monitor Redis cache efficiency via the Redis CLI:

```bash
# Monitor cache hit vs miss ratio
redis-cli info stats | grep -E "keyspace_hits|keyspace_misses"

# Calculate Hit Rate %:
# Hit Rate = keyspace_hits / (keyspace_hits + keyspace_misses) * 100

# Inspect memory consumption and eviction policy
redis-cli info memory | grep -E "used_memory_human|maxmemory_policy"

# Watch live commands in real-time
redis-cli monitor
```

---

## 5. QA Verification Checklist

- [ ] **Cache Invalidation on Mutation**: Confirm that all `PUT`, `POST`, and `DELETE` operations reliably delete or update related Redis keys.
- [ ] **Stampede Protection**: Verify under load (k6/Artillery) that key expiration does not cause exponential spikes in database connection pools.
- [ ] **Negative Caching (Penetration)**: Non-existent IDs must be cached with short TTLs or rejected by filters to protect SQL servers.
- [ ] **TTL Expiration Validation**: Ensure all cached keys possess explicit TTLs (`EX`/`PX`) to prevent unbounded memory leaks (`maxmemory-policy` evictions).
