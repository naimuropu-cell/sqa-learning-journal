# WebRTC Peer-to-Peer Connection & Media Stream Testing

## 1. Overview of WebRTC Architecture

**WebRTC (Web Real-Time Communication)** is an open standard enabling real-time audio, video, and arbitrary peer-to-peer (P2P) data transfer directly between web browsers and mobile devices without requiring intermediate media servers.

Testing WebRTC applications introduces unique testing challenges compared to traditional client-server REST APIs:
- Connections rely on **Signaling** (usually via WebSockets) to exchange Session Description Protocol (SDP) offers, answers, and **ICE Candidates**.
- Media streams bypass web servers and route over UDP via **STUN** (Session Traversal Utilities for NAT) or relay via **TURN** (Traversal Using Relays around NAT).
- Network conditions (packet loss, jitter, latency) fluctuate dynamically, requiring adaptive bitrate streaming (e.g., VP8/VP9/AV1, Opus).

```
┌────────────────────────────────────────────────────────┐
│           Signaling Server (WebSocket / HTTP)          │
│           (Exchanges SDP Offers & ICE Candidates)      │
└──────────┬──────────────────────────────────┬──────────┘
           │                                  │
    SDP Offer / Ice                    SDP Answer / Ice
           │                                  │
           ▼                                  ▼
   ┌───────────────┐                  ┌───────────────┐
   │ Peer A (QA-1) │                  │ Peer B (QA-2) │
   └───────┬───────┘                  └───────┬───────┘
           │                                  │
           │  Direct P2P Media Stream (SRTP)  │
           └──────────────────────────────────┘
                      (or via TURN relay)
```

---

## 2. Core Protocol Stack & Testing Focus Areas

| Component | Protocol / Standard | Purpose | QA Test Verification |
| :--- | :--- | :--- | :--- |
| **Signaling** | WebSocket / JSON | Exchanging SDP metadata & network candidates | Signaling state transitions: `stable`, `have-local-offer`, `have-remote-offer` |
| **NAT Traversal** | STUN (RFC 5389) / TURN (RFC 5766) | Discovering public IP / relaying traffic across symmetric NATs | Verify fallback to TURN relay when direct UDP is blocked by firewall |
| **Security & Encryption** | DTLS (Datagram TLS) / SRTP | Cryptographic handshake and media payload encryption | Verify key exchange and handshake completion |
| **Data Channels** | SCTP over DTLS | Non-media bidirectional arbitrary data transfer | In-order vs. out-of-order delivery; throughput and backpressure |

---

## 3. Automated End-to-End Testing with Playwright

Playwright allows launching two isolated browser contexts to simulate both Peer A (Caller) and Peer B (Callee) on the same machine, using fake media streams to ensure reproducible video and audio feeds:

```typescript
import { test, expect, chromium } from '@playwright/test';

test.describe('WebRTC Peer Connection Automated Verification', () => {

    test('should establish bidirectional WebRTC call between two peers', async () => {
        // Launch browser with fake media streams enabled
        const browser = await chromium.launch({
            args: [
                '--use-fake-ui-for-media-stream',
                '--use-fake-device-for-media-stream',
                '--allow-file-access-from-files'
            ]
        });

        // Create two independent browser contexts (isolated cookies, storage, RTC states)
        const callerContext = await browser.newContext();
        const calleeContext = await browser.newContext();

        const callerPage = await callerContext.newPage();
        const calleePage = await calleeContext.newPage();

        // Step 1: Caller joins room
        await callerPage.goto('https://staging-meet.example.com/room/qa-automation-101');
        await callerPage.fill('#displayName', 'Alice-Caller');
        await callerPage.click('#btnJoin');

        // Step 2: Callee joins the same room
        await calleePage.goto('https://staging-meet.example.com/room/qa-automation-101');
        await calleePage.fill('#displayName', 'Bob-Callee');
        await calleePage.click('#btnJoin');

        // Step 3: Wait for ICE Connection State to reach 'connected' or 'completed'
        await callerPage.waitForFunction(async () => {
            const pc = (window as any).localPeerConnection as RTCPeerConnection;
            return pc && (pc.iceConnectionState === 'connected' || pc.iceConnectionState === 'completed');
        }, { timeout: 15000 });

        await calleePage.waitForFunction(async () => {
            const pc = (window as any).localPeerConnection as RTCPeerConnection;
            return pc && (pc.iceConnectionState === 'connected' || pc.iceConnectionState === 'completed');
        }, { timeout: 15000 });

        // Step 4: Verify Remote Video Elements are actively rendering frames
        const isRemotePlaying = await callerPage.evaluate(() => {
            const video = document.querySelector<HTMLVideoElement>('#remoteVideo');
            return video && video.currentTime > 0 && !video.paused && !video.ended && video.readyState >= 2;
        });

        expect(isRemotePlaying).toBe(true);

        await browser.close();
    });
});
```

