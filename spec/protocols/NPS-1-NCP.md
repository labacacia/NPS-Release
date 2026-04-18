English | [中文版](./NPS-1-NCP.cn.md)

# NPS-1: Neural Communication Protocol (NCP)

**Spec Number**: NPS-1  
**Status**: Draft  
**Version**: 0.4  
**Date**: 2026-04-14  
**Port**: 17433 (default, shared across the protocol suite)  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  

> This document is the NCP detailed specification. For the suite overview, see [NPS-0-Overview.md](NPS-0-Overview.md).

---

## 1. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be interpreted as described in RFC 2119.

---

## 2. Protocol Overview

NCP is the framing and communication base layer of NPS. It defines the frame structure, encoding tiers, and semantic-compression mechanism for AI-to-AI communication. All higher-layer protocols (NWP / NIP / NDP / NOP) are carried as NCP frames.

### 2.1 Roles

- **Sender**: the party that originates a frame (Agent or node).
- **Receiver**: the party that accepts and processes a frame.
- **Relay**: transparently forwards frames without modifying the payload (optional, for proxy scenarios).

### 2.2 Transport Modes

NCP supports two transport modes. The frame format is identical in both; only the carrier differs.

| Mode | Carrier | Use case | Recommended phase |
|------|---------|---------|-------------------|
| **HTTP mode** | NCP frames serialized in the HTTP body, paired with `X-NWP-*` request headers | Overlay deployment, compatible with existing Web infrastructure, firewall-friendly | Recommended for Phase 1 |
| **Native mode** | Direct TCP/QUIC connection; NCP frame is the on-the-wire format with no HTTP overhead | High-performance Agent-to-Agent, low-latency scenarios | Phase 2+ |

**HTTP mode**

```
POST /nwp/products/query HTTP/1.1
Host: api.example.com:17433
Content-Type: application/nwp-frame
X-NWP-Agent: urn:nps:agent:ca.innolotus.com:550e8400

[NCP Frame bytes / JSON]
```

**Native mode**

```
TCP connect → api.example.com:17433
→ HelloFrame (handshake, 0x06)
← CapsFrame (capability negotiation, 0x04)
→ QueryFrame (0x10)
← CapsFrame (response, 0x04)
```

The frame payload is identical in both modes. In native mode, errors are returned via ErrorFrame (0xFE); in HTTP mode, the HTTP status code is returned in parallel (mapping: [status-codes.md](../status-codes.md)).

### 2.3 Unified Port

The whole NPS suite shares **port 17433** by default. Frames from different protocols are routed by the Frame Type byte:

| Frame type range | Protocol |
|------------------|----------|
| 0x01–0x0F | NCP |
| 0x10–0x1F | NWP |
| 0x20–0x2F | NIP |
| 0x30–0x3F | NDP |
| 0x40–0x4F | NOP |
| 0xF0–0xFF | Reserved (including ErrorFrame 0xFE) |

Implementations MAY assign dedicated ports per protocol for isolated deployment, but default behavior MUST support single-port multiplexing.

### 2.4 Frame-Exchange Patterns

| Pattern | Description | Typical frame sequence |
|---------|-------------|------------------------|
| Request-response | Single query or operation | QueryFrame → CapsFrame |
| Streaming push | Large datasets or real-time data | QueryFrame → StreamFrame × N → StreamFrame(is_last=true) |
| Incremental subscription | Change notifications | subscribe → DiffFrame × N |
| Multi-agent sync | Task-state alignment | AlignStream × N (see NPS-5) |

### 2.5 Connection State Machine

```
┌──────────┐  connect()  ┌──────────────────┐  anchor_ok  ┌─────────────┐
│  CLOSED  │ ──────────→ │ ANCHOR_NEGOTIATE  │ ──────────→ │ ESTABLISHED │
└──────────┘             └──────────────────┘             └─────────────┘
                                │ timeout/error                  │ close()
                                ↓                                ↓
                          ┌──────────┐                   ┌──────────────┐
                          │  FAILED  │                   │   CLOSING    │
                          └──────────┘                   └──────────────┘
                                                                 │
                                                                 ↓
                                                           ┌──────────┐
                                                           │  CLOSED  │
                                                           └──────────┘
```

