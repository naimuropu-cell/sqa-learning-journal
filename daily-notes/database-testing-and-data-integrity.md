# Database Testing and Data Integrity Verification for QA

## Introduction

User interfaces and API responses are only reflections of what is stored in the underlying database. A bug in UI validation might allow malformed data through, but a failure in the database layer can result in data corruption, financial calculation errors, compliance violations, and catastrophic loss of business data.

**Database Testing (Back-End Testing)** is the process of validating the schema, tables, triggers, stored procedures, data integrity, and transactions within a database management system (RDBMS / NoSQL).

---

## The Four Pillars of Data Integrity

QA engineers must verify data integrity across four foundational dimensions:

```
                  ┌─────────────────────────────────────────┐
                  │          Database Data Integrity        │
                  └─────────────────────────────────────────┘
                       │              │            │
       ┌───────────────┴────┐         │      ┌─────┴─────────────────┐
       ▼                    ▼         │      ▼                       ▼
┌──────────────┐   ┌──────────────┐   │ ┌──────────────┐   ┌───────────────────┐
│ Entity       │   │ Referential  │   │ │ Domain       │   │ User-Defined      │
│ Integrity    │   │ Integrity    │   │ │ Integrity    │   │ Integrity         │
│ (Primary Key,│   │ (Foreign Key,│   │ │ (Data Types, │   │ (Custom Rules,    │
│ Not Null)    │   │ No Orphans)  │   │ │ Constraints) │   │ Business Logic)   │
└──────────────┘   └──────────────┘   │ └──────────────┘   └───────────────────┘
```

1. **Entity Integrity**: Ensures that every table has a unique primary key and that primary key columns cannot contain `NULL` values.
2. **Referential Integrity**: Ensures that relationships between tables remain consistent. Foreign keys must always reference an existing primary key or be explicitly `NULL` (no "orphan" records).
3. **Domain Integrity**: Ensures valid data entries within specified columns (e.g., proper data types, formats, lengths, and `CHECK` constraints like `age >= 18`).
4. **User-Defined Integrity**: Enforces custom business rules that are not covered by the other three (e.g., an account balance cannot fall below $0 without overdraft approval).

---

## Validating ACID Properties

In transactional applications (e.g., banking, e-commerce, healthcare), QA engineers must verify that operations strictly adhere to **ACID** properties:

| Property | Meaning | QA Testing Strategy |
| :--- | :--- | :--- |
| **Atomicity** | All operations in a transaction succeed, or none do ("All or Nothing"). | Simulate a network or power failure midway through a multi-step checkout; verify that money is NOT deducted if order creation fails. |
| **Consistency** | Database moves from one valid state to another, maintaining all schema rules. | Validate that total credits equal total debits after a fund transfer. |
| **Isolation** | Concurrent transactions execute without interfering with one another. | Execute simultaneous debit requests on the same account using multi-threading to check for race conditions. |
| **Durability** | Once committed, data survives system crashes, restarts, or power outages. | Verify that committed records persist immediately after a database restart or failover. |

---

## Practical SQL Queries for QA Verification

### 1. Detecting Orphan Records (Referential Integrity Check)
To find orders that point to non-existent users:

```sql
SELECT 
    o.order_id, 
    o.user_id, 
    o.created_at
FROM orders o
LEFT JOIN users u ON o.user_id = u.id
WHERE u.id IS NULL;
```
*Expected Result:* Zero rows returned. Any row indicates a broken foreign key constraint.

### 2. Identifying Duplicate Entries Violating Business Keys
To verify that unique email constraints or composite keys are properly maintained:

```sql
SELECT 
    email, 
    COUNT(*) as occurrences
FROM accounts
GROUP BY email
HAVING COUNT(*) > 1;
```

### 3. Verifying Nullability & Boundary Range Constraints
To detect records violating domain constraints:

```sql
SELECT 
    product_id, 
    price, 
    stock_quantity
FROM inventory
WHERE price < 0.00 
   OR stock_quantity < 0 
   OR sku IS NULL;
```

---

## Schema Migration & Migration Testing

Whenever backend teams update the database schema via tools like **Flyway** or **Liquibase**, QA must conduct migration testing:

1. **Forward Migration**: Apply migration scripts on a copy of production data and verify that columns, tables, and views are created accurately without data loss.
2. **Backward Rollback**: Execute rollback scripts to confirm the database safely reverts to the previous version in the event of a botched deployment.
3. **Data Type Alteration**: Check that altering a column (e.g., `VARCHAR(50)` to `VARCHAR(255)` or `INT` to `BIGINT`) preserves existing data without truncation.

---

## Common Database Testing Tools

* **DBeaver / DataGrip**: Multi-platform database GUIs for running manual inspection queries, viewing ER diagrams, and inspecting triggers.
* **DbUnit**: A JUnit extension targeted at putting database into a known state between test runs.
* **Testcontainers**: Spawns lightweight, throwaway Docker containers of PostgreSQL, MySQL, or MongoDB directly in automated test pipelines.
* **Flyway / Liquibase**: Database versioning tools used to test migration scripts in CI/CD.

---

## SQA Interview Questions & Answers

### Q: What is the difference between Data Validation in the UI vs. Database Testing?
**Answer:**
UI validation only checks input formatting in the browser (e.g., HTML5 regex, required field warnings), which can be bypassed using API clients (Postman, curl) or browser DevTools. Database testing verifies that the database itself enforces constraints, transactions, and business logic regardless of how the request reached the database.

### Q: How do you verify database state after executing an API test?
**Answer:**
By performing direct assertions against the database. In an automated test suite (e.g., using Python pytest or Java REST Assured), after sending a `POST /api/orders` request, the test executes a `SELECT` query against the database to assert that the record exists with the exact status (`PENDING`), correct timestamp, foreign key references, and matching calculated totals.

---

## Key Takeaways

* Front-end checks are easily bypassed; database constraints are the ultimate line of defense for data integrity.
* ACID compliance is critical for transactional financial and inventory systems.
* QA engineers should write SQL queries to proactively audit referential integrity, null constraints, and migration scripts.

---

## Conclusion

Database testing is a crucial skill for any modern QA engineer. Ensuring schema consistency, transactional integrity, and absence of orphan data safeguards applications from insidious bugs that bypass API and UI layers.
