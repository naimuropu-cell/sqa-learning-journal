# Database Isolation Levels & Concurrency Phenomena Guide for QA

## Introduction

In multi-user transactional databases (such as banking systems, booking platforms, and e-commerce stores), thousands of concurrent database transactions execute simultaneously.

The **Isolation** property in ACID guarantees that concurrent transactions do not interfere with one another. However, achieving 100% isolation (**Serializable**) requires aggressive locking that severely degrades database throughput and performance.

To balance performance against consistency, relational database engines (PostgreSQL, MySQL, SQL Server, Oracle) provide **Four Standard Transaction Isolation Levels**.

QA engineers must understand these isolation levels and their associated concurrency phenomena to test financial calculations, inventory reservations, and prevent data corruption under peak load.

---

## The Four Standard Isolation Levels & Concurrency Phenomena

```
┌─────────────────────────────────────────────────────────────────────────┐
│              Isolation Levels vs. Permitted Phenomena Matrix            │
├────────────────────┬─────────────┬─────────────────────┬────────────────┤
│ Isolation Level    │ Dirty Read  │ Non-Repeatable Read │ Phantom Read   │
├────────────────────┼─────────────┼─────────────────────┼────────────────┤
│ 1. Read            │ Permitted ❌│ Permitted ❌        │ Permitted ❌   │
│    Uncommitted     │ (Danger!)   │                     │                │
├────────────────────┼─────────────┼─────────────────────┼────────────────┤
│ 2. Read Committed  │ Prevented ✅│ Permitted ❌        │ Permitted ❌   │
│    (Postgres/Oracle│             │                     │                │
│     default)       │             │                     │                │
├────────────────────┼─────────────┼─────────────────────┼────────────────┤
│ 3. Repeatable Read │ Prevented ✅│ Prevented ✅        │ Permitted ❌   │
│    (MySQL default) │             │                     │ (Prevented in  │
│                    │             │                     │  InnoDB MVCC)  │
├────────────────────┼─────────────┼─────────────────────┼────────────────┤
│ 4. Serializable    │ Prevented ✅│ Prevented ✅        │ Prevented ✅   │
│    (Highest Rigor) │ (Slowest)   │                     │                │
└────────────────────┴─────────────┴─────────────────────┴────────────────┘
```

---

## Deep Dive into the Concurrency Read Phenomena

### 1. Dirty Read (Reading Uncommitted Data)
* **The Scenario**:
  1. Transaction 1 transfers $500 from Account A to Account B.
  2. Transaction 2 reads Account B's balance and sees the new +$500.
  3. Transaction 1 fails and executes a `ROLLBACK`.
* **The Glitch**: Transaction 2 operated on data that *never officially existed* in the database!

### 2. Non-Repeatable Read (Fuzzy Read)
* **The Scenario**:
  1. Transaction 1 reads a user's balance: `$100`.
  2. Transaction 2 updates the balance to `$50` and `COMMITS`.
  3. Transaction 1 reads the exact same row again within the same transaction: it now reads `$50`!
* **The Glitch**: The data within the same transaction changed between two reads of the same row.

### 3. Phantom Read
* **The Scenario**:
  1. Transaction 1 executes a range query: `SELECT COUNT(*) FROM orders WHERE total > 100` (Returns 5 rows).
  2. Transaction 2 inserts a brand new order with `total = 150` and `COMMITS`.
  3. Transaction 1 executes the exact same query: it now returns 6 rows!
* **The Glitch**: A "phantom" row appeared inside the query range.

---

## Practical QA Testing: Testing Isolation via Two Concurrent SQL Terminals

QA can verify database isolation by opening two terminal connections to PostgreSQL / MySQL:

```sql
-- CONNECTION 1:
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
SELECT balance FROM accounts WHERE id = 1; -- Returns 1000

-- CONNECTION 2:
BEGIN;
UPDATE accounts SET balance = 500 WHERE id = 1;
-- (Do NOT commit yet!)

-- CONNECTION 1 (Testing Dirty Read):
SELECT balance FROM accounts WHERE id = 1;
-- Result: Must return 1000 (Confirms Dirty Read is PREVENTED ✅)

-- CONNECTION 2:
COMMIT;

-- CONNECTION 1 (Testing Non-Repeatable Read):
SELECT balance FROM accounts WHERE id = 1;
-- Under READ COMMITTED: Returns 500!
-- Under REPEATABLE READ: Would still return 1000!
```

---

## SQA Interview Questions & Answers

### Q: Why isn't every database configured to use the "Serializable" isolation level by default?
**Answer:**
Serializable provides complete transaction safety, but at a massive cost to system concurrency and throughput. It requires either strict two-phase locking (2PL) of large table ranges or complex optimistic concurrency tracking (SSI). In high-throughput applications, Serializable causes widespread lock contention, transaction timeouts, deadlocks, and severe slowdowns. Most production databases use **Read Committed** or **Repeatable Read** paired with application-level optimistic locking.

### Q: What is MVCC (Multi-Version Concurrency Control)?
**Answer:**
MVCC is the modern concurrency mechanism used by PostgreSQL, MySQL (InnoDB), and Oracle to allow readers and writers to operate concurrently without blocking each other ("readers never block writers, and writers never block readers"). When a transaction updates a row, the database creates a new timestamped version of the row instead of overwriting it in place. Older active transactions continue reading their consistent older snapshot of the row.

---

## Key Takeaways

* ACID Isolation prevents concurrent transactions from corrupting shared data.
* The 3 classic phenomena are Dirty Reads, Non-Repeatable Reads, and Phantom Reads.
* Test database concurrency across multiple simultaneous terminal connections to verify transaction boundaries.

---

## Conclusion

Understanding database isolation levels allows QA engineers to design targeted concurrency tests for financial and transactional systems. By validating transaction boundaries and locking behavior, QA ensures that databases maintain data integrity under peak concurrent user loads.
