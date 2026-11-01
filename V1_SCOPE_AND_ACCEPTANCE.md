# ResQMesh — V1 Scope and Acceptance Criteria

**This file defines what Iteration 1 is allowed to build.**

The purpose of V1 is not to complete ResQMesh. The purpose is to prove the smallest **real offline Android vertical slice** on two physical phones.

---

## 1. V1 definition

A successful V1 is an installable Android debug APK that can:

1. launch on two compatible Android phones,
2. obtain required permissions,
3. create an SOS on Phone A,
4. persist that SOS locally,
5. discover Phone B over BLE,
6. connect locally without internet,
7. transfer the complete SOS over BLE GATT,
8. validate and persist it on Phone B,
9. display the received SOS on Phone B,
10. return a valid hop receipt to Phone A,
11. survive duplicate transmission without duplicating the SOS,
12. survive a failed transfer without falsely marking success.

---

## 2. UI scope

The UI should be **functional and intentionally simple**.

Suggested screens/sections:

### Home / Send
- device/node short ID for debugging,
- permission/capability status,
- emergency type selector,
- optional short message,
- location status,
- `Create SOS` button,
- current SOS state.

### Nearby peers
- discovery running/stopped,
- discovered ResQMesh peers,
- connection state,
- send action / selected peer if manual selection is needed for V1.

### Received SOS
- received emergency entries,
- timestamp,
- emergency type,
- location status/coordinates where available,
- sender short ID for debugging,
- persisted state.

### Diagnostics
- Bluetooth enabled,
- BLE advertising supported,
- scan permission,
- advertise permission,
- connect permission,
- location permission/quality,
- basic transport events.

The UI must reflect real state. No fake animated network results.

---

## 3. Required persistence

V1 must persist:
- installation `nodeId`,
- locally created SOS records,
- received SOS records,
- delivery/relay state needed for the two-phone flow.

Phone B must still show the received SOS after app process death/app relaunch.

---

## 4. Required networking

### Discovery
- BLE advertising,
- BLE scanning,
- app-specific service identity,
- capability checks,
- timeout/state handling.

### Transfer
- BLE GATT,
- framing/chunking,
- serialized writes,
- reassembly,
- timeout handling,
- receipt path.

### Not required in V1
- Wi-Fi Direct implementation,
- multi-hop forwarding,
- background-always-on discovery.

However, architecture must leave a clean `Transport` boundary for Wi-Fi Direct later.

---

## 5. Required protocol behavior

V1 must support:
- versioned SOS packet,
- UUID message IDs,
- UUID node IDs,
- packet hard-size limit,
- short-message limit,
- optional location,
- `HOP_RECEIPT`,
- optional `DELIVERY_ACK` only through explicit Responder/Test Gateway mode,
- deduplication.

A normal relay hop must not look like responder delivery.

---

## 6. Required failure behavior

The following must be handled without crashes or false success:

- Bluetooth disabled,
- permission denied,
- BLE advertising unsupported,
- no peer found,
- connection failure,
- peer disconnects during transfer,
- malformed frame,
- oversized packet,
- malformed JSON,
- duplicate SOS,
- receiver database failure,
- receipt timeout,
- location unavailable.

When location fails, SOS creation must still be possible.

---

## 7. Explicit V1 exclusions

Do **not** implement unless required to fix the basic vertical slice:

- three-phone routing,
- store-carry-forward scheduling,
- battery-based route selection,
- graph/pathfinding,
- Wi-Fi Direct transport,
- iOS,
- Firebase,
- cloud server,
- login/accounts,
- maps requiring online tiles,
- media attachments,
- hardware gateway,
- radio/satellite backhaul,
- background permanent service,
- polished animations,
- complex cryptography,
- responder identity infrastructure,
- production release signing.

---

## 8. Build acceptance gate

The implementation is not complete merely because code was written.

Before claiming V1 implementation completion:

```powershell
.\gradlew.bat testDebugUnitTest
.\gradlew.bat lintDebug
.\gradlew.bat assembleDebug
```

At minimum:
- compilation succeeds,
- unit tests for pure protocol/framing logic pass,
- no unresolved dependency/version errors,
- no known fatal lint/security issue is ignored,
- debug APK exists.

Expected output path:

```text
app\build\outputs\apk\debug\app-debug.apk
```

If project structure differs, document the actual path.

---

## 9. Physical two-phone acceptance test

Use Phone A and Phone B.

### Environment
- app installed on both,
- mobile data OFF,
- both disconnected from normal Wi-Fi routers,
- Bluetooth ON,
- app foregrounded,
- required permissions granted.

### Test A — normal transfer
1. Start ResQMesh on both phones.
2. Confirm capability diagnostics are healthy.
3. Phone A creates an SOS.
4. Verify it appears as persisted/pending on A.
5. Discover B.
6. Transfer SOS.
7. Verify B displays the correct SOS.
8. Relaunch B and verify it still exists.
9. Verify A receives **hop receipt**, not a responder-delivery message.

### Test B — duplicate
1. Send the same `messageId` again.
2. B must not create a duplicate SOS.
3. A may receive an `ALREADY_PRESENT` hop receipt.

### Test C — broken transfer
1. Begin a transfer.
2. Interrupt Bluetooth / terminate receiving app / move out of range.
3. Verify A does not record successful hop receipt.
4. Verify no impossible “delivered” state appears.

### Test D — location unavailable
1. Deny/disable location.
2. Create SOS.
3. SOS must still be created and transferable with location marked unavailable.

### Test E — responder test
1. Put B explicitly in Responder/Test Gateway mode.
2. Send a new SOS.
3. B persists it.
4. A may now receive a distinct **delivery ACK**.
5. UI must make the distinction obvious.

---

## 10. V1 completion statement

Only after build + physical tests should the project claim:

> “The ResQMesh V1 prototype successfully transferred and durably stored an SOS between two Android phones using local peer-to-peer communication without an internet connection.”

Do not claim multi-hop mesh forwarding until a three-phone relay has actually been demonstrated.
