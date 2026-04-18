English | [中文版](./NPS-4-NDP.md)

# NPS-4: Neural Discovery Protocol (NDP)

**Spec Number**: NPS-4  
**Status**: Draft  
**Version**: 0.2  
**Date**: 2026-04-12  
**Port**: 17433 (default, shared) / 17436 (optional dedicated)  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Depends-On**: NPS-1 (NCP v0.3), NPS-3 (NIP v0.2)  

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
| `signature` | string | required | Signature with IdentFrame private key; prevents forgery |

**Example**

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
  "signature": "ed25519:..."
}
```

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

---

## 6. Error Codes

| Error Code | NPS Status | Description |
|------------|------------|-------------|
| `NDP-RESOLVE-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | `nwp://` address could not be resolved |
| `NDP-RESOLVE-AMBIGUOUS` | `NPS-CLIENT-CONFLICT` | Conflicting resolution results (multiple inconsistent registrations) |
| `NDP-RESOLVE-TIMEOUT` | `NPS-SERVER-TIMEOUT` | Resolution request timed out |
| `NDP-ANNOUNCE-SIGNATURE-INVALID` | `NPS-AUTH-UNAUTHENTICATED` | AnnounceFrame signature verification failed |
| `NDP-ANNOUNCE-NID-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | NID in AnnounceFrame does not match the signing certificate |
| `NDP-GRAPH-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | GraphFrame sequence numbers are not contiguous |
| `NDP-REGISTRY-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | NDP Registry temporarily unavailable |

HTTP-mode status mapping: see [status-codes.en.md](../status-codes.en.md).

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
| 0.2 | 2026-04-12 | Unified port 17433; error codes switched to NPS-status-code mapping; error-code list completed |
| 0.1 | 2026-04-10 | Initial skeleton: AnnounceFrame / ResolveFrame / GraphFrame, DNS TXT spec, initialization flow |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
