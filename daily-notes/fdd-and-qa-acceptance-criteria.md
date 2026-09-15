# Feature-Driven Development (FDD) & QA Acceptance Criteria Engineering

## 1. Introduction to Feature-Driven Development (FDD)

**Feature-Driven Development (FDD)** is an agile, iterative software development methodology centered around delivering tangible, client-valued functionality in short, two-week iterations. Originally conceived by Jeff De Luca and Peter Coad, FDD organizes work around a five-step lifecycle:

1. **Develop an Overall Model**: Domain modeling and high-level architectural walkthrough.
2. **Build a Feature List**: Decompose the domain into subject areas, business activities, and individual small features.
3. **Plan by Feature**: Sequence features by business priority, complexity, and dependencies.
4. **Design by Feature**: Detailed technical design and class/interface modeling.
5. **Build by Feature**: Code implementation, unit testing, inspection, and integration.

For Software Quality Engineers (SQA), FDD shifts the QA focus from late-stage manual defect hunting to upfront **Acceptance Criteria Engineering**, ensuring that every small feature has unambiguous, testable, and automated verification criteria before development begins.

```
┌────────────────────────────────────────────────────────┐
│            The 5 Core Activities of FDD                │
├───────────────┬───────────────┬────────────────────────┤
│ 1. Overall    │ 2. Feature    │ 3. Plan by Feature     │
│    Model      │    List       │                        │
└───────┬───────┴───────┬───────┴────────┬───────────────┘
        │               │                │
        ▼               ▼                ▼
┌────────────────────────────────────────────────────────┐
│ 4. Design by Feature  │ 5. Build by Feature            │
│  - Acceptance Criteria│  - TDD / Automated Verification│
│  - Boundary Modeling  │  - Continuous Integration Gate │
└───────────────────────┴────────────────────────────────┘
```

---

## 2. Standard Feature Naming Template in FDD

FDD enforces a strict syntactic pattern for naming features to guarantee clarity:

$$\langle\text{Action}\rangle\ \langle\text{Result}\rangle\ \text{by/for/of/to a(n)}\ \langle\text{Object}\rangle$$

Examples:
- `Calculate the total sales tax of an Order`
- `Authorize a credit card transaction for a Customer`
- `Send an email verification link to a New User`
- `Export audit logs to a CSV Report`

By constraining features to this granular format, features are small enough to be coded, tested, and shipped in **under two days to two weeks**.

---

## 3. Engineering Testable Acceptance Criteria

Vague acceptance criteria lead to defects, scope creep, and team misalignment. QA engineers apply the **Given-When-Then** (Gherkin) framework or tabular rule matrices to establish clear boundaries.

### Example Feature: `Calculate the total sales tax of an Order`

#### Bad Acceptance Criteria (Ambiguous & Untestable):
- "System should calculate sales tax properly."
- "Tax should depend on location and fast delivery."

#### Good Acceptance Criteria (Precise, Boundary-Tested & Executable):

```gherkin
Feature: Order Sales Tax Calculation

  Scenario: Standard non-exempt customer in single-jurisdiction state
    Given a customer with billing zip code "94103" (California, 8.5% combined rate)
    And an order containing:
      | Item               | Unit Price | Quantity | Tax Exempt |
      | Ergonomic Keyboard | $100.00    | 1        | False      |
      | Wireless Mouse     | $50.00     | 1        | False      |
    When the system calculates sales tax for the order
    Then the computed sales tax amount must be "$12.75"
    And the order total must equal "$162.75"

  Scenario: Tax exemption status applied to corporate customer
    Given a corporate customer with valid tax-exempt certificate "EX-9941"
    And an order containing items valued at "$500.00"
    When the system calculates sales tax
    Then the computed sales tax amount must be "$0.00"

  Scenario Outline: Boundary rounding rule verification
    Given an item with pre-tax price <item_price> and tax rate <rate>%
    When tax is calculated using half-up rounding to two decimal places
    Then the resulting tax must equal <expected_tax>

    Examples:
      | item_price | rate  | expected_tax |
      | $10.005    | 8.25  | $0.83        |
      | $10.004    | 8.25  | $0.83        |
      | $10.000    | 0.00  | $0.00        |
```

---

## 4. Acceptance Criteria Verification Checklist for QA

Before approving a feature specification into the sprint backlog, QA must run the **INVEST & SMART** audit:

| Quality Attribute | QA Verification Question | Defect Risk if Missing |
| :--- | :--- | :--- |
| **Specific** | Does the criterion specify exact numeric boundaries, status codes, and error messages? | Developer assumes defaults that mismatch user expectations. |
| **Measurable** | Can an automated assertion evaluate PASS/FAIL with zero human subjectivity? | Flaky or unverifiable test cases. |
| **Negative Paths** | Are invalid inputs, database timeouts, and permission denials explicitly defined? | Unhandled 500 exceptions in production. |
| **Performance SLA** | Is there a latency/throughput expectation defined (e.g., $p95 < 200\text{ms}$)? | Functional feature passes but degrades server performance. |
| **Data Integrity** | Are database schema cascades, transactions, and unique constraints detailed? | Orphaned database records or duplicate billing. |

---

## 5. Integrating Acceptance Criteria into Automated CI/CD

In modern Quality Engineering, Acceptance Criteria serve as the exact specification for executable test suites:

```typescript
// Automated Cucumber / Playwright Step Definitions
import { Given, When, Then } from '@cucumber/cucumber';
import { expect } from '@playwright/test';
import { TaxCalculatorService } from '../services/TaxCalculatorService';

let zipCode: string;
let items: Array<{ price: number; exempt: boolean }> = [];
let calculatedTax: number;

Given('a customer with billing zip code {string}', (zip: string) => {
    zipCode = zip;
});

Given('an item priced at {float} with tax exempt status {string}', (price: number, exemptStr: string) => {
    items.push({ price, exempt: exemptStr === 'true' });
});

When('the system calculates sales tax for the order', () => {
    const service = new TaxCalculatorService();
    calculatedTax = service.calculateTax(zipCode, items);
});

Then('the computed sales tax amount must equal {float}', (expectedTax: number) => {
    expect(calculatedTax).toBeCloseTo(expectedTax, 2);
});
```

---

## 6. SQA Interview Questions & Answers

### Q1: How does Feature-Driven Development (FDD) differ from Scrum?
> **Answer**:
> While Scrum is management- and sprint-oriented (organizing work into time-boxed sprints with daily standups and retrospectives without prescribing specific software engineering techniques), **FDD** is an engineering- and domain-centric methodology. FDD decomposes systems into tiny client-valued features using object-oriented domain modeling, strict feature naming conventions, class ownership, and formal design/code inspections.

### Q2: Why is the "Given-When-Then" format preferred over free-form acceptance criteria?
> **Answer**:
> Free-form text often leaves implicit assumptions, lacks negative error handling, and cannot be parsed programmatically. **Given-When-Then** structures criteria into preconditions (Given), actions (When), and observable postconditions (Then). This structure maps 1:1 to executable automated tests (e.g., Cucumber, SpecFlow, Behave) and eliminates ambiguity between Product Managers, Developers, and QA Engineers.

---

## 7. Key Takeaways & Best Practices

- Standardize feature descriptions using the FDD syntax: $\langle\text{Action}\rangle\ \langle\text{Result}\rangle\ \text{by/for/of}\ \langle\text{Object}\rangle$.
- Formulate acceptance criteria with explicit boundary values, negative error paths, and non-functional performance expectations.
- Use acceptance criteria directly as the blueprint for automated BDD and API regression test suites.
