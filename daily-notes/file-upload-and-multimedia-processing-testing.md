# File Upload and Multimedia Processing Security Testing Guide

## Introduction

File upload functionality is a standard requirement across modern applications—enabling users to upload profile pictures, resume PDFs, invoices, and video clips.

However, file upload is simultaneously one of the most dangerous attack surfaces in web security. Allowing users to upload arbitrary binary files to a web server introduces severe risks:
* **Remote Code Execution (RCE)**: An attacker uploads an executable web shell script (`shell.php` or `exploit.jsp`) and navigates to it to take over the server.
* **Denial of Service (Zip Bombs)**: A 42-kilobyte ZIP archive that decompresses into 4.5 petabytes of unallocated zeroes, exhausting server disk and memory.
* **Stored XSS via SVG**: Uploading vector images containing embedded JavaScript scripts that execute in the browser of anyone viewing the image.

QA engineers must execute rigorous functional and security test suites around file uploads and media processing pipelines.

---

## The Comprehensive File Upload QA Matrix

```
┌─────────────────────────────────────────────────────────────┐
│                 File Upload Testing Vectors                 │
├─────────────────────┬───────────────────────────────────────┤
│ Testing Vector      │ Quality & Security Verification       │
├─────────────────────┼───────────────────────────────────────┤
│ 1. MIME / Magic Byte│ Verifying actual file signature bytes,│
│    Validation       │ not just naive `.jpg` file extension  │
├─────────────────────┼───────────────────────────────────────┤
│ 2. File Size Limits │ Exact boundary tests: 0 Byte (empty), │
│                     │ Max allowed, Max + 1 Byte             │
├─────────────────────┼───────────────────────────────────────┤
│ 3. SVG Sanitization │ Stripping <script> tags from SVGs     │
├─────────────────────┼───────────────────────────────────────┤
│ 4. EXIF Stripping   │ Removing GPS coordinates & device     │
│                     │ metadata from uploaded JPEGs          │
├─────────────────────┼───────────────────────────────────────┤
│ 5. Chunked Uploads  │ Resuming uploads after network drops  │
└─────────────────────┴───────────────────────────────────────┘
```

---

## 1. Magic Bytes vs. Extension Spoofing

A common developer mistake is verifying file types using the file name extension (`file.endsWith('.jpg')`) or the client-supplied `Content-Type: image/jpeg` header:
* **The Exploit**: An attacker renames a malicious PHP script to `exploit.php.jpg`.
* **The Defense**: The server must inspect the **Magic Bytes** (the first few bytes of binary header data). For instance:
  * JPEG: `FF D8 FF`
  * PNG: `89 50 4E 47 0D 0A 1A 0A`
  * PDF: `25 50 44 46` (`%PDF`)
* **QA Test**: Create a plain text file containing `malicious code`, rename it to `test.png`, and upload it. The server must reject it with **HTTP 415 Unsupported Media Type**.

---

## 2. Testing SVG Vector Files for Stored XSS

SVG (Scalable Vector Graphics) is an XML-based image format that can execute JavaScript:

```xml
<?xml version="1.0" standalone="no"?>
<svg xmlns="http://www.w3.org/2000/svg">
  <circle cx="50" cy="50" r="40" fill="red" />
  <script type="text/javascript">
    alert('XSS via Uploaded SVG Image!');
  </script>
</svg>
```
* **QA Test**: Upload the SVG above as an avatar. View the avatar image URL directly in a browser tab.
* **Expected Result**: The script must NOT execute. The server must either serve SVGs with `Content-Disposition: attachment` (forcing download), sanitize the XML with a library like DOMPurify, or re-encode it to a raster PNG.

---

## 3. EXIF Metadata Privacy Testing

When users take photos on smartphones, cameras automatically embed EXIF metadata inside the JPEG image:
* GPS Latitude and Longitude of the user's home
* Date and time of capture
* Camera serial number and phone model

**QA Test**:
1. Take a photo on a smartphone with Location Services enabled.
2. Upload the photo as a public product image.
3. Download the uploaded image from the staging server.
4. Run `exiftool image.jpg` in terminal.
5. **Assertion**: GPS coordinates and private hardware metadata must be completely stripped by the image processing pipeline before storage in S3/cloud buckets.

---

## Practical Test Automation: File Uploads in Playwright

```typescript
import { test, expect } from '@playwright/test';
import path from 'path';

test.describe('Profile Picture Upload Suite', () => {
  test('Uploading valid PNG updates avatar preview', async ({ page }) => {
    await page.goto('/profile/settings');

    const fileChooserPromise = page.waitForEvent('filechooser');
    await page.click('#upload-avatar-btn');
    const fileChooser = await fileChooserPromise;

    // Set valid test file
    await fileChooser.setFiles(path.join(__dirname, '../fixtures/valid-avatar.png'));

    await page.click('#save-profile-btn');
    await expect(page.locator('.avatar-preview')).toBeVisible();
  });

  test('Uploading file exceeding 5MB displays validation error', async ({ page }) => {
    await page.goto('/profile/settings');

    // Attach 6MB test file
    const uploadInput = page.locator('input[type="file"]');
    await uploadInput.setInputFiles(path.join(__dirname, '../fixtures/oversized-6mb.jpg'));

    // Assert error toast
    const errorToast = page.locator('.toast-error');
    await expect(errorToast).toHaveText('File size exceeds the 5MB limit.');
  });
});
```

---

## SQA Interview Questions & Answers

### Q: What is a "Zip Bomb" (Decompression Bomb) and how does QA test against it?
**Answer:**
A Zip Bomb is a tiny, highly compressed archive (e.g., 42 KB) that unpacks into gigabytes or petabytes of unallocated data. When a server attempts to decompress the file, it exhausts memory and disk space, crashing the server (Denial of Service). QA tests against decompression bombs by asserting that the file processor limits the maximum allowable uncompressed byte count and aborts decompression if the output exceeds the threshold (e.g., maximum 50 MB uncompressed).

### Q: Why should uploaded user files be stored on a separate domain (e.g., AWS S3 / CDN) rather than on the application web server?
**Answer:**
Storing and serving user files from a separate domain (or isolated cloud storage bucket like S3) prevents uploaded files from executing server-side code (eliminating Remote Code Execution). Furthermore, serving files from a separate origin isolates the application's sensitive cookies and session tokens from being stolen if an attacker manages to upload a file with an embedded client-side script.

---

## Key Takeaways

* Never validate file types based on file extensions alone; verify Magic Bytes in the binary header.
* Sanitize SVG images to prevent embedded Stored XSS attacks.
* Strip EXIF location metadata from uploaded images to protect user privacy.

---

## Conclusion

File upload mechanisms demand meticulous functional and security validation. By verifying MIME signatures, enforcing strict byte limits, stripping private metadata, and automating upload workflows, QA engineers shield applications from code execution, privacy leaks, and storage exhaustion.
