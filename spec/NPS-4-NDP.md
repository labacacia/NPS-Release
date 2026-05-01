English | [中文版](./NPS-4-NDP.cn.md)

# NPS-4: Neural Discovery Protocol (NDP)

**Spec Number**: NPS-4  
**Status**: Proposed  
**Version**: 0.6  
**Date**: 2026-05-01  
**Port**: 17433 (default, shared) / 17436 (optional dedicated)  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Depends-On**: NPS-1 (NCP v0.6), NPS-3 (NIP v0.4)  

---

## 1. Terminology

The key words "MUST", "SHOULD", and "MAY" in this document are to be interpreted as described in RFC 2119.

---

## 2. Protocol Overview

NDP is DNS for the AI era. Agents use NDP to resolve `nwp://` addresses to physical endpoints, broadcast their own capabilities, and subscribe to node-graph changes. NDP supports both decentralized and centralized network topologies.

### 2.1 Resolution Modes

| Mode | Use case | Priority |
|------|----------|----------|
| Local registry (UDP multicast) | Intranet / LAN | Highest |
| DNS TXT records | Public internet, distributed | Medium |
| NPS Cloud Registry | Global, centralized | Lowest (fallback) |

---

## 3. Frame Types

### 3.1 AnnounceFrame (0x30)

A node or agent broadcasts its presence and capabilities.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | required | Fixed value `0x30` |
| `nid` | string | required | Publisher NID |
| `node_type` | string | conditional | Node type (required for node entities) |
| `addresses` | array | required | Physical address list; each entry carries `host` / `port` / `protocol` |
| `capabilities` | array | required | Capability list (reuses the NIP capability vocabulary) |
| `ttl` | uint32 | required | Broadcast validity in seconds; `0` = offline notification |
| `timestamp` | string | required | Broadcast time (ISO 8601 UTC) |
| `activation_mode` | string | conditional | One of `ephemeral` / `resident` / `hybrid`. REQUIRED for publishers claiming NPS-Node Profile L1+ compliance. OPTIONAL for pre-L1 publishers. Receivers MUST treat an absent field as `ephemeral` (backward compatibility with NPS v1.0-alpha.2 publishers). See §3.1.1. |
| `node_roles` | array of strings | optional | Node-functionality role(s) carried by this publisher. Each value is one of `"memory"`, `"action"`, `"complex"`, `"anchor"`, `"bridge"`. The legacy value `"gateway"` was removed in v1.0-alpha.3 (NPS-CR-0001); parsers MUST reject it with `NDP-ANNOUNCE-ROLE-REMOVED`. Any other unrecognized value MUST be rejected with `NDP-ANNOUNCE-ROLE-UNKNOWN`. Single-role nodes may send a one-element array (`"node_roles": ["memory"]`). Absent means "single role per `node_type`" — i.e. the receiver SHOULD fall back to the `node_type` field. Parsers MUST also accept the legacy field name `node_kind` as an alias for `node_roles` during the alpha transition window. (NPS-CR-0001; renamed from `node_kind` in NDP v0.6 — see M1 naming fix) |
| `cluster_anchor` | string (NID) | optional | For non-Anchor nodes joining a cluster, identifies the Anchor Node they register with. Absent for standalone nodes and for Anchor Nodes themselves. (NPS-CR-0001) |
| `bridge_protocols` | array of strings | optional | For nodes declaring `"bridge"` in `node_roles`, lists supported external protocols. Standard values: `"http"`, `"grpc"`, `"mcp"`, `"a2a"`. Open-ended; third-party adapters MAY register additional values via future CRs. MUST be absent for nodes that do not declare `"bridge"`. (NPS-CR-0001) |
| `activation_endpoint` | object | conditional | Push target for `resident` / `hybrid` publishers; same shape as an `addresses[]` entry. REQUIRED when `activation_mode ∈ {resident, hybrid}`; MUST be absent otherwise. |
| `spawn_spec_ref` | string | optional | Opaque reference the publishing daemon can resolve to construct an agent process on demand. Meaningful for `ephemeral` and `hybrid` cold start. Content schema is standardized at NPS-Node Profile L3 (see future NPS-Daemon-Spec). |
| `signature` | string | required | Signature with IdentFrame private key; prevents forgery |

**Example (L1+ publisher, ephemeral)**