### 2.6 Connection Handshake Sequence (Native Mode)

In native mode, connection setup MUST follow the handshake below:

```
Client (Agent)                        Server (Node)
      │                                     │
      │── TCP/QUIC connect ──────────────→  │
      │── HelloFrame (0x06) ─────────────→  │  declare client version & capabilities
      │                                     │  version negotiation & capability intersection
      │  ←──────────────── CapsFrame (0x04) │  return negotiated server capabilities
      │                                     │  (on incompatibility: ErrorFrame + disconnect)
      │  [optional] preload AnchorFrame     │
      │── GET /.schema ──────────────────→  │
      │  ←──────────────── AnchorFrame(s)   │
      │                                     │
      │  ── ESTABLISHED ───────────────── ESTABLISHED
```

**Version negotiation rules**

- Server MUST pick `min(client.nps_version, server.nps_version)` as the session version.
- If `client.min_version > server.nps_version`, Server MUST return `NCP-VERSION-INCOMPATIBLE` and close the connection.
- The negotiated encoding = best intersection of `client.supported_encodings ∩ server.supported_encodings` (Tier-2 preferred).
- The negotiated `max_frame_payload` = `min(client.max_frame_payload, server.max_frame_payload)`.

**Error handling**

If the handshake fails (incompatible version, empty capability set, etc.), Server MUST return:

```json
{
  "frame": "0xFE",
  "status": "NPS-PROTO-VERSION-INCOMPATIBLE",
  "error": "NCP-VERSION-INCOMPATIBLE",
  "message": "No compatible NPS version",
  "details": { "server_version": "0.4", "client_min_version": "0.5" }
}
```

---

## 3. Frame Format

### 3.1 Fixed Header (4 bytes, default) / Extended Header (8 bytes, large-frame mode)

**Default frame header (4 bytes)**

```
 Byte 0          Byte 1          Byte 2–3
┌───────────────┬───────────────┬───────────────────────────┐
│  Frame Type   │     Flags     │      Payload Length       │
│   (1 byte)    │   (1 byte)    │        (2 bytes, BE)      │
└───────────────┴───────────────┴───────────────────────────┘
```

**Extended frame header (8 bytes, EXT=1)**

```
 Byte 0          Byte 1          Byte 2–5               Byte 6–7
┌───────────────┬───────────────┬───────────────────────┬──────────────┐
│  Frame Type   │     Flags     │   Payload Length      │   Reserved   │
│   (1 byte)    │   (1 byte)    │   (4 bytes, BE)       │  (2 bytes)   │
└───────────────┴───────────────┴───────────────────────┘──────────────┘
```

- **Frame Type**: frame-type byte, see the frame namespace (§2.3).
- **Flags**: control bits, see §3.2.
- **Payload Length**: 2 bytes by default (max 65,535); 4 bytes in extended mode (max 4,294,967,295).

### 3.2 Flags Field (Bit-level Definition)

```
Bit 7   Bit 6   Bit 5   Bit 4   Bit 3   Bit 2   Bit 1   Bit 0
┌───────┬───────┬───────┬───────┬───────┬───────┬───────┬───────┐
│  EXT  │  RSV  │  RSV  │  RSV  │  ENC  │ FINAL │  T1   │  T0  │
└───────┴───────┴───────┴───────┴───────┴───────┴───────┴───────┘
```

| Bit | Name | Description |
|-----|------|-------------|
| 0–1 | T0, T1 | Encoding tier: `00`=Tier-1 JSON, `01`=Tier-2 MsgPack, `10`=Reserved (former Tier-3 slot), `11`=Reserved |
| 2 | FINAL | Final-chunk flag for streaming frames (StreamFrame only; fixed to 1 elsewhere) |
| 3 | ENC | Payload uses application-layer E2E encryption (see §7.4); TLS is independent, MAY be 0 in dev mode |
| 4–6 | RSV | Reserved; senders MUST set to 0, receivers MUST ignore |
| 7 | EXT | Extended-header flag: `0`=default 4-byte header (payload ≤ 64 KB), `1`=extended 8-byte header (payload ≤ 4 GB) |

### 3.3 Frame-Size Configuration

