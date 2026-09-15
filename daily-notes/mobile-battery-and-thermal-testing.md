# Mobile Battery Consumption & Thermal Throttling Testing

## 1. Overview of Mobile Power & Thermal Performance

Modern mobile applications (iOS and Android) interact with complex device hardware: multicore CPUs, GPUs, 5G/Wi-Fi modems, GPS location chips, camera sensors, and neural processing units (NPUs). Inefficient coding practices can trigger **battery drain** and **thermal throttling**, leading to:
- Excessive device temperature causing OS-level CPU/GPU throttling (stuttering frames and dropped framerate).
- Operating system background kills (Android Foreground Service limits and iOS Background Execution limits).
- Negative app store reviews and uninstalls due to battery exhaustion.

As QA Engineers, evaluating battery and thermal performance requires quantifiable measurement tools, automated stress scenarios, and hardware profiling rather than subjective impressions.

```
┌────────────────────────────────────────────────────────┐
│               Hardware Energy Drain Triggers           │
├───────────────────┬───────────────────┬────────────────┤
│ CPU WakeLocks     │ Radio Wake-ups    │ Screen & GPU   │
│ Infinite loops /  │ Unbatched API     │ High frame     │
│ unreleased locks  │ polling (keeps    │ rerenders &    │
│ keep CPU at 100%  │ modem at full Rx) │ animations     │
└───────────────────┴───────────────────┴────────────────┘
```

---

## 2. Key Battery Draining Antipatterns

| Antipattern | Mechanism | Impact | Best Practice Alternative |
| :--- | :--- | :--- | :--- |
| **WakeLock Leaks** | App acquires `PARTIAL_WAKE_LOCK` and fails to release it in `finally` blocks | Device CPU cannot enter deep sleep (Doze mode) | Use `WorkManager` with system-managed job scheduling. |
| **Chatty Network Polling** | Polling backend API every 5 seconds over cellular radio | Mobile cellular modem transitions from low-power Idle to high-power Active state continuously | Use WebSockets, HTTP/2 Server Push, or Firebase Cloud Messaging (FCM) push notifications. |
| **Aggressive GPS Tracking** | Requesting `PRIORITY_HIGH_ACCURACY` continuously | High GPS chip power draw | Use geofencing or passive location updates with coarse resolution when app is backgrounded. |
| **Unthrottled Render Loops** | Canvas or animation rendering at 120 FPS continuously off-screen | Heavy GPU utilization and heating | Pause animations when view is detached or obscured. |

---

## 3. Profiling Android Battery with Battery Historian & ADB

Google provides **Battery Historian** to parse bugreports and visualize power consumption over time.

### Automated Test Workflow via ADB:
```bash
# Step 1: Reset battery stats on connected test device
adb shell dumpsys batterystats --reset

# Step 2: Enable full wake lock and battery tracking
adb shell dumpsys batterystats --enable full-wake-history

# Step 3: Run automated UI stress test (e.g., Appium or Maestro for 30 minutes)
maestro test .maestro/soak-test-flow.yaml

# Step 4: Extract bug report for analysis
adb bugreport battery-report.zip

# Step 5: Convert and analyze via Battery Historian docker container
docker run -p 9999:9999 gcr.io/android-battery-historian/stable:3.0 --port 9999
# Open http://localhost:9999 and upload battery-report.zip
```

### Inspecting Battery Drain via Terminal:
```bash
# Inspect estimated mAh drain per application UID
adb shell dumpsys batterystats | grep -E "Uid u0_a[0-9]+"
```

---

## 4. Measuring Thermal State & Throttling on iOS

On iOS devices, thermal states are classified into four tiers:
1. `nominal`: Normal operation, no thermal issues.
2. `fair`: Slightly elevated temperature; no throttling yet.
3. `serious`: High thermal load; system reduces CPU/GPU clock speeds and dims screen brightness.
4. `critical`: Extreme heat; device prepares to shut down or kill applications.

### Automated Test Observer in Swift / XCUITest:

```swift
import XCTest

final class ThermalStressTests: XCTestCase {

    func testContinuousVideoProcessingThermalStability() {
        let app = XCUIApplication()
        app.launch()

        // Navigate to intensive AR / video processing screen
        app.buttons["start_ar_session"].tap()

        let expectation = XCTKVOExpectation(
            keyPath: "thermalState",
            object: ProcessInfo.processInfo,
            expectedValue: ProcessInfo.ThermalState.critical.rawValue
        )
        expectation.isInverted = true // Test fails if critical thermal state is reached

        // Run heavy workload for 15 minutes
        let startTime = Date()
        while Date().timeIntervalSince(startTime) < 900 {
            let currentThermalState = ProcessInfo.processInfo.thermalState
            
            // Assert device does not enter serious or critical thermal throttling
            XCTAssertNotEqual(currentThermalState, .critical, "Thermal state reached CRITICAL!")
            XCTAssertNotEqual(currentThermalState, .serious, "Thermal state reached SERIOUS!")

            Thread.sleep(forTimeInterval: 5)
        }
    }
}
```

---

## 5. Performance QA Benchmark Checklist

| Evaluation Area | Pass Benchmark Criteria | Method of Validation |
| :--- | :--- | :--- |
| **Idle Background Drain** | < 0.5% battery drop per hour when backgrounded | Run 4-hour soak test with screen locked |
| **Active Screen Time** | < 8% battery drop per hour of active usage | Automated Maestro/Appium continuous user loop |
| **Thermal Temperature** | Device skin temperature < 40°C ($104^\circ\text{F}$) | Infrared thermometer / Device temperature sensors |
| **Radio State Transitions** | Radio sleep duty cycle > 80% | Battery Historian Cellular Radio timeline |

---

## 6. SQA Interview Questions & Answers

### Q1: What is Android Doze Mode, and how should QA test for it?
> **Answer**:
> **Doze Mode** is an Android power-management feature that reduces battery consumption by deferring background CPU and network activity when a device is unplugged, stationary, and screen-off for an extended period.
> QA can test Doze mode compliance without waiting hours by forcing the state via ADB:
> ```bash
> adb shell dumpsys deviceidle force-idle
> ```
> QA verifies that background jobs pause and resume cleanly once the device exits idle mode (`adb shell dumpsys deviceidle unforce`).

### Q2: Why does continuous cellular radio polling consume more power than downloading a single large file?
> **Answer**:
> Cellular modems use state machines: **Idle $\to$ Connected $\to$ Full Power (Rx/Tx)**. 
> Transitioning to Full Power incurs a high energy spike, and the radio stays in a intermediate Tail state for 10–20 seconds after transmission before powering down. Frequent periodic polling prevents the modem from ever returning to the low-power Idle state, causing continuous battery depletion.

---

## 7. Key Takeaways & Best Practices

- Integrate automated battery drain benchmarks into release candidate testing cycles.
- Use Battery Historian to audit background WakeLocks and radio state churn.
- Assert that thermal states never transition to `serious` or `critical` during extended intensive features (e.g., video calling, games, augmented reality).
