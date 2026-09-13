# Concurrency & Race Condition Testing Guide for QA

## Introduction

An application may execute flawlessly when tested by a single QA engineer clicking buttons sequentially. However, in production, thousands of concurrent users interact with the exact same database records, inventory stock, and account balances at the exact same millisecond.

A **Race Condition** occurs when the outcome of a system depends on the unpredictable sequence or timing of concurrent threads or processes.

Concurrency defects are among the most elusive and financially catastrophic bugs in software engineering: they enable users to double-spend account balances, purchase items that are out of stock, or overwrite concurrent database edits.

---

## The Anatomy of a Race Condition: "Check-Then-Act"

The most common concurrency vulnerability arises from the non-atomic **Check-Then-Act** anti-pattern:

```
THREAD A (User clicks 'Withdraw $100')         THREAD B (User clicks 'Withdraw $100' simultaneously)
┌────────────────────────────────────────┐     ┌────────────────────────────────────────┐
│ 1. Read Balance: $100                  │     │ 1. Read Balance: $100                  │
│ 2. Check: Is balance >= $100? YES!     │     │ 2. Check: Is balance >= $100? YES!     │
│ 3. Deduct $100 (Balance = $0)          │     │ 3. Deduct $100 (Balance = $0)          │
│ 4. Dispense $100                       │     │ 4. Dispense $100                       │
└────────────────────────────────────────┘     └────────────────────────────────────────┘
                    Total Cash Dispensed: $200! (Account only had $100!)
```

---

## Common Concurrency Defects

```
┌─────────────────────────────────────────────────────────────┐
│                 Common Concurrency Defect Types             │
├─────────────────────┬───────────────────────────────────────┤
│ Defect Type         │ Real-World Example                    │
├─────────────────────┼───────────────────────────────────────┤
│ 1. Double-Spend     │ Transferring the same bank balance to │
│                     │ two recipients via concurrent clicks  │
├─────────────────────┼───────────────────────────────────────┤
│ 2. Inventory        │ Two buyers simultaneously purchase the│
│    Oversell         │ single remaining item in stock        │
├─────────────────────┼───────────────────────────────────────┤
│ 3. Coupon Abuse     │ Redeeming a "single-use" promo coupon │
│                     │ multiple times via parallel requests  │
├─────────────────────┼───────────────────────────────────────┤
│ 4. Lost Update      │ User A and User B edit a doc at once; │
│                     │ User B's save silently wipes User A   │
└─────────────────────┴───────────────────────────────────────┘
```

---

## Database Locking Strategies Tested by QA

1. **Pessimistic Locking (`SELECT ... FOR UPDATE`)**:
   * The database locks the row the instant Thread A reads it. Thread B must wait until Thread A completes its transaction.
   * *QA Test*: Verify that concurrent requests wait in a queue without throwing unhandled deadlock exceptions.
2. **Optimistic Locking (`version` column)**:
   * Each record includes a version number (e.g., `version: 4`). When updating, the query asserts `WHERE version = 4`. If another thread updated it first, the version is now 5, and the update fails safely.
   * *QA Test*: Verify the second transaction is safely rejected with `HTTP 409 Conflict` and a friendly retry prompt.

---

## Automated Concurrency Testing Script (Node.js)

To reliably test for race conditions, fire concurrent requests simultaneously using `Promise.all()`:

```typescript
import axios from 'axios';
import { test, expect } from '@playwright/test';

test('Simultaneous checkout requests on last stock item prevents overselling', async () => {
  const ENDPOINT = 'https://api.example.com/v1/orders/checkout';
  const PRODUCT_ID = 'prod_limited_item_1'; // Item with stock = 1

  // Fire 5 simultaneous checkout requests at the exact same millisecond
  const requestPromises = [1, 2, 3, 4, 5].map((userId) =>
    axios.post(ENDPOINT, {
      productId: PRODUCT_ID,
      userId: `customer_${userId}`,
    }).then(res => ({ success: true, status: res.status }))
      .catch(err => ({ success: false, status: err.response?.status }))
  );

  const results = await Promise.all(requestPromises);

  // Assertions
  const successfulPurchases = results.filter((r) => r.success && r.status === 201);
  const rejectedPurchases = results.filter((r) => !r.success && r.status === 409);

  console.log(`Successful purchases: ${successfulPurchases.length}`);
  console.log(`Rejected purchases: ${rejectedPurchases.length}`);

  // Exactly ONE purchase must succeed!
  expect(successfulPurchases.length).toBe(1);
  expect(rejectedPurchases.length).toBe(4);
});
```

---

## SQA Interview Questions & Answers

### Q: Why do traditional manual test cases fail to detect race conditions?
**Answer:**
Human testers interact with user interfaces at human speeds (fractions of a second to multiple seconds apart). Race conditions only manifest when two or more operations interleave at the millisecond or microsecond level inside the operating system or database transaction window. Uncovering race conditions requires programmatic multi-threading tools, parallel API scripts, or high-concurrency load injectors.

### Q: What is a Deadlock and how can QA detect it?
**Answer:**
A deadlock occurs when two concurrent transactions each hold a lock that the other transaction requires to finish, leaving both processes frozen in an infinite wait. QA detects deadlocks during stress and load testing by observing sudden connection pool exhaustion, elevated API timeout rates (HTTP 504), and database error logs reporting `Deadlock found when trying to get lock; try restarting transaction`.

---

## Key Takeaways

* Concurrency bugs occur when asynchronous threads access shared mutable state without proper locking.
* Test for double-spend and inventory overselling by firing concurrent requests using `Promise.all()` or k6.
* Verify database protection mechanisms: Optimistic Locking (versioning) and Pessimistic Locking (`SELECT FOR UPDATE`).

---

## Conclusion

Concurrency and race condition testing protects software from severe financial exploits and data corruption. By simulating simultaneous multi-threaded transactions in automated test suites, QA engineers ensure that systems remain atomic, consistent, and strictly isolated under peak real-world concurrency.
