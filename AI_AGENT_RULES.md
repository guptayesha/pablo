# ResQMesh — AI Agent Rules

**Applies to Fable, Grok, Codex, Opus or any other coding agent working in this repository.**

---

## 1. Read before editing

Before writing implementation code, read:

1. `PROJECT_CONTEXT.md`
2. `VERIFIED_ANDROID_SPEC.md`
3. `PROTOCOL_SPEC.md`
4. `V1_SCOPE_AND_ACCEPTANCE.md`
5. `ARCHITECTURE_GUIDELINES.md`
6. `SECURITY_AND_RELIABILITY.md`
7. `BUILD_AND_TEST_PLAYBOOK.md`
8. this file

Do not infer a different product from filenames or previous browser simulation code.

---

## 2. V1 scope is binding

Do not expand iteration 1 into:
- iOS,
- cloud,
- Firebase,
- multi-hop,
- Wi-Fi Direct implementation,
- background permanent services,
- complex routing,
- polished production UI,
- custom cryptography.

Build the smallest real two-phone offline vertical slice first.

---

## 3. No fake completion

Never claim:
- “build passes” without running the build,
- “tests pass” without running tests,
- “BLE works” without distinguishing compile-time from physical-device evidence,
- “offline works” because mocked state changed,
- “delivered” because a relay merely received the packet.

If a physical test cannot be executed autonomously, state exactly what remains for the human to do.

---

## 4. Build, do not only generate code

After implementation changes:

```powershell
.\gradlew.bat testDebugUnitTest
.\gradlew.bat lintDebug
.\gradlew.bat assembleDebug
```

Resolve compile/dependency errors before declaring the iteration complete.

Do not stop immediately after creating files.

---

## 5. Respect the frozen toolchain

Do not casually:
- raise compileSdk/targetSdk to 37,
- migrate to AGP 9.x,
- change Gradle major version,
- change JDK baseline,
- replace Room/Compose lines,
- add large third-party frameworks.

If an actual incompatibility is encountered:
1. show the exact error,
2. verify the dependency constraint,
3. propose the smallest compatible correction.

---

## 6. Architecture ownership

Do not bypass the layers.

Forbidden patterns include:
- raw `BluetoothGatt` logic in Compose screens,
- direct Room DAO writes from UI,
- protocol parsing inside Android Bluetooth callbacks,
- hardcoded phone identifiers/routes,
- a giant singleton containing UI + BLE + database + routing,
- mutable global packet state.

Keep discovery, transport, protocol, storage and UI separable.

---

## 7. Correctness invariants

Never violate:

- sender persists before transfer,
- receiver persists before hop receipt,
- partial transfer is not success,
- GATT write completion is not durable receipt,
- duplicate message ID does not create duplicate SOS,
- hop receipt is not delivery ACK,
- location failure does not block SOS,
- malformed peer input cannot crash the app,
- sender retains the SOS after uncertain failure.

---

## 8. Security behavior

Treat all peer data as untrusted.

Do not:
- trust remote lengths,
- allocate unbounded buffers,
- log full GPS/message payloads by default,
- use IMEI/MAC/serial as identity,
- export components unnecessarily,
- add homemade encryption/signing,
- request irrelevant Android permissions.

Follow `SECURITY_AND_RELIABILITY.md`.

---

## 9. Documentation behavior

Do not overwrite or delete the authoritative handoff files.

If implementation requires a deviation, record:
- original assumption,
- observed error/device behavior,
- evidence/log,
- chosen change,
- impact.

A short implementation/change log may be created for this purpose.

---

## 10. Human/device boundary

The agent may:
- write code,
- run Gradle,
- inspect source,
- run unit/lint tasks,
- use ADB on authorized connected devices,
- inspect logcat.

The agent must not fabricate:
- physical taps,
- permission approvals,
- radio behavior,
- two-phone results it did not observe.

When human interaction is needed, provide the smallest explicit test instruction and wait for the result.

---

## 11. First-agent objective

The first major implementation agent should finish with:

- coherent Android project,
- minimal Compose test UI,
- Room persistence,
- app-scoped node ID,
- SOS/protocol models,
- BLE advertising/scanning,
- BLE GATT transport,
- framing/reassembly,
- hop receipt,
- duplicate protection,
- compatibility/permission handling,
- unit tests for pure logic,
- successful debug build,
- exact APK path,
- clear physical two-phone test instructions.

Do not use remaining time to add unrelated features before the above is stable.
