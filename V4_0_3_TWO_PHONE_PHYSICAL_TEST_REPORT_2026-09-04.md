# Sampark / ResQMesh V4.0.3 Two-Phone Physical Test Report

## 1 — Environment

Test date: 2026-09-04  
Campaign start: 2026-09-04 10:08:35 +05:30  
Campaign end: 2026-09-04 14:53:28 +05:30

### PHONE A

- Manufacturer: Samsung
- Model: SM-A146B (Galaxy A14 5G)
- Serial: `RZCW10MTE3R`
- Android version: 15
- API: 35

### PHONE B

- Manufacturer: realme
- Model: RMX5033 (Realme Narzo 80 Pro 5G)
- Serial: `S4QWEMKJLRWCKREE`
- Android version: 16
- API: 36

APK: `app/build/outputs/apk/debug/app-debug.apk`  
APK size: 12,483,991 bytes  
APK SHA-256: `E6553B50DF163217EF0AD30444C7A32157AA26D7A91BD7ED754352755E435E7A`

Application package: `com.mavens.resqmesh`  
Public product name: Sampark  
Internal project terminology: ResQMesh

Evidence directory: `physical_test_logs/2026-09-04_two_phone/`

Primary initial log paths:

- `physical_test_logs/2026-09-04_two_phone/phone_A_samsung.log`
- `physical_test_logs/2026-09-04_two_phone/phone_B_realme.log`

Separate frozen phase logs are cited with each transfer below. No production source, package, protocol, Room object, BLE identifier, manifest, toolchain, or build configuration was modified during this campaign.

## 2 — Preflight

- Both devices enumerated as authorized ADB `device` entries: PASS.
- Serial-to-device mapping established from explicit Android properties: PASS.
- Expected APK existed and was not rebuilt or substituted: PASS.
- APK replacement-installed on Samsung: PASS, 10:11:09.
- APK replacement-installed on Realme: PASS, 10:11:36; installed artifact hash independently verified.
- Same installed APK hash on both phones: PASS.
- Package installed on both: PASS; `versionCode=1`, `versionName=0.1.0`, `targetSdk=36`.
- Both applications launched and foregrounded before testing: PASS.
- Bluetooth enabled on both: PASS.
- Required permissions on both: `BLUETOOTH_SCAN`, `BLUETOOTH_ADVERTISE`, `BLUETOOTH_CONNECT`, `ACCESS_FINE_LOCATION`, and `ACCESS_COARSE_LOCATION` granted.
- Normal Wi-Fi disconnected and mobile data OFF on both during the physical transfer campaign: PASS.
- Airplane mode was not required; Bluetooth remained enabled except on unrelated isolated third phones.
- Responder/Test Gateway mode OFF on both normal-test participants: PASS after visible Realme switch correction and advertisement refresh.
- Mesh scanning, advertising, and GATT-server readiness: PASS.
- Apps foregrounded for normal tests: PASS.
- Stable pre-Test-2 discovery: one normal Relay Node each; Samsung saw Realme `#77-8C`, Realme saw Samsung `#5A-7C`.

At 10:27:14, a delayed ADB/OEM package-replacement transaction force-stopped Realme PID 4957. Evidence proves Android `installPackageLI` caused the termination, with no fatal application exception. The user accepted this separate classification:

**HARNESS INTERRUPTION — delayed adb package replacement killed Realme process externally.**

It is not counted as an application crash, BLE failure, historical GATT regression, or Test 2 protocol failure. The recovered transfer baseline began with Samsung PID 30234 and Realme replacement PID 13662. No further reinstall occurred.

## 3 — Acceptance Matrix

