# Webhook Testing & Delivery Verification Guide for QA

## Introduction

In modern event-driven architectures, software systems do not merely expose APIs for inbound requests—they dispatch **Webhooks** (HTTP callbacks) to inform external third-party systems about events in real time.

For example:
* **Stripe** fires webhooks when a customer's subscription payment succeeds (`invoice.payment_succeeded`).
* **GitHub** dispatches webhooks when code is pushed to a repository (`push`).
* **Shopify** dispatches webhooks when an order is created (`orders/create`).

Testing webhooks is inherently asynchronous and involves two distinct perspectives: testing your application as a **Webhook Provider** (sending events reliably) and testing your application as a **Webhook Consumer** (receiving, parsing, and acknowledging incoming events securely).

---

## Webhook Architecture & Delivery Flow

```
┌─────────────────────────────┐             ┌─────────────────────────────┐
│      Webhook Producer       │             │      Webhook Consumer       │
│  (e.g., Payment Gateway)    │             │  (e.g., E-Commerce Server)  │
└──────────────┬──────────────┘             └──────────────┬──────────────┘
               │                                           │
               │ 1. Event Occurs (Payment Succeeded)       │
               │──────────────────────────────────────────►│
               │    POST /webhooks/stripe                  │
               │    Headers: X-Hub-Signature-256           │
               │                                           │
               │ 2. Immediate Fast Acknowledgment          │
               │◄──────────────────────────────────────────│
               │    HTTP 200 OK (Within 2 seconds)         │
               │                                           │
               │ 3. Asynchronous Background Processing     │
               │    (Fulfill order, send receipt email)    │
```

---

## What QA Must Test in Webhook Systems

```
┌─────────────────────────────────────────────────────────────┐
│                 Webhook QA Verification Areas               │
├─────────────────────┬───────────────────────────────────────┤
│ 1. HMAC Signatures  │ Cryptographic verification prevents   │
│                     │ attackers from spoofing fake events   │
├─────────────────────┼───────────────────────────────────────┤
│ 2. Idempotency      │ Handling duplicate webhook deliveries │
│                     │ without duplicate database records    │
├─────────────────────┼───────────────────────────────────────┤
│ 3. Delivery Retries │ Exponential backoff when consumer     │
│                     │ endpoint returns 5xx or times out     │
├─────────────────────┼───────────────────────────────────────┤
│ 4. Fast ACK (200 OK)│ Responding with HTTP 200 before       │
│                     │ heavy background processing begins    │
└─────────────────────┴───────────────────────────────────────┘
```

---

## 1. HMAC Signature Verification (Security Testing)

To guarantee that incoming webhooks genuinely originate from the trusted provider (and not a malicious actor spoofing requests), webhooks include an HMAC signature header (e.g., `X-Signature`, `Stripe-Signature`):

* **How it works**: The provider hashes the payload body with a shared secret key using `SHA-256`.
* **QA Test 1 (Valid Signature)**: Send request with accurate HMAC hash. Verify application accepts event (`HTTP 200`).
* **QA Test 2 (Tampered Payload)**: Send request with modified payload or incorrect hash. Verify application rejects request with **HTTP 401 Unauthorized** or **HTTP 403 Forbidden**.

---

## 2. Testing Webhook Retry Policies & Exponential Backoff

When the receiving server experiences temporary downtime (HTTP 500/503) or network timeouts:
* **The Rule**: A reliable producer must never drop events immediately. It must retry delivery using **Exponential Backoff** (e.g., retry after 1 min, 5 mins, 30 mins, 2 hours, 12 hours).
* **QA Test**: Mock the consumer endpoint to return `HTTP 500 Internal Server Error`. Verify that:
  1. The producer logs a failed delivery attempt.
  2. Successive retries occur at increasing intervals.
  3. If all retries fail, the event is routed to an admin Dead-Letter Queue (DLQ).

---

## 3. Webhook Idempotency Testing

Network hiccups frequently cause the provider to send the exact same webhook event twice.
* **The Failure**: If an e-commerce store processes `payment_succeeded` twice, it might ship two products or grant double subscription days.
* **QA Test**: Send the exact same webhook payload (with identical `event_id`) twice.
* **Expected Result**: Both requests return `HTTP 200 OK`, but the database executes order fulfillment only once.

---

## Practical Tooling for Webhook Testing

1. **ngrok**: Exposes local test servers to the public internet (`ngrok http 3000`), allowing real sandbox webhooks (from Stripe/PayPal) to hit local QA machines.
2. **Webhook.site**: Instantly generates unique disposable URLs to inspect incoming webhook payloads, headers, and timings.
3. **Postman**: Used to simulate incoming webhooks by generating HMAC SHA-256 signatures dynamically in the Pre-request Script.

### Pre-Request Script in Postman for HMAC SHA-256:
```javascript
const crypto = require('crypto-js');

const secret = "whsec_test_secret_key_123";
const payload = pm.request.body.raw;
const timestamp = Math.floor(Date.now() / 1000);

const signature = crypto.HmacSHA256(`${timestamp}.${payload}`, secret).toString();

pm.request.headers.add({
    key: 'X-Webhook-Signature',
    value: `t=${timestamp},v1=${signature}`
});
```

---

## SQA Interview Questions & Answers

### Q: Why should a webhook consumer return HTTP 200 before processing business logic?
**Answer:**
Webhook providers typically enforce strict response timeouts (e.g., 2 to 5 seconds). If the consumer attempts to process complex business logic (generating PDFs, charging credit cards, querying external databases) synchronously inside the webhook endpoint, the request will time out. The provider will mark the delivery as failed and trigger duplicate retries. The consumer should validate the signature, queue the event into a background job (Redis/RabbitMQ), and return `HTTP 200 OK` within milliseconds.

### Q: What is a Replay Attack on webhooks and how is it mitigated?
**Answer:**
A replay attack occurs when an eavesdropper intercepts a valid signed webhook request and resends it to the consumer to duplicate actions. It is mitigated by including a timestamp in the signature header. The consumer verifies that the timestamp is within a safe tolerance window (e.g., within the last 5 minutes) and discards expired timestamps.

---

## Key Takeaways

* Test webhooks for cryptographic signature validation (HMAC SHA-256) to block spoofing.
* Verify idempotency: duplicate events must produce identical outcomes without duplicating records.
* Ensure consumers acknowledge events immediately with HTTP 200, delegating heavy work to background workers.

---

## Conclusion

Webhooks form the asynchronous connective tissue of modern internet services. By methodically verifying signature verification, retry backoff algorithms, and idempotent processing, QA engineers ensure that real-time event workflows remain robust, secure, and loss-free.
