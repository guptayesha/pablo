# ResQMesh — Verified Android Technical Specification

**Purpose:** Technical source of truth for the first Android implementation  
**Verification date:** 2026-08-31  
**Priority:** Prefer current Android/Google primary documentation over memory or model assumptions.

---

## 1. Supported Android range

### Frozen baseline
- **minSdk:** 26 (Android 8.0)
- **compileSdk:** 36
- **targetSdk:** 36

The minimum SDK is an engineering choice: it gives broad device coverage while avoiding unnecessary complexity from very old Android behavior.

Targeting API 36 is deliberate for this deadline. Do not silently raise the project to API 37 merely because a newer dependency requests it.

---

## 2. Build toolchain baseline

Use a conservative, mutually compatible stack:

- **JDK:** 17
- **Android Gradle Plugin:** 8.13.2
- **Gradle:** 8.13
- **Kotlin:** 2.3.21 or a confirmed Kotlin 2.3.x version compatible with AGP 8.13.2
- **Compose:** 1.11.x stable line; 1.11.4 is the preferred baseline
- **Material 3:** stable 1.4.x line
- **Room:** 2.8.4
- **Room code generation:** KSP for Kotlin
- **Navigation:** if needed, stay on an API-36-compatible stable line; do not upgrade the entire project only to obtain the newest Navigation/Compose artifacts.

### Important rule
Do **not** use “latest everything.”

Newer Compose releases have moved their compile SDK / AGP baseline forward. The first prototype gains no useful feature from forcing an API-37 / AGP-9.x migration.

Leave `buildToolsVersion` unspecified unless there is a concrete need; let the chosen AGP select its compatible default.

---

## 3. Bluetooth permissions

### Android 12 / API 31 and newer
The app may require the runtime Nearby Devices permissions relevant to its behavior:

- `BLUETOOTH_SCAN`
- `BLUETOOTH_ADVERTISE`
- `BLUETOOTH_CONNECT`

Request only the permissions needed by the implemented features.

### Android 11 / API 30 and older
Use the legacy Bluetooth permission declarations required by those versions and constrain them with the appropriate `maxSdkVersion` so modern Android does not receive unnecessary legacy permissions.

### Location is separate
ResQMesh itself needs foreground location for SOS coordinates, so the application still needs:
- `ACCESS_COARSE_LOCATION`
- `ACCESS_FINE_LOCATION`

Do not remove location permission just because modern Bluetooth discovery can use the Nearby Devices permission group.

---

## 4. Wi-Fi Direct permissions and behavior

Wi-Fi Direct is a valid infrastructure-free peer-to-peer transport and does **not** require internet access or a Wi-Fi access point.

For later Wi-Fi Direct support:

### Android 13 / API 33 and newer
Use:
- `NEARBY_WIFI_DEVICES`

### Older Android versions
Relevant Wi-Fi P2P discovery/connection APIs depend on the older fine-location permission path.

### Manifest/socket nuance
Wi-Fi Direct uses standard sockets, so Android requires the manifest `INTERNET` permission even though no internet connection is required. This permission must never be described to evaluators as evidence that the core ResQMesh path needs the internet.

### Location Mode nuance
Android documents that APIs including:
- `discoverPeers()`,
- `discoverServices()`,
- `requestPeers()`

also require the device's **Location Mode** to be enabled.

This is one reason Wi-Fi Direct is not mandatory for the first BLE vertical slice.

---

## 5. V1 discovery approach

Use **Bluetooth Low Energy advertising + scanning** for nearby peer discovery.

### Advertisement content
Keep V1 advertisements compatible with the legacy advertising budget even on devices that support extended advertising.

Advertisements should contain only small discovery information, such as:
- ResQMesh service UUID,
- protocol major version / capability bits if they fit,
- no SOS body,
- no user message,
- no GPS coordinates,
- no personal identity.

Do not assume every Android phone supports BLE advertising. Perform a runtime capability check.

If BLE advertising is unavailable on a test device:
1. report the limitation clearly in diagnostics,
2. do not crash,
3. preserve the architecture so a later Wi-Fi Direct transport/discovery path can serve that hardware.

---

## 6. V1 transport approach

Use **BLE GATT** for the first two-phone SOS transfer.

This is an engineering choice to reduce the number of moving parts before the core packet lifecycle is proven.

### Requirements
- do not assume one SOS fits in one GATT write,
- negotiate/use the actual effective MTU when available,
- use conservative chunking if MTU information is unavailable,
- serialize GATT writes; do not issue uncontrolled concurrent writes,
- implement framing and reassembly,
- enforce transfer timeout,
- handle disconnect during transfer,
- reject malformed frames,
- never mark success until the receiver has durably committed the complete packet.

### Suggested symmetric model
Each capable ResQMesh device can:
- advertise a GATT service,
- host a small GATT server,
- scan for peers,
- connect as a GATT client when it has a packet to send.

Use an RX write characteristic and an acknowledgement/notification path, or another simple design that provides equivalent correctness.

The exact characteristic layout may be selected during implementation, but it must obey the protocol invariants.

