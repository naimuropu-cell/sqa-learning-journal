
# Severity vs Priority in Software Testing

## Introduction

In Software Testing, **Severity** and **Priority** are two important attributes used in defect management to determine the impact of a bug and how quickly it should be fixed.

Although they are related, they represent different aspects of a defect.

---

# What is Severity?

**Severity** defines how much impact a defect has on the functionality of an application.

It answers the question:

**"How badly does this defect affect the system?"**

Severity is usually decided by the **QA Engineer** based on the technical impact.

---

## Types of Severity

### 1. Critical Severity

The application or a major feature becomes unusable.

**Example:**

* Application crashes after login
* Payment system completely stops working

---

### 2. High Severity

A major functionality is affected, but a workaround may exist.

**Example:**

* User cannot reset password
* Checkout fails for all customers

---

### 3. Medium Severity

A feature works partially but has an issue.

**Example:**

* Search filter gives incorrect results
* Profile update has minor issues

---

### 4. Low Severity

Minor issues that do not affect core functionality.

**Example:**

* Alignment problem
* Typo in a message

---

# What is Priority?

**Priority** defines how quickly a defect should be fixed.

It answers the question:

**"How soon should this bug be resolved?"**

Priority is usually decided by the **Project Manager, Product Owner, or Business Team**.

---

## Types of Priority

### 1. High Priority

Needs immediate fixing.

**Example:**

* Login failure in a banking application
* Payment failure before a major release

---

### 2. Medium Priority

Should be fixed in the upcoming release.

**Example:**

* Incorrect notification message

---

### 3. Low Priority

Can be fixed later.

**Example:**

* Minor UI improvement

---

# Severity vs Priority

| Severity                    | Priority                                 |
| --------------------------- | ---------------------------------------- |
| Measures impact of a defect | Measures urgency of fixing               |
| Focuses on technical damage | Focuses on business importance           |
| Usually decided by QA       | Usually decided by Business/Product team |
| "How bad is it?"            | "How soon should it be fixed?"           |

---

# Real-World Examples

## Example 1

**Bug:** Company logo is incorrect on the homepage.

* Severity: Low
* Priority: High

Why?

The functionality is not affected, but the company brand image is important.

---

## Example 2

**Bug:** Application crashes when clicking a rarely used report feature.

* Severity: High
* Priority: Low

Why?

The impact is serious, but the feature is rarely used.

---

## Example 3

**Bug:** Payment is not working.

* Severity: Critical
* Priority: High

Why?

It affects revenue and customers immediately.

---

# Interview Question

### Q: What is the difference between Severity and Priority?

**Answer:**

Severity describes the impact of a defect on the system, while Priority describes how urgently the defect should be fixed.

---

# Key Takeaway

Severity = **Impact**

Priority = **Urgency**

A defect can have:

* High Severity + High Priority
* High Severity + Low Priority
* Low Severity + High Priority
* Low Severity + Low Priority

---

## Conclusion

Understanding Severity and Priority helps QA teams communicate defect importance effectively and ensures that critical issues are fixed at the right time.