| # | Acceptance item | Result | Evidence summary |
|---:|---|---|---|
| 1 | A<->B mutual discovery | PASS | Each phone independently logged peer discovery and displayed exactly one normal Relay Node after role correction. |
| 2 | Samsung A->Realme B | PASS | Fresh `e42c94c7` inserted `STORED` on Realme; receipt returned. |
| 3 | Realme B->Samsung A | PASS | Fresh `1ff98335` inserted `STORED` on Samsung; receipt returned. |
| 4 | Repeated Realme transfers | PASS | Three additional fresh sends, `09be1370`, `d9e6639f`, and `7c4f4512`, all succeeded at MTU 517. |
| 5 | Long SOS | PASS | Actual 240-byte ASCII payload preserved byte-for-byte and stored once. |
| 6 | Physical multi-frame behaviour | PASS | Multiple physical transactions logged two frames; first proof was 508-byte packet with 512-byte and 24-byte writes. |
| 7 | Duplicate handling | PASS | `3b984c6f` remained one row; receiver returned `ALREADY_PRESENT`. |
| 8 | Controlled failed transfer | PASS | Frame 0 confirmed, frame 1 unconfirmed after controlled receiver termination; no receipt or receiver record. |
| 9 | Both processes alive during required normal campaign | PASS | Samsung PID 30234 and recovered Realme PID 13662 remained healthy through required normal tests. |
| 10 | No historical GATT IllegalArgumentException recurrence | PASS | No app `BluetoothGatt.writeCharacteristic` attribute-length `IllegalArgumentException`. |
| 11 | No false Delivery ACK | PASS | No normal Relay Node generated or caused acceptance of a Delivery ACK. |
| 12 | messageId preservation | PASS | Sender creation/receipt and receiver persistence fragments matched for every labeled relay. |
| 13 | Receiver durable record evidence | PASS | Fresh `STORED` logs and UI count increases were observed for Tests 2B, 3, all three repeats, and long SOS. |
| 14 | HOP_RECEIPT evidence | PASS | `STORED` or expected `ALREADY_PRESENT` receipts were logged and accepted. |
| 15 | Physical persistence-before-HOP ordering observability | PASS | Receiver-local logs directly show `persisted: STORED` before reply/HOP receipt generation on multiple fresh transactions. |

## 4 — Per-Transfer Evidence

### Test 2A — Existing Samsung SOS / supporting transaction

TEST: Test 2 initial transaction  
Approximate timestamp: 10:43:59.929–10:44:00.761  
Direction: Samsung A -> Realme B  
Human test label: `TP-A2B-01`  
messageId fragment: `1546c7dd` (`#1546C7`)  
Sender initial state: locally stored; manually selected after recovered baseline  
Receiver initial state: later proven already to contain the same messageId  
Sender transfer result: transport and receipt success  
Receiver durable result: `ALREADY_PRESENT`; fresh insertion by this transaction NOT PROVEN  
HOP_RECEIPT: `ALREADY_PRESENT`  
DELIVERY_ACK: none  
Sender process alive: yes, PID 30234  
Receiver process alive: yes, PID 13662  
Relevant sender log reference: `phone_A_samsung_test2_attempt1_2026-09-04_10-49-45.222.log`  
Relevant receiver log reference: `phone_B_realme_test2_attempt1_2026-09-04_10-49-45.222.log`  
Notes: MTU 517; 508-byte packet; two physical writes of 512 and 24 bytes, both status 0; complete reassembly and duplicate-safe receipt. This was preserved as physical multi-frame and dedupe evidence, but Test 2 required Test 2B for fresh insertion.

### Test 2B — Fresh Samsung-to-Realme normal SOS

TEST: Test 2 final qualifying transaction  
Approximate timestamp: 11:08:34.519–11:08:36.132  
Direction: Samsung A -> Realme B  
Human test label: `TP-A2B-02`  
messageId fragment: `e42c94c7` (`#E42C94`)  
Sender initial state: `STORED LOCALLY · READY TO SEND`; no automatic first hop  
Receiver initial state: 8 stored; new ID absent  
Sender transfer result: 509-byte packet sent in two frames at MTU 517  
Receiver durable result: fresh `STORED`; count 8 -> 9; one row  
HOP_RECEIPT: `STORED`, accepted by Samsung  
DELIVERY_ACK: none  
Sender process alive: yes, PID 30234  
Receiver process alive: yes, PID 13662  
Relevant sender log reference: `phone_A_samsung_test2b_complete_2026-09-04_11-12-49.457.log`, lines 31763, 31766, 31769–31774  
Relevant receiver log reference: `phone_B_realme_test2b_complete_2026-09-04_11-12-49.457.log`, lines 36212, 36315, 36369, 36632, 36654–36666  
Notes: receiver-local ordering was packet complete -> `STORED` at 11:08:34.954 -> receipt at 11:08:34.957.

