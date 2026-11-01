# ResQMesh — Security and Reliability Baseline

**Scope:** Security-conscious V1, not a claim of complete production-grade trust.

---

## 1. Security position

V1 must be:
- defensively coded,
- permission-minimized,
- robust against malformed local packets,
- safe with user/emergency data,
- failure-correct.

V1 is **not yet a complete authenticated emergency network**.

Do not claim end-to-end encryption, verified human identity, verified responder identity, cryptographic non-repudiation, complete replay resistance, or “military-grade” security.

Those require a coherent key/trust architecture.

---

## 2. Threats V1 must already handle

Assume nearby peers are untrusted.

Handle at minimum:

- malformed JSON,
- malformed transfer frames,
- oversized packet declarations,
- invalid UUIDs,
- invalid coordinates,
- overlong text,
- repeated/duplicate SOS,
- repeated connection attempts,
- interrupted transfers,
- stale or unrelated receipts,
- spoofed node IDs,
- unexpected protocol versions,
- resource-exhaustion attempts,
- invalid state transitions.

A nearby device must not be able to crash ResQMesh simply by sending bad bytes.

---

## 3. Hard limits

Enforce limits before expensive work.

Recommended:
- maximum complete protocol packet: **4 KiB**
- short message: **240 UTF-8 bytes**
- bounded transfer/chunk count,
- bounded simultaneous transfer sessions,
- bounded retry count/time,
- bounded discovered-peer list retention.

Never allocate memory solely from an untrusted `totalLength` field without checking the hard maximum first.

---

## 4. Input validation

Validate:
- UUID syntax,
- protocol version,
- packet type/payload combination,
- message byte length,
- coordinates,
- accuracy,
- hop counters,
- frame sequence,
- expected transfer ID,
- checksum where used.

Do not use remote text directly as a file path, database table/column name, shell command or format string.

Room uses parameterized queries/DAO APIs; do not construct SQL from peer input.

---

## 5. Duplicate and replay baseline

Use `messageId` as the dedupe key with a database uniqueness constraint.

A duplicate SOS:
- does not create another emergency record,
- may return an `ALREADY_PRESENT` hop receipt,
- does not trigger uncontrolled duplicate forwarding later.

This is a **basic replay/duplicate defense**, not full cryptographic replay protection. A malicious peer can still fabricate a new UUID in V1.

---

## 6. Receipt authenticity limitation

A V1 hop receipt proves only that the currently connected application peer reported successful storage under the implemented session.

Without application-level identity/signatures, it does not prove the real-world identity of the peer, that the peer is a responder, or that a rescue authority saw the SOS.

Therefore:
- distinguish hop receipt from delivery ACK,
- only explicit Responder/Test Gateway mode produces the demo delivery ACK,
- label prototype trust limitations honestly.

---

## 7. Data storage

Use app-private Android storage.

Recommended defaults:
- Room database in internal app storage,
- node ID in app-private preferences/storage,
- no external/shared-storage database,
- do not expose SOS records through exported providers,
- disable unnecessary cloud backup of emergency data for the prototype.

Do not claim Room data is independently encrypted merely because it is app-private.

Android device-level storage encryption may protect the device, but it is not the same as application-specific database encryption.

---

## 8. Android component exposure

Default to non-exported components.

Rules:
- only the launcher activity should be exported when required by its launcher intent filter,
- services/receivers/providers should be `exported=false` unless a documented need exists,
- use appropriate receiver registration flags on modern Android,
- do not expose a custom IPC surface in V1.

---

## 9. Permissions

Request only what the implemented feature requires.

V1 should not request:
- contacts,
- SMS,
- microphone,
- camera,
- call logs,
- phone state,
- storage/media access,
- accessibility service,
- device admin.

Nearby-device and location permission requests must have a clear UI rationale tied to discovering nearby ResQMesh phones and attaching optional location to an SOS.

If the user denies location, the SOS still works without coordinates.

---

## 10. Logging

Debugging is necessary, but logs must be intentionally sanitized.

Do not routinely log:
- full SOS message body,
- exact GPS coordinates,
- full node UUIDs,
- full message UUIDs,
- database contents.

Prefer:
- short ID prefixes,
- packet type,
- byte length,
- state transition,
- error category,
- timing.

Never log future cryptographic keys/tokens.

Release builds should reduce diagnostic verbosity.

---

## 11. BLE reliability

Treat BLE as unreliable/disconnectable.

Requirements:
- one controlled GATT operation at a time,
- transfer timeout,
- disconnect cleanup,
- reassembly keyed by transfer ID,
- strict sequence validation,
- no success on partial data,
- persistence before hop receipt,
- sender retains durable copy after any uncertain failure.

A connection callback is not proof of message delivery.

A successful GATT write callback is not proof of durable receiver storage.

Only the application-level hop receipt establishes V1 hop success.

---

## 12. Resource exhaustion

Protect against local abuse:

- cap packet size,
- cap active transfer sessions,
- discard stale incomplete sessions,
- cap queue sizes,
- rate-limit repeated malformed transfer attempts if needed,
- avoid infinite retry loops,
- use timeouts,
- clean up GATT objects,
- ensure scan/advertise operations stop when no longer needed.

Do not leave indefinite scanning/transfers running because of an exception path.

---

## 13. Location privacy

Location is sensitive emergency data.

Rules:
- collect only when creating an SOS or when the user explicitly enables the relevant behavior,
- store capture time and accuracy,
- do not continuously collect location for V1,
- do not send location in BLE advertisements,
- do not upload it to a cloud service,
- display a clear unavailable/approximate/precise state.

---

## 14. Cryptography roadmap

Architecture must leave room for later:

- origin key pair,
- signed immutable `SosPayload`,
- responder/gateway identity,
- trusted key distribution or certificate model,
- signed delivery ACK,
- nonce/replay windows where appropriate,
- optional confidentiality if threat model justifies it.

Do not create a homemade cryptosystem in V1.

If cryptography is added later, use established Android/Java cryptographic primitives and Keystore-based key protection where appropriate, followed by dedicated review.

---

## 15. Security review gate

After the first working two-phone implementation:

1. run static/lint checks,
2. review manifest permissions and exported components,
3. review all packet parsing and size limits,
4. review GATT lifecycle and timeouts,
5. review logs for emergency-data leakage,
6. review Room uniqueness/state transitions,
7. run malformed/oversized/duplicate tests,
8. perform an independent code review before multi-hop expansion.

Security quality is a continuing engineering process, not a one-time checkbox.