```json
{
  "frame": "0x30",
  "nid": "urn:nps:node:api.example.com:products",
  "node_type": "memory",
  "addresses": [
    { "host": "10.0.0.5", "port": 17434, "protocol": "nwp" }
  ],
  "capabilities": ["nwp:query", "nwp:stream"],
  "ttl": 300,
  "timestamp": "2026-04-10T00:00:00Z",
  "activation_mode": "ephemeral",
  "signature": "ed25519:..."
}
```

**Example (L2+ publisher, resident)**

```json
{
  "frame": "0x30",
  "nid": "urn:nps:agent:labacacia:writer-42",
  "node_type": "agent",
  "addresses": [
    { "host": "10.0.0.5", "port": 17434, "protocol": "nwp" }
  ],
  "capabilities": ["nwp:invoke"],
  "ttl": 300,
  "timestamp": "2026-04-24T00:00:00Z",
  "activation_mode": "resident",
  "activation_endpoint": { "host": "10.0.0.5", "port": 17440, "protocol": "nwp" },
  "signature": "ed25519:..."
}
```

#### 3.1.1 Activation semantics

`activation_mode` tells receivers whether the publisher expects outbound traffic by push or by pull:

| Mode | Sender behavior | Receiver expectation |
|------|-----------------|----------------------|
| `ephemeral` | Deliver via inbox + pull; publisher may not be running when the frame arrives. | Frame is queued in the publisher's per-NID inbox. |
| `resident` | Push over a long-lived connection to `activation_endpoint`. | Publisher accepts pushed frames on the declared endpoint. |
| `hybrid` | Attempt push to `activation_endpoint` first; fall back to inbox if the endpoint is unreachable within the publisher's wake budget. | Publisher wakes from hibernation on first frame; push target resumes once awake. |

**Backward compatibility**: NPS v1.0-alpha.2 publishers do not emit `activation_mode`. Receivers MUST handle an absent field as `ephemeral` so that alpha.2 AnnounceFrames continue to resolve correctly. A receiver MUST NOT reject an AnnounceFrame solely for lacking the field.

**Conformance reference**: See [NPS-Node Profile §1.3 and §6](./services/NPS-Node-Profile.md) for the compliance framing and per-level requirements around these fields.

---

### 3.2 ResolveFrame (0x31)

Resolves an `nwp://` URL to a physical endpoint.

**Request**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | required | Fixed value `0x31` |
| `target` | string | required | The `nwp://` URL to resolve |
| `requester_nid` | string | optional | Requester NID (for authorization checks) |

**Response**

```json
{
  "target": "nwp://api.example.com/products",
  "resolved": {
    "host": "10.0.0.5",
    "port": 17434,
    "cert_fingerprint": "sha256:a3f9...",
    "ttl": 300
  }
}
```

---

### 3.3 GraphFrame (0x32)

Node-graph synchronization and change subscription.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | required | Fixed value `0x32` |
| `initial_sync` | bool | required | `true` = full push, `false` = incremental patch |
| `nodes` | array | conditional | Node list for a full sync |
| `patch` | array | conditional | JSON Patch (RFC 6902) for an incremental sync |
| `seq` | uint64 | required | Monotonically increasing graph version sequence |

---

## 4. Agent Initialization Flow

```
Agent starts
  │
  ├─ 1. NIP: load IdentFrame (from file or fetch from CA)
  │
  ├─ 2. NDP: send AnnounceFrame (broadcast agent online)
  │
  ├─ 3. NDP: send GraphFrame subscription (receive node graph)
  │         ← GraphFrame(initial_sync=true)  full graph
  │         ← GraphFrame(initial_sync=false) incremental updates (ongoing)
  │
  └─ Ready: can initiate NWP requests
```

---

## 5. DNS TXT Record Specification

A node MAY publish discovery information via DNS TXT records without requiring a central registry:

```
# Node discovery record
_nps-node.api.example.com.  IN TXT  "v=nps1 type=memory port=17434 nid=urn:nps:node:api.example.com:products fp=sha256:a3f9..."

# CA discovery record
_nps-ca.mycompany.com.      IN TXT  "v=nps1 ca=https://ca.mycompany.com/.well-known/nps-ca"
```

**TXT record keys**

| Key | Required | Description |
|-----|----------|-------------|
| `v` | required | Version; fixed `nps1` |
| `type` | optional | Node type: `memory` / `action` / `complex` |
| `port` | optional | Port (default 17434) |
| `nid` | required | Node NID |
| `fp` | optional | Node certificate fingerprint |
| `ca` | conditional | CA discovery endpoint (required for CA records) |

