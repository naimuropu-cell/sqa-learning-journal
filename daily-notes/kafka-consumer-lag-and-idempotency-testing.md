# Kafka Consumer Lag, Rebalance & Idempotency Testing in Event-Driven Systems

## 1. The Challenges of Testing Event-Driven Architectures

In distributed, asynchronous systems using **Apache Kafka**, testing strategies must account for complexities absent in standard synchronous REST APIs:
- **Asynchronous Execution**: Producers send events without waiting for consumer processing; standard HTTP response codes (`200 OK`) only confirm that the broker acknowledged receipt, not that business logic executed successfully.
- **Partition Rebalancing**: Adding/removing consumer instances triggers rebalance events, potentially leading to delayed consumption or re-delivery.
- **Message Duplication**: Due to network retries and "at-least-once" delivery semantics, consumers inevitably receive identical messages more than once.
- **Out-of-Order Delivery**: Partition keys determine ordering. Misconfigured partition hashing can cause sequence violations (e.g., `ORDER_SHIPPED` processed before `ORDER_CREATED`).

```
┌────────────────────────────────────────────────────────────────────────┐
│                          Apache Kafka Cluster                          │
│                                                                        │
│   Topic: "order-events" [Partitions: 0, 1, 2]                          │
│   ├── Partition 0: [Msg 0] [Msg 1] [Msg 2] [Msg 3] ... [Msg N] (Latest)│
│   │                                       ▲            ▲               │
│   │                                Current Offset   Log End Offset     │
│   │                                       └──────┬─────┘               │
│   │                                        Consumer Lag                │
└──────────────────────────────────────────────┬─────────────────────────┘
                                               │
                        ┌──────────────────────┴──────────────────────┐
                        ▼                                             ▼
            ┌──────────────────────┐                      ┌──────────────────────┐
            │  Consumer Pod A      │                      │  Consumer Pod B      │
            │  - Message Processing│                      │  - Message Processing│
            │  - Idempotent DB Ops │                      │  - Idempotent DB Ops │
            └──────────────────────┘                      └──────────────────────┘
```

---

## 2. Consumer Lag Testing & Metrics

**Consumer Lag** is the difference between the latest produced message offset (*Log End Offset / LEO*) and the last committed offset by the consumer group (*Current Offset*):

$$\text{Consumer Lag} = \text{Log End Offset} - \text{Current Offset}$$

### Why High Consumer Lag is a Critical QA Finding:
1. **Data Freshness SLA Breaches**: Downstream services operate on stale data (e.g., stock levels, pricing updates).
2. **Memory Leaks & Backpressure**: Heavy lag often indicates unhandled exceptions, slow database locks, or garbage collection pauses in the consumer service.
3. **Cascading Rebalance Storms**: If consumer heartbeat threads are starved because message processing exceeds `max.poll.interval.ms`, Kafka assumes the consumer died and constantly triggers expensive rebalances.

### Command-Line Lag Verification

During performance testing, QA engineers monitor lag using Kafka's built-in script:

```bash
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe \
  --group order-processing-group
```

Output inspection:

```
GROUP                  TOPIC        PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID
order-processing-group order-events 0          14022           15500           1478            consumer-order-1
order-processing-group order-events 1          13890           13892           2               consumer-order-2
order-processing-group order-events 2          14105           16200           2095            consumer-order-3
```

---

## 3. Testing Message Deduplication & Idempotency

Because networks are inherently unreliable, Kafka provides **at-least-once delivery** by default. Consumers must be designed to be completely **idempotent**: processing the exact same event multiple times must yield the exact same system state without duplicate side-effects (e.g., double charging a credit card or creating multiple shipment orders).

### Common Deduplication Patterns Tested by QA

1. **Unique Message Key / Deduplication Table**:
   - The consumer records processed `event_id` or `idempotency_key` in a Redis cache or relational database table with a unique constraint within the same transaction.
2. **Conditional Database Upserts (`ON CONFLICT DO NOTHING`)**:
   - Insertion queries ignore or update existing records without creating duplicates.
3. **State Machine Transitions**:
   - An order in `PAID` status ignores subsequent duplicate `PAYMENT_RECEIVED` events.

