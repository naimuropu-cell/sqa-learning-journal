# Exploratory Testing & Session-Based Test Management (SBTM)

## Introduction

Automated scripts and detailed test cases are outstanding at verifying known expectations (e.g., "does the login button submit with valid credentials?"). However, they only test what has already been anticipated by the author. They suffer from the **Pesticide Paradox**: running the exact same test repeatedly ceases to find new defects.

**Exploratory Testing** is an interactive, analytical testing approach where test design, test execution, and system learning happen simultaneously. It leverages human intuition, critical thinking, and technical curiosity to uncover unexpected, elusive bugs that automated scripts miss.

---

## Scripted Testing vs. Exploratory Testing

| Dimension | Scripted Testing | Exploratory Testing |
| :--- | :--- | :--- |
| **Philosophy** | Verification of predefined requirements. | Investigation, discovery, and learning. |
| **Test Design** | Written prior to execution. | Designed dynamically during execution. |
| **Flexibility** | Low; strictly follow step 1 to step N. | High; adapt path based on observed anomalies. |
| **Bug Discovery** | Finds regressions in established flows. | Finds deep edge cases, UI glitches, and race conditions. |
| **Best Used For** | Repetitive smoke and regression checks. | New features, complex workflows, and usability audits. |

---

## Session-Based Test Management (SBTM)

To avoid exploratory testing degenerating into aimless "ad-hoc clicking," James and Jonathan Bach created **Session-Based Test Management (SBTM)**. SBTM introduces structure, accountability, and measurable metrics to exploratory testing.

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Define Charter                                           │
│    "Explore [Target] With [Resources] To Discover [Goal]"   │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Timeboxed Session (60 - 90 mins)                         │
│    - Focus exclusively on charter mission                   │
│    - Record notes, bugs, and follow-up ideas                │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Quantify Session Breakdown (TBS Metrics)                 │
│    - Test & Report Time (%)                                 │
│    - Bug Investigation Time (%)                             │
│    - Setup & Blocker Time (%)                               │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Debrief with Lead / Team                                 │
│    - Review charter findings, bugs logged, and risks        │
└─────────────────────────────────────────────────────────────┘
```

---

## Testing Heuristics: SFDIPOT

When exploring a feature, QA engineers use mnemonics like **SFDIPOT** (San Francisco Depot) to spark creative testing angles:

* **S - Structure**: What physical files, code modules, or cloud infrastructure compose the system?
* **F - Function**: What is the application designed to do? (User actions, calculations, error states).
* **D - Data**: What inputs, media types, boundary quantities, and file formats does it accept?
* **I - Interfaces**: How does it interact with browsers, APIs, CLI, operating systems, or third parties?
* **P - Platform**: What operating systems, devices, viewports, or network conditions is it running on?
* **O - Operations**: How do real personas use it? (Power users, novices, malicious attackers).
* **T - Time**: How does time impact behavior? (Timeouts, timezones, leap years, concurrent access).

---

## Practical SBTM Charter & Session Sheet Example

Here is a real-world session report template used in professional QA workflows:

```markdown
# SBTM Session Report

* **Charter:** Explore the multi-currency checkout workflow using invalid credit cards and expired promotions to discover calculation anomalies and payment gateway handling bugs.
* **Tester:** Md. Naimur Rahman Apu
* **Date / Duration:** 2026-09-12 | 60 minutes (Timeboxed)
* **Environment:** Staging v2.14 (Chrome 128 / macOS)

### Time Breakdown (TBS Metrics)
* **Test & Report Time:** 70%
* **Bug Investigation:** 20%
* **Setup / Obstacles:** 10%

### Bugs Discovered
1. **BUG-4012 (High):** Currency symbol does not update when switching from USD to EUR on the payment confirmation modal, displaying "$100.00 EUR".
2. **BUG-4015 (Medium):** Rapid double-clicking on the "Apply Promo" button fires concurrent requests resulting in duplicate discount deduction.

### Observations & Follow-Up Ideas
* Note: Gateway timeout message is overly technical (`Error: 504 Gateway Timeout from Stripe Adapter`); recommend user-friendly wording.
* Idea: Explore cart abandonment email triggers when a payment fails mid-checkout.
```

---

## SQA Interview Questions & Answers

### Q: Is Exploratory Testing the same as Ad-Hoc Testing?
**Answer:**
No. **Ad-Hoc Testing** is informal, unstructured, and unguided testing without specific charters or formal debriefings. **Exploratory Testing** (especially when practiced via SBTM) is a rigorous, disciplined engineering discipline with structured charters, timeboxing, recorded heuristics, notes, and metrics presented during debriefs.

### Q: Why is Exploratory Testing essential even when you have 100% automated regression coverage?
**Answer:**
Automated tests only verify explicit, pre-scripted assertions. They cannot observe that an element rendered with an ugly overlap, that a notification banner blocked an essential button, or that a user journey felt slow and confusing. Exploratory testing utilizes human cognition to spot nuance, usability issues, and emergent system bugs.

---

## Key Takeaways

* Exploratory testing combines learning, test design, and execution in real time.
* SBTM brings structure and accountability to exploratory testing through charters, timeboxing, and TBS metrics.
* Mnemonics like SFDIPOT guide comprehensive coverage across data, platforms, and operational behaviors.

---

## Conclusion

Exploratory testing is not an alternative to test automation—it is its most potent partner. While automated pipelines catch regressions, exploratory testers push system boundaries, uncover critical defects, and ensure a superior user experience.