| Parameter | Default | Max | Description |
|-----------|---------|-----|-------------|
| `max_frame_payload` | 65,535 (64 KB) | 4,294,967,295 (4 GB) | Maximum payload bytes per frame |

- Default header (EXT=0): payload up to **65,535 bytes**, sufficient for most scenarios.
- Extended header (EXT=1): payload up to **4,294,967,295 bytes**, for large embeddings or bulk document transfer.
- Implementations MUST support the default mode; SHOULD support extended mode.
- When the payload exceeds the negotiated `max_frame_payload`, the sender MUST chunk via StreamFrame.
- `max_frame_payload` is negotiated via CapsFrame at connection setup.

**Chunking rules**

- StreamFrame chunking: `seq` starts at 0 and increments; last chunk has FINAL=1.
- Each chunk's frame MUST respect the connection's negotiated `max_frame_payload`.

---

## 4. Frame Types

### 4.1 AnchorFrame (0x01)

Schema-anchor frame, **published by a Node**, used to establish a global Schema reference and eliminate redundant transmission.

**Fields**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | required | Fixed value `0x01` |
| `anchor_id` | string | required | SHA-256 digest of the Schema, formatted `sha256:{64 hex chars}` |
| `schema` | object | required | Schema definition object |
| `schema.fields` | array | required | Field descriptors (see below) |
| `ttl` | uint32 | optional | Cache validity in seconds; default 3600; `0` = do not cache |

**schema.fields element**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | required | Field name |
| `type` | string | required | Data type: `string` / `uint64` / `int64` / `decimal` / `bool` / `timestamp` / `bytes` / `object` / `array` |
| `semantic` | string | optional | Semantic annotation, format `{domain}.{concept}` (see semantic type system) |
| `nullable` | bool | optional | Whether null is permitted; default false |

**anchor_id computation rules**

1. Serialize `schema` to canonical JSON per **RFC 8785 (JSON Canonicalization Scheme, JCS)**.
   - JCS guarantees Unicode normalization, object-key ordering by UTF-16, and no extra whitespace.
   - Implementations MUST use a standard JCS library (do not implement your own) to keep .NET / Python / TS SDKs consistent.
2. Compute the SHA-256 digest of the JCS output as UTF-8 bytes.
3. Format as `sha256:{lowercase hex}`.

> **Note**: For JCS reference implementations, see https://www.rfc-editor.org/rfc/rfc8785#appendix-A

**Example**

```json
{
  "frame": "0x01",
  "anchor_id": "sha256:a3f9b2c1d4e5f6789012345678901234567890abcdef1234567890abcdef12",
  "schema": {
    "fields": [
      { "name": "id",    "type": "uint64",  "semantic": "entity.id" },
      { "name": "name",  "type": "string",  "semantic": "entity.label" },
      { "name": "price", "type": "decimal", "semantic": "commerce.price.usd" },
      { "name": "stock", "type": "uint64",  "semantic": "commerce.inventory.count" }
    ]
  },
  "ttl": 3600
}
```

**Semantic type system (partial)**

| Semantic | Meaning |
|----------|---------|
| `entity.id` | Unique entity identifier |
| `entity.label` | Human-readable entity name |
| `entity.description` | Entity description text |
| `entity.timestamp.created` | Creation time |
| `entity.timestamp.updated` | Last-updated time |
| `commerce.price.usd` | USD price |
| `commerce.price.cny` | CNY price |
| `commerce.inventory.count` | Inventory count |
| `geo.latitude` | Latitude |
| `geo.longitude` | Longitude |

---

### 4.2 DiffFrame (0x02)

Incremental-data frame carrying only a patch of changed fields, for subscription or polling scenarios.

**Fields**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | required | Fixed value `0x02` |
| `anchor_ref` | string | required | anchor_id of the base Schema |
| `base_seq` | uint64 | required | Version sequence this diff is based on |
| `patch_format` | string | optional | Patch format: `json_patch` (default) or `binary_bitset` (recommended for Tier-2) |
| `patch` | array \| bytes | required | Patch data (format decided by `patch_format`, see below) |
| `entity_id` | string | optional | ID of the entity being updated |

**patch_format: json_patch** (default, for Tier-1 JSON)

