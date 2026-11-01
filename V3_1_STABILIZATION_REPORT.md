# ResQMesh V3.1 Stabilization Report

1. **Exact files changed**

   - `app/src/main/java/com/mavens/resqmesh/data/db/SosEntity.kt`
   - `app/src/main/java/com/mavens/resqmesh/data/db/ResQMeshDatabase.kt`
   - `app/src/main/java/com/mavens/resqmesh/domain/mesh/RelayRoutingPolicy.kt`
   - `app/src/main/java/com/mavens/resqmesh/domain/mesh/MeshEngine.kt`
   - `app/src/test/java/com/mavens/resqmesh/domain/mesh/RelayRoutingPolicyTest.kt`
   - `app/src/test/java/com/mavens/resqmesh/domain/mesh/MeshEngineTest.kt`
   - `app/src/androidTest/java/com/mavens/resqmesh/data/db/ResQMeshDatabaseMigrationTest.kt`
   - `app/schemas/com.mavens.resqmesh.data.db.ResQMeshDatabase/1.json`
   - `app/schemas/com.mavens.resqmesh.data.db.ResQMeshDatabase/2.json`
   - `app/build.gradle.kts`
   - `gradle/libs.versions.toml`
   - `V3_1_STABILIZATION_REPORT.md`

2. **Room migration/schema fix**

   `SosEntity` now declares Room database defaults matching the physical V2 schema: `@ColumnInfo(defaultValue = "0")` for `hopCount`, `"8"` for `maxHops`, and `"''"` for `relayAttemptedPeerTokens`. `inboundSenderNodeId` remains nullable with SQLite's implicit `NULL` default. Schema export is enabled and the checked-in V1/V2 schema files support `MigrationTestHelper` validation.

3. **Database version**

   Unchanged at version 2.

4. **MIGRATION_1_2 SQL**

   Unchanged. It remains an additive four-column migration and preserves existing rows; no destructive migration or reset was added.

5. **Real migration test operation**

   `ResQMeshDatabaseMigrationTest` uses AndroidX Room's `MigrationTestHelper` with the real exported V1 schema. It creates a V1 SQLite database, inserts a representative legacy SOS containing optional data, closes it, runs and schema-validates `MIGRATION_1_2`, then queries the migrated row. It proves the row survived and verifies `inboundSenderNodeId = NULL`, `hopCount = 0`, `maxHops = 8`, and `relayAttemptedPeerTokens = ''`.

6. **Migration test execution**

   Actually executed on the connected `SM-A146B` running Android 15: 1 test, 0 failures, 0 errors, 0 skipped. It also compiled successfully as part of the debug Android test source set.

7. **Automatic origin exclusion**

   `RelayRoutingPolicy.selectNextHop` derives the immutable SOS origin's existing `NodeToken` and excludes that token before the existing responder/RSSI/deterministic ordering. If the origin is the only otherwise eligible peer, automatic selection returns `null`.

8. **Manual relay preservation**

   Origin exclusion is confined to automatic `selectNextHop`. The shared manual `isEligible` function was not given the origin restriction, and a regression test proves an origin peer remains manually eligible when it passes the existing self, immediate-sender, and hop-limit rules.

9. **Pending responder-role state**

   `MeshEngine.PendingReceipt` now stores `peerWasAdvertisedResponder`, captured from the selected `DiscoveredPeer.isResponder` at `relaySosTo` start alongside the expected link and peer token.

10. **Non-responder DELIVERY_ACK handling**

    After all existing active-message, link, token, and packet validation, an ACK from a selected peer that was not advertised as a responder is ignored with a sanitized semantic rejection log. No responder-delivery state is persisted, the app does not crash, and the pending operation continues waiting for a valid `HOP_RECEIPT`.

11. **Tests added/changed**

    - Three routing policy regressions: origin skipped in favor of another relay, origin-only yields no automatic hop, and manual origin eligibility remains intact.
    - Two MeshEngine regressions: non-responder ACK rejection followed by normal hop confirmation, and retained wrong-link/wrong-token ACK rejection.
    - Existing responder ACK test continues to prove a responder ACK is persisted distinctly before the hop receipt completes the transfer.
    - One real Room V1-to-V2 migration instrumentation test.

12. **Test results**

    `testDebugUnitTest`: 76 tests, 0 failures, 0 errors, 0 skipped across 8 suites. `connectedDebugAndroidTest`: 1 migration test, 0 failures, 0 errors, 0 skipped.

13. **lintDebug**

    Passed: 0 errors and 21 existing warnings. Report: `app/build/reports/lint-results-debug.html`.

14. **assembleDebug**

    Passed.

15. **APK**

    `app/build/outputs/apk/debug/app-debug.apk`

16. **GATT/framing changes**

    None. No BLE/GATT/framing implementation file was modified.

17. **V2 512-byte fix**

    Intact. The 4 BLE frame-budget tests and 14 framing tests all pass, including MTU 23, MTU 517 capped at 512 bytes, the 498/499-byte boundary, multi-frame reassembly, and maximum 4 KiB packet behavior. The existing `IllegalArgumentException` send guards remain unchanged.

18. **Known limitations**

    The responder advertisement remains prototype metadata and is not cryptographically authenticated. Reverse DELIVERY_ACK propagation was intentionally not added. No physical three-phone relay test was performed in this pass; only the single-device Room instrumentation test ran on hardware.

ROOM V1->V2 MIGRATION: PASS — entity metadata and migration SQL now agree; version remains 2.
REAL MIGRATION TEST: PASS — executed 1/1 on an Android 15 device.
AUTO ORIGIN EXCLUSION: PASS — automatic routing cannot select the original SOS origin.
RESPONDER ACK ROLE CHECK: PASS — non-responder ACKs are ignored without ending hop-receipt wait.
MANUAL RELAY: PRESERVED — origin exclusion is automatic-only.
V2 BLE REGRESSION: PASS — 18/18 BLE framing/budget tests.
UNIT TESTS: PASS — 76/76.
LINT: PASS — 0 errors, 21 warnings.
BUILD: PASS — debug APK produced.
PHYSICAL TESTING: Room migration ran on one physical device; three-phone relay was not physically tested.
