# WebSocket Heartbeat (Ping/Pong), Dead Connection Detection & Auto-Reconnect Testing

## 1. The Challenge of "Half-Open" Dead WebSocket Connections

Unlike short-lived HTTP requests that terminate after each response, **WebSockets (RFC 6455)** establish persistent, bidirectional TCP connections.

In mobile environments, cellular networks, or through NAT firewalls, connections frequently enter a **"half-open" state**:
- A mobile device travels through a tunnel or switches from Wi-Fi to 4G/5G.
- The underlying TCP connection silently dies without sending a formal `FIN` or `RST` packet.
- **The Bug**: The client still believes it is connected; the server still holds the connection in memory; messages sent by either side vanish into a black hole without raising an error.

To detect and recover from dead sockets, robust WebSocket implementations use **Heartbeats (Ping/Pong control frames)** and **Exponential Backoff Reconnection**.

```
┌───────────────────────────┐                        ┌───────────────────────────┐
│     WebSocket Client      │                        │     WebSocket Server      │
└─────────────┬─────────────┘                        └─────────────┬─────────────┘
              │                                                    │
              │  1. Heartbeat Interval (Every 30s): Ping Frame     │
              ├───────────────────────────────────────────────────►│
              │  2. Immediate Pong Frame Response                  │
              │◄───────────────────────────────────────────────────┤
              │                                                    │
              │  (Simulated Network Loss / Half-Open Drop)         │
              │  ⚡⚡⚡ Network Severed ⚡⚡⚡                              │
              │                                                    │
              │  3. Ping Frame Sent ──► [Packet Dropped]           │
              │  4. Awaiting Pong Timeout (5s elapsed)             │
              │     ⚠️ Dead Connection Detected!                   │
              │     Force client.close() & Trigger Reconnect       │
              │                                                    │
              │  5. Reconnect Attempt 1: backoff = 1s + jitter     │
              │  6. Reconnect Attempt 2: backoff = 2s + jitter     │
              │  7. Connection Restored! Sync Missed State         │
              ├───────────────────────────────────────────────────►│
```

---

## 2. Testing Heartbeat & Ping/Pong Control Frames

Under RFC 6455:
- Opcode `0x9`: Ping frame (can carry application data).
- Opcode `0xA`: Pong frame (must echo the exact application data of the Ping).

### Key Failure Modes to Test:
1. **Missing Pong Timeout**: If the server fails to reply to a Ping within $T_{\text{timeout}}$ (e.g., 5 seconds), the client must cleanly terminate the socket and transition to `DISCONNECTED`.
2. **Zombie Server Detection**: If the client stops sending Pings, the server's idle connection reaper must close the socket to prevent file descriptor leaks.

---

## 3. Automated WebSocket Reconnection & Resiliency Test (Node.js)

Below is an automated test suite using `ws` simulating network drops, verifying ping/pong timeouts, and asserting exponential backoff with jitter:

```javascript
const WebSocket = require('ws');
const http = require('http');
const { expect } = require('chai');

describe('WebSocket Heartbeat & Reconnect Resilience Suite', () => {
  let server;
  let wss;
  const PORT = 8089;

  beforeEach((done) => {
    server = http.createServer();
    wss = new WebSocket.Server({ server });
    server.listen(PORT, done);
  });

  afterEach((done) => {
    wss.close(() => server.close(done));
  });

  it('should cleanly detect unresponsive server and trigger reconnect within timeout budget', (done) => {
    // 1. Setup server that intentionally ignores Ping frames (simulating frozen event loop)
    wss.on('connection', (ws) => {
      // Overwrite pong handler so server never responds to Pings
      ws.pong = () => {};
    });

    const client = new WebSocket(`ws://localhost:${PORT}`);
    let pingSent = false;
    let deadSocketDetected = false;

    client.on('open', () => {
      // Send Ping frame
      client.ping('heartbeat_check');
      pingSent = true;

      // Set watchdog timer: If no pong received within 1500ms, terminate
      const watchdog = setTimeout(() => {
        deadSocketDetected = true;
        client.terminate(); // Force close dead socket
      }, 1500);

      client.on('pong', () => {
        clearTimeout(watchdog);
      });
    });

    client.on('close', () => {
      expect(pingSent).to.be.true;
      expect(deadSocketDetected).to.be.true;
      done();
    });
  });

  it('should verify exponential backoff with randomized jitter on connection retries', () => {
    // Reconnect timing algorithm: Backoff = Min(Max_Backoff, Base * 2^attempt) + Jitter
    function calculateBackoff(attempt, base = 1000, max = 30000) {
      const exponential = Math.min(max, base * Math.pow(2, attempt));
      const jitter = Math.floor(Math.random() * (exponential * 0.2)); // 20% jitter
      return exponential + jitter;
    }

    const delays = [0, 1, 2, 3, 4].map((attempt) => calculateBackoff(attempt));

    // Assert strictly increasing backoff intervals
    for (let i = 1; i < delays.length; i++) {
      expect(delays[i]).to.be.greaterThan(delays[i - 1]);
    }
    // Attempt 0 should be ~1000ms (+ up to 200ms jitter)
    expect(delays[0]).to.be.within(1000, 1200);
  });
});
```

---

## 4. Testing State Re-Synchronization on Reconnection

A frequent functional defect in real-time chat, trading, and collaboration tools is **missed messages during the reconnect window**:
1. Client loses connection for 5 seconds.
2. Server broadcasts 3 new messages.
3. Client reconnects.
4. **The Bug**: Client never requests missing sequence IDs; UI displays a gap in messages.

### QA Verification Pattern:
- Client must send `last_received_sequence_id: 104` upon reconnecting.
- Server must replay messages `105`, `106`, and `107` before emitting new live events.

---

## 5. QA Verification Checklist

- [ ] **Ping/Pong Heartbeats**: Verify bidirectional heartbeats occur at configured intervals (e.g., every 25–30s).
- [ ] **Half-Open Detection**: Simulate abrupt network cutoff using Toxiproxy or firewall rules; verify client drops connection within timeout.
- [ ] **Exponential Backoff**: Validate that reconnect retries do not DDOS the server when the backend undergoes a rolling restart (Thundering Herd).
- [ ] **State Reconciliation**: Confirm that client synchronizes missing messages, offline queue items, and updated unread counts after reconnection.
