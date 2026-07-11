# Pre-request Scripts in Postman

## Introduction

A **Pre-request Script** is JavaScript code that runs **before** an API request is sent.

It is used to prepare the request by generating dynamic values, setting variables, creating timestamps, or modifying headers automatically.

Pre-request Scripts help reduce manual work and make API testing more efficient.

---

## Why Use Pre-request Scripts?

Many APIs require dynamic data that changes with every request.

Examples include:

* Authentication tokens
* Current timestamp
* Random email address
* Unique username
* Request signature
* Custom headers

Instead of updating these values manually, a Pre-request Script generates them automatically.

---

## Where to Write a Pre-request Script?

In Postman:

```text
Request
   ↓
Pre-request Script Tab
   ↓
Write JavaScript Code
   ↓
Send Request
```

The script executes before the request is sent.

---

## Example 1: Generate a Random Number

```javascript
const randomNumber = Math.floor(Math.random() * 10000);

pm.environment.set("random_number", randomNumber);
```

Now you can use:

```text
{{random_number}}
```

inside your request.

---

## Example 2: Generate a Unique Email

```javascript
const email = "user" + Date.now() + "@example.com";

pm.environment.set("email", email);
```

Request Body:

```json
{
  "email": "{{email}}",
  "password": "Password123"
}
```

Each request sends a unique email address.

---

## Example 3: Store Current Timestamp

```javascript
pm.environment.set(
    "current_time",
    Date.now()
);
```

Useful for APIs that require timestamps.

---

## Example 4: Generate a Random Username

```javascript
const username =
    "user_" + Math.floor(Math.random() * 100000);

pm.environment.set("username", username);
```

Now use:

```text
{{username}}
```

---

## Real-World Example

Suppose you are testing a **User Registration API**.

The API does not allow duplicate email addresses.

Without a Pre-request Script:

* [user@test.com](mailto:user@test.com) ❌
* [user@test.com](mailto:user@test.com) ❌

The second request fails because the email already exists.

With a Pre-request Script:

* [user1720812345678@example.com](mailto:user1720812345678@example.com) ✅
* [user1720812354321@example.com](mailto:user1720812354321@example.com) ✅

Each request automatically generates a new email.

---

## Benefits

* Eliminates repetitive manual work
* Creates dynamic test data
* Reduces duplicate data issues
* Makes API tests reusable
* Supports automated testing workflows

---

## Common Use Cases

* User Registration APIs
* Login APIs
* JWT Authentication
* Payment APIs
* Timestamp Validation
* Data-Driven Testing

---

## Interview Questions

### Q: What is a Pre-request Script in Postman?

**Answer:**

A Pre-request Script is JavaScript code that runs before an API request is sent. It is used to generate dynamic values, set variables, and prepare the request automatically.

---

### Q: What is the difference between a Pre-request Script and a Test Script?

| Pre-request Script                   | Test Script                                                      |
| ------------------------------------ | ---------------------------------------------------------------- |
| Runs before the request              | Runs after the response                                          |
| Prepares request data                | Validates the response                                           |
| Creates variables and dynamic values | Verifies status codes, response body, headers, and response time |

---

## Key Takeaway

* **Pre-request Script = Before Request**
* **Test Script = After Response**

Using both together helps create powerful, automated API tests in Postman.

---

## Conclusion

Pre-request Scripts are an essential feature of Postman for creating dynamic and reusable API tests. Combined with Test Scripts, they enable QA engineers to build efficient, maintainable, and scalable API testing workflows.
