# Database Connection Pool Exhaustion, Leak Detection & Timeout Testing

## 1. The Critical Role of Connection Pooling

Opening and closing TCP connections directly to a relational database (PostgreSQL, MySQL, Oracle) is computationally expensive: it involves TCP three-way handshakes, TLS negotiation, authentication, memory allocation, and process creation.

To optimize throughput, applications use **Connection Pools** (like **HikariCP** in Java/Spring, **pg-pool** in Node.js, or **SQLAlchemy** in Python). The pool maintains a pre-warmed set of reusable connections.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Connection Pool Architecture                    │
│                                                                        │
│   Incoming HTTP Requests (100 concurrent workers)                      │
│   ───► [Worker 1] [Worker 2] [Worker 3] ... [Worker 100]               │
│               │            │            │                              │
│               ▼            ▼            ▼                              │
│   ┌──────────────────────────────────────────────────────────────┐     │
│   │            Connection Pool (HikariCP / pg-pool)              │     │
│   │   Max Size: 10 Connections                                   │     │
│   │   [Conn 1: Active] [Conn 2: Active] ... [Conn 10: Active]    │     │
│   └──────────────────────────────┬───────────────────────────────┘     │
│                                  │                                     │
│                     Connection Queue Backlog                           │
│                     (90 Requests Waiting...)                           │
│                                  │                                     │
│                     ⚠️ Exceeds ConnectionTimeout?                      │
│                     💥 Throws PoolTimeoutException 💥                  │
└────────────────────────────────────────────────────────────────────────┘
```

However, under production load or due to coding defects, pools suffer from:
1. **Connection Leaks**: Code acquires a connection from the pool but fails to call `client.release()` or `conn.close()` inside a `finally` block.
2. **Pool Starvation**: Long-running database transactions or slow external API calls made inside database transactions hold connections for seconds, starving all other requests.
3. **Improper Timeout Configuration**: Requests wait indefinitely in the queue, exhausting server thread pools and crashing the application.

---

## 2. Key Pool Metrics Every QA Engineer Must Monitor

| Metric Name | Target Description | Failure Indication |
| :--- | :--- | :--- |
| `pool.active_connections` | Connections currently executing queries | At maximum capacity for sustained periods |
| `pool.idle_connections` | Warm connections waiting for new work | Drops to 0 during load tests |
| `pool.pending_requests` | Threads queued waiting for a free connection | Rapidly climbing queue indicates starvation |
| `connectionTimeoutMillis` | Max time a thread waits for a connection | High rate of `500 Internal Server Error` |
| `leakDetectionThreshold` | Time before HikariCP logs a connection leak warning | Warns in logs: `Apparent connection leak detected` |

---

## 3. Automated Connection Leak Stress Test Script (Node.js)

Below is an automated test script intentionally modeling a connection leak defect and verifying that monitoring detects the exhaustion:

```javascript
const { Pool } = require('pg');
const { expect } = require('chai');

describe('Database Connection Pool Resilience & Leak Detection Suite', () => {
  let pool;

  before(() => {
    // Configure small pool to quickly surface pool exhaustion
    pool = new Pool({
      connectionString: 'postgresql://postgres:postgres@localhost:5432/test_db',
      max: 5,                       // Only 5 maximum connections allowed
      connectionTimeoutMillis: 2000, // Timeout after 2 seconds if no connection available
    });
  });

  after(async () => {
    await pool.end();
  });

  it('should detect pool exhaustion when connections are leaked without calling release()', async () => {
    // 1. Simulate a buggy service that forgets to release connections
    const leakedConnections = [];
    for (let i = 0; i < 5; i++) {
      const client = await pool.connect();
      await client.query('SELECT 1');
      leakedConnections.push(client);
      // BUG: Omitted client.release()!
    }

    // Pool must now be 100% saturated
    expect(pool.totalCount).to.equal(5);
    expect(pool.waitingCount).to.equal(0);

    // 2. Attempt to acquire a 6th connection
    const startTime = Date.now();
    try {
      await pool.connect();
      expect.fail('Expected pool.connect() to throw timeout exception');
    } catch (err) {
      const duration = Date.now() - startTime;

      // Verify it failed because of timeout (exceeded 2000ms)
      expect(duration).to.be.at.least(1900);
      expect(err.message).to.include('timeout');
    } finally {
      // Clean up: Release leaked connections
      leakedConnections.forEach((client) => client.release());
    }

    // Verify pool recovered after releasing
    const recoveryClient = await pool.connect();
    expect(recoveryClient).to.not.be.null;
    recoveryClient.release();
  });
});
```

---

## 4. Querying Active Connections Directly from the Database

During performance tests, QA engineers verify database-level connection usage:

### PostgreSQL Connection Query:
```sql
SELECT
  count(*) AS total_connections,
  count(*) FILTER (WHERE state = 'active') AS active_connections,
  count(*) FILTER (WHERE state = 'idle') AS idle_connections,
  count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_transaction_leaks
FROM pg_stat_activity
WHERE datname = 'app_production';
```

> **Critical QA Finding**: Any connection in the `idle in transaction` state for more than 5 seconds represents a critical defect where a developer began a transaction (`BEGIN`), executed a query, and forgot to commit or rollback!

---

## 5. QA Verification Checklist

- [ ] **Timeout Enforcement**: Ensure `connectionTimeoutMillis` is set to a reasonable limit (e.g., 2,000–5,000ms) to fail fast rather than hanging threads indefinitely.
- [ ] **Leak Detection Threshold**: Verify HikariCP / ORM has `leakDetectionThreshold` enabled in non-production environments to flag unclosed connections in logs.
- [ ] **Pool Sizing Formula**: Validate that `maxPoolSize` aligns with database CPU cores rather than being arbitrarily set to excessively high numbers (`Connections = ((CPU_cores * 2) + effective_spindle_count)`).
- [ ] **Health Check Validation**: Ensure connection test queries (`SELECT 1`) validate connection health before handing connections to application threads.