### Test 3 — Fresh Realme-to-Samsung normal SOS

TEST: Test 3  
Approximate timestamp: 11:27:53–11:27:56  
Direction: Realme B -> Samsung A  
Human test label: `TP-B2A-01`  
messageId fragment: `1ff98335` (`#1FF983`)  
Sender initial state: locally stored and ready; no automatic first hop  
Receiver initial state: 3 stored; new ID absent  
Sender transfer result: 509-byte packet in two frames at MTU 517  
Receiver durable result: fresh `STORED`; count 3 -> 4; one row  
HOP_RECEIPT: `STORED`, accepted by Realme  
DELIVERY_ACK: none  
Sender process alive: yes, PID 13662  
Receiver process alive: yes, PID 30234  
Relevant sender log reference: `phone_B_realme_test3_complete_2026-09-04_11-30-28.497.log`, lines 31998, 32204, 32212, 32217, 32226  
Relevant receiver log reference: `phone_A_samsung_test3_complete_2026-09-04_11-30-28.497.log`, lines 6664, 6736, 6745, 6755–6758  
Notes: receiver-local `STORED` preceded receipt generation by three milliseconds.

### Test 4 — Realme repeat 01

TEST: Test 4 repeat 01  
Approximate timestamp: 11:43:08.160–11:43:10.228  
Direction: Realme B -> Samsung A  
Human test label: `TP-REALME-REPEAT-01`  
messageId fragment: `09be1370` (`#09BE13`)  
Sender initial state: fresh local record; receiver absence proven  
Receiver initial state: 5 stored  
Sender transfer result: 518-byte packet in two frames, MTU 517  
Receiver durable result: fresh `STORED`; count 5 -> 6; one row  
HOP_RECEIPT: `STORED`  
DELIVERY_ACK: none  
Sender process alive: yes, PID 13662  
Receiver process alive: yes, PID 30234  
Relevant sender log reference: `phone_B_realme_test4_repeat01_checkpoint_2026-09-04_11-49-52.447.log`  
Relevant receiver log reference: `phone_A_samsung_test4_repeat01_checkpoint_2026-09-04_11-49-52.447.log`  
Notes: the initially reported Inbox refresh issue was retracted by the human as observation error; no UI live-update defect was observed.

### Test 4 — Realme repeat 02

TEST: Test 4 repeat 02  
Approximate timestamp: 12:59:49.947–12:59:52.246  
Direction: Realme B -> Samsung A  
Human test label: `TP-REALME-REPEAT-02`  
messageId fragment: `d9e6639f` (`#D9E663`)  
Sender initial state: fresh local record; unrelated peers isolated  
Receiver initial state: 9 stored; new ID absent  
Sender transfer result: 519-byte packet in two frames, MTU 517  
Receiver durable result: fresh `STORED`; count 9 -> 10; one row  
HOP_RECEIPT: `STORED`  
DELIVERY_ACK: none  
Sender process alive: yes, PID 13662  
Receiver process alive: yes, PID 30234  
Relevant sender log reference: `phone_B_realme_test4_repeat02_complete_2026-09-04_13-03-44.815.log`, lines 46942, 47112, 47185, 47187–47194  
Relevant receiver log reference: `phone_A_samsung_test4_repeat02_complete_2026-09-04_13-03-44.815.log`, lines 12276, 12375, 12381, 12390–12393  
Notes: clean replacement boundary was established after isolating Krrish's A17 and OnePlus Nord CE 5G.

### Test 4 — Realme repeat 03

