# Postman Environment Variables & Collections

## Introduction

While testing APIs, QA engineers often work with multiple environments such as Development, Testing, Staging, and Production.

Postman Environment Variables help testers store and reuse dynamic values like URLs, tokens, IDs, and user data without manually changing them in every request.

Collections help organize multiple API requests together and make API testing easier to manage.

---

# What are Postman Variables?

Variables are placeholders that store reusable values.

Instead of writing the same value repeatedly, we store it once and use it whenever needed.

### Example:

Without variable:

```
https://api.example.com/users
```

With variable:

```
{{base_url}}/users
```

Here:

```
{{base_url}}
```

is a variable.

---

# Types of Postman Variables

Postman provides different scopes of variables:

## 1. Global Variables

Available across all collections and environments.

Example:

```
{{username}}
```

Used when a value is needed everywhere.

---

## 2. Environment Variables

Used for a specific environment.

Example:

Development:

```
base_url = http://dev-api.com
```

Production:

```
base_url = https://api.com
```

Same request can work in different environments.

---

## 3. Collection Variables

Available only inside a specific collection.

Useful for project-based API testing.

---

## 4. Local Variables

Temporary variables available only during a single request execution.

---

# Creating an Environment

Steps:

1. Open Postman
2. Click Environment selector
3. Select "Create Environment"
4. Add variables

Example:

| Variable | Initial Value           |
| -------- | ----------------------- |
| base_url | https://api.example.com |
| token    | abc123                  |

Then use:

```
{{base_url}}/login
```

---

# Using Variables in API Requests

Example:

## Without Variable

```
https://jsonplaceholder.typicode.com/users
```

## With Variable

```
{{base_url}}/users
```

Benefits:

* Easy maintenance
* Less manual work
* Faster testing

---

# What is a Postman Collection?

A Collection is a group of related API requests stored together.

Example:

E-commerce API Collection:

```
E-commerce APIs
│
├── Register User
├── Login User
├── Get Products
├── Add To Cart
├── Checkout
└── Payment
```

---

# Why Use Collections?

Collections help QA engineers:

* Organize API requests
* Share test cases with team members
* Run multiple APIs together
* Perform regression testing
* Export and maintain API documentation

---

# Real Project Example

Suppose you are testing a Banking API.

Environment:

```
base_url = https://bank-api.com
token = generated_token
```

Collection:

```
Banking API Testing

├── Login API
├── Account Details API
├── Transfer Money API
├── Transaction History API
└── Logout API
```

Flow:

1. Login API generates token.
2. Token is saved in environment variable.
3. Other APIs use the token automatically.

---

# Saving Dynamic Token Automatically

Example:

Login API response:

```json
{
 "token": "abc123xyz"
}
```

Test Script:

```javascript
let response = pm.response.json();

pm.environment.set(
    "token",
    response.token
);
```

Now other requests can use:

```
Bearer {{token}}
```

---

# Interview Questions

### Q: Why do we use Environment Variables in Postman?

Answer:

"Environment Variables allow us to store and reuse dynamic values like base URLs, authentication tokens, and IDs. They help test APIs easily across different environments."

---

### Q: What is a Postman Collection?

Answer:

"A Postman Collection is a group of API requests organized together. It helps manage, execute, and share API test scenarios efficiently."

---

# Key Takeaway

Variables make API testing flexible, while Collections make API testing organized and reusable.

**Variables = Store reusable data**
**Collections = Organize API requests**

---

## Conclusion

Understanding Postman Variables and Collections is an important step toward professional API testing. These features allow QA engineers to create maintainable, scalable, and efficient API test suites.
