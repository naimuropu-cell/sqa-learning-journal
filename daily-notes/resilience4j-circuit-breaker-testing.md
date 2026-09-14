# Resilience4j Circuit Breaker & Fault Tolerance Testing

## 1. Introduction & Overview

In distributed microservices architectures, cascading failures represent one of the most severe operational risks. When an upstream downstream service experiences degradation, high latency, or total downtime, callers that block waiting for replies can exhaust thread pools, connections, and system memory, eventually causing an entire application cluster to crash.

**Resilience4j** is a lightweight fault tolerance library for Java (inspired by Netflix Hystrix) designed for functional programming. It provides higher-order functions (decorators) to enhance any functional interface with:
- **Circuit Breaker**: Stops executing downstream calls when failure rates exceed a defined threshold.
- **Rate Limiter**: Limits the rate of requests over a sliding time window.
- **Time Limiter**: Sets execution timeouts.
- **Retry**: Retries failed operations with backoff and jitter.
- **Bulkhead**: Limits concurrent executions to isolate failures.

As QA and Software Quality Engineers, testing fault tolerance requires orchestrating deliberate downstream faults, validating state transitions (CLOSED $\to$ OPEN $\to$ HALF_OPEN $\to$ CLOSED), verifying fallback behaviors, and measuring recovery performance.

```
       [ Client Request ]
               │
               ▼
   ┌───────────────────────┐
   │ Resilience4j Decorator│
   │  ┌─────────────────┐  │
   │  │ Circuit Breaker │  │
   │  └────────┬────────┘  │
   └───────────┼───────────┘
               │
      ┌────────┴────────┐
[State: CLOSED]    [State: OPEN]
      │                 │
      ▼                 ▼
[Target Microservice] [Fallback Handler / Cached Response]
```

---

## 2. Circuit Breaker State Machine & Transitions

The Resilience4j Circuit Breaker transitions between three primary states and two special states:

| State | Behavior | Request Execution | Transition Trigger |
| :--- | :--- | :--- | :--- |
| **CLOSED** | Normal operations | Calls are executed against the downstream service | Failure rate or slow call rate exceeds threshold $\to$ **OPEN** |
| **OPEN** | Circuit tripped / Short-circuited | Calls fail fast with `CallNotPermittedException` without touching downstream | Wait duration in open state expires $\to$ **HALF_OPEN** |
| **HALF_OPEN** | Trial / Recovery mode | Permits a limited number of test calls to evaluate downstream health | If failure rate < threshold $\to$ **CLOSED**; If failure rate $\ge$ threshold $\to$ **OPEN** |
| **DISABLED** | Circuit breaker forced off | All calls pass through; no monitoring or state changes | Manual operational intervention |
| **FORCED_OPEN** | Circuit breaker locked open | All calls immediately fail fast | Manual operational intervention during upstream maintenance |

```
                Failure rate > threshold
       ┌──────────────────────────────────────┐
       │                                      │
       ▼                                      │
  ┌─────────┐   Wait duration expires    ┌─────────┐
  │  OPEN   ├───────────────────────────►│HALF_OPEN│
  └─────────┘                            └────┬────┘
       ▲                                      │
       │        Failure rate > threshold      │
       └──────────────────────────────────────┤
                                              │ Success rate ok
                                              ▼
                                         ┌─────────┐
                                         │ CLOSED  │
                                         └─────────┘
```

---

## 3. Configuration Parameters to Test

When evaluating Resilience4j resilience configurations, QA engineers must verify the sliding window metrics:

```yaml
resilience4j.circuitbreaker:
  instances:
    paymentService:
      slidingWindowType: COUNT_BASED       # or TIME_BASED
      slidingWindowSize: 10                # number of calls recorded
      minimumNumberOfCalls: 5              # min calls required before calculating failure rate
      failureRateThreshold: 50.0           # percentage (50%)
      slowCallRateThreshold: 50.0          # percentage of calls slower than threshold
      slowCallDurationThreshold: 2000ms    # calls taking > 2s counted as slow
      waitDurationInOpenState: 10000ms     # time circuit remains OPEN before HALF_OPEN
      permittedNumberOfCallsInHalfOpenState: 3 # test calls allowed in HALF_OPEN
      automaticTransitionFromOpenToHalfOpenEnabled: true
```

---

## 4. Automated Integration Testing with WireMock & JUnit 5

Below is an automated test suite verifying state transitions using Spring Boot Test, Resilience4j, and WireMock.