### Automated Test: Verifying Idempotency Under Duplication

Below is a Mocha/Chai integration test using `kafkajs` that sends duplicate order payloads and verifies that only one database record is created and no secondary billing occurs:

```javascript
const { Kafka } = require('kafkajs');
const { expect } = require('chai');
const { pool } = require('../src/db');

const kafka = new Kafka({
  clientId: 'qa-test-runner',
  brokers: ['localhost:9092'],
});

describe('Kafka Consumer Idempotency & Deduplication Suite', () => {
  const producer = kafka.producer();
  const testOrderId = `ord_test_${Date.now()}`;
  const deduplicationKey = `idemp_${Date.now()}`;

  before(async () => {
    await producer.connect();
  });

  after(async () => {
    await producer.disconnect();
    // Cleanup test data
    await pool.query('DELETE FROM orders WHERE order_id = $1', [testOrderId]);
  });

  it('should process the initial event and record the transaction', async () => {
    const payload = {
      order_id: testOrderId,
      idempotency_key: deduplicationKey,
      amount: 199.99,
      timestamp: new Date().toISOString(),
    };

    // Send first event
    await producer.send({
      topic: 'order-events',
      messages: [{ key: testOrderId, value: JSON.stringify(payload) }],
    });

    // Wait for consumer processing
    await new Promise((resolve) => setTimeout(resolve, 3000));

    const result = await pool.query('SELECT * FROM orders WHERE order_id = $1', [testOrderId]);
    expect(result.rows).to.have.lengthOf(1);
    expect(result.rows[0].status).to.equal('CONFIRMED');
  });

  it('should ignore subsequent duplicate events with the same idempotency key', async () => {
    const duplicatePayload = {
      order_id: testOrderId,
      idempotency_key: deduplicationKey, // Same key
      amount: 199.99,
      timestamp: new Date().toISOString(),
    };

    // Resend the exact same event 3 times to simulate network retries
    for (let i = 0; i < 3; i++) {
      await producer.send({
        topic: 'order-events',
        messages: [{ key: testOrderId, value: JSON.stringify(duplicatePayload) }],
      });
    }

    // Wait for consumer processing
    await new Promise((resolve) => setTimeout(resolve, 4000));

    // Verify database still only contains one record
    const result = await pool.query('SELECT * FROM orders WHERE order_id = $1', [testOrderId]);
    expect(result.rows).to.have.lengthOf(1);

    // Verify audit logs or payment history does not show duplicate charges
    const auditLogs = await pool.query('SELECT * FROM order_audit_logs WHERE order_id = $1', [testOrderId]);
    expect(auditLogs.rows).to.have.lengthOf(1);
  });
});
```

---

## 4. Poison Pill & Dead Letter Queue (DLQ) Testing

A **Poison Pill** is a message that cannot be processed by the consumer due to malformed JSON, schema version mismatch, or bad data types. If not handled properly, a poison pill halts partition consumption entirely as the consumer repeatedly fails and crashes.

### QA Test Scenario: Poison Pill Recovery
1. **Inject Poison Pill**: Send an invalid payload (e.g., non-JSON raw bytes or missing required fields) into the topic.
2. **Verify Error Handling**: Consumer catches the deserialization exception without crashing the pod.
3. **Verify DLQ Routing**: The malformed message is routed to `order-events-dlq` along with error headers (`x-exception-message`, `x-original-topic`, `x-failed-timestamp`).
4. **Verify Partition Progress**: Subsequent valid messages continue to be processed without lag accumulation.

---

## 5. QA Verification Checklist

- [ ] **Consumer Lag Thresholds**: Alert triggered when lag exceeds 10,000 messages or remains stagnant for > 5 minutes.
- [ ] **Rebalance Testing**: Graceful commit of offsets during rolling deployments or Kubernetes pod scaling.
- [ ] **Idempotent Handling**: 100% duplicate message resistance across all critical financial and inventory topics.
- [ ] **Dead Letter Queue (DLQ)**: Verification that poison pills do not block queue processing and include accurate diagnostic metadata.
- [ ] **Ordering Guarantees**: Proper partitioning key usage to prevent out-of-order state mutations.
