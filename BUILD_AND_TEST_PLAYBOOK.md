# ResQMesh — Build and Test Playbook

**Environment:** Windows + Android Studio SDK + JDK 17 + ADB + physical Android phones

---

## 1. Expected local prerequisites

Before implementation/building, verify:

```powershell
java --version
adb devices
agent --version
```

Expected:
- Java 17 active for command-line Gradle work,
- at least one test phone visible as `device`,
- Cursor Agent CLI available.

Project directory:

```text
C:\Users\aaryarishi\Documents\ResQMesh_Android
```

---

## 2. Java/Gradle rule

The machine also contains a newer Android Studio runtime.

The ResQMesh Gradle build should use **JDK 17** for the frozen AGP/Gradle stack.

Known local JDK 17 path:

```text
C:\Program Files\Java\jdk-17.0.5
```

If Gradle unexpectedly launches with another JDK, diagnose `JAVA_HOME`, Android Studio Gradle JDK settings and Gradle environment before upgrading dependencies.

Do not “solve” a JDK mismatch by moving the project to AGP 9/API 37 without explicit approval.

---

## 3. Initial build loop

From the project root:

```powershell
.\gradlew.bat --version
.\gradlew.bat testDebugUnitTest
.\gradlew.bat lintDebug
.\gradlew.bat assembleDebug
```

The first command must show the expected Gradle/JVM environment.

Expected debug APK path:

```text
app\build\outputs\apk\debug\app-debug.apk
```

---

## 4. Install APK

One connected device:

```powershell
adb install -r ".\app\build\outputs\apk\debug\app-debug.apk"
```

Multiple connected devices:

```powershell
adb devices
adb -s <DEVICE_SERIAL> install -r ".\app\build\outputs\apk\debug\app-debug.apk"
```

Repeat for Phone A and Phone B.

---

## 5. Logs

General:

```powershell
adb logcat
```

Prefer filtered logs using an application-specific tag/package once known.

When multiple phones are attached:

```powershell
adb -s <DEVICE_SERIAL> logcat
```

Record:
- crash stack traces,
- BLE status/error codes,
- permission failures,
- connection lifecycle,
- transfer/session ID prefixes,
- packet lengths,
- persistence outcome.

Do not log full emergency payloads unless temporarily necessary and then remove that logging.

---

## 6. Unit tests required in V1

Pure Kotlin tests should cover at minimum:

### Protocol
- valid SOS round-trip serialization,
- malformed packet rejection,
- unsupported protocol version,
- invalid coordinates,
- overlong UTF-8 message,
- packet-size boundary.

### Framing/reassembly
- one-frame packet,
- multi-frame packet,
- out-of-order/missing chunk rejection or handling,
- duplicate chunk handling,
- incorrect length,
- checksum failure,
- oversized declared transfer.

### Deduplication/state
- new message accepted,
- duplicate message recognized,
- hop receipt does not equal delivery ACK,
- failed transfer does not advance durable state.

Hardware BLE behavior is not replaced by these tests.

---

## 7. Physical test preparation

For both phones:

- install same APK,
- grant Nearby Devices permissions,
- grant location permission if testing location,
- Bluetooth ON,
- app in foreground,
- disconnect from normal Wi-Fi network,
- mobile data OFF.

Confirm the app diagnostics before transfer.

---

## 8. Mandatory two-phone tests

### P1 — Discovery
Expected:
- both apps start cleanly,
- peer discovery finds the ResQMesh peer,
- no internet is needed.

### P2 — Normal SOS
Expected:
- A persists SOS,
- A transfers it,
- B persists/displays it,
- B sends hop receipt,
- A shows hop success, not responder delivery.

### P3 — Persistence
Expected:
- force-close/relaunch B,
- received SOS remains visible.

### P4 — Duplicate
Expected:
- resend same message ID,
- no duplicate database/UI record,
- duplicate-safe receipt allowed.

### P5 — Transfer interruption
Interrupt mid-transfer.

Expected:
- no false positive,
- A retains pending SOS,
- incomplete data is discarded/cleaned on B,
- next retry can recover.

### P6 — Bluetooth disabled
Expected:
- clear state/error,
- no crash,
- user can recover after enabling Bluetooth.

### P7 — Permission denial
Expected:
- explain missing capability,
- no crash,
- permission can be retried through UI.

### P8 — Location unavailable
Expected:
- SOS still created,
- location represented unavailable,
- transfer succeeds.

### P9 — Oversized/malformed packet
Expected:
- reject safely,
- no crash,
- no database insertion,
- no positive receipt.

### P10 — Responder/Test Gateway
Expected:
- only explicit responder mode returns a delivery ACK,
- UI distinguishes delivery ACK from hop receipt.

---

## 9. Definition of a valid result

A valid test result includes:
- build output,
- actual device behavior,
- logs where relevant.

Invalid evidence:
- “the code looks correct,”
- model statement that testing passed when commands were not run,
- browser simulation,
- emulator-only BLE demonstration,
- mocked peer list,
- hardcoded “received” state.

---

## 10. Bug workflow

For every failure:

1. reproduce,
2. capture exact device + Android version,
3. capture logs,
4. identify layer: permission, discovery, connection, framing, protocol, persistence, or UI state,
5. fix the smallest layer,
6. run unit tests,
7. rebuild,
8. reinstall,
9. rerun the failing physical test,
10. rerun the normal transfer regression.

Do not patch a single peer ID or test scenario.

---

## 11. After V1 passes

Only then start the next branch/milestone:

```text
A → B → C
```

The two-phone tests remain the regression suite while multi-hop is developed.
