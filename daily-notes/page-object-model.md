# Page Object Model (POM) Design Pattern in Test Automation

## Introduction

As test automation suites grow, maintaining automated scripts becomes one of the biggest challenges for QA automation engineers. When a web application's user interface changes (e.g., an ID or CSS selector is renamed), tests that reference those locators directly will break across dozens of test files.

The **Page Object Model (POM)** is an industry-standard architectural design pattern that solves this problem by creating an abstraction layer between test scripts and web page UI elements.

---

## What is Page Object Model?

Under POM:
* Each web page (or distinct UI component, such as a navbar, modal, or footer) is represented by a dedicated **Page Class**.
* The Page Class stores all **locators** (selectors) and **actions** (methods like `clickLogin()`, `enterCredentials()`, `searchProduct()`).
* The **Test Scripts** contain only the test assertions and business flows, calling methods from the Page Classes without knowing the underlying element locators.

```
       [ Test Scripts ] (Validates scenarios, assertions)
              │
              ▼ Calls methods
      [ Page Object Class ] (Encapsulates locators & actions)
              │
              ▼ Interacts with
       [ Web Application DOM ] (Browser UI)
```

---

## The Problem POM Solves

### Without POM (Tight Coupling & Code Duplication)
```typescript
// test_login.spec.ts
await page.locator('#txt-username').fill('admin');
await page.locator('#txt-password').fill('secret');
await page.locator('button.btn-primary').click();

// test_checkout.spec.ts (Repeats same locators!)
await page.locator('#txt-username').fill('admin');
await page.locator('#txt-password').fill('secret');
await page.locator('button.btn-primary').click();
```
*If `#txt-username` changes to `#user-login`, every single test file must be manually located and updated.*

### With POM (Separation of Concerns)
```typescript
// pages/LoginPage.ts
export class LoginPage {
  readonly page;
  readonly usernameInput;
  readonly passwordInput;
  readonly submitButton;

  constructor(page) {
    this.page = page;
    this.usernameInput = page.locator('#txt-username');
    this.passwordInput = page.locator('#txt-password');
    this.submitButton = page.locator('button.btn-primary');
  }

  async login(username, password) {
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

```typescript
// tests/login.spec.ts
const loginPage = new LoginPage(page);
await loginPage.login('admin', 'secret');
await expect(page).toHaveURL('/dashboard');
```
*If a locator changes, you update it **once** in `LoginPage.ts`, and all tests work instantly.*

---

## Standard Project Directory Structure

A clean automation project using POM typically organizes files as follows:

```
project-root/
│
├── pages/                  # Page Object Classes
│   ├── BasePage.ts         # Common reusable actions (click, type, wait)
│   ├── LoginPage.ts        # Login elements & actions
│   ├── DashboardPage.ts    # Dashboard navigation & widgets
│   └── CartPage.ts         # Checkout elements & actions
│
├── tests/                  # Test Specs & Assertions
│   ├── auth.spec.ts
│   ├── checkout.spec.ts
│   └── search.spec.ts
│
├── fixtures/               # Test data & environment setups
│   └── testData.json
│
└── playwright.config.ts    # Automation tool configuration
```

---

## Key Advantages of Page Object Model

1. **High Maintainability**: UI changes require editing locators in a single file rather than modifying multiple tests.
2. **Reusability**: Page methods (such as logging in or navigating to settings) can be reused across hundreds of test suites.
3. **Readability**: Test scripts read like human-readable user journeys rather than technical DOM manipulation code.
4. **Reduced Code Duplication**: Adheres strictly to the DRY (Don't Repeat Yourself) software engineering principle.

---

## Best Practices for Implementing POM

* **Never place assertions inside Page Classes**: Page classes should perform actions and return page state; assertions belong in the test files (`tests/*.spec.ts`).
* **Keep page methods focused**: Create small, focused actions (e.g., `fillSearchInput(query)`, `clickSearch()`) rather than gigantic multi-step functions.
* **Component-Based Objects**: For complex apps, create component objects for reusable elements like Navbars, Modals, or Tables rather than stuffing everything into a single monolithic page class.
* **Inherit from a BasePage**: Common utility methods like taking screenshots, handling custom waits, or getting page titles should reside in a shared `BasePage`.

---

## Interview Questions & Answers

### Q: Why shouldn't assertions be placed inside Page Objects?
**Answer:** 
Page Objects represent the state and behavior of the UI, not test outcomes. Keeping assertions inside test files ensures that page classes remain reusable across different scenarios (e.g., verifying successful login vs verifying failed login with invalid credentials).

### Q: What is the difference between Page Object Model (POM) and Page Factory?
**Answer:** 
POM is an architectural design pattern applicable to any framework (Playwright, Cypress, Selenium). Page Factory is a built-in Selenium library implementation (primarily in Java) that uses annotations like `@FindBy` and lazy element initialization.

---

## Key Takeaways

* POM separates the test execution logic from the UI structure.
* UI locators and user interactions live in Page Classes; assertions live in Test Files.
* Adopting POM is essential for scalable, enterprise-grade test automation suites.

---

## Conclusion

Mastering the Page Object Model is a core milestone for QA professionals transitioning into test automation. It enables testing teams to build clean, maintainable, and resilient automation frameworks that effortlessly adapt to changing application requirements.
