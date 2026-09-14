# Zero-Trust Security Architecture Testing Guide for QA

## Introduction

Traditional enterprise security operated on a "castle-and-moat" perimeter model: once a user or service passed through the external firewall or VPN into the corporate intranet, they were implicitly trusted with broad access to internal databases, APIs, and microservices.

Modern security has abandoned this model in favor of **Zero-Trust Security Architecture (ZTA)**, guided by a single core principle:
> *"Never Trust, Always Verify."*

In a Zero-Trust architecture, every single request—whether originating from an external public browser or an internal backend service in the same Kubernetes cluster—must be explicitly authenticated, authorized, and cryptographically encrypted.

QA engineers must adapt security testing strategies to validate micro-segmentation, mutual TLS (mTLS), and continuous identity verification.

---

## The Three Pillars of Zero-Trust Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 Zero-Trust Core Principles                  │
├─────────────────────┬───────────────────────────────────────┤
│ Principle           │ What QA Must Validate                │
├─────────────────────┼───────────────────────────────────────┤
│ 1. Explicit         │ Always authenticate and authorize     │
│    Verification     │ based on all available data points    │
├─────────────────────┼───────────────────────────────────────┤
│ 2. Least Privilege  │ Limit user and service access with    │
│    Access           │ Just-In-Time (JIT) and Just-Enough-   │
│                     │ Access (JEA) policies                 │
├─────────────────────┼───────────────────────────────────────┤
│ 3. Assume Breach    │ Minimize blast radius; encrypt all    │
│                     │ communications (mTLS) and log all     │
│                     │ sessions continuously                 │
└─────────────────────┴───────────────────────────────────────┘
```

---

## Mutual TLS (mTLS) Between Microservices

In traditional architectures, traffic between microservices is unencrypted plain HTTP. In Zero-Trust, services enforce **Mutual TLS (mTLS)**:

```
[ Service A (Client) ]                                [ Service B (Server) ]
          │                                                      │
          │ 1. Client initiates TLS handshake                    │
          ├─────────────────────────────────────────────────────►│
          │ 2. Server presents Certificate (Proves Identity)     │
          │◄─────────────────────────────────────────────────────┤
          │ 3. Client presents Client Certificate (mTLS!)        │
          ├─────────────────────────────────────────────────────►│
          │ 4. Encrypted, Authenticated Communication Established│
          │◄────────────────────────────────────────────────────►│
```

### QA Verification for mTLS:
* **Positive Test**: Call Service B using valid client certificate and private key. Verify `HTTP 200 OK`.
* **Negative Test (Missing Cert)**: Call Service B over plain HTTP or without a client certificate. Verify connection is rejected at the TLS handshake level (`SSL_ERROR_CERT_REQUIRED`).
* **Negative Test (Untrusted CA)**: Call Service B using a self-signed or unauthorized certificate. Verify rejection with `SSL_ALERT_BAD_CERTIFICATE`.

---

## Testing Least-Privilege & Micro-Segmentation

Micro-segmentation prevents an attacker who breaches a single low-level container from freely scanning the internal network.

### QA Security Test Scenarios:
1. **Network Policy Enforcement**: Log into a frontend container pod. Attempt to `curl` or `ping` the payment database directly on port 5432.
   * **Expected Result**: Network connection timed out or rejected (Kubernetes NetworkPolicy blocks direct frontend-to-database traffic).
2. **Service Account Tokens**: Verify that microservices utilize scoped service account tokens (e.g., Kubernetes SPIFFE/SPIRE identities) rather than static, permanent credentials.

---

## SQA Interview Questions & Answers

### Q: How does Zero-Trust change API testing compared to traditional network models?
**Answer:**
In traditional models, internal APIs behind a firewall often lacked authentication headers or rate limits. In a Zero-Trust architecture, internal microservice APIs require the exact same rigorous authentication, JWT validation, payload schema checks, and mTLS encryption as external public APIs. QA must test inter-service authentication tokens on every internal endpoint.

### Q: What is the difference between standard TLS and Mutual TLS (mTLS)?
**Answer:**
* **Standard TLS (One-Way)**: Only the server presents a certificate to prove its identity to the client (e.g., your browser verifying it is talking to `google.com`).
* **Mutual TLS (Two-Way)**: Both the server and the client present cryptographic certificates to authenticate each other's identity before any application data is exchanged, preventing unauthorized services from making requests.

---

## Key Takeaways

* Zero-Trust assumes that attackers are already inside the network; perimeter defense is insufficient.
* Test that all inter-service communication enforces Mutual TLS (mTLS).
* Validate micro-segmentation: ensure compromised services cannot communicate with unauthorized internal systems.

---

## Conclusion

Zero-Trust security transforms quality assurance from perimeter checks into continuous, pervasive identity validation. By testing mTLS handshakes, micro-segmentation policies, and scoped authorization tokens, QA engineers ensure that systems remain fortified against internal and external threats.
