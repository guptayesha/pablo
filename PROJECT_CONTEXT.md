# ResQMesh — Project Context

**Status:** Authoritative project context for the Android MVP  
**Team:** Mavens  
**Project:** ResQMesh  
**Verified through:** 2026-08-31

---

## 1. What ResQMesh is

ResQMesh is an **infrastructure-independent emergency messaging system** for disaster situations where normal cellular networks or internet connectivity may be unavailable or unreliable.

The core idea is:

1. A victim creates an SOS on an Android phone.
2. Nearby ResQMesh phones discover one another using local radios.
3. The SOS is transferred locally to another phone.
4. A receiving phone stores the SOS durably.
5. Later versions can relay the same SOS across multiple phones using store-carry-forward behavior.
6. Eventually the SOS can reach a responder phone or a gateway that has another form of backhaul.

The core product does **not** require cellular data, a SIM card, a Wi-Fi router, a cloud database, or an internet connection. It does require compatible local wireless radios to be enabled.

---

## 2. Accurate product claim

Use this wording as the technical baseline:

> **ResQMesh enables infrastructure-independent, store-and-forward emergency messaging across nearby smartphones, allowing SOS packets to propagate opportunistically until they reach a responder or gateway.**

Do **not** claim guaranteed delivery everywhere, unlimited range, that Android provides a ready-made mesh API, that a packet can move when there is no physical sequence of encounters between devices, or that V1 has complete cryptographic trust or authentication.

ResQMesh creates the mesh behavior at the **application layer** using local peer-to-peer links supplied by Android.

---

## 3. Current project stage

Earlier work includes:
- problem selection and feasibility analysis,
- presentation/submission material,
- protocol and routing discussions,
- a browser-based simulation of multi-hop forwarding and failure scenarios,
- review of selective/battery-aware routing,
- store-carry-forward behavior,
- responder/gateway concepts,
- optional ResQHub / radio backhaul concepts.

The browser simulation is **reference material only**. It is not the Android implementation and its source code should not be copied into the Android project.

The current priority is a **real installable Android APK** that proves the core offline transfer on physical phones.

---

## 4. Immediate development objective

The first hard milestone is deliberately limited to **two Android phones**.

### Phone A
- user creates an SOS,
- SOS is persisted locally,
- discovers Phone B,
- establishes a BLE connection,
- transfers the SOS,
- receives a hop receipt only after Phone B has durably stored the SOS.

### Phone B
- advertises/discovers as a ResQMesh peer,
- receives a framed/chunked SOS,
- reassembles it,
- validates it,
- deduplicates it,
- persists it,
- displays it in the minimal UI,
- sends a hop receipt.

A separate **Responder/Test Gateway mode** may be used to demonstrate a true end-to-end delivery acknowledgement.

---

## 5. Development ladder

Do not jump ahead of this sequence.

### Milestone 1 — Two-phone vertical slice
- minimal functional UI,
- local SOS creation,
- local persistence,
- BLE discovery,
- BLE GATT transfer,
- receiving and persistence,
- hop receipt,
- duplicate protection,
- basic failure handling,
- physical-device test without internet.

### Milestone 2 — Three-phone relay
- A → B → C,
- forwarding state,
- hop count / TTL,
- packet-holder correctness,
- dedupe across relay nodes.

### Milestone 3 — Store-carry-forward
- no next hop available,
- packet remains stored,
- later peer encounter,
- forwarding resumes correctly.

### Milestone 4 — Better routing
- battery/capability-aware decisions,
- bounded retry policy,
- peer quality metrics where justified.

### Milestone 5 — Responder/gateway behavior
- actual delivery ACK,
- responder inbox,
- optional backhaul integration.

### Milestone 6 — Product polish and security expansion
- polished UI,
- stronger application-level trust/signing,
- abuse/replay controls,
- background operation where justified,
- optional Wi-Fi Direct high-throughput path,
- optional gateway hardware/radio integration.

---

## 6. V1 non-goals

The first implementation must **not** attempt all of these at once:

- iOS support,
- cloud backend,
- Firebase dependency for the core path,
- three-or-more-phone routing,
- complex battery routing,
- ResQHub hardware,
- VHF/UHF integration,
- satellite integration,
- images/video/audio attachments,
- polished competition UI,
- background-always-on disaster mode,
- custom PKI,
- improvised end-to-end encryption,
- Wi-Fi Direct as a mandatory first transport.

These are future layers, not requirements for the first APK.

---

## 7. Design principles inherited from simulator work

The Android implementation must preserve these invariants:

1. **Never hardcode scenario-specific routing patches.**
2. **Never trust a “PASS” claim without actually running the build/test.**
3. **The current holder is the last device that has confirmed durable receipt.**
4. **Failed or partial transfers must not advance packet state.**
5. **Persistence happens before receipt acknowledgement.**
6. **A relay receipt is not the same as responder delivery.**
7. **Duplicate packets must not create duplicate SOS records or forwarding storms.**
8. **Every state transition must be explainable by an actual event.**
9. **The networking core must remain independent of the UI.**
10. **Transports must be replaceable without rewriting the protocol or storage layers.**

---

## 8. Terminology

### SOS
The immutable emergency content created by the origin device.

### Mesh envelope
Mutable forwarding metadata around an SOS or control packet.

### Hop receipt
Confirmation that the **next relay** successfully validated and durably persisted a packet.

### Delivery ACK
Confirmation that a **responder/gateway** received the SOS. This is the only event that should produce a user-facing message equivalent to “delivered to responders.”

### Store-carry-forward
A device stores an undelivered SOS when no suitable peer is available and retries when a future peer appears.

### Transport
The mechanism used to move bytes between two peers. BLE GATT is the V1 transport. Wi-Fi Direct is a later transport behind the same abstraction.

---

## 9. Source-of-truth order

When implementation decisions conflict, use this priority:

1. `V1_SCOPE_AND_ACCEPTANCE.md`
2. `VERIFIED_ANDROID_SPEC.md`
3. `PROTOCOL_SPEC.md`
4. `SECURITY_AND_RELIABILITY.md`
5. `ARCHITECTURE_GUIDELINES.md`
6. `BUILD_AND_TEST_PLAYBOOK.md`
7. `PROJECT_CONTEXT.md`
8. `AI_AGENT_RULES.md` for agent behavior

If a real Android API/device behavior contradicts a document, **do not silently work around it**. Record the contradiction, verify it, and make the smallest justified adjustment.
