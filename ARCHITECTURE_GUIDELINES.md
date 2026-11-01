# ResQMesh — Android Architecture Guidelines

**Goal:** Clean enough to grow from a two-phone prototype into multi-hop ResQMesh without a rewrite.

---

## 1. Architectural shape

Use a layered architecture with clear ownership:

```text
Compose UI
   │
ViewModels / UI state
   │
Application use cases / MeshEngine
   ├───────────────┬─────────────────┬──────────────────┐
   │               │                 │                  │
Protocol        Repository       PeerDiscovery      Transport
   │               │                 │                  │
serialization    Room            BLE scanner/       BLE GATT
validation       database        advertiser         V1 transport
framing

Future:
Transport → Wi-Fi Direct
Routing → multi-hop/store-carry-forward
Gateway → responder/backhaul
```

The UI must not directly operate `BluetoothGatt`, Room DAOs or raw transport frames.

---

## 2. Suggested package responsibilities

Exact package names may vary, but maintain equivalent separation:

```text
ui/
  screens/
  components/
  state/
  viewmodel/

domain/
  model/
  usecase/
  mesh/

protocol/
  model/
  serialization/
  validation/
  framing/

data/
  db/
  dao/
  entity/
  repository/

discovery/
  PeerDiscovery.kt
  ble/

transport/
  Transport.kt
  ble/
  wifi/              # future; may be empty/not created in V1

platform/
  permissions/
  capabilities/
  location/

diagnostics/
```

Avoid a single giant `MainActivity`, `BluetoothManager` or `MeshManager` that owns everything.

---

## 3. Core interfaces

### `PeerDiscovery`

Conceptually:

```text
start()
stop()
observePeers()
observeState()
```

The mesh engine should consume discovered peers without knowing scanner implementation details.

### `Transport`

Conceptually:

```text
connect(peer)
send(peer, packetBytes)
observeIncomingPackets()
disconnect(peer)
observeConnectionState()
```

The protocol/routing layer must not know whether the transport is BLE or Wi-Fi Direct.

### Repository

Conceptually:

```text
createLocalSos(...)
getSos(messageId)
insertReceivedSos(...)
observeSos(...)
markHopReceipt(...)
markDeliveryAck(...)
```

Database transactions should enforce correctness.

---

## 4. Mesh engine responsibility

For V1 the mesh engine coordinates:
- SOS creation,
- location capture attempt,
- persistence,
- peer discovery state,
- transfer request,
- receipt processing,
- user-visible state.

It must **not** contain UI code.

Future routing logic should be added behind a routing policy abstraction rather than hardcoded into transport callbacks.

---

## 5. State ownership

Use a single source of truth.

### Durable state
Room is authoritative for:
- SOS existence,
- created/received records,
- persistent delivery state.

### Ephemeral state
StateFlow/Flow may represent:
- scanning,
- peer list,
- connection status,
- current transfer progress,
- temporary errors.

Do not treat a Compose variable as proof that a packet exists on disk.

---

## 6. Concurrency

Use Kotlin coroutines and structured concurrency.

Rules:
- bind long-lived scopes to appropriate application/service/repository lifetimes,
- bind UI collection to lifecycle,
- never launch uncontrolled `GlobalScope`,
- serialize BLE GATT operations,
- cancel timeouts/jobs when connection/session ends,
- close GATT resources deterministically,
- protect shared transfer state from concurrent mutation,
- bound channels/queues.

A single peer disconnect must not leak a coroutine or leave a phantom connection state.

---

## 7. BLE structure

Prefer a small dedicated BLE subsystem:

```text
BleAdvertiser
BleScanner
BleGattServer
BleGattClient / BleConnection
BleTransport
BleTransferSession
```

Do not scatter Android Bluetooth callbacks throughout ViewModels.

Translate platform callbacks into internal domain events.

### Important
Android GATT operations are callback-driven and can report busy/failure states. Implement a controlled operation queue rather than firing writes simultaneously.

---

## 8. Protocol boundary

The transport layer receives/sends **bytes**.

The protocol layer:
- creates packet models,
- serializes/deserializes,
- validates,
- enforces limits.

The framing layer:
- splits complete packet bytes,
- reassembles chunks,
- checks transfer metadata/checksum.

This separation allows Wi-Fi Direct later to reuse the same JSON protocol without inheriting BLE chunking logic.

---

## 9. Persistence boundary

Do not acknowledge receipt from a BLE callback immediately.

Correct flow:

```text
complete bytes
→ validate
→ dedupe
→ Room transaction
→ successful commit
→ generate hop receipt
```

If persistence fails, no positive receipt is sent.

---

## 10. Permissions and capabilities

Centralize OS-version and permission logic.

Suggested abstractions:
- `PermissionManager`
- `DeviceCapabilities`
- `LocationProvider`

Do not scatter `Build.VERSION.SDK_INT` checks across unrelated files.

UI observes a clear permission/capability state and can explain what is missing.

---

## 11. Node identity

Generate the installation UUID once and persist it in app-private preferences/storage.

Node ID is not a human identity and must not be derived from device name, phone number, IMEI or MAC address.

For UI/debugging, show only a short prefix of the UUID.

---

## 12. Error model

Define typed failures where practical, for example:

```text
PermissionDenied
BluetoothDisabled
AdvertisingUnsupported
PeerNotFound
ConnectionFailed
TransferTimeout
MalformedFrame
PacketTooLarge
ProtocolInvalid
DuplicatePacket
PersistenceFailed
Disconnected
```

Avoid returning raw exception strings directly to the user.

Log enough context for debugging without leaking full SOS data.

---

## 13. Dependency policy

Keep dependencies minimal.

Prefer:
- Android framework Bluetooth APIs,
- Jetpack Compose,
- Lifecycle/ViewModel,
- Room,
- Kotlin coroutines,
- kotlinx.serialization.

Do not introduce a large networking framework for a local BLE transfer.

Do not add Firebase/Retrofit/OkHttp merely because they are familiar; the V1 core has no cloud API.

---

## 14. Testability

Pure Kotlin logic should be testable without Android hardware:
- protocol validation,
- serialization,
- frame split/reassembly,
- byte-size limits,
- dedupe decision logic,
- state transitions where modeled separately.

Android-integration behavior is tested on real hardware.

---

## 15. Future-proofing without overengineering

Design extension points for:
- Wi-Fi Direct transport,
- routing policy,
- multi-hop forwarding,
- store-carry-forward scheduler,
- responder/gateway role,
- application-level signatures.

Do not implement those layers prematurely.

A clean interface is enough until the next milestone requires the implementation.