`patch` is an array of JSON Patch operations (RFC 6902); each element has `op`, `path`, and `value`.

**patch_format: binary_bitset** (Tier-2 MsgPack, ~15–20% smaller)

`patch` is a binary byte stream (MsgPack `bin` type) structured as:

```
┌──────────────────────────────────────────────────────┐
│  Changed Fields Bitset (ceil(N/8) bytes, N=field count)│
│  Bit i=1 indicates the i-th field changed (schema.fields order)│
├──────────────────────────────────────────────────────┤
│  Values (new values for changed fields only, in field order, MsgPack-encoded)│
└──────────────────────────────────────────────────────┘
```

Rules:
- Senders MUST only use `binary_bitset` inside Tier-2 (MsgPack) frames.
- Receivers that do not support `binary_bitset` MUST return `NCP-DIFF-FORMAT-UNSUPPORTED`.
- When `patch_format` is omitted, default is `json_patch`.

**Example (json_patch)**

```json
{
  "frame": "0x02",
  "anchor_ref": "sha256:a3f9b2c1...",
  "base_seq": 42,
  "patch_format": "json_patch",
  "entity_id": "product:1001",
  "patch": [
    { "op": "replace", "path": "/price", "value": 299.00 },
    { "op": "replace", "path": "/stock", "value": 48 }
  ]
}
```

---

### 4.3 StreamFrame (0x03)

Streaming data-chunk frame, used for large datasets, real-time push, or fragmentation of oversized payloads.

**Fields**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | required | Fixed value `0x03` |
| `stream_id` | string | required | Unique stream identifier (UUID v4) |
| `seq` | uint32 | required | Chunk sequence, starts at 0 and increments; receivers reassemble by `seq` |
| `is_last` | bool | required | `true` marks the final chunk |
| `anchor_ref` | string | optional | Schema anchor_id (carried on the first chunk; may be omitted later) |
| `data` | array | required | Chunk data, conforming to the referenced Schema |
| `window_size` | uint32 | optional | Remaining frames the receiver can buffer (see flow-control semantics) |
| `error_code` | string | optional | Present when the stream aborts abnormally; `is_last` is forced to true |

**Flow-control semantics**

NCP provides application-layer semantic back-pressure, complementing TCP/QUIC transport-layer flow control:

- **Initial window**: the sender carries `window_size` in the first frame (seq=0), declaring how many frames the receiver can still accept.
- **Window consumption**: the receiver's logical window decrements by 1 per received frame.
- **Window update**: when the receiver is ready for more data, it sends a reverse StreamFrame (`data=[]`, `is_last=false`) carrying a new `window_size` to refresh the window.
- **Back-pressure pause**: if the receiver sends `window_size=0`, the sender MUST pause until it receives a non-zero `window_size`.
- **Omission semantics**: absence of `window_size` means application-layer flow control is not enabled (rely on the transport).
- If the sender continues after the window is exhausted, the receiver SHOULD return `NCP-STREAM-WINDOW-OVERFLOW` and MAY terminate the stream.

---

### 4.4 CapsFrame (0x04)

Encapsulates a full response body; the most common response frame. Also used for capability negotiation at connection setup.

**Fields**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | required | Fixed value `0x04` |
| `anchor_ref` | string | required | Schema anchor_id |
| `count` | uint32 | required | Length of `data`; MUST equal `len(data)` |
| `data` | array | required | Data records; each conforms to the anchor_ref Schema |
| `next_cursor` | string | optional | Next-page cursor, Base64-URL encoded; null = last page |
| `token_est` | uint32 | optional | Estimated NPT consumption for this response (see [token-budget.md](../token-budget.md)) |
| `tokenizer_used` | string | optional | Identifier of the tokenizer actually applied |
| `cached` | bool | optional | `true` = response came from server-side cache |
| `inline_anchor` | object | optional | Latest AnchorFrame included inline when the Schema has been updated, avoiding an extra RTT (see §5.4) |

**Connection-negotiation CapsFrame**

During native-mode connection setup, the Node returns its negotiated capabilities via CapsFrame:

