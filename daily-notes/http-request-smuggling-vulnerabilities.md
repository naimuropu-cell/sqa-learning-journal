# HTTP Request Smuggling Vulnerabilities & Discrepancy Testing

## 1. Overview of HTTP Request Smuggling (HRS)

In modern web application architectures, incoming HTTP requests pass through a chain of intermediaries (Reverse Proxies, Load Balancers, CDNs, Web Application Firewalls) before reaching backend application servers. To optimize throughput, these intermediary proxies reuse persistent backend TCP connections across multiple client requests.

**HTTP Request Smuggling** occurs when a frontend proxy and a backend server disagree on how to parse the boundary or length of an HTTP request. By sending a crafted request containing ambiguous parsing headers (such as both `Content-Length` and `Transfer-Encoding`), an attacker can smuggle an unparsed prefix into the backend connection pool. This smuggled request prepends itself to the next legitimate user's request, causing:
- Cache poisoning (serving malicious responses to all users)
- Credential and session token hijacking
- Bypassing frontend security controls and access filters
- Request reflection and execution of unauthorized administrative actions

```
[ Attacker Request ] ──► [ Frontend Proxy ] ═════════════════► [ Backend Server ]
                          Interprets CL: 4                      Interprets TE: chunked
                                                                Smuggled payload left
                                                                in TCP socket buffer!
                                                                         │
[ Victim Request ]   ──► [ Frontend Proxy ] ═════════════════►           ▼
                          Forwarded normally                   [ Prepend Smuggled Body ]
                                                               Executed as Attacker wants!
```

---

## 2. Core Request Smuggling Variants (CL.TE vs TE.CL)

The two classic specifications governing HTTP/1.1 request length are:
1. `Content-Length` (CL): Explicit byte count of the message body.
2. `Transfer-Encoding: chunked` (TE): Message body sent in consecutive hex-length chunks terminated with a zero chunk (`0\r\n\r\n`).

RFC 7230 dictates that if both headers are present, `Transfer-Encoding` must take precedence. However, discrepancies occur when systems fail to strictly enforce this or handle malformed variations:

| Variant | Frontend Behavior | Backend Behavior | Smuggling Mechanism |
| :--- | :--- | :--- | :--- |
| **CL.TE** | Uses `Content-Length` | Uses `Transfer-Encoding` | Frontend sends full byte length; backend finishes reading at the `0` chunk, leaving remaining bytes in the socket for the next request. |
| **TE.CL** | Uses `Transfer-Encoding` | Uses `Content-Length` | Frontend streams chunks; backend reads only $N$ bytes indicated by CL, leaving trailing chunk data in the socket. |
| **TE.TE** | Both support TE | Obfuscation bypass | Attacker obfuscates TE header (e.g., `Transfer-Encoding: xchunked` or `Transfer-encoding: [tab]chunked`); one server ignores it and falls back to CL. |
| **H2.CL / H2.TE** | HTTP/2 frontend downgrading | HTTP/1.1 backend | Frontend decodes HTTP/2 frames and translates to HTTP/1.1 with conflicting CL/TE headers. |

---

## 3. Discrepancy Payloads & Mechanics

### CL.TE Example Payload

```http
POST /search HTTP/1.1
Host: vulnerable-target.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 49
Transfer-Encoding: chunked

0

POST /admin/deleteUser?id=12 HTTP/1.1
Foo: x
```

**What Happens**:
1. Frontend sees `Content-Length: 49` $\to$ forwards the entire message over a persistent TCP connection.
2. Backend sees `Transfer-Encoding: chunked` $\to$ parses the `0\r\n\r\n` as the end of the request.
3. The remaining string `POST /admin/deleteUser?id=12 HTTP/1.1\r\nFoo: x` remains queued in the backend TCP socket buffer.
4. When the next user sends `GET /home HTTP/1.1`, the backend prepends the queued buffer to it:
   `POST /admin/deleteUser?id=12 HTTP/1.1\r\nFoo: xGET /home HTTP/1.1`
   resulting in unauthorized execution with the victim's session context!