---

## 4. Extracting WebRTC Diagnostics via `chrome://webrtc-internals` & `getStats()`

WebRTC exposes standard statistics through `RTCPeerConnection.getStats()`:

```javascript
// Function executed inside test to audit media quality
async function extractRTCQualityMetrics(peerConnection) {
    const stats = await peerConnection.getStats();
    let metrics = { packetsLost: 0, jitter: 0, roundTripTime: 0, framesDecoded: 0 };

    stats.forEach(report => {
        if (report.type === 'inbound-rtp' && report.kind === 'video') {
            metrics.packetsLost = report.packetsLost;
            metrics.jitter = report.jitter;
            metrics.framesDecoded = report.framesDecoded;
        }
        if (report.type === 'candidate-pair' && report.state === 'succeeded') {
            metrics.roundTripTime = report.currentRoundTripTime;
        }
    });

    return metrics;
}
```

---

## 5. Network Impairment Test Matrix

QA engineers should use tools like **Toxiproxy** or **Clumsy** to inject adverse conditions:

| Scenario | Network Condition | Expected WebRTC Behavior | Verification Criteria |
| :--- | :--- | :--- | :--- |
| **Packet Loss** | Inject 10% packet drop | Opus audio uses Forward Error Correction (FEC); video bitrate throttles | Audio intelligible; no crash |
| **High Latency** | Add 300ms round-trip delay | Jitter buffer expands | Video frames buffered smoothly without severe frame freeze |
| **STUN Blocked** | Firewall drops UDP port 3478 | Automatically switches from P2P to TURN relay on TCP/TLS port 443 | Call connects via relay candidate |
| **Network Switch** | Switch Wi-Fi to LTE | ICE Restart triggered automatically | Media resumes within 2–3 seconds |

---

## 6. SQA Interview Questions & Answers

### Q1: What is the difference between a STUN server and a TURN server?
> **Answer**:
> A **STUN (Session Traversal Utilities for NAT)** server allows a client behind a router/NAT to discover its public IP address and port mapping so direct peer-to-peer connection can be established. It does not relay media traffic.
> A **TURN (Traversal Using Relays around NAT)** server is a relay server used when symmetric NATs or restrictive corporate firewalls prevent direct peer-to-peer connections. Both peers send media to the TURN server, which relays it. TURN consumes substantial server bandwidth and compute.

### Q2: What are ICE candidates, and what are the three main types?
> **Answer**:
> **ICE (Interactive Connectivity Establishment) candidates** represent network connection options (IP, port, protocol) discovered by a peer. The three main types are:
> 1. **Host**: The device's local network interface IP (e.g., local Wi-Fi or LAN IP).
> 2. **Server Reflexive (srflx)**: The public IP address and port discovered via a STUN server.
> 3. **Relay (relay)**: The public IP and port allocated on a TURN relay server when direct P2P fails.

---

## 7. Key Takeaways & Best Practices

- Always test with `--use-fake-device-for-media-stream` in automated CI pipelines to avoid needing physical webcams and microphones.
- Monitor `RTCPeerConnection.getStats()` in performance benchmarks to quantify jitter, frame drops, and RTT.
- Ensure automated test suites explicitly validate fallback to TURN relay servers when UDP traffic is blocked.
