# ResQMesh — V1 Implementation Report

**Date:** 2026-08-31
**Scope:** Milestone 1 — two-phone offline BLE vertical slice
**Status:** Built, unit-tested, linted, APK produced, single-device launch smoke test passed. Two-phone physical acceptance test NOT yet performed (requires two phones and human interaction).

---

## 1. Architecture implemented

Layered, matching `ARCHITECTURE_GUIDELINES.md`:

```
Compose UI (MainActivity + 4 tab screens)
  ↓ observes StateFlow only
MeshViewModel (single ViewModel, no BLE/DB types)
  ↓
MeshEngine (pure Kotlin use-case core: create/send/receive/receipt/ACK)
  ├── protocol/      PacketCodec, PacketValidator, FrameCodec, FrameReassembler (pure Kotlin)
  ├── data/          SosRepository interface → RoomSosRepository → Room DB
  ├── discovery/     PeerDiscovery interface → BlePeerDiscovery (scanner + advertiser)
  ├── transport/     Transport interface → BleTransport (GATT client + GATT server)
  └── platform/      PermissionPolicy, DeviceCapabilities, LocationProvider
```

- The UI never touches `BluetoothGatt`, Room DAOs or raw frames.
- `Transport` and `PeerDiscovery` are transport-agnostic interfaces; a future
  Wi-Fi Direct transport implements the same boundary without touching
  protocol, persistence, engine or UI.
- `MeshEngine`, the protocol layer, framing and the repository interface are
  pure Kotlin (no Android imports) and are unit tested on the JVM.
- Composition root is `ResQMeshApplication` (no DI framework).

## 2. Important files/packages created

```
app/src/main/java/com/mavens/resqmesh/
  ResQMeshApplication.kt                 composition root
  protocol/ProtocolConstants.kt          version=1, 4096 B packet cap, 240 B message cap
  protocol/model/Packets.kt              SosPayload, MeshEnvelope, HopReceipt, DeliveryAck, enums
  protocol/PacketCodec.kt                JSON encode/decode, size gate before parse
  protocol/PacketValidator.kt            strict field/UUID/coords/hops/bytes validation
  protocol/framing/FrameCodec.kt         14-byte binary frame header + CRC32
  protocol/framing/FrameReassembler.kt   bounded, ordered, checksummed reassembly
  domain/mesh/MeshEngine.kt              sender/receiver state machines, receipt semantics
  domain/model/SosRecord.kt              domain view of persisted SOS
  data/db/ (SosEntity, SosDao, ResQMeshDatabase)   Room, messageId = primary key
  data/repository/ (SosRepository, RoomSosRepository)
  data/identity/NodeIdentityStore.kt     random UUID node id, responder-mode flag
  discovery/ (PeerDiscovery, ble/BleScanner, ble/BleAdvertiser, ble/BlePeerDiscovery)
  transport/ (Transport, ble/BleConstants, ble/GattCompat, ble/BleGattServer,
              ble/BleClientLink, ble/BleTransport)
  platform/ (permissions/PermissionPolicy, capabilities/DeviceCapabilities,
             location/LocationProvider, location/AndroidLocationProvider)
  diagnostics/ (MeshLogger, DiagnosticsLog)      sanitized bounded event log
  ui/ (MainActivity, MainScreen, MeshViewModel, theme/Theme,
       screens/SendScreen, PeersScreen, InboxScreen, DiagnosticsScreen)
app/src/test/java/com/mavens/resqmesh/
  protocol/PacketCodecTest.kt, protocol/PacketValidatorTest.kt,
  protocol/framing/FramingTest.kt, domain/mesh/MeshEngineTest.kt, protocol/TestFixtures.kt
```

## 3. Exact resolved toolchain/dependency versions

| Component | Version |
|---|---|
| JDK (launcher + daemon) | 17.0.5 (`C:\Program Files\Java\jdk-17.0.5`) |
| Gradle | 8.13 (wrapper, official distribution) |
| Android Gradle Plugin | 8.13.2 |
| Kotlin | 2.3.21 (with Compose compiler + serialization plugins at same version) |
| KSP | 2.3.11 |
| compileSdk / targetSdk / minSdk | 36 / 36 / 26 |
| Compose UI/foundation | 1.11.4 |
| Material 3 | 1.4.0 |
| Room (+KSP compiler) | 2.8.4 |
| kotlinx-coroutines | 1.10.2 |
| kotlinx-serialization-json | 1.9.0 |
| activity-compose | 1.11.0 |
| lifecycle-* | 2.9.4 |
| core-ktx | 1.17.0 |
| junit | 4.13.2 |

