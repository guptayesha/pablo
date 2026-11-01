# ResQMesh — V1 Protocol Specification

**Scope:** Two-phone offline SOS vertical slice, designed to extend to multi-hop later  
**Protocol philosophy:** Small, versioned, defensive, store-before-acknowledge.

---

## 1. Goals

The V1 protocol must provide:

- a unique SOS identity,
- compact emergency content,
- optional location,
- versioning,
- bounded payload size,
- reliable transfer over a chunked transport,
- duplicate detection,
- durable receipt semantics,
- hop receipt semantics,
- an extensible path to responder delivery ACKs,
- no dependency on cloud time, cloud identity or hardware identifiers.

---

## 2. Separation of responsibilities

### `SosPayload`
Represents the emergency information created by the origin. It should remain logically immutable while being relayed.

### `MeshEnvelope`
Carries packet and forwarding metadata. This part may change between hops in future multi-hop versions.

### `TransportFrame`
Carries chunks of one serialized protocol packet over BLE GATT. The transport frame is not the SOS protocol itself.

---

## 3. Packet types

Reserve explicit packet types from the beginning:

- `SOS`
- `HOP_RECEIPT`
- `DELIVERY_ACK`

Future types may be added without changing the meaning of existing packet types. Unknown future packet types must not crash the app.

---

## 4. SOS payload

Recommended logical fields:

```text
SosPayload
- messageId: UUID
- originNodeId: UUID
- createdAtEpochMs: Long
- emergencyType: enum
- shortMessage: optional bounded UTF-8 text
- location: optional LocationPayload
```

### `LocationPayload`

```text
LocationPayload
- latitude: Double
- longitude: Double
- accuracyMeters: Float
- capturedAtEpochMs: Long
- quality: PRECISE | APPROXIMATE
```

If location is unavailable, omit the location object and represent that state in local/UI state.

### Emergency type
Keep the enum small, for example:
- `GENERAL`
- `MEDICAL`
- `FIRE`
- `TRAPPED`
- `OTHER`

Do not make an unknown or unsupported emergency type fatal to the whole app; validate and handle protocol versions cleanly.

---

## 5. Mesh envelope

Recommended logical fields:

```text
MeshEnvelope
- protocolVersion: Int
- packetType: PacketType
- packetId: UUID
- senderNodeId: UUID
- lastHopNodeId: UUID?
- hopCount: Int
- maxHops: Int
- priority: enum
- sos: SosPayload?                 // for SOS
- receipt: HopReceiptPayload?      // for HOP_RECEIPT
- deliveryAck: DeliveryAckPayload? // for DELIVERY_ACK
```

For V1 two-phone transfer:
- `hopCount` begins at 0 at origin,
- `maxHops` is still present so the wire format is future-ready,
- forwarding logic is not yet implemented.

A reasonable future default `maxHops` may be chosen later; do not make a routing policy decision part of V1 transport correctness.

---

## 6. Hop receipt

A hop receipt means:

> “The immediate receiving node completely received, validated, deduplicated as appropriate, and durably stored the packet.”

Recommended fields:

```text
HopReceiptPayload
- originalMessageId: UUID
- receiverNodeId: UUID
- receivedAtEpochMs: Long
- status: STORED | ALREADY_PRESENT
```

A hop receipt must **not** be rendered as “delivered to rescue services.”

If the same SOS is resent and already exists, `ALREADY_PRESENT` can still prove that the receiver durably possesses it.

---

## 7. Delivery ACK

A delivery ACK means:

> “A trusted responder/gateway role in the prototype has received the SOS.”

Recommended fields:

```text
DeliveryAckPayload
- originalMessageId: UUID
- gatewayNodeId: UUID
- acknowledgedAtEpochMs: Long
```

For the first demo, a device can explicitly enter **Responder/Test Gateway mode** to generate this ACK.

Do not automatically treat every peer as a responder.

---

## 8. Serialization

Use compact JSON with `kotlinx.serialization` for V1.