---

## 4. Automated Detection & Testing with Python

QA security tests can probe for HTTP request smuggling by measuring timing delays caused by socket timeouts:

```python
import socket
import ssl
import time

def probe_cl_te_vulnerability(host, port=443):
    """
    Sends a CL.TE timing probe. If vulnerable, the backend server will wait
    for the remaining bytes indicated by Content-Length, causing an observable delay.
    """
    context = ssl.create_default_context()
    
    # Crafted CL.TE timing probe
    # Frontend CL: 4 bytes (reads '1\r\nZ\r\n')
    # Backend chunked reads chunk '1' (Z), then expects '0\r\n' which never arrives, causing timeout
    payload = (
        "POST / HTTP/1.1\r\n"
        f"Host: {host}\r\n"
        "User-Agent: SQA-Security-Probe/1.0\r\n"
        "Connection: keep-alive\r\n"
        "Content-Type: application/x-www-form-urlencoded\r\n"
        "Content-Length: 4\r\n"
        "Transfer-Encoding: chunked\r\n\r\n"
        "1\r\n"
        "Z\r\n"
        "Q\r\n\r\n"
    )

    start_time = time.time()
    try:
        with socket.create_connection((host, port), timeout=15) as sock:
            with context.wrap_socket(sock, server_hostname=host) as ssock:
                ssock.sendall(payload.encode('utf-8'))
                response = ssock.recv(1024)
                elapsed = time.time() - start_time

                if elapsed >= 10:
                    print(f"[!] POTENTIAL CL.TE VULNERABILITY DETECTED! Elapsed: {elapsed:.2f}s")
                else:
                    print(f"[-] Safe: Response returned in {elapsed:.2f}s")
    except socket.timeout:
        print("[!] Socket timed out after 15s — Strong indicator of Request Smuggling!")
    except Exception as e:
        print(f"[-] Probe encountered error: {e}")

if __name__ == "__main__":
    probe_cl_te_vulnerability("example.com")
```

---

## 5. Defensive Hardening & QA Audit Checklist

| Hardening Control | Purpose | Verification Method |
| :--- | :--- | :--- |
| **HTTP/2 End-to-End** | Replaces text-based CL/TE parsing with binary framing | Verify ALPN negotiation between proxy and backend |
| **Reject Ambiguous Requests** | Drop requests containing both `Content-Length` and `Transfer-Encoding` with 400 Bad Request | Send combined CL/TE probe; verify 400 response |
| **Normalize Headers in Proxy** | Frontend strips untrusted hop-by-hop headers and standardizes TE | Verify backend logs show sanitized headers |
| **Disable Backend Keep-Alive** | Forces a clean TCP connection per request (performance trade-off) | Verify TCP FIN packet per request on backend interface |

---

## 6. SQA Interview Questions & Answers

### Q1: Why does HTTP/2 mitigate classic HTTP/1.1 Request Smuggling?
> **Answer**:
> HTTP/1.1 uses text-based stream boundaries parsed via carriage returns (`\r\n`) and explicit headers (`Content-Length` or `chunked`), creating ambiguity when proxies parse differently. 
> HTTP/2 uses strict **binary framing** where message length is explicitly encoded in a 24-bit frame header field rather than header text, removing ambiguous length demarcation entirely. (Note: "H2.CL" can still occur if a frontend translates HTTP/2 back into malformed HTTP/1.1).

### Q2: What is the risk of cache poisoning through request smuggling?
> **Answer**:
> If an attacker smuggles a request that gets served by an intermediary caching layer (e.g., CDN or Varnish), the attacker's manipulated response (e.g., redirecting `/static/main.js` to an attacker-controlled script) can be cached by the CDN and served to thousands of legitimate users globally.

---

## 7. Key Takeaways & Best Practices

- Never allow backend servers to process both `Transfer-Encoding` and `Content-Length` on the same incoming connection.
- Modern reverse proxies (like NGINX, Envoy, HAProxy) must be kept updated to enforce strict RFC parsing.
- Include automated smuggling timing probes in security regression test pipelines.