SDK components used: platform android-36, build-tools 36.0.0 (AGP additionally auto-installed build-tools 35.0.0, its default).

## 4. BLE service/characteristic design

| Item | UUID | Role |
|---|---|---|
| Service | `8e4f0001-1f5a-4c8b-9d3e-6a2b5c7d9e01` | ResQMesh identity, advertised + scan filter |
| RX characteristic | `8e4f0002-…` | client → server framed packet data, write-with-response |
| TX characteristic | `8e4f0003-…` | server → client framed replies via notifications (CCCD `0x2902`) |

- Every foreground instance hosts the GATT server and advertises the service
  UUID only (no name/TX power/SOS data/coordinates/identity) within the legacy
  31-byte budget; extended advertising is not required.
- Sender connects as GATT client: connect → request MTU 517 (fallback 23) →
  discover services → enable notifications → write chunks sequentially.
- Every GATT operation is awaited via `CompletableDeferred` under a `Mutex`;
  no concurrent GATT operations are ever issued. Per-op timeout 10 s, whole
  transfer budget 45 s, receipt wait 20 s, stale inbound sessions dropped
  after 30 s, max 4 concurrent inbound reassembly sessions.
- Advertising unsupported is a reported diagnostic state, not a crash.
- Bluetooth MAC addresses are used only as transient link handles, never
  persisted, never treated as node identity.

## 5. Protocol/framing implementation

- Compact JSON via kotlinx.serialization; explicit `protocolVersion = 1`;
  packet types `SOS`, `HOP_RECEIPT`, `DELIVERY_ACK`; unknown JSON fields
  ignored within the size cap; malformed JSON rejected without exceptions.
- Hard limits enforced: 4096-byte complete packet (checked before parsing and
  before any reassembly allocation), 240 UTF-8 **bytes** for the short message
  (byte length, not char count, both at creation and validation).
- `SosPayload` (immutable, sign-ready) is separated from `MeshEnvelope`
  (forwarding metadata: hopCount, maxHops, priority, sender/lastHop).
- Transport frame: 14-byte big-endian header
  `magic(1) version(1) transferId(2) seq(2) totalChunks(2) totalLength(2) crc32(4)`
  + payload; fits the worst-case 20-byte ATT payload at default MTU 23.
  CRC32 covers the complete packet and is verified after reassembly
  (corruption detection only — explicitly not authentication).
- Reassembly enforces: bounded declared length (≤ 4 KiB, checked before
  allocation), bounded chunk count (≤ 1024), strict sequential order,
  idempotent duplicate-chunk handling, header consistency across frames,
  exact final length, checksum match. Any violation discards the transfer.

## 6. Persistence design

- Room database `resqmesh.db` in app-private internal storage; backups
  disabled (`allowBackup=false` + full `dataExtractionRules` exclusions).
- Single `sos` table; `messageId` is the **primary key**, so duplicate
  message IDs cannot create duplicate rows. Deduplication uses
  `INSERT … OR IGNORE` (returns -1 → `ALREADY_PRESENT`).
- Receiver flow: reassemble → size/CRC → parse → validate → insert →
  **only after a committed row** send `HOP_RECEIPT` (STORED or
  ALREADY_PRESENT). Persistence failure ⇒ no receipt of any kind.
- Sender flow: SOS persisted locally before any transfer; failure/timeout/
  disconnect keeps it pending; only a validated application-level
  `HOP_RECEIPT` for a locally-originated messageId updates state.
- Node identity: random UUID generated on first launch, stored in
  app-private SharedPreferences; no hardware identifiers anywhere.

## 7. Security measures implemented

- All peer input treated as untrusted: size gates before allocation/parse,
  strict UUID syntax (rejects `UUID.fromString` leniency), coordinate/
  accuracy/hop-count/timestamp range checks, packet-type↔payload matching,
  byte-length message check, frame sequence/consistency checks.
- Bounded resources: packet size, chunk count, inbound sessions, tracked
  peers (32), diagnostics buffer (200), all operations time-limited.
- Receipts/ACKs only accepted for messageIds this device originated
  (direction=LOCAL); unrelated/stale receipts are ignored.
