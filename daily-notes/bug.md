# Bug Life Cycle in Software Testing

## Introduction

Bug Life Cycle (also known as Defect Life Cycle) describes the different stages a defect goes through from the moment it is identified until it is closed.

A proper defect life cycle helps QA teams track, manage, and resolve issues efficiently.

---

## Stages of Bug Life Cycle

## 1. New

When a QA Engineer identifies a defect and reports it for the first time, the defect status becomes **New**.

### Example:

A tester finds that the login button is not working and creates a bug report.

**Status:** New

---

## 2. Assigned

The reported defect is reviewed and assigned to a developer for investigation and fixing.

### Example:

The QA Lead assigns the login issue to the responsible developer.

**Status:** Assigned

---

## 3. Open / In Progress

The developer starts analyzing the issue and works on fixing the defect.

### Example:

The developer checks the login code and identifies the root cause.

**Status:** Open

---

## 4. Fixed / Resolved

After making changes, the developer marks the defect as fixed and sends it back to QA for verification.

### Example:

The developer fixes the login validation issue.

**Status:** Fixed

---

## 5. Retest

The QA Engineer tests the application again to verify whether the defect has actually been resolved.

### Example:

The tester checks login functionality after receiving the new build.

**Status:** Retest

---

## 6. Verified

If the fix works correctly, QA confirms that the defect is resolved.

**Status:** Verified

---

## 7. Closed

After successful verification, the defect is closed.

**Status:** Closed

---

## 8. Reopened

If the defect still exists after the fix, QA reopens it and sends it back to the developer.

### Example:

The login issue still occurs after the developer's fix.

**Status:** Reopened

---

## 9. Rejected / Invalid

Sometimes a reported issue is not considered a valid defect.

Reasons:

* Working as expected
* Duplicate issue
* Not reproducible
* Requirement misunderstanding

**Status:** Rejected

---

## 10. Deferred

A defect may be postponed for a future release due to:

* Low priority
* Time constraints
* Business decisions

**Status:** Deferred

---

# Bug Life Cycle Flow

```
New
 ↓
Assigned
 ↓
Open
 ↓
Fixed
 ↓
Retest
 ↓
Verified
 ↓
Closed

If issue exists:
Retest → Reopened → Developer Fix
```

---

# Real-World Example

### Scenario:

A user cannot upload a profile picture.

### Flow:

1. QA reports the issue → **New**
2. Developer receives it → **Assigned**
3. Developer investigates → **Open**
4. Developer fixes upload issue → **Fixed**
5. QA tests again → **Retest**
6. Upload works successfully → **Verified**
7. Issue completed → **Closed**

---

# Importance of Bug Life Cycle

* Provides clear defect tracking
* Improves communication between teams
* Prevents missing issues
* Helps measure software quality
* Maintains testing history

---

# Interview Question

### Q: What happens if a defect fails during retesting?

**Answer:**

If a defect is not fixed successfully during retesting, the QA Engineer reopens the defect and assigns it back to the development team for further investigation.

---

# Key Takeaway

A Bug Life Cycle provides a structured process for managing defects from discovery to resolution.

**Report → Fix → Verify → Close**

---

## Conclusion

Understanding the Bug Life Cycle is essential for every QA Engineer because it helps teams manage defects effectively and deliver stable, high-quality software.