TEST: Test 4 repeat 03  
Approximate timestamp: 13:19:35.622–13:19:37.959  
Direction: Realme B -> Samsung A  
Human test label: `TP-REALME-REPEAT-03`  
messageId fragment: `7c4f4512` (`#7C4F45`)  
Sender initial state: fresh local record; exactly one authorized peer  
Receiver initial state: 10 stored; new ID absent  
Sender transfer result: 519-byte packet in two frames, MTU 517  
Receiver durable result: fresh `STORED`; count 10 -> 11; one row  
HOP_RECEIPT: `STORED`  
DELIVERY_ACK: none  
Sender process alive: yes, PID 13662  
Receiver process alive: yes, PID 30234  
Relevant sender log reference: `phone_B_realme_test4_repeat03_complete_2026-09-04_13-22-48.451.log`, lines 49376, 49395, 49662, 49944, 49958–49971  
Relevant receiver log reference: `phone_A_samsung_test4_repeat03_complete_2026-09-04_13-22-48.451.log`, lines 13994, 14055, 14075, 14082–14085  
Notes: third and final planned additional Realme send.

### Test 5 — Long SOS

TEST: Test 5  
Approximate timestamp: 13:50:43.179–13:50:45.448  
Direction: Realme B -> Samsung A  
Human test label: `TP-LONG-01` prefix  
messageId fragment: `3b984c6f` (`#3B984C`)  
Sender initial state: actual stored 240-byte ASCII payload captured as baseline  
Receiver initial state: 11 stored; new ID absent  
Sender transfer result: 739-byte packet in two frames, MTU 517  
Receiver durable result: fresh `STORED`; count 11 -> 12; one row  
HOP_RECEIPT: `STORED`  
DELIVERY_ACK: none  
Sender process alive: yes, PID 13662  
Receiver process alive: yes, PID 30234  
Relevant sender log reference: `phone_B_realme_test5_long_complete_2026-09-04_13-54-30.067.log`, lines 206529, 206586, 206705, 206835, 206841–206877  
Relevant receiver log reference: `phone_A_samsung_test5_long_complete_2026-09-04_13-54-30.067.log`, lines 16212, 16283, 16290, 16301–16304  
Notes: receiver payload hash exactly matched the captured sender payload hash.

### Test 6 — Same-ID duplicate rerelay

TEST: Test 6  
Approximate timestamps: 14:09:44.735–14:09:49.087 and 14:14:07.986–14:14:11.954  
Direction: Realme B -> Samsung A  
Human test label: existing long SOS  
messageId fragment: `3b984c6f` (`#3B984C`)  
Sender initial state: existing locally created, hop-confirmed SOS; natural rerelay path enabled  
Receiver initial state: 12 stored; same messageId present once  
Sender transfer result: both observed GATT cycles delivered 739 bytes in two frames at MTU 517  
Receiver durable result: `ALREADY_PRESENT` both times; count remained 12; one row  
HOP_RECEIPT: `ALREADY_PRESENT`, accepted both times  
DELIVERY_ACK: none  
Sender process alive: yes, PID 13662  
Receiver process alive: yes, PID 30234  
Relevant sender log reference: `phone_B_realme_test6_duplicate_complete_2026-09-04_14-16-12.619.log`, lines 35485–35489, 36241, 36607, 36612, 53562–53566, 54265, 54288, 54296  
Relevant receiver log reference: `phone_A_samsung_test6_duplicate_complete_2026-09-04_14-16-12.619.log`, lines 8554, 8557, 8559–8560, 12695–12698  
Notes: the human reported one intended rerelay, but logs contain two distinct UI-associated GATT cycles, each preceded by a separate touch-up event. No automatic-relay marker or third transfer occurred. The discrepancy is retained as test-interaction/harness ambiguity, not an automatic forwarding storm.

### Test 7 — Controlled interrupted transfer

