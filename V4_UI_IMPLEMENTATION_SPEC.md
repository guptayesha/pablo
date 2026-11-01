# ResQMesh V4 UI Implementation Spec

## Hierarchy of truth

1. The working V3.1 Android implementation and persisted state.
2. Authoritative project, protocol, architecture, security, and test documents.
3. The five final `design/v4` PNGs for visual composition.
4. Their HTML files for dimensions, spacing, hierarchy, and component intent.
5. `resqmesh_tactical_lifeline/DESIGN.md` for visual tokens only.

Functional correctness always wins over the Stitch examples.

## Frozen backend

V4 is a native Compose presentation milestone. BLE/GATT, framing, MTU limits, discovery semantics, packet validation, relay routing, Room schema/migrations, repositories, durable receipt semantics, and `MeshEngine` behavior remain unchanged. UI code calls the existing manual relay path and observes existing flows.

## Visual foundation

- Dark navy canvas with tonal card surfaces and thin structural borders.
- Blue communicates mesh/discovery and confirmed next-hop storage.
- Green is reserved for persisted Responder/Test Gateway acknowledgement and ready state.
- Amber communicates stored/waiting state.
- Deep red is concentrated around emergency selection and SOS creation.
- System sans-serif supplies heading/body hierarchy; system monospace is limited to IDs, RSSI, timestamps, hop counts, and compact status telemetry.
- Four bottom destinations only: SOS, Mesh, Inbox, Status. Journey is a back-navigable detail view.

## Semantic corrections to Stitch

- SOS creation stores locally; it does not automatically transmit. The primary action is `CREATE EMERGENCY SOS`.
- A fresh SOS says it is stored locally and ready to send, then guides the user to Mesh.
- Hop receipt means the immediate next peer durably stored the SOS. It is not responder delivery.
- Responder acknowledgement is displayed only from persisted `DELIVERY_ACK` fields.
- `TransferState` is used only for current progress.
- Peer names come from the real discovery token and role metadata. No model names, people, units, batteries, distance estimates, or invented identities appear.
- Diagnostics contain only capability, discovery, transport, responder, location, store configuration, and real bounded log state already exposed to UI.
- Cellular/internet is described as not required, never guessed as connected or disconnected.
- No self-test, cryptography claim, route reconstruction, TTL expiry, background operation, or unsupported radio appears.

## Real-data-only screen behavior

### SOS

Shows the real app node prefix, discovery/peer state, existing `EmergencyType` values, current location attempt state and accuracy, a 240 UTF-8-byte message composer, local creation result, and the latest/selected persisted local SOS. A reliable tap preserves the established creation interaction.

### Mesh

Shows real scan/advertise state and discovered peers. Each peer uses a token-derived short label, advertised relay/responder role, RSSI-derived qualitative signal label, existing routing eligibility, and the real manual relay action. Local SOS remains manually sent; received SOS may auto-relay only under existing foreground V3.1 rules.

### Inbox

Combines real local and received Room records. Safe filters are All, Created Here, and From Peers. Cards distinguish waiting, hop-confirmed, persisted responder-acknowledged, and hop-limit state and open Journey details.

### Journey

Builds a device-local timeline from direction, create/receive times, inbound sender, hop metadata, persisted hop receipt, persisted Delivery ACK, and matching current transfer progress. It never infers intermediate nodes or a complete route.

### Status

Shows real capability snapshot, scan/advertise state, nearby count, transport/server state, current location state, responder toggle wiring, local Room configuration, and real sanitized in-memory diagnostics. Refresh re-reads the actual capability snapshot.

## Data separation

Production composables contain no preview fixtures or mock peers/SOS records. UI-only mapping and formatting helpers derive display values from live domain state and do not persist presentation state.
