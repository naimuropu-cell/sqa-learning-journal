# React Component Testing with React Testing Library (RTL)

## Introduction

In modern frontend web applications, user interfaces are composed of reusable UI components (buttons, search inputs, modal dialogs, data tables).

While End-to-End (E2E) tests in Playwright verify the complete assembled system, they are slower and more expensive to execute. **Component Testing** sits directly in the middle of the Testing Pyramid: it isolates individual UI components to verify their behavior, rendering states, and accessibility without spinning up the entire backend.

**React Testing Library (RTL)** is the industry standard tool for testing React components, guided by a core philosophy:
> *"The more your tests resemble the way your software is used, the more confidence they can give you."*

---

## Testing User Behavior vs. Implementation Details

A common pitfall in frontend testing (common with older tools like Enzyme) was testing **implementation details**:
* Asserting on internal component state (`wrapper.state('isOpen')`).
* Asserting on internal function names or component class names.
* **The Problem**: When developers refactor code (e.g., converting class components to React Hooks) without altering user-facing behavior, implementation-detail tests break, resulting in high maintenance overhead.

**React Testing Library tests like a human user**:
* It queries elements by accessible role (`button`, `heading`) and visible text.
* It simulates clicks, typing, and tab keyboard navigation.

---

## RTL Query Priority Order

RTL provides multiple ways to locate elements, prioritized by accessibility:

```
┌─────────────────────────────────────────────────────────────┐
│                 RTL Locator Priority Guide                  │
├─────────────────────┬───────────────────────────────────────┤
│ 1. getByRole        │ Queries accessible role (button, link,│
│    (Preferred ⭐)   │ textbox, heading) with name option    │
├─────────────────────┼───────────────────────────────────────┤
│ 2. getByLabelText   │ Locates form controls via <label>     │
├─────────────────────┼───────────────────────────────────────┤
│ 3. getByPlaceholder │ Locates inputs via placeholder text   │
├─────────────────────┼───────────────────────────────────────┤
│ 4. getByText        │ Locates non-interactive visible text  │
├─────────────────────┼───────────────────────────────────────┤
│ 5. getByTestId      │ Last resort: data-testid="submit-btn" │
└─────────────────────┴───────────────────────────────────────┘
```

---

## `getBy` vs. `queryBy` vs. `findBy`

| Query Prefix | If Element Found | If Element NOT Found | Asynchronous? | Primary Use Case |
| :--- | :--- | :--- | :---: | :--- |
| **`getBy...`** | Returns element | Throws Error ❌ | No | Asserting an element is immediately present. |
| **`queryBy...`** | Returns element | Returns `null` ✅ | No | Asserting an element is NOT in the DOM (`toBeNull()`). |
| **`findBy...`** | Returns element | Throws Error (timeout)| Yes (Returns Promise) | Waiting for elements that appear after an async API call. |

---

## `userEvent` vs. `fireEvent`

* `fireEvent.click(button)`: Dispatches a synthetic, single DOM click event without firing companion events (hover, focus, mousedown, mouseup).
* `userEvent.click(button)`: Simulates real browser user behavior—moves the virtual mouse, hovers over the element, gains focus, presses mouse down, releases mouse up, and triggers click. **Always prefer `@testing-library/user-event`**.

---

## Practical Component Test Example

Here is a test suite verifying a `CouponApplier` component:

```typescript
import React from 'react';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { CouponApplier } from './CouponApplier';

describe('CouponApplier Component', () => {
  test('Renders input and applies discount on button click', async () => {
    const user = userEvent.setup();
    const handleApply = jest.fn();

    render(<CouponApplier onApply={handleApply} />);

    // 1. Locate elements using accessible roles
    const inputField = screen.getByRole('textbox', { name: /coupon code/i });
    const applyButton = screen.getByRole('button', { name: /apply discount/i });

    // 2. User interaction: type code and click submit
    await user.type(inputField, 'SAVE20');
    await user.click(applyButton);

    // 3. Assert callback was triggered with proper argument
    expect(handleApply).toHaveBeenCalledTimes(1);
    expect(handleApply).toHaveBeenCalledWith('SAVE20');
  });

  test('Shows error message when submitting empty coupon', async () => {
    const user = userEvent.setup();
    render(<CouponApplier onApply={jest.fn()} />);

    // Click without typing
    await user.click(screen.getByRole('button', { name: /apply discount/i }));

    // Assert error message appears asynchronously
    const errorMessage = await screen.findByText(/please enter a valid coupon code/i);
    expect(errorMessage).toBeInTheDocument();
  });
});
```

---

## SQA Interview Questions & Answers

### Q: Why should you use `getByRole` instead of `getByTestId`?
**Answer:**
`getByRole` tests the application from the perspective of real users and assistive technologies (screen readers). If a button cannot be located via `getByRole('button', { name: /submit/i })`, it means the button is either missing an accessible label or has been built with an inaccessible generic `<div>` tag. `getByTestId` bypasses accessibility and provides false confidence that an element is usable when it may actually be inaccessible.

### Q: When should you use `queryBy` instead of `getBy`?
**Answer:**
Use `queryBy` exclusively when asserting that an element is **absent** from the screen (e.g., `expect(screen.queryByText(/error/i)).toBeNull()`). Using `getBy` for this assertion causes the test to fail immediately with an unhandled exception before the assertion can evaluate.

---

## Key Takeaways

* Component testing provides fast, high-confidence feedback on isolated UI components.
* Prioritize queries that reflect user perception: `getByRole` and `getByLabelText`.
* Use `findBy` for asynchronous UI updates and `queryBy` for asserting element absence.

---

## Conclusion

React Testing Library encourages testing habits that mirror real-world user interactions. By focusing on accessibility roles and behavioral outcomes rather than internal state, QA engineers write component tests that are resilient, maintainable, and deeply protective of user experience.