- Only the launcher activity is exported; no services/receivers/providers.
- Permissions: legacy `BLUETOOTH`/`BLUETOOTH_ADMIN` (maxSdk 30),
  `BLUETOOTH_SCAN` (neverForLocation)/`BLUETOOTH_ADVERTISE`/`BLUETOOTH_CONNECT`
  (API 31+), `ACCESS_FINE/COARSE_LOCATION` for SOS coordinates. **No INTERNET
  permission.** Nothing else.
- Logs are sanitized: 8-char UUID prefixes only, byte counts, states, error
  categories; no SOS bodies, no coordinates, no full UUIDs, no DB contents.
- No cryptography implemented and none claimed; `SosPayload` kept as an
  immutable unit ready for future signing. HOP_RECEIPT vs DELIVERY_ACK
  wording in the UI explicitly says relay storage is NOT responder delivery.
- Room access is DAO/parameterized only; no SQL built from peer input.

## 8. Commands actually executed

```powershell
.\gradlew.bat --version              # Gradle 8.13, launcher+daemon JVM 17.0.5
.\gradlew.bat testDebugUnitTest      # twice (after fixes)
.\gradlew.bat lintDebug              # three times (iterating on findings)
.\gradlew.bat assembleDebug
.\gradlew.bat testDebugUnitTest lintDebug assembleDebug   # final combined gate
adb devices
adb -s RZCW10MTE3R install -r .\app\build\outputs\apk\debug\app-debug.apk   # twice (incl. final APK)
adb -s RZCW10MTE3R shell am start -n com.mavens.resqmesh/.ui.MainActivity
adb -s RZCW10MTE3R logcat …          # crash inspection
```

## 9. Unit-test result

`testDebugUnitTest`: **49 tests, 0 failures** (4 suites: PacketCodecTest,
PacketValidatorTest, FramingTest, MeshEngineTest). Covers: round-trip
serialization, malformed/truncated/invalid-UTF-8 JSON, 4 KiB boundary and
over-limit, oversized encode rejection, unknown-field tolerance, version/UUID/
coordinate/accuracy/hop/byte-length validation, single- and multi-frame
reassembly, missing/out-of-order/duplicate chunks, cross-transfer frames,
length mismatch/overflow, checksum corruption, oversized declared transfer,
worst-case max-packet chunking, persist-before-receipt, duplicate ⇒ single
record + ALREADY_PRESENT, persistence failure ⇒ no receipt, malformed inbound
ignored, relay never sends DELIVERY_ACK, responder mode sends ACK only after
persistence, transport failure/receipt timeout ⇒ Failed + SOS kept pending,
valid receipt ⇒ HopConfirmed, unrelated receipt ignored, receipt ≠ delivery
ACK, SOS creation without location, over-byte-limit message rejected.

## 10. Lint result

`lintDebug`: **BUILD SUCCESSFUL — 0 errors, 21 warnings.**
Initial run had 6 `MissingPermission` errors on the GATT compat helpers;
fixed properly by annotating them with `@RequiresPermission(BLUETOOTH_CONNECT)`
(all call sites already catch `SecurityException`) — not by suppression.
Remaining warnings are: newer-version suggestions (deliberately frozen
toolchain per spec), `OldTargetApi` (targetSdk 36 is the frozen baseline),
`ObsoleteSdkInt` on the `mipmap-anydpi-v26` folder (AAPT requires that folder
name for adaptive icons), and a deprecated `TabRow` overload (cosmetic).

## 11. Build result

`assembleDebug`: **BUILD SUCCESSFUL**. No unresolved dependencies.

## 12. Exact APK path

```
C:\Users\aaryarishi\Documents\ResQMesh_Android\app\build\outputs\apk\debug\app-debug.apk
```
(~12.2 MB, debug-signed.)

## 13. Connected-device smoke-test result

One authorized device connected: Samsung SM-A146B (Galaxy A14 5G),
Android 15 / API 35, serial RZCW10MTE3R.

- Final APK installed via `adb install -r`: **Success**.
- Launched via `am start`: process started, stayed alive (verified via
  `pidof` after several seconds), app log line `ResQMesh/app: ResQMesh
  started` observed.
- `logcat -s AndroidRuntime:E`: **no fatal exceptions** for
  `com.mavens.resqmesh`.
- No runtime permissions were granted yet (requires human taps); the app ran
  without crashing in that state, which exercises the permission-denied path.

**Only one phone was connected, so BLE peer-to-peer transfer was NOT tested
and is NOT claimed.** No UI interaction, permission grants, discovery or
transfer were performed or fabricated.

## 14. Deviations from the authoritative specification