```json
{
  "frame": "0x04",
  "anchor_ref": "nps:system:caps",
  "count": 1,
  "data": [{
    "nps_version": "0.4",
    "session_version": "0.4",
    "max_frame_payload": 65535,
    "negotiated_encoding": "msgpack",
    "supported_protocols": ["ncp", "nwp", "nip"],
    "ext_support": true,
    "max_concurrent_streams": 32,
    "e2e_enc_algorithms": ["aes-256-gcm", "chacha20-poly1305"]
  }]
}
```

**Data-response example**

```json
{
  "frame": "0x04",
  "anchor_ref": "sha256:a3f9b2c1...",
  "count": 2,
  "data": [
    { "id": 1001, "name": "iPhone 15 Pro", "price": 999.00, "stock": 42 },
    { "id": 1002, "name": "MacBook Air M3", "price": 1299.00, "stock": 15 }
  ],
  "next_cursor": "eyJpZCI6MTAwM30",
  "token_est": 180
}
```

---

### 4.5 AlignFrame (0x05) — Deprecated

> ⚠️ AlignFrame was marked **Deprecated** in NCP v0.2.  
> Use NOP AlignStream (0x43) instead, see [NPS-5-NOP.md](NPS-5-NOP.md).  
> AlignFrame will be removed in NPS v1.0.

---

### 4.6 HelloFrame (0x06)

Client-side handshake frame in native mode, **sent by the Agent (client) first after the connection is established**, declaring the client's protocol version and capabilities. The Server responds with a CapsFrame (see §2.6).

> **Naming note**: The NIP layer defines IdentFrame (0x20) for identity-certificate exchange. NCP HelloFrame is only for version and capability negotiation and does not carry identity; the responsibilities are cleanly separated.

**Fields**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | required | Fixed value `0x06` |
| `nps_version` | string | required | Highest NPS version the client supports; format `"major.minor"` |
| `min_version` | string | optional | Lowest NPS version the client supports; defaults to `nps_version` |
| `supported_encodings` | array[string] | required | Supported encodings, e.g. `["json", "msgpack"]`, in descending priority |
| `supported_protocols` | array[string] | required | Supported higher-layer protocols, e.g. `["ncp", "nwp", "nip"]` |
| `agent_id` | string | optional | Agent NID (NIP identity identifier), format `urn:nps:agent:{domain}:{id}` |
| `max_frame_payload` | uint32 | optional | Maximum payload bytes the client can accept; default 65535 |
| `ext_support` | bool | optional | Whether extended header (EXT=1) is supported; default false |
| `max_concurrent_streams` | uint32 | optional | Maximum concurrent streams the client can handle; default 32 |
| `e2e_enc_algorithms` | array[string] | optional | E2E encryption algorithms the client supports (see §7.4), e.g. `["aes-256-gcm"]` |

**Rules**

- HelloFrame MUST be the first frame the client sends in native mode; nothing else may precede it.
- HTTP mode does not use HelloFrame (capabilities flow via `X-NWP-*` headers).
- HelloFrame is unencrypted (ENC=0) and SHOULD use Tier-1 JSON (the encoding has not yet been negotiated).
- The Server MUST reply with a CapsFrame or ErrorFrame within 5 seconds of receiving HelloFrame; otherwise the client SHOULD disconnect.

**HelloFrame example**

```json
{
  "frame": "0x06",
  "nps_version": "0.4",
  "min_version": "0.3",
  "supported_encodings": ["msgpack", "json"],
  "supported_protocols": ["ncp", "nwp"],
  "agent_id": "urn:nps:agent:ca.innolotus.com:550e8400",
  "max_frame_payload": 65535,
  "ext_support": false,
  "max_concurrent_streams": 16,
  "e2e_enc_algorithms": ["aes-256-gcm", "chacha20-poly1305"]
}
```

---

### 4.7 ErrorFrame (0xFE)

The unified NPS error frame, shared across all protocol layers. Carries error responses in native mode.

**Fields**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | required | Fixed value `0xFE` |
| `status` | string | required | NPS status code (see [status-codes.md](../status-codes.md)) |
| `error` | string | required | Protocol-level error code (e.g. `NCP-ANCHOR-NOT-FOUND`) |
| `message` | string | optional | Human-readable error description |
| `details` | object | optional | Structured error context (e.g. `anchor_ref`, `stream_id`) |