> **Version-string formats**: The DNS TXT `v` key uses the compact value `nps1` following DNS TXT record conventions (lowercase, no spaces). The NCP native-mode connection preamble uses the RFC-style token `NPS/1.0\n` (see NPS-1 §2.6.1). Both identify NPS protocol version 1, but in different encoding contexts; they are not interchangeable and operate at different protocol layers.

---

## 6. Error Codes

| Error Code | NPS Status | Description |
|------------|------------|-------------|
| `NDP-RESOLVE-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | `nwp://` address could not be resolved |
| `NDP-RESOLVE-AMBIGUOUS` | `NPS-CLIENT-CONFLICT` | Conflicting resolution results (multiple inconsistent registrations) |
| `NDP-RESOLVE-TIMEOUT` | `NPS-SERVER-TIMEOUT` | Resolution request timed out |
| `NDP-ANNOUNCE-SIGNATURE-INVALID` | `NPS-AUTH-UNAUTHENTICATED` | AnnounceFrame signature verification failed |
| `NDP-ANNOUNCE-NID-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | NID in AnnounceFrame does not match the signing certificate |
| `NDP-ANNOUNCE-ROLE-REMOVED` | `NPS-CLIENT-BAD-FRAME` | `node_roles` contains the removed legacy value `"gateway"` (NPS-CR-0001); response SHOULD include a `hint` pointing to NPS-CR-0001 |
| `NDP-ANNOUNCE-ROLE-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` | `node_roles` contains an unrecognized value (see `NDP-ANNOUNCE-ROLE-REMOVED` for the `"gateway"` case specifically) |
| `NDP-GRAPH-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | GraphFrame sequence numbers are not contiguous |
| `NDP-REGISTRY-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | NDP Registry temporarily unavailable |

HTTP-mode status mapping: see [status-codes.md](./status-codes.md).

---

## 7. Security Considerations

### 7.1 Announce anti-forgery
Receivers MUST verify the `signature` on every AnnounceFrame to confirm the publisher holds the private key for the claimed NID; this prevents fake node announcements.

### 7.2 Registry anti-pollution
A centralized registry (NPS Cloud) MUST require a valid IdentFrame from the announcer and verify it through NIP CA before admitting an AnnounceFrame to the registry.

---

## 8. Change Log

| Version | Date | Changes |
|---------|------|---------|
| 0.6 | 2026-05-01 | **Breaking rename (pre-1.0)**: `node_kind` renamed to `node_roles` (array of strings only; single-string form retired). Parsers MUST accept `node_kind` as a parse-time alias through alpha.5 for backward compat. Constraint added: NWP NWM `node_type` MUST be one of the values in `node_roles` (see NWP §2.1 Node Role Resolution). Fixes M1 naming-disambiguation issue — `node_kind` (multi-role discovery field) and `node_type` (single operative role) were confusingly similar with no documented cross-protocol constraint. |
| 0.5 | 2026-04-26 | AnnounceFrame (0x30) gains three additive fields supporting NPS-CR-0001 — `node_kind` (string OR array of strings; values `"memory"`/`"action"`/`"complex"`/`"anchor"`/`"bridge"`; legacy `"gateway"` rejected), `cluster_anchor` (NID — for non-Anchor members of a cluster), `bridge_protocols` (array of strings — for `"bridge"` nodes; standard values `"http"`/`"grpc"`/`"mcp"`/`"a2a"`). All additive and backward-compatible: pre-alpha.3 publishers omit `node_kind` and receivers fall back to `node_type`. Depends-On upgraded to NCP v0.6 (NPS-RFC-0001) + NIP v0.4 (NPS-RFC-0003). |
| 0.4 | 2026-04-24 | AnnounceFrame (0x30) gains three additive fields — `activation_mode` (required for NPS-Node Profile L1+), `activation_endpoint` (required for `resident` / `hybrid`), `spawn_spec_ref` (L3, optional). New §3.1.1 Activation semantics with backward-compatibility rule for pre-alpha.3 publishers. `Depends-On` NCP version corrected to v0.5. |
| 0.3 | 2026-04-19 | Status Draft → Proposed; bilingual unification (EN primary + CN mirror); no wire-layer change |
| 0.2 | 2026-04-12 | Unified port 17433; error codes switched to NPS-status-code mapping; error-code list completed |
| 0.1 | 2026-04-10 | Initial skeleton: AnnounceFrame / ResolveFrame / GraphFrame, DNS TXT spec, initialization flow |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
