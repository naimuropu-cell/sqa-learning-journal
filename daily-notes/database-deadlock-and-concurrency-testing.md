# Database Concurrency, Row-Lock Contention & Deadlock Testing for SQA

## 1. Concurrency Anomalies in Modern Databases

When hundreds of concurrent transactions execute against relational databases (such as PostgreSQL, MySQL InnoDB, or Microsoft SQL Server), database engines balance performance against data integrity using **ACID** transactions.

QA engineers must validate systems under multi-threaded concurrency to prevent critical transaction anomalies:

| Concurrency Anomaly | Description | Minimum Isolation Level Required |
| :--- | :--- | :--- |
| **Dirty Read** | Transaction reads uncommitted data written by a concurrent transaction | Read Committed |
| **Non-Repeatable Read** | Re-reading the same row within a transaction returns altered values | Repeatable Read |
| **Phantom Read** | Re-executing a range query returns newly inserted rows matching criteria | Serializable |
| **Lost Update** | Two transactions read the same value, calculate, and overwrite without synchronization | Repeatable Read / Optimistic Locking |
| **Deadlock** | Two or more transactions mutually block each other, each holding a lock the other needs | Database Engine Aborts One (Error 40P01 / 1213) |

```
┌────────────────────────────────────────────────────────┐
│                   Deadlock Cycle                       │
│                                                        │
│       Transaction 1               Transaction 2        │
│    ┌──────────────────┐        ┌──────────────────┐    │
│    │ Holds Lock on A  │        │ Holds Lock on B  │    │
│    └────────┬─────────┘        └────────┬─────────┘    │
│             │                           │              │
│             ▼                           ▼              │
│    ┌──────────────────┐        ┌──────────────────┐    │
│    │ Waiting for B    │◄───────┤ Waiting for A    │    │
│    └──────────────────┘        └──────────────────┘    │
└────────────────────────────────────────────────────────┘
```

---

## 2. Inducing Deadlocks in QA Test Environments

To verify that applications handle deadlocks gracefully (e.g., triggering automatic retries with jitter rather than throwing unexpected `500 Internal Server Error` responses to users), QA tests intentionally induce deadlocks.

### Reproducing Mutual Inversion Deadlock (PostgreSQL / MySQL)

Consider a wallet transfer scenario where two users simultaneously transfer money to each other:

#### Thread 1 (Transfer from User 1 to User 2):
```sql
BEGIN;
-- Step 1: Lock User 1
UPDATE accounts SET balance = balance - 50 WHERE account_id = 1;

-- (Wait 1 second for Thread 2 to acquire lock on User 2)

-- Step 3: Attempt to lock User 2 -> BLOCKED waiting for Thread 2!
UPDATE accounts SET balance = balance + 50 WHERE account_id = 2;
COMMIT;
```

#### Thread 2 (Transfer from User 2 to User 1):
```sql
BEGIN;
-- Step 2: Lock User 2
UPDATE accounts SET balance = balance - 50 WHERE account_id = 2;

-- Step 4: Attempt to lock User 1 -> DEADLOCK DETECTED!
UPDATE accounts SET balance = balance + 50 WHERE account_id = 1;
COMMIT;
```

### Database Reaction:
PostgreSQL detects the cycle via its deadlock detector daemon and aborts one transaction:
```text
ERROR: deadlock detected
DETAIL: Process 23145 waits for ShareLock on transaction 1092; blocked by process 23146.
Process 23146 waits for ShareLock on transaction 1091; blocked by process 23145.
HINT: See server log for query details.
CONTEXT: while updating tuple (0,2) in relation "accounts"
SQLSTATE: 40P01
```

---

## 3. Automated Concurrency & Deadlock Stress Script

Below is a Node.js test script utilizing worker pools to simulate 50 concurrent wallet transfers across identical accounts to verify error retry policies:

```javascript
const { Pool } = require('pg');
const { expect } = require('chai');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL || 'postgresql://postgres:postgres@localhost:5432/test_db',
  max: 20,
});

// Idempotent Transfer Service with Exponential Backoff Retry
async function transferWithRetry(fromId, toId, amount, maxRetries = 3) {
  let attempts = 0;
  while (attempts < maxRetries) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN');

      // Enforce deterministic lock ordering to prevent deadlocks:
      // Always lock lower ID first!
      const [firstId, secondId] = fromId < toId ? [fromId, toId] : [toId, fromId];
      await client.query('SELECT * FROM accounts WHERE id = $1 FOR UPDATE', [firstId]);
      await client.query('SELECT * FROM accounts WHERE id = $2 FOR UPDATE', [secondId]);

      await client.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [amount, fromId]);
      await client.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [amount, toId]);

      await client.query('COMMIT');
      client.release();
      return { success: true, attempts: attempts + 1 };
    } catch (err) {
      await client.query('ROLLBACK');
      client.release();

      // Check PostgreSQL Deadlock Error Code: 40P01
      if (err.code === '40P01') {
        attempts++;
        const backoff = Math.floor(Math.random() * 100) + Math.pow(2, attempts) * 50;
        await new Promise((res) => setTimeout(res, backoff));
      } else {
        throw err;
      }
    }
  }
  throw new Error(`Transaction failed after ${maxRetries} deadlock retries`);
}

describe('Database High-Contention Concurrency Suite', () => {
  it('should successfully complete 50 bidirectional concurrent transfers without data loss', async () => {
    const promises = [];
    for (let i = 0; i < 25; i++) {
      promises.push(transferWithRetry(1, 2, 10));
      promises.push(transferWithRetry(2, 1, 10));
    }

    const results = await Promise.all(promises);
    expect(results).to.have.lengthOf(50);
    results.forEach((res) => expect(res.success).to.be.true);

    const { rows } = await pool.query('SELECT SUM(balance) as total FROM accounts WHERE id IN (1, 2)');
    // Balance sum must remain constant (Conservation of Money)
    expect(Number(rows[0].total)).to.equal(1000);
  });
});
```

---

## 4. QA Deadlock Monitoring Commands

### PostgreSQL Monitoring
```sql
-- View all currently blocked and blocking queries
SELECT
  blocked_locks.pid     AS blocked_pid,
  blocked_activity.usename  AS blocked_user,
  blocking_locks.pid    AS blocking_pid,
  blocking_activity.usename AS blocking_user,
  blocked_activity.query    AS blocked_statement,
  blocking_activity.query   AS blocking_statement
FROM  pg_catalog.pg_locks         blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks         blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

---

## 5. QA Verification Checklist

- [ ] **Deterministic Lock Ordering**: Verify that multi-row updates always sort resource IDs before acquiring locks.
- [ ] **Deadlock Exception Retries**: Ensure application code catches SQL State `40P01` (Postgres) / `1213` (MySQL) and performs retries with randomized backoff.
- [ ] **Transaction Duration Minimization**: Keep transactions short; eliminate external network calls (e.g., third-party HTTP requests) inside database transactions.
- [ ] **Isolation Level Audits**: Verify whether operations require `SERIALIZABLE` or if `READ COMMITTED` with selective `FOR UPDATE` locking provides sufficient protection.
