# ETL and Data Pipeline Testing Guide for QA Engineers

## Introduction

In data-driven organizations, critical business decisions—such as revenue reporting, marketing spend allocation, customer churn prediction, and regulatory compliance—depend entirely on **Data Pipelines and ETL (Extract, Transform, Load)** processes.

Data is extracted from dozens of disparate operational sources (PostgreSQL, MongoDB, Salesforce, Google Ads), transformed into standardized business formats, and loaded into analytical data warehouses like **Snowflake**, **Google BigQuery**, **Databricks**, or **Amazon Redshift**.

A bug in an ETL pipeline does not merely cause an error modal; it corrupts executive financial dashboards, skews machine learning models, and triggers severe legal penalties. Testing ETL requires specialized data verification methodologies.

---

## The ETL Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 Operational Source Systems                  │
│  • Transaction DB (Postgres)   • CRM (Salesforce)           │
│  • Event Logs (Kafka)          • Third-Party APIs           │
└──────────────────────────────┬──────────────────────────────┘
                               │ 1. EXTRACT (Raw Ingestion)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                   Staging Data Lake / S3                    │
└──────────────────────────────┬──────────────────────────────┘
                               │ 2. TRANSFORM (Business Logic)
                               │    - Deduplication & Cleansing
                               │    - Currency conversion & Math
                               ▼
┌─────────────────────────────────────────────────────────────┐
│             Enterprise Data Warehouse (Snowflake)           │
│  • Dimension Tables (Users, Stores, Products)               │
│  • Fact Tables (Transactions, Orders, Revenue)              │
└──────────────────────────────┬──────────────────────────────┘
                               │ 3. LOAD (BI & Dashboards)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 Tableau / Looker / PowerBI                  │
└─────────────────────────────────────────────────────────────┘
```

---

## The Six Pillars of Data Quality Testing

```
┌─────────────────────────────────────────────────────────────┐
│                   The 6 Pillars of Data Quality             │
├─────────────────────┬───────────────────────────────────────┤
│ Dimension           │ Quality Verification Question         │
├─────────────────────┼───────────────────────────────────────┤
│ 1. Completeness     │ Did 100% of rows from source make it  │
│                     │ to the target warehouse without loss? │
├─────────────────────┼───────────────────────────────────────┤
│ 2. Accuracy         │ Are financial calculations and currency│
│                     │ conversions mathematically exact?     │
├─────────────────────┼───────────────────────────────────────┤
│ 3. Uniqueness       │ Are duplicate records eliminated? Are │
│                     │ primary/surrogate keys unique?        │
├─────────────────────┼───────────────────────────────────────┤
│ 4. Consistency      │ Does customer balance in the report   │
│                     │ match the live operational database?  │
├─────────────────────┼───────────────────────────────────────┤
│ 5. Validity         │ Do values conform to defined formats  │
│                     │ (valid emails, dates in ISO 8601)?    │
├─────────────────────┼───────────────────────────────────────┤
│ 6. Timeliness       │ Does the pipeline finish within its   │
│                     │ batch SLA (e.g., ready by 6:00 AM)?   │
└─────────────────────┴───────────────────────────────────────┘
```

---

## Practical SQL Reconciliation Queries for QA

### 1. Row Count Reconciliation (Completeness)
Ensure no records were dropped during extraction:
```sql
-- Source Count vs Target Count Check
SELECT 
    (SELECT COUNT(*) FROM source_db.orders WHERE order_date = '2026-09-13') AS source_count,
    (SELECT COUNT(*) FROM warehouse.fact_orders WHERE order_date = '2026-09-13') AS target_count;
```

### 2. Financial Aggregation Reconciliation (Accuracy)
Ensure transformations did not introduce rounding errors:
```sql
SELECT 
    ABS(s.source_sum - t.target_sum) AS discrepancy_amount
FROM 
    (SELECT SUM(amount) AS source_sum FROM source_db.payments WHERE status = 'SUCCESS') s,
    (SELECT SUM(amount) AS target_sum FROM warehouse.fact_payments WHERE status = 'SUCCESS') t;
-- Expected: discrepancy_amount = 0.00
```

### 3. Duplicate Detection on Business Keys (Uniqueness)
```sql
SELECT 
    customer_id, 
    transaction_id, 
    COUNT(*) AS occurrences
FROM warehouse.fact_transactions
GROUP BY customer_id, transaction_id
HAVING COUNT(*) > 1;
-- Expected: 0 rows returned
```

---

## Automated Data Testing Tools: Great Expectations & dbt

Modern QA teams do not manually execute SQL queries every morning. They automate data assertions using frameworks like **Great Expectations** or **dbt (data build tool)**:

### Sample Great Expectations Assertion:
```python
import great_expectations as ge

df = ge.read_csv("s3://warehouse-staging/fact_orders.csv")

# 1. Assert Column Uniqueness
df.expect_column_values_to_be_unique("order_id")

# 2. Assert Non-Nullability
df.expect_column_values_to_not_be_null("customer_id")

# 3. Assert Numeric Ranges
df.expect_column_values_to_be_between("order_total", min_value=0.01, max_value=100000.00)

# 4. Assert Allowed Values (Set Integrity)
df.expect_column_values_to_be_in_set("order_status", ["PENDING", "PAID", "REFUNDED", "CANCELED"])
```

---

## SQA Interview Questions & Answers

### Q: What is Schema Drift in ETL and how does QA test against it?
**Answer:**
Schema drift occurs when an upstream operational database changes its schema without notifying the data engineering team (e.g., adding a new column, renaming an attribute, or altering a column type from integer to string). If the ETL pipeline is not resilient, it crashes or ingests nulls. QA tests schema drift by injecting schema variations in staging and verifying that pipelines either handle new columns gracefully or trigger automated Slack/PagerDuty alerts without silent data corruption.

### Q: What is the difference between Initial Load and Incremental Load testing?
**Answer:**
* **Initial (Full) Load**: Testing the migration of the entire historical dataset from scratch into an empty warehouse, testing high-volume throughput, bulk indexing, and memory allocation.
* **Incremental Load (Delta / CDC)**: Testing the extraction of only newly created or modified records since the last batch run (Change Data Capture). QA tests that incremental runs capture new rows, update modified rows, and do not create duplicate entries.

---

## Key Takeaways

* ETL testing verifies data across the 6 pillars: Completeness, Accuracy, Uniqueness, Consistency, Validity, and Timeliness.
* Use SQL reconciliation queries to compare row counts and financial totals between source and target.
* Automate data pipeline quality gates using tools like Great Expectations and dbt tests.

---

## Conclusion

Data pipeline and ETL testing ensure that the foundation of enterprise decision-making remains untainted. By establishing rigorous data reconciliations, verifying schema contracts, and automating boundary checks, QA engineers ensure that analytics dashboards and machine learning models operate on trustworthy, pristine data.