Requirements:
- explicit protocol version,
- explicit enums,
- required fields validated after parse,
- unknown future fields may be ignored only within the hard packet-size limit,
- malformed JSON is rejected safely,
- parsing failure never crashes the process.

### Hard size limit
Set a strict upper bound on the serialized protocol packet **before parsing**.

Recommended V1 ceiling:
- **4 KiB per complete protocol packet**

This is intentionally generous for a text SOS but small enough to bound memory and abuse.

### Short message
Recommended maximum:
- **240 UTF-8 bytes**

Validate bytes, not only Kotlin character count.

No images, files, audio or binary attachments in V1.

---

## 9. Transport framing

BLE GATT must treat the serialized packet as a byte sequence that can span many writes.

A transport transfer should include enough metadata to reassemble safely, for example:

```text
Transfer
- transferId
- totalLength
- totalChunks
- sequenceNumber per chunk
- payload bytes
- end/completion condition
- checksum for accidental corruption detection
```

A compact binary frame header is acceptable even though the application packet is JSON.

### Integrity checksum
A CRC/checksum may be used to detect accidental transfer corruption. It is **not** authentication and must never be described as a security signature.

### Chunk size
Derive from the effective BLE connection/MTU where possible. Maintain a conservative fallback. Never hardcode a large MTU as guaranteed across all Android phones.

---

## 10. Receiver state machine

For an incoming SOS:

```text
START
  ↓
accept bounded transfer metadata
  ↓
receive chunks
  ↓
reassemble complete byte array
  ↓
verify length / checksum
  ↓
enforce max packet size
  ↓
parse
  ↓
validate protocol fields
  ↓
check messageId in Room
  ├── already exists → return ALREADY_PRESENT hop receipt
  └── new
       ↓
     persist in Room transaction
       ↓
     commit succeeds
       ↓
     return STORED hop receipt
```

No receipt before persistence.

---

## 11. Sender state machine

For a locally created SOS:

```text
create UUID + payload
  ↓
persist locally
  ↓
discover/select peer
  ↓
connect
  ↓
serialize packet
  ↓
send framed chunks serially
  ↓
wait for hop receipt with timeout
  ├── valid receipt → local state may record peer possession
  └── timeout/disconnect/error → keep packet pending locally
```

Failure never deletes the sender's durable copy.

---

## 12. Deduplication

The SOS `messageId` is the primary deduplication key.

Requirements:
- Room unique index/constraint,
- duplicate insert must be handled intentionally,
- a retransmitted identical SOS must not create a second visible emergency,
- receiving a duplicate may return an `ALREADY_PRESENT` hop receipt,
- future forwarding logic must suppress repeated forwarding storms.

Do not dedupe only by timestamp, sender name, text or location.

---

## 13. Time semantics

Wall-clock timestamps are useful for humans but **must not be trusted for protocol ordering across phones**.

Phone clocks may be wrong or different.

Do not use remote timestamps as authority for ownership, delivery, correctness, or accepting/rejecting packets solely because one phone's time differs. Use explicit state events and IDs.

---

## 14. Validation rules

At minimum reject:
- packet larger than the hard limit,
- invalid UUID syntax,
- unsupported/invalid protocol major version,
- negative hop count,
- nonsensical max-hop value,
- latitude outside `[-90, 90]`,
- longitude outside `[-180, 180]`,
- negative/absurd accuracy values,
- overlong message,
- wrong packet-type/payload combination,
- missing required fields,
- malformed frame/chunk sequence.

Unknown data must not lead to unbounded allocation.

---

## 15. Protocol versioning

Start with a small integer major version, e.g. `1`.

Rules:
- sender includes the version,
- receiver validates compatibility,
- incompatible major versions fail safely with a clear diagnostic,
- future optional fields must not silently change existing semantics.

Do not encode behavior solely in app version strings.

---

## 16. V1 security boundary

The V1 protocol is **defensive but not cryptographically authenticated**.

It must be designed so the immutable `SosPayload` can later be signed without requiring a wire-format rewrite.

Do not add homemade encryption, homemade signatures or key exchange in V1.

See `SECURITY_AND_RELIABILITY.md`.
