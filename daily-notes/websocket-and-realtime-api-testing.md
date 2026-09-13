# WebSocket and Real-Time API Testing Guide for QA

## Introduction

Standard REST APIs follow a synchronous **Request-Response** model over HTTP: the client requests data, the server responds, and the connection closes. However, modern applications like live trading dashboards, multiplayer games, collaborative editors (like Google Docs), and instant messaging apps require instant, two-way data streaming without the overhead of continuous HTTP polling.

**WebSockets (RFC 6455)** provide a persistent, bi-directional, full-duplex communication channel over a single TCP connection. Testing WebSockets requires specialized approaches and tools, as QA engineers must validate continuous streaming messages, connection drops, and frame schemas rather than isolated HTTP requests.

---

## HTTP REST vs. WebSocket Architecture

```
HTTP REST (Half-Duplex)             WebSocket (Full-Duplex)
┌────────┐          ┌────────┐      ┌────────┐          ┌────────┐
│ Client │          │ Server │      │ Client │          │ Server │
└───┬────┘          └───┬────┘      └───┬────┘          └───┬────┘
    │  Request (GET)    │               │  HTTP Upgrade Handshake│
    ├──────────────────►│               ├───────────────────────►│
    │  Response (200 OK)│               │  101 Switching Protocol│
    │◄──────────────────┤               │◄───────────────────────┤
    │  Connection Closes│               │  Persistent Connection │
    │                   │               │◄──────────────────────►│
    │  Request (POST)   │               │   Streaming Msg (Client)
    ├──────────────────►│               ├───────────────────────►│
    │  Response (201)   │               │   Streaming Msg (Server)
    │◄──────────────────┤               │◄───────────────────────┤
```

---

## The WebSocket Lifecycle

1. **The Handshake**: Client initiates an HTTP request with specific upgrade headers:
   * `Upgrade: websocket`
   * `Connection: Upgrade`
   * `Sec-WebSocket-Key: <random-base64-key>`
   * Server responds with HTTP `101 Switching Protocols`.
2. **Data Framing & Exchange**: Messages flow in both directions simultaneously as lightweight frames (UTF-8 text or binary).
3. **Heartbeats (Ping / Pong)**: Either peer sends a `Ping` frame; the recipient must respond with a `Pong` to verify the connection is still alive.
4. **Closure**: Either party sends a `Close` frame containing a status code (e.g., `1000 Normal Closure`, `1001 Going Away`, `1006 Abnormal Closure`).

---

## Key Test Scenarios for WebSockets

### 1. Connection & Authentication Testing
* Verify that unauthorized connection attempts without valid JWTs or session cookies are rejected with `401 Unauthorized` or `403 Forbidden`.
* Test connection attempts over insecure `ws://` vs secure `wss://`.

### 2. Message Schema & Business Logic Validation
* Send valid and invalid JSON payloads; verify the server processes valid commands and broadcasts expected updates to listening clients.
* Validate error payloads when invalid or malformed frames are transmitted.

### 3. Reconnection & Network Resiliency Testing
* Cut the network connection abruptly (e.g., toggling offline mode in DevTools); verify the client detects the drop and implements **Exponential Backoff** to reconnect.
* Verify message history synchronization upon successful reconnection (ensuring no messages were dropped).

### 4. High-Concurrency & Load Testing
* Test how the server behaves when 10,000 concurrent clients connect and listen to the same topic.
* Observe memory consumption and CPU utilization for connection leaks.

---

## Testing WebSockets with Postman

Postman natively supports WebSocket testing:
1. Click **New** ➔ **WebSocket Request**.
2. Enter the WebSocket URL (e.g., `wss://echo.websocket.events`).
3. Set query params, authentication headers, or sub-protocols.
4. Click **Connect** and observe the live message timeline showing incoming and outgoing frames.
5. Create JSON message templates and send them with one click.

---

## Practical Automation Example: WebSocket Testing in Node.js

Here is an automated test using the `ws` library with Jest/Mocha assertions:

```javascript
const WebSocket = require('ws');

describe('Real-Time Chat WebSocket API', () => {
  let wsClient;
  const WS_URL = 'wss://echo.websocket.events';

  afterEach((done) => {
    if (wsClient && wsClient.readyState === WebSocket.OPEN) {
      wsClient.close();
    }
    done();
  });

  test('Handshake establishes and echoes broadcast payload', (done) => {
    wsClient = new WebSocket(WS_URL);

    wsClient.on('open', () => {
      const payload = JSON.stringify({ event: 'chat_message', user: 'tester', text: 'Hello QA' });
      wsClient.send(payload);
    });

    wsClient.on('message', (data) => {
      const message = JSON.parse(data.toString());
      expect(message.event).toBe('chat_message');
      expect(message.text).toBe('Hello QA');
      expect(message.user).toBe('tester');
      done(); // Signal asynchronous test completion
    });

    wsClient.on('error', (err) => {
      done(err); // Fail test if socket encounters error
    });
  });
});
```

---

## SQA Interview Questions & Answers

### Q: What is the purpose of Ping and Pong frames in WebSockets?
**Answer:**
Because WebSockets maintain long-lived TCP connections, routers or firewalls may drop idle connections after a period of inactivity. Ping and Pong frames act as heartbeats to keep the connection alive, detect stale or "half-open" sockets, and measure connection latency between client and server.

### Q: How do you verify an application's behavior when a WebSocket connection drops?
**Answer:**
Simulate network interruption (via browser DevTools offline mode, proxy drops with Charles/Mitmproxy, or programmatic socket termination). Verify that:
1. The UI displays an appropriate "Reconnecting..." notification without crashing.
2. The client attempts reconnection with exponential backoff (e.g., 1s, 2s, 4s, 8s).
3. Once the connection is re-established, the client queries for missed messages or state updates to maintain data consistency.

---

## Key Takeaways

* WebSockets maintain persistent, bidirectional TCP connections ideal for real-time applications.
* Testing requires verifying handshakes, bidirectional payloads, heartbeats (ping/pong), and disconnection status codes.
* Always test reconnection and resilience under flaky network conditions.

---

## Conclusion

As real-time features become standard across modern applications, QA engineers must expand their skill set beyond traditional HTTP testing. Mastering WebSocket protocols and testing tools ensures that real-time data streaming remains fast, accurate, and resilient under adverse network conditions.
