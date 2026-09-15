# HTTP Strict Transport Security (HSTS) & SSL/TLS Configuration Testing

## 1. Overview of Transport Layer Security (TLS) & HSTS

Securing web communications requires end-to-end transport layer encryption. Even if a web server supports HTTPS, attackers can execute Man-in-the-Middle (MITM) and SSL-stripping attacks (e.g., via `sslstrip`) if the initial HTTP-to-HTTPS redirect can be intercepted on unencrypted networks (public Wi-Fi).

**HTTP Strict Transport Security (HSTS)** (RFC 6797) is an HTTP response header that instructs web browsers to:
1. Automatically upgrade all insecure `http://` requests to `https://` client-side before any network packet is transmitted.
2. Refuse to let users click past certificate error warnings (no "Proceed anyway" bypass).

For QA and Security Engineers, testing SSL/TLS and HSTS involves auditing cipher suites, certificate validity, expiration warnings, forward secrecy, and HSTS preload readiness.

```
[ User types: example.com ] 
             │
      ┌──────┴──────┐
      ▼             ▼
[ Without HSTS ]  [ With HSTS Enabled ]
Sends HTTP GET    Browser checks local HSTS cache
Plaintext over    Rewrites internally to:
the wire!         https://example.com:443
(MITM Risk)       (Secure TLS Handshake directly)
```

---

## 2. HSTS Header Syntax & Directives

```http
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

| Directive | Description | Recommended Production Value |
| :--- | :--- | :--- |
| `max-age=<seconds>` | Time (in seconds) the browser remembers to enforce HTTPS | `31536000` (1 year) or `63072000` (2 years) |
| `includeSubDomains` | Applies strict HTTPS enforcement to all subdomains (e.g., `api.example.com`, `dev.example.com`) | Mandatory for full domain security |
| `preload` | Consents to inclusion in Google/Chrome/Firefox browser hardcoded HSTS Preload list | Mandatory for zero-day protection on first user visit |

---

## 3. Automated TLS & HSTS Testing with Bash & `testssl.sh`

In QA pipelines, **`testssl.sh`** or **`sslscan`** provides automated command-line auditing of SSL/TLS endpoints.

```bash
# Audit TLS protocols, cipher suites, and vulnerabilities
./testssl.sh --severity MEDIUM https://staging.example.com

# Quick verification of HSTS headers via cURL
curl -sI https://staging.example.com | grep -i "strict-transport-security"
```

### Python Automated Assertion Suite:

```python
import ssl
import socket
import datetime
import requests
import pytest

DOMAIN = "staging.example.com"
PORT = 443

def test_hsts_header_presence():
    response = requests.get(f"https://{DOMAIN}", timeout=5)
    hsts_header = response.headers.get("Strict-Transport-Security")

    assert hsts_header is not None, "HSTS header is missing from HTTPS response!"
    assert "max-age=" in hsts_header, "HSTS header missing max-age directive!"

    # Parse max-age value
    parts = [p.strip() for p in hsts_header.split(";")]
    max_age_part = next((p for p in parts if p.startswith("max-age=")), None)
    max_age = int(max_age_part.split("=")[1])

    # Assert at least 1 year (31536000 seconds)
    assert max_age >= 31536000, f"HSTS max-age is too short: {max_age}s (expected >= 31536000s)"
    assert "includesubdomains" in hsts_header.lower(), "includeSubDomains directive missing!"

def test_certificate_expiration_and_validity():
    context = ssl.create_default_context()
    with socket.create_connection((DOMAIN, PORT), timeout=5) as sock:
        with context.wrap_socket(sock, server_hostname=DOMAIN) as ssock:
            cert = ssock.getpeercert()

            # Check expiration date
            not_after_str = cert['notAfter']
            not_after = datetime.datetime.strptime(not_after_str, '%b %d %H:%M:%S %Y %Z')
            days_left = (not_after - datetime.datetime.utcnow()).days

            print(f"[*] Certificate valid for {days_left} more days (Expires: {not_after})")
            assert days_left > 14, f"Certificate expiring imminently! Only {days_left} days remaining."

            # Assert TLS Version is modern (TLS 1.2 or TLS 1.3)
            tls_version = ssock.version()
            assert tls_version in ["TLSv1.2", "TLSv1.3"], f"Insecure TLS protocol version: {tls_version}"

def test_http_to_https_automatic_redirect():
    # Verify plain HTTP immediately issues 301 Permanent Redirect to HTTPS
    response = requests.get(f"http://{DOMAIN}", allow_redirects=False, timeout=5)
    assert response.status_code in [301, 308], f"Expected 301/308 redirect, got {response.status_code}"
    assert response.headers.get("Location", "").startswith("https://"), "Redirect target is not HTTPS!"
```

---

## 4. Modern TLS Configuration Checklist

| Security Requirement | Pass Criteria | Insecure Finding |
| :--- | :--- | :--- |
| **Deprecated Protocols** | SSL 2.0, SSL 3.0, TLS 1.0, and TLS 1.1 disabled | Any connection accepted on TLS 1.0/1.1 |
| **Weak Ciphers** | RC4, 3DES, CBC-mode ciphers, and NULL ciphers disabled | Support for `DES-CBC3-SHA` |
| **Forward Secrecy (PFS)** | ECDHE (Elliptic Curve Diffie-Hellman Ephemeral) enabled | Static RSA key exchange used |
| **ALPN (Application-Layer Protocol Negotiation)**| Negotiates `h2` (HTTP/2) or `h3` (HTTP/3) | Falls back to HTTP/1.1 only |
| **HSTS Preload List** | Listed on `hstspreload.org` status check | First-time visits vulnerable to SSL stripping |

---

## 5. SQA Interview Questions & Answers

### Q1: What is "SSL Stripping," and how does HSTS prevent it?
> **Answer**:
> In an **SSL Stripping attack**, a user types `example.com` without specifying `https://`. An attacker in the middle intercepts the unencrypted HTTP request and proxies it to the backend server over HTTPS, but returns plain unencrypted HTTP responses to the user's browser. The user sees an unencrypted website without knowing the connection was downgraded.
> **HSTS** prevents this by instructing the browser's internal network stack to automatically rewrite any `http://example.com` request to `https://example.com` before the browser touches the physical network.

### Q2: What are the risks of deploying `includeSubDomains` in an HSTS header without auditing existing infrastructure?
> **Answer**:
> If an organization sets `includeSubDomains` on `example.com`, every internal, legacy, or developer subdomain (e.g., `internal-legacy.example.com`, `printer.example.com`, `vpn.example.com`) will be forced to connect over HTTPS. If any of those internal subdomains do not have valid SSL certificates configured, they will immediately become completely inaccessible to users with no browser bypass option.

---

## 6. Key Takeaways & Best Practices

- Validate HSTS headers in automated staging smoke tests before every major release.
- Ensure `max-age` is at least 1 year (`31536000`) and includes `includeSubDomains` once all subdomains support TLS.
- Monitor SSL certificate expiration dates using automated CI alerts when expiration is within 30 days.
- Ensure weak protocols (SSL 3.0, TLS 1.0, TLS 1.1) and obsolete ciphers (RC4, 3DES) are disabled at the load balancer or edge proxy.
