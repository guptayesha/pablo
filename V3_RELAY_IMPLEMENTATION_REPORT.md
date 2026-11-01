# ResQMesh V3 Relay Implementation Report

**Date:** 2026-09-04  
**Scope:** Controlled three-phone application-layer SOS relay milestone  
**Evidence:** JVM/unit, lint, and debug build only. No physical three-phone result is claimed.

## 1. Relay architecture

`MeshEngine` remains the application coordinator. It reconstructs the stored immutable
`SosPayload`, creates a fresh per-hop `MeshEnvelope`, and delegates bytes to the existing
`Transport`. `RelayRoutingPolicy` selects one eligible discovered peer. Room is authoritative
for the SOS, inbound hop metadata, next-hop confirmation, responder acknowledgement, and a
bounded attempted-peer set. `PeerDiscovery` supplies RSSI, a compact app-derived node token,
and the explicitly advertised responder capability.

The receiver lifecycle is:

```text
receive -> decode -> validate -> dedupe -> durable insert -> ACK/receipt to inbound peer
                                                    -> separate relay trigger
```

Onward failure never changes the already-completed inbound storage/receipt result.

## 2. Exact files changed

Production:

- `app/src/main/java/com/mavens/resqmesh/ResQMeshApplication.kt`
- `app/src/main/java/com/mavens/resqmesh/data/db/ResQMeshDatabase.kt`
- `app/src/main/java/com/mavens/resqmesh/data/db/SosDao.kt`
- `app/src/main/java/com/mavens/resqmesh/data/db/SosEntity.kt`
- `app/src/main/java/com/mavens/resqmesh/data/repository/RoomSosRepository.kt`
- `app/src/main/java/com/mavens/resqmesh/data/repository/SosRepository.kt`
- `app/src/main/java/com/mavens/resqmesh/discovery/PeerAdvertisement.kt` (new)
- `app/src/main/java/com/mavens/resqmesh/discovery/PeerDiscovery.kt`
- `app/src/main/java/com/mavens/resqmesh/discovery/ble/BleAdvertiser.kt`
- `app/src/main/java/com/mavens/resqmesh/discovery/ble/BlePeerDiscovery.kt`
- `app/src/main/java/com/mavens/resqmesh/discovery/ble/BleScanner.kt`
- `app/src/main/java/com/mavens/resqmesh/domain/mesh/MeshEngine.kt`
- `app/src/main/java/com/mavens/resqmesh/domain/mesh/RelayRoutingPolicy.kt` (new)
- `app/src/main/java/com/mavens/resqmesh/domain/model/SosRecord.kt`
- `app/src/main/java/com/mavens/resqmesh/protocol/PacketValidator.kt`
- `app/src/main/java/com/mavens/resqmesh/transport/ble/BleConstants.kt` (comment only)
- `app/src/main/java/com/mavens/resqmesh/ui/MeshViewModel.kt`
- `app/src/main/java/com/mavens/resqmesh/ui/screens/InboxScreen.kt`
- `app/src/main/java/com/mavens/resqmesh/ui/screens/PeersScreen.kt`

Tests:

- `app/src/test/java/com/mavens/resqmesh/data/repository/RoomSosRepositoryTest.kt` (new)
- `app/src/test/java/com/mavens/resqmesh/discovery/PeerAdvertisementCodecTest.kt` (new)
- `app/src/test/java/com/mavens/resqmesh/domain/mesh/MeshEngineTest.kt`
- `app/src/test/java/com/mavens/resqmesh/domain/mesh/RelayRoutingPolicyTest.kt` (new)
- `app/src/test/java/com/mavens/resqmesh/protocol/PacketValidatorTest.kt`

Documentation:

- `V3_RELAY_IMPLEMENTATION_REPORT.md` (new)

## 3. Protocol format

The application packet format did not change. Existing `SOS`, `HOP_RECEIPT`, and
`DELIVERY_ACK` types and existing envelope hop fields are reused. Discovery now includes a
10-byte version/role/node-token service-data element in the legacy-budget scan response.

## 4. Room schema

Room changed from version 1 to 2 through an additive, non-destructive migration. Added fields:

- `inboundSenderNodeId`
- `hopCount`
- `maxHops`
- `relayAttemptedPeerTokens`

The attempted-peer list is capped at 16 entries. Existing SOS rows are preserved. Migrated V1
received rows lack an inbound sender and therefore remain manually relayable but are excluded
from automatic relay to avoid accidentally returning them to an unknown previous hop.

## 5. Same-message identity

The stored record is converted back to a value-equal `SosPayload`; `messageId`, origin,
creation time, emergency content, and location are unchanged. Only the new envelope's
`packetId`, `senderNodeId`, `lastHopNodeId`, and `hopCount` describe the new transfer.

## 6. Duplicate and loop prevention

- Room's `messageId` primary key remains the final dedupe barrier.
- Only a newly `STORED` inbound SOS triggers automatic relay; `ALREADY_PRESENT` only gets an
  idempotent receipt and never creates a forwarding cycle.