```java
package com.example.sqa.resilience;

import com.github.tomakehurst.wiremock.junit5.WireMockTest;
import io.github.resilience4j.circuitbreaker.CallNotPermittedException;
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@WireMockTest(httpPort = 8089)
public class CircuitBreakerIntegrationTest {

    @Autowired
    private PaymentClient paymentClient;

    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;

    private CircuitBreaker circuitBreaker;

    @BeforeEach
    void setUp() {
        circuitBreaker = circuitBreakerRegistry.circuitBreaker("paymentService");
        circuitBreaker.reset();
    }

    @Test
    @DisplayName("Verify circuit transitions from CLOSED to OPEN on repeated 500 errors")
    void testCircuitBreakerTripsOnFailureRate() {
        // Stub downstream payment service to return 500 Internal Server Error
        stubFor(post(urlEqualTo("/api/v1/payments"))
                .willReturn(aResponse().withStatus(500)));

        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);

        // Send 10 failing calls to exceed failure threshold
        for (int i = 0; i < 10; i++) {
            try {
                paymentClient.processPayment("TXN_" + i, 100.0);
            } catch (Exception ignored) {
                // Expected downstream exception
            }
        }

        // Verify circuit breaker transitioned to OPEN
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.OPEN);

        // Immediate subsequent call must fail fast with CallNotPermittedException
        assertThatThrownBy(() -> paymentClient.processPayment("TXN_FAST_FAIL", 50.0))
                .isInstanceOf(CallNotPermittedException.class);

        // Downstream should NOT receive the fast-fail call (total calls received = 10)
        verify(10, postRequestedFor(urlEqualTo("/api/v1/payments")));
    }

    @Test
    @DisplayName("Verify fallback execution when circuit is OPEN")
    void testFallbackMechanismWhenCircuitOpen() {
        circuitBreaker.transitionToOpenState();

        PaymentResponse response = paymentClient.processPaymentWithFallback("TXN_FALLBACK", 75.0);

        assertThat(response.isFallback()).isTrue();
        assertThat(response.getStatus()).isEqualTo("QUEUED_FOR_RETRY");
    }
}
```

---

## 5. QA Chaos Engineering Test Matrix

| Test Scenario | Injection Method | Expected Resilience Behavior | Verification Metric |
| :--- | :--- | :--- | :--- |
| **High Latency Injection** | WireMock `fixedDelay(3000)` | Slow call rate exceeded $\to$ Circuit trips OPEN | Downstream latency metrics drop; fast failure occurs |
| **Cascading Database Outage** | 503 Service Unavailable | Fallback returns cached response or polite error | Error rate isolated to payment module; dashboard intact |
| **Flapping / Intermittent Network** | 50% packet drop via Toxiproxy | Retry policy attempts backoff; avoids tripping if under threshold | Retry count adheres to `maxAttempts: 3` |
| **Recovery Validation** | Restore 200 OK after `waitDuration` | Half-Open state executes permitted calls and closes | State returns to `CLOSED`; metrics reset |

---

## 6. SQA Interview Questions & Answers

### Q1: What is the difference between a Circuit Breaker and a Retry pattern?
> **Answer**: 
> A **Retry** pattern re-executes a failed call in the hope of immediate transient recovery (e.g., brief network blip). However, if the downstream service is overloaded or dead, retries compound the problem by hammering the failing system (the "thundering herd" problem).
> A **Circuit Breaker** detects persistent failures and proactively halts all outbound requests to protect both the caller and the downstream dependency, allowing the downstream system time to heal before testing recovery.

### Q2: How does Resilience4j calculate failure rates in a sliding window?
> **Answer**:
> Resilience4j uses either a **Count-Based** sliding window (e.g., the last $N$ calls) or a **Time-Based** sliding window (e.g., calls within the last $M$ seconds divided into buckets). The failure rate is calculated as:
> $$\text{Failure Rate} = \left(\frac{\text{Failed Calls} + \text{Slow Calls}}{\text{Total Measured Calls}}\right) \times 100$$
> The rate is only computed after `minimumNumberOfCalls` have been recorded to avoid tripping on a single transient startup failure.

---

## 7. Key Takeaways & Best Practices

- Always configure realistic `minimumNumberOfCalls` to prevent false trips during low-traffic periods.
- Pair Circuit Breakers with **Bulkheads** to ensure slow downstream dependencies do not starve the caller's primary worker thread pool.
- Implement explicit **Fallbacks** for critical user journeys (e.g., read-only cached catalog views or asynchronous event queuing).
- Monitor circuit state transitions in real time using Prometheus metrics (`resilience4j.circuitbreaker.state`).