**Example**

```json
{
  "frame": "0xFE",
  "status": "NPS-CLIENT-NOT-FOUND",
  "error": "NCP-ANCHOR-NOT-FOUND",
  "message": "Schema anchor not found in cache, please resend AnchorFrame",
  "details": { "anchor_ref": "sha256:a3f9b2c1..." }
}
```

---

## 5. Schema Anchoring Mechanism

### 5.1 Schema Ownership

AnchorFrame is published by the **Node (the data-model owner)**. The Agent fetches the AnchorFrame via the node manifest or a Schema endpoint, caches it locally, and references it in subsequent requests.

### 5.2 Standard Flow

```
Agent                              Node
  │                                  │
  │── GET /.nwm ─────────────────→│  read manifest, fetch schema_anchors
  │←── NWM(schema_anchors) ───────│
  │                                  │
  │── GET /.schema ──────────────→│  fetch full AnchorFrame (optional, on-demand)
  │←── AnchorFrame(schema) ───────│  Agent caches anchor_id → schema locally
  │                                  │
  │── QueryFrame(anchor_ref) ────→│  request carries only anchor_ref
  │←── CapsFrame(anchor_ref) ─────│  response also only references anchor_ref
  │                                  │
  │── QueryFrame(anchor_ref) ────→│  no Schema retransmission on later requests
  │←── CapsFrame(anchor_ref) ─────│
```

### 5.3 Cache Semantics

- The Agent MUST cache AnchorFrames fetched from a Node, keyed by `anchor_id`.
- The cache validity equals the AnchorFrame's `ttl` in seconds.
- The Agent SHOULD preload all AnchorFrames likely to be used after the connection is established.

### 5.4 Cache-Miss Handling

#### 5.4.1 Schema Updated (Stale anchor_ref)

If the Agent references an outdated `anchor_ref` and the Node has a newer version, behavior depends on the request's `auto_anchor` flag (defined in NWP QueryFrame):

- **auto_anchor=true (default)**: the Node SHOULD attach an `inline_anchor` field in the CapsFrame response containing the latest AnchorFrame, avoiding an extra RTT:

```json
{
  "frame": "0x04",
  "anchor_ref": "sha256:new_anchor...",
  "count": 2,
  "data": [ ... ],
  "inline_anchor": {
    "frame": "0x01",
    "anchor_id": "sha256:new_anchor...",
    "schema": { "fields": [ ... ] },
    "ttl": 3600
  }
}
```

  Upon receiving a response with `inline_anchor`, the Agent MUST update its local cache (replacing the Schema for the old `anchor_id`).

- **auto_anchor=false**: the Node SHOULD return `NCP-ANCHOR-STALE` with the old anchor_id, letting the Agent proactively fetch the new AnchorFrame.

#### 5.4.2 Unknown anchor_ref

If the Agent references an `anchor_ref` the Node does not recognize (hand-crafted, cross-node misreference, etc.), the Node MUST return:

```json
{
  "frame": "0xFE",
  "status": "NPS-CLIENT-NOT-FOUND",
  "error": "NCP-ANCHOR-NOT-FOUND",
  "details": { "anchor_ref": "sha256:..." }
}
```

> **Note**: `NCP-ANCHOR-STALE` means the anchor exists but has been updated; `NCP-ANCHOR-NOT-FOUND` means the anchor was never registered at this Node.

---

## 6. Status Codes and Error Codes

NPS uses a two-tier error model:

1. **NPS status codes**: transport-level status categories, see [status-codes.md](../status-codes.md).
2. **Protocol error codes**: specific error identifiers, prefixed by the owning protocol.

### NCP Error Codes

