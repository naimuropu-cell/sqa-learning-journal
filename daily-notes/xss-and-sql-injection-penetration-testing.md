# Practical XSS and SQL Injection Security Testing Guide for QA

## Introduction

Injection attacks occur when untrusted user input is passed directly into an interpreter (a web browser engine, a SQL database engine, or an operating system shell) without proper sanitization, validation, or escaping.

Among web security vulnerabilities, **Cross-Site Scripting (XSS)** and **SQL Injection (SQLi)** have historically accounted for the vast majority of critical data breaches, account takeovers, and compliance penalties.

QA engineers must possess practical knowledge of how these attacks operate to design targeted test cases that uncover injection flaws before malicious actors can exploit them.

---

## Part 1: Cross-Site Scripting (XSS) Deep Dive

XSS occurs when an application includes untrusted data in a web page without proper validation or escaping, allowing an attacker to execute arbitrary JavaScript in the victim's browser.

```
┌─────────────────────────────────────────────────────────────┐
│                       Types of XSS                          │
├───────────────────┬─────────────────────┬───────────────────┤
│ 1. Stored XSS     │ 2. Reflected XSS    │ 3. DOM-Based XSS  │
│ (Persistent)      │ (Non-Persistent)    │ (Client-Side)     │
├───────────────────┼─────────────────────┼───────────────────┤
│ Payload is saved  │ Payload is bounced  │ Vulnerability is  │
│ in database (e.g.,│ off server in URL   │ strictly in client│
│ comments, profile)│ or search query     │ JavaScript code   │
│ Executes for every│ Executes only for   │ (e.g., innerHTML) │
│ visitor viewing it│ user clicking link  │ Never hits server │
└───────────────────┴─────────────────────┴───────────────────┘
```

### Common QA Test Payloads for XSS:
```html
<!-- Basic Alert Verification -->
<script>alert('XSS-QA-PROBE')</script>

<!-- Image Error Handler Bypass (Bypasses naive <script> filters) -->
<img src="invalid-image" onerror="alert(document.domain)">

<!-- SVG Vector (Common in modern rich-text editors) -->
<svg onload="alert('XSS')">

<!-- JavaScript URI in Links -->
<a href="javascript:alert('XSS')">Click to Claim Prize</a>
```

### Verification & Remediation:
* **Context-Aware Escaping**: Ensure characters like `<`, `>`, `"`, `'`, and `&` are rendered as HTML entities (`&lt;`, `&gt;`, `&quot;`).
* **Content Security Policy (CSP)**: Assert that HTTP response headers contain a strict `Content-Security-Policy` header restricting inline script execution (`script-src 'self'`).

---

## Part 2: SQL Injection (SQLi) Deep Dive

SQL Injection occurs when user input is concatenated directly into a dynamic SQL query string instead of using parameterized queries.

```
Vulnerable Code:
SELECT * FROM users WHERE username = '' OR '1'='1' --' AND password = 'password'
                                      ^^^^^^^^^^^^
                                      Always evaluates to TRUE!
```

### 1. In-Band (Classic) SQLi
* **Authentication Bypass**: Submitting `' OR '1'='1' --` in a login form's username field to authenticate as the first user in the database (usually admin).
* **UNION-Based Extraction**: Appending `UNION SELECT null, username, password_hash FROM admin_users --` to harvest database tables.

### 2. Blind / Time-Based SQLi
When an application displays no database error messages or query results, attackers verify SQL execution by forcing the database engine to pause:
```sql
-- PostgreSQL / MySQL Sleep Test
' OR pg_sleep(5); --
' OR SLEEP(5); --

-- SQL Server Wait Test
'; WAITFOR DELAY '0:0:5'; --
```
* **QA Test**: Send the payload via Postman. If the HTTP response takes exactly 5.0+ seconds to return, the query is vulnerable to SQL injection.

---

## Practical Security Test Case Template

| Test Case ID | Test Objective | Input / Payload | Expected Result |
| :--- | :--- | :--- | :--- |
| **SEC-XSS-01** | Test comment input field for Stored XSS | `<script>alert('XSS')</script>` | Input is sanitized or encoded as `&lt;script&gt;`; script does NOT execute in browser. |
| **SEC-XSS-02** | Test image onerror event handling | `<img src=x onerror=alert(1)>` | Browser displays broken image icon; alert dialog does not trigger. |
| **SEC-SQL-01** | Test login form against auth bypass | `' OR '1'='1' /*` | Login fails with "Invalid credentials" error; no authentication bypass occurs. |
| **SEC-SQL-02** | Test query parameter for time-based SQLi | `?category=electronics' AND SLEEP(5)--` | Response returns within standard latency (<200ms); no server delay injected. |

---

## SQA Interview Questions & Answers

### Q: What is the difference between Stored XSS and Reflected XSS?
**Answer:**
In **Stored (Persistent) XSS**, the malicious payload is stored directly in the database (e.g., inside a product review or user bio) and executes every time any user visits that page. In **Reflected XSS**, the payload is not stored; it is embedded within a crafted URL parameter (e.g., `?search=<script>...`) and only executes when an unsuspecting user clicks the malicious link.

### Q: How are SQL Injections completely prevented in modern software?
**Answer:**
SQL injections are eradicated through the mandatory use of **Parameterized Queries (Prepared Statements)** or modern Object-Relational Mappers (ORMs like Prisma, Hibernate, or Entity Framework). Parameterized queries treat user input strictly as literal data rather than executable SQL code, preventing input from altering the query structure regardless of what special characters are entered.

---

## Key Takeaways

* Never trust user input; validate on the server and encode output for the browser.
* Test for XSS across input fields, profile settings, and URL query strings.
* Verify that all database operations utilize parameterized queries and prepared statements.

---

## Conclusion

Understanding injection mechanics enables QA engineers to design targeted security test suites that expose dangerous input vulnerabilities. By routinely fuzzing applications with XSS and SQLi payloads, QA teams prevent catastrophic data leaks and protect corporate integrity.
