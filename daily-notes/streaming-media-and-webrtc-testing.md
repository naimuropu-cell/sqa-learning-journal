# Streaming Media (HLS/DASH) & WebRTC Testing Guide for QA

## Introduction

From video-on-demand platforms (Netflix, YouTube) to live streaming (Twitch) and real-time interactive communications (Zoom, Google Meet, telehealth), media streaming represents over 70% of global internet traffic.

Testing streaming media goes beyond functional checks (e.g., "did the play button click?"). QA engineers must evaluate **Quality of Experience (QoE)**—verifying buffer health, adaptive bitrate switching, playback latency, and real-time peer-to-peer connection stability across erratic network conditions.

---

## Streaming Protocols Overview

```
┌─────────────────────────────────────────────────────────────┐
│                 Streaming Media Protocols                   │
├──────────────────────────────┬──────────────────────────────┤
│ Protocol                     │ Primary Use Case             │
├──────────────────────────────┼──────────────────────────────┤
│ 1. HLS (HTTP Live Streaming) │ Apple standard; iOS, Safari, │
│    (.m3u8 manifest + .ts)    │ VOD, broadcast streaming     │
│ 2. MPEG-DASH                 │ International ISO standard;  │
│    (.mpd manifest + .mp4)    │ Android, Web, Smart TVs      │
│ 3. WebRTC                    │ Sub-second real-time peer-   │
│    (SRTP / SCTP over UDP)    │ to-peer audio, video, calls  │
└──────────────────────────────┴──────────────────────────────┘
```

---

## Core Quality of Experience (QoE) Metrics

```
┌─────────────────────────────────────────────────────────────┐
│                  Key Streaming QoE Metrics                  │
├──────────────────────────────┬──────────────────────────────┤
│ Metric                       │ Target Value                 │
├──────────────────────────────┼──────────────────────────────┤
│ 1. Video Startup Time (VST)  │ < 1.5 seconds                │
│ 2. Rebuffering Ratio         │ < 0.5% of total watch time   │
│ 3. Average Bitrate           │ Consistent with bandwidth    │
│ 4. Dropped Frames Rate       │ < 1% during active playback  │
│ 5. Audio-Video Sync Drift    │ < 40ms offset                │
└──────────────────────────────┴──────────────────────────────┘
```

* **Video Startup Time (VST / Time to First Frame)**: Time elapsed between a user clicking "Play" and the first video frame rendering on screen.
* **Adaptive Bitrate (ABR) Switching**: Verifying that when network bandwidth throttles (e.g., dropping from 10 Mbps to 1 Mbps), the video player shifts down from 1080p to 480p smoothly without freezing.
* **Audio/Video Sync**: Verifying that audio track playback does not lead or lag video frames.

---

## WebRTC Testing & Real-Time Communication

WebSockets and HTTP stream data through a central server, introducing 1-3 seconds of latency. **WebRTC** delivers sub-500ms real-time audio/video directly between browser peers.

```
┌──────────┐                                      ┌──────────┐
│ Client A │ ──► [ Signaling Server (WS) ] ◄── │ Client B │
└────┬─────┘     (Exchanges SDP Offer/Answer)     └────┬─────┘
     │                                                 │
     │       STUN/TURN Server (NAT Traversal)          │
     │◄───────────────────────────────────────────────►│
     │                                                 │
     │                                                 │
     └─────────── Direct P2P Media Stream ────────────┘
                 (Encrypted SRTP over UDP)
```

### What QA Must Verify in WebRTC:
1. **STUN/TURN Traversal**: Test video calls behind strict enterprise corporate firewalls and symmetric NATs where direct P2P connections are blocked. Verify the call falls back to a TURN relay server.
2. **Network Impairment Simulation**: Using tools like Clumsy or Chrome DevTools Network throttling:
   * Inject 5% to 15% packet loss.
   * Verify audio clarity remains intact (Opus codec packet loss concealment).
   * Verify video resolution gracefully scales down rather than disconnecting.
3. **Diagnostics via `chrome://webrtc-internals`**:
   * Inspect live inbound/outbound RTP graphs, jitter buffers, round-trip time (RTT), and audio energy levels.

---

## Automated Media Testing with Playwright

Playwright can launch browsers with virtual camera/microphone streams to automate video testing in headless CI environments:

```typescript
import { test, expect } from '@playwright/test';

test.use({
  launchOptions: {
    args: [
      '--use-fake-ui-for-media-stream',     // Auto-accept camera/mic permissions
      '--use-fake-device-for-media-stream', // Feeds synthetic test video pattern
    ],
  },
});

test('Video call room establishes two-way media stream', async ({ context }) => {
  // Spawn two participants in separate browser tabs
  const caller = await context.newPage();
  const receiver = await context.newPage();

  await caller.goto('https://meet.example.com/room/qa-test-123');
  await receiver.goto('https://meet.example.com/room/qa-test-123');

  // Verify remote video element is rendering frames on both ends
  const remoteVideoCaller = caller.locator('#remote-video-stream');
  await expect(remoteVideoCaller).toBeVisible();

  // Validate video is actively playing (currentTime advancing)
  const isPlaying = await caller.evaluate(() => {
    const video = document.querySelector('#remote-video-stream') as HTMLVideoElement;
    return !video.paused && video.currentTime > 0 && !video.ended;
  });

  expect(isPlaying).toBe(true);
});
```

---

## SQA Interview Questions & Answers

### Q: What is Adaptive Bitrate Streaming (ABR) and how do you test it?
**Answer:**
ABR is a technique where video files are encoded into multiple quality profiles (e.g., 1080p, 720p, 480p, 360p) and sliced into short segments (2-6 seconds). A manifest file (`.m3u8` or `.mpd`) lists these profiles. During playback, the video player monitors network speed and dynamically requests higher or lower resolution segments. QA tests ABR by simulating fluctuating network bandwidth using proxies (e.g., Charles, Fiddler) and verifying that resolution scales up and down smoothly without rebuffering freezes.

### Q: Why do WebRTC connections require a TURN server?
**Answer:**
Many users sit behind strict corporate firewalls or symmetric NAT routers that prevent direct peer-to-peer UDP packet traversal. A TURN (Traversal Using Relays around NAT) server acts as a cloud relay server that forwards encrypted audio and video packets between peers when direct P2P connections cannot be established.

---

## Key Takeaways

* Streaming media quality is evaluated via QoE metrics: Video Startup Time, Rebuffering Ratio, and A/V sync.
* Test ABR by throttling network bandwidth to observe seamless resolution shifts.
* Use fake media devices (`--use-fake-device-for-media-stream`) to automate WebRTC and video calls in CI pipelines.

---

## Conclusion

Testing streaming media and real-time WebRTC applications demands a sophisticated blend of protocol analysis, network impairment simulation, and automated browser stream verification. By tracking key QoE indicators, QA engineers guarantee exceptional playback and communication experiences across global network conditions.
