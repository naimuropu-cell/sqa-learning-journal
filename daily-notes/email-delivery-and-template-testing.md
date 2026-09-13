# Email Delivery & HTML Template Testing Guide for QA

## Introduction

Transactional emails (order confirmations, password resets, multi-factor authentication OTPs, account activation links) are critical communication channels in any web or mobile application.

However, email testing is uniquely notoriously challenging:
* **Fragmented Email Clients**: Unlike modern web browsers that follow standardized W3C specifications, email clients (such as Microsoft Outlook for Windows) render HTML using the **Microsoft Word engine**, lacking support for modern CSS flexbox, grid, and web fonts.
* **Deliverability Hazards**: Misconfigured DNS records (SPF, DKIM, DMARC) or spam-trigger words can cause legitimate transactional emails to land in spam folders or be rejected outright.

QA engineers must validate both the **visual rendering** of responsive email templates and the **functional delivery** of transactional messages.

---

## The Email Deliverability & Authentication Protocols

Before an email reaches a customer's inbox, the recipient's mail server evaluates three DNS authentication protocols:

```
┌─────────────────────────────────────────────────────────────┐
│                 Email Security & Deliverability             │
├─────────────────┬───────────────────────────────────────────┤
│ Protocol        │ Verification Purpose                      │
├─────────────────┼───────────────────────────────────────────┤
│ 1. SPF (Sender  │ DNS TXT record declaring which mail       │
│    Policy)      │ servers are authorized to send for domain │
├─────────────────┼───────────────────────────────────────────┤
│ 2. DKIM (Domain │ Cryptographic public-key signature        │
│    Keys)        │ attached to header ensuring message wasn't│
│                 │ tampered with in transit                  │
├─────────────────┼───────────────────────────────────────────┤
│ 3. DMARC        │ Enforces policy (none / quarantine /      │
│                 │ reject) if SPF or DKIM fails              │
└─────────────────┴───────────────────────────────────────────┘
```

---

## Key Testing Areas for HTML Email Templates

### 1. Cross-Client Layout Rendering
* **Outlook Windows (MS Word Engine)**: Verify that tables (`<table>`) are used for layout structure rather than `<div>`, `flex`, or `grid`.
* **Dark Mode Compatibility**: Test how email templates look in Apple Mail and Gmail Dark Mode. Ensure dark mode inversion does not turn black text invisible on dark backgrounds or invert transparent PNG logos into black blobs.
* **Image Fallbacks**: Many email clients block external images by default. Verify that:
  * Images include descriptive `alt` text.
  * Essential information (receipt totals, activation links) is formatted as text and HTML buttons, never baked into static images!

### 2. Dynamic Token Replacement (Handlebars / Mustache)
* **The Glitch**: The template contains `Hello {{user.firstName}}, your order total is ${{order.total}}`.
* **QA Test**: Test with empty or null user properties. Verify that missing first names render with a sensible fallback (`"Hello valued customer"`) rather than literal unrendered code (`"Hello {{user.firstName}}"`)!

### 3. Link & Unsubscribe Testing
* Verify all links use secure HTTPS and contain appropriate campaign UTM tags.
* Verify compliance with the **CAN-SPAM Act** and **GDPR**: marketing emails must contain a working one-click "Unsubscribe" link and valid physical business address.

---

## Automated Transactional Email Testing

Never use real Gmail or Outlook accounts in automated test suites; email providers will quickly rate-limit or block test accounts. Instead, use virtual mock SMTP servers like **MailHog** or **Mailosaur**.

### Automated OTP Verification Example with Playwright & MailHog

```typescript
import { test, expect } from '@playwright/test';

test('User receives password reset email and extracts OTP code', async ({ page, request }) => {
  // 1. Trigger Password Reset in Web App
  await page.goto('/forgot-password');
  await page.fill('#email', 'qa-tester@example.com');
  await page.click('#submit-btn');

  // 2. Poll MailHog REST API for incoming test email
  let resetEmail: any = null;
  await expect.poll(async () => {
    const res = await request.get('http://localhost:8025/api/v2/messages');
    const messages = (await res.json()).items;
    resetEmail = messages.find((m: any) => m.To[0].Mailbox === 'qa-tester');
    return resetEmail;
  }, {
    message: 'Expected MailHog to receive password reset email within 5 seconds',
    timeout: 5000,
  }).toBeDefined();

  // 3. Extract 6-digit OTP code using Regex
  const emailBody = resetEmail.Content.Body;
  const otpMatch = emailBody.match(/\b\d{6}\b/);
  const otpCode = otpMatch ? otpMatch[0] : '';
  expect(otpCode).toHaveLength(6);

  // 4. Enter extracted OTP into web UI to complete verification
  await page.fill('#otp-input', otpCode);
  await page.click('#verify-otp-btn');
  await expect(page.locator('.success-banner')).toContainText('Password reset verified');
});
```

---

## SQA Interview Questions & Answers

### Q: Why is testing HTML emails fundamentally different from testing web pages?
**Answer:**
Web pages run in modern browsers adhering to uniform standards (W3C, HTML5, CSS3). Email clients (Outlook, Gmail, Apple Mail, Yahoo) employ drastically different and outdated rendering engines. For instance, desktop Outlook renders emails using Microsoft Word's engine, which ignores modern CSS layout properties like Flexbox, CSS Grid, and custom fonts. Email templates must rely on nested `<table>` structures and inline styles.

### Q: How do you verify transactional email flows in CI/CD without sending real emails to real mailboxes?
**Answer:**
By routing application SMTP traffic to a mock SMTP capture tool like **MailHog**, **Mailtrap**, or **Mailosaur**. The application transmits standard emails over SMTP, but the mock server traps all messages in a local sandbox without delivering them over the internet. The automated test suite then calls the mock server's REST API to assert subject lines, parse HTML contents, and extract dynamic OTP verification links.

---

## Key Takeaways

* Email clients use varied rendering engines; test templates across desktop Outlook, Gmail, and Apple Mail.
* Always provide fallbacks for images and dynamic templating tokens (`{{user.name}}`).
* Use MailHog or Mailosaur in CI/CD pipelines to validate transactional emails and extract OTPs automatically.

---

## Conclusion

Transactional email reliability is vital for customer onboarding and security authentication. By testing responsive email rendering, DNS deliverability protocols, and automated OTP extraction, QA engineers guarantee seamless and dependable communication with end users.
