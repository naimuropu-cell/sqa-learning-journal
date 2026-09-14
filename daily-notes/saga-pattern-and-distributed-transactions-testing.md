# Saga Pattern & Distributed Transactions Testing Guide for QA

## Introduction

In traditional monolithic architectures with a single relational database, maintaining transactional consistency was simple: developers wrapped operations inside an ACID transaction (`BEGIN TRANSACTION ... COMMIT`). If any step failed, the database executed a complete automatic `ROLLBACK`.

In a **Microservices Architecture**, each microservice owns its own private database. A business transaction (e.g., booking a vacation with flight, hotel, and car rental) spans multiple independent services across the network. Because traditional distributed two-phase commit (2PC) protocols do not scale and lock resources across networks, modern distributed systems implement the **Saga Pattern**.

Testing Sagas requires verifying asynchronous workflows, distributed state machines, and—most crucially—**Compensating Transactions** (the distributed "undo" operations).

---

## What is the Saga Pattern?

A **Saga** is a design pattern that manages distributed transactions as a sequence of local transactions:
1. Each microservice updates its own local database.
2. The service publishes an event or message to trigger the next local transaction in the saga.
3. If a step fails, the saga executes a series of **Compensating Transactions** that undo the changes made by preceding steps.

```
HAPPY PATH:
[ Order Service ] ──(Order Created)──► [ Payment Service ] ──(Payment Authorized)──► [ Stock Service ] ──► Complete! ✅

FAILURE SCENARIO & COMPENSATING TRANSACTIONS:
[ Order Service ] ──(Order Created)──► [ Payment Service ] ──(Payment Authorized)──► [ Stock Service ] ──► OUT OF STOCK! ❌
      │                                       │
      ▼                                       ▼
Compensating: Cancel Order ◄────────── Compensating: Refund Payment
```

---

## Choreography vs. Orchestration Sagas

| Dimension | Choreography-Based Saga | Orchestration-Based Saga |
| :--- | :--- | :--- |
| **Coordination** | Decentralized; services listen to events | Centralized; a dedicated Saga Orchestrator directs steps |
| **Coupling** | Loosely coupled via message broker (Kafka) | Services communicate with orchestrator (Temporal, Camunda) |
| **Complexity** | Difficult to track flow as services grow | Flow is explicitly visible in orchestrator code/dashboard |
| **Best For** | Simple workflows (2–3 services) | Complex workflows with numerous steps and compensation logic |

---

## Critical QA Test Scenarios for Distributed Sagas

```
┌─────────────────────────────────────────────────────────────┐
│                 Saga Testing Scenarios                      │
├─────────────────────┬───────────────────────────────────────┤
│ Scenario            │ Verification Objective                │
├─────────────────────┼───────────────────────────────────────┤
│ 1. Complete Success │ All local transactions commit; final  │
│    (Happy Path)     │ aggregate state is COMPLETED          │
├─────────────────────┼───────────────────────────────────────┤
│ 2. Failure at End   │ When final service fails, preceding   │
│    (Full Rollback)  │ services execute compensating actions │
│                     │ in reverse sequential order           │
├─────────────────────┼───────────────────────────────────────┤
│ 3. Compensating     │ If an "undo" operation fails (e.g.,   │
│    Failure          │ refund timeout), system alerts human  │
│                     │ operator via Dead-Letter Queue (DLQ)  │
├─────────────────────┼───────────────────────────────────────┤
│ 4. Idempotent Undo  │ If compensation event is delivered    │
│                     │ twice, refund is NOT processed twice  │
└─────────────────────┴───────────────────────────────────────┘
```

---

## Step-by-Step QA Verification Workflow for Compensating Transactions

### Test Case: Simulating Insufficient Stock during E-Commerce Checkout
1. **Setup Preconditions**: Seed the database with a product having `inventory_stock = 0`.
2. **Execute Transaction**: Submit a checkout request for $100.
3. **Observe Service 1 (Order Service)**: Creates order with status `PENDING`.
4. **Observe Service 2 (Payment Service)**: Charges credit card $100 successfully.
5. **Observe Service 3 (Inventory Service)**: Fails with `INSUFFICIENT_STOCK`.
6. **Assert Compensating Actions**:
   * Verify Payment Service receives failure event and issues an **automatic refund of $100**.
   * Verify Order Service transitions order status to `CANCELED`.
   * Verify customer receives an email notification: *"Order canceled due to out-of-stock item; payment has been refunded."*

---

## SQA Interview Questions & Answers

### Q: Why can't we use traditional database ROLLBACK in microservices?
**Answer:**
In a microservices architecture, there is no single shared database. The Order Service writes to PostgreSQL, the Payment Service talks to Stripe's external API, and the Inventory Service writes to MongoDB. Standard SQL `ROLLBACK` cannot operate across physical network boundaries or external third-party payment gateways. The Saga pattern solves this by issuing explicit, opposite application-level business actions (Compensating Transactions, such as issuing a refund or cancelling an order reservation).

### Q: What is a "Pivot Transaction" in a Saga?
**Answer:**
A **Pivot Transaction** is the point of no return in a saga lifecycle. Once the pivot transaction commits, the saga is guaranteed to run to completion and cannot be rolled back. Any step *before* the pivot transaction must be compensatable; any step *after* the pivot transaction must be retried until it succeeds.

---

## Key Takeaways

* Microservices replace traditional ACID transactions with the Saga Pattern.
* Every forward transaction that modifies state must have a corresponding Compensating Transaction.
* QA must rigorously test failure scenarios at every step to ensure compensating rollbacks restore data consistency.

---

## Conclusion

Testing distributed sagas requires thinking beyond isolated API responses to examine system-wide eventual consistency. By verifying forward workflows, reverse compensating transactions, and idempotent recovery, QA engineers ensure that distributed architectures remain reliable and financially sound even when components fail mid-transaction.