TEST: Test 7  
Approximate timestamp: 14:40:44.384–14:40:54.503  
Direction: Realme B -> Samsung A  
Human test label: `TP-FAIL-01`  
messageId fragment: `fadb032f` (`#FADB03`)  
Sender initial state: fresh local record; Samsung count 12 and ID absent  
Receiver initial state: app alive at PID 30234; MTU 517 link established  
Sender transfer result: `Failed — write not confirmed at frame 1`  
Receiver durable result: no complete packet, no persistence event, no row; final count 12  
HOP_RECEIPT: none  
DELIVERY_ACK: none  
Sender process alive: yes, PID 13662 unchanged  
Receiver process alive: intentionally force-stopped; healthy manual relaunches observed, final PID 7477  
Relevant sender log reference: `phone_B_realme_test7_controlled_failure_complete_2026-09-04_14-53-28.937.log`, lines 19285, 19668, 19687, 20555, 20557  
Relevant receiver log reference: `phone_A_samsung_test7_controlled_failure_complete_2026-09-04_14-53-28.937.log`, lines 4962, 5031–5032, 5039, 5045, 5047, 7410, 7697  
Notes: frame indices are zero-based; failure at frame 1 after sequential confirmation proves frame 0 completed and the second frame did not confirm. Samsung Mesh remained OFF after the later human relaunch, and no automatic retry occurred.

## 5 — Realme Historical Crash Regression

Historical failure:  
`BluetoothGatt.writeCharacteristic -> IllegalArgumentException` due to attempting a GATT attribute value larger than 512 bytes when negotiated MTU was 517.

Frozen fix being physically regressed:  
`safeFrameBudget(mtu) = min(mtu - 3, 512)`

Observed Realme MTU if logs expose it: 517 on all qualifying Realme sender links, including the controlled-failure setup.

Number of successful Realme-as-sender transfers in this campaign: 7 physically logged successful GATT transfer cycles in authorized labeled tests: Test 3, three Test 4 repeats, Test 5, and two Test 6 duplicate cycles. Five of these were fresh normal/long transfers; Test 6 supplied duplicate-safe transfers. Test 7 added one deliberately interrupted, correctly failed send.

Did Realme process die during required transfer tests? NO.  
Did `BluetoothGatt.writeCharacteristic` attribute-length `IllegalArgumentException` recur? NO.  
Did any relevant FATAL EXCEPTION occur? NO.

Verdict for historical crash regression: **PASS**

The three mandatory additional repeat sends all succeeded from unchanged Realme PID 13662. Including Test 3, four consecutive fresh Realme-to-Samsung normal transfers completed at MTU 517 before the long and duplicate tests.

Realme/OPlus system Bluetooth PID 517 repeatedly emitted a pre-negotiation ATT warning that an unrelated shared attribute value was truncated to 19 bytes. It was not emitted by the ResQMesh Java process, did not become `BluetoothGatt.writeCharacteristic -> IllegalArgumentException`, and did not impair any qualifying transfer.

## 6 — Long / Multi-Frame

Test message byte count: 240 UTF-8 bytes, plain ASCII  
Direction: Realme B -> Samsung A  
messageId: `3b984c6f...` (`#3B984C`)  
Transfer result: PASS; 739-byte complete protocol packet  
Receiver content preserved: PASS, exact byte equality  
Sender payload SHA-256: `DF96DFA211E86597F46CA531AAACC48A50CE16A04FE481D79E2741C2BF609F28`  
Receiver payload SHA-256: `DF96DFA211E86597F46CA531AAACC48A50CE16A04FE481D79E2741C2BF609F28`  
Receiver durable row count/result: one fresh row, `STORED`; total 11 -> 12  
HOP_RECEIPT: `STORED`  
Process health: both PIDs unchanged and healthy

Evidence of number of frames: Realme production transport logged `packet sent, 739 bytes in 2 frames`; Samsung logged complete 739-byte packet reception and subsequent persistence.

**LONG SOS: PASS**  
**MULTI-FRAME PHYSICAL VALIDATION: PASS**

This is additional corroboration. Physical multi-frame behavior was already proven by the earlier 508-byte Samsung send using directly logged 512-byte and 24-byte writes.

## 7 — Duplicate

Original messageId: `3b984c6f...` (`#3B984C`)  
Second relay attempt: naturally exposed by the unmodified UI; no database or packet manipulation  
Receiver duplicate-row result: no new row; stored count remained 12; one visible `#3B984C`  
Receipt: `ALREADY_PRESENT`  
Forwarding behavior: no automatic-relay marker and no continuing storm  
Process health: both unchanged and healthy