---

## 7. BLE packet sizing nuance

Legacy BLE advertisements have a small payload budget. Extended advertising can be larger on supported Android 8+ chipsets, but V1 must not depend on it.

GATT payload size is governed by the negotiated/effective ATT MTU and operation overhead. Therefore:
- derive chunk size dynamically,
- keep a safe fallback,
- never hardcode a phone-specific MTU as universally available.

The transport layer owns chunking. The protocol layer should operate on complete serialized packets.

---

## 8. Location behavior

The SOS location path should use Android's framework location facilities without making Google Play Services or internet connectivity a hard dependency.

### States
Represent location explicitly as:
- `PRECISE`
- `APPROXIMATE`
- `UNAVAILABLE`

### Rules
- attempt a fresh GNSS/location fix for a bounded time,
- a sufficiently recent cached fix may be used if clearly marked with capture time and accuracy,
- include latitude, longitude, accuracy and capture timestamp when available,
- never block SOS creation/transmission because location cannot be obtained,
- support the user granting approximate rather than precise location,
- handle permission changes and process recreation safely.

Offline GNSS is possible, but a fix is not guaranteed indoors or with poor satellite visibility.

---

## 9. Foreground/background scope

### V1
Keep the app **foreground-operated** during discovery and transfer.

This intentionally avoids introducing Android background execution and foreground-service complexity into the first proof.

### Later
If prolonged/background device communication is added:
- use the appropriate foreground service type,
- for modern Android, account for `connectedDevice` service requirements,
- request the corresponding foreground-service permission(s),
- account for Android 12+ restrictions on starting foreground services from the background,
- account for Android 14+ foreground service type enforcement.

Do not implement a permanent background service in V1.

---

## 10. Device capability checks

Create a small compatibility/capability layer. At runtime detect and expose at minimum:

- Bluetooth hardware available,
- Bluetooth enabled,
- BLE scanning available,
- BLE advertising available,
- required Nearby Devices permissions granted,
- coarse/fine location status,
- current location precision status,
- Location Mode enabled,
- Wi-Fi available,
- Wi-Fi Direct support/state when that transport is implemented.

Unsupported capabilities must produce an understandable state, not a crash.

---

## 11. Device identifiers

Do not use IMEI, Android serial number, Bluetooth MAC address, Wi-Fi MAC address, or other hardware identifiers as the ResQMesh node identity.

Generate a random app-scoped UUID on first launch and persist it in app-private storage.

A separate random UUID is generated for every SOS message.

---

## 12. Local persistence

Use Room for structured durable storage.

Persist before acknowledging receipt.

At minimum the database must make these operations safe:
- insert a newly created SOS,
- insert a newly received SOS,
- detect a duplicate by message ID,
- update local forwarding/delivery state,
- survive process death/app relaunch,
- return pending/received SOS records to the UI.

Enforce a unique constraint/index on the message ID.

---

## 13. Emulator vs physical device

Emulator tooling can help with UI, protocol tests, database tests and general Android behavior.

It is not sufficient acceptance evidence for BLE on real phones.

The real acceptance gate is **two physical Android phones** with real radios and permissions.

---

## 14. Offline demo condition

For a clean demonstration:

- mobile data OFF,
- phones disconnected from Wi-Fi routers/access points,
- Bluetooth ON,
- Wi-Fi may remain ON for future Wi-Fi Direct testing,
- core packet transfer must succeed without cloud access.

Do not rely only on “Airplane Mode” as the definition of offline because device-specific settings can disable Bluetooth/Wi-Fi radios. If Airplane Mode is demonstrated, re-enable the local radios explicitly.

---

## 15. Primary references

Official Android documentation used for the verified assumptions:

- Bluetooth permissions  
  https://developer.android.com/develop/connectivity/bluetooth/bt-permissions

- Android 12 Bluetooth behavior changes  
  https://developer.android.com/about/versions/12/behavior-changes-12

- Wi-Fi Direct / Wi-Fi P2P overview  
  https://developer.android.com/develop/connectivity/wifi/wifip2p

- Wi-Fi Direct connection guide  
  https://developer.android.com/develop/connectivity/wifi/wifi-direct

- Nearby Wi-Fi devices permission  
  https://developer.android.com/develop/connectivity/wifi/wifi-permissions

- Foreground service launch requirements  
  https://developer.android.com/develop/background-work/services/fgs/launch

- Room documentation  
  https://developer.android.com/training/data-storage/room

- AGP 8.13 release notes  
  https://developer.android.com/build/releases/agp-8-13-0-release-notes

- AndroidX stable channel  
  https://developer.android.com/jetpack/androidx/versions/stable-channel

- Android identity guidance  
  https://developer.android.com/identity/user-data-ids

---

## 16. Change-control rule

If implementation reveals an OEM/API behavior that contradicts this specification:
1. reproduce it,
2. collect logs,
3. verify against current Android documentation,
4. document the deviation,
5. change the smallest possible layer.

Do not redesign the protocol or upgrade the entire toolchain as the first reaction to a device-specific problem.
