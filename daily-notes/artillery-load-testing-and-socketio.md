# Real-Time Load Testing with Artillery & Socket.io

## 1. Overview of Real-Time Load Testing

Modern web and mobile applications rely heavily on bidirectional, full-duplex communication protocols such as **WebSockets** and **Socket.io** for live dashboards, chat rooms, collaborative editing, collaborative gaming, and live financial tickers.

Traditional HTTP load testing tools (like standard ApacheBench or simple cURL loops) cannot accurately simulate real-time workloads because:
1. WebSockets maintain persistent, long-lived TCP connections rather than transient request-response round-trips.
2. Messages are event-driven and can be initiated either by the client or pushed asynchronously by the server.
3. Memory and file descriptor limits on servers scale with concurrent open connections (C10K/C100K problem), not just requests-per-second (RPS).

**Artillery** is a modern, cloud-scale load and stress testing tool written in Node.js that natively supports HTTP, WebSockets, Socket.io, and serverless distributed test execution.

```
┌────────────────────────────────────────────────────────┐
│               Artillery Load Generator                 │
│  Simulating Virtual Users (VUs) with Socket.io Engine  │
└──────────┬─────────────────┬─────────────────┬─────────┘
           │                 │                 │
    WS Handshake      WS Handshake      WS Handshake
           │                 │                 │
           ▼                 ▼                 ▼
┌────────────────────────────────────────────────────────┐
│            Socket.io Cluster / Redis Adapter           │
│   - Connection Pool Exhaustion Monitoring              │
│   - Broadcast Latency & Ack Timing Metrics             │
│   - Event Deserialization Throughput                   │
└────────────────────────────────────────────────────────┘
```

---

## 2. Key Metrics for Real-Time WebSocket Testing

When executing load tests against Socket.io architectures, QA engineers must monitor:

| Metric Name | Target Description | Failure Indication |
| :--- | :--- | :--- |
| `vusers.created` / `vusers.completed` | Total virtual users spawned and closed | VUs getting stuck or timing out |
| `websocket.connect_time` | Latency of the HTTP Upgrade handshake | Handshake bottleneck, proxy queueing |
| `websocket.send_rate` | Messages emitted per second | CPU bottleneck on generator or client |
| `websocket.message_latency` | Round-trip time (RTT) for emitted message acknowledgements | Server event loop lag, Redis pub/sub backpressure |
| `socket.io.reconnections` | Frequency of client disconnects and reconnections | Network timeouts, server crashes, pod restarts |

---

## 3. Artillery Configuration Script (`artillery-socketio.yml`)

Below is an Artillery test scenario modeling a chat room load test with 500 concurrent virtual users emitting messages and awaiting server acknowledgements.

```yaml
config:
  target: "http://localhost:4000"
  engines:
    socketio-v3: {} # Supports Socket.io v3 / v4
  phases:
    - name: "Warm-up phase"
      duration: 30
      arrivalRate: 5
      rampTo: 20
    - name: "Sustained peak load"
      duration: 120
      arrivalRate: 25
      maxVusers: 500
  variables:
    chatRooms:
      - "qa-general"
      - "performance-triage"
      - "alerts-feed"

scenarios:
  - name: "Chat user join, emit and receive events"
    engine: socketio-v3
    flow:
      # Step 1: Connect and join room
      - emit:
          channel: "join-room"
          data:
            room: "{{ chatRooms }}"
            username: "tester_{{ $randomNumber(1000, 9999) }}"
      - think: 2

      # Step 2: Loop emitting simulated chat messages
      - loop:
          - emit:
              channel: "send-message"
              data:
                text: "Performance audit timestamp: {{ $timestamp }}"
              # Await server acknowledgement callback
              acknowledge:
                match:
                  json: "$.status"
                  value: "DELIVERED"
          - think: 3
        count: 5

      # Step 3: Graceful leave and disconnect
      - emit:
          channel: "leave-room"
      - think: 1
```

---

## 4. Custom Verification Hooks with JavaScript (`custom-hooks.js`)

Artillery allows importing custom JavaScript processors to validate payload signatures, measure custom metrics, or handle authentication tokens.

```javascript
module.exports = {
    setAuthHeaders,
    verifyMessageDelivery
};

function setAuthHeaders(context, events, done) {
    // Generate simulated HMAC or JWT for the virtual user
    const token = Buffer.from(`user_${context.vars.$randomNumber(1, 1000)}:secret`).toString('base64');
    context.vars.authToken = token;
    return done();
}

function verifyMessageDelivery(context, events, done) {
    // Custom metric tracking
    events.emit('counter', 'messages.verified_count', 1);
    return done();
}
```

---

## 5. Interpreting Artillery Execution Reports

Executing the load test via terminal:

```bash
npx artillery run artillery-socketio.yml --output report.json
npx artillery report report.json
```

Key summary fields to inspect:
```json
{
  "aggregate": {
    "counters": {
      "vusers.created": 500,
      "vusers.completed": 500,
      "vusers.failed": 0
    },
    "summaries": {
      "websocket.message_latency": {
        "min": 4.2,
        "max": 142.8,
        "median": 12.1,
        "p95": 28.6,
        "p99": 64.3
      }
    }
  }
}
```

If $p99$ message latency spikes significantly while server CPU is under 50%, investigate:
1. **Node.js Event Loop Delay**: Blocking synchronous calls in Socket.io event listeners.
2. **Redis Adapter Bottlenecks**: High serialization or network saturation across Redis pub/sub channels when broadcasting to large rooms.
3. **OS File Descriptors**: Verify `ulimit -n` on the server accommodates target connection pools.

---

## 6. SQA Interview Questions & Answers

### Q1: How does load testing WebSockets differ fundamentally from load testing REST APIs?
> **Answer**:
> REST APIs use stateless, short-lived HTTP connections that close or return to a keep-alive pool after response completion; the primary load metric is Requests Per Second (RPS). 
> WebSockets establish stateful, persistent TCP connections maintained over minutes or hours. Load testing WebSockets stresses operating system file descriptors, memory allocated per active socket buffer, event-loop responsiveness, and broadcast fan-out scaling (1 message delivered to $N$ subscribers).

### Q2: What is the C10K / C100K problem in the context of real-time QA testing?
> **Answer**:
> The C10K (10,000 concurrent connections) and C100K problem refers to the challenge of an operating system and server software handling thousands of concurrent open socket connections simultaneously. In QA testing, stress tests identify if thread-per-connection architectures deadlock or exhaust memory, and verify that non-blocking asynchronous event loops (like Epoll / Kqueue in Node.js or Netty) handle connection scaling cleanly.

---

## 7. Key Takeaways & Best Practices

- Always ramp up Virtual Users gradually during WebSocket tests to avoid SYN flood protection blocks.
- Configure socket acknowledgment callbacks to measure true round-trip time (RTT) latency.
- Monitor both client-side connection metrics and server-side socket memory footprint during extended soak tests.