| Error code | NPS status | Description |
|------------|------------|-------------|
| `NCP-ANCHOR-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | Referenced Schema not found |
| `NCP-ANCHOR-SCHEMA-INVALID` | `NPS-CLIENT-BAD-FRAME` | Schema in AnchorFrame is malformed |
| `NCP-ANCHOR-ID-MISMATCH` | `NPS-CLIENT-CONFLICT` | Same anchor_id received with different Schema (anchor-poisoning defense) |
| `NCP-FRAME-UNKNOWN-TYPE` | `NPS-CLIENT-BAD-FRAME` | Unknown frame-type byte |
| `NCP-FRAME-PAYLOAD-TOO-LARGE` | `NPS-LIMIT-PAYLOAD` | Payload exceeds negotiated max_frame_payload |
| `NCP-FRAME-FLAGS-INVALID` | `NPS-CLIENT-BAD-FRAME` | Reserved bits in Flags are non-zero |
| `NCP-STREAM-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | Non-contiguous StreamFrame sequence |
| `NCP-STREAM-NOT-FOUND` | `NPS-STREAM-NOT-FOUND` | stream_id does not refer to an existing stream |
| `NCP-STREAM-LIMIT-EXCEEDED` | `NPS-STREAM-LIMIT` | Max concurrent streams per connection exceeded |
| `NCP-ENCODING-UNSUPPORTED` | `NPS-SERVER-ENCODING-UNSUPPORTED` | Requested encoding tier is not supported |
| `NCP-ANCHOR-STALE` | `NPS-CLIENT-CONFLICT` | anchor_ref exists but the Schema has been updated (paired with inline_anchor) |
| `NCP-DIFF-FORMAT-UNSUPPORTED` | `NPS-CLIENT-BAD-FRAME` | DiffFrame used a patch_format the receiver does not support |
| `NCP-VERSION-INCOMPATIBLE` | `NPS-PROTO-VERSION-INCOMPATIBLE` | Client min_version exceeds server's supported version |
| `NCP-STREAM-WINDOW-OVERFLOW` | `NPS-STREAM-LIMIT` | Sender continued after the flow-control window was exhausted |

HTTP-mode status mapping: see [status-codes.md](../status-codes.md).

---

## 7. Security Considerations

### 7.1 Replay-Attack Defense
CapsFrame and QueryFrame should be protected at the TLS layer. In non-TLS scenarios (dev mode), the Agent SHOULD include a `nonce` field.

### 7.2 Anchor Poisoning
Schemas are published by the Node and referenced read-only by the Agent. Nodes MUST enforce idempotency on anchor_id — the same anchor_id MUST always correspond to the same Schema. If the Agent observes a differing Schema under the same anchor_id, it SHOULD disconnect and report a security event.

### 7.3 Stream Flooding
Nodes SHOULD limit max concurrent streams per connection (recommended default: 32, negotiated via CapsFrame `max_concurrent_streams`). On overflow, return `NCP-STREAM-LIMIT-EXCEEDED`.

### 7.4 E2E Encryption (Formal Definition of the ENC Flag)

When frame-header flag **ENC=1**, the frame payload uses application-layer end-to-end encryption. E2E encryption is independent of TLS and applies where a Relay across multiple hops is untrusted.

**Algorithm selection**

Negotiated via the `e2e_enc_algorithms` field in the handshake CapsFrame, priority descending:

| Algorithm | Identifier | Key length | Nonce | Tag |
|-----------|-----------|------------|-------|-----|
| AES-256-GCM | `aes-256-gcm` | 256-bit | 12 bytes | 16 bytes |
| ChaCha20-Poly1305 | `chacha20-poly1305` | 256-bit | 12 bytes | 16 bytes |

**Payload layout when ENC=1**

```
┌────────────────┬──────────────────────────────────────┬─────────────┐
│   Nonce        │   Encrypted Payload                  │  Auth Tag   │
│  (12 bytes)    │   (ciphertext of original payload)   │  (16 bytes) │
└────────────────┴──────────────────────────────────────┴─────────────┘
```

- `Payload Length` (in the header) = 12 + len(encrypted) + 16
- The nonce MUST be unique per frame; random generation via a CSPRNG is recommended.
- AAD (Additional Authenticated Data) = the frame header (4 or 8 bytes).
- Key management: E2E keys are distributed through NIP (Neural Identity Protocol) key-exchange and are not defined at the NCP layer.

**Implementation requirements**

- If HelloFrame's `e2e_enc_algorithms` is empty, E2E encryption is not supported and neither side MUST set ENC=1.
- If ENC=1 but no E2E algorithm has been negotiated for the session, the receiver MUST return `NCP-ENC-NOT-NEGOTIATED` and drop the frame.
- In dev/debug mode (non-production only), ENC MAY be 0, but implementations SHOULD log a warning.

