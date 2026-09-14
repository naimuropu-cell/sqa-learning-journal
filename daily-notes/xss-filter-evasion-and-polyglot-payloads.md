# XSS Filter Evasion & Polyglot Payload Security Testing Guide

## Introduction

Many software engineering teams attempt to defend against Cross-Site Scripting (XSS) by implementing naive blacklist filters or regex replacement rules—such as stripping out the string `<script>` or removing the word `alert`.

Blacklists almost always fail. Web browsers are extraordinarily complex parsing engines supporting hundreds of HTML5 tags, SVG elements, event handlers, and character encodings. Attackers craft obfuscated payloads that easily bypass naive filters while still executing arbitrary JavaScript in the victim's browser.

**XSS Filter Evasion & Polyglot Testing** is an advanced security testing practice where QA engineers verify that input sanitization and output encoding frameworks remain resilient against sophisticated, obfuscated injection vectors.

---

## The Flaws of Naive Blacklist Sanitizers

```
┌─────────────────────────────────────────────────────────────┐
│                 Naive Filter vs. Evasion Vector             │
├─────────────────────┬───────────────────┬───────────────────┤
│ Naive Filter Rule   │ Attacker Evasion  │ Browser Execution │
├─────────────────────┼───────────────────┼───────────────────┤
│ 1. Strips "<script>"│ `<ScRiPt>`        │ Case-insensitive  │
│    case-sensitively │                   │ HTML parser runs it│
├─────────────────────┼───────────────────┼───────────────────┤
│ 2. Strips once      │ `<scr<script>ipt>`│ Inner tag stripped;│
│    (No recursion)   │                   │ outer forms tag!  │
├─────────────────────┼───────────────────┼───────────────────┤
│ 3. Blocks "<script>"│ `<img src=x       │ Event handler     │
│    entirely         │  onerror=alert(1)>│ fires immediately │
├─────────────────────┼───────────────────┼───────────────────┤
│ 4. Blocks spaces    │ `<img/src=x/      │ Slash treated as  │
│    (e.g., regex \s) │  onerror=alert(1)>│ valid separator   │
└─────────────────────┴───────────────────┴───────────────────┘
```

---

## Advanced XSS Evasion Techniques QA Must Test

### 1. JavaScript Pseudo-Protocols in Links & Iframes
If an application allows users to enter a website URL for their profile:
* **The Exploit**:
  ```html
  <a href="javascript:alert(document.cookie)">Click to visit website</a>
  ```
* **HTML Entity Encoding Variant**:
  ```html
  <a href="&#x6a;&#x61;&#x76;&#x61;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;:alert(1)">Click Me</a>
  ```
  Browsers automatically decode HTML entities inside `href` attributes *before* evaluating the protocol, executing the script!

### 2. SVG & MathML XML Vectors
Modern browsers support inline SVG and MathML markup directly inside HTML5:
```html
<svg><animate onbegin=alert(1) attributeName=x dur=1s>
<math><mtext><table><mglyph><style><!--</style><img src=x onerror=alert(1)>
```
These vectors completely bypass sanitizers that only check for standard HTML tags like `<p>` or `<div>`.

---

## What is an XSS Polyglot?

An **XSS Polyglot** is a single, masterfully crafted payload that simultaneously executes across multiple distinct contexts—whether injected into a raw HTML body, inside an HTML attribute, within an existing `<script>` block, or inside a URL parameter:

### The Classic Universal Polyglot Payload:
```html
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
```

When tested by QA, if this single payload triggers an `alert()` dialog in any component, an unmitigated XSS vulnerability is proven to exist!

---

## The True Fix: Context-Aware Encoding & DOMPurify

QA must verify that development teams do not write custom regex filters. Instead, applications must rely on proven industry frameworks:

1. **Client-Side Sanitization**: Using **DOMPurify** to sanitize rich HTML input:
   ```javascript
   const cleanHtml = DOMPurify.sanitize(userInput);
   ```
2. **Context-Aware Output Encoding**: Encoding data based on *where* it is rendered:
   * Inside HTML Body: `&lt;` and `&gt;`
   * Inside HTML Attribute: `&quot;` and `&#x27;`
   * Inside JavaScript Variable: Unicode hex encoding (`\u0022`)

---

## SQA Interview Questions & Answers

### Q: Why is sanitizing input on the server often insufficient without output encoding?
**Answer:**
Different browser rendering contexts interpret data differently. An input string that is completely benign inside an HTML body (e.g., `O'Connor`) can instantly break out of context and execute malicious code if rendered inside a JavaScript string literal (`var name = 'O'Connor';`). Security requires context-aware output encoding at the exact moment the data is rendered into the browser DOM.

### Q: What is the role of a Content Security Policy (CSP) in defending against filter evasion?
**Answer:**
Even if an attacker discovers a novel filter evasion payload that bypasses application sanitizers and injects an inline script, a strict Content Security Policy (e.g., `script-src 'self'`) instructs the browser engine to block all inline script execution (`<script>` or `onerror=`), neutralizing the injected payload and acting as an essential secondary defense-in-depth layer.

---

## Key Takeaways

* Naive blacklists (stripping `<script>`) are easily bypassed with event handlers, casing, and SVG tags.
* Use universal polyglot payloads to test complex input fields across multiple rendering contexts.
* Verify context-aware output encoding and certified sanitizers like DOMPurify.

---

## Conclusion

Filter evasion testing proves that simple keyword filtering is never enough for modern web security. By challenging input fields with obfuscated tags, SVG vectors, and polyglots, QA engineers guarantee that applications remain immune to sophisticated cross-site scripting exploits.
