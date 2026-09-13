# State Transition Testing & Finite State Machines (FSM) Guide

## Introduction

Many software systems do not simply produce outputs based on inputs alone; their behavior is fundamentally dictated by their **current state** and their **historical sequence of events**.

For example, clicking the "Cancel Order" button might succeed if an order is in the `PENDING` state, but must be strictly blocked if the order is already `SHIPPED`. Similarly, entering an incorrect password once shows an error, but entering it three times transitions the user account into the `LOCKED` state.

**State Transition Testing** is a black-box test design technique used when an application's behavior is modeled as a **Finite State Machine (FSM)**. It systematically verifies all valid and invalid transitions between system states.

---

## Core Concepts of State Transition Testing

* **State**: A condition or mode in which an application waits for one or more events (e.g., `Draft`, `Submitted`, `Approved`, `Rejected`).
* **Event (Trigger)**: An external input, user action, or timer that causes a change in state (e.g., clicking `Submit`, payment confirmation webhook, session timeout).
* **Transition**: The movement from one state to another triggered by an event.
* **Action**: An output or response produced as a result of the transition (e.g., sending a confirmation email, displaying an error modal).

---

## Real-World Example: E-Commerce Order Lifecycle

```
                 [ DRAFT ]
                     │
                     │ (Event: Submit Cart)
                     ▼
                 [ PENDING ]
                /          \
  (Event: Pay) /            \ (Event: Cancel)
              ▼              ▼
          [ PAID ]       [ CANCELED ]
             │
             │ (Event: Dispatch)
             ▼
        [ SHIPPED ]
             │
             │ (Event: Deliver)
             ▼
       [ DELIVERED ]
```

---

## Creating the State Transition Table

A State Transition Table maps every possible state against every possible event to expose **both valid and illegal transitions**:

| Current State | Event: Submit | Event: Pay | Event: Cancel | Event: Dispatch | Event: Deliver |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Draft** | ➔ **Pending** | Invalid ❌ | ➔ **Canceled** | Invalid ❌ | Invalid ❌ |
| **Pending** | Invalid ❌ | ➔ **Paid** | ➔ **Canceled** | Invalid ❌ | Invalid ❌ |
| **Paid** | Invalid ❌ | Invalid ❌ | ➔ **Refunded** | ➔ **Shipped** | Invalid ❌ |
| **Shipped** | Invalid ❌ | Invalid ❌ | Invalid ❌ | Invalid ❌ | ➔ **Delivered**|
| **Delivered**| Invalid ❌ | Invalid ❌ | Invalid ❌ | Invalid ❌ | Invalid ❌ |
| **Canceled** | Invalid ❌ | Invalid ❌ | Invalid ❌ | Invalid ❌ | Invalid ❌ |

> [!IMPORTANT]
> **QA Secret**: 80% of critical business bugs occur when the system permits an **Invalid Transition**—for example, a user canceling an order after it has already shipped, or submitting payment on an already canceled order!

---

## State Transition Coverage Criteria

QA teams use two primary coverage criteria:

1. **0-Switch Coverage (Branch Coverage)**:
   * Every valid individual transition is tested at least once ($A \rightarrow B$).
   * *Example*: `Draft` ➔ `Pending`, `Pending` ➔ `Paid`, `Paid` ➔ `Shipped`.
2. **1-Switch Coverage (Sequence Coverage)**:
   * Every sequence of two consecutive transitions is tested ($A \rightarrow B \rightarrow C$).
   * *Example*: `Draft` ➔ `Pending` ➔ `Canceled` vs. `Draft` ➔ `Pending` ➔ `Paid`.
   * Uncovers bugs where entering a state from Path 1 works, but entering from Path 2 causes corruption.

---

## Concrete Test Cases Generated from the State Table

| Test Case ID | Initial State | Event / Trigger | Expected Outcome / Next State | Category |
| :--- | :--- | :--- | :--- | :--- |
| **ST-01** | `Draft` | User submits cart | Order moves to `Pending`; payment screen opens | Valid Transition |
| **ST-02** | `Pending` | Payment succeeds | Order moves to `Paid`; invoice emailed | Valid Transition |
| **ST-03** | `Pending` | User clicks Cancel | Order moves to `Canceled`; inventory released | Valid Transition |
| **ST-04** | `Shipped` | User clicks Cancel | Error displayed: "Cannot cancel shipped order"; state remains `Shipped` | **Negative (Invalid)** |
| **ST-05** | `Canceled` | Gateway sends Pay webhook | Webhook rejected with HTTP 409 Conflict; state remains `Canceled` | **Negative (Invalid)** |

---

## SQA Interview Questions & Answers

### Q: Why is testing invalid state transitions so important?
**Answer:**
Applications often handle expected happy-path transitions properly, but fail to guard against illegal transitions. An illegal transition allows malicious users or concurrent API calls to bypass business rules (e.g., downloading digital goods before payment transitions to completed, or altering shipping addresses after a package has already left the warehouse). Testing invalid transitions guarantees state machine integrity.

### Q: What is the difference between 0-Switch and 1-Switch coverage?
**Answer:**
* **0-Switch Coverage** tests each single transition independently ($S_1 \rightarrow S_2$).
* **1-Switch Coverage** tests pairs of consecutive transitions ($S_1 \rightarrow S_2 \rightarrow S_3$). 1-Switch testing verifies that the application retains proper state memory and does not fail depending on which historical route was taken to reach a given state.

---

## Key Takeaways

* Use State Transition Testing whenever system behavior depends on past actions and current lifecycle states.
* Build State Transition Tables to systematically identify both valid transitions and illegal negative paths.
* Ensure backend APIs strictly validate state preconditions rather than relying on UI button states alone.

---

## Conclusion

State Transition Testing provides a structured, comprehensive approach for validating complex business workflows. By mapping all permissible and prohibited state paths, QA engineers ensure that systems remain deterministic, secure, and resilient throughout their entire lifecycle.