**New error codes**

| Error code | NPS status | Description |
|------------|------------|-------------|
| `NCP-ENC-NOT-NEGOTIATED` | `NPS-CLIENT-BAD-FRAME` | ENC=1 but no E2E algorithm negotiated for this session |
| `NCP-ENC-AUTH-FAILED` | `NPS-CLIENT-BAD-FRAME` | E2E Auth Tag verification failed (possible tampering) |

---

## 8. Encoding Tiers

All frame types support the following encoding tiers, selected via the T0/T1 bits of Flags:

| Tier | Format | Flag | Use case |
|------|--------|------|----------|
| Tier-1 | JSON | `00` | Development, debugging, compatibility mode |
| Tier-2 | MsgPack (binary) | `01` | Production, ~60% size reduction |
| — | Reserved | `10` | Reserved for future high-performance encoding |
| — | Reserved | `11` | Reserved |

Defaults: Tier-2 for production, Tier-1 for development.

> **Note**: The former Tier-3 MatrixTensor concept has been removed from the tier system; `10` is marked Reserved. If a high-performance encoding format (e.g. dedicated vector/tensor encoding) is defined in the future, it will be allocated through a formal RFC process.

---

## 9. Implementation Notes

- Recommended AnchorFrame cache limit: 1000 entries per connection, LRU-evicted on overflow.
- MsgPack (Tier-2) codec: use the official MessagePack library; do not implement your own.
- When StreamFrame `seq` overflows (reaches `0xFFFFFFFF` as uint32), the new stream MUST use a new `stream_id`.
- Support for extended header (EXT=1) SHOULD be declared in the CapsFrame capability negotiation.
- **anchor_id computation**: use an RFC 8785 JCS standard library; do not implement canonicalization yourself (cross-language SDK consistency).
- **HelloFrame timeout**: if the Server does not receive a HelloFrame within 5 seconds, it SHOULD send an ErrorFrame and close the connection.
- **DiffFrame binary_bitset**: only after both sides have negotiated support via CapsFrame; forbidden in Tier-1 JSON mode.
- **E2E encryption nonce**: a per-frame random nonce (12 bytes, CSPRNG); never reuse; key rotation SHOULD be at least once per 2^32 frames.
- **inline_anchor handling**: after receiving `inline_anchor`, the Agent MUST first verify the anchor_id (JCS + SHA-256), then update the cache.
- **Tier-2 performance optimization (.NET)**: `Tier2MsgPackCodec` is recommended to use MessagePack AOT code generation to avoid reflection overhead in high-frequency frame parsing (implementation guidance; does not affect protocol interoperability).

---

## 10. Change Log

| Version | Date | Changes |
|---------|------|---------|
| 0.4 | 2026-04-14 | Added HelloFrame (0x06); new §2.6 handshake sequence & version-negotiation rules; anchor_id computation now explicitly references RFC 8785 JCS; DiffFrame gains `patch_format` (json_patch / binary_bitset); CapsFrame gains `inline_anchor`; StreamFrame flow control formalized (window_size protocol); §7.4 E2E encryption section (ENC flag, AES-256-GCM / ChaCha20-Poly1305, payload layout); §5.4 auto-anchor protocol (NCP-ANCHOR-STALE + inline_anchor); added error codes NCP-ANCHOR-STALE, NCP-DIFF-FORMAT-UNSUPPORTED, NCP-VERSION-INCOMPATIBLE, NCP-STREAM-WINDOW-OVERFLOW, NCP-ENC-NOT-NEGOTIATED, NCP-ENC-AUTH-FAILED |
| 0.3 | 2026-04-12 | Dual transport (HTTP / native); unified port 17433; configurable frame size (EXT bit); ErrorFrame (0xFE); NPS status-code system; Tier-3 marked Reserved; AnchorFrame ownership clarified as Node-published; token estimation switched to NPT |
| 0.2 | 2026-04-10 | AnchorFrame / DiffFrame / StreamFrame / CapsFrame / AlignFrame definitions; bit-level Flags definition; AlignFrame marked Deprecated |
| 0.1 | 2026-03-01 | Initial frame format and encoding tiers |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