- Self and the immediate inbound application node are ineligible.
- Automatic relay excludes persisted attempted peer tokens.
- One peer is selected per event; no flooding or recursive fan-out is used.
- `maxHops` is enforced before onward relay.
- Receipts/ACKs must match both the active link and the advertised peer token, and control
  payload identity must match the envelope sender.

## 7. Manual relay

Inbox records have a minimal **Select for relay** action. The Peers tab sends the selected
local or received SOS using **Send / Relay**. Self, immediate previous hop, and hop-limit
targets are disabled/rejected. Manual retry remains possible after an automatic failure.

## 8. Automatic relay

Implemented for newly stored received SOS records. Selection prefers an explicitly advertised
Responder/Test Gateway, then the strongest RSSI, with deterministic tie-breaking. It excludes
self, inbound sender, hop-limit records, and already-attempted peers. The origin's local SOS is
still deliberately user-sent for deterministic control.

## 9. Store-carry-forward

Implemented in the existing foreground model. A newly stored SOS and each discovered-peer
list change emit bounded relay events; there is no polling loop or service. With no eligible
peer, the row remains `Waiting for relay`. A later new peer can trigger one serialized onward
attempt. Failed peers are not automatically retried in the same persisted attempt window;
manual retry remains available.

## 10. DELIVERY_ACK status

A third-hop responder still sends `DELIVERY_ACK` and `HOP_RECEIPT` as distinct packets after
durable storage. A relay holder can persist both against a received SOS. Reverse propagation
from B back to origin A is not implemented; existing validation was not weakened.

## 11. BLE/GATT changes

BLE discovery advertising/scanning changed only to exchange the compact app token and role
metadata needed for safe selection. No GATT transport implementation, characteristic layout,
framing, reassembly, MTU handling, write queue, or notification path changed.

## 12. Android 16 frame regression

The 512-byte complete attribute ceiling remains intact. `safeFrameBudget(517) == 512`, the
14-byte header still gives a 498-byte one-frame packet boundary, 499 bytes still splits, and
the existing `IllegalArgumentException -> SendOutcome.Failed` guards remain in both write and
notification paths.

## 13. Tests added/expanded

Coverage now includes A->B->C identity preservation, durable per-hop receipt behavior,
A->B->C->B duplicate/loop resistance, failed and absent next hops, manual target exclusions,
event-driven later-peer carry, no rapid retry to the same failed peer, responder-at-third-hop
ACK separation, deterministic next-hop policy, bounded persisted attempt state, discovery
metadata parsing, control sender/session binding, and all prior protocol/framing regressions.

## 14-17. Automated validation and APK

- `testDebugUnitTest`: **71 tests, 0 failures, 0 errors** across 8 suites.
- `lintDebug`: **BUILD SUCCESSFUL**, 0 errors, 21 pre-existing/non-fatal warnings.
- `assembleDebug`: **BUILD SUCCESSFUL**.
- APK: `app/build/outputs/apk/debug/app-debug.apk`

## 18. Known limitations

- Physical three-phone BLE behavior is untested and must be validated on devices.
- Foreground-only, one outgoing transfer at a time.
- No reverse responder ACK propagation to the origin.
- No timed automatic retry to the same failed peer; a new eligible peer event or manual retry
  is required. Attempt history is bounded to 16 peer tokens.
- Discovery tokens and responder capability are prototype metadata, not authenticated identity.
- Compatible V3 peers must return the scan-response metadata to appear as eligible peers.
- This is controlled single-next-hop routing, not arbitrary-scale or guaranteed-delivery mesh.

## 19. Recommended physical three-phone sequence

Preparation on A, B, and C: install the same APK; disable mobile data; disconnect ordinary
Wi-Fi; enable Bluetooth; grant Nearby Devices (and optional location); foreground ResQMesh;
turn Mesh discovery on; keep Responder mode off initially.

1. On A create SOS X, select it, then Peers -> B -> **Send / Relay**.
2. Verify A shows B's hop receipt and B Inbox contains exactly one X.
3. On B Inbox select X, then Peers -> C -> **Send / Relay**.
4. Verify B shows C's hop receipt and C Inbox contains exactly one X with the same displayed
   message-ID prefix as A/B.
5. Enable Responder/Test Gateway on C, wait for peers to refresh, create new SOS Y on A, and
   repeat A->B->C. Verify B records a responder DELIVERY_ACK separately from C's hop receipt.
6. For carry, create X2 and send A->B while C discovery is off. Confirm B says Waiting for
   relay, then enable C discovery; observe automatic attempt or use B's manual action.
7. For loop resistance, leave all discovery on and resend X toward a device that already has
   it. Verify `ALREADY_PRESENT`, one Inbox row per phone, and no continuing transfers.
8. For failure, interrupt Bluetooth during B->C. Verify B retains the SOS as waiting/pending,
   no false hop/final delivery appears, and the app remains alive; restore Bluetooth and retry
   manually or introduce another peer.

Collect sanitized `ResQMesh/*` logcat from all phones for every failure. Physical success must
only be reported after the above observations are made on real devices.