Two UI-associated transfer cycles were physically logged despite one intended human rerelay. Both independently produced `ALREADY_PRESENT`, no duplicate row, and no downstream forwarding loop. The discrepancy is preserved in `TEST6_DUPLICATE_EVIDENCE_3B984C_2026-09-04.md`.

**DUPLICATE HANDLING: PASS**

## 8 — Controlled Failure

Direction: Realme B -> Samsung A  
messageId: `fadb032f...` (`#FADB03`)  
Interruption type: one-shot ADB force-stop of receiver app only  
Interruption timing: immediately after Samsung GATT server logged notifications enabled at MTU 517  
Sender post-failure state: local SOS retained; `WAITING FOR RELAY`; no next-peer durable confirmation  
Receiver post-failure state: no complete packet event, no persistence event, stored count remained 12, `#FADB03` absent  
HOP_RECEIPT result: none generated or accepted  
Packet-holder/durable-state result: sender state did not falsely advance; receiver gained no false complete row  
Retry availability: retained SOS remains available for manual retry when an eligible peer is present  
Process health: Realme remained PID 13662; Samsung termination was intentional and later manual launches were healthy

The sender reported `write not confirmed at frame 1`. Because frame indices are zero-based and writes are sequentially confirmed, frame 0 had completed before the second frame failed. The interruption therefore established meaningful partial-transfer failure semantics.

The human manually relaunched Samsung after observing the forced-stop outcome. No automatic retry or new transfer occurred during the following five-minute interval with Mesh OFF. That relaunch is classified separately from the intentional interruption.

**CONTROLLED FAILED TRANSFER: PASS**

## 9 — Crash / Error Summary

FATAL EXCEPTION: **NO**  
Relevant AndroidRuntime process crash: **NO**  
Room failure: **NO**  
SQLite failure: **NO** — no application-correlated failure; unrelated Samsung system battery-statistics errors were excluded by PID/package  
GATT-length IllegalArgumentException: **NO**  
BluetoothGatt.writeCharacteristic fatal failure: **NO**  
Protocol fatal failure: **NO**  
Framing/reassembly fatal failure: **NO**  
Unexpected process death: **NO**  
False responder Delivery ACK: **NO**  
messageId mutation: **NO**  
Durable-state advancement after failed transfer: **NO**

Excluded intentional/external lifecycle events:

- 10:27 Realme PID 4957 termination: delayed ADB/OEM package replacement (`installPackageLI`), accepted harness interference.
- Test 7 Samsung PID 30234 termination: deliberate controlled receiver force-stop.
- Post-Test-7 Samsung PID 5827 termination: Android `remove task` after the separate human relaunch, with no fatal exception.

Unexpected third ResQMesh-capable peers were detected between Test 4 boundaries. Krrish's A17 and OnePlus Nord CE 5G were physically isolated with Bluetooth OFF; contaminated intervals were preserved but not used as qualifying repeat evidence. Their bounded traffic produced no crash, false Delivery ACK, duplicate row, or forwarding storm.

## 10 — Known Limitations

The campaign did not classify these accepted constraints as defects:

- Foreground-only mesh operation.
- One outgoing transfer at a time.
- Locally created SOS records require a manual first hop.
- Received SOS records may auto-relay under existing V3.1 foreground rules.
- Reverse responder Delivery ACK propagation to the original origin is not implemented.
- Responder identity is prototype metadata and is not cryptographically authenticated.
- Wi-Fi Direct is not implemented.
- iOS is not implemented.
- Location may be precise, approximate, or unavailable.

## Final Verdict

TWO-PHONE V4.0.3 PHYSICAL ACCEPTANCE: PASS

All required core acceptance conditions were physically proven. Optional tests also completed: long SOS PASS, physical multi-frame PASS, duplicate handling PASS, and controlled failed transfer PASS.

READY FOR THREE-PHONE ACCEPTANCE CAMPAIGN