1. **Kotlin `jvmTarget` DSL**: Kotlin 2.3.21 removed `kotlinOptions.jvmTarget`;
   migrated to the `compilerOptions { jvmTarget.set(JvmTarget.JVM_17) }` DSL.
   Same JVM 17 target, new syntax only.
2. **KSP version**: KSP switched to standalone versioning; used `2.3.11`
   (the current KSP2 line compatible with Kotlin 2.3.x). Verified by build.
3. **DELIVERY_ACK send order**: in responder mode the ACK is sent *before*
   the hop receipt (both strictly after durable persistence), because the
   sender may close the link as soon as the hop receipt arrives. Semantics
   per spec are unchanged; only wire order differs.
4. **Adaptive icon folder**: lint suggests merging `mipmap-anydpi-v26` into
   `mipmap-anydpi`, but AAPT refuses to resolve the adaptive icon from the
   unversioned folder; kept `-v26` (warning accepted).
5. AGP auto-installed build-tools 35.0.0 alongside the present 36.0.0 —
   normal AGP behavior, not a toolchain change.

## 15. Known limitations

- BLE transfer correctness on real radios is not yet demonstrated (needs two
  phones). GATT behavior varies by OEM; the state machine has timeouts and
  cleanup but has only been exercised in unit tests below the Android layer.
- V1 is foreground-only by design: leaving the app stops scanning,
  advertising and the GATT server (restarted on return if the mesh toggle is on).
- Peer list shows transient BLE addresses (OS-randomized MACs) until a
  transfer reveals the peer's real nodeId in the receipt; peers are labeled
  by address suffix only.
- One outgoing transfer at a time (deliberate serialization).
- Hop receipts are not cryptographically authenticated (documented V1
  boundary); a nearby device could fabricate receipts. No replay protection
  beyond messageId dedupe.
- Received-SOS rows do not record which link they arrived from (deliberate:
  MACs are not persisted).
- No retry scheduler: a failed send stays pending and must be re-sent
  manually from the UI (store-carry-forward is Milestone 3).
- Location uses framework `LocationManager` only; indoors a fix may time out
  (8 s budget) and the SOS is sent without coordinates, as specified.

## 16. Remaining human steps — two-phone physical acceptance test

Preparation (both phones):
1. `adb install -r .\app\build\outputs\apk\debug\app-debug.apk` on Phone A and Phone B.
2. On each phone: mobile data OFF, disconnect from Wi-Fi networks, Bluetooth ON.
3. Open ResQMesh, tap **Grant permissions** (Send tab or Diag tab), allow
   Nearby Devices and (optionally) Location "while using".
4. On the **Diag** tab confirm: Bluetooth available/enabled, BLE scanning,
   BLE advertising supported, permissions OK.
5. On the **Peers** tab, turn the Mesh discovery switch ON on both phones.

Test A — normal transfer (spec Test A / P1–P2):
1. Phone A → Send tab → pick emergency type, optional message → CREATE SOS.
2. Verify the SOS appears under "My SOS records" as "Pending: no relay has
   confirmed storage yet" (it is persisted).
3. Peers tab: Phone B appears as "Peer …xxxxx" → tap **Send SOS**.
4. Watch transfer status: connecting → sending → waiting for hop receipt →
   "hop receipt … STORED by <node> (relay storage, NOT responder delivery)".
5. Phone B → Inbox tab: the SOS appears with type/message/location/sender.

Test P3 — persistence: force-close ResQMesh on B, relaunch → SOS still in Inbox.

Test B — duplicate: on A, select the same SOS, send again → B must show only
one record; A gets an ALREADY_PRESENT hop receipt.

Test C — broken transfer: start a send, then turn off Bluetooth on B (or walk
out of range) mid-transfer → A must show "failed … (SOS kept pending)", never
a receipt; B must not show a partial SOS.

Test D — location unavailable: deny/disable location on A, create SOS →
creation succeeds, location shows UNAVAILABLE, transfer still works.

Test E — responder ACK: on B, Diag tab → enable **Responder / Test Gateway
mode**. A creates and sends a NEW SOS → A's record must show BOTH the hop
receipt line and a distinct "DELIVERY ACK: responder/gateway … received this
SOS" line. With responder mode OFF, no delivery ACK must ever appear.

Collect `adb logcat -s ResQMesh` (all categories are prefixed `ResQMesh/…`)
from both phones for any failure.

Only after Tests A–E pass on physical phones may the project claim the V1
completion statement from `V1_SCOPE_AND_ACCEPTANCE.md` §10.
